# Part 58: Actor Model Deep Dive

## สารบัญ
1. [Actor Model Theory](#actor-model-theory)
2. [Erlang Process Model](#erlang-process-model)
3. [Message Passing Patterns](#message-passing-patterns)
4. [Process Hierarchies](#process-hierarchies)
5. [Failure Handling](#failure-handling)
6. [ตัวอย่างจริง: Distributed Task System](#distributed-task-system)

---

## Actor Model Theory

```
Actor Model (Carl Hewitt, 1973):
- ทุก actor คือ unit of computation แบบ isolated
- Actors communicate ONLY via messages (no shared state)
- เมื่อ receive message, actor สามารถ:
  1. Send messages to other actors
  2. Create new actors
  3. Change its own behavior (next message handler)
  4. Update its own state

Erlang implements Actor Model faithfully:
- Process = Actor
- Mailbox = Message queue
- spawn/spawn_link = create new actor
- send (!) = send message
- receive = handle message
- gen_server:handle_call/cast = behavior change
```

---

## Erlang Process Model

```erlang
%% Process: lightest-weight Actor
%% - 300 bytes initial heap
%% - Independent GC per process
%% - No shared memory (data is copied on send)
%% - Can run millions of concurrent processes

%% Process lifecycle
lifecycle_demo() ->
    %% Create
    Pid = spawn(fun() ->
        %% Actor behavior: wait for messages
        actor_loop(#{count => 0})
    end),
    
    %% Send messages
    Pid ! {increment, 5},
    Pid ! {get_count, self()},
    
    receive
        {count, N} -> io:format("Count: ~p~n", [N])
    after 1000 ->
        timeout
    end.

actor_loop(State) ->
    receive
        {increment, N} ->
            actor_loop(State#{count := maps:get(count, State) + N});
        {get_count, From} ->
            From ! {count, maps:get(count, State)},
            actor_loop(State);
        stop ->
            ok
    end.

%% Process identity
process_info_demo(Pid) ->
    [
        process_info(Pid, registered_name),
        process_info(Pid, current_function),
        process_info(Pid, message_queue_len),
        process_info(Pid, memory),
        process_info(Pid, links),
        process_info(Pid, monitors)
    ].

%% Process limits
process_limits() ->
    #{
        max_processes => erlang:system_info(process_limit),
        current => erlang:system_info(process_count),
        pid_max => erlang:system_info(max_processes)
    }.
```

---

## Message Passing Patterns

```erlang
%% Pattern 1: Fire and forget (async)
fire_and_forget(Pid, Message) ->
    Pid ! Message.

%% Pattern 2: Request-Reply (sync)
request_reply(Pid, Request) ->
    Ref = make_ref(),
    Pid ! {request, self(), Ref, Request},
    receive
        {reply, Ref, Response} -> Response
    after 5000 ->
        {error, timeout}
    end.

%% Pattern 3: Selective receive
selective_receive(Pid) ->
    Pid ! task1,
    Pid ! task2,
    Pid ! task3,
    %% Process in priority order, not arrival order
    receive
        {high_priority, Msg} -> handle_urgent(Msg)
    after 0 ->
        receive
            {normal, Msg} -> handle_normal(Msg)
        end
    end.

%% Pattern 4: Broadcast
broadcast(Pids, Message) ->
    [Pid ! Message || Pid <- Pids].

%% Pattern 5: Gather (fan-out + collect)
gather(Requests) ->
    Parent = self(),
    Refs = [begin
        Ref = make_ref(),
        spawn(fun() ->
            Result = process(Req),
            Parent ! {Ref, Result}
        end),
        Ref
    end || Req <- Requests],
    [receive {R, Result} -> Result end || R <- Refs].

%% Pattern 6: Timeout + retry
robust_call(Pid, Request) ->
    robust_call(Pid, Request, 3).

robust_call(_, _, 0) -> {error, max_retries};
robust_call(Pid, Request, Retries) ->
    Ref = make_ref(),
    Pid ! {Request, self(), Ref},
    receive
        {ok, Ref, Result} -> {ok, Result};
        {error, Ref, Reason} -> {error, Reason}
    after 2000 ->
        robust_call(Pid, Request, Retries - 1)
    end.

%% Pattern 7: Synchronous with timeout handling
call_with_deadline(Pid, Request, Deadline) ->
    Remaining = Deadline - erlang:monotonic_time(millisecond),
    if Remaining =< 0 ->
        {error, deadline_exceeded};
    true ->
        Ref = make_ref(),
        Pid ! {request, self(), Ref, Request},
        receive
            {reply, Ref, Response} -> {ok, Response}
        after Remaining ->
            {error, timeout}
        end
    end.
```

---

## Process Hierarchies

```erlang
%% Supervision tree: OTP way to structure actors

%% Root supervisor
-module(app_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init(_) ->
    %% Strategy: one_for_one (restart only failed child)
    %% other strategies: one_for_all, rest_for_one, simple_one_for_one
    {ok, {#{strategy => one_for_one,
             intensity => 10,  %% max restarts
             period => 60},    %% per 60 seconds
     [
         #{id => data_store,
           start => {data_store, start_link, []},
           restart => permanent,
           type => worker},
         
         #{id => worker_sup,
           start => {worker_sup, start_link, []},
           restart => permanent,
           type => supervisor},   %% nested supervisor
         
         #{id => event_handler,
           start => {event_handler, start_link, []},
           restart => transient,  %% only restart if abnormal exit
           type => worker}
     ]}}.

%% Dynamic worker supervisor
-module(worker_sup).
-behaviour(supervisor).
-export([start_link/0, start_worker/1, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_worker(WorkerId) ->
    supervisor:start_child(?MODULE, [WorkerId]).

init(_) ->
    %% simple_one_for_one: template for dynamic children
    {ok, {#{strategy => simple_one_for_one,
             intensity => 100,
             period => 60},
     [#{id => worker,
        start => {dynamic_worker, start_link, []},
        restart => temporary,  %% don't restart
        type => worker}]}}.

%% Worker with linked sub-actors
-module(task_coordinator).
-behaviour(gen_server).
-export([start_link/1, run_task/2]).

run_task(CoordPid, Task) ->
    gen_server:call(CoordPid, {run, Task}, 60000).

handle_call({run, Task}, From, State) ->
    %% Spawn subtasks as linked processes
    SubPids = [begin
        Pid = spawn_link(fun() ->
            SubResult = process_subtask(Sub),
            exit({subtask_done, SubResult})
        end),
        Pid
    end || Sub <- split_task(Task)],
    
    %% Collect subtask results
    Results = collect_subtask_results(SubPids, []),
    Reply = merge_results(Results),
    {reply, Reply, State}.

collect_subtask_results([], Acc) -> lists:reverse(Acc);
collect_subtask_results([Pid | Rest], Acc) ->
    receive
        {'EXIT', Pid, {subtask_done, Result}} ->
            collect_subtask_results(Rest, [Result | Acc]);
        {'EXIT', Pid, Reason} ->
            {error, Reason}
    after 30000 ->
        {error, timeout}
    end.
```

---

## Failure Handling

```erlang
%% Actor failure isolation: key to Erlang reliability

%% Link: bidirectional death notification
link_demo() ->
    Pid = spawn_link(fun() ->
        receive crash -> error(intentional_crash) end
    end),
    %% If Pid crashes, this process also dies (linked)
    Pid ! crash.

%% Monitor: unidirectional death notification
monitor_demo() ->
    Pid = spawn(fun() ->
        timer:sleep(1000)
    end),
    Ref = monitor(process, Pid),
    receive
        {'DOWN', Ref, process, Pid, Reason} ->
            io:format("Process ~p died: ~p~n", [Pid, Reason])
    end.

%% Trap exits: convert death to message
trap_exit_demo() ->
    process_flag(trap_exit, true),
    Pid = spawn_link(fun() ->
        error(simulated_crash)
    end),
    receive
        {'EXIT', Pid, Reason} ->
            io:format("Caught exit from ~p: ~p~n", [Pid, Reason])
    end.

%% Let it crash philosophy
good_handler(Req) ->
    %% Don't defensively handle every error
    %% Let bad inputs crash this process
    %% Supervisor will restart it
    process_request(Req).  %% may crash on invalid input

bad_handler(Req) ->
    try process_request(Req)
    catch
        _:_ -> {error, something_went_wrong}  %% hiding bugs!
    end.

%% However, expected errors SHOULD be handled
better_handler(Req) ->
    case validate(Req) of
        {ok, Valid} ->
            process_request(Valid);
        {error, Reason} ->
            {error, {invalid_request, Reason}}
    end.

%% Error propagation via exit signals
propagate_error() ->
    spawn(fun() ->
        process_flag(trap_exit, true),
        Workers = [spawn_link(fun worker/0) || _ <- lists:seq(1, 5)],
        collect_or_die(Workers, [])
    end).

collect_or_die([], Acc) ->
    io:format("All done: ~p~n", [Acc]);
collect_or_die(Remaining, Acc) ->
    receive
        {done, Pid, Result} ->
            collect_or_die(lists:delete(Pid, Remaining), [Result | Acc]);
        {'EXIT', Pid, Reason} when Reason /= normal ->
            logger:error("Worker failed", #{pid => Pid, reason => Reason}),
            %% Kill remaining workers
            [exit(P, kill) || P <- Remaining, P /= Pid],
            {error, worker_failed}
    end.

worker() ->
    timer:sleep(rand:uniform(1000)),
    self() ! {done, self(), computed}.
```

---

## ตัวอย่างจริง: Distributed Task System

```erlang
%% Distributed task execution across Erlang cluster

-module(task_system).
-export([start/0, submit_task/2, get_result/1]).

start() ->
    task_registry_sup:start_link().

submit_task(TaskId, TaskFun) ->
    %% Find least loaded node
    Node = pick_node(),
    rpc:call(Node, task_executor, run, [TaskId, TaskFun]).

get_result(TaskId) ->
    task_store:lookup(TaskId).

pick_node() ->
    Nodes = [node() | nodes()],
    LoadInfo = [{N, get_load(N)} || N <- Nodes],
    {LeastLoaded, _} = lists:min(fun({_, L1}, {_, L2}) -> L1 =< L2 end, LoadInfo),
    LeastLoaded.

get_load(Node) ->
    rpc:call(Node, erlang, system_info, [process_count]).

%% task_executor.erl
-module(task_executor).
-export([run/2]).

run(TaskId, TaskFun) ->
    Pid = spawn(fun() ->
        Result = try TaskFun()
            catch Class:Reason ->
                {error, {Class, Reason}}
            end,
        task_store:store(TaskId, Result)
    end),
    {ok, Pid}.

%% task_store.erl: distributed via Mnesia
-module(task_store).
-export([store/2, lookup/1]).

store(TaskId, Result) ->
    mnesia:transaction(fun() ->
        mnesia:write({task_results, TaskId, Result, erlang:system_time()})
    end).

lookup(TaskId) ->
    case mnesia:transaction(fun() ->
        mnesia:read(task_results, TaskId)
    end) of
        {atomic, [{_, _, Result, _}]} -> {ok, Result};
        {atomic, []} -> {error, not_found}
    end.
```

---

## สรุป Part 58

| Actor Concept | Erlang Implementation |
|---------------|----------------------|
| Actor | Process (spawn) |
| Message | ! (send operator) |
| Mailbox | Process mailbox |
| Behavior change | receive new pattern |
| Create actor | spawn/spawn_link |
| Failure | exit signal + supervisor |
| Location transparency | registered name / global |

---

*[← Part 57: Erlang & Elixir](part_57_erlang_elixir.md) | [Part 59: Large-Scale System Design →](part_59_large_scale.md)*
