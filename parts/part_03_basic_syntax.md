# Part 3: Syntax พื้นฐานของ Erlang

## สารบัญ
1. [โครงสร้างไฟล์ .erl](#โครงสร้างไฟล์-erl)
2. [Comments](#comments)
3. [Atoms](#atoms)
4. [Numbers](#numbers)
5. [Variables](#variables)
6. [Strings](#strings)
7. [Atoms vs Strings](#atoms-vs-strings)
8. [Lists](#lists)
9. [Tuples](#tuples)
10. [Maps](#maps)
11. [Binaries](#binaries)
12. [Operators](#operators)
13. [Expressions และ Statements](#expressions-และ-statements)
14. [Module System](#module-system)
15. [Function Declarations](#function-declarations)
16. [สรุป](#สรุป)

---

## โครงสร้างไฟล์ .erl

ทุกไฟล์ Erlang มีโครงสร้างพื้นฐานดังนี้:

```erlang
%% ชื่อ module ต้องตรงกับชื่อไฟล์ (ไม่รวม .erl)
-module(module_name).

%% Module attributes (attributes)
-author("Your Name").
-vsn("1.0.0").

%% Exports - functions ที่เปิดให้เรียกจากภายนอก
-export([function1/0, function2/1, function3/2]).

%% Imports (ไม่ค่อยใช้ แต่มีให้ใช้)
-import(lists, [map/2, filter/2]).

%% Includes (header files)
-include("my_header.hrl").
-include_lib("kernel/include/logger.hrl").

%% Defines (macros)
-define(MAX_SIZE, 1000).
-define(VERSION, "1.0.0").

%% Type definitions
-type name() :: binary() | string().
-type age() :: non_neg_integer().

%% Record definitions
-record(person, {
    name :: name(),
    age  :: age()
}).

%% ตัวอย่าง functions
function1() ->
    ok.

function2(Arg1) ->
    Arg1.

function3(Arg1, Arg2) ->
    Arg1 + Arg2.
```

---

## Comments

Erlang ใช้ `%` สำหรับ comments:

```erlang
% Single line comment

%% Two percent signs - ใช้สำหรับ module-level comments (convention)

%%% Three percent signs - ใช้สำหรับ top-of-file description (convention)

%%% @doc
%%% Module description
%%% @end

% ไม่มี multi-line comment syntax
% ต้องใส่ % ทุกบรรทัด

%--- Section divider (convention) ---
```

### Edoc Comments

```erlang
%% @doc
%% คำอธิบาย function นี้
%%
%% @param Name ชื่อของบุคคล
%% @returns สตริงทักทาย
%% @end
-spec greet(Name :: string()) -> string().
greet(Name) ->
    "Hello, " ++ Name ++ "!".
```

---

## Atoms

**Atom** คือ literal constant ที่มีชื่อเป็นค่าของมันเอง

```erlang
% Atoms ขึ้นต้นด้วยตัวพิมพ์เล็ก
hello
world
ok
error
true
false
nil
undefined

% Atoms ที่มีอักขระพิเศษ ต้องใส่ quotes
'Hello World'
'hello-world'
'with spaces'
'CamelCase'
'123starts_with_number'

% Atoms ที่ใช้บ่อย
ok           % สำหรับ success
error        % สำหรับ failure
true         % boolean true
false        % boolean false
undefined    % ค่าที่ไม่ได้กำหนด
nil          % ค่าว่าง (ใช้น้อยกว่า undefined)
```

### การใช้งาน Atoms

```erlang
% ใช้เป็น tag ใน tuple
{ok, Result} = some_function().
{error, Reason} = another_function().

% ใช้เป็น boolean
IsValid = true.
HasError = false.

% ใช้ใน pattern matching
handle_result({ok, Value}) ->
    io:format("Success: ~p~n", [Value]);
handle_result({error, Reason}) ->
    io:format("Error: ~p~n", [Reason]).

% Atoms ไม่ใช่ Strings!
Atom = hello.
% io:format("~s~n", [Atom]).  % ไม่ถูก! ต้องใช้ ~p หรือ atom_to_list/1

% แปลง Atom เป็น String
AtomAsString = atom_to_list(hello).  % "hello"
% แปลง String เป็น Atom
StringAsAtom = list_to_atom("hello").  % hello
```

### Atom Limits

```erlang
% Erlang จำกัด atoms ไว้ที่ 1,048,576 (2^20) ตัวต่อ VM
% ระวัง! อย่า dynamic generate atoms มากเกินไป

% อันตราย - สร้าง atom จาก user input
% list_to_atom(UserInput)  % BAD! อาจทำให้ atom table เต็ม

% ปลอดภัยกว่า - ใช้ list_to_existing_atom/1
% list_to_existing_atom("hello")  % OK - ถ้า hello มีอยู่แล้ว
% list_to_existing_atom("new_atom")  % throws error ถ้าไม่มี
```

---

## Numbers

### Integers

```erlang
% ทศนิยม (Decimal)
42
-17
0
1000000

% เลขฐาน 2 (Binary)
2#1010      % = 10
2#1111      % = 15

% เลขฐาน 8 (Octal)
8#17        % = 15
8#777       % = 511

% เลขฐาน 16 (Hexadecimal)
16#FF       % = 255
16#DEADBEEF % = 3735928559

% ฐานอื่นๆ (2 ถึง 36)
36#Z        % = 35
16#1F       % = 31

% Character codes
$A          % = 65
$a          % = 97
$\n         % = 10 (newline)
$\t         % = 9  (tab)
$\s         % = 32 (space)
```

### Floats

```erlang
% Float ต้องมีจุดทศนิยม
3.14
-2.5
1.0
0.001

% Scientific notation
1.0e10      % = 10000000000.0
1.5e-3      % = 0.0015
2.3e+5      % = 230000.0
```

### การคำนวณ

```erlang
% Integer arithmetic
10 + 3.     % = 13
10 - 3.     % = 7
10 * 3.     % = 30
10 div 3.   % = 3  (integer division)
10 rem 3.   % = 1  (remainder)

% Float arithmetic
10.0 / 3.   % = 3.3333...
10 / 3.     % = 3.3333... (/ always returns float!)

% Mixed
10 + 3.14.  % = 13.14 (integer + float = float)

% Math functions
math:sqrt(16).     % = 4.0
math:pow(2, 10).   % = 1024.0
math:sin(math:pi() / 2).  % = 1.0
math:log(math:exp(1)).    % = 1.0
```

### Integer Functions ที่สำคัญ

```erlang
abs(-5).        % = 5
max(3, 7).      % = 7
min(3, 7).      % = 3

% Type checking
is_integer(42).  % = true
is_float(3.14).  % = true
is_number(42).   % = true (integers and floats)

% Conversion
float(5).        % = 5.0
round(3.7).      % = 4
trunc(3.7).      % = 3
floor(3.7).      % = 3
ceiling(3.2).    % = 4

% Integer to list (string representation)
integer_to_list(42).      % = "42"
integer_to_list(255, 16). % = "FF" (base 16)

% List to integer
list_to_integer("42").    % = 42
list_to_integer("FF", 16).% = 255
```

---

## Variables

Variables ใน Erlang เริ่มต้นด้วยตัวพิมพ์ใหญ่หรือ underscore (`_`)

```erlang
% ตัวแปรปกติ - ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่
Name = "Alice".
Age = 30.
Score = 95.5.

% CamelCase (convention)
FirstName = "Alice".
LastName = "Smith".
UserEmail = "alice@example.com".

% Wildcard - _ เดี่ยวๆ ใช้เมื่อไม่สนใจค่า
{_, SecondValue} = {1, 2}.   % ไม่สนใจค่าแรก
SecondValue.  % = 2

% Named wildcard - _VariableName ใช้เพื่อ document แต่ไม่ binding
{_FirstIgnored, ActualValue} = {1, 2}.
ActualValue.  % = 2
```

### Single Assignment (Immutability)

```erlang
% ตัวแปรถูก bind เพียงครั้งเดียว!
X = 5.
% X = 10.  % ERROR: variable X already bound to 5

% ใช้ f() ใน shell เพื่อ unbound
f(X).
X = 10.  % OK แล้วใน shell

% ในโปรแกรมจริง สร้างตัวแปรใหม่
X1 = 5.
X2 = X1 + 1.   % = 6 (X1 ยังเป็น 5)
X3 = X2 * 2.   % = 12

% Pattern matching กับ variables
{A, B, C} = {1, 2, 3}.
% A = 1, B = 2, C = 3
```

### Variable Naming Convention

```erlang
% ชื่อที่ดี
UserName = "alice".
MaxRetries = 3.
DatabaseUrl = "postgres://...".
IsValid = true.

% Erlang community conventions:
% - ใช้ CamelCase
% - ชื่อ descriptive
% - ไม่ใช้ I, J, K (ยกเว้น loop counter ที่ชัดเจน)
% - Prefix _ สำหรับ unused variables
```

---

## Strings

Erlang มี strings 2 แบบ:

### 1. List-based Strings (ดั้งเดิม)

```erlang
% String เป็น list ของ integers (character codes)
Name = "Alice".
% เท่ากับ [65, 108, 105, 99, 101]

% String operations
"Hello" ++ " " ++ "World".  % = "Hello World" (concatenation)
length("Hello").             % = 5
hd("Hello").                 % = 72 ($H)
tl("Hello").                 % = "ello"

% ตรวจสอบว่าเป็น string
is_list("hello").  % = true (string เป็น list)

% String comparison
"abc" == "abc".  % = true
"abc" < "def".   % = true (lexicographic)

% string module
string:upper("hello").         % = "HELLO"
string:lower("HELLO").         % = "hello"
string:len("hello").           % = 5
string:sub_string("hello", 2). % = "ello"
string:tokens("a,b,c", ",").   % = ["a", "b", "c"]
string:join(["a", "b", "c"], ",").  % = "a,b,c"
```

### 2. Binary Strings (แนะนำสำหรับ Production)

```erlang
% Binary string ใช้ << >> notation
Name = <<"Alice">>.
Greeting = <<"Hello, World!">>.

% ใช้ binary เมื่อ:
% - ทำงานกับ network data
% - ต้องการ performance สูง
% - ทำงานกับ UTF-8 text

% Binary concatenation
A = <<"Hello">>.
B = <<", ">>.
C = <<"World!">>.
Result = <<A/binary, B/binary, C/binary>>.
% = <<"Hello, World!">>

% Binary ใน function argument
process_name(<<"Alice">>) -> 
    io:format("Processing Alice~n");
process_name(Name) when is_binary(Name) ->
    io:format("Processing: ~s~n", [Name]).
```

---

## Atoms vs Strings

ความสับสนที่พบบ่อยใน Erlang:

```erlang
% Atom
hello         % ตัว atom ที่มีชื่อว่า hello
is_atom(hello)  % = true

% String (list)
"hello"       % list ของ characters [104, 101, 108, 108, 111]
is_list("hello")  % = true

% Binary String
<<"hello">>   % binary representation
is_binary(<<"hello">>)  % = true

% การแปลงระหว่างกัน
atom_to_list(hello).           % = "hello"
list_to_atom("hello").         % = hello
atom_to_binary(hello).         % = <<"hello">>
binary_to_atom(<<"hello">>).   % = hello
```

### เมื่อไรใช้อะไร?

```erlang
% ใช้ Atom:
% - ใน pattern matching (tags, statuses)
{ok, Result}
{error, not_found}
Status = running

% - เป็น function names, module names
% - เป็น map keys ที่รู้ล่วงหน้า
Person = #{name => "Alice", age => 30}

% ใช้ String (binary):
% - ข้อความที่มาจาก user
% - Network data
% - Database values
% - Dynamic content

% ใช้ String (list):
% - เมื่อต้องการ character-by-character processing
% - Legacy code
% - เข้ากันได้กับ io:format patterns
```

---

## Lists

List เป็น data structure พื้นฐานที่สำคัญที่สุดใน Erlang

```erlang
% List ว่าง
[].

% List ของ integers
[1, 2, 3, 4, 5].

% List ของ atoms
[hello, world, ok].

% List ผสม (ได้แต่ไม่แนะนำ)
[1, hello, "world", {ok, result}].

% Nested lists
[[1, 2], [3, 4], [5, 6]].
```

### List Operations พื้นฐาน

```erlang
% Head (element แรก) และ Tail (ที่เหลือ)
List = [1, 2, 3, 4, 5].

hd(List).   % = 1 (head)
tl(List).   % = [2, 3, 4, 5] (tail)

% Pattern matching กับ list
[H | T] = [1, 2, 3, 4, 5].
% H = 1, T = [2, 3, 4, 5]

[First, Second | Rest] = [a, b, c, d].
% First = a, Second = b, Rest = [c, d]

% Concatenation
[1, 2] ++ [3, 4].    % = [1, 2, 3, 4]
[1, 2, 3] -- [2].    % = [1, 3] (remove)

% Length
length([1, 2, 3]).   % = 3

% Member check
lists:member(3, [1, 2, 3, 4]).  % = true

% เข้าถึง element ที่ n
lists:nth(2, [a, b, c]).  % = b (1-indexed)
```

### Lists Module Functions

```erlang
% lists:map/2 - apply function ทุก element
lists:map(fun(X) -> X * 2 end, [1, 2, 3]).
% = [2, 4, 6]

% lists:filter/2 - กรอง elements
lists:filter(fun(X) -> X > 2 end, [1, 2, 3, 4, 5]).
% = [3, 4, 5]

% lists:foldl/3 - reduce จากซ้ายไปขวา
lists:foldl(fun(X, Acc) -> X + Acc end, 0, [1, 2, 3, 4, 5]).
% = 15 (sum)

% lists:foldr/3 - reduce จากขวาไปซ้าย
lists:foldr(fun(X, Acc) -> [X | Acc] end, [], [1, 2, 3]).
% = [1, 2, 3] (สร้าง list ใหม่)

% lists:sort/1 - เรียงลำดับ
lists:sort([3, 1, 4, 1, 5, 9, 2, 6]).
% = [1, 1, 2, 3, 4, 5, 6, 9]

% lists:reverse/1 - กลับด้าน
lists:reverse([1, 2, 3]).
% = [3, 2, 1]

% lists:flatten/1 - ทำ list ให้แบน
lists:flatten([[1, 2], [3, [4, 5]]]).
% = [1, 2, 3, 4, 5]

% lists:zip/2 - รวม 2 lists
lists:zip([1, 2, 3], [a, b, c]).
% = [{1, a}, {2, b}, {3, c}]

% lists:unzip/1 - แยก list ของ tuples
lists:unzip([{1, a}, {2, b}, {3, c}]).
% = {[1, 2, 3], [a, b, c]}
```

---

## Tuples

Tuple คือ sequence ของ values ที่มีขนาดคงที่

```erlang
% Tuple ว่าง
{}.

% Tuple 2 elements (pair)
{hello, world}.
{ok, Result}.
{error, Reason}.

% Tuple 3 elements
{point, 3, 4}.
{rgb, 255, 128, 0}.
{person, "Alice", 30}.

% Tuple ซ้อน
{{1, 2}, {3, 4}}.

% Accessing tuple elements
T = {alice, 30, "alice@example.com"}.
element(1, T).  % = alice
element(2, T).  % = 30
element(3, T).  % = "alice@example.com"

% Pattern matching (ดีกว่า element/2)
{Name, Age, Email} = {alice, 30, "alice@example.com"}.
% Name = alice, Age = 30, Email = "alice@example.com"

% tuple_size
tuple_size({a, b, c}).  % = 3
```

### Tagged Tuples (Convention)

```erlang
% Convention: ใช้ atom เป็น tag แรก
{ok, "success"}
{error, "not found"}
{point, 3.0, 4.0}
{user, "alice", 30}
{http_request, get, "/api/users"}

% ทำให้ pattern matching ชัดเจนขึ้น
process({ok, Data}) ->
    use_data(Data);
process({error, Reason}) ->
    handle_error(Reason).
```

---

## Maps

Maps คือ key-value data structure (เหมือน Dictionary/Hash Map)

```erlang
% สร้าง Map
Empty = #{}.

Person = #{
    name => "Alice",
    age  => 30,
    email => "alice@example.com"
}.

% Access values
maps:get(name, Person).   % = "Alice"
maps:get(age, Person).    % = 30

% Alternative syntax
Person#{name}.  % ไม่ถูกต้องใน Erlang! ต้องใช้ maps:get

% Pattern matching กับ Map
#{name := Name, age := Age} = Person.
% Name = "Alice", Age = 30

% Update Map (สร้างใหม่)
UpdatedPerson = Person#{age => 31}.
% #{name => "Alice", age => 31, email => "alice@example.com"}

% เพิ่ม key ใหม่
WithPhone = Person#{phone => "123-456-7890"}.

% ลบ key
maps:remove(email, Person).

% ตรวจสอบ key
maps:is_key(name, Person).   % = true
maps:is_key(phone, Person).  % = false

% ดู keys และ values
maps:keys(Person).    % = [age, email, name] (sorted)
maps:values(Person).  % = [30, "alice@example.com", "Alice"]

% Size
maps:size(Person).    % = 3

% แปลงเป็น list
maps:to_list(Person). % = [{age, 30}, {email, "alice@..."}, {name, "Alice"}]
maps:from_list([{a, 1}, {b, 2}]).  % = #{a => 1, b => 2}
```

---

## Binaries

Binary คือ sequence ของ bytes ที่มีประสิทธิภาพสูง

```erlang
% Binary literals
<<>>.           % empty binary
<<1, 2, 3>>.   % binary of 3 bytes
<<"hello">>.   % string as binary

% Binary pattern matching
<<First, Rest/binary>> = <<1, 2, 3, 4, 5>>.
% First = 1, Rest = <<2, 3, 4, 5>>

<<A:8, B:8, C:8>> = <<1, 2, 3>>.
% A = 1, B = 2, C = 3

% Binary segments
<<Int:32/integer-big>> = <<0, 0, 1, 0>>.  % = 256

% Binary operations
byte_size(<<"hello">>).   % = 5
bit_size(<<"hello">>).    % = 40
binary_part(<<"hello world">>, 6, 5).  % = <<"world">>

% Binary concatenation
A = <<"hello">>.
B = <<" world">>.
<<A/binary, B/binary>>.  % = <<"hello world">>
```

### Binary Comprehensions

```erlang
% Binary comprehension
<< <<X*2>> || <<X>> <= <<1, 2, 3, 4, 5>> >>.
% = <<2, 4, 6, 8, 10>>
```

---

## Operators

### Arithmetic Operators

```erlang
5 + 3.    % = 8
5 - 3.    % = 2
5 * 3.    % = 15
5 / 3.    % = 1.6666... (float division)
5 div 3.  % = 1 (integer division)
5 rem 3.  % = 2 (remainder)
-5.       % = -5 (negation)
```

### Comparison Operators

```erlang
5 == 5.      % = true (equal)
5 /= 5.      % = false (not equal)
5 =:= 5.     % = true (exactly equal, no type coercion)
5 =/= 5.     % = false (not exactly equal)
5 > 3.       % = true
5 < 3.       % = false
5 >= 5.      % = true
5 =< 5.      % = true (note: =< not <=)

% เปรียบเทียบ type ต่างกัน
5 == 5.0.    % = true (== อนุญาต type coercion)
5 =:= 5.0.  % = false (=:= ไม่อนุญาต type coercion)
```

### Logical Operators

```erlang
% Boolean operators
true and false.   % = false
true or false.    % = true
not true.         % = false
true xor false.   % = true

% Short-circuit operators (แนะนำ)
true andalso false.   % = false (short-circuit)
false orelse true.    % = true (short-circuit)

% ความแตกต่าง:
% and/or: ประเมินทั้งสองด้านเสมอ
% andalso/orelse: short-circuit (ดีกว่าเพราะปลอดภัยกว่า)

% ตัวอย่าง:
is_valid(X) ->
    is_integer(X) andalso X > 0.
% ถ้า is_integer(X) เป็น false จะไม่ประเมิน X > 0
```

### Bitwise Operators

```erlang
band(5, 3).   % = 1 (bitwise AND): 101 & 011 = 001
bor(5, 3).    % = 7 (bitwise OR):  101 | 011 = 111
bxor(5, 3).   % = 6 (bitwise XOR): 101 ^ 011 = 110
bnot(5).      % = -6 (bitwise NOT)
bsl(5, 2).    % = 20 (shift left):  101 << 2 = 10100
bsr(20, 2).   % = 5 (shift right): 10100 >> 2 = 101
```

### List Operators

```erlang
[1, 2] ++ [3, 4].   % = [1, 2, 3, 4] (concatenate)
[1, 2, 3] -- [2].   % = [1, 3] (difference)
```

### Operator Precedence

```erlang
% ลำดับความสำคัญ (สูงไปต่ำ):
% 1. : (record/map access)
% 2. # (record field)
% 3. unary +, -, bnot, not
% 4. /, *, div, rem, band, and
% 5. +, -, bor, bxor, or, xor
% 6. ++, --
% 7. ==, /=, =:=, =/=, <, =<, >, >=
% 8. andalso
% 9. orelse
% 10. = (pattern match)
% 11. ! (send message)
```

---

## Expressions และ Statements

ทุกอย่างใน Erlang เป็น **Expression** ที่มีค่า return

```erlang
% ทุก expression มีค่า
1 + 1.            % = 2
"hello".          % = "hello"
ok.               % = ok
[1, 2, 3].       % = [1, 2, 3]

% Compound expressions
X = begin
    A = 1,
    B = 2,
    A + B      % ค่า expression สุดท้ายเป็นค่าของ begin...end
end.
% X = 3

% Semicolons vs Commas vs Dots
% , (comma) - แยก expressions ในบล็อก (ทุก expression ถูกประเมิน)
% ; (semicolon) - แยก clauses ใน case, if, receive, fun
% . (dot) - จบ declaration (module attribute, function)
```

---

## Module System

```erlang
%% mymodule.erl

-module(mymodule).

%% Export list - กำหนดว่า function ไหนเรียกจากภายนอกได้
-export([
    public_function/0,
    another_public/1
]).

%% ไม่ export = private function (เรียกได้แค่ภายใน module)
private_helper() ->
    "I'm private".

public_function() ->
    private_helper() ++ " but I'm public".

another_public(X) ->
    X * 2.
```

### เรียกใช้ Functions

```erlang
% เรียก function ใน module เดียวกัน
add(1, 2).

% เรียก function ข้าม module
lists:sort([3, 1, 2]).

% เรียกด้วย Variable module (dynamic dispatch)
Module = lists.
Module:sort([3, 1, 2]).
```

---

## Function Declarations

```erlang
%% Function declaration
function_name(Pattern1, Pattern2) ->
    Body;
function_name(Pattern3, Pattern4) ->
    Body2.

%% หลายๆ clauses
greet("Alice") ->
    "Hello, Alice!";
greet("Bob") ->
    "Hey Bob!";
greet(Name) ->
    "Hello, " ++ Name ++ "!".

%% Function ที่ return หลาย types
parse_number(Str) ->
    try
        {ok, list_to_integer(Str)}
    catch
        _:_ ->
            try
                {ok, list_to_float(Str)}
            catch
                _:_ -> {error, not_a_number}
            end
    end.
```

### Function Arity

```erlang
% Function ระบุด้วย Name/Arity
greet/0    % function greet ที่ไม่มี argument
greet/1    % function greet ที่มี 1 argument
add/2      % function add ที่มี 2 arguments

% ชื่อเดียวกัน Arity ต่างกัน = คนละ function!
-export([greet/0, greet/1]).

greet() ->
    greet("World").

greet(Name) ->
    "Hello, " ++ Name ++ "!".
```

---

## สรุป Part 3

เราได้เรียนรู้ Syntax พื้นฐานของ Erlang:

| เรื่อง | สรุป |
|--------|------|
| Comments | ใช้ `%` |
| Atoms | Literal constants ที่ขึ้นต้นด้วยตัวพิมพ์เล็ก |
| Numbers | Integer, Float, Binary, Hex, Octal |
| Variables | ขึ้นต้นด้วยตัวพิมพ์ใหญ่, assign ครั้งเดียว |
| Strings | List-based `"hello"` หรือ Binary `<<"hello">>` |
| Lists | `[1, 2, 3]` - mutable sequence |
| Tuples | `{a, b, c}` - fixed-size sequence |
| Maps | `#{key => value}` - key-value store |
| Binaries | `<<1, 2, 3>>` - byte sequences |
| Operators | `+`, `-`, `*`, `div`, `rem`, `and`, `or` |

### ก้าวต่อไป

ใน **Part 4** เราจะเรียน Data Types โดยละเอียด:
- ประเภทข้อมูลทั้งหมดของ Erlang
- Type Checking Functions
- Type Conversion
- Comparison Ordering

---

## แบบฝึกหัด Part 3

```erlang
% แบบฝึกหัดที่ 1: พิมพ์ค่าต่อไปนี้และดูผลลัพธ์
% ใน Erlang shell:
1> 16#FF.
2> $A.
3> 2#11111111.
4> [72, 101, 108, 108, 111].
5> <<72, 101, 108, 108, 111>>.
6> <<"Hello">>.
7> atom_to_list(hello).
8> list_to_atom("world").
```

```erlang
% แบบฝึกหัดที่ 2: เขียน module ที่มี functions ต่อไปนี้
-module(practice).
-export([...]).

% area_circle/1 - คำนวณพื้นที่วงกลม (รับ radius)
% area_rectangle/2 - คำนวณพื้นที่สี่เหลี่ยม (รับ width, height)
% full_name/2 - รวม first_name กับ last_name เป็น binary
% is_adult/1 - ตรวจสอบว่าอายุ >= 18 หรือไม่
```

---

*[← Part 2: การติดตั้ง](part_02_installation.md) | [Part 4: Data Types →](part_04_data_types.md)*
