# Part 40: Observability — Logging, Metrics, Tracing

## สารบัญ
1. [Logger — OTP 21+ Built-in](#logger--otp-21-built-in)
2. [Metrics กับ Prometheus](#metrics-กับ-prometheus)
3. [Health Checks](#health-checks)
4. [Distributed Tracing (OpenTelemetry)](#distributed-tracing-opentelemetry)
5. [Crash Reports and SASL](#crash-reports-and-sasl)
6. [ตัวอย่างจริง: Observability Stack](#ตัวอย่างจริง-observability-stack)

---

## Logger — OTP 21+ Built-in

```erlang
%% logger: OTP 21+ built-in logging framework
%% แทนที่ error_logger เดิม

%% Basic logging
logger:debug("Debug message").
logger:info("User ~p logged in", [UserId]).
logger:warning("High memory: ~p MB", [MemMB]).
logger:error("Database error: ~p", [Reason]).
logger:critical("System failure: ~p", [Reason]).

%% Structured logging (map metadata)
logger:info("Request completed", #{
    method => <<"GET">>,
    path => <<"/api/users">>,
    status => 200,
    duration_ms => 45,
    user_id => 123
}).

%% Macro-based (includes module, line, function)
-include_lib("kernel/include/logger.hrl").

?LOG_INFO("Started processing"),
?LOG_ERROR("Failed: ~p", [Reason]),
?LOG_WARNING(#{event => rate_limited, user => UserId}).

%% Logger configuration (sys.config)
[{kernel, [
    {logger, [
        {handler, default, logger_std_h, #{
            level => info,
            formatter => {logger_formatter, #{
                template => [time, " ", level, " [", pid, "] ", msg, "\n"],
                time_designator => $T,
                time_offset => "Z"
            }},
            config => #{
                type => {file, "/var/log/myapp/app.log"},
                max_no_bytes => 10485760,    %% 10 MB
                max_no_files => 5            %% 5 rotated files
            }
        }}
    ]}
}}].

%% Add custom handler
logger:add_handler(my_handler, my_log_handler, #{
    level => error,
    config => #{slack_webhook => "https://..."}
}).

%% Custom handler module
-module(my_log_handler).
-behaviour(logger_handler).

log(#{level := error, msg := {report, Map}} = Event, _Config) ->
    notify_slack(format_error(Map)),
    ok;
log(_Event, _Config) ->
    ok.

%% Set log level at runtime
logger:set_primary_config(level, debug).
logger:set_handler_config(default, level, warning).

%% Filter logs by module
logger:add_primary_filter(module_filter, {
    fun(#{meta := #{module := M}}, Modules) ->
        case lists:member(M, Modules) of
            true -> log;
            false -> stop
        end
    end,
    [my_module, another_module]
}).
```

---

## Metrics กับ Prometheus

```erlang
%% prometheus.erl: Prometheus metrics for Erlang
%% deps: {prometheus, "4.10.0"}

%% Counter: monotonically increasing
prometheus_counter:new([
    {name, http_requests_total},
    {help, "Total HTTP requests"},
    {labels, [method, status, path]}
]).

%% Increment
prometheus_counter:inc(http_requests_total, [<<"GET">>, 200, <<"/api/users">>]).
prometheus_counter:inc(http_requests_total, [<<"POST">>, 201, <<"/api/users">>], 1).

%% Gauge: can go up and down
prometheus_gauge:new([
    {name, active_connections},
    {help, "Currently active connections"}
]).
prometheus_gauge:inc(active_connections).
prometheus_gauge:dec(active_connections).
prometheus_gauge:set(active_connections, 42).

%% Histogram: observe values
prometheus_histogram:new([
    {name, request_duration_seconds},
    {buckets, [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]},
    {help, "HTTP request duration"},
    {labels, [method, path]}
]).
prometheus_histogram:observe(request_duration_seconds, [<<"GET">>, <<"/api">>], 0.045).

%% Summary
prometheus_summary:new([
    {name, request_size_bytes},
    {help, "HTTP request size"},
    {quantiles, [{0.5, 0.05}, {0.9, 0.01}, {0.99, 0.001}]}
]).

%% Expose metrics endpoint (Cowboy handler)
-module(metrics_handler).
-export([init/2]).

init(Req, State) ->
    Body = prometheus_text_format:format(),
    Req2 = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain; version=0.0.4">>},
        Body, Req),
    {ok, Req2, State}.

%% VM metrics (automatic)
prometheus_vm_memory_collector:register().
prometheus_vm_system_info_collector:register().
prometheus_vm_statistics_collector:register().
prometheus_vm_msacc_collector:register().

%% Default metrics exposed:
%% erlang_vm_memory_bytes_total{kind="processes"}
%% erlang_vm_process_count
%% erlang_vm_scheduler_utilization{scheduler_id="1"}
%% etc.
```

---

## Health Checks

```erlang
%% Health check endpoint สำหรับ Kubernetes/load balancer

-module(health_handler).
-export([init/2]).

init(Req, State) ->
    case check_health() of
        {ok, Details} ->
            Req2 = cowboy_req:reply(200,
                #{<<"content-type">> => <<"application/json">>},
                jsx:encode(#{status => <<"ok">>, details => Details}),
                Req),
            {ok, Req2, State};
        {error, Details} ->
            Req2 = cowboy_req:reply(503,
                #{<<"content-type">> => <<"application/json">>},
                jsx:encode(#{status => <<"unhealthy">>, details => Details}),
                Req),
            {ok, Req2, State}
    end.

check_health() ->
    Checks = [
        {database, check_db()},
        {cache, check_cache()},
        {memory, check_memory()}
    ],
    case [K || {K, {error, _}} <- Checks] of
        [] -> {ok, maps:from_list([{K, ok} || {K, ok} <- Checks])};
        Failed -> {error, #{failed => Failed}}
    end.

check_db() ->
    case db_pool:query("SELECT 1", []) of
        {ok, _, _} -> ok;
        {error, R} -> {error, R}
    end.

check_cache() ->
    case catch kv_store:get(health_check_key) of
        {'EXIT', _} -> {error, cache_unavailable};
        _ -> ok
    end.

check_memory() ->
    Mem = erlang:memory(total),
    MaxMem = 1024 * 1024 * 1024,  %% 1 GB threshold
    case Mem < MaxMem of
        true -> ok;
        false -> {error, #{used_bytes => Mem, max_bytes => MaxMem}}
    end.

%% Readiness vs Liveness
%% /health/live   - is process running? (restart if fails)
%% /health/ready  - ready to serve traffic? (remove from LB if fails)
```

---

## Distributed Tracing (OpenTelemetry)

```erlang
%% opentelemetry: distributed tracing
%% deps: {opentelemetry, "1.3.0"}, {opentelemetry_api, "1.2.2"}

%% Start a trace
tracer:start_span(<<"process_order">>, #{
    attributes => #{<<"order_id">> => OrderId}
}, fun() ->
    %% Everything here is part of the span

    %% Create child span
    tracer:with_span(<<"validate_inventory">>, fun() ->
        inventory:validate(Items)
    end),

    tracer:with_span(<<"charge_payment">>, #{
        attributes => #{<<"amount">> => Amount}
    }, fun() ->
        payments:charge(PaymentInfo)
    end)
end).

%% Manual span creation
Ctx = otel_tracer:start_span(otel_tracer:current_tracer(),
    <<"my_operation">>, #{}).
%% ... do work ...
otel_span:set_attribute(Ctx, <<"result">>, <<"ok">>).
otel_span:end_span(Ctx).

%% Add attributes
otel_span:set_attribute(otel_tracer:current_span_ctx(), <<"key">>, <<"value">>).

%% Add event
otel_span:add_event(otel_tracer:current_span_ctx(),
    <<"item_added">>,
    #{<<"item_id">> => ItemId}).

%% Mark error
otel_span:set_status(otel_tracer:current_span_ctx(),
    opentelemetry:status(error, <<"Database error">>)).

%% Export to Jaeger/Zipkin/Tempo (sys.config)
[{opentelemetry, [
    {span_processor, batch},
    {exporter, {opentelemetry_exporter, #{
        endpoints => [{http, "localhost", 4317, []}]
    }}}
]}].
```

---

## Crash Reports and SASL

```erlang
%% SASL (System Application Support Libraries): crash reports, progress reports

%% Enable SASL in sys.config:
[{sasl, [
    {sasl_error_logger, {file, "/var/log/myapp/sasl.log"}},
    {errlog_type, error}  %% error | progress | all
]}].

%% Crash report format:
%% =CRASH REPORT====
%% crasher:
%%   pid: <0.123.0>
%%   registered_name: my_server
%%   error_info: {error, {badmatch, undefined}, [stacktrace...]}
%% neighbours:
%%   supervisor: my_supervisor
%%   ...

%% Progress reports (startup):
%% =PROGRESS REPORT====
%%   supervisor: {local, my_supervisor}
%%   started: [{pid, <0.45.0>}, {id, my_worker}, ...]

%% Programmatic crash report handling
-module(crash_report_handler).

handle_crash(#{
    pid := Pid,
    error_info := {Type, Reason, Stack},
    registered_name := Name
}) ->
    Msg = io_lib:format("Process ~p (~p) crashed: ~p:~p~n~p",
                        [Pid, Name, Type, Reason, Stack]),
    notify_pagerduty(Msg).

%% sys module: control/inspect running processes
sys:get_state(my_server).
sys:replace_state(my_server, fun(S) -> S#{debug => true} end).
sys:suspend(my_server).    %% pause
sys:resume(my_server).     %% resume
sys:statistics(my_server, true).   %% enable stats
sys:statistics(my_server, get).    %% read stats
sys:trace(my_server, true).        %% trace messages

%% recon: production-safe inspection
recon:proc_count(reductions, 10).        %% top CPU hogs
recon:proc_count(message_queue_len, 10). %% stuck processes
recon:proc_window(reductions, 10, 1000). %% sample 1s window
recon:get_state(my_server).              %% safe get_state

%% observer: GUI
observer:start().
%% Applications, Processes, Ports, Mnesia, Charts
```

---

## ตัวอย่างจริง: Observability Stack

```erlang
%% ไฟล์: observability.erl
%% Complete observability setup: metrics + logging + health

-module(observability).
-behaviour(application).

-export([start/2, stop/1]).
-export([record_request/3, record_error/2, get_stats/0]).

start(normal, _) ->
    %% Setup Prometheus metrics
    setup_metrics(),
    %% Start health check server
    start_health_server(),
    observability_sup:start_link().

stop(_) -> ok.

setup_metrics() ->
    %% HTTP metrics
    prometheus_counter:new([
        {name, http_requests_total},
        {help, "Total HTTP requests"},
        {labels, [method, status]}
    ]),
    prometheus_histogram:new([
        {name, http_request_duration_seconds},
        {help, "HTTP request latency"},
        {labels, [method, status]},
        {buckets, [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]}
    ]),
    prometheus_gauge:new([
        {name, active_requests},
        {help, "Currently processing requests"}
    ]),
    %% Business metrics
    prometheus_counter:new([
        {name, orders_processed_total},
        {help, "Total orders processed"},
        {labels, [status]}
    ]),
    %% VM metrics
    prometheus_vm_memory_collector:register(),
    prometheus_vm_system_info_collector:register().

start_health_server() ->
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/health", health_handler, []},
            {"/health/live", liveness_handler, []},
            {"/health/ready", readiness_handler, []},
            {"/metrics", metrics_handler, []}
        ]}
    ]),
    {ok, _} = cowboy:start_clear(health_listener,
        [{port, 9090}],
        #{env => #{dispatch => Dispatch}}).

%% Record an HTTP request
record_request(Method, Status, DurationMs) ->
    prometheus_counter:inc(http_requests_total, [Method, Status]),
    prometheus_histogram:observe(
        http_request_duration_seconds,
        [Method, Status],
        DurationMs / 1000
    ).

record_error(Type, Reason) ->
    logger:error("Error", #{type => Type, reason => Reason}),
    prometheus_counter:inc(errors_total, [Type]).

get_stats() ->
    #{
        memory => erlang:memory(),
        processes => erlang:system_info(process_count),
        schedulers => erlang:statistics(scheduler_wall_time)
    }.


%% Middleware: record metrics for every request
-module(metrics_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    Start = erlang:monotonic_time(millisecond),
    prometheus_gauge:inc(active_requests),
    Req2 = cowboy_req:set_resp_header(
        <<"x-request-id">>,
        generate_request_id(),
        Req
    ),
    %% Store start time for response handler
    {ok, Req2, Env#{request_start => Start}}.

%% After response (via stream handler)
after_response(Status, Method, Env) ->
    Start = maps:get(request_start, Env, erlang:monotonic_time(millisecond)),
    Duration = erlang:monotonic_time(millisecond) - Start,
    prometheus_gauge:dec(active_requests),
    observability:record_request(Method, Status, Duration).

generate_request_id() ->
    <<A:32>> = crypto:strong_rand_bytes(4),
    integer_to_binary(A, 16).
```

---

## สรุป Part 40

| Tool | Purpose |
|------|---------|
| `logger` | Structured logging (OTP 21+) |
| `prometheus.erl` | Metrics exposition |
| `/health` endpoint | Kubernetes health checks |
| `opentelemetry` | Distributed tracing |
| `sys` module | Runtime process inspection |
| `observer` | GUI monitoring |
| `recon` | Safe production inspection |
| `SASL` | Crash/supervisor reports |

---

*[← Part 39: Performance Tuning](part_39_performance.md) | [Part 41: Release Management →](part_41_releases.md)*
