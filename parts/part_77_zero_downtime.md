# Part 77: Zero-Downtime Deployment

## สารบัญ
1. [Hot Code Upgrade (appup/relup)](#hot-upgrade)
2. [Rolling Deployments](#rolling)
3. [Blue-Green Deployment](#blue-green)
4. [Database Migrations without Downtime](#db-migrations)
5. [Feature Flags](#feature-flags)
6. [ตัวอย่างจริง: Production Hot Upgrade](#production-upgrade)

---

## Hot Code Upgrade (appup/relup)

```erlang
%% Erlang's killer feature: upgrade running system without restart
%% All state is preserved through the upgrade

%% 1. Write the new module version
%% 2. Create .appup file describing the upgrade path
%% 3. Build relup with rebar3
%% 4. Install upgrade on running node

%% my_app.appup — upgrade instructions
%% Located in ebin/ directory
{"2.0.0",
 [{"1.0.0", [                      %% upgrade from 1.0.0
    %% Load new modules
    {load_module, my_server},
    {load_module, my_utils},
    
    %% For gen_server with state migration:
    {update, my_stateful_server,
        {advanced, []},            %% triggers code_change callback
        supervisor},
    
    %% Run arbitrary upgrade code
    {apply, {my_upgrade, run, []}}
 ]}],
 [{"1.0.0", [                      %% downgrade to 1.0.0
    {load_module, my_server},
    {load_module, my_utils},
    {update, my_stateful_server, {advanced, []}, supervisor}
 ]}]
}.

%% Handle code change in gen_server
-module(my_stateful_server).
-behaviour(gen_server).

%% Old state shape (v1):
%% #{name => Name, count => Count}

%% New state shape (v2):
%% #{name => Name, count => Count, created_at => Timestamp}

code_change("1.0.0", OldState, _Extra) ->
    %% Migrate state from v1 to v2
    NewState = OldState#{created_at => erlang:system_time(second)},
    {ok, NewState};

code_change(_OldVsn, State, _Extra) ->
    {ok, State}.

%% Build and install upgrade
upgrade_commands() ->
    %% Build release
    "rebar3 release",
    
    %% Build upgrade package
    "rebar3 relup",
    
    %% Install on running node
    %% Copy _build/default/rel/my_app/releases/2.0.0/ to node
    %% Then in Erlang shell:
    %% release_handler:unpack_release("my_app-2.0.0")
    %% release_handler:install_release("2.0.0")
    %% release_handler:make_permanent("2.0.0")
    ok.

%% Programmatic upgrade
install_upgrade(Version) ->
    case release_handler:unpack_release("my_app-" ++ Version) of
        {ok, Vsn} ->
            case release_handler:install_release(Vsn) of
                {ok, _, _} ->
                    release_handler:make_permanent(Vsn),
                    logger:info("Upgrade complete", #{version => Version}),
                    ok;
                {error, Reason} ->
                    logger:error("Upgrade failed", #{reason => Reason}),
                    {error, Reason}
            end;
        {error, Reason} ->
            {error, Reason}
    end.

%% Rollback if needed
rollback_upgrade(PreviousVersion) ->
    release_handler:install_release(PreviousVersion),
    release_handler:make_permanent(PreviousVersion).
```

---

## Rolling Deployments

```erlang
%% rolling_deploy.erl — deploy one node at a time

-module(rolling_deploy).
-export([deploy/2, status/1]).

deploy(Nodes, ReleaseVersion) ->
    logger:info("Starting rolling deployment", #{
        nodes => length(Nodes),
        version => ReleaseVersion
    }),
    
    deploy_nodes(Nodes, ReleaseVersion, #{
        success => [],
        failed => [],
        total => length(Nodes)
    }).

deploy_nodes([], _, Results) ->
    logger:info("Rolling deployment complete", Results),
    Results;

deploy_nodes([Node | Rest], Version, Results) ->
    logger:info("Deploying to node", #{node => Node, remaining => length(Rest)});
    
    case deploy_to_node(Node, Version) of
        ok ->
            %% Health check before continuing
            case wait_for_healthy(Node, 60000) of
                ok ->
                    NewResults = Results#{success := [Node | maps:get(success, Results)]},
                    deploy_nodes(Rest, Version, NewResults);
                {error, timeout} ->
                    logger:error("Node unhealthy after deploy, stopping rollout", #{node => Node}),
                    Results#{failed := [Node]}  %% Stop deployment
            end;
        
        {error, Reason} ->
            logger:error("Deploy failed on node", #{node => Node, reason => Reason}),
            Results#{failed := [Node]}  %% Stop deployment
    end.

deploy_to_node(Node, Version) ->
    %% Remote shell to node
    case rpc:call(Node, rolling_deploy, install_upgrade, [Version], 30000) of
        ok -> ok;
        {badrpc, Reason} -> {error, Reason};
        Error -> Error
    end.

wait_for_healthy(Node, TimeoutMs) ->
    wait_for_healthy(Node, TimeoutMs, erlang:monotonic_time(millisecond)).

wait_for_healthy(_, TimeoutMs, StartMs) when erlang:monotonic_time(millisecond) - StartMs > TimeoutMs ->
    {error, timeout};
wait_for_healthy(Node, TimeoutMs, StartMs) ->
    case rpc:call(Node, health_check, status, [], 5000) of
        {ok, healthy} -> ok;
        _ ->
            timer:sleep(2000),
            wait_for_healthy(Node, TimeoutMs, StartMs)
    end.

%% Drain node before deployment
drain_node(Node) ->
    %% Remove from load balancer
    lb_controller:remove_node(Node),
    
    %% Wait for in-flight requests
    timer:sleep(5000),
    
    ok.

restore_node(Node) ->
    lb_controller:add_node(Node).
```

---

## Blue-Green Deployment

```erlang
%% Erlang distributed nodes make blue-green natural:
%% - Blue cluster: nodes app_1@host1, app_2@host2 (current production)
%% - Green cluster: nodes app_3@host3, app_4@host4 (new version)

-module(blue_green).
-export([switch_traffic/1, health_check_cluster/1, rollback/0]).

switch_traffic(green) ->
    %% Step 1: Health check green cluster
    case health_check_cluster(green) of
        ok ->
            %% Step 2: Atomic traffic switch at load balancer
            ok = lb_controller:set_active_cluster(green),
            logger:info("Traffic switched to green cluster"),
            
            %% Step 3: Keep blue warm for quick rollback (5 minutes)
            erlang:send_after(300000, self(), drain_blue),
            ok;
        {error, Reason} ->
            logger:error("Green cluster unhealthy, aborting switch", #{reason => Reason}),
            {error, Reason}
    end.

health_check_cluster(ClusterName) ->
    Nodes = get_cluster_nodes(ClusterName),
    
    Results = [{N, rpc:call(N, health_check, full, [], 10000)} || N <- Nodes],
    
    Unhealthy = [{N, R} || {N, R} <- Results, R =/= {ok, healthy}],
    
    case Unhealthy of
        [] -> ok;
        _ -> {error, {unhealthy_nodes, Unhealthy}}
    end.

rollback() ->
    %% Instant rollback to blue
    ok = lb_controller:set_active_cluster(blue),
    logger:info("Rolled back to blue cluster"),
    ok.

get_cluster_nodes(blue) ->
    application:get_env(my_app, blue_nodes, []);
get_cluster_nodes(green) ->
    application:get_env(my_app, green_nodes, []).
```

---

## Database Migrations without Downtime

```erlang
%% Zero-downtime DB migrations strategy:
%% Phase 1: Add new column (nullable) — deploy with old code still works
%% Phase 2: Backfill data — run migration script
%% Phase 3: Deploy new code — reads new column
%% Phase 4: Add NOT NULL constraint — old code no longer deployed

%% migration_runner.erl

-module(migration_runner).
-export([run/1, status/0]).

-define(BATCH_SIZE, 1000).

run(MigrationId) ->
    case migration_log:get_status(MigrationId) of
        not_run ->
            execute_migration(MigrationId);
        completed ->
            {ok, already_done};
        {failed, Reason} ->
            logger:error("Migration previously failed", #{id => MigrationId, reason => Reason}),
            {error, previous_failure}
    end.

%% Backfill with batching to avoid locking
backfill_column(Table, Column, DefaultFn, BatchSize) ->
    backfill_batch(Table, Column, DefaultFn, BatchSize, 0, 0).

backfill_batch(Table, Column, DefaultFn, BatchSize, Offset, Updated) ->
    %% Get batch of rows missing the column
    Query = io_lib:format(
        "SELECT id FROM ~s WHERE ~s IS NULL ORDER BY id LIMIT ~p OFFSET ~p",
        [Table, Column, BatchSize, Offset]
    ),
    
    case db:query(Query, []) of
        {ok, [], _} ->
            logger:info("Backfill complete", #{table => Table, column => Column, updated => Updated}),
            {ok, Updated};
        
        {ok, Rows, _} ->
            Ids = [Id || {Id} <- Rows],
            
            %% Update in single transaction
            Values = [{Id, DefaultFn(Id)} || Id <- Ids],
            batch_update(Table, Column, Values),
            
            %% Sleep to reduce DB load
            timer:sleep(100),
            
            backfill_batch(Table, Column, DefaultFn, BatchSize, Offset + length(Rows), Updated + length(Rows))
    end.

batch_update(Table, Column, Values) ->
    %% UPDATE table SET column = v WHERE id = id
    [db:execute(
        io_lib:format("UPDATE ~s SET ~s = $1 WHERE id = $2", [Table, Column]),
        [Value, Id]
     ) || {Id, Value} <- Values].

%% Safe add column (non-blocking in PostgreSQL)
add_column_safe(Table, Column, Type) ->
    %% PostgreSQL: adding nullable column doesn't lock the table
    db:execute(
        io_lib:format("ALTER TABLE ~s ADD COLUMN IF NOT EXISTS ~s ~s", [Table, Column, Type]),
        []
    ).

%% Add index concurrently (PostgreSQL)
add_index_concurrent(Table, Columns) ->
    IndexName = "idx_" ++ Table ++ "_" ++ string:join(Columns, "_"),
    Query = io_lib:format(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS ~s ON ~s (~s)",
        [IndexName, Table, string:join(Columns, ", ")]
    ),
    db:execute(Query, []).
```

---

## Feature Flags

```erlang
%% feature_flags.erl — gradual rollout control

-module(feature_flags).
-export([is_enabled/2, enable/2, disable/1, set_rollout_percentage/2]).

%% Flag types:
%% - boolean: on/off for everyone
%% - percentage: on for X% of users
%% - user_list: on for specific user IDs
%% - segment: on for users with matching attribute

is_enabled(FlagName, UserId) ->
    case ets:lookup(feature_flags, FlagName) of
        [{FlagName, #{type := boolean, enabled := true}}] ->
            true;
        [{FlagName, #{type := percentage, percentage := P}}] ->
            %% Deterministic: same user always gets same result
            erlang:phash2({FlagName, UserId}, 100) < P;
        [{FlagName, #{type := user_list, users := Users}}] ->
            lists:member(UserId, Users);
        [{FlagName, #{type := segment, segment := Seg}}] ->
            user_segments:has_segment(UserId, Seg);
        _ ->
            false  %% Unknown flag → disabled by default
    end.

enable(FlagName, Config) ->
    gen_server:call(?MODULE, {set_flag, FlagName, Config#{enabled => true}}).

disable(FlagName) ->
    gen_server:call(?MODULE, {set_flag, FlagName, #{enabled => false}}).

set_rollout_percentage(FlagName, Percentage) when Percentage >= 0, Percentage =< 100 ->
    enable(FlagName, #{type => percentage, percentage => Percentage}).

%% Usage in code
handle_request(Req, State) ->
    UserId = get_user_id(Req),
    
    Response = case feature_flags:is_enabled(new_checkout_flow, UserId) of
        true ->
            new_checkout_handler:handle(Req);
        false ->
            old_checkout_handler:handle(Req)
    end,
    
    {ok, Response, State}.

%% Admin API for flag management
-module(flags_admin_handler).
-behaviour(cowboy_handler).

init(Req, State) ->
    #{flag := FlagName} = cowboy_req:bindings(Req),
    
    case cowboy_req:method(Req) of
        <<"PUT">> ->
            {ok, Body, Req2} = cowboy_req:read_body(Req),
            Config = jsx:decode(Body, [return_maps]),
            feature_flags:enable(FlagName, Config),
            {ok, cowboy_req:reply(200, Req2), State};
        
        <<"DELETE">> ->
            feature_flags:disable(FlagName),
            {ok, cowboy_req:reply(204, Req), State}
    end.
```

---

## ตัวอย่างจริง: Production Hot Upgrade

```erlang
%% Complete workflow for hot code upgrade in production

-module(prod_upgrade).
-export([upgrade/2, verify/1, rollback/1]).

upgrade(NewVersion, OldVersion) ->
    Steps = [
        {pre_upgrade_checks, fun() -> pre_upgrade_checks(NewVersion) end},
        {backup_state, fun() -> backup_state() end},
        {set_maintenance_flag, fun() -> feature_flags:enable(maintenance_mode, #{type => boolean}) end},
        {drain_load, fun() -> timer:sleep(5000) end},
        {install_release, fun() -> install_upgrade(NewVersion) end},
        {verify_health, fun() -> verify_health(NewVersion) end},
        {clear_maintenance, fun() -> feature_flags:disable(maintenance_mode) end},
        {post_upgrade_smoke_test, fun() -> smoke_test(NewVersion) end}
    ],
    
    execute_steps(Steps, OldVersion).

execute_steps([], _) ->
    {ok, complete};
execute_steps([{StepName, Fun} | Rest], Rollback) ->
    logger:info("Executing upgrade step", #{step => StepName}),
    
    case safe_execute(Fun) of
        ok ->
            execute_steps(Rest, Rollback);
        {error, Reason} ->
            logger:error("Upgrade step failed, initiating rollback", #{
                step => StepName,
                reason => Reason
            }),
            rollback(Rollback),
            {error, {step_failed, StepName, Reason}}
    end.

safe_execute(Fun) ->
    try
        case Fun() of
            ok -> ok;
            {ok, _} -> ok;
            Error -> Error
        end
    catch
        Class:Reason:Stack ->
            logger:error("Step threw exception", #{class => Class, reason => Reason, stack => Stack}),
            {error, {exception, Reason}}
    end.

pre_upgrade_checks(Version) ->
    Checks = [
        {disk_space, fun() -> check_disk_space() end},
        {release_exists, fun() -> check_release_exists(Version) end},
        {no_pending_migrations, fun() -> check_no_pending_migrations() end}
    ],
    
    Failed = [{Name, Err} || {Name, Fun} <- Checks,
              (fun() -> case Fun() of ok -> false; Err -> {true, Err} end end)() =/= false],
    
    case Failed of
        [] -> ok;
        _ -> {error, {pre_checks_failed, Failed}}
    end.

check_disk_space() ->
    %% Need at least 500MB free
    case disk_free_bytes() > 500 * 1024 * 1024 of
        true -> ok;
        false -> {error, insufficient_disk_space}
    end.

verify_health(Version) ->
    %% Wait for process supervisor to stabilize
    timer:sleep(2000),
    
    %% Check all supervisors healthy
    case supervisor:count_children(my_sup) of
        #{active := Active, specs := Specs} when Active =:= Specs ->
            ok;
        Counts ->
            {error, {unhealthy_supervisors, Counts}}
    end.

smoke_test(_Version) ->
    %% Run critical path test
    case http_test:get("http://localhost:8080/health") of
        {ok, 200, _} -> ok;
        Other -> {error, {smoke_test_failed, Other}}
    end.

backup_state() ->
    %% Snapshot Mnesia tables
    mnesia:backup("/var/backup/mnesia_" ++
                  calendar:format_time(calendar:local_time()) ++ ".bak").

disk_free_bytes() ->
    %% Get available disk space
    case file:read_file_info("/") of
        {ok, #file_info{size = _}} -> 1000000000;  %% simplified
        _ -> 0
    end.

check_release_exists(Version) ->
    RelDir = "/app/releases/" ++ Version,
    case filelib:is_dir(RelDir) of
        true -> ok;
        false -> {error, release_not_found}
    end.

check_no_pending_migrations() ->
    case migration_runner:pending_count() of
        0 -> ok;
        N -> {error, {pending_migrations, N}}
    end.
```

---

## สรุป Part 77

| Strategy | When to Use |
|----------|------------|
| Hot code upgrade | Minor changes, state migration |
| Rolling deployment | Kubernetes/multi-node, gradual |
| Blue-green | Major changes, instant rollback |
| Feature flags | Gradual user rollout |
| DB migration pattern | Always (4-phase) |

**กฎทอง**: ทดสอบ upgrade process ใน staging ก่อนทุกครั้ง ไม่มี shortcut

---

*[← Part 76: RTB](part_76_rtb.md) | [Part 78: ML Serving →](part_78_ml_serving.md)*
