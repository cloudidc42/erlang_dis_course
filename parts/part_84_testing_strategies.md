# Part 84: Advanced Testing Strategies

## สารบัญ
1. [Test Pyramid for Erlang](#test-pyramid)
2. [Integration Testing](#integration)
3. [Contract Testing](#contract)
4. [Chaos Engineering](#chaos)
5. [Fuzz Testing](#fuzzing)
6. [ตัวอย่างจริง: Full Test Suite](#full-suite)

---

## Test Pyramid for Erlang

```
     /\
    /  \
   / E2E \       — 5% of tests, slow, expensive
  /--------\
 / Integration\ — 20% of tests, moderate speed
/------------\
/ Unit Tests   \ — 75% of tests, fast, isolated
/----------------\

Erlang testing tools:
- Unit: EUnit (built-in)
- Properties: PropEr
- Integration: Common Test
- E2E: Playwright/Selenium + running node
- Mocking: Meck
- Load: wrk, tsung, custom load_tester
```

```erlang
%% Run tests at each level
test_commands() ->
    %% Unit tests
    "rebar3 eunit",
    
    %% Property-based tests
    "rebar3 proper",
    
    %% Integration tests (Common Test)
    "rebar3 ct",
    
    %% Coverage
    "rebar3 cover --min_coverage 80",
    
    %% Static analysis
    "rebar3 dialyzer",
    
    %% All together
    "rebar3 do eunit, proper, ct, cover, dialyzer".
```

---

## Integration Testing with Common Test

```erlang
%% test/integration/order_SUITE.erl

-module(order_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([suite/0, all/0, groups/0]).
-export([init_per_suite/1, end_per_suite/1, init_per_testcase/2, end_per_testcase/2]).
-export([create_order_success/1, create_order_insufficient_funds/1,
         order_saga_compensation/1, concurrent_orders/1]).

suite() ->
    [{timetrap, {seconds, 60}}].

all() ->
    [{group, order_lifecycle}, {group, failure_scenarios}].

groups() ->
    [
        {order_lifecycle, [sequence], [create_order_success]},
        {failure_scenarios, [parallel], [create_order_insufficient_funds, order_saga_compensation]},
        {load, [], [concurrent_orders]}
    ].

%% Setup: start test application
init_per_suite(Config) ->
    %% Start test database
    {ok, _} = start_test_postgres(),
    
    %% Run migrations
    migrate:up(),
    
    %% Start the application
    application:ensure_all_started(my_app),
    
    Config.

end_per_suite(_Config) ->
    application:stop(my_app),
    stop_test_postgres().

init_per_testcase(_TestCase, Config) ->
    %% Clean database between tests
    db:execute("TRUNCATE orders, inventory, payments CASCADE", []),
    
    %% Seed test data
    seed_test_data(),
    
    Config.

end_per_testcase(_TestCase, _Config) ->
    ok.

%% Test cases
create_order_success(Config) ->
    UserId = <<"test_user_1">>,
    Items = [#{product_id => <<"prod_1">>, quantity => 2}],
    
    {ok, OrderId} = order_service:create(UserId, Items),
    
    %% Verify order was created
    {ok, Order} = order_service:get(OrderId),
    
    ct:pal("Created order: ~p", [Order]),
    
    OrderId = maps:get(id, Order),
    UserId = maps:get(user_id, Order),
    pending = maps:get(status, Order),
    
    %% Verify inventory was reserved
    {ok, Reservation} = inventory_service:get_reservation_for_order(OrderId),
    2 = maps:get(quantity, Reservation),
    
    ok.

create_order_insufficient_funds(Config) ->
    %% User with empty wallet
    UserId = <<"broke_user">>,
    Items = [#{product_id => <<"expensive_prod">>, quantity => 1}],
    
    {error, insufficient_funds} = order_service:create(UserId, Items),
    
    %% Verify no order was created
    [] = order_service:list_by_user(UserId),
    
    %% Verify inventory was NOT reserved (compensation worked)
    [] = inventory_service:list_reservations_by_user(UserId),
    
    ok.

order_saga_compensation(Config) ->
    %% Simulate payment service failure
    meck:expect(payment_service, charge, fun(_, _) -> {error, payment_failed} end),
    
    UserId = <<"test_user_2">>,
    Items = [#{product_id => <<"prod_1">>, quantity => 1}],
    
    {error, {payment_failed, _}} = order_service:create(UserId, Items),
    
    %% Verify compensation: inventory reservation was cancelled
    [] = inventory_service:list_reservations_by_user(UserId),
    
    meck:unload(payment_service),
    ok.

concurrent_orders(Config) ->
    UserId = <<"concurrent_user">>,
    NumOrders = 100,
    
    %% Fire 100 concurrent order requests
    Pids = [spawn_monitor(fun() ->
        Items = [#{product_id => <<"prod_", (integer_to_binary(rand:uniform(5)))/binary>>,
                   quantity => 1}],
        Result = order_service:create(UserId, Items),
        exit({result, Result})
    end) || _ <- lists:seq(1, NumOrders)],
    
    %% Collect results
    Results = [receive
        {'DOWN', Ref, process, Pid, {result, R}} -> R
    end || {Pid, Ref} <- Pids],
    
    Successes = [ok || {ok, _} <- Results],
    Failures = [err || {error, _} <- Results],
    
    ct:pal("~p successes, ~p failures", [length(Successes), length(Failures)]),
    
    %% Verify no double inventory reservation
    Orders = order_service:list_by_user(UserId),
    ct:pal("Created ~p orders", [length(Orders)]),
    
    ok.
```

---

## Contract Testing (Consumer-Driven)

```erlang
%% contract_test.erl — verify service contracts between services

-module(order_service_contract_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, verify_create_order_contract/1,
         verify_get_order_contract/1]).

all() -> [verify_create_order_contract, verify_get_order_contract].

%% Verify that order service response matches expected contract
verify_create_order_contract(_Config) ->
    Request = #{user_id => <<"user1">>, items => [#{product_id => <<"p1">>, quantity => 1}]},
    
    {ok, 201, Body} = http_client:post("localhost", 8080, "/api/v1/orders",
                                        jiffy:encode(Request)),
    
    Response = jiffy:decode(Body, [return_maps]),
    
    %% Contract: response must have these fields
    true = maps:is_key(<<"id">>, Response),
    true = maps:is_key(<<"status">>, Response),
    true = maps:is_key(<<"created_at">>, Response),
    true = maps:is_key(<<"user_id">>, Response),
    
    %% Contract: field types
    true = is_binary(maps:get(<<"id">>, Response)),
    true = is_binary(maps:get(<<"status">>, Response)),
    true = is_integer(maps:get(<<"created_at">>, Response)),
    
    %% Contract: valid status values
    Status = maps:get(<<"status">>, Response),
    true = lists:member(Status, [<<"pending">>, <<"confirmed">>, <<"failed">>]),
    
    ok.

verify_get_order_contract(_Config) ->
    %% First create an order
    {ok, OrderId} = create_test_order(),
    
    {ok, 200, Body} = http_client:get("localhost", 8080, "/api/v1/orders/" ++ binary_to_list(OrderId)),
    
    Response = jiffy:decode(Body, [return_maps]),
    
    %% Contract: all required fields present
    RequiredFields = [<<"id">>, <<"user_id">>, <<"status">>, <<"items">>,
                      <<"total_amount">>, <<"created_at">>],
    
    [true = maps:is_key(F, Response) || F <- RequiredFields],
    
    %% Contract: items is a list
    true = is_list(maps:get(<<"items">>, Response)),
    
    ok.
```

---

## Chaos Engineering

```erlang
%% chaos.erl — controlled failure injection for resilience testing

-module(chaos).
-export([start_experiment/1, inject_failure/2, stop_experiment/1]).

-record(experiment, {
    id,
    type,
    target,
    duration_ms,
    start_time,
    metrics_before,
    metrics_after
}).

%% Run a chaos experiment
start_experiment(#{type := Type, target := Target, duration_ms := Duration}) ->
    ExperimentId = generate_id(),
    
    %% Capture baseline metrics
    MetricsBefore = capture_metrics(),
    
    %% Start the failure injection
    ok = inject_failure(Type, Target),
    
    %% Wait for duration
    timer:sleep(Duration),
    
    %% Stop failure injection
    ok = stop_failure(Type, Target),
    
    %% Wait for system to stabilize
    timer:sleep(5000),
    
    %% Capture post-experiment metrics
    MetricsAfter = capture_metrics(),
    
    %% Analyze results
    Analysis = analyze_experiment(MetricsBefore, MetricsAfter),
    
    logger:info("Chaos experiment complete", #{
        id => ExperimentId,
        type => Type,
        target => Target,
        analysis => Analysis
    }),
    
    {ok, Analysis}.

%% Failure types
inject_failure(network_partition, Nodes) ->
    %% Block network between node groups
    [erlang:disconnect_node(N) || N <- Nodes],
    ok;

inject_failure(cpu_spike, _Target) ->
    %% Spawn CPU-intensive processes
    [spawn(fun() -> busy_loop(60000) end) || _ <- lists:seq(1, erlang:system_info(schedulers_online))],
    ok;

inject_failure(memory_pressure, _Target) ->
    %% Allocate large binary
    spawn(fun() ->
        LargeBin = crypto:strong_rand_bytes(100 * 1024 * 1024),  %% 100MB
        timer:sleep(30000),
        _ = LargeBin
    end),
    ok;

inject_failure(process_kill, {supervisor, SupervisorName}) ->
    %% Kill a random child of the supervisor
    Children = supervisor:which_children(SupervisorName),
    {_, Pid, _, _} = lists:nth(rand:uniform(length(Children)), Children),
    exit(Pid, kill),
    ok;

inject_failure(slow_db, _) ->
    %% Make DB calls slow
    meck:expect(db, execute, fun(SQL, Params) ->
        timer:sleep(rand:uniform(5000)),  %% 0-5 second delay
        meck:passthrough([SQL, Params])
    end),
    ok;

inject_failure(db_down, _) ->
    %% Simulate DB unavailability
    meck:expect(db, execute, fun(_, _) -> {error, connection_refused} end),
    ok.

stop_failure(network_partition, Nodes) ->
    [net_adm:ping(N) || N <- Nodes],
    ok;
stop_failure(slow_db, _) ->
    meck:unload(db),
    ok;
stop_failure(db_down, _) ->
    meck:unload(db),
    ok;
stop_failure(_, _) ->
    ok.

busy_loop(Ms) ->
    Start = erlang:monotonic_time(millisecond),
    busy_loop_until(Start + Ms).

busy_loop_until(Until) ->
    case erlang:monotonic_time(millisecond) < Until of
        true -> busy_loop_until(Until);
        false -> ok
    end.

capture_metrics() ->
    #{
        p99_latency => metrics:get_percentile(request_latency, 99),
        error_rate => metrics:get_rate(errors, 60),
        throughput => metrics:get_rate(requests, 60),
        process_count => erlang:system_info(process_count),
        memory_mb => erlang:memory(total) div (1024 * 1024)
    }.

analyze_experiment(Before, After) ->
    #{
        latency_change => maps:get(p99_latency, After) - maps:get(p99_latency, Before),
        error_rate_change => maps:get(error_rate, After) - maps:get(error_rate, Before),
        throughput_change => maps:get(throughput, After) - maps:get(throughput, Before),
        system_recovered => maps:get(error_rate, After) < 0.01  %% <1% error rate
    }.

generate_id() ->
    base64:encode(crypto:strong_rand_bytes(6)).
```

---

## ตัวอย่างจริง: Full Test Suite

```erlang
%% test/full_suite_SUITE.erl — comprehensive test coverage

-module(full_suite_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, groups/0, suite/0]).
-export([init_per_suite/1, end_per_suite/1]).

suite() -> [{timetrap, {minutes, 30}}].

groups() ->
    [
        %% Fast unit tests (run first)
        {unit, [parallel], [
            {?MODULE, validate_order_input},
            {?MODULE, calculate_order_total},
            {?MODULE, user_authorization}
        ]},
        
        %% Integration tests (run sequentially in test DB)
        {integration, [sequence], [
            {?MODULE, create_user_flow},
            {?MODULE, place_order_flow},
            {?MODULE, cancel_order_flow}
        ]},
        
        %% Failure scenarios (run in parallel, each isolated)
        {failure, [parallel], [
            {?MODULE, db_connection_lost},
            {?MODULE, payment_service_timeout},
            {?MODULE, inventory_insufficient}
        ]},
        
        %% Property tests
        {property, [], [
            {?MODULE, order_total_always_positive},
            {?MODULE, user_id_uniqueness}
        ]}
    ].

all() ->
    [{group, unit}, {group, integration}, {group, failure}, {group, property}].

%% Sample test: order total calculation
calculate_order_total(_Config) ->
    Items = [
        #{price => 10.00, quantity => 2},   %% 20.00
        #{price => 5.50, quantity => 3}     %% 16.50
    ],
    
    Total = order_calculator:total(Items),
    
    %% Float comparison with epsilon
    true = abs(Total - 36.50) < 0.001,
    
    ok.

%% Property test: total is always non-negative
order_total_always_positive(_Config) ->
    proper:quickcheck(
        ?FORALL(Items, gen_items(),
            order_calculator:total(Items) >= 0
        ),
        [{numtests, 1000}, {to_file, ct:get_config(log_dir)}]
    ).

gen_items() ->
    proper_types:list(proper_types:map(
        proper_types:elements([price, quantity]),
        proper_types:union([
            proper_types:float(0, 10000),
            proper_types:integer(1, 100)
        ])
    )).
```

---

## สรุป Part 84

| Test Type | Tool | Speed | Coverage |
|-----------|------|-------|---------|
| Unit | EUnit | Fast | Functions |
| Property | PropEr | Medium | Edge cases |
| Integration | Common Test | Slow | Workflows |
| Contract | Custom | Medium | API contracts |
| Chaos | Custom | Very slow | Resilience |
| Load | wrk/tsung | Slow | Performance |

**Goal**: 80% line coverage + 100% critical path coverage + chaos tests for all external deps

---

*[← Part 83: Microservices](part_83_microservices.md) | [Part 85: Data Pipelines →](part_85_data_pipelines.md)*
