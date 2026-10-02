# Part 94: Final Project — Real-Time Chat System

## สารบัญ
1. [Architecture Overview](#architecture)
2. [WebSocket Connection Handler](#websocket)
3. [Room Management](#rooms)
4. [Message Persistence](#persistence)
5. [Presence System](#presence)
6. [ตัวอย่างจริง: Complete Chat Server](#complete)

---

## Architecture Overview

```
Chat System Architecture:
                                                    
  Browser ──── WebSocket ──── chat_conn (cowboy_ws)
                                    │
                                    │ join_room/send_msg
                                    ▼
                              room_manager (gen_server)
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                   room_process           room_process
                   (per room)             (per room)
                         │
                    ┌────┴────┐
                    │         │
               pg (pubsub)   db_store
               (delivery)   (history)

  presence_tracker: tracks online users per room
  typing_tracker: tracks who's typing (ephemeral)
```

---

## WebSocket Connection Handler

```erlang
%% chat_conn.erl — WebSocket handler for chat

-module(chat_conn).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

-record(state, {
    user_id,
    username,
    rooms = [],
    last_heartbeat
}).

%% HTTP upgrade to WebSocket
init(Req, _State) ->
    %% Authenticate via token in query string
    QS = cowboy_req:parse_qs(Req),
    Token = proplists:get_value(<<"token">>, QS),
    
    case auth:verify_token(Token) of
        {ok, #{user_id := UserId, username := Username}} ->
            {cowboy_websocket, Req, #state{user_id = UserId, username = Username},
             #{idle_timeout => 60000, max_frame_size => 65536}};
        {error, _} ->
            Req2 = cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
            {ok, Req2, undefined}
    end.

websocket_init(State) ->
    %% Send welcome message
    Welcome = #{type => <<"connected">>, user_id => State#state.user_id,
                username => State#state.username},
    {reply, {text, jiffy:encode(Welcome)}, State}.

%% Handle messages from browser
websocket_handle({text, Data}, State) ->
    case jiffy:decode(Data, [return_maps]) of
        #{<<"type">> := Type} = Msg ->
            handle_client_message(Type, Msg, State);
        _ ->
            error_reply(<<"invalid_message">>, State)
    end;
websocket_handle(ping, State) ->
    {reply, pong, State#state{last_heartbeat = erlang:system_time(second)}};
websocket_handle(_Frame, State) ->
    {ok, State}.

handle_client_message(<<"join_room">>, #{<<"room_id">> := RoomId}, State) ->
    case room_manager:join(RoomId, State#state.user_id, self()) of
        {ok, History} ->
            NewState = State#state{rooms = [RoomId | State#state.rooms]},
            
            %% Send room history
            HistoryMsg = #{type => <<"room_history">>, room_id => RoomId,
                            messages => History},
            
            %% Send current members
            Members = presence_tracker:get_members(RoomId),
            MembersMsg = #{type => <<"room_members">>, room_id => RoomId,
                            members => Members},
            
            {reply, [
                {text, jiffy:encode(HistoryMsg)},
                {text, jiffy:encode(MembersMsg)}
            ], NewState};
        {error, Reason} ->
            error_reply(Reason, State)
    end;

handle_client_message(<<"send_message">>, #{<<"room_id">> := RoomId, 
                                             <<"content">> := Content}, State) ->
    case lists:member(RoomId, State#state.rooms) of
        true ->
            Message = #{
                id => generate_message_id(),
                room_id => RoomId,
                user_id => State#state.user_id,
                username => State#state.username,
                content => Content,
                timestamp => erlang:system_time(millisecond)
            },
            room_manager:broadcast(RoomId, Message),
            {ok, State};
        false ->
            error_reply(<<"not_in_room">>, State)
    end;

handle_client_message(<<"typing">>, #{<<"room_id">> := RoomId, 
                                       <<"is_typing">> := IsTyping}, State) ->
    typing_tracker:update(RoomId, State#state.user_id, IsTyping),
    {ok, State};

handle_client_message(<<"leave_room">>, #{<<"room_id">> := RoomId}, State) ->
    room_manager:leave(RoomId, State#state.user_id),
    NewState = State#state{rooms = lists:delete(RoomId, State#state.rooms)},
    {ok, NewState};

handle_client_message(_, _, State) ->
    error_reply(<<"unknown_message_type">>, State).

%% Handle messages from room processes
websocket_info({chat_message, Message}, State) ->
    {reply, {text, jiffy:encode(Message#{type => <<"message">>})}, State};

websocket_info({user_joined, RoomId, User}, State) ->
    Msg = #{type => <<"user_joined">>, room_id => RoomId, user => User},
    {reply, {text, jiffy:encode(Msg)}, State};

websocket_info({user_left, RoomId, UserId}, State) ->
    Msg = #{type => <<"user_left">>, room_id => RoomId, user_id => UserId},
    {reply, {text, jiffy:encode(Msg)}, State};

websocket_info({typing_update, RoomId, TypingUsers}, State) ->
    Msg = #{type => <<"typing">>, room_id => RoomId, users => TypingUsers},
    {reply, {text, jiffy:encode(Msg)}, State};

websocket_info(_Info, State) ->
    {ok, State}.

terminate(_Reason, _Req, State) when is_record(State, state) ->
    %% Clean up: leave all rooms
    [room_manager:leave(R, State#state.user_id) || R <- State#state.rooms],
    ok;
terminate(_Reason, _Req, _State) ->
    ok.

error_reply(Reason, State) ->
    Msg = #{type => <<"error">>, reason => Reason},
    {reply, {text, jiffy:encode(Msg)}, State}.

generate_message_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).
```

---

## Room Management

```erlang
%% room_manager.erl — manages chat rooms

-module(room_manager).
-behaviour(gen_server).
-export([start_link/0, join/3, leave/2, broadcast/2, get_rooms/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    rooms = #{}    %% room_id => room_pid
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

join(RoomId, UserId, ConnPid) ->
    %% Get or create room process
    gen_server:call(?MODULE, {join, RoomId, UserId, ConnPid}).

leave(RoomId, UserId) ->
    gen_server:cast(?MODULE, {leave, RoomId, UserId}).

broadcast(RoomId, Message) ->
    gen_server:cast(?MODULE, {broadcast, RoomId, Message}).

init([]) ->
    {ok, #state{}}.

handle_call({join, RoomId, UserId, ConnPid}, _From, #state{rooms = Rooms} = State) ->
    RoomPid = case maps:get(RoomId, Rooms, undefined) of
        undefined ->
            %% Create new room
            {ok, Pid} = room:start_link(RoomId),
            Pid;
        Pid ->
            Pid
    end,
    
    Result = room:join(RoomPid, UserId, ConnPid),
    NewRooms = maps:put(RoomId, RoomPid, Rooms),
    {reply, Result, State#state{rooms = NewRooms}};

handle_call(get_rooms, _From, #state{rooms = Rooms} = State) ->
    {reply, maps:keys(Rooms), State}.

handle_cast({leave, RoomId, UserId}, #state{rooms = Rooms} = State) ->
    case maps:get(RoomId, Rooms, undefined) of
        undefined -> {noreply, State};
        RoomPid ->
            room:leave(RoomPid, UserId),
            {noreply, State}
    end;

handle_cast({broadcast, RoomId, Message}, #state{rooms = Rooms} = State) ->
    case maps:get(RoomId, Rooms, undefined) of
        undefined -> {noreply, State};
        RoomPid ->
            room:broadcast(RoomPid, Message),
            {noreply, State}
    end.

handle_info({'DOWN', _Ref, process, Pid, _Reason}, #state{rooms = Rooms} = State) ->
    %% Room process died, remove from registry
    NewRooms = maps:filter(fun(_, RPid) -> RPid =/= Pid end, Rooms),
    {noreply, State#state{rooms = NewRooms}}.

%% room.erl — per-room process
-module(room).
-behaviour(gen_server).
-export([start_link/1, join/3, leave/2, broadcast/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    room_id,
    members = #{},   %% user_id => conn_pid
    message_count = 0
}).

start_link(RoomId) ->
    gen_server:start_link(?MODULE, RoomId, []).

join(RoomPid, UserId, ConnPid) ->
    gen_server:call(RoomPid, {join, UserId, ConnPid}).

leave(RoomPid, UserId) ->
    gen_server:cast(RoomPid, {leave, UserId}).

broadcast(RoomPid, Message) ->
    gen_server:cast(RoomPid, {broadcast, Message}).

init(RoomId) ->
    process_flag(trap_exit, true),
    {ok, #state{room_id = RoomId}}.

handle_call({join, UserId, ConnPid}, _From, #state{members = Members, room_id = RoomId} = State) ->
    %% Monitor the connection
    monitor(process, ConnPid),
    
    %% Get recent message history
    History = message_store:get_history(RoomId, 50),
    
    %% Notify existing members
    User = #{user_id => UserId},
    [ConnP ! {user_joined, RoomId, User} || {_, ConnP} <- maps:to_list(Members)],
    
    NewMembers = maps:put(UserId, ConnPid, Members),
    presence_tracker:mark_online(RoomId, UserId),
    
    {reply, {ok, History}, State#state{members = NewMembers}};

handle_cast({leave, UserId}, #state{members = Members, room_id = RoomId} = State) ->
    NewMembers = maps:remove(UserId, Members),
    presence_tracker:mark_offline(RoomId, UserId),
    
    %% Notify remaining members
    [ConnP ! {user_left, RoomId, UserId} || {_, ConnP} <- maps:to_list(NewMembers)],
    
    {noreply, State#state{members = NewMembers}};

handle_cast({broadcast, #{room_id := RoomId} = Message}, #state{members = Members} = State) ->
    %% Store message
    message_store:save(Message),
    
    %% Deliver to all members
    [ConnP ! {chat_message, Message} || {_, ConnP} <- maps:to_list(Members)],
    
    {noreply, State#state{message_count = State#state.message_count + 1}}.
```

---

## Message Persistence

```erlang
%% message_store.erl — chat message persistence

-module(message_store).
-export([save/1, get_history/2, get_since/2]).

%% Save message to PostgreSQL
save(#{id := Id, room_id := RoomId, user_id := UserId,
       username := Username, content := Content, timestamp := Ts}) ->
    db:execute(
        "INSERT INTO messages (id, room_id, user_id, username, content, created_at) "
        "VALUES ($1, $2, $3, $4, $5, to_timestamp($6::bigint / 1000.0))",
        [Id, RoomId, UserId, Username, Content, Ts]
    ).

%% Get last N messages for a room
get_history(RoomId, Limit) ->
    case db:execute(
        "SELECT id, user_id, username, content, "
        "       extract(epoch from created_at) * 1000 as ts "
        "FROM messages "
        "WHERE room_id = $1 "
        "ORDER BY created_at DESC "
        "LIMIT $2",
        [RoomId, Limit]
    ) of
        {ok, Rows} ->
            lists:reverse([row_to_message(R, RoomId) || R <- Rows]);
        _ ->
            []
    end.

get_since(RoomId, SinceTimestamp) ->
    case db:execute(
        "SELECT id, user_id, username, content, "
        "       extract(epoch from created_at) * 1000 as ts "
        "FROM messages "
        "WHERE room_id = $1 AND created_at > to_timestamp($2::bigint / 1000.0) "
        "ORDER BY created_at",
        [RoomId, SinceTimestamp]
    ) of
        {ok, Rows} -> [row_to_message(R, RoomId) || R <- Rows];
        _ -> []
    end.

row_to_message({Id, UserId, Username, Content, Ts}, RoomId) ->
    #{
        id => Id,
        room_id => RoomId,
        user_id => UserId,
        username => Username,
        content => Content,
        timestamp => round(Ts)
    }.
```

---

## Presence System

```erlang
%% presence_tracker.erl — track online users per room

-module(presence_tracker).
-behaviour(gen_server).
-export([start_link/0, mark_online/2, mark_offline/2, get_members/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(PRESENCE_TABLE, presence).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    ets:new(?PRESENCE_TABLE, [named_table, bag, public, {read_concurrency, true}]),
    {ok, #{}}.

mark_online(RoomId, UserId) ->
    Timestamp = erlang:system_time(second),
    ets:insert(?PRESENCE_TABLE, {{RoomId, UserId}, Timestamp}).

mark_offline(RoomId, UserId) ->
    ets:delete_object(?PRESENCE_TABLE, {{RoomId, UserId}, '_'}).

get_members(RoomId) ->
    [{UserId, Ts} || {{_R, UserId}, Ts} <- ets:match_object(?PRESENCE_TABLE, {{RoomId, '_'}, '_'})].

%% typing_tracker.erl — ephemeral typing indicators
-module(typing_tracker).
-export([update/3, get_typing/1]).

-define(TYPING_TABLE, typing_users).
-define(TYPING_TIMEOUT_MS, 5000).

update(RoomId, UserId, true) ->
    ets:insert(?TYPING_TABLE, {{RoomId, UserId}, erlang:system_time(millisecond)}),
    notify_room(RoomId);
update(RoomId, UserId, false) ->
    ets:delete(?TYPING_TABLE, {RoomId, UserId}),
    notify_room(RoomId).

get_typing(RoomId) ->
    Now = erlang:system_time(millisecond),
    Cutoff = Now - ?TYPING_TIMEOUT_MS,
    
    [UserId || {{_R, UserId}, Ts} <- ets:match_object(?TYPING_TABLE, {{RoomId, '_'}, '_'}),
               Ts > Cutoff].

notify_room(RoomId) ->
    TypingUsers = get_typing(RoomId),
    Members = presence_tracker:get_members(RoomId),
    %% Broadcast to room via room process
    room_manager:broadcast_typing(RoomId, TypingUsers, Members).
```

---

## ตัวอย่างจริง: Complete Chat Server

```erlang
%% chat_app.erl — application entry point

-module(chat_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_Type, _Args) ->
    %% Initialize storage
    init_schema(),
    
    %% Start supervision tree
    chat_sup:start_link().

-module(chat_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{strategy => one_for_one, intensity => 5, period => 10},
    
    Children = [
        #{id => db_pool, start => {db_pool, start_link, [#{size => 20}]},
          restart => permanent, shutdown => 5000, type => worker},
        
        #{id => presence_tracker, start => {presence_tracker, start_link, []},
          restart => permanent, shutdown => 5000, type => worker},
        
        #{id => room_manager, start => {room_manager, start_link, []},
          restart => permanent, shutdown => 5000, type => worker},
        
        #{id => chat_http, start => {chat_http, start_link, []},
          restart => permanent, shutdown => 5000, type => worker}
    ],
    
    {ok, {SupFlags, Children}}.

-module(chat_http).
-export([start_link/0]).

start_link() ->
    Port = application:get_env(chat, port, 8080),
    
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/ws", chat_conn, []},
            {"/api/rooms", rooms_handler, []},
            {"/api/rooms/:room_id/messages", messages_handler, []},
            {"/health", health_handler, []}
        ]}
    ]),
    
    cowboy:start_clear(chat_http, [{port, Port}, {num_acceptors, 100}], #{
        env => #{dispatch => Dispatch},
        middlewares => [cowboy_router, auth_middleware, cowboy_handler]
    }).

init_schema() ->
    db:execute("CREATE TABLE IF NOT EXISTS messages (
        id TEXT PRIMARY KEY,
        room_id TEXT NOT NULL,
        user_id TEXT NOT NULL,
        username TEXT NOT NULL,
        content TEXT NOT NULL,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    )", []),
    
    db:execute("CREATE INDEX IF NOT EXISTS idx_messages_room_time
        ON messages(room_id, created_at DESC)", []).
```

---

## สรุป Part 94

| Component | Implementation |
|-----------|---------------|
| WebSocket | cowboy_websocket behaviour |
| Room state | gen_server per room |
| Delivery | Direct process messages via stored PIDs |
| History | PostgreSQL with room+time index |
| Presence | ETS table (bag type) |
| Typing | ETS with TTL-based cleanup |

**Scalability**: This design scales horizontally — use pg (process groups) and distributed Erlang to span multiple nodes, with Redis for cross-node presence.

---

*[← Part 93: BEAM Internals](part_93_beam_internals.md) | [Part 95: Final Project — E-Commerce Platform →](part_95_final_project_ecommerce.md)*
