# Part 86: Distributed Coordination

## สารบัญ
1. [Leader Election](#leader-election)
2. [Distributed Consensus](#consensus)
3. [Distributed Locks](#locks)
4. [Cluster Membership](#membership)
5. [Global Configuration](#config)
6. [ตัวอย่างจริง: Cluster Scheduler](#scheduler)

---

## Leader Election

```erlang
%% leader_election.erl — Bully algorithm for leader election

-module(leader_election).
-behaviour(gen_server).
-export([start_link/1, get_leader/0, am_i_leader/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    node_id,           %% unique numeric ID for this node
    leader = undefined,
    election_in_progress = false,
    nodes = []
}).

start_link(NodeId) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, NodeId, []).

get_leader() ->
    gen_server:call(?MODULE, get_leader).

am_i_leader() ->
    gen_server:call(?MODULE, am_i_leader).

init(NodeId) ->
    %% Monitor cluster nodes
    net_kernel:monitor_nodes(true),
    
    %% Start election when we join
    erlang:send_after(1000, self(), start_election),
    
    {ok, #state{node_id = NodeId}}.

handle_info(start_election, State) ->
    {noreply, start_election(State)};

handle_info({nodeup, Node}, State) ->
    logger:info("Node joined", #{node => Node}),
    Nodes = [Node | State#state.nodes],
    {noreply, State#state{nodes = Nodes}};

handle_info({nodedown, Node}, State) ->
    logger:info("Node left", #{node => Node}),
    Nodes = lists:delete(Node, State#state.nodes),
    NewState = State#state{nodes = Nodes},
    
    %% If leader went down, start new election
    case State#state.leader =:= Node of
        true ->
            {noreply, start_election(NewState#state{leader = undefined})};
        false ->
            {noreply, NewState}
    end;

handle_info({election, FromNodeId, FromNode}, #state{node_id = MyId} = State) ->
    case MyId > FromNodeId of
        true ->
            %% I have higher ID, send OK and start my own election
            gen_server:cast({?MODULE, FromNode}, {ok, MyId, node()}),
            {noreply, start_election(State)};
        false ->
            %% Other node has higher ID, let them proceed
            {noreply, State}
    end;

handle_info({coordinator, LeaderId, LeaderNode}, State) ->
    logger:info("New leader elected", #{leader => LeaderNode, id => LeaderId}),
    {noreply, State#state{leader = LeaderNode, election_in_progress = false}};

handle_info({ok, _HigherNodeId, _Node}, State) ->
    %% Someone with higher ID responded, they'll take over
    {noreply, State#state{election_in_progress = false}};

handle_info(election_timeout, #state{election_in_progress = true, node_id = MyId} = State) ->
    %% No one with higher ID responded — I am the leader!
    logger:info("I am the leader", #{node => node(), id => MyId}),
    broadcast_coordinator(MyId),
    {noreply, State#state{leader = node(), election_in_progress = false}};

handle_info(election_timeout, State) ->
    {noreply, State}.

handle_call(get_leader, _From, State) ->
    {reply, State#state.leader, State};
handle_call(am_i_leader, _From, State) ->
    {reply, State#state.leader =:= node(), State}.

start_election(#state{node_id = MyId, nodes = Nodes} = State) ->
    logger:info("Starting election", #{my_id => MyId}),
    
    HigherNodes = [N || N <- Nodes, node_id(N) > MyId],
    
    case HigherNodes of
        [] ->
            %% I have the highest ID, declare myself leader
            broadcast_coordinator(MyId),
            State#state{leader = node(), election_in_progress = false};
        _ ->
            %% Send election messages to higher nodes
            [gen_server:cast({?MODULE, N}, {election, MyId, node()}) || N <- HigherNodes],
            %% Wait for OK responses
            erlang:send_after(3000, self(), election_timeout),
            State#state{election_in_progress = true}
    end.

broadcast_coordinator(MyId) ->
    [gen_server:cast({?MODULE, N}, {coordinator, MyId, node()}) || N <- nodes()].

node_id(Node) ->
    %% Extract numeric ID from node name, e.g. node3@host -> 3
    [Name, _Host] = string:split(atom_to_list(Node), "@"),
    case re:run(Name, "[0-9]+$", [{capture, first, list}]) of
        {match, [Num]} -> list_to_integer(Num);
        _ -> 0
    end.
```

---

## Distributed Locks with Redis

```erlang
%% dist_lock.erl — Redis-based distributed locking (Redlock algorithm)

-module(dist_lock).
-export([acquire/3, release/2, with_lock/4]).

-define(LOCK_TTL_MS, 30000).      %% 30 second TTL
-define(RETRY_TIMES, 3).
-define(RETRY_DELAY_MS, 200).

%% Acquire lock with retries
acquire(LockKey, ClientId, TtlMs) ->
    acquire_with_retry(LockKey, ClientId, TtlMs, ?RETRY_TIMES).

acquire_with_retry(_Key, _ClientId, _Ttl, 0) ->
    {error, lock_not_acquired};
acquire_with_retry(Key, ClientId, TtlMs, Retries) ->
    case try_acquire(Key, ClientId, TtlMs) of
        ok -> {ok, ClientId};
        {error, _} ->
            timer:sleep(?RETRY_DELAY_MS + rand:uniform(?RETRY_DELAY_MS)),
            acquire_with_retry(Key, ClientId, TtlMs, Retries - 1)
    end.

try_acquire(Key, ClientId, TtlMs) ->
    %% SET key value NX PX ttl
    case redis:command(["SET", Key, ClientId, "NX", "PX", TtlMs]) of
        {ok, <<"OK">>} -> ok;
        {ok, undefined} -> {error, already_locked};
        Error -> Error
    end.

%% Release lock (only if we own it)
release(LockKey, ClientId) ->
    %% Atomic check-and-delete via Lua script
    Script = "if redis.call('GET', KEYS[1]) == ARGV[1] then "
             "  return redis.call('DEL', KEYS[1]) "
             "else "
             "  return 0 "
             "end",
    case redis:command(["EVAL", Script, 1, LockKey, ClientId]) of
        {ok, 1} -> ok;
        {ok, 0} -> {error, not_owner};
        Error -> Error
    end.

%% Execute function with lock
with_lock(Key, TtlMs, Fun, Timeout) ->
    ClientId = lock_client_id(),
    
    case acquire(Key, ClientId, TtlMs) of
        {ok, ClientId} ->
            try
                Fun()
            after
                release(Key, ClientId)
            end;
        {error, Reason} ->
            {error, {lock_failed, Reason}}
    end.

lock_client_id() ->
    NodeId = atom_to_binary(node()),
    Rand = base64:encode(crypto:strong_rand_bytes(6)),
    <<NodeId/binary, ":", Rand/binary>>.

%% Extend lock TTL while work is in progress
extend_lock(Key, ClientId, TtlMs) ->
    Script = "if redis.call('GET', KEYS[1]) == ARGV[1] then "
             "  return redis.call('PEXPIRE', KEYS[1], ARGV[2]) "
             "else "
             "  return 0 "
             "end",
    redis:command(["EVAL", Script, 1, Key, ClientId, TtlMs]).
```

---

## Cluster Membership with pg

```erlang
%% cluster_membership.erl — track cluster members using pg

-module(cluster_membership).
-behaviour(gen_server).
-export([start_link/0, register/2, get_members/1, get_all_members/0]).
-export([init/1, handle_info/2]).

-define(SCOPE, cluster_members).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% Use pg for cluster-wide process groups
    pg:start_link(?SCOPE),
    net_kernel:monitor_nodes(true),
    {ok, #{}}.

register(ServiceType, Pid) ->
    pg:join(?SCOPE, ServiceType, Pid).

get_members(ServiceType) ->
    pg:get_members(?SCOPE, ServiceType).

get_all_members() ->
    pg:which_groups(?SCOPE).

handle_info({nodeup, Node}, State) ->
    logger:info("Cluster node joined", #{node => Node}),
    %% Publish cluster event
    event_bus:publish(cluster_node_up, #{node => Node}),
    {noreply, State};

handle_info({nodedown, Node}, State) ->
    logger:info("Cluster node left", #{node => Node}),
    %% All processes on that node automatically removed from pg
    event_bus:publish(cluster_node_down, #{node => Node}),
    {noreply, State}.

%% Call a function on every member of a service type
broadcast_call(ServiceType, Fun) ->
    Members = get_members(ServiceType),
    [Fun(Pid) || Pid <- Members].

%% Call a function on a random member (load balancing)
random_call(ServiceType, Fun) ->
    Members = get_members(ServiceType),
    case Members of
        [] -> {error, no_members};
        _ ->
            Pid = lists:nth(rand:uniform(length(Members)), Members),
            Fun(Pid)
    end.
```

---

## ตัวอย่างจริง: Cluster Scheduler

```erlang
%% cluster_scheduler.erl — distributed work scheduler with leader election

-module(cluster_scheduler).
-behaviour(gen_server).
-export([start_link/0, schedule/2, get_pending/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    jobs = #{},           %% jobId => job spec
    running = #{},        %% jobId => {pid, node}
    tick_interval = 5000
}).

-record(job, {
    id,
    schedule,            %% cron expression or interval
    handler_module,
    handler_args,
    last_run,
    next_run
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    erlang:send_after(1000, self(), tick),
    {ok, #state{}}.

%% Only leader executes scheduled jobs
handle_info(tick, State) ->
    NewState = case leader_election:am_i_leader() of
        true -> run_due_jobs(State);
        false -> State
    end,
    erlang:send_after(State#state.tick_interval, self(), tick),
    {noreply, NewState}.

schedule(JobSpec, HandlerModule) ->
    gen_server:call(?MODULE, {schedule, JobSpec, HandlerModule}).

handle_call({schedule, #{id := Id} = Spec, HandlerModule}, _From, #state{jobs = Jobs} = State) ->
    Job = #job{
        id = Id,
        schedule = maps:get(schedule, Spec),
        handler_module = HandlerModule,
        handler_args = maps:get(args, Spec, []),
        next_run = compute_next_run(maps:get(schedule, Spec))
    },
    {reply, ok, State#state{jobs = Jobs#{Id => Job}}}.

run_due_jobs(#state{jobs = Jobs, running = Running} = State) ->
    Now = erlang:system_time(second),
    
    DueJobs = maps:filter(fun(_Id, Job) ->
        Job#job.next_run =< Now andalso not maps:is_key(Job#job.id, Running)
    end, Jobs),
    
    NewRunning = maps:fold(fun(Id, Job, Acc) ->
        %% Distribute job to least-loaded node
        TargetNode = select_target_node(),
        
        Pid = spawn(TargetNode, fun() ->
            logger:info("Running job", #{job_id => Id, node => node()}),
            try Job#job.handler_module:run(Job#job.handler_args)
            catch C:R ->
                logger:error("Job failed", #{job_id => Id, class => C, reason => R})
            end,
            %% Notify scheduler
            gen_server:cast({?MODULE, node()}, {job_done, Id})
        end),
        
        Acc#{Id => {Pid, TargetNode}}
    end, Running, DueJobs),
    
    %% Update next_run for started jobs
    NewJobs = maps:map(fun(Id, Job) ->
        case maps:is_key(Id, DueJobs) of
            true -> Job#job{
                last_run = Now,
                next_run = compute_next_run(Job#job.schedule)
            };
            false -> Job
        end
    end, Jobs),
    
    State#state{jobs = NewJobs, running = NewRunning}.

handle_cast({job_done, JobId}, #state{running = Running} = State) ->
    {noreply, State#state{running = maps:remove(JobId, Running)}}.

select_target_node() ->
    %% Pick least-loaded node from cluster
    AllNodes = [node() | nodes()],
    
    Loads = [{N, get_node_load(N)} || N <- AllNodes],
    {TargetNode, _} = lists:min(fun({_, L1}, {_, L2}) -> L1 < L2 end, Loads),
    TargetNode.

get_node_load(Node) ->
    try
        rpc:call(Node, erlang, statistics, [run_queue], 2000)
    catch _:_ ->
        999  %% assume overloaded if unreachable
    end.

compute_next_run({interval, Seconds}) ->
    erlang:system_time(second) + Seconds;
compute_next_run({cron, _Expr}) ->
    %% Simplified: next minute
    erlang:system_time(second) + 60.

get_pending() ->
    gen_server:call(?MODULE, get_pending).

handle_call(get_pending, _From, #state{jobs = Jobs} = State) ->
    Now = erlang:system_time(second),
    Pending = maps:filter(fun(_, J) -> J#job.next_run > Now end, Jobs),
    {reply, Pending, State}.
```

---

## สรุป Part 86

| Problem | Solution | Library |
|---------|---------|---------|
| Leader election | Bully algorithm | Custom gen_server |
| Distributed locks | Redlock | Redis |
| Cluster membership | Process groups | pg (built-in) |
| Node monitoring | nodeup/nodedown | net_kernel |
| Work distribution | RPC + run_queue | rpc (built-in) |

**Erlang advantage**: Distributed primitives (nodes(), rpc, pg) built into the VM — no external coordination service needed for same-cluster work.

---

*[← Part 85: Data Pipelines](part_85_data_pipelines.md) | [Part 87: Advanced Message Patterns →](part_87_message_patterns.md)*
