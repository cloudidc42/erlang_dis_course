# Part 66: RabbitMQ Internals และ Advanced Patterns

## สารบัญ
1. [AMQP Protocol](#amqp)
2. [Exchange Types](#exchanges)
3. [Queue Patterns](#queue-patterns)
4. [Reliability Patterns](#reliability)
5. [RabbitMQ Clustering](#clustering)
6. [ตัวอย่างจริง: Event-Driven Order System](#event-driven)

---

## AMQP Protocol

```erlang
%% AMQP: Advanced Message Queuing Protocol
%% RabbitMQ implements AMQP 0-9-1

%% Connection hierarchy:
%% Connection → Channel → Exchange/Queue/Binding

-module(rabbitmq_client).
-export([start/0, publish/3, consume/2]).

start() ->
    %% Connection (TCP)
    {ok, Connection} = amqp_connection:start(#amqp_params_network{
        host = "localhost",
        port = 5672,
        username = <<"guest">>,
        password = <<"guest">>,
        virtual_host = <<"/">>,
        heartbeat = 60
    }),
    
    %% Channel (multiplexed over connection)
    {ok, Channel} = amqp_connection:open_channel(Connection),
    
    %% Setup topology
    setup_topology(Channel),
    
    {ok, Connection, Channel}.

setup_topology(Channel) ->
    %% Declare exchange
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"orders">>,
        type = <<"topic">>,
        durable = true
    }),
    
    %% Declare queue
    #'queue.declare_ok'{queue = Queue} = amqp_channel:call(Channel, #'queue.declare'{
        queue = <<"order_notifications">>,
        durable = true,
        arguments = [
            {<<"x-message-ttl">>, long, 86400000},    %% 24h TTL
            {<<"x-dead-letter-exchange">>, longstr, <<"orders.dlx">>},
            {<<"x-max-length">>, long, 100000}
        ]
    }),
    
    %% Bind queue to exchange
    amqp_channel:call(Channel, #'queue.bind'{
        queue = Queue,
        exchange = <<"orders">>,
        routing_key = <<"order.#">>  %% matches order.created, order.cancelled, etc.
    }),
    
    Queue.

publish(Channel, RoutingKey, Payload) ->
    Msg = #amqp_msg{
        props = #'P_basic'{
            content_type = <<"application/json">>,
            delivery_mode = 2,        %% persistent
            message_id = generate_message_id(),
            timestamp = erlang:system_time(second)
        },
        payload = Payload
    },
    amqp_channel:call(Channel, #'basic.publish'{
        exchange = <<"orders">>,
        routing_key = RoutingKey,
        mandatory = true   %% fail if no queue matched
    }, Msg).

consume(Channel, Queue) ->
    %% QoS: process 10 messages at a time (back-pressure)
    amqp_channel:call(Channel, #'basic.qos'{prefetch_count = 10}),
    
    %% Start consuming
    amqp_channel:subscribe(Channel, #'basic.consume'{
        queue = Queue,
        no_ack = false  %% manual ack required
    }, self()).

generate_message_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).
```

---

## Exchange Types

```erlang
%% Direct Exchange: route by exact routing_key match
setup_direct(Channel) ->
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"direct_ex">>,
        type = <<"direct">>
    }),
    amqp_channel:call(Channel, #'queue.bind'{
        queue = <<"email_queue">>,
        exchange = <<"direct_ex">>,
        routing_key = <<"email">>
    }).

%% Fanout Exchange: broadcast to all bound queues (no routing key)
setup_fanout(Channel) ->
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"broadcast">>,
        type = <<"fanout">>
    }).
    %% All queues bound to broadcast receive every message

%% Topic Exchange: route by pattern (#=multi-word, *=one-word)
%% Example patterns: "order.#", "*.payment.*", "user.created"
setup_topic(Channel) ->
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"events">>,
        type = <<"topic">>
    }),
    %% order_svc gets all order events
    amqp_channel:call(Channel, #'queue.bind'{
        queue = <<"order_events">>,
        exchange = <<"events">>,
        routing_key = <<"order.#">>
    }),
    %% audit_log gets everything
    amqp_channel:call(Channel, #'queue.bind'{
        queue = <<"audit_log">>,
        exchange = <<"events">>,
        routing_key = <<"#">>
    }).

%% Headers Exchange: route by message headers
setup_headers(Channel) ->
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"headers_ex">>,
        type = <<"headers">>
    }),
    amqp_channel:call(Channel, #'queue.bind'{
        queue = <<"premium_q">>,
        exchange = <<"headers_ex">>,
        routing_key = <<"">>,
        arguments = [
            {<<"x-match">>, longstr, <<"all">>},
            {<<"tier">>, longstr, <<"premium">>},
            {<<"region">>, longstr, <<"us">>}
        ]
    }).
```

---

## Queue Patterns

```erlang
%% Dead Letter Queue: catch rejected/expired messages
setup_dlq(Channel) ->
    %% Dead letter exchange
    amqp_channel:call(Channel, #'exchange.declare'{
        exchange = <<"orders.dlx">>,
        type = <<"topic">>
    }),
    
    %% Dead letter queue
    amqp_channel:call(Channel, #'queue.declare'{
        queue = <<"orders.dead">>,
        durable = true
    }),
    
    amqp_channel:call(Channel, #'queue.bind'{
        queue = <<"orders.dead">>,
        exchange = <<"orders.dlx">>,
        routing_key = <<"#">>
    }),
    
    %% Main queue with DLQ configured
    amqp_channel:call(Channel, #'queue.declare'{
        queue = <<"orders.main">>,
        durable = true,
        arguments = [
            {<<"x-dead-letter-exchange">>, longstr, <<"orders.dlx">>},
            {<<"x-message-ttl">>, long, 300000}  %% 5min TTL
        ]
    }).

%% Priority Queue
setup_priority_queue(Channel) ->
    amqp_channel:call(Channel, #'queue.declare'{
        queue = <<"priority_tasks">>,
        durable = true,
        arguments = [{<<"x-max-priority">>, long, 10}]  %% priority 0-10
    }).

publish_priority(Channel, Task, Priority) ->
    Msg = #amqp_msg{
        props = #'P_basic'{
            priority = Priority,  %% 0=lowest, 10=highest
            delivery_mode = 2
        },
        payload = jsx:encode(Task)
    },
    amqp_channel:call(Channel, #'basic.publish'{
        exchange = <<"">>,
        routing_key = <<"priority_tasks">>
    }, Msg).

%% Delayed Message Queue (using RabbitMQ Delayed Message Plugin)
publish_delayed(Channel, Message, DelayMs) ->
    Msg = #amqp_msg{
        props = #'P_basic'{
            headers = [{<<"x-delay">>, long, DelayMs}],
            delivery_mode = 2
        },
        payload = jsx:encode(Message)
    },
    amqp_channel:call(Channel, #'basic.publish'{
        exchange = <<"delayed_exchange">>,
        routing_key = <<"scheduled_tasks">>
    }, Msg).
```

---

## ตัวอย่างจริง: Event-Driven Order System

```erlang
%% order_event_consumer.erl: consume and process order events

-module(order_event_consumer).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_info/2, handle_call/3]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init(_) ->
    {ok, Conn, Channel} = rabbitmq_client:start(),
    Queue = rabbitmq_client:setup_topology(Channel),
    rabbitmq_client:consume(Channel, Queue),
    {ok, #{channel => Channel, pending_acks => #{}}}.

%% Handle incoming message
handle_info({#'basic.deliver'{delivery_tag = Tag, routing_key = RK}, Msg}, 
            #{channel := Channel, pending_acks := Pending} = State) ->
    Payload = Msg#amqp_msg.payload,
    Event = jsx:decode(Payload, [return_maps]),
    
    %% Process asynchronously
    Self = self(),
    spawn(fun() ->
        Result = process_event(RK, Event),
        Self ! {event_processed, Tag, Result}
    end),
    
    {noreply, State#{pending_acks := Pending#{Tag => processing}}};

%% Ack after successful processing
handle_info({event_processed, Tag, ok}, #{channel := Channel} = State) ->
    amqp_channel:cast(Channel, #'basic.ack'{delivery_tag = Tag}),
    {noreply, State};

%% Nack on failure (requeue or DLQ)
handle_info({event_processed, Tag, {error, Reason}}, #{channel := Channel} = State) ->
    logger:error("Event processing failed", #{tag => Tag, reason => Reason}),
    amqp_channel:cast(Channel, #'basic.nack'{
        delivery_tag = Tag,
        requeue = false  %% send to DLQ
    }),
    {noreply, State}.

process_event(<<"order.created">>, Event) ->
    OrderId = maps:get(<<"order_id">>, Event),
    logger:info("Processing new order", #{order_id => OrderId}),
    notification_service:send_confirmation(OrderId),
    fulfillment_service:start_fulfillment(OrderId);

process_event(<<"order.cancelled">>, Event) ->
    OrderId = maps:get(<<"order_id">>, Event),
    notification_service:send_cancellation(OrderId);

process_event(<<"order.shipped">>, Event) ->
    OrderId = maps:get(<<"order_id">>, Event),
    TrackingNo = maps:get(<<"tracking_number">>, Event),
    notification_service:send_shipping(OrderId, TrackingNo);

process_event(RoutingKey, _) ->
    logger:warning("Unknown event", #{routing_key => RoutingKey}),
    ok.
```

---

## สรุป Part 66

| Exchange Type | Pattern | Use Case |
|---------------|---------|----------|
| Direct | exact key | Task queues |
| Fanout | broadcast | Notifications |
| Topic | wildcard patterns | Events routing |
| Headers | header match | Complex routing |
| Default | queue name | Simple publish |

**Key settings**: `durable=true`, `no_ack=false`, `prefetch_count` for back-pressure

---

*[← Part 65: Production System](part_65_production_system.md) | [Part 67: Kafka Integration →](part_67_kafka.md)*
