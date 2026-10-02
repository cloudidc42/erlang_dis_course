# Part 92: Advanced OTP Patterns

## สารบัญ
1. [Supervisor Strategies](#supervisor)
2. [proc_lib and Special Processes](#proc-lib)
3. [sys Module](#sys-module)
4. [Application Callbacks](#application)
5. [Dynamic Supervision](#dynamic)
6. [ตัวอย่างจริง: Self-Healing System](#self-healing)

---

## Supervisor Strategies

```erlang
%% supervisor.erl — all supervisor strategies

%% Strategy: one_for_one (default)
%% - Only crashed child is restarted
%% - Use when children are independent

%% Strategy: one_for_all
%% - All children are restarted when any crashes
%% - Use when children depend on each other

%% Strategy: rest_for_one
%% - Crashed child + all children started after it are restarted
%% - Use for ordered initialization chains

%% Strategy: simple_one_for_one
%% - Dynamic children, all of same type
%% - Use for worker pools, session managers

-module(complex_supervisor).
-behaviour(supervisor).
-export([start_link/0, init/1]).

init([]) ->
    %% Restart intensity: max 5 restarts in 10 seconds
    SupFlags = #{
        strategy => one_for_one,
        intensity => 5,
        period => 10
    },
    
    ChildSpecs = [
        %% Permanent: always restart (databases, caches)
        #{
            id => db_pool,
            start => {db_pool, start_link, [#{size => 10}]},
            restart => permanent,
            shutdown => 5000,     %% wait 5s for graceful shutdown
            type => worker,
            modules => [db_pool]
        },
        
        %% Transient: restart only on abnormal exit
        #{
            id => cache_warmer,
            start => {cache_warmer, start_link, []},
            restart => transient,  %% don't restart if it exits normally
            shutdown => 2000,
            type => worker
        },
        
        %% Temporary: never restart (one-shot tasks)
        #{
            id => startup_migration,
            start => {migration, start_link, []},
            restart => temporary,
            shutdown => 30000,     %% wait up to 30s for migration
            type => worker
        },
        
        %% Supervisor child (nested supervision)
        #{
            id => http_sup,
            start => {http_supervisor, start_link, []},
            restart => permanent,
            shutdown => infinity,  %% wait for all http children
            type => supervisor,
            modules => [http_supervisor]
        }
    ],
    
    {ok, {SupFlags, ChildSpecs}}.

%% Dynamic children with simple_one_for_one
-module(session_supervisor).
-behaviour(supervisor).
-export([start_link/0, start_session/1, stop_session/1, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_session(SessionId) ->
    supervisor:start_child(?MODULE, [SessionId]).

stop_session(SessionPid) ->
    supervisor:terminate_child(?MODULE, SessionPid).

init([]) ->
    ChildSpec = #{
        id => session,
        start => {session, start_link, []},
        restart => temporary,
        shutdown => 5000,
        type => worker
    },
    
    {ok, {#{strategy => simple_one_for_one, intensity => 100, period => 1}, [ChildSpec]}}.
```

---

## proc_lib and Special Processes

```erlang
%% proc_lib.erl — for processes that don't use OTP behaviors
%% but still participate in supervision trees

-module(special_process).
-export([start_link/0, init/1]).

start_link() ->
    %% proc_lib:start_link blocks until init_ack
    proc_lib:start_link(?MODULE, init, [self()]).

init(Parent) ->
    %% Register with supervisor/system
    process_flag(trap_exit, true),
    
    %% Tell parent we initialized successfully
    proc_lib:init_ack(Parent, {ok, self()}),
    
    %% Enter the main loop
    loop(#{parent => Parent}).

loop(State) ->
    receive
        %% Handle sys module messages
        {system, From, Request} ->
            sys:handle_system_msg(Request, From, maps:get(parent, State),
                                   ?MODULE, [], State);
        
        %% Handle exit from parent
        {'EXIT', Parent, Reason} when Parent =:= maps:get(parent, State) ->
            %% Graceful shutdown
            cleanup(State),
            exit(Reason);
        
        %% Normal messages
        {call, From, Request} ->
            Response = handle_call(Request, State),
            From ! {reply, Response},
            loop(State);
        
        Msg ->
            loop(handle_info(Msg, State))
    end.

%% sys module callbacks
system_continue(Parent, _Debug, State) ->
    loop(State#{parent => Parent}).

system_terminate(Reason, _Parent, _Debug, State) ->
    cleanup(State),
    exit(Reason).

system_get_state(State) ->
    {ok, State}.

system_replace_state(StateFun, State) ->
    NewState = StateFun(State),
    {ok, NewState, NewState}.

handle_call({get, Key}, State) ->
    maps:get(Key, State, undefined).

handle_info(_Msg, State) ->
    State.

cleanup(_State) ->
    ok.
```

---

## sys Module

```erlang
%% sys module: debug and inspect running processes

%% Suspend a process
sys:suspend(Pid).

%% Resume a suspended process
sys:resume(Pid).

%% Get state of a gen_server
{ok, State} = sys:get_state(Pid).

%% Replace state
sys:replace_state(Pid, fun(S) -> S#{debug => true} end).

%% Enable tracing
sys:trace(Pid, true).

%% Get statistics
sys:statistics(Pid, get).

%% Get process info (callbacks the process supports)
sys:get_status(Pid).

%% Practical: inspect a gen_server in production
inspect_process(Name) ->
    Pid = whereis(Name),
    
    %% Get state without interrupting the process
    {ok, State} = sys:get_state(Pid),
    io:format("State: ~p~n", [State]),
    
    %% Get statistics
    Stats = sys:statistics(Pid, get),
    io:format("Stats: ~p~n", [Stats]),
    
    %% Brief trace (100ms)
    sys:trace(Pid, true),
    timer:sleep(100),
    sys:trace(Pid, false).

%% Debug options for gen_server
debug_server() ->
    gen_server:start_link(?MODULE, [], [
        {debug, [trace, statistics, log]}
    ]).
```

---

## Application Callbacks

```erlang
%% myapp.erl — application callback module

-module(myapp).
-behaviour(application).
-export([start/2, stop/1, prep_stop/1]).

start(normal, _Args) ->
    logger:info("Starting myapp", #{version => ?VSN}),
    
    %% Initialize ETS tables before supervisor
    init_ets_tables(),
    
    %% Start top-level supervisor
    case myapp_sup:start_link() of
        {ok, Pid} ->
            %% Post-startup: load configuration, warm caches
            spawn(fun() -> post_startup() end),
            {ok, Pid};
        Error ->
            Error
    end;

start({takeover, OtherNode}, _Args) ->
    %% This node is taking over from OtherNode (Mnesia failover)
    logger:warning("Taking over from ~p", [OtherNode]),
    myapp_sup:start_link();

start({failover, OtherNode}, _Args) ->
    %% Starting up after OtherNode crashed
    logger:warning("Failover from ~p", [OtherNode]),
    myapp_sup:start_link().

prep_stop(State) ->
    %% Called before stop/1
    %% Gracefully drain connections
    logger:info("Preparing to stop, draining connections"),
    connection_manager:drain(30000),
    State.

stop(_State) ->
    logger:info("myapp stopped"),
    ok.

init_ets_tables() ->
    [ets:new(T, [named_table, public, T_type | T_opts])
     || {T, T_type, T_opts} <- [
         {users_cache, set, [{read_concurrency, true}]},
         {session_store, set, [{write_concurrency, true}]},
         {rate_limits, set, [{write_concurrency, true}]}
     ]].

post_startup() ->
    timer:sleep(1000),  %% Let all processes start
    cache_warmer:warm_all(),
    logger:info("Post-startup complete").

%% sys.config for application environment
application_env() ->
    [{myapp, [
        {port, 8080},
        {db_pool_size, 20},
        {cache_ttl, 3600},
        {max_connections, 10000}
    ]}].

%% Reading application config
get_config(Key, Default) ->
    application:get_env(myapp, Key, Default).
```

---

## Dynamic Supervision

```erlang
%% dynamic_supervision.erl — add/remove children at runtime

-module(worker_manager).
-behaviour(gen_server).
-export([start_link/0, add_worker/1, remove_worker/1, list_workers/0]).
-export([init/1, handle_call/3, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

add_worker(WorkerId) ->
    gen_server:call(?MODULE, {add, WorkerId}).

remove_worker(WorkerId) ->
    gen_server:call(?MODULE, {remove, WorkerId}).

list_workers() ->
    supervisor:which_children(worker_sup).

init([]) ->
    %% Start the dynamic supervisor
    {ok, _} = supervisor:start_link({local, worker_sup}, worker_supervisor, []),
    {ok, #{}}.

handle_call({add, WorkerId}, _From, State) ->
    ChildSpec = #{
        id => WorkerId,
        start => {worker, start_link, [WorkerId]},
        restart => temporary,  %% don't auto-restart removed workers
        shutdown => 5000,
        type => worker
    },
    
    case supervisor:start_child(worker_sup, ChildSpec) of
        {ok, Pid} ->
            logger:info("Worker added", #{id => WorkerId, pid => Pid}),
            {reply, {ok, Pid}, State};
        {error, Reason} ->
            {reply, {error, Reason}, State}
    end;

handle_call({remove, WorkerId}, _From, State) ->
    case supervisor:terminate_child(worker_sup, WorkerId) of
        ok ->
            supervisor:delete_child(worker_sup, WorkerId),
            logger:info("Worker removed", #{id => WorkerId}),
            {reply, ok, State};
        {error, Reason} ->
            {reply, {error, Reason}, State}
    end.

handle_info({'EXIT', Pid, Reason}, State) ->
    logger:info("Worker exited", #{pid => Pid, reason => Reason}),
    {noreply, State}.

%% Supervisor for workers
-module(worker_supervisor).
-behaviour(supervisor).

init([]) ->
    {ok, {#{strategy => simple_one_for_one, intensity => 5, period => 10}, [
        #{id => worker, start => {worker, start_link, []},
          restart => temporary, shutdown => 5000, type => worker}
    ]}}.
```

---

## ตัวอย่างจริง: Self-Healing System

```erlang
%% self_healing_sup.erl — supervisor that adapts to failures

-module(self_healing_sup).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_info/2, handle_call/3]).

-record(state, {
    processes = #{},     %% pid => {type, args, restart_count}
    max_restart_count = 5,
    restart_window_ms = 60000
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    process_flag(trap_exit, true),
    
    %% Start initial set of processes
    Workers = [
        {db_writer, fun() -> db_writer:start_link() end},
        {cache_sync, fun() -> cache_sync:start_link() end},
        {metrics_collector, fun() -> metrics_collector:start_link() end}
    ],
    
    State = lists:foldl(fun({Name, StartFun}, S) ->
        start_and_track(Name, StartFun, S)
    end, #state{}, Workers),
    
    {ok, State}.

start_and_track(Name, StartFun, State) ->
    case StartFun() of
        {ok, Pid} ->
            Processes = maps:put(Pid, #{
                name => Name,
                start_fun => StartFun,
                restart_count => 0,
                started_at => erlang:monotonic_time(millisecond)
            }, State#state.processes),
            State#state{processes = Processes};
        Error ->
            logger:error("Failed to start process", #{name => Name, error => Error}),
            State
    end.

handle_info({'EXIT', Pid, Reason}, State) ->
    case maps:take(Pid, State#state.processes) of
        {ProcessInfo, Remaining} ->
            Name = maps:get(name, ProcessInfo),
            RestartCount = maps:get(restart_count, ProcessInfo),
            
            logger:warning("Managed process died", #{
                name => Name, reason => Reason, restarts => RestartCount
            }),
            
            MaxRestarts = State#state.max_restart_count,
            
            case RestartCount < MaxRestarts of
                true ->
                    %% Restart with exponential backoff
                    Delay = min(1000 * round(math:pow(2, RestartCount)), 30000),
                    logger:info("Restarting process", #{name => Name, delay_ms => Delay}),
                    
                    StartFun = maps:get(start_fun, ProcessInfo),
                    timer:send_after(Delay, {restart, Name, StartFun, RestartCount + 1}),
                    
                    {noreply, State#state{processes = Remaining}};
                
                false ->
                    %% Too many restarts, alert and give up
                    alert_manager:critical(process_restart_limit, #{name => Name}),
                    logger:error("Process gave up restarting", #{name => Name}),
                    {noreply, State#state{processes = Remaining}}
            end;
        
        error ->
            %% Unknown process
            {noreply, State}
    end;

handle_info({restart, Name, StartFun, RestartCount}, State) ->
    logger:info("Attempting restart", #{name => Name, attempt => RestartCount}),
    
    NewState = case StartFun() of
        {ok, Pid} ->
            Processes = maps:put(Pid, #{
                name => Name,
                start_fun => StartFun,
                restart_count => RestartCount,
                started_at => erlang:monotonic_time(millisecond)
            }, State#state.processes),
            State#state{processes = Processes};
        Error ->
            logger:error("Restart failed", #{name => Name, error => Error}),
            State
    end,
    
    {noreply, NewState}.
```

---

## สรุป Part 92

| Pattern | Use Case |
|---------|---------|
| one_for_one | Independent children |
| one_for_all | Interdependent children |
| rest_for_one | Ordered initialization |
| simple_one_for_one | Dynamic identical children |
| proc_lib | Custom processes in supervision trees |
| sys module | Runtime inspection/debugging |
| Application callbacks | Startup/shutdown lifecycle |

**OTP philosophy**: Build fault-tolerant trees where failures are isolated and recovery is automatic at the right level.

---

*[← Part 91: Parse Transforms](part_91_parse_transforms.md) | [Part 93: BEAM Internals →](part_93_beam_internals.md)*
