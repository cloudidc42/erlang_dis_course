# Part 93: BEAM Internals

## สารบัญ
1. [BEAM Process Model](#process-model)
2. [Garbage Collection](#gc)
3. [Scheduler Internals](#scheduler)
4. [Memory Allocator](#allocator)
5. [ETS Internals](#ets-internals)
6. [ตัวอย่างจริง: Tuning for Your Workload](#tuning)

---

## BEAM Process Model

```
BEAM Process Memory Layout:
┌─────────────────────────────────────────────┐
│ Process Control Block (PCB)                  │
│   - PID, status, priority                    │
│   - Message queue (mailbox)                   │
│   - Links and monitors                        │
│   - Trap exit flag                           │
│   - Dictionary (process dict)                │
├─────────────────────────────────────────────┤
│ Stack                                        │
│   - Function call frames                     │
│   - Local variables                          │
│   - Return addresses                         │
├─────────────────────────────────────────────┤
│ Heap                                         │
│   - Terms allocated by this process          │
│   - Grows upward                             │
├─────────────────────────────────────────────┤
│ Old Heap (generational GC)                   │
│   - Long-lived terms moved here              │
│   - GC'd less frequently                     │
└─────────────────────────────────────────────┘

Key facts:
- Default heap: 233 words (233 * 8 = 1864 bytes on 64-bit)
- Stack and heap share memory, grow toward each other
- GC happens per-process, independently
- No shared mutable state between processes (except ETS/Mnesia)
```

```erlang
%% Observe process memory
observe_process(Pid) ->
    Info = process_info(Pid, [
        heap_size,          %% words in young heap
        old_heap_size,      %% words in old heap
        stack_size,         %% words in stack
        total_heap_size,    %% total heap words
        memory,             %% total bytes
        message_queue_len,  %% messages waiting
        reductions,         %% work done (scheduler units)
        garbage_collection  %% GC info
    ]),
    
    io:format("Process ~p memory:~n", [Pid]),
    [{K, V} || {K, V} <- Info, io:format("  ~p: ~p~n", [K, V])].

%% Process spawn cost
spawn_timing() ->
    N = 100000,
    T1 = erlang:monotonic_time(microsecond),
    Pids = [spawn(fun() -> ok end) || _ <- lists:seq(1, N)],
    T2 = erlang:monotonic_time(microsecond),
    
    %% Wait for all to finish
    [receive {'DOWN', _, process, P, _} -> ok end 
     || P <- [begin monitor(process, Pid), Pid end || Pid <- Pids]],
    
    io:format("~p spawns in ~p μs (~p μs/spawn)~n",
              [N, T2 - T1, (T2 - T1) div N]).
    %% Typical: ~1-2 μs per spawn
```

---

## Garbage Collection

```erlang
%% Understanding GC strategy

%% Erlang uses per-process generational GC:
%% 1. Minor GC: collect young generation (frequent, fast)
%% 2. Major GC: collect old generation (infrequent, slow)
%% 3. Full sweep: reclaim fragmented memory (rare)

%% GC is triggered when:
%% - Heap runs out of space
%% - Erlang allocates more than fullsweep_after minors
%% - Manual: erlang:garbage_collect(Pid)

%% Monitor GC pressure
setup_gc_monitoring() ->
    %% Enable GC tracing (heavy, use in development only)
    erlang:trace(all, true, [garbage_collection]),
    
    %% Receive GC events
    receive_gc_events().

receive_gc_events() ->
    receive
        {trace, Pid, gc_minor_start, Info} ->
            io:format("Minor GC start: ~p, heap: ~p words~n",
                      [Pid, proplists:get_value(heap_size, Info)]);
        {trace, Pid, gc_minor_end, Info} ->
            io:format("Minor GC end: ~p, reclaimed: ~p words~n",
                      [Pid, proplists:get_value(heap_size, Info)]);
        {trace, Pid, gc_major_start, _Info} ->
            io:format("Major GC start: ~p~n", [Pid]);
        _ -> ok
    end,
    receive_gc_events().

%% Tune GC per process
start_long_lived_server() ->
    spawn_opt(fun server_loop/0, [
        %% Start with large heap to reduce early GCs
        {min_heap_size, 10000},
        
        %% Large binary heap (for processes handling large binaries)
        {min_bin_vheap_size, 46422},
        
        %% Full GC sweep after N minor GCs (0 = always full)
        {fullsweep_after, 20},
        
        %% Move to old gen after surviving N minor GCs
        {max_gen_gcs, 0}   %% 0 = disable generational
    ]).

server_loop() ->
    receive
        _ -> server_loop()
    end.

%% Binary memory: ref-counted, shared across processes
binary_memory_facts() ->
    %% Large binaries (> 64 bytes) are NOT on process heap
    %% They live in a shared "binary heap"
    %% Process heap holds a reference
    
    LargeBin = crypto:strong_rand_bytes(1000),
    
    %% Sub-binaries share the large binary's memory
    <<Header:100/binary, Rest/binary>> = LargeBin,
    
    %% Both Header and Rest reference LargeBin (no copy!)
    %% LargeBin is freed when no more references exist (ref-count)
    
    %% Force independent copy:
    HeaderCopy = binary:copy(Header),
    RestCopy = binary:copy(Rest),
    
    {HeaderCopy, RestCopy}.
```

---

## Scheduler Internals

```erlang
%% BEAM scheduler architecture

%% Each scheduler thread handles a run queue of processes
%% Processes get a "reduction budget" per scheduling slot
%% Default: 2000 reductions per time slice (~1ms)

%% What counts as a reduction:
%% - Function calls: 1 reduction each
%% - Message sends: 1 reduction
%% - BIFs (built-in functions): varies
%% - Port operations: varies

%% Scheduler work stealing: idle schedulers steal from busy ones

scheduler_stats() ->
    %% Enable wall time tracking
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(1000),
    
    Stats = erlang:statistics(scheduler_wall_time),
    
    [begin
        Util = if A + I > 0 -> A * 100.0 / (A + I); true -> 0 end,
        io:format("Scheduler ~p: ~.1f% utilization~n", [S, Util])
    end || {S, A, I} <- Stats].

%% Run queue depth
run_queue_stats() ->
    %% Total across all schedulers
    Total = statistics(run_queue),
    
    %% Per scheduler (if supported)
    PerSched = [statistics({run_queue, S}) 
                || S <- lists:seq(1, erlang:system_info(schedulers_online))],
    
    io:format("Total run queue: ~p~n", [Total]),
    io:format("Per scheduler: ~p~n", [PerSched]).

%% Process priority levels
priority_examples() ->
    %% Low priority: background work, batch processing
    spawn_opt(fun() -> background_work() end, [{priority, low}]),
    
    %% Normal: default
    spawn(fun() -> normal_work() end),
    
    %% High: latency-sensitive work (use sparingly)
    spawn_opt(fun() -> latency_sensitive() end, [{priority, high}]),
    
    %% Max: system processes, use only for critical VM operations
    %% spawn_opt(fun() -> ... end, [{priority, max}]).
    ok.

background_work() -> timer:sleep(100), background_work().
normal_work() -> timer:sleep(10), normal_work().
latency_sensitive() -> timer:sleep(1), latency_sensitive().
```

---

## Memory Allocator

```erlang
%% BEAM memory allocator internals

%% Memory regions:
%% - Process heap: per-process, managed by GC
%% - Binary heap: large binaries, ref-counted
%% - ETS: outside process heaps, fixed addresses
%% - Atom table: never freed, global
%% - Code space: loaded modules

%% Allocator strategies (set in vm.args +M flags):
allocator_configs() ->
    %% Binary large object (BLO) allocator for large binaries
    %% +MBas aobf  — age-ordered best fit
    %% +MBas aobff — age-ordered best fit (fragmented)
    
    %% Literal area allocator
    %% +MLas aobf
    
    %% Heap allocator  
    %% +MHas aobf
    
    %% Enable carrier migration between allocators
    %% +Mcmm 10
    ok.

%% Monitor memory usage
memory_stats() ->
    Memory = erlang:memory(),
    
    #{
        total_mb => maps:get(total, Memory) div (1024*1024),
        processes_mb => maps:get(processes, Memory) div (1024*1024),
        processes_used_mb => maps:get(processes_used, Memory) div (1024*1024),
        ets_mb => maps:get(ets, Memory) div (1024*1024),
        binary_mb => maps:get(binary, Memory) div (1024*1024),
        code_mb => maps:get(code, Memory) div (1024*1024),
        atom_mb => maps:get(atom, Memory) div (1024*1024),
        system_mb => maps:get(system, Memory) div (1024*1024)
    }.

%% Detect memory leaks
detect_leaks() ->
    Before = erlang:memory(total),
    
    %% Do work...
    
    erlang:garbage_collect(),
    timer:sleep(1000),
    
    After = erlang:memory(total),
    Diff = After - Before,
    
    case Diff > 1024 * 1024 of  %% 1MB threshold
        true ->
            logger:warning("Possible memory leak detected", #{diff_bytes => Diff});
        false ->
            ok
    end.
```

---

## ตัวอย่างจริง: Tuning for Your Workload

```erlang
%% Workload-specific tuning guide

%% Profile 1: High-throughput, short-lived requests
%% - Many short-lived processes
%% - Minimal shared state
high_throughput_config() ->
    %% vm.args:
    %% +P 10000000  — allow 10M processes
    %% +Q 65536     — max ports
    %% +K true      — kernel poll
    %% +S 0:0       — auto-detect schedulers
    %% Process heap: small initial size (fast allocation)
    %% +hms 233     — default (keep small)
    
    %% Application: use direct message passing over gen_server:call
    ok.

%% Profile 2: Low-latency, real-time system
low_latency_config() ->
    %% vm.args:
    %% +S 4:4  — pin schedulers to avoid migration
    %% +sbt db — bind schedulers to topology (NUMA-aware)
    %% +sub true — scheduler utilization balancer
    
    %% OS: CPU pinning with taskset
    %% taskset -c 0-3 erl ...
    
    %% Application: use {priority, high} for latency-sensitive
    %% Avoid large message sends (copy cost)
    ok.

%% Profile 3: Memory-constrained environment
memory_constrained_config() ->
    %% vm.args:
    %% +P 1000000   — reduce from default 5M
    %% +hms 100     — tiny initial heap
    %% +hmbs 32     — tiny binary heap
    
    %% Application:
    %% - Hibernate idle processes: {noreply, State, hibernate}
    %% - Use binaries (shared, not copied) over lists for large data
    %% - Monitor ETS table sizes
    ok.

%% Runtime self-tuning
self_tune() ->
    %% Measure current state
    RunQueue = statistics(run_queue),
    ProcessCount = erlang:system_info(process_count),
    MemMb = erlang:memory(total) div (1024*1024),
    
    %% Tune based on state
    if
        RunQueue > 100 ->
            logger:warning("Run queue high, adding schedulers");
        ProcessCount > erlang:system_info(process_limit) * 0.8 ->
            logger:warning("Approaching process limit");
        MemMb > 3000 ->
            logger:warning("Memory high, triggering GC"),
            [erlang:garbage_collect(P) || P <- erlang:processes()];
        true ->
            ok
    end.
```

---

## สรุป Part 93

| Component | Key Facts |
|-----------|-----------|
| Process | ~2KB default heap, independent GC, ~1μs spawn cost |
| GC | Generational, per-process, stop-the-process (not world) |
| Schedulers | 1 per CPU core, work-stealing, 2000 reductions/slice |
| Binaries | Large (>64B) in shared heap, ref-counted |
| ETS | Outside process heaps, O(1) access, lock-free reads |
| Atoms | Never GC'd, 1048576 limit, use binary_to_existing_atom |

**Key insight**: BEAM's design trades per-process overhead for system-wide predictability — millions of lightweight processes with isolated GC beats a single large heap with global GC pauses.

---

*[← Part 92: Advanced OTP](part_92_advanced_otp.md) | [Part 94: Final Projects →](part_94_final_project_chat.md)*
