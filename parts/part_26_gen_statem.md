# Part 26: gen_statem — State Machine Behaviour

## สารบัญ
1. [State Machine คืออะไร](#state-machine-คืออะไร)
2. [Callback Modes](#callback-modes)
3. [State and Data](#state-and-data)
4. [State Transitions](#state-transitions)
5. [Internal Events และ Timeouts](#internal-events-และ-timeouts)
6. [ตัวอย่างจริง: Connection State Machine](#ตัวอย่างจริง-connection-state-machine)
7. [ตัวอย่างจริง: Traffic Light](#ตัวอย่างจริง-traffic-light)

---

## State Machine คืออะไร

```erlang
%% gen_statem = Generic State Machine (OTP 19+)
%% แทนที่ gen_fsm เดิม (deprecated)

%% State machine มี:
%% - States: สถานะที่เป็นไปได้ (atom)
%% - Events: input ที่รับได้
%% - Transitions: state + event -> new state + actions
%% - Data: ข้อมูลที่เก็บข้ามสถานะ

%% ตัวอย่าง: Lock
%%   locked --(insert_coin)--> unlocked
%%   unlocked --(push)--> locked
%%   unlocked --(timeout 30s)--> locked

%% ทำไมใช้ gen_statem แทน gen_server?
%% - Complex logic ที่มีหลาย state
%% - Behavior เปลี่ยนตาม state
%% - Event queuing/postponing
%% - Built-in timeout handling per state
```

---

## Callback Modes

```erlang
%% gen_statem มี 2 callback modes:
%% 1. state_functions - แต่ละ state เป็น function ชื่อตาม state
%% 2. handle_event_function - function เดียวจัดการทุก state

%% MODE 1: state_functions
-module(lock_fsm).
-behaviour(gen_statem).

callback_mode() -> state_functions.

%% callback: StateName(EventType, EventContent, Data)
locked(enter, _OldState, Data) ->
    {keep_state, Data};
locked({call, From}, {insert_coin, Coin}, Data) ->
    case valid_coin(Coin) of
        true -> {next_state, unlocked, Data, [{reply, From, ok}]};
        false -> {keep_state, Data, [{reply, From, invalid_coin}]}
    end;
locked(EventType, Event, Data) ->
    handle_common(EventType, Event, Data).

unlocked(enter, _OldState, Data) ->
    {keep_state, Data, [{state_timeout, 30000, lock_timeout}]};
unlocked({call, From}, push, Data) ->
    {next_state, locked, Data, [{reply, From, ok}]};
unlocked(state_timeout, lock_timeout, Data) ->
    {next_state, locked, Data};
unlocked(EventType, Event, Data) ->
    handle_common(EventType, Event, Data).

%% MODE 2: handle_event_function (more flexible)
callback_mode() -> handle_event_function.

handle_event(enter, OldState, NewState, Data) ->
    io:format("~p -> ~p~n", [OldState, NewState]),
    {keep_state, Data};
handle_event({call, From}, {insert_coin, Coin}, locked, Data) ->
    case valid_coin(Coin) of
        true -> {next_state, unlocked, Data, [{reply, From, ok}]};
        false -> {keep_state, Data, [{reply, From, invalid_coin}]}
    end;
handle_event({call, From}, push, unlocked, Data) ->
    {next_state, locked, Data, [{reply, From, ok}]};
handle_event(EventType, Event, State, Data) ->
    io:format("Unhandled ~p in state ~p: ~p~n", [EventType, State, Event]),
    {keep_state, Data}.
```

---

## State and Data

```erlang
%% State = atom (ชื่อ state)
%% Data = term ใดๆ (ข้อมูลที่ persist ข้าม state)

%% init/1 ต้องคืน state และ data
init([Opts]) ->
    Data = #{
        attempts => 0,
        max_attempts => maps:get(max_attempts, Opts, 3),
        user => undefined
    },
    {ok, idle, Data}.

%% เปลี่ยน state พร้อม update data
handle_event({call, From}, {login, User, Pass}, idle, Data) ->
    case auth:verify(User, Pass) of
        ok ->
            NewData = Data#{user => User, attempts => 0},
            {next_state, authenticated, NewData, [{reply, From, ok}]};
        error ->
            #{attempts := N, max_attempts := Max} = Data,
            case N + 1 >= Max of
                true ->
                    {next_state, locked_out, Data#{attempts => N+1},
                     [{reply, From, {error, locked_out}}]};
                false ->
                    {keep_state, Data#{attempts => N+1},
                     [{reply, From, {error, invalid_credentials}}]}
            end
    end;

%% keep_state_and_data = keep current state and data
handle_event(cast, ping, _State, _Data) ->
    keep_state_and_data;

%% keep_state = keep current state, update data
handle_event(cast, {update_config, Config}, _State, Data) ->
    {keep_state, Data#{config => Config}}.
```

---

## State Transitions

```erlang
%% Return values จาก callback:

%% next_state: เปลี่ยน state
{next_state, NewState, NewData}
{next_state, NewState, NewData, Actions}

%% keep_state: คง state ไว้
{keep_state, NewData}
{keep_state, NewData, Actions}
keep_state_and_data       %% shorthand

%% stop
{stop, Reason}
{stop, Reason, NewData}
{stop_and_reply, Reason, Replies}
{stop_and_reply, Reason, Replies, NewData}

%% Actions:
Actions = [Action]

Action = 
    {reply, From, Reply} |      %% reply to call
    {next_event, Type, Event} | %% inject new event
    {state_timeout, Time, EventContent} |  %% timeout for current state
    {timeout, Time, EventContent} |        %% generic timeout
    {{timeout, Name}, Time, EventContent} |%% named timeout
    postpone |                             %% defer event to next state
    hibernate.

%% postpone: event ถูก queue ไว้, จะถูก process ใน state ต่อไป
%% ใช้เมื่อ: event มาก่อน state พร้อมรับ
handle_event(cast, {message, _Msg}, connecting, Data) ->
    {keep_state, Data, [postpone]};  %% will retry when in connected state

%% next_event: inject event into self
handle_event(enter, _Old, authenticated, Data) ->
    {keep_state, Data, [{next_event, cast, load_user_data}]};
handle_event(cast, load_user_data, authenticated, Data) ->
    UserData = db:load_user(maps:get(user, Data)),
    {keep_state, Data#{user_data => UserData}}.
```

---

## Internal Events และ Timeouts

```erlang
%% Timeout types:

%% 1. state_timeout: reset เมื่อ enter state
handle_event(enter, _, waiting_payment, Data) ->
    {keep_state, Data, [{state_timeout, 300000, payment_expired}]};  %% 5 min

handle_event(state_timeout, payment_expired, waiting_payment, Data) ->
    {next_state, cancelled, Data};

%% 2. Generic timeout: reset เมื่อ set
handle_event({call, From}, start_heartbeat, connected, Data) ->
    {keep_state, Data,
     [{reply, From, ok}, {timeout, 5000, heartbeat}]};

handle_event(timeout, heartbeat, connected, Data) ->
    send_ping(Data),
    {keep_state, Data, [{timeout, 5000, heartbeat}]};  %% reset

handle_event(info, pong, connected, Data) ->
    {keep_state, Data, [{timeout, 5000, heartbeat}]};  %% reset on pong

%% 3. Named timeout: multiple independent timeouts
handle_event(enter, _, session_active, Data) ->
    {keep_state, Data, [
        {{timeout, idle_check}, 60000, idle_warning},
        {{timeout, max_session}, 3600000, session_expire}
    ]};

handle_event({timeout, idle_check}, idle_warning, session_active, Data) ->
    notify_user(idle_warning),
    {keep_state, Data, [{{timeout, idle_check}, 120000, force_logout}]};

handle_event({timeout, max_session}, session_expire, session_active, Data) ->
    {next_state, logged_out, Data};

%% Cancel timeout
{keep_state, Data, [{{timeout, idle_check}, cancel}]}
```

---

## ตัวอย่างจริง: Connection State Machine

```erlang
-module(conn_fsm).
-behaviour(gen_statem).

%% States: disconnected -> connecting -> connected -> disconnected
%% Data: #{host, port, socket, retries, max_retries, pending_msgs}

-export([start_link/2, send/2, disconnect/1]).
-export([callback_mode/0, init/1, terminate/3, code_change/4]).
-export([disconnected/3, connecting/3, connected/3]).

start_link(Host, Port) ->
    gen_statem:start_link(?MODULE, [Host, Port], []).

send(Pid, Msg) ->
    gen_statem:call(Pid, {send, Msg}).

disconnect(Pid) ->
    gen_statem:cast(Pid, disconnect).

callback_mode() -> [state_functions, state_enter].
%% state_enter: ให้ receive enter event เมื่อ enter state

init([Host, Port]) ->
    Data = #{
        host => Host,
        port => Port,
        socket => undefined,
        retries => 0,
        max_retries => 5,
        pending_msgs => []
    },
    {ok, disconnected, Data}.

%% State: disconnected
disconnected(enter, _OldState, Data) ->
    io:format("[disconnected] Entered~n"),
    {keep_state, Data#{socket => undefined}};

disconnected({call, From}, connect, Data) ->
    {next_state, connecting, Data, [{reply, From, ok}]};

disconnected({call, From}, {send, _Msg}, Data) ->
    {keep_state, Data, [{reply, From, {error, not_connected}}]};

disconnected(cast, disconnect, Data) ->
    {keep_state, Data};

disconnected(EventType, Event, Data) ->
    io:format("[disconnected] Unhandled ~p: ~p~n", [EventType, Event]),
    {keep_state, Data}.

%% State: connecting
connecting(enter, _OldState, #{host := Host, port := Port} = Data) ->
    io:format("[connecting] Connecting to ~s:~p~n", [Host, Port]),
    Self = self(),
    spawn(fun() ->
        case gen_tcp:connect(Host, Port, [binary, {active, true}], 5000) of
            {ok, Socket} -> gen_statem:cast(Self, {connected, Socket});
            {error, Reason} -> gen_statem:cast(Self, {connect_failed, Reason})
        end
    end),
    {keep_state, Data, [{state_timeout, 10000, connect_timeout}]};

connecting(cast, {connected, Socket}, Data) ->
    io:format("[connecting] Connected!~n"),
    {next_state, connected, Data#{socket => Socket, retries => 0}};

connecting(cast, {connect_failed, Reason}, #{retries := R, max_retries := Max} = Data) ->
    io:format("[connecting] Failed: ~p (attempt ~p/~p)~n", [Reason, R+1, Max]),
    case R + 1 >= Max of
        true ->
            {next_state, disconnected, Data#{retries => 0}};
        false ->
            Delay = min(1000 * (R + 1), 30000),  %% exponential backoff
            {keep_state, Data#{retries => R+1},
             [{state_timeout, Delay, retry_connect}]}
    end;

connecting(state_timeout, connect_timeout, Data) ->
    {next_state, disconnected, Data};

connecting(state_timeout, retry_connect, Data) ->
    %% Re-enter connecting to retry
    {repeat_state, Data};

connecting({call, From}, {send, Msg}, #{pending_msgs := Pending} = Data) ->
    %% Queue message, will be sent when connected
    {keep_state, Data#{pending_msgs => Pending ++ [Msg]},
     [{reply, From, queued}]};

connecting(_, _, Data) ->
    {keep_state, Data}.

%% State: connected
connected(enter, _OldState, #{pending_msgs := Pending, socket := Socket} = Data) ->
    io:format("[connected] Sending ~p queued messages~n", [length(Pending)]),
    lists:foreach(fun(Msg) -> do_send(Socket, Msg) end, Pending),
    {keep_state, Data#{pending_msgs => []},
     [{timeout, 30000, keepalive}]};  %% start keepalive

connected({call, From}, {send, Msg}, #{socket := Socket} = Data) ->
    case do_send(Socket, Msg) of
        ok -> {keep_state, Data, [{reply, From, ok}, {timeout, 30000, keepalive}]};
        {error, Reason} ->
            {next_state, disconnected, Data,
             [{reply, From, {error, Reason}}]}
    end;

connected(cast, disconnect, #{socket := Socket} = Data) ->
    gen_tcp:close(Socket),
    {next_state, disconnected, Data};

connected(timeout, keepalive, #{socket := Socket} = Data) ->
    case do_send(Socket, <<"ping">>) of
        ok -> {keep_state, Data, [{timeout, 30000, keepalive}]};
        {error, _} -> {next_state, connecting, Data}
    end;

connected(info, {tcp, _Socket, Data_}, State) ->
    io:format("[connected] Received: ~p~n", [Data_]),
    {keep_state, State, [{timeout, 30000, keepalive}]};

connected(info, {tcp_closed, _Socket}, Data) ->
    io:format("[connected] Connection closed~n"),
    {next_state, connecting, Data};

connected(info, {tcp_error, _Socket, Reason}, Data) ->
    io:format("[connected] TCP error: ~p~n", [Reason]),
    {next_state, connecting, Data};

connected(_, _, Data) ->
    {keep_state, Data}.

terminate(_Reason, _State, #{socket := Socket}) when Socket =/= undefined ->
    gen_tcp:close(Socket);
terminate(_, _, _) ->
    ok.

code_change(_Vsn, State, Data, _Extra) ->
    {ok, State, Data}.

do_send(Socket, Msg) ->
    gen_tcp:send(Socket, term_to_binary(Msg)).
```

---

## ตัวอย่างจริง: Traffic Light

```erlang
-module(traffic_light).
-behaviour(gen_statem).

-export([start_link/0, current_state/1]).
-export([callback_mode/0, init/1, handle_event/4]).

start_link() ->
    gen_statem:start_link(?MODULE, [], []).

current_state(Pid) ->
    gen_statem:call(Pid, current_state).

callback_mode() -> [handle_event_function, state_enter].

%% Times (ms)
-define(RED_TIME, 10000).
-define(GREEN_TIME, 8000).
-define(YELLOW_TIME, 3000).

init([]) ->
    {ok, red, #{cycle => 0}}.

%% Enter state -> set timer
handle_event(enter, _, red, Data) ->
    io:format("🔴 RED (~p ms)~n", [?RED_TIME]),
    {keep_state, Data, [{state_timeout, ?RED_TIME, next}]};

handle_event(enter, _, green, Data) ->
    io:format("🟢 GREEN (~p ms)~n", [?GREEN_TIME]),
    {keep_state, Data, [{state_timeout, ?GREEN_TIME, next}]};

handle_event(enter, _, yellow, Data) ->
    io:format("🟡 YELLOW (~p ms)~n", [?YELLOW_TIME]),
    {keep_state, Data, [{state_timeout, ?YELLOW_TIME, next}]};

%% Transitions
handle_event(state_timeout, next, red, Data) ->
    {next_state, green, Data};

handle_event(state_timeout, next, green, Data) ->
    {next_state, yellow, Data};

handle_event(state_timeout, next, yellow, #{cycle := C} = Data) ->
    io:format("Cycle ~p complete~n", [C + 1]),
    {next_state, red, Data#{cycle => C + 1}};

%% API calls
handle_event({call, From}, current_state, State, Data) ->
    {keep_state, Data, [{reply, From, {State, Data}}]};

handle_event(cast, emergency, _, Data) ->
    io:format("🚨 EMERGENCY - All red!~n"),
    {next_state, red, Data#{emergency => true}};

handle_event(_, _, _, Data) ->
    {keep_state, Data}.

%% ใช้งาน
{ok, Light} = traffic_light:start_link().
timer:sleep(15000).
traffic_light:current_state(Light).
%% {green, #{cycle => 1}}
```

---

## สรุป Part 26

| Return | Effect |
|--------|--------|
| `{next_state, S, D}` | Transition to state S with data D |
| `{keep_state, D}` | Stay in current state, update data |
| `keep_state_and_data` | No change |
| `{stop, Reason}` | Terminate |

| Action | Effect |
|--------|--------|
| `{reply, From, R}` | Reply to call |
| `{state_timeout, T, E}` | Timeout that resets on state enter |
| `{timeout, T, E}` | Generic timeout |
| `postpone` | Defer event to next state |
| `{next_event, Type, E}` | Inject new event |

---

*[← Part 25: Supervisor](part_25_supervisor.md) | [Part 27: gen_event →](part_27_gen_event.md)*
