# Part 62: Advanced Testing

## สารบัญ
1. [Property-Based Testing กับ PropEr](#property-based)
2. [Mutation Testing](#mutation-testing)
3. [Load Testing](#load-testing)
4. [Chaos Testing](#chaos-testing)
5. [Contract Testing](#contract-testing)
6. [ตัวอย่างจริง: Test Suite สำหรับ Distributed System](#distributed-test-suite)

---

## Property-Based Testing กับ PropEr

```erlang
%% PropEr: test properties (ไม่ใช่ examples)
%% ใส่ใน rebar.config: {deps, [{proper, "1.4.0"}]}

-module(account_prop_tests).
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").

%% Property: balance never goes negative
prop_balance_non_negative() ->
    ?FORALL(Ops, list(account_op()),
        begin
            FinalBalance = apply_ops(Ops, 1000),
            FinalBalance >= 0
        end).

%% Custom generator: valid account operations
account_op() ->
    frequency([
        {5, {deposit, range(1, 1000)}},
        {3, {withdraw, range(1, 500)}},
        {1, {set, range(0, 10000)}}
    ]).

apply_ops([], Balance) -> Balance;
apply_ops([{deposit, N} | Rest], Balance) ->
    apply_ops(Rest, Balance + N);
apply_ops([{withdraw, N} | Rest], Balance) ->
    case Balance - N >= 0 of
        true -> apply_ops(Rest, Balance - N);
        false -> apply_ops(Rest, Balance)  %% reject if would go negative
    end;
apply_ops([{set, N} | Rest], _) ->
    apply_ops(Rest, N).

%% Property: list sort is idempotent
prop_sort_idempotent() ->
    ?FORALL(List, list(integer()),
        lists:sort(lists:sort(List)) =:= lists:sort(List)).

%% Property: encoding/decoding roundtrip
prop_json_roundtrip() ->
    ?FORALL(Map, json_map(),
        begin
            Encoded = jsx:encode(Map),
            Decoded = jsx:decode(Encoded, [return_maps]),
            normalize(Map) =:= normalize(Decoded)
        end).

json_map() ->
    map(binary(), oneof([integer(), float(), binary(), boolean(), null])).

%% Property: concurrent operations are linearizable
prop_concurrent_counter() ->
    ?FORALL({N, Ops}, {range(1, 10), list(range(1, 100))},
        begin
            {ok, C} = counter:start_link(0),
            %% Spawn N workers, each applying Ops
            Pids = [spawn(fun() ->
                [counter:increment(C, Op) || Op <- Ops]
            end) || _ <- lists:seq(1, N)],
            %% Wait for all workers
            timer:sleep(100),
            Expected = N * lists:sum(Ops),
            counter:value(C) =:= Expected
        end).

%% Run properties
run_all() ->
    proper:quickcheck(prop_balance_non_negative(), [{numtests, 1000}]),
    proper:quickcheck(prop_sort_idempotent(), [{numtests, 500}]),
    proper:quickcheck(prop_json_roundtrip(), [{numtests, 200}]).

%% Shrinking: PropEr automatically finds minimal failing case
%% If prop fails with [1,2,3,4,5], it tries [1,2,3,4], [1,2,3]... → minimal example
```

---

## Mutation Testing

```erlang
%% Mutation testing: change code, verify tests catch it
%% Use mumak or manual mutations

%% Original code
-module(calculator).
balance_check(Amount, Balance) ->
    Amount =< Balance.

transfer(From, To, Amount) ->
    FromBalance = get_balance(From),
    case balance_check(Amount, FromBalance) of
        true ->
            update_balance(From, FromBalance - Amount),
            update_balance(To, get_balance(To) + Amount),
            ok;
        false ->
            {error, insufficient_funds}
    end.

%% Mutations to test:
%% 1. Change =< to < (off-by-one)
%% 2. Change + to - in update
%% 3. Remove balance_check call
%% 4. Swap From/To in update_balance

%% Good test suite catches ALL mutations:
-module(calculator_tests).
-include_lib("eunit/include/eunit.hrl").

balance_check_test_() ->
    [
        %% Catches mutation 1 (=< vs <)
        ?_assert(calculator:balance_check(100, 100)),   %% exact amount OK
        ?_assert(calculator:balance_check(99, 100)),    %% less OK
        ?_assertNot(calculator:balance_check(101, 100)) %% more NOT OK
    ].

transfer_test_() ->
    {setup,
     fun() ->
         ets:new(balances, [named_table, public]),
         ets:insert(balances, [{alice, 1000}, {bob, 500}])
     end,
     fun(_) -> ets:delete(balances) end,
     [
         %% Normal transfer
         ?_assertEqual(ok, calculator:transfer(alice, bob, 200)),
         %% Catches mutation 2 (+ vs -)
         ?_assertEqual(800, get_balance(alice)),
         ?_assertEqual(700, get_balance(bob)),
         %% Catches mutation 3 (removed check)
         ?_assertEqual({error, insufficient_funds}, calculator:transfer(alice, bob, 900))
     ]}.
```

---

## Load Testing

```erlang
%% Load testing: verify system under expected/peak load

-module(load_tester).
-export([run/2, run_scenario/3]).

%% Configuration
run(Config, Scenario) ->
    #{
        concurrent_users => ConcUsers,
        duration_seconds => Duration,
        ramp_up_seconds => RampUp
    } = Config,
    
    Results = run_scenario(Scenario, ConcUsers, Duration, RampUp),
    analyze_results(Results).

run_scenario(ScenarioFun, ConcUsers, DurationSec, RampUpSec) ->
    Parent = self(),
    StartTime = erlang:monotonic_time(millisecond),
    EndTime = StartTime + DurationSec * 1000,
    
    %% Ramp up users gradually
    RampInterval = RampUpSec * 1000 div ConcUsers,
    
    Workers = [begin
        timer:sleep(I * RampInterval),
        spawn_link(fun() ->
            worker_loop(ScenarioFun, EndTime, Parent)
        end)
    end || I <- lists:seq(0, ConcUsers - 1)],
    
    %% Collect results
    collect_results(Workers, StartTime, EndTime, []).

worker_loop(ScenarioFun, EndTime, Parent) ->
    Now = erlang:monotonic_time(millisecond),
    if Now >= EndTime ->
        ok;
    true ->
        {Latency, Result} = measure(ScenarioFun),
        Parent ! {result, self(), #{latency => Latency, result => Result}},
        worker_loop(ScenarioFun, EndTime, Parent)
    end.

measure(Fun) ->
    Start = erlang:monotonic_time(microsecond),
    Result = try {ok, Fun()}
    catch _:R -> {error, R}
    end,
    End = erlang:monotonic_time(microsecond),
    {End - Start, Result}.

collect_results(Workers, StartTime, EndTime, Acc) ->
    Now = erlang:monotonic_time(millisecond),
    if Now >= EndTime + 5000 ->  %% 5s buffer after end
        Acc;
    true ->
        receive
            {result, _Pid, Result} ->
                collect_results(Workers, StartTime, EndTime, [Result | Acc])
        after 100 ->
            collect_results(Workers, StartTime, EndTime, Acc)
        end
    end.

analyze_results(Results) ->
    {Oks, Errors} = lists:partition(
        fun(#{result := {ok, _}}) -> true; (_) -> false end,
        Results
    ),
    Latencies = [L || #{latency := L} <- Oks],
    SortedLat = lists:sort(Latencies),
    N = length(SortedLat),
    #{
        total_requests => length(Results),
        successful => length(Oks),
        failed => length(Errors),
        error_rate => length(Errors) / length(Results) * 100,
        p50_ms => percentile(SortedLat, 50) / 1000,
        p95_ms => percentile(SortedLat, 95) / 1000,
        p99_ms => percentile(SortedLat, 99) / 1000,
        max_ms => lists:last(SortedLat) / 1000,
        rps => N / (lists:last(SortedLat) / 1000000)
    }.

percentile(Sorted, P) ->
    Idx = round(length(Sorted) * P / 100),
    lists:nth(max(1, min(Idx, length(Sorted))), Sorted).

%% HTTP load test scenario
http_scenario() ->
    fun() ->
        Url = "http://localhost:8080/api/users",
        {ok, {{_, 200, _}, _, _}} = httpc:request(get, {Url, []}, [{timeout, 5000}], []),
        ok
    end.

%% Usage
example() ->
    Config = #{
        concurrent_users => 100,
        duration_seconds => 60,
        ramp_up_seconds => 10
    },
    Results = load_tester:run(Config, http_scenario()),
    io:format("Load test results: ~p~n", [Results]).
```

---

## ตัวอย่างจริง: Distributed System Tests

```erlang
%% Common Test suite for distributed Erlang system

-module(distributed_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, groups/0, init_per_suite/1, end_per_suite/1]).
-export([test_leader_election/1, test_data_replication/1, test_partition_healing/1]).

all() -> [{group, distributed}].

groups() ->
    [{distributed, [sequence], [
        test_leader_election,
        test_data_replication,
        test_partition_healing
    ]}].

init_per_suite(Config) ->
    %% Start 3-node cluster
    Nodes = start_cluster(3),
    [{nodes, Nodes} | Config].

end_per_suite(Config) ->
    [rpc:call(N, init, stop, []) || N <- proplists:get_value(nodes, Config)].

test_leader_election(Config) ->
    Nodes = proplists:get_value(nodes, Config),
    
    %% Kill leader
    {ok, Leader} = find_leader(Nodes),
    ct:pal("Killing leader: ~p", [Leader]),
    rpc:call(Leader, erlang, halt, [0]),
    
    %% Wait for new election
    timer:sleep(500),
    
    %% Verify new leader elected
    RemainingNodes = Nodes -- [Leader],
    {ok, NewLeader} = find_leader_eventually(RemainingNodes, 5000),
    ct:pal("New leader: ~p", [NewLeader]),
    
    true = (NewLeader /= Leader).

test_data_replication(Config) ->
    [Node1, Node2 | _] = proplists:get_value(nodes, Config),
    
    %% Write on node1
    Key = <<"test_key">>,
    Value = <<"test_value_", (integer_to_binary(erlang:unique_integer([positive])))/binary>>,
    ok = rpc:call(Node1, kv_store, put, [Key, Value]),
    
    %% Read from node2 (should replicate)
    timer:sleep(100),
    {ok, Value} = rpc:call(Node2, kv_store, get, [Key]).

test_partition_healing(Config) ->
    [Node1, Node2, Node3] = proplists:get_value(nodes, Config),
    
    %% Partition node3 from cluster
    simulate_partition(Node3, [Node1, Node2]),
    timer:sleep(200),
    
    %% Write during partition
    ok = rpc:call(Node1, kv_store, put, [<<"partitioned">>, <<"value1">>]),
    ok = rpc:call(Node3, kv_store, put, [<<"partitioned">>, <<"value2">>]),
    
    %% Heal partition
    heal_partition(Node3, [Node1, Node2]),
    timer:sleep(500),
    
    %% Verify consistent state (last write wins or CRDT merge)
    V1 = rpc:call(Node1, kv_store, get, [<<"partitioned">>]),
    V2 = rpc:call(Node3, kv_store, get, [<<"partitioned">>]),
    ct:pal("After heal: node1=~p, node3=~p", [V1, V2]),
    V1 = V2.  %% must converge

%% Helpers
start_cluster(N) ->
    [begin
        NodeName = list_to_atom("testnode" ++ integer_to_list(I) ++ "@localhost"),
        {ok, Node} = slave:start(localhost, list_to_atom("testnode" ++ integer_to_list(I)),
            "-pa " ++ code:lib_dir() ++ "/ebin"),
        rpc:call(Node, kv_store, start, []),
        Node
    end || I <- lists:seq(1, N)].

find_leader(Nodes) ->
    Leaders = [N || N <- Nodes, rpc:call(N, kv_store, is_leader, []) =:= true],
    case Leaders of
        [L | _] -> {ok, L};
        [] -> {error, no_leader}
    end.

find_leader_eventually(_, Timeout) when Timeout =< 0 -> {error, timeout};
find_leader_eventually(Nodes, Timeout) ->
    case find_leader(Nodes) of
        {ok, _} = Ok -> Ok;
        {error, _} ->
            timer:sleep(100),
            find_leader_eventually(Nodes, Timeout - 100)
    end.

simulate_partition(Node, Others) ->
    rpc:call(Node, net_kernel, disconnect, [N] || N <- Others).

heal_partition(Node, Others) ->
    [net_adm:ping_node(Node, N) || N <- Others].
```

---

## สรุป Part 62

| Testing Type | Tool | Purpose |
|--------------|------|---------|
| Unit | EUnit | Function-level |
| Integration | Common Test | System-level |
| Property | PropEr | Invariants |
| Mutation | Manual/mumak | Test quality |
| Load | Custom | Performance |
| Chaos | Custom | Resilience |
| Contract | Custom | API compatibility |

---

*[← Part 61: CRDTs](part_61_crdts.md) | [Part 63: Grafana Monitoring →](part_63_monitoring.md)*
