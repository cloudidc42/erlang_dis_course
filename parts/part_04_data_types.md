# Part 4: ชนิดข้อมูล (Data Types) ใน Erlang

## สารบัญ
1. [ภาพรวม Data Types](#ภาพรวม-data-types)
2. [Terms และ Values](#terms-และ-values)
3. [Atoms (ทบทวน + เพิ่มเติม)](#atoms-ทบทวน--เพิ่มเติม)
4. [Numbers](#numbers)
5. [Booleans](#booleans)
6. [Lists (ทบทวน + เพิ่มเติม)](#lists-ทบทวน--เพิ่มเติม)
7. [Tuples (ทบทวน + เพิ่มเติม)](#tuples-ทบทวน--เพิ่มเติม)
8. [Maps](#maps)
9. [Binaries และ Bitstrings](#binaries-และ-bitstrings)
10. [Strings](#strings)
11. [PIDs (Process IDs)](#pids-process-ids)
12. [Ports](#ports)
13. [References](#references)
14. [Funs (Anonymous Functions)](#funs-anonymous-functions)
15. [Type Checking Functions](#type-checking-functions)
16. [Type Conversion](#type-conversion)
17. [Term Ordering](#term-ordering)
18. [สรุป](#สรุป)

---

## ภาพรวม Data Types

Erlang มี data types ทั้งหมดดังนี้:

```
Erlang Data Types
├── Immediate Types (ขนาดเล็ก, เก็บโดยตรง)
│   ├── Atom (hello, ok, error, true, false)
│   ├── Small Integer (-268435455 to 268435455)
│   └── PID, Port, Reference (เล็กๆ)
│
└── Box Types (เก็บใน Heap)
    ├── Big Integer (ใหญ่กว่า small integer)
    ├── Float (64-bit double)
    ├── List ([1, 2, 3])
    ├── Tuple ({a, b, c})
    ├── Map (#{key => value})
    ├── Binary (<<1, 2, 3>>)
    ├── Bitstring (<<1:4, 2:4>>)
    └── Fun (fun() -> ok end)
```

---

## Terms และ Values

ใน Erlang เราเรียกทุก value ว่า **term**

```erlang
% ทุกอย่างเป็น term
42              % integer term
3.14            % float term
hello           % atom term
"world"         % list term (string)
[1, 2, 3]       % list term
{ok, result}    % tuple term
#{a => 1}       % map term
<<1, 2, 3>>     % binary term
fun() -> ok end % fun term
self()          % pid term

% Term สามารถ nested ได้
{user, <<"Alice">>, 30, [friend1, friend2], #{score => 100}}.
```

---

## Atoms (ทบทวน + เพิ่มเติม)

### Predefined Atoms

```erlang
% Atoms ที่ถูก build-in
true       % boolean true
false      % boolean false
undefined  % ค่าที่ไม่ได้กำหนด (convention)
nil        % null/none (ใช้น้อยกว่า undefined)
ok         % success result
error      % failure result
infinity   % infinity ใน math context
```

### Atom ใน Pattern Matching

```erlang
% Pattern matching ที่พบบ่อย
handle({ok, Value}) ->
    {success, Value};
handle({error, not_found}) ->
    {failure, "Record not found"};
handle({error, permission_denied}) ->
    {failure, "Permission denied"};
handle({error, Reason}) ->
    {failure, Reason}.

% ใน case
process_response(Response) ->
    case Response of
        {ok, 200, Body} ->
            parse_body(Body);
        {ok, 404, _} ->
            not_found;
        {ok, StatusCode, _} ->
            {unexpected_status, StatusCode};
        {error, timeout} ->
            retry;
        {error, _} = Error ->
            Error
    end.
```

### Atom Table

```erlang
% ดู atom count
erlang:system_info(atom_count).
% = 9xxx (เพิ่มขึ้นเมื่อสร้าง atom ใหม่)

erlang:system_info(atom_limit).
% = 1048576 (default limit)

% ระวัง! อย่า dynamic create atoms จากข้อมูล user
% list_to_atom(UserInput)  % DANGEROUS!

% ปลอดภัยกว่า:
erlang:list_to_existing_atom(UserInput).
% จะ throw exception ถ้า atom ไม่มีอยู่ก่อน
```

---

## Numbers

### Integer Range

```erlang
% Erlang integers มีขนาดไม่จำกัด (arbitrary precision)!
SmallInt = 42.
BigInt = 1000000000000000000000000000000.   % ได้เลย!

% ตัวอย่าง: คำนวณ factorial ของตัวเลขใหญ่
factorial(0) -> 1;
factorial(N) -> N * factorial(N - 1).

factorial(100).
% = 933262154439441526816992388562667004907159682643816214685929638952175999932299156089414639761565182862536979208272237582511852109168640000000000000000000000000

% ไม่มี integer overflow ใน Erlang!
```

### Float (IEEE 754 Double Precision)

```erlang
% Float มีความแม่นยำ 15-16 หลัก
3.14.
-2.718281828459045.
1.0e308.    % ใกล้ max float
1.0e-308.   % ใกล้ min positive float

% Float special values
math:pow(10, 400).  % = inf (infinity)
math:log(-1).       % = nan (not a number)

% ระวัง floating point precision!
0.1 + 0.2.  % = 0.30000000000000004 ไม่ใช่ 0.3!
0.1 + 0.2 =:= 0.3.  % = false!!

% ทางแก้: ใช้ round/2 หรือ trunc/1
round(0.1 + 0.2, 1) =:= 0.3.  % = true
% หรือ เปรียบเทียบด้วย tolerance
abs(0.1 + 0.2 - 0.3) < 1.0e-10.  % = true
```

### Number Formatting

```erlang
% ดู number เป็น different bases
X = 255.
io:format("Decimal: ~p~n", [X]).      % 255
io:format("Hex: ~.16b~n", [X]).       % ff
io:format("Binary: ~.2b~n", [X]).     % 11111111
io:format("Octal: ~.8b~n", [X]).      % 377

% Number conversion
integer_to_list(255, 16).   % = "FF"
integer_to_list(255, 2).    % = "11111111"
integer_to_binary(255).     % = <<"255">>
float_to_list(3.14).        % = "3.14000000000000012434e+00"
float_to_list(3.14, [{decimals, 2}]).  % = "3.14"
```

---

## Booleans

Erlang ไม่มี boolean type แยกต่างหาก แต่ใช้ atoms `true` และ `false`

```erlang
% Booleans เป็น atoms
true.
false.

is_atom(true).   % = true
is_atom(false).  % = true

% Boolean operations
true and false.   % = false
true or false.    % = true
not true.         % = false
true xor false.   % = true

% Short-circuit (ดีกว่า)
true andalso false.  % = false (ไม่ evaluate ด้านขวาถ้าด้านซ้ายเป็น false)
false orelse true.   % = true (ไม่ evaluate ด้านขวาถ้าด้านซ้ายเป็น true)

% Comparison functions return boolean atoms
is_integer(42).   % = true
is_float(3.14).   % = true
is_atom(hello).   % = true
is_list([1,2]).   % = true
1 > 0.            % = true
```

### Boolean Pitfalls

```erlang
% ระวัง! Erlang ไม่มี truthy/falsy เหมือน JavaScript/Python
% ใน Erlang มีแค่ true กับ false เท่านั้น!

% ผิด - นี่จะไม่ทำงานตามที่คิด
check(Value) ->
    if
        Value -> "truthy"  % จะ fail ถ้า Value ไม่ใช่ true (atom)!
    end.

% ถูก
check(Value) ->
    if
        Value =:= true -> "it's true";
        Value =:= false -> "it's false"
    end.
% หรือ
check(true) -> "it's true";
check(false) -> "it's false".
```

---

## Lists (ทบทวน + เพิ่มเติม)

### Proper Lists vs Improper Lists

```erlang
% Proper List: ลงท้ายด้วย []
[1, 2, 3].           % = [1 | [2 | [3 | []]]]
[].                  % empty proper list

% Improper List: ลงท้ายด้วยอะไรก็ได้ที่ไม่ใช่ []
[1 | 2].             % improper list
[1, 2 | 3].          % improper list

% ในทางปฏิบัติ ใช้ proper lists เสมอ
% Improper lists ใช้ใน specialized contexts เช่น io:format iolist

% Internal representation ของ list
%     [1, 2, 3]
%     = [1 | [2 | [3 | []]]]
%     ในหน่วยความจำ:
%     1 ──→ 2 ──→ 3 ──→ []
%     (linked list)
```

### List Comprehensions

```erlang
% [Expression || Pattern <- List]
[X * 2 || X <- [1, 2, 3, 4, 5]].
% = [2, 4, 6, 8, 10]

% กับ filter
[X || X <- [1, 2, 3, 4, 5, 6], X rem 2 =:= 0].
% = [2, 4, 6]

% หลาย generators
[{X, Y} || X <- [1, 2, 3], Y <- [a, b]].
% = [{1,a}, {1,b}, {2,a}, {2,b}, {3,a}, {3,b}]

% Binary comprehension
<< <<X*2>> || <<X>> <= <<1, 2, 3>> >>.
% = <<2, 4, 6>>
```

### Deep List Operations

```erlang
% Flatten nested lists
lists:flatten([[1, [2, 3]], [4, [5, 6]]]).
% = [1, 2, 3, 4, 5, 6]

% Group by
group_by(Fun, List) ->
    lists:foldl(fun(X, Map) ->
        Key = Fun(X),
        maps:update_with(Key, fun(V) -> [X | V] end, [X], Map)
    end, #{}, List).

% ตัวอย่าง
Numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].
group_by(fun(X) -> X rem 2 end, Numbers).
% = #{0 => [10, 8, 6, 4, 2], 1 => [9, 7, 5, 3, 1]}

% Chunk list
chunk(_, []) -> [];
chunk(N, List) ->
    {Chunk, Rest} = lists:split(min(N, length(List)), List),
    [Chunk | chunk(N, Rest)].

chunk(3, [1, 2, 3, 4, 5, 6, 7]).
% = [[1, 2, 3], [4, 5, 6], [7]]
```

---

## Tuples (ทบทวน + เพิ่มเติม)

### Tuples vs Lists

```erlang
% Lists:
% - ขนาดเปลี่ยนได้
% - Random access ช้า O(n)
% - เหมาะกับ sequences

% Tuples:
% - ขนาดคงที่
% - Random access เร็ว O(1)
% - เหมาะกับ records/structs

% ดังนั้น:
% ใช้ Tuple: ข้อมูลที่มีโครงสร้างคงที่
Point = {3, 4}.
Color = {255, 128, 0}.
Person = {name, "Alice", age, 30}.

% ใช้ List: sequences ที่อาจเปลี่ยนขนาด
Names = ["Alice", "Bob", "Charlie"].
Numbers = [1, 2, 3, 4, 5].
```

### Tuple Manipulation

```erlang
T = {a, b, c, d, e}.

% Get element (1-indexed)
element(1, T).   % = a
element(3, T).   % = c
element(5, T).   % = e

% Set element (creates new tuple!)
setelement(2, T, z).  % = {a, z, c, d, e}
% T ยังคงเป็น {a, b, c, d, e}

% Size
tuple_size(T).  % = 5

% Convert to/from list
tuple_to_list(T).       % = [a, b, c, d, e]
list_to_tuple([a,b,c]). % = {a, b, c}
```

---

## Maps

### Map Syntax ครบทุกแบบ

```erlang
% สร้าง map
Empty = #{}.
M1 = #{a => 1, b => 2, c => 3}.

% Update syntax (:= = update existing key, => = create or update)
M2 = M1#{a := 10}.     % update a (ถ้า a ไม่มี จะ crash!)
M3 = M1#{d => 4}.      % เพิ่ม d (ถ้า d มีอยู่แล้ว จะ update)

% Pattern matching
#{a := ValueA} = M1.   % ValueA = 1
#{a := A, b := B} = M1.  % A = 1, B = 2

% maps module functions
maps:get(a, M1).           % = 1
maps:get(z, M1, default).  % = default (ถ้าไม่มี key z)
maps:put(d, 4, M1).        % = #{a=>1, b=>2, c=>3, d=>4}
maps:remove(a, M1).        % = #{b=>2, c=>3}
maps:update(a, 100, M1).   % = #{a=>100, b=>2, c=>3}
maps:is_key(a, M1).        % = true
maps:keys(M1).             % = [a, b, c]
maps:values(M1).           % = [1, 2, 3]
maps:size(M1).             % = 3
maps:to_list(M1).          % = [{a,1}, {b,2}, {c,3}]
maps:from_list([{a,1}]).   % = #{a => 1}
maps:merge(M1, #{d=>4}).   % = #{a=>1, b=>2, c=>3, d=>4}

% Map with complex values
Config = #{
    database => #{
        host => <<"localhost">>,
        port => 5432,
        name => <<"mydb">>
    },
    cache => #{
        ttl => 3600,
        max_size => 10000
    }
}.

% Access nested map
DbHost = maps:get(host, maps:get(database, Config)).
% = <<"localhost">>
```

### Map Comprehensions (OTP 26+)

```erlang
% Map comprehension
#{K => V * 2 || K := V <- #{a => 1, b => 2, c => 3}}.
% = #{a => 2, b => 4, c => 6}

% Filter map
#{K => V || K := V <- #{a => 1, b => 2, c => 3}, V > 1}.
% = #{b => 2, c => 3}
```

---

## Binaries และ Bitstrings

### Binary Basics

```erlang
% Binary = sequence of 8-bit integers
B1 = <<1, 2, 3, 4, 5>>.
B2 = <<"Hello, World!">>.

% Binary size
byte_size(B1).          % = 5
byte_size(B2).          % = 13
bit_size(B1).           % = 40

% Binary matching
<<First, Rest/binary>> = B1.
First.   % = 1
Rest.    % = <<2, 3, 4, 5>>

% ดึง bytes
binary_part(B2, 0, 5).  % = <<"Hello">>
binary_part(B2, 7, 5).  % = <<"World">>
```

### Bit Syntax

```erlang
% Format: <<Value:Size/TypeSpecifier>>
% TypeSpecifier: integer, float, binary, bytes, bits, bitstring, utf8, utf16, utf32

% Pack/unpack data
% สร้าง binary ที่มี 2-byte integer
<<1000:16/integer-big>>.    % = <<3, 232>>
<<1000:16/integer-little>>. % = <<232, 3>>

% Unpack
<<Port:16/big>> = <<3, 232>>.
Port.  % = 1000

% Float
<<3.14:64/float>>.          % = <<64, 9, 30, 184, 81, 235, 133, 31>>
<<F:64/float>> = <<64, 9, 30, 184, 81, 235, 133, 31>>.
F.  % = 3.14

% Parsing network packet (TCP header example)
parse_tcp_header(<<SourcePort:16, DestPort:16,
                   SeqNum:32, AckNum:32,
                   DataOffset:4, _Reserved:3, Flags:9,
                   WindowSize:16, Checksum:16, UrgentPtr:16,
                   Rest/binary>>) ->
    #{source_port => SourcePort,
      dest_port => DestPort,
      seq_num => SeqNum,
      ack_num => AckNum,
      flags => Flags,
      window_size => WindowSize,
      payload => Rest}.
```

### Bitstring

```erlang
% Bitstring = ลำดับของ bits (ไม่จำเป็นต้องหาร 8 ลงตัว)
<<1:3, 7:5>>.   % = 1 bit 1, และ 7 ใน 5 bits = <<0b00111111>> = <<63>>

% ตรวจสอบ
is_bitstring(<<1:3>>).  % = true
is_binary(<<1:3>>).     % = false (ไม่ได้หาร 8 ลงตัว)
is_binary(<<1, 2, 3>>). % = true
```

### Binary Operations

```erlang
% Concatenate binaries
B1 = <<"Hello">>.
B2 = <<", ">>.
B3 = <<"World!">>.
<<B1/binary, B2/binary, B3/binary>>.
% = <<"Hello, World!">>

% Split binary
binary:split(<<"Hello, World!">>, <<",">>).
% = [<<"Hello">>, <<" World!">>]

binary:split(<<"a,b,c">>, <<",">>, [global]).
% = [<<"a">>, <<"b">>, <<"c">>]

% Replace in binary
binary:replace(<<"Hello, World!">>, <<"World">>, <<"Erlang">>).
% = <<"Hello, Erlang!">>

% Copy binary (force allocation)
binary:copy(<<"x">>, 5).  % = <<"xxxxx">>

% Convert case
string:uppercase(<<"hello">>).  % = <<"HELLO">> (OTP 20+)
string:lowercase(<<"HELLO">>).  % = <<"hello">>
```

---

## Strings

### List Strings vs Binary Strings

```erlang
% List String (ดั้งเดิม, ใช้สำหรับ compatibility)
ListStr = "Hello, World!".
% = [72, 101, 108, 108, 111, 44, 32, 87, 111, 114, 108, 100, 33]

% Binary String (แนะนำสำหรับ production)
BinStr = <<"Hello, World!">>.
% = <<72, 101, 108, 108, 111, 44, 32, 87, 111, 114, 108, 100, 33>>

% แปลงระหว่างกัน
list_to_binary("Hello").  % = <<"Hello">>
binary_to_list(<<"Hello">>).  % = "Hello"
```

### Unicode Support

```erlang
% Erlang รองรับ Unicode (UTF-8) ผ่าน binary
Greeting = <<"สวัสดี">>.  % UTF-8 bytes

% Unicode string operations
unicode:characters_to_binary("สวัสดี").
unicode:characters_to_list(<<"สวัสดี">>).

% String module รองรับ Unicode
string:length("สวัสดี").  % = 6 (grapheme clusters)
string:length(<<"สวัสดี">>).  % = 6

% Codepoints
[H|_] = unicode:characters_to_list("A").
H.  % = 65

% Binary ที่มี Unicode
<<H/utf8, _/binary>> = <<"สวัสดี">>.
% H = 3626 (codepoint ของ ส)
```

---

## PIDs (Process IDs)

```erlang
% PID คือ identifier ของ process
MyPid = self().
% = <0.108.0> (format: <Node.Serial.Creation>)

% Spawn process และได้ PID
Pid = spawn(fun() -> receive stop -> ok end end).

% ส่ง message
Pid ! hello.

% ตรวจสอบ
is_pid(Pid).     % = true
is_pid(hello).   % = false

% PID เป็น string
pid_to_list(self()).    % = "<0.108.0>"
list_to_pid("<0.108.0>").  % = <0.108.0>

% ตรวจสอบว่า process ยังมีชีวิตอยู่
erlang:is_process_alive(Pid).  % = true หรือ false
```

---

## Ports

```erlang
% Port คือ interface กับ external programs/resources
% เช่น OS process, network socket, device

% ดู ports ที่เปิดอยู่
erlang:ports().

% Port ที่ถูกสร้างจาก open_port
Port = open_port({spawn, "cat"}, [binary]).

% ส่งข้อมูลไปที่ port
port_command(Port, <<"Hello">>).

% ตรวจสอบ
is_port(Port).  % = true

% Close port
port_close(Port).
```

---

## References

```erlang
% Reference เป็น unique value ที่สร้างด้วย make_ref()
Ref = make_ref().
% = #Ref<0.1234567890.123456789.123456>

% References ไม่ซ้ำกันในระบบ Erlang
Ref1 = make_ref().
Ref2 = make_ref().
Ref1 =:= Ref2.  % = false เสมอ!

% ใช้ Reference เป็น unique identifier
send_and_wait(Pid, Message) ->
    Ref = make_ref(),
    Pid ! {self(), Ref, Message},
    receive
        {Ref, Response} -> Response
    after 5000 ->
        timeout
    end.
```

---

## Funs (Anonymous Functions)

```erlang
% Fun syntax
Square = fun(X) -> X * X end.
Square(5).  % = 25

% Multi-clause fun
Classify = fun
    (X) when X > 0 -> positive;
    (X) when X < 0 -> negative;
    (0) -> zero
end.
Classify(5).   % = positive
Classify(-3).  % = negative
Classify(0).   % = zero

% Fun ที่รับ fun เป็น argument
apply_twice(F, X) ->
    F(F(X)).

apply_twice(fun(X) -> X * 2 end, 3).  % = 12

% Closure - fun เข้าถึงตัวแปรใน scope
make_adder(N) ->
    fun(X) -> X + N end.

Add5 = make_adder(5).
Add5(10).  % = 15
Add5(20).  % = 25

% Named fun (สำหรับ recursion)
Factorial = fun F(0) -> 1;
               F(N) -> N * F(N-1)
           end.
Factorial(5).  % = 120
```

### Fun References

```erlang
% อ้างอิง function ที่มีชื่อ
% Format: fun Module:Function/Arity

% Local function
Double = fun double/1.
double(X) -> X * 2.
Double(5).  % = 10

% Module function
Sort = fun lists:sort/1.
Sort([3, 1, 2]).  % = [1, 2, 3]

% Current module function
Squares = lists:map(fun square/1, [1, 2, 3]).
square(X) -> X * X.
```

---

## Type Checking Functions

```erlang
% ตรวจสอบประเภทข้อมูล
is_atom(hello).         % true
is_atom("hello").       % false
is_atom(42).            % false

is_integer(42).         % true
is_integer(42.0).       % false
is_integer(42.0).       % false

is_float(3.14).         % true
is_float(3).            % false

is_number(42).          % true
is_number(3.14).        % true
is_number("42").        % false

is_list([1, 2, 3]).     % true
is_list("hello").       % true (string เป็น list!)
is_list([]).            % true

is_tuple({1, 2}).       % true
is_tuple([1, 2]).       % false

is_map(#{a => 1}).      % true

is_binary(<<"hello">>). % true
is_binary("hello").     % false

is_bitstring(<<1:3>>).  % true
is_binary(<<1:3>>).     % false

is_boolean(true).       % true
is_boolean(false).      % true
is_boolean(1).          % false

is_function(fun() -> ok end).      % true
is_function(fun() -> ok end, 0).   % true (arity 0)
is_function(fun(X) -> X end, 1).   % true (arity 1)

is_pid(self()).         % true
is_port(Port).          % true (ถ้า Port เป็น port)
is_reference(make_ref()).  % true

is_nil([]).             % ไม่มีใน Erlang, ใช้ =:= [] แทน
```

---

## Type Conversion

```erlang
% Integer conversions
integer_to_list(42).        % = "42"
integer_to_list(42, 16).    % = "2A" (base 16)
integer_to_binary(42).      % = <<"42">>
list_to_integer("42").      % = 42
list_to_integer("2A", 16).  % = 42 (base 16)
binary_to_integer(<<"42">>).% = 42

% Float conversions
float_to_list(3.14).                   % = "3.14000000000000012434e+00"
float_to_list(3.14, [{decimals, 2}]).  % = "3.14"
float_to_binary(3.14).                 % = <<"3.14000000000000012434e+00">>
list_to_float("3.14").                 % = 3.14
binary_to_float(<<"3.14">>).          % = 3.14

% Atom conversions
atom_to_list(hello).        % = "hello"
atom_to_binary(hello).      % = <<"hello">>
list_to_atom("hello").      % = hello (CAREFUL: creates new atom)
binary_to_atom(<<"hello">>).% = hello (CAREFUL)
erlang:list_to_existing_atom("hello").   % safe version

% Number conversions
float(42).     % = 42.0 (integer to float)
round(3.7).    % = 4    (float to integer)
trunc(3.7).    % = 3    (float to integer, truncate)
floor(3.7).    % = 3    (float to integer, round down)
ceiling(3.2).  % = 4    (float to integer, round up)

% List/Binary conversions
list_to_binary([72, 101, 108, 108, 111]).  % = <<"Hello">>
binary_to_list(<<"Hello">>).               % = [72, 101, 108, 108, 111]

% Tuple/List conversions
tuple_to_list({a, b, c}).  % = [a, b, c]
list_to_tuple([a, b, c]).  % = {a, b, c}
```

---

## Term Ordering

Erlang กำหนดลำดับการเปรียบเทียบระหว่าง types ต่างกัน:

```
ลำดับจากน้อยไปมาก:
numbers < atoms < references < funs < ports < pids < tuples < maps < nil < lists < bit strings

number:  integers, floats
nil:     []
```

```erlang
% ตัวอย่างการเปรียบเทียบข้าม types
1 < 1.0.           % false (1 =:= 1.0 ด้วย == แต่ =/= ด้วย =:=)
1 < hello.         % true (number < atom)
hello < {a}.       % true (atom < tuple)
{a} < [].          % true (tuple < nil = [])
[] < [a].          % true ([] < list ที่มี elements)
[a] < <<"a">>.     % true (list < bitstring/binary)

% Useful สำหรับ sorting mixed types
lists:sort([<<"a">>, [b], {c}, d, 1]).
% = [1, d, {c}, [b], <<"a">>]
```

---

## สรุป Part 4

| Data Type | Syntax | ใช้เมื่อ |
|-----------|--------|---------|
| Atom | `hello`, `ok`, `true` | Constants, tags, boolean |
| Integer | `42`, `16#FF`, `2#1010` | Numbers, counters |
| Float | `3.14`, `1.0e10` | Decimals, scientific |
| List | `[1, 2, 3]`, `"str"` | Sequences |
| Tuple | `{a, b, c}` | Fixed-size records |
| Map | `#{k => v}` | Key-value data |
| Binary | `<<"hello">>`, `<<1,2,3>>` | Bytes, efficient strings |
| Fun | `fun(X) -> X end` | First-class functions |
| PID | `self()` | Process identifiers |
| Reference | `make_ref()` | Unique tokens |

### ก้าวต่อไป

ใน **Part 5** เราจะเรียน Variables และ Pattern Matching อย่างละเอียด:
- Pattern Matching syntax ทั้งหมด
- Destructuring
- Guards ใน Pattern Matching
- ตัวอย่างการใช้งานจริง

---

## แบบฝึกหัด Part 4

```erlang
% แบบฝึกหัดที่ 1: Type detection
% เขียน function describe_type/1 ที่รับ term ใดๆ
% และ return string อธิบาย type ของมัน

describe_type(42) -> "integer";
describe_type(3.14) -> "float";
% ... เพิ่ม cases อื่นๆ

% แบบฝึกหัดที่ 2: Data conversion
% เขียน function safe_to_integer/1 ที่:
% - รับ integer, float, string, หรือ binary
% - แปลงเป็น integer
% - return {ok, Integer} หรือ {error, Reason}

% แบบฝึกหัดที่ 3: Map operations
% สร้าง map สำหรับ inventory system:
% #{sku => "WIDGET-001", name => "Widget", price => 9.99, quantity => 100}
% เขียน functions สำหรับ:
% - add_item/2 - เพิ่ม quantity
% - remove_item/2 - ลด quantity
% - apply_discount/2 - ลด price ด้วย percentage
```

---

*[← Part 3: Basic Syntax](part_03_basic_syntax.md) | [Part 5: Variables and Pattern Matching →](part_05_variables_pattern_matching.md)*
