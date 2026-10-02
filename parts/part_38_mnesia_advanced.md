# Part 38: Mnesia Advanced

## สารบัญ
1. [Schema Management](#schema-management)
2. [Table Replication](#table-replication)
3. [Transactions and Locking](#transactions-and-locking)
4. [QLC — Query Language](#qlc--query-language)
5. [Fragmented Tables](#fragmented-tables)
6. [Mnesia Events](#mnesia-events)
7. [ตัวอย่างจริง: Session Store Cluster](#ตัวอย่างจริง-session-store-cluster)

---

## Schema Management

```erlang
%% Mnesia Schema: ต้องสร้างก่อน start

%% Single node setup
mnesia:create_schema([node()]).
application:start(mnesia).

%% Multi-node: สร้าง schema ก่อน connect nodes
Nodes = [node1@host, node2@host, node3@host],
mnesia:create_schema(Nodes).
%% start mnesia on all nodes
[rpc:call(N, application, start, [mnesia]) || N <- Nodes].

%% Add node to existing schema
mnesia:change_config(extra_db_nodes, [NewNode]).
mnesia:change_table_copy_type(schema, NewNode, disc_copies).

%% Backup and restore
mnesia:backup("/backup/mnesia.bak").
mnesia:restore("/backup/mnesia.bak", [{default_op, recreate_tables}]).

%% Dump tables to disk
mnesia:dump_tables([my_table]).

%% Delete schema (dangerous!)
mnesia:delete_schema([node()]).

%% Table types
%% ram_copies:      in-memory only, lost on crash
%% disc_copies:     in-memory + disk (transaction log)
%% disc_only_copies: disk only, slow but persistent

%% Change table type
mnesia:change_table_copy_type(my_table, node(), disc_copies).

%% Table info
mnesia:table_info(my_table, all).
mnesia:table_info(my_table, size).
mnesia:table_info(my_table, memory).
mnesia:table_info(my_table, attributes).
mnesia:table_info(my_table, disc_copies).  %% which nodes have disc copies
```

---

## Table Replication

```erlang
%% Replicate table across nodes for fault tolerance

-record(session, {id, user_id, data, expires_at}).

create_replicated_table(Nodes) ->
    mnesia:create_table(session, [
        {attributes, record_info(fields, session)},
        {type, set},
        {ram_copies, Nodes},    %% all nodes have in-memory copy
        {index, [user_id]}
    ]).

%% Add/remove replicas at runtime
add_replica(Node) ->
    mnesia:add_table_copy(session, Node, ram_copies).

remove_replica(Node) ->
    mnesia:del_table_copy(session, Node).

%% move replica
mnesia:move_table_copy(session, OldNode, NewNode).

%% Replication modes:
%% Full replication (all nodes have copy):
%%   ram_copies on all nodes
%%   High availability, but uses more memory

%% Selective replication (some nodes):
%%   ram_copies on [node1, node2]   %% fast access
%%   disc_copies on [node3]          %% persistence

%% Write to replicated table: writes synchronously to ALL replicas
mnesia:transaction(fun() ->
    mnesia:write(#session{id = <<"s1">>, user_id = 1, data = #{}})
end).

%% majority writes (needs majority of replicas)
mnesia:transaction(fun() ->
    mnesia:write(Record)
end, [], [{majority, true}]).

%% dirty async write (faster but unsafe)
mnesia:dirty_write(Record).  %% writes to all replicas async

%% Subscriber for table changes
mnesia:subscribe({table, session, detailed}).
%% Receives: {mnesia_table_event, {write, session, NewRecord, OldRecords, ActivityId}}
%%           {mnesia_table_event, {delete, session, Key, OldRecords, ActivityId}}
```

---

## Transactions and Locking

```erlang
%% Transactions in Mnesia

%% ACID: Atomic, Consistent, Isolated, Durable

%% Basic transaction
{atomic, Result} = mnesia:transaction(fun() ->
    [Session] = mnesia:read(session, SessionId),
    NewSession = Session#session{data = NewData},
    mnesia:write(NewSession),
    NewSession
end).

%% Abort transaction
mnesia:transaction(fun() ->
    case some_check() of
        ok -> mnesia:write(Record);
        error -> mnesia:abort(reason)  %% rollback
    end
end).

%% Nested transactions
mnesia:transaction(fun() ->
    inner_operation(),  %% runs in same transaction
    ok
end).

%% Locking
%% read lock (default for reads):
mnesia:read({session, Id}).

%% write lock (default for writes):
mnesia:write(Record).

%% exclusive read (prevent concurrent reads):
mnesia:read({session, Id}, write).

%% sticky write (prefer local node):
mnesia:s_write(Record).

%% Match lock (lock entire table):
mnesia:lock({table, session}, read).
mnesia:lock({table, session}, write).

%% Optimistic locking (no lock, check version):
update_optimistic(Id, NewData) ->
    mnesia:transaction(fun() ->
        [#session{data = OldData} = S] = mnesia:read(session, Id),
        case maps:get(version, OldData) =:= maps:get(version, NewData) - 1 of
            true -> mnesia:write(S#session{data = NewData});
            false -> mnesia:abort(version_conflict)
        end
    end).

%% Activity: single operation without full transaction overhead
mnesia:activity(transaction, fun() ->
    mnesia:read(session, Id)
end).

mnesia:activity(async_dirty, fun() ->
    mnesia:write(Record)  %% async, no transaction overhead
end).
```

---

## QLC — Query Language

```erlang
%% QLC: SQL-like queries for Mnesia/ETS/DETS

-include_lib("stdlib/include/qlc.hrl").

%% Basic QLC
{atomic, Sessions} = mnesia:transaction(fun() ->
    qlc:e(qlc:q(
        [S || S <- mnesia:table(session),
              S#session.expires_at > os:system_time(second)]
    ))
end).

%% QLC with joins
{atomic, Result} = mnesia:transaction(fun() ->
    qlc:e(qlc:q(
        [{U#user.name, S#session.data}
         || U <- mnesia:table(user),
            S <- mnesia:table(session),
            U#user.id =:= S#session.user_id,
            U#user.active =:= true]
    ))
end).

%% QLC with sort and limit
{atomic, Top10} = mnesia:transaction(fun() ->
    Q = qlc:q([U || U <- mnesia:table(user), U#user.age > 18]),
    Sorted = qlc:sort(Q, [{order, fun(A, B) -> A#user.age < B#user.age end}]),
    Cursor = qlc:cursor(Sorted),
    R = qlc:next_answers(Cursor, 10),
    qlc:delete_cursor(Cursor),
    R
end).

%% Count
{atomic, Count} = mnesia:transaction(fun() ->
    qlc:e(qlc:q([1 || _ <- mnesia:table(session)]))
end),
length(Count).  %% or use fold

%% Aggregate (sum, avg)
{atomic, Total} = mnesia:transaction(fun() ->
    Balances = qlc:e(qlc:q([A#account.balance || A <- mnesia:table(account)])),
    lists:sum(Balances)
end).

%% cursor for large result sets
{atomic, _} = mnesia:transaction(fun() ->
    Q = qlc:q([S || S <- mnesia:table(session)]),
    Cursor = qlc:cursor(Q),
    process_cursor(Cursor)
end).

process_cursor(Cursor) ->
    case qlc:next_answers(Cursor, 100) of
        [] ->
            qlc:delete_cursor(Cursor);
        Batch ->
            process_batch(Batch),
            process_cursor(Cursor)
    end.
```

---

## Fragmented Tables

```erlang
%% Fragmented Tables: horizontal sharding สำหรับ large tables
%% ข้อมูลถูกแบ่งเป็น fragments, แต่ละ fragment อยู่บน node ต่างๆ

%% Create fragmented table
mnesia:create_table(big_log, [
    {attributes, record_info(fields, log_entry)},
    {frag_properties, [
        {n_fragments, 8},          %% initial fragments
        {n_disc_copies, 1},        %% disc copies per fragment
        {node_pool, nodes()}       %% distribute across nodes
    ]}
]).

%% Add more fragments
mnesia:change_table_frag(big_log, {add_frag, nodes()}).

%% Increase disc copies
mnesia:change_table_frag(big_log, {add_node, NewNode}).

%% Use fragmented table: same API as normal table
mnesia:activity(
    transaction,
    fun() -> mnesia:write(#log_entry{...}) end,
    [],
    mnesia_frag
).

%% Custom hash function for sharding
mnesia:create_table(user_data, [
    {frag_properties, [
        {n_fragments, 4},
        {hash_module, my_hash_module}  %% custom sharding
    ]}
]).

-module(my_hash_module).
-export([key_to_frag/2]).

key_to_frag(Key, NumFrags) ->
    Hash = erlang:phash2(Key),
    (Hash rem NumFrags) + 1.
```

---

## Mnesia Events

```erlang
%% Subscribe to Mnesia events

%% System events
mnesia:subscribe(system).
%% Receives:
%% {mnesia_system_event, {mnesia_up, Node}}
%% {mnesia_system_event, {mnesia_down, Node}}
%% {mnesia_system_event, {inconsistent_database, Context, Node}}
%% {mnesia_system_event, {mnesia_overload, Details}}

%% Table events
mnesia:subscribe({table, session, simple}).
%% Receives:
%% {mnesia_table_event, {write, Record, ActivityId}}
%% {mnesia_table_event, {delete, {Table, Key}, ActivityId}}
%% {mnesia_table_event, {delete_object, OldRecord, ActivityId}}

mnesia:subscribe({table, session, detailed}).
%% Includes old records before change

mnesia:unsubscribe({table, session, simple}).

%% Event handler
-module(mnesia_event_handler).
-behaviour(gen_server).

init([]) ->
    mnesia:subscribe(system),
    mnesia:subscribe({table, session, simple}),
    {ok, #{}}.

handle_info({mnesia_system_event, {mnesia_up, Node}}, State) ->
    io:format("Mnesia up on ~p~n", [Node]),
    {noreply, State};

handle_info({mnesia_system_event, {mnesia_down, Node}}, State) ->
    io:format("Mnesia down on ~p~n", [Node]),
    alert_ops(Node),
    {noreply, State};

handle_info({mnesia_table_event, {write, Record, _}}, State) ->
    invalidate_cache(Record),
    {noreply, State};

handle_info({mnesia_table_event, {delete, {session, Key}, _}}, State) ->
    io:format("Session deleted: ~p~n", [Key]),
    {noreply, State};

handle_info(_, State) -> {noreply, State}.
```

---

## ตัวอย่างจริง: Session Store Cluster

```erlang
%% ไฟล์: session_store.erl
%% Distributed session store ที่ใช้ Mnesia

-module(session_store).
-behaviour(gen_server).

-include_lib("stdlib/include/qlc.hrl").

-export([start_link/0,
         create/2, get/1, update/2, delete/1,
         get_user_sessions/1, cleanup_expired/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(session, {
    id :: binary(),
    user_id :: integer(),
    data = #{} :: map(),
    created_at :: integer(),
    expires_at :: integer()
}).

-define(SESSION_TTL, 86400).  %% 24 hours

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% API
create(UserId, Data) ->
    gen_server:call(?MODULE, {create, UserId, Data}).

get(SessionId) ->
    gen_server:call(?MODULE, {get, SessionId}).

update(SessionId, NewData) ->
    gen_server:call(?MODULE, {update, SessionId, NewData}).

delete(SessionId) ->
    gen_server:cast(?MODULE, {delete, SessionId}).

get_user_sessions(UserId) ->
    gen_server:call(?MODULE, {user_sessions, UserId}).

cleanup_expired() ->
    gen_server:cast(?MODULE, cleanup).

%% Callbacks
init([]) ->
    setup_mnesia(),
    %% Schedule periodic cleanup
    erlang:send_after(3600000, self(), cleanup_tick),  %% every hour
    {ok, #{}}.

setup_mnesia() ->
    Nodes = [node() | nodes()],
    case mnesia:create_table(session, [
        {attributes, record_info(fields, session)},
        {type, set},
        {ram_copies, Nodes},
        {index, [user_id, expires_at]}
    ]) of
        {atomic, ok} -> ok;
        {aborted, {already_exists, session}} -> ok  %% table exists
    end.

handle_call({create, UserId, Data}, _From, State) ->
    Now = os:system_time(second),
    Session = #session{
        id = generate_session_id(),
        user_id = UserId,
        data = Data,
        created_at = Now,
        expires_at = Now + ?SESSION_TTL
    },
    {atomic, ok} = mnesia:transaction(fun() ->
        mnesia:write(Session)
    end),
    {reply, {ok, Session#session.id}, State};

handle_call({get, SessionId}, _From, State) ->
    Now = os:system_time(second),
    Result = mnesia:dirty_read(session, SessionId),
    Response = case Result of
        [#session{expires_at = Exp} = S] when Exp > Now ->
            %% Extend session TTL on access
            mnesia:dirty_write(S#session{expires_at = Now + ?SESSION_TTL}),
            {ok, S};
        [#session{}] ->
            mnesia:dirty_delete({session, SessionId}),
            {error, expired};
        [] ->
            {error, not_found}
    end,
    {reply, Response, State};

handle_call({update, SessionId, NewData}, _From, State) ->
    {atomic, Result} = mnesia:transaction(fun() ->
        case mnesia:read(session, SessionId) of
            [S] ->
                Now = os:system_time(second),
                Updated = S#session{
                    data = maps:merge(S#session.data, NewData),
                    expires_at = Now + ?SESSION_TTL
                },
                mnesia:write(Updated),
                {ok, Updated};
            [] ->
                {error, not_found}
        end
    end),
    {reply, Result, State};

handle_call({user_sessions, UserId}, _From, State) ->
    Now = os:system_time(second),
    {atomic, Sessions} = mnesia:transaction(fun() ->
        qlc:e(qlc:q(
            [S || S <- mnesia:table(session),
                  S#session.user_id =:= UserId,
                  S#session.expires_at > Now]
        ))
    end),
    {reply, Sessions, State}.

handle_cast({delete, SessionId}, State) ->
    mnesia:dirty_delete({session, SessionId}),
    {noreply, State};

handle_cast(cleanup, State) ->
    do_cleanup(),
    {noreply, State}.

handle_info(cleanup_tick, State) ->
    do_cleanup(),
    erlang:send_after(3600000, self(), cleanup_tick),
    {noreply, State};
handle_info(_, State) -> {noreply, State}.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

do_cleanup() ->
    Now = os:system_time(second),
    {atomic, _} = mnesia:transaction(fun() ->
        Expired = mnesia:select(session,
            [{ #session{expires_at = '$1', id = '$2', _ = '_'},
               [{'<', '$1', Now}],
               ['$2'] }]
        ),
        [mnesia:delete({session, Id}) || Id <- Expired],
        length(Expired)
    end).

generate_session_id() ->
    <<A:32, B:16, C:16, D:16, E:48>> = crypto:strong_rand_bytes(16),
    SessionId = io_lib:format("~8.16.0b-~4.16.0b-~4.16.0b-~4.16.0b-~12.16.0b",
                               [A, B, C, D, E]),
    list_to_binary(SessionId).

%% ใช้งาน
session_store:start_link().
{ok, SessId} = session_store:create(1, #{ip => <<"127.0.0.1">>}).
{ok, Session} = session_store:get(SessId).
session_store:update(SessId, #{last_page => <<"/dashboard">>}).
session_store:get_user_sessions(1).
session_store:delete(SessId).
```

---

## สรุป Part 38

| Operation | Function | Performance |
|-----------|----------|-------------|
| Write | `mnesia:write` | Medium (WAL) |
| Read | `mnesia:read` | Fast |
| Dirty write | `mnesia:dirty_write` | Fastest |
| Dirty read | `mnesia:dirty_read` | Fastest |
| Transaction | `mnesia:transaction` | Slowest (ACID) |
| QLC query | `qlc:e(qlc:q(...))` | Depends |

| Copy type | Speed | Persistence | Fault tolerance |
|-----------|-------|-------------|-----------------|
| ram_copies | Fastest | None | Node death = data lost |
| disc_copies | Fast | Yes (WAL) | Survives restart |
| disc_only | Slow | Yes | Low memory usage |

---

*[← Part 37: Advanced OTP Patterns](part_37_advanced_otp.md) | [Part 39: Performance Tuning →](part_39_performance.md)*
