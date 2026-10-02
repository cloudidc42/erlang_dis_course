# Part 15: Recursion — การเรียกตัวเองซ้ำ

## สารบัญ
1. [Why Recursion in Erlang](#why-recursion-in-erlang)
2. [Basic Recursion](#basic-recursion)
3. [Tail Recursion](#tail-recursion)
4. [Accumulator Pattern](#accumulator-pattern)
5. [Mutual Recursion](#mutual-recursion)
6. [Tree Recursion](#tree-recursion)
7. [Trampoline Pattern](#trampoline-pattern)
8. [ตัวอย่างจริง: Data Processing](#ตัวอย่างจริง-data-processing)

---

## Why Recursion in Erlang

Erlang ไม่มี loop statements เช่น `for`, `while`, `do-while`:

```erlang
%% ไม่มีสิ่งนี้ใน Erlang!
%% for i = 1 to 10: print(i)
%% while running: do_work()

%% ต้องใช้ recursion แทน
print_range(Current, Max) when Current > Max -> ok;
print_range(Current, Max) ->
    io:format("~p~n", [Current]),
    print_range(Current + 1, Max).

print_range(1, 10).  %% prints 1 to 10

%% ทำไม recursion ถึงเหมาะกับ Erlang:
%% 1. Immutable variables - ไม่สามารถ mutate counter ได้
%% 2. Tail call optimization - ไม่ stack overflow ถ้า tail recursive
%% 3. Expressive - code อ่านง่ายเมื่อเข้าใจ pattern
%% 4. Natural fit - functional programming style
```

---

## Basic Recursion

```erlang
%% Pattern พื้นฐาน: Base case + Recursive case

%% Factorial
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N - 1).

factorial(5).  %% = 5 * 4 * 3 * 2 * 1 = 120

%% Stack trace:
%% factorial(5)
%% = 5 * factorial(4)
%% = 5 * (4 * factorial(3))
%% = 5 * (4 * (3 * factorial(2)))
%% = 5 * (4 * (3 * (2 * factorial(1))))
%% = 5 * (4 * (3 * (2 * (1 * factorial(0)))))
%% = 5 * (4 * (3 * (2 * (1 * 1))))
%% = 120
%% ปัญหา: stack grows with N!

%% Sum of list
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

sum([1,2,3,4,5]).  %% = 15

%% Length of list
my_length([]) -> 0;
my_length([_|T]) -> 1 + my_length(T).

my_length([a,b,c]).  %% = 3

%% Fibonacci (non-tail recursive)
fib(0) -> 0;
fib(1) -> 1;
fib(N) when N > 1 -> fib(N-1) + fib(N-2).
%% ช้ามาก! O(2^n) - exponential time
%% fib(30) ทำการคำนวณซ้ำหลายล้านครั้ง

%% Fibonacci (memoized)
fib_memo(N) -> fib_memo(N, #{0 => 0, 1 => 1}).

fib_memo(N, Cache) ->
    case maps:find(N, Cache) of
        {ok, Value} -> Value;
        error ->
            V1 = fib_memo(N-1, Cache),
            V2 = fib_memo(N-2, Cache),
            V1 + V2
    end.
%% ยังไม่ถูกต้อง 100% แต่แสดง concept
```

---

## Tail Recursion

```erlang
%% Tail recursive = recursive call เป็น last action
%% Erlang optimizes tail calls ไม่ต้อง push stack frame

%% Non-tail recursive (ปัญหา stack overflow สำหรับ N ใหญ่)
sum_nontail([]) -> 0;
sum_nontail([H|T]) ->
    H + sum_nontail(T).  %% + H ทำหลัง recursive call = NOT tail recursive

%% Tail recursive version
sum_tail([]) -> 0;
sum_tail(List) -> sum_tail(List, 0).

sum_tail([], Acc) -> Acc;
sum_tail([H|T], Acc) ->
    sum_tail(T, Acc + H).   %% recursive call เป็น last action = TAIL RECURSIVE

%% Stack สำหรับ sum_tail([1,2,3,4,5])
%% sum_tail([1,2,3,4,5], 0)
%% sum_tail([2,3,4,5], 1)
%% sum_tail([3,4,5], 3)
%% sum_tail([4,5], 6)
%% sum_tail([5], 10)
%% sum_tail([], 15)
%% 15
%% Stack size = constant!

%% Factorial tail recursive
factorial_tail(N) -> factorial_tail(N, 1).

factorial_tail(0, Acc) -> Acc;
factorial_tail(N, Acc) when N > 0 ->
    factorial_tail(N - 1, N * Acc).

factorial_tail(5).  %% = 120

%% Fibonacci tail recursive - O(n) time, O(1) space
fib_tail(N) -> fib_tail(N, 0, 1).

fib_tail(0, A, _) -> A;
fib_tail(N, A, B) when N > 0 ->
    fib_tail(N - 1, B, A + B).

fib_tail(10).  %% = 55

%% Length tail recursive
length_tail(List) -> length_tail(List, 0).

length_tail([], Acc) -> Acc;
length_tail([_|T], Acc) -> length_tail(T, Acc + 1).

%% Reverse tail recursive
reverse_tail(List) -> reverse_tail(List, []).

reverse_tail([], Acc) -> Acc;
reverse_tail([H|T], Acc) -> reverse_tail(T, [H|Acc]).

%% flatten tail recursive
flatten_tail(List) -> flatten_tail(List, []).

flatten_tail([], Acc) ->
    lists:reverse(Acc);
flatten_tail([H|T], Acc) when is_list(H) ->
    flatten_tail(H ++ T, Acc);
flatten_tail([H|T], Acc) ->
    flatten_tail(T, [H|Acc]).

flatten_tail([1,[2,3],[4,[5,6]]]).  %% = [1,2,3,4,5,6]
```

---

## Accumulator Pattern

```erlang
%% Pattern ที่ใช้บ่อยที่สุดใน Erlang recursion

%% ตัวอย่าง: Map function
map_acc([], _F) -> [];
map_acc(List, F) -> map_acc(List, F, []).

map_acc([], _F, Acc) ->
    lists:reverse(Acc);  %% reverse เพราะ prepend
map_acc([H|T], F, Acc) ->
    map_acc(T, F, [F(H) | Acc]).

map_acc([1,2,3,4,5], fun(X) -> X*2 end).
%% = [2,4,6,8,10]

%% Filter function
filter_acc([], _P) -> [];
filter_acc(List, P) -> filter_acc(List, P, []).

filter_acc([], _P, Acc) ->
    lists:reverse(Acc);
filter_acc([H|T], P, Acc) ->
    case P(H) of
        true  -> filter_acc(T, P, [H|Acc]);
        false -> filter_acc(T, P, Acc)
    end.

filter_acc([1,2,3,4,5,6], fun(X) -> X rem 2 =:= 0 end).
%% = [2,4,6]

%% Fold left (accumulator changes with each element)
fold_acc([], _F, Acc) -> Acc;
fold_acc([H|T], F, Acc) ->
    fold_acc(T, F, F(Acc, H)).

fold_acc([1,2,3,4,5], fun(Acc, X) -> Acc + X end, 0).  %% = 15
fold_acc([1,2,3,4,5], fun(Acc, X) -> Acc * X end, 1).  %% = 120

%% Group by (accumulate into map)
group_by(List, KeyFn) ->
    lists:foldl(
        fun(Item, Acc) ->
            Key = KeyFn(Item),
            maps:update_with(
                Key,
                fun(V) -> [Item|V] end,
                [Item],
                Acc
            )
        end,
        #{},
        List
    ).

People = [
    #{name => "Alice", dept => engineering},
    #{name => "Bob", dept => marketing},
    #{name => "Carol", dept => engineering},
    #{name => "Dave", dept => marketing},
    #{name => "Eve", dept => hr}
].

group_by(People, fun(P) -> maps:get(dept, P) end).
%% = #{engineering => [Carol, Alice], marketing => [Dave, Bob], hr => [Eve]}

%% Partition
partition_acc([], _P) -> {[], []};
partition_acc(List, P) -> partition_acc(List, P, [], []).

partition_acc([], _P, Yes, No) ->
    {lists:reverse(Yes), lists:reverse(No)};
partition_acc([H|T], P, Yes, No) ->
    case P(H) of
        true  -> partition_acc(T, P, [H|Yes], No);
        false -> partition_acc(T, P, Yes, [H|No])
    end.

partition_acc([1,2,3,4,5,6], fun(X) -> X rem 2 =:= 0 end).
%% = {[2,4,6], [1,3,5]}
```

---

## Mutual Recursion

```erlang
%% Functions ที่เรียกกันเองสลับกัน

%% ตัวอย่าง: even/odd detection
even(0) -> true;
even(N) when N > 0 -> odd(N - 1);
even(N) when N < 0 -> odd(N + 1).

odd(0) -> false;
odd(N) when N > 0 -> even(N - 1);
odd(N) when N < 0 -> even(N + 1).

even(10).  %% = true
odd(7).    %% = true
even(7).   %% = false

%% ตัวอย่าง: State machine
running(start) -> running(working, 0);
running(State, Count) -> running_impl(State, Count).

running_impl(working, Count) ->
    io:format("Working... ~p~n", [Count]),
    receive
        pause -> paused(Count);
        stop  -> done(Count)
    after 1000 ->
        running_impl(working, Count + 1)
    end.

paused(Count) ->
    io:format("Paused at ~p~n", [Count]),
    receive
        resume -> running_impl(working, Count);
        stop   -> done(Count)
    end.

done(Count) ->
    io:format("Done! Total: ~p~n", [Count]).

%% ตัวอย่าง: Token scanner
scan([], Tokens) ->
    lists:reverse(Tokens);
scan(Input, Tokens) ->
    {Token, Rest} = next_token(Input),
    scan(Rest, [Token|Tokens]).

next_token([H|T]) when H =:= $  -> next_token(T);
next_token([H|T]) when H >= $0, H =< $9 ->
    scan_number([H|T], []);
next_token([H|T]) when H >= $a, H =< $z ->
    scan_identifier([H|T], []);
next_token([H|T]) ->
    {H, T}.

scan_number([H|T], Acc) when H >= $0, H =< $9 ->
    scan_number(T, [H|Acc]);
scan_number(Rest, Acc) ->
    {list_to_integer(lists:reverse(Acc)), Rest}.

scan_identifier([H|T], Acc) when H >= $a, H =< $z;
                                  H >= $A, H =< $Z;
                                  H >= $0, H =< $9;
                                  H =:= $_ ->
    scan_identifier(T, [H|Acc]);
scan_identifier(Rest, Acc) ->
    {list_to_atom(lists:reverse(Acc)), Rest}.
```

---

## Tree Recursion

```erlang
%% Recursive data structures

%% Binary tree (tuple representation)
%% tree() = nil | {node, Left, Value, Right}

%% Insert into BST
insert(nil, Value) ->
    {node, nil, Value, nil};
insert({node, Left, V, Right}, Value) when Value < V ->
    {node, insert(Left, Value), V, Right};
insert({node, Left, V, Right}, Value) when Value > V ->
    {node, Left, V, insert(Right, Value)};
insert(Tree, _) ->
    Tree.  %% Duplicate, ignore

%% Search BST
search(nil, _) -> false;
search({node, _, V, _}, V) -> true;
search({node, Left, V, _}, Value) when Value < V ->
    search(Left, Value);
search({node, _, _, Right}, Value) ->
    search(Right, Value).

%% In-order traversal (sorted)
inorder(nil) -> [];
inorder({node, Left, V, Right}) ->
    inorder(Left) ++ [V] ++ inorder(Right).

%% Tree height
height(nil) -> 0;
height({node, Left, _, Right}) ->
    1 + max(height(Left), height(Right)).

%% Build BST from list
build_bst(List) ->
    lists:foldl(fun insert/2, nil, List).

%% ตัวอย่าง
Tree = build_bst([5, 3, 7, 1, 4, 6, 8]).
inorder(Tree).     %% = [1, 3, 4, 5, 6, 7, 8]
height(Tree).      %% = 3
search(Tree, 4).   %% = true
search(Tree, 9).   %% = false

%% JSON-like tree traversal
walk_json(Value, _F) when is_number(Value) -> Value;
walk_json(Value, _F) when is_boolean(Value) -> Value;
walk_json(null, _F) -> null;
walk_json(Value, F) when is_binary(Value) -> F(Value);
walk_json(Value, F) when is_list(Value) ->
    [walk_json(V, F) || V <- Value];
walk_json(Value, F) when is_map(Value) ->
    maps:map(fun(_K, V) -> walk_json(V, F) end, Value).

%% Example: uppercase all strings in JSON structure
walk_json(
    #{<<"name">> => <<"alice">>, <<"tags">> => [<<"erlang">>, <<"functional">>]},
    fun string:uppercase/1
).
%% = #{<<"name">> => <<"ALICE">>, <<"tags">> => [<<"ERLANG">>, <<"FUNCTIONAL">>]}
```

---

## Trampoline Pattern

```erlang
%% Trampoline ช่วยทำ mutual recursion แบบ tail recursive
%% ไม่ต้อง trust Erlang's optimization สำหรับ mutual recursion

%% Thunk = function ที่ return function หรือ value
-type thunk(A) :: A | fun(() -> thunk(A)).

%% Trampoline runner
trampoline(Thunk) when is_function(Thunk, 0) ->
    trampoline(Thunk());
trampoline(Value) ->
    Value.

%% ตัวอย่าง: large number even check
even_tramp(0) -> true;
even_tramp(N) when N > 0 -> fun() -> odd_tramp(N-1) end.

odd_tramp(0) -> false;
odd_tramp(N) when N > 0 -> fun() -> even_tramp(N-1) end.

%% ใช้งาน
trampoline(even_tramp(1000000)).  %% = true (ไม่ stack overflow!)
trampoline(odd_tramp(999999)).    %% = true

%% Trampoline กับ value wrapper
-type step(A) :: {done, A} | {next, fun(() -> step(A))}.

run({done, Value}) -> Value;
run({next, F}) -> run(F()).

%% Factorial ด้วย trampoline
fact_step(0, Acc) -> {done, Acc};
fact_step(N, Acc) -> {next, fun() -> fact_step(N-1, N*Acc) end}.

run(fact_step(1000, 1)).  %% = factorial(1000), ไม่ stack overflow
```

---

## ตัวอย่างจริง: Data Processing

```erlang
%% ไฟล์: data_processor.erl
-module(data_processor).
-export([
    pipeline/2,
    chunk/2,
    sliding_window/2,
    find_all/2,
    deep_find/2,
    flatten_keys/1,
    tree_map/2
]).

%% Pipeline: apply list of functions to data
-spec pipeline(term(), [fun()]) -> term().
pipeline(Data, []) -> Data;
pipeline(Data, [F|Rest]) ->
    pipeline(F(Data), Rest).

%% ตัวอย่าง
pipeline([1,2,3,4,5], [
    fun(L) -> lists:filter(fun(X) -> X rem 2 =:= 0 end, L) end,
    fun(L) -> lists:map(fun(X) -> X * 10 end, L) end,
    fun lists:sum/1
]).
%% = 60 (filter evens: [2,4], * 10: [20,40], sum: 60)

%% Split list into chunks of N
-spec chunk([term()], pos_integer()) -> [[term()]].
chunk([], _N) -> [];
chunk(List, N) ->
    chunk(List, N, []).

chunk([], _, Acc) ->
    lists:reverse(Acc);
chunk(List, N, Acc) ->
    {Chunk, Rest} = lists:split(min(N, length(List)), List),
    chunk(Rest, N, [Chunk | Acc]).

chunk([1,2,3,4,5,6,7], 3).
%% = [[1,2,3], [4,5,6], [7]]

%% Sliding window
-spec sliding_window([term()], pos_integer()) -> [[term()]].
sliding_window(List, Size) ->
    sliding_window(List, Size, []).

sliding_window(List, Size, Acc) when length(List) < Size ->
    lists:reverse(Acc);
sliding_window(List, Size, Acc) ->
    Window = lists:sublist(List, Size),
    sliding_window(tl(List), Size, [Window | Acc]).

sliding_window([1,2,3,4,5], 3).
%% = [[1,2,3], [2,3,4], [3,4,5]]

%% Find all occurrences in nested structure
-spec find_all(fun(), term()) -> [term()].
find_all(Pred, Value) ->
    find_all(Pred, Value, []).

find_all(Pred, Value, Acc) when is_list(Value) ->
    lists:foldl(
        fun(Item, A) -> find_all(Pred, Item, A) end,
        case Pred(Value) of true -> [Value|Acc]; false -> Acc end,
        Value
    );
find_all(Pred, Value, Acc) when is_map(Value) ->
    maps:fold(
        fun(_, V, A) -> find_all(Pred, V, A) end,
        case Pred(Value) of true -> [Value|Acc]; false -> Acc end,
        Value
    );
find_all(Pred, Value, Acc) ->
    case Pred(Value) of
        true -> [Value|Acc];
        false -> Acc
    end.

%% Deep find in nested map
-spec deep_find(binary(), map()) -> [term()].
deep_find(Key, Map) when is_map(Map) ->
    Found = case maps:find(Key, Map) of
        {ok, V} -> [V];
        error -> []
    end,
    NestedFound = maps:fold(
        fun(_, V, Acc) when is_map(V) ->
            deep_find(Key, V) ++ Acc;
           (_, _, Acc) ->
            Acc
        end,
        [],
        Map
    ),
    Found ++ NestedFound.

%% Flatten nested map keys with dot notation
-spec flatten_keys(map()) -> map().
flatten_keys(Map) ->
    flatten_keys(Map, <<>>, #{}).

flatten_keys(Map, Prefix, Acc) when is_map(Map) ->
    maps:fold(
        fun(K, V, A) ->
            FullKey = case Prefix of
                <<>> -> atom_to_binary(K);
                P -> <<P/binary, ".", (atom_to_binary(K))/binary>>
            end,
            flatten_keys(V, FullKey, A)
        end,
        Acc,
        Map
    );
flatten_keys(Value, Key, Acc) ->
    Acc#{Key => Value}.

flatten_keys(#{a => 1, b => #{c => 2, d => #{e => 3}}}).
%% = #{<<"a">> => 1, <<"b.c">> => 2, <<"b.d.e">> => 3}

%% Map function over tree structure
-spec tree_map(fun(), term()) -> term().
tree_map(F, Tree) when is_list(Tree) ->
    [tree_map(F, Item) || Item <- Tree];
tree_map(F, Tree) when is_map(Tree) ->
    maps:map(fun(_, V) -> tree_map(F, V) end, Tree);
tree_map(F, Tree) when is_tuple(Tree) ->
    list_to_tuple([tree_map(F, E) || E <- tuple_to_list(Tree)]);
tree_map(F, Leaf) ->
    F(Leaf).

%% Uppercase all binary values recursively
tree_map(
    fun(V) when is_binary(V) -> string:uppercase(V);
       (V) -> V
    end,
    #{name => <<"alice">>, hobbies => [<<"erlang">>, <<"hiking">>]}
).
%% = #{name => <<"ALICE">>, hobbies => [<<"ERLANG">>, <<"HIKING">>]}
```

---

## สรุป Part 15

| Pattern | ใช้เมื่อ | ข้อดี |
|---------|---------|-------|
| Basic recursion | Simple base+recursive | ชัดเจน, อ่านง่าย |
| Tail recursion | Lists ขนาดใหญ่ | ไม่ stack overflow |
| Accumulator | Map/filter/fold | Tail-safe |
| Mutual recursion | State machines, parsers | ธรรมชาติ |
| Tree recursion | Nested data, trees | Structural |
| Trampoline | Deep mutual recursion | ป้องกัน stack overflow |

**กฎสำคัญ**: ทุก function ที่ process list ขนาดใหญ่ต้องเป็น tail recursive หรือใช้ `lists:foldl/3`

---

*[← Part 14: Guards](part_14_guards.md) | [Part 16: Higher-Order Functions →](part_16_higher_order.md)*
