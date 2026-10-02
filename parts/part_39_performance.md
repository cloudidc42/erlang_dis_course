# Part 39: Performance Tuning

## สารบัญ
1. [Profiling — หาคอขวด](#profiling--หาคอขวด)
2. [VM Tuning — BEAM Parameters](#vm-tuning--beam-parameters)
3. [Memory Management](#memory-management)
4. [Process Scheduling](#process-scheduling)
5. [Binary Optimization](#binary-optimization)
6. [ETS Performance](#ets-performance)
7. [ตัวอย่างจริง: Benchmarking](#ตัวอย่างจริง-benchmarking)

---

## Profiling — หาคอขวด

```erlang
%% fprof: function call profiler (overhead สูง, ใช้ใน dev เท่านั้น)

%% รัน profiling
fprof:apply(my_module, my_function, [arg1]).
fprof:profile().
fprof:analyse([totals, {dest, "profile.txt"}]).

%% ดู output: total time, own time, call count
%% {[totals], 0, 0, [{Caller, Count, Acc, Own}]}

%% eprof: lightweight profiler
eprof:start_profiling([self()]).
my_heavy_function().
eprof:stop_profiling().
eprof:analyze(total).

%% cprof: call count profiler (lightest overhead)
cprof:start().
my_function().
cprof:pause().
cprof:analyse(my_module).
cprof:stop().

%% percept2: concurrent profiling
percept2:profile("output.dat",
    fun() -> my_concurrent_code() end,
    [procs, ports, schedulers]).
percept2:analyze("output.dat").
%% percept2:start_webserver(8888)  %% web UI

%% timer:tc: simple timing
{Microseconds, Result} = timer:tc(fun() -> my_function() end).
{Microseconds, Result} = timer:tc(my_module, my_function, [Args]).

%% Erlang tracing
erlang:trace(all, true, [call, timestamp]).
erlang:trace_pattern({my_module, '_', '_'}, true, [local]).

%% recon_trace (safe production tracing)
%% deps: {recon, "2.5.4"}
recon_trace:calls({my_module, my_function, '_'}, 100).
recon_trace:clear().
```

---

## VM Tuning — BEAM Parameters

```bash
# Scheduler tuning
+S 8:8              # 8 schedulers, 8 online
+SP 50:50           # limit to 50% of logical processors
+sbt db             # bind schedulers to processor groups

# Dirty schedulers (I/O, NIFs)
+SDio 8             # 8 dirty I/O schedulers
+SDcpu 8            # 8 dirty CPU schedulers

# Reduction budget
+zdbbl 8192         # distribution buffer busy limit (KB)

# Process limits
+P 10000000         # max processes (default 262144)

# Port limits
+Q 65536            # max ports

# Atom table
+t 1048576          # max atoms

# Kernel poll
+K true             # use OS kernel polling (select/epoll/kqueue)

# Async threads (for file I/O, deprecated in OTP 24+)
+A 128              # async threads

# Stack sizes
+hms 4096           # min heap size (words)
+hmbs 46422         # min binary heap size (words)
+hmax 0             # max heap size (0 = unlimited)

# GC
+c true             # cpu_sup (CPU usage monitoring)

# Memory allocator
+M (many options):
+Mmcs 40            # multiblock carrier size
+Mm <size>          # mbcs_pool size

# Crash dump
+d                  # suppress core dump
-env ERL_CRASH_DUMP_SECONDS 10  # time to write crash dump

# Full example for production:
# erl +K true +P 10000000 +Q 65536 +A 0 +SDio 8 \
#     +zdbbl 8192 +hms 4096 -smp enable -sname myapp
```

---

## Memory Management

```erlang
%% Erlang GC: per-process, generational, copying
%% - Minor GC: frequent, fast (young generation)
%% - Major GC: less frequent, full heap
%% - Fullsweep: copy entire heap (triggered by threshold)

%% Monitor memory
erlang:memory().
%% [{total, N}, {processes, N}, {system, N}, {atom, N},
%%  {binary, N}, {code, N}, {ets, N}]

erlang:memory(total).
erlang:memory([processes, binary, ets]).

%% Process memory
process_info(Pid, [memory, heap_size, stack_size, message_queue_len]).

%% Trigger GC
erlang:garbage_collect().          %% current process
erlang:garbage_collect(Pid).       %% specific process

%% GC options per process
spawn_opt(fun() -> ... end, [{min_heap_size, 1024}, {fullsweep_after, 0}]).
erlang:process_flag(min_heap_size, 1024).
erlang:process_flag(fullsweep_after, 0).  %% always full GC (safer for long-lived)
erlang:process_flag(max_heap_size, #{size => 1048576, kill => true}).

%% Binary memory: shared heap, reference counted
%% Large binaries (> 64 bytes) เก็บใน binary heap, shared
%% เมื่อ reference ลด -> GC

%% Memory leak via binary:
%% ระวัง: reference ไปยัง sub-binary ทำให้ parent binary ไม่ถูก GC
safe_binary(<<_:8, Rest/binary>>) ->
    binary:copy(Rest).  %% copy to free parent reference

%% recon: memory analysis
recon:proc_count(memory, 10).     %% top 10 memory processes
recon:proc_count(binary_memory, 10).
recon_alloc:memory(usage).         %% allocator usage
recon_alloc:fragmentation(current). %% memory fragmentation

%% Identifying memory leaks
%% 1. Monitor erlang:memory(total) over time
%% 2. Use recon:proc_count(memory, N) to find large processes
%% 3. Check message queues: process_info(Pid, message_queue_len)
%% 4. ETS: ets:info(table, memory) * erlang:system_info(wordsize)
```

---

## Process Scheduling

```erlang
%% Erlang scheduler: preemptive, based on reductions
%% 1 reduction = 1 function call (approximately)
%% Default: 2000 reductions per timeslice
%% ~1ms per timeslice

%% Scheduler info
erlang:system_info(schedulers).        %% number of schedulers
erlang:system_info(schedulers_online). %% online schedulers
erlang:statistics(scheduler_wall_time). %% per-scheduler wall time

%% Process priority (use sparingly)
spawn_opt(fun() -> ... end, [{priority, high}]).
erlang:process_flag(priority, high).
%% Priorities: max > high > normal > low

%% Yield explicitly (long-running pure computation)
yield_heavy_loop([], Acc) -> Acc;
yield_heavy_loop([H|T], Acc) ->
    case erlang:yield() of  %% allow other processes to run
        true -> yield_heavy_loop(T, [compute(H)|Acc])
    end.

%% Or schedule in chunks
process_in_chunks([], _ChunkSize) -> ok;
process_in_chunks(List, ChunkSize) ->
    {Chunk, Rest} = case length(List) > ChunkSize of
        true -> lists:split(ChunkSize, List);
        false -> {List, []}
    end,
    [process_item(I) || I <- Chunk],
    erlang:send_after(0, self(), {continue, Rest}),
    receive
        {continue, R} -> process_in_chunks(R, ChunkSize)
    end.

%% Async dirty scheduler (for CPU-bound NIFs)
%% nif:heavy_computation(Data).  %% runs in dirty scheduler
%% doesn't block normal schedulers

%% Monitor scheduler utilization
msacc:start().
timer:sleep(1000).
msacc:stop().
msacc:print(msacc:stats()).
```

---

## Binary Optimization

```erlang
%% Binary: most common performance topic in Erlang

%% 1. Pattern matching on binary is O(1) for head, efficient for tail
parse_header(<<Magic:4/binary, Len:32/big, Rest/binary>>) ->
    {Magic, Len, Rest}.

%% 2. Avoid unnecessary copies
%% Bad: concatenate in loop
bad_concat(List) ->
    lists:foldl(fun(B, Acc) -> <<Acc/binary, B/binary>> end, <<>>, List).

%% Good: IOList, then convert once
good_concat(List) ->
    iolist_to_binary(List).

%% 3. Binary comprehension
build_binary(List) ->
    << <<X:8>> || X <- List >>.

%% 4. binary:copy for freeing sub-binary references
safe_subbin(Bin) ->
    binary:copy(Bin).

%% 5. refc binaries vs heap binaries
%% > 64 bytes: refc (reference counted, off process heap)
%% <= 64 bytes: heap binary (on process heap)
%% Refc binaries are shared: passing them to other processes is O(1)

%% 6. Matching vs binary module
%% Pattern matching is usually faster for known formats
%% binary module for dynamic operations

%% Benchmark: pattern match vs binary:part
bench_pm(Bin) ->
    <<_:8, Rest/binary>> = Bin,
    Rest.

bench_bp(Bin) ->
    binary:part(Bin, 1, byte_size(Bin) - 1).

%% 7. Binaries in ETS
%% Large binaries in ETS: stored once, shared
ets:insert(cache, {key, LargeBinary}).
%% LargeBinary is NOT copied into ETS, it's shared

%% When you read back, you get a reference (not a copy)
[{key, Bin}] = ets:lookup(cache, key).
%% Bin references the ETS copy, no copy made
```

---

## ETS Performance

```erlang
%% ETS is designed for high-performance concurrent access

%% Optimize reads
ets:new(cache, [
    set,
    named_table,
    public,
    {read_concurrency, true},   %% optimize concurrent reads
    {write_concurrency, true},  %% optimize concurrent writes
    {decentralized_counters, true}  %% OTP 23+: faster counters
]).

%% read_concurrency vs write_concurrency
%% read_concurrency: better for read-heavy (many concurrent reads)
%% write_concurrency: better for write-heavy (many concurrent writes)
%% Both can cause overhead for sequential access

%% Ordered set vs Set
%% ordered_set: O(log N) for all ops, sorted
%% set/bag: O(1) average for all ops

%% ETS select (compiled match spec) is faster than match
MatchSpec = ets:fun2ms(fun({_, Age, _} = X) when Age > 18 -> X end).
ets:select(my_table, MatchSpec).

%% Update counter atomically
ets:update_counter(counters, hit_count, 1).
ets:update_counter(counters, hit_count, {2, 1}).  %% increment pos 2

%% Bulk operations
ets:insert(my_table, [{K1, V1}, {K2, V2}, {K3, V3}]).  %% atomic bulk insert

%% Table inheritance: transfer when owner dies
ets:new(shared_cache, [named_table, {heir, supervisor_pid, transfer_data}]).

%% Sizing: measure ETS memory
ets_memory_bytes(Table) ->
    ets:info(Table, memory) * erlang:system_info(wordsize).
```

---

## ตัวอย่างจริง: Benchmarking

```erlang
%% ไฟล์: bench.erl
-module(bench).
-export([run/0, compare/3]).

%% Simple benchmark
bench(Name, Fun, N) ->
    {Time, _} = timer:tc(fun() ->
        [Fun() || _ <- lists:seq(1, N)]
    end),
    Avg = Time / N,
    io:format("~p: ~.2f µs/op (~.0f ops/sec)~n",
              [Name, Avg, 1000000 / Avg]).

%% Compare two implementations
compare(Name1, Fun1, Name2, Fun2) ->
    N = 10000,
    {T1, _} = timer:tc(fun() -> [Fun1() || _ <- lists:seq(1, N)] end),
    {T2, _} = timer:tc(fun() -> [Fun2() || _ <- lists:seq(1, N)] end),
    if T1 < T2 ->
        io:format("~p is ~.1fx faster than ~p~n", [Name1, T2/T1, Name2]);
    true ->
        io:format("~p is ~.1fx faster than ~p~n", [Name2, T1/T2, Name1])
    end.

run() ->
    %% String concatenation
    io:format("=== Binary concatenation ===~n"),
    bench(iolist, fun() ->
        iolist_to_binary([<<"hello">>, <<" ">>, <<"world">>])
    end, 100000),
    bench(concat, fun() ->
        <<"hello">> = <<<<"hello">>/binary, <<" ">>/binary, <<"world">>/binary>>
    end, 100000),

    %% Map vs Record access
    io:format("~n=== Map vs Record access ===~n"),
    Map = #{name => alice, age => 30},
    bench(map_get, fun() -> maps:get(name, Map) end, 1000000),
    R = {rec, alice, 30},
    bench(record_get, fun() -> element(2, R) end, 1000000),

    %% ETS vs process state
    io:format("~n=== ETS vs process dict ===~n"),
    ets:new(bench_ets, [named_table, set, public]),
    ets:insert(bench_ets, {key, value}),
    bench(ets_lookup, fun() -> ets:lookup(bench_ets, key) end, 1000000),
    put(key, value),
    bench(proc_dict, fun() -> get(key) end, 1000000).

%% Output example:
%% === Binary concatenation ===
%% iolist: 0.12 µs/op (8333333 ops/sec)
%% concat: 0.18 µs/op (5555556 ops/sec)
%%
%% === Map vs Record access ===
%% map_get: 0.08 µs/op (12500000 ops/sec)
%% record_get: 0.05 µs/op (20000000 ops/sec)
%%
%% === ETS vs process dict ===
%% ets_lookup: 0.15 µs/op (6666667 ops/sec)
%% proc_dict: 0.04 µs/op (25000000 ops/sec)
```

---

## สรุป Part 39: Performance Tips

| Tip | Impact |
|-----|--------|
| Use IOList instead of binary concat | High |
| `{read_concurrency, true}` on ETS | High (multi-thread) |
| `binary:copy` after sub-binary | Medium (GC) |
| `fullsweep_after, 0` for long-lived | Medium |
| Pattern match vs maps:get | Low |
| `+K true` kernel poll | High (I/O heavy) |
| Dirty scheduler for NIFs | Critical (heavy NIF) |
| Avoid large message queues | High |

---

*[← Part 38: Mnesia Advanced](part_38_mnesia_advanced.md) | [Part 40: Observability →](part_40_observability.md)*
