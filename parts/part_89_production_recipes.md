# Part 89: Production Recipes and Playbooks

## สารบัญ
1. [Release Management](#release)
2. [Incident Response Playbook](#incident)
3. [Database Migration Patterns](#migration)
4. [Graceful Degradation](#degradation)
5. [Capacity Planning](#capacity)
6. [ตัวอย่างจริง: Production Runbook](#runbook)

---

## Release Management

```erlang
%% relx configuration for production release

%% rebar.config relx section:
relx_config() ->
    [{relx, [
        {release, {my_app, "1.0.0"}, [
            my_app,
            sasl    %% system application support library (release tools)
        ]},
        
        %% Include ERTS (portable release)
        {include_erts, true},
        {extended_start_script, true},
        
        %% Production mode (no development tools)
        {dev_mode, false},
        
        %% vm.args for production
        {vm_args, "config/vm.args"},
        
        %% sys.config
        {sys_config, "config/sys.config"},
        
        {overlay, [
            %% Copy scripts
            {copy, "scripts/start.sh", "bin/start.sh"},
            {copy, "config/", "releases/{{release_version}}/config/"}
        ]}
    ]}].

%% Generate release
generate_release_commands() ->
    "rebar3 release",
    "rebar3 tar",           %% creates _build/prod/rel/my_app/my_app-1.0.0.tar.gz
    "_build/prod/rel/my_app/bin/my_app start",
    "_build/prod/rel/my_app/bin/my_app daemon",    %% background
    "_build/prod/rel/my_app/bin/my_app console",   %% attached console
    "_build/prod/rel/my_app/bin/my_app remote_console",  %% attach to running
    "_build/prod/rel/my_app/bin/my_app stop".

%% Hot upgrade procedure
hot_upgrade_procedure() ->
    Steps = [
        "1. Build new version: rebar3 as prod release",
        "2. Create appup files for changed applications",
        "3. Generate relup: rebar3 relup",
        "4. Copy new release tarball to /releases/2.0.0/",
        "5. Unpack: release_handler:unpack_release('my_app-2.0.0')",
        "6. Install: release_handler:install_release('2.0.0')",
        "7. Verify system is stable",
        "8. Make permanent: release_handler:make_permanent('2.0.0')",
        "or Roll back: release_handler:install_release('1.0.0')"
    ],
    [io:format("~s~n", [S]) || S <- Steps].

%% Programmatic hot upgrade
do_hot_upgrade(NewVersion) ->
    {ok, OtherVersions, NewVersion} = release_handler:check_install_release(NewVersion),
    
    logger:info("Starting hot upgrade", #{to => NewVersion}),
    
    case release_handler:install_release(NewVersion) of
        {ok, _, _} ->
            logger:info("Upgrade successful", #{version => NewVersion}),
            release_handler:make_permanent(NewVersion),
            {ok, upgraded};
        {error, Reason} ->
            logger:error("Upgrade failed, rolling back", #{reason => Reason}),
            [release_handler:install_release(V) || V <- OtherVersions],
            {error, Reason}
    end.
```

---

## Incident Response Playbook

```erlang
%% incident_tools.erl — diagnostic tools for production incidents

-module(incident_tools).
-export([snapshot/0, diagnose_high_cpu/0, diagnose_memory_leak/0,
         find_stuck_processes/0, dump_message_queues/0]).

%% Full system snapshot for incident report
snapshot() ->
    #{
        timestamp => erlang:system_time(second),
        node => node(),
        uptime_seconds => element(1, statistics(wall_clock)) div 1000,
        
        %% Process stats
        process_count => erlang:system_info(process_count),
        process_limit => erlang:system_info(process_limit),
        
        %% Memory
        memory => erlang:memory(),
        
        %% Schedulers
        scheduler_count => erlang:system_info(schedulers_online),
        run_queue => statistics(run_queue),
        
        %% I/O
        io => statistics(io),
        
        %% GC
        gc_stats => statistics(garbage_collection),
        
        %% Top processes by memory
        top_by_memory => top_processes(memory, 10),
        
        %% Top processes by message queue
        top_by_queue => top_processes(message_queue_len, 10)
    }.

top_processes(Metric, N) ->
    Procs = erlang:processes(),
    
    WithMetric = lists:filtermap(fun(Pid) ->
        try
            Info = process_info(Pid, [Metric, registered_name, current_function]),
            Value = proplists:get_value(Metric, Info),
            {true, {Value, Pid, Info}}
        catch _:_ -> false
        end
    end, Procs),
    
    Sorted = lists:reverse(lists:sort(WithMetric)),
    lists:sublist(Sorted, N).

%% Diagnose high CPU
diagnose_high_cpu() ->
    erlang:system_flag(scheduler_wall_time, true),
    timer:sleep(1000),
    Stats = erlang:statistics(scheduler_wall_time),
    
    ByUtil = lists:sort(fun({_, A1, I1}, {_, A2, I2}) ->
        A1/(A1+I1) > A2/(A2+I2)
    end, Stats),
    
    [begin
        Util = A / (A + I),
        logger:info("Scheduler ~p: ~.1f% utilization", [S, Util * 100])
    end || {S, A, I} <- ByUtil],
    
    %% Find CPU-intensive processes
    RunQueue = statistics(run_queue),
    logger:info("Run queue length: ~p", [RunQueue]),
    
    ok.

%% Diagnose memory leak
diagnose_memory_leak() ->
    Memory = erlang:memory(),
    logger:info("Memory snapshot", Memory),
    
    %% Top ETS tables by size
    Tables = ets:all(),
    TableSizes = [{T, ets:info(T, memory)} || T <- Tables],
    Top = lists:sublist(lists:reverse(lists:keysort(2, TableSizes)), 10),
    
    logger:info("Top ETS tables by memory", #{tables => Top}),
    
    %% Check for binary references
    BinMemory = erlang:memory(binary),
    logger:info("Binary memory", #{bytes => BinMemory}),
    
    ok.

%% Find processes with large message queues (stuck processes)
find_stuck_processes() ->
    Threshold = 100,
    Stuck = lists:filtermap(fun(Pid) ->
        case process_info(Pid, [message_queue_len, registered_name, current_function,
                                  dictionary, status]) of
            undefined -> false;
            Info ->
                QLen = proplists:get_value(message_queue_len, Info),
                case QLen > Threshold of
                    true -> {true, {Pid, QLen, Info}};
                    false -> false
                end
        end
    end, erlang:processes()),
    
    case Stuck of
        [] ->
            logger:info("No stuck processes found");
        _ ->
            [logger:warning("Stuck process found", #{pid => Pid, queue_len => QLen})
             || {Pid, QLen, _} <- Stuck]
    end,
    Stuck.

dump_message_queues() ->
    Procs = erlang:processes(),
    [begin
        case process_info(P, [messages, registered_name]) of
            [{messages, [_|_] = Msgs} | _] = Info when length(Msgs) > 10 ->
                logger:info("Process with messages", #{pid => P, count => length(Msgs), info => Info});
            _ -> ok
        end
    end || P <- Procs].
```

---

## Database Migration Patterns

```erlang
%% db_migration.erl — zero-downtime migrations

-module(db_migration).
-export([run/0, status/0, rollback/1]).

%% Migration registry
migrations() ->
    [
        {1, "Create users table", fun migrate_001/0, fun rollback_001/0},
        {2, "Add email index", fun migrate_002/0, fun rollback_002/0},
        {3, "Add user_settings column", fun migrate_003/0, fun rollback_003/0}
    ].

run() ->
    {ok, Current} = get_current_version(),
    Pending = [{V, D, Up, Down} || {V, D, Up, Down} <- migrations(), V > Current],
    
    case Pending of
        [] ->
            logger:info("Database is up to date", #{version => Current});
        _ ->
            logger:info("Running ~p migrations", [length(Pending)]),
            [run_migration(V, D, Up) || {V, D, Up, _} <- Pending]
    end.

run_migration(Version, Description, UpFun) ->
    logger:info("Running migration", #{version => Version, description => Description}),
    
    T1 = erlang:monotonic_time(millisecond),
    ok = UpFun(),
    Duration = erlang:monotonic_time(millisecond) - T1,
    
    db:execute("INSERT INTO schema_migrations (version, description, applied_at) VALUES ($1, $2, NOW())",
               [Version, Description]),
    
    logger:info("Migration complete", #{version => Version, duration_ms => Duration}).

rollback(Version) ->
    case lists:keyfind(Version, 1, migrations()) of
        {Version, _, _, Down} ->
            logger:info("Rolling back migration", #{version => Version}),
            ok = Down(),
            db:execute("DELETE FROM schema_migrations WHERE version = $1", [Version]),
            ok;
        false ->
            {error, migration_not_found}
    end.

get_current_version() ->
    case db:execute("SELECT MAX(version) FROM schema_migrations", []) of
        {ok, [{null}]} -> {ok, 0};
        {ok, [{V}]} -> {ok, V};
        {error, _} ->
            %% Table doesn't exist, create it
            db:execute("CREATE TABLE IF NOT EXISTS schema_migrations "
                       "(version INTEGER PRIMARY KEY, description TEXT, applied_at TIMESTAMPTZ)", []),
            {ok, 0}
    end.

status() ->
    {ok, Current} = get_current_version(),
    All = migrations(),
    Applied = [V || {V, _, _, _} <- All, V =< Current],
    Pending = [V || {V, _, _, _} <- All, V > Current],
    #{current_version => Current, applied => Applied, pending => Pending}.

%% Example migrations
migrate_001() ->
    db:execute("CREATE TABLE users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(100) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMPTZ DEFAULT NOW()
    )", []),
    ok.

rollback_001() ->
    db:execute("DROP TABLE IF EXISTS users", []),
    ok.

migrate_002() ->
    %% Non-blocking index creation (CONCURRENTLY)
    db:execute("CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email)", []),
    ok.

rollback_002() ->
    db:execute("DROP INDEX CONCURRENTLY IF EXISTS idx_users_email", []),
    ok.

migrate_003() ->
    %% Add nullable column first (backward compatible)
    db:execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS settings JSONB", []),
    ok.

rollback_003() ->
    db:execute("ALTER TABLE users DROP COLUMN IF EXISTS settings", []),
    ok.
```

---

## Graceful Degradation

```erlang
%% graceful_degradation.erl — degrade gracefully under load

-module(graceful_degradation).
-export([with_fallback/3, shed_load/1, enable_circuit_breaker/1]).

%% Execute with fallback on failure
with_fallback(PrimaryFun, FallbackFun, Timeout) ->
    case timed_execute(PrimaryFun, Timeout) of
        {ok, Result} -> {ok, Result};
        {error, _Reason} ->
            metrics:counter(degraded_requests, [fallback], 1),
            case FallbackFun() of
                {ok, _} = R -> R;
                _ -> {error, both_failed}
            end
    end.

timed_execute(Fun, Timeout) ->
    Ref = make_ref(),
    Parent = self(),
    spawn_link(fun() -> Parent ! {Ref, try {ok, Fun()} catch _:R -> {error, R} end} end),
    receive
        {Ref, Result} -> Result
    after Timeout ->
        {error, timeout}
    end.

%% Load shedding: reject requests above threshold
-module(load_shedder).
-behaviour(gen_server).
-export([check/0, start_link/1]).

-record(state, {
    max_queue_depth,
    current_depth = 0
}).

check() ->
    gen_server:call(?MODULE, check).

handle_call(check, _From, #state{current_depth = D, max_queue_depth = Max} = State) ->
    case D < Max of
        true ->
            {reply, ok, State#state{current_depth = D + 1}};
        false ->
            metrics:counter(load_shed_total, [], 1),
            {reply, {error, overloaded}, State}
    end.

%% Feature flags for degradation
-module(feature_flags).
-export([is_enabled/1, disable/1, enable/1]).

-define(FLAGS_TABLE, feature_flags).

is_enabled(Feature) ->
    case ets:lookup(?FLAGS_TABLE, Feature) of
        [{Feature, false}] -> false;
        _ -> true  %% enabled by default
    end.

disable(Feature) ->
    ets:insert(?FLAGS_TABLE, {Feature, false}),
    logger:warning("Feature disabled", #{feature => Feature}).

enable(Feature) ->
    ets:delete(?FLAGS_TABLE, Feature),
    logger:info("Feature enabled", #{feature => Feature}).
```

---

## ตัวอย่างจริง: Production Runbook

```
## Runbook: High Memory Alert

### Alert condition
memory_bytes{node="prod-1"} > 3.5GB

### Investigation steps

1. Connect to node:
   _build/prod/rel/my_app/bin/my_app remote_console

2. Get memory snapshot:
   incident_tools:snapshot().

3. Check top processes:
   incident_tools:top_processes(memory, 20).

4. Check ETS tables:
   [{T, ets:info(T, memory)} || T <- ets:all()] |>
   lists:sort(fun({_,A},{_,B}) -> A > B end) |>
   lists:sublist(10).

5. Force full GC:
   [erlang:garbage_collect(P) || P <- erlang:processes()].

6. If leak in specific process, kill and let supervisor restart:
   exit(Pid, kill).

### Common causes
- User cache not expiring → check TTL config in user_cache
- Binary leaks (subbinary refs) → check binary memory, look for large message queues
- Mnesia table growing → check table size: mnesia:table_info(T, size)

## Runbook: High CPU Alert

### Alert condition
scheduler_utilization > 90% for 5 minutes

### Investigation steps
1. incident_tools:diagnose_high_cpu().
2. Look at run queue: statistics(run_queue).
3. Find hot processes: incident_tools:top_processes(reductions, 10).
4. Profile: eprof:start_profiling([Pid]), timer:sleep(5000), eprof:stop_profiling(), eprof:analyse().

### Common causes
- Tight loops in gen_server — check handle_info for timer loops
- Busy wait pattern — look for receive after 0 without sleep
- Crypto operations without dirty scheduler

## Runbook: OOM Kill

### Prevention
1. Set Erlang max heap: +hmax in vm.args
2. Enable memory alarms: memsup (from os_mon app)
3. Monitor with Prometheus + alert at 80%

### After OOM kill
1. Check crash dumps: ls /var/log/erl_crash.dump
2. Analyze: crashdump_viewer:start()
3. Find cause in crash dump: binaries, processes, ETS tables
```

---

## สรุป Part 89

| Area | Tool/Technique |
|------|---------------|
| Releases | relx + appup + relup |
| Incident diagnosis | incident_tools module |
| DB migrations | Versioned migrations with rollback |
| Graceful degradation | Fallbacks, load shedding, feature flags |
| Runbooks | Documented procedures per alert |

**Production mantra**: Plan for failure, measure everything, automate recovery, document procedures.

---

*[← Part 88: Observability](part_88_observability.md) | [Part 90: NIFs and Native Code →](part_90_nifs.md)*
