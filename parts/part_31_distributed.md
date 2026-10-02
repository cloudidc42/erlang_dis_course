# Part 31: Distributed Erlang

## สารบัญ
1. [Distributed Erlang คืออะไร](#distributed-erlang-คืออืออะไร)
2. [Node Names และ Cookies](#node-names-และ-cookies)
3. [Connecting Nodes](#connecting-nodes)
4. [Remote Spawning](#remote-spawning)
5. [Global Registration](#global-registration)
6. [RPC — Remote Procedure Call](#rpc--remote-procedure-call)
7. [net_kernel และ epmd](#net_kernel-และ-epmd)
8. [ตัวอย่างจริง: Distributed Counter Cluster](#ตัวอย่างจริง-distributed-counter-cluster)

---

## Distributed Erlang คืออะไร

```erlang
%% Distributed Erlang: หลาย Erlang nodes ทำงานร่วมกัน
%% - หลาย machines (IP network)
%% - หลาย OS processes บน machine เดียว
%% - Transparent location: รู้ Pid ก็ส่งได้ไม่ว่าอยู่ node ไหน

%% ข้อดี:
%% - Horizontal scaling: เพิ่ม nodes ตามโหลด
%% - Fault tolerance: node ล้ม, อื่นยังทำงาน
%% - Geographic distribution: nodes ต่างประเทศ
%% - Resource sharing: ใช้ CPU/memory ของหลาย machines

%% สิ่งสำคัญ:
%% - Nodes เชื่อมต่อกันผ่าน TCP
%% - ต้องมี same Erlang cookie (security)
%% - epmd (Erlang Port Mapper Daemon) ทำ name resolution
%% - Message passing works the same (Pid ! Msg)

%% Node naming formats:
%% Short name: foo@hostname      (ไม่มี domain)
%% Long name:  foo@host.domain.com  (ใช้ FQDN)
%% ต้องใช้รูปแบบเดียวกันในทุก node ของ cluster
```

---

## Node Names และ Cookies

```bash
# Start node ด้วย short name
$ erl -sname node1
# เป็น node1@hostname

# Start node ด้วย long name
$ erl -name node1@192.168.1.1

# กำหนด cookie
$ erl -sname node1 -setcookie my_secret_cookie

# หรือสร้าง ~/.erlang.cookie (shared across nodes)
# $ echo "my_secret_cookie" > ~/.erlang.cookie
# $ chmod 400 ~/.erlang.cookie

# Start หลาย nodes บน machine เดียว
$ erl -sname a -setcookie dev_cookie &
$ erl -sname b -setcookie dev_cookie &
$ erl -sname c -setcookie dev_cookie &
```

```erlang
%% ใน Erlang code
node().           %% ชื่อ node ปัจจุบัน: node1@hostname
nodes().          %% list ของ connected nodes: [node2@hostname, ...]
nodes(connected). %% same as nodes()
nodes(hidden).    %% hidden nodes

erlang:set_cookie(node(), my_cookie).  %% set cookie programmatically
erlang:get_cookie().                   %% get current cookie

%% Check if node is alive (distributed)
net_kernel:longnames().  %% true ถ้า -name, false ถ้า -sname

%% Start distribution from running node
net_kernel:start([node1, shortnames]).
```

---

## Connecting Nodes

```erlang
%% Connect to another node
net_adm:ping(node2@hostname).
%% pong = success, pang = failure

%% Connect explicitly
true = net_kernel:connect_node(node2@hostname).

%% Disconnect
erlang:disconnect_node(node2@hostname).

%% Monitor nodes (ได้รับ message เมื่อ node up/down)
net_kernel:monitor_nodes(true).
%% receives: {nodeup, NodeName}
%% receives: {nodedown, NodeName}

net_kernel:monitor_nodes(true, [nodedown_reason]).
%% {nodedown, Node, [{nodedown_reason, Reason}]}

%% Auto-connect: ถ้า send message ไป Pid ที่ node อื่น
%% Erlang จะพยายาม connect อัตโนมัติ

%% ตัวอย่าง: monitor nodes ใน gen_server
init([]) ->
    net_kernel:monitor_nodes(true),
    {ok, #{nodes => []}}.

handle_info({nodeup, Node}, #{nodes := Ns} = State) ->
    io:format("Node joined: ~p~n", [Node]),
    {noreply, State#{nodes => [Node | Ns]}};

handle_info({nodedown, Node}, #{nodes := Ns} = State) ->
    io:format("Node left: ~p~n", [Node]),
    {noreply, State#{nodes => lists:delete(Node, Ns)}}.

%% Hidden nodes: connect แต่ไม่ appear ใน nodes()
net_kernel:connect_node(monitor@host).  %% normal
% vs
net_kernel:hidden_connect_node(monitor@host).  %% hidden
```

---

## Remote Spawning

```erlang
%% spawn บน node อื่น
RemotePid = spawn(OtherNode, Module, Function, Args).
RemotePid = spawn(OtherNode, fun() -> ... end).  %% fun ต้อง be serializable

%% spawn_link บน node อื่น
RemotePid = spawn_link(OtherNode, Module, Function, Args).

%% ส่ง message ข้าม node (transparent!)
RemotePid ! {hello, self()}.
%% เหมือนกับ local process

%% รับ message กลับ
receive
    {reply, Value} -> Value
end.

%% ตัวอย่าง: distributed computation
distribute_work(Nodes, Tasks) ->
    Workers = [
        {spawn(Node, ?MODULE, do_task, [Task]), Task}
        || {Node, Task} <- lists:zip(Nodes, Tasks)
    ],
    [
        receive
            {Pid, Result} -> Result
        end
        || {Pid, _} <- Workers
    ].

do_task(Task) ->
    Result = compute(Task),
    Owner ! {self(), Result}.

%% Remote spawn with monitor
{Pid, Ref} = spawn_monitor(RemoteNode, Module, Fun, Args).
receive
    {'DOWN', Ref, process, Pid, Reason} ->
        handle_worker_death(Reason)
end.
```

---

## Global Registration

```erlang
%% global module: register names across cluster

%% Register
global:register_name(my_server, self()).
%% หรือ
global:register_name(my_server, Pid).

%% Lookup
global:whereis_name(my_server).
%% Pid หรือ undefined

%% Unregister
global:unregister_name(my_server).

%% Send to globally registered process
global:send(my_server, Message).

%% List all global names
global:registered_names().

%% gen_server ที่ register globally
gen_server:start_link({global, my_global_server}, ?MODULE, [], []).

%% Call global server
gen_server:call({global, my_global_server}, Request).

%% global sync (force resync global name table)
global:sync().

%% Handle name conflicts (2 nodes register same name)
%% ต้อง implement resolve_conflict/3
global:register_name(Name, Pid, fun(Name, Pid1, Pid2) ->
    %% เลือก Pid ที่ keep
    case node(Pid1) > node(Pid2) of
        true -> Pid1;
        false -> Pid2
    end
end).

%% gproc: alternative registry (more flexible)
%% gproc:reg({n, l, my_name}).
%% gproc:where({n, l, my_name}).
```

---

## RPC — Remote Procedure Call

```erlang
%% rpc module: call functions on remote nodes

%% Synchronous call (blocks)
rpc:call(Node, Module, Function, Args).
rpc:call(Node, Module, Function, Args, Timeout).
%% Returns: Result หรือ {badrpc, Reason}

%% ตัวอย่าง
rpc:call(node2@host, io, format, ["Hello from ~p~n", [node()]]).
rpc:call(node2@host, erlang, node, []).

%% Safe call with error handling
case rpc:call(Node, Module, Fun, Args) of
    {badrpc, nodedown} -> {error, node_unavailable};
    {badrpc, Reason} -> {error, Reason};
    Result -> {ok, Result}
end.

%% Async call (non-blocking)
Keys = [rpc:async_call(Node, Mod, Fun, Args) || Node <- Nodes],
Results = [rpc:yield(Key) || Key <- Keys].

%% Cast (fire-and-forget)
rpc:cast(Node, Module, Function, Args).

%% Multi-call: call on multiple nodes simultaneously
rpc:multicall([Node1, Node2, Node3], Mod, Fun, Args).
%% Returns: {Results, BadNodes}
%% Results = [Result || successful node]
%% BadNodes = [Node || failed node]

%% multicall with timeout
rpc:multicall([Node1, Node2], Mod, Fun, Args, 5000).

%% ตัวอย่าง: get status from all nodes
get_cluster_status() ->
    AllNodes = [node() | nodes()],
    {Results, BadNodes} = rpc:multicall(AllNodes, my_app, status, []),
    #{
        ok => lists:zip(AllNodes -- BadNodes, Results),
        down => BadNodes
    }.
```

---

## net_kernel และ epmd

```erlang
%% epmd (Erlang Port Mapper Daemon)
%% - ทำงานเป็น OS process
%% - Listen port 4369
%% - Maps node names to ports
%% - ทุก node register ตัวเองกับ local epmd

%% Start epmd manually
%% $ epmd -daemon

%% Check running nodes
%% $ epmd -names
%% epmd: up and running on port 4369 with data:
%% name node1 at port 52345

%% Disable epmd (สำหรับ cloud)
%% erl -proto_dist inet_tcp -epmd_port 0 ...

%% net_kernel
net_kernel:start([node_name, shortnames]).  %% เปิด distribution
net_kernel:stop().                           %% ปิด distribution

%% Kernel poll (improve performance)
%% erl +K true -sname node1

%% Distribution buffer (tuning)
%% erl +zdbbl 8192 -sname node1   %% 8192 KiB distribution buffer

%% TLS distribution (production security)
%% ต้องตั้งค่า ssl_dist.config
{server, [
    {certfile, "/path/to/server.crt"},
    {keyfile, "/path/to/server.key"},
    {cacertfile, "/path/to/ca.crt"}
]}.
{client, [
    {certfile, "/path/to/client.crt"},
    {keyfile, "/path/to/client.key"},
    {cacertfile, "/path/to/ca.crt"}
]}.
%% erl -ssl_dist_opt server_certfile /path/cert.crt ...
```

---

## ตัวอย่างจริง: Distributed Counter Cluster

```erlang
%% ไฟล์: dist_counter.erl
-module(dist_counter).
-behaviour(gen_server).

%% API
-export([start_link/0, increment/0, get/0, cluster_get/0]).
%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    count = 0 :: non_neg_integer(),
    known_nodes = [] :: [node()]
}).

start_link() ->
    gen_server:start_link({global, {dist_counter, node()}}, ?MODULE, [], []).

increment() ->
    %% Increment on this node
    gen_server:cast({global, {dist_counter, node()}}, increment),
    %% Broadcast to other nodes
    [gen_server:cast({global, {dist_counter, N}}, increment)
     || N <- nodes()].

get() ->
    gen_server:call({global, {dist_counter, node()}}, get).

cluster_get() ->
    AllNodes = [node() | nodes()],
    Results = [
        case rpc:call(N, gen_server, call, [{global, {dist_counter, N}}, get], 1000) of
            {badrpc, _} -> {N, error};
            Count -> {N, Count}
        end
        || N <- AllNodes
    ],
    Total = lists:sum([C || {_, C} <- Results, is_integer(C)]),
    #{per_node => Results, total => Total}.

init([]) ->
    net_kernel:monitor_nodes(true),
    %% Connect to all known nodes at start
    [net_adm:ping(N) || N <- nodes()],
    {ok, #state{known_nodes = nodes()}}.

handle_call(get, _From, #state{count = N} = State) ->
    {reply, N, State}.

handle_cast(increment, #state{count = N} = State) ->
    {noreply, State#state{count = N + 1}}.

handle_info({nodeup, Node}, #state{known_nodes = KN} = State) ->
    io:format("[~p] Node joined cluster: ~p~n", [node(), Node]),
    {noreply, State#state{known_nodes = [Node | KN]}};

handle_info({nodedown, Node}, #state{known_nodes = KN} = State) ->
    io:format("[~p] Node left cluster: ~p~n", [node(), Node]),
    {noreply, State#state{known_nodes = lists:delete(Node, KN)}};

handle_info(_, State) ->
    {noreply, State}.

terminate(_, _) -> ok.
code_change(_, State, _) -> {ok, State}.
```

```bash
# Terminal 1
$ erl -sname node1 -setcookie cluster_cookie
(node1@host)1> dist_counter:start_link().

# Terminal 2
$ erl -sname node2 -setcookie cluster_cookie
(node2@host)1> net_adm:ping(node1@host).
pong
(node2@host)2> dist_counter:start_link().

# Terminal 3
$ erl -sname node3 -setcookie cluster_cookie
(node3@host)1> net_adm:ping(node1@host).
pong

# On any node:
(node1@host)3> dist_counter:increment().
(node1@host)4> dist_counter:increment().
(node1@host)5> dist_counter:cluster_get().
#{
  per_node => [{node1@host,2},{node2@host,2},{node3@host,2}],
  total => 6
}
```

---

## สรุป Part 31

| Function | Description |
|----------|-------------|
| `net_adm:ping/1` | Connect to node |
| `nodes/0` | List connected nodes |
| `spawn(Node, M, F, A)` | Spawn on remote node |
| `rpc:call/4,5` | Synchronous remote call |
| `rpc:multicall/4,5` | Call on multiple nodes |
| `global:register_name/2` | Global name registration |
| `net_kernel:monitor_nodes/1` | Monitor node up/down |

---

*[← Part 30: Testing](part_30_testing.md) | [Part 32: Distributed Patterns →](part_32_distributed_patterns.md)*
