# Part 53: Advanced Concurrency Patterns

## สารบัญ
1. [Back-pressure Patterns](#back-pressure-patterns)
2. [Work Stealing](#work-stealing)
3. [Pipeline Processing](#pipeline-processing)
4. [Fan-out/Fan-in](#fan-outfan-in)
5. [Bounded Concurrency](#bounded-concurrency)
6. [ตัวอย่างจริง: Concurrent Data Processing](#ตัวอย่างจริง-concurrent-data-processing)

---

## Back-pressure Patterns

```erlang
%% Back-pressure: ป้องกัน consumer ถูก overwhelm ด้วย producer

%% Pattern 1: Demand-driven (pull model)
-module(demand_source).
-behaviour(gen_server).

init(_) ->
    {ok, #{consumers => [], pending => []}}.

%% Consumer requests N items
handle_call({demand, N, ConsumerPid}, _From, State) ->
    Pending = maps:get(pending, State),
    Consumers = maps:get(consumers, State),
    NewConsumers = [{ConsumerPid, N} | Consumers],
    NewState = State#{consumers := NewConsumers},
    {reply, ok, maybe_dispatch(NewState)}.

handle_cast({produce, Items}, State) ->
    Pending = maps:get(pending, State) ++ Items,
    {noreply, maybe_dispatch(State#{pending := Pending})}.

maybe_dispatch(#{consumers := [], pending := _} = State) -> State;
maybe_dispatch(#{pending := []} = State) -> State;
maybe_dispatch(#{consumers := [{CPid, N} | RestC], pending := Pending} = State) ->
    {Send, RestP} = case length(Pending) >= N of
        true -> lists:split(N, Pending);
        false -> {Pending, []}
    end,
    CPid ! {items, Send},
    maybe_dispatch(State#{consumers := RestC, pending := RestP}).

%% Pattern 2: Token bucket back-pressure
-module(token_bp).
-export([new/2, acquire/2, release/2]).

new(MaxTokens, _RefillRate) ->
    ets:new(token_bp, [named_table, set, public]),
    ets:insert(token_bp, {tokens, MaxTokens}).

acquire(Table, N) ->
    case ets:update_counter(Table, tokens, {2, -N, 0, 0}) of
        Available when Available >= 0 -> ok;
        _ ->
            %% Not enough tokens, wait
            timer:sleep(10),
            acquire(Table, N)
    end.

release(Table, N) ->
    ets:update_counter(Table, tokens, N).

%% Pattern 3: Mailbox size limit via gen_server call
-module(bounded_queue).
-behaviour(gen_server).
-export([push/2, pop/1]).

-define(MAX_QUEUE, 10000).

push(Pid, Item) ->
    case gen_server:call(Pid, {push, Item}, 5000) of
        ok -> ok;
        {error, queue_full} -> {error, back_pressure}
    end.

pop(Pid) ->
    gen_server:call(Pid, pop).

handle_call({push, Item}, _From, State) ->
    Queue = maps:get(queue, State),
    case queue:len(Queue) >= ?MAX_QUEUE of
        true ->
            {reply, {error, queue_full}, State};
        false ->
            {reply, ok, State#{queue := queue:in(Item, Queue)}}
    end;

handle_call(pop, _From, State) ->
    Queue = maps:get(queue, State),
    case queue:out(Queue) of
        {{value, Item}, NewQ} ->
            {reply, {ok, Item}, State#{queue := NewQ}};
        {empty, _} ->
            {reply, {error, empty}, State}
    end.
```

---

## Work Stealing

```erlang
%% Work stealing: idle workers steal tasks from busy workers

-module(work_stealing_pool).
-behaviour(gen_server).
-export([start_link/1, submit/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link(NumWorkers) ->
    gen_server:start_link(?MODULE, NumWorkers, []).

submit(PoolPid, Task) ->
    gen_server:cast(PoolPid, {submit, Task}).

init(NumWorkers) ->
    Workers = [start_worker(self()) || _ <- lists:seq(1, NumWorkers)],
    Queues = maps:from_list([{W, queue:new()} || W <- Workers]),
    {ok, #{workers => Workers, queues => Queues}}.

start_worker(PoolPid) ->
    Pid = spawn_link(fun() -> worker_loop(PoolPid, queue:new()) end),
    Pid.

%% Distribute task round-robin initially
handle_cast({submit, Task}, #{workers := Workers, queues := Queues} = State) ->
    {LeastBusy, _} = lists:foldl(fun(W, {Best, BestLen}) ->
        Len = queue:len(maps:get(W, Queues, queue:new())),
        case Len < BestLen of
            true -> {W, Len};
            false -> {Best, BestLen}
        end
    end, {hd(Workers), infinity}, Workers),
    LeastBusy ! {task, Task},
    {noreply, State};

%% Worker idle: can it steal from others?
handle_call({steal_work, WorkerPid}, _From, #{queues := Queues} = State) ->
    BusiestWorker = find_busiest(WorkerPid, Queues),
    case BusiestWorker of
        none -> {reply, no_work, State};
        {Busy, BusyQ} ->
            case queue:out(BusyQ) of
                {{value, Task}, NewBusyQ} ->
                    NewQueues = maps:put(Busy, NewBusyQ, Queues),
                    {reply, {task, Task}, State#{queues := NewQueues}};
                {empty, _} ->
                    {reply, no_work, State}
            end
    end.

find_busiest(ExcludePid, Queues) ->
    Candidates = maps:to_list(maps:remove(ExcludePid, Queues)),
    case Candidates of
        [] -> none;
        _ ->
            {BusiestPid, BusiestQ} = lists:foldl(fun({P, Q}, {BP, BQ}) ->
                case queue:len(Q) > queue:len(BQ) of
                    true -> {P, Q};
                    false -> {BP, BQ}
                end
            end, hd(Candidates), tl(Candidates)),
            case queue:len(BusiestQ) > 0 of
                true -> {BusiestPid, BusiestQ};
                false -> none
            end
    end.

worker_loop(PoolPid, LocalQueue) ->
    receive
        {task, Task} ->
            execute_task(Task),
            worker_loop(PoolPid, LocalQueue)
    after 0 ->
        %% No local work: try to steal
        case gen_server:call(PoolPid, {steal_work, self()}) of
            {task, Task} ->
                execute_task(Task),
                worker_loop(PoolPid, LocalQueue);
            no_work ->
                %% Actually idle
                receive
                    {task, Task} ->
                        execute_task(Task),
                        worker_loop(PoolPid, LocalQueue)
                end
        end
    end.

execute_task({Fun, Args}) ->
    catch apply(Fun, Args).
```

---

## Pipeline Processing

```erlang
%% Pipeline: chain of stages, each processes and passes to next

-module(pipeline).
-export([new/1, process/2, stats/1]).

%% Create pipeline from list of stage functions
new(Stages) ->
    {ok, Pid} = gen_server:start_link(?MODULE, Stages, []),
    Pid.

process(Pipeline, Data) ->
    gen_server:call(Pipeline, {process, Data}).

init(Stages) ->
    %% Spawn a process per stage
    StagePids = spawn_pipeline(Stages, self()),
    {ok, #{stages => StagePids, stats => #{}}}.

spawn_pipeline([], _Sink) -> [];
spawn_pipeline([Stage | Rest], Sink) ->
    NextPid = case Rest of
        [] -> Sink;
        _ -> hd(spawn_pipeline(Rest, Sink))
    end,
    Pid = spawn_link(fun() -> stage_loop(Stage, NextPid, #{count => 0}) end),
    [Pid | spawn_pipeline(Rest, Sink)].  %% Note: simplified

stage_loop(StageFun, Next, Stats) ->
    receive
        {data, Item, ReplyTo} ->
            case StageFun(Item) of
                {ok, Result} ->
                    case Next of
                        Pid when is_pid(Pid) ->
                            Pid ! {data, Result, ReplyTo};
                        _ ->
                            ReplyTo ! {pipeline_result, Result}
                    end;
                {error, Reason} ->
                    ReplyTo ! {pipeline_error, Reason}
            end,
            stage_loop(StageFun, Next, maps:update(count, fun(C) -> C+1 end, Stats));
        stop -> ok
    end.

%% Example pipeline: HTTP data processing
http_pipeline() ->
    Stages = [
        fun parse_json/1,
        fun validate_schema/1,
        fun transform_data/1,
        fun persist_to_db/1
    ],
    pipeline:new(Stages).

parse_json(Raw) ->
    try {ok, jsx:decode(Raw, [return_maps])}
    catch _:_ -> {error, invalid_json}
    end.

validate_schema(Data) ->
    case maps:get(<<"id">>, Data, undefined) of
        undefined -> {error, missing_id};
        _ -> {ok, Data}
    end.

transform_data(Data) ->
    Transformed = maps:fold(fun(K, V, Acc) ->
        Acc#{string:lowercase(K) => V}
    end, #{}, Data),
    {ok, Transformed}.

persist_to_db(Data) ->
    db:insert(Data).
```

---

## Fan-out/Fan-in

```erlang
%% Fan-out: broadcast work to multiple workers
%% Fan-in: collect results from multiple workers

-module(fan).
-export([map_parallel/2, map_parallel/3]).

%% Parallel map with timeout
map_parallel(Fun, List) ->
    map_parallel(Fun, List, 30000).

map_parallel(Fun, List, Timeout) ->
    Parent = self(),
    Refs = lists:map(fun(Item) ->
        Ref = make_ref(),
        spawn(fun() ->
            Result = try {ok, Fun(Item)}
                     catch _:R -> {error, R}
                     end,
            Parent ! {Ref, Result}
        end),
        {Ref, Item}
    end, List),
    collect_results(Refs, Timeout).

collect_results([], _) -> [];
collect_results([{Ref, _Item} | Rest], Timeout) ->
    receive
        {Ref, Result} ->
            [Result | collect_results(Rest, Timeout)]
    after Timeout ->
        [{error, timeout} | collect_results(Rest, 0)]
    end.

%% Better: collect all within timeout
map_parallel_timed(Fun, List, Timeout) ->
    Parent = self(),
    Tag = make_ref(),
    Pids = [begin
        spawn(fun() ->
            Result = try {ok, Fun(Item)}
                     catch _:R -> {error, R}
                     end,
            Parent ! {Tag, I, Result}
        end)
    end || {I, Item} <- lists:enumerate(List)],
    Deadline = erlang:monotonic_time(millisecond) + Timeout,
    collect_indexed(length(List), Deadline, Tag, #{}).

collect_indexed(0, _, _, Results) ->
    [maps:get(I, Results) || I <- lists:seq(1, map_size(Results))];
collect_indexed(N, Deadline, Tag, Results) ->
    Remaining = Deadline - erlang:monotonic_time(millisecond),
    receive
        {Tag, I, Result} ->
            collect_indexed(N - 1, Deadline, Tag, Results#{I => Result})
    after max(0, Remaining) ->
        %% Fill remaining with timeout errors
        maps:values(maps:merge(
            maps:from_list([{I, {error, timeout}} || I <- lists:seq(1, N + map_size(Results))]),
            Results
        ))
    end.

%% Aggregation: reduce fan-in results
aggregate(Results, AggregateFun) ->
    {OkResults, Errors} = lists:partition(
        fun({ok, _}) -> true; (_) -> false end,
        Results
    ),
    Values = [V || {ok, V} <- OkResults],
    #{
        result => AggregateFun(Values),
        errors => length(Errors),
        ok_count => length(Values)
    }.
```

---

## Bounded Concurrency

```erlang
%% Semaphore: limit concurrent operations
-module(semaphore).
-behaviour(gen_server).
-export([new/1, acquire/1, release/1, acquire/2]).

new(Limit) ->
    gen_server:start_link(?MODULE, Limit, []).

acquire(Sem) ->
    acquire(Sem, infinity).

acquire(Sem, Timeout) ->
    gen_server:call(Sem, acquire, Timeout).

release(Sem) ->
    gen_server:cast(Sem, release).

init(Limit) ->
    {ok, #{available => Limit, waiting => queue:new()}}.

handle_call(acquire, From, #{available := N} = State) when N > 0 ->
    {reply, ok, State#{available := N - 1}};
handle_call(acquire, From, #{waiting := WQ} = State) ->
    %% No capacity: put caller in wait queue
    {noreply, State#{waiting := queue:in(From, WQ)}}.

handle_cast(release, #{available := N, waiting := WQ} = State) ->
    case queue:out(WQ) of
        {{value, From}, NewWQ} ->
            gen_server:reply(From, ok),
            {noreply, State#{waiting := NewWQ}};
        {empty, _} ->
            {noreply, State#{available := N + 1}}
    end.

%% Concurrency limiter: wrap any function
with_semaphore(Sem, Fun) ->
    ok = semaphore:acquire(Sem),
    try Fun()
    after semaphore:release(Sem)
    end.

%% Parallel processing with concurrency limit
parallel_limited(Items, Fun, MaxConcurrent) ->
    Sem = semaphore:new(MaxConcurrent),
    map_parallel(fun(Item) ->
        with_semaphore(Sem, fun() -> Fun(Item) end)
    end, Items).
```

---

## ตัวอย่างจริง: Concurrent Data Processing

```erlang
%% image_processor.erl: process 1000 images concurrently

-module(image_processor).
-export([process_batch/2]).

process_batch(ImagePaths, Opts) ->
    MaxConcurrent = maps:get(max_concurrent, Opts, 10),
    Timeout = maps:get(timeout, Opts, 60000),
    
    %% Create semaphore for concurrency control
    Sem = semaphore:new(MaxConcurrent),
    
    %% Setup pipeline stages
    Pipeline = [
        fun read_image/1,
        fun decode_image/1,
        fun resize_image/1,
        fun compress_image/1,
        fun upload_to_s3/1
    ],
    
    %% Process all images with bounded concurrency
    Results = fan:map_parallel(
        fun(Path) ->
            with_semaphore(Sem, fun() ->
                run_pipeline(Path, Pipeline)
            end)
        end,
        ImagePaths,
        Timeout
    ),
    
    %% Aggregate results
    {OkCount, Errors} = lists:foldl(fun
        ({ok, _}, {Ok, E}) -> {Ok + 1, E};
        ({error, R}, {Ok, E}) -> {Ok, [R | E]}
    end, {0, []}, Results),
    
    #{
        processed => OkCount,
        failed => length(Errors),
        errors => Errors
    }.

run_pipeline(Input, []) -> {ok, Input};
run_pipeline(Input, [Stage | Rest]) ->
    case Stage(Input) of
        {ok, Output} -> run_pipeline(Output, Rest);
        {error, _} = E -> E
    end.

read_image(Path) ->
    case file:read_file(Path) of
        {ok, Data} -> {ok, #{path => Path, data => Data}};
        {error, R} -> {error, {read_error, Path, R}}
    end.

decode_image(#{data := Data} = State) ->
    %% Use NIF for fast decoding
    case image_nif:decode(Data) of
        {ok, Pixels} -> {ok, State#{pixels => Pixels}};
        error -> {error, decode_failed}
    end.

resize_image(#{pixels := Pixels} = State) ->
    Resized = image_nif:resize(Pixels, 800, 600),
    {ok, State#{pixels => Resized}}.

compress_image(#{pixels := Pixels} = State) ->
    Compressed = image_nif:encode_jpeg(Pixels, 85),
    {ok, State#{compressed => Compressed}}.

upload_to_s3(#{path := Path, compressed := Data}) ->
    Key = filename:basename(Path),
    s3:put_object(<<"my-bucket">>, Key, Data).
```

---

## สรุป Part 53

| Pattern | Problem Solved | Key Mechanism |
|---------|----------------|---------------|
| Back-pressure | Producer overwhelms consumer | Token bucket, demand |
| Work Stealing | Load imbalance | Idle workers steal tasks |
| Pipeline | Sequential stages | Process chain |
| Fan-out/in | Parallel scatter-gather | spawn + collect |
| Semaphore | Limit concurrency | Counter + wait queue |

---

*[← Part 52: Memory Performance](part_52_memory_performance.md) | [Part 54: Metaprogramming →](part_54_metaprogramming.md)*
