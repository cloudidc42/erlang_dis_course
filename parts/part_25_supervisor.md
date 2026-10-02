# Part 25: Supervisor — Fault Tolerant Process Trees

## สารบัญ
1. [Supervisor คืออะไร](#supervisor-คืออะไร)
2. [Child Specification](#child-specification)
3. [Restart Strategies](#restart-strategies)
4. [Intensity และ Period](#intensity-และ-period)
5. [Dynamic Children (simple_one_for_one)](#dynamic-children-simple_one_for_one)
6. [Nested Supervisors](#nested-supervisors)
7. [ตัวอย่างจริง: Web Server Supervisor Tree](#ตัวอย่างจริง-web-server-supervisor-tree)

---

## Supervisor คืออะไร

```erlang
%% Supervisor = process ที่ทำหน้าที่ดูแล children processes
%% เมื่อ child crash -> supervisor restart ตาม strategy ที่กำหนด
%% Root supervisor ถูก start โดย Application behaviour

%% โครงสร้าง:
%%   Application
%%     └── Root Supervisor
%%           ├── Worker 1 (gen_server)
%%           ├── Worker 2 (gen_server)
%%           └── Sub-Supervisor
%%                 ├── Worker 3
%%                 └── Worker 4

%% ข้อดี:
%% - Automatic restart เมื่อ crash
%% - Controlled shutdown (ปิดตามลำดับ)
%% - Visibility (รู้ว่า children มีอะไรบ้าง)
%% - Fault isolation (child crash ไม่กระทบ sibling โดยตรง)

-module(my_sup).
-behaviour(supervisor).

-export([start_link/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy => one_for_one,
        intensity => 3,
        period => 10
    },
    ChildSpecs = [
        #{
            id => my_worker,
            start => {my_worker, start_link, []},
            restart => permanent,
            shutdown => 5000,
            type => worker,
            modules => [my_worker]
        }
    ],
    {ok, {SupFlags, ChildSpecs}}.
```

---

## Child Specification

```erlang
%% Child spec แบบ map (OTP 18+)
ChildSpec = #{
    id => my_worker,          %% unique ID (term) - required
    start => {Mod, Fun, Args},%% how to start - required
    restart => permanent,      %% when to restart - optional (default: permanent)
    shutdown => 5000,         %% how to stop (ms or brutal_kill or infinity) - optional (default: 5000)
    type => worker,           %% worker | supervisor - optional (default: worker)
    modules => [Mod]          %% for hot upgrade - optional (default: [Mod from start])
},

%% restart options:
%% permanent - always restart (default)
%% temporary - never restart
%% transient - restart only if abnormal exit (reason != normal, shutdown, {shutdown, T})

%% shutdown options:
%% 5000      - send exit(Pid, shutdown), wait 5s, then brutal_kill
%% brutal_kill - kill immediately with exit(Pid, kill)
%% infinity   - wait forever (for supervisors)

%% Examples:

%% Worker: permanent, 5s shutdown
worker_spec(Name, Mod, Args) ->
    #{
        id => Name,
        start => {Mod, start_link, Args},
        restart => permanent,
        shutdown => 5000,
        type => worker,
        modules => [Mod]
    }.

%% Supervisor child: infinity shutdown
sup_spec(Name, Mod) ->
    #{
        id => Name,
        start => {Mod, start_link, []},
        restart => permanent,
        shutdown => infinity,
        type => supervisor,
        modules => [Mod]
    }.

%% Temporary worker (task, won't restart)
temp_spec(Id, Fun) ->
    #{
        id => Id,
        start => {erlang, apply, [Fun, []]},
        restart => temporary,
        shutdown => 1000,
        type => worker
    }.
```

---

## Restart Strategies

```erlang
%% one_for_one (most common)
%% ถ้า child crash -> restart แค่ child นั้น
%% ใช้เมื่อ: children ทำงานอิสระจากกัน

init_one_for_one([]) ->
    {ok, {
        #{strategy => one_for_one, intensity => 5, period => 60},
        [
            worker_spec(cache, cache_server, []),
            worker_spec(logger, log_server, []),
            worker_spec(api, api_server, [])
        ]
    }}.

%% one_for_all
%% ถ้า child crash -> restart ทุก child
%% ใช้เมื่อ: children depend กัน, ต้อง restart พร้อมกัน

init_one_for_all([]) ->
    {ok, {
        #{strategy => one_for_all, intensity => 3, period => 10},
        [
            worker_spec(db, db_conn, []),         %% ถ้า db crash
            worker_spec(cache, cache_server, []), %% cache ก็ต้อง restart
            worker_spec(api, api_server, [])      %% api ก็ต้อง restart
        ]
    }}.

%% rest_for_one
%% ถ้า child crash -> restart child นั้น + children ที่ start ทีหลัง
%% ใช้เมื่อ: children มี dependency ตามลำดับ

init_rest_for_one([]) ->
    {ok, {
        #{strategy => rest_for_one, intensity => 3, period => 10},
        [
            worker_spec(config, config_server, []),   %% order matters!
            worker_spec(db, db_server, []),            %% depends on config
            worker_spec(api, api_server, [])           %% depends on db
        ]
        %% ถ้า db crash -> db + api restart (config ไม่ restart)
    }}.

%% simple_one_for_one (dynamic workers)
%% All children มี spec เดียวกัน, add/remove ได้ตอน runtime
%% ต้องใช้ supervisor:start_child/2 เพื่อเพิ่ม child

init_pool([]) ->
    {ok, {
        #{strategy => simple_one_for_one, intensity => 5, period => 10},
        [
            #{
                id => worker,
                start => {pool_worker, start_link, []},
                restart => temporary,  %% หรือ permanent
                shutdown => 5000,
                type => worker
            }
        ]
    }}.
```

---

## Intensity และ Period

```erlang
%% intensity = จำนวน restart สูงสุดใน period วินาที
%% ถ้า restart เกิน intensity -> supervisor หยุดทำงาน (crash to its parent)

%% Example: intensity => 3, period => 10
%% ยอมให้ restart 3 ครั้งใน 10 วินาที
%% ครั้งที่ 4 ใน 10 วินาทีเดียวกัน = supervisor crash

%% เลือก intensity/period ตาม use case:
%% Development: {3, 10}  - restart ได้บ่อย
%% Production:  {5, 60}  - 5 restarts ใน 1 minute
%% Strict:      {1, 5}   - restart ครั้งเดียวใน 5s

%% เมื่อ supervisor crash -> parent supervisor handle
%% Root supervisor crash -> application stops

%% การ debug supervisor restart loop:
observer:start().  %% GUI: ดู process tree

%% หรือ
supervisor:which_children(my_sup).
%% [{Id, Pid, Type, Modules}]

supervisor:count_children(my_sup).
%% #{specs => N, active => N, supervisors => N, workers => N}
```

---

## Dynamic Children (simple_one_for_one)

```erlang
%% simple_one_for_one สำหรับ dynamic worker pool

-module(worker_sup).
-behaviour(supervisor).

-export([start_link/0, start_worker/1, stop_worker/1, workers/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

%% Dynamic API
start_worker(Args) ->
    supervisor:start_child(?MODULE, [Args]).

stop_worker(Pid) ->
    supervisor:terminate_child(?MODULE, Pid).

workers() ->
    [{Id, Pid} || {Id, Pid, _, _} <- supervisor:which_children(?MODULE),
                  is_pid(Pid)].

init([]) ->
    SupFlags = #{
        strategy => simple_one_for_one,
        intensity => 10,
        period => 60
    },
    ChildSpec = #{
        id => worker,
        start => {my_worker, start_link, []},
        restart => temporary,   %% temporary = ไม่ restart เมื่อ crash
        shutdown => 2000,
        type => worker,
        modules => [my_worker]
    },
    {ok, {SupFlags, [ChildSpec]}}.

%% ใช้งาน
worker_sup:start_link().
{ok, W1} = worker_sup:start_worker([{task, compute}]).
{ok, W2} = worker_sup:start_worker([{task, render}]).
worker_sup:workers().  %% [{worker, <0.100.0>}, {worker, <0.101.0>}]
worker_sup:stop_worker(W1).

%% OTP 18+: via module
supervisor:start_child(?MODULE, [Args]).
%% เพิ่ม Args เข้า ChildSpec start Args
%% {my_worker, start_link, []} + [Args] = my_worker:start_link(Args)
```

---

## Nested Supervisors

```erlang
%% Nested supervisors = supervision tree
%% แต่ละ sub-supervisor ดูแล concern ของตัวเอง

%% root_sup
%%   ├── config_server (gen_server)
%%   ├── db_sup (supervisor)
%%   │     ├── db_pool_sup (supervisor: simple_one_for_one)
%%   │     │     └── db_conn × N
%%   │     └── db_monitor (gen_server)
%%   └── web_sup (supervisor)
%%         ├── http_listener (worker)
%%         └── request_sup (supervisor: simple_one_for_one)
%%               └── request_handler × N

-module(root_sup).
-behaviour(supervisor).

init([]) ->
    {ok, {
        #{strategy => one_for_one, intensity => 3, period => 10},
        [
            #{id => config, start => {config_server, start_link, []},
              type => worker, shutdown => 5000},
            #{id => db_sup, start => {db_sup, start_link, []},
              type => supervisor, shutdown => infinity},
            #{id => web_sup, start => {web_sup, start_link, []},
              type => supervisor, shutdown => infinity}
        ]
    }}.

-module(db_sup).
-behaviour(supervisor).

init([]) ->
    PoolSize = application:get_env(myapp, db_pool_size, 5),
    {ok, {
        #{strategy => one_for_all, intensity => 3, period => 10},
        [
            #{id => db_pool_sup, start => {db_pool_sup, start_link, [PoolSize]},
              type => supervisor, shutdown => infinity},
            #{id => db_monitor, start => {db_monitor, start_link, []},
              type => worker, shutdown => 5000}
        ]
    }}.

-module(db_pool_sup).
-behaviour(supervisor).

start_link(PoolSize) ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, [PoolSize]).

init([PoolSize]) ->
    Workers = [
        #{id => {db_conn, I},
          start => {db_conn, start_link, [I]},
          type => worker,
          shutdown => 2000}
        || I <- lists:seq(1, PoolSize)
    ],
    {ok, {#{strategy => one_for_one, intensity => 5, period => 60}, Workers}}.
```

---

## ตัวอย่างจริง: Web Server Supervisor Tree

```erlang
%% ไฟล์: webserver_sup.erl
-module(webserver_sup).
-behaviour(supervisor).

-export([start_link/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Port = application:get_env(webserver, port, 8080),
    MaxConns = application:get_env(webserver, max_connections, 1000),

    SupFlags = #{
        strategy => rest_for_one,   %% HTTP ต้องการ config ก่อน
        intensity => 5,
        period => 60
    },

    Children = [
        %% 1. Config server (ต้องขึ้นก่อน)
        #{
            id => config_srv,
            start => {config_srv, start_link, []},
            restart => permanent,
            shutdown => 5000,
            type => worker
        },
        %% 2. Connection manager
        #{
            id => conn_mgr,
            start => {conn_manager, start_link, [MaxConns]},
            restart => permanent,
            shutdown => 5000,
            type => worker
        },
        %% 3. Request handler pool supervisor
        #{
            id => handler_pool_sup,
            start => {handler_pool_sup, start_link, []},
            restart => permanent,
            shutdown => infinity,
            type => supervisor
        },
        %% 4. HTTP Acceptor (depends on pool)
        #{
            id => http_acceptor,
            start => {http_acceptor, start_link, [Port]},
            restart => permanent,
            shutdown => 5000,
            type => worker
        }
    ],
    {ok, {SupFlags, Children}}.


%% ไฟล์: handler_pool_sup.erl
-module(handler_pool_sup).
-behaviour(supervisor).

-export([start_link/0, start_handler/1, stop_handler/1]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_handler(Request) ->
    supervisor:start_child(?MODULE, [Request]).

stop_handler(Pid) ->
    supervisor:terminate_child(?MODULE, Pid).

init([]) ->
    {ok, {
        #{strategy => simple_one_for_one, intensity => 100, period => 10},
        [
            #{
                id => request_handler,
                start => {request_handler, start_link, []},
                restart => temporary,   %% one-shot: no auto restart
                shutdown => 30000,      %% 30s to finish request
                type => worker,
                modules => [request_handler]
            }
        ]
    }}.


%% ไฟล์: request_handler.erl
-module(request_handler).
-behaviour(gen_server).

-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

start_link(Request) ->
    gen_server:start_link(?MODULE, [Request], []).

init([Request]) ->
    %% Process request async
    gen_server:cast(self(), {process, Request}),
    {ok, #{request => Request, start_time => erlang:monotonic_time(millisecond)}}.

handle_cast({process, #{method := Method, path := Path} = Req}, State) ->
    Response = route_request(Method, Path, Req),
    send_response(maps:get(socket, Req, undefined), Response),
    {stop, normal, State};   %% done, exit normally

handle_cast(_, State) -> {noreply, State}.
handle_call(_, _, State) -> {reply, ok, State}.
handle_info(_, State) -> {noreply, State}.

terminate(normal, #{start_time := T}) ->
    Duration = erlang:monotonic_time(millisecond) - T,
    io:format("Request handled in ~p ms~n", [Duration]);
terminate(Reason, _State) ->
    io:format("Request handler crash: ~p~n", [Reason]).

code_change(_, State, _) -> {ok, State}.

route_request(<<"GET">>, <<"/health">>, _) ->
    {200, #{status => ok}};
route_request(<<"GET">>, <<"/api/", Rest/binary>>, Req) ->
    api_handler:handle(get, Rest, Req);
route_request(<<"POST">>, <<"/api/", Rest/binary>>, Req) ->
    api_handler:handle(post, Rest, Req);
route_request(_, _, _) ->
    {404, #{error => not_found}}.

send_response(undefined, _) -> ok;
send_response(Socket, {Status, Body}) ->
    Json = jsx:encode(Body),
    Resp = iolist_to_binary([
        "HTTP/1.1 ", integer_to_binary(Status), " OK\r\n",
        "Content-Type: application/json\r\n",
        "Content-Length: ", integer_to_binary(byte_size(Json)), "\r\n",
        "\r\n",
        Json
    ]),
    gen_tcp:send(Socket, Resp).
```

```erlang
%% ตรวจสอบ supervision tree
supervisor:which_children(webserver_sup).
%% [
%%   {http_acceptor, <0.234.0>, worker, [http_acceptor]},
%%   {handler_pool_sup, <0.233.0>, supervisor, [handler_pool_sup]},
%%   {conn_mgr, <0.232.0>, worker, [conn_manager]},
%%   {config_srv, <0.231.0>, worker, [config_srv]}
%% ]

supervisor:count_children(webserver_sup).
%% #{active => 4, specs => 4, supervisors => 1, workers => 3}

%% Force restart a child
supervisor:terminate_child(webserver_sup, http_acceptor).
supervisor:restart_child(webserver_sup, http_acceptor).

%% Delete and re-add with new spec
supervisor:terminate_child(webserver_sup, conn_mgr).
supervisor:delete_child(webserver_sup, conn_mgr).
supervisor:start_child(webserver_sup, NewSpec).
```

---

## สรุป Part 25

| Strategy | เมื่อ child crash | ใช้เมื่อ |
|----------|-------------------|---------|
| one_for_one | restart แค่ child นั้น | children ทำงานอิสระ |
| one_for_all | restart ทุก child | children depend กัน |
| rest_for_one | restart child นั้น + หลังๆ | pipeline dependency |
| simple_one_for_one | restart child นั้น | dynamic worker pool |

| Restart | เมื่อไหร่ restart |
|---------|-----------------|
| permanent | ทุกครั้ง |
| transient | เฉพาะ abnormal exit |
| temporary | ไม่ restart เลย |

---

*[← Part 24: gen_server](part_24_gen_server.md) | [Part 26: gen_statem →](part_26_gen_statem.md)*
