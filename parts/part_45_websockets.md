# Part 45: WebSockets Advanced — Real-Time Applications

## สารบัญ
1. [WebSocket Protocol](#websocket-protocol)
2. [Cowboy WebSocket Handler](#cowboy-websocket-handler)
3. [Connection Management](#connection-management)
4. [Pub/Sub Pattern](#pubsub-pattern)
5. [Room-based Chat](#room-based-chat)
6. [ตัวอย่างจริง: Real-Time Dashboard](#ตัวอย่างจริง-real-time-dashboard)

---

## WebSocket Protocol

```
WebSocket: full-duplex communication over single TCP connection
- HTTP Upgrade: GET /ws HTTP/1.1 → 101 Switching Protocols
- Frame-based: text frame, binary frame, ping, pong, close
- Low latency: ไม่ต้องสร้าง connection ใหม่ทุกครั้ง
- กว่า HTTP long-polling มาก
```

---

## Cowboy WebSocket Handler

```erlang
%% ws_handler.erl: basic WebSocket handler
-module(ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

%% HTTP upgrade to WebSocket
init(Req, Opts) ->
    %% Extract query params before upgrade
    UserId = cowboy_req:binding(user_id, Req),
    Token = cowboy_req:header(<<"sec-websocket-protocol">>, Req),
    case auth:verify_token(Token) of
        {ok, Claims} ->
            State = #{user_id => UserId, claims => Claims},
            {cowboy_websocket, Req, State, #{
                idle_timeout => 300000,  %% 5 min idle timeout
                max_frame_size => 65536  %% 64KB max frame
            }};
        {error, _} ->
            Req2 = cowboy_req:reply(401, Req),
            {ok, Req2, Opts}
    end.

%% Called after WebSocket handshake
websocket_init(State) ->
    %% Register this connection
    connection_registry:register(maps:get(user_id, State), self()),
    %% Send welcome message
    Welcome = jsx:encode(#{type => <<"welcome">>,
                           user_id => maps:get(user_id, State)}),
    {[{text, Welcome}], State}.

%% Handle incoming frames from client
websocket_handle({text, Data}, State) ->
    case jsx:decode(Data, [return_maps]) of
        #{<<"type">> := <<"ping">>} ->
            Reply = jsx:encode(#{type => <<"pong">>}),
            {[{text, Reply}], State};

        #{<<"type">> := <<"subscribe">>, <<"topic">> := Topic} ->
            pubsub:subscribe(Topic, self()),
            NewState = State#{subscriptions => [Topic | maps:get(subscriptions, State, [])]},
            Ack = jsx:encode(#{type => <<"subscribed">>, topic => Topic}),
            {[{text, Ack}], NewState};

        #{<<"type">> := <<"message">>, <<"topic">> := Topic, <<"data">> := MsgData} ->
            pubsub:publish(Topic, MsgData),
            {ok, State};

        _ ->
            Error = jsx:encode(#{type => <<"error">>, message => <<"Unknown message type">>}),
            {[{text, Error}], State}
    end;
websocket_handle({binary, Data}, State) ->
    %% Handle binary frames (e.g., file uploads)
    {ok, State};
websocket_handle({ping, _}, State) ->
    {ok, State};  %% cowboy auto-replies with pong
websocket_handle({pong, _}, State) ->
    {ok, State}.

%% Handle messages from Erlang processes
websocket_info({pubsub_message, Topic, Data}, State) ->
    Msg = jsx:encode(#{type => <<"message">>, topic => Topic, data => Data}),
    {[{text, Msg}], State};

websocket_info({send, Data}, State) ->
    {[{text, Data}], State};

websocket_info(heartbeat, State) ->
    Msg = jsx:encode(#{type => <<"heartbeat">>, time => erlang:system_time(millisecond)}),
    erlang:send_after(30000, self(), heartbeat),
    {[{text, Msg}], State};

websocket_info(_Info, State) ->
    {ok, State}.

%% Cleanup on disconnect
terminate(Reason, _Req, State) ->
    UserId = maps:get(user_id, State),
    connection_registry:unregister(UserId, self()),
    Subs = maps:get(subscriptions, State, []),
    [pubsub:unsubscribe(T, self()) || T <- Subs],
    logger:info("WebSocket disconnected", #{user_id => UserId, reason => Reason}),
    ok.
```

---

## Connection Management

```erlang
%% connection_registry.erl: track all active connections
-module(connection_registry).
-behaviour(gen_server).
-export([start_link/0, register/2, unregister/2, get_connections/1,
         send_to_user/2, send_to_all/1, count/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(UserId, Pid) ->
    gen_server:call(?MODULE, {register, UserId, Pid}).

unregister(UserId, Pid) ->
    gen_server:cast(?MODULE, {unregister, UserId, Pid}).

get_connections(UserId) ->
    gen_server:call(?MODULE, {get_connections, UserId}).

send_to_user(UserId, Message) ->
    gen_server:cast(?MODULE, {send_to_user, UserId, Message}).

send_to_all(Message) ->
    gen_server:cast(?MODULE, {send_to_all, Message}).

count() ->
    gen_server:call(?MODULE, count).

init(_) ->
    %% ETS: {user_id, pid} — multiple pids per user (multi-device)
    ets:new(connections, [named_table, bag, public]),
    {ok, #{}}.

handle_call({register, UserId, Pid}, _From, State) ->
    monitor(process, Pid),
    ets:insert(connections, {UserId, Pid}),
    {reply, ok, State};

handle_call({get_connections, UserId}, _From, State) ->
    Pids = [Pid || {_, Pid} <- ets:lookup(connections, UserId)],
    {reply, Pids, State};

handle_call(count, _From, State) ->
    Count = ets:info(connections, size),
    {reply, Count, State}.

handle_cast({unregister, UserId, Pid}, State) ->
    ets:delete_object(connections, {UserId, Pid}),
    {noreply, State};

handle_cast({send_to_user, UserId, Message}, State) ->
    Pids = [Pid || {_, Pid} <- ets:lookup(connections, UserId)],
    lists:foreach(fun(Pid) -> Pid ! {send, Message} end, Pids),
    {noreply, State};

handle_cast({send_to_all, Message}, State) ->
    ets:foldl(fun({_, Pid}, _) ->
        Pid ! {send, Message}
    end, ok, connections),
    {noreply, State}.

handle_info({'DOWN', _Ref, process, Pid, _Reason}, State) ->
    ets:match_delete(connections, {'_', Pid}),
    {noreply, State}.
```

---

## Pub/Sub Pattern

```erlang
%% pubsub.erl: publish-subscribe for WebSocket messages
-module(pubsub).
-behaviour(gen_server).
-export([start_link/0, subscribe/2, unsubscribe/2, publish/2, topics/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

subscribe(Topic, Pid) ->
    gen_server:call(?MODULE, {subscribe, Topic, Pid}).

unsubscribe(Topic, Pid) ->
    gen_server:cast(?MODULE, {unsubscribe, Topic, Pid}).

publish(Topic, Message) ->
    gen_server:cast(?MODULE, {publish, Topic, Message}).

topics() ->
    gen_server:call(?MODULE, topics).

init(_) ->
    %% ETS: {topic, pid}
    ets:new(subscriptions, [named_table, bag, public]),
    {ok, #{}}.

handle_call({subscribe, Topic, Pid}, _From, State) ->
    ets:insert(subscriptions, {Topic, Pid}),
    monitor(process, Pid),
    {reply, ok, State};

handle_call(topics, _From, State) ->
    Topics = lists:usort([T || {T, _} <- ets:tab2list(subscriptions)]),
    {reply, Topics, State}.

handle_cast({unsubscribe, Topic, Pid}, State) ->
    ets:delete_object(subscriptions, {Topic, Pid}),
    {noreply, State};

handle_cast({publish, Topic, Message}, State) ->
    Subscribers = [Pid || {_, Pid} <- ets:lookup(subscriptions, Topic)],
    lists:foreach(fun(Pid) ->
        Pid ! {pubsub_message, Topic, Message}
    end, Subscribers),
    {noreply, State}.

handle_info({'DOWN', _Ref, process, Pid, _Reason}, State) ->
    ets:match_delete(subscriptions, {'_', Pid}),
    {noreply, State}.
```

---

## Room-based Chat

```erlang
%% chat_room.erl: WebSocket chat rooms
-module(chat_room).
-behaviour(gen_server).
-export([start_link/1, join/2, leave/2, send/3, get_history/1, members/1]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link(RoomId) ->
    gen_server:start_link(?MODULE, RoomId, []).

join(RoomPid, #{user_id := UserId, ws_pid := WsPid}) ->
    gen_server:call(RoomPid, {join, UserId, WsPid}).

leave(RoomPid, WsPid) ->
    gen_server:cast(RoomPid, {leave, WsPid}).

send(RoomPid, UserId, Message) ->
    gen_server:cast(RoomPid, {message, UserId, Message}).

get_history(RoomPid) ->
    gen_server:call(RoomPid, get_history).

members(RoomPid) ->
    gen_server:call(RoomPid, members).

init(RoomId) ->
    State = #{
        room_id => RoomId,
        members => #{},   %% WsPid -> UserId
        history => [],    %% last 50 messages
        max_history => 50
    },
    {ok, State}.

handle_call({join, UserId, WsPid}, _From, State) ->
    monitor(process, WsPid),
    NewMembers = maps:put(WsPid, UserId, maps:get(members, State)),
    NewState = State#{members := NewMembers},
    %% Notify others
    broadcast_to_all(
        jsx:encode(#{type => <<"user_joined">>, user_id => UserId}),
        NewMembers, WsPid
    ),
    %% Send history to new joiner
    History = maps:get(history, State),
    {reply, {ok, History}, NewState};

handle_call(get_history, _From, State) ->
    {reply, maps:get(history, State), State};

handle_call(members, _From, State) ->
    {reply, maps:values(maps:get(members, State)), State}.

handle_cast({leave, WsPid}, State) ->
    Members = maps:get(members, State),
    case maps:get(WsPid, Members, undefined) of
        undefined -> {noreply, State};
        UserId ->
            NewMembers = maps:remove(WsPid, Members),
            broadcast_to_all(
                jsx:encode(#{type => <<"user_left">>, user_id => UserId}),
                NewMembers, none
            ),
            {noreply, State#{members := NewMembers}}
    end;

handle_cast({message, UserId, Text}, State) ->
    Msg = #{
        type => <<"chat">>,
        user_id => UserId,
        text => Text,
        timestamp => erlang:system_time(millisecond)
    },
    Encoded = jsx:encode(Msg),
    Members = maps:get(members, State),
    broadcast_to_all(Encoded, Members, none),
    %% Update history
    History = maps:get(history, State),
    MaxH = maps:get(max_history, State),
    NewHistory = lists:sublist([Msg | History], MaxH),
    {noreply, State#{history := NewHistory}}.

broadcast_to_all(Msg, Members, ExcludePid) ->
    maps:foreach(fun(WsPid, _) when WsPid =/= ExcludePid ->
        WsPid ! {send, Msg};
    (_, _) -> ok
    end, Members).
```

---

## ตัวอย่างจริง: Real-Time Dashboard

```erlang
%% dashboard_ws.erl: real-time metrics dashboard
-module(dashboard_ws).
-export([init/2, websocket_init/1, websocket_handle/2, websocket_info/2]).

init(Req, Opts) ->
    {cowboy_websocket, Req, Opts, #{idle_timeout => 60000}}.

websocket_init(State) ->
    %% Subscribe to metrics updates
    metrics_broadcaster:subscribe(self()),
    %% Send initial snapshot
    Snapshot = get_current_metrics(),
    Msg = jsx:encode(#{type => <<"snapshot">>, data => Snapshot}),
    %% Schedule periodic updates
    erlang:send_after(5000, self(), tick),
    {[{text, Msg}], State}.

websocket_handle({text, Data}, State) ->
    case jsx:decode(Data, [return_maps]) of
        #{<<"type">> := <<"subscribe">>, <<"metric">> := Metric} ->
            metrics_broadcaster:subscribe_metric(Metric, self()),
            {ok, State#{metrics => [Metric | maps:get(metrics, State, [])]}};
        _ ->
            {ok, State}
    end.

websocket_info({metric_update, MetricName, Value}, State) ->
    Msg = jsx:encode(#{type => <<"update">>,
                       metric => MetricName,
                       value => Value,
                       timestamp => erlang:system_time(millisecond)}),
    {[{text, Msg}], State};

websocket_info(tick, State) ->
    Metrics = get_current_metrics(),
    Msg = jsx:encode(#{type => <<"tick">>, data => Metrics}),
    erlang:send_after(5000, self(), tick),
    {[{text, Msg}], State}.

get_current_metrics() ->
    #{
        memory_total => erlang:memory(total),
        process_count => erlang:system_info(process_count),
        message_queues => get_top_queues(),
        reductions => element(2, erlang:statistics(reductions))
    }.

get_top_queues() ->
    Processes = processes(),
    Queues = [{P, process_info(P, message_queue_len)} || P <- Processes],
    Top5 = lists:sublist(
        lists:reverse(lists:sort([{Q, P} || {P, {_, Q}} <- Queues])),
        5
    ),
    [#{pid => pid_to_list(P), queue_len => Q} || {Q, P} <- Top5].

%% metrics_broadcaster.erl
-module(metrics_broadcaster).
-behaviour(gen_server).
-export([start_link/0, subscribe/1, subscribe_metric/2, broadcast/2]).
-export([init/1, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

subscribe(Pid) ->
    gen_server:cast(?MODULE, {subscribe, all, Pid}).

subscribe_metric(Metric, Pid) ->
    gen_server:cast(?MODULE, {subscribe, Metric, Pid}).

broadcast(Metric, Value) ->
    gen_server:cast(?MODULE, {broadcast, Metric, Value}).

init(_) ->
    ets:new(subscribers, [named_table, bag]),
    {ok, #{}}.

handle_cast({subscribe, Topic, Pid}, State) ->
    ets:insert(subscribers, {Topic, Pid}),
    monitor(process, Pid),
    {noreply, State};

handle_cast({broadcast, Metric, Value}, State) ->
    %% Notify metric-specific subscribers
    [Pid ! {metric_update, Metric, Value}
     || {_, Pid} <- ets:lookup(subscribers, Metric)],
    %% Notify 'all' subscribers
    [Pid ! {metric_update, Metric, Value}
     || {_, Pid} <- ets:lookup(subscribers, all)],
    {noreply, State};

handle_info({'DOWN', _, process, Pid, _}, State) ->
    ets:match_delete(subscribers, {'_', Pid}),
    {noreply, State}.
```

---

## JavaScript Client

```javascript
// dashboard.js: WebSocket client

class DashboardClient {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.reconnectDelay = 1000;
        this.maxReconnectDelay = 30000;
    }

    connect() {
        const token = localStorage.getItem('auth_token');
        this.ws = new WebSocket(this.url, [token]);

        this.ws.onopen = () => {
            console.log('Connected');
            this.reconnectDelay = 1000;
            this.ws.send(JSON.stringify({type: 'subscribe', metric: 'memory'}));
        };

        this.ws.onmessage = (event) => {
            const msg = JSON.parse(event.data);
            this.handleMessage(msg);
        };

        this.ws.onclose = () => {
            console.log('Disconnected, reconnecting...');
            setTimeout(() => this.connect(), this.reconnectDelay);
            this.reconnectDelay = Math.min(this.reconnectDelay * 2, this.maxReconnectDelay);
        };
    }

    handleMessage(msg) {
        switch (msg.type) {
            case 'snapshot':
                updateDashboard(msg.data);
                break;
            case 'update':
                updateMetric(msg.metric, msg.value);
                break;
            case 'tick':
                updateDashboard(msg.data);
                break;
        }
    }

    send(data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        }
    }
}

const client = new DashboardClient('wss://api.example.com/ws/dashboard');
client.connect();
```

---

## สรุป Part 45

| Pattern | Use Case |
|---------|----------|
| Pub/Sub | Topic-based message broadcast |
| Room | Group chat, gaming |
| Connection Registry | User presence, direct messages |
| Dashboard | Real-time metrics |
| Heartbeat | Keep-alive, detect dead connections |

---

*[← Part 44: gRPC](part_44_grpc.md) | [Part 46: GraphQL →](part_46_graphql.md)*
