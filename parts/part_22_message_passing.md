# Part 22: Message Passing Advanced — Communication Patterns

## สารบัญ
1. [Message Passing พื้นฐาน (ทบทวน)](#message-passing-พื้นฐาน-ทบทวน)
2. [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
3. [Request-Reply Pattern](#request-reply-pattern)
4. [Publish-Subscribe Pattern](#publish-subscribe-pattern)
5. [Pipeline Pattern](#pipeline-pattern)
6. [Scatter-Gather Pattern](#scatter-gather-pattern)
7. [Timeout Handling](#timeout-handling)
8. [Mailbox Management](#mailbox-management)
9. [ตัวอย่างจริง: Task Queue](#ตัวอย่างจริง-task-queue)

---

## Message Passing พื้นฐาน (ทบทวน)

```erlang
%% Erlang message passing rules:
%% 1. Messages ถูกส่งแบบ asynchronous (non-blocking)
%% 2. Messages มาถึง in order จาก sender เดียวกัน (FIFO per sender)
%% 3. Messages ไม่รับประกัน delivery (ใน distributed nodes)
%% 4. Mailbox: process มี unlimited mailbox
%% 5. Selective receive: เลือกรับเฉพาะ pattern ที่ match

%% Send & receive
Pid = self().
Pid ! {hello, "World"}.
receive
    {hello, Name} -> io:format("Hello ~s~n", [Name])
end.

%% Multiple messages
Pid = spawn(fun() ->
    receive X -> io:format("1st: ~p~n", [X]) end,
    receive Y -> io:format("2nd: ~p~n", [Y]) end,
    receive Z -> io:format("3rd: ~p~n", [Z]) end
end).

Pid ! a,
Pid ! b,
Pid ! c.
%% Output: 1st: a, 2nd: b, 3rd: c (in order)

%% Message ordering: guaranteed from same sender
% Sender                    Receiver
Pid ! msg1,              %  msg1 arrives before msg2
Pid ! msg2.              %  msg2 arrives after msg1 (from same sender)
```

---

## Synchronous vs Asynchronous

```erlang
%% Asynchronous (cast): fire and forget
cast(ServerPid, Message) ->
    ServerPid ! {cast, Message}.

%% ตัวอย่าง
ServerPid ! {cast, {update_counter, 5}}.
%% ส่งแล้วทำงานต่อทันที ไม่รอ reply

%% Synchronous (call): wait for reply
call(ServerPid, Request) ->
    Ref = make_ref(),
    ServerPid ! {call, self(), Ref, Request},
    receive
        {reply, Ref, Response} ->
            {ok, Response}
    after 5000 ->
        {error, timeout}
    end.

%% Server ที่จัดการทั้ง cast และ call
server_loop(State) ->
    receive
        {cast, Msg} ->
            NewState = handle_cast(Msg, State),
            server_loop(NewState);
        {call, From, Ref, Req} ->
            {Reply, NewState} = handle_call(Req, State),
            From ! {reply, Ref, Reply},
            server_loop(NewState)
    end.

handle_call(get_count, #{count := N} = State) ->
    {N, State};
handle_call({set_count, N}, State) ->
    {ok, State#{count => N}}.

handle_cast({increment, By}, #{count := N} = State) ->
    State#{count => N + By};
handle_cast(reset, State) ->
    State#{count => 0}.

%% เปรียบเทียบ
%% Cast: เร็วกว่า, ไม่บล็อก, ไม่รู้ result
%% Call: ช้ากว่า, บล็อกรอ, รู้ result
```

---

## Request-Reply Pattern

```erlang
%% Pattern ที่พบบ่อยที่สุด

%% Simple request-reply
simple_rr(ServerPid, Request) ->
    ServerPid ! {request, self(), Request},
    receive
        {response, Reply} -> Reply
    after 5000 ->
        {error, timeout}
    end.

%% Request-reply กับ unique reference (avoid cross-talk)
safe_rr(ServerPid, Request) ->
    Ref = make_ref(),
    ServerPid ! {request, self(), Ref, Request},
    receive
        {response, Ref, Reply} -> {ok, Reply}
    after 5000 ->
        {error, timeout}
    end.
%% Ref ป้องกันการรับ reply ที่ไม่ใช่ของเรา

%% Async call: send and receive later
async_call(ServerPid, Request) ->
    Ref = make_ref(),
    ServerPid ! {async_request, self(), Ref, Request},
    {pending, Ref}.

collect_async({pending, Ref}) ->
    receive
        {response, Ref, Reply} -> {ok, Reply}
    after 5000 ->
        {error, timeout}
    end.

%% ใช้งาน: send multiple requests then collect
Ref1 = async_call(Server1, req1),
Ref2 = async_call(Server2, req2),
Ref3 = async_call(Server3, req3),
%% ทำงานอื่นระหว่างรอ...
R1 = collect_async(Ref1),
R2 = collect_async(Ref2),
R3 = collect_async(Ref3).

%% Tagged messages
send_tagged(Pid, Tag, Data) ->
    Pid ! {Tag, Data}.

receive_tagged(Tag) ->
    receive
        {Tag, Data} -> {ok, Data}
    after 1000 ->
        {error, timeout}
    end.

%% Conversation pattern
conversation(Pid) ->
    Pid ! {greet, self()},
    receive
        {greeting, Msg} ->
            io:format("Server says: ~s~n", [Msg]),
            Pid ! {thanks, self()},
            receive
                {welcome, FinalMsg} ->
                    io:format("Server: ~s~n", [FinalMsg])
            end
    end.
```

---

## Publish-Subscribe Pattern

```erlang
%% pub/sub: publishers send to topic, subscribers receive

%% Simple pub-sub manager
-module(pubsub).
-export([start/0, subscribe/2, unsubscribe/2, publish/2]).

start() ->
    Pid = spawn(fun() -> ps_loop(#{}) end),
    register(pubsub, Pid),
    {ok, Pid}.

subscribe(Topic, Subscriber) ->
    pubsub ! {subscribe, Topic, Subscriber}.

unsubscribe(Topic, Subscriber) ->
    pubsub ! {unsubscribe, Topic, Subscriber}.

publish(Topic, Message) ->
    pubsub ! {publish, Topic, Message}.

ps_loop(Subscribers) ->
    receive
        {subscribe, Topic, Pid} ->
            Current = maps:get(Topic, Subscribers, []),
            Updated = maps:put(Topic, [Pid | Current], Subscribers),
            ps_loop(Updated);

        {unsubscribe, Topic, Pid} ->
            Current = maps:get(Topic, Subscribers, []),
            Updated = maps:put(Topic, lists:delete(Pid, Current), Subscribers),
            ps_loop(Updated);

        {publish, Topic, Message} ->
            lists:foreach(
                fun(Pid) ->
                    case is_process_alive(Pid) of
                        true -> Pid ! {event, Topic, Message};
                        false -> ok
                    end
                end,
                maps:get(Topic, Subscribers, [])
            ),
            ps_loop(Subscribers)
    end.
```

```erlang
%% ใช้งาน pub-sub
pubsub:start().

%% Subscribe
Self = self().
pubsub:subscribe(user_events, Self).
pubsub:subscribe(system_events, Self).

%% Publisher
spawn(fun() ->
    timer:sleep(100),
    pubsub:publish(user_events, #{type => login, user => alice})
end).

spawn(fun() ->
    timer:sleep(200),
    pubsub:publish(system_events, #{type => alert, msg => "High CPU"})
end).

%% Receive
receive {event, Topic, Msg} -> io:format("~p: ~p~n", [Topic, Msg]) end.
receive {event, Topic, Msg} -> io:format("~p: ~p~n", [Topic, Msg]) end.
```

---

## Pipeline Pattern

```erlang
%% Pipeline: ส่งข้อมูลผ่าน chain of processes

%% Start pipeline
start_pipeline(Stages) ->
    %% สร้าง stages จากขวาไปซ้าย
    lists:foldr(
        fun(StageFun, NextPid) ->
            spawn(fun() -> stage_loop(StageFun, NextPid) end)
        end,
        self(),  %% final output goes to caller
        Stages
    ).

stage_loop(Fun, Next) ->
    receive
        {data, Input} ->
            Output = Fun(Input),
            Next ! {data, Output},
            stage_loop(Fun, Next);
        done ->
            Next ! done,
            ok
    end.

%% ใช้งาน
Pipeline = start_pipeline([
    fun(X) -> X * 2 end,
    fun(X) -> X + 1 end,
    fun(X) -> X * X end
]).

Pipeline ! {data, 3}.
%% 3 -> *2 -> 6 -> +1 -> 7 -> *7 -> 49

receive {data, Result} -> io:format("Result: ~p~n", [Result]) end.
%% = 49

Pipeline ! done.

%% ตัวอย่าง: data processing pipeline
image_pipeline() ->
    start_pipeline([
        fun decode_image/1,
        fun resize_image/1,
        fun apply_filter/1,
        fun encode_image/1
    ]).

log_pipeline() ->
    start_pipeline([
        fun parse_log_line/1,
        fun enrich_with_geo/1,
        fun filter_noise/1,
        fun aggregate/1
    ]).
```

---

## Scatter-Gather Pattern

```erlang
%% Scatter-Gather: ส่งงานไปหลาย workers แล้วรวม results

%% Scatter: ส่งงานออก
scatter(Tasks, WorkerFun) ->
    Parent = self(),
    Refs = [
        begin
            Ref = make_ref(),
            spawn(fun() ->
                Result = WorkerFun(Task),
                Parent ! {result, Ref, Result}
            end),
            {Ref, Task}
        end
        || Task <- Tasks
    ],
    Refs.

%% Gather: รวม results
gather(Refs, Timeout) ->
    gather(Refs, Timeout, []).

gather([], _Timeout, Results) ->
    {ok, lists:reverse(Results)};
gather(Pending, Timeout, Results) ->
    receive
        {result, Ref, Value} ->
            case lists:keyfind(Ref, 1, Pending) of
                {Ref, _Task} ->
                    Remaining = lists:keydelete(Ref, 1, Pending),
                    gather(Remaining, Timeout, [{Ref, Value} | Results]);
                false ->
                    gather(Pending, Timeout, Results)
            end
    after Timeout ->
        {partial, Results, Pending}
    end.

%% ใช้งาน
scatter_gather(Tasks, WorkerFun, Timeout) ->
    Refs = scatter(Tasks, WorkerFun),
    gather(Refs, Timeout).

%% ตัวอย่าง: parallel HTTP requests
fetch_all(Urls) ->
    scatter_gather(
        Urls,
        fun(Url) -> http_get(Url) end,
        5000
    ).

%% Fan-out/Fan-in
fan_out(Input, Workers) ->
    lists:map(fun(W) -> W ! Input, W end, Workers).

fan_in(N) ->
    [receive Result -> Result end || _ <- lists:seq(1, N)].

%% MapReduce-like
map_reduce(Data, MapFun, ReduceFun) ->
    %% Map phase: scatter
    Parent = self(),
    Pids = [spawn(fun() -> Parent ! {mapped, MapFun(Item)} end)
            || Item <- Data],
    
    %% Collect mapped results
    Mapped = [receive {mapped, V} -> V end || _ <- Pids],
    
    %% Reduce phase
    lists:foldl(ReduceFun, hd(Mapped), tl(Mapped)).

map_reduce([1,2,3,4,5],
    fun(X) -> X * X end,
    fun(A, Acc) -> A + Acc end).
%% = 55 (1+4+9+16+25)
```

---

## Timeout Handling

```erlang
%% receive กับ timeout
receive_with_timeout(Timeout) ->
    receive
        {data, Value} -> {ok, Value}
    after Timeout ->
        {error, timeout}
    end.

%% ไม่มี timeout = รอตลอดไป (อาจ deadlock)
%% after 0 = check mailbox แล้วทำงานต่อทันที
%% after infinity = รอตลอดไป (explicit infinite wait)

%% ตัวอย่าง: receive หลาย messages ด้วย timeout
collect_responses(N, Timeout) ->
    collect_responses(N, Timeout, []).

collect_responses(0, _Timeout, Acc) ->
    {ok, lists:reverse(Acc)};
collect_responses(N, Timeout, Acc) ->
    receive
        {response, Value} ->
            collect_responses(N - 1, Timeout, [Value | Acc])
    after Timeout ->
        {partial, lists:reverse(Acc), N}
    end.

%% Deadline-based timeout (ไม่ reset timeout แต่ละ iteration)
deadline_receive(DeadlineMs) ->
    deadline_receive(DeadlineMs, []).

deadline_receive(Deadline, Acc) ->
    Now = erlang:monotonic_time(millisecond),
    Remaining = Deadline - Now,
    if Remaining =< 0 ->
        {timeout, lists:reverse(Acc)};
    true ->
        receive
            {data, V} ->
                deadline_receive(Deadline, [V | Acc])
        after Remaining ->
            {timeout, lists:reverse(Acc)}
        end
    end.

%% ใช้ deadline เพื่อรวบรวมผลใน 1 วินาที
Deadline = erlang:monotonic_time(millisecond) + 1000,
deadline_receive(Deadline).

%% Retry with backoff
retry_with_backoff(Fun, MaxRetries, BaseDelay) ->
    retry_loop(Fun, MaxRetries, BaseDelay, 0).

retry_loop(_Fun, MaxRetries, _, Attempt) when Attempt >= MaxRetries ->
    {error, max_retries};
retry_loop(Fun, MaxRetries, BaseDelay, Attempt) ->
    case Fun() of
        {ok, _} = Ok -> Ok;
        {error, _} ->
            Delay = BaseDelay * (1 bsl Attempt),
            timer:sleep(Delay),
            retry_loop(Fun, MaxRetries, BaseDelay, Attempt + 1)
    end.
```

---

## Mailbox Management

```erlang
%% Mailbox เก็บ messages ที่ยังไม่ได้รับ
%% ถ้า mailbox เติบโตเร็ว -> memory issues!

%% Check mailbox size
process_info(self(), message_queue_len).
%% = {message_queue_len, 0}

%% List messages (debug only!)
process_info(self(), messages).
%% = {messages, [msg1, msg2, ...]}

%% Flush specific pattern
flush_pattern(Pattern) ->
    receive
        Pattern -> flush_pattern(Pattern)
    after 0 ->
        ok
    end.

flush_pattern({old_data, _}).

%% Flush all (drain mailbox)
drain_mailbox() ->
    receive
        _ -> drain_mailbox()
    after 0 ->
        ok
    end.

%% Process with mailbox overflow protection
protected_loop(State, MaxQueueLen) ->
    case process_info(self(), message_queue_len) of
        {message_queue_len, Len} when Len > MaxQueueLen ->
            io:format("WARNING: mailbox overflow ~p messages~n", [Len]),
            drain_old_messages(Len - MaxQueueLen div 2);
        _ -> ok
    end,
    receive
        Msg -> handle_msg(Msg, State)
    end.

drain_old_messages(0) -> ok;
drain_old_messages(N) ->
    receive
        _ -> drain_old_messages(N - 1)
    after 0 ->
        ok
    end.

%% Best practices:
%% 1. ใช้ selective receive อย่างระวัง (scan mailbox ทั้งหมด)
%% 2. ประมวลผล messages ให้เร็ว (ไม่บล็อกใน handler)
%% 3. Monitor mailbox size ใน production
%% 4. Drop messages เมื่อ overloaded (load shedding)
%% 5. ใช้ back-pressure mechanism
```

---

## ตัวอย่างจริง: Task Queue

```erlang
%% ไฟล์: task_queue.erl
-module(task_queue).
-export([
    start/0, start/1,
    submit/2,
    submit_sync/3,
    get_stats/1,
    stop/1
]).

-record(state, {
    queue :: queue:queue(),
    workers :: [pid()],
    pending :: #{reference() => {pid(), reference()}},
    max_workers :: pos_integer(),
    completed :: non_neg_integer(),
    failed :: non_neg_integer()
}).

start() -> start(4).

start(MaxWorkers) ->
    spawn(?MODULE, init, [MaxWorkers]).

init(MaxWorkers) ->
    process_flag(trap_exit, true),
    loop(#state{
        queue = queue:new(),
        workers = [],
        pending = #{},
        max_workers = MaxWorkers,
        completed = 0,
        failed = 0
    }).

%% Submit async task
submit(QueuePid, Fun) ->
    QueuePid ! {submit, Fun}.

%% Submit sync task (wait for result)
submit_sync(QueuePid, Fun, Timeout) ->
    Ref = make_ref(),
    QueuePid ! {submit_sync, self(), Ref, Fun},
    receive
        {result, Ref, Value} -> {ok, Value};
        {error, Ref, Reason} -> {error, Reason}
    after Timeout ->
        {error, timeout}
    end.

get_stats(QueuePid) ->
    Ref = make_ref(),
    QueuePid ! {get_stats, self(), Ref},
    receive
        {stats, Ref, Stats} -> Stats
    after 1000 ->
        {error, timeout}
    end.

stop(QueuePid) ->
    QueuePid ! stop.

%% Internal loop
loop(#state{queue = Q, workers = Workers, max_workers = Max} = State) ->
    case queue:is_empty(Q) orelse length(Workers) >= Max of
        false ->
            %% можно spawn worker
            {{value, Task}, Q2} = queue:out(Q),
            {TaskFun, Caller} = Task,
            NewState = spawn_worker(TaskFun, Caller, State#state{queue = Q2}),
            loop(NewState);
        true ->
            receive_message(State)
    end.

receive_message(State) ->
    receive
        {submit, Fun} ->
            NewQ = queue:in({Fun, none}, State#state.queue),
            loop(State#state{queue = NewQ});

        {submit_sync, From, Ref, Fun} ->
            NewQ = queue:in({Fun, {From, Ref}}, State#state.queue),
            loop(State#state{queue = NewQ});

        {worker_done, WorkerPid, Result, Caller} ->
            handle_worker_done(WorkerPid, Result, Caller, State);

        {get_stats, From, Ref} ->
            Stats = #{
                queue_length => queue:len(State#state.queue),
                active_workers => length(State#state.workers),
                completed => State#state.completed,
                failed => State#state.failed
            },
            From ! {stats, Ref, Stats},
            loop(State);

        {'EXIT', Pid, Reason} when Reason =/= normal ->
            handle_worker_crash(Pid, Reason, State);

        stop ->
            [exit(W, shutdown) || W <- State#state.workers],
            ok
    end.

spawn_worker(Fun, Caller, State) ->
    Parent = self(),
    Worker = spawn_link(fun() ->
        try
            Result = Fun(),
            Parent ! {worker_done, self(), {ok, Result}, Caller}
        catch
            Class:Reason ->
                Parent ! {worker_done, self(), {error, {Class, Reason}}, Caller}
        end
    end),
    State#state{workers = [Worker | State#state.workers]}.

handle_worker_done(WorkerPid, Result, Caller, State) ->
    Workers = lists:delete(WorkerPid, State#state.workers),
    case Caller of
        {From, Ref} ->
            case Result of
                {ok, Value} -> From ! {result, Ref, Value};
                {error, Reason} -> From ! {error, Ref, Reason}
            end;
        none -> ok
    end,
    {Completed, Failed} = case Result of
        {ok, _} -> {State#state.completed + 1, State#state.failed};
        {error, _} -> {State#state.completed, State#state.failed + 1}
    end,
    loop(State#state{workers = Workers, completed = Completed, failed = Failed}).

handle_worker_crash(Pid, Reason, State) ->
    io:format("Worker ~p crashed: ~p~n", [Pid, Reason]),
    Workers = lists:delete(Pid, State#state.workers),
    loop(State#state{workers = Workers, failed = State#state.failed + 1}).
```

```erlang
%% ใช้งาน
Q = task_queue:start(4).

%% Submit async tasks
[task_queue:submit(Q, fun() ->
    timer:sleep(rand:uniform(500)),
    io:format("Task ~p done~n", [I])
end) || I <- lists:seq(1, 10)].

%% Submit sync task
task_queue:submit_sync(Q, fun() -> 1 + 1 end, 5000).
%% = {ok, 2}

%% Stats
timer:sleep(2000),
task_queue:get_stats(Q).
%% = #{queue_length => 0, active_workers => 2, completed => 8, failed => 0}

task_queue:stop(Q).
```

---

## สรุป Part 22

| Pattern | ใช้เมื่อ | Notes |
|---------|---------|-------|
| Cast (async) | Fire and forget | เร็ว, ไม่รอ result |
| Call (sync) | Need result | ช้ากว่า, ต้องระวัง timeout |
| Pub-Sub | 1-to-many | Topic-based routing |
| Pipeline | Sequential stages | ข้อมูลไหลผ่าน stages |
| Scatter-Gather | Parallel work | รวม results จากหลาย workers |

---

*[← Part 21: Processes](part_21_processes.md) | [Part 23: OTP Overview →](part_23_otp_overview.md)*
