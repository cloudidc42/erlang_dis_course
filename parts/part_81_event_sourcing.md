# Part 81: Event Sourcing and CQRS

## สารบัญ
1. [Event Sourcing Concepts](#concepts)
2. [Event Store Implementation](#event-store)
3. [Aggregate Pattern](#aggregate)
4. [CQRS Read Models](#cqrs)
5. [Event Projections](#projections)
6. [ตัวอย่างจริง: Bank Account System](#bank-account)

---

## Event Sourcing Concepts

```
Event Sourcing: เก็บ events แทน state ปัจจุบัน

Traditional:
  users table → {id, name, email, balance}
  UPDATE users SET balance = 100 WHERE id = 1

Event Sourcing:
  events table → [{user_created, name, email},
                  {deposit, 100},
                  {withdrawal, 50}]
  State = fold(events, initial_state)

ข้อดี:
  - Audit trail สมบูรณ์ — รู้ทุก event ที่เกิดขึ้น
  - Time travel — reconstruct state ที่เวลาใดก็ได้
  - Event replay — rebuild read models
  - Debugging — rerun events ใน test

CQRS (Command Query Responsibility Segregation):
  Write side: Commands → Events → Aggregate
  Read side: Events → Projections → Read Models
```

---

## Event Store Implementation

```erlang
%% event_store.erl — append-only event storage

-module(event_store).
-export([append/3, load/2, load_from/3, subscribe/2]).

-record(event, {
    stream_id,
    sequence,
    event_type,
    data,
    metadata,
    occurred_at
}).

%% PostgreSQL-backed event store
create_schema() ->
    SQL = "
    CREATE TABLE IF NOT EXISTS events (
        id BIGSERIAL PRIMARY KEY,
        stream_id TEXT NOT NULL,
        sequence BIGINT NOT NULL,
        event_type TEXT NOT NULL,
        data JSONB NOT NULL,
        metadata JSONB DEFAULT '{}',
        occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        UNIQUE(stream_id, sequence)
    );
    CREATE INDEX IF NOT EXISTS events_stream_id_idx ON events (stream_id);
    CREATE INDEX IF NOT EXISTS events_occurred_at_idx ON events (occurred_at);
    ",
    db:execute(SQL, []).

%% Append events to a stream
append(StreamId, ExpectedVersion, Events) ->
    F = fun(Conn) ->
        %% Check current version (optimistic concurrency)
        {ok, CurrentVersion} = get_current_version(Conn, StreamId),
        
        case CurrentVersion of
            ExpectedVersion ->
                %% Append all events in sequence
                lists:foldl(fun(Event, Seq) ->
                    insert_event(Conn, StreamId, Seq, Event),
                    Seq + 1
                end, ExpectedVersion + 1, Events),
                ok;
            _ ->
                {error, {version_conflict, CurrentVersion, ExpectedVersion}}
        end
    end,
    
    epgsql:with_transaction(db_conn(), F).

insert_event(Conn, StreamId, Sequence, #{type := Type, data := Data} = Event) ->
    Meta = maps:get(metadata, Event, #{}),
    epgsql:equery(Conn,
        "INSERT INTO events (stream_id, sequence, event_type, data, metadata) "
        "VALUES ($1, $2, $3, $4, $5)",
        [StreamId, Sequence, atom_to_binary(Type), jiffy:encode(Data), jiffy:encode(Meta)]).

get_current_version(Conn, StreamId) ->
    case epgsql:equery(Conn, "SELECT COALESCE(MAX(sequence), 0) FROM events WHERE stream_id = $1",
                       [StreamId]) of
        {ok, _, [{Version}]} -> {ok, Version};
        _ -> {ok, 0}
    end.

%% Load all events for a stream
load(StreamId, StartFrom) ->
    {ok, _, Rows} = epgsql:equery(db_conn(),
        "SELECT sequence, event_type, data, metadata, occurred_at "
        "FROM events "
        "WHERE stream_id = $1 AND sequence >= $2 "
        "ORDER BY sequence",
        [StreamId, StartFrom]),
    
    [parse_event(R) || R <- Rows].

load_from(StreamId, StartFrom, EndAt) ->
    {ok, _, Rows} = epgsql:equery(db_conn(),
        "SELECT sequence, event_type, data, metadata, occurred_at "
        "FROM events "
        "WHERE stream_id = $1 AND sequence BETWEEN $2 AND $3 "
        "ORDER BY sequence",
        [StreamId, StartFrom, EndAt]),
    
    [parse_event(R) || R <- Rows].

parse_event({Seq, Type, DataJson, MetaJson, OccurredAt}) ->
    #event{
        sequence = Seq,
        event_type = binary_to_atom(Type),
        data = jiffy:decode(DataJson, [return_maps]),
        metadata = jiffy:decode(MetaJson, [return_maps]),
        occurred_at = OccurredAt
    }.

%% Subscribe to new events
subscribe(StreamId, Pid) ->
    %% Uses PostgreSQL LISTEN/NOTIFY or polling
    event_subscription_manager:subscribe(StreamId, Pid).
```

---

## Aggregate Pattern

```erlang
%% aggregate.erl — base behavior for event-sourced aggregates

-module(aggregate).
-export([load/2, execute/3]).

%% Load aggregate state by replaying events
load(AggregateModule, AggregateId) ->
    StreamId = stream_id(AggregateModule, AggregateId),
    Events = event_store:load(StreamId, 0),
    
    InitialState = AggregateModule:initial_state(),
    
    {State, Version} = lists:foldl(fun(#event{event_type = Type, data = Data, sequence = Seq}, {S, _}) ->
        NewState = AggregateModule:apply_event(S, Type, Data),
        {NewState, Seq}
    end, {InitialState, 0}, Events),
    
    {ok, State, Version}.

%% Execute a command on an aggregate
execute(AggregateModule, AggregateId, Command) ->
    {ok, State, Version} = load(AggregateModule, AggregateId),
    
    case AggregateModule:handle_command(State, Command) of
        {ok, Events} ->
            StreamId = stream_id(AggregateModule, AggregateId),
            case event_store:append(StreamId, Version, Events) of
                ok ->
                    publish_events(StreamId, Events),
                    {ok, Events};
                {error, {version_conflict, _, _}} ->
                    %% Retry on conflict
                    execute(AggregateModule, AggregateId, Command)
            end;
        {error, Reason} ->
            {error, Reason}
    end.

stream_id(Module, Id) ->
    ModName = atom_to_binary(Module),
    <<"stream:", ModName/binary, ":", Id/binary>>.

publish_events(StreamId, Events) ->
    [event_bus:publish(StreamId, Event) || Event <- Events].
```

---

## ตัวอย่างจริง: Bank Account System

```erlang
%% bank_account.erl — event-sourced bank account aggregate

-module(bank_account).
-export([initial_state/0, handle_command/2, apply_event/3]).
-export([open/2, deposit/3, withdraw/3, close/2]).

%% State
-record(account, {
    id,
    owner_id,
    balance = 0,
    status = pending,   %% pending | active | frozen | closed
    version = 0
}).

initial_state() ->
    #account{}.

%% Public API — execute commands
open(AccountId, OwnerId) ->
    aggregate:execute(?MODULE, AccountId, {open_account, OwnerId}).

deposit(AccountId, Amount, Description) when Amount > 0 ->
    aggregate:execute(?MODULE, AccountId, {deposit, Amount, Description}).

withdraw(AccountId, Amount, Description) when Amount > 0 ->
    aggregate:execute(?MODULE, AccountId, {withdraw, Amount, Description}).

close(AccountId, Reason) ->
    aggregate:execute(?MODULE, AccountId, {close_account, Reason}).

%% Command handlers — validate and produce events
handle_command(#account{status = pending}, {open_account, OwnerId}) ->
    {ok, [#{type => account_opened, data => #{owner_id => OwnerId, initial_balance => 0}}]};

handle_command(#account{status = _}, {open_account, _}) ->
    {error, account_already_opened};

handle_command(#account{status = active}, {deposit, Amount, Description}) ->
    {ok, [#{type => amount_deposited, data => #{amount => Amount, description => Description}}]};

handle_command(#account{status = frozen}, {deposit, _, _}) ->
    {error, account_frozen};

handle_command(#account{status = active, balance = Balance}, {withdraw, Amount, Description}) ->
    case Balance >= Amount of
        true ->
            {ok, [#{type => amount_withdrawn, data => #{amount => Amount, description => Description}}]};
        false ->
            {error, insufficient_funds}
    end;

handle_command(#account{status = active}, {close_account, Reason}) ->
    {ok, [#{type => account_closed, data => #{reason => Reason}}]};

handle_command(_, _) ->
    {error, invalid_command}.

%% Event appliers — update state based on event
apply_event(State, account_opened, #{owner_id := OwnerId}) ->
    State#account{owner_id = OwnerId, status = active, balance = 0};

apply_event(#account{balance = Balance} = State, amount_deposited, #{amount := Amount}) ->
    State#account{balance = Balance + Amount};

apply_event(#account{balance = Balance} = State, amount_withdrawn, #{amount := Amount}) ->
    State#account{balance = Balance - Amount};

apply_event(State, account_closed, _) ->
    State#account{status = closed};

apply_event(State, _, _) ->
    State.  %% Unknown event — ignore (forward compatibility)

%% Load current state for queries
get_account(AccountId) ->
    aggregate:load(?MODULE, AccountId).

%% Get transaction history
get_transactions(AccountId) ->
    StreamId = <<"stream:bank_account:", AccountId/binary>>,
    Events = event_store:load(StreamId, 0),
    
    [format_transaction(E) || E <- Events,
     lists:member(E#event.event_type, [amount_deposited, amount_withdrawn])].

format_transaction(#event{event_type = amount_deposited, data = Data, occurred_at = Ts}) ->
    #{type => deposit, amount => maps:get(amount, Data), description => maps:get(description, Data),
      timestamp => Ts};
format_transaction(#event{event_type = amount_withdrawn, data = Data, occurred_at = Ts}) ->
    #{type => withdrawal, amount => maps:get(amount, Data), description => maps:get(description, Data),
      timestamp => Ts}.

%% Time travel: get account state at a specific time
get_account_at(AccountId, AtTime) ->
    StreamId = <<"stream:bank_account:", AccountId/binary>>,
    EventsBefore = lists:takewhile(fun(#event{occurred_at = Ts}) ->
        Ts =< AtTime
    end, event_store:load(StreamId, 0)),
    
    lists:foldl(fun(#event{event_type = Type, data = Data}, State) ->
        apply_event(State, Type, Data)
    end, initial_state(), EventsBefore).
```

---

## CQRS Read Models

```erlang
%% account_balance_projection.erl — read model updated by events

-module(account_balance_projection).
-behaviour(gen_server).
-export([start_link/0, get_balance/1, get_all_balances/0]).
-export([init/1, handle_cast/2, handle_call/3]).

%% This process maintains a read-optimized view
%% Updated by consuming events from event_bus

init([]) ->
    ets:new(account_balances, [named_table, public, {read_concurrency, true}]),
    
    %% Subscribe to all bank account events
    event_bus:subscribe(<<"stream:bank_account:*">>, self()),
    
    %% Rebuild projection from event store on startup
    rebuild_projection(),
    
    {ok, #{}}.

%% Query read model
get_balance(AccountId) ->
    case ets:lookup(account_balances, AccountId) of
        [{AccountId, Balance}] -> {ok, Balance};
        [] -> not_found
    end.

get_all_balances() ->
    ets:tab2list(account_balances).

%% Handle events from event_bus
handle_cast({event, StreamId, #event{event_type = account_opened}}, State) ->
    AccountId = extract_account_id(StreamId),
    ets:insert(account_balances, {AccountId, 0}),
    {noreply, State};

handle_cast({event, StreamId, #event{event_type = amount_deposited, data = #{amount := Amount}}}, State) ->
    AccountId = extract_account_id(StreamId),
    ets:update_counter(account_balances, AccountId, Amount, {AccountId, 0}),
    {noreply, State};

handle_cast({event, StreamId, #event{event_type = amount_withdrawn, data = #{amount := Amount}}}, State) ->
    AccountId = extract_account_id(StreamId),
    ets:update_counter(account_balances, AccountId, -Amount, {AccountId, 0}),
    {noreply, State}.

%% Rebuild from event store
rebuild_projection() ->
    AllAccountEvents = event_store:load_all_by_type([account_opened, amount_deposited, amount_withdrawn]),
    [begin
        AccountId = extract_account_id(E#event.stream_id),
        update_balance(AccountId, E)
    end || E <- AllAccountEvents].

update_balance(AccountId, #event{event_type = account_opened}) ->
    ets:insert(account_balances, {AccountId, 0});
update_balance(AccountId, #event{event_type = amount_deposited, data = #{amount := A}}) ->
    ets:update_counter(account_balances, AccountId, A, {AccountId, 0});
update_balance(AccountId, #event{event_type = amount_withdrawn, data = #{amount := A}}) ->
    ets:update_counter(account_balances, AccountId, -A, {AccountId, 0}).

extract_account_id(<<"stream:bank_account:", Id/binary>>) -> Id.
```

---

## สรุป Part 81

| Pattern | Erlang Implementation |
|---------|---------------------|
| Event Store | PostgreSQL + epgsql |
| Aggregate | Module + aggregate:execute/3 |
| Optimistic lock | Expected version check |
| Event bus | pg module (process groups) |
| Read model | ETS projection |
| Time travel | Replay events up to timestamp |
| Rebuild | Load all events, fold state |

**เมื่อใช้ Event Sourcing**: เมื่อต้องการ audit trail สมบูรณ์, undo/redo, หรือหลาย read models จาก data เดียวกัน

---

*[← Part 80: Performance](part_80_performance.md) | [Part 82: API Design Patterns →](part_82_api_design.md)*
