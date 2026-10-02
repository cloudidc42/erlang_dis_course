# Part 83: Microservices Patterns

## สารบัญ
1. [Service Discovery](#service-discovery)
2. [Circuit Breaker Pattern](#circuit-breaker)
3. [Saga Pattern for Distributed Transactions](#saga)
4. [API Composition](#api-composition)
5. [Service Mesh Integration](#service-mesh)
6. [ตัวอย่างจริง: E-Commerce Microservices](#ecommerce)

---

## Service Discovery

```erlang
%% service_registry.erl — in-cluster service discovery

-module(service_registry).
-behaviour(gen_server).
-export([start_link/0, register/3, deregister/2, discover/1, health_check/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Uses pg (process groups) for Erlang cluster, Consul for cross-cluster

-record(service_entry, {
    name,
    host,
    port,
    metadata = #{},
    health_status = healthy,
    last_health_check
}).

init([]) ->
    %% Subscribe to node up/down events
    net_kernel:monitor_nodes(true),
    
    %% Start health check timer
    erlang:send_after(5000, self(), run_health_checks),
    
    {ok, #{services => #{}}}.

%% Register this service
register(ServiceName, Port, Metadata) ->
    {ok, IP} = get_local_ip(),
    Entry = #service_entry{
        name = ServiceName,
        host = inet:ntoa(IP),
        port = Port,
        metadata = Metadata,
        last_health_check = erlang:system_time(second)
    },
    
    %% Register on this node
    gen_server:call(?MODULE, {register, ServiceName, Entry}),
    
    %% Broadcast to cluster
    gen_server:abcast(nodes(), ?MODULE, {register_remote, ServiceName, Entry}),
    
    ok.

deregister(ServiceName, Host) ->
    gen_server:call(?MODULE, {deregister, ServiceName, Host}),
    gen_server:abcast(nodes(), ?MODULE, {deregister_remote, ServiceName, Host}).

%% Discover available instances
discover(ServiceName) ->
    gen_server:call(?MODULE, {discover, ServiceName}).

handle_call({discover, ServiceName}, _From, #{services := Services} = State) ->
    Instances = maps:get(ServiceName, Services, []),
    HealthyInstances = [I || I <- Instances, I#service_entry.health_status =:= healthy],
    {reply, HealthyInstances, State};

handle_call({register, ServiceName, Entry}, _From, #{services := Services} = State) ->
    Existing = maps:get(ServiceName, Services, []),
    %% Replace if same host:port, otherwise add
    Updated = [Entry | lists:filter(fun(E) ->
        not (E#service_entry.host =:= Entry#service_entry.host andalso
             E#service_entry.port =:= Entry#service_entry.port)
    end, Existing)],
    {reply, ok, State#{services := Services#{ServiceName => Updated}}}.

handle_info(run_health_checks, State) ->
    NewState = run_health_checks(State),
    erlang:send_after(5000, self(), run_health_checks),
    {noreply, NewState}.

run_health_checks(#{services := Services} = State) ->
    NewServices = maps:map(fun(_ServiceName, Instances) ->
        [check_instance(I) || I <- Instances]
    end, Services),
    State#{services := NewServices}.

check_instance(#service_entry{host = Host, port = Port} = Entry) ->
    case ping_service(Host, Port) of
        ok -> Entry#service_entry{health_status = healthy,
                                   last_health_check = erlang:system_time(second)};
        _ -> Entry#service_entry{health_status = unhealthy,
                                  last_health_check = erlang:system_time(second)}
    end.

ping_service(Host, Port) ->
    case gen_tcp:connect(Host, Port, [binary, {packet, 0}], 2000) of
        {ok, Socket} ->
            gen_tcp:close(Socket),
            ok;
        {error, _} ->
            unhealthy
    end.

get_local_ip() ->
    {ok, Ifs} = inet:getifaddrs(),
    case [Addr || {_Name, Opts} <- Ifs,
                  {addr, Addr} <- Opts,
                  tuple_size(Addr) =:= 4,    %% IPv4
                  Addr =/= {127, 0, 0, 1}] of  %% not loopback
        [IP | _] -> {ok, IP};
        [] -> {ok, {127, 0, 0, 1}}
    end.

%% Client: get endpoint for a service
get_endpoint(ServiceName) ->
    case discover(ServiceName) of
        [] -> {error, no_healthy_instances};
        Instances ->
            %% Round-robin selection
            Idx = erlang:phash2(erlang:unique_integer(), length(Instances)) + 1,
            Instance = lists:nth(Idx, Instances),
            {ok, Instance#service_entry.host, Instance#service_entry.port}
    end.
```

---

## Circuit Breaker Pattern

```erlang
%% circuit_breaker.erl — per-service circuit breaker

-module(circuit_breaker).
-behaviour(gen_server).
-export([call/3, start_link/2]).
-export([init/1, handle_call/3, handle_info/2]).

%% States: closed (normal), open (failing), half-open (testing)

-record(cb_state, {
    service,
    state = closed,       %% closed | open | half_open
    failures = 0,
    successes = 0,
    last_failure_at,
    options
}).

-define(FAILURE_THRESHOLD, 5).
-define(SUCCESS_THRESHOLD, 2).
-define(RECOVERY_TIMEOUT, 30000).

start_link(ServiceName, Options) ->
    gen_server:start_link({via, gproc, {n, l, {cb, ServiceName}}},
                          ?MODULE, {ServiceName, Options}, []).

call(ServiceName, Fun, Timeout) ->
    gen_server:call({via, gproc, {n, l, {cb, ServiceName}}},
                    {call, Fun, Timeout}, Timeout + 1000).

init({ServiceName, Options}) ->
    {ok, #cb_state{service = ServiceName, options = Options}}.

handle_call({call, Fun, Timeout}, _From, #cb_state{state = open} = State) ->
    %% Circuit is open: check if recovery timeout elapsed
    Now = erlang:monotonic_time(millisecond),
    RecoveryMs = maps:get(recovery_timeout, State#cb_state.options, ?RECOVERY_TIMEOUT),
    
    case Now - State#cb_state.last_failure_at > RecoveryMs of
        true ->
            %% Try half-open: allow one request
            execute_call(Fun, Timeout, State#cb_state{state = half_open});
        false ->
            %% Still open: fail fast
            metrics:counter(circuit_breaker_rejected, 1, [{service, State#cb_state.service}]),
            {reply, {error, circuit_open}, State}
    end;

handle_call({call, Fun, Timeout}, _From, State) ->
    execute_call(Fun, Timeout, State).

execute_call(Fun, Timeout, State) ->
    case timed_execute(Fun, Timeout) of
        {ok, Result} ->
            NewState = handle_success(State),
            {reply, {ok, Result}, NewState};
        {error, Reason} ->
            NewState = handle_failure(State, Reason),
            {reply, {error, Reason}, NewState}
    end.

timed_execute(Fun, Timeout) ->
    Parent = self(),
    Ref = make_ref(),
    
    spawn_link(fun() ->
        Result = try {ok, Fun()} catch Class:Reason -> {error, {Class, Reason}} end,
        Parent ! {Ref, Result}
    end),
    
    receive
        {Ref, Result} -> Result
    after Timeout ->
        {error, timeout}
    end.

handle_success(#cb_state{state = half_open, successes = S} = State) ->
    Threshold = maps:get(success_threshold, State#cb_state.options, ?SUCCESS_THRESHOLD),
    case S + 1 >= Threshold of
        true ->
            logger:info("Circuit breaker closed", #{service => State#cb_state.service}),
            State#cb_state{state = closed, successes = 0, failures = 0};
        false ->
            State#cb_state{successes = S + 1}
    end;
handle_success(State) ->
    State#cb_state{failures = 0}.

handle_failure(State, Reason) ->
    Now = erlang:monotonic_time(millisecond),
    Threshold = maps:get(failure_threshold, State#cb_state.options, ?FAILURE_THRESHOLD),
    NewFailures = State#cb_state.failures + 1,
    
    case NewFailures >= Threshold of
        true ->
            logger:warning("Circuit breaker opened", #{
                service => State#cb_state.service,
                failures => NewFailures,
                reason => Reason
            }),
            State#cb_state{state = open, failures = NewFailures, last_failure_at = Now};
        false ->
            State#cb_state{failures = NewFailures}
    end.

%% Usage
call_with_cb(ServiceName, Request) ->
    circuit_breaker:call(ServiceName, fun() ->
        case service_registry:get_endpoint(ServiceName) of
            {ok, Host, Port} ->
                http_client:post(Host, Port, "/api", Request);
            Error ->
                Error
        end
    end, 5000).
```

---

## Saga Pattern

```erlang
%% order_saga.erl — distributed transaction using saga pattern

-module(order_saga).
-behaviour(gen_statem).
-export([start/1, get_status/1]).
-export([callback_mode/0, init/1]).
-export([starting/3, reserving_inventory/3, charging_payment/3,
         confirming_order/3, compensating/3, completed/3, failed/3]).

callback_mode() -> state_functions.

start(OrderData) ->
    {ok, Pid} = gen_statem:start_link(?MODULE, OrderData, []),
    Pid.

init(OrderData) ->
    SagaId = generate_saga_id(),
    {ok, starting, #{
        saga_id => SagaId,
        order_data => OrderData,
        compensation_log => [],
        error => undefined
    }, [{next_event, internal, begin_saga}]}.

%% Step 1: Start saga
starting(internal, begin_saga, Data) ->
    logger:info("Starting order saga", #{saga_id => maps:get(saga_id, Data)}),
    {next_state, reserving_inventory, Data, [{next_event, internal, reserve}]}.

%% Step 2: Reserve inventory
reserving_inventory(internal, reserve, #{order_data := Order} = Data) ->
    case inventory_service:reserve(Order) of
        {ok, ReservationId} ->
            %% Log compensation action
            CompLog = [{cancel_inventory_reservation, ReservationId} | maps:get(compensation_log, Data)],
            NewData = Data#{reservation_id => ReservationId, compensation_log => CompLog},
            {next_state, charging_payment, NewData, [{next_event, internal, charge}]};
        {error, Reason} ->
            {next_state, failed, Data#{error => {inventory_failed, Reason}},
             [{next_event, internal, compensate}]}
    end;

%% Step 3: Charge payment
charging_payment(internal, charge, #{order_data := Order} = Data) ->
    case payment_service:charge(Order) of
        {ok, PaymentId} ->
            CompLog = [{refund_payment, PaymentId} | maps:get(compensation_log, Data)],
            NewData = Data#{payment_id => PaymentId, compensation_log => CompLog},
            {next_state, confirming_order, NewData, [{next_event, internal, confirm}]};
        {error, Reason} ->
            {next_state, compensating, Data#{error => {payment_failed, Reason}},
             [{next_event, internal, run_compensation}]}
    end;

%% Step 4: Confirm order
confirming_order(internal, confirm, Data) ->
    OrderId = order_service:create(Data),
    notification_service:send(order_confirmed, Data),
    {next_state, completed, Data#{order_id => OrderId}}.

%% Compensation: undo previous steps
compensating(internal, run_compensation, #{compensation_log := Log} = Data) ->
    [execute_compensation(Action) || Action <- Log],
    {next_state, failed, Data}.

execute_compensation({cancel_inventory_reservation, ReservationId}) ->
    inventory_service:cancel_reservation(ReservationId);
execute_compensation({refund_payment, PaymentId}) ->
    payment_service:refund(PaymentId);
execute_compensation({cancel_order, OrderId}) ->
    order_service:cancel(OrderId).

%% Terminal states
completed(_, _, Data) -> {keep_state, Data}.
failed(_, _, Data) -> {keep_state, Data}.

get_status(SagaPid) ->
    gen_statem:call(SagaPid, get_status).

generate_saga_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).
```

---

## ตัวอย่างจริง: E-Commerce Microservices

```erlang
%% Service interactions for place_order flow

-module(order_orchestrator).
-export([place_order/2]).

place_order(UserId, CartId) ->
    %% Start distributed saga
    SagaData = #{
        user_id => UserId,
        cart_id => CartId,
        created_at => erlang:system_time(second)
    },
    
    SagaPid = order_saga:start(SagaData),
    
    %% Wait for saga to complete (or timeout)
    case wait_for_saga(SagaPid, 30000) of
        {completed, #{order_id := OrderId}} ->
            {ok, OrderId};
        {failed, #{error := Error}} ->
            {error, Error};
        timeout ->
            {error, saga_timeout}
    end.

wait_for_saga(SagaPid, TimeoutMs) ->
    MonRef = monitor(process, SagaPid),
    
    receive
        {saga_done, SagaPid, Result} ->
            demonitor(MonRef, [flush]),
            Result;
        {'DOWN', MonRef, process, SagaPid, Reason} ->
            {failed, #{error => {saga_crashed, Reason}}}
    after TimeoutMs ->
        demonitor(MonRef, [flush]),
        timeout
    end.

%% Service client with circuit breaker
-module(inventory_service_client).
-export([reserve/1, cancel_reservation/1]).

reserve(OrderData) ->
    circuit_breaker:call(inventory_service, fun() ->
        {Host, Port} = get_service_endpoint(inventory_service),
        
        Request = #{
            items => maps:get(items, OrderData),
            user_id => maps:get(user_id, OrderData)
        },
        
        case http_client:post(Host, Port, "/api/v1/reservations", Request) of
            {ok, 201, Body} ->
                #{<<"reservation_id">> := ResId} = jiffy:decode(Body, [return_maps]),
                {ok, ResId};
            {ok, 409, _} ->
                {error, out_of_stock};
            {ok, Status, _} ->
                {error, {unexpected_status, Status}}
        end
    end, 5000).

cancel_reservation(ReservationId) ->
    circuit_breaker:call(inventory_service, fun() ->
        {Host, Port} = get_service_endpoint(inventory_service),
        
        case http_client:delete(Host, Port, "/api/v1/reservations/" ++ binary_to_list(ReservationId)) of
            {ok, 204, _} -> ok;
            _ -> {error, cancellation_failed}
        end
    end, 3000).

get_service_endpoint(ServiceName) ->
    case service_registry:get_endpoint(ServiceName) of
        {ok, Host, Port} -> {Host, Port};
        {error, Reason} -> error({service_unavailable, ServiceName, Reason})
    end.
```

---

## สรุป Part 83

| Pattern | Use Case |
|---------|---------|
| Service Discovery | Find service instances automatically |
| Circuit Breaker | Prevent cascade failures |
| Saga | Distributed transactions without 2PC |
| API Composition | Aggregate multiple service calls |
| Bulkhead | Isolate failure domains |

**Erlang advantages**: Process model = natural bulkhead, message passing = loose coupling, supervisors = resilient service management

---

*[← Part 82: API Design](part_82_api_design.md) | [Part 84: Testing Strategies →](part_84_testing_strategies.md)*
