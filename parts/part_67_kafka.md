# Part 67: Kafka Integration — High-Throughput Streaming

## สารบัญ
1. [Kafka Concepts](#concepts)
2. [Producer Patterns](#producer)
3. [Consumer Groups](#consumer-groups)
4. [Exactly-Once Semantics](#exactly-once)
5. [Stream Processing](#stream-processing)
6. [ตัวอย่างจริง: Analytics Pipeline](#analytics)

---

## Kafka Concepts

```
Kafka: distributed log / event streaming platform
- Producers write to topics
- Consumers read from topics
- Topics split into partitions (parallelism)
- Messages retained for configurable time (not deleted on consume)

vs RabbitMQ:
- Kafka: high throughput, log-based, replay events
- RabbitMQ: message queuing, routing, more flexible

Erlang library: brod (Klarna's library)
```

---

## Producer Patterns

```erlang
%% brod: production-grade Kafka client for Erlang

%% rebar.config
%% {deps, [{brod, "3.17.0"}]}

-module(kafka_producer).
-export([start/0, publish/3, publish_batch/2]).

start() ->
    %% Configure client
    BootstrapHosts = [{"kafka1", 9092}, {"kafka2", 9092}],
    ClientConfig = [
        {auto_start_producers, true},
        {allow_topic_auto_creation, false},
        {default_producer_config, [
            {required_acks, -1},     %% -1 = all ISR must ack (strongest guarantee)
            {ack_timeout, 10000},
            {compression, gzip},
            {max_retries, 3},
            {retry_backoff_ms, 1000}
        ]}
    ],
    
    brod:start_client(BootstrapHosts, kafka_client, ClientConfig),
    
    %% Start producer for topic
    brod:start_producer(kafka_client, <<"events">>, []).

%% Publish single message
publish(Topic, Key, Value) when is_binary(Key), is_binary(Value) ->
    %% Partition by key (same key → same partition → ordering)
    brod:produce_sync(kafka_client, Topic, hash, Key, Value).

%% Publish with headers (Kafka 0.11+)
publish_with_headers(Topic, Key, Value, Headers) ->
    KafkaHeaders = [{K, V} || {K, V} <- maps:to_list(Headers)],
    brod:produce_sync(kafka_client, Topic, hash, Key,
                      #kafka_message{key = Key, value = Value, headers = KafkaHeaders}).

%% Async publish (higher throughput)
publish_async(Topic, Key, Value) ->
    {ok, CallRef} = brod:produce(kafka_client, Topic, hash, Key, Value),
    CallRef.  %% can await ack later

%% Batch publish for throughput
publish_batch(Topic, Messages) ->
    %% Group by partition key for ordering
    Grouped = lists:foldl(fun(#{key := K, value := V}, Acc) ->
        Acc#{K => [V | maps:get(K, Acc, [])]}
    end, #{}, Messages),
    
    maps:foreach(fun(Key, Values) ->
        Batch = [{K, V} || {K, V} <- [{Key, Val} || Val <- Values]],
        brod:produce_sync_offset(kafka_client, Topic, hash, Key,
                                  [#{key => Key, value => V} || V <- Values])
    end, Grouped).

%% Transaction producer (exactly-once)
publish_transactional(Topic, Messages) ->
    TxnId = generate_txn_id(),
    brod_transaction:new(kafka_client, TxnId, fun(TxnRef) ->
        [brod_transaction:produce(TxnRef, Topic, hash, K, V) || {K, V} <- Messages]
    end).

generate_txn_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).
```

---

## Consumer Groups

```erlang
%% Consumer group: distribute partitions among instances

-module(kafka_consumer_group).
-behaviour(brod_group_subscriber).
-export([start_link/0, init/2, handle_message/4]).

start_link() ->
    GroupConfig = [
        {offset_commit_policy, commit_to_kafka_v2},
        {offset_commit_interval_seconds, 5},
        {max_bytes, 1048576},  %% 1MB per fetch
        {begin_offset, latest}  %% or 'earliest' to replay
    ],
    
    brod:start_link_group_subscriber(
        kafka_client,
        <<"my-consumer-group">>,
        [<<"events">>],     %% topics to subscribe
        GroupConfig,
        _CallbackModule = ?MODULE,
        _CallbackInitArg = []
    ).

init(_GroupId, []) ->
    {ok, #{}}.

%% Called for each message
handle_message(Topic, Partition, Message, State) ->
    #kafka_message{
        offset = Offset,
        key = Key,
        value = Value,
        headers = Headers
    } = Message,
    
    logger:debug("Received Kafka message", #{
        topic => Topic,
        partition => Partition,
        offset => Offset,
        key => Key
    }),
    
    case process_message(Topic, Key, Value) of
        ok ->
            {ok, ack, State};   %% commit offset
        {error, _} ->
            {ok, ack, State}    %% ack anyway (or use DLT pattern)
    end.

process_message(<<"events">>, Key, Value) ->
    Event = jsx:decode(Value, [return_maps]),
    EventType = maps:get(<<"type">>, Event),
    handle_event(EventType, Event).

handle_event(<<"user.signup">>, #{<<"user_id">> := UserId}) ->
    welcome_email:send(UserId),
    analytics:track(user_signup, UserId);

handle_event(<<"order.completed">>, #{<<"order_id">> := OrderId}) ->
    metrics:increment(orders_completed),
    analytics:revenue_event(OrderId);

handle_event(Type, _) ->
    logger:info("Unhandled event type", #{type => Type}),
    ok.

%% Partition assignment: control which partitions this instance processes
%% brod handles rebalancing when instances join/leave the group
```

---

## Stream Processing

```erlang
%% Kafka Streams pattern in Erlang: stateful stream processing

-module(stream_processor).
-behaviour(gen_server).
-export([start_link/0, get_counts/0]).
-export([init/1, handle_cast/2, handle_call/3]).

%% Count events per user in 1-minute windows
start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init(_) ->
    %% Start consumer
    kafka_consumer_group:start_link(),
    %% Window state: {WindowStart, UserId} → Count
    {ok, #{windows => #{}, current_window => current_window()}}.

handle_cast({event, UserId, EventType}, State) ->
    Window = current_window(),
    Key = {Window, UserId},
    NewWindows = maps:update_with(Key, fun(C) -> C + 1 end, 1, maps:get(windows, State)),
    
    %% Emit completed windows
    NewState = emit_old_windows(State#{windows := NewWindows}),
    
    {noreply, NewState};

handle_call(get_counts, _From, #{windows := Windows} = State) ->
    {reply, Windows, State}.

current_window() ->
    erlang:system_time(second) div 60 * 60.  %% 1-minute window

emit_old_windows(#{windows := Windows, current_window := CW} = State) ->
    Now = current_window(),
    %% Emit windows older than current
    {Old, Current} = maps:partition(fun({W, _}, _) -> W < Now end, Windows),
    
    maps:foreach(fun({Window, UserId}, Count) ->
        %% Publish aggregated result back to Kafka
        Result = jsx:encode(#{window => Window, user_id => UserId, count => Count}),
        kafka_producer:publish(<<"event_counts">>, UserId, Result)
    end, Old),
    
    State#{windows := Current, current_window := Now}.
```

---

## ตัวอย่างจริง: Analytics Pipeline

```erlang
%% analytics_pipeline.erl: ingest → process → aggregate → store

-module(analytics_pipeline).
-export([start/0]).

start() ->
    %% Stage 1: Consume raw events from Kafka
    consumer_sup:start_link([
        {topic, <<"raw_events">>},
        {group_id, <<"analytics">>},
        {processor, fun enrich_event/1}
    ]),
    
    %% Stage 2: Enrich and filter
    %% Stage 3: Aggregate in time windows
    %% Stage 4: Store to ClickHouse/BigQuery

enrich_event(#{<<"user_id">> := UserId} = Event) ->
    %% Add user profile data
    UserProfile = user_cache:get(UserId),
    EnrichedEvent = maps:merge(Event, #{
        <<"country">> => maps:get(country, UserProfile, <<"unknown">>),
        <<"plan">> => maps:get(plan, UserProfile, <<"free">>),
        <<"enriched_at">> => erlang:system_time(millisecond)
    }),
    
    %% Filter: skip internal users
    case maps:get(<<"is_internal">>, UserProfile, false) of
        true -> skip;
        false -> {ok, EnrichedEvent}
    end.

%% Real-time funnel analysis
-module(funnel_analyzer).
-export([track_step/3, get_funnel/1]).

track_step(FunnelId, UserId, Step) ->
    Timestamp = erlang:system_time(second),
    Key = {FunnelId, UserId},
    
    case ets:lookup(funnel_state, Key) of
        [{_, Steps}] ->
            case should_advance(Steps, Step) of
                true ->
                    NewSteps = Steps#{Step => Timestamp},
                    ets:insert(funnel_state, {Key, NewSteps}),
                    maybe_complete_funnel(FunnelId, UserId, NewSteps);
                false -> ok
            end;
        [] ->
            ets:insert(funnel_state, {Key, #{Step => Timestamp}})
    end.

get_funnel(FunnelId) ->
    %% Aggregate all users' funnel progress
    AllEntries = ets:match(funnel_state, {{FunnelId, '$1'}, '$2'}),
    Funnel = #{
        total_users => length(AllEntries),
        completion_rate => completion_rate(AllEntries),
        avg_time_to_complete => avg_time(AllEntries),
        drop_off_points => drop_off_analysis(AllEntries)
    },
    Funnel.

should_advance(CompletedSteps, NextStep) ->
    Required = required_previous_step(NextStep),
    Required =:= undefined orelse maps:is_key(Required, CompletedSteps).

required_previous_step(signup_complete) -> email_verified;
required_previous_step(first_purchase) -> signup_complete;
required_previous_step(_) -> undefined.

maybe_complete_funnel(FunnelId, UserId, Steps) ->
    FinalStep = final_step(FunnelId),
    case maps:is_key(FinalStep, Steps) of
        true ->
            Duration = maps:get(FinalStep, Steps) - maps:get(hd(maps:keys(Steps)), Steps),
            kafka_producer:publish(<<"funnel_completions">>, UserId,
                jsx:encode(#{funnel => FunnelId, user => UserId, duration_seconds => Duration}));
        false -> ok
    end.

final_step(signup_funnel) -> first_purchase;
final_step(_) -> complete.

completion_rate(Entries) ->
    Completed = length([E || {_, Steps} <- Entries, maps:is_key(first_purchase, Steps)]),
    case length(Entries) of
        0 -> 0;
        N -> Completed / N * 100
    end.

avg_time(Entries) ->
    Durations = [maps:get(first_purchase, Steps, 0) - maps:get(email_verified, Steps, 0)
                 || {_, Steps} <- Entries, maps:is_key(first_purchase, Steps), maps:is_key(email_verified, Steps)],
    case Durations of
        [] -> 0;
        D -> lists:sum(D) div length(D)
    end.

drop_off_analysis(Entries) ->
    Steps = [email_verified, signup_complete, first_purchase],
    StepCounts = maps:from_list([{Step, 0} || Step <- Steps]),
    Final = lists:foldl(fun({_, UserSteps}, Acc) ->
        lists:foldl(fun(Step, A) ->
            case maps:is_key(Step, UserSteps) of
                true -> A#{Step => maps:get(Step, A, 0) + 1};
                false -> A
            end
        end, Acc, Steps)
    end, StepCounts, Entries),
    Final.
```

---

## สรุป Part 67

| Kafka Feature | Erlang/brod API |
|---------------|-----------------|
| Produce sync | `brod:produce_sync/5` |
| Produce async | `brod:produce/5` |
| Consumer group | `brod_group_subscriber` |
| Transactions | `brod_transaction` |
| Offset management | `commit_to_kafka_v2` |
| Partition strategy | `hash`, `random`, `round_robin` |

---

*[← Part 66: RabbitMQ](part_66_rabbitmq.md) | [Part 68: Advanced OTP Patterns →](part_68_advanced_otp.md)*
