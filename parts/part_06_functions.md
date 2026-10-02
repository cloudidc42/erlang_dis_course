# Part 6: Functions และ Function Clauses

## สารบัญ
1. [Function Basics](#function-basics)
2. [Function Clauses](#function-clauses)
3. [Guards ใน Functions](#guards-ใน-functions)
4. [Recursive Functions](#recursive-functions)
5. [Tail Recursion](#tail-recursion)
6. [Higher-Order Functions](#higher-order-functions)
7. [Anonymous Functions (Funs)](#anonymous-functions-funs)
8. [Function References](#function-references)
9. [Closures](#closures)
10. [apply/2 และ apply/3](#apply2-และ-apply3)
11. [Type Specifications](#type-specifications)
12. [Function Best Practices](#function-best-practices)
13. [สรุป](#สรุป)

---

## Function Basics

```erlang
%% ไฟล์: functions_demo.erl
-module(functions_demo).
-export([add/2, greet/1, greet/0, factorial/1]).

%% Basic function
add(A, B) ->
    A + B.

%% Multiple clauses (different arity = different function)
greet() ->
    greet("World").

greet(Name) ->
    io:format("Hello, ~s!~n", [Name]).

%% Recursive function
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N - 1).
```

### Function Syntax

```erlang
% Format:
% FunctionName(Pattern1, Pattern2, ...) [when Guard] ->
%     Body.
%
% หลาย clauses แยกด้วย ;
% Clause สุดท้ายจบด้วย .

% ตัวอย่าง
max(A, B) when A >= B -> A;
max(_, B) -> B.

% Multiple expressions ใน body - แยกด้วย ,
% Expression สุดท้ายเป็นค่า return
process(X) ->
    Doubled = X * 2,
    Tripled = X * 3,
    {Doubled, Tripled}.  % return tuple
```

### Return Values

```erlang
% ทุก function return ค่าของ expression สุดท้าย
always_returns() ->
    ok.    % return ok

add(A, B) ->
    A + B.  % return ผลการบวก

get_info(Name, Age) ->
    {Name, Age, "info"}.  % return tuple

% ไม่มี explicit return keyword
% Expression สุดท้ายเสมอ
```

---

## Function Clauses

Function ใน Erlang สามารถมีหลาย clauses ที่ match ตาม pattern

```erlang
%% function ที่มีหลาย clauses
day_type(1) -> weekend;
day_type(7) -> weekend;
day_type(Day) when Day >= 2, Day =< 6 -> weekday;
day_type(_) -> {error, invalid_day}.

day_type(1).  % = weekend
day_type(3).  % = weekday
day_type(8).  % = {error, invalid_day}

%% Clauses กับ Complex Patterns
describe_list([]) ->
    "empty";
describe_list([_]) ->
    "single element";
describe_list([_, _]) ->
    "two elements";
describe_list([H|T]) ->
    "starts with " ++ term_to_string(H) ++
    ", has " ++ integer_to_list(length(T)) ++ " more elements".

term_to_string(X) when is_atom(X) -> atom_to_list(X);
term_to_string(X) when is_integer(X) -> integer_to_list(X);
term_to_string(X) when is_list(X) -> X;
term_to_string(_) -> "?".
```

### Clause Ordering

```erlang
% Clauses ถูกตรวจสอบจาก TOP ถึง BOTTOM
% ใส่ specific clauses ก่อน general clauses

% ผิด - general clause ก่อน!
wrong_order(X) -> general_case;
wrong_order(0) -> zero_case.   % ไม่มีทางถึงบรรทัดนี้!

% ถูก - specific ก่อน
correct_order(0) -> zero_case;      % specific
correct_order(X) when X > 0 -> positive;  % more specific
correct_order(_) -> other.          % general/catch-all

% ตัวอย่างจริง: HTTP status code
http_message(200) -> <<"OK">>;
http_message(201) -> <<"Created">>;
http_message(204) -> <<"No Content">>;
http_message(301) -> <<"Moved Permanently">>;
http_message(302) -> <<"Found">>;
http_message(400) -> <<"Bad Request">>;
http_message(401) -> <<"Unauthorized">>;
http_message(403) -> <<"Forbidden">>;
http_message(404) -> <<"Not Found">>;
http_message(409) -> <<"Conflict">>;
http_message(422) -> <<"Unprocessable Entity">>;
http_message(429) -> <<"Too Many Requests">>;
http_message(500) -> <<"Internal Server Error">>;
http_message(502) -> <<"Bad Gateway">>;
http_message(503) -> <<"Service Unavailable">>;
http_message(Code) when Code >= 100, Code < 200 -> <<"1xx Informational">>;
http_message(Code) when Code >= 200, Code < 300 -> <<"2xx Success">>;
http_message(Code) when Code >= 300, Code < 400 -> <<"3xx Redirect">>;
http_message(Code) when Code >= 400, Code < 500 -> <<"4xx Client Error">>;
http_message(Code) when Code >= 500, Code < 600 -> <<"5xx Server Error">>;
http_message(_) -> <<"Unknown">>.
```

---

## Guards ใน Functions

```erlang
% Guards เพิ่มเงื่อนไขนอกเหนือจาก Pattern

classify_number(0) ->
    zero;
classify_number(N) when N > 0, is_integer(N) ->
    positive_integer;
classify_number(N) when N < 0, is_integer(N) ->
    negative_integer;
classify_number(N) when is_float(N), N > 0 ->
    positive_float;
classify_number(N) when is_float(N), N < 0 ->
    negative_float.

% Guard ด้วย andalso และ orelse
safe_process(Data) when is_map(Data) andalso map_size(Data) > 0 ->
    process_map(Data);
safe_process(Data) when is_list(Data) andalso length(Data) > 0 ->
    process_list(Data);
safe_process(_) ->
    {error, empty_or_invalid}.

% Guard หลาย conditions (comma = AND)
validate_age(Age) when is_integer(Age), Age >= 0, Age =< 150 ->
    valid;
validate_age(_) ->
    invalid.

% Guard ด้วย semicolon (OR)
is_string(S) when is_list(S); is_binary(S) ->
    true;
is_string(_) ->
    false.
```

---

## Recursive Functions

Erlang ไม่มี loops! ใช้ Recursion แทน

```erlang
%% Basic Recursion

%% Sum of a list
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

%% Product of a list
product([]) -> 1;
product([H|T]) -> H * product(T).

%% Maximum in a list
max_in_list([X]) -> X;
max_in_list([H|T]) ->
    MaxRest = max_in_list(T),
    if H > MaxRest -> H;
       true -> MaxRest
    end.

%% Or more elegantly:
max_in_list2([X]) -> X;
max_in_list2([H|T]) ->
    erlang:max(H, max_in_list2(T)).

%% Count occurrences
count(_, []) -> 0;
count(X, [X|T]) -> 1 + count(X, T);
count(X, [_|T]) -> count(X, T).

count(3, [1, 3, 2, 3, 3, 4]).  % = 3

%% Deep map (apply function to all elements in nested list)
deep_map(Fun, []) -> [];
deep_map(Fun, [H|T]) when is_list(H) ->
    [deep_map(Fun, H) | deep_map(Fun, T)];
deep_map(Fun, [H|T]) ->
    [Fun(H) | deep_map(Fun, T)].

deep_map(fun(X) -> X * 2 end, [1, [2, 3], [4, [5, 6]]]).
% = [2, [4, 6], [8, [10, 12]]]
```

### Mutual Recursion

```erlang
%% สองฟังก์ชันที่เรียกกันและกัน

is_even(0) -> true;
is_even(N) when N > 0 -> is_odd(N - 1).

is_odd(0) -> false;
is_odd(N) when N > 0 -> is_even(N - 1).

is_even(4).  % = true
is_odd(7).   % = true

%% Tree traversal
%% Tree = nil | {node, Value, Left, Right}

tree_sum(nil) -> 0;
tree_sum({node, V, L, R}) ->
    V + tree_sum(L) + tree_sum(R).

tree_depth(nil) -> 0;
tree_depth({node, _, L, R}) ->
    1 + erlang:max(tree_depth(L), tree_depth(R)).

tree_to_list(nil) -> [];
tree_to_list({node, V, L, R}) ->
    tree_to_list(L) ++ [V] ++ tree_to_list(R).

Tree = {node, 5,
        {node, 3, {node, 1, nil, nil}, {node, 4, nil, nil}},
        {node, 8, {node, 7, nil, nil}, {node, 9, nil, nil}}}.

tree_sum(Tree).     % = 37
tree_depth(Tree).   % = 3
tree_to_list(Tree). % = [1, 3, 4, 5, 7, 8, 9]
```

---

## Tail Recursion

Tail Recursion สำคัญมากใน Erlang เพื่อ performance:

```erlang
%% Non-tail-recursive (Stack overflow สำหรับ large input!)
sum_bad([]) -> 0;
sum_bad([H|T]) -> H + sum_bad(T).  % ต้องรอผลจาก sum_bad(T) ก่อน
% Stack: sum_bad([1,2,3]) = 1 + sum_bad([2,3])
%                              = 1 + 2 + sum_bad([3])
%                                        = 1 + 2 + 3 + sum_bad([])
%                                                         = 1+2+3+0

%% Tail-recursive (ใช้ Stack คงที่!)
sum_good(List) -> sum_good(List, 0).
sum_good([], Acc) -> Acc;
sum_good([H|T], Acc) -> sum_good(T, Acc + H).  % tail call!
% Stack: sum_good([1,2,3], 0)
%      = sum_good([2,3], 1)
%      = sum_good([3], 3)
%      = sum_good([], 6)
%      = 6

%% Tail recursive reverse
reverse(List) -> reverse(List, []).
reverse([], Acc) -> Acc;
reverse([H|T], Acc) -> reverse(T, [H|Acc]).

%% Tail recursive flatten
flatten(List) -> flatten(List, []).
flatten([], Acc) -> lists:reverse(Acc);
flatten([[]], Acc) -> flatten([], Acc);
flatten([[]|T], Acc) -> flatten(T, Acc);
flatten([[H|T1]|T2], Acc) -> flatten([H|[T1|T2]], Acc);
flatten([H|T], Acc) -> flatten(T, [H|Acc]).

%% Tail recursive fibonacci (ประสิทธิภาพสูงมาก!)
fibonacci(N) -> fibonacci(N, 0, 1).
fibonacci(0, A, _) -> A;
fibonacci(N, A, B) -> fibonacci(N-1, B, A+B).

fibonacci(50).  % = 12586269025 (เร็วมาก!)
```

### Trampoline Pattern

สำหรับ mutual recursion ที่ต้องการ tail recursion:

```erlang
%% Trampoline - สำหรับกรณีที่ต้องการ continue ใน tail position

-type thunk() :: fun(() -> term() | thunk()).

trampoline(Fun) when is_function(Fun) ->
    trampoline(Fun());
trampoline(Value) ->
    Value.

%% Usage
count_down(0) ->
    done;
count_down(N) ->
    fun() -> count_down(N - 1) end.

trampoline(count_down(1000000)).  % = done (ไม่ stack overflow)
```

---

## Higher-Order Functions

Functions ที่รับ หรือ return functions อื่น:

```erlang
%% map - apply function ทุก element
my_map(_, []) -> [];
my_map(F, [H|T]) -> [F(H) | my_map(F, T)].

my_map(fun(X) -> X * 2 end, [1, 2, 3, 4, 5]).
% = [2, 4, 6, 8, 10]

%% filter - กรอง elements
my_filter(_, []) -> [];
my_filter(Pred, [H|T]) ->
    case Pred(H) of
        true -> [H | my_filter(Pred, T)];
        false -> my_filter(Pred, T)
    end.

my_filter(fun(X) -> X > 3 end, [1, 2, 3, 4, 5, 6]).
% = [4, 5, 6]

%% fold - reduce list to single value
my_foldl(_, Acc, []) -> Acc;
my_foldl(F, Acc, [H|T]) -> my_foldl(F, F(H, Acc), T).

my_foldl(fun(X, Sum) -> X + Sum end, 0, [1, 2, 3, 4, 5]).
% = 15

%% compose - สร้าง function ใหม่จาก 2 functions
compose(F, G) ->
    fun(X) -> F(G(X)) end.

Double = fun(X) -> X * 2 end.
Increment = fun(X) -> X + 1 end.

DoubleIncrement = compose(Double, Increment).
DoubleIncrement(5).  % = 12 (5+1=6, 6*2=12)

%% curry - แปลง function ที่รับหลาย args เป็น function chain
curry(F) ->
    fun(X) -> fun(Y) -> F(X, Y) end end.

Add = fun(A, B) -> A + B end.
CurriedAdd = curry(Add).
Add5 = CurriedAdd(5).
Add5(3).  % = 8
Add5(10). % = 15

%% pipe - chain functions
pipe(Value, Functions) ->
    lists:foldl(fun(F, Acc) -> F(Acc) end, Value, Functions).

pipe(5, [
    fun(X) -> X * 2 end,   % 10
    fun(X) -> X + 1 end,   % 11
    fun(X) -> X * X end    % 121
]).
% = 121
```

---

## Anonymous Functions (Funs)

```erlang
%% Basic fun
Square = fun(X) -> X * X end.
Square(5).  % = 25

%% Multi-line fun
ProcessList = fun(List) ->
    Filtered = lists:filter(fun(X) -> X > 0 end, List),
    lists:map(fun(X) -> X * 2 end, Filtered)
end.

ProcessList([-1, 2, -3, 4, -5]).  % = [4, 8]

%% Fun กับหลาย clauses
Describe = fun
    (0) -> "zero";
    (N) when N > 0 -> "positive";
    (_) -> "negative"
end.

Describe(0).   % = "zero"
Describe(5).   % = "positive"
Describe(-3).  % = "negative"

%% Recursive fun (ใช้ Named Fun)
Factorial = fun F(0) -> 1;
               F(N) when N > 0 -> N * F(N - 1)
           end.

Factorial(6).  % = 720

%% Fun เป็น argument
apply_n_times(_, X, 0) -> X;
apply_n_times(F, X, N) -> apply_n_times(F, F(X), N-1).

apply_n_times(fun(X) -> X * 2 end, 1, 5).  % = 32 (2^5)

%% Fun ใน Data Structure
Operations = #{
    add => fun(A, B) -> A + B end,
    multiply => fun(A, B) -> A * B end,
    power => fun(A, B) -> math:pow(A, B) end
}.

(maps:get(add, Operations))(3, 4).       % = 7
(maps:get(multiply, Operations))(3, 4).  % = 12
```

---

## Function References

```erlang
%% อ้างอิง function ที่มีชื่อ
%% Format: fun Module:Function/Arity  หรือ  fun Function/Arity

%% Local function reference
-module(demo).
-export([double/1, process/1]).

double(X) -> X * 2.

process(List) ->
    lists:map(fun double/1, List).  % local function ref

%% Module function reference
process2(List) ->
    lists:map(fun erlang:abs/1, List).  % erlang module

process3(List) ->
    lists:filter(fun is_integer/1, List).  % BIF

%% Dynamic function reference
call_function(Module, Function, Args) ->
    erlang:apply(Module, Function, Args).

call_function(lists, sort, [[3, 1, 2]]).  % = [1, 2, 3]

%% ตรวจสอบ fun arity
F = fun(X, Y) -> X + Y end.
erlang:fun_info(F, arity).  % = {arity, 2}

%% is_function
is_function(F).           % = true
is_function(F, 2).        % = true (arity 2)
is_function(F, 1).        % = false (arity 1 = false)
```

---

## Closures

```erlang
%% Closure = fun ที่ "จำ" ตัวแปรจาก scope ที่สร้าง

make_counter(Start) ->
    fun() ->
        % ดักจับ Start จาก outer scope
        Start  % แต่ ใน Erlang ตัวแปรเปลี่ยนไม่ได้!
    end.

Counter = make_counter(0).
Counter().  % = 0

%% Mutable counter ต้องใช้ process (ใน Part 21)

%% Useful closure: make_adder
make_adder(N) ->
    fun(X) -> X + N end.

Add10 = make_adder(10).
Add10(5).   % = 15
Add10(20).  % = 30

%% make_multiplier
make_multiplier(N) ->
    fun(X) -> X * N end.

Triple = make_multiplier(3).
Triple(7).  % = 21

%% Closure กับ complex context
make_validator(MinAge, MaxAge) ->
    fun(Age) when is_integer(Age), Age >= MinAge, Age =< MaxAge ->
        valid;
       (_) ->
        invalid
    end.

AdultValidator = make_validator(18, 65).
AdultValidator(25).  % = valid
AdultValidator(15).  % = invalid
AdultValidator(70).  % = invalid

%% Config-based processing
make_processor(Config) ->
    MaxRetries = maps:get(max_retries, Config, 3),
    Timeout = maps:get(timeout, Config, 5000),
    fun(Task) ->
        run_with_retry(Task, MaxRetries, Timeout)
    end.

run_with_retry(Task, 0, _Timeout) ->
    {error, max_retries_exceeded};
run_with_retry(Task, Retries, Timeout) ->
    case Task() of
        {ok, Result} -> {ok, Result};
        {error, _} -> run_with_retry(Task, Retries - 1, Timeout)
    end.
```

---

## apply/2 และ apply/3

```erlang
%% erlang:apply/3 - เรียก function แบบ dynamic
erlang:apply(lists, sort, [[3, 1, 2]]).  % = [1, 2, 3]
erlang:apply(math, sqrt, [16]).          % = 4.0

%% erlang:apply/2 - เรียก fun กับ args list
F = fun(X, Y) -> X + Y end.
erlang:apply(F, [3, 4]).  % = 7

%% ใช้กับ dynamic dispatch
dispatch(Module, Function, Args) ->
    erlang:apply(Module, Function, Args).

dispatch(io, format, ["Hello, ~s!~n", ["World"]]).
% = Hello, World!

%% รัน functions จาก config
run_middleware(Request, []) ->
    {ok, Request};
run_middleware(Request, [{Module, Function} | Rest]) ->
    case erlang:apply(Module, Function, [Request]) of
        {ok, NewRequest} -> run_middleware(NewRequest, Rest);
        {error, _} = Error -> Error
    end.
```

---

## Type Specifications

Erlang มี Type Specifications สำหรับ documentation และ dialyzer:

```erlang
-module(typed_functions).

%% Type definitions
-type name() :: binary().
-type age() :: non_neg_integer().
-type user() :: #{name := name(), age := age()}.
-type result(T) :: {ok, T} | {error, term()}.

%% Function specs
-spec create_user(name(), age()) -> result(user()).
create_user(Name, Age) when is_binary(Name), is_integer(Age), Age >= 0 ->
    {ok, #{name => Name, age => Age}};
create_user(_, _) ->
    {error, invalid_input}.

-spec get_name(user()) -> name().
get_name(#{name := Name}) -> Name.

-spec get_age(user()) -> age().
get_age(#{age := Age}) -> Age.

%% Complex specs
-spec process_list(list(T), fun((T) -> U)) -> list(U).
process_list(List, Fun) ->
    lists:map(Fun, List).

%% Optional args
-spec greet(string()) -> ok.
-spec greet() -> ok.
greet() -> greet("World").
greet(Name) -> io:format("Hello, ~s!~n", [Name]).
```

---

## Function Best Practices

```erlang
%% 1. ตั้งชื่อ Function ให้ชัดเจน
%% ไม่ดี
f(X) -> X * 2.
process(L) -> lists:filter(fun(X) -> X > 0 end, L).

%% ดี
double(X) -> X * 2.
filter_positive(Numbers) ->
    lists:filter(fun(N) -> N > 0 end, Numbers).

%% 2. Function ควรทำสิ่งเดียว (Single Responsibility)
%% ไม่ดี
process_and_save(Data) ->
    Processed = transform(Data),
    Validated = validate(Processed),
    save_to_db(Validated),
    send_email(Validated),
    log_result(Validated).

%% ดีกว่า
process(Data) ->
    with_result(transform(Data), fun(Processed) ->
        validate(Processed)
    end).

save_and_notify(Data) ->
    with_result(save_to_db(Data), fun(Saved) ->
        send_email(Saved),
        log_result(Saved),
        {ok, Saved}
    end).

%% 3. ใช้ Pattern Matching แทน if/case เมื่อทำได้
%% ไม่ดี
handle(Tuple) ->
    if
        element(1, Tuple) =:= ok -> do_ok(element(2, Tuple));
        element(1, Tuple) =:= error -> do_error(element(2, Tuple))
    end.

%% ดี
handle({ok, Value}) -> do_ok(Value);
handle({error, Reason}) -> do_error(Reason).

%% 4. ใช้ Guards อย่างเหมาะสม
%% ไม่ดี
process(X) ->
    if
        is_integer(X) andalso X > 0 -> positive;
        true -> other
    end.

%% ดี
process(X) when is_integer(X), X > 0 -> positive;
process(_) -> other.

%% 5. ใช้ Tail Recursion สำหรับ list processing
%% ไม่ดี (อาจ stack overflow สำหรับ list ใหญ่)
my_length([]) -> 0;
my_length([_|T]) -> 1 + my_length(T).

%% ดี
my_length(List) -> my_length(List, 0).
my_length([], Acc) -> Acc;
my_length([_|T], Acc) -> my_length(T, Acc + 1).
```

---

## ตัวอย่างครบวงจร: Calculator Module

```erlang
%% ไฟล์: calculator.erl
-module(calculator).
-export([
    eval/1,
    add/2, subtract/2, multiply/2, divide/2,
    power/2, sqrt/1, factorial/1
]).

-type number_result() :: {ok, number()} | {error, term()}.

%% Main eval function
-spec eval(term()) -> number_result().
eval({add, A, B}) ->
    {ok, A + B};
eval({subtract, A, B}) ->
    {ok, A - B};
eval({multiply, A, B}) ->
    {ok, A * B};
eval({divide, _, 0}) ->
    {error, division_by_zero};
eval({divide, A, B}) ->
    {ok, A / B};
eval({power, A, B}) ->
    {ok, math:pow(A, B)};
eval({sqrt, X}) when X >= 0 ->
    {ok, math:sqrt(X)};
eval({sqrt, _}) ->
    {error, negative_sqrt};
eval({factorial, N}) when is_integer(N), N >= 0 ->
    {ok, factorial(N)};
eval({compose, Op1, Op2}) ->
    case eval(Op1) of
        {ok, V1} ->
            case eval(Op2) of
                {ok, V2} -> {ok, {V1, V2}};
                Error -> Error
            end;
        Error ->
            Error
    end;
eval(N) when is_number(N) ->
    {ok, N};
eval(Other) ->
    {error, {unknown_operation, Other}}.

%% Individual functions
-spec add(number(), number()) -> number().
add(A, B) -> A + B.

-spec subtract(number(), number()) -> number().
subtract(A, B) -> A - B.

-spec multiply(number(), number()) -> number().
multiply(A, B) -> A * B.

-spec divide(number(), number()) -> number_result().
divide(_, 0) -> {error, division_by_zero};
divide(_, 0.0) -> {error, division_by_zero};
divide(A, B) -> {ok, A / B}.

-spec power(number(), number()) -> number().
power(A, B) -> math:pow(A, B).

-spec sqrt(number()) -> number_result().
sqrt(X) when X >= 0 -> {ok, math:sqrt(X)};
sqrt(_) -> {error, negative_sqrt}.

-spec factorial(non_neg_integer()) -> non_neg_integer().
factorial(N) -> factorial(N, 1).
factorial(0, Acc) -> Acc;
factorial(N, Acc) -> factorial(N - 1, N * Acc).
```

```erlang
% ทดสอบ
calculator:eval({add, 3, 4}).           % = {ok, 7}
calculator:eval({divide, 10, 0}).       % = {error, division_by_zero}
calculator:eval({factorial, 10}).       % = {ok, 3628800}
calculator:eval({sqrt, -4}).            % = {error, negative_sqrt}
calculator:eval({compose, {add, 1, 2}, {multiply, 3, 4}}).
% = {ok, {3, 12}}
```

---

## สรุป Part 6

| Feature | Description |
|---------|-------------|
| Function Clauses | หลาย patterns ต่อ function |
| Guards | `when` เพื่อเพิ่มเงื่อนไข |
| Recursion | แทน loops |
| Tail Recursion | ประสิทธิภาพดี ไม่ stack overflow |
| Higher-Order Functions | map, filter, fold |
| Anonymous Functions | `fun(...) -> ... end` |
| Closures | ดักจับ variables จาก scope ภายนอก |
| Type Specs | `-spec` สำหรับ documentation |

### ก้าวต่อไป

ใน **Part 7** เราจะเรียน Modules:
- Module Structure
- Export/Import
- Module Attributes
- Module Info
- OTP Behaviours overview

---

## แบบฝึกหัด Part 6

```erlang
%% แบบฝึกหัดที่ 1: Tail Recursive Functions
%% เขียน tail-recursive versions ของ:
%% - sum/1 - บวกทุก element ใน list
%% - product/1 - คูณทุก element
%% - max/1 - หาค่ามากที่สุด
%% - flatten/1 - ทำ nested list ให้แบน

%% แบบฝึกหัดที่ 2: Higher-Order Functions
%% เขียน:
%% - partition/2 - แบ่ง list เป็น 2 ส่วนตาม predicate
%%   partition(fun(X) -> X > 3 end, [1,2,3,4,5])
%%   = {[4,5], [1,2,3]}
%%
%% - take_while/2 - เอา elements จนกว่า predicate เป็น false
%%   take_while(fun(X) -> X < 4 end, [1,2,3,4,5])
%%   = [1,2,3]
%%
%% - drop_while/2 - ทิ้ง elements จนกว่า predicate เป็น false
%%   drop_while(fun(X) -> X < 4 end, [1,2,3,4,5])
%%   = [4,5]

%% แบบฝึกหัดที่ 3: Closures
%% เขียน make_rate_limiter/1 ที่:
%% - รับ MaxCalls per second
%% - return function ที่ตรวจว่าสามารถ call ได้หรือไม่
%% (hint: ใช้ process dictionary หรือ ETS - เรียนใน Part 24)
```

---

*[← Part 5: Variables and Pattern Matching](part_05_variables_pattern_matching.md) | [Part 7: Modules →](part_07_modules.md)*
