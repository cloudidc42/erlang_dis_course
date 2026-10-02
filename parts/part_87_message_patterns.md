# Part 87: Advanced Message Patterns

## สารบัญ
1. [Request-Reply Patterns](#request-reply)
2. [Publish-Subscribe with pg](#pubsub)
3. [Dead Letter Queue](#dlq)
4. [Message Deduplication](#dedup)
5. [RabbitMQ Integration](#rabbitmq)
6. [ตัวอย่างจริง: Event Bus](#eventbus)

---

## Request-Reply Patterns

```erlang
%% Synchronous request with timeout
request(Pid, Message, Timeout) ->
    Ref = make_ref(),
    Pid ! {request, self(), Ref, Message},
    receive
        {reply, Ref, Response} -> {ok, Response}
    after Timeout ->
        {error, timeout}
    end.

%% Async request with correlation ID
async_request(Pid, Message) ->
    CorrelationId = make_ref(),
    Pid ! {async_request, self(), CorrelationId, Message},
    CorrelationId.

collect_reply(CorrelationId, Timeout) ->
    receive
        {async_reply, CorrelationId, Response} -> {ok, Response}
    after Timeout ->
        {error, timeout}
    end.

%% Scatter-gather: send to multiple, collect all replies
scatter_gather(Workers, Message, Timeout) ->
    Refs = [{W, async_request(W, Message)} || W <- Workers],
    
    collect_all(Refs, Timeout, #{}).

collect_all([], _Timeout, Results) ->
    {ok, Results};
collect_all(Remaining, Timeout, Results) ->
    T1 = erlang:monotonic_time(millisecond),
    
    receive
        {async_reply, Ref, Response} ->
            case lists:keyfind(Ref, 2, Remaining) of
                {Worker, Ref} ->
                    Elapsed = erlang:monotonic_time(millisecond) - T1,
                    NewRemaining = lists:keydelete(Ref, 2, Remaining),
                    collect_all(NewRemaining, max(0, Timeout - Elapsed),
                                Results#{Worker => Response});
                false ->
                    collect_all(Remaining, Timeout, Results)
            end
    after Timeout ->
        %% Return partial results with timeouts
        Timeouts = [{W, timeout} || {W, _} <- Remaining],
        {partial, maps:merge(Results, maps:from_list(Timeouts))}
    end.
```

---

## Publish-Subscribe with pg

```erlang
%% event_bus.erl — cluster-wide pub/sub using pg

-module(event_bus).
-export([start_link/0, subscribe/2, unsubscribe/2, publish/2, publish_local/2]).

-define(SCOPE, event_bus).

start_link() ->
    pg:start_link(?SCOPE).

%% Subscribe to a topic (cluster-wide)
subscribe(Topic, HandlerPid) ->
    pg:join(?SCOPE, Topic, HandlerPid).

%% Unsubscribe
unsubscribe(Topic, HandlerPid) ->
    pg:leave(?SCOPE, Topic, HandlerPid).

%% Publish to all subscribers across the cluster
publish(Topic, Event) ->
    Subscribers = pg:get_members(?SCOPE, Topic),
    [Pid ! {event, Topic, Event} || Pid <- Subscribers],
    ok.

%% Publish only to local node subscribers
publish_local(Topic, Event) ->
    Subscribers = pg:get_local_members(?SCOPE, Topic),
    [Pid ! {event, Topic, Event} || Pid <- Subscribers],
    ok.

%% Wildcard topic matching (custom implementation)
-module(topic_router).
-export([subscribe/2, publish/2]).

-define(TOPIC_TABLE, topic_subscriptions).

init() ->
    ets:new(?TOPIC_TABLE, [named_table, bag, public, {read_concurrency, true}]).

subscribe(TopicPattern, Pid) ->
    ets:insert(?TOPIC_TABLE, {TopicPattern, Pid}),
    ok.

publish(Topic, Event) ->
    %% Find matching subscriptions (exact and wildcard)
    Parts = binary:split(Topic, <<".">>, [global]),
    Patterns = generate_patterns(Parts),
    
    Subscribers = lists:usort(lists:flatmap(fun(P) ->
        [Pid || {_, Pid} <- ets:lookup(?TOPIC_TABLE, P)]
    end, Patterns)),
    
    [Pid ! {event, Topic, Event} || Pid <- Subscribers],
    ok.

%% Generate wildcard patterns: "a.b.c" → ["a.b.c", "a.b.*", "a.*", "*"]
generate_patterns(Parts) ->
    generate_patterns(Parts, []).

generate_patterns(Parts, Acc) ->
    Full = binary_join(Parts, <<".">>),
    NewAcc = [Full | Acc],
    
    case lists:reverse(Parts) of
        [] -> NewAcc;
        [_ | Rest] ->
            WildParts = lists:reverse(Rest) ++ [<<"*">>],
            generate_patterns(lists:reverse(Rest), [binary_join(WildParts, <<".">>) | NewAcc])
    end.

binary_join([], _Sep) -> <<>>;
binary_join([H | T], Sep) ->
    lists:foldl(fun(P, Acc) -> <<Acc/binary, Sep/binary, P/binary>> end, H, T).
```

---

## Dead Letter Queue

```erlang
%% dlq.erl — dead letter queue for failed messages

-module(dlq).
-behaviour(gen_server).
-export([start_link/0, enqueue/3, process/1, list/0]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(dead_letter, {
    id,
    original_topic,
    message,
    failure_reason,
    attempts = 0,
    failed_at,
    last_retry_at
}).

init([]) ->
    ets:new(dead_letters, [named_table, ordered_set, public]),
    {ok, #{}}.

%% Move failed message to DLQ
enqueue(Topic, Message, FailureReason) ->
    Id = generate_id(),
    Entry = #dead_letter{
        id = Id,
        original_topic = Topic,
        message = Message,
        failure_reason = FailureReason,
        failed_at = erlang:system_time(second)
    },
    ets:insert(dead_letters, {Id, Entry}),
    
    %% Alert for DLQ growth
    Count = ets:info(dead_letters, size),
    case Count > 1000 of
        true -> alert_manager:notify(dlq_size_high, #{count => Count});
        false -> ok
    end,
    
    {ok, Id}.

%% Retry processing a dead letter
process(DeadLetterId) ->
    case ets:lookup(dead_letters, DeadLetterId) of
        [{_, #dead_letter{original_topic = Topic, message = Message, attempts = N} = DL}] ->
            case event_bus:publish(Topic, Message) of
                ok ->
                    ets:delete(dead_letters, DeadLetterId),
                    {ok, requeued};
                {error, Reason} ->
                    UpdatedDL = DL#dead_letter{
                        attempts = N + 1,
                        failure_reason = Reason,
                        last_retry_at = erlang:system_time(second)
                    },
                    ets:insert(dead_letters, {DeadLetterId, UpdatedDL}),
                    {error, Reason}
            end;
        [] ->
            {error, not_found}
    end.

list() ->
    [{Id, DL} || {Id, DL} <- ets:tab2list(dead_letters)].

generate_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).

%% Auto-retry worker
-module(dlq_retry_worker).
-behaviour(gen_server).

init(_) ->
    erlang:send_after(60000, self(), retry_failed),
    {ok, #{}}.

handle_info(retry_failed, State) ->
    DeadLetters = dlq:list(),
    
    Retryable = [Id || {Id, DL} <- DeadLetters,
                       DL#dead_letter.attempts < 5,
                       should_retry_now(DL)],
    
    [dlq:process(Id) || Id <- Retryable],
    
    erlang:send_after(60000, self(), retry_failed),
    {noreply, State}.

should_retry_now(#dead_letter{last_retry_at = undefined, failed_at = At, attempts = 0}) ->
    %% First retry after 5 minutes
    erlang:system_time(second) - At > 300;
should_retry_now(#dead_letter{last_retry_at = LastRetry, attempts = N}) ->
    %% Exponential backoff: 5, 15, 45, 135 minutes
    BackoffSeconds = 300 * trunc(math:pow(3, N - 1)),
    erlang:system_time(second) - LastRetry > BackoffSeconds.
```

---

## Message Deduplication

```erlang
%% message_dedup.erl — idempotent message processing

-module(message_dedup).
-export([process_once/3, is_processed/1, mark_processed/2]).

-define(DEDUP_TABLE, message_dedup).
-define(TTL_SECONDS, 86400).  %% Keep dedup records for 24 hours

init() ->
    ets:new(?DEDUP_TABLE, [named_table, set, public, {read_concurrency, true}]).

%% Process message only if not already processed
process_once(MessageId, Fun, Timeout) ->
    case is_processed(MessageId) of
        true ->
            {ok, already_processed};
        false ->
            %% Race condition: multiple processes may try simultaneously
            %% Use ETS insert with check to win the race
            case ets:insert_new(?DEDUP_TABLE, {MessageId, in_progress, erlang:system_time(second)}) of
                true ->
                    %% We won the race, process the message
                    try
                        Result = Fun(),
                        mark_processed(MessageId, Result),
                        {ok, Result}
                    catch Class:Reason ->
                        %% Remove from dedup so it can be retried
                        ets:delete(?DEDUP_TABLE, MessageId),
                        erlang:raise(Class, Reason, erlang:get_stacktrace())
                    end;
                false ->
                    %% Another process is handling it
                    wait_for_result(MessageId, Timeout)
            end
    end.

is_processed(MessageId) ->
    case ets:lookup(?DEDUP_TABLE, MessageId) of
        [{_, processed, _}] -> true;
        _ -> false
    end.

mark_processed(MessageId, _Result) ->
    ExpiresAt = erlang:system_time(second) + ?TTL_SECONDS,
    ets:insert(?DEDUP_TABLE, {MessageId, processed, ExpiresAt}).

wait_for_result(MessageId, Timeout) ->
    wait_for_result(MessageId, Timeout, erlang:monotonic_time(millisecond)).

wait_for_result(_MessageId, 0, _Start) ->
    {error, timeout};
wait_for_result(MessageId, Timeout, Start) ->
    timer:sleep(50),
    Elapsed = erlang:monotonic_time(millisecond) - Start,
    
    case ets:lookup(?DEDUP_TABLE, MessageId) of
        [{_, processed, _}] -> {ok, already_processed};
        [{_, in_progress, _}] ->
            Remaining = Timeout - Elapsed,
            wait_for_result(MessageId, max(0, Remaining), Start);
        [] ->
            %% Processing failed and was removed
            {error, processing_failed}
    end.

%% Cleanup expired dedup entries
cleanup() ->
    Now = erlang:system_time(second),
    ets:select_delete(?DEDUP_TABLE, [{{'_', processed, '$1'}, [{'<', '$1', Now}], [true]}]).
```

---

## RabbitMQ Integration

```erlang
%% rabbitmq_consumer.erl — reliable RabbitMQ consumer

-module(rabbitmq_consumer).
-behaviour(gen_server).
-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-include_lib("amqp_client/include/amqp_client.hrl").

-record(state, {
    connection,
    channel,
    queue,
    handler_mod,
    prefetch_count = 10
}).

start_link(Config) ->
    gen_server:start_link(?MODULE, Config, []).

init(#{queue := Queue, handler := HandlerMod} = Config) ->
    {ok, Connection} = amqp_connection:start(#amqp_params_network{
        host = maps:get(host, Config, "localhost"),
        port = maps:get(port, Config, 5672),
        username = maps:get(username, Config, <<"guest">>),
        password = maps:get(password, Config, <<"guest">>),
        virtual_host = maps:get(vhost, Config, <<"/">>)
    }),
    
    {ok, Channel} = amqp_connection:open_channel(Connection),
    
    %% Set prefetch (QoS)
    amqp_channel:call(Channel, #'basic.qos'{prefetch_count = maps:get(prefetch, Config, 10)}),
    
    %% Declare queue
    amqp_channel:call(Channel, #'queue.declare'{
        queue = Queue,
        durable = true
    }),
    
    %% Start consuming
    amqp_channel:subscribe(Channel, #'basic.consume'{queue = Queue}, self()),
    
    {ok, #state{connection = Connection, channel = Channel, queue = Queue,
                handler_mod = HandlerMod}}.

handle_info(#'basic.consume_ok'{}, State) ->
    {noreply, State};

handle_info({#'basic.deliver'{delivery_tag = Tag}, #amqp_msg{payload = Payload}},
            #state{channel = Channel, handler_mod = HandlerMod} = State) ->
    
    Message = jiffy:decode(Payload, [return_maps]),
    
    case HandlerMod:handle_message(Message) of
        ok ->
            %% Acknowledge successful processing
            amqp_channel:cast(Channel, #'basic.ack'{delivery_tag = Tag});
        {error, Reason} ->
            logger:error("Message processing failed", #{reason => Reason}),
            %% Nack with requeue=false (sends to DLQ if configured)
            amqp_channel:cast(Channel, #'basic.nack'{
                delivery_tag = Tag,
                requeue = false
            })
    end,
    
    {noreply, State};

handle_info(#'basic.cancel'{}, State) ->
    %% RabbitMQ cancelled our consumer (e.g. queue deleted)
    logger:warning("Consumer cancelled by broker"),
    {stop, normal, State}.

%% Publish to RabbitMQ
-module(rabbitmq_publisher).
-export([publish/3]).

publish(Exchange, RoutingKey, Message) ->
    Channel = get_channel(),
    
    Payload = jiffy:encode(Message),
    
    amqp_channel:cast(Channel, #'basic.publish'{
        exchange = Exchange,
        routing_key = RoutingKey
    }, #amqp_msg{
        props = #'P_basic'{
            delivery_mode = 2,  %% persistent
            content_type = <<"application/json">>,
            message_id = base64:encode(crypto:strong_rand_bytes(9))
        },
        payload = Payload
    }).

get_channel() ->
    %% Get from pool
    rabbitmq_channel_pool:checkout().
```

---

## ตัวอย่างจริง: Event Bus

```erlang
%% Full event bus with routing, filtering, and error handling

-module(full_event_bus).
-behaviour(gen_server).
-export([start_link/0, subscribe/3, unsubscribe/2, publish/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(subscription, {
    id,
    topic_pattern,
    filter_fun,    %% optional predicate
    handler_pid,
    handler_fun
}).

init([]) ->
    ets:new(subscriptions, [named_table, bag, public, {read_concurrency, true}]),
    {ok, #{}}.

subscribe(TopicPattern, FilterFun, HandlerFun) ->
    SubId = make_ref(),
    Sub = #subscription{
        id = SubId,
        topic_pattern = TopicPattern,
        filter_fun = FilterFun,
        handler_pid = self(),
        handler_fun = HandlerFun
    },
    ets:insert(subscriptions, {TopicPattern, Sub}),
    {ok, SubId}.

unsubscribe(SubId, TopicPattern) ->
    ets:match_delete(subscriptions, {TopicPattern, #subscription{id = SubId, _ = '_'}}).

publish(Topic, Event) ->
    Subscriptions = find_matching_subscriptions(Topic),
    
    Passes = lists:filter(fun(Sub) ->
        case Sub#subscription.filter_fun of
            undefined -> true;
            Fun -> Fun(Event)
        end
    end, Subscriptions),
    
    [deliver(Sub, Topic, Event) || Sub <- Passes],
    ok.

deliver(#subscription{handler_fun = Fun, handler_pid = Pid}, Topic, Event) ->
    %% Non-blocking delivery in subscriber's process
    spawn(fun() ->
        try Fun(Topic, Event)
        catch Class:Reason ->
            logger:error("Event handler failed", #{
                class => Class, reason => Reason, topic => Topic
            })
        end
    end).

find_matching_subscriptions(Topic) ->
    Parts = binary:split(Topic, <<".">>, [global]),
    Patterns = generate_patterns(Parts),
    
    lists:usort(lists:flatmap(fun(P) ->
        [Sub || {_, Sub} <- ets:lookup(subscriptions, P)]
    end, Patterns)).

generate_patterns([]) -> [<<"*">>];
generate_patterns(Parts) ->
    ExactTopic = iolist_to_binary(lists:join(<<".">>, Parts)),
    WildPatterns = [begin
        Prefix = lists:sublist(Parts, N),
        iolist_to_binary(lists:join(<<".">>, Prefix ++ [<<"*">>]))
    end || N <- lists:seq(1, length(Parts) - 1)],
    [ExactTopic, <<"*">> | WildPatterns].
```

---

## สรุป Part 87

| Pattern | Use Case |
|---------|---------|
| Request-Reply | Synchronous coordination |
| Scatter-Gather | Parallel fan-out + collect |
| Pub-Sub (pg) | Cluster-wide events |
| Dead Letter Queue | Failed message handling |
| Deduplication | Exactly-once processing |
| RabbitMQ | External message broker |

**Key insight**: Erlang's message passing naturally implements many messaging patterns — often without a broker.

---

*[← Part 86: Distributed Coordination](part_86_distributed_coordination.md) | [Part 88: Observability →](part_88_observability.md)*
