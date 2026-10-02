# Part 52: Memory and Performance Deep Dive

## สารบัญ
1. [BEAM Memory Architecture](#beam-memory-architecture)
2. [Process Heap Internals](#process-heap-internals)
3. [Binary Heap และ ProcBin](#binary-heap-และ-procbin)
4. [Allocator Tuning](#allocator-tuning)
5. [Scheduler Deep Dive](#scheduler-deep-dive)
6. [ตัวอย่างจริง: Memory Leak Investigation](#ตัวอย่างจริง-memory-leak-investigation)

---

## BEAM Memory Architecture

```erlang
%% BEAM memory areas:
%%
%% ┌────────────────────────────────────────┐
%% │              System Memory              │
%% ├────────────────────────────────────────┤
%% │  Atom table  │  Module table  │  Code   │
%% ├────────────────────────────────────────┤
%% │           ETS Memory                   │
%% ├────────────────────────────────────────┤
%% │    Process heaps (per-process GC)       │
%% │  ┌────────┐ ┌────────┐ ┌────────┐    │
%% │  │ Proc 1  │ │ Proc 2  │ │ Proc N  │    │
%% │  └────────┘ └────────┘ └────────┘    │
%% ├────────────────────────────────────────┤
%% │  Binary heap (shared, ref-counted)     │
%% └────────────────────────────────────────┘

%% Memory stats
memory_report() ->
    Mem = erlang:memory(),
    io:format("Memory Report:~n"),
    [io:format("  ~p: ~s~n", [K, format_bytes(V)]) || {K, V} <- Mem].

format_bytes(B) when B < 1024 -> io_lib:format("~B B", [B]);
format_bytes(B) when B < 1048576 -> io_lib:format("~.1f KB", [B/1024]);
format_bytes(B) when B < 1073741824 -> io_lib:format("~.1f MB", [B/1048576]);
format_bytes(B) -> io_lib:format("~.2f GB", [B/1073741824]).

%% Detailed memory breakdown
detailed_memory() ->
    #{
        total => erlang:memory(total),
        processes => erlang:memory(processes),
        processes_used => erlang:memory(processes_used),
        system => erlang:memory(system),
        atom => erlang:memory(atom),
        atom_used => erlang:memory(atom_used),
        binary => erlang:memory(binary),
        code => erlang:memory(code),
        ets => erlang:memory(ets)
    }.
```

---

## Process Heap Internals

```erlang
%% Process heap: private to each process
%% Stack + Heap in single memory area (grows from both ends)
%%
%%  ┌──────────────────────┐  High address
%%  │      Stack           │  <- grows down
%%  │                      │
%%  │    (free space)      │
%%  │                      │
%%  │      Heap            │  -> grows up
%%  └──────────────────────┘  Low address
%%  Old heap (after GC copy)
%%
%% Minor GC: collect young generation only
%% Full GC: collect everything, triggered by fullsweep_after

%% Process memory info
process_memory_info(Pid) ->
    Info = process_info(Pid, [
        memory,          %% total memory in bytes
        heap_size,       %% heap size in words
        stack_size,      %% stack depth in words
        total_heap_size, %% including old heap
        minor_gcs,       %% minor GC count
        garbage_collection
    ]),
    maps:from_list(Info).

%% GC tuning per process
spawn_with_tuned_gc(Fun) ->
    spawn_opt(Fun, [
        {min_heap_size, 1024},     %% avoid early GCs for small processes
        {min_bin_vheap_size, 46422}, %% binary heap virtual size
        {fullsweep_after, 10},      %% full GC every 10 minor GCs
        {max_heap_size, #{
            size => 100 * 1024 * 1024,  %% 100MB max
            kill => true,               %% kill process if exceeded
            error_logger => true        %% log before killing
        }}
    ]).

%% Monitor for memory-hungry processes
find_memory_hogs(Top) ->
    Processes = processes(),
    Memories = [{P, process_info(P, memory)} || P <- Processes],
    Valid = [{P, M} || {P, {memory, M}} <- Memories],
    Sorted = lists:reverse(lists:keysort(2, Valid)),
    lists:sublist(Sorted, Top).

%% Detect process memory leaks
detect_growing_processes() ->
    %% Sample every 30 seconds, alert if memory grew > 10MB
    Snapshot1 = get_process_snapshots(),
    timer:sleep(30000),
    Snapshot2 = get_process_snapshots(),
    find_growers(Snapshot1, Snapshot2, 10 * 1024 * 1024).

get_process_snapshots() ->
    maps:from_list([{P, M} ||
        P <- processes(),
        {memory, M} <- [process_info(P, memory)],
        M =/= undefined
    ]).

find_growers(Before, After, Threshold) ->
    maps:fold(fun(Pid, AfterMem, Acc) ->
        BeforeMem = maps:get(Pid, Before, 0),
        Growth = AfterMem - BeforeMem,
        case Growth > Threshold of
            true ->
                Name = process_info(Pid, registered_name),
                [{Pid, Name, BeforeMem, AfterMem, Growth} | Acc];
            false -> Acc
        end
    end, [], After).
```

---

## Binary Heap และ ProcBin

```erlang
%% Binary types:
%% HeapBin: <= 64 bytes, stored ON process heap, copied on send
%% ReferencedBin (ProcBin): > 64 bytes, stored in SHARED binary heap
%%                           reference counted, O(1) to send to other process

%% Common binary memory leak:
bad_process(LargeBin) ->
    %% Extracting sub-binary keeps reference to ENTIRE LargeBin
    <<_:8, SubBin/binary>> = LargeBin,
    %% SubBin.data points into LargeBin's memory
    %% LargeBin cannot be GC'd until SubBin is gone!
    process_sub(SubBin).

%% Fix: binary:copy to break the reference
good_process(LargeBin) ->
    <<_:8, SubBin/binary>> = LargeBin,
    CopiedSub = binary:copy(SubBin),  %% independent copy
    process_sub(CopiedSub).
    %% Now LargeBin can be GC'd

%% Detect binary memory pressure
binary_memory_usage() ->
    Processes = processes(),
    Bins = [begin
        case process_info(P, binary) of
            {binary, Bs} ->
                Total = lists:sum([Size || {_, Size, _} <- Bs]),
                {P, Total};
            _ -> {P, 0}
        end
    end || P <- Processes],
    lists:reverse(lists:keysort(2, Bins)).

%% Force GC to release binary references
flush_binaries(Pid) ->
    erlang:garbage_collect(Pid, []).

%% Check if GC is needed
needs_gc(Pid) ->
    case process_info(Pid, [binary, garbage_collection]) of
        [{binary, Bins}, {garbage_collection, GCInfo}] ->
            BinaryMem = lists:sum([S || {_, S, _} <- Bins]),
            HeapMem = proplists:get_value(heap_size, GCInfo, 0),
            BinaryMem > HeapMem * 8;  %% binary >> heap = leak
        _ -> false
    end.
```

---

## Allocator Tuning

```erlang
%% BEAM allocator: manages memory in carriers (large blocks)
%% Multiblock carrier (mbcs): multiple small objects
%% Singleblock carrier (sbcs): single large object

%% View allocator stats
allocator_stats() ->
    recon_alloc:memory(usage).

%% Common metrics:
%% usage = used / (used + unused) — fragmentation indicator
%% If usage < 50%, high fragmentation → consider tuning

%% VM flags for allocator tuning:
%%
%% +MBacul 0        %% disable carrier utilization limit
%% +MBas aobf       %% allocation strategy: address-order best-fit
%% +MBasbcst 0      %% no singleblock carrier size threshold
%% +MMmcs 40        %% max multiblock carrier size (MB)
%%
%% For long-running systems with fragmentation:
%% +MBas aobf +MHas aobf  %% best-fit strategy

%% Check carrier fragmentation
carrier_fragmentation() ->
    recon_alloc:fragmentation(current).
    %% Returns per-allocator fragmentation

%% Compact memory: move to smaller carriers
compact_memory() ->
    erlang:system_flag(fullsweep_after, 0),
    [erlang:garbage_collect(P) || P <- processes()],
    erlang:system_flag(fullsweep_after, 65535).
```

---

## Scheduler Deep Dive

```erlang
%% BEAM scheduler: preemptive, cooperative within timeslice
%% Each scheduler runs on 1 OS thread
%% Processes get ~2000 reductions before preemption

%% Scheduler stats
scheduler_stats() ->
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(1000),
    Stats = erlang:statistics(scheduler_wall_time),
    [begin
        Util = Active / (Active + Idle) * 100,
        #{scheduler => S, utilization => Util}
    end || {S, Active, Idle} <- Stats].

%% Detect scheduler imbalance
%% (some schedulers much busier than others)
scheduler_imbalance() ->
    Stats = scheduler_stats(),
    Utils = [maps:get(utilization, S) || S <- Stats],
    Max = lists:max(Utils),
    Min = lists:min(Utils),
    Imbalance = Max - Min,
    #{max => Max, min => Min, imbalance => Imbalance}.

%% Process migration: schedulers steal work from busy schedulers
%% This is automatic, but can tune:
%%
%% +sbt db   %% bind schedulers to processor groups (NUMA)
%% +scl false %% disable scheduler compaction (force all schedulers active)

%% Measure reduction rate
reduction_rate() ->
    {R1, _} = erlang:statistics(reductions),
    timer:sleep(1000),
    {R2, _} = erlang:statistics(reductions),
    R2 - R1.  %% reductions per second

%% Dirty scheduler usage
dirty_scheduler_stats() ->
    #{
        dirty_cpu => erlang:system_info(dirty_cpu_schedulers),
        dirty_cpu_online => erlang:system_info(dirty_cpu_schedulers_online),
        dirty_io => erlang:system_info(dirty_io_schedulers)
    }.

%% Detect I/O bottleneck
io_wait_time() ->
    Stats = erlang:statistics(io),
    {input, In} = proplists:lookup(input, Stats),
    {output, Out} = proplists:lookup(output, Stats),
    #{input_bytes => In, output_bytes => Out}.
```

---

## ตัวอย่างจริง: Memory Leak Investigation

```erlang
%% Tools and procedure for investigating memory leaks

%% Step 1: Confirm memory is growing
monitor_memory_trend(Minutes) ->
    Samples = collect_samples(Minutes * 60, 60),  %% 1 sample/second
    analyze_trend(Samples).

collect_samples(TotalSeconds, _) when TotalSeconds =< 0 -> [];
collect_samples(Total, Interval) ->
    Sample = #{
        time => erlang:system_time(second),
        total => erlang:memory(total),
        processes => erlang:memory(processes),
        binary => erlang:memory(binary),
        ets => erlang:memory(ets)
    },
    timer:sleep(Interval * 1000),
    [Sample | collect_samples(Total - Interval, Interval)].

analyze_trend(Samples) ->
    First = hd(Samples),
    Last = lists:last(Samples),
    Growth = maps:get(total, Last) - maps:get(total, First),
    #{
        duration_seconds => maps:get(time, Last) - maps:get(time, First),
        total_growth => Growth,
        processes_growth => maps:get(processes, Last) - maps:get(processes, First),
        binary_growth => maps:get(binary, Last) - maps:get(binary, First),
        ets_growth => maps:get(ets, Last) - maps:get(ets, First)
    }.

%% Step 2: Find what's growing
identify_leak_source() ->
    %% Check ETS tables
    ets_sizes = ets_table_sizes(),
    %% Check top processes by memory
    process_hogs = find_memory_hogs(20),
    %% Check binary memory
    binary_holders = binary_memory_usage(),
    %% Check message queues
    queue_lengths = process_queue_lengths(20).

ets_table_sizes() ->
    [begin
        Name = ets:info(T, name),
        Size = ets:info(T, memory) * erlang:system_info(wordsize),
        #{table => Name, size_bytes => Size, objects => ets:info(T, size)}
    end || T <- ets:all()].

process_queue_lengths(Top) ->
    Ps = processes(),
    Qs = [{P, Q} ||
        P <- Ps,
        {message_queue_len, Q} <- [process_info(P, message_queue_len)],
        Q > 0
    ],
    lists:sublist(lists:reverse(lists:keysort(2, Qs)), Top).

%% Step 3: Deep dive into specific process
investigate_process(Pid) ->
    #{
        name => element(2, process_info(Pid, registered_name)),
        memory => element(2, process_info(Pid, memory)),
        heap_size => element(2, process_info(Pid, heap_size)),
        total_heap_size => element(2, process_info(Pid, total_heap_size)),
        stack_size => element(2, process_info(Pid, stack_size)),
        message_queue_len => element(2, process_info(Pid, message_queue_len)),
        current_function => element(2, process_info(Pid, current_function)),
        initial_call => element(2, process_info(Pid, initial_call)),
        binary_count => length(element(2, process_info(Pid, binary)))
    }.

%% Step 4: Find the leak in code
%% Common causes:
%% 1. Growing ETS table (forgotten to delete)
%% 2. Growing message queue (slow consumer)
%% 3. Binary references held too long
%% 4. Timer leaks (erlang:send_after without cancel)
%% 5. Monitor leaks (demonitor never called)

find_timer_leaks() ->
    %% Count pending timers
    TimerInfo = erlang:system_info(timer_count),
    logger:info("Pending timers", #{count => TimerInfo}).

find_monitor_leaks() ->
    Ps = processes(),
    MonitorCounts = [{P, MC} ||
        P <- Ps,
        {monitors, Ms} <- [process_info(P, monitors)],
        MC <- [length(Ms)],
        MC > 100  %% suspicious if > 100 monitors
    ],
    MonitorCounts.
```

---

## สรุป Part 52

| Memory Type | GC | Sharing |
|-------------|----|---------| 
| Heap binary (≤64B) | Process GC | Copied on send |
| Ref binary (>64B) | Ref count | O(1) pass |
| ETS | Manual | Accessible to all |
| Atom | Never | Global |
| Code | Never GC'd | Global |

**Top causes of memory issues**: ETS without TTL, large binary sub-references, growing message queues, monitor/timer leaks.

---

*[← Part 51: High Availability](part_51_high_availability.md) | [Part 53: Concurrency Patterns →](part_53_concurrency_patterns.md)*
