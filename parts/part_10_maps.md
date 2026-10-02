# Part 10: Maps — โครงสร้างข้อมูล Key-Value

## สารบัญ
1. [Maps Overview](#maps-overview)
2. [Creating Maps](#creating-maps)
3. [Accessing Map Values](#accessing-map-values)
4. [Updating Maps](#updating-maps)
5. [maps Module Functions](#maps-module-functions)
6. [Map Pattern Matching](#map-pattern-matching)
7. [Map Comprehensions](#map-comprehensions)
8. [Nested Maps](#nested-maps)
9. [Maps ใน Functions](#maps-ใน-functions)
10. [Maps vs Records vs Proplists](#maps-vs-records-vs-proplists)
11. [ตัวอย่างจริง: Configuration System](#ตัวอย่างจริง-configuration-system)
12. [สรุป](#สรุป)

---

## Maps Overview

Maps เป็น key-value data structure ที่เพิ่มเข้ามาใน Erlang/OTP 17:

```erlang
%% Map = #{key => value, ...}
%% Keys สามารถเป็น term ใดๆ ก็ได้
%% Values สามารถเป็น term ใดๆ ก็ได้

%% ข้อดีของ Maps:
%% + Dynamic keys (ไม่ต้อง define ล่วงหน้า)
%% + Pattern matching ที่ยืดหยุ่น
%% + Native JSON-like structure
%% + ประสิทธิภาพดีสำหรับ lookup O(log n)

Empty = #{}.
Simple = #{name => "Alice", age => 30}.
Mixed = #{
    1 => "one",
    hello => <<"world">>,
    [1,2,3] => tuple_key
}.
```

---

## Creating Maps

```erlang
%% Direct literal
M1 = #{a => 1, b => 2, c => 3}.

%% Empty map
M2 = #{}.

%% From list of pairs
M3 = maps:from_list([{a, 1}, {b, 2}, {c, 3}]).
%% = #{a => 1, b => 2, c => 3}

%% From keys with same value
M4 = maps:from_keys([a, b, c], 0).
%% = #{a => 0, b => 0, c => 0}

%% Merge two maps
M5 = maps:merge(#{a => 1, b => 2}, #{b => 99, c => 3}).
%% = #{a => 1, b => 99, c => 3} (second map wins on conflict)

%% maps:merge_with/3 - merge with custom conflict resolver (OTP 24+)
maps:merge_with(
    fun(_Key, V1, V2) -> V1 + V2 end,
    #{a => 1, b => 2},
    #{b => 10, c => 3}
).
%% = #{a => 1, b => 12, c => 3}
```

---

## Accessing Map Values

```erlang
M = #{name => "Alice", age => 30, email => "alice@example.com"}.

%% maps:get/2 - ถ้าไม่มี key จะ throw exception
maps:get(name, M).           %% = "Alice"
maps:get(age, M).            %% = 30
%% maps:get(phone, M).       %% throws {badkey, phone}!

%% maps:get/3 - กับ default value (safe)
maps:get(phone, M, undefined).   %% = undefined
maps:get(name, M, "Unknown").    %% = "Alice"

%% maps:find/2 - return {ok, Value} | error
maps:find(name, M).     %% = {ok, "Alice"}
maps:find(phone, M).    %% = error

%% Pattern matching (ดีที่สุด)
#{name := Name} = M.
Name.  %% = "Alice"

#{name := N, age := A} = M.
N.  %% = "Alice"
A.  %% = 30
```

---

## Updating Maps

```erlang
M = #{name => "Alice", age => 30}.

%% Update existing key (:= operator)
M1 = M#{age := 31}.
%% = #{name => "Alice", age => 31}
%% ถ้า age ไม่มี -> throw exception!

%% Update or add key (=> operator)
M2 = M#{age => 31, email => "alice@new.com"}.
%% = #{name => "Alice", age => 31, email => "alice@new.com"}

%% Remove key
M3 = maps:remove(age, M).
%% = #{name => "Alice"}

%% maps:put/3 - เหมือน => operator
M4 = maps:put(email, "alice@example.com", M).
%% = #{name => "Alice", age => 30, email => "alice@example.com"}

%% maps:update/3 - update existing key (ถ้าไม่มีจะ throw)
M5 = maps:update(age, 31, M).
%% = #{name => "Alice", age => 31}

%% maps:update_with/3 - update ด้วย function
M6 = maps:update_with(age, fun(V) -> V + 1 end, M).
%% = #{name => "Alice", age => 31}

%% maps:update_with/4 - update หรือ default ถ้าไม่มี
M7 = maps:update_with(score, fun(V) -> V + 10 end, 0, M).
%% = #{name => "Alice", age => 30, score => 0}
%% (score ไม่มี -> ใส่ default 0)

%% maps:put สำหรับ update + add
M8 = maps:put(name, "Bob", M).
%% = #{name => "Bob", age => 30}
```

---

## maps Module Functions

### Basic Operations

```erlang
M = #{a => 1, b => 2, c => 3, d => 4}.

%% Size
maps:size(M).     %% = 4
map_size(M).      %% = 4 (BIF, เร็วกว่า)

%% Keys and Values
maps:keys(M).     %% = [a, b, c, d]
maps:values(M).   %% = [1, 2, 3, 4]

%% Check key existence
maps:is_key(a, M).       %% = true
maps:is_key(z, M).       %% = false

%% Convert
maps:to_list(M).          %% = [{a,1}, {b,2}, {c,3}, {d,4}]
maps:from_list([{a,1}]).  %% = #{a => 1}
```

### Higher-Order Functions

```erlang
M = #{a => 1, b => 2, c => 3}.

%% maps:map/2 - transform values
maps:map(fun(_K, V) -> V * 2 end, M).
%% = #{a => 2, b => 4, c => 6}

%% maps:filter/2 - keep pairs that satisfy predicate
maps:filter(fun(_K, V) -> V > 1 end, M).
%% = #{b => 2, c => 3}

%% maps:filtermap/2 (OTP 24+) - filter and transform
maps:filtermap(
    fun(_K, V) when V > 1 -> {true, V * 10};
       (_, _) -> false
    end,
    M
).
%% = #{b => 20, c => 30}

%% maps:fold/3 - fold over key-value pairs
maps:fold(fun(_K, V, Acc) -> Acc + V end, 0, M).
%% = 6 (sum of values)

%% maps:foreach/2 - side effects
maps:foreach(fun(K, V) -> io:format("~p => ~p~n", [K, V]) end, M).

%% maps:map vs maps:fold example
%% Invert map (swap keys and values)
invert_map(Map) ->
    maps:fold(
        fun(K, V, Acc) -> Acc#{V => K} end,
        #{},
        Map
    ).

invert_map(#{a => 1, b => 2, c => 3}).
%% = #{1 => a, 2 => b, 3 => c}
```

### Intersection/Difference

```erlang
M1 = #{a => 1, b => 2, c => 3}.
M2 = #{b => 20, c => 30, d => 40}.

%% maps:intersect/2 - keys in both (use second value)
maps:intersect(M1, M2).
%% = #{b => 20, c => 30}

%% maps:intersect_with/3 - intersect with merge function
maps:intersect_with(fun(_K, V1, V2) -> V1 + V2 end, M1, M2).
%% = #{b => 22, c => 33}

%% Difference (ไม่มี built-in แต่ implement ได้)
map_difference(M1, M2) ->
    maps:filter(fun(K, _) -> not maps:is_key(K, M2) end, M1).

map_difference(M1, M2).
%% = #{a => 1}
```

---

## Map Pattern Matching

```erlang
%% Pattern matching ใน function
get_name(#{name := Name}) -> Name.
get_name(#{<<"name">> := Name}) -> Name.

%% กับ guards
process(#{role := admin} = User) ->
    admin_process(User);
process(#{role := user, active := true} = User) ->
    user_process(User);
process(#{active := false}) ->
    {error, inactive}.

%% Partial match - ไม่ต้อง match ทุก key
print_info(#{name := Name, age := Age}) ->
    io:format("~s is ~p years old~n", [Name, Age]).

print_info(#{name => "Alice", age => 30, email => "alice@..."}).
%% "Alice is 30 years old"

%% ใน case
route(#{method := <<"GET">>, path := <<"/users">>}) ->
    list_users();
route(#{method := <<"GET">>, path := <<"/users/", Id/binary>>}) ->
    get_user(binary_to_integer(Id));
route(#{method := <<"POST">>, path := <<"/users">>}) ->
    create_user();
route(_) ->
    {error, not_found}.
```

---

## Map Comprehensions

OTP 26 เพิ่ม Map Comprehensions:

```erlang
%% Map comprehension syntax
%% #{Key => Value || Key := Value <- Map [, Filter]}

M = #{a => 1, b => 2, c => 3, d => 4}.

%% Double all values
#{K => V*2 || K := V <- M}.
%% = #{a => 2, b => 4, c => 6, d => 8}

%% Filter and transform
#{K => V*10 || K := V <- M, V > 2}.
%% = #{c => 30, d => 40}

%% Create map from list
#{X => X*X || X <- [1,2,3,4,5]}.
%% = #{1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

%% Swap keys and values
#{V => K || K := V <- #{a=>1, b=>2}}.
%% = #{1 => a, 2 => b}

%% From zip of two lists
Keys = [a, b, c].
Values = [1, 2, 3].
#{K => V || K <- Keys, V <- Values, hd([K]) =:= hd([K])}.
%% ไม่ได้ทำ zip จริงๆ ต้องใช้ lists:zip

%% Map comprehension + list comprehension
Zipped = lists:zip([a,b,c], [1,2,3]).
#{K => V || {K, V} <- Zipped}.
%% = #{a => 1, b => 2, c => 3}
```

---

## Nested Maps

```erlang
%% สร้าง nested map
Config = #{
    database => #{
        host => <<"localhost">>,
        port => 5432,
        name => <<"mydb">>,
        pool => #{
            size => 10,
            overflow => 5
        }
    },
    cache => #{
        backend => redis,
        host => <<"localhost">>,
        port => 6379,
        ttl => 3600
    },
    web => #{
        port => 8080,
        host => <<"0.0.0.0">>,
        ssl => false
    }
}.

%% Access nested values
DbHost = maps:get(host, maps:get(database, Config)).
%% = <<"localhost">>

%% Pattern matching nested
#{database := #{host := DbH, port := DbP}} = Config.
DbH.  %% = <<"localhost">>
DbP.  %% = 5432

%% Update nested value
update_db_port(Config, NewPort) ->
    Db = maps:get(database, Config),
    NewDb = Db#{port := NewPort},
    Config#{database := NewDb}.

%% Utility function สำหรับ deep access
get_in(Map, []) -> Map;
get_in(Map, [Key | Rest]) ->
    case maps:find(Key, Map) of
        {ok, Value} -> get_in(Value, Rest);
        error -> undefined
    end.

get_in(Config, [database, host]).
%% = <<"localhost">>

get_in(Config, [database, pool, size]).
%% = 10

get_in(Config, [non_existent, key]).
%% = undefined

%% Utility function สำหรับ deep update
put_in(Map, [Key], Value) ->
    Map#{Key => Value};
put_in(Map, [Key | Rest], Value) ->
    Nested = maps:get(Key, Map, #{}),
    Map#{Key => put_in(Nested, Rest, Value)}.

put_in(Config, [database, port], 5433).
%% = Config with database.port = 5433
```

---

## Maps ใน Functions

```erlang
%% ตัวอย่าง: Process HTTP Request
handle_request(Request) ->
    #{method := Method, path := Path, body := Body} = Request,
    route(Method, Path, Body).

route(<<"GET">>, <<"/users">>, _) ->
    {ok, list_users()};
route(<<"GET">>, <<"/users/", Id/binary>>, _) ->
    UserId = binary_to_integer(Id),
    get_user(UserId);
route(<<"POST">>, <<"/users">>, Body) ->
    Params = decode_json(Body),
    create_user(Params);
route(Method, Path, _) ->
    {error, {not_found, Method, Path}}.

%% ตัวอย่าง: Config-driven behavior
make_handler(Config) ->
    Timeout = maps:get(timeout, Config, 5000),
    MaxRetries = maps:get(max_retries, Config, 3),
    fun(Request) ->
        handle_with_retry(Request, MaxRetries, Timeout)
    end.

%% ตัวอย่าง: Accumulate with maps
word_count(Text) ->
    Words = string:tokens(binary_to_list(Text), " \t\n"),
    lists:foldl(
        fun(Word, Counts) ->
            maps:update_with(Word, fun(V) -> V + 1 end, 1, Counts)
        end,
        #{},
        Words
    ).

word_count(<<"hello world hello erlang">>).
%% = #{"hello" => 2, "world" => 1, "erlang" => 1}
```

---

## Maps vs Records vs Proplists

```erlang
%% Comparison: User data in 3 ways

%% 1. Proplist (ดั้งเดิม, ใช้น้อยในโค้ดใหม่)
UserProplist = [{name, "Alice"}, {age, 30}, {email, "alice@..."}].
proplists:get_value(name, UserProplist).
%% + ง่าย, compatible กับโค้ดเก่า
%% - ช้า O(n) lookup, ไม่มี type checking

%% 2. Record (compile-time struct)
-record(user, {name, age, email}).
UserRecord = #user{name = "Alice", age = 30, email = "alice@..."}.
UserRecord#user.name.
%% + เร็วที่สุด (compile-time indexing), type-safe
%% - ต้อง define ล่วงหน้า, ไม่ยืดหยุ่น runtime

%% 3. Map (runtime key-value)
UserMap = #{name => "Alice", age => 30, email => "alice@..."}.
maps:get(name, UserMap).
%% + ยืดหยุ่น, JSON-friendly, dynamic
%% - ช้ากว่า record เล็กน้อย, ไม่มี compile-time check

%% คำแนะนำ:
%% Records: OTP internal state, performance-critical, fixed schema
%% Maps: API data, config, external interfaces, flexible schema
%% Proplists: options/flags, backward compatibility

%% เปรียบเทียบ performance
bench_record() ->
    R = #user{name = "Alice", age = 30},
    timer:tc(fun() ->
        lists:foreach(fun(_) -> R#user.name end, lists:seq(1, 1000000))
    end).

bench_map() ->
    M = #{name => "Alice", age => 30},
    timer:tc(fun() ->
        lists:foreach(fun(_) -> maps:get(name, M) end, lists:seq(1, 1000000))
    end).
%% Record: ~50ms, Map: ~100ms (approximate)
```

---

## ตัวอย่างจริง: Configuration System

```erlang
%% ไฟล์: config_manager.erl
-module(config_manager).
-export([
    load/1,
    get/2,
    get/3,
    set/3,
    merge/2,
    validate/2,
    to_env_vars/1
]).

-type config() :: #{atom() | binary() => term()}.

%% Load config from file
-spec load(file:filename()) -> {ok, config()} | {error, term()}.
load(Filename) ->
    case file:consult(Filename) of
        {ok, [Terms]} when is_map(Terms) ->
            {ok, Terms};
        {ok, Terms} when is_list(Terms) ->
            {ok, maps:from_list(Terms)};
        {error, _} = Error ->
            Error
    end.

%% Get value with dot-notation path
-spec get(config(), string() | [atom()]) -> term() | undefined.
get(Config, Path) when is_list(Path), is_integer(hd(Path)) ->
    %% String path like "database.host"
    Keys = [list_to_atom(K) || K <- string:tokens(Path, ".")],
    get_nested(Config, Keys);
get(Config, Path) when is_list(Path) ->
    get_nested(Config, Path).

%% Get with default
-spec get(config(), string() | [atom()], term()) -> term().
get(Config, Path, Default) ->
    case get(Config, Path) of
        undefined -> Default;
        Value -> Value
    end.

%% Set value
-spec set(config(), [atom()], term()) -> config().
set(Config, [Key], Value) ->
    Config#{Key => Value};
set(Config, [Key | Rest], Value) ->
    Nested = maps:get(Key, Config, #{}),
    Config#{Key => set(Nested, Rest, Value)}.

%% Merge configs (second overrides first)
-spec merge(config(), config()) -> config().
merge(Base, Override) ->
    maps:fold(
        fun(K, V, Acc) when is_map(V) ->
            BaseV = maps:get(K, Acc, #{}),
            case is_map(BaseV) of
                true -> Acc#{K => merge(BaseV, V)};
                false -> Acc#{K => V}
            end;
           (K, V, Acc) ->
            Acc#{K => V}
        end,
        Base,
        Override
    ).

%% Validate config against schema
-spec validate(config(), map()) -> ok | {error, [term()]}.
validate(Config, Schema) ->
    Errors = maps:fold(
        fun(Key, {required, Type}, Acc) ->
            case maps:find(Key, Config) of
                {ok, V} ->
                    case check_type(V, Type) of
                        ok -> Acc;
                        {error, E} -> [{invalid_type, Key, E} | Acc]
                    end;
                error ->
                    [{missing_required, Key} | Acc]
            end;
           (Key, {optional, Type, Default}, Acc) ->
            case maps:find(Key, Config) of
                {ok, V} ->
                    case check_type(V, Type) of
                        ok -> Acc;
                        {error, E} -> [{invalid_type, Key, E} | Acc]
                    end;
                error ->
                    [{missing_optional, Key, Default} | Acc]
            end
        end,
        [],
        Schema
    ),
    case [E || E <- Errors, element(1, E) /= missing_optional] of
        [] -> ok;
        Errors2 -> {error, Errors2}
    end.

%% Convert to environment variables format
-spec to_env_vars(config()) -> [{string(), string()}].
to_env_vars(Config) ->
    to_env_vars(Config, "").

to_env_vars(Config, Prefix) ->
    maps:fold(
        fun(K, V, Acc) when is_map(V) ->
            NewPrefix = case Prefix of
                "" -> atom_to_list(K);
                P -> P ++ "_" ++ atom_to_list(K)
            end,
            Acc ++ to_env_vars(V, string:to_upper(NewPrefix));
           (K, V, Acc) ->
            EnvKey = case Prefix of
                "" -> string:to_upper(atom_to_list(K));
                P -> P ++ "_" ++ string:to_upper(atom_to_list(K))
            end,
            [{EnvKey, term_to_env_value(V)} | Acc]
        end,
        [],
        Config
    ).

%%----

get_nested(_, []) -> undefined;
get_nested(Map, [Key]) ->
    maps:get(Key, Map, undefined);
get_nested(Map, [Key | Rest]) ->
    case maps:find(Key, Map) of
        {ok, Value} when is_map(Value) -> get_nested(Value, Rest);
        {ok, _} -> undefined;
        error -> undefined
    end.

check_type(V, integer) -> case is_integer(V) of true -> ok; false -> {error, not_integer} end;
check_type(V, float) -> case is_float(V) of true -> ok; false -> {error, not_float} end;
check_type(V, binary) -> case is_binary(V) of true -> ok; false -> {error, not_binary} end;
check_type(V, boolean) -> case is_boolean(V) of true -> ok; false -> {error, not_boolean} end;
check_type(V, atom) -> case is_atom(V) of true -> ok; false -> {error, not_atom} end;
check_type(_, _) -> ok.

term_to_env_value(V) when is_binary(V) -> binary_to_list(V);
term_to_env_value(V) when is_integer(V) -> integer_to_list(V);
term_to_env_value(V) when is_float(V) -> float_to_list(V);
term_to_env_value(V) when is_atom(V) -> atom_to_list(V);
term_to_env_value(V) -> lists:flatten(io_lib:format("~p", [V])).
```

```erlang
%% ตัวอย่างการใช้งาน
Config = #{
    database => #{
        host => <<"localhost">>,
        port => 5432,
        name => <<"mydb">>
    },
    web => #{
        port => 8080
    }
}.

config_manager:get(Config, "database.host").
%% = <<"localhost">>

config_manager:get(Config, [database, port]).
%% = 5432

config_manager:get(Config, "missing.key", <<"default">>).
%% = <<"default">>

Override = #{database => #{port => 5433}}.
Merged = config_manager:merge(Config, Override).
config_manager:get(Merged, [database, port]).
%% = 5433 (overridden)
config_manager:get(Merged, [database, host]).
%% = <<"localhost">> (from base)

config_manager:to_env_vars(Config).
%% = [{"DATABASE_HOST", "localhost"}, {"DATABASE_PORT", "5432"}, ...]
```

---

## สรุป Part 10

| Operation | Syntax | Notes |
|-----------|--------|-------|
| Create | `#{k => v}` | Literal |
| Access | `maps:get(k, M)` | Throws if missing |
| Safe access | `maps:get(k, M, Def)` | With default |
| Pattern match | `#{k := Var} = M` | Bind key |
| Update | `M#{k := v}` | Key must exist |
| Add/update | `M#{k => v}` | Create or update |
| Remove | `maps:remove(k, M)` | Returns new map |
| Size | `map_size(M)` | BIF |
| Keys | `maps:keys(M)` | Returns list |
| Map fn | `maps:map(F, M)` | Transform values |
| Filter | `maps:filter(P, M)` | Keep pairs |
| Fold | `maps:fold(F, I, M)` | Reduce |

---

*[← Part 9: Tuples and Records](part_09_tuples_records.md) | [Part 11: Strings and Binaries →](part_11_strings_binaries.md)*
