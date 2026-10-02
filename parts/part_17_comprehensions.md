# Part 17: Comprehensions — List, Binary, Map

## สารบัญ
1. [List Comprehensions](#list-comprehensions)
2. [Multiple Generators](#multiple-generators)
3. [Filters](#filters)
4. [Nested Comprehensions](#nested-comprehensions)
5. [Binary Comprehensions](#binary-comprehensions)
6. [Map Comprehensions (OTP 26+)](#map-comprehensions-otp-26)
7. [Comprehension Patterns](#comprehension-patterns)
8. [ตัวอย่างจริง: Data Transformation](#ตัวอย่างจริง-data-transformation)

---

## List Comprehensions

```erlang
%% Syntax: [Expression || Generator, ...Filter...]
%% Generator: Pattern <- List
%% Filter: BooleanExpression

%% พื้นฐาน: double each element
[X * 2 || X <- [1,2,3,4,5]].
%% = [2,4,6,8,10]

%% เหมือนกับ lists:map
lists:map(fun(X) -> X * 2 end, [1,2,3,4,5]).
%% = [2,4,6,8,10]

%% Squares
[X * X || X <- lists:seq(1, 10)].
%% = [1,4,9,16,25,36,49,64,81,100]

%% With transformation
[{X, X*X} || X <- lists:seq(1, 5)].
%% = [{1,1}, {2,4}, {3,9}, {4,16}, {5,25}]

%% Pattern in generator (extract from tuples)
[Name || {Name, _Age} <- [{alice, 30}, {bob, 25}, {carol, 35}]].
%% = [alice, bob, carol]

[Age || {_Name, Age} <- [{alice, 30}, {bob, 25}, {carol, 35}]].
%% = [30, 25, 35]

%% Increment each character
[$X + 1 || $X <- "hello"].
%% = "ifmmp" (each char + 1)
```

---

## Multiple Generators

```erlang
%% Multiple generators = Cartesian product
[{X, Y} || X <- [1,2,3], Y <- [a,b]].
%% = [{1,a}, {1,b}, {2,a}, {2,b}, {3,a}, {3,b}]

%% Order matters: inner loop is rightmost generator
[{X, Y} || X <- [1,2], Y <- [3,4]].
%% = [{1,3}, {1,4}, {2,3}, {2,4}]

%% Multiplication table
[{X, Y, X*Y} || X <- lists:seq(1,3), Y <- lists:seq(1,3)].
%% = [{1,1,1},{1,2,2},{1,3,3},{2,1,2},...,{3,3,9}]

%% Zip-like with index
WithIndex = fun(List) ->
    Indexed = lists:zip(lists:seq(0, length(List)-1), List),
    [{I, V} || {I, V} <- Indexed]
end.
WithIndex([a,b,c]).
%% = [{0,a}, {1,b}, {2,c}]

%% Independent generators + filtering for triangles
[{A, B, C} || 
    C <- lists:seq(1, 20),
    B <- lists:seq(1, C),
    A <- lists:seq(1, B),
    A*A + B*B =:= C*C].
%% = [{3,4,5}, {6,8,10}, {5,12,13}, {8,15,17}, {9,12,15}]  (Pythagorean triples)

%% Dependent generators (Y depends on X)
[{X, Y} || X <- [1,2,3,4], Y <- lists:seq(1, X)].
%% = [{1,1}, {2,1}, {2,2}, {3,1}, {3,2}, {3,3}, {4,1}, {4,2}, {4,3}, {4,4}]
```

---

## Filters

```erlang
%% Filter: keep elements where condition is true
[X || X <- lists:seq(1, 20), X rem 2 =:= 0].
%% = [2,4,6,8,10,12,14,16,18,20]

%% Multiple filters (AND)
[X || X <- lists:seq(1, 50), X rem 2 =:= 0, X rem 3 =:= 0].
%% = [6, 12, 18, 24, 30, 36, 42, 48]

%% Pattern matching as filter (non-matching patterns are skipped!)
People = [{alice, 30}, {bob, 25}, {carol, 35}, {dave, 28}].
[Name || {Name, Age} <- People, Age >= 30].
%% = [alice, carol]

%% Pattern matching for type filtering
Mixed = [1, "hello", 2, :atom, 3, [list]].
[X || X <- Mixed, is_integer(X)].
%% = [1, 2, 3]

%% Nested pattern as filter
Data = [{ok, 1}, {error, bad}, {ok, 2}, {ok, 3}, {error, very_bad}].
[V || {ok, V} <- Data].
%% = [1, 2, 3]  (error tuples are automatically skipped)

[E || {error, E} <- Data].
%% = [bad, very_bad]

%% Guard-like filters
[X || X <- lists:seq(1, 100), is_prime(X)].  %% primes up to 100

%% ตัวอย่าง is_prime
is_prime(2) -> true;
is_prime(N) when N < 2 -> false;
is_prime(N) ->
    Limit = trunc(math:sqrt(N)) + 1,
    not lists:any(fun(D) -> N rem D =:= 0 end, lists:seq(2, Limit)).
```

---

## Nested Comprehensions

```erlang
%% Flatten nested list
[[1,2],[3,4],[5,6]].
[X || L <- [[1,2],[3,4],[5,6]], X <- L].
%% = [1,2,3,4,5,6]

%% เหมือนกับ lists:flatten/1 (1 level)
lists:append([[1,2],[3,4],[5,6]]).
%% = [1,2,3,4,5,6]

%% Flatten matrix rows into single list
Matrix = [[1,2,3],[4,5,6],[7,8,9]].
[X || Row <- Matrix, X <- Row].
%% = [1,2,3,4,5,6,7,8,9]

%% Transform matrix
[[X * 2 || X <- Row] || Row <- Matrix].
%% = [[2,4,6],[8,10,12],[14,16,18]]

%% Transpose matrix
transpose(Matrix) ->
    case Matrix of
        [] -> [];
        [[]|_] -> [];
        _ ->
            [lists:map(fun hd/1, Matrix) |
             transpose(lists:map(fun tl/1, Matrix))]
    end.

%% Using comprehension
transpose2(Matrix) ->
    Rows = length(Matrix),
    Cols = length(hd(Matrix)),
    [[lists:nth(R, lists:nth(C, Matrix)) || R <- lists:seq(1, Rows)]
     || C <- lists:seq(1, Cols)].

%% Map all values in list of maps
Users = [#{name => "alice", age => 30}, #{name => "bob", age => 25}].
[#{name := N} || #{name := N} <- Users].
%% = ["alice", "bob"]

[U#{age => maps:get(age, U) + 1} || U <- Users].
%% = [#{name => "alice", age => 31}, #{name => "bob", age => 26}]
```

---

## Binary Comprehensions

```erlang
%% Syntax: << Expression || BitStringGenerator, ... >>
%% Generator: Pattern <= Binary

%% Double each byte
<< <<X*2>> || <<X>> <= <<1,2,3,4,5>> >>.
%% = <<2,4,6,8,10>>

%% Uppercase binary (ASCII)
<< <<(X - 32)>> || <<X>> <= <<"HELLO">>, X >= $a, X =< $z >>.
%% ไม่ถูกสำหรับ uppercase, ตัวอย่างที่ถูกต้อง:

%% Convert to lowercase
<< <<(X + 32)>> || <<X>> <= <<"HELLO">>, X >= $A, X =< $Z >>.
%% = <<"hello">>

%% Filter bytes
<< <<X>> || <<X>> <= <<"hello world">>, X =/= $  >>.
%% = <<"helloworld">> (remove spaces)

%% Extract first byte from each 2-byte pair
<< <<A>> || <<A, _B>> <= <<1,2,3,4,5,6>> >>.
%% = <<1,3,5>>

%% XOR encryption
encrypt(Data, Key) ->
    KeySize = byte_size(Key),
    << <<(X bxor binary:at(Key, I rem KeySize))>> 
       || {I, X} <- lists:enumerate(binary_to_list(Data)) >>.
%% Note: lists:enumerate is OTP 25+

%% simpler XOR
xor_bytes(Data, Key) ->
    KLen = byte_size(Key),
    << <<(binary:at(Data, I) bxor binary:at(Key, I rem KLen))>>
       || I <- lists:seq(0, byte_size(Data) - 1) >>.

xor_bytes(<<"Hello">>, <<16#FF>>).  %% = encrypted binary

%% Parse binary protocol
%% Given binary: <<ID:16, Len:16, Data:Len/binary, ...>>
parse_packets(<<>>) -> [];
parse_packets(<<Id:16, Len:16, Data:Len/binary, Rest/binary>>) ->
    [{Id, Data} | parse_packets(Rest)].

%% Binary comprehension for bit manipulation
%% Interleave bits from two binaries
interleave(<<>>, <<>>) -> <<>>;
interleave(<<A:1, RestA/bitstring>>, <<B:1, RestB/bitstring>>) ->
    Tail = interleave(RestA, RestB),
    <<A:1, B:1, Tail/bitstring>>.
```

---

## Map Comprehensions (OTP 26+)

```erlang
%% Syntax: #{Key => Value || KeyGenerator, ...}
%% Generator: K := V <- Map  OR  Expr <- List

M = #{a => 1, b => 2, c => 3}.

%% Double all values
#{K => V*2 || K := V <- M}.
%% = #{a => 2, b => 4, c => 6}

%% Filter by value
#{K => V || K := V <- M, V > 1}.
%% = #{b => 2, c => 3}

%% Transform keys
#{atom_to_binary(K) => V || K := V <- M}.
%% = #{<<"a">> => 1, <<"b">> => 2, <<"c">> => 3}

%% From list of pairs
#{ K => V || {K, V} <- [{a, 1}, {b, 2}, {c, 3}] }.
%% = #{a => 1, b => 2, c => 3}

%% Invert (swap keys and values)
#{V => K || K := V <- M}.
%% = #{1 => a, 2 => b, 3 => c}

%% Create lookup map from list
Users = [#{id => 1, name => "Alice"}, #{id => 2, name => "Bob"}].
#{Id => User || #{id := Id} = User <- Users}.
%% = #{1 => #{id=>1,name=>"Alice"}, 2 => #{id=>2,name=>"Bob"}}

%% Index a list
Words = [<<"hello">>, <<"world">>, <<"erlang">>].
#{I => W || {I, W} <- lists:enumerate(Words)}.
%% = #{1 => <<"hello">>, 2 => <<"world">>, 3 => <<"erlang">>}
```

---

## Comprehension Patterns

```erlang
%% Pattern: flatten then filter
FlatFiltered = [X || SubList <- [[1,2,3],[4,5],[6,7,8]], X <- SubList, X > 4].
%% = [5, 6, 7, 8]

%% Pattern: safe extract from list
SafeValues = [V || {ok, V} <- [call1(), call2(), call3()]].
%% Skips errors automatically

%% Pattern: build lookup table
Lookup = #{X => X*X || X <- lists:seq(0, 100)}.
maps:get(7, Lookup).   %% = 49
maps:get(12, Lookup).  %% = 144

%% Pattern: cross-product with constraint
Pairs = [{X, Y} || X <- lists:seq(1, 10), Y <- lists:seq(X+1, 10), X + Y =:= 11].
%% = [{1,10}, {2,9}, {3,8}, {4,7}, {5,6}]

%% Pattern: decode binary data
%% Binary format: <<Count:8, Data:Count/binary, ...>>
decode_segments(Bin) ->
    decode_segments(Bin, []).

decode_segments(<<>>, Acc) ->
    lists:reverse(Acc);
decode_segments(<<Len:8, Data:Len/binary, Rest/binary>>, Acc) ->
    decode_segments(Rest, [Data|Acc]).

%% Pattern: comprehension vs map+filter comparison
%% Using comprehension
Result1 = [X*2 || X <- List, X > 5].

%% Using map+filter
Result2 = lists:map(
    fun(X) -> X*2 end,
    lists:filter(fun(X) -> X > 5 end, List)
).

%% Both are equivalent; comprehension is usually cleaner

%% Pattern: complex data reshaping
Events = [
    #{type => view, user => alice, page => home},
    #{type => click, user => alice, button => signup},
    #{type => view, user => bob, page => home},
    #{type => view, user => alice, page => docs}
].

%% Count views per user
UserViews = lists:foldl(
    fun(#{type := view, user := User}, Acc) ->
        maps:update_with(User, fun(V) -> V+1 end, 1, Acc);
       (_, Acc) -> Acc
    end,
    #{},
    Events
).
%% = #{alice => 2, bob => 1}

%% Extract alice's events
[E || #{user := alice} = E <- Events].
%% = all alice's events
```

---

## ตัวอย่างจริง: Data Transformation

```erlang
%% ไฟล์: data_transform.erl
-module(data_transform).
-export([
    to_csv/2,
    from_csv/2,
    normalize/1,
    pivot/1,
    unnest/2
]).

%% Convert list of maps to CSV
-spec to_csv([map()], [atom()]) -> binary().
to_csv([], _Columns) -> <<>>;
to_csv(Rows, Columns) ->
    Header = iolist_to_binary(
        lists:join(<<",">>, [atom_to_binary(C) || C <- Columns])
    ),
    DataRows = [
        iolist_to_binary(lists:join(<<",">>, [
            format_cell(maps:get(Col, Row, <<>>)) || Col <- Columns
        ]))
        || Row <- Rows
    ],
    iolist_to_binary(lists:join(<<"\n">>, [Header | DataRows])).

format_cell(V) when is_binary(V) -> 
    case binary:match(V, [<<",">>, <<"\n">>, <<"\"">>]) of
        nomatch -> V;
        _ -> <<"\"", (binary:replace(V, <<"\"">>, <<"\"\"">>, [global]))/binary, "\"">>
    end;
format_cell(V) when is_integer(V) -> integer_to_binary(V);
format_cell(V) when is_float(V) -> float_to_binary(V, [{decimals, 4}, compact]);
format_cell(V) when is_atom(V) -> atom_to_binary(V);
format_cell(_) -> <<>>.

%% Parse CSV to list of maps
-spec from_csv(binary(), [atom()]) -> [map()].
from_csv(CSV, Columns) ->
    Lines = binary:split(CSV, <<"\n">>, [global, trim_all]),
    DataLines = case Lines of
        [_Header | Data] -> Data;  %% skip header
        _ -> Lines
    end,
    [parse_row(Line, Columns) || Line <- DataLines].

parse_row(Line, Columns) ->
    Values = split_csv_row(Line),
    maps:from_list(lists:zip(Columns, Values ++ lists:duplicate(
        max(0, length(Columns) - length(Values)), undefined))).

split_csv_row(Line) ->
    %% Simple CSV split (doesn't handle quoted fields with commas)
    binary:split(Line, <<",">>, [global]).

%% Normalize: convert list of maps to consistent schema
-spec normalize([map()]) -> [map()].
normalize([]) -> [];
normalize(Rows) ->
    AllKeys = lists:usort(
        lists:append([maps:keys(Row) || Row <- Rows])
    ),
    [normalize_row(Row, AllKeys) || Row <- Rows].

normalize_row(Row, AllKeys) ->
    maps:from_list([{K, maps:get(K, Row, null)} || K <- AllKeys]).

%% Pivot: group rows into columns
-spec pivot([map()]) -> map().
pivot(Rows) ->
    case Rows of
        [] -> #{};
        [First | _] ->
            Keys = maps:keys(First),
            #{K => [maps:get(K, Row, null) || Row <- Rows] || K <- Keys}
    end.

%% Unnest: flatten nested list/map fields
-spec unnest([map()], atom()) -> [map()].
unnest(Rows, NestedKey) ->
    lists:flatmap(
        fun(Row) ->
            case maps:get(NestedKey, Row, []) of
                [] -> [Row];
                Items when is_list(Items) ->
                    [maps:merge(maps:remove(NestedKey, Row), Item) 
                     || Item <- Items];
                Item when is_map(Item) ->
                    [maps:merge(maps:remove(NestedKey, Row), Item)]
            end
        end,
        Rows
    ).
```

```erlang
%% ทดสอบ data_transform

Rows = [
    #{name => <<"Alice">>, age => 30, city => <<"Bangkok">>},
    #{name => <<"Bob">>, age => 25, city => <<"Chiang Mai">>},
    #{name => <<"Carol">>, age => 35, city => <<"Phuket">>}
].

%% to CSV
Csv = data_transform:to_csv(Rows, [name, age, city]).
%% = <<"name,age,city\nAlice,30,Bangkok\nBob,25,Chiang Mai\nCarol,35,Phuket">>

%% Normalize (add missing fields)
Mixed = [#{a => 1, b => 2}, #{b => 3, c => 4}, #{a => 5}].
data_transform:normalize(Mixed).
%% = [#{a=>1,b=>2,c=>null}, #{a=>null,b=>3,c=>4}, #{a=>5,b=>null,c=>null}]

%% Pivot
data_transform:pivot(Rows).
%% = #{name => [<<"Alice">>,<<"Bob">>,<<"Carol">>], age => [30,25,35], ...}

%% Unnest
Orders = [
    #{order_id => 1, items => [#{sku => <<"A">>, qty => 2}, #{sku => <<"B">>, qty => 1}]},
    #{order_id => 2, items => [#{sku => <<"C">>, qty => 3}]}
].

data_transform:unnest(Orders, items).
%% = [#{order_id=>1, sku=><<"A">>, qty=>2},
%%    #{order_id=>1, sku=><<"B">>, qty=>1},
%%    #{order_id=>2, sku=><<"C">>, qty=>3}]
```

---

## สรุป Part 17

| Type | Syntax | ใช้เมื่อ |
|------|--------|---------|
| List comprehension | `[E \|\| P <- L, Filter]` | Transform/filter lists |
| Binary comprehension | `<< B \|\| P <= Bin >>` | Process binary data |
| Map comprehension | `#{K => V \|\| K := V <- M}` | Transform maps (OTP 26+) |
| Multi-generator | `[E \|\| X <- L1, Y <- L2]` | Cartesian product |
| Pattern filter | `[V \|\| {ok, V} <- L]` | Safe extraction |

**ประสิทธิภาพ**: List comprehensions ใช้ list append internally จึงอาจไม่เหมาะกับ list ขนาดใหญ่มาก — ใช้ `lists:foldl` แทนถ้าต้องการประสิทธิภาพสูง

---

*[← Part 16: Higher-Order Functions](part_16_higher_order.md) | [Part 18: BIFs →](part_18_bifs.md)*
