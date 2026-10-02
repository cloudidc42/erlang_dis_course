# Part 70: Production War Stories

## สารบัญ
1. [Memory Leak Investigation](#memory-leak)
2. [Latency Spike Analysis](#latency-spike)
3. [Cascading Failures](#cascading)
4. [Data Race under Load](#data-race)
5. [Runaway Process Bug](#runaway-process)
6. [Lessons Learned](#lessons)

---

## Memory Leak Investigation

```erlang
%% Real scenario: system memory growing 100MB/hour

%% Step 1: Monitor what's growing
detect_leak() ->
    S1 = memory_snapshot(),
    timer:sleep(3600000),  %% wait 1 hour
    S2 = memory_snapshot(),
    compare_snapshots(S1, S2).

memory_snapshot() ->
    #{
        total => erlang:memory(total),
        processes => erlang:memory(processes),
        binary => erlang:memory(binary),
        ets => erlang:memory(ets),
        process_count => erlang:system_info(process_count),
        ets_tables => [{ets:info(T, name), ets:info(T, size)} || T <- ets:all()]
    }.

compare_snapshots(S1, S2) ->
    maps:map(fun(K, V2) ->
        V1 = maps:get(K, S1, 0),
        #{before => V1, after => V2, growth => V2 - V1}
    end, S2).

%% Step 2: Found ETS table growing without bound

%% The bug: session cache never cleaned up
-module(session_cache_bug).
-export([store/2, get/1]).

%% BUG: no TTL, no cleanup
store(SessionId, Data) ->
    ets:insert(sessions, {SessionId, Data}).

get(SessionId) ->
    case ets:lookup(sessions, SessionId) of
        [{_, Data}] -> {ok, Data};
        [] -> not_found
    end.

%% FIX: add TTL and cleanup
-module(session_cache_fix).
-export([store/2, get/1]).
-define(TTL, 3600).  %% 1 hour

store(SessionId, Data) ->
    Expiry = erlang:system_time(second) + ?TTL,
    ets:insert(sessions, {SessionId, Data, Expiry}).

get(SessionId) ->
    case ets:lookup(sessions, SessionId) of
        [{_, Data, Expiry}] ->
            case erlang:system_time(second) < Expiry of
                true -> {ok, Data};
                false ->
                    ets:delete(sessions, SessionId),
                    not_found
            end;
        [] -> not_found
    end.

%% Periodic cleanup (add to supervisor)
cleanup_expired() ->
    Now = erlang:system_time(second),
    ets:select_delete(sessions, [
        {{'_', '_', '$1'}, [{'=<', '$1', Now}], [true]}
    ]).
```

---

## Latency Spike Analysis

```erlang
%% Scenario: P99 latency spiked from 10ms to 800ms every 5 minutes

%% Investigation: GC pause on large processes

%% Found: request handler accumulating large state
-module(handler_bug).
handle_request(Req, #{log := Log} = State) ->
    %% BUG: accumulating entire request in log
    NewLog = [Req | Log],
    process_request(Req),
    {ok, State#{log := NewLog}}.  %% Log grows forever!

%% When process GC runs → large heap → long pause

%% Fix: bounded log + separate large data
-module(handler_fix).
-define(MAX_LOG, 1000).

handle_request(Req, #{log := Log} = State) ->
    LogEntry = #{
        method => cowboy_req:method(Req),
        path => cowboy_req:path(Req),
        ts => erlang:system_time(millisecond)
    },
    NewLog = lists:sublist([LogEntry | Log], ?MAX_LOG),
    process_request(Req),
    {ok, State#{log := NewLog}}.

%% Another cause: scheduler contention
%% Fix: use dirty schedulers for CPU-heavy NIFs
%% or break work into small reductions

check_gc_impact() ->
    %% Monitor GC pauses
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(60000),
    Stats = erlang:statistics(scheduler_wall_time),
    Utilizations = [Active / (Active + Idle) || {_, Active, Idle} <- Stats],
    MaxUtil = lists:max(Utilizations),
    io:format("Max scheduler utilization: ~.1f%~n", [MaxUtil * 100]).
```

---

## Cascading Failures

```erlang
%% Scenario: database slow → connection pool exhausted → timeouts everywhere

%% The chain:
%% 1. DB query taking 5s (normally <50ms)
%% 2. Connection pool (10 connections) fills up
%% 3. Gen_server call timeouts cascade
%% 4. Supervisor restarts processes → more DB connections
%% 5. DB overwhelmed by reconnections → crash

%% Fix 1: Circuit breaker stops cascade
-module(db_circuit_breaker).
-behaviour(gen_server).

-define(FAILURE_THRESHOLD, 5).
-define(RECOVERY_TIMEOUT, 30000).

query(SQL, Params) ->
    gen_server:call(?MODULE, {query, SQL, Params}, 5000).

handle_call({query, SQL, Params}, From, #{state := open} = State) ->
    %% Circuit is open: fail fast, don't hit DB
    {reply, {error, circuit_open}, State};

handle_call({query, SQL, Params}, From, #{state := closed, failures := F} = State) ->
    case db_pool:query(SQL, Params) of
        {ok, _} = Ok ->
            {reply, Ok, State#{failures := 0}};
        {error, _} = Err ->
            NewFailures = F + 1,
            NewState = case NewFailures >= ?FAILURE_THRESHOLD of
                true ->
                    logger:critical("DB circuit breaker opened"),
                    erlang:send_after(?RECOVERY_TIMEOUT, self(), try_recovery),
                    State#{state := open, failures := NewFailures};
                false ->
                    State#{failures := NewFailures}
            end,
            {reply, Err, NewState}
    end;

handle_info(try_recovery, State) ->
    case db_pool:query("SELECT 1", []) of
        {ok, _} ->
            logger:info("DB circuit breaker closed"),
            {noreply, State#{state := closed, failures := 0}};
        _ ->
            erlang:send_after(?RECOVERY_TIMEOUT, self(), try_recovery),
            {noreply, State}
    end.

%% Fix 2: Timeout hierarchy (outer > inner)
outer_request() ->
    gen_server:call(my_server, request, 10000).  %% 10s outer

inner_db_call() ->
    db_pool:query("SELECT ...", [], 5000).  %% 5s inner (shorter!)
    %% Outer timeout fires BEFORE inner → clean failure

%% Fix 3: Backpressure on connection pool
checked_query(SQL, Params) ->
    case poolboy:checkout(db_pool, false) of  %% false = don't block
        full ->
            metrics:increment(db_pool_full),
            {error, pool_exhausted};
        Worker ->
            try poolboy:transaction(db_pool, fun(W) ->
                db_worker:query(W, SQL, Params)
            end)
            catch _:_ -> {error, failed}
            end
    end.
```

---

## Runaway Process Bug

```erlang
%% Scenario: process count growing from 100 to 1,000,000

%% The bug: recursive spawn without termination
-module(spawner_bug).
start() ->
    spawn(fun loop/0).

loop() ->
    %% BUG: spawns a new process every iteration
    %% Old process stays alive (it has a mailbox message)
    spawn(fun loop/0),
    receive
        _ -> ok
    after 0 ->
        loop()
    end.

%% The fix: tail recursion without spawning
-module(spawner_fix).
start() ->
    spawn(fun worker/0).

worker() ->
    receive
        process_item ->
            do_work(),
            worker()  %% tail recursive, same process
    end.

%% Detect runaway processes early
monitor_process_count() ->
    case erlang:system_info(process_count) of
        N when N > 500000 ->
            logger:critical("Process count too high", #{count => N}),
            find_top_spawners();
        _ -> ok
    end.

find_top_spawners() ->
    AllPids = processes(),
    %% Find processes that spawn many children
    Spawners = [{P, count_children(P)} || P <- AllPids],
    Top = lists:sublist(lists:reverse(lists:keysort(2, Spawners)), 10),
    [io:format("~p: ~p children~n", [P, C]) || {P, C} <- Top].

count_children(Pid) ->
    case process_info(Pid, links) of
        {links, Links} -> length(Links);
        _ -> 0
    end.
```

---

## Lessons Learned

```erlang
%% Key lessons from production incidents

%% 1. Always set max_heap_size to prevent process memory explosion
spawn_safe(Fun) ->
    spawn_opt(Fun, [
        {max_heap_size, #{
            size => 50 * 1024 * 1024,  %% 50MB
            kill => true,
            error_logger => true
        }}
    ]).

%% 2. Circuit breakers on all external calls
%% 3. Timeouts on every gen_server:call
%% 4. Monitor message queue lengths in production

health_check() ->
    AllPids = processes(),
    LongQueues = [{P, Q} ||
        P <- AllPids,
        {message_queue_len, Q} <- [process_info(P, message_queue_len)],
        Q > 1000],
    case LongQueues of
        [] -> ok;
        _ ->
            logger:warning("Long message queues detected", #{queues => LongQueues})
    end.

%% 5. Use bounded ETS tables
bounded_ets_insert(Table, Key, Value, MaxSize) ->
    case ets:info(Table, size) >= MaxSize of
        true ->
            %% Evict oldest entry
            case ets:first(Table) of
                '$end_of_table' -> ok;
                OldKey -> ets:delete(Table, OldKey)
            end;
        false -> ok
    end,
    ets:insert(Table, {Key, Value}).

%% 6. Log crashes with full context
handle_crash(Class, Reason, Stack) ->
    logger:critical("Process crashed", #{
        class => Class,
        reason => Reason,
        stacktrace => Stack,
        node => node(),
        memory => erlang:memory(total),
        processes => erlang:system_info(process_count),
        time => erlang:system_time(millisecond)
    }).

%% 7. Chaos engineering: test before it fails in prod
simulate_db_slowdown() ->
    meck:expect(db_pool, query, fun(SQL, Params) ->
        timer:sleep(5000),
        meck:passthrough([SQL, Params])
    end).
```

---

## สรุป Part 70

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| Memory leak | Growing ETS without TTL | TTL + periodic cleanup |
| Latency spikes | Large process GC | Bounded state, smaller heaps |
| Cascading failure | Slow DB → pool exhaustion | Circuit breaker |
| Runaway processes | Recursive spawn | Tail recursive worker |
| Message queue growth | Slow consumer | Back-pressure, monitoring |

**"Let it crash" + supervisor ≠ ignore monitoring**

---

*[← Part 69: Profiling](part_69_profiling.md) | [Part 71: Erlang Ecosystem →](part_71_ecosystem.md)*
