# Part 51: High-Availability Patterns

## สารบัญ
1. [HA คืออะไร](#ha-คืออะไร)
2. [Failover Strategies](#failover-strategies)
3. [Data Replication](#data-replication)
4. [Split-Brain Prevention](#split-brain-prevention)
5. [Chaos Engineering](#chaos-engineering)
6. [ตัวอย่างจริง: 99.99% Availability System](#ตัวอย่างจริง-9999-availability-system)

---

## HA คืออะไร

```
High Availability = ระบบที่ทำงานต่อเนื่องแม้มี failures

Availability tiers:
- 99%    = 87.6 hours downtime/year
- 99.9%  = 8.76 hours downtime/year
- 99.99% = 52.6 minutes downtime/year
- 99.999%= 5.26 minutes downtime/year (five nines)

Erlang/OTP ช่วย HA ด้วย:
- Let it crash + supervision (auto-restart)
- Process isolation (one crash ≠ system crash)
- Hot code reloading (zero-downtime upgrades)
- Distribution (run on multiple machines)
- Mnesia distributed database
```

---

## Failover Strategies

```erlang
%% Primary-Backup failover กับ global process registration

-module(primary_backup).
-behaviour(gen_server).
-export([start_link/0, become_primary/0, get_primary/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link(?MODULE, [], []).

init(_) ->
    %% Try to become primary
    attempt_primary(),
    {ok, #{role => backup, primary => undefined}}.

attempt_primary() ->
    case global:register_name(primary_service, self()) of
        yes ->
            logger:info("Became primary", #{node => node()}),
            global:sync(),
            self() ! became_primary;
        no ->
            PrimaryPid = global:whereis_name(primary_service),
            monitor(process, PrimaryPid),
            logger:info("Running as backup", #{primary => PrimaryPid})
    end.

handle_info(became_primary, State) ->
    %% Start primary responsibilities
    start_primary_tasks(),
    {noreply, State#{role => primary}};

handle_info({'DOWN', _, process, _Pid, Reason}, State) ->
    logger:warning("Primary died, attempting takeover", #{reason => Reason}),
    timer:sleep(rand:uniform(500)),  %% Jitter to prevent thundering herd
    attempt_primary(),
    {noreply, State};

handle_call(get_role, _From, #{role := Role} = State) ->
    {reply, Role, State}.

get_primary() ->
    global:whereis_name(primary_service).

start_primary_tasks() ->
    %% Start jobs that only primary should run
    scheduler:start(),
    health_reporter:start_reporting().

%% Health check with failover
become_primary() ->
    gen_server:call(?MODULE, become_primary).

%% Erlang OTP application takeover
%% normal: initial start
%% {takeover, OtherNode}: takeover from another node
%% {failover, OtherNode}: restart after failure
handle_start_type({takeover, OtherNode}, State) ->
    logger:info("Taking over from", #{node => OtherNode}),
    %% Transfer state from old node
    OldState = rpc:call(OtherNode, my_server, get_state, []),
    migrate_state(OldState, State);
handle_start_type({failover, _}, State) ->
    %% Recover from crash: load last checkpoint
    recover_from_checkpoint(State);
handle_start_type(normal, State) ->
    State.
```

---

## Data Replication

```erlang
%% Synchronized state across nodes

-module(replicated_store).
-behaviour(gen_server).
-export([start_link/0, put/2, get/1, delete/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({global, ?MODULE}, ?MODULE, [], []).

put(Key, Value) ->
    gen_server:call({global, ?MODULE}, {put, Key, Value}).

get(Key) ->
    gen_server:call({global, ?MODULE}, {get, Key}).

init(_) ->
    ets:new(store, [named_table, set, public]),
    net_kernel:monitor_nodes(true),
    {ok, #{version => 0, peers => nodes()}}.

handle_call({put, Key, Value}, _From, #{version := V} = State) ->
    NewV = V + 1,
    ets:insert(store, {Key, Value, NewV}),
    %% Replicate to all nodes asynchronously
    replicate_to_peers({put, Key, Value, NewV}),
    {reply, ok, State#{version := NewV}};

handle_call({get, Key}, _From, State) ->
    Result = case ets:lookup(store, Key) of
        [{_, Value, _}] -> {ok, Value};
        [] -> {error, not_found}
    end,
    {reply, Result, State}.

handle_info({replicate, {put, Key, Value, Version}}, #{version := V} = State) ->
    %% Only apply if newer
    case Version > V of
        true ->
            ets:insert(store, {Key, Value, Version}),
            {noreply, State#{version := Version}};
        false ->
            {noreply, State}
    end;

handle_info({nodeup, Node}, State) ->
    logger:info("Node up, syncing", #{node => Node}),
    sync_to_node(Node),
    {noreply, State#{peers := [Node | maps:get(peers, State, [])]}};

handle_info({nodedown, Node}, State) ->
    {noreply, State#{peers := lists:delete(Node, maps:get(peers, State, []))}}.

replicate_to_peers(Op) ->
    Peers = nodes(),
    [gen_server:cast({?MODULE, N}, {replicate, Op}) || N <- Peers].

sync_to_node(Node) ->
    AllData = ets:tab2list(store),
    rpc:cast(Node, ?MODULE, sync_data, [AllData]).

%% CRDTs: conflict-free replicated data types
%% G-Counter: grow-only, works with concurrent increments
-module(g_counter).

new() ->
    #{node() => 0}.

increment(Counter) ->
    N = maps:get(node(), Counter, 0),
    Counter#{node() => N + 1}.

value(Counter) ->
    lists:sum(maps:values(Counter)).

merge(C1, C2) ->
    maps:fold(fun(Node, V1, Acc) ->
        V2 = maps:get(Node, C2, 0),
        Acc#{Node => max(V1, V2)}
    end, C2, C1).
```

---

## Split-Brain Prevention

```erlang
%% Split-brain: network partition แบ่ง cluster เป็น 2 กลุ่ม
%% แต่ละกลุ่มคิดว่าตัวเองเป็น primary
%% ป้องกันด้วย: quorum, fencing, witness node

-module(quorum_manager).
-export([is_quorum/0, cluster_size/0, my_votes/0]).

%% ต้องมี majority (N/2 + 1) ถึงจะ operate ได้
is_quorum() ->
    TotalNodes = cluster_size(),
    AliveNodes = length([N || N <- nodes(), net_adm:ping(N) =:= pong]) + 1,
    Required = (TotalNodes div 2) + 1,
    AliveNodes >= Required.

cluster_size() ->
    %% Read from config, not dynamic
    application:get_env(my_app, cluster_size, 3).

%% Decision: should this node handle requests?
should_serve_requests() ->
    case is_quorum() of
        true -> true;
        false ->
            logger:warning("Lost quorum, stopping service"),
            false
    end.

%% Fencing token: prevent old primary from causing damage
%% When new primary elected, it gets higher fencing token
%% Old primary's operations are rejected if token too low
-module(fencing).
-export([get_token/0, check_token/1, new_epoch/0]).

get_token() ->
    persistent_term:get(fencing_token, 0).

check_token(Token) ->
    Current = get_token(),
    Token >= Current.

new_epoch() ->
    Current = get_token(),
    New = Current + 1,
    persistent_term:put(fencing_token, New),
    broadcast_new_token(New),
    New.

broadcast_new_token(Token) ->
    [rpc:cast(N, fencing, set_token, [Token]) || N <- nodes()].

%% Witness node: tiebreaker for even-sized clusters
-module(witness_node).
-export([vote/1]).

vote(RequestingNode) ->
    %% Simple: vote for whoever asks first
    case persistent_term:get(voted_for, undefined) of
        undefined ->
            persistent_term:put(voted_for, RequestingNode),
            {ok, voted};
        AlreadyVoted when AlreadyVoted =:= RequestingNode ->
            {ok, already_voted};
        _ ->
            {error, already_voted_for_other}
    end.
```

---

## Chaos Engineering

```erlang
%% chaos.erl: controlled failure injection for testing HA

-module(chaos).
-export([start/0, inject_failure/1, kill_random_process/0,
         partition_network/2, restore_network/0]).

start() ->
    logger:warning("CHAOS MODE ENABLED - DO NOT USE IN PRODUCTION"),
    {ok, _} = gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Inject various failure types
inject_failure(Type) ->
    gen_server:cast(?MODULE, {inject, Type}).

%% Kill a random worker process
kill_random_process() ->
    Workers = supervisor:which_children(my_worker_sup),
    case Workers of
        [] -> ok;
        _ ->
            {_, Pid, _, _} = lists:nth(rand:uniform(length(Workers)), Workers),
            exit(Pid, kill),
            logger:info("Chaos: killed process", #{pid => Pid})
    end.

%% Simulate network partition
partition_network(Node1, Node2) ->
    rpc:call(Node1, erlang, disconnect_node, [Node2]),
    logger:info("Chaos: partitioned nodes", #{node1 => Node1, node2 => Node2}).

restore_network() ->
    [net_adm:ping(N) || N <- all_nodes()].

handle_cast({inject, {slow_response, DelayMs}}, State) ->
    %% Make responses slow
    timer:sleep(DelayMs),
    {noreply, State};

handle_cast({inject, {memory_pressure, MB}}, State) ->
    %% Allocate memory to trigger GC pressure
    Bin = binary:copy(<<0>>, MB * 1024 * 1024),
    spawn(fun() -> timer:sleep(5000), _ = Bin end),
    {noreply, State};

handle_cast({inject, cpu_spike}, State) ->
    %% Spike CPU
    spawn(fun() ->
        Deadline = erlang:monotonic_time(second) + 5,
        heavy_loop(Deadline)
    end),
    {noreply, State}.

heavy_loop(Deadline) ->
    case erlang:monotonic_time(second) < Deadline of
        true ->
            _ = lists:foldl(fun(X, Acc) -> X + Acc end, 0, lists:seq(1, 10000)),
            heavy_loop(Deadline);
        false -> ok
    end.

all_nodes() ->
    application:get_env(my_app, cluster_nodes, []).
```

---

## ตัวอย่างจริง: 99.99% Availability System

```erlang
%% ha_system.erl: patterns for high availability

%% 1. Supervision tree สำหรับ auto-recovery
%% 2. Distributed Mnesia สำหรับ data HA
%% 3. Global registration สำหรับ single active process
%% 4. Quorum check ก่อน write

%% ha_sup.erl
-module(ha_sup).
-behaviour(supervisor).

init(_) ->
    %% restart_strategy: one_for_one, fast restarts (10x in 60s)
    {ok, {
        {one_for_one, 10, 60},
        [
            #{id => quorum_manager,
              start => {quorum_manager, start_link, []},
              restart => permanent,
              type => worker},
            #{id => ha_leader,
              start => {ha_leader, start_link, []},
              restart => permanent,
              type => worker},
            #{id => ha_data,
              start => {ha_data, start_link, []},
              restart => permanent,
              type => worker}
        ]
    }}.

%% ha_leader.erl: leader election + primary duties
-module(ha_leader).
-behaviour(gen_server).

init(_) ->
    %% Monitor all nodes
    net_kernel:monitor_nodes(true),
    %% Elect leader
    self() ! elect_leader,
    {ok, #{role => follower, leader => undefined, term => 0}}.

handle_info(elect_leader, State) ->
    %% Simple election: smallest node name wins
    AllNodes = [node() | nodes()],
    Leader = lists:min(AllNodes),
    case Leader =:= node() of
        true ->
            logger:info("I am the leader", #{term => maps:get(term, State) + 1}),
            announce_leadership(),
            {noreply, State#{role => leader, leader => node(),
                            term => maps:get(term, State) + 1}};
        false ->
            {noreply, State#{role => follower, leader => Leader}}
    end;

handle_info({nodedown, Node}, #{leader := Node} = State) ->
    logger:warning("Leader died, re-electing"),
    timer:send_after(rand:uniform(300), self(), elect_leader),
    {noreply, State#{role => follower, leader => undefined}};

handle_info({nodeup, _Node}, State) ->
    %% New node joined, re-elect to give it a chance
    timer:send_after(1000, self(), elect_leader),
    {noreply, State}.

announce_leadership() ->
    [rpc:cast(N, ha_leader, set_leader, [node()]) || N <- nodes()].

%% ha_data.erl: HA data with Mnesia
setup_ha_data() ->
    %% Create schema on all nodes
    Nodes = [node() | nodes()],
    mnesia:create_schema(Nodes),
    mnesia:start(),
    
    %% Create table with full replication
    mnesia:create_table(ha_data, [
        {attributes, [key, value, version, updated_at]},
        {disc_copies, Nodes},
        {type, set}
    ]),
    
    %% Majority writes: requires (N/2 + 1) nodes
    mnesia:change_table_majority(ha_data, true).

write_ha(Key, Value) ->
    case quorum_manager:is_quorum() of
        true ->
            mnesia:transaction(fun() ->
                mnesia:write({ha_data, Key, Value,
                              erlang:unique_integer([monotonic]),
                              erlang:system_time(millisecond)})
            end);
        false ->
            {error, no_quorum}
    end.

read_ha(Key) ->
    mnesia:dirty_read(ha_data, Key).
```

---

## สรุป Part 51

| Pattern | Availability Gain | Complexity |
|---------|------------------|------------|
| Supervision trees | 99.9% (process crashes) | Low |
| Multi-node cluster | 99.99% (node failures) | Medium |
| Quorum writes | Prevents split-brain | Medium |
| Active-passive | Simple failover | Low |
| Active-active | No single point | High |
| Chaos testing | Validate HA claims | High |

---

*[← Part 50: CI/CD](part_50_cicd.md) | [Part 52: Memory and Performance Deep Dive →](part_52_memory_performance.md)*
