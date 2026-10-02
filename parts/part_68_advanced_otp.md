# Part 68: Advanced OTP Patterns

## สารบัญ
1. [gen_statem Advanced Usage](#gen-statem)
2. [proc_lib และ sys Module](#proc-lib)
3. [Special Processes](#special-processes)
4. [Application Lifecycle](#app-lifecycle)
5. [Code Upgrades กับ appup](#code-upgrades)
6. [ตัวอย่างจริง: Connection State Machine](#connection-state-machine)

---

## gen_statem Advanced Usage

```erlang
%% gen_statem: finite state machine with event handling
%% More powerful than gen_fsm (deprecated)

%% Callback modes: state_functions or handle_event_function
%% state_functions: function per state (traditional)
%% handle_event_function: single function (flexible)

-module(order_fsm).
-behaviour(gen_statem).
-export([start_link/1, submit/1, pay/2, ship/2, deliver/1, cancel/2]).
-export([init/1, callback_mode/0]).
%% State function callbacks
-export([pending/3, payment_pending/3, processing/3, shipped/3, delivered/3, cancelled/3]).

callback_mode() -> state_functions.

start_link(OrderId) ->
    gen_statem:start_link(?MODULE, OrderId, []).

%% Public API
submit(Pid) -> gen_statem:call(Pid, submit).
pay(Pid, PaymentId) -> gen_statem:call(Pid, {pay, PaymentId}).
ship(Pid, TrackingNo) -> gen_statem:call(Pid, {ship, TrackingNo}).
deliver(Pid) -> gen_statem:call(Pid, deliver).
cancel(Pid, Reason) -> gen_statem:call(Pid, {cancel, Reason}).

init(OrderId) ->
    {ok, pending, #{order_id => OrderId, events => []}}.

%% State: pending
pending({call, From}, submit, Data) ->
    notify(data_order_id(Data), <<"Order submitted">>),
    {next_state, payment_pending, add_event(submitted, Data),
     [{reply, From, ok}]};

pending({call, From}, {cancel, Reason}, Data) ->
    {next_state, cancelled, add_event({cancelled, Reason}, Data),
     [{reply, From, ok}]};

pending(_, _, Data) ->
    {keep_state, Data, [postpone]}.  %% postpone unknown events

%% State: payment_pending
payment_pending({call, From}, {pay, PaymentId}, Data) ->
    {next_state, processing, Data#{payment_id => PaymentId},
     [{reply, From, ok},
      {state_timeout, 3600000, payment_confirmed}]};  %% 1h timeout

payment_pending(state_timeout, payment_confirmed, Data) ->
    %% Auto-cancel if payment not confirmed in 1 hour
    {next_state, cancelled, add_event({cancelled, payment_timeout}, Data)};

payment_pending({call, From}, {cancel, Reason}, Data) ->
    {next_state, cancelled, add_event({cancelled, Reason}, Data),
     [{reply, From, ok}]};

payment_pending(_, _, Data) -> {keep_state, Data, [postpone]}.

%% State: processing
processing({call, From}, {ship, TrackingNo}, Data) ->
    {next_state, shipped, Data#{tracking => TrackingNo},
     [{reply, From, ok}]};

processing(_, _, Data) -> {keep_state, Data, [postpone]}.

%% State: shipped
shipped({call, From}, deliver, Data) ->
    {next_state, delivered, Data,
     [{reply, From, ok}]};

shipped(_, _, Data) -> {keep_state, Data, [postpone]}.

%% Terminal states
delivered(_, _, Data) -> {keep_state, Data, []}.
cancelled(_, _, Data) -> {keep_state, Data, []}.

add_event(Event, #{events := Events} = Data) ->
    Data#{events := [{Event, erlang:system_time()} | Events]}.

data_order_id(#{order_id := Id}) -> Id.
notify(OrderId, Msg) -> event_bus:publish(OrderId, Msg).
```

---

## proc_lib และ sys Module

```erlang
%% proc_lib: start processes that integrate with OTP error handling
-module(custom_worker).
-export([start_link/0, start/0]).

start_link() ->
    proc_lib:start_link(?MODULE, start, []).

start() ->
    %% Complete initialization before returning from start_link
    proc_lib:init_ack({ok, self()}),
    
    %% Register with sys for debugging
    sys:handle_debug(debug_opts(), fun format_msg/3, ?MODULE, {started}),
    
    loop(initial_state()).

loop(State) ->
    receive
        Msg ->
            NewState = handle(Msg, State),
            loop(NewState)
    end.

debug_opts() ->
    [].  %% or [{debug, [trace, statistics]}]

format_msg(Device, Event, Extra) ->
    io:format(Device, "~p ~p~n", [Event, Extra]).

%% sys module: debug running processes
debug_process(Pid) ->
    %% Get process status
    sys:get_status(Pid),
    
    %% Trace all messages
    sys:trace(Pid, true),
    
    %% Get state (if process implements sys callbacks)
    sys:get_state(Pid),
    
    %% Statistics
    sys:statistics(Pid, true),
    timer:sleep(5000),
    sys:statistics(Pid, get),
    
    %% Suspend/resume
    sys:suspend(Pid),
    sys:resume(Pid).

%% Implementing sys callbacks in custom process
system_continue(Parent, Debug, State) ->
    loop(State).

system_terminate(Reason, _Parent, _Debug, State) ->
    cleanup(State),
    exit(Reason).

system_code_change(State, _Module, _OldVsn, _Extra) ->
    {ok, State}.

system_get_state(State) ->
    {ok, State}.

system_replace_state(StateFun, State) ->
    NewState = StateFun(State),
    {ok, NewState, NewState}.
```

---

## Application Lifecycle

```erlang
%% Full application with proper lifecycle management

-module(my_app).
-behaviour(application).
-export([start/2, stop/1, prep_stop/1]).

start(normal, _Args) ->
    logger:info("Starting my_app"),
    
    %% Validate configuration
    validate_config(),
    
    %% Start supervision tree
    case my_sup:start_link() of
        {ok, Pid} ->
            %% Post-start initialization
            schedule_background_jobs(),
            register_with_service_discovery(),
            {ok, Pid};
        Error ->
            Error
    end.

prep_stop(State) ->
    logger:info("Preparing to stop my_app"),
    %% Deregister from service discovery (stop new requests)
    service_discovery:deregister(),
    %% Wait for in-flight requests to complete
    drain_connections(5000),
    State.

stop(_State) ->
    logger:info("Stopped my_app"),
    ok.

validate_config() ->
    Required = [db_host, db_port, kafka_hosts],
    Missing = [K || K <- Required, application:get_env(my_app, K) =:= undefined],
    case Missing of
        [] -> ok;
        Keys -> error({missing_config, Keys})
    end.

schedule_background_jobs() ->
    %% Hourly cleanup
    schedule_job(fun cleanup_expired_sessions/0, 3600000),
    %% Daily reports
    schedule_job(fun generate_daily_report/0, 86400000).

schedule_job(Fun, IntervalMs) ->
    spawn(fun() ->
        timer:sleep(IntervalMs),
        Fun(),
        schedule_job(Fun, IntervalMs)
    end).

drain_connections(TimeoutMs) ->
    %% Signal to stop accepting new requests
    connection_pool:stop_accepting(),
    %% Wait for active requests
    Deadline = erlang:monotonic_time(millisecond) + TimeoutMs,
    wait_for_drain(Deadline).

wait_for_drain(Deadline) ->
    case connection_pool:active_count() of
        0 -> ok;
        N ->
            Now = erlang:monotonic_time(millisecond),
            if Now >= Deadline ->
                logger:warning("Drain timeout, ~p connections still active", [N]);
            true ->
                timer:sleep(100),
                wait_for_drain(Deadline)
            end
    end.

register_with_service_discovery() ->
    {ok, IP} = inet:getaddr("localhost", inet),
    Port = application:get_env(my_app, http_port, 8080),
    service_discovery:register(#{
        id => node(),
        name => <<"my_app">>,
        address => inet:ntoa(IP),
        port => Port,
        health_check_url => "http://" ++ inet:ntoa(IP) ++ ":" ++ integer_to_list(Port) ++ "/health"
    }).
```

---

## ตัวอย่างจริง: Connection State Machine

```erlang
%% tcp_connection.erl: state machine for TCP connection lifecycle

-module(tcp_connection).
-behaviour(gen_statem).
-export([start_link/1, send/2, close/1]).
-export([callback_mode/0, init/1]).
-export([connecting/3, handshaking/3, connected/3, closing/3]).

callback_mode() -> state_functions.

start_link(Opts) ->
    gen_statem:start_link(?MODULE, Opts, []).

send(Pid, Data) -> gen_statem:call(Pid, {send, Data}).
close(Pid) -> gen_statem:call(Pid, close).

init(#{host := Host, port := Port} = Opts) ->
    %% Start connecting asynchronously
    Self = self(),
    spawn(fun() ->
        Result = gen_tcp:connect(Host, Port, [binary, {active, once}]),
        Self ! {connect_result, Result}
    end),
    {ok, connecting, #{opts => Opts, buffer => <<>>, callers => []}}.

connecting(info, {connect_result, {ok, Socket}}, Data) ->
    gen_tcp:controlling_process(Socket, self()),
    %% Start TLS handshake
    ssl:connect(Socket, ssl_opts(Data), 5000),
    {next_state, handshaking, Data#{socket => Socket}};

connecting(info, {connect_result, {error, Reason}}, Data) ->
    reply_all_callers({error, Reason}, Data),
    {stop, {connect_failed, Reason}, Data};

connecting({call, From}, _Request, Data) ->
    %% Queue calls until connected
    {keep_state, Data#{callers := [From | maps:get(callers, Data, [])]}, [postpone]}.

handshaking(info, {ssl, Socket, <<"HANDSHAKE_OK\n">>}, #{socket := Socket} = Data) ->
    %% Reply to any queued callers
    {next_state, connected, Data, [{next_event, internal, flush_pending}]};

handshaking(internal, flush_pending, #{callers := Callers} = Data) ->
    [gen_statem:reply(From, ok) || From <- Callers],
    {keep_state, Data#{callers := []}};

handshaking(info, {ssl_error, _, Reason}, Data) ->
    {stop, {handshake_failed, Reason}, Data}.

connected({call, From}, {send, Payload}, #{socket := Socket} = Data) ->
    case ssl:send(Socket, frame(Payload)) of
        ok -> {keep_state, Data, [{reply, From, ok}]};
        {error, Reason} ->
            {next_state, closing, Data, [{reply, From, {error, Reason}}]}
    end;

connected(info, {ssl, Socket, Data}, #{socket := Socket, buffer := Buf} = State) ->
    ssl:setopts(Socket, [{active, once}]),
    Combined = <<Buf/binary, Data/binary>>,
    {Messages, Rest} = parse_messages(Combined),
    [handle_message(M, State) || M <- Messages],
    {keep_state, State#{buffer := Rest}};

connected(info, {ssl_closed, _}, Data) ->
    {next_state, closing, Data};

connected({call, From}, close, Data) ->
    {next_state, closing, Data, [{reply, From, ok}]}.

closing(_, _, Data) ->
    ssl:close(maps:get(socket, Data)),
    {stop, normal, Data}.

ssl_opts(_Data) -> [{verify, verify_peer}, {cacertfile, "/etc/ssl/certs/ca-certificates.crt"}].
frame(Data) -> <<(byte_size(Data)):32/big, Data/binary>>.

parse_messages(Buffer) ->
    parse_messages(Buffer, []).
parse_messages(<<Len:32/big, Rest/binary>>, Acc) when byte_size(Rest) >= Len ->
    <<Msg:Len/binary, Remaining/binary>> = Rest,
    parse_messages(Remaining, [Msg | Acc]);
parse_messages(Rest, Acc) ->
    {lists:reverse(Acc), Rest}.

handle_message(Msg, _State) ->
    Event = jsx:decode(Msg, [return_maps]),
    event_bus:publish(maps:get(<<"type">>, Event), Event).

reply_all_callers(Reply, #{callers := Callers}) ->
    [gen_statem:reply(From, Reply) || From <- Callers].
```

---

## สรุป Part 68

| Pattern | Use Case |
|---------|----------|
| gen_statem | Complex lifecycle with states |
| state_functions | State-specific handlers |
| handle_event | Shared logic across states |
| proc_lib | Custom OTP-compatible processes |
| sys module | Debug, trace, inspect any process |
| Application lifecycle | Proper startup/shutdown |
| prep_stop/1 | Graceful shutdown hook |

---

*[← Part 67: Kafka](part_67_kafka.md) | [Part 69: Performance Profiling →](part_69_profiling.md)*
