# Part 63: Grafana Monitoring Stack

## สารบัญ
1. [Monitoring Stack Overview](#stack)
2. [Prometheus Metrics Export](#prometheus)
3. [Grafana Dashboards](#grafana)
4. [Alerting Rules](#alerting)
5. [Distributed Tracing with Jaeger](#jaeger)
6. [ตัวอย่างจริง: Full Observability Stack](#full-stack)

---

## Stack Overview

```
Erlang App
    ↓ metrics (Prometheus format)
Prometheus (scrape every 15s)
    ↓ data
Grafana (dashboards, alerts)
    ↓ alerts
Alertmanager → PagerDuty / Slack

Erlang App
    ↓ traces (OpenTelemetry)
Jaeger / Tempo
    ↓ 
Grafana (trace explorer)

Erlang App
    ↓ logs (structured JSON)
Promtail / Fluentd
    ↓
Loki
    ↓
Grafana (log explorer)
```

---

## Prometheus Metrics Export

```erlang
%% metrics_collector.erl: สร้าง comprehensive metrics

-module(metrics_collector).
-export([start_link/0, init/1, handle_call/3, handle_info/2]).

%% Metric types:
%% - Counter: monotonically increasing (requests, errors)
%% - Gauge: current value (connections, memory, queue depth)
%% - Histogram: distribution (request latency)
%% - Summary: percentiles (same, with server-side calculation)

setup_metrics() ->
    %% Application metrics
    prometheus_counter:new([
        {name, http_requests_total},
        {labels, [method, path, status]},
        {help, "Total HTTP requests"}
    ]),
    
    prometheus_histogram:new([
        {name, http_request_duration_seconds},
        {labels, [method, path]},
        {buckets, [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]},
        {help, "HTTP request duration in seconds"}
    ]),
    
    prometheus_gauge:new([
        {name, active_connections},
        {help, "Current active connections"}
    ]),
    
    prometheus_gauge:new([
        {name, queue_depth},
        {labels, [queue_name]},
        {help, "Number of items in queue"}
    ]),
    
    prometheus_counter:new([
        {name, database_queries_total},
        {labels, [query_type, table]},
        {help, "Database queries"}
    ]),
    
    prometheus_histogram:new([
        {name, database_query_duration_seconds},
        {labels, [query_type]},
        {buckets, [0.0001, 0.001, 0.01, 0.1, 1.0]},
        {help, "DB query duration"}
    ]),
    
    %% Business metrics
    prometheus_counter:new([
        {name, orders_created_total},
        {labels, [region]},
        {help, "Orders created"}
    ]),
    
    prometheus_gauge:new([
        {name, revenue_per_hour},
        {labels, [currency]},
        {help, "Revenue per hour"}
    ]).

%% Record HTTP request
record_http_request(Method, Path, Status, DurationSec) ->
    prometheus_counter:inc(http_requests_total, [Method, Path, Status]),
    prometheus_histogram:observe(http_request_duration_seconds, [Method, Path], DurationSec).

%% Cowboy middleware for auto-instrumentation
-module(prometheus_middleware).
-export([execute/1]).

execute(Env) ->
    #{req := Req} = Env,
    Method = cowboy_req:method(Req),
    Path = cowboy_req:path(Req),
    Start = erlang:monotonic_time(microsecond),
    
    Result = cowboy_next:execute(Env),
    
    Status = integer_to_binary(cowboy_req:resp_status(maps:get(req, Result, Req))),
    Duration = (erlang:monotonic_time(microsecond) - Start) / 1_000_000,
    
    record_http_request(Method, Path, Status, Duration),
    Result.

%% BEAM VM metrics collector
collect_vm_metrics() ->
    %% Process metrics
    prometheus_gauge:set(erlang_process_count, erlang:system_info(process_count)),
    
    %% Memory metrics
    Mem = erlang:memory(),
    [prometheus_gauge:set(
        list_to_atom("erlang_memory_" ++ atom_to_list(Type) ++ "_bytes"),
        Bytes
    ) || {Type, Bytes} <- Mem],
    
    %% Scheduler metrics
    erlang:system_flag(scheduler_wall_time, true),
    SchedulerStats = erlang:statistics(scheduler_wall_time),
    [begin
        Util = case Active + Idle of
            0 -> 0;
            Total -> Active / Total
        end,
        prometheus_gauge:set(erlang_scheduler_utilization, [integer_to_binary(Id)], Util)
    end || {Id, Active, Idle} <- SchedulerStats],
    
    %% GC stats
    {GCCount, GCTime, _} = erlang:statistics(garbage_collection),
    prometheus_counter:inc(erlang_gc_runs_total, GCCount),
    prometheus_counter:inc(erlang_gc_time_microseconds_total, GCTime).
```

---

## Grafana Dashboards

```json
// dashboard.json: Erlang Application Dashboard
{
  "title": "Erlang Application",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [{
        "expr": "rate(http_requests_total[5m])",
        "legendFormat": "{{method}} {{path}} {{status}}"
      }]
    },
    {
      "title": "Request Duration P99",
      "type": "graph",
      "targets": [{
        "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
        "legendFormat": "P99 {{path}}"
      }]
    },
    {
      "title": "Error Rate %",
      "type": "singlestat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{status=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100"
      }],
      "thresholds": "1,5",
      "colorBackground": true
    },
    {
      "title": "Erlang Process Count",
      "type": "graph",
      "targets": [{
        "expr": "erlang_process_count",
        "legendFormat": "{{instance}}"
      }]
    },
    {
      "title": "Memory Usage",
      "type": "graph",
      "targets": [
        {"expr": "erlang_memory_total_bytes", "legendFormat": "Total"},
        {"expr": "erlang_memory_processes_bytes", "legendFormat": "Processes"},
        {"expr": "erlang_memory_binary_bytes", "legendFormat": "Binary"},
        {"expr": "erlang_memory_ets_bytes", "legendFormat": "ETS"}
      ]
    },
    {
      "title": "Scheduler Utilization",
      "type": "heatmap",
      "targets": [{
        "expr": "erlang_scheduler_utilization",
        "legendFormat": "Scheduler {{id}}"
      }]
    },
    {
      "title": "DB Query Duration",
      "type": "graph",
      "targets": [
        {"expr": "histogram_quantile(0.50, rate(database_query_duration_seconds_bucket[5m]))", "legendFormat": "P50"},
        {"expr": "histogram_quantile(0.95, rate(database_query_duration_seconds_bucket[5m]))", "legendFormat": "P95"},
        {"expr": "histogram_quantile(0.99, rate(database_query_duration_seconds_bucket[5m]))", "legendFormat": "P99"}
      ]
    }
  ]
}
```

---

## Alerting Rules

```yaml
# prometheus_rules.yaml
groups:
  - name: erlang_alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate {{ $value | humanizePercentage }}"
          description: "Error rate above 5% for 2 minutes"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency: {{ $value | humanizeDuration }}"

      - alert: HighProcessCount
        expr: erlang_process_count > 1000000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Erlang process count too high: {{ $value }}"

      - alert: MemoryPressure
        expr: erlang_memory_total_bytes / (1024^3) > 8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 8GB: {{ $value | humanize }}GB"

      - alert: NodeDown
        expr: up{job="erlang_app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Erlang node {{ $labels.instance }} is down"
```

```erlang
%% Erlang side: alertmanager integration
-module(alert_sender).
-export([fire/3]).

fire(AlertName, Labels, Annotations) ->
    Payload = jsx:encode([#{
        labels => maps:merge(Labels, #{alertname => AlertName}),
        annotations => Annotations,
        startsAt => format_time(erlang:system_time(millisecond))
    }]),
    
    httpc:request(post,
        {"http://alertmanager:9093/api/v2/alerts", [], "application/json", Payload},
        [{timeout, 5000}], []).
```

---

## Distributed Tracing Jaeger

```erlang
%% OpenTelemetry tracing
-module(tracer).
-export([start_span/2, end_span/1, add_event/2, set_attribute/3]).

%% Trace an HTTP request end-to-end
handle_request(Req, State) ->
    %% Extract trace context from incoming headers
    TraceCtx = otel_propagator_text_map:extract(cowboy_req:headers(Req)),
    
    %% Start span
    SpanCtx = otel_tracer:start_span(
        otel_tracer:current_tracer(),
        <<"http_request">>,
        #{
            kind => server,
            attributes => #{
                <<"http.method">> => cowboy_req:method(Req),
                <<"http.url">> => cowboy_req:url(Req),
                <<"http.scheme">> => <<"http">>
            }
        }
    ),
    Token = otel_ctx:attach(otel_tracer:set_current_span(SpanCtx)),
    
    try
        %% Process request
        Result = do_handle(Req, State),
        
        %% Record outcome
        Status = cowboy_req:resp_status(element(2, Result)),
        otel_span:set_attribute(SpanCtx, <<"http.status_code">>, Status),
        
        Result
    catch Class:Reason:Stack ->
        otel_span:record_exception(SpanCtx, Class, Reason, Stack, []),
        otel_span:set_status(SpanCtx, error, iolist_to_binary(io_lib:format("~p", [Reason]))),
        erlang:raise(Class, Reason, Stack)
    after
        otel_span:end_span(SpanCtx),
        otel_ctx:detach(Token)
    end.

%% Trace database call
with_db_span(QueryName, QueryFun) ->
    SpanCtx = otel_tracer:start_span(
        otel_tracer:current_tracer(),
        <<"db_query">>,
        #{
            attributes => #{
                <<"db.system">> => <<"postgresql">>,
                <<"db.operation">> => QueryName
            }
        }
    ),
    Token = otel_ctx:attach(otel_tracer:set_current_span(SpanCtx)),
    try
        Result = QueryFun(),
        Result
    after
        otel_span:end_span(SpanCtx),
        otel_ctx:detach(Token)
    end.
```

---

## สรุป Part 63

| Component | Purpose | Port |
|-----------|---------|------|
| Prometheus | Metrics scraping | 9090 |
| Grafana | Dashboards + alerts | 3000 |
| Alertmanager | Alert routing | 9093 |
| Jaeger | Distributed tracing | 16686 |
| Loki | Log aggregation | 3100 |

**Erlang libraries**: prometheus.erl, opentelemetry-erlang, logger

---

*[← Part 62: Advanced Testing](part_62_advanced_testing.md) | [Part 64: Database Patterns →](part_64_database_patterns.md)*
