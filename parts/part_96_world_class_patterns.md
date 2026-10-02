# Part 96: World-Class Patterns

## สารบัญ
1. [WhatsApp-Scale Architecture](#whatsapp-scale)
2. [Ejabberd-Style XMPP Server](#xmpp)
3. [Riak-Style Distributed KV Store](#riak)
4. [High-Availability Patterns](#ha)
5. [Multi-Region Deployment](#multi-region)
6. [ตัวอย่างจริง: Millions of Connections](#millions)

---

## WhatsApp-Scale Architecture

```
WhatsApp (acquired for $19B, ~50 engineers, ~450M users)
Built on Erlang/FreeBSD

Key architectural decisions:
1. One process per connection (millions of lightweight processes)
2. Message routing via distributed process registry
3. Presence via efficient pub/sub
4. Offline message storage in Mnesia/PostgreSQL
5. Media on S3, links sent over message protocol

Lessons:
- Erlang's actor model = natural fit for messaging
- Per-process GC prevents latency spikes
- Hot code reload = zero-downtime updates
- Supervisor trees = self-healing without manual ops

Scale achieved:
- 2 servers → 2.5M concurrent connections (2012)
- ~50 Erlang engineers maintained this
```

```erlang
%% Simplified WhatsApp-style message routing

-module(message_router).
-export([route/2, register_session/2, unregister_session/1]).

%% Global process registry for user sessions
%% Each user's active connection is a registered process

route(ToUserId, Message) ->
    case gproc:lookup_pids({n, g, {user_session, ToUserId}}) of
        [] ->
            %% User offline: store for later delivery
            offline_store:save(ToUserId, Message);
        [Pid | _] ->
            %% User online: deliver immediately
            Pid ! {deliver_message, Message},
            ok
    end.

register_session(UserId, ConnPid) ->
    gproc:reg({n, g, {user_session, UserId}}, ConnPid),
    
    %% Deliver queued offline messages
    OfflineMessages = offline_store:pop_all(UserId),
    [ConnPid ! {deliver_message, M} || M <- OfflineMessages].

unregister_session(UserId) ->
    gproc:unreg({n, g, {user_session, UserId}}).

%% Connection process: one per connected user
-module(user_connection).
-behaviour(gen_server).
-export([start_link/2]).
-export([init/1, handle_info/2, handle_cast/2]).

-record(state, {user_id, socket, buffer = <<>>}).

start_link(UserId, Socket) ->
    gen_server:start_link(?MODULE, {UserId, Socket}, []).

init({UserId, Socket}) ->
    %% Register this connection
    message_router:register_session(UserId, self()),
    
    %% Handle socket in active mode
    inet:setopts(Socket, [{active, once}]),
    
    {ok, #state{user_id = UserId, socket = Socket}}.

handle_info({deliver_message, Message}, #state{socket = Socket} = State) ->
    %% Encode and send to client
    Frame = encode_message(Message),
    gen_tcp:send(Socket, Frame),
    {noreply, State};

handle_info({tcp, Socket, Data}, #state{buffer = Buffer} = State) ->
    NewBuffer = <<Buffer/binary, Data/binary>>,
    {Messages, Rest} = decode_frames(NewBuffer),
    
    [process_incoming(M, State) || M <- Messages],
    
    inet:setopts(Socket, [{active, once}]),
    {noreply, State#state{buffer = Rest}};

handle_info({tcp_closed, _}, State) ->
    message_router:unregister_session(State#state.user_id),
    {stop, normal, State}.

process_incoming(#{to := ToUserId} = Message, #state{user_id = FromUserId}) ->
    MessageWithMeta = Message#{
        from => FromUserId,
        id => generate_id(),
        timestamp => erlang:system_time(millisecond)
    },
    message_router:route(ToUserId, MessageWithMeta).

encode_message(Message) ->
    Data = jiffy:encode(Message),
    Size = byte_size(Data),
    <<Size:32/big, Data/binary>>.

decode_frames(Buffer) ->
    decode_frames(Buffer, []).

decode_frames(<<Size:32/big, Rest/binary>>, Acc) when byte_size(Rest) >= Size ->
    <<Frame:Size/binary, Remaining/binary>> = Rest,
    Message = jiffy:decode(Frame, [return_maps]),
    decode_frames(Remaining, [Message | Acc]);
decode_frames(Buffer, Acc) ->
    {lists:reverse(Acc), Buffer}.

generate_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).
```

---

## Riak-Style Distributed KV Store

```erlang
%% Simplified consistent hashing + replication

-module(dkv).
-export([put/2, get/1, delete/1]).

-define(RING_SIZE, 64).          %% partitions
-define(REPLICATION_FACTOR, 3).  %% N copies

%% Consistent hash ring
get_preference_list(Key) ->
    Hash = erlang:phash2(Key, ?RING_SIZE),
    
    %% Get this partition's primary node + next N-1 nodes
    AllNodes = [node() | nodes()],
    
    case length(AllNodes) of
        0 -> [node()];
        N ->
            Start = Hash rem N,
            %% Wrap-around ring
            Nodes = AllNodes ++ AllNodes,
            lists:sublist(lists:nthtail(Start, Nodes), ?REPLICATION_FACTOR)
    end.

%% Write with quorum
put(Key, Value) ->
    Nodes = get_preference_list(Key),
    VClock = vector_clock:increment(node(), get_vclock(Key)),
    
    WriteResults = [begin
        try rpc:call(N, dkv_vnode, put, [Key, Value, VClock], 5000)
        catch _:_ -> {error, node_down}
        end
    end || N <- Nodes],
    
    Successes = [ok || ok <- WriteResults],
    Required = (?REPLICATION_FACTOR div 2) + 1,  %% quorum
    
    case length(Successes) >= Required of
        true -> ok;
        false -> {error, write_quorum_failed}
    end.

%% Read with quorum reconciliation
get(Key) ->
    Nodes = get_preference_list(Key),
    
    ReadResults = [begin
        try rpc:call(N, dkv_vnode, get, [Key], 5000)
        catch _:_ -> {error, node_down}
        end
    end || N <- Nodes],
    
    Responses = [R || {ok, R} <- ReadResults],
    Required = (?REPLICATION_FACTOR div 2) + 1,
    
    case length(Responses) >= Required of
        true ->
            %% Reconcile: return most recent by vector clock
            Sorted = lists:sort(fun(A, B) ->
                vector_clock:descends(maps:get(vclock, A), maps:get(vclock, B))
            end, Responses),
            {ok, maps:get(value, hd(Sorted))};
        false ->
            {error, read_quorum_failed}
    end.

delete(Key) ->
    %% Tombstone deletion
    put(Key, tombstone).

%% Vector clock
-module(vector_clock).
-export([new/0, increment/2, descends/2, merge/2]).

new() -> #{}.

increment(Node, Clock) ->
    Counter = maps:get(Node, Clock, 0),
    Clock#{Node => Counter + 1}.

descends(ClockA, ClockB) ->
    %% A descends B if A[n] >= B[n] for all n
    maps:fold(fun(Node, CountB, Acc) ->
        CountA = maps:get(Node, ClockA, 0),
        Acc andalso CountA >= CountB
    end, true, ClockB).

merge(ClockA, ClockB) ->
    AllNodes = lists:usort(maps:keys(ClockA) ++ maps:keys(ClockB)),
    maps:from_list([{N, max(maps:get(N, ClockA, 0), maps:get(N, ClockB, 0))}
                    || N <- AllNodes]).

get_vclock(Key) ->
    case dkv_vnode:get(Key) of
        {ok, #{vclock := VClock}} -> VClock;
        _ -> vector_clock:new()
    end.
```

---

## High-Availability Patterns

```erlang
%% ha_patterns.erl — patterns for high availability

%% 1. Active-Active with conflict resolution
-module(active_active).
-export([write/2, read/1]).

write(Key, Value) ->
    %% Write to all nodes
    Nodes = [node() | nodes()],
    [rpc:call(N, local_store, write, [Key, Value]) || N <- Nodes].

read(Key) ->
    %% Read from any node, prefer local
    case local_store:read(Key) of
        {ok, Value} -> {ok, Value};
        not_found ->
            %% Try other nodes
            case try_remote_read(Key, nodes()) of
                {ok, Value} ->
                    %% Sync to local
                    local_store:write(Key, Value),
                    {ok, Value};
                not_found -> not_found
            end
    end.

try_remote_read(_Key, []) -> not_found;
try_remote_read(Key, [Node | Rest]) ->
    case rpc:call(Node, local_store, read, [Key], 2000) of
        {ok, Value} -> {ok, Value};
        _ -> try_remote_read(Key, Rest)
    end.

%% 2. Mnesia high-availability configuration
mnesia_ha_config() ->
    %% Configure Mnesia for high availability
    
    %% Write to majority of nodes (quorum)
    mnesia:change_config(extra_db_nodes, [node() | nodes()]),
    
    %% Create table with full replication
    mnesia:create_table(users, [
        {disc_copies, [node() | nodes()]},  %% all nodes have copy
        {attributes, record_info(fields, user)}
    ]),
    
    %% Auto-merge partitions on reconnect
    application:set_env(mnesia, dc_dump_limit, 40),
    application:set_env(mnesia, dump_log_time_threshold, 180000).

%% 3. Automatic failover
-module(failover_manager).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_info/2]).

init([]) ->
    %% Monitor all cluster nodes
    net_kernel:monitor_nodes(true),
    {ok, #{active_primary => node()}}.

handle_info({nodedown, FailedNode}, State) ->
    logger:warning("Node failed", #{node => FailedNode}),
    
    %% Check if it was primary
    case maps:get(active_primary, State) =:= FailedNode of
        true ->
            %% Trigger new leader election
            leader_election:start_election(),
            {noreply, State#{active_primary => undefined}};
        false ->
            %% Secondary failed, not critical
            {noreply, State}
    end;

handle_info({nodeup, NewNode}, State) ->
    logger:info("Node recovered", #{node => NewNode}),
    
    %% Sync state to recovered node
    sync_state_to_node(NewNode),
    
    {noreply, State}.

sync_state_to_node(Node) ->
    %% Send current state to reconnected node
    rpc:call(Node, state_sync, receive_full_sync, [get_full_state()]).

get_full_state() ->
    %% Collect all relevant state
    #{
        ets_tables => dump_ets_tables(),
        config => application:get_all_env(my_app)
    }.

dump_ets_tables() ->
    [{T, ets:tab2list(T)} || T <- ets:all(), is_atom(T)].
```

---

## ตัวอย่างจริง: Millions of Connections

```erlang
%% Load test and benchmarks for 1M+ connections

%% System configuration for 1M connections:
system_config_for_1m() ->
    %% vm.args:
    %% +P 2000000    -- 2M processes (1 per connection + overhead)
    %% +Q 65536      -- max ports
    %% +K true       -- kernel poll (epoll)
    %% +S 16:16      -- all CPU cores
    
    %% /etc/sysctl.conf:
    %% fs.file-max = 2000000
    %% net.core.somaxconn = 65535
    %% net.ipv4.tcp_tw_reuse = 1
    %% net.ipv4.ip_local_port_range = 1024 65535
    
    %% /etc/security/limits.conf:
    %% * soft nofile 1000000
    %% * hard nofile 1000000
    ok.

%% Benchmark: measure connection overhead
connection_benchmark() ->
    N = 1000000,
    
    %% Measure memory per connection
    Before = erlang:memory(processes),
    
    %% Spawn N idle connection processes
    Pids = [spawn(fun idle_connection/0) || _ <- lists:seq(1, N)],
    
    After = erlang:memory(processes),
    PerConnection = (After - Before) div N,
    
    io:format("Memory per idle connection: ~p bytes~n", [PerConnection]),
    %% Expected: ~1-3KB per connection (stack + PCB)
    
    %% Cleanup
    [exit(P, kill) || P <- Pids].

idle_connection() ->
    receive
        stop -> ok;
        _ -> idle_connection()
    end.

%% Benchmark message throughput
message_throughput_test() ->
    %% Create N receiver processes
    Receivers = [spawn(fun receiver_loop/0) || _ <- lists:seq(1, 10000)],
    
    %% Send M messages to each
    T1 = erlang:monotonic_time(microsecond),
    
    [begin
        [R ! {msg, N} || R <- Receivers]
    end || N <- lists:seq(1, 100)],
    
    T2 = erlang:monotonic_time(microsecond),
    
    TotalMessages = length(Receivers) * 100,
    Duration = (T2 - T1) / 1000,
    Throughput = TotalMessages * 1000 / Duration,
    
    io:format("~p messages in ~p ms = ~p msg/sec~n",
              [TotalMessages, round(Duration), round(Throughput)]).

receiver_loop() ->
    receive
        {msg, _} -> receiver_loop();
        stop -> ok
    end.
```

---

## สรุป Part 96

| Scale Level | Techniques |
|-------------|-----------|
| 10K connections | Default config |
| 100K connections | Tune +P, file limits |
| 1M connections | All OS tuning + kernel poll |
| 10M connections | Multi-node cluster + load balancer |
| 450M users (WhatsApp) | Distributed Erlang + smart routing |

**World-class insight**: Erlang was purpose-built for exactly these workloads. The BEAM VM's process model, fault tolerance, and hot code reload aren't afterthoughts — they're the core design.

---

*[← Part 95: E-Commerce](part_95_final_project_ecommerce.md) | [Part 97: Erlang Ecosystem Deep Dive →](part_97_ecosystem_deep_dive.md)*
