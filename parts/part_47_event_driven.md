# Part 47: Event-Driven Architecture

## สารบัญ
1. [Event-Driven คืออะไร](#event-driven-คืออะไร)
2. [Message Queue กับ RabbitMQ](#message-queue-กับ-rabbitmq)
3. [Apache Kafka กับ Erlang](#apache-kafka-กับ-erlang)
4. [Event Bus ภายใน Application](#event-bus-ภายใน-application)
5. [CQRS Pattern](#cqrs-pattern)
6. [ตัวอย่างจริง: Order Processing Pipeline](#ตัวอย่างจริง-order-processing-pipeline)

---

## Event-Driven คืออะไร

```
Event-Driven Architecture (EDA):
- Services communicate through events (ไม่ใช่ direct calls)
- Producer: publish event → Message Broker → Consumer: process event
- Loose coupling: producer ไม่รู้จัก consumers
- Async: producer ไม่ต้องรอ consumer
- Resilient: consumer สามารถ retry ได้

Patterns:
- Event Notification: แจ้งว่าอะไรเกิดขึ้น
- Event-Carried State Transfer: ส่งข้อมูลพร้อม event
- Event Sourcing: เก็บ state เป็น stream of events
- CQRS: แยก read/write models
```

---

## Message Queue กับ RabbitMQ

```erlang
%% amqp_client: RabbitMQ client
%% deps: {amqp_client, "3.12.0"}

-module(rabbitmq_client).
-export([start/0, publish/3, consume/2]).

start() ->
    Params = #amqp_params_network{
        host = "localhost",
        port = 5672,
        username = <<"guest">>,
        password = <<"guest">>,
        virtual_host = <<"/">>
    },
    {ok, Connection} = amqp_connection:start(Params),
    {ok, Channel} = amqp_connection:open_channel(Connection),
    {Channel, Connection}.

%% Declare exchange and queue
setup_topology(Channel) ->
    %% Declare exchange
    Exchange = #'exchange.declare'{
        exchange = <<"orders">>,
        type = <<"topic">>,
        durable = true
    },
    #'exchange.declare_ok'{} = amqp_channel:call(Channel, Exchange),

    %% Declare queue
    Queue = #'queue.declare'{
        queue = <<"order_processing">>,
        durable = true,
        arguments = [
            {<<"x-dead-letter-exchange">>, longstr, <<"orders.dlx">>},
            {<<"x-message-ttl">>, long, 3600000}  %% 1 hour TTL
        ]
    },
    #'queue.declare_ok'{} = amqp_channel:call(Channel, Queue),

    %% Bind queue to exchange
    Bind = #'queue.bind'{
        queue = <<"order_processing">>,
        exchange = <<"orders">>,
        routing_key = <<"order.created">>
    },
    #'queue.bind_ok'{} = amqp_channel:call(Channel, Bind).

%% Publish message
publish(Channel, RoutingKey, Payload) ->
    Msg = #amqp_msg{
        props = #'P_basic'{
            content_type = <<"application/json">>,
            delivery_mode = 2,  %% persistent
            message_id = generate_id(),
            timestamp = erlang:system_time(second)
        },
        payload = jsx:encode(Payload)
    },
    Publish = #'basic.publish'{
        exchange = <<"orders">>,
        routing_key = RoutingKey
    },
    amqp_channel:cast(Channel, Publish, Msg).

%% Consume messages
consume(Channel, Queue) ->
    Sub = #'basic.consume'{
        queue = Queue,
        no_ack = false  %% manual ack for reliability
    },
    #'basic.consume_ok'{} = amqp_channel:subscribe(Channel, Sub, self()),
    consume_loop(Channel).

consume_loop(Channel) ->
    receive
        {#'basic.deliver'{delivery_tag = Tag}, #amqp_msg{payload = Payload}} ->
            case process_message(jsx:decode(Payload, [return_maps])) of
                ok ->
                    amqp_channel:cast(Channel, #'basic.ack'{delivery_tag = Tag});
                {error, _Reason} ->
                    %% Reject and requeue once
                    amqp_channel:cast(Channel, #'basic.nack'{
                        delivery_tag = Tag,
                        requeue = true
                    })
            end,
            consume_loop(Channel)
    end.

process_message(#{<<"type">> := <<"order.created">>, <<"order">> := Order}) ->
    %% Process the order
    order_processor:handle(Order).

generate_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).
```

---

## Apache Kafka กับ Erlang

```erlang
%% brod: Kafka client for Erlang
%% deps: {brod, "3.18.0"}

-module(kafka_client).
-export([start/0, produce/3, start_consumer/2]).

start() ->
    KafkaBrokers = [{"localhost", 9092}],
    ok = brod:start_client(KafkaBrokers, kafka_client, []),
    ok = brod:start_producer(kafka_client, <<"orders">>, []).

%% Produce message
produce(Topic, Key, Value) ->
    Encoded = jsx:encode(Value),
    case brod:produce_sync(kafka_client, Topic, hash, Key, Encoded) of
        ok -> ok;
        {error, Reason} ->
            logger:error("Kafka produce failed", #{reason => Reason}),
            {error, Reason}
    end.

%% Produce with callback
produce_async(Topic, Key, Value) ->
    Encoded = jsx:encode(Value),
    brod:produce(kafka_client, Topic, hash, Key, Encoded).

%% Consumer group
start_consumer(Topics, GroupId) ->
    Config = [
        {begin_offset, latest},
        {offset_commit_policy, commit_to_kafka_v2},
        {offset_commit_interval_seconds, 5}
    ],
    brod:start_link_group_subscriber(
        kafka_client,
        GroupId,
        Topics,
        [{session_timeout_seconds, 10}],
        Config,
        message,
        kafka_message_handler,  %% callback module
        []
    ).

%% Message handler module
-module(kafka_message_handler).
-behaviour(brod_group_subscriber).
-export([init/2, handle_message/4]).

init(GroupId, _Config) ->
    logger:info("Consumer started", #{group => GroupId}),
    {ok, #{}}.

handle_message(_Topic, Partition, Message, State) ->
    #kafka_message{
        offset = Offset,
        key = Key,
        value = Value
    } = Message,

    Decoded = jsx:decode(Value, [return_maps]),
    case process_event(Decoded) of
        ok ->
            {ok, ack, State};
        {error, Reason} ->
            logger:error("Message processing failed", #{
                offset => Offset,
                reason => Reason
            }),
            %% Ack anyway to avoid poison pill
            {ok, ack, State}
    end.

process_event(#{<<"type">> := Type} = Event) ->
    handle_event(Type, Event).

handle_event(<<"order.created">>, Event) ->
    order_service:process_new_order(Event);
handle_event(<<"payment.processed">>, Event) ->
    order_service:confirm_payment(Event);
handle_event(Type, _) ->
    logger:warning("Unknown event type", #{type => Type}),
    ok.
```

---

## Event Bus ภายใน Application

```erlang
%% internal_event_bus.erl: in-process event bus
-module(event_bus).
-behaviour(gen_server).
-export([start_link/0, publish/2, subscribe/2, unsubscribe/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(EVENT_TABLE, event_subscriptions).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

publish(EventType, Payload) ->
    gen_server:cast(?MODULE, {publish, EventType, Payload}).

subscribe(EventType, Handler) ->
    gen_server:call(?MODULE, {subscribe, EventType, Handler}).

unsubscribe(EventType, Handler) ->
    gen_server:cast(?MODULE, {unsubscribe, EventType, Handler}).

init(_) ->
    ets:new(?EVENT_TABLE, [named_table, bag, protected]),
    {ok, #{}}.

handle_call({subscribe, EventType, Handler}, _From, State) ->
    ets:insert(?EVENT_TABLE, {EventType, Handler}),
    {reply, ok, State}.

handle_cast({publish, EventType, Payload}, State) ->
    Event = #{
        type => EventType,
        payload => Payload,
        timestamp => erlang:system_time(millisecond),
        id => make_ref()
    },
    Handlers = [H || {_, H} <- ets:lookup(?EVENT_TABLE, EventType)],
    %% Also get wildcard handlers
    Wildcards = [H || {_, H} <- ets:lookup(?EVENT_TABLE, '*')],
    AllHandlers = Handlers ++ Wildcards,
    %% Async dispatch
    [spawn(fun() -> dispatch(H, Event) end) || H <- AllHandlers],
    {noreply, State};

handle_cast({unsubscribe, EventType, Handler}, State) ->
    ets:delete_object(?EVENT_TABLE, {EventType, Handler}),
    {noreply, State}.

dispatch(Fun, Event) when is_function(Fun) ->
    catch Fun(Event);
dispatch({Module, Function}, Event) ->
    catch Module:Function(Event);
dispatch(Pid, Event) when is_pid(Pid) ->
    Pid ! {event, Event}.

%% Usage example:
setup_handlers() ->
    event_bus:subscribe(order_created, fun handle_order_created/1),
    event_bus:subscribe(user_registered, fun send_welcome_email/1),
    event_bus:subscribe('*', fun audit_log/1).

handle_order_created(#{payload := #{order_id := Id}}) ->
    inventory:reserve(Id),
    notifications:send_order_confirmation(Id).

send_welcome_email(#{payload := #{user_id := UserId}}) ->
    email:send_welcome(UserId).

audit_log(Event) ->
    audit:log(Event).
```

---

## CQRS Pattern

```erlang
%% CQRS: Command Query Responsibility Segregation
%% Commands: write side (changes state)
%% Queries: read side (returns data, no state change)

%% Command handler
-module(command_handler).
-export([handle/1]).

handle({create_order, Command}) ->
    case validate_create_order(Command) of
        ok ->
            #{user_id := UserId, items := Items} = Command,
            OrderId = generate_order_id(),
            %% Write to event store
            Events = [
                {order_created, #{
                    order_id => OrderId,
                    user_id => UserId,
                    items => Items,
                    timestamp => erlang:system_time(millisecond)
                }},
                {inventory_reserved, #{order_id => OrderId, items => Items}}
            ],
            event_store:append(OrderId, Events),
            %% Publish to event bus
            [event_bus:publish(Type, Payload) || {Type, Payload} <- Events],
            {ok, OrderId};
        {error, Reason} ->
            {error, Reason}
    end;

handle({cancel_order, #{order_id := OrderId, reason := Reason}}) ->
    case order_query:get(OrderId) of
        {ok, #{status := Status}} when Status =:= pending ->
            Event = {order_cancelled, #{
                order_id => OrderId,
                reason => Reason,
                timestamp => erlang:system_time(millisecond)
            }},
            event_store:append(OrderId, [Event]),
            event_bus:publish(order_cancelled, element(2, Event)),
            ok;
        {ok, #{status := Status}} ->
            {error, {cannot_cancel, Status}};
        {error, Reason} ->
            {error, Reason}
    end.

validate_create_order(#{user_id := UserId, items := Items})
  when is_integer(UserId), length(Items) > 0 -> ok;
validate_create_order(_) ->
    {error, invalid_command}.

%% Query handler: Read model (denormalized, optimized for reads)
-module(order_query).
-export([get/1, list_by_user/2, get_stats/0]).

%% Read from materialized view (maintained by projections)
get(OrderId) ->
    case ets:lookup(order_read_model, OrderId) of
        [{_, Order}] -> {ok, Order};
        [] -> {error, not_found}
    end.

list_by_user(UserId, Opts) ->
    Page = maps:get(page, Opts, 1),
    PageSize = maps:get(page_size, Opts, 20),
    AllOrders = ets:match_object(order_read_model, {'_', #{user_id => UserId, '_' => '_'}}),
    paginate(AllOrders, Page, PageSize).

get_stats() ->
    Total = ets:info(order_read_model, size),
    #{total_orders => Total}.

paginate(List, Page, Size) ->
    Start = (Page - 1) * Size,
    Slice = lists:sublist(lists:nthtail(Start, List), Size),
    #{items => Slice, page => Page, total => length(List)}.

%% Projection: keep read model up to date
-module(order_projection).
-export([project/1]).

project({order_created, #{order_id := Id, user_id := UserId, items := Items}}) ->
    Order = #{
        id => Id,
        user_id => UserId,
        items => Items,
        status => pending,
        total => calculate_total(Items),
        created_at => erlang:system_time(millisecond)
    },
    ets:insert(order_read_model, {Id, Order});

project({order_cancelled, #{order_id := Id, reason := Reason}}) ->
    case ets:lookup(order_read_model, Id) of
        [{_, Order}] ->
            Updated = Order#{status => cancelled, cancel_reason => Reason},
            ets:insert(order_read_model, {Id, Updated});
        [] -> ok
    end;

project({payment_received, #{order_id := Id}}) ->
    case ets:lookup(order_read_model, Id) of
        [{_, Order}] ->
            ets:insert(order_read_model, {Id, Order#{status => paid}});
        [] -> ok
    end.
```

---

## ตัวอย่างจริง: Order Processing Pipeline

```erlang
%% order_pipeline.erl: async event-driven order processing

%% Flow:
%% 1. API receives order → publish to Kafka
%% 2. Inventory service: check/reserve items
%% 3. Payment service: charge
%% 4. Notification service: send email/SMS
%% 5. Fulfillment service: start delivery

%% Order API handler
-module(orders_api).

create_order(UserId, Items, PaymentInfo) ->
    OrderId = generate_id(),
    %% Publish order.created event
    kafka_client:produce(<<"orders">>, OrderId, #{
        type => <<"order.created">>,
        order_id => OrderId,
        user_id => UserId,
        items => Items,
        payment => PaymentInfo,
        timestamp => erlang:system_time(millisecond)
    }),
    {accepted, OrderId}.  %% Return immediately, process async

%% Pipeline stages (each is a Kafka consumer)

%% Stage 1: Inventory check
-module(inventory_consumer).
handle_event(#{<<"type">> := <<"order.created">>, <<"order_id">> := OrderId,
               <<"items">> := Items}) ->
    case inventory:check_and_reserve(Items) of
        {ok, ReservationId} ->
            kafka_client:produce(<<"orders">>, OrderId, #{
                type => <<"inventory.reserved">>,
                order_id => OrderId,
                reservation_id => ReservationId
            });
        {error, out_of_stock} ->
            kafka_client:produce(<<"orders">>, OrderId, #{
                type => <<"order.failed">>,
                order_id => OrderId,
                reason => <<"out_of_stock">>
            })
    end.

%% Stage 2: Payment
-module(payment_consumer).
handle_event(#{<<"type">> := <<"inventory.reserved">>, <<"order_id">> := OrderId}) ->
    %% Load order details from read model
    {ok, Order} = order_query:get(OrderId),
    case payment:charge(Order) of
        {ok, PaymentId} ->
            kafka_client:produce(<<"orders">>, OrderId, #{
                type => <<"payment.processed">>,
                order_id => OrderId,
                payment_id => PaymentId
            });
        {error, Reason} ->
            %% Compensating transaction: cancel inventory reservation
            inventory:cancel_reservation(maps:get(reservation_id, Order)),
            kafka_client:produce(<<"orders">>, OrderId, #{
                type => <<"order.failed">>,
                order_id => OrderId,
                reason => Reason
            })
    end.

%% Stage 3: Fulfillment
-module(fulfillment_consumer).
handle_event(#{<<"type">> := <<"payment.processed">>, <<"order_id">> := OrderId}) ->
    {ok, Order} = order_query:get(OrderId),
    ShipmentId = fulfillment:create_shipment(Order),
    notifications:send_order_shipped(Order, ShipmentId).
```

---

## สรุป Part 47

| Pattern | When to Use |
|---------|------------|
| Event Notification | Simple async workflows |
| Message Queue (RabbitMQ) | Task queues, retry logic |
| Event Streaming (Kafka) | High-throughput, replay |
| Internal Event Bus | In-process pub/sub |
| CQRS | Complex read/write requirements |
| Event Sourcing | Audit trail, time travel |

---

*[← Part 46: GraphQL](part_46_graphql.md) | [Part 48: Microservices →](part_48_microservices.md)*
