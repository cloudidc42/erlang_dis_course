# Part 61: CRDT Deep Dive

## สารบัญ
1. [CRDT คืออะไร](#crdt-theory)
2. [State-based CRDTs](#state-based)
3. [Operation-based CRDTs](#op-based)
4. [ตัวอย่าง: Shopping Cart CRDT](#shopping-cart)
5. [ตัวอย่าง: Collaborative Text Editing](#text-editing)
6. [CRDTs ใน Production](#production)

---

## CRDT คืออะไร

```
CRDT = Conflict-free Replicated Data Type

ปัญหา: replica หลายตัว update พร้อมกัน → conflict
CRDT solution: design data structure ที่ merge กันได้เสมอ

Properties:
1. Commutativity: A ∪ B = B ∪ A (ลำดับไม่สำคัญ)
2. Associativity: (A ∪ B) ∪ C = A ∪ (B ∪ C)
3. Idempotency: A ∪ A = A (apply ซ้ำได้)

Types:
- State-based (CvRDT): merge entire state
- Operation-based (CmRDT): merge operations

Use cases:
- Shopping carts (ใน Amazon)
- Collaborative docs (Google Docs)
- Distributed counters
- Online/offline sync
- Multi-master databases
```

---

## State-based CRDTs

```erlang
%% G-Counter: grow-only counter (distributed)
-module(g_counter).
-export([new/1, increment/2, value/1, merge/2]).

new(NodeId) ->
    #{node_id => NodeId, counts => #{NodeId => 0}}.

increment(#{node_id := NodeId, counts := Counts} = Counter, N) ->
    CurrentVal = maps:get(NodeId, Counts, 0),
    Counter#{counts := Counts#{NodeId := CurrentVal + N}}.

value(#{counts := Counts}) ->
    maps:fold(fun(_, V, Acc) -> Acc + V end, 0, Counts).

%% Merge: take max for each node's count
merge(#{counts := C1}, #{counts := C2}) ->
    MergedCounts = maps:fold(fun(Node, Val, Acc) ->
        Acc#{Node => max(Val, maps:get(Node, Acc, 0))}
    end, C1, C2),
    #{counts => MergedCounts}.

%% PN-Counter: increment and decrement
-module(pn_counter).
-export([new/1, increment/2, decrement/2, value/1, merge/2]).

new(NodeId) ->
    #{
        node_id => NodeId,
        positive => #{NodeId => 0},
        negative => #{NodeId => 0}
    }.

increment(#{node_id := N, positive := P} = C, By) ->
    C#{positive := P#{N => maps:get(N, P, 0) + By}}.

decrement(#{node_id := N, negative := Neg} = C, By) ->
    C#{negative := Neg#{N => maps:get(N, Neg, 0) + By}}.

value(#{positive := P, negative := Neg}) ->
    Sum = fun(Map) -> maps:fold(fun(_, V, A) -> A + V end, 0, Map) end,
    Sum(P) - Sum(Neg).

merge(#{positive := P1, negative := N1}, #{positive := P2, negative := N2}) ->
    MaxMerge = fun(M1, M2) ->
        maps:fold(fun(K, V, Acc) ->
            Acc#{K => max(V, maps:get(K, Acc, 0))}
        end, M1, M2)
    end,
    #{positive => MaxMerge(P1, P2), negative => MaxMerge(N1, N2)}.

%% 2P-Set: two-phase set (add, remove, no re-add)
-module(twop_set).
-export([new/0, add/2, remove/2, member/2, merge/2]).

new() -> #{added => #{}, removed => #{}}.

add(Elem, #{added := A} = S) ->
    S#{added := A#{Elem => true}}.

remove(Elem, #{removed := R} = S) ->
    S#{removed := R#{Elem => true}}.

member(Elem, #{added := A, removed := R}) ->
    maps:is_key(Elem, A) andalso not maps:is_key(Elem, R).

merge(#{added := A1, removed := R1}, #{added := A2, removed := R2}) ->
    #{
        added => maps:merge(A1, A2),
        removed => maps:merge(R1, R2)
    }.

%% OR-Set: observe-remove set (allows re-add)
-module(or_set).
-export([new/0, add/3, remove/2, member/2, merge/2]).

new() -> #{}.

%% Each element has unique tags for each add operation
add(Elem, NodeId, Set) ->
    Tag = {NodeId, erlang:unique_integer([monotonic])},
    Tags = maps:get(Elem, Set, #{}),
    Set#{Elem => Tags#{Tag => true}}.

remove(Elem, Set) ->
    maps:remove(Elem, Set).

member(Elem, Set) ->
    case maps:get(Elem, Set, #{}) of
        Tags when map_size(Tags) > 0 -> true;
        _ -> false
    end.

merge(Set1, Set2) ->
    maps:fold(fun(Elem, Tags2, Acc) ->
        Tags1 = maps:get(Elem, Acc, #{}),
        Acc#{Elem => maps:merge(Tags1, Tags2)}
    end, Set1, Set2).
```

---

## Operation-based CRDTs

```erlang
%% Op-based: broadcast operations to all replicas
%% Requires: exactly-once delivery

%% LWW-Register: Last-Write-Wins register
-module(lww_register).
-export([new/0, set/3, get/1, merge/2]).

new() -> #{value => undefined, timestamp => 0}.

set(Value, Timestamp, _Reg) ->
    #{value => Value, timestamp => Timestamp}.

get(#{value := V}) -> V.

merge(R1 = #{timestamp := T1}, R2 = #{timestamp := T2}) ->
    if T1 >= T2 -> R1;
    true -> R2
    end.

%% Usage
demo_lww() ->
    R1 = lww_register:new(),
    R2 = lww_register:new(),
    
    R1a = lww_register:set(<<"Alice">>, 1000, R1),
    R2a = lww_register:set(<<"Bob">>, 2000, R2),  %% later timestamp
    
    Merged = lww_register:merge(R1a, R2a),
    <<"Bob">> = lww_register:get(Merged).  %% Bob wins (higher timestamp)

%% MV-Register: Multi-value register (causal consistency)
-module(mv_register).
-export([new/1, set/3, get/1, merge/2]).

new(NodeId) ->
    #{node_id => NodeId, values => #{}, vector_clock => #{}}.

%% Vector clock: track causal history
set(Value, NodeId, #{vector_clock := VC} = Reg) ->
    NewVC = VC#{NodeId => maps:get(NodeId, VC, 0) + 1},
    Reg#{
        values => #{Value => NewVC},
        vector_clock => NewVC
    }.

get(#{values := Values}) ->
    %% Returns all concurrent values (conflict)
    maps:keys(Values).

merge(#{values := V1}, #{values := V2}) ->
    %% Keep values that aren't dominated by others
    AllValues = maps:merge(V1, V2),
    Concurrent = maps:filter(fun(Val, VC) ->
        not lists:any(fun({OtherVal, OtherVC}) ->
            OtherVal /= Val andalso dominates(OtherVC, VC)
        end, maps:to_list(AllValues))
    end, AllValues),
    #{values => Concurrent, vector_clock => max_vc(V1, V2)}.

dominates(VC1, VC2) ->
    %% VC1 dominates VC2 if every node's counter in VC1 >= VC2
    %% and at least one is strictly greater
    maps:fold(fun(Node, C2, {AllGe, AnyGt}) ->
        C1 = maps:get(Node, VC1, 0),
        {AllGe andalso C1 >= C2, AnyGt orelse C1 > C2}
    end, {true, false}, VC2).

max_vc(VC1, VC2) ->
    maps:fold(fun(Node, V, Acc) ->
        Acc#{Node => max(V, maps:get(Node, Acc, 0))}
    end, VC1, VC2).
```

---

## Shopping Cart CRDT

```erlang
%% Amazon-style shopping cart: OR-Set per item, quantity as PN-Counter

-module(cart_crdt).
-export([new/1, add_item/4, remove_item/2, update_quantity/4, to_list/1, merge/2]).

new(NodeId) ->
    #{node_id => NodeId, items => #{}}.

add_item(ItemId, Qty, #{node_id := N, items := Items} = Cart, Meta) ->
    Tag = {N, erlang:unique_integer([monotonic])},
    ItemEntry = maps:get(ItemId, Items, #{tags => #{}, quantity => 0, meta => Meta}),
    NewEntry = ItemEntry#{
        tags := (maps:get(tags, ItemEntry))#{Tag => true},
        quantity := maps:get(quantity, ItemEntry) + Qty
    },
    Cart#{items := Items#{ItemId => NewEntry}}.

remove_item(ItemId, #{items := Items} = Cart) ->
    %% Remove by clearing all current tags
    Cart#{items := maps:remove(ItemId, Items)}.

update_quantity(ItemId, NewQty, #{items := Items} = Cart, _NodeId) ->
    case maps:get(ItemId, Items, undefined) of
        undefined -> Cart;  %% item not in cart
        Entry ->
            Cart#{items := Items#{ItemId => Entry#{quantity := NewQty}}}
    end.

to_list(#{items := Items}) ->
    [{ItemId, maps:get(quantity, Entry), maps:get(meta, Entry)}
     || {ItemId, Entry} <- maps:to_list(Items),
        map_size(maps:get(tags, Entry)) > 0].  %% only active items

merge(#{items := I1}, #{items := I2}) ->
    MergedItems = maps:fold(fun(ItemId, Entry2, Acc) ->
        case maps:get(ItemId, Acc, undefined) of
            undefined ->
                Acc#{ItemId => Entry2};
            Entry1 ->
                MergedEntry = #{
                    tags => maps:merge(maps:get(tags, Entry1), maps:get(tags, Entry2)),
                    quantity => max(maps:get(quantity, Entry1), maps:get(quantity, Entry2)),
                    meta => maps:get(meta, Entry1)
                },
                Acc#{ItemId => MergedEntry}
        end
    end, I1, I2),
    #{items => MergedItems}.

%% Usage example
example() ->
    Cart1 = new(node1),
    Cart2 = new(node2),
    
    %% Both add items independently (offline)
    C1a = add_item(item_a, 2, Cart1, #{name => <<"Apple">>}),
    C2a = add_item(item_b, 1, Cart2, #{name => <<"Banana">>}),
    
    %% Sync when online
    Merged = merge(C1a, C2a),
    Items = to_list(Merged),
    %% → [{item_a, 2, ...}, {item_b, 1, ...}]
    Items.
```

---

## สรุป Part 61

| CRDT Type | Operation | Use Case |
|-----------|-----------|----------|
| G-Counter | +only | Page views |
| PN-Counter | +/- | Stock count |
| 2P-Set | add/remove | Removed items |
| OR-Set | add/remove/re-add | Shopping cart |
| LWW-Register | set (timestamp) | Last value |
| MV-Register | set (vector clock) | Collaborative edit |

**Conflict-free** = automatic merge, no coordination needed

---

*[← Part 60: Raft Consensus](part_60_raft.md) | [Part 62: Advanced Testing →](part_62_advanced_testing.md)*
