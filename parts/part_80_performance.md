# Part 80: Performance Optimization at Scale

## สารบัญ
1. [BEAM VM Tuning](#beam-tuning)
2. [Process Architecture for Throughput](#process-arch)
3. [ETS Optimization](#ets-optimization)
4. [Network and I/O Tuning](#network-tuning)
5. [Memory Optimization](#memory)
6. [ตัวอย่างจริง: Optimizing to 1M+ requests/sec](#benchmark)

---

## BEAM VM Tuning

```erlang
%% vm.args — BEAM VM parameters for production

%% +P 5000000        — max processes (default 262144)
%% +Q 65536          — max ports (file handles, sockets)
%% +K true           — kernel poll (epoll/kqueue — much better I/O)
%% +A 64             — async thread pool (for file I/O)
%% +S 8:8            — 8 schedulers, 8 online (match CPU cores)
%% +SDio 4           — 4 dirty I/O schedulers
%% +SDcpu 2          — 2 dirty CPU schedulers

%% Memory allocator tuning:
%% +MBas aobf        — best-fit allocator strategy
%% +MHas aobf
%% +Mlambda 0.75     — carrier fragmentation threshold

%% Scheduler spread work across all cores:
%% +sbt db           — scheduler bind type: db (depends on topology)
%% +sub true         — scheduler utilization balancer

%% GC tuning:
%% +hms 100000       — minimum heap size (100K words)
%% +hmbs 46422       — minimum binary virtual heap

%% Code snippet to read and apply at runtime:
tune_vm_at_runtime() ->
    %% Increase process limit (can be done at boot only via +P)
    
    %% Enable kernel poll for better I/O
    erlang:system_flag(scheduler_wall_time, true),
    
    %% Adjust scheduler work stealing
    erlang:system_flag(multi_scheduling, enabled),
    
    %% Monitor schedulers
    SchedulerStats = erlang:statistics(scheduler_wall_time),
    SchedulerUtils = [{S, A/(A+I)} || {S, A, I} <- SchedulerStats, A+I > 0],
    logger:info("Scheduler utilization", #{stats => SchedulerUtils}).
```

---

## Process Architecture for Throughput

```erlang
%% High-throughput architectures

%% 1. Sharded gen_servers (avoid single bottleneck)
-module(sharded_counter).
-export([increment/1, get/1]).

-define(SHARDS, 32).

increment(Key) ->
    ShardId = shard_for(Key),
    gen_server:cast({via, gproc, {n, l, {counter_shard, ShardId}}},
                    {increment, Key}).

get(Key) ->
    ShardId = shard_for(Key),
    gen_server:call({via, gproc, {n, l, {counter_shard, ShardId}}},
                    {get, Key}).

shard_for(Key) ->
    erlang:phash2(Key, ?SHARDS).

%% Start all shards under supervisor
start_shards() ->
    [supervisor:start_child(counter_sup, [N]) || N <- lists:seq(0, ?SHARDS - 1)].

%% 2. Worker pool with work stealing
-module(worker_pool).
-export([submit/1, start_link/1]).

-record(pool, {
    workers = [],
    queue = queue:new(),
    waiting = []
}).

submit(Work) ->
    gen_server:call(?MODULE, {submit, Work}).

handle_call({submit, Work}, From, #pool{workers = [], queue = Q} = State) ->
    %% No free workers, queue the work
    NewQ = queue:in({From, Work}, Q),
    {noreply, State#pool{queue = NewQ}};

handle_call({submit, Work}, _From, #pool{workers = [W | Rest]} = State) ->
    %% Send to free worker
    gen_server:cast(W, {execute, Work, self()}),
    {reply, ok, State#pool{workers = Rest}}.

handle_cast({worker_done, WorkerPid, Result}, #pool{queue = Q, workers = Workers} = State) ->
    case queue:out(Q) of
        {{value, {From, Work}}, NewQ} ->
            gen_server:cast(WorkerPid, {execute, Work, self()}),
            gen_server:reply(From, Result),
            {noreply, State#pool{queue = NewQ}};
        {empty, _} ->
            {noreply, State#pool{workers = [WorkerPid | Workers]}}
    end.

%% 3. Actor pattern — avoid gen_server overhead for simple processing
fast_actor(Fun) ->
    spawn_opt(fun() -> fast_loop(Fun) end, [
        {message_queue_data, off_heap},  %% move mailbox off heap
        {priority, high}
    ]).

fast_loop(Fun) ->
    receive
        {From, Request} ->
            Response = Fun(Request),
            From ! {response, Response};
        stop -> ok
    end,
    fast_loop(Fun).

%% 4. Selective receive pattern for priority
priority_receive() ->
    receive
        {priority, high, Msg} ->
            process_high(Msg)
    after 0 ->
        receive
            {priority, normal, Msg} ->
                process_normal(Msg)
        after 100 ->
            idle
        end
    end.
```

---

## ETS Optimization

```erlang
%% ETS performance patterns

%% 1. Optimal table configuration
create_hot_read_table() ->
    ets:new(hot_cache, [
        named_table,
        public,
        set,
        {read_concurrency, true},   %% Multiple readers, no lock
        {write_concurrency, auto},  %% Fine-grained locking for writes
        {decentralized_counters, true}  %% Better counter performance
    ]).

%% 2. Atomic counters without process bottleneck
-define(COUNTER_TABLE, counters).

init_counters() ->
    ets:new(?COUNTER_TABLE, [named_table, public, {write_concurrency, true}]).

increment_counter(Name) ->
    ets:update_counter(?COUNTER_TABLE, Name, 1, {Name, 0}).

get_counter(Name) ->
    case ets:lookup(?COUNTER_TABLE, Name) of
        [{Name, Count}] -> Count;
        [] -> 0
    end.

%% 3. Memory-efficient storage
store_compressed(Key, Data) ->
    Compressed = zlib:compress(term_to_binary(Data)),
    ets:insert(cache_table, {Key, Compressed}).

load_compressed(Key) ->
    case ets:lookup(cache_table, Key) of
        [{Key, Compressed}] ->
            {ok, binary_to_term(zlib:uncompress(Compressed))};
        [] ->
            not_found
    end.

%% 4. Batch reads for efficiency
batch_lookup(Keys) ->
    lists:filtermap(fun(Key) ->
        case ets:lookup(my_table, Key) of
            [{Key, Value}] -> {true, {Key, Value}};
            [] -> false
        end
    end, Keys).

%% 5. Efficient select with match spec (avoid full scan)
find_users_in_country(Country) ->
    %% Uses index if country is a known field position
    MatchSpec = [{
        {user, '$1', Country, '$3'},  %% {user, Id, Country, Name}
        [],                            %% no guards
        ['$_']                         %% return whole record
    }],
    ets:select(users, MatchSpec).

%% 6. Sharded ETS for write scalability
-define(ETS_SHARDS, 16).

ets_shard(Key) ->
    erlang:phash2(Key, ?ETS_SHARDS).

sharded_insert(Key, Value) ->
    Shard = ets_shard(Key),
    TableName = list_to_atom("shard_" ++ integer_to_list(Shard)),
    ets:insert(TableName, {Key, Value}).

sharded_lookup(Key) ->
    Shard = ets_shard(Key),
    TableName = list_to_atom("shard_" ++ integer_to_list(Shard)),
    ets:lookup(TableName, Key).
```

---

## Network and I/O Tuning

```erlang
%% Network performance

%% 1. TCP options for high performance
tcp_opts() ->
    [
        binary,
        {active, false},           %% manual receive control
        {reuseaddr, true},
        {nodelay, true},           %% disable Nagle (low latency)
        {sndbuf, 65536},           %% 64KB send buffer
        {recbuf, 65536},           %% 64KB receive buffer
        {send_timeout, 5000},
        {send_timeout_close, true},
        {keepalive, true},
        {linger, {true, 0}}        %% fast close
    ].

%% 2. HTTP connection keep-alive pooling
http_pool_config() ->
    hackney_pool:start_pool(my_pool, [
        {timeout, 5000},
        {max_connections, 200},
        {max_idle_time, 30000}
    ]).

http_request_with_pool(Url, Body) ->
    hackney:post(Url, [], Body, [
        {pool, my_pool},
        {follow_redirect, true},
        {recv_timeout, 5000}
    ]).

%% 3. Async I/O with {active, once}
-module(tcp_server).

handle_socket(Socket) ->
    inet:setopts(Socket, [{active, once}]),
    receive_loop(Socket, <<>>).

receive_loop(Socket, Buffer) ->
    receive
        {tcp, Socket, Data} ->
            NewBuffer = <<Buffer/binary, Data/binary>>,
            {Messages, Rest} = parse_frames(NewBuffer),
            [process_message(M) || M <- Messages],
            inet:setopts(Socket, [{active, once}]),  %% re-arm
            receive_loop(Socket, Rest);
        {tcp_closed, Socket} ->
            ok;
        {tcp_error, Socket, Reason} ->
            logger:error("TCP error", #{reason => Reason})
    after 30000 ->
        gen_tcp:send(Socket, ping_frame()),
        receive_loop(Socket, Buffer)
    end.

%% 4. Batch network writes
batch_sends(Socket, Messages) ->
    %% Accumulate into single send instead of many small sends
    IoData = [encode_frame(M) || M <- Messages],
    gen_tcp:send(Socket, IoData).  %% single syscall
```

---

## Memory Optimization

```erlang
%% Memory optimization patterns

%% 1. Binary sharing (avoid copying)
process_large_binary(LargeBin) ->
    %% Sub-binary shares memory with original
    <<Header:100/binary, Rest/binary>> = LargeBin,
    
    %% Pass sub-binaries by reference (no copy until process dies)
    spawn(fun() -> process_header(Header) end),
    spawn(fun() -> process_rest(Rest) end).

%% Force binary to be independent (stop sharing)
copy_binary(Bin) ->
    binary:copy(Bin).

%% 2. Avoid atom explosion (atoms are never GC'd)
safe_atom(String) when is_list(String) ->
    %% NEVER: list_to_atom(UserInput) — can exhaust atom table!
    %% ALWAYS: check first or use binary
    try binary_to_existing_atom(list_to_binary(String), utf8)
    catch error:badarg -> {error, unknown_atom}
    end.

%% 3. ETS for large shared state (not in process heap)
cache_in_ets(Key, Value) ->
    %% ETS memory is outside process heaps
    %% Process just holds a reference
    ets:insert(shared_cache, {Key, Value}).

%% 4. Hibernate processes to free memory
hibernate_idle_connection(State) ->
    %% GC + compact process memory
    %% Use when process will be idle for a long time
    {noreply, State, hibernate}.  %% gen_server reply

%% 5. Message queue pressure relief
handle_mailbox_pressure() ->
    {message_queue_len, Len} = process_info(self(), message_queue_len),
    case Len > 10000 of
        true ->
            logger:warning("Message queue pressure", #{len => Len}),
            %% Drop old messages
            flush_old_messages(Len - 1000);
        false ->
            ok
    end.

flush_old_messages(0) -> ok;
flush_old_messages(N) ->
    receive _ -> flush_old_messages(N - 1)
    after 0 -> ok
    end.

%% 6. Process spawn_opt for memory efficiency
spawn_lean(Fun) ->
    spawn_opt(Fun, [
        {min_heap_size, 100},      %% small initial heap
        {min_bin_vheap_size, 46},  %% small binary heap
        {fullsweep_after, 10}      %% aggressive GC
    ]).
```

---

## ตัวอย่างจริง: Optimizing to 1M+ req/sec

```erlang
%% Benchmark results and optimizations at each stage

%% Stage 1: Baseline (simple gen_server handler)
%% ~50K req/sec, P99 = 5ms

%% Stage 2: Remove gen_server overhead for hot path
%% Direct process spawn per request
%% ~200K req/sec, P99 = 3ms

-module(fast_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    %% No gen_server call overhead — direct processing
    Method = cowboy_req:method(Req),
    Path = cowboy_req:path(Req),
    
    Response = handle(Method, Path, Req),
    
    {ok, Response, State}.

handle(<<"GET">>, <<"/api/v1/counter/", Key/binary>>, Req) ->
    %% Direct ETS read — no message passing
    Count = case ets:lookup(counters, Key) of
        [{Key, N}] -> N;
        [] -> 0
    end,
    cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        integer_to_binary(Count), Req);

handle(<<"POST">>, <<"/api/v1/counter/", Key/binary>>, Req) ->
    %% Atomic ETS increment — no lock contention
    Count = ets:update_counter(counters, Key, 1, {Key, 0}),
    cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        integer_to_binary(Count), Req).

%% Stage 3: Cowboy tuning
cowboy_start() ->
    cowboy:start_clear(http, [
        {port, 8080},
        {num_acceptors, erlang:system_info(schedulers_online) * 2},
        {max_connections, infinity}
    ], #{
        env => #{dispatch => routes()},
        request_timeout => 30000,
        idle_timeout => 60000
    }).

%% Stage 4: OS-level tuning
%% /etc/sysctl.conf:
%%   net.core.somaxconn = 65535
%%   net.ipv4.tcp_tw_reuse = 1
%%   net.ipv4.ip_local_port_range = 1024 65535
%%   net.core.netdev_max_backlog = 65536
%%   fs.file-max = 2000000

%% /etc/security/limits.conf:
%%   * soft nofile 65536
%%   * hard nofile 65536

%% vm.args:
%%   +P 5000000  +Q 65536  +K true  +S 16:16

%% Stage 5: Benchmark results
benchmark_results() ->
    #{
        baseline => #{rps => 50000, p99_ms => 5, memory_mb => 200},
        ets_direct => #{rps => 500000, p99_ms => 1, memory_mb => 150},
        with_os_tuning => #{rps => 1200000, p99_ms => 0.5, memory_mb => 180},
        notes => "Single 32-core machine, simple counter workload"
    }.

%% For complex workloads (DB queries, JSON parsing):
realistic_benchmark() ->
    #{
        baseline_with_postgres => #{rps => 10000, p99_ms => 20},
        with_connection_pool => #{rps => 50000, p99_ms => 5},
        with_caching => #{rps => 200000, p99_ms => 2},
        notes => "50% cache hit rate, simple SELECT queries"
    }.
```

---

## สรุป Part 80

| Optimization | Impact | Complexity |
|-------------|--------|-----------|
| ETS with read_concurrency | 3-5x read throughput | Low |
| Sharded gen_servers | 10-32x write throughput | Medium |
| Kernel poll (+K true) | 2-3x network throughput | Low |
| Binary {active, once} | Lower memory usage | Medium |
| Process hibernation | 50-90% memory on idle | Low |
| PBKDF2 → Argon2 | Better security (not speed) | Medium |
| OS TCP tuning | 2-3x connection capacity | Low |

**Rule of thumb**: profile first, optimize second — อย่า optimize ก่อนวัด

---

*[← Part 79: Security](part_79_security.md) | [Part 81: Event Sourcing →](part_81_event_sourcing.md)*
