# Part 74: Game Server Patterns

## สารบัญ
1. [Game Loop Architecture](#game-loop)
2. [Player Connection Management](#connections)
3. [Game State Synchronization](#sync)
4. [Room/Match Management](#rooms)
5. [Anti-Cheat Patterns](#anti-cheat)
6. [ตัวอย่างจริง: Multiplayer Card Game Server](#card-game)

---

## Game Loop Architecture

```
Erlang ทำไมดีกับ Game Server:
- 1 process per player/connection (Actor Model)
- Message passing = event queue ตามธรรมชาติ
- Hot code reload = patch โดยไม่ต้อง restart
- Distributed = scale horizontally
- Low latency garbage collection

Architecture:
  Client → WebSocket → player_process (gen_server)
                              ↓
                        game_room (gen_server)
                              ↓
                     game_engine (pure functions)
```

```erlang
%% game_loop.erl — tick-based game loop

-module(game_loop).
-behaviour(gen_server).
-export([start_link/2, tick/1, get_state/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(TICK_RATE_MS, 50).   %% 20 ticks/second (typical for action games)

-record(loop_state, {
    game_id,
    tick = 0,
    game_state,
    players = #{},    %% PlayerId => player_data
    timer
}).

start_link(GameId, InitialState) ->
    gen_server:start_link(?MODULE, {GameId, InitialState}, []).

init({GameId, InitialState}) ->
    Timer = schedule_tick(),
    State = #loop_state{
        game_id = GameId,
        game_state = InitialState,
        timer = Timer
    },
    {ok, State}.

%% Scheduled tick
handle_info(tick, State) ->
    T1 = erlang:monotonic_time(millisecond),
    
    %% 1. Process queued player inputs
    Inputs = collect_inputs(State),
    
    %% 2. Apply game physics/logic
    NewGameState = game_engine:update(State#loop_state.game_state, Inputs, ?TICK_RATE_MS),
    
    %% 3. Detect game events
    Events = game_engine:detect_events(State#loop_state.game_state, NewGameState),
    
    %% 4. Broadcast state to players
    broadcast_game_state(State#loop_state.players, NewGameState, Events),
    
    %% 5. Handle game end condition
    NewState = case game_engine:is_over(NewGameState) of
        {true, Winner} ->
            handle_game_over(Winner, State),
            State#loop_state{game_state = NewGameState};
        false ->
            T2 = erlang:monotonic_time(millisecond),
            TickDuration = T2 - T1,
            
            %% Adaptive: if tick took too long, don't schedule more ticks
            NextTimer = case TickDuration > ?TICK_RATE_MS of
                true ->
                    logger:warning("Game tick overrun", #{game_id => State#loop_state.game_id, duration_ms => TickDuration}),
                    schedule_tick();
                false ->
                    Delay = ?TICK_RATE_MS - TickDuration,
                    schedule_tick_after(Delay)
            end,
            
            State#loop_state{
                tick = State#loop_state.tick + 1,
                game_state = NewGameState,
                timer = NextTimer
            }
    end,
    
    {noreply, NewState}.

%% Receive player input (fire-and-forget, no block)
handle_cast({player_input, PlayerId, Input}, State) ->
    %% Queue input for next tick
    NewPlayers = maps:update_with(PlayerId, fun(P) ->
        P#{input_queue => [Input | maps:get(input_queue, P, [])]}
    end, State#loop_state.players),
    {noreply, State#loop_state{players = NewPlayers}};

handle_cast({player_join, PlayerId, PlayerPid}, State) ->
    NewPlayers = maps:put(PlayerId, #{pid => PlayerPid, input_queue => []}, State#loop_state.players),
    {noreply, State#loop_state{players = NewPlayers}};

handle_cast({player_leave, PlayerId}, State) ->
    NewPlayers = maps:remove(PlayerId, State#loop_state.players),
    NewGameState = game_engine:remove_player(State#loop_state.game_state, PlayerId),
    {noreply, State#loop_state{players = NewPlayers, game_state = NewGameState}}.

handle_call(get_state, _From, State) ->
    {reply, State#loop_state.game_state, State}.

collect_inputs(#loop_state{players = Players}) ->
    maps:fold(fun(PlayerId, #{input_queue := Queue}, Acc) ->
        [{PlayerId, lists:reverse(Queue)} | Acc]  %% oldest first
    end, [], Players).

broadcast_game_state(Players, GameState, Events) ->
    Msg = #{
        type => <<"game_state">>,
        state => game_engine:serialize(GameState),
        events => Events
    },
    Encoded = jiffy:encode(Msg),
    [player_conn:send(maps:get(pid, P), Encoded) || P <- maps:values(Players)].

schedule_tick() ->
    erlang:send_after(?TICK_RATE_MS, self(), tick).

schedule_tick_after(Delay) ->
    erlang:send_after(max(0, Delay), self(), tick).

handle_game_over(Winner, State) ->
    notify_players(State#loop_state.players, Winner),
    match_manager:game_completed(State#loop_state.game_id, Winner).
```

---

## Player Connection Management

```erlang
%% player_conn.erl — WebSocket handler per player

-module(player_conn).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, websocket_terminate/3]).
-export([send/2]).

-record(state, {
    player_id,
    session_token,
    current_room,
    last_ping_at,
    latency_ms = 0
}).

init(Req, _Opts) ->
    Token = cowboy_req:header(<<"x-session-token">>, Req),
    case auth_service:validate_token(Token) of
        {ok, PlayerId} ->
            {cowboy_websocket, Req, #state{player_id = PlayerId, session_token = Token},
             #{idle_timeout => 60000, compress => true}};
        {error, invalid} ->
            Req2 = cowboy_req:reply(401, Req),
            {ok, Req2, undefined}
    end.

websocket_init(State) ->
    %% Register this connection
    player_registry:register(State#state.player_id, self()),
    
    %% Send initial state
    InitMsg = jiffy:encode(#{type => <<"connected">>, player_id => State#state.player_id}),
    {reply, {text, InitMsg}, State}.

websocket_handle({text, Data}, State) ->
    case jiffy:decode(Data, [return_maps]) of
        #{<<"type">> := <<"ping">>, <<"ts">> := ClientTs} ->
            %% Calculate latency
            ServerTs = erlang:system_time(millisecond),
            Latency = ServerTs - ClientTs,
            Pong = jiffy:encode(#{type => <<"pong">>, ts => ServerTs, latency => Latency}),
            {reply, {text, Pong}, State#state{latency_ms = Latency, last_ping_at = ServerTs}};
        
        #{<<"type">> := <<"join_room">>, <<"room_id">> := RoomId} ->
            handle_join_room(RoomId, State);
        
        #{<<"type">> := <<"input">>, <<"data">> := InputData} ->
            handle_player_input(InputData, State);
        
        #{<<"type">> := <<"leave_room">>} ->
            handle_leave_room(State);
        
        _ ->
            {ok, State}
    end;

websocket_handle(ping, State) ->
    {reply, pong, State}.

websocket_info({send, Data}, State) ->
    {reply, {text, Data}, State};

websocket_info({'DOWN', _, process, RoomPid, _}, #state{current_room = RoomPid} = State) ->
    %% Room crashed
    ErrorMsg = jiffy:encode(#{type => <<"error">>, reason => <<"room_crashed">>}),
    {reply, {text, ErrorMsg}, State#state{current_room = undefined}}.

websocket_terminate(_, _, State) ->
    %% Cleanup
    player_registry:unregister(State#state.player_id),
    case State#state.current_room of
        undefined -> ok;
        RoomPid -> game_room:player_leave(RoomPid, State#state.player_id)
    end,
    ok.

handle_join_room(RoomId, State) ->
    case match_manager:get_room(RoomId) of
        {ok, RoomPid} ->
            ok = game_room:player_join(RoomPid, State#state.player_id, self()),
            monitor(process, RoomPid),
            OkMsg = jiffy:encode(#{type => <<"joined_room">>, room_id => RoomId}),
            {reply, {text, OkMsg}, State#state{current_room = RoomPid}};
        {error, not_found} ->
            Err = jiffy:encode(#{type => <<"error">>, reason => <<"room_not_found">>}),
            {reply, {text, Err}, State}
    end.

handle_player_input(InputData, #state{current_room = undefined} = State) ->
    {ok, State};
handle_player_input(InputData, State) ->
    game_loop:player_input(State#state.current_room, State#state.player_id, InputData),
    {ok, State}.

handle_leave_room(#state{current_room = undefined} = State) ->
    {ok, State};
handle_leave_room(State) ->
    game_room:player_leave(State#state.current_room, State#state.player_id),
    OkMsg = jiffy:encode(#{type => <<"left_room">>}),
    {reply, {text, OkMsg}, State#state{current_room = undefined}}.

%% Send to player from game loop
send(PlayerPid, Data) ->
    PlayerPid ! {send, Data}.
```

---

## Room/Match Management

```erlang
%% match_manager.erl — matchmaking and room lifecycle

-module(match_manager).
-behaviour(gen_server).
-export([start_link/0, find_match/2, create_room/1, get_room/1, game_completed/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(MIN_PLAYERS, 2).
-define(MAX_PLAYERS, 4).
-define(MATCHMAKING_TIMEOUT_MS, 30000).

-record(mm_state, {
    rooms = #{},          %% RoomId => {pid, players, status}
    waiting_queue = [],   %% [{PlayerId, SkillRating, JoinedAt}]
    room_sup
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Player requests to find a match
find_match(PlayerId, SkillRating) ->
    gen_server:call(?MODULE, {find_match, PlayerId, SkillRating}).

%% Create a private room
create_room(Options) ->
    gen_server:call(?MODULE, {create_room, Options}).

get_room(RoomId) ->
    gen_server:call(?MODULE, {get_room, RoomId}).

game_completed(GameId, Winner) ->
    gen_server:cast(?MODULE, {game_completed, GameId, Winner}).

init([]) ->
    {ok, RoomSup} = game_room_sup:start_link(),
    {ok, #mm_state{room_sup = RoomSup}}.

handle_call({find_match, PlayerId, SkillRating}, _From, State) ->
    JoinedAt = erlang:monotonic_time(millisecond),
    Player = {PlayerId, SkillRating, JoinedAt},
    
    NewQueue = [Player | State#mm_state.waiting_queue],
    
    %% Try to form a match
    {Match, RemainingQueue} = try_match(NewQueue),
    
    NewState = State#mm_state{waiting_queue = RemainingQueue},
    
    Result = case Match of
        {matched, Players} ->
            RoomId = start_room(Players, NewState),
            {ok, {matched, RoomId}};
        waiting ->
            %% Schedule timeout
            erlang:send_after(?MATCHMAKING_TIMEOUT_MS, self(), {matchmaking_timeout, PlayerId}),
            {ok, waiting}
    end,
    
    {reply, Result, NewState};

handle_call({get_room, RoomId}, _From, #mm_state{rooms = Rooms} = State) ->
    case maps:get(RoomId, Rooms, undefined) of
        {Pid, _, _} -> {reply, {ok, Pid}, State};
        undefined -> {reply, {error, not_found}, State}
    end.

handle_cast({game_completed, GameId, Winner}, #mm_state{rooms = Rooms} = State) ->
    %% Record result, cleanup room reference
    record_match_result(GameId, Winner),
    {noreply, State#mm_state{rooms = maps:remove(GameId, Rooms)}}.

handle_info({matchmaking_timeout, PlayerId}, State) ->
    %% Remove from queue if still waiting
    NewQueue = [P || {PId, _, _} = P <- State#mm_state.waiting_queue, PId =/= PlayerId],
    
    case length(NewQueue) < length(State#mm_state.waiting_queue) of
        true ->
            %% Player was still waiting, notify them
            case player_registry:get_pid(PlayerId) of
                {ok, Pid} -> player_conn:send(Pid, jiffy:encode(#{type => <<"matchmaking_timeout">>}));
                _ -> ok
            end;
        false -> ok
    end,
    
    {noreply, State#mm_state{waiting_queue = NewQueue}}.

%% Skill-based matchmaking
try_match(Queue) ->
    SortedQueue = lists:sort(fun({_, R1, T1}, {_, R2, T2}) ->
        %% Sort by skill, then by wait time (longer wait = lower priority)
        case abs(R1 - R2) < 100 of
            true -> T1 < T2;
            false -> R1 < R2
        end
    end, Queue),
    
    case length(SortedQueue) >= ?MIN_PLAYERS of
        true ->
            Players = lists:sublist(SortedQueue, ?MIN_PLAYERS),
            Remaining = lists:nthtail(?MIN_PLAYERS, SortedQueue),
            {{matched, Players}, Remaining};
        false ->
            {waiting, SortedQueue}
    end.

start_room(Players, #mm_state{room_sup = Sup, rooms = Rooms} = State) ->
    RoomId = generate_room_id(),
    PlayerIds = [PId || {PId, _, _} <- Players],
    
    {ok, RoomPid} = supervisor:start_child(Sup, [RoomId, PlayerIds]),
    monitor(process, RoomPid),
    
    %% Notify players
    [notify_player(PId, {matched, RoomId}) || PId <- PlayerIds],
    
    RoomId.

generate_room_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).

notify_player(PlayerId, Notification) ->
    case player_registry:get_pid(PlayerId) of
        {ok, Pid} ->
            Msg = jiffy:encode(#{type => <<"matched">>, room_id => element(2, Notification)}),
            player_conn:send(Pid, Msg);
        _ -> ok
    end.

record_match_result(GameId, Winner) ->
    %% Store to database asynchronously
    spawn(fun() -> db:execute("INSERT INTO match_results (game_id, winner) VALUES ($1, $2)",
                              [GameId, Winner]) end).
```

---

## ตัวอย่างจริง: Multiplayer Card Game

```erlang
%% card_game_engine.erl — pure functions for game logic

-module(card_game_engine).
-export([new_game/1, deal_cards/1, play_card/3, is_over/1, serialize/1]).

-define(HAND_SIZE, 7).

%% Create new game state
new_game(PlayerIds) ->
    Deck = shuffle(full_deck()),
    
    {Hands, RemainingDeck} = deal_hands(Deck, PlayerIds, ?HAND_SIZE),
    
    [TopCard | DrawPile] = RemainingDeck,
    
    #{
        players => PlayerIds,
        hands => Hands,         %% #{PlayerId => [cards]}
        draw_pile => DrawPile,
        discard_pile => [TopCard],
        current_player => hd(PlayerIds),
        direction => clockwise,
        turn => 1
    }.

%% Play a card
play_card(GameState, PlayerId, Card) ->
    #{current_player := Current, hands := Hands} = GameState,
    
    %% Validate turn
    case Current of
        PlayerId ->
            Hand = maps:get(PlayerId, Hands),
            
            %% Validate card is in hand
            case lists:member(Card, Hand) of
                true ->
                    TopCard = hd(maps:get(discard_pile, GameState)),
                    
                    %% Validate play is legal
                    case is_valid_play(Card, TopCard) of
                        true ->
                            apply_card(GameState, PlayerId, Card);
                        false ->
                            {error, invalid_play}
                    end;
                false ->
                    {error, card_not_in_hand}
            end;
        _ ->
            {error, not_your_turn}
    end.

is_valid_play({Color, _}, {Color, _}) -> true;  %% same color
is_valid_play({_, Number}, {_, Number}) -> true; %% same number
is_valid_play({wild, _}, _) -> true;             %% wild card
is_valid_play(_, _) -> false.

apply_card(GameState, PlayerId, Card) ->
    #{hands := Hands, discard_pile := Discard} = GameState,
    
    %% Remove card from hand
    NewHand = lists:delete(Card, maps:get(PlayerId, Hands)),
    NewHands = Hands#{PlayerId => NewHand},
    
    %% Add to discard pile
    NewDiscard = [Card | Discard],
    
    %% Apply card effect
    StateAfterEffect = apply_card_effect(Card, GameState#{
        hands := NewHands,
        discard_pile := NewDiscard
    }),
    
    %% Advance turn
    FinalState = advance_turn(StateAfterEffect, PlayerId),
    
    {ok, FinalState}.

apply_card_effect({_, skip}, State) ->
    %% Skip next player
    advance_turn(State, maps:get(current_player, State));

apply_card_effect({_, reverse}, #{direction := clockwise} = State) ->
    State#{direction := counter_clockwise};
apply_card_effect({_, reverse}, State) ->
    State#{direction := clockwise};

apply_card_effect({_, draw_two}, State) ->
    NextPlayer = get_next_player(State),
    draw_cards(State, NextPlayer, 2);

apply_card_effect({wild, draw_four}, State) ->
    NextPlayer = get_next_player(State),
    draw_cards(State, NextPlayer, 4);

apply_card_effect(_, State) -> State.

advance_turn(State, _CurrentPlayer) ->
    Next = get_next_player(State),
    State#{current_player := Next, turn := maps:get(turn, State) + 1}.

get_next_player(#{players := Players, current_player := Current, direction := Dir}) ->
    Idx = index_of(Current, Players),
    N = length(Players),
    NextIdx = case Dir of
        clockwise -> (Idx + 1) rem N;
        counter_clockwise -> (Idx - 1 + N) rem N
    end,
    lists:nth(NextIdx + 1, Players).

draw_cards(#{draw_pile := [], discard_pile := [Top | Rest]} = State, PlayerId, N) ->
    %% Reshuffle discard pile into draw pile
    NewDraw = shuffle(Rest),
    draw_cards(State#{draw_pile := NewDraw, discard_pile := [Top]}, PlayerId, N);

draw_cards(#{draw_pile := Draw, hands := Hands} = State, PlayerId, N) ->
    {DrawnCards, NewDraw} = lists:split(min(N, length(Draw)), Draw),
    NewHand = maps:get(PlayerId, Hands) ++ DrawnCards,
    State#{draw_pile := NewDraw, hands := Hands#{PlayerId => NewHand}}.

is_over(#{hands := Hands}) ->
    case [{PId, length(H)} || {PId, H} <- maps:to_list(Hands), length(H) =:= 0] of
        [{WinnerId, 0} | _] -> {true, WinnerId};
        [] -> false
    end.

serialize(State) ->
    %% Convert to JSON-safe format
    State#{
        hands => maps:map(fun(_, Hand) -> length(Hand) end, maps:get(hands, State))
        %% Don't send other players' cards!
    }.

serialize_for_player(State, PlayerId) ->
    AllHands = maps:get(hands, State),
    VisibleHands = maps:map(fun(PId, Hand) ->
        case PId =:= PlayerId of
            true -> Hand;             %% own cards are visible
            false -> length(Hand)     %% others: just count
        end
    end, AllHands),
    State#{hands := VisibleHands}.

%% Deck utilities
full_deck() ->
    Colors = [red, blue, green, yellow],
    Numbers = lists:seq(0, 9),
    SpecialCards = [skip, reverse, draw_two],
    
    NumberCards = [{Color, N} || Color <- Colors, N <- Numbers],
    SpecialCardsList = [{Color, S} || Color <- Colors, S <- SpecialCards],
    WildCards = [{wild, wild} || _ <- lists:seq(1, 4)] ++
                [{wild, draw_four} || _ <- lists:seq(1, 4)],
    
    NumberCards ++ SpecialCardsList ++ WildCards.

shuffle(List) ->
    [X || {_, X} <- lists:sort([{rand:uniform(), X} || X <- List])].

deal_hands(Deck, PlayerIds, HandSize) ->
    lists:foldl(fun(PlayerId, {Hands, RemainingDeck}) ->
        {Hand, Rest} = lists:split(HandSize, RemainingDeck),
        {Hands#{PlayerId => Hand}, Rest}
    end, {#{}, Deck}, PlayerIds).

index_of(Elem, List) ->
    index_of(Elem, List, 0).
index_of(Elem, [Elem | _], Idx) -> Idx;
index_of(Elem, [_ | Rest], Idx) -> index_of(Elem, Rest, Idx + 1).
```

---

## สรุป Part 74

| Component | Pattern |
|-----------|---------|
| Game loop | gen_server + timer |
| Player conn | cowboy_websocket handler |
| Game state | Pure functions (easy to test) |
| Matchmaking | gen_server queue |
| Room lifecycle | supervisor tree |
| Anti-cheat | Server-authoritative state |

**หลักการสำคัญ**: Game logic เป็น pure function — ทดสอบได้ง่าย ไม่มี side effect

---

*[← Part 73: Cloud Deployment](part_73_cloud_deployment.md) | [Part 75: IoT with Erlang →](part_75_iot.md)*
