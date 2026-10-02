# Part 8: Lists และการดำเนินการกับ Lists

## สารบัญ
1. [List Basics ทบทวน](#list-basics-ทบทวน)
2. [lists Module Functions](#lists-module-functions)
3. [List Comprehensions](#list-comprehensions)
4. [Recursive List Processing](#recursive-list-processing)
5. [Efficient List Operations](#efficient-list-operations)
6. [IOLists](#iolists)
7. [Sorted Lists](#sorted-lists)
8. [Set Operations](#set-operations)
9. [Association Lists (Proplists)](#association-lists-proplists)
10. [ตัวอย่างจริง](#ตัวอย่างจริง)
11. [สรุป](#สรุป)

---

## List Basics ทบทวน

```erlang
%% List เป็น singly-linked list
%% [1, 2, 3] = [1 | [2 | [3 | []]]]

%% Internal representation:
%%    ┌───┬───┐    ┌───┬───┐    ┌───┬───┐
%%    │ 1 │ ──┼──→ │ 2 │ ──┼──→ │ 3 │ [] │
%%    └───┴───┘    └───┴───┘    └───┴───┘

%% Head/Tail pattern
[H|T] = [1, 2, 3, 4, 5].
%% H = 1
%% T = [2, 3, 4, 5]

%% Prepend O(1) - เร็วมาก
NewList = [0 | [1, 2, 3]].  % [0, 1, 2, 3]

%% Append O(n) - ช้ากว่า
OldList = [1, 2, 3] ++ [4, 5].  % [1, 2, 3, 4, 5]

%% Length O(n) - ต้องวิ่งทั้ง list
length([1, 2, 3, 4, 5]).  % = 5
```

---

## lists Module Functions

### Basics

```erlang
%% lists:append/1, lists:append/2
lists:append([[1,2], [3,4], [5,6]]).   % = [1,2,3,4,5,6]
lists:append([1,2], [3,4]).            % = [1,2,3,4]

%% lists:reverse/1, lists:reverse/2
lists:reverse([1,2,3,4,5]).            % = [5,4,3,2,1]
lists:reverse([1,2,3], [4,5]).         % = [3,2,1,4,5]

%% lists:member/2
lists:member(3, [1,2,3,4,5]).         % = true
lists:member(9, [1,2,3,4,5]).         % = false

%% lists:nth/2, lists:last/1
lists:nth(3, [a,b,c,d,e]).            % = c (1-indexed)
lists:last([1,2,3,4,5]).              % = 5

%% lists:droplast/1
lists:droplast([1,2,3,4,5]).          % = [1,2,3,4]

%% lists:flatten/1, lists:flatten/2
lists:flatten([[1,[2,3]],[4,[5,6]]]). % = [1,2,3,4,5,6]
lists:flatten([[1,2],[3,4]], [5,6]).  % = [1,2,3,4,5,6]

%% lists:flatlength/1 - เร็วกว่า length(flatten(L))
lists:flatlength([[1,[2,3]],[4]]).    % = 4
```

### Map, Filter, Fold

```erlang
%% lists:map/2 - transform each element
lists:map(fun(X) -> X * X end, [1,2,3,4,5]).
%% = [1, 4, 9, 16, 25]

lists:map(fun(X) -> {X, X*X} end, [1,2,3]).
%% = [{1,1}, {2,4}, {3,9}]

%% lists:filtermap/2 - filter and transform in one pass
lists:filtermap(
    fun(X) when X > 2 -> {true, X * 10};
       (_) -> false
    end,
    [1,2,3,4,5]).
%% = [30, 40, 50]

%% lists:filter/2 - keep elements that satisfy predicate
lists:filter(fun(X) -> X rem 2 =:= 0 end, [1,2,3,4,5,6]).
%% = [2, 4, 6]

%% lists:foldl/3 - fold left (accumulate from left to right)
lists:foldl(fun(X, Acc) -> Acc + X end, 0, [1,2,3,4,5]).
%% = 15

lists:foldl(fun(X, Acc) -> [X*2 | Acc] end, [], [1,2,3]).
%% = [6, 4, 2] (reversed!)

%% lists:foldr/3 - fold right (accumulate from right to left)
lists:foldr(fun(X, Acc) -> [X*2 | Acc] end, [], [1,2,3]).
%% = [2, 4, 6] (correct order)

%% lists:foreach/2 - side effects only, returns ok
lists:foreach(fun(X) -> io:format("~p~n", [X]) end, [1,2,3]).
%% prints 1, 2, 3 and returns ok
```

### Search Functions

```erlang
%% lists:keyfind/3 - ค้นหาใน list ของ tuples
lists:keyfind(alice, 1, [{alice, 30}, {bob, 25}]).
%% = {alice, 30}

lists:keyfind(charlie, 1, [{alice, 30}, {bob, 25}]).
%% = false

%% lists:keymember/3 - ตรวจว่ามีอยู่ไหม
lists:keymember(alice, 1, [{alice, 30}, {bob, 25}]).
%% = true

%% lists:keydelete/3 - ลบ element ที่ match
lists:keydelete(alice, 1, [{alice, 30}, {bob, 25}, {alice, 35}]).
%% = [{bob, 25}, {alice, 35}] (ลบแค่ตัวแรกที่เจอ)

%% lists:keyreplace/4 - แทนที่ด้วย tuple ใหม่
lists:keyreplace(alice, 1, [{alice, 30}, {bob, 25}], {alice, 31}).
%% = [{alice, 31}, {bob, 25}]

%% lists:keysort/2 - sort by key position
lists:keysort(2, [{alice, 30}, {charlie, 25}, {bob, 35}]).
%% = [{charlie, 25}, {alice, 30}, {bob, 35}]

%% lists:keystore/4 - update or add
lists:keystore(alice, 1, [{bob, 25}], {alice, 30}).
%% = [{bob, 25}, {alice, 30}]

%% lists:keytake/3 - ดึงออก + return ที่เหลือ
lists:keytake(alice, 1, [{alice, 30}, {bob, 25}]).
%% = {value, {alice, 30}, [{bob, 25}]}
```

### Sort Functions

```erlang
%% lists:sort/1 - sort natural order
lists:sort([3, 1, 4, 1, 5, 9, 2, 6]).
%% = [1, 1, 2, 3, 4, 5, 6, 9]

%% lists:sort/2 - sort with comparator
lists:sort(fun(A, B) -> A > B end, [3, 1, 4, 1, 5]).
%% = [5, 4, 3, 1, 1] (descending)

%% lists:usort/1 - sort + remove duplicates
lists:usort([3, 1, 4, 1, 5, 9, 2, 6, 5]).
%% = [1, 2, 3, 4, 5, 6, 9]

%% lists:usort/2 - unique sort with comparator
lists:usort(fun(A, B) -> A =< B end, [3, 1, 1, 2]).
%% = [1, 2, 3]

%% lists:msort/1 - merge sort (เสถียรกว่า sort/1)
lists:msort([3, 1, 2, 1]).  % = [1, 1, 2, 3]
```

### Partition Functions

```erlang
%% lists:partition/2 - แบ่งเป็น 2 lists
lists:partition(fun(X) -> X > 3 end, [1,2,3,4,5,6]).
%% = {[4,5,6], [1,2,3]}

%% lists:splitwith/2 - แบ่งตาม predicate จนกว่า false
lists:splitwith(fun(X) -> X < 4 end, [1,2,3,4,5,6]).
%% = {[1,2,3], [4,5,6]}

%% lists:split/2 - แบ่งที่ตำแหน่ง N
lists:split(3, [1,2,3,4,5,6]).
%% = {[1,2,3], [4,5,6]}

%% lists:sublist/2, lists:sublist/3
lists:sublist([1,2,3,4,5], 3).      % = [1,2,3]
lists:sublist([1,2,3,4,5], 2, 3).   % = [2,3,4]

%% lists:takewhile/2
lists:takewhile(fun(X) -> X < 4 end, [1,2,3,4,5]).
%% = [1,2,3]

%% lists:dropwhile/2
lists:dropwhile(fun(X) -> X < 4 end, [1,2,3,4,5]).
%% = [4,5]
```

### Zip/Unzip

```erlang
%% lists:zip/2 - รวม 2 lists เป็น list of pairs
lists:zip([1,2,3], [a,b,c]).
%% = [{1,a}, {2,b}, {3,c}]

%% lists:zip3/3 - รวม 3 lists
lists:zip3([1,2,3], [a,b,c], [x,y,z]).
%% = [{1,a,x}, {2,b,y}, {3,c,z}]

%% lists:unzip/1 - แยก list of pairs
lists:unzip([{1,a}, {2,b}, {3,c}]).
%% = {[1,2,3], [a,b,c]}

%% lists:unzip3/1
lists:unzip3([{1,a,x}, {2,b,y}]).
%% = {[1,2], [a,b], [x,y]}

%% lists:zipwith/3 - zip และ apply function
lists:zipwith(fun(X, Y) -> X + Y end, [1,2,3], [4,5,6]).
%% = [5, 7, 9]
```

### Miscellaneous

```erlang
%% lists:duplicate/2 - สร้าง list ของ element ซ้ำ
lists:duplicate(5, hello).
%% = [hello, hello, hello, hello, hello]

%% lists:seq/2, lists:seq/3 - สร้าง sequence
lists:seq(1, 5).           % = [1, 2, 3, 4, 5]
lists:seq(1, 10, 2).       % = [1, 3, 5, 7, 9]
lists:seq(10, 1, -1).      % = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]

%% lists:sum/1, lists:max/1, lists:min/1
lists:sum([1,2,3,4,5]).    % = 15
lists:max([3,1,4,1,5,9]).  % = 9
lists:min([3,1,4,1,5,9]).  % = 1

%% lists:concat/1 - แปลง list ของ terms เป็น string
lists:concat([hello, " ", world, " ", 123]).
%% = "hello world 123"

%% lists:flatten + map
lists:flatmap(fun(X) -> [X, X*2] end, [1,2,3]).
%% = [1, 2, 2, 4, 3, 6]

%% lists:enumerate/1 (OTP 25+)
lists:enumerate([a, b, c]).
%% = [{1,a}, {2,b}, {3,c}]

lists:enumerate(10, [a, b, c]).
%% = [{10,a}, {11,b}, {12,c}]
```

---

## List Comprehensions

```erlang
%% Basic syntax
%% [ Expression || Generator, Filter, Generator, ... ]

%% Simple map
[X * 2 || X <- [1,2,3,4,5]].
%% = [2, 4, 6, 8, 10]

%% With filter
[X || X <- [1,2,3,4,5,6,7,8,9,10], X rem 2 =:= 0].
%% = [2, 4, 6, 8, 10]

%% Multiple generators (cartesian product)
[{X, Y} || X <- [1,2,3], Y <- [a,b]].
%% = [{1,a},{1,b},{2,a},{2,b},{3,a},{3,b}]

%% Nested generators
Matrix = [[1,2,3],[4,5,6],[7,8,9]].
[X || Row <- Matrix, X <- Row].
%% = [1,2,3,4,5,6,7,8,9]

%% Pattern matching in generator
Users = [{alice, 30}, {bob, 25}, {charlie, 35}].
[Name || {Name, Age} <- Users, Age > 28].
%% = [alice, charlie]

%% Combine map and filter
[X * X || X <- lists:seq(1, 10), X rem 2 =:= 0].
%% = [4, 16, 36, 64, 100]

%% Pythagorean triples
[{A, B, C} || 
    C <- lists:seq(1, 20),
    B <- lists:seq(1, C),
    A <- lists:seq(1, B),
    A*A + B*B =:= C*C].
%% = [{3,4,5}, {6,8,10}, {5,12,13}, ...]

%% Matrix operations
Transpose = fun(Matrix) ->
    [[lists:nth(I, Row) || Row <- Matrix]
     || I <- lists:seq(1, length(hd(Matrix)))]
end.

Transpose([[1,2,3],[4,5,6],[7,8,9]]).
%% = [[1,4,7],[2,5,8],[3,6,9]]
```

### Binary Comprehensions

```erlang
%% Binary comprehension
<< <<X*2>> || <<X>> <= <<1, 2, 3, 4, 5>> >>.
%% = <<2, 4, 6, 8, 10>>

%% กับ filter
<< <<X>> || <<X>> <= <<1,2,3,4,5,6>>, X rem 2 =:= 0 >>.
%% = <<2, 4, 6>>

%% กับ string
<< (string:to_upper(X)) || X <- "hello world" >>.
%% ไม่ถูกต้อง (string เป็น list ไม่ใช่ binary)

%% ถูกต้อง
<< <<(C bor 32)>> || <<C>> <= <<"HELLO">>, C >= $A, C =< $Z >>.
%% = <<"hello">> (convert to lowercase)
```

---

## Recursive List Processing

```erlang
%% สร้าง utility functions แบบ recursive

%% take/2 - เอา N elements แรก
take(_, []) -> [];
take(0, _) -> [];
take(N, [H|T]) when N > 0 -> [H | take(N-1, T)].

take(3, [1,2,3,4,5]).  % = [1,2,3]

%% drop/2 - ทิ้ง N elements แรก
drop(0, List) -> List;
drop(_, []) -> [];
drop(N, [_|T]) when N > 0 -> drop(N-1, T).

drop(3, [1,2,3,4,5]).  % = [4,5]

%% chunk/2 - แบ่ง list เป็น sublists
chunk(_, []) -> [];
chunk(N, List) ->
    {Head, Tail} = lists:split(min(N, length(List)), List),
    [Head | chunk(N, Tail)].

chunk(3, [1,2,3,4,5,6,7]).  % = [[1,2,3],[4,5,6],[7]]

%% sliding_window/2 - sliding window
sliding_window(_, []) -> [];
sliding_window(N, List) when length(List) < N -> [];
sliding_window(N, [_|T] = List) ->
    [lists:sublist(List, N) | sliding_window(N, T)].

sliding_window(3, [1,2,3,4,5]).
%% = [[1,2,3],[2,3,4],[3,4,5]]

%% group_by/2 - group elements by key function
group_by(KeyFun, List) ->
    lists:foldl(
        fun(Item, Map) ->
            Key = KeyFun(Item),
            maps:update_with(Key, fun(V) -> [Item|V] end, [Item], Map)
        end,
        #{},
        List
    ).

group_by(fun(X) -> X rem 3 end, [1,2,3,4,5,6,7,8,9]).
%% = #{0 => [9,6,3], 1 => [7,4,1], 2 => [8,5,2]}

%% unique/1 - remove duplicates (preserving order)
unique(List) ->
    unique(List, #{}).
unique([], _Seen) -> [];
unique([H|T], Seen) ->
    case maps:is_key(H, Seen) of
        true -> unique(T, Seen);
        false -> [H | unique(T, Seen#{H => true})]
    end.

unique([1,2,3,2,1,4,3,5]).  % = [1,2,3,4,5]

%% interleave/2 - สลับ elements
interleave([], List2) -> List2;
interleave(List1, []) -> List1;
interleave([H1|T1], [H2|T2]) ->
    [H1, H2 | interleave(T1, T2)].

interleave([1,3,5], [2,4,6]).  % = [1,2,3,4,5,6]
```

---

## Efficient List Operations

```erlang
%% Performance tips สำคัญมาก!

%% 1. Prepend แทน Append - O(1) vs O(n)
%% ไม่ดี
bad_build(N) ->
    bad_build(N, []).
bad_build(0, Acc) -> Acc;
bad_build(N, Acc) ->
    bad_build(N-1, Acc ++ [N]).  %% O(n²)!

%% ดี
good_build(N) ->
    lists:reverse(good_build(N, [])).
good_build(0, Acc) -> Acc;
good_build(N, Acc) ->
    good_build(N-1, [N | Acc]).  %% O(n)

%% 2. ใช้ ++ อย่างระมัดระวัง
%% ++ สร้าง new list ทุกครั้ง
[1,2,3] ++ [4,5,6].  % สร้าง list ใหม่ขนาด 6

%% ถ้า build list ใหญ่ ใช้ foldr หรือ prepend + reverse
lists:foldr(fun(X, Acc) -> [X | Acc] end, [], [1,2,3,4,5]).
%% = [1,2,3,4,5]

%% 3. ใช้ IOList แทนการ concat strings
%% ดู section IOLists ด้านล่าง

%% 4. ใช้ lists:foldl (tail recursive) แทน foldr เมื่อลำดับไม่สำคัญ
%% foldl = O(n), stack-safe
%% foldr = O(n), แต่ต้อง build stack

%% 5. ใช้ erlang:length/1 น้อยที่สุด
%% length เป็น O(n) - ถ้าต้องการแค่ empty check ใช้ pattern matching
is_empty([]) -> true;
is_empty(_) -> false.

%% อย่าทำ:
check_list(List) when length(List) > 0 ->  % O(n)
    ...

%% ทำ:
check_list([_|_]) ->  % O(1) pattern matching
    ...

%% 6. lists:append/1 กับ list ของ lists
lists:append([[1,2],[3,4],[5,6]]).  % ดีกว่า [1,2]++[3,4]++[5,6]
```

---

## IOLists

IOList เป็นวิธีที่มีประสิทธิภาพสูงในการสร้าง string/binary ขนาดใหญ่:

```erlang
%% IOList = list ที่มี: integers, binaries, หรือ nested iolists
%% ไม่ต้อง flatten หรือ concat จริงๆ จนกว่าจะส่งออก

%% ตัวอย่าง
IOList = ["Hello", <<", ">>, "World", $!, 10].
%% [72,101,108,108,111, <<", ">>, 87,111,114,108,100, 33, 10]

%% แปลงเป็น binary
iolist_to_binary(IOList).
%% = <<"Hello, World!\n">>

%% ส่ง IOList ไปยัง file/socket โดยตรงได้เลย
file:write(File, IOList).  % ไม่ต้อง flatten!
gen_tcp:send(Socket, IOList).

%% ทำไม IOList ดีกว่า String Concatenation?
build_html_bad(Title, Body) ->
    "<html><head><title>" ++ Title ++ "</title></head>" ++
    "<body>" ++ Body ++ "</body></html>".
%% สร้าง intermediate strings เยอะมาก!

build_html_good(Title, Body) ->
    ["<html><head><title>", Title, "</title></head>",
     "<body>", Body, "</body></html>"].
%% ไม่มี allocation จนกว่าจะ flatten จริงๆ!

%% การสร้าง JSON string ด้วย IOList
encode_json(#{} = Map) ->
    Pairs = maps:fold(
        fun(K, V, Acc) ->
            Pair = [encode_json_key(K), ":", encode_json_value(V)],
            [Pair | Acc]
        end,
        [],
        Map
    ),
    ["{", lists:join(",", Pairs), "}"];
encode_json(List) when is_list(List) ->
    ["[", lists:join(",", [encode_json(X) || X <- List]), "]"];
encode_json(Str) when is_binary(Str) ->
    [$", Str, $"];
encode_json(N) when is_number(N) ->
    io_lib:format("~p", [N]);
encode_json(true) -> "true";
encode_json(false) -> "false";
encode_json(null) -> "null".

encode_json_key(K) when is_atom(K) ->
    [$", atom_to_list(K), $"];
encode_json_key(K) when is_binary(K) ->
    [$", K, $"].

encode_json_value(V) -> encode_json(V).
```

---

## Sorted Lists

```erlang
%% ordsets module - sorted lists without duplicates

%% สร้าง ordset
S1 = ordsets:from_list([3, 1, 4, 1, 5, 9, 2, 6]).
%% = [1, 2, 3, 4, 5, 6, 9]

S2 = ordsets:from_list([2, 4, 6, 8, 10]).
%% = [2, 4, 6, 8, 10]

%% Operations
ordsets:union(S1, S2).           % = [1, 2, 3, 4, 5, 6, 8, 9, 10]
ordsets:intersection(S1, S2).    % = [2, 4, 6]
ordsets:subtract(S1, S2).        % = [1, 3, 5, 9]
ordsets:is_subset(S2, S1).       % = false

ordsets:add_element(7, S1).      % = [1, 2, 3, 4, 5, 6, 7, 9]
ordsets:del_element(5, S1).      % = [1, 2, 3, 4, 6, 9]
ordsets:is_element(5, S1).       % = true
ordsets:size(S1).                % = 7
```

---

## Set Operations

```erlang
%% sets module - hash set (ไม่ sorted, เร็วกว่าสำหรับ lookup)

S1 = sets:from_list([1, 2, 3, 4, 5]).
S2 = sets:from_list([3, 4, 5, 6, 7]).

sets:union(S1, S2).        % set ที่รวมกัน
sets:intersection(S1, S2). % set ที่ตัดกัน = {3,4,5}
sets:subtract(S1, S2).     % S1 - S2 = {1,2}
sets:is_subset(S1, S2).    % = false

sets:add_element(6, S1).   % เพิ่ม element
sets:del_element(1, S1).   % ลบ element
sets:is_element(3, S1).    % = true
sets:size(S1).             % = 5
sets:to_list(S1).          % แปลงเป็น list

%% gb_sets module - general balanced binary trees
%% เร็วสุดสำหรับ ordered set operations
GBS = gb_sets:from_list([3, 1, 4, 1, 5]).
gb_sets:largest(GBS).   % = 5
gb_sets:smallest(GBS).  % = 1
gb_sets:next(gb_sets:iterator(GBS)).  % iterator
```

---

## Association Lists (Proplists)

```erlang
%% Proplist = list of {key, value} tuples
PropList = [{name, "Alice"}, {age, 30}, {email, "alice@example.com"}].

%% proplists module
proplists:get_value(name, PropList).
%% = "Alice"

proplists:get_value(phone, PropList, undefined).
%% = undefined (default value)

proplists:get_bool(active, [{active, true}]).
%% = true

proplists:is_defined(name, PropList).
%% = true

proplists:delete(age, PropList).
%% = [{name, "Alice"}, {email, "alice@example.com"}]

proplists:to_map(PropList).
%% = #{name => "Alice", age => 30, email => "alice@example.com"}

%% Compact representation
CompactList = [name, active, {age, 30}].
%% name และ active ถือว่าเป็น {name, true} และ {active, true}
proplists:get_value(name, CompactList).   % = true
proplists:get_bool(active, CompactList).  % = true
proplists:get_value(age, CompactList).    % = 30
```

---

## ตัวอย่างจริง

### Statistics Module

```erlang
-module(statistics).
-export([mean/1, median/1, mode/1, std_dev/1, variance/1,
         percentile/2, histogram/2]).

mean([]) -> {error, empty_list};
mean(List) ->
    lists:sum(List) / length(List).

median([]) -> {error, empty_list};
median(List) ->
    Sorted = lists:sort(List),
    N = length(Sorted),
    case N rem 2 of
        0 ->
            Mid1 = lists:nth(N div 2, Sorted),
            Mid2 = lists:nth(N div 2 + 1, Sorted),
            (Mid1 + Mid2) / 2;
        1 ->
            lists:nth((N + 1) div 2, Sorted)
    end.

mode([]) -> {error, empty_list};
mode(List) ->
    Counts = lists:foldl(
        fun(X, Map) ->
            maps:update_with(X, fun(V) -> V + 1 end, 1, Map)
        end,
        #{},
        List
    ),
    {Mode, _MaxCount} = maps:fold(
        fun(K, V, {BestK, BestV}) ->
            if V > BestV -> {K, V};
               true -> {BestK, BestV}
            end
        end,
        {undefined, -1},
        Counts
    ),
    Mode.

variance([]) -> {error, empty_list};
variance(List) ->
    Avg = mean(List),
    Diffs = [math:pow(X - Avg, 2) || X <- List],
    lists:sum(Diffs) / length(List).

std_dev(List) ->
    math:sqrt(variance(List)).

percentile(List, P) when P >= 0, P =< 100 ->
    Sorted = lists:sort(List),
    N = length(Sorted),
    Index = round(P / 100 * (N - 1)) + 1,
    lists:nth(min(Index, N), Sorted).

histogram(List, NumBuckets) ->
    Min = lists:min(List),
    Max = lists:max(List),
    Range = Max - Min,
    BucketSize = Range / NumBuckets,
    lists:foldl(
        fun(X, Hist) ->
            Bucket = min(trunc((X - Min) / BucketSize), NumBuckets - 1),
            maps:update_with(Bucket, fun(V) -> V + 1 end, 1, Hist)
        end,
        #{},
        List
    ).
```

```erlang
% ทดสอบ
Data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].
statistics:mean(Data).       % = 5.5
statistics:median(Data).     % = 5.5
statistics:std_dev(Data).    % = 2.8722...
statistics:percentile(Data, 75).  % = 8
statistics:histogram(Data, 3).    % = #{0=>3, 1=>3, 2=>4} approx
```

---

## สรุป Part 8

| Function/Pattern | Description |
|-----------------|-------------|
| `[H\|T]` | Destructure head/tail |
| `lists:map/2` | Transform each element |
| `lists:filter/2` | Keep matching elements |
| `lists:foldl/3` | Reduce to single value |
| `[X \|\| X <- L, cond]` | List comprehension |
| `lists:sort/1` | Sort list |
| `lists:usort/1` | Sort + unique |
| `lists:keysort/2` | Sort by tuple key |
| `iolist_to_binary/1` | Efficient string building |
| `ordsets` | Sorted set operations |
| `proplists` | Key-value list |

---

*[← Part 7: Modules](part_07_modules.md) | [Part 9: Tuples and Records →](part_09_tuples_records.md)*
