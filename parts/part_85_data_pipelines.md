# Part 85: Data Pipeline Patterns

## สารบัญ
1. [GenStage Pipeline Architecture](#genstage)
2. [ETL with Flow](#flow)
3. [Streaming Processing](#streaming)
4. [Backpressure Management](#backpressure)
5. [Data Transformation Patterns](#transform)
6. [ตัวอย่างจริง: Log Processing Pipeline](#log-pipeline)

---

## GenStage Pipeline Architecture

```erlang
%% GenStage: producer → producer_consumer → consumer

%% 1. Producer: generates demand-driven events
-module(db_producer).
-behaviour(gen_stage).
-export([start_link/1, init/1, handle_demand/2, handle_info/2]).

-record(state, {
    query,
    offset = 0,
    batch_size = 1000,
    exhausted = false
}).

start_link(Query) ->
    gen_stage:start_link(?MODULE, Query, []).

init(Query) ->
    {producer, #state{query = Query}}.

%% Called when downstream consumer is ready for more data
handle_demand(Demand, #state{exhausted = true} = State) ->
    {stop, normal, State};

handle_demand(Demand, #state{query = Q, offset = Off, batch_size = BS} = State) ->
    BatchSize = min(Demand, BS),
    
    case db:execute(Q, [Off, Off + BatchSize]) of
        {ok, []} ->
            %% No more data
            {noreply, [], State#state{exhausted = true}};
        {ok, Rows} ->
            NewOffset = Off + length(Rows),
            Events = [row_to_event(R) || R <- Rows],
            {noreply, Events, State#state{offset = NewOffset}}
    end.

row_to_event({Id, Data, Ts}) ->
    #{id => Id, data => Data, timestamp => Ts}.

%% 2. Transformer: enriches events
-module(enrichment_stage).
-behaviour(gen_stage).
-export([start_link/1, init/1, handle_events/3]).

start_link(EnrichFun) ->
    gen_stage:start_link(?MODULE, EnrichFun, []).

init(EnrichFun) ->
    {producer_consumer, EnrichFun}.

handle_events(Events, _From, EnrichFun) ->
    EnrichedEvents = lists:map(fun(Event) ->
        case EnrichFun(Event) of
            {ok, Enriched} -> Enriched;
            error -> Event  %% pass through on error
        end
    end, Events),
    {noreply, EnrichedEvents, EnrichFun}.

%% 3. Consumer: persists events
-module(db_consumer).
-behaviour(gen_stage).
-export([start_link/1, init/1, handle_events/3]).

-record(state, {
    target_table,
    batch = [],
    batch_size = 500,
    flush_interval = 1000
}).

start_link(TargetTable) ->
    gen_stage:start_link(?MODULE, TargetTable, []).

init(TargetTable) ->
    %% Request events in batches
    erlang:send_after(1000, self(), flush),
    {consumer, #state{target_table = TargetTable}, [{max_demand, 500}, {min_demand, 100}]}.

handle_events(Events, _From, #state{batch = Batch, batch_size = BS} = State) ->
    NewBatch = Batch ++ Events,
    
    case length(NewBatch) >= BS of
        true ->
            flush_batch(NewBatch, State),
            {noreply, [], State#state{batch = []}};
        false ->
            {noreply, [], State#state{batch = NewBatch}}
    end.

handle_info(flush, #state{batch = []} = State) ->
    erlang:send_after(1000, self(), flush),
    {noreply, [], State};
handle_info(flush, #state{batch = Batch} = State) ->
    flush_batch(Batch, State),
    erlang:send_after(1000, self(), flush),
    {noreply, [], State#state{batch = []}}.

flush_batch(Events, #state{target_table = Table}) ->
    Rows = [event_to_row(E) || E <- Events],
    db:bulk_insert(Table, Rows),
    logger:info("Flushed batch", #{count => length(Events), table => Table}).

event_to_row(#{id := Id, data := Data, timestamp := Ts}) ->
    {Id, jiffy:encode(Data), Ts}.
```

---

## ETL with Flow

```erlang
%% Using Flow (built on GenStage) for parallel data processing

-module(etl_pipeline).
-export([run/2]).

run(SourceQuery, DestTable) ->
    %% Create a flow from a producer
    Flow = flow:from_stages([db_producer:start_link(SourceQuery)]),
    
    %% Transform in parallel (4 workers)
    Flow2 = flow:map(Flow, fun transform_record/1, [{stages, 4}]),
    
    %% Filter out invalid records
    Flow3 = flow:filter(Flow2, fun is_valid/1),
    
    %% Enrich in parallel
    Flow4 = flow:flat_map(Flow3, fun enrich/1),
    
    %% Partition by user_id for consistent ordering
    Flow5 = flow:partition(Flow4, [{key, fun(E) -> maps:get(user_id, E) end}]),
    
    %% Reduce to compute aggregates per partition
    Flow6 = flow:reduce(Flow5, fun() -> #{} end, fun aggregate/2),
    
    %% Emit final results
    flow:run(Flow6, [db_consumer:start_link(DestTable)]).

transform_record(#{raw_data := Raw} = Record) ->
    Record#{
        processed_at => erlang:system_time(second),
        data => parse_raw(Raw)
    }.

is_valid(#{data := Data}) ->
    maps:is_key(user_id, Data) andalso maps:is_key(event_type, Data).

enrich(#{data := #{user_id := UserId}} = Event) ->
    case user_cache:get(UserId) of
        {ok, User} -> [Event#{user => User}];
        not_found -> []  %% drop events for unknown users
    end.

aggregate(Event, Acc) ->
    EventType = maps:get(event_type, Event),
    Count = maps:get(EventType, Acc, 0),
    Acc#{EventType => Count + 1}.

parse_raw(Raw) ->
    case jiffy:decode(Raw, [return_maps]) of
        Map when is_map(Map) -> Map;
        _ -> #{}
    end.
```

---

## Streaming Processing with Windows

```erlang
%% stream_processor.erl — time-window aggregations

-module(stream_processor).
-behaviour(gen_server).
-export([start_link/1, push_event/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(window, {
    events = [],
    start_time,
    duration_ms
}).

-record(state, {
    window_duration_ms = 60000,   %% 1-minute windows
    slide_interval_ms = 10000,    %% slide every 10 seconds
    current_window,
    emit_fun
}).

start_link(EmitFun) ->
    gen_server:start_link(?MODULE, EmitFun, []).

push_event(Pid, Event) ->
    gen_server:cast(Pid, {event, Event}).

init(EmitFun) ->
    Now = erlang:monotonic_time(millisecond),
    Window = #window{start_time = Now, duration_ms = 60000},
    erlang:send_after(10000, self(), slide_window),
    {ok, #state{current_window = Window, emit_fun = EmitFun}}.

handle_cast({event, Event}, #state{current_window = W} = State) ->
    NewWindow = W#window{events = [Event | W#window.events]},
    {noreply, State#state{current_window = NewWindow}}.

handle_info(slide_window, #state{current_window = W, emit_fun = EmitFun} = State) ->
    Now = erlang:monotonic_time(millisecond),
    
    %% Compute aggregates for current window
    WindowStats = compute_window_stats(W),
    EmitFun(WindowStats),
    
    %% Create new sliding window (keep events from overlap)
    CutoffTime = Now - State#state.window_duration_ms,
    RecentEvents = [E || E <- W#window.events, maps:get(timestamp, E, 0) > CutoffTime],
    NewWindow = W#window{events = RecentEvents, start_time = Now},
    
    erlang:send_after(State#state.slide_interval_ms, self(), slide_window),
    {noreply, State#state{current_window = NewWindow}}.

compute_window_stats(#window{events = Events, start_time = Start, duration_ms = Duration}) ->
    #{
        event_count => length(Events),
        unique_users => count_unique(Events, user_id),
        by_type => count_by(Events, event_type),
        window_start => Start,
        window_duration_ms => Duration
    }.

count_unique(Events, Field) ->
    length(lists:usort([maps:get(Field, E, undefined) || E <- Events])).

count_by(Events, Field) ->
    lists:foldl(fun(E, Acc) ->
        Key = maps:get(Field, E, unknown),
        Acc#{Key => maps:get(Key, Acc, 0) + 1}
    end, #{}, Events).
```

---

## Backpressure Management

```erlang
%% backpressure.erl — demand-driven flow control

%% GenStage handles backpressure via demand:
%% - Consumer requests N events
%% - Producer sends at most N events
%% - No event loss, no unbounded queues

%% Manual backpressure for non-GenStage scenarios:
-module(bounded_queue).
-behaviour(gen_server).
-export([start_link/1, push/2, pop/1, size/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    queue = queue:new(),
    max_size,
    waiting_producers = [],
    waiting_consumers = []
}).

start_link(MaxSize) ->
    gen_server:start_link(?MODULE, MaxSize, []).

push(Queue, Item) ->
    gen_server:call(Queue, {push, Item}, infinity).

pop(Queue) ->
    gen_server:call(Queue, pop, infinity).

init(MaxSize) ->
    {ok, #state{max_size = MaxSize}}.

handle_call({push, Item}, From, #state{queue = Q, max_size = Max, waiting_consumers = WC} = State) ->
    case {queue:len(Q) >= Max, WC} of
        {true, []} ->
            %% Queue full, block producer
            {noreply, State#state{waiting_producers = State#state.waiting_producers ++ [{From, Item}]}};
        {_, [Consumer | Rest]} ->
            %% Waiting consumer: deliver directly
            gen_server:reply(Consumer, {ok, Item}),
            {reply, ok, State#state{waiting_consumers = Rest}};
        {false, []} ->
            %% Room in queue
            {reply, ok, State#state{queue = queue:in(Item, Q)}}
    end;

handle_call(pop, From, #state{queue = Q, waiting_producers = WP} = State) ->
    case {queue:out(Q), WP} of
        {{empty, _}, []} ->
            %% Queue empty, block consumer
            {noreply, State#state{waiting_consumers = State#state.waiting_consumers ++ [From]}};
        {{empty, _}, [{Producer, Item} | Rest]} ->
            %% Unblock a waiting producer
            gen_server:reply(Producer, ok),
            {reply, {ok, Item}, State#state{waiting_producers = Rest}};
        {{{value, Item}, NewQ}, _} ->
            %% Unblock waiting producer if any
            NewState = case WP of
                [{Producer, NewItem} | Rest] ->
                    gen_server:reply(Producer, ok),
                    State#state{queue = queue:in(NewItem, NewQ), waiting_producers = Rest};
                [] ->
                    State#state{queue = NewQ}
            end,
            {reply, {ok, Item}, NewState}
    end.

size(Queue) ->
    gen_server:call(Queue, size).
```

---

## ตัวอย่างจริง: Log Processing Pipeline

```erlang
%% log_pipeline.erl — high-throughput log ingestion and analysis

-module(log_pipeline).
-export([start/0]).

-define(PIPELINE_WORKERS, 8).

start() ->
    %% Source: receive logs via UDP syslog
    {ok, LogSource} = log_source:start_link(#{port => 514}),
    
    %% Parse stage: parse syslog format
    ParseWorkers = [begin
        {ok, Pid} = log_parser:start_link(),
        Pid
    end || _ <- lists:seq(1, ?PIPELINE_WORKERS)],
    
    %% Enrich stage: add GeoIP, user info
    EnrichWorkers = [begin
        {ok, Pid} = log_enricher:start_link(geoip_db),
        Pid
    end || _ <- lists:seq(1, ?PIPELINE_WORKERS)],
    
    %% Router: route to different sinks based on severity
    {ok, Router} = log_router:start_link(#{
        critical => elasticsearch_sink,
        error => elasticsearch_sink,
        warning => elasticsearch_sink,
        info => s3_sink,
        debug => null_sink
    }),
    
    %% Connect stages with GenStage subscriptions
    [gen_stage:sync_subscribe(P, [{to, LogSource}, {max_demand, 200}])
     || P <- ParseWorkers],
    
    [gen_stage:sync_subscribe(E, [{to, P}, {max_demand, 100}])
     || {P, E} <- lists:zip(ParseWorkers, EnrichWorkers)],
    
    [gen_stage:sync_subscribe(Router, [{to, E}, {max_demand, 50}])
     || E <- EnrichWorkers],
    
    logger:info("Log pipeline started", #{workers => ?PIPELINE_WORKERS}).

%% Log source: UDP receiver
-module(log_source).
-behaviour(gen_stage).

init(#{port := Port}) ->
    {ok, Socket} = gen_udp:open(Port, [binary, {active, true}, {recbuf, 65536}]),
    {producer, #{socket => Socket, buffer => []}}.

handle_demand(Demand, #{buffer := Buffer} = State) ->
    case length(Buffer) >= Demand of
        true ->
            {Events, Rest} = lists:split(Demand, Buffer),
            {noreply, Events, State#{buffer := Rest}};
        false ->
            %% Return what we have, wait for more via UDP
            {noreply, Buffer, State#{buffer := []}}
    end.

handle_info({udp, _Socket, _IP, _Port, Data}, State) ->
    Buffer = maps:get(buffer, State),
    {noreply, [], State#{buffer := Buffer ++ [Data]}}.

%% Log parser
-module(log_parser).
-behaviour(gen_stage).

init([]) -> {producer_consumer, #{}}.

handle_events(RawLogs, _From, State) ->
    Parsed = lists:filtermap(fun(Raw) ->
        case parse_syslog(Raw) of
            {ok, Log} -> {true, Log};
            _ -> false
        end
    end, RawLogs),
    {noreply, Parsed, State}.

parse_syslog(<<$<, Rest/binary>>) ->
    case binary:split(Rest, <<">>">>) of
        [PriorityBin, Message] ->
            Priority = binary_to_integer(PriorityBin),
            Severity = priority_to_severity(Priority rem 8),
            {ok, #{
                severity => Severity,
                message => Message,
                timestamp => erlang:system_time(second),
                raw => <<$<, Rest/binary>>
            }};
        _ -> error
    end;
parse_syslog(_) -> error.

priority_to_severity(0) -> critical;
priority_to_severity(1) -> alert;
priority_to_severity(2) -> critical;
priority_to_severity(3) -> error;
priority_to_severity(4) -> warning;
priority_to_severity(5) -> notice;
priority_to_severity(6) -> info;
priority_to_severity(7) -> debug.

%% S3 sink for bulk storage
-module(s3_sink).
-behaviour(gen_stage).

-record(state, {
    bucket,
    buffer = [],
    flush_size = 10000,
    flush_interval_ms = 30000
}).

init(Bucket) ->
    erlang:send_after(30000, self(), flush),
    {consumer, #state{bucket = Bucket}}.

handle_events(Events, _From, #state{buffer = Buf, flush_size = FS} = State) ->
    NewBuf = Buf ++ Events,
    case length(NewBuf) >= FS of
        true -> flush_to_s3(NewBuf, State), {noreply, [], State#state{buffer = []}};
        false -> {noreply, [], State#state{buffer = NewBuf}}
    end.

handle_info(flush, #state{buffer = []} = State) ->
    erlang:send_after(State#state.flush_interval_ms, self(), flush),
    {noreply, [], State};
handle_info(flush, #state{buffer = Buf} = State) ->
    flush_to_s3(Buf, State),
    erlang:send_after(State#state.flush_interval_ms, self(), flush),
    {noreply, [], State#state{buffer = []}}.

flush_to_s3(Events, #state{bucket = Bucket}) ->
    Key = io_lib:format("logs/~s/~s.jsonl", [
        calendar:system_time_to_rfc3339(erlang:system_time(second), [{unit, second}]),
        base64:encode(crypto:strong_rand_bytes(6))
    ]),
    Body = iolist_to_binary([jiffy:encode(E) ++ "\n" || E <- Events]),
    s3:put_object(Bucket, Key, Body),
    logger:info("Flushed logs to S3", #{key => Key, count => length(Events)}).
```

---

## สรุป Part 85

| Pattern | Library | Use Case |
|---------|---------|---------|
| GenStage | gen_stage | Demand-driven pipelines |
| Flow | flow | Parallel ETL |
| Bounded Queue | Custom | Backpressure |
| Sliding Window | Custom gen_server | Time-based aggregations |
| Log pipeline | Multi-stage | Syslog ingestion |

**Throughput tip**: Use `max_demand` and `min_demand` in GenStage to control batch sizes and reduce overhead per event.

---

*[← Part 84: Testing Strategies](part_84_testing_strategies.md) | [Part 86: Distributed Coordination →](part_86_distributed_coordination.md)*
