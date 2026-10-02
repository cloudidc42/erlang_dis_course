# Part 71: Erlang Ecosystem — Libraries และ Tools

## สารบัญ
1. [Web Frameworks](#web)
2. [Database Libraries](#database)
3. [Message Queues](#messaging)
4. [Utilities](#utilities)
5. [Testing Tools](#testing)
6. [ตัวอย่างจริง: Technology Stack สำหรับ Production App](#stack)

---

## Web Frameworks

```erlang
%% Cowboy: low-level HTTP server (most popular)
%% Documentation: https://ninenines.eu/docs/en/cowboy/2.10/guide/

%% Router setup
cowboy_routes() ->
    cowboy:start_clear(http, [{port, 8080}], #{
        env => #{dispatch => cowboy_router:compile([
            {'_', [
                {"/api/users", users_handler, []},
                {"/api/users/:id", user_handler, []},
                {"/ws", ws_handler, []},
                {"/health", health_handler, []}
            ]}
        ])}
    }).

%% Elli: minimal, fast HTTP server
%% Better for very high throughput simple APIs

%% ChicagoBoss: MVC framework (less maintained)

%% N2O: WebSocket framework
%% Nitrogen: web framework with components

%% REST framework comparison
framework_comparison() ->
    #{
        cowboy => #{
            throughput => very_high,
            flexibility => high,
            learning_curve => medium,
            use_case => "production APIs, WebSocket"
        },
        elli => #{
            throughput => highest,
            flexibility => low,
            learning_curve => low,
            use_case => "simple REST APIs"
        }
    }.
```

---

## Database Libraries

```erlang
%% PostgreSQL: epgsql (pure Erlang, most popular)
%% {deps, [{epgsql, "4.7.0"}]}

epgsql_example() ->
    {ok, C} = epgsql:connect(#{
        host => "localhost",
        port => 5432,
        username => "user",
        password => "pass",
        database => "mydb",
        timeout => 5000
    }),
    
    {ok, Cols, Rows} = epgsql:equery(C, "SELECT id, name FROM users WHERE id = $1", [1]),
    epgsql:close(C),
    {Cols, Rows}.

%% MySQL: emysql or mysql_otp
mysql_example() ->
    mysql:start_link(mydb, [{user, "root"}, {password, "pass"}, {database, "mydb"}]),
    mysql:query(mydb, "SELECT * FROM users WHERE id = ?", [1]).

%% MongoDB: mongodb-erlang
mongodb_example() ->
    {ok, Connection} = mc_worker_api:connect([{database, <<"mydb">>}]),
    mc_worker_api:find(Connection, <<"users">>, #{<<"_id">> => 1}).

%% Redis: eredis (most popular)
redis_example() ->
    {ok, C} = eredis:start_link(),
    {ok, <<"OK">>} = eredis:q(C, ["SET", "key", "value"]),
    {ok, Value} = eredis:q(C, ["GET", "key"]),
    Value.

%% Connection pooling: poolboy
poolboy_setup() ->
    PoolArgs = [
        {name, {local, db_pool}},
        {worker_module, db_worker},
        {size, 10},
        {max_overflow, 5}
    ],
    poolboy:start_link(PoolArgs, []).

with_db(Fun) ->
    poolboy:transaction(db_pool, Fun).

%% SQLite: sqlite3 (NIF)
sqlite_example() ->
    {ok, Db} = esqlite3:open("mydb.sqlite"),
    ok = esqlite3:exec("CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)", Db),
    ok = esqlite3:exec("INSERT INTO users(name) VALUES(?)", ["Alice"], Db),
    Rows = esqlite3:q("SELECT * FROM users", Db),
    Rows.
```

---

## Utilities

```erlang
%% JSON: jsx (pure Erlang) or jiffy (NIF, faster)
%% {deps, [{jsx, "3.1.0"}]} or {jiffy, "1.1.1"}

json_comparison() ->
    Data = #{<<"name">> => <<"Alice">>, <<"age">> => 30},
    
    %% jsx: pure Erlang, 100% compatible
    Encoded1 = jsx:encode(Data),
    Decoded1 = jsx:decode(Encoded1, [return_maps]),
    
    %% jiffy: NIF, 3-5x faster
    Encoded2 = jiffy:encode(Data),
    Decoded2 = jiffy:decode(Encoded2, [return_maps]),
    
    {Decoded1, Decoded2}.

%% HTTP client libraries
http_clients() ->
    %% httpc: built-in, good enough for simple cases
    httpc:request(get, {"http://example.com", []}, [], []),
    
    %% hackney: feature-rich, connection pooling
    %% {deps, [{hackney, "1.20.1"}]}
    hackney:get("http://example.com", [], <<>>, []),
    
    %% gun: HTTP/2 + WebSocket client
    %% {deps, [{gun, "2.0.1"}]}
    {ok, ConnPid} = gun:open("example.com", 443, #{transport => tls}),
    StreamRef = gun:get(ConnPid, "/"),
    gun:await(ConnPid, StreamRef).

%% UUID generation
uuid_libs() ->
    %% uuid: pure Erlang
    %% {deps, [{uuid, "2.0.4", {pkg, uuid_erl}}]}
    uuid:to_string(uuid:uuid4()),
    
    %% Or roll your own
    custom_uuid().

custom_uuid() ->
    <<A:32, B:16, C:16, D:16, E:48>> = crypto:strong_rand_bytes(16),
    iolist_to_binary(io_lib:format(
        "~8.16.0b-~4.16.0b-4~3.16.0b-~4.16.0b-~12.16.0b",
        [A, B, C band 16#0fff, D band 16#3fff bor 16#8000, E])).

%% Logging: lager (traditional) or OTP 21+ logger
%% {deps, [{lager, "3.9.2"}]}  %% if using old OTP

logger_example() ->
    %% OTP 21+ logger (preferred)
    logger:info("User logged in", #{user_id => 123, ip => "1.2.3.4"}),
    logger:error("DB query failed", #{query => "SELECT...", error => timeout}).

%% Time: calendar module + erlang:system_time
time_utils() ->
    %% Current time
    Now = erlang:system_time(millisecond),
    
    %% Format ISO8601
    {{Year, Month, Day}, {Hour, Min, Sec}} = calendar:local_time(),
    io_lib:format("~4..0b-~2..0b-~2..0bT~2..0b:~2..0b:~2..0b",
                  [Year, Month, Day, Hour, Min, Sec]).
```

---

## Testing Tools

```erlang
%% EUnit: unit testing (built-in)
-module(my_tests).
-include_lib("eunit/include/eunit.hrl").

add_test() ->
    ?assertEqual(3, math:add(1, 2)).

add_test_() ->  %% test generator
    [?_assertEqual(3, math:add(1, 2)),
     ?_assertException(error, badarith, math:add(a, b))].

%% Common Test: integration/system testing
%% Better for complex test scenarios, HTML reports

%% PropEr: property-based testing
%% {deps, [{proper, "1.4.0"}]}

%% Meck: mock library
%% {deps, [{meck, "0.9.2"}]}
meck_example() ->
    meck:new(external_api, [passthrough]),
    meck:expect(external_api, call, fun(_) -> {ok, mocked_response} end),
    
    %% Test code that uses external_api
    Result = my_module:do_something(),
    
    ?assertEqual({ok, processed}, Result),
    meck:validate(external_api),
    meck:unload(external_api).

%% Rebar3 commands
test_commands() ->
    %% Run all tests
    "rebar3 eunit",
    "rebar3 ct",
    %% Coverage report
    "rebar3 cover",
    %% Dialyzer
    "rebar3 dialyzer",
    %% Property tests
    "rebar3 proper".
```

---

## ตัวอย่างจริง: Production Stack

```erlang
%% rebar.config for a production Erlang web application

{erl_opts, [debug_info, {parse_transform, lager_transform}]}.

{deps, [
    %% Web server
    {cowboy, "2.10.0"},
    
    %% JSON
    {jiffy, "1.1.1"},
    
    %% Database
    {epgsql, "4.7.0"},
    {poolboy, "1.5.2"},
    
    %% Redis
    {eredis, "1.7.0"},
    
    %% HTTP client
    {hackney, "1.20.1"},
    
    %% JWT
    {jose, "1.11.9"},
    
    %% Metrics
    {prometheus, "4.10.0"},
    
    %% Distributed tracing
    {opentelemetry, "1.3.0"},
    {opentelemetry_api, "1.2.2"},
    
    %% Logging structured
    %% OTP 21+ logger is built-in, no extra dep needed
    
    %% Testing
    {meck, "0.9.2"},
    {proper, "1.4.0"}
]}.

{relx, [
    {release, {my_app, "1.0.0"}, [
        my_app,
        cowboy,
        jiffy,
        epgsql,
        poolboy,
        eredis,
        hackney,
        jose,
        prometheus,
        opentelemetry,
        sasl  %% crash reports
    ]},
    {sys_config, "config/sys.config"},
    {vm_args, "config/vm.args"},
    {dev_mode, false},
    {include_erts, true},
    {extended_start_script, true}
]}.

{profiles, [
    {test, [
        {deps, [{meck, "0.9.2"}, {proper, "1.4.0"}]},
        {erl_opts, [debug_info, nowarn_export_all]}
    ]},
    {prod, [
        {relx, [{dev_mode, false}, {include_erts, true}]}
    ]}
]}.
```

---

## สรุป Part 71

| Category | Popular Libraries |
|----------|------------------|
| Web server | cowboy, elli |
| PostgreSQL | epgsql, pgsql |
| Redis | eredis, erldis |
| JSON | jiffy (fast), jsx (pure) |
| HTTP client | hackney, gun, httpc |
| Connection pool | poolboy |
| JWT | jose |
| Metrics | prometheus.erl |
| Mocking | meck |
| Properties | proper |
| Config | cuttlefish |

---

*[← Part 70: War Stories](part_70_war_stories.md) | [Part 72: Advanced Patterns →](part_72_advanced_patterns.md)*
