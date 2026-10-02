# Part 30: Testing — EUnit, Common Test, PropEr

## สารบัญ
1. [EUnit — Unit Testing](#eunit--unit-testing)
2. [EUnit Assertions](#eunit-assertions)
3. [EUnit Test Organization](#eunit-test-organization)
4. [Common Test — Integration Testing](#common-test--integration-testing)
5. [PropEr — Property-Based Testing](#proper--property-based-testing)
6. [Mocking กับ Meck](#mocking-กับ-meck)
7. [Code Coverage](#code-coverage)
8. [ตัวอย่างจริง: Testing gen_server](#ตัวอย่างจริง-testing-gen_server)

---

## EUnit — Unit Testing

```erlang
%% EUnit: framework ที่มากับ OTP stdlib
%% test functions ลงท้ายด้วย _test หรือ _test_ (generator)
%% รัน: rebar3 eunit

%% ไฟล์: src/math_utils.erl
-module(math_utils).
-export([factorial/1, fibonacci/1, is_prime/1]).

factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N - 1).

fibonacci(0) -> 0;
fibonacci(1) -> 1;
fibonacci(N) when N > 1 -> fibonacci(N-1) + fibonacci(N-2).

is_prime(2) -> true;
is_prime(N) when N < 2 -> false;
is_prime(N) when N rem 2 =:= 0 -> false;
is_prime(N) ->
    Limit = trunc(math:sqrt(N)),
    not lists:any(fun(I) -> N rem I =:= 0 end, lists:seq(3, Limit, 2)).


%% ไฟล์: test/math_utils_tests.erl
-module(math_utils_tests).
-include_lib("eunit/include/eunit.hrl").

%% Simple test (ลงท้ายด้วย _test)
factorial_basic_test() ->
    ?assertEqual(1, math_utils:factorial(0)),
    ?assertEqual(1, math_utils:factorial(1)),
    ?assertEqual(120, math_utils:factorial(5)).

fibonacci_test() ->
    ?assertEqual(0, math_utils:fibonacci(0)),
    ?assertEqual(1, math_utils:fibonacci(1)),
    ?assertEqual(8, math_utils:fibonacci(6)).

is_prime_test() ->
    ?assert(math_utils:is_prime(2)),
    ?assert(math_utils:is_prime(7)),
    ?assert(math_utils:is_prime(97)),
    ?assertNot(math_utils:is_prime(1)),
    ?assertNot(math_utils:is_prime(4)),
    ?assertNot(math_utils:is_prime(100)).

%% Test generator (ลงท้ายด้วย _test_) -> returns list of tests
factorial_values_test_() ->
    [
        ?_assertEqual(1, math_utils:factorial(0)),
        ?_assertEqual(2, math_utils:factorial(2)),
        ?_assertEqual(6, math_utils:factorial(3)),
        ?_assertEqual(24, math_utils:factorial(4)),
        ?_assertEqual(120, math_utils:factorial(5))
    ].

%% Foreach: run same test with multiple inputs
prime_list_test_() ->
    Primes = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29],
    [?_assert(math_utils:is_prime(P)) || P <- Primes].

%% รัน EUnit
%% $ rebar3 eunit
%% หรือจาก shell:
%% eunit:test(math_utils_tests).
%% eunit:test(math_utils_tests, [verbose]).
```

---

## EUnit Assertions

```erlang
-include_lib("eunit/include/eunit.hrl").

%% Basic assertions
?assert(Expr)                   %% true
?assertNot(Expr)                %% false
?assertEqual(Expected, Got)     %% =:= (strict equal)
?assertNotEqual(V1, V2)        %% =/=
?assertMatch(Pattern, Expr)     %% pattern match
?assertNoMatch(Pattern, Expr)
?assertError(Pattern, Expr)     %% should throw error
?assertExit(Pattern, Expr)      %% should call exit
?assertThrow(Pattern, Expr)     %% should throw

%% Examples
?assert(1 + 1 =:= 2).
?assertEqual(<<"hello">>, string:lowercase(<<"HELLO">>)).
?assertMatch({ok, _}, some_function()).
?assertMatch({ok, #{name := "Alice"}}, get_user(1)).
?assertError(badarg, erlang:binary_to_integer(<<"abc">>)).
?assertThrow({not_found, user}, get_user(-1)).

%% Test generators (lazy, _test_ suffix)
?_assert(Expr)
?_assertEqual(Expected, Got)
%% etc. - same as above but returns test fun

%% Setup and teardown
-define(SETUP(F), {setup, fun setup/0, fun teardown/1, F}).

setup() ->
    {ok, Pid} = my_server:start_link(),
    Pid.

teardown(Pid) ->
    gen_server:stop(Pid).

server_test_() ->
    ?SETUP(fun(Pid) ->
        [
            ?_assertEqual(0, my_server:get_count(Pid)),
            ?_assertEqual(ok, my_server:increment(Pid)),
            ?_assertEqual(1, my_server:get_count(Pid))
        ]
    end).

%% Timeout test
slow_operation_test_() ->
    {timeout, 30, fun() ->
        Result = slow_function(),
        ?assertEqual(ok, Result)
    end}.
```

---

## EUnit Test Organization

```erlang
%% Descriptive test groups
my_module_test_() ->
    {"my_module tests",
     [
         {"Basic operations",
          [
              ?_assertEqual(2, add(1, 1)),
              ?_assertEqual(0, add(1, -1))
          ]},
         {"Edge cases",
          [
              ?_assertEqual(0, add(0, 0)),
              ?_assertError(badarg, add(a, b))
          ]}
     ]}.

%% Foreachx: run test with each item
data_test_() ->
    Data = [
        {1, 1},
        {2, 2},
        {5, 120}
    ],
    {foreachx,
     fun({N, _}) -> io:format("Testing factorial(~p)~n", [N]) end,
     fun(_, _) -> ok end,
     [
         {Item, fun({N, Expected}, _) ->
             ?_assertEqual(Expected, factorial(N))
         end}
         || Item = {_, _} <- Data
     ]}.

%% Inline tests ใน module itself (alternative style)
-ifdef(TEST).
-include_lib("eunit/include/eunit.hrl").

add_test() ->
    ?assertEqual(3, add(1, 2)).

-endif.

%% rebar3 eunit สนับสนุนทั้ง separate test files และ inline tests
```

---

## Common Test — Integration Testing

```erlang
%% Common Test (CT): integration/system testing
%% ไฟล์: test/my_SUITE.erl
%% CT ต้องการ _SUITE suffix

-module(my_SUITE).
-include_lib("common_test/include/ct.hrl").

%% Required callbacks
-export([all/0, suite/0, init_per_suite/1, end_per_suite/1,
         init_per_testcase/2, end_per_testcase/2]).

%% Test cases
-export([
    test_basic_operations/1,
    test_concurrent_access/1,
    test_error_handling/1
]).

%% list all test cases
all() ->
    [test_basic_operations, test_concurrent_access, test_error_handling].

%% suite metadata
suite() ->
    [{timetrap, {seconds, 30}}].  %% default per-test timeout

%% Setup for entire test suite
init_per_suite(Config) ->
    %% Start applications
    {ok, _} = application:ensure_all_started(my_app),
    Config.   %% Config = proplist, เพิ่ม data ที่ต้องการ pass ไป tests

end_per_suite(_Config) ->
    application:stop(my_app),
    ok.

%% Setup per test case
init_per_testcase(_TestCase, Config) ->
    %% Create fresh state
    {ok, Pid} = my_server:start_link(),
    [{server_pid, Pid} | Config].

end_per_testcase(_TestCase, Config) ->
    Pid = proplists:get_value(server_pid, Config),
    gen_server:stop(Pid).

%% Test cases
test_basic_operations(Config) ->
    Pid = proplists:get_value(server_pid, Config),
    ok = my_server:put(Pid, key1, value1),
    {ok, value1} = my_server:get(Pid, key1),
    ok = my_server:delete(Pid, key1),
    miss = my_server:get(Pid, key1).

test_concurrent_access(Config) ->
    Pid = proplists:get_value(server_pid, Config),
    %% Spawn 100 concurrent clients
    Self = self(),
    Pids = [
        spawn(fun() ->
            ok = my_server:put(Pid, {key, I}, I),
            Self ! {done, I}
        end)
        || I <- lists:seq(1, 100)
    ],
    %% Wait for all
    [receive {done, I} -> ok end || I <- lists:seq(1, 100)],
    %% Verify
    100 = my_server:size(Pid),
    _ = Pids.

test_error_handling(Config) ->
    Pid = proplists:get_value(server_pid, Config),
    %% Test invalid operations
    {error, not_found} = my_server:update(Pid, nonexistent, fun(V) -> V end).

%% รัน CT
%% $ rebar3 ct
%% $ rebar3 ct --suite my_SUITE
%% $ rebar3 ct --case test_basic_operations

%% CT groups: organize related tests
groups() ->
    [
        {smoke_tests, [parallel], [test_basic_operations]},
        {load_tests, [sequence, {timetrap, {minutes, 5}}], [test_concurrent_access]}
    ].

all() -> [{group, smoke_tests}, {group, load_tests}].
```

---

## PropEr — Property-Based Testing

```erlang
%% PropEr: generates random test cases, finds edge cases automatically
%% ใส่ใน deps: {proper, "1.4.0"}

-module(math_props).
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").

%% Property: ประกาศ invariant ที่ต้องเป็นจริงเสมอ
%% PropEr จะ generate random inputs ทดสอบ

%% factorial คืนค่า >= 1 เสมอ
prop_factorial_positive() ->
    ?FORALL(N, pos_integer(),
        math_utils:factorial(N) >= 1
    ).

%% factorial(N) > factorial(N-1) สำหรับ N > 1
prop_factorial_increasing() ->
    ?FORALL(N, ?SUCHTHAT(X, pos_integer(), X > 1),
        math_utils:factorial(N) > math_utils:factorial(N - 1)
    ).

%% List reverse twice = original
prop_list_double_reverse() ->
    ?FORALL(L, list(integer()),
        lists:reverse(lists:reverse(L)) =:= L
    ).

%% Map put/get roundtrip
prop_map_roundtrip() ->
    ?FORALL({Key, Value}, {term(), term()},
        begin
            Map = #{Key => Value},
            maps:get(Key, Map) =:= Value
        end
    ).

%% Custom generators
pos_integer() -> ?SUCHTHAT(N, integer(), N > 0).

non_empty_list(Gen) -> non_empty(list(Gen)).

user_gen() ->
    ?LET({Name, Age, Email},
         {non_empty(list(char())), integer(18, 100), binary()},
         #{name => list_to_binary(Name), age => Age, email => Email}).

%% Property with custom generator
prop_user_validation() ->
    ?FORALL(User, user_gen(),
        case validate_user(User) of
            ok -> true;
            {error, _} -> false
        end
    ).

%% Stateful property testing
prop_counter_model() ->
    ?FORALL(Commands, commands(?MODULE),
        begin
            {ok, Pid} = counter:start_link(),
            {History, State, Result} = run_commands(?MODULE, Commands, [{pid, Pid}]),
            gen_server:stop(Pid),
            ?WHENFAIL(
                io:format("History: ~p\nState: ~p\nResult: ~p\n",
                          [History, State, Result]),
                aggregate(command_names(Commands), Result =:= ok)
            )
        end
    ).

%% รัน PropEr ผ่าน EUnit
proper_test_() ->
    {timeout, 60,
     ?_assertEqual([], proper:module(?MODULE, [{numtests, 100}, nocolors]))}.

%% รัน จาก shell
proper:quickcheck(math_props:prop_list_double_reverse()).
proper:quickcheck(math_props:prop_list_double_reverse(), [{numtests, 1000}]).
```

---

## Mocking กับ Meck

```erlang
%% Meck: mock modules for testing
%% {meck, "0.9.2"} ใน test profile deps

-module(user_service_test).
-include_lib("eunit/include/eunit.hrl").

%% Test with mocked dependency
get_user_test() ->
    %% Mock db module
    meck:new(db, [non_strict]),
    meck:expect(db, find_user, fun(1) -> {ok, #{id => 1, name => "Alice"}} end),

    %% Test
    {ok, User} = user_service:get_user(1),
    ?assertEqual("Alice", maps:get(name, User)),

    %% Verify mock was called
    ?assert(meck:called(db, find_user, [1])),
    ?assertEqual(1, meck:num_calls(db, find_user, [1])),

    %% Cleanup (important!)
    meck:unload(db).

%% Mock with passthrough (call original for some args)
partial_mock_test() ->
    meck:new(my_module, [passthrough]),
    meck:expect(my_module, expensive_function, fun(_) -> cached_result end),
    % all other functions work normally
    meck:unload(my_module).

%% Setup/teardown pattern with meck
-define(setup(F), {setup, fun setup/0, fun teardown/1, F}).

setup() ->
    meck:new([db, cache], [non_strict]),
    meck:expect(db, find_user, fun(Id) -> {ok, #{id => Id}} end),
    meck:expect(cache, get, fun(_) -> miss end),
    meck:expect(cache, put, fun(_, _) -> ok end).

teardown(_) ->
    meck:unload().

user_service_test_() ->
    ?setup(fun(_) ->
        [
            ?_assertMatch({ok, #{id := 1}}, user_service:get_user(1)),
            ?_assert(meck:called(db, find_user, [1]))
        ]
    end).
```

---

## Code Coverage

```erlang
%% rebar.config
{cover_enabled, true}.
{cover_opts, [verbose]}.
{cover_excl_mods, [my_module_tests]}.  %% exclude test files

%% รัน tests + coverage
%% $ rebar3 eunit
%% $ rebar3 cover  %% generate HTML report

%% หรือรวม
%% $ rebar3 do eunit, cover

%% HTML report: _build/test/cover/index.html

%% Minimum coverage threshold (ตั้งใน CI)
{cover_opts, [
    verbose,
    {min_coverage, 80}  %% fail if < 80%
]}.

%% cover module ใน shell
cover:compile_directory("src").
cover:start().
my_module:some_function().  %% run code
cover:analyse_to_file(my_module, "my_module.html", [html]).
cover:stop().
```

---

## ตัวอย่างจริง: Testing gen_server

```erlang
%% ไฟล์: test/kv_store_tests.erl
-module(kv_store_tests).
-include_lib("eunit/include/eunit.hrl").

%% Test fixture
setup() ->
    {ok, Pid} = kv_store:start_link(),
    Pid.

teardown(Pid) ->
    gen_server:stop(Pid),
    timer:sleep(50).  %% wait for process to stop

kv_store_test_() ->
    {foreach,
     fun setup/0,
     fun teardown/1,
     [
         fun test_put_get/1,
         fun test_ttl_expiry/1,
         fun test_delete/1,
         fun test_flush/1,
         fun test_stats/1,
         fun test_concurrent/1
     ]
    }.

test_put_get(_Pid) ->
    [
        fun() ->
            kv_store:put(k1, v1),
            ?assertEqual({ok, v1}, kv_store:get(k1))
        end,
        fun() ->
            ?assertEqual(miss, kv_store:get(nonexistent))
        end,
        fun() ->
            kv_store:put(k2, <<"binary">>),
            ?assertMatch({ok, <<"binary">>}, kv_store:get(k2))
        end
    ].

test_ttl_expiry(_Pid) ->
    fun() ->
        kv_store:put(expire_key, value, 1),  %% 1 second TTL
        ?assertMatch({ok, value}, kv_store:get(expire_key)),
        timer:sleep(1100),
        ?assertEqual(miss, kv_store:get(expire_key))
    end.

test_delete(_Pid) ->
    fun() ->
        kv_store:put(del_key, value),
        ?assertMatch({ok, value}, kv_store:get(del_key)),
        kv_store:delete(del_key),
        ?assertEqual(miss, kv_store:get(del_key))
    end.

test_flush(_Pid) ->
    fun() ->
        kv_store:put(a, 1),
        kv_store:put(b, 2),
        kv_store:put(c, 3),
        ?assertEqual(3, kv_store:size()),
        kv_store:flush(),
        ?assertEqual(0, kv_store:size())
    end.

test_stats(_Pid) ->
    fun() ->
        kv_store:put(s1, v1),
        kv_store:get(s1),   %% hit
        kv_store:get(s1),   %% hit
        kv_store:get(s99),  %% miss
        Stats = kv_store:stats(),
        ?assertEqual(2, maps:get(hits, Stats)),
        ?assertEqual(1, maps:get(misses, Stats)),
        ?assert(maps:get(hit_rate, Stats) > 0.6)
    end.

test_concurrent(_Pid) ->
    fun() ->
        Self = self(),
        N = 100,
        [spawn(fun() ->
            kv_store:put({key, I}, I),
            Self ! done
        end) || I <- lists:seq(1, N)],
        [receive done -> ok end || _ <- lists:seq(1, N)],
        ?assertEqual(N, kv_store:size())
    end.
```

---

## สรุป Part 30

| Framework | ใช้เมื่อ | Command |
|-----------|---------|---------|
| EUnit | Unit tests, fast | `rebar3 eunit` |
| Common Test | Integration, System | `rebar3 ct` |
| PropEr | Property-based | `rebar3 eunit` |
| Meck | Mocking | (library) |

| Assertion | ความหมาย |
|-----------|---------|
| `?assert(E)` | E เป็น true |
| `?assertEqual(A, B)` | A =:= B |
| `?assertMatch(P, E)` | E matches P |
| `?assertError(P, E)` | E throws error matching P |

---

*[← Part 29: Rebar3](part_29_rebar3.md) | [Part 31: Distributed Erlang →](part_31_distributed.md)*

---
> **สำเร็จ! จบ Section 1: Beginner (Parts 1-30)**
> Parts 31+ จะเข้าสู่ระดับ Intermediate: Distributed Erlang, Networking, Databases, Advanced OTP Patterns
