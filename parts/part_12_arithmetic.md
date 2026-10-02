# Part 12: Arithmetic และ Math Operations

## สารบัญ
1. [Integer Arithmetic](#integer-arithmetic)
2. [Float Arithmetic](#float-arithmetic)
3. [Integer Operations พิเศษ](#integer-operations-พิเศษ)
4. [Math Module](#math-module)
5. [Bitwise Operations](#bitwise-operations)
6. [Number Formatting](#number-formatting)
7. [Arbitrary Precision](#arbitrary-precision)
8. [ตัวอย่างจริง: Calculator และ Statistics](#ตัวอย่างจริง-calculator-และ-statistics)

---

## Integer Arithmetic

```erlang
%% Operators พื้นฐาน
5 + 3.    %% = 8
5 - 3.    %% = 2
5 * 3.    %% = 15
5 / 3.    %% = 1.6666... (always returns float!)
5 div 3.  %% = 1 (integer division)
5 rem 3.  %% = 2 (remainder/modulo)

%% หมายเหตุ: / ใน Erlang ให้ผลเป็น float เสมอ
10 / 2.   %% = 5.0 (ไม่ใช่ 5!)
10 div 2. %% = 5 (integer division)

%% Negative numbers
-5 + 3.    %% = -2
-5 div 2.  %% = -2 (truncates toward zero)
-5 rem 2.  %% = -1 (sign matches dividend)
5 rem -2.  %% = 1  (sign matches dividend)

%% Exponentiation (ไม่มี ** operator!)
%% ต้องใช้ math:pow หรือ recursive
math:pow(2, 10).        %% = 1024.0 (float)
erlang:integer_pow(2, 10).  %% = 1024 (integer, OTP 26+)
trunc(math:pow(2, 10)). %% = 1024 (convert float to int)

%% Integer literals ใน bases ต่างๆ
255.         %% decimal
16#FF.       %% hex = 255
8#377.       %% octal = 255
2#11111111.  %% binary = 255

%% ตัวเลขใหญ่
1000000000000000000.            %% 10^18 (no overflow!)
factorial(100).                 %% ตัวเลขขนาดใหญ่มาก
```

---

## Float Arithmetic

```erlang
%% Float literals
3.14.
-2.5.
1.0e10.     %% = 10000000000.0
1.5e-3.     %% = 0.0015

%% Float operators
3.14 + 1.0.    %% = 4.140000000000000
3.14 * 2.0.    %% = 6.28
3.14 / 2.0.    %% = 1.57

%% Precision issues
0.1 + 0.2.     %% = 0.30000000000000004 (floating point!)
0.1 + 0.2 == 0.3.  %% = false

%% วิธีเปรียบเทียบ float
abs(0.1 + 0.2 - 0.3) < 1.0e-10.  %% = true

epsilon_equal(A, B) ->
    abs(A - B) < 1.0e-9.

%% Conversion
trunc(3.7).      %% = 3 (truncate, ไม่ปัดเศษ)
round(3.5).      %% = 4 (round)
round(3.4).      %% = 3
floor(3.9).      %% = 3 (round down)
ceil(3.1).       %% = 4 (round up)

float(42).       %% = 42.0 (int to float)
float_to_list(3.14, [{decimals, 2}]).   %% = "3.14"

%% Float formatting
io:format("~f~n", [3.14]).       %% = 3.140000
io:format("~.2f~n", [3.14]).     %% = 3.14
io:format("~e~n", [3.14]).       %% = 3.140000e+00
io:format("~g~n", [3.14]).       %% = 3.14
io:format("~.4g~n", [3.14159]).  %% = 3.142
```

---

## Integer Operations พิเศษ

```erlang
%% abs - absolute value
abs(-5).    %% = 5
abs(5).     %% = 5

%% max / min
max(3, 7).    %% = 7
min(3, 7).    %% = 3

%% erlang:min/max กับ any type
max(a, b).    %% = b (atom comparison)
max("hello", "world").  %% = "world"

%% Number checks
is_integer(42).    %% = true
is_float(3.14).    %% = true
is_number(42).     %% = true (integer or float)
is_number(hello).  %% = false

%% erlang math functions
erlang:phash2(Term, Range).  %% hash function (0 to Range-1)
erlang:phash2(some_term, 100).  %% = some number 0-99

%% gcd (ไม่มี built-in แต่ implement ได้)
gcd(A, 0) -> A;
gcd(A, B) -> gcd(B, A rem B).

gcd(48, 18).  %% = 6

%% lcm
lcm(A, B) -> (A * B) div gcd(A, B).
lcm(4, 6).   %% = 12
```

---

## Math Module

```erlang
%% Trigonometric functions (radians)
math:sin(math:pi() / 2).   %% = 1.0
math:cos(0.0).             %% = 1.0
math:tan(math:pi() / 4).   %% = 1.0

math:asin(1.0).            %% = 1.5707... (pi/2)
math:acos(1.0).            %% = 0.0
math:atan(1.0).            %% = 0.7853... (pi/4)
math:atan2(1.0, 1.0).      %% = 0.7853... (y, x)

%% Hyperbolic
math:sinh(1.0).            %% = 1.1752...
math:cosh(0.0).            %% = 1.0
math:tanh(0.5).            %% = 0.4621...

%% Exponential and logarithmic
math:exp(1.0).             %% = 2.7182... (e^1)
math:exp(2.0).             %% = 7.3890... (e^2)

math:log(math:exp(1.0)).   %% = 1.0 (natural log)
math:log(100.0).           %% = 4.6051...
math:log2(8.0).            %% = 3.0
math:log10(1000.0).        %% = 3.0

%% Power and root
math:pow(2.0, 8.0).        %% = 256.0
math:sqrt(9.0).            %% = 3.0
math:sqrt(2.0).            %% = 1.4142...

%% Constants
math:pi().                 %% = 3.14159265358979...

%% เปลี่ยน degrees เป็น radians
to_radians(Deg) -> Deg * math:pi() / 180.
to_degrees(Rad) -> Rad * 180 / math:pi().

math:sin(to_radians(90)).  %% = 1.0
to_degrees(math:pi()).     %% = 180.0
```

---

## Bitwise Operations

```erlang
%% ใช้กับ integer เท่านั้น

%% AND
5 band 3.    %% = 1   (101 & 011 = 001)

%% OR
5 bor 3.     %% = 7   (101 | 011 = 111)

%% XOR
5 bxor 3.    %% = 6   (101 ^ 011 = 110)

%% NOT (bitwise complement)
bnot 5.      %% = -6  (~101 = ...11111010)

%% Left shift
1 bsl 3.     %% = 8   (1 << 3)
5 bsl 2.     %% = 20  (101 << 2 = 10100)

%% Right shift
8 bsr 1.     %% = 4   (1000 >> 1 = 100)
20 bsr 2.    %% = 5   (10100 >> 2 = 101)

%% ตัวอย่าง: Flags
-define(FLAG_READ,    1 bsl 0).  %% = 1
-define(FLAG_WRITE,   1 bsl 1).  %% = 2
-define(FLAG_EXECUTE, 1 bsl 2).  %% = 4

set_flag(Flags, Flag) -> Flags bor Flag.
clear_flag(Flags, Flag) -> Flags band (bnot Flag).
has_flag(Flags, Flag) -> (Flags band Flag) =:= Flag.

Perms = 0.
Perms1 = set_flag(Perms, ?FLAG_READ).     %% = 1
Perms2 = set_flag(Perms1, ?FLAG_WRITE).   %% = 3
has_flag(Perms2, ?FLAG_READ).             %% = true
has_flag(Perms2, ?FLAG_EXECUTE).          %% = false
Perms3 = clear_flag(Perms2, ?FLAG_WRITE). %% = 1

%% ตัวอย่าง: Pack/Unpack data
pack_rgb(R, G, B) ->
    (R bsl 16) bor (G bsl 8) bor B.

unpack_rgb(Color) ->
    R = (Color bsr 16) band 16#FF,
    G = (Color bsr 8) band 16#FF,
    B = Color band 16#FF,
    {R, G, B}.

Color = pack_rgb(255, 128, 0).   %% = 16744448
unpack_rgb(Color).               %% = {255, 128, 0}

%% ตัวอย่าง: Check power of 2
is_power_of_two(N) when N > 0 ->
    N band (N - 1) =:= 0.

is_power_of_two(4).   %% = true
is_power_of_two(8).   %% = true
is_power_of_two(6).   %% = false
```

---

## Number Formatting

```erlang
%% io_lib:format สำหรับ formatted strings
io_lib:format("~b", [255]).           %% = "11111111" (binary)
io_lib:format("~o", [255]).           %% = "377" (octal)
io_lib:format("~.16b", [255]).        %% = "ff" (lowercase hex)
io_lib:format("~16.16B", [255]).      %% = "000000FF" (padded hex)

%% Convert to different bases
integer_to_list(255, 2).   %% = "11111111"
integer_to_list(255, 8).   %% = "377"
integer_to_list(255, 16).  %% = "FF"
integer_to_list(255, 36).  %% = "73" (base 36)

list_to_integer("FF", 16).     %% = 255
list_to_integer("11111111", 2). %% = 255

%% Number padding
pad_number(N, Width) ->
    Str = integer_to_list(N),
    Pad = lists:duplicate(max(0, Width - length(Str)), $0),
    list_to_binary([Pad, Str]).

pad_number(42, 5).   %% = <<"00042">>
pad_number(12345, 3). %% = <<"12345">> (no truncation)

%% Format with separators
format_number(N) when N >= 1000000 ->
    format_with_sep(N, 3, <<",">>);
format_number(N) ->
    integer_to_binary(N).

format_with_sep(N, GroupSize, Sep) ->
    Str = integer_to_list(N),
    insert_separators(lists:reverse(Str), GroupSize, Sep, []).

insert_separators([], _, _, Acc) ->
    list_to_binary(lists:reverse(Acc));
insert_separators(Digits, GroupSize, Sep, Acc) ->
    {Group, Rest} = lists:split(min(GroupSize, length(Digits)), Digits),
    NewAcc = case {Acc, Rest} of
        {[], _} -> lists:reverse(Group);
        {_, []} -> lists:reverse(Group) ++ Acc;
        _ -> lists:reverse(Group) ++ binary_to_list(Sep) ++ Acc
    end,
    insert_separators(Rest, GroupSize, Sep, NewAcc).

format_with_sep(1234567, 3, <<",">>).  %% = <<"1,234,567">>
```

---

## Arbitrary Precision

```erlang
%% Erlang integers ไม่มี overflow!
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N - 1).

factorial(10).   %% = 3628800
factorial(20).   %% = 2432902008176640000
factorial(50).   %% = 30414093201713378043612608166979581188299763898377856000000000000
factorial(100).  %% = ตัวเลข 158 หลัก!

%% Fibonacci with large numbers
fib(0) -> 0;
fib(1) -> 1;
fib(N) -> fib(N-1) + fib(N-2).

fib(100).  %% = 354224848179261915075

%% Power of large numbers
math:pow(2, 64).     %% = 1.8446744073709552e19 (float, approximate)
1 bsl 64.            %% = 18446744073709551616 (exact integer!)

%% ความยาวของตัวเลข
number_length(N) when N < 0 ->
    number_length(-N) + 1;  %% count minus sign
number_length(0) -> 1;
number_length(N) ->
    length(integer_to_list(N)).

number_length(factorial(100)).  %% = 158

%% Modular arithmetic
pow_mod(Base, Exp, Mod) ->
    pow_mod(Base, Exp, Mod, 1).

pow_mod(_, 0, _, Acc) -> Acc;
pow_mod(Base, Exp, Mod, Acc) when Exp rem 2 == 1 ->
    pow_mod(Base * Base rem Mod, Exp div 2, Mod, Acc * Base rem Mod);
pow_mod(Base, Exp, Mod, Acc) ->
    pow_mod(Base * Base rem Mod, Exp div 2, Mod, Acc).

pow_mod(2, 10, 1000).   %% = 24 (2^10 mod 1000)
pow_mod(2, 100, 100).   %% = 76
```

---

## ตัวอย่างจริง: Calculator และ Statistics

```erlang
%% ไฟล์: math_utils.erl
-module(math_utils).
-export([
    mean/1,
    median/1,
    mode/1,
    variance/1,
    std_dev/1,
    percentile/2,
    sum_of_squares/1,
    normalize/1,
    clamp/3,
    lerp/3
]).

%% Mean (average)
-spec mean([number()]) -> float().
mean([]) -> error(empty_list);
mean(List) ->
    lists:sum(List) / length(List).

%% Median
-spec median([number()]) -> number().
median([]) -> error(empty_list);
median(List) ->
    Sorted = lists:sort(List),
    N = length(Sorted),
    case N rem 2 of
        0 ->
            Mid = N div 2,
            (lists:nth(Mid, Sorted) + lists:nth(Mid + 1, Sorted)) / 2;
        1 ->
            lists:nth((N + 1) div 2, Sorted)
    end.

%% Mode (most frequent value)
-spec mode([term()]) -> term().
mode([]) -> error(empty_list);
mode(List) ->
    Freq = lists:foldl(
        fun(X, Acc) -> maps:update_with(X, fun(V) -> V + 1 end, 1, Acc) end,
        #{},
        List
    ),
    {MaxVal, _} = maps:fold(
        fun(K, V, {_, MaxV} = Acc) ->
            case V > MaxV of true -> {K, V}; false -> Acc end
        end,
        {hd(List), 0},
        Freq
    ),
    MaxVal.

%% Variance (population variance)
-spec variance([number()]) -> float().
variance([]) -> error(empty_list);
variance(List) ->
    M = mean(List),
    SumSq = lists:foldl(fun(X, Acc) -> Acc + (X - M) * (X - M) end, 0, List),
    SumSq / length(List).

%% Standard Deviation
-spec std_dev([number()]) -> float().
std_dev(List) ->
    math:sqrt(variance(List)).

%% Percentile (0-100)
-spec percentile([number()], 0..100) -> float().
percentile([], _) -> error(empty_list);
percentile(List, P) ->
    Sorted = lists:sort(List),
    N = length(Sorted),
    Index = P / 100 * (N - 1),
    Low = trunc(Index),
    High = Low + 1,
    Fraction = Index - Low,
    LowVal = lists:nth(Low + 1, Sorted),
    case High >= N of
        true -> LowVal * 1.0;
        false ->
            HighVal = lists:nth(High + 1, Sorted),
            LowVal + Fraction * (HighVal - LowVal)
    end.

%% Sum of squares
-spec sum_of_squares([number()]) -> number().
sum_of_squares(List) ->
    lists:foldl(fun(X, Acc) -> Acc + X * X end, 0, List).

%% Normalize list to [0, 1]
-spec normalize([number()]) -> [float()].
normalize([]) -> [];
normalize(List) ->
    Min = lists:min(List),
    Max = lists:max(List),
    Range = Max - Min,
    case Range of
        0 -> [0.5 || _ <- List];
        _ -> [(X - Min) / Range || X <- List]
    end.

%% Clamp value between Min and Max
-spec clamp(number(), number(), number()) -> number().
clamp(Value, Min, Max) ->
    max(Min, min(Max, Value)).

%% Linear interpolation
-spec lerp(number(), number(), float()) -> float().
lerp(A, B, T) ->
    A + T * (B - A).
```

```erlang
%% ทดสอบ math_utils
Data = [4, 7, 13, 2, 1, 9, 3, 8, 5, 6].

math_utils:mean(Data).       %% = 5.8
math_utils:median(Data).     %% = 5.5
math_utils:mode([1,2,2,3,3,3]).  %% = 3
math_utils:variance(Data).   %% = 12.96
math_utils:std_dev(Data).    %% = 3.6
math_utils:percentile(Data, 25).   %% = 2.25 (Q1)
math_utils:percentile(Data, 75).   %% = 8.25 (Q3)
math_utils:normalize(Data).  %% = [0.25, 0.5, 1.0, 0.083, 0.0, 0.667, 0.167, 0.583, 0.333, 0.417]
math_utils:clamp(150, 0, 100).   %% = 100
math_utils:clamp(-10, 0, 100).   %% = 0
math_utils:lerp(0, 100, 0.25).   %% = 25.0
```

---

## สรุป Part 12

| Operation | Syntax | Notes |
|-----------|--------|-------|
| Add/Sub/Mul | `+`, `-`, `*` | Standard |
| Float div | `/` | Always returns float |
| Integer div | `div` | Truncates toward zero |
| Remainder | `rem` | Sign matches dividend |
| Bitwise AND | `band` | Integers only |
| Bitwise OR | `bor` | Integers only |
| Left shift | `bsl` | Integers only |
| Right shift | `bsr` | Integers only |
| Square root | `math:sqrt/1` | Returns float |
| Power | `math:pow/2` | Returns float |
| Pi | `math:pi/0` | = 3.14159... |
| Round | `round/1` | Returns integer |
| Truncate | `trunc/1` | Returns integer |
| Floor | `floor/1` | Returns integer |
| Ceil | `ceil/1` | Returns integer |

---

*[← Part 11: Strings and Binaries](part_11_strings_binaries.md) | [Part 13: Control Flow →](part_13_control_flow.md)*
