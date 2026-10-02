# Part 14: Guards — การตรวจสอบเงื่อนไขขั้นสูง

## สารบัญ
1. [Guards Overview](#guards-overview)
2. [Guard Expressions ที่ใช้ได้](#guard-expressions-ที่ใช้ได้)
3. [Guards ใน Functions](#guards-ใน-functions)
4. [Guards ใน case/if](#guards-ใน-caseif)
5. [Compound Guards](#compound-guards)
6. [Type Testing Guards](#type-testing-guards)
7. [Custom Guard Macros](#custom-guard-macros)
8. [ตัวอย่างจริง: Validation Library](#ตัวอย่างจริง-validation-library)

---

## Guards Overview

Guards เป็นเงื่อนไขพิเศษที่ใช้เพิ่มเติมจาก Pattern Matching:

```erlang
%% Guards ต้อง:
%% 1. ไม่มี side effects
%% 2. ทำงานในเวลา O(1) เสมอ
%% 3. เป็น BIF หรือ guard functions เท่านั้น
%% 4. ถ้า guard throw exception -> guard ล้มเหลว (ไม่ crash)

%% ตัวอย่าง
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N - 1).
%% "when N > 0" = guard

%% Guard ใน function clause
head([H|_]) when is_integer(H), H > 0 -> H.

%% Guard ใน case
case N of
    N when N > 100 -> big;
    N when N > 0   -> small;
    _              -> non_positive
end.

%% Guard ใน if
if
    is_integer(X), X > 0 -> positive_integer;
    is_float(X)          -> float;
    true                 -> other
end.
```

---

## Guard Expressions ที่ใช้ได้

### Type Testing

```erlang
is_atom(X)         %% atom
is_binary(X)       %% binary
is_bitstring(X)    %% bitstring (binary ที่ไม่จำเป็นต้องเป็น bytes)
is_boolean(X)      %% true หรือ false
is_float(X)        %% float
is_function(X)     %% any fun
is_function(X, N)  %% fun กับ N arity
is_integer(X)      %% integer
is_list(X)         %% list
is_map(X)          %% map
is_number(X)       %% integer หรือ float
is_pid(X)          %% process id
is_port(X)         %% port
is_record(X, Tag)  %% record
is_record(X, Tag, N) %% record กับ N fields
is_reference(X)    %% reference
is_tuple(X)        %% tuple

%% ตัวอย่าง
-spec add(X, Y) -> number() when
    X :: number(),
    Y :: number().
add(X, Y) when is_number(X), is_number(Y) ->
    X + Y.
```

### Comparison Operators

```erlang
X =:= Y    %% identical equality (type-safe)
X =/= Y    %% not identical
X == Y     %% equal (with type coercion: 1 == 1.0)
X /= Y     %% not equal
X < Y      %% less than
X =< Y     %% less than or equal
X > Y      %% greater than
X >= Y     %% greater than or equal

%% ความแตกต่าง =:= vs ==
1 =:= 1.0   %% = false (different types)
1 == 1.0    %% = true (coerced)

1 =/= 1.0   %% = true
1 /= 1.0    %% = false
```

### Arithmetic in Guards

```erlang
%% สามารถใช้ arithmetic ใน guards
double_check(X) when X * 2 > 100 -> big;
double_check(_) -> small.

%% Abs, round, trunc, float in guards
abs(X) > 5
round(X) =:= 5
trunc(X) > 0
float(X) > 1.5

%% String length
length(List) > 5
byte_size(Binary) < 100
tuple_size(Tuple) =:= 3
map_size(Map) > 0
```

### Other Guard Functions

```erlang
%% Size functions
size(X)          %% tuple_size or byte_size (deprecated, avoid)
tuple_size(X)    %% size of tuple
byte_size(X)     %% size of binary in bytes
bit_size(X)      %% size of bitstring in bits
map_size(X)      %% number of keys in map
length(X)        %% length of list

%% Access functions
element(N, Tuple)  %% Nth element of tuple (1-indexed)
hd(List)           %% head of list
tl(List)           %% tail of list

%% Node
node()        %% current node name
node(Pid)     %% node of a pid

%% self
self()        %% current process pid

%% Other
is_map_key(Key, Map)   %% OTP 21+
```

---

## Guards ใน Functions

```erlang
%% Single guard
positive(N) when N > 0 -> true;
positive(_) -> false.

%% Guard กับ type check
safe_div(A, B) when is_number(A), is_number(B), B =/= 0 ->
    A / B;
safe_div(_, 0) ->
    {error, division_by_zero};
safe_div(A, B) when not (is_number(A) and is_number(B)) ->
    {error, not_numbers}.

%% Multiple guards (AND = ,  OR = ;)
validate_age(Age) when is_integer(Age), Age >= 0, Age =< 150 ->
    {ok, Age};
validate_age(_) ->
    {error, invalid_age}.

%% Guard กับ string validation
valid_name(Name) when is_binary(Name),
                      byte_size(Name) >= 2,
                      byte_size(Name) =< 100 ->
    true;
valid_name(_) ->
    false.

%% Guard กับ tuple patterns
add_vector({X1, Y1}, {X2, Y2})
    when is_number(X1), is_number(Y1),
         is_number(X2), is_number(Y2) ->
    {X1 + X2, Y1 + Y2}.

%% Guard ใน recursion
sum_positive([]) -> 0;
sum_positive([H|T]) when H > 0 -> H + sum_positive(T);
sum_positive([_|T]) -> sum_positive(T).  %% skip non-positive

sum_positive([1, -2, 3, -4, 5]).  %% = 9
```

---

## Guards ใน case/if

```erlang
%% Guards ใน case
classify_value(V) ->
    case V of
        V when is_integer(V), V > 0   -> positive_integer;
        V when is_integer(V), V < 0   -> negative_integer;
        V when is_integer(V)          -> zero;
        V when is_float(V), V > 0.0   -> positive_float;
        V when is_float(V)            -> non_positive_float;
        V when is_binary(V)           -> binary;
        V when is_list(V)             -> list;
        V when is_map(V)              -> map;
        V when is_atom(V)             -> atom;
        _                             -> other
    end.

%% Guards ใน if
check_config(Config) ->
    if
        is_map(Config), map_size(Config) > 0 ->
            {ok, Config};
        is_map(Config) ->
            {error, empty_config};
        true ->
            {error, not_a_map}
    end.

%% Guards กับ complex patterns
route(Method, Path, Body)
    when is_atom(Method),
         is_binary(Path),
         byte_size(Path) > 0,
         (is_binary(Body) orelse Body =:= undefined) ->
    do_route(Method, Path, Body);
route(_, _, _) ->
    {error, invalid_request}.
```

---

## Compound Guards

```erlang
%% AND = , (comma)
f(X) when is_integer(X), X > 0, X < 100 -> ok.
%% ทุก condition ต้องเป็น true

%% OR = ; (semicolon)
is_string(X) when is_list(X); is_binary(X) -> true;
is_string(_) -> false.
%% อย่างน้อยหนึ่ง condition ต้องเป็น true

%% Guard sequences: semicolon แยก sequences, comma ใน sequence
%% each sequence is AND, sequences ต่อกันด้วย OR
f(X) when X > 0, X < 10;   %% sequence 1: AND
          X > 100, X < 200  %% sequence 2: AND
-> ok.
%% = (X > 0 AND X < 10) OR (X > 100 AND X < 200)

%% 'not' ใน guards
not_zero(X) when X =/= 0 -> true;
not_zero(_) -> false.

even(X) when X rem 2 =:= 0 -> true;
even(_) -> false.

odd(X) when not (X rem 2 =:= 0) -> true;
odd(_) -> false.

%% 'and' / 'or' / 'not' operators (ไม่ short-circuit!)
f(X) when (X > 0) and (X < 100) -> ok.  %% เหมือน ,
g(X) when (X < 0) or (X > 100) -> ok.   %% เหมือน ;

%% 'andalso' / 'orelse' (short-circuit, OTP 17+)
h(X) when is_integer(X) andalso X > 0 -> ok.
%% ถ้า is_integer(X) = false -> X > 0 ไม่ถูก evaluate

%% คำแนะนำ: ใช้ andalso/orelse ใน guards
safe_check(X) when is_integer(X) andalso X > 0 andalso X < 100 ->
    {ok, X};
safe_check(X) when is_float(X) orelse is_integer(X) ->
    {number, X};
safe_check(_) ->
    {error, invalid}.
```

---

## Type Testing Guards

```erlang
%% ตัวอย่าง type dispatch
process(X) when is_integer(X) ->
    {integer, X};
process(X) when is_float(X) ->
    {float, X};
process(X) when is_binary(X) ->
    {binary, X};
process(X) when is_list(X) ->
    {list, X};
process(X) when is_map(X) ->
    {map, X};
process(X) when is_atom(X) ->
    {atom, X};
process(X) when is_tuple(X) ->
    {tuple, X};
process(X) when is_pid(X) ->
    {pid, X};
process(X) when is_function(X) ->
    {function, X};
process(_) ->
    {unknown}.

%% is_function กับ arity
apply_if_valid(F, Args)
    when is_function(F, length(Args)) ->
    apply(F, Args);
apply_if_valid(F, Args)
    when is_function(F) ->
    {error, wrong_arity};
apply_if_valid(_, _) ->
    {error, not_a_function}.

%% Record guards
-record(point, {x, y}).
-record(circle, {center, radius}).

area(#circle{radius = R}) when is_number(R), R > 0 ->
    math:pi() * R * R;
area(Shape) when is_record(Shape, circle) ->
    {error, invalid_radius}.

%% is_record/2 example
is_valid_point(P) when is_record(P, point) ->
    is_number(P#point.x) andalso is_number(P#point.y);
is_valid_point(_) ->
    false.
```

---

## Custom Guard Macros

```erlang
%% สร้าง guard macros ด้วย -define

%% Type guards
-define(is_non_neg_integer(X), (is_integer(X) andalso X >= 0)).
-define(is_pos_integer(X), (is_integer(X) andalso X > 0)).
-define(is_string(X), (is_list(X) orelse is_binary(X))).
-define(is_json(X), (is_map(X) orelse is_list(X))).

%% Range guards
-define(in_range(X, Low, High), (X >= Low andalso X =< High)).
-define(in_range_exclusive(X, Low, High), (X > Low andalso X < High)).

%% HTTP status guards
-define(is_2xx(Code), (?in_range(Code, 200, 299))).
-define(is_4xx(Code), (?in_range(Code, 400, 499))).
-define(is_5xx(Code), (?in_range(Code, 500, 599))).

%% Usage
validate_port(Port) when ?is_pos_integer(Port), ?in_range(Port, 1, 65535) ->
    {ok, Port};
validate_port(_) ->
    {error, invalid_port}.

handle_status(Code) when ?is_2xx(Code) -> success;
handle_status(Code) when ?is_4xx(Code) -> client_error;
handle_status(Code) when ?is_5xx(Code) -> server_error;
handle_status(_) -> unknown.

%% Guard macros สำหรับ complex types
-define(is_user(X), (is_map(X) andalso is_map_key(name, X) andalso is_map_key(email, X))).
-define(is_email(X), (is_binary(X) andalso byte_size(X) > 3)).

create_account(User) when ?is_user(User) ->
    #{email := Email} = User,
    case ?is_email(Email) of
        true -> {ok, User};
        false -> {error, invalid_email}
    end;
create_account(_) ->
    {error, invalid_user}.
```

---

## ตัวอย่างจริง: Validation Library

```erlang
%% ไฟล์: validator.erl
-module(validator).
-export([
    validate/2,
    required/1,
    type/2,
    range/3,
    length/3,
    pattern/2,
    one_of/2,
    custom/2
]).

-type value() :: term().
-type rule() :: fun((value()) -> ok | {error, binary()}).
-type schema() :: #{atom() => [rule()]}.

%% Validate a map against a schema
-spec validate(map(), schema()) -> {ok, map()} | {error, map()}.
validate(Data, Schema) ->
    Errors = maps:fold(
        fun(Field, Rules, Acc) ->
            Value = maps:get(Field, Data, undefined),
            FieldErrors = run_rules(Value, Rules, Field),
            case FieldErrors of
                [] -> Acc;
                Errs -> Acc#{Field => Errs}
            end
        end,
        #{},
        Schema
    ),
    case map_size(Errors) of
        0 -> {ok, Data};
        _ -> {error, Errors}
    end.

run_rules(Value, Rules, _Field) ->
    lists:filtermap(
        fun(Rule) ->
            case Rule(Value) of
                ok -> false;
                {error, Msg} -> {true, Msg}
            end
        end,
        Rules
    ).

%% Rule builders

%% Required field
-spec required(term()) -> rule().
required(undefined) -> {error, <<"is required">>};
required(<<>>) -> {error, <<"cannot be empty">>};
required([]) -> {error, <<"cannot be empty">>};
required(_) -> ok.

required() -> fun required/1.

%% Type check
-spec type(atom(), value()) -> ok | {error, binary()}.
type(integer, V) when is_integer(V) -> ok;
type(float, V) when is_float(V) -> ok;
type(number, V) when is_number(V) -> ok;
type(binary, V) when is_binary(V) -> ok;
type(string, V) when is_binary(V) -> ok;
type(atom, V) when is_atom(V) -> ok;
type(list, V) when is_list(V) -> ok;
type(map, V) when is_map(V) -> ok;
type(boolean, V) when is_boolean(V) -> ok;
type(Type, _) ->
    {error, iolist_to_binary(["must be of type ", atom_to_list(Type)])}.

type(Type) -> fun(V) -> type(Type, V) end.

%% Range check
-spec range(number(), number(), value()) -> ok | {error, binary()}.
range(Min, Max, V) when is_number(V), V >= Min, V =< Max -> ok;
range(Min, Max, _) ->
    {error, iolist_to_binary(
        io_lib:format("must be between ~p and ~p", [Min, Max])
    )}.

range(Min, Max) -> fun(V) -> range(Min, Max, V) end.

%% Length check
-spec length(pos_integer(), pos_integer(), value()) -> ok | {error, binary()}.
length(Min, Max, V) when is_binary(V) ->
    Len = byte_size(V),
    check_length(Len, Min, Max);
length(Min, Max, V) when is_list(V) ->
    Len = erlang:length(V),
    check_length(Len, Min, Max);
length(_, _, _) ->
    {error, <<"must have a length">>}.

check_length(Len, Min, _Max) when Len < Min ->
    {error, iolist_to_binary(io_lib:format("must be at least ~p characters", [Min]))};
check_length(Len, _Min, Max) when Len > Max ->
    {error, iolist_to_binary(io_lib:format("must be at most ~p characters", [Max]))};
check_length(_, _, _) ->
    ok.

length(Min, Max) -> fun(V) -> ?MODULE:length(Min, Max, V) end.

%% Pattern check
-spec pattern(binary(), value()) -> ok | {error, binary()}.
pattern(Regex, V) when is_binary(V) ->
    case re:run(V, Regex) of
        {match, _} -> ok;
        nomatch -> {error, iolist_to_binary(["must match pattern: ", Regex])}
    end;
pattern(_, _) ->
    {error, <<"must be a string">>}.

pattern(Regex) -> fun(V) -> pattern(Regex, V) end.

%% Enum check
-spec one_of([term()], value()) -> ok | {error, binary()}.
one_of(Choices, V) ->
    case lists:member(V, Choices) of
        true -> ok;
        false ->
            ChoiceStrs = [io_lib:format("~p", [C]) || C <- Choices],
            {error, iolist_to_binary(
                ["must be one of: ", lists:join(", ", ChoiceStrs)]
            )}
    end.

one_of(Choices) -> fun(V) -> one_of(Choices, V) end.

%% Custom validator
-spec custom(fun(), value()) -> ok | {error, binary()}.
custom(Fun, V) ->
    case Fun(V) of
        true  -> ok;
        false -> {error, <<"failed custom validation">>};
        ok    -> ok;
        {error, _} = E -> E
    end.

custom(Fun) -> fun(V) -> custom(Fun, V) end.
```

```erlang
%% ทดสอบ validator
Schema = #{
    name  => [validator:required(),
              validator:type(binary),
              validator:length(2, 100)],
    age   => [validator:required(),
              validator:type(integer),
              validator:range(0, 150)],
    email => [validator:required(),
              validator:type(binary),
              validator:pattern(<<"^[^@]+@[^@]+\\.[^@]+$">>)],
    role  => [validator:one_of([admin, user, moderator])]
}.

validator:validate(
    #{name => <<"Alice">>, age => 30, email => <<"alice@example.com">>, role => user},
    Schema
).
%% = {ok, #{name => <<"Alice">>, age => 30, ...}}

validator:validate(
    #{name => <<"A">>, age => -1, email => <<"bad-email">>},
    Schema
).
%% = {error, #{
%%     name => [<<"must be at least 2 characters">>],
%%     age => [<<"must be between 0 and 150">>],
%%     email => [<<"must match pattern: ...">>],
%%     role => [<<"is required">>]
%% }}
```

---

## สรุป Part 14

| Guard Type | ตัวอย่าง | หมายเหตุ |
|-----------|---------|---------|
| Type check | `is_integer(X)` | ปลอดภัยเสมอ |
| Comparison | `X > 0, X < 100` | , = AND |
| OR | `is_list(X); is_binary(X)` | ; = OR |
| Short-circuit | `is_integer(X) andalso X > 0` | ดีกว่า , |
| NOT | `not is_integer(X)` | หรือ `=/=` |
| Arithmetic | `abs(X) > 5` | ได้ |
| Size | `length(L) > 3` | ได้ |
| Custom macro | `-define(is_pos(X), ...)` | สะดวก |

**กฎสำคัญ**: Guards ต้องไม่มี side effects และ throw exception ใน guard = guard fails (ไม่ crash)

---

*[← Part 13: Control Flow](part_13_control_flow.md) | [Part 15: Recursion →](part_15_recursion.md)*
