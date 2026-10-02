# Part 97: Erlang Ecosystem Deep Dive

## สารบัญ
1. [Cowboy: HTTP Server Internals](#cowboy)
2. [Ranch: Connection Acceptor](#ranch)
3. [Poolboy: Worker Pool](#poolboy)
4. [Lager vs Logger](#logging)
5. [Rebar3 Plugins](#rebar3)
6. [ตัวอย่างจริง: Complete Hex Package](#hex-package)

---

## Cowboy: HTTP Server Internals

```erlang
%% cowboy architecture:
%%
%%  ranch listener
%%      └─ cowboy_acceptor (N processes)
%%           └─ cowboy_clear / cowboy_tls  (per connection)
%%                └─ cowboy_http / cowboy_http2
%%                     └─ cowboy_stream_h (stream handler chain)
%%                          └─ your handler module

%% Full-featured cowboy_rest handler
-module(api_resource).
-behaviour(cowboy_rest).

-export([
    init/2,
    allowed_methods/2,
    content_types_provided/2,
    content_types_accepted/2,
    is_authorized/2,
    forbidden/2,
    resource_exists/2,
    delete_resource/2,
    to_json/2,
    from_json/2
]).

init(Req, State) ->
    %% Extract path bindings
    Id = cowboy_req:binding(id, Req),
    {cowboy_rest, Req, State#{id => Id}}.

allowed_methods(Req, State) ->
    {[<<"GET">>, <<"POST">>, <<"PUT">>, <<"DELETE">>], Req, State}.

content_types_provided(Req, State) ->
    {[{<<"application/json">>, to_json}], Req, State}.

content_types_accepted(Req, State) ->
    {[{<<"application/json">>, from_json}], Req, State}.

is_authorized(Req, State) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            case jwt:verify(Token) of
                {ok, Claims} ->
                    {true, Req, State#{claims => Claims}};
                {error, _} ->
                    {{false, <<"Bearer">>}, Req, State}
            end;
        _ ->
            {{false, <<"Bearer">>}, Req, State}
    end.

forbidden(Req, #{claims := Claims} = State) ->
    UserId = maps:get(<<"sub">>, Claims),
    ResourceId = maps:get(id, State),
    
    case acl:can(UserId, read, ResourceId) of
        true -> {false, Req, State};
        false -> {true, Req, State}
    end.

resource_exists(Req, #{id := Id} = State) ->
    case db:get(Id) of
        {ok, Resource} -> {true, Req, State#{resource => Resource}};
        not_found -> {false, Req, State}
    end.

to_json(Req, #{resource := Resource} = State) ->
    Body = jiffy:encode(Resource),
    {Body, Req, State}.

from_json(Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case jiffy:decode(Body, [return_maps]) of
        Data when is_map(Data) ->
            Id = maps:get(id, State),
            ok = db:update(Id, Data),
            {true, Req2, State};
        _ ->
            {false, Req2, State}
    end.

delete_resource(Req, #{id := Id} = State) ->
    ok = db:delete(Id),
    {true, Req, State}.

%% WebSocket handler
-module(ws_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, _) ->
    %% Upgrade to WebSocket with 60s idle timeout
    {cowboy_websocket, Req, #{}, #{idle_timeout => 60000}}.

websocket_init(State) ->
    %% Subscribe to updates
    pubsub:subscribe(updates),
    {ok, State}.

websocket_handle({text, Data}, State) ->
    Msg = jiffy:decode(Data, [return_maps]),
    handle_message(Msg, State);
websocket_handle({ping, _}, State) ->
    {ok, State}.  %% cowboy auto-replies to pings

websocket_info({update, Data}, State) ->
    Payload = jiffy:encode(Data),
    {reply, {text, Payload}, State}.

terminate(_Reason, _Req, _State) ->
    pubsub:unsubscribe(updates).

handle_message(#{<<"type">> := <<"subscribe">>, <<"topic">> := Topic}, State) ->
    pubsub:subscribe(Topic),
    {ok, State#{subscriptions => [Topic | maps:get(subscriptions, State, [])]}};
handle_message(#{<<"type">> := <<"ping">>}, State) ->
    {reply, {text, <<"{\"type\":\"pong\"}">>}, State};
handle_message(_Unknown, State) ->
    {ok, State}.

%% SSE (Server-Sent Events) handler
-module(sse_handler).
-export([init/2]).

init(Req, State) ->
    Req2 = cowboy_req:stream_reply(200, #{
        <<"content-type">> => <<"text/event-stream">>,
        <<"cache-control">> => <<"no-cache">>,
        <<"x-accel-buffering">> => <<"no">>
    }, Req),
    
    loop(Req2, State).

loop(Req, State) ->
    receive
        {event, Data} ->
            Payload = [<<"data: ">>, jiffy:encode(Data), <<"\n\n">>],
            cowboy_req:stream_body(Payload, nofin, Req),
            loop(Req, State);
        
        stop ->
            cowboy_req:stream_body(<<>>, fin, Req),
            {ok, Req, State}
    after 30000 ->
        %% Keep-alive comment
        cowboy_req:stream_body(<<": keepalive\n\n">>, nofin, Req),
        loop(Req, State)
    end.
```

---

## Ranch: Connection Acceptor

```erlang
%% Ranch is cowboy's underlying connection acceptor
%% You can use it directly for non-HTTP protocols

%% Custom TCP protocol with ranch
-module(my_protocol).
-behaviour(ranch_protocol).
-export([start_link/3]).

start_link(Ref, Transport, Opts) ->
    Pid = spawn_link(fun() -> init(Ref, Transport, Opts) end),
    {ok, Pid}.

init(Ref, Transport, Opts) ->
    %% Take ownership of the socket
    {ok, Socket} = ranch:handshake(Ref),
    Transport:setopts(Socket, [{active, once}, {packet, 4}]),
    loop(Socket, Transport, Opts).

loop(Socket, Transport, Opts) ->
    receive
        {tcp, Socket, Data} ->
            handle_data(Data, Socket, Transport),
            Transport:setopts(Socket, [{active, once}]),
            loop(Socket, Transport, Opts);
        
        {tcp_closed, Socket} ->
            ok;
        
        {tcp_error, Socket, _Reason} ->
            Transport:close(Socket)
    after 60000 ->
        Transport:close(Socket)
    end.

handle_data(Data, Socket, Transport) ->
    Response = process(Data),
    Transport:send(Socket, Response).

process(Data) ->
    %% Echo for demo
    Data.

%% Start ranch listener
start_ranch_listener() ->
    ranch:start_listener(my_listener,
        ranch_tcp,
        #{port => 9090, num_acceptors => 100},
        my_protocol,
        []
    ).
```

---

## Poolboy: Worker Pool

```erlang
%% poolboy: fixed-size worker pool

%% Pool configuration in supervisor
db_pool_spec() ->
    PoolOpts = [
        {name, {local, db_pool}},
        {worker_module, db_worker},
        {size, 20},           %% normal pool size
        {max_overflow, 10},   %% allow up to 10 extra workers under load
        {strategy, lifo}      %% last-in-first-out (warmer connections)
    ],
    WorkerArgs = [#{
        host => "localhost",
        port => 5432,
        database => "mydb"
    }],
    poolboy:child_spec(db_pool, PoolOpts, WorkerArgs).

%% Worker module
-module(db_worker).
-behaviour(poolboy_worker).
-behaviour(gen_server).
-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link(Args) ->
    gen_server:start_link(?MODULE, Args, []).

init(#{host := Host, port := Port, database := DB}) ->
    {ok, Conn} = epgsql:connect(Host, "user", "pass", #{
        port => Port,
        database => DB,
        timeout => 5000
    }),
    {ok, #{conn => Conn}}.

handle_call({query, SQL, Params}, _From, #{conn := Conn} = State) ->
    Result = epgsql:equery(Conn, SQL, Params),
    {reply, Result, State}.

%% Using the pool
query(SQL, Params) ->
    poolboy:transaction(db_pool, fun(Worker) ->
        gen_server:call(Worker, {query, SQL, Params}, 10000)
    end).

%% With timeout handling
query_with_timeout(SQL, Params, TimeoutMs) ->
    case poolboy:checkout(db_pool, true, TimeoutMs) of
        full ->
            {error, pool_full};
        Worker ->
            try
                gen_server:call(Worker, {query, SQL, Params}, TimeoutMs)
            after
                poolboy:checkin(db_pool, Worker)
            end
    end.

%% Monitor pool health
pool_status() ->
    #{
        workers => poolboy:status(db_pool)
    }.
```

---

## Lager vs Logger

```erlang
%% Modern Erlang uses the built-in logger (OTP 21+)
%% Legacy code uses lager

%% === Modern: OTP logger ===

%% Configuration in sys.config
logger_config() ->
    [{kernel, [
        {logger, [
            %% Console handler
            {handler, default, logger_std_h, #{
                level => info,
                formatter => {logger_formatter, #{
                    template => [time, " [", level, "] ", msg, "\n"]
                }}
            }},
            
            %% File handler
            {handler, file, logger_std_h, #{
                level => debug,
                config => #{file => "/var/log/myapp.log"},
                formatter => {logger_formatter, #{
                    template => [time, " [", level, "] ", file, ":", line, " ", msg, "\n"]
                }}
            }}
        ]},
        {logger_level, info}
    ]}].

%% Structured logging
log_request(Method, Path, Status, DurationMs) ->
    logger:info("HTTP request", #{
        method => Method,
        path => Path,
        status => Status,
        duration_ms => DurationMs
    }).

%% Custom formatter for JSON output (Kubernetes/ELK friendly)
-module(json_formatter).
-export([format/2]).

format(#{level := Level, msg := {string, Msg}, meta := Meta}, _Config) ->
    Map = #{
        timestamp => format_time(maps:get(time, Meta)),
        level => Level,
        message => Msg,
        pid => maps:get(pid, Meta, undefined)
    },
    [jiffy:encode(Map), "\n"].

format_time(T) ->
    calendar:system_time_to_rfc3339(erlang:convert_time_unit(T, microsecond, second)).

%% === Legacy: Lager (for older codebases) ===
lager_usage() ->
    lager:debug("Debug message: ~p", [some_value]),
    lager:info("Request processed"),
    lager:warning("Disk usage high: ~p%", [85]),
    lager:error("Failed to connect: ~p", [{error, econnrefused}]).
```

---

## Rebar3 Plugins

```erlang
%% rebar.config — full production configuration

{erl_opts, [
    debug_info,
    {parse_transform, lager_transform},  %% if using lager
    {d, 'APP_VSN', "1.2.3"}
]}.

{deps, [
    {cowboy, "2.10.0"},
    {jiffy, "1.1.1"},
    {epgsql, "4.7.0"},
    {poolboy, "1.5.2"},
    {eredis, "1.7.1"},
    {hackney, "1.20.1"},
    {prometheus, "4.10.0"},
    {opentelemetry, "1.3.0"},
    {proper, "1.4.0", [test]},
    {meck, "0.9.2", [test]}
]}.

{relx, [
    {release, {myapp, "1.0.0"}, [myapp, sasl]},
    {mode, prod},
    {sys_config, "./config/sys.config"},
    {vm_args, "./config/vm.args"},
    {include_erts, true},
    {extended_start_script, true}
]}.

{profiles, [
    {dev, [
        {erl_opts, [debug_info]},
        {relx, [{mode, dev}, {include_erts, false}]}
    ]},
    {test, [
        {deps, [{meck, "0.9.2"}, {proper, "1.4.0"}]},
        {erl_opts, [debug_info, export_all]}
    ]},
    {prod, [
        {erl_opts, [no_debug_info]},
        {relx, [{mode, prod}]}
    ]}
]}.

{ct_opts, [
    {dir, "test"},
    {logdir, "_build/test/logs"}
]}.

{cover_opts, [verbose]}.
{cover_enabled, true}.

{dialyzer, [
    {warnings, [unknown, underspecs]},
    {plt_apps, top_level_deps}
]}.

{xref_checks, [
    exports_not_used,
    undefined_function_calls
]}.

%% Useful rebar3 commands:
%%   rebar3 compile
%%   rebar3 shell
%%   rebar3 ct
%%   rebar3 eunit
%%   rebar3 proper
%%   rebar3 dialyzer
%%   rebar3 xref
%%   rebar3 cover
%%   rebar3 release
%%   rebar3 as prod release
%%   rebar3 as prod tar
```

---

## ตัวอย่างจริง: Complete Hex Package

```erlang
%% How to publish a library to Hex.pm

%% 1. rebar.config
hex_rebar_config() ->
    "
{erl_opts, [debug_info]}.

{project_plugins, [rebar3_hex]}.

{hex, [
    {doc, #{provider => ex_doc}}
]}.

{ex_doc, [
    {source_url, <<\"https://github.com/myorg/mylib\">>},
    {extras, [\"README.md\", \"CHANGELOG.md\"]},
    {main, <<\"readme\">>}
]}.
    ".

%% 2. Library structure
%%
%% mylib/
%%   src/
%%     mylib.app.src
%%     mylib.erl          -- public API
%%     mylib_internal.erl -- internal, no docs needed
%%   test/
%%     mylib_tests.erl
%%   README.md
%%   CHANGELOG.md
%%   LICENSE
%%   rebar.config

%% 3. mylib.app.src
app_src() ->
    "
{application, mylib, [
    {description, \"My awesome Erlang library\"},
    {vsn, \"1.0.0\"},
    {registered, []},
    {applications, [kernel, stdlib, crypto]},
    {env, []},
    {modules, []},
    {licenses, [\"Apache-2.0\"]},
    {links, [{\"GitHub\", \"https://github.com/myorg/mylib\"}]}
]}.
    ".

%% 4. Public API module
-module(mylib).

%% @doc Short description of the module.
%%
%% == Examples ==
%%
%% ```
%% mylib:do_something(42).
%% '''

-export([do_something/1, version/0]).

%% @doc Do something with Value.
-spec do_something(term()) -> term().
do_something(Value) ->
    mylib_internal:process(Value).

%% @doc Return library version.
-spec version() -> binary().
version() ->
    {ok, Vsn} = application:get_key(mylib, vsn),
    list_to_binary(Vsn).

%% 5. Publish commands:
%%   rebar3 hex publish
%%   rebar3 hex publish --dry-run  %% verify without publishing
%%   rebar3 hex docs               %% publish documentation
```

---

## สรุป Part 97

| Library | Purpose | Version |
|---------|---------|---------|
| cowboy | HTTP/WS server | 2.10+ |
| ranch | TCP acceptors | 2.1+ |
| poolboy | Worker pools | 1.5+ |
| epgsql | PostgreSQL driver | 4.7+ |
| eredis | Redis client | 1.7+ |
| hackney | HTTP client | 1.20+ |
| jiffy | JSON (NIF) | 1.1+ |
| prometheus | Metrics | 4.10+ |
| opentelemetry | Tracing | 1.3+ |
| proper | Property testing | 1.4+ |

---

*[← Part 96: World-Class Patterns](part_96_world_class_patterns.md) | [Part 98: Financial Systems →](part_98_financial_systems.md)*
