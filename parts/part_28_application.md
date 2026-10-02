# Part 28: Application Behaviour

## สารบัญ
1. [Application คืออะไร](#application-คืออะไร)
2. [.app.src File](#appsrc-file)
3. [Application Callbacks](#application-callbacks)
4. [Environment Variables](#environment-variables)
5. [Included Applications](#included-applications)
6. [ตัวอย่างจริง: Production Application](#ตัวอย่างจริง-production-application)

---

## Application คืออะไร

```erlang
%% Application = unit ของ deployment ใน Erlang/OTP
%% แต่ละ app มี:
%% - .app file (metadata: name, vsn, modules, deps)
%% - Application callback module (start/stop lifecycle)
%% - Root supervisor (process tree)
%% - Environment configuration

%% Erlang node เป็น collection ของ applications:
%%   kernel (ต้องมีเสมอ)
%%   stdlib (ต้องมีเสมอ)
%%   sasl (system application support library)
%%   my_app (your app)
%%   another_app (dependency)

%% Application lifecycle:
%%   loaded -> started -> running -> stopped

%% ดู running apps
application:which_applications().
%% [{my_app, "My Application", "1.0.0"}, {kernel, ...}, ...]

application:start(my_app).
application:stop(my_app).
application:ensure_all_started(my_app).  %% start ทุก deps ก่อน
```

---

## .app.src File

```erlang
%% ไฟล์: src/my_app.app.src
{application, my_app,
 [
  {description, "My Erlang Application"},
  {vsn, "1.0.0"},

  %% Registered names (สำหรับ tool analysis)
  {registered, [my_app_sup, my_app_server]},

  %% Dependencies (ต้อง start ก่อน)
  {applications,
   [kernel,
    stdlib,
    crypto,        %% ถ้าใช้ crypto
    inets,         %% ถ้าใช้ HTTP client
    jsx,           %% JSON lib
    hackney        %% HTTP client
   ]},

  %% Modules ทั้งหมด (rebar3 generate อัตโนมัติ)
  {modules, []},

  %% Application callback module
  {mod, {my_app_app, []}},

  %% Environment defaults
  {env,
   [
    {port, 8080},
    {pool_size, 10},
    {db_host, "localhost"},
    {db_port, 5432},
    {log_level, info}
   ]},

  %% Maximum restart intensity for top-level supervisor
  {maxT, infinity},

  %% Extra metadata
  {maintainers, ["team@example.com"]},
  {licenses, ["Apache 2.0"]},
  {links, [{"GitHub", "https://github.com/..."}]}
 ]}.

%% rebar3 build จะสร้าง _build/default/lib/my_app/ebin/my_app.app
```

---

## Application Callbacks

```erlang
%% ไฟล์: src/my_app_app.erl
-module(my_app_app).
-behaviour(application).

-export([start/2, stop/1]).

%% start/2 ถูกเรียกเมื่อ application:start/1
%% StartType:
%%   normal - standard start
%%   {takeover, Node} - taking over from another node
%%   {failover, Node} - failing over from crashed node
start(normal, _StartArgs) ->
    my_app_sup:start_link();

start({takeover, _Node}, _) ->
    my_app_sup:start_link();

start({failover, _Node}, _) ->
    my_app_sup:start_link().

%% stop/1 ถูกเรียกเมื่อ application:stop/1
%% State = ค่าที่คืนจาก start/2 (thunk returned from start_link)
stop(_State) ->
    ok.

%% Optional callbacks:
%% prep_stop/1 - ก่อน supervisor tree shutdown
prep_stop(State) ->
    %% Drain queues, finish requests, etc.
    drain_pending_requests(),
    State.

%% start_phases/3 - สำหรับ phased startup (หลาย apps)
start_phase(Phase, StartType, PhaseArgs) ->
    case Phase of
        init -> do_init(StartType, PhaseArgs);
        load_config -> do_load_config();
        connect_db -> do_connect_db()
    end.
```

---

## Environment Variables

```erlang
%% ใน .app.src: {env, [{key, default_value}]}

%% อ่าน env ใน code
application:get_env(my_app, port).
%% {ok, 8080} หรือ undefined

application:get_env(my_app, port, 3000).
%% 8080 หรือ 3000 (ถ้าไม่ได้ set)

%% Set at runtime
application:set_env(my_app, port, 9090).
application:set_env(my_app, port, 9090, [{persistent, true}]).

%% Unset
application:unset_env(my_app, port).

%% Get all env
application:get_all_env(my_app).
%% [{port, 8080}, {pool_size, 10}, ...]

%% Override จาก command line:
%% erl -my_app port 9090 -my_app pool_size 20

%% sys.config file (ทั่วไปกว่า):
%% [
%%   {my_app, [
%%     {port, 9090},
%%     {db_host, "prod-db.example.com"}
%%   ]},
%%   {lager, [
%%     {log_root, "/var/log/my_app"}
%%   ]}
%% ]
%%
%% เปิดใช้:
%% erl -config sys

%% vm.args (VM parameters):
%% -name myapp@hostname
%% -cookie secret_cookie
%% +K true
%% +A 64

%% config helper module
-module(config).

-export([get/2, get/3, require/2]).

get(Key, Default) ->
    application:get_env(?APP_NAME, Key, Default).

get(App, Key, Default) ->
    application:get_env(App, Key, Default).

require(App, Key) ->
    case application:get_env(App, Key) of
        {ok, Value} -> Value;
        undefined ->
            error({missing_config, App, Key})
    end.
```

---

## Included Applications

```erlang
%% Included applications: app ที่ included ใน release แต่ไม่ auto-start
%% ต้องเรียก application:start/1 เองจาก parent app

%% ใน .app.src ของ parent:
{included_applications, [child_app]}.

%% ใน parent's start/2:
start(normal, []) ->
    application:start(child_app),  %% start manually
    parent_sup:start_link().

%% Application dependencies graph:
%%   my_app
%%     ├── [depends on] kernel, stdlib, crypto
%%     └── [includes] plugin_a, plugin_b

%% ทำไมใช้ included?
%% - ต้องการ control การ start ของ dependency
%% - dependency ต้องถูก configure ก่อน start
%% - Distributed failover scenarios
```

---

## ตัวอย่างจริง: Production Application

```erlang
%% ไฟล์: src/store_app.app.src
{application, store_app,
 [
  {description, "E-Commerce Store Backend"},
  {vsn, "2.1.0"},
  {registered, [store_sup, store_db_pool, store_cache, store_api]},
  {applications, [kernel, stdlib, crypto, ssl, inets, jsx, epgsql]},
  {modules, []},
  {mod, {store_app, []}},
  {env, [
    {http_port, 8080},
    {db_pool_size, 10},
    {cache_ttl, 300},
    {log_level, info},
    {metrics_enabled, true}
  ]}
 ]}.


%% ไฟล์: src/store_app.erl
-module(store_app).
-behaviour(application).

-export([start/2, stop/1, prep_stop/1]).

start(normal, _Args) ->
    %% Validate required config
    validate_config(),

    %% Initialize metrics
    init_metrics(),

    %% Start root supervisor
    case store_sup:start_link() of
        {ok, Pid} ->
            io:format("Store App started on port ~p~n",
                      [application:get_env(store_app, http_port, 8080)]),
            {ok, Pid};
        {error, Reason} ->
            {error, Reason}
    end.

prep_stop(State) ->
    %% Graceful shutdown: stop accepting new requests
    io:format("Preparing to stop store_app...~n"),
    store_api:stop_accepting(),
    timer:sleep(2000),  %% wait for in-flight requests
    State.

stop(_State) ->
    io:format("Store App stopped~n"),
    ok.

validate_config() ->
    Required = [http_port, db_pool_size],
    Missing = [K || K <- Required,
                    application:get_env(store_app, K) =:= undefined],
    case Missing of
        [] -> ok;
        _ -> error({missing_required_config, Missing})
    end.

init_metrics() ->
    case application:get_env(store_app, metrics_enabled, false) of
        true ->
            %% Initialize counters, histograms, etc.
            metrics:init([
                {counter, http_requests_total},
                {counter, db_queries_total},
                {histogram, request_duration_ms}
            ]);
        false ->
            ok
    end.


%% ไฟล์: src/store_sup.erl
-module(store_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    DbConfig = #{
        host => application:get_env(store_app, db_host, "localhost"),
        port => application:get_env(store_app, db_port, 5432),
        pool_size => application:get_env(store_app, db_pool_size, 10)
    },
    HttpPort = application:get_env(store_app, http_port, 8080),

    SupFlags = #{
        strategy => rest_for_one,  %% API needs DB and Cache first
        intensity => 5,
        period => 60
    },

    Children = [
        %% DB Pool (starts first)
        #{id => db_pool,
          start => {store_db_pool, start_link, [DbConfig]},
          type => supervisor,
          shutdown => infinity},

        %% Cache (starts second)
        #{id => cache,
          start => {store_cache, start_link, []},
          type => worker,
          shutdown => 5000},

        %% Business logic workers
        #{id => inventory,
          start => {store_inventory, start_link, []},
          type => worker,
          shutdown => 5000},

        %% HTTP API (starts last, depends on all above)
        #{id => api,
          start => {store_api, start_link, [HttpPort]},
          type => worker,
          shutdown => 10000}
    ],

    {ok, {SupFlags, Children}}.


%% ไฟล์: src/store_db_pool.erl
-module(store_db_pool).
-behaviour(supervisor).

-export([start_link/1, checkout/0, checkin/1, with_conn/1]).
-export([init/1]).

start_link(Config) ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, [Config]).

checkout() ->
    case ets:match_object(db_pool, {conn, '_', available}) of
        [{conn, Pid, available} | _] ->
            ets:insert(db_pool, {conn, Pid, busy}),
            {ok, Pid};
        [] ->
            {error, pool_exhausted}
    end.

checkin(Pid) ->
    ets:insert(db_pool, {conn, Pid, available}).

with_conn(Fun) ->
    case checkout() of
        {ok, Conn} ->
            try
                Result = Fun(Conn),
                checkin(Conn),
                {ok, Result}
            catch
                E:R ->
                    checkin(Conn),
                    {error, {E, R}}
            end;
        Error ->
            Error
    end.

init([#{host := H, port := P, pool_size := Size}]) ->
    ets:new(db_pool, [named_table, public, bag]),

    Workers = [
        #{id => {db_conn, I},
          start => {store_db_conn, start_link, [H, P, I]},
          type => worker,
          shutdown => 2000}
        || I <- lists:seq(1, Size)
    ],

    {ok, {
        #{strategy => one_for_one, intensity => 5, period => 60},
        Workers
    }}.


%% Rebar3 release config: rebar.config
%% {relx, [
%%   {release, {store_app, "2.1.0"}, [store_app, sasl]},
%%   {mode, dev},
%%   {sys_config, "config/sys.config"},
%%   {vm_args, "config/vm.args"},
%%   {extended_start_script, true}
%% ]}.

%% Build release:
%% $ rebar3 release
%% $ ./_build/default/rel/store_app/bin/store_app start
%% $ ./_build/default/rel/store_app/bin/store_app status
%% $ ./_build/default/rel/store_app/bin/store_app stop
```

---

## สรุป Part 28

| Callback | Called when |
|----------|-------------|
| `start/2` | `application:start/1` |
| `stop/1` | `application:stop/1` |
| `prep_stop/1` | Before supervisor shutdown |

| Function | Purpose |
|----------|---------|
| `application:start/1` | Start application |
| `application:stop/1` | Stop application |
| `application:ensure_all_started/1` | Start with all deps |
| `application:get_env/2,3` | Read config |
| `application:set_env/3,4` | Set config at runtime |
| `application:which_applications/0` | List running apps |

---

*[← Part 27: gen_event](part_27_gen_event.md) | [Part 29: Rebar3 →](part_29_rebar3.md)*
