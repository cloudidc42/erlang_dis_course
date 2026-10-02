# Part 59: Large-Scale System Design

## สารบัญ
1. [WhatsApp Scale Architecture](#whatsapp-scale)
2. [Discord/Telegram Patterns](#discord-telegram)
3. [Real-time Presence System](#presence-system)
4. [Message Delivery Guarantees](#message-delivery)
5. [Horizontal Scaling](#horizontal-scaling)
6. [ตัวอย่างจริง: Chat Platform Core](#chat-platform)

---

## WhatsApp Scale Architecture

```
WhatsApp (2014): 450 million users, 2 engineers per billion users
Built with Erlang/FreeBSD

Key design decisions:
1. Single Erlang node per user connection (process = user session)
2. Ejabberd XMPP server (Erlang)
3. Custom Mnesia + ETS for session data
4. FreeBSD for high file descriptor limits
5. ~2 million concurrent connections per server

Architecture:
Client → Load Balancer → Gateway (Erlang) → Message Router → Storage
                                           ↓
                                     Presence Server
                                     Push Notifications
                                     Media Server

Erlang contribution:
- Each connected user = 1 Erlang process (~2KB)
- 2 million processes per node
- Fault tolerance: process crashes don't affect others
- No garbage collection pauses (per-process GC)
```

---

## Discord/Telegram Patterns

```erlang
%% Discord: Erlang → Elixir migration
%% Key pattern: Guild (server) as single process

%% guild_process.erl
-module(guild_process).
-behaviour(gen_server).
-export([start_link/1, send_message/3, get_members/1]).

start_link(GuildId) ->
    gen_server:start_link({via, registry, {guild, GuildId}}, ?MODULE, GuildId, []).

send_message(GuildId, ChannelId, Message) ->
    gen_server:cast(via_name(GuildId), {send_message, ChannelId, Message}).

get_members(GuildId) ->
    gen_server:call(via_name(GuildId), get_members).

via_name(GuildId) ->
    {via, registry, {guild, GuildId}}.

init(GuildId) ->
    State = #{
        guild_id => GuildId,
        members => #{},      %% UserId → MemberInfo
        channels => #{},     %% ChannelId → ChannelState
        voice_states => #{}  %% UserId → VoiceState
    },
    {ok, State}.

handle_cast({send_message, ChannelId, Message}, State) ->
    %% Fan out to all members connected to this channel
    Members = get_channel_members(ChannelId, State),
    [send_to_user(MemberId, Message) || MemberId <- Members],
    {noreply, update_last_message(ChannelId, Message, State)};

handle_call(get_members, _From, #{members := Members} = State) ->
    {reply, Members, State}.

send_to_user(UserId, Message) ->
    case user_session:find(UserId) of
        {ok, SessionPid} ->
            SessionPid ! {push_message, Message};
        {error, offline} ->
            push_notification:send(UserId, Message)
    end.

%% Telegram key pattern: MTProto + custom auth
%% High throughput message routing
-module(telegram_router).
-export([route_message/2]).

route_message(#{to := UserId, payload := Payload} = Msg, State) ->
    Shard = user_shard(UserId),
    NodeForShard = shard_to_node(Shard),
    case net_adm:ping(NodeForShard) of
        pong ->
            rpc:call(NodeForShard, user_delivery, deliver, [UserId, Payload]);
        pang ->
            %% Failover to replica
            Replica = replica_node(NodeForShard),
            rpc:call(Replica, user_delivery, deliver, [UserId, Payload])
    end.

user_shard(UserId) ->
    erlang:phash2(UserId, 256).

shard_to_node(Shard) ->
    Nodes = pg:get_members(shard_nodes),
    lists:nth((Shard rem length(Nodes)) + 1, Nodes).
```

---

## Real-time Presence System

```erlang
%% Presence: track who is online, last seen, typing status
%% Requires: very frequent updates, low latency, eventual consistency

-module(presence_manager).
-behaviour(gen_server).
-export([start_link/0, set_online/2, set_offline/1, get_presence/1]).
-export([subscribe_presence/2, broadcast_status/2]).

-define(PRESENCE_TTL, 30).  %% seconds

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

set_online(UserId, Meta) ->
    gen_server:cast(?MODULE, {online, UserId, Meta}).

set_offline(UserId) ->
    gen_server:cast(?MODULE, {offline, UserId}).

get_presence(UserId) ->
    gen_server:call(?MODULE, {get, UserId}).

subscribe_presence(SubscriberId, TargetUserId) ->
    gen_server:cast(?MODULE, {subscribe, SubscriberId, TargetUserId}).

init(_) ->
    %% ETS for fast local reads
    ets:new(presence, [named_table, set, public, {read_concurrency, true}]),
    ets:new(presence_subs, [named_table, bag, public]),
    %% Timer to expire stale presence
    {ok, _} = timer:send_interval(5000, cleanup_stale),
    {ok, #{}}.

handle_cast({online, UserId, Meta}, State) ->
    Now = erlang:system_time(second),
    ets:insert(presence, {UserId, online, Meta, Now}),
    notify_subscribers(UserId, online),
    {noreply, State};

handle_cast({offline, UserId}, State) ->
    Now = erlang:system_time(second),
    ets:insert(presence, {UserId, offline, #{}, Now}),
    notify_subscribers(UserId, offline),
    {noreply, State};

handle_cast({subscribe, SubId, TargetId}, State) ->
    ets:insert(presence_subs, {TargetId, SubId}),
    {noreply, State};

handle_call({get, UserId}, _From, State) ->
    Presence = case ets:lookup(presence, UserId) of
        [{_, Status, Meta, LastSeen}] ->
            #{status => Status, meta => Meta, last_seen => LastSeen};
        [] ->
            #{status => unknown}
    end,
    {reply, Presence, State};

handle_info(cleanup_stale, State) ->
    Cutoff = erlang:system_time(second) - ?PRESENCE_TTL,
    %% Find and expire stale online entries
    Stale = ets:select(presence, [
        {{'$1', online, '_', '$2'}, [{'<', '$2', Cutoff}], ['$1']}
    ]),
    [begin
        ets:insert(presence, {UserId, offline, #{}, Cutoff}),
        notify_subscribers(UserId, offline)
    end || UserId <- Stale],
    {noreply, State}.

notify_subscribers(UserId, Status) ->
    Subscribers = ets:lookup(presence_subs, UserId),
    [begin
        case user_session:find(SubId) of
            {ok, Pid} -> Pid ! {presence_update, UserId, Status};
            _ -> ok
        end
    end || {_, SubId} <- Subscribers].

broadcast_status(UserId, Status) ->
    %% Fan out to all nodes in cluster
    [rpc:call(N, presence_manager, set_online, [UserId, Status])
     || N <- nodes()].
```

---

## Message Delivery Guarantees

```erlang
%% At-most-once, at-least-once, exactly-once

%% At-most-once (fire and forget)
send_fire_forget(UserId, Message) ->
    case user_session:find(UserId) of
        {ok, Pid} -> Pid ! {message, Message};
        _ -> ok  %% message lost if offline
    end.

%% At-least-once (retry until ack)
-module(reliable_sender).
-behaviour(gen_server).
-export([send/3]).

send(UserId, Message, Opts) ->
    MaxRetries = maps:get(max_retries, Opts, 5),
    RetryInterval = maps:get(retry_ms, Opts, 1000),
    MsgId = generate_id(),
    gen_server:cast(?MODULE, {send, MsgId, UserId, Message, MaxRetries, RetryInterval}).

handle_cast({send, MsgId, UserId, Message, Retries, _}, State) when Retries =< 0 ->
    logger:error("Failed to deliver message", #{msg_id => MsgId, user => UserId}),
    metrics:increment(message_delivery_failed),
    {noreply, State};

handle_cast({send, MsgId, UserId, Message, Retries, Interval}, State) ->
    case attempt_delivery(UserId, MsgId, Message) of
        ok ->
            metrics:increment(message_delivered),
            {noreply, State};
        {error, _} ->
            erlang:send_after(Interval, self(),
                {retry, MsgId, UserId, Message, Retries - 1, Interval * 2}),
            {noreply, State}
    end.

handle_info({retry, MsgId, UserId, Message, Retries, Interval}, State) ->
    handle_cast({send, MsgId, UserId, Message, Retries, Interval}, State).

attempt_delivery(UserId, MsgId, Message) ->
    case user_session:find(UserId) of
        {ok, Pid} ->
            Pid ! {message, MsgId, Message},
            {ok, sent};
        {error, offline} ->
            offline_queue:enqueue(UserId, MsgId, Message)
    end.

%% Exactly-once: idempotency key
process_message(MsgId, UserId, Message) ->
    case dedup_store:already_processed(MsgId) of
        true ->
            {ok, duplicate_ignored};
        false ->
            Result = do_process(UserId, Message),
            dedup_store:mark_processed(MsgId),
            {ok, Result}
    end.
```

---

## ตัวอย่างจริง: Chat Platform Core

```erlang
%% Minimal but scalable chat platform
%% Handles: rooms, messages, presence, history

-module(chat_platform).
-export([start/0]).

start() ->
    chat_sup:start_link().

%% chat_room.erl
-module(chat_room).
-behaviour(gen_server).
-export([start_link/1, join/2, leave/2, send_message/3]).

-define(HISTORY_SIZE, 100).

start_link(RoomId) ->
    gen_server:start_link({via, gproc, {n, l, {room, RoomId}}}, ?MODULE, RoomId, []).

join(RoomId, User) ->
    gen_server:call(via(RoomId), {join, User}).

leave(RoomId, UserId) ->
    gen_server:cast(via(RoomId), {leave, UserId}).

send_message(RoomId, From, Text) ->
    gen_server:cast(via(RoomId), {message, From, Text}).

via(RoomId) -> {via, gproc, {n, l, {room, RoomId}}}.

init(RoomId) ->
    {ok, #{
        room_id => RoomId,
        members => #{},
        history => queue:new(),
        msg_count => 0
    }}.

handle_call({join, #{id := UserId} = User}, _From, State) ->
    #{members := Members, history := History} = State,
    NewMembers = Members#{UserId => User},
    %% Send recent history to new member
    HistoryList = queue:to_list(History),
    {reply, {ok, HistoryList}, State#{members := NewMembers}};

handle_cast({leave, UserId}, #{members := Members} = State) ->
    NewMembers = maps:remove(UserId, Members),
    broadcast_system(<<"user_left">>, UserId, State),
    {noreply, State#{members := NewMembers}};

handle_cast({message, From, Text}, State) ->
    #{members := Members, history := History, msg_count := N} = State,
    Msg = #{
        id => N + 1,
        from => From,
        text => Text,
        ts => erlang:system_time(millisecond)
    },
    %% Update history (bounded)
    NewHistory = case queue:len(History) >= ?HISTORY_SIZE of
        true ->
            queue:in(Msg, queue:drop(History));
        false ->
            queue:in(Msg, History)
    end,
    %% Broadcast to all members
    [send_to_user(UserId, {chat_message, Msg}) ||
        {UserId, _} <- maps:to_list(Members),
        UserId /= maps:get(id, From)],
    {noreply, State#{history := NewHistory, msg_count := N + 1}}.

broadcast_system(Event, Data, #{members := Members}) ->
    Msg = #{type => system, event => Event, data => Data},
    [send_to_user(UserId, {chat_message, Msg}) || {UserId, _} <- maps:to_list(Members)].

send_to_user(UserId, Msg) ->
    case gproc:whereis_name({n, l, {user_session, UserId}}) of
        Pid when is_pid(Pid) -> Pid ! Msg;
        undefined -> ok
    end.
```

---

## สรุป Part 59

| Scale Lesson | Erlang Solution |
|--------------|-----------------|
| Millions of connections | Lightweight processes (2KB each) |
| Per-user state | gen_server per user |
| Real-time updates | message passing, no polling |
| Fault isolation | process crash doesn't affect others |
| Horizontal scaling | Erlang clustering (nodes) |
| Presence | ETS + TTL cleanup |
| Message delivery | retry + idempotency keys |

---

*[← Part 58: Actor Model](part_58_actor_model.md) | [Part 60: Raft Consensus →](part_60_raft.md)*
