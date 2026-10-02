# Part 36: Database Access

## สารบัญ
1. [ETS — Erlang Term Storage](#ets--erlang-term-storage)
2. [DETS — Disk-Based ETS](#dets--disk-based-ets)
3. [Mnesia — Distributed Database](#mnesia--distributed-database)
4. [PostgreSQL กับ epgsql](#postgresql-กับ-epgsql)
5. [Connection Pooling กับ poolboy](#connection-pooling-กับ-poolboy)
6. [ตัวอย่างจริง: Repository Pattern](#ตัวอย่างจริง-repository-pattern)

---

## ETS — Erlang Term Storage

```erlang
%% ETS: in-memory, concurrent key-value store
%% ในตัว OTP, ไม่ต้อง deps
%% อ่าน/เขียนได้จากหลาย process พร้อมกัน (concurrent reads, serialized writes)
%% Data หายเมื่อ process ที่ owns ตาย

%% สร้าง ETS table
ets:new(my_table, [
    named_table,    %% register ชื่อ
    set,            %% type: set | bag | duplicate_bag | ordered_set
    public,         %% access: public | protected | private
    {keypos, 1},    %% position of key in tuple (default 1)
    {read_concurrency, true},   %% optimize for concurrent reads
    {write_concurrency, true}   %% optimize for concurrent writes
]).

%% set: ไม่ซ้ำ key, 1 value per key
%% bag: อาจซ้ำ key, หลาย values per key (tuples ต้องต่างกัน)
%% duplicate_bag: อาจซ้ำ key + value
%% ordered_set: sorted by key (ช้ากว่า set)

%% Insert
ets:insert(my_table, {key1, value1}).
ets:insert(my_table, {user, 1, "Alice", 30}).    %% tuple record
ets:insert_new(my_table, {key1, v}).              %% fail if key exists

%% Lookup
ets:lookup(my_table, key1).           %% [{key1, value1}]
ets:lookup_element(my_table, key1, 2). %% value1 (2nd element)

%% Match Spec
ets:match(my_table, {user, '$1', '$2', '_'}).  %% [[Id, Name] | ...]
ets:match_object(my_table, {user, '_', "Alice", '_'}).  %% full tuples
ets:select(my_table, MatchSpec).       %% powerful query

%% Update
ets:update_element(my_table, key1, {2, new_value}).
ets:update_counter(my_table, counter_key, 1).   %% atomic increment

%% Delete
ets:delete(my_table, key1).      %% delete by key
ets:delete_object(my_table, {key1, old_value}).
ets:delete_all_objects(my_table).

%% Table info
ets:info(my_table).
ets:info(my_table, size).    %% number of objects
ets:info(my_table, memory).  %% memory in words

%% Traversal
ets:first(my_table).         %% first key
ets:next(my_table, Key).     %% next key after Key
ets:foldl(fun({K,V}, Acc) -> [{K,V}|Acc] end, [], my_table).

%% Tab2list (caution: copies all data)
AllData = ets:tab2list(my_table).

%% Give ownership to another process
ets:give_away(my_table, NewOwnerPid, GiftData).

%% heir: table survives owner death
ets:new(my_table, [named_table, {heir, HeirPid, HeirData}]).
```

---

## DETS — Disk-Based ETS

```erlang
%% DETS: disk-based persistent version of ETS
%% ข้อจำกัด: max 2GB, ช้ากว่า ETS, ไม่มี ordered_set

dets:open_file(my_dets, [
    {file, "my_data.dets"},
    {type, set},
    {access, read_write}
]).

dets:insert(my_dets, {key, value}).
dets:lookup(my_dets, key).
dets:delete(my_dets, key).
dets:sync(my_dets).     %% flush to disk
dets:close(my_dets).

%% ETS + DETS: cache ใน memory, persist ไป disk
load_from_disk() ->
    ets:new(cache, [named_table, set, public]),
    dets:open_file(persist, [{file, "cache.dets"}]),
    dets:to_ets(persist, cache),  %% copy disk -> memory
    ok.

save_to_disk() ->
    ets:to_dets(cache, persist),  %% copy memory -> disk
    dets:sync(persist).
```

---

## Mnesia — Distributed Database

```erlang
%% Mnesia: distributed, in-memory (+ disk) DB
%% - ACID transactions
%% - Replication across nodes
%% - Secondary indexes
%% - Fragmented tables

%% Setup (ทำครั้งเดียว)
mnesia:create_schema([node()]).  %% สร้าง schema
mnesia:start().

%% Define table (record)
-record(user, {id, name, email, age}).

mnesia:create_table(user, [
    {attributes, record_info(fields, user)},
    {type, set},
    {ram_copies, [node()]},    %% in-memory (fast, lost on crash)
    %% {disc_copies, [node()]}, %% memory + disk (persistent)
    %% {disc_only_copies, [node()]} %% disk only (slow, persistent)
    {index, [email]}           %% secondary index on email
]).

%% For distributed: replicate on multiple nodes
mnesia:create_table(user, [
    {attributes, record_info(fields, user)},
    {ram_copies, [node1@host, node2@host]},
    {disc_copies, [node3@host]}  %% persistent copy
]).

%% Write
mnesia:transaction(fun() ->
    mnesia:write(#user{id=1, name="Alice", email="alice@ex.com", age=30})
end).

%% Read
{atomic, Users} = mnesia:transaction(fun() ->
    mnesia:read(user, 1)  %% by primary key
end).

%% Query (QLC - Query List Comprehension)
{atomic, Result} = mnesia:transaction(fun() ->
    qlc:e(qlc:q([ U || U <- mnesia:table(user), U#user.age > 25 ]))
end).

%% Index read
{atomic, Users} = mnesia:transaction(fun() ->
    mnesia:index_read(user, "alice@ex.com", #user.email)
end).

%% Delete
mnesia:transaction(fun() ->
    mnesia:delete({user, 1})
end).

%% Dirty operations (no transaction, faster, no ACID)
mnesia:dirty_write(#user{id=2, name="Bob"}).
mnesia:dirty_read(user, 2).
mnesia:dirty_delete({user, 2}).

%% Pattern matching
mnesia:dirty_match_object(#user{_ = '_', age = 30}).
%% หรือใช้ select
mnesia:dirty_select(user, [{#user{age='$1', _ = '_'}, [{'>', '$1', 25}], ['$_']}]).
```

---

## PostgreSQL กับ epgsql

```erlang
%% epgsql: PostgreSQL driver
%% deps: {epgsql, "4.7.1"}

%% Connect
{ok, Conn} = epgsql:connect(#{
    host => "localhost",
    port => 5432,
    username => "myapp",
    password => "password",
    database => "mydb",
    timeout => 5000,
    ssl => false
}).

%% Simple query
{ok, Columns, Rows} = epgsql:squery(Conn, "SELECT * FROM users").

%% Prepared statement (safe from SQL injection)
{ok, Stmt} = epgsql:parse(Conn, "find_user",
    "SELECT id, name, email FROM users WHERE id = $1", [int4]).

{ok, Cols, Rows} = epgsql:execute(Conn, Stmt, [UserId]).

%% ปลอดภัยกว่า equery:
{ok, Cols, Rows} = epgsql:equery(Conn,
    "SELECT * FROM users WHERE age > $1 AND active = $2",
    [25, true]).

%% Insert
{ok, RowsAffected} = epgsql:equery(Conn,
    "INSERT INTO users (name, email, age) VALUES ($1, $2, $3)",
    ["Alice", "alice@example.com", 30]).

%% Insert and get ID
{ok, _, [{Id}]} = epgsql:equery(Conn,
    "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
    ["Bob", "bob@example.com"]).

%% Transaction
epgsql:with_transaction(Conn, fun(C) ->
    {ok, _, _} = epgsql:equery(C, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", [100, FromId]),
    {ok, _, _} = epgsql:equery(C, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", [100, ToId]),
    ok
end).

%% Batch/Pipeline queries
{ok, Results} = epgsql:pipeline(Conn, [
    {equery, ["SELECT count(*) FROM users", []]},
    {equery, ["SELECT * FROM users LIMIT 10", []]}
]).

%% Types
%% Erlang -> PostgreSQL:
%% integer -> int4/int8
%% float -> float4/float8
%% binary -> text/varchar/bytea
%% true/false -> boolean
%% {{Y,M,D},{H,Mi,S}} -> timestamp
%% {Y,M,D} -> date
%% null -> NULL

%% Close
epgsql:close(Conn).
```

---

## Connection Pooling กับ poolboy

```erlang
%% poolboy: generic process pool
%% deps: {poolboy, "1.5.2"}

%% Pool worker (เป็น gen_server)
-module(db_worker).
-behaviour(gen_server).
-behaviour(poolboy_worker).

-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

start_link(Args) ->
    gen_server:start_link(?MODULE, Args, []).

init(Args) ->
    Host = maps:get(host, Args, "localhost"),
    Port = maps:get(port, Args, 5432),
    User = maps:get(username, Args, "postgres"),
    Pass = maps:get(password, Args, ""),
    DB = maps:get(database, Args, "postgres"),
    {ok, Conn} = epgsql:connect(#{
        host => Host, port => Port,
        username => User, password => Pass, database => DB
    }),
    {ok, #{conn => Conn}}.

handle_call({query, SQL, Params}, _From, #{conn := C} = State) ->
    Result = epgsql:equery(C, SQL, Params),
    {reply, Result, State}.

handle_cast(_, State) -> {noreply, State}.
handle_info(_, State) -> {noreply, State}.
terminate(_, #{conn := C}) -> epgsql:close(C).
code_change(_, S, _) -> {ok, S}.

%% Pool configuration
pool_config() ->
    DbArgs = #{
        host => "localhost",
        port => 5432,
        username => "myapp",
        password => "secret",
        database => "mydb"
    },
    [
        {name, {local, db_pool}},
        {worker_module, db_worker},
        {size, 10},           %% initial workers
        {max_overflow, 5},    %% extra workers under load
        {strategy, lifo}      %% lifo | fifo
    ].

start_pool() ->
    poolboy:start_link(pool_config(), DbArgs).

%% Use pool
query(SQL, Params) ->
    poolboy:transaction(db_pool, fun(Worker) ->
        gen_server:call(Worker, {query, SQL, Params})
    end).

%% ใน supervisor
#{
    id => db_pool,
    start => {poolboy, start_link, [pool_config(), #{...}]},
    type => supervisor,
    shutdown => infinity
}
```

---

## ตัวอย่างจริง: Repository Pattern

```erlang
%% ไฟล์: users_repo.erl
%% Repository pattern: database access layer

-module(users_repo).

-export([
    find/1, find_by_email/1, list/1,
    create/1, update/2, delete/1,
    exists/1
]).

-define(TABLE, users).

%% ====== Query functions ======

find(Id) ->
    case query("SELECT id, name, email, age, created_at FROM users WHERE id = $1", [Id]) of
        {ok, _, [Row]} -> {ok, row_to_map(Row)};
        {ok, _, []} -> {error, not_found}
    end.

find_by_email(Email) ->
    case query("SELECT id, name, email, age FROM users WHERE email = $1", [Email]) of
        {ok, _, [Row]} -> {ok, row_to_map(Row)};
        {ok, _, []} -> {error, not_found}
    end.

list(Opts) ->
    Page = maps:get(page, Opts, 1),
    Limit = maps:get(limit, Opts, 20),
    Offset = (Page - 1) * Limit,
    OrderBy = maps:get(order_by, Opts, "id"),
    Filter = maps:get(filter, Opts, #{}),

    {WhereClause, Params} = build_where(Filter),

    SQL = io_lib:format(
        "SELECT id, name, email, age FROM users ~s ORDER BY ~s LIMIT $~p OFFSET $~p",
        [WhereClause, OrderBy, length(Params) + 1, length(Params) + 2]
    ),

    case query(lists:flatten(SQL), Params ++ [Limit, Offset]) of
        {ok, _, Rows} -> {ok, [row_to_map(R) || R <- Rows]}
    end.

create(#{name := Name, email := Email} = Data) ->
    Age = maps:get(age, Data, null),
    case query(
        "INSERT INTO users (name, email, age) VALUES ($1, $2, $3) RETURNING id, name, email, age, created_at",
        [Name, Email, Age]
    ) of
        {ok, _, [Row]} -> {ok, row_to_map(Row)};
        {error, {error, error, <<"23505">>, _Msg, _Detail}} ->
            {error, email_already_exists}
    end.

update(Id, Changes) ->
    {ok, Existing} = find(Id),
    Updated = maps:merge(Existing, Changes),
    case query(
        "UPDATE users SET name = $1, email = $2, age = $3 WHERE id = $4 RETURNING id, name, email, age",
        [maps:get(name, Updated), maps:get(email, Updated), maps:get(age, Updated), Id]
    ) of
        {ok, _, [Row]} -> {ok, row_to_map(Row)};
        {ok, _, []} -> {error, not_found}
    end.

delete(Id) ->
    case query("DELETE FROM users WHERE id = $1", [Id]) of
        {ok, 1} -> ok;
        {ok, 0} -> {error, not_found}
    end.

exists(Id) ->
    case query("SELECT 1 FROM users WHERE id = $1 LIMIT 1", [Id]) of
        {ok, _, [_]} -> true;
        {ok, _, []} -> false
    end.

%% ====== Internal ======

query(SQL, Params) ->
    db_pool:query(SQL, Params).

%% Map PostgreSQL row tuple to Erlang map
row_to_map({Id, Name, Email, Age, CreatedAt}) ->
    #{id => Id, name => Name, email => Email,
      age => Age, created_at => CreatedAt};
row_to_map({Id, Name, Email, Age}) ->
    #{id => Id, name => Name, email => Email, age => Age};
row_to_map({Id, Name, Email}) ->
    #{id => Id, name => Name, email => Email}.

build_where(Filter) when map_size(Filter) =:= 0 ->
    {"", []};
build_where(Filter) ->
    {Clauses, Params} = maps:fold(
        fun(Key, Value, {Cs, Ps}) ->
            N = length(Ps) + 1,
            Clause = atom_to_list(Key) ++ " = $" ++ integer_to_list(N),
            {[Clause | Cs], Ps ++ [Value]}
        end,
        {[], []},
        Filter
    ),
    {"WHERE " ++ string:join(lists:reverse(Clauses), " AND "), Params}.

%% ใช้งาน
{ok, User} = users_repo:find(1).
{ok, Users} = users_repo:list(#{page => 1, limit => 10}).
{ok, Users2} = users_repo:list(#{filter => #{age => 30}}).
{ok, NewUser} = users_repo:create(#{name => <<"Alice">>, email => <<"a@b.com">>}).
{ok, Updated} = users_repo:update(1, #{age => 31}).
ok = users_repo:delete(1).
```

---

## สรุป Part 36

| Storage | Type | ใช้เมื่อ |
|---------|------|---------|
| ETS | In-memory | Cache, fast lookup |
| DETS | Disk-based | Small persistent store |
| Mnesia | Distributed DB | Complex queries, replication |
| PostgreSQL | RDBMS | Full SQL, ACID, production |

| ETS type | Key uniqueness |
|----------|---------------|
| set | unique key |
| ordered_set | unique, sorted |
| bag | duplicate keys, unique values |
| duplicate_bag | fully duplicate |

---

*[← Part 35: Cowboy HTTP Server](part_35_cowboy.md) | [Part 37: Advanced OTP Patterns →](part_37_advanced_otp.md)*
