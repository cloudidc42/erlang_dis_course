# Part 88: Observability and Tracing

## สารบัญ
1. [Structured Logging](#logging)
2. [Distributed Tracing with OpenTelemetry](#otel)
3. [Metrics with Prometheus](#prometheus)
4. [Health Checks and Dashboards](#health)
5. [Alerting](#alerting)
6. [ตัวอย่างจริง: Full Observability Stack](#full-stack)

---

## Structured Logging

```erlang
%% Erlang logger with structured metadata

%% app_logger.erl — logging helper
-module(app_logger).
-export([info/2, warning/2, error/2, debug/2]).
-export([with_context/2]).

info(Msg, Meta) ->
    logger:info(Msg, add_common_meta(Meta)).

warning(Msg, Meta) ->
    logger:warning(Msg, add_common_meta(Meta)).

error(Msg, Meta) ->
    logger:error(Msg, add_common_meta(Meta)).

debug(Msg, Meta) ->
    logger:debug(Msg, add_common_meta(Meta)).

add_common_meta(Meta) ->
    Base = #{
        node => node(),
        pid => list_to_binary(pid_to_list(self())),
        timestamp => erlang:system_time(millisecond)
    },
    %% Merge request context from process dictionary
    Context = case get(request_context) of
        undefined -> #{};
        Ctx -> Ctx
    end,
    maps:merge(maps:merge(Base, Context), Meta).

with_context(Context, Fun) ->
    OldCtx = get(request_context),
    put(request_context, maps:merge(OldCtx || #{}, Context)),
    try Fun()
    after
        put(request_context, OldCtx)
    end.

%% Configure JSON formatter in sys.config
logger_config() ->
    [{logger, [
        {handler, default, logger_std_h, #{
            formatter => {logger_formatter, #{
                template => [msg, "\n"],
                single_line => true
            }}
        }},
        %% JSON handler for production
        {handler, json, logger_std_h, #{
            config => #{file => "/var/log/myapp/app.log"},
            formatter => {logger_json_formatter, #{
                template => [time, level, pid, mfa, msg, meta]
            }}
        }}
    ]}].

%% Usage example in request handler
handle_request(Req) ->
    RequestId = cowboy_req:header(<<"x-request-id">>, Req, generate_request_id()),
    UserId = get_user_id(Req),
    
    app_logger:with_context(#{request_id => RequestId, user_id => UserId}, fun() ->
        app_logger:info("Request started", #{
            method => cowboy_req:method(Req),
            path => cowboy_req:path(Req)
        }),
        
        Result = process_request(Req),
        
        app_logger:info("Request completed", #{
            status => maps:get(status, Result)
        }),
        
        Result
    end).
```

---

## Distributed Tracing with OpenTelemetry

```erlang
%% OpenTelemetry tracing using opentelemetry-erlang
%% {deps, [{opentelemetry, "~> 1.3"}, {opentelemetry_api, "~> 1.2"}]}

-module(tracer).
-export([start_span/2, end_span/1, with_span/3, add_attribute/2]).

start_span(SpanName, Attributes) ->
    %% Start span, propagate from current context
    Ctx = otel_ctx:get_current(),
    
    SpanCtx = otel_tracer:start_span(Ctx, opentelemetry:get_tracer(?MODULE),
                                      SpanName, #{attributes => Attributes}),
    
    %% Store in process dict for nested spans
    otel_ctx:attach(otel_tracer:set_current_span(Ctx, SpanCtx)),
    SpanCtx.

end_span(SpanCtx) ->
    otel_span:end_span(SpanCtx, undefined).

with_span(SpanName, Attributes, Fun) ->
    SpanCtx = start_span(SpanName, Attributes),
    try
        Result = Fun(),
        otel_span:set_status(SpanCtx, ?OTEL_STATUS_OK),
        Result
    catch Class:Reason:Stacktrace ->
        otel_span:record_exception(SpanCtx, Class, Reason, Stacktrace, []),
        otel_span:set_status(SpanCtx, ?OTEL_STATUS_ERROR, atom_to_binary(Reason)),
        erlang:raise(Class, Reason, Stacktrace)
    after
        end_span(SpanCtx)
    end.

add_attribute(Key, Value) ->
    Ctx = otel_ctx:get_current(),
    SpanCtx = otel_tracer:current_span_ctx(Ctx),
    otel_span:set_attribute(SpanCtx, Key, Value).

%% HTTP middleware to propagate trace context
-module(tracing_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    %% Extract trace context from incoming headers
    Headers = cowboy_req:headers(Req),
    Propagator = opentelemetry:get_text_map_propagator(),
    Ctx = otel_propagator_text_map:extract(Propagator, Headers,
                                            fun(K, Map) -> maps:get(K, Map, undefined) end),
    
    otel_ctx:attach(Ctx),
    
    %% Start server span
    SpanCtx = tracer:start_span(cowboy_req:path(Req), #{
        <<"http.method">> => cowboy_req:method(Req),
        <<"http.url">> => cowboy_req:uri(Req)
    }),
    
    %% Inject trace ID into response headers
    TraceId = otel_span:trace_id(SpanCtx),
    Req2 = cowboy_req:set_resp_header(<<"x-trace-id">>,
                                       otel_id_generator:format_trace_id(TraceId), Req),
    
    try
        cowboy_middleware:execute(Req2, Env)
    after
        tracer:end_span(SpanCtx)
    end.

%% Database tracing wrapper
traced_db_query(SQL, Params) ->
    tracer:with_span(<<"db.query">>, #{
        <<"db.type">> => <<"postgresql">>,
        <<"db.statement">> => SQL
    }, fun() ->
        case db:execute(SQL, Params) of
            {ok, Result} ->
                tracer:add_attribute(<<"db.rows_affected">>, length(Result)),
                {ok, Result};
            Error ->
                Error
        end
    end).
```

---

## Metrics with Prometheus

```erlang
%% metrics.erl — Prometheus metrics

-module(metrics).
-export([init/0, counter/3, histogram/3, gauge/3, summary/3]).

-define(DEFAULT_BUCKETS, [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]).

init() ->
    %% Declare metrics
    prometheus_counter:new([
        {name, http_requests_total},
        {help, "Total HTTP requests"},
        {labels, [method, path, status_code]}
    ]),
    
    prometheus_histogram:new([
        {name, http_request_duration_seconds},
        {help, "HTTP request duration in seconds"},
        {labels, [method, path]},
        {buckets, ?DEFAULT_BUCKETS}
    ]),
    
    prometheus_gauge:new([
        {name, active_connections},
        {help, "Currently active connections"},
        {labels, [protocol]}
    ]),
    
    prometheus_gauge:new([
        {name, queue_depth},
        {help, "Number of items in queue"},
        {labels, [queue_name]}
    ]),
    
    prometheus_summary:new([
        {name, db_query_duration_seconds},
        {help, "Database query duration"},
        {labels, [operation]},
        {quantiles, [{0.5, 0.05}, {0.9, 0.01}, {0.99, 0.001}]}
    ]).

counter(Name, Labels, Value) ->
    prometheus_counter:inc(Name, labels_list(Labels), Value).

histogram(Name, Labels, Value) ->
    prometheus_histogram:observe(Name, labels_list(Labels), Value).

gauge(Name, Labels, Value) ->
    prometheus_gauge:set(Name, labels_list(Labels), Value).

summary(Name, Labels, Value) ->
    prometheus_summary:observe(Name, labels_list(Labels), Value).

labels_list(Labels) when is_map(Labels) ->
    [V || {_, V} <- maps:to_list(Labels)];
labels_list(Labels) when is_list(Labels) ->
    Labels.

%% Middleware: automatically track HTTP metrics
-module(metrics_middleware).
-behaviour(cowboy_middleware).

execute(Req, Env) ->
    T1 = erlang:monotonic_time(microsecond),
    Method = cowboy_req:method(Req),
    Path = normalize_path(cowboy_req:path(Req)),
    
    metrics:gauge(active_connections, [http], 1),
    
    try
        {ok, Req2, Env2} = cowboy_middleware:execute(Req, Env),
        Duration = (erlang:monotonic_time(microsecond) - T1) / 1_000_000,
        Status = cowboy_req:resp_status(Req2),
        
        metrics:counter(http_requests_total, [Method, Path, Status], 1),
        metrics:histogram(http_request_duration_seconds, [Method, Path], Duration),
        metrics:gauge(active_connections, [http], -1),
        
        {ok, Req2, Env2}
    catch _:_ = E ->
        metrics:counter(http_requests_total, [Method, Path, 500], 1),
        metrics:gauge(active_connections, [http], -1),
        throw(E)
    end.

normalize_path(Path) ->
    %% Replace UUIDs and IDs with placeholders to avoid high cardinality
    re:replace(Path, <<"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}">>,
               <<":uuid">>, [{return, binary}, global]).
```

---

## Health Checks and Dashboards

```erlang
%% health_checker.erl — comprehensive health checks

-module(health_checker).
-export([check_all/0, check/1]).

-define(TIMEOUT_MS, 5000).

check_all() ->
    Checks = [
        {database, fun check_database/0},
        {redis, fun check_redis/0},
        {rabbitmq, fun check_rabbitmq/0},
        {memory, fun check_memory/0},
        {process_count, fun check_process_count/0},
        {scheduler_utilization, fun check_schedulers/0}
    ],
    
    Results = [begin
        T1 = erlang:monotonic_time(millisecond),
        Status = try CheckFun()
                 catch _:Reason -> {unhealthy, Reason}
                 end,
        Duration = erlang:monotonic_time(millisecond) - T1,
        {Name, Status, Duration}
    end || {Name, CheckFun} <- Checks],
    
    Overall = case lists:all(fun({_, {healthy, _}, _}) -> true;
                                (_) -> false end, Results) of
        true -> healthy;
        false -> unhealthy
    end,
    
    #{status => Overall, checks => Results}.

check_database() ->
    case db:execute("SELECT 1", []) of
        {ok, _} -> {healthy, #{latency_ms => measure_db_latency()}};
        {error, Reason} -> {unhealthy, Reason}
    end.

check_redis() ->
    case redis:command(["PING"]) of
        {ok, <<"PONG">>} -> {healthy, #{}};
        Error -> {unhealthy, Error}
    end.

check_rabbitmq() ->
    case amqp_connection:start(#amqp_params_network{}) of
        {ok, Conn} ->
            amqp_connection:close(Conn),
            {healthy, #{}};
        {error, Reason} ->
            {unhealthy, Reason}
    end.

check_memory() ->
    TotalMb = erlang:memory(total) div (1024 * 1024),
    MaxMb = 4096,  %% 4GB limit
    
    case TotalMb < MaxMb * 0.9 of
        true -> {healthy, #{total_mb => TotalMb, limit_mb => MaxMb}};
        false -> {unhealthy, #{reason => memory_high, total_mb => TotalMb}}
    end.

check_process_count() ->
    Count = erlang:system_info(process_count),
    Max = erlang:system_info(process_limit),
    Pct = Count * 100 div Max,
    
    case Pct < 80 of
        true -> {healthy, #{count => Count, max => Max, percent => Pct}};
        false -> {unhealthy, #{reason => too_many_processes, percent => Pct}}
    end.

check_schedulers() ->
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(100),
    Stats = erlang:statistics(scheduler_wall_time),
    
    Utilizations = [A / (A + I) || {_, A, I} <- Stats, A + I > 0],
    AvgUtil = lists:sum(Utilizations) / length(Utilizations),
    
    case AvgUtil < 0.95 of
        true -> {healthy, #{avg_utilization => AvgUtil}};
        false -> {unhealthy, #{reason => cpu_overloaded, utilization => AvgUtil}}
    end.

measure_db_latency() ->
    T1 = erlang:monotonic_time(microsecond),
    db:execute("SELECT 1", []),
    (erlang:monotonic_time(microsecond) - T1) div 1000.
```

---

## ตัวอย่างจริง: Full Observability Stack

```erlang
%% observability.erl — configure full observability stack

-module(observability).
-export([setup/0]).

setup() ->
    setup_logging(),
    setup_tracing(),
    setup_metrics(),
    setup_health_endpoint(),
    ok.

setup_logging() ->
    %% JSON structured logging to stdout
    logger:set_handler_config(default, #{
        formatter => {logger_formatter, #{
            template => [time, " ", level, " ", pid, " ", msg, "\n"],
            single_line => true
        }}
    }),
    
    %% Set log level from environment
    Level = case os:getenv("LOG_LEVEL") of
        "debug" -> debug;
        "warning" -> warning;
        "error" -> error;
        _ -> info
    end,
    logger:set_primary_config(level, Level).

setup_tracing() ->
    %% Configure OTLP exporter
    OtlpEndpoint = os:getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://jaeger:4318"),
    
    application:set_env(opentelemetry_exporter, otlp_endpoint, OtlpEndpoint),
    application:set_env(opentelemetry, tracer_provider, #{
        sampler => {parent_based, #{root => {trace_id_ratio_based, 0.1}}}  %% 10% sampling
    }),
    
    {ok, _} = application:ensure_all_started(opentelemetry).

setup_metrics() ->
    metrics:init(),
    
    %% Start Prometheus HTTP server
    prometheus_httpd:start([
        {port, 9090},
        {path, "/metrics"}
    ]).

setup_health_endpoint() ->
    %% Add health check routes to cowboy
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/health", health_handler, []},
            {"/health/live", liveness_handler, []},
            {"/health/ready", readiness_handler, []},
            {"/metrics", prometheus_cowboy2_handler, []}
        ]}
    ]),
    
    cowboy:start_clear(health_http, [{port, 8090}], #{
        env => #{dispatch => Dispatch}
    }).

%% Liveness: is the process alive?
-module(liveness_handler).
-export([init/2]).
init(Req, State) ->
    {ok, cowboy_req:reply(200, #{}, <<"OK">>, Req), State}.

%% Readiness: can we serve traffic?
-module(readiness_handler).
-export([init/2]).
init(Req, State) ->
    case health_checker:check_all() of
        #{status := healthy} ->
            {ok, cowboy_req:reply(200, #{<<"content-type">> => <<"application/json">>},
                                   jiffy:encode(#{status => <<"healthy">>}), Req), State};
        #{status := unhealthy} = Result ->
            {ok, cowboy_req:reply(503, #{<<"content-type">> => <<"application/json">>},
                                   jiffy:encode(Result), Req), State}
    end.
```

---

## สรุป Part 88

| Pillar | Tool | Config |
|--------|------|--------|
| Logs | Erlang logger + JSON formatter | sys.config |
| Traces | OpenTelemetry + Jaeger/Tempo | OTEL env vars |
| Metrics | Prometheus + Grafana | prometheus.yml |
| Health | Custom + cowboy endpoints | /health, /ready |
| Alerting | Grafana alertmanager | Alert rules |

**Golden rule**: Instrument at the boundary (HTTP handler, DB call, external service) — not deep in business logic.

---

*[← Part 87: Message Patterns](part_87_message_patterns.md) | [Part 89: Production Recipes →](part_89_production_recipes.md)*
