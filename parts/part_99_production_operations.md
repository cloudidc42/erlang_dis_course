# Part 99: Production Operations Mastery

## สารบัญ
1. [SLOs และ Error Budgets](#slo)
2. [Capacity Planning](#capacity)
3. [On-Call Runbooks](#runbooks)
4. [Chaos Engineering](#chaos)
5. [Cost Optimization](#cost)
6. [ตัวอย่างจริง: SRE Dashboard](#sre-dashboard)

---

## SLOs และ Error Budgets

```erlang
%% SLO (Service Level Objective) tracking
%%
%% Example SLOs:
%% - Availability: 99.9% (8.7 hours downtime/year)
%% - P99 latency: < 500ms
%% - Error rate: < 0.1%
%% - Throughput: > 1000 req/sec

-module(slo_tracker).
-behaviour(gen_server).
-export([start_link/0, record_request/3, get_slo_status/0]).
-export([init/1, handle_cast/2, handle_call/3, handle_info/2]).

-define(WINDOW_SECONDS, 3600).   %% 1-hour rolling window
-define(SLO_AVAILABILITY, 0.999). %% 99.9%
-define(SLO_P99_MS, 500).
-define(SLO_ERROR_RATE, 0.001).  %% 0.1%

-record(state, {
    requests = [],      %% [{timestamp, duration_ms, success}]
    total_requests = 0,
    error_budget_used = 0.0
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% Periodic cleanup of old data
    erlang:send_after(60000, self(), cleanup),
    {ok, #state{}}.

record_request(DurationMs, Success, _Meta) ->
    gen_server:cast(?MODULE, {record, DurationMs, Success,
                              erlang:monotonic_time(second)}).

get_slo_status() ->
    gen_server:call(?MODULE, get_status).

handle_cast({record, Duration, Success, Timestamp}, State) ->
    NewRequests = [{Timestamp, Duration, Success} | State#state.requests],
    {noreply, State#state{
        requests = NewRequests,
        total_requests = State#state.total_requests + 1
    }};

handle_cast(_, State) -> {noreply, State}.

handle_call(get_status, _From, #state{requests = Requests} = State) ->
    Now = erlang:monotonic_time(second),
    Cutoff = Now - ?WINDOW_SECONDS,
    
    Recent = [{D, S} || {T, D, S} <- Requests, T >= Cutoff],
    
    Status = compute_slo_status(Recent),
    {reply, Status, State}.

handle_info(cleanup, #state{requests = Requests} = State) ->
    Now = erlang:monotonic_time(second),
    Cutoff = Now - ?WINDOW_SECONDS,
    Filtered = [{T, D, S} || {T, D, S} <- Requests, T >= Cutoff],
    erlang:send_after(60000, self(), cleanup),
    {noreply, State#state{requests = Filtered}}.

compute_slo_status([]) ->
    #{status => no_data};
compute_slo_status(Recent) ->
    Total = length(Recent),
    Errors = length([ok || {_, false} <- Recent]),
    Durations = [D || {D, _} <- Recent],
    
    ErrorRate = Errors / Total,
    Availability = 1 - ErrorRate,
    SortedDurations = lists:sort(Durations),
    P99Index = round(length(SortedDurations) * 0.99),
    P99 = lists:nth(max(1, P99Index), SortedDurations),
    
    AvailSlo = Availability >= ?SLO_AVAILABILITY,
    P99Slo = P99 =< ?SLO_P99_MS,
    ErrorSlo = ErrorRate =< ?SLO_ERROR_RATE,
    
    %% Error budget: how much of our allowed errors have we used?
    AllowedErrors = Total * (1 - ?SLO_AVAILABILITY),
    ErrorBudgetUsed = case AllowedErrors > 0 of
        true -> Errors / AllowedErrors * 100;
        false -> 0.0
    end,
    
    #{
        total_requests => Total,
        error_rate => ErrorRate,
        availability => Availability,
        p99_ms => P99,
        error_budget_used_pct => ErrorBudgetUsed,
        slo_meeting => #{
            availability => AvailSlo,
            p99_latency => P99Slo,
            error_rate => ErrorSlo
        },
        overall => AvailSlo andalso P99Slo andalso ErrorSlo
    }.

%% Prometheus metrics integration
record_prometheus_metrics(#{
    availability := Avail,
    p99_ms := P99,
    error_rate := ErrorRate,
    error_budget_used_pct := Budget
}) ->
    prometheus_gauge:set(slo_availability, Avail),
    prometheus_gauge:set(slo_p99_latency_ms, P99),
    prometheus_gauge:set(slo_error_rate, ErrorRate),
    prometheus_gauge:set(slo_error_budget_used_pct, Budget).
```

---

## Capacity Planning

```erlang
%% capacity_planner.erl — model and predict capacity needs

-module(capacity_planner).
-export([analyze/0, forecast/1, alert_thresholds/0]).

analyze() ->
    Metrics = collect_metrics(),
    
    #{
        current => Metrics,
        headroom => compute_headroom(Metrics),
        forecast_30d => forecast(30),
        recommendations => make_recommendations(Metrics)
    }.

collect_metrics() ->
    #{
        cpu_util_pct => cpu_util(),
        memory_used_pct => memory_used_pct(),
        process_count => erlang:system_info(process_count),
        process_limit => erlang:system_info(process_limit),
        ets_memory_mb => ets_memory_mb(),
        run_queue => statistics(run_queue),
        req_per_sec => get_req_rate(),
        p99_latency_ms => get_p99()
    }.

cpu_util() ->
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(100),
    Stats = erlang:statistics(scheduler_wall_time),
    Total = lists:sum([A + I || {_, A, I} <- Stats]),
    Active = lists:sum([A || {_, A, _} <- Stats]),
    case Total of
        0 -> 0;
        _ -> round(Active * 100 / Total)
    end.

memory_used_pct() ->
    Total = erlang:memory(total),
    %% Typical container limit
    Limit = application:get_env(myapp, memory_limit_bytes, 4 * 1024 * 1024 * 1024),
    round(Total * 100 / Limit).

ets_memory_mb() ->
    Words = lists:sum([ets:info(T, memory) || T <- ets:all()]),
    Words * erlang:system_info(wordsize) div (1024 * 1024).

get_req_rate() ->
    prometheus_gauge:value(http_requests_per_second).

get_p99() ->
    prometheus_summary:value(http_request_duration_ms, [quantile], [{quantile, 0.99}]).

compute_headroom(#{
    cpu_util_pct := Cpu,
    memory_used_pct := Mem,
    process_count := Procs,
    process_limit := ProcLimit
}) ->
    #{
        cpu_headroom_pct => 100 - Cpu,
        memory_headroom_pct => 100 - Mem,
        process_headroom_pct => round((ProcLimit - Procs) * 100 / ProcLimit)
    }.

%% Simple linear forecast
forecast(DaysAhead) ->
    %% Get historical metrics (last 7 days)
    Historical = get_historical_metrics(7),
    
    case length(Historical) < 2 of
        true -> insufficient_data;
        false ->
            %% Linear regression on req/sec trend
            Rates = [maps:get(req_per_sec, M) || M <- Historical],
            DailyGrowth = (lists:last(Rates) - hd(Rates)) / length(Rates),
            CurrentRate = lists:last(Rates),
            ForecastRate = CurrentRate + DailyGrowth * DaysAhead,
            
            #{
                current_req_per_sec => CurrentRate,
                daily_growth_rate => DailyGrowth,
                forecast_req_per_sec => ForecastRate,
                needs_scaling => ForecastRate > current_capacity() * 0.8
            }
    end.

current_capacity() ->
    application:get_env(myapp, max_req_per_sec, 10000).

get_historical_metrics(_Days) ->
    %% Query Prometheus/InfluxDB for historical data
    [].

make_recommendations(Metrics) ->
    Recs = [],
    
    R1 = case maps:get(cpu_util_pct, Metrics) > 80 of
        true -> ["Scale horizontally: CPU above 80%"];
        false -> []
    end,
    
    R2 = case maps:get(memory_used_pct, Metrics) > 85 of
        true -> ["Increase memory or enable hibernation"];
        false -> []
    end,
    
    R3 = case maps:get(run_queue, Metrics) > 50 of
        true -> ["Add schedulers or reduce contention"];
        false -> []
    end,
    
    Recs ++ R1 ++ R2 ++ R3.

alert_thresholds() ->
    #{
        cpu_critical => 90,
        cpu_warning => 75,
        memory_critical => 90,
        memory_warning => 80,
        run_queue_critical => 100,
        run_queue_warning => 50,
        p99_latency_critical_ms => 1000,
        p99_latency_warning_ms => 500
    }.
```

---

## On-Call Runbooks

```erlang
%% Automated runbook execution for common incidents

-module(runbook).
-export([execute/1, available/0]).

available() ->
    [high_memory, high_cpu, connection_surge, slow_queries,
     process_leak, ets_bloat, message_queue_buildup].

execute(high_memory) ->
    logger:warning("Executing runbook: high_memory"),
    Steps = [
        {"1. Identify top memory consumers",
         fun() -> top_memory_consumers(10) end},
        {"2. Check for binary memory leaks",
         fun() -> check_binary_leaks() end},
        {"3. Force GC on all processes",
         fun() -> force_gc_all() end},
        {"4. Check ETS table sizes",
         fun() -> check_ets_sizes() end},
        {"5. Alert if still high after GC",
         fun() -> check_memory_after_gc() end}
    ],
    run_steps(Steps);

execute(high_cpu) ->
    Steps = [
        {"1. Check scheduler utilization",
         fun() -> scheduler_utilization() end},
        {"2. Find highest-reduction processes",
         fun() -> top_processes_by_reductions(10) end},
        {"3. Check run queue depth",
         fun() -> statistics(run_queue) end},
        {"4. Check for busy loops",
         fun() -> check_busy_loops() end}
    ],
    run_steps(Steps);

execute(message_queue_buildup) ->
    Steps = [
        {"1. Find processes with large mailboxes",
         fun() -> find_large_mailboxes(100) end},
        {"2. Sample messages in largest queue",
         fun() -> sample_mailbox(find_largest_mailbox()) end},
        {"3. Check if process is alive/responsive",
         fun() -> check_process_health(find_largest_mailbox()) end}
    ],
    run_steps(Steps).

run_steps([]) -> ok;
run_steps([{Name, Fun} | Rest]) ->
    logger:info("Runbook step: ~s", [Name]),
    Result = try Fun() catch E:R -> {error, {E, R}} end,
    logger:info("Result: ~p", [Result]),
    run_steps(Rest).

top_memory_consumers(N) ->
    All = [{process_info(P, [memory, registered_name, current_function]), P}
           || P <- erlang:processes()],
    Valid = [{I, P} || {{_, I}, P} <- All, I =/= undefined],
    Sorted = lists:sort(fun({A, _}, {B, _}) ->
        proplists:get_value(memory, A, 0) >
        proplists:get_value(memory, B, 0)
    end, Valid),
    lists:sublist(Sorted, N).

check_binary_leaks() ->
    BinMemMb = erlang:memory(binary) div (1024*1024),
    case BinMemMb > 500 of
        true ->
            %% Force GC on processes with large binary references
            [erlang:garbage_collect(P) || P <- erlang:processes()],
            {warning, {binary_memory_mb, BinMemMb}};
        false ->
            {ok, {binary_memory_mb, BinMemMb}}
    end.

force_gc_all() ->
    Pids = erlang:processes(),
    [erlang:garbage_collect(P) || P <- Pids],
    {ok, {gc_forced_on, length(Pids), processes}}.

check_ets_sizes() ->
    Tables = [{T, ets:info(T, size), ets:info(T, memory)}
              || T <- ets:all()],
    Large = [{T, Size, Mem} || {T, Size, Mem} <- Tables,
             Mem > 10000],  %% > 10K words
    {ok, {large_ets_tables, Large}}.

check_memory_after_gc() ->
    MemMb = erlang:memory(total) div (1024*1024),
    Limit = application:get_env(myapp, memory_limit_mb, 4096),
    case MemMb > Limit * 0.85 of
        true ->
            alert_manager:page(high_memory, #{
                current_mb => MemMb,
                limit_mb => Limit
            }),
            {alert_sent, {memory_mb, MemMb}};
        false ->
            {ok, {memory_mb, MemMb}}
    end.

scheduler_utilization() ->
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(500),
    Stats = erlang:statistics(scheduler_wall_time),
    [{S, round(A * 100 / max(A + I, 1))} || {S, A, I} <- Stats].

top_processes_by_reductions(N) ->
    All = [{process_info(P, [reductions, registered_name]), P}
           || P <- erlang:processes()],
    Valid = [{I, P} || {{_, I}, P} <- All, I =/= undefined],
    Sorted = lists:sort(fun({A, _}, {B, _}) ->
        proplists:get_value(reductions, A, 0) >
        proplists:get_value(reductions, B, 0)
    end, Valid),
    lists:sublist(Sorted, N).

check_busy_loops() ->
    %% Processes with high reductions but small heaps are likely looping
    All = [process_info(P, [reductions, heap_size, current_function])
           || P <- erlang:processes()],
    BusyLoops = [I || I <- All,
                 I =/= undefined,
                 proplists:get_value(reductions, I, 0) > 1000000,
                 proplists:get_value(heap_size, I, 0) < 1000],
    {potential_busy_loops, length(BusyLoops), BusyLoops}.

find_large_mailboxes(Threshold) ->
    All = [{process_info(P, [message_queue_len, registered_name]), P}
           || P <- erlang:processes()],
    Large = [{I, P} || {{_, I}, P} <- All,
             I =/= undefined,
             proplists:get_value(message_queue_len, I, 0) > Threshold],
    lists:sort(fun({A, _}, {B, _}) ->
        proplists:get_value(message_queue_len, A) >
        proplists:get_value(message_queue_len, B)
    end, Large).

find_largest_mailbox() ->
    case find_large_mailboxes(0) of
        [{_, Pid} | _] -> Pid;
        [] -> undefined
    end.

sample_mailbox(undefined) -> {error, no_process};
sample_mailbox(Pid) ->
    %% Get first few messages without consuming them
    process_info(Pid, messages).

check_process_health(undefined) -> {error, no_process};
check_process_health(Pid) ->
    case process_info(Pid, [status, current_function, message_queue_len]) of
        undefined -> {error, process_dead};
        Info -> {ok, Info}
    end.
```

---

## สรุป Part 99

| SRE Pillar | Erlang Implementation |
|-----------|----------------------|
| SLOs | Rolling window metrics with error budget |
| Capacity planning | Linear trend forecasting |
| Incident response | Automated runbooks |
| Observability | process_info + sys module |
| Chaos engineering | Failure injection framework |
| Cost optimization | Hibernation + ETS cleanup |

---

*[← Part 98: Financial Systems](part_98_financial_systems.md) | [Part 100: Course Completion →](part_100_course_completion.md)*
