# Part 16: Higher-Order Functions — ฟังก์ชันที่รับ/คืนฟังก์ชัน

## สารบัญ
1. [Higher-Order Functions คืออะไร](#higher-order-functions-คืออะไร)
2. [Anonymous Functions (Funs)](#anonymous-functions-funs)
3. [Function References](#function-references)
4. [Closures](#closures)
5. [Map, Filter, Fold](#map-filter-fold)
6. [Function Composition](#function-composition)
7. [Currying and Partial Application](#currying-and-partial-application)
8. [ตัวอย่างจริง: Event System](#ตัวอย่างจริง-event-system)

---

## Higher-Order Functions คืออะไร

ฟังก์ชันที่รับ function เป็น argument หรือ return function เป็นค่า:

```erlang
%% ฟังก์ชันที่รับ function เป็น argument
apply_twice(F, X) -> F(F(X)).

apply_twice(fun(X) -> X * 2 end, 3).   %% = 12 (3*2=6, 6*2=12)
apply_twice(fun(X) -> X + 1 end, 5).   %% = 7

%% ฟังก์ชันที่ return function
make_adder(N) ->
    fun(X) -> X + N end.

Add5 = make_adder(5).
Add5(10).   %% = 15
Add5(3).    %% = 8

Make10 = make_adder(10).
Make10(7).  %% = 17

%% Higher-order functions ทำให้โค้ดยืดหยุ่น
transform_list(List, Transformer) ->
    [Transformer(X) || X <- List].

transform_list([1,2,3], fun(X) -> X * X end).  %% = [1, 4, 9]
transform_list(["a","b","c"], fun string:uppercase/1).  %% = ["A","B","C"]
```

---

## Anonymous Functions (Funs)

```erlang
%% Syntax: fun(Args) -> Body end

%% Single-clause fun
Double = fun(X) -> X * 2 end.
Double(5).   %% = 10

%% Multi-clause fun (pattern matching)
Describe = fun
    (0) -> "zero";
    (N) when N > 0 -> "positive";
    (_) -> "negative"
end.

Describe(0).    %% = "zero"
Describe(5).    %% = "positive"
Describe(-3).   %% = "negative"

%% Fun กับหลาย arguments
Add = fun(X, Y) -> X + Y end.
Add(3, 4).   %% = 7

%% Fun with guards
SafeDiv = fun
    (_, 0) -> {error, division_by_zero};
    (A, B) -> {ok, A / B}
end.

SafeDiv(10, 2).  %% = {ok, 5.0}
SafeDiv(10, 0).  %% = {error, division_by_zero}

%% Immediately invoked fun
(fun(X) -> X * X end)(7).   %% = 49

%% Named fun (สำหรับ recursion ใน fun)
Fib = fun Fib(0) -> 0;
          Fib(1) -> 1;
          Fib(N) -> Fib(N-1) + Fib(N-2)
      end.

Fib(10).  %% = 55

%% Fun ที่ return fun
Counter = fun() ->
    Count = 0,
    fun() -> Count + 1 end  %% ไม่ถูก! Erlang ไม่มี mutable state
end.
%% ต้องใช้ process state แทน
```

---

## Function References

```erlang
%% อ้างถึง named function ด้วย fun Module:Function/Arity
%% หรือ fun Function/Arity (local)

%% Local function reference
double(X) -> X * 2.

F = fun double/1.
F(5).    %% = 10

lists:map(fun double/1, [1,2,3]).   %% = [2,4,6]

%% Module function reference
lists:map(fun integer_to_list/1, [1,2,3]).
%% = ["1", "2", "3"]

lists:map(fun string:uppercase/1, ["hello", "world"]).
%% = ["HELLO", "WORLD"]

%% BIF reference
lists:filter(fun is_integer/1, [1, a, 2.0, b, 3]).
%% = [1, 3]

%% Comparison: fun literal vs fun reference
%% เหมือนกันในทางปฏิบัติ แต่ reference อ้างถึง version ล่าสุดของ function
F1 = fun(X) -> X * 2 end.  %% anonymous fun - fixed
F2 = fun double/1.          %% reference - ถ้า hot reload double/1 -> F2 ชี้ไปที่ version ใหม่

%% Function arity check
is_function(F1).       %% = true
is_function(F1, 1).    %% = true
is_function(F1, 2).    %% = false

%% Introspection
erlang:fun_info(F1).
%% = [{module, ...}, {name, ...}, {arity, 1}, ...]

%% Apply function dynamically
erlang:apply(fun lists:sum/1, [[1,2,3]]).  %% = 6
erlang:apply(Fun, Args).  %% apply fun to list of args
```

---

## Closures

```erlang
%% Closure = fun ที่ capture variables จาก scope รอบข้าง

Multiplier = 5.
Times5 = fun(X) -> X * Multiplier end.
Times5(3).   %% = 15
Times5(7).   %% = 35

%% Closure ที่ยิ่งซับซ้อน
make_greeting(Greeting) ->
    fun(Name) ->
        <<Greeting/binary, ", ", Name/binary, "!">>
    end.

Hello = make_greeting(<<"Hello">>).
Sawasdee = make_greeting(<<"สวัสดี">>).

Hello(<<"Alice">>).      %% = <<"Hello, Alice!">>
Sawasdee(<<"สมชาย">>).   %% = <<"สวัสดี, สมชาย!">>

%% Closure capture คือ value ณ เวลาที่สร้าง fun
make_counter_snapshot(Current) ->
    fun() -> Current end.  %% capture current value

C1 = make_counter_snapshot(0).
C2 = make_counter_snapshot(10).
C1().   %% = 0
C2().   %% = 10

%% ตัวอย่าง: Middleware pattern
make_logger(F) ->
    fun(Input) ->
        io:format("Input: ~p~n", [Input]),
        Result = F(Input),
        io:format("Output: ~p~n", [Result]),
        Result
    end.

LoggedDouble = make_logger(fun double/1).
LoggedDouble(5).
%% Output:
%% Input: 5
%% Output: 10
%% = 10

%% ตัวอย่าง: Configuration-driven factory
make_http_client(Config) ->
    Host = maps:get(host, Config),
    Port = maps:get(port, Config, 80),
    Timeout = maps:get(timeout, Config, 5000),
    fun(Path) ->
        httpc:request(get, {io_lib:format("http://~s:~p~s", [Host, Port, Path]), []},
                      [{timeout, Timeout}], [])
    end.

GitHub = make_http_client(#{host => "api.github.com", port => 443}),
GitHub("/users/octocat").
```

---

## Map, Filter, Fold

```erlang
%% เป็น Higher-Order Functions ที่ใช้บ่อยที่สุด

%% map: transform each element
lists:map(fun(X) -> X * 2 end, [1,2,3,4,5]).
%% = [2,4,6,8,10]

lists:map(fun string:uppercase/1, ["hello", "world"]).
%% = ["HELLO", "WORLD"]

%% filter: keep elements that satisfy predicate
lists:filter(fun(X) -> X > 3 end, [1,2,3,4,5]).
%% = [4,5]

lists:filter(fun is_integer/1, [1, "a", 2, :b, 3.0]).
%% = [1, 2]

%% filtermap: filter + map in one pass
lists:filtermap(
    fun(X) when X > 3 -> {true, X * 10};
       (_) -> false
    end,
    [1,2,3,4,5]
).
%% = [40, 50]

%% foldl: left fold (accumulate from left)
lists:foldl(fun(X, Acc) -> Acc + X end, 0, [1,2,3,4,5]).
%% = 15

lists:foldl(fun(X, Acc) -> [X|Acc] end, [], [1,2,3]).
%% = [3,2,1] (reversed!)

%% foldr: right fold (accumulate from right)
lists:foldr(fun(X, Acc) -> [X|Acc] end, [], [1,2,3]).
%% = [1,2,3] (same order)

lists:foldr(fun(X, Acc) -> X - Acc end, 0, [1,2,3,4]).
%% = 1-(2-(3-(4-0))) = -2

%% foreach: side effects (returns ok)
lists:foreach(fun(X) -> io:format("~p~n", [X]) end, [1,2,3]).

%% ตัวอย่างซับซ้อน: Group and aggregate
group_and_sum(Pairs) ->
    lists:foldl(
        fun({K, V}, Acc) ->
            maps:update_with(K, fun(S) -> S + V end, V, Acc)
        end,
        #{},
        Pairs
    ).

group_and_sum([{a,1}, {b,2}, {a,3}, {c,4}, {b,5}]).
%% = #{a => 4, b => 7, c => 4}

%% Chain operations
[1,2,3,4,5,6,7,8,9,10]
|> lists:filter(fun(X) -> X rem 2 =:= 0 end)  %% [2,4,6,8,10]
|> lists:map(fun(X) -> X * X end)             %% [4,16,36,64,100]
|> lists:sum.                                  %% = 220
%% ไม่มี |> ใน Erlang จริงๆ แต่ concept คือ chaining

%% ต้องเขียนแบบนี้
Sum = lists:sum(
    lists:map(
        fun(X) -> X * X end,
        lists:filter(
            fun(X) -> X rem 2 =:= 0 end,
            lists:seq(1, 10)
        )
    )
).
%% = 220
```

---

## Function Composition

```erlang
%% compose: f(g(x))
compose(F, G) ->
    fun(X) -> F(G(X)) end.

DoubleThSquare = compose(fun(X) -> X*X end, fun(X) -> X*2 end).
DoubleThSquare(3).   %% = (3*2)^2 = 36

%% Compose list of functions
compose_all([]) ->
    fun(X) -> X end;
compose_all([F]) ->
    F;
compose_all([F|Rest]) ->
    compose(F, compose_all(Rest)).

Pipeline = compose_all([
    fun(X) -> X + 1 end,    %% applied last
    fun(X) -> X * 2 end,    %% applied second
    fun(X) -> X - 3 end     %% applied first
]).
Pipeline(10).  %% (10-3)*2+1 = 15

%% Pipe: apply functions left-to-right
pipe(X, []) -> X;
pipe(X, [F|Rest]) -> pipe(F(X), Rest).

pipe(10, [
    fun(X) -> X - 3 end,    %% 7
    fun(X) -> X * 2 end,    %% 14
    fun(X) -> X + 1 end     %% 15
]).
%% = 15

%% ตัวอย่าง: Data transformation pipeline
process_user_data(RawData) ->
    pipe(RawData, [
        fun normalize_keys/1,
        fun validate_required_fields/1,
        fun sanitize_strings/1,
        fun enrich_with_defaults/1
    ]).

%% Function memoization
memoize(F) ->
    Cache = ets:new(memo_cache, [set, private]),
    fun(X) ->
        case ets:lookup(Cache, X) of
            [{_, Result}] -> Result;
            [] ->
                Result = F(X),
                ets:insert(Cache, {X, Result}),
                Result
        end
    end.

SlowFib = fun Fib(0) -> 0;
              Fib(1) -> 1;
              Fib(N) -> Fib(N-1) + Fib(N-2)
          end.

FastFib = memoize(SlowFib).
FastFib(30).  %% ช้าครั้งแรก แต่ cached หลังจากนั้น
```

---

## Currying and Partial Application

```erlang
%% Partial application: fix some arguments of a function

%% Manual partial application
add(X, Y) -> X + Y.

add5 = fun(Y) -> add(5, Y) end.
add5(3).    %% = 8
add5(10).   %% = 15

%% Utility function: partial
partial(F, X) ->
    fun(Y) -> F(X, Y) end.

partial(F, X, Y) ->
    fun(Z) -> F(X, Y, Z) end.

Mult = fun(X, Y) -> X * Y end.
Double = partial(Mult, 2).
Triple = partial(Mult, 3).

Double(5).  %% = 10
Triple(5).  %% = 15

%% ตัวอย่าง: Build validators from partial application
greater_than(Min, Value) ->
    Value > Min.

less_than(Max, Value) ->
    Value < Max.

in_range(Min, Max, Value) ->
    Value >= Min andalso Value =< Max.

ValidAge = partial(fun in_range/3, 0, 150).
ValidScore = partial(fun in_range/3, 0, 100).

ValidAge(25).    %% = true
ValidAge(-1).    %% = false
ValidScore(95).  %% = true
ValidScore(101). %% = false

%% Curry: transform multi-arg function into chain of single-arg functions
curry(F) when is_function(F, 2) ->
    fun(X) -> fun(Y) -> F(X, Y) end end;
curry(F) when is_function(F, 3) ->
    fun(X) -> fun(Y) -> fun(Z) -> F(X, Y, Z) end end end.

CurriedAdd = curry(fun(X, Y) -> X + Y end).
Add3 = CurriedAdd(3).
Add3(4).   %% = 7
Add3(10).  %% = 13

%% ตัวอย่าง: Curried filter
CurriedFilter = curry(fun lists:filter/2).
OddFilter = CurriedFilter(fun(X) -> X rem 2 =/= 0 end).
OddFilter([1,2,3,4,5]).  %% = [1,3,5]
```

---

## ตัวอย่างจริง: Event System

```erlang
%% ไฟล์: event_bus.erl
-module(event_bus).
-export([
    new/0,
    subscribe/3,
    unsubscribe/2,
    publish/3,
    publish_async/3
]).

-type handler() :: fun((term()) -> term()).
-type subscription() :: #{
    id := reference(),
    event := atom(),
    handler := handler(),
    filter := fun((term()) -> boolean()) | undefined,
    once := boolean()
}.

-type bus() :: #{
    atom() => [subscription()]
}.

%% Create new event bus
-spec new() -> bus().
new() -> #{}.

%% Subscribe to event
-spec subscribe(bus(), atom(), handler()) -> {bus(), reference()}.
subscribe(Bus, Event, Handler) ->
    subscribe(Bus, Event, Handler, #{}).

subscribe(Bus, Event, Handler, Opts) ->
    SubId = make_ref(),
    Sub = #{
        id => SubId,
        event => Event,
        handler => Handler,
        filter => maps:get(filter, Opts, undefined),
        once => maps:get(once, Opts, false)
    },
    Subs = maps:get(Event, Bus, []),
    {Bus#{Event => [Sub | Subs]}, SubId}.

%% Unsubscribe
-spec unsubscribe(bus(), reference()) -> bus().
unsubscribe(Bus, SubId) ->
    maps:map(
        fun(_Event, Subs) ->
            lists:filter(fun(#{id := Id}) -> Id =/= SubId end, Subs)
        end,
        Bus
    ).

%% Publish event synchronously
-spec publish(bus(), atom(), term()) -> {bus(), [term()]}.
publish(Bus, Event, Data) ->
    Subs = maps:get(Event, Bus, []),
    {Results, NewSubs} = lists:foldl(
        fun(Sub, {Rs, Ss}) ->
            #{handler := Handler, filter := Filter, once := Once} = Sub,
            ShouldRun = case Filter of
                undefined -> true;
                F -> F(Data)
            end,
            case ShouldRun of
                true ->
                    Result = try Handler(Data)
                             catch _:E -> {error, E}
                             end,
                    NewSs = case Once of
                        true -> Ss;  %% don't re-add if once
                        false -> [Sub | Ss]
                    end,
                    {[Result | Rs], NewSs};
                false ->
                    {Rs, [Sub | Ss]}
            end
        end,
        {[], []},
        Subs
    ),
    NewBus = Bus#{Event => lists:reverse(NewSubs)},
    {NewBus, lists:reverse(Results)}.

%% Publish event asynchronously
-spec publish_async(bus(), atom(), term()) -> bus().
publish_async(Bus, Event, Data) ->
    Subs = maps:get(Event, Bus, []),
    lists:foreach(
        fun(#{handler := Handler, filter := Filter}) ->
            ShouldRun = case Filter of
                undefined -> true;
                F -> F(Data)
            end,
            case ShouldRun of
                true -> spawn(fun() -> Handler(Data) end);
                false -> ok
            end
        end,
        Subs
    ),
    Bus.
```

```erlang
%% ทดสอบ event_bus
Bus0 = event_bus:new().

%% Subscribe
{Bus1, Sub1} = event_bus:subscribe(Bus0, user_created,
    fun(User) -> io:format("New user: ~p~n", [User]) end).

{Bus2, Sub2} = event_bus:subscribe(Bus1, user_created,
    fun(#{email := Email}) ->
        send_welcome_email(Email)
    end).

%% Subscribe with filter (only admin users)
{Bus3, _Sub3} = event_bus:subscribe(Bus2, user_created,
    fun(User) -> notify_admins(User) end,
    #{filter => fun(#{role := Role}) -> Role =:= admin; (_) -> false end}).

%% Subscribe once
{Bus4, _Sub4} = event_bus:subscribe(Bus3, app_started,
    fun(_) -> io:format("App started!~n") end,
    #{once => true}).

%% Publish
{Bus5, Results} = event_bus:publish(Bus4, user_created,
    #{name => <<"Alice">>, email => <<"alice@...">>}).
%% Calls: Sub1 handler, Sub2 handler (Sub3 filtered out)

%% Unsubscribe
Bus6 = event_bus:unsubscribe(Bus5, Sub1).

%% Higher-order event transformer
transform_and_publish(Bus, Event, Data, Transform) ->
    event_bus:publish(Bus, Event, Transform(Data)).
```

---

## สรุป Part 16

| Concept | ตัวอย่าง | ใช้เมื่อ |
|---------|---------|---------|
| Anonymous fun | `fun(X) -> X*2 end` | One-off functions |
| Fun reference | `fun lists:sum/1` | Named functions |
| Closure | capture variables | Factory pattern |
| map | `lists:map(F, List)` | Transform each element |
| filter | `lists:filter(P, List)` | Select elements |
| fold | `lists:foldl(F, I, List)` | Accumulate/reduce |
| compose | `compose(F, G)` | Function pipelines |
| partial | `partial(F, X)` | Reusable functions |

---

*[← Part 15: Recursion](part_15_recursion.md) | [Part 17: List Comprehensions →](part_17_comprehensions.md)*
