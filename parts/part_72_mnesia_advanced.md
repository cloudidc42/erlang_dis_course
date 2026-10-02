# Part 72: Advanced Mnesia — Distributed Database Patterns

## สารบัญ
1. [Mnesia Architecture](#architecture)
2. [Schema Design](#schema)
3. [Transactions และ Locking](#transactions)
4. [Replication Strategies](#replication)
5. [Fragmented Tables](#fragmentation)
6. [ตัวอย่างจริง: Distributed Session Store](#session-store)

---

## Mnesia Architecture

```
Mnesia = distributed, in-memory database built into OTP
- Tables: RAM, disk, or both
- Transactions: ACID across nodes
- No separate server process needed
- Written in Erlang, lives in BEAM

Storage types:
  ram_copies   — in-memory, lost on crash
  disc_copies  — in-memory + persisted to disk log
  disc_only_copies — disk only, slower reads

Use cases:
  - Session state
  - Configuration data
  - Routing tables
  - Small/medium datasets needing distribution
  - NOT for large data (use PostgreSQL/Cassandra)
```

---

## Schema Design

```erlang
%% Define records as Mnesia table schemas

-record(user, {
    id,           %% primary key (first field after table name)
    email,
    name,
    created_at,
    status        %% active | suspended | deleted
}).

-record(session, {
    token,        %% key
    user_id,
    ip_address,
    expires_at,
    data          %% arbitrary map
}).

-record(config, {
    key,          %% {app, setting} tuple as key
    value,
    updated_at
}).

%% Create schema on disk (one-time setup per node)
setup_schema() ->
    %% Stop mnesia first if running
    mnesia:stop(),
    
    %% Create schema on all nodes
    Nodes = [node() | nodes()],
    case mnesia:create_schema(Nodes) of
        ok -> ok;
        {error, {_, {already_exists, _}}} -> ok  %% already set up
    end,
    
    mnesia:start().

%% Create tables after schema exists
create_tables() ->
    Nodes = [node() | nodes()],
    
    mnesia:create_table(user, [
        {attributes, record_info(fields, user)},
        {disc_copies, Nodes},    %% replicated on all nodes + persisted
        {index, [email, status]} %% secondary indexes
    ]),
    
    mnesia:create_table(session, [
        {attributes, record_info(fields, session)},
        {ram_copies, Nodes},     %% RAM only (sessions are ephemeral)
        {index, [user_id]}
    ]),
    
    mnesia:create_table(config, [
        {attributes, record_info(fields, config)},
        {disc_copies, Nodes},
        {type, set}              %% set (default), bag, or ordered_set
    ]),
    
    %% Wait for tables to be available on this node
    mnesia:wait_for_tables([user, session, config], 10000).

%% Add table copy to new node
add_node_to_cluster(NewNode) ->
    %% Connect to new node
    net_adm:ping(NewNode),
    
    %% Add schema copy
    mnesia:change_config(extra_db_nodes, [NewNode]),
    
    %% Replicate each table
    Tables = [user, session, config],
    [mnesia:add_table_copy(T, NewNode, disc_copies) || T <- Tables],
    
    ok.
```

---

## CRUD Operations

```erlang
-module(mnesia_ops).
-export([create_user/3, get_user/1, find_by_email/1, update_status/2, delete_user/1]).
-export([list_users/1, count_users/0]).

%% Create (write in transaction)
create_user(Id, Email, Name) ->
    User = #user{
        id = Id,
        email = Email,
        name = Name,
        created_at = erlang:system_time(second),
        status = active
    },
    F = fun() ->
        %% Check uniqueness first
        case mnesia:index_read(user, Email, #user.email) of
            [] ->
                mnesia:write(User),
                {ok, User};
            [_Existing] ->
                {error, email_taken}
        end
    end,
    mnesia:transaction(F).

%% Read by primary key
get_user(Id) ->
    case mnesia:dirty_read(user, Id) of  %% dirty_read = no transaction overhead
        [#user{} = User] -> {ok, User};
        [] -> not_found
    end.

%% Read by secondary index
find_by_email(Email) ->
    case mnesia:dirty_index_read(user, Email, #user.email) of
        [#user{} = User] -> {ok, User};
        [] -> not_found
    end.

%% Update
update_status(UserId, NewStatus) ->
    F = fun() ->
        case mnesia:wread({user, UserId}) of  %% wread = read with write lock
            [User] ->
                mnesia:write(User#user{status = NewStatus}),
                ok;
            [] ->
                {error, not_found}
        end
    end,
    mnesia:transaction(F).

%% Delete
delete_user(UserId) ->
    mnesia:transaction(fun() ->
        %% Cascade: delete user's sessions too
        Sessions = mnesia:index_read(session, UserId, #session.user_id),
        [mnesia:delete({session, S#session.token}) || S <- Sessions],
        mnesia:delete({user, UserId})
    end).

%% List with filter (match spec)
list_users(Status) ->
    MatchSpec = [{#user{status = Status, _ = '_'}, [], ['$_']}],
    mnesia:dirty_select(user, MatchSpec).

%% Count (expensive on large tables!)
count_users() ->
    mnesia:table_info(user, size).

%% Efficient range query with ordered_set
%% (requires {type, ordered_set})
range_query(StartId, EndId) ->
    mnesia:dirty_select(user, [
        {#user{id = '$1', _ = '_'},
         [{'>=', '$1', StartId}, {'=<', '$1', EndId}],
         ['$_']}
    ]).
```

---

## Transactions และ Locking

```erlang
%% Transaction isolation: mnesia uses optimistic concurrency
%% - Reads: shared locks
%% - Writes: exclusive locks
%% - Conflicts: retry automatically (up to 40 times by default)

%% Nested transactions (inner transaction joins outer)
nested_transaction() ->
    mnesia:transaction(fun() ->
        create_user_internal(1, "alice@ex.com", "Alice"),
        mnesia:transaction(fun() ->
            %% This joins the outer transaction
            create_session_internal("token1", 1)
        end)
    end).

%% Activity API: run with specific access mode
activity_examples() ->
    %% Transaction: full ACID
    mnesia:activity(transaction, fun() ->
        mnesia:read({user, 1})
    end),
    
    %% Sync transaction: wait for all replicas to commit
    mnesia:activity(sync_transaction, fun() ->
        mnesia:write(#user{id = 2, name = "Bob"})
    end),
    
    %% Async dirty: no locks, no transaction, fastest
    mnesia:activity(async_dirty, fun() ->
        mnesia:read({user, 1})
    end),
    
    %% Sync dirty: dirty but waits for disk write
    mnesia:activity(sync_dirty, fun() ->
        mnesia:write(#user{id = 3, name = "Carol"})
    end).

%% Manual lock management
explicit_locks() ->
    mnesia:transaction(fun() ->
        %% Acquire table write lock (blocks all readers/writers)
        mnesia:lock({table, user}, write),
        
        %% Read all users
        Users = mnesia:match_object(#user{_ = '_'}),
        
        %% Batch update
        [mnesia:write(U#user{status = suspended}) || U <- Users],
        
        ok
    end).

%% Handle transaction conflicts
safe_transfer(FromId, ToId, Amount) ->
    F = fun() ->
        [From] = mnesia:wread({account, FromId}),
        [To] = mnesia:wread({account, ToId}),
        
        case From#account.balance >= Amount of
            false -> mnesia:abort({insufficient_funds, FromId});
            true ->
                mnesia:write(From#account{balance = From#account.balance - Amount}),
                mnesia:write(To#account{balance = To#account.balance + Amount}),
                ok
        end
    end,
    
    case mnesia:transaction(F) of
        {atomic, ok} -> ok;
        {aborted, {insufficient_funds, _} = Reason} -> {error, Reason};
        {aborted, Reason} -> {error, Reason}
    end.
```

---

## Replication Strategies

```erlang
%% Table types by node role
setup_cluster_tables() ->
    AllNodes = mnesia:system_info(running_db_nodes),
    
    %% Core nodes replicate everything
    CoreNodes = get_core_nodes(),
    
    %% Edge nodes only cache hot data
    EdgeNodes = get_edge_nodes(),
    
    %% Full replication on core nodes
    mnesia:create_table(orders, [
        {attributes, record_info(fields, order)},
        {disc_copies, CoreNodes}
    ]),
    
    %% RAM cache on edge nodes for fast reads
    [mnesia:add_table_copy(orders, N, ram_copies) || N <- EdgeNodes],
    
    ok.

%% Subscribe to table changes
subscribe_to_changes() ->
    mnesia:subscribe({table, user, detailed}),
    
    receive_loop().

receive_loop() ->
    receive
        {mnesia_table_event, {write, user, NewRecord, OldRecords, ActivityId}} ->
            logger:info("User updated", #{new => NewRecord, old => OldRecords}),
            receive_loop();
        {mnesia_table_event, {delete, user, {user, Id}, _OldRecords, _}} ->
            logger:info("User deleted", #{id => Id}),
            receive_loop()
    end.

%% Handle node failures
setup_node_monitoring() ->
    mnesia:subscribe(system),
    
    receive
        {mnesia_system_event, {mnesia_down, Node}} ->
            logger:error("Mnesia node down", #{node => Node}),
            handle_node_down(Node);
        {mnesia_system_event, {mnesia_up, Node}} ->
            logger:info("Mnesia node back up", #{node => Node}),
            handle_node_recovery(Node)
    end.

handle_node_down(DownNode) ->
    %% Check if we still have quorum
    RunningNodes = mnesia:system_info(running_db_nodes),
    AllNodes = mnesia:system_info(db_nodes),
    Quorum = length(AllNodes) div 2 + 1,
    
    case length(RunningNodes) >= Quorum of
        true ->
            logger:info("Quorum maintained, continuing", #{running => RunningNodes});
        false ->
            logger:critical("Lost quorum! Going read-only"),
            %% Switch to read-only mode
            application:set_env(my_app, read_only, true)
    end.
```

---

## Fragmented Tables

```erlang
%% Fragmentation: split large tables across nodes
%% Useful when table is too large for single node RAM

setup_fragmented_table() ->
    mnesia:create_table(big_events, [
        {attributes, record_info(fields, event)},
        {frag_properties, [
            {node_pool, [node1@host, node2@host, node3@host]},
            {n_fragments, 12},         %% 12 fragments (4 per node)
            {n_disc_copies, 2},        %% 2 replicas per fragment
            {n_ram_copies, 1}
        ]}
    ]).

%% Write to fragmented table (transparent)
write_event(Event) ->
    mnesia:activity(sync_dirty, fun() ->
        mnesia:write(big_events, Event, write)
    end, [], mnesia_frag).

%% Read from fragmented table
read_event(Id) ->
    mnesia:activity(async_dirty, fun() ->
        mnesia:read(big_events, Id)
    end, [], mnesia_frag).

%% Iterate all fragments
scan_all_events() ->
    mnesia:activity(async_dirty, fun() ->
        mnesia:foldl(fun(Event, Acc) ->
            [Event | Acc]
        end, [], big_events)
    end, [], mnesia_frag).
```

---

## ตัวอย่างจริง: Distributed Session Store

```erlang
-module(session_store).
-export([create/2, get/1, refresh/2, delete/1, cleanup_expired/0]).

-define(SESSION_TTL, 3600).  %% 1 hour

%% Create session
create(UserId, Data) ->
    Token = generate_token(),
    ExpiresAt = erlang:system_time(second) + ?SESSION_TTL,
    
    Session = #session{
        token = Token,
        user_id = UserId,
        ip_address = maps:get(ip, Data, undefined),
        expires_at = ExpiresAt,
        data = Data
    },
    
    case mnesia:transaction(fun() -> mnesia:write(Session) end) of
        {atomic, ok} -> {ok, Token};
        {aborted, Reason} -> {error, Reason}
    end.

%% Get session (validates expiry)
get(Token) ->
    case mnesia:dirty_read(session, Token) of
        [#session{expires_at = ExpiresAt} = Session] ->
            case erlang:system_time(second) < ExpiresAt of
                true -> {ok, Session};
                false ->
                    delete(Token),
                    expired
            end;
        [] -> not_found
    end.

%% Refresh session TTL
refresh(Token, AdditionalSeconds) ->
    F = fun() ->
        case mnesia:wread({session, Token}) of
            [Session] ->
                NewExpiry = erlang:system_time(second) + AdditionalSeconds,
                mnesia:write(Session#session{expires_at = NewExpiry}),
                ok;
            [] -> not_found
        end
    end,
    mnesia:transaction(F).

%% Delete session (logout)
delete(Token) ->
    mnesia:transaction(fun() ->
        mnesia:delete({session, Token})
    end).

%% Cleanup expired sessions (run periodically)
cleanup_expired() ->
    Now = erlang:system_time(second),
    MatchSpec = [{
        #session{token = '$1', expires_at = '$2', _ = '_'},
        [{'=<', '$2', Now}],
        ['$1']
    }],
    
    ExpiredTokens = mnesia:dirty_select(session, MatchSpec),
    
    mnesia:transaction(fun() ->
        [mnesia:delete({session, T}) || T <- ExpiredTokens],
        length(ExpiredTokens)
    end).

generate_token() ->
    base64:encode(crypto:strong_rand_bytes(32)).

%% Statistics
session_stats() ->
    TotalSessions = mnesia:table_info(session, size),
    Now = erlang:system_time(second),
    
    ActiveSessions = length(mnesia:dirty_select(session, [
        {#session{expires_at = '$1', _ = '_'},
         [{'>', '$1', Now}],
         [true]}
    ])),
    
    #{
        total => TotalSessions,
        active => ActiveSessions,
        expired => TotalSessions - ActiveSessions
    }.
```

---

## สรุป Part 72

| Feature | Mnesia API |
|---------|-----------|
| Create table | `mnesia:create_table/2` |
| Transaction | `mnesia:transaction/1` |
| Dirty read | `mnesia:dirty_read/2` |
| Index read | `mnesia:index_read/3` |
| Select | `mnesia:dirty_select/2` |
| Subscribe | `mnesia:subscribe/1` |
| Fragmented | `mnesia_frag` activity |

**ข้อควรระวัง**: Mnesia ไม่เหมาะกับ table ที่มีข้อมูลมากกว่า 1-2GB ให้ใช้ PostgreSQL หรือ Cassandra แทน

---

*[← Part 71: Ecosystem](part_71_ecosystem.md) | [Part 73: Cloud Deployment →](part_73_cloud_deployment.md)*
