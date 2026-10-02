# Part 64: Database Patterns — Sharding และ Multi-Region

## สารบัญ
1. [Database Sharding](#sharding)
2. [Read Replicas](#read-replicas)
3. [Multi-Region Architecture](#multi-region)
4. [Connection Pooling Strategies](#connection-pooling)
5. [Database Migration](#migration)
6. [ตัวอย่างจริง: Multi-Tenant SaaS Database](#multi-tenant)

---

## Database Sharding

```erlang
%% Sharding: split data across multiple databases
%% Key strategies: hash-based, range-based, directory-based

-module(shard_router).
-export([get_shard/1, route_query/2, all_shards/0]).

-define(NUM_SHARDS, 16).

%% Hash-based sharding (consistent hashing)
get_shard(Key) ->
    Hash = erlang:phash2(Key, ?NUM_SHARDS),
    shard_name(Hash).

shard_name(N) ->
    list_to_atom("db_shard_" ++ integer_to_list(N)).

%% Route read to appropriate shard
route_query(Key, QueryFun) ->
    Shard = get_shard(Key),
    db_pool:with_connection(Shard, QueryFun).

%% Scatter-gather: query all shards
scatter_gather(QueryFun) ->
    Parent = self(),
    Shards = all_shards(),
    Pids = [begin
        Shard = shard_name(I),
        spawn(fun() ->
            Result = db_pool:with_connection(Shard, QueryFun),
            Parent ! {I, Result}
        end)
    end || I <- lists:seq(0, ?NUM_SHARDS - 1)],
    collect_shard_results(?NUM_SHARDS, []).

collect_shard_results(0, Acc) -> lists:sort(Acc);
collect_shard_results(N, Acc) ->
    receive
        {ShardId, {ok, Rows}} ->
            collect_shard_results(N - 1, Rows ++ Acc);
        {ShardId, {error, Reason}} ->
            logger:error("Shard query failed", #{shard => ShardId, reason => Reason}),
            collect_shard_results(N - 1, Acc)
    after 10000 ->
        logger:warning("Shard query timeout, ~p remaining", [N]),
        Acc
    end.

all_shards() ->
    [shard_name(I) || I <- lists:seq(0, ?NUM_SHARDS - 1)].

%% Consistent hashing ring (for shard rebalancing)
-module(consistent_hash).
-export([new/0, add_node/2, remove_node/2, get_node/2]).

new() ->
    ets:new(hash_ring, [ordered_set, public, named_table]).

%% Add node with V virtual nodes for even distribution
add_node(Node, VNodes) ->
    [begin
        Hash = erlang:phash2({Node, I}, 1 bsl 32),
        ets:insert(hash_ring, {Hash, Node})
    end || I <- lists:seq(1, VNodes)].

remove_node(Node, VNodes) ->
    [begin
        Hash = erlang:phash2({Node, I}, 1 bsl 32),
        ets:delete(hash_ring, Hash)
    end || I <- lists:seq(1, VNodes)].

get_node(Key, _Ring) ->
    Hash = erlang:phash2(Key, 1 bsl 32),
    case ets:next(hash_ring, Hash) of
        '$end_of_table' ->
            %% Wrap around to first
            [{_, Node}] = ets:first(hash_ring),
            Node;
        NextHash ->
            [{_, Node}] = ets:lookup(hash_ring, NextHash),
            Node
    end.
```

---

## Read Replicas

```erlang
%% Read/write splitting: writes → primary, reads → replicas
-module(db_router).
-export([query/2, execute/2]).

%% Read: load balance across replicas
query(SQL, Params) ->
    Replica = pick_replica(),
    db_pool:query(Replica, SQL, Params).

%% Write: always to primary
execute(SQL, Params) ->
    db_pool:execute(primary, SQL, Params).

pick_replica() ->
    Replicas = application:get_env(myapp, db_replicas, []),
    case Replicas of
        [] -> primary;  %% fallback if no replicas
        _ ->
            %% Round-robin with health check
            N = ets:update_counter(db_state, replica_index, {2, 1, length(Replicas), 0}),
            Replica = lists:nth(N + 1, Replicas),
            case is_replica_healthy(Replica) of
                true -> Replica;
                false -> pick_replica_excluding(Replica, Replicas)
            end
    end.

is_replica_healthy(Replica) ->
    case db_pool:ping(Replica) of
        ok -> true;
        _ -> false
    end.

pick_replica_excluding(Excluded, Replicas) ->
    Healthy = [R || R <- Replicas, R /= Excluded, is_replica_healthy(R)],
    case Healthy of
        [] -> primary;  %% all replicas down, use primary
        _ -> lists:nth(rand:uniform(length(Healthy)), Healthy)
    end.

%% Replication lag monitoring
-module(replication_monitor).
-export([start_link/0]).

init(_) ->
    timer:send_interval(5000, check_replication),
    {ok, #{}}.

handle_info(check_replication, State) ->
    Replicas = application:get_env(myapp, db_replicas, []),
    [check_replica_lag(R) || R <- Replicas],
    {noreply, State}.

check_replica_lag(Replica) ->
    case db_pool:query(Replica, "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int", []) of
        {ok, [{LagSeconds}]} ->
            prometheus_gauge:set(replication_lag_seconds, [atom_to_list(Replica)], LagSeconds),
            if LagSeconds > 10 ->
                logger:warning("High replication lag", #{replica => Replica, lag_seconds => LagSeconds});
            true -> ok
            end;
        _ -> ok
    end.
```

---

## Multi-Region Architecture

```erlang
%% Multi-region: data closer to users, disaster recovery

-module(region_router).
-export([get_primary_region/0, route_to_region/2, failover/1]).

-define(REGIONS, [
    #{name => <<"us-east-1">>, lat => 37.9, lng => -77.5},
    #{name => <<"eu-west-1">>, lat => 53.3, lng => -6.3},
    #{name => <<"ap-southeast-1">>, lat => 1.3, lng => 103.8}
]).

%% Route user to nearest region
route_by_location(UserLat, UserLng) ->
    {_, NearestRegion} = lists:min([
        {haversine(UserLat, UserLng, maps:get(lat, R), maps:get(lng, R)), R}
        || R <- ?REGIONS
    ]),
    maps:get(name, NearestRegion).

haversine(Lat1, Lng1, Lat2, Lng2) ->
    R = 6371,  %% Earth radius km
    DLat = (Lat2 - Lat1) * math:pi() / 180,
    DLng = (Lng2 - Lng1) * math:pi() / 180,
    A = math:sin(DLat/2) * math:sin(DLat/2) +
        math:cos(Lat1 * math:pi() / 180) * math:cos(Lat2 * math:pi() / 180) *
        math:sin(DLng/2) * math:sin(DLng/2),
    C = 2 * math:atan2(math:sqrt(A), math:sqrt(1 - A)),
    R * C.

%% Cross-region data sync
-module(region_sync).
-export([sync_record/2]).

sync_record(Table, Record) ->
    %% Write to local first (fast)
    db:write(Table, Record),
    
    %% Async sync to other regions
    Regions = other_regions(),
    [begin
        spawn(fun() ->
            sync_to_region(Region, Table, Record)
        end)
    end || Region <- Regions].

sync_to_region(Region, Table, Record) ->
    RegionUrl = region_api_url(Region),
    Body = jsx:encode(#{table => Table, record => Record}),
    case httpc:request(post, {RegionUrl ++ "/sync", [], "application/json", Body},
                       [{timeout, 5000}], []) of
        {ok, {{_, 200, _}, _, _}} -> ok;
        Error ->
            logger:error("Region sync failed", #{region => Region, error => Error}),
            retry_queue:enqueue({region_sync, Region, Table, Record})
    end.

%% Failover
failover(FailedRegion) ->
    logger:critical("Region failed, failing over", #{region => FailedRegion}),
    %% Update DNS to route away from failed region
    dns:update_weighted_routing(FailedRegion, 0),
    %% Promote replica in secondary region to primary
    BackupRegion = get_backup_region(FailedRegion),
    db_admin:promote_replica(BackupRegion),
    %% Alert team
    pagerduty:trigger(high, "Region failover", #{failed => FailedRegion, backup => BackupRegion}).

other_regions() ->
    All = application:get_env(myapp, regions, []),
    Current = get_primary_region(),
    [R || R <- All, R /= Current].
```

---

## ตัวอย่างจริง: Multi-Tenant SaaS

```erlang
%% Multi-tenant: each tenant gets isolated data

-module(tenant_db).
-export([query/3, execute/3, create_tenant/1]).

%% Approach 1: Schema-per-tenant
%% Pros: good isolation, easy migration per tenant
%% Cons: more schemas to manage

query(TenantId, SQL, Params) ->
    Schema = tenant_schema(TenantId),
    FullSQL = <<"SET search_path TO ", Schema/binary, "; ", SQL/binary>>,
    db_pool:query(pick_shard(TenantId), FullSQL, Params).

tenant_schema(TenantId) ->
    <<"tenant_", TenantId/binary>>.

%% Approach 2: Row-level security (Postgres RLS)
%% tenant_id column in every table

query_rls(TenantId, SQL, Params) ->
    Connection = db_pool:checkout(pick_shard(TenantId)),
    try
        %% Set tenant context for RLS
        db_pool:execute(Connection, "SET app.tenant_id = $1", [TenantId]),
        db_pool:query(Connection, SQL, Params)
    after
        db_pool:checkin(Connection)
    end.

%% Tenant provisioning
create_tenant(TenantId) ->
    Schema = tenant_schema(TenantId),
    Shard = pick_shard(TenantId),
    
    %% Create schema
    db_pool:execute(Shard, <<"CREATE SCHEMA ", Schema/binary>>, []),
    
    %% Run migrations for new tenant
    run_migrations(Shard, Schema),
    
    %% Record in tenant registry
    db_pool:execute(primary, "INSERT INTO tenants(id, schema, shard, created_at) VALUES($1, $2, $3, NOW())",
                    [TenantId, Schema, atom_to_binary(Shard)]),
    
    {ok, TenantId}.

run_migrations(Shard, Schema) ->
    MigrationDir = code:priv_dir(myapp) ++ "/migrations",
    {ok, Files} = file:list_dir(MigrationDir),
    SortedFiles = lists:sort(Files),
    [run_migration(Shard, Schema, F) || F <- SortedFiles].

run_migration(Shard, Schema, File) ->
    {ok, SQL} = file:read_file(filename:join([code:priv_dir(myapp), "migrations", File])),
    %% Replace {schema} placeholder with actual schema name
    ActualSQL = binary:replace(SQL, <<"{schema}">>, Schema, [global]),
    db_pool:execute(Shard, ActualSQL, []).

pick_shard(TenantId) ->
    Hash = erlang:phash2(TenantId, 16),
    list_to_atom("db_shard_" ++ integer_to_list(Hash)).
```

---

## สรุป Part 64

| Pattern | Use Case | Trade-off |
|---------|----------|-----------|
| Hash sharding | Even distribution | Cross-shard queries hard |
| Range sharding | Range queries | Hot spots possible |
| Read replicas | Read-heavy workload | Eventual consistency |
| Multi-region | Low latency globally | Data sync complexity |
| Schema-per-tenant | SaaS isolation | Schema management |
| Row-level security | Simple multi-tenant | Less isolation |

---

*[← Part 63: Monitoring](part_63_monitoring.md) | [Part 65: Full Production System →](part_65_production_system.md)*
