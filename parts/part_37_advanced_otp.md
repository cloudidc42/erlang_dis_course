# Part 37: Advanced OTP Patterns

## สารบัญ
1. [Worker Pool Pattern](#worker-pool-pattern)
2. [Circuit Breaker](#circuit-breaker)
3. [Rate Limiter](#rate-limiter)
4. [Saga Pattern (Distributed Transactions)](#saga-pattern-distributed-transactions)
5. [Event Sourcing](#event-sourcing)
6. [ตัวอย่างจริง: Service with Resiliency](#ตัวอย่างจริง-service-with-resiliency)

---

## Worker Pool Pattern

```erlang
%% Worker Pool: จัดการ pool ของ workers, limit concurrency

-module(worker_pool).
-behaviour(gen_server).

-export([start_link/2, run/2, run_async/2, stats/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    size :: pos_integer(),
    worker_sup :: pid(),
    free = [] :: [pid()],
    busy = #{} :: #{pid() => {pid(), reference()}},
    queue = queue:new() :: queue:queue(),
    completed = 0 :: non_neg_integer(),
    failed = 0 :: non_neg_integer()
}).

start_link(Name, Size) ->
    gen_server:start_link({local, Name}, ?MODULE, [Size], []).

%% Synchronous: wait for result
run(Pool, Fun) ->
    gen_server:call(Pool, {run, Fun}, 30000).

%% Asynchronous: fire and forget
run_async(Pool, Fun) ->
    gen_server:cast(Pool, {run_async, Fun}).

stats() ->
    gen_server:call(?MODULE, stats).

init([Size]) ->
    {ok, SupPid} = pool_worker_sup:start_link(),
    Workers = [start_worker(SupPid) || _ <- lists:seq(1, Size)],
    {ok, #state{size = Size, worker_sup = SupPid, free = Workers}}.

start_worker(Sup) ->
    {ok, W} = supervisor:start_child(Sup, []),
    W.

handle_call({run, Fun}, From, State) ->
    NewState = enqueue({call, From, Fun}, State),
    {noreply, dispatch(NewState)};

handle_call(stats, _From, #state{free = F, busy = B, queue = Q,
                                  completed = C, failed = Fail} = S) ->
    Stats = #{
        free => length(F),
        busy => map_size(B),
        queued => queue:len(Q),
        completed => C,
        failed => Fail
    },
    {reply, Stats, S}.

handle_cast({run_async, Fun}, State) ->
    NewState = enqueue({cast, Fun}, State),
    {noreply, dispatch(NewState)}.

handle_info({worker_done, Worker, {ok, Result}},
            #state{busy = Busy, free = Free, completed = C} = State) ->
    case maps:get(Worker, Busy, undefined) of
        {CallerRef, MRef} when is_reference(CallerRef) ->
            %% Sync call, send reply
            gen_server:reply(CallerRef, Result);
        _ ->
            ok
    end,
    erlang:demonitor(maps:get(Worker, Busy, {undefined, make_ref()})),
    NewState = State#state{
        busy = maps:remove(Worker, Busy),
        free = [Worker | Free],
        completed = C + 1
    },
    {noreply, dispatch(NewState)};

handle_info({worker_done, Worker, {error, Reason}},
            #state{busy = Busy, free = Free, failed = Fail} = State) ->
    case maps:get(Worker, Busy, undefined) of
        {CallerRef, _} when is_reference(CallerRef) ->
            gen_server:reply(CallerRef, {error, Reason});
        _ ->
            ok
    end,
    NewState = State#state{
        busy = maps:remove(Worker, Busy),
        free = [Worker | Free],
        failed = Fail + 1
    },
    {noreply, dispatch(NewState)};

handle_info({'DOWN', _Ref, process, Worker, _Reason},
            #state{busy = Busy, worker_sup = Sup} = State) ->
    NewWorker = start_worker(Sup),
    NewBusy = maps:remove(Worker, Busy),
    {noreply, dispatch(State#state{busy = NewBusy, free = [NewWorker | State#state.free]})};

handle_info(_, State) -> {noreply, State}.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

enqueue(Item, #state{queue = Q} = State) ->
    State#state{queue = queue:in(Item, Q)}.

dispatch(#state{free = [], queue = Q} = State) ->
    case queue:is_empty(Q) of
        true -> State;
        false -> State  %% no free workers
    end;
dispatch(#state{free = [Worker | Rest], queue = Q,
                busy = Busy} = State) ->
    case queue:out(Q) of
        {empty, _} ->
            State;
        {{value, {call, From, Fun}}, NewQ} ->
            Self = self(),
            MRef = erlang:monitor(process, Worker),
            spawn(fun() ->
                Result = try {ok, Fun()} catch E:R -> {error, {E, R}} end,
                Self ! {worker_done, Worker, Result}
            end),
            dispatch(State#state{
                free = Rest,
                busy = Busy#{Worker => {From, MRef}},
                queue = NewQ
            });
        {{value, {cast, Fun}}, NewQ} ->
            Self = self(),
            MRef = erlang:monitor(process, Worker),
            spawn(fun() ->
                Result = try {ok, Fun()} catch E:R -> {error, {E, R}} end,
                Self ! {worker_done, Worker, Result}
            end),
            dispatch(State#state{
                free = Rest,
                busy = Busy#{Worker => {async, MRef}},
                queue = NewQ
            })
    end.
```

---

## Circuit Breaker

```erlang
%% Circuit Breaker: หยุดส่ง requests เมื่อ service ล้ม
%% States: closed (normal) -> open (failing) -> half_open (testing)

-module(circuit_breaker).
-behaviour(gen_statem).

-export([start_link/2, call/2, state/0]).
-export([callback_mode/0, init/1, handle_event/4]).

-record(data, {
    threshold = 5 :: pos_integer(),   %% failures to open
    reset_timeout = 30000 :: pos_integer(),  %% ms before trying again
    failures = 0 :: non_neg_integer(),
    success_in_half_open = 0 :: non_neg_integer()
}).

start_link(Name, Opts) ->
    gen_statem:start_link({local, Name}, ?MODULE, [Opts], []).

call(Cb, Fun) ->
    gen_statem:call(Cb, {call, Fun}).

state() ->
    gen_statem:call(?MODULE, get_state).

callback_mode() -> handle_event_function.

init([Opts]) ->
    Data = #data{
        threshold = maps:get(threshold, Opts, 5),
        reset_timeout = maps:get(reset_timeout, Opts, 30000)
    },
    {ok, closed, Data}.

%% CLOSED state: normal operation
handle_event({call, From}, {call, Fun}, closed, Data) ->
    try
        Result = Fun(),
        {keep_state, Data#data{failures = 0}, [{reply, From, {ok, Result}}]}
    catch
        _:Reason ->
            NewData = Data#data{failures = Data#data.failures + 1},
            case NewData#data.failures >= NewData#data.threshold of
                true ->
                    io:format("[CircuitBreaker] Opening circuit after ~p failures~n",
                              [NewData#data.failures]),
                    Actions = [
                        {reply, From, {error, Reason}},
                        {state_timeout, NewData#data.reset_timeout, try_reset}
                    ],
                    {next_state, open, NewData, Actions};
                false ->
                    {keep_state, NewData, [{reply, From, {error, Reason}}]}
            end
    end;

handle_event({call, From}, get_state, closed, Data) ->
    {keep_state, Data, [{reply, From, #{state => closed, failures => Data#data.failures}}]};

%% OPEN state: reject all calls
handle_event({call, From}, {call, _Fun}, open, Data) ->
    {keep_state, Data, [{reply, From, {error, circuit_open}}]};

handle_event(state_timeout, try_reset, open, Data) ->
    io:format("[CircuitBreaker] Half-opening circuit~n"),
    {next_state, half_open, Data#data{success_in_half_open = 0}};

handle_event({call, From}, get_state, open, Data) ->
    {keep_state, Data, [{reply, From, #{state => open}}]};

%% HALF_OPEN state: allow one test call
handle_event({call, From}, {call, Fun}, half_open, Data) ->
    try
        Result = Fun(),
        io:format("[CircuitBreaker] Test successful, closing circuit~n"),
        {next_state, closed, Data#data{failures = 0},
         [{reply, From, {ok, Result}}]}
    catch
        _:Reason ->
            io:format("[CircuitBreaker] Test failed, re-opening~n"),
            Actions = [
                {reply, From, {error, Reason}},
                {state_timeout, Data#data.reset_timeout, try_reset}
            ],
            {next_state, open, Data, Actions}
    end;

handle_event({call, From}, get_state, half_open, Data) ->
    {keep_state, Data, [{reply, From, #{state => half_open}}]};

handle_event(_, _, State, Data) ->
    {keep_state, Data}.

%% ใช้งาน
{ok, _} = circuit_breaker:start_link(db_cb, #{threshold => 3, reset_timeout => 10000}).

%% Call through circuit breaker
case circuit_breaker:call(db_cb, fun() -> db:query("SELECT 1") end) of
    {ok, Result} -> use(Result);
    {error, circuit_open} -> use_cache();
    {error, Reason} -> handle_error(Reason)
end.
```

---

## Rate Limiter

```erlang
%% Token Bucket Rate Limiter

-module(rate_limiter).
-behaviour(gen_server).

-export([start_link/3, allow/1, allow/2, stats/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(bucket, {
    capacity :: pos_integer(),
    rate :: pos_integer(),     %% tokens per second
    tokens :: float(),
    last_refill :: integer()   %% monotonic_time(millisecond)
}).

start_link(Name, Capacity, Rate) ->
    gen_server:start_link({local, Name}, ?MODULE, [Capacity, Rate], []).

allow(Name) ->
    allow(Name, 1).

allow(Name, Tokens) ->
    gen_server:call(Name, {allow, Tokens}).

stats(Name) ->
    gen_server:call(Name, stats).

init([Capacity, Rate]) ->
    Bucket = #bucket{
        capacity = Capacity,
        rate = Rate,
        tokens = float(Capacity),
        last_refill = erlang:monotonic_time(millisecond)
    },
    {ok, Bucket}.

handle_call({allow, Requested}, _From, Bucket) ->
    Now = erlang:monotonic_time(millisecond),
    Bucket2 = refill(Bucket, Now),
    case Bucket2#bucket.tokens >= Requested of
        true ->
            {reply, allow, Bucket2#bucket{tokens = Bucket2#bucket.tokens - Requested}};
        false ->
            {reply, deny, Bucket2}
    end;

handle_call(stats, _From, #bucket{tokens = T, capacity = C} = B) ->
    {reply, #{tokens => T, capacity => C, fill_percent => T / C * 100}, B}.

handle_cast(_, B) -> {noreply, B}.
handle_info(_, B) -> {noreply, B}.
terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

refill(#bucket{rate = Rate, capacity = Cap, tokens = T, last_refill = Last} = B, Now) ->
    ElapsedMs = Now - Last,
    NewTokens = min(float(Cap), T + Rate * ElapsedMs / 1000.0),
    B#bucket{tokens = NewTokens, last_refill = Now}.

%% ใช้งาน
rate_limiter:start_link(api_rl, 100, 10).  %% 100 tokens, 10/sec

case rate_limiter:allow(api_rl) of
    allow -> process_request();
    deny -> reply_429_too_many_requests()
end.
```

---

## Saga Pattern (Distributed Transactions)

```erlang
%% Saga: sequence ของ steps, แต่ละ step มี compensating action
%% ถ้า step ล้ม -> run compensating actions ย้อนกลับ

-module(saga).

-export([run/1]).

%% saga = list ของ {Step, Compensate}
%% Step/Compensate เป็น fun ที่รับ/คืน {ok, State} หรือ {error, Reason}

run(Steps) ->
    run_steps(Steps, [], #{}).

run_steps([], Completed, State) ->
    compensate(Completed, State),  %% actually this is success path
    {ok, State};
run_steps([{Step, Compensate} | Rest], Done, State) ->
    case Step(State) of
        {ok, NewState} ->
            run_steps(Rest, [{Compensate, NewState} | Done], NewState);
        {error, Reason} ->
            compensate(Done, State),
            {error, Reason}
    end.

compensate([], _) -> ok;
compensate([{CompFun, StateAtStep} | Rest], _State) ->
    catch CompFun(StateAtStep),  %% best effort
    compensate(Rest, StateAtStep).

%% ตัวอย่าง: Order Saga
place_order(OrderData) ->
    saga:run([
        %% Step 1: Reserve inventory
        {fun(State) ->
            case inventory:reserve(maps:get(items, OrderData)) of
                {ok, ResvId} -> {ok, State#{reservation_id => ResvId}};
                {error, R} -> {error, R}
            end
         end,
         fun(#{reservation_id := ResvId}) ->
            inventory:cancel_reservation(ResvId)
         end},

        %% Step 2: Charge payment
        {fun(#{reservation_id := _} = State) ->
            case payments:charge(maps:get(payment, OrderData)) of
                {ok, TxnId} -> {ok, State#{transaction_id => TxnId}};
                {error, R} -> {error, R}
            end
         end,
         fun(#{transaction_id := TxnId}) ->
            payments:refund(TxnId)
         end},

        %% Step 3: Create order record
        {fun(State) ->
            Order = maps:merge(OrderData, State),
            case orders:create(Order) of
                {ok, OrderId} -> {ok, State#{order_id => OrderId}};
                {error, R} -> {error, R}
            end
         end,
         fun(#{order_id := OrderId}) ->
            orders:cancel(OrderId)
         end}
    ]).

%% ถ้า Step 3 ล้ม -> refund payment, cancel reservation
```

---

## Event Sourcing

```erlang
%% Event Sourcing: store events, derive state from replay

-module(event_store).
-behaviour(gen_server).

-export([start_link/0, append/2, load/1, load_from/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

init([]) ->
    ets:new(events, [named_table, ordered_set, public]),
    {ok, #{next_seq => 1}}.

%% Append event for an aggregate
append(AggregateId, Event) ->
    gen_server:call(?MODULE, {append, AggregateId, Event}).

%% Load all events for aggregate
load(AggregateId) ->
    Pattern = {{AggregateId, '$1'}, '$2', '$3'},
    ets:select(events, [{Pattern, [], [{{'$1', '$2', '$3'}}]}]).

%% Load events after sequence N
load_from(AggregateId, FromSeq) ->
    [E || {Seq, _, _} = E <- load(AggregateId), Seq > FromSeq].

handle_call({append, AggId, Event}, _From, #{next_seq := Seq} = State) ->
    Timestamp = os:system_time(millisecond),
    ets:insert(events, {{AggId, Seq}, Event, Timestamp}),
    {reply, {ok, Seq}, State#{next_seq => Seq + 1}}.

handle_cast(_, S) -> {noreply, S}.
handle_info(_, S) -> {noreply, S}.
terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.


%% Account aggregate with event sourcing
-module(account).

-export([create/1, deposit/2, withdraw/2, get_state/1]).

%% Commands -> Events
create(AccountId) ->
    event_store:append(AccountId, {created, #{id => AccountId, balance => 0}}).

deposit(AccountId, Amount) when Amount > 0 ->
    event_store:append(AccountId, {deposited, #{amount => Amount}}).

withdraw(AccountId, Amount) ->
    State = get_state(AccountId),
    case maps:get(balance, State, 0) >= Amount of
        true ->
            event_store:append(AccountId, {withdrawn, #{amount => Amount}});
        false ->
            {error, insufficient_funds}
    end.

%% Reconstruct state from events
get_state(AccountId) ->
    Events = event_store:load(AccountId),
    lists:foldl(fun({_Seq, Event, _T}, State) ->
        apply_event(Event, State)
    end, #{}, Events).

apply_event({created, #{id := Id}}, _) ->
    #{id => Id, balance => 0};
apply_event({deposited, #{amount := A}}, #{balance := B} = S) ->
    S#{balance => B + A};
apply_event({withdrawn, #{amount := A}}, #{balance := B} = S) ->
    S#{balance => B - A}.

%% ใช้งาน
account:create(acc1).
account:deposit(acc1, 1000).
account:deposit(acc1, 500).
account:withdraw(acc1, 200).
account:get_state(acc1).
%% #{id => acc1, balance => 1300}
```

---

## ตัวอย่างจริง: Service with Resiliency

```erlang
%% ไฟล์: resilient_service.erl
%% Service ที่มี: circuit breaker + retry + timeout + rate limiting

-module(resilient_service).

-export([call/2]).

-define(MAX_RETRIES, 3).
-define(TIMEOUT, 5000).
-define(RATE_LIMITER, service_rl).
-define(CIRCUIT_BREAKER, service_cb).

call(Operation, Args) ->
    %% 1. Check rate limit
    case rate_limiter:allow(?RATE_LIMITER) of
        deny ->
            {error, rate_limited};
        allow ->
            %% 2. Call through circuit breaker with retry
            call_with_retry(Operation, Args, ?MAX_RETRIES)
    end.

call_with_retry(_, _, 0) ->
    {error, max_retries_exceeded};
call_with_retry(Op, Args, Retries) ->
    case circuit_breaker:call(?CIRCUIT_BREAKER, fun() ->
        with_timeout(?TIMEOUT, fun() -> do_call(Op, Args) end)
    end) of
        {ok, Result} ->
            {ok, Result};
        {error, circuit_open} ->
            {error, service_unavailable};
        {error, timeout} ->
            {error, timeout};
        {error, _Reason} when Retries > 1 ->
            Delay = backoff_delay(?MAX_RETRIES - Retries + 1),
            timer:sleep(Delay),
            call_with_retry(Op, Args, Retries - 1);
        {error, Reason} ->
            {error, Reason}
    end.

with_timeout(Timeout, Fun) ->
    Self = self(),
    Ref = make_ref(),
    Pid = spawn(fun() ->
        Result = try Fun() catch E:R -> {error, {E, R}} end,
        Self ! {Ref, Result}
    end),
    receive
        {Ref, Result} -> Result
    after Timeout ->
        exit(Pid, kill),
        {error, timeout}
    end.

backoff_delay(Attempt) ->
    min(1000 * (1 bsl (Attempt - 1)), 30000).  %% exponential, max 30s

do_call(get_user, [Id]) ->
    http_client:get("/users/" ++ integer_to_list(Id));
do_call(create_user, [Data]) ->
    http_client:post("/users", Data).
```

---

## สรุป Part 37

| Pattern | Problem Solved |
|---------|---------------|
| Worker Pool | Limit concurrency, reuse workers |
| Circuit Breaker | Protect from cascading failures |
| Rate Limiter | Prevent overload |
| Saga | Distributed transaction consistency |
| Event Sourcing | Audit trail, temporal queries |

---

*[← Part 36: Database Access](part_36_database.md) | [Part 38: Mnesia Advanced →](part_38_mnesia_advanced.md)*
