# Part 69: Performance Profiling และ Optimization

## สารบัญ
1. [fprof — Function Profiler](#fprof)
2. [eprof — Time Profiler](#eprof)
3. [cprof — Call Count Profiler](#cprof)
4. [recon Library](#recon)
5. [Flamegraph Visualization](#flamegraph)
6. [ตัวอย่างจริง: Optimize Hot Path](#hot-path)

---

## fprof — Function Profiler

```erlang
%% fprof: detailed profiling with call graph
%% Best for: understanding where time is spent

%% Profile specific function
profile_function() ->
    fprof:apply(my_module, expensive_function, [arg1, arg2]),
    fprof:profile(),
    fprof:analyse([{dest, "/tmp/fprof_analysis.txt"}, {totals, true}]).

%% Profile a running system
profile_running() ->
    fprof:trace(start),
    timer:sleep(10000),  %% trace for 10 seconds
    fprof:trace(stop),
    fprof:profile(),
    fprof:analyse([
        {dest, "/tmp/profile.txt"},
        {totals, true},
        {sort, own}  %% sort by own time (not including callees)
    ]).

%% Profile specific process
profile_process(Pid) ->
    fprof:trace(start, [cpu_timestamp, {procs, [Pid]}]),
    timer:sleep(5000),
    fprof:trace(stop),
    fprof:profile(),
    fprof:analyse([{dest, "/tmp/proc_profile.txt"}]).

%% Read fprof output
%% {TOTALS,   0.234,  0.189,  <<"..."}
%% ^ module  ^ acc%  ^ own%   ^ function name
%% acc% = time including callees, own% = time in this function only

%% Filter output to see hot functions
parse_fprof_analysis(File) ->
    {ok, Binary} = file:read_file(File),
    Lines = binary:split(Binary, <<"\n">>, [global]),
    %% Find functions with own time > 1%
    HotLines = [L || L <- Lines, 
                     re:run(L, "\\d+\\.\\d+\\s+\\d+\\.\\d+") =/= nomatch],
    HotLines.
```

---

## eprof — Time Profiler

```erlang
%% eprof: simpler profiler, good for overview
%% Lower overhead than fprof

%% Profile a function call
eprof_function() ->
    eprof:start_profiling([self()]),
    my_module:my_function(args),
    eprof:stop_profiling(),
    eprof:analyze(total),  %% or 'procs'
    eprof:log("/tmp/eprof.txt").

%% Profile multiple processes
eprof_processes(Pids) ->
    eprof:start_profiling(Pids),
    timer:sleep(5000),
    eprof:stop_profiling(),
    eprof:analyze(procs).  %% per-process breakdown

%% Typical eprof output:
%%                                               FUNCTION      CALLS        %      TIME  [uS / CALLS]
%% -----------------------------------------------------------------------
%% erts_internal                          trace_pattern/2          1        0.0         0    [0.00]
%% my_app                           expensive_compute/1        500       15.3    153000  [306.00]
%% my_app                                 decode_json/1       5000       42.1    421000   [84.20]

%% Focus: find function with highest % time
```

---

## cprof — Call Count Profiler

```erlang
%% cprof: lightest profiler, counts calls only
%% Safe to use in production

%% Count calls to specific module
cprof_module() ->
    cprof:start(),
    my_module:run_workload(),
    cprof:pause(),
    
    %% Analyze
    Analysis = cprof:analyse(my_module),
    io:format("~p~n", [Analysis]),
    cprof:stop().

%% Count calls across all modules
cprof_all() ->
    cprof:start(),
    timer:sleep(1000),
    cprof:pause(),
    
    Analysis = cprof:analyse(),
    %% Sort by call count
    Sorted = lists:sort(fun({_, C1}, {_, C2}) -> C1 > C2 end, Analysis),
    Top10 = lists:sublist(Sorted, 10),
    io:format("Top 10 called functions:~n~p~n", [Top10]),
    
    cprof:stop().

%% Output example:
%% [{my_module, 42, [{my_module, hot_function, 1, 28},
%%                   {my_module, helper, 2, 14}]}]
```

---

## recon Library

```erlang
%% recon: production-safe introspection
%% {deps, [{recon, "2.5.4"}]}

%% Top processes by reductions (CPU work)
top_by_reductions() ->
    recon:proc_count(reductions, 10).

%% Top by memory
top_by_memory() ->
    recon:proc_count(memory, 10).

%% Top by message queue length
top_by_queue() ->
    recon:proc_count(message_queue_len, 10).

%% Trace function calls (safe, limited)
trace_function() ->
    recon_trace:calls({my_module, my_function, '_'}, 100),
    %% Will trace at most 100 calls, then auto-stops
    timer:sleep(5000),
    recon_trace:clear().

%% Trace with args and return value
trace_with_return() ->
    recon_trace:calls({my_module, my_function, '_'},
                      100,
                      [{scope, local}, return_to]).

%% Trace all calls to a module
trace_module() ->
    recon_trace:calls({my_module, '_', '_'}, 50).

%% Memory analysis
memory_analysis() ->
    #{
        total => recon_alloc:memory(usage),
        fragmentation => recon_alloc:fragmentation(current),
        allocated => recon_alloc:memory(allocated),
        used => recon_alloc:memory(used)
    }.

%% Find processes leaking memory over time
find_memory_growth() ->
    Snapshot1 = recon:proc_window(memory, 5, 2000),
    timer:sleep(2000),
    Snapshot2 = recon:proc_window(memory, 5, 2000),
    
    %% Compare
    PidsS1 = maps:from_list([{Pid, Mem} || {Pid, Mem, _} <- Snapshot1]),
    [{Pid, Mem2 - maps:get(Pid, PidsS1, 0)}
     || {Pid, Mem2, _} <- Snapshot2,
        maps:get(Pid, PidsS1, 0) < Mem2].
```

---

## Flamegraph Visualization

```erlang
%% Generate flamegraph data from fprof output

%% Using erlang-profiler or perf + stackcollapse
generate_flamegraph() ->
    %% Step 1: Profile
    fprof:apply(my_module, main, []),
    fprof:profile(),
    fprof:analyse([{dest, "/tmp/fprof.txt"}, {callers, true}]),
    
    %% Step 2: Convert to flamegraph format
    %% Use: https://github.com/brendangregg/FlameGraph
    %% fprof2fg.escript converts fprof output to folded stacks
    
    %% Step 3: Generate SVG
    %% ./flamegraph.pl /tmp/folded_stacks.txt > flame.svg
    
    %% Alternative: use recon_trace + dtrace/systemtap
    ok.

%% Linux perf-based profiling
linux_perf_profile(Pid, DurationSec) ->
    os:cmd(io_lib:format("perf record -F 99 -p ~p -g -- sleep ~p", [Pid, DurationSec])),
    os:cmd("perf script | ./stackcollapse-perf.pl > /tmp/perf.folded"),
    os:cmd("./flamegraph.pl /tmp/perf.folded > /tmp/flame.svg"),
    {ok, "/tmp/flame.svg"}.
```

---

## ตัวอย่างจริง: Optimize Hot Path

```erlang
%% Before optimization: slow JSON processing

slow_process_events(Events) ->
    [begin
        %% Problem 1: parse JSON on every event
        Decoded = jsx:decode(Event, [return_maps]),
        %% Problem 2: create new map on every transformation
        Transformed = maps:fold(fun(K, V, Acc) ->
            Acc#{string:lowercase(K) => V}
        end, #{}, Decoded),
        %% Problem 3: format string is slow
        io:format("Processed: ~p~n", [Transformed])
    end || Event <- Events].

%% After profiling: found 80% time in jsx:decode and maps:fold

fast_process_events(Events) ->
    [begin
        %% Fix 1: use jiffy (C NIF, faster than jsx)
        Decoded = jiffy:decode(Event, [return_maps]),
        %% Fix 2: specific key access instead of fold
        UserId = maps:get(<<"userId">>, Decoded),
        EventType = maps:get(<<"eventType">>, Decoded),
        %% Fix 3: structured logging (no io_lib:format)
        logger:info("Processed", #{user => UserId, type => EventType})
    end || Event <- Events].

%% Benchmark
benchmark() ->
    Events = generate_test_events(10000),
    
    {SlowTime, _} = timer:tc(fun() -> slow_process_events(Events) end),
    {FastTime, _} = timer:tc(fun() -> fast_process_events(Events) end),
    
    io:format("Slow: ~p ms, Fast: ~p ms, Speedup: ~.1fx~n",
              [SlowTime div 1000, FastTime div 1000, SlowTime / FastTime]).

%% Another optimization: binary matching instead of string operations
slow_parse_log(Line) ->
    Parts = string:split(binary_to_list(Line), " "),
    list_to_binary(lists:nth(3, Parts)).

fast_parse_log(Line) ->
    %% Pattern match directly on binary — O(n) scan, no list conversion
    case Line of
        <<_:_/binary, " ", _:_/binary, " ", Level:4/binary, _/binary>> ->
            Level;
        _ ->
            <<>>
    end.

%% ETS operations: read_concurrency for hot tables
setup_fast_table() ->
    ets:new(hot_data, [
        named_table,
        set,
        public,
        {read_concurrency, true},   %% multiple readers without lock
        {write_concurrency, true}   %% concurrent writes use fine-grained locking
    ]).

%% Avoid term_to_binary/binary_to_term in hot path
slow_cache_key(UserId, Resource) ->
    term_to_binary({UserId, Resource}).  %% heap allocation, slow

fast_cache_key(UserId, Resource) when is_binary(UserId), is_binary(Resource) ->
    <<UserId/binary, ":", Resource/binary>>.  %% binary concat, fast
```

---

## สรุป Part 69

| Profiler | Overhead | Best For |
|----------|----------|----------|
| cprof | Very low | Production call counts |
| eprof | Low | Time distribution overview |
| fprof | High | Detailed call graph |
| recon_trace | Configurable | Production tracing |
| Linux perf | Low | Native/NIF profiling |

**Profile → Identify hot path → Measure → Optimize → Measure again**

---

*[← Part 68: Advanced OTP](part_68_advanced_otp.md) | [Part 70: Production War Stories →](part_70_war_stories.md)*
