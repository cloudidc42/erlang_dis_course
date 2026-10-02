# Part 5: Variables และ Pattern Matching

## สารบัญ
1. [Variables ใน Erlang](#variables-ใน-erlang)
2. [Single Assignment (Immutability)](#single-assignment-immutability)
3. [Pattern Matching คืออะไร?](#pattern-matching-คืออะไร)
4. [Pattern Matching กับ Atoms](#pattern-matching-กับ-atoms)
5. [Pattern Matching กับ Tuples](#pattern-matching-กับ-tuples)
6. [Pattern Matching กับ Lists](#pattern-matching-กับ-lists)
7. [Pattern Matching กับ Maps](#pattern-matching-กับ-maps)
8. [Pattern Matching กับ Binaries](#pattern-matching-กับ-binaries)
9. [Pattern Matching ใน Function Clauses](#pattern-matching-ใน-function-clauses)
10. [Guards](#guards)
11. [Destructuring](#destructuring)
12. [Nested Patterns](#nested-patterns)
13. [Pattern Matching กับ case](#pattern-matching-กับ-case)
14. [Common Patterns](#common-patterns)
15. [สรุป](#สรุป)

---

## Variables ใน Erlang

### กฎการตั้งชื่อ Variable

```erlang
% ถูกต้อง - ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่หรือ _
Name = "Alice".
Age = 30.
FirstName = "Bob".
_Unused = "ignored".
X = 42.
ABC = atom.

% ผิด - ขึ้นต้นด้วยตัวพิมพ์เล็ก (จะถือเป็น Atom!)
% name = "Alice".  % นี่คือ match pattern ไม่ใช่ assignment!
% age = 30.        % ผิด
```

### Variable Scope

```erlang
% Variables ใน Erlang มี function scope
calculate(X, Y) ->
    Sum = X + Y,     % Sum เข้าถึงได้ตลอด function
    Diff = X - Y,    % Diff เข้าถึงได้ตลอด function
    {Sum, Diff}.     % return ทั้งคู่

% Variables ใน case, if, receive ถูก export ออกมา
process(Value) ->
    Result = case Value of
        positive when Value > 0 ->
            Msg = "positive",  % Msg ถูก bind ที่นี่
            {ok, Msg};         % และเข้าถึงได้ที่นี่
        _ ->
            {error, "not positive"}
    end,
    Result.  % Result เข้าถึงได้ที่นี่
```

---

## Single Assignment (Immutability)

Erlang ใช้ Single Assignment: ตัวแปรสามารถ bind ได้เพียงครั้งเดียว

```erlang
% ครั้งแรก - binding
X = 5.     % X ถูก bind กับ 5
Y = 10.    % Y ถูก bind กับ 10

% พยายาม rebind - Error!
% X = 6.  % ** exception error: no match of right hand side value 6

% แต่สามารถ match กับค่าเดิมได้
X = 5.     % OK! - เป็นการ verify ว่า X = 5

% การใช้ตัวแปรใหม่แทน
X1 = X + 1.   % X1 = 6
X2 = X1 * 2.  % X2 = 12
% X ยังคงเป็น 5, X1 = 6, X2 = 12
```

### ทำไม Immutability ดี?

```erlang
% ใน Mutable Languages:
% x = 5
% do_something()  % อาจเปลี่ยนค่า x!
% print(x)  % ค่า x อาจเปลี่ยนแล้ว

% ใน Erlang:
X = 5.
do_something(X).  % ไม่สามารถเปลี่ยนค่า X ได้!
io:format("~p~n", [X]).  % X ยังคงเป็น 5 แน่ๆ

% ผลดี:
% 1. ไม่มี race conditions
% 2. Thread-safe โดยอัตโนมัติ
% 3. โค้ดอ่านง่ายขึ้น - รู้ว่า X จะเป็นค่าอะไรตลอด
% 4. Testing ง่ายขึ้น - ไม่มี hidden state
```

---

## Pattern Matching คืออะไร?

Pattern Matching ใน Erlang คือกลไกพื้นฐานที่ใช้ได้ทุกที่

```erlang
% Pattern Matching = ตรวจสอบว่า structure ตรงกัน และ bind ตัวแปร
% Format: Pattern = Expression

% ตัวอย่างง่ายๆ
5 = 5.          % OK - 5 matches 5
% 5 = 6.        % ERROR - 5 does not match 6

X = 5.          % X ถูก bind กับ 5
5 = X.          % OK - X (= 5) matches 5

{A, B} = {1, 2}.   % A = 1, B = 2
[H|T] = [1,2,3].   % H = 1, T = [2,3]
#{k := V} = #{k => 1}.  % V = 1
```

### = ไม่ใช่ Assignment!

```erlang
% = คือ Match Operator ไม่ใช่ Assignment!

% ใน Python/JavaScript:
% x = 5   (assignment: x ได้รับค่า 5)
% 5 = x   (syntax error!)

% ใน Erlang:
X = 5.   % binding: ถ้า X ยังไม่ bind -> bind X กับ 5
         %          ถ้า X bind แล้ว -> ตรวจสอบว่า X = 5 ไหม
5 = X.   % matching: ตรวจสอบว่า 5 = X (= 5) -> OK

% = ทำงาน 2 อย่าง:
% 1. ตรวจสอบว่า Pattern ตรงกับ Value
% 2. Bind ตัวแปรที่ยังไม่ bind ให้ตรงกับ Value ที่สอดคล้อง
```

---

## Pattern Matching กับ Atoms

```erlang
% Match exact atom
ok = ok.           % OK
% ok = error.      % ERROR: no match

% Bind atom to variable
Status = ok.
ok = Status.       % OK - verify ว่า Status เป็น ok

% ใช้ใน function
handle(ok) ->
    "success";
handle(error) ->
    "failure";
handle(Other) ->
    "unknown: " ++ atom_to_list(Other).

handle(ok).    % = "success"
handle(error). % = "failure"
handle(warn).  % = "unknown: warn"
```

---

## Pattern Matching กับ Tuples

```erlang
% Match structure
{1, 2, 3} = {1, 2, 3}.   % OK
% {1, 2, 3} = {1, 2, 4}. % ERROR

% Extract values
{X, Y} = {10, 20}.
% X = 10, Y = 20

% Match with atoms (tagged tuples)
{ok, Value} = {ok, 42}.
% Value = 42

% Nested tuple
{{A, B}, C} = {{1, 2}, 3}.
% A = 1, B = 2, C = 3

% Ignore values with _
{_, Second, _} = {a, b, c}.
% Second = b

% Partial match (ขนาดต้องตรงกัน!)
{First, _} = {hello, world}.
% First = hello

% ใช้ใน function
process({ok, Data}) ->
    {success, Data};
process({error, Reason}) ->
    {failure, Reason};
process({redirect, Url, StatusCode}) ->
    {redirect, Url, StatusCode}.

process({ok, 42}).           % = {success, 42}
process({error, not_found}). % = {failure, not_found}
```

---

## Pattern Matching กับ Lists

```erlang
% Match empty list
[] = [].   % OK
% [] = [1]. % ERROR

% Match non-empty list: Head | Tail
[H|T] = [1, 2, 3, 4, 5].
% H = 1
% T = [2, 3, 4, 5]

% Multiple elements
[A, B | Rest] = [1, 2, 3, 4, 5].
% A = 1
% B = 2
% Rest = [3, 4, 5]

[First, Second, Third] = [x, y, z].
% First = x, Second = y, Third = z

% ใช้ _ เพื่อ ignore
[_, Second | _] = [a, b, c, d, e].
% Second = b

% Deep pattern
[[H|_]|_] = [[1, 2], [3, 4]].
% H = 1

% ใน function - recursion pattern ที่พบบ่อย
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

sum([1, 2, 3, 4, 5]).  % = 15

% Pattern matching กับ string (list of chars)
% ระวัง: string เป็น list ดังนั้น match ได้
[First | _] = "Hello".
First.  % = 72 (char code ของ H)
```

---

## Pattern Matching กับ Maps

```erlang
% Map pattern matching ใช้ :=
#{key := Value} = #{key => 42}.
% Value = 42

% หลาย keys
#{name := Name, age := Age} = #{name => "Alice", age => 30}.
% Name = "Alice"
% Age = 30

% Map ไม่ต้อง match ทุก key!
#{a := A} = #{a => 1, b => 2, c => 3}.
% A = 1 (OK! ไม่ต้อง match b และ c)

% ใน function
greet(#{name := Name}) ->
    io:format("Hello, ~s!~n", [Name]).

update_age(#{age := Age} = Person) ->
    Person#{age := Age + 1}.

update_age(#{name => "Alice", age => 30}).
% = #{name => "Alice", age => 31}
```

---

## Pattern Matching กับ Binaries

```erlang
% Basic binary pattern
<<A, B, C>> = <<1, 2, 3>>.
% A = 1, B = 2, C = 3

% With rest
<<First, Rest/binary>> = <<"Hello">>.
% First = 72 (H)
% Rest = <<"ello">>

% Bit patterns
<<Int:32/big>> = <<0, 0, 1, 0>>.
% Int = 256

% String pattern
<<"Hello", Rest/binary>> = <<"Hello, World!">>.
% Rest = <<", World!">>

% Complex binary protocol
parse_packet(<<
    Type:8,
    Length:16/big,
    Data:Length/binary,
    Rest/binary
>>) ->
    {Type, Data, Rest}.

parse_packet(<<1, 0, 5, "hello", 99>>).
% = {1, <<"hello">>, <<99>>}

% UTF-8 string matching
<<FirstChar/utf8, _/binary>> = <<"Hello">>.
% FirstChar = 72 (H unicode codepoint)
```

---

## Pattern Matching ใน Function Clauses

Pattern Matching ที่ทรงพลังที่สุดคือใน function clauses:

```erlang
%% ไฟล์: patterns.erl
-module(patterns).
-export([describe/1, process/1, fibonacci/1, zip/2]).

%% Describe value - multiple clauses
describe(true) -> "boolean true";
describe(false) -> "boolean false";
describe(undefined) -> "undefined";
describe(nil) -> "nil";
describe([]) -> "empty list";
describe([_|_]) -> "non-empty list";
describe({}) -> "empty tuple";
describe({_}) -> "1-element tuple";
describe({_, _}) -> "2-element tuple";
describe(X) when is_integer(X) -> "integer: " ++ integer_to_list(X);
describe(X) when is_float(X) -> "float";
describe(X) when is_atom(X) -> "atom: " ++ atom_to_list(X);
describe(X) when is_binary(X) -> "binary";
describe(X) when is_map(X) -> "map with " ++ integer_to_list(maps:size(X)) ++ " keys";
describe(_) -> "unknown".

%% Process result tuples
process({ok, Value}) ->
    {success, Value};
process({ok, Value, Metadata}) ->
    {success, Value, Metadata};
process({error, not_found}) ->
    {failure, 404, "Not found"};
process({error, unauthorized}) ->
    {failure, 403, "Unauthorized"};
process({error, Reason}) when is_atom(Reason) ->
    {failure, 500, atom_to_list(Reason)};
process({error, Reason}) when is_binary(Reason) ->
    {failure, 500, Reason}.

%% Fibonacci - classic recursion with pattern matching
fibonacci(0) -> 0;
fibonacci(1) -> 1;
fibonacci(N) when N > 1 -> fibonacci(N-1) + fibonacci(N-2).

%% Zip two lists
zip([], _) -> [];
zip(_, []) -> [];
zip([H1|T1], [H2|T2]) -> [{H1, H2} | zip(T1, T2)].
```

```erlang
% ทดสอบ
describe(true).         % = "boolean true"
describe([]).           % = "empty list"
describe([1,2,3]).      % = "non-empty list"
describe(42).           % = "integer: 42"
describe(hello).        % = "atom: hello"

process({ok, 42}).          % = {success, 42}
process({error, not_found}).% = {failure, 404, "Not found"}

fibonacci(10).  % = 55
zip([1,2,3], [a,b,c]).  % = [{1,a}, {2,b}, {3,c}]
```

---

## Guards

Guards เป็นเงื่อนไขเพิ่มเติมใน Pattern Matching

```erlang
% Syntax: Pattern when Guard ->

% ใน function clause
absolute(X) when X >= 0 -> X;
absolute(X) when X < 0 -> -X.

% Guards ที่ใช้ได้:
% Comparison: ==, /=, =:=, =/=, <, >, =<, >=
% Type checks: is_integer/1, is_float/1, is_atom/1, etc.
% Arithmetic: +, -, *, /, div, rem, abs/1
% Boolean: and, or, not, andalso, orelse

% ตัวอย่างหลาย guards
classify_age(Age) when Age < 0 ->
    invalid;
classify_age(Age) when Age < 13 ->
    child;
classify_age(Age) when Age < 18 ->
    teenager;
classify_age(Age) when Age < 65 ->
    adult;
classify_age(_) ->
    senior.

% Guards หลายเงื่อนไข
is_valid_user(Name, Age) when is_binary(Name),
                               byte_size(Name) > 0,
                               is_integer(Age),
                               Age >= 0,
                               Age =< 150 ->
    true;
is_valid_user(_, _) ->
    false.

% Comma (,) = AND
% Semicolon (;) = OR
is_number_or_atom(X) when is_number(X); is_atom(X) ->
    true;
is_number_or_atom(_) ->
    false.

% andalso และ orelse ก็ใช้ได้ใน guard
safe_divide(A, B) when is_number(A), is_number(B), B =/= 0 ->
    A / B;
safe_divide(_, 0) ->
    {error, division_by_zero}.
```

### Guard BIFs (Built-in Functions ที่ใช้ใน Guard ได้)

```erlang
% Type checks
is_atom/1, is_binary/1, is_bitstring/1, is_boolean/1,
is_float/1, is_function/1, is_function/2, is_integer/1,
is_list/1, is_map/1, is_number/1, is_pid/1, is_port/1,
is_record/2, is_record/3, is_reference/1, is_tuple/1

% Arithmetic
abs/1, ceil/1, floor/1, round/1, trunc/1,
element/2, hd/1, tl/1, length/1, map_size/1,
node/0, node/1, self/0, size/1, tuple_size/1,
byte_size/1, bit_size/1

% เฉพาะ true/false expression
% ไม่สามารถเรียก user-defined functions ใน guard!
```

---

## Destructuring

Destructuring คือการ extract ค่าออกจาก structure

```erlang
% Destructuring tuple
{X, Y, Z} = {1, 2, 3}.

% Destructuring list
[First | Rest] = [1, 2, 3, 4, 5].
[A, B, C | _] = [1, 2, 3, 4, 5].

% Destructuring nested structure
{ok, {user, Name, Age}} = {ok, {user, "Alice", 30}}.
% Name = "Alice", Age = 30

% Destructuring map
#{name := Name, scores := #{math := Math, english := English}} =
    #{name => "Bob",
      scores => #{math => 95, english => 88}}.
% Name = "Bob", Math = 95, English = 88

% Destructuring กับ = pattern (alias)
% @ (ไม่มีใน Erlang แต่ใช้ = แทนได้)
process_user({user, Name, Age} = User) ->
    io:format("Processing ~s (age ~p)~n", [Name, Age]),
    % User คือ tuple ทั้งหมด
    save_user(User).

% "As" pattern
handle_packet(<<Type:8, _/binary>> = Packet) ->
    case Type of
        1 -> process_type1(Packet);
        2 -> process_type2(Packet);
        _ -> {error, unknown_type}
    end.
```

---

## Nested Patterns

```erlang
% Pattern ซ้อน - ทรงพลังมาก
deep_pattern({ok, [{first, X} | _]}) ->
    {found, X};
deep_pattern({ok, []}) ->
    empty;
deep_pattern({error, _}) ->
    error.

deep_pattern({ok, [{first, 42}]}).   % = {found, 42}
deep_pattern({ok, []}).              % = empty
deep_pattern({error, not_found}).    % = error

% ตัวอย่างจริง: Parse HTTP response
parse_http({status, 200, Body}) ->
    {ok, Body};
parse_http({status, 404, _}) ->
    {error, not_found};
parse_http({status, 500, ErrorMsg}) ->
    {error, {server_error, ErrorMsg}};
parse_http({headers, #{<<"content-type">> := CT} = Headers, Body}) ->
    {ok, CT, Headers, Body}.

% Tree processing
sum_tree({node, Value, Left, Right}) ->
    Value + sum_tree(Left) + sum_tree(Right);
sum_tree(nil) ->
    0.

% Tree = {node, 1, {node, 2, nil, nil}, {node, 3, nil, nil}}
sum_tree({node, 1, {node, 2, nil, nil}, {node, 3, nil, nil}}).
% = 1 + 2 + 3 = 6
```

---

## Pattern Matching กับ case

```erlang
% case expression
process(Input) ->
    case Input of
        {ok, Value} when is_integer(Value) ->
            {integer, Value};
        {ok, Value} when is_binary(Value) ->
            {string, Value};
        {ok, _} ->
            {unknown_type};
        {error, not_found} ->
            not_found;
        {error, Reason} ->
            {error, Reason};
        _ ->
            {unexpected, Input}
    end.

% case ซ้อน
handle_request(Method, Path) ->
    case Method of
        <<"GET">> ->
            case Path of
                <<"/users">> -> list_users();
                <<"/users/", Id/binary>> -> get_user(Id);
                _ -> not_found
            end;
        <<"POST">> ->
            case Path of
                <<"/users">> -> create_user();
                _ -> not_found
            end;
        _ ->
            method_not_allowed
    end.
```

---

## Common Patterns

### Result Pattern

```erlang
% Pattern ที่พบบ่อยที่สุดใน Erlang
{ok, Value}    % success
{error, Reason}  % failure

% Chain operations
fetch_and_process(Id) ->
    case fetch_user(Id) of
        {ok, User} ->
            case validate_user(User) of
                {ok, Valid} ->
                    process_user(Valid);
                {error, Reason} ->
                    {error, Reason}
            end;
        {error, Reason} ->
            {error, Reason}
    end.

% Cleaner version ด้วย helper
fetch_and_process2(Id) ->
    with_result(fetch_user(Id), fun(User) ->
        with_result(validate_user(User), fun(Valid) ->
            process_user(Valid)
        end)
    end).

with_result({ok, Value}, Fun) -> Fun(Value);
with_result({error, _} = Error, _Fun) -> Error.
```

### State Pattern

```erlang
% ใช้ tuple เป็น state
loop(State) ->
    receive
        {add, N} ->
            NewState = State + N,
            loop(NewState);
        {subtract, N} ->
            NewState = State - N,
            loop(NewState);
        {get, Caller} ->
            Caller ! {state, State},
            loop(State);
        stop ->
            ok
    end.
```

### Accumulator Pattern

```erlang
% ใช้ pattern matching กับ accumulator
sum_list(List) -> sum_list(List, 0).
sum_list([], Acc) -> Acc;
sum_list([H|T], Acc) -> sum_list(T, Acc + H).

% Reverse list ด้วย accumulator
reverse_list(List) -> reverse_list(List, []).
reverse_list([], Acc) -> Acc;
reverse_list([H|T], Acc) -> reverse_list(T, [H|Acc]).

% filter ด้วย accumulator
filter_list(Pred, List) -> filter_list(Pred, List, []).
filter_list(_, [], Acc) -> lists:reverse(Acc);
filter_list(Pred, [H|T], Acc) ->
    case Pred(H) of
        true -> filter_list(Pred, T, [H|Acc]);
        false -> filter_list(Pred, T, Acc)
    end.
```

---

## ตัวอย่างจริง: Parser

```erlang
%% ไฟล์: json_parser_simple.erl
-module(json_parser_simple).
-export([parse/1]).

%% Simple JSON-like value parser
parse(<<"null">>) -> null;
parse(<<"true">>) -> true;
parse(<<"false">>) -> false;
parse(<<$", Rest/binary>>) ->
    {Str, _} = parse_string(Rest, <<>>),
    Str;
parse(<<$[, Rest/binary>>) ->
    parse_array(Rest, []);
parse(<<${, Rest/binary>>) ->
    parse_object(Rest, #{});
parse(Num) when is_binary(Num) ->
    try binary_to_integer(Num)
    catch _ -> binary_to_float(Num)
    end.

parse_string(<<$", Rest/binary>>, Acc) ->
    {Acc, Rest};
parse_string(<<$\\, $", Rest/binary>>, Acc) ->
    parse_string(Rest, <<Acc/binary, $">>);
parse_string(<<C, Rest/binary>>, Acc) ->
    parse_string(Rest, <<Acc/binary, C>>).

parse_array(<<$], Rest/binary>>, Acc) ->
    {lists:reverse(Acc), Rest};
parse_array(<<$,, Rest/binary>>, Acc) ->
    parse_array(Rest, Acc);
parse_array(Data, Acc) ->
    {Value, Rest} = parse_value(Data),
    parse_array(Rest, [Value|Acc]).

parse_object(<<$}, Rest/binary>>, Acc) ->
    {Acc, Rest};
parse_object(<<$", Rest/binary>>, Acc) ->
    {Key, <<$:, ValueRest/binary>>} = parse_string(Rest, <<>>),
    {Value, Rest2} = parse_value(ValueRest),
    parse_object(skip_comma(Rest2), Acc#{Key => Value}).

skip_comma(<<$,, Rest/binary>>) -> Rest;
skip_comma(Rest) -> Rest.

parse_value(Data) ->
    % simplified - จริงๆ ต้องซับซ้อนกว่านี้
    {parse(Data), <<>>}.
```

---

## Pattern Matching Best Practices

```erlang
% 1. ใส่ specific patterns ก่อน general patterns
handle({ok, []}) ->         % specific
    empty;
handle({ok, [H|_]}) ->     % less specific
    {first, H};
handle({ok, _}) ->          % general
    ok;
handle(_) ->                % catch-all
    error.

% 2. ใช้ Guards เพื่อความชัดเจน
process(X) when is_integer(X), X > 0 ->
    positive_integer;
process(X) when is_integer(X), X < 0 ->
    negative_integer;
process(0) ->
    zero.

% 3. ใช้ _ สำหรับค่าที่ไม่ใช้
{_, ImportantValue, _} = some_tuple().

% 4. ใช้ named wildcard เพื่อ documentation
{_ErrorCode, Reason} = get_error().
% _ErrorCode บอกว่า value นี้เป็น error code แต่เราไม่ใช้

% 5. Match as early as possible
%    แทนที่จะ:
handle(Data) ->
    Value = maps:get(key, Data),
    process(Value).

%    ทำ:
handle(#{key := Value}) ->
    process(Value).

% 6. Use complete pattern matching - avoid partial matches
% Partial match อาจ crash ถ้า input ไม่ตรง
handle({ok, Value}) -> Value.  % จะ crash ถ้า input เป็น {error, _}

% Complete match:
handle({ok, Value}) -> Value;
handle({error, Reason}) -> throw({error, Reason}).
```

---

## สรุป Part 5

Pattern Matching คือหัวใจของ Erlang:

| Feature | Description |
|---------|-------------|
| Single Assignment | ตัวแปร bind ครั้งเดียว |
| `=` Operator | Match + Bind |
| Tuple Matching | `{A, B} = {1, 2}` |
| List Matching | `[H\|T] = [1,2,3]` |
| Map Matching | `#{k := V} = #{k=>1}` |
| Binary Matching | `<<A, B>> = <<1, 2>>` |
| Function Clauses | Multiple patterns per function |
| Guards | Extra conditions with `when` |
| Destructuring | Extract values from structures |
| Nested Patterns | Match deep structures |

### ก้าวต่อไป

ใน **Part 6** เราจะเรียน Functions อย่างละเอียด:
- Function Clauses
- Guards ใน Functions
- Recursive Functions
- Tail Recursion
- Higher-Order Functions
- Anonymous Functions (Funs)

---

## แบบฝึกหัด Part 5

```erlang
%% แบบฝึกหัดที่ 1: เขียน function ต่อไปนี้
%% shape_area/1 ที่รับ shape tuple และคำนวณพื้นที่

%% Shapes:
%% {circle, Radius}
%% {rectangle, Width, Height}
%% {triangle, Base, Height}
%% {square, Side}

shape_area({circle, R}) -> math:pi() * R * R;
shape_area({rectangle, W, H}) -> W * H;
% ... เพิ่ม cases อื่นๆ

%% แบบฝึกหัดที่ 2: เขียน function สำหรับ binary protocol
%% parse_message/1 ที่ parse binary protocol:
%% Byte 0: message type (1=ping, 2=pong, 3=data)
%% Byte 1-2: payload length (big-endian)
%% Byte 3+: payload

parse_message(<<Type:8, Length:16/big, Payload:Length/binary>>) ->
    % ...

%% แบบฝึกหัดที่ 3: Binary Tree Pattern Matching
%% insert/2 ที่ insert value ลง Binary Search Tree
%% search/2 ที่ค้นหา value ใน BST

insert(Value, nil) -> {node, Value, nil, nil};
insert(Value, {node, NodeVal, Left, Right}) when Value < NodeVal ->
    {node, NodeVal, insert(Value, Left), Right};
% ... เพิ่ม cases
```

---

*[← Part 4: Data Types](part_04_data_types.md) | [Part 6: Functions →](part_06_functions.md)*
