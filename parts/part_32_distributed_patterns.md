# Part 32: Distributed Patterns

## สารบัญ
1. [Leader Election](#leader-election)
2. [Consistent Hashing](#consistent-hashing)
3. [Gossip Protocol](#gossip-protocol)
4. [Distributed Task Queue](#distributed-task-queue)
5. [ตัวอย่างจริง: Distributed Cache Cluster](#ตัวอย่างจริง-distributed-cache-cluster)

---

## Leader Election

```erlang
%% Leader Election: เลือก "หัวหน้า" node ใน cluster
%% ใช้เมื่อต้องการ single coordinator

-module(leader_election).
-behaviour(gen_server).

-export([start_link/0, get_leader/0, am_leader/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    leader = undefined :: node() | undefined,
    election_ref = undefined :: reference() | undefined
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_leader() ->
    gen_server:call(?MODULE, get_leader).

am_leader() ->
    gen_server:call(?MODULE, am_leader).

init([]) ->
    net_kernel:monitor_nodes(true),
    %% Start election after brief delay (allow cluster to form)
    erlang:send_after(500, self(), start_election),
    {ok, #state{}}.

handle_call(get_leader, _From, #state{leader = L} = State) ->
    {reply, L, State};
handle_call(am_leader, _From, #state{leader = L} = State) ->
    {reply, L =:= node(), State}.

handle_cast({election_result, Leader}, State) ->
    io:format("[~p] New leader: ~p~n", [node(), Leader]),
    {noreply, State#state{leader = Leader}};
handle_cast(_, State) -> {noreply, State}.

handle_info(start_election, State) ->
    Leader = elect_leader(),
    broadcast_leader(Leader),
    {noreply, State#state{leader = Leader}};

handle_info({nodeup, _Node}, State) ->
    erlang:send_after(200, self(), start_election),
    {noreply, State};

handle_info({nodedown, Node}, #state{leader = Node} = State) ->
    %% Leader died, start new election
    erlang:send_after(200, self(), start_election),
    {noreply, State#state{leader = undefined}};

handle_info({nodedown, _Node}, State) ->
    {noreply, State};

handle_info(_, State) -> {noreply, State}.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

%% Simple strategy: alphabetically smallest node name becomes leader
elect_leader() ->
    AllNodes = lists:sort([node() | nodes()]),
    hd(AllNodes).

broadcast_leader(Leader) ->
    [gen_server:cast({?MODULE, N}, {election_result, Leader})
     || N <- nodes()],
    gen_server:cast(?MODULE, {election_result, Leader}).
```

---

## Consistent Hashing

```erlang
%% Consistent Hashing: map keys to nodes ด้วย minimal redistribution
%% เมื่อเพิ่ม/ลบ node เพียง fraction ของ keys ย้าย

-module(consistent_hash).

-export([new/1, add_node/2, remove_node/2, get_node/2, get_nodes/3]).

-define(VIRTUAL_NODES, 150).  %% virtual nodes per real node

%% Ring = sorted list of {HashValue, Node}
new(Nodes) ->
    Ring = lists:flatten([virtual_nodes(N) || N <- Nodes]),
    lists:sort(Ring).

add_node(Node, Ring) ->
    NewVnodes = virtual_nodes(Node),
    lists:sort(Ring ++ NewVnodes).

remove_node(Node, Ring) ->
    [Entry || {_, N} = Entry <- Ring, N =/= Node].

get_node(Key, Ring) ->
    Hash = hash(Key),
    find_node(Hash, Ring).

%% Get N distinct nodes for replication
get_nodes(Key, N, Ring) ->
    Hash = hash(Key),
    Sorted = reorder_from(Hash, Ring),
    Unique = unique_nodes(Sorted, N, []),
    Unique.

%% Internal

virtual_nodes(Node) ->
    [{hash({Node, I}), Node} || I <- lists:seq(1, ?VIRTUAL_NODES)].

hash(Key) ->
    <<H:128>> = crypto:hash(md5, term_to_binary(Key)),
    H.

find_node(Hash, Ring) ->
    case lists:dropwhile(fun({H, _}) -> H < Hash end, Ring) of
        [] -> element(2, hd(Ring));  %% wrap around
        [{_, Node} | _] -> Node
    end.

reorder_from(Hash, Ring) ->
    {Before, After} = lists:splitwith(fun({H, _}) -> H < Hash end, Ring),
    After ++ Before.

unique_nodes(_, 0, Acc) -> lists:reverse(Acc);
unique_nodes([], _, Acc) -> lists:reverse(Acc);
unique_nodes([{_, Node} | Rest], N, Acc) ->
    case lists:member(Node, Acc) of
        true -> unique_nodes(Rest, N, Acc);
        false -> unique_nodes(Rest, N - 1, [Node | Acc])
    end.

%% ใช้งาน
Ring = consistent_hash:new([node1, node2, node3]).
Node = consistent_hash:get_node(my_key, Ring).
%% {node2@host}

Replicas = consistent_hash:get_nodes(my_key, 2, Ring).
%% [node2@host, node3@host]

%% เพิ่ม node ใหม่ (เพียง ~33% keys ย้าย)
NewRing = consistent_hash:add_node(node4, Ring).
```

---

## Gossip Protocol

```erlang
%% Gossip: propagate information through random peer communication
%% ทุก node ส่ง state ไปยัง random peers เป็นระยะ
%% ในที่สุดทุก node จะมีข้อมูลตรงกัน (eventual consistency)

-module(gossip_server).
-behaviour(gen_server).

-export([start_link/0, broadcast/2, get_state/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    data = #{} :: map(),
    version = 0 :: non_neg_integer()
}).

-define(GOSSIP_INTERVAL, 1000).  %% ms
-define(FANOUT, 3).               %% nodes to gossip to each round

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

broadcast(Key, Value) ->
    gen_server:cast(?MODULE, {local_update, Key, Value}).

get_state() ->
    gen_server:call(?MODULE, get_state).

init([]) ->
    erlang:send_after(?GOSSIP_INTERVAL, self(), gossip_tick),
    {ok, #state{}}.

handle_call(get_state, _From, #state{data = D} = State) ->
    {reply, D, State}.

handle_cast({local_update, K, V}, #state{data = D, version = Ver} = State) ->
    NewData = D#{K => #{value => V, version => Ver + 1, origin => node()}},
    {noreply, State#state{data = NewData, version => Ver + 1}};

handle_cast({gossip, RemoteData}, #state{data = LocalData} = State) ->
    %% Merge: keep highest version
    Merged = maps:fold(
        fun(K, #{version := RVer} = REntry, Acc) ->
            case maps:get(K, Acc, undefined) of
                #{version := LVer} when LVer >= RVer -> Acc;
                _ -> Acc#{K => REntry}
            end
        end,
        LocalData,
        RemoteData
    ),
    {noreply, State#state{data = Merged}};

handle_cast(_, State) -> {noreply, State}.

handle_info(gossip_tick, #state{data = Data} = State) ->
    Peers = select_random_peers(?FANOUT),
    lists:foreach(
        fun(Peer) ->
            gen_server:cast({?MODULE, Peer}, {gossip, Data})
        end,
        Peers
    ),
    erlang:send_after(?GOSSIP_INTERVAL, self(), gossip_tick),
    {noreply, State};

handle_info(_, State) -> {noreply, State}.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

select_random_peers(N) ->
    Nodes = nodes(),
    case length(Nodes) =< N of
        true -> Nodes;
        false ->
            Shuffled = [X || {_, X} <- lists:sort([{rand:uniform(), N2} || N2 <- Nodes])],
            lists:sublist(Shuffled, N)
    end.
```

---

## Distributed Task Queue

```erlang
%% Task Queue ที่กระจายงานข้าม cluster

-module(dist_task_queue).
-behaviour(gen_server).

-export([start_link/0, submit/1, submit/2, worker_ready/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    pending = queue:new() :: queue:queue(),
    workers = #{} :: #{pid() => node()},
    results = #{} :: #{reference() => term()}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Submit task, returns Ref for tracking
submit(Task) ->
    gen_server:call(?MODULE, {submit, Task}).

submit(Task, Timeout) ->
    Ref = gen_server:call(?MODULE, {submit, Task}),
    receive
        {task_result, Ref, Result} -> {ok, Result}
    after Timeout ->
        {error, timeout}
    end.

worker_ready(WorkerPid) ->
    gen_server:cast(?MODULE, {worker_ready, WorkerPid}).

init([]) ->
    %% Spawn workers on all nodes
    AllNodes = [node() | nodes()],
    [spawn_worker(N) || N <- AllNodes],
    {ok, #state{}}.

spawn_worker(Node) ->
    spawn(Node, fun() -> worker_loop() end).

worker_loop() ->
    dist_task_queue:worker_ready(self()),
    receive
        {task, Ref, Caller, Task} ->
            Result = try Task() catch E:R -> {error, {E, R}} end,
            Caller ! {task_result, Ref, Result},
            worker_loop()
    end.

handle_call({submit, Task}, {CallerPid, _}, State) ->
    Ref = make_ref(),
    NewState = assign_or_queue(Ref, CallerPid, Task, State),
    {reply, Ref, NewState}.

handle_cast({worker_ready, WorkerPid}, #state{pending = Q, workers = W} = State) ->
    case queue:out(Q) of
        {{value, {Ref, Caller, Task}}, NewQ} ->
            %% Assign pending task to ready worker
            WorkerPid ! {task, Ref, Caller, Task},
            {noreply, State#state{pending = NewQ}};
        {empty, _} ->
            %% No tasks, add to idle workers
            {noreply, State#state{workers = W#{WorkerPid => node(WorkerPid)}}}
    end.

handle_info({'DOWN', _, process, Pid, _}, #state{workers = W} = State) ->
    %% Worker died, spawn replacement
    case maps:get(Pid, W, undefined) of
        undefined -> {noreply, State};
        Node ->
            spawn_worker(Node),
            {noreply, State#state{workers = maps:remove(Pid, W)}}
    end;
handle_info(_, State) -> {noreply, State}.

assign_or_queue(Ref, Caller, Task, #state{pending = Q, workers = W} = State) ->
    case maps:keys(W) of
        [] ->
            %% No idle workers, queue
            State#state{pending = queue:in({Ref, Caller, Task}, Q)};
        [Worker | _] ->
            Worker ! {task, Ref, Caller, Task},
            State#state{workers = maps:remove(Worker, W)}
    end.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.
```

---

## ตัวอย่างจริง: Distributed Cache Cluster

```erlang
%% ไฟล์: dist_cache.erl
%% Distributed cache ที่ shard data ข้าม nodes ด้วย consistent hashing

-module(dist_cache).
-behaviour(gen_server).

-export([start_link/0, put/2, put/3, get/1, delete/1,
         cluster_stats/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-define(LOCAL_CACHE, local_cache).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Find responsible node and put
put(Key, Value) ->
    put(Key, Value, infinity).

put(Key, Value, TTL) ->
    Node = get_node_for_key(Key),
    case Node =:= node() of
        true -> local_put(Key, Value, TTL);
        false -> rpc:call(Node, ?MODULE, local_put, [Key, Value, TTL])
    end.

get(Key) ->
    Node = get_node_for_key(Key),
    case Node =:= node() of
        true -> local_get(Key);
        false -> rpc:call(Node, ?MODULE, local_get, [Key])
    end.

delete(Key) ->
    Node = get_node_for_key(Key),
    case Node =:= node() of
        true -> local_delete(Key);
        false -> rpc:call(Node, ?MODULE, local_delete, [Key])
    end.

cluster_stats() ->
    AllNodes = [node() | nodes()],
    {Results, _Bad} = rpc:multicall(AllNodes, ?MODULE, local_stats, []),
    Totals = lists:foldl(
        fun(Stats, Acc) ->
            maps:fold(fun(K, V, A) ->
                maps:update_with(K, fun(Old) -> Old + V end, V, A)
            end, Acc, Stats)
        end,
        #{},
        Results
    ),
    #{per_node => lists:zip(AllNodes, Results), totals => Totals}.

%% Local operations (called via RPC from other nodes)
local_put(Key, Value, TTL) ->
    gen_server:cast(?MODULE, {put, Key, Value, TTL}).

local_get(Key) ->
    gen_server:call(?MODULE, {get, Key}).

local_delete(Key) ->
    gen_server:cast(?MODULE, {delete, Key}).

local_stats() ->
    gen_server:call(?MODULE, stats).

get_node_for_key(Key) ->
    Ring = persistent_term:get(cache_ring, undefined),
    case Ring of
        undefined ->
            %% No ring yet, use local
            node();
        _ ->
            consistent_hash:get_node(Key, Ring)
    end.

init([]) ->
    net_kernel:monitor_nodes(true),
    ets:new(?LOCAL_CACHE, [named_table, public, set]),
    update_ring(),
    erlang:send_after(5000, self(), cleanup),
    {ok, #{hits => 0, misses => 0}}.

handle_call({get, Key}, _From, #{hits := H, misses := M} = State) ->
    Now = erlang:monotonic_time(second),
    case ets:lookup(?LOCAL_CACHE, Key) of
        [{Key, Value, infinity}] ->
            {reply, {ok, Value}, State#{hits => H + 1}};
        [{Key, Value, Exp}] when Exp > Now ->
            {reply, {ok, Value}, State#{hits => H + 1}};
        [{Key, _, _}] ->
            ets:delete(?LOCAL_CACHE, Key),
            {reply, miss, State#{misses => M + 1}};
        [] ->
            {reply, miss, State#{misses => M + 1}}
    end;

handle_call(stats, _From, #{hits := H, misses := M} = State) ->
    {reply, #{hits => H, misses => M, keys => ets:info(?LOCAL_CACHE, size)}, State}.

handle_cast({put, Key, Value, TTL}, State) ->
    Now = erlang:monotonic_time(second),
    Exp = case TTL of infinity -> infinity; T -> Now + T end,
    ets:insert(?LOCAL_CACHE, {Key, Value, Exp}),
    {noreply, State};

handle_cast({delete, Key}, State) ->
    ets:delete(?LOCAL_CACHE, Key),
    {noreply, State}.

handle_info({nodeup, _}, State) ->
    update_ring(),
    {noreply, State};

handle_info({nodedown, _}, State) ->
    update_ring(),
    {noreply, State};

handle_info(cleanup, State) ->
    Now = erlang:monotonic_time(second),
    Expired = ets:select(?LOCAL_CACHE,
        [{ {'$1', '_', '$2'}, [{'<', '$2', Now}, {'/=', '$2', infinity}], ['$1'] }]
    ),
    [ets:delete(?LOCAL_CACHE, K) || K <- Expired],
    erlang:send_after(5000, self(), cleanup),
    {noreply, State};

handle_info(_, State) -> {noreply, State}.

terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.

update_ring() ->
    Nodes = [node() | nodes()],
    Ring = consistent_hash:new(Nodes),
    persistent_term:put(cache_ring, Ring).
```

---

## สรุป Part 32

| Pattern | ใช้เมื่อ |
|---------|---------|
| Leader Election | ต้องการ coordinator เดียว |
| Consistent Hashing | Distribute data/load ข้าม nodes |
| Gossip Protocol | Propagate state, eventual consistency |
| Distributed Task Queue | Fan-out computation ข้าม cluster |

---

*[← Part 31: Distributed Erlang](part_31_distributed.md) | [Part 33: TCP/UDP Networking →](part_33_networking.md)*
