# Part 23: OTP Overview — Open Telecom Platform

## สารบัญ
1. [OTP คืออะไร](#otp-คืออะไร)
2. [OTP Behaviours](#otp-behaviours)
3. [Application Structure](#application-structure)
4. [Supervision Trees](#supervision-trees)
5. [Release Management](#release-management)
6. [OTP Design Principles](#otp-design-principles)
7. [ตัวอย่างจริง: OTP Application Skeleton](#ตัวอย่างจริง-otp-application-skeleton)

---

## OTP คืออะไร

```
OTP = Open Telecom Platform
     ชุด libraries, tools, และ design patterns สำหรับ Erlang

ประกอบด้วย:
1. Standard Library (lists, io, file, etc.)
2. Behaviours (gen_server, supervisor, gen_statem, gen_event, application)
3. Tools (observer, debugger, dialyzer, etc.)
4. Release handling (relup, appup, etc.)

ทำไมต้องใช้ OTP:
- Proven patterns สำหรับ fault-tolerant systems
- Hot code reloading support
- Consistent error handling
- Built-in monitoring and debugging
- Easy to build distributed systems

ไม่ใช้ OTP = reinventing the wheel
```

```erlang
%% ระบบ production ที่ใช้ OTP:
%% WhatsApp: 2M connections on 1 server
%% RabbitMQ: message broker
%% CouchDB: database
%% Riak: distributed key-value
%% Discord: Presence service
%% Ericsson AXD 301: 9-nines availability!

%% OTP Versions
%% R12 - first major production release
%% 17 - maps added
%% 19 - improved supervisor, gen_statem
%% 21 - logger, socket
%% 24 - proc_lib:set_label, partial maps
%% 25 - selects, improved crypto
%% 26 - map comprehensions, JSON module
%% 27 - ...continuing improvements
```

---

## OTP Behaviours

```erlang
%% Behaviour = protocol/interface ที่ process ต้องทำตาม
%% คล้าย interface ใน OOP แต่สำหรับ process lifecycle

%% 1. gen_server - Generic Server
%%    Use case: stateful server, request-reply
%%    Callbacks: init, handle_call, handle_cast, handle_info, terminate, code_change

%% 2. supervisor - Supervision Tree
%%    Use case: monitor and restart child processes
%%    Callbacks: init

%% 3. gen_statem - Generic State Machine
%%    Use case: finite state machines (connection states, protocols)
%%    Callbacks: init, callback_mode, handle_event, terminate, code_change

%% 4. gen_event - Event Manager
%%    Use case: event handling, callbacks, plugins
%%    Callbacks: init, handle_event, handle_call, handle_info, terminate, code_change

%% 5. application - OTP Application
%%    Use case: top-level application lifecycle
%%    Callbacks: start, stop

%% แต่ละ Behaviour มี:
%% - Container process (behaviour module เอง)
%% - Callback module (โค้ดของเรา)
%% - Standard protocol สำหรับ lifecycle

%% ตัวอย่าง gen_server skeleton
-module(my_server).
-behaviour(gen_server).

%% API
-export([start_link/0, get_count/0]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_count() ->
    gen_server:call(?MODULE, get_count).

init([]) ->
    {ok, #{count => 0}}.

handle_call(get_count, _From, #{count := N} = State) ->
    {reply, N, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) -> ok.
code_change(_OldVsn, State, _Extra) -> {ok, State}.
```

---

## Application Structure

```erlang
%% OTP Application structure:
%%
%% myapp/
%%   ├── ebin/              compiled beam files
%%   ├── src/               source files
%%   │   ├── myapp.app.src  application resource file
%%   │   ├── myapp_app.erl  application callback
%%   │   ├── myapp_sup.erl  top supervisor
%%   │   └── myapp_server.erl
%%   ├── include/           header files
%%   ├── priv/              private resources
%%   └── rebar.config       build config

%% Application Resource File (.app.src):
{application, myapp, [
    {description, "My OTP Application"},
    {vsn, "0.1.0"},
    {registered, [myapp_server]},
    {mod, {myapp_app, []}},
    {applications, [
        kernel,
        stdlib
    ]},
    {env, [
        {port, 8080},
        {host, "localhost"}
    ]},
    {modules, [myapp_app, myapp_sup, myapp_server]}
]}.

%% Application callback
-module(myapp_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    myapp_sup:start_link().

stop(_State) ->
    ok.

%% Application startup order
%% kernel -> stdlib -> dependencies -> myapp
%% OTP starts applications in dependency order
```

---

## Supervision Trees

```erlang
%% Supervision Tree คือ hierarchy ของ supervisors และ workers

%% Structure:
%%
%%           Application
%%               |
%%          Top Supervisor
%%         /      |        \
%%    Worker1  Supervisor2  Worker3
%%             /         \
%%          Worker4    Worker5

%% Restart strategies:
%% one_for_one  - restart only crashed child
%% one_for_all  - restart all if any crashes
%% rest_for_one - restart crashed + those started after it
%% simple_one_for_one - dynamic workers of same type

%% Supervisor spec
-module(myapp_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy  => one_for_one,   %% restart strategy
        intensity => 10,             %% max 10 restarts
        period    => 60              %% in 60 seconds
    },
    Children = [
        #{
            id       => myapp_server,
            start    => {myapp_server, start_link, []},
            restart  => permanent,   %% always restart
            shutdown => 5000,        %% wait 5s before kill
            type     => worker,
            modules  => [myapp_server]
        },
        #{
            id       => myapp_cache,
            start    => {myapp_cache, start_link, []},
            restart  => transient,   %% restart only if abnormal
            shutdown => 5000,
            type     => worker,
            modules  => [myapp_cache]
        }
    ],
    {ok, {SupFlags, Children}}.

%% Restart options:
%% permanent  - always restart (default for workers)
%% transient  - restart only if abnormal exit
%% temporary  - never restart

%% Dynamic children (simple_one_for_one)
-module(worker_sup).
-behaviour(supervisor).

init([]) ->
    SupFlags = #{
        strategy  => simple_one_for_one,
        intensity => 100,
        period    => 60
    },
    Children = [
        #{
            id      => worker,
            start   => {myapp_worker, start_link, []},
            restart => temporary,
            shutdown => 5000,
            type    => worker
        }
    ],
    {ok, {SupFlags, Children}}.

%% Add dynamic child
supervisor:start_child(worker_sup, [JobArgs]).

%% Remove dynamic child
supervisor:terminate_child(worker_sup, ChildPid).
supervisor:delete_child(worker_sup, ChildId).
```

---

## Release Management

```erlang
%% Release = self-contained system ที่ run ได้
%% ประกอบด้วย: BEAM VM + applications + config

%% rebar3 release config (rebar.config)
{relx, [
    {release, {myapp, "1.0.0"}, [
        myapp,
        sasl
    ]},
    {dev_mode, false},
    {include_erts, true},      %% รวม Erlang runtime
    {extended_start_script, true}
]}.

%% Build release
%% $ rebar3 release
%% สร้าง _build/default/rel/myapp/

%% Start release
%% $ ./_build/default/rel/myapp/bin/myapp start
%% $ ./_build/default/rel/myapp/bin/myapp console
%% $ ./_build/default/rel/myapp/bin/myapp daemon

%% Hot upgrade
%% 1. เพิ่ม version ใน .app.src
%% 2. สร้าง appup file (อธิบาย changes)
%% 3. สร้าง relup file
%% 4. $ myapp upgrade 2.0.0

%% Application environment
%% ดึง config ที่ตั้งค่าใน .app file
application:get_env(myapp, port).       %% = {ok, 8080} | undefined
application:get_env(myapp, port, 3000). %% = 8080 (default = 3000)

application:set_env(myapp, port, 9090). %% override at runtime
```

---

## OTP Design Principles

```erlang
%% 1. Process == Unit of Concurrency AND Error isolation
%%    แต่ละ independent concern อยู่ใน process แยก

%% 2. Supervision Trees
%%    Hierarchical error recovery
%%    ปัญหาที่ level ล่างสุดจัดการที่ level นั้น
%%    ถ้าจัดการไม่ได้ escalate ขึ้นไป

%% 3. Let It Crash
%%    ไม่ต้อง defensive programming
%%    Supervisor จัดการ restart
%%    State เริ่มใหม่ clean เมื่อ restart

%% 4. Behaviours for Common Patterns
%%    ใช้ gen_server แทน custom message loop
%%    Consistent interface สำหรับ monitoring, debugging

%% 5. Code as Data
%%    Hot code reloading
%%    code_change callback สำหรับ state migration

%% 6. Fault Recovery เป็น First-class
%%    Timeouts ทุกที่
%%    Expected failure modes
%%    Graceful degradation

%% ตัวอย่าง: bad vs good design

%% BAD: ทำทุกอย่างใน 1 process
bad_server() ->
    receive
        Req ->
            Data = db:query(Req),          %% ถ้า db crash -> server crash
            Cached = cache:lookup(Req),    %% ถ้า cache crash -> server crash
            Result = process(Data, Cached),
            bad_server()
    end.

%% GOOD: แยก concerns, let supervisor handle
good_server() ->
    receive
        Req ->
            %% gen_server calls กับ timeout
            Data = gen_server:call(db_server, Req, 5000),
            Cached = gen_server:call(cache_server, Req, 1000),
            Result = process(Data, Cached),
            good_server()
    end.
%% ถ้า db_server crash -> supervisor restart db_server
%% good_server ยังทำงานต่อ (หรือ crash ตาม design)
```

---

## ตัวอย่างจริง: OTP Application Skeleton

```erlang
%% ไฟล์: src/myapp.app.src
{application, myapp, [
    {description, "Example OTP Application"},
    {vsn, "0.1.0"},
    {registered, [myapp_server, myapp_cache]},
    {mod, {myapp_app, []}},
    {applications, [kernel, stdlib, crypto]},
    {env, [
        {port, 8080},
        {db_host, "localhost"},
        {db_port, 5432}
    ]}
]}.
```

```erlang
%% ไฟล์: src/myapp_app.erl
-module(myapp_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    Port = application:get_env(myapp, port, 8080),
    io:format("Starting myapp on port ~p~n", [Port]),
    myapp_sup:start_link().

stop(_State) ->
    io:format("Stopping myapp~n"),
    ok.
```

```erlang
%% ไฟล์: src/myapp_sup.erl
-module(myapp_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    {ok, {
        #{strategy => one_for_one, intensity => 5, period => 10},
        [
            child_spec(myapp_server, worker),
            child_spec(myapp_cache, worker)
        ]
    }}.

child_spec(Module, Type) ->
    #{
        id       => Module,
        start    => {Module, start_link, []},
        restart  => permanent,
        shutdown => 5000,
        type     => Type,
        modules  => [Module]
    }.
```

```erlang
%% ไฟล์: src/myapp_server.erl
-module(myapp_server).
-behaviour(gen_server).
-export([start_link/0, get/1, set/2, delete/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(state, {
    data = #{} :: map(),
    requests = 0 :: non_neg_integer()
}).

%% API
start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get(Key) ->
    gen_server:call(?MODULE, {get, Key}).

set(Key, Value) ->
    gen_server:cast(?MODULE, {set, Key, Value}).

delete(Key) ->
    gen_server:cast(?MODULE, {delete, Key}).

%% Callbacks
init([]) ->
    {ok, #state{}}.

handle_call({get, Key}, _From, #state{data = Data, requests = N} = State) ->
    Value = maps:get(Key, Data, undefined),
    {reply, Value, State#state{requests = N + 1}};

handle_call(stats, _From, State) ->
    {reply, #{requests => State#state.requests, keys => map_size(State#state.data)}, State}.

handle_cast({set, Key, Value}, #state{data = Data} = State) ->
    {noreply, State#state{data = Data#{Key => Value}}};

handle_cast({delete, Key}, #state{data = Data} = State) ->
    {noreply, State#state{data = maps:remove(Key, Data)}}.

handle_info(Info, State) ->
    io:format("Unexpected info: ~p~n", [Info]),
    {noreply, State}.

terminate(Reason, _State) ->
    io:format("Server terminating: ~p~n", [Reason]),
    ok.

code_change(_OldVsn, State, _Extra) ->
    {ok, State}.
```

```erlang
%% ไฟล์: src/myapp_cache.erl  
-module(myapp_cache).
-behaviour(gen_server).
-export([start_link/0, put/3, get/1, invalidate/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-record(entry, {value, expires_at}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

put(Key, Value, TTLSeconds) ->
    gen_server:cast(?MODULE, {put, Key, Value, TTLSeconds}).

get(Key) ->
    gen_server:call(?MODULE, {get, Key}).

invalidate(Key) ->
    gen_server:cast(?MODULE, {invalidate, Key}).

init([]) ->
    %% Schedule periodic cleanup
    erlang:send_after(60000, self(), cleanup),
    {ok, #{}}.

handle_call({get, Key}, _From, Cache) ->
    Now = erlang:monotonic_time(second),
    case maps:get(Key, Cache, undefined) of
        #entry{value = V, expires_at = Exp} when Exp > Now ->
            {reply, {ok, V}, Cache};
        _ ->
            {reply, miss, Cache}
    end.

handle_cast({put, Key, Value, TTL}, Cache) ->
    Exp = erlang:monotonic_time(second) + TTL,
    {noreply, Cache#{Key => #entry{value = Value, expires_at = Exp}}};

handle_cast({invalidate, Key}, Cache) ->
    {noreply, maps:remove(Key, Cache)}.

handle_info(cleanup, Cache) ->
    Now = erlang:monotonic_time(second),
    Clean = maps:filter(fun(_, #entry{expires_at = Exp}) -> Exp > Now end, Cache),
    erlang:send_after(60000, self(), cleanup),
    {noreply, Clean}.

terminate(_Reason, _State) -> ok.
code_change(_OldVsn, State, _Extra) -> {ok, State}.
```

```erlang
%% ใช้งาน
%% Start application
application:start(myapp).

%% Use API
myapp_server:set(name, "Alice").
myapp_server:get(name).   %% = "Alice"

myapp_cache:put(session_123, #{user => alice}, 3600).
myapp_cache:get(session_123).  %% = {ok, #{user => alice}}

%% Inspect supervision tree
supervisor:which_children(myapp_sup).
%% = [{myapp_cache,<0.xxx.0>,worker,[myapp_cache]},
%%    {myapp_server,<0.yyy.0>,worker,[myapp_server]}]

%% Stop application  
application:stop(myapp).
```

---

## สรุป Part 23

| Behaviour | ใช้เมื่อ | Key Feature |
|-----------|---------|------------|
| `gen_server` | Stateful server | call/cast/info |
| `supervisor` | Manage children | Restart strategies |
| `gen_statem` | State machines | Event-driven states |
| `gen_event` | Event dispatch | Plugin-like handlers |
| `application` | App lifecycle | Start/stop OTP app |

OTP คือ foundation ของทุก production Erlang system — เรียนรู้ให้ลึกก่อนเขียน custom solutions

---

*[← Part 22: Message Passing](part_22_message_passing.md) | [Part 24: gen_server →](part_24_gen_server.md)*
