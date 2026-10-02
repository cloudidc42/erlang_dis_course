# Part 21: Processes — Actor Model และ Concurrency

## สารบัญ
1. [Erlang Process คืออะไร](#erlang-process-คืออะไร)
2. [Spawning Processes](#spawning-processes)
3. [Process Identity](#process-identity)
4. [Message Passing](#message-passing)
5. [Process Monitoring](#process-monitoring)
6. [Process Linking](#process-linking)
7. [Registered Processes](#registered-processes)
8. [Process Dictionary](#process-dictionary)
9. [ตัวอย่างจริง: Concurrent Counter](#ตัวอย่างจริง-concurrent-counter)

---

## Erlang Process คืออะไร

```erlang
%% Erlang Process คือ lightweight unit of concurrency
%% ไม่ใช่ OS process หรือ OS thread!

%% ลักษณะสำคัญ:
%% - Isolation: ไม่แชร์ memory กัน (share nothing)
%% - Lightweight: สร้างได้หลายล้าน process ใน VM เดียว
%% - Fast: spawn < 1µs, initial heap ~300 words
%% - Communication: ผ่าน message passing เท่านั้น
%% - Location transparent: ทำงานกับ processes บน node อื่นได้เหมือนกัน

%% Actor Model:
%% - แต่ละ process เป็น "actor"
%% - Actor มี mailbox รับ messages
%% - Actor ประมวลผลทีละ message
%% - Actor สื่อสารผ่าน message เท่านั้น

%% เปรียบเทียบ:
%% OS Thread: ใช้ MB memory, limited number, shared memory, locks
%% Erlang Process: ใช้ KB memory, millions possible, isolated, message passing

%% ตัวเลขจริง
%% WhatsApp: 2 million connections บน 1 server ด้วย Erlang
%% Discord: ใช้ Erlang สำหรับ Presence service
%% RabbitMQ: ทุก connection, channel, queue เป็น process
```

---

## Spawning Processes

```erlang
%% spawn/1 - create new process with fun
Pid = spawn(fun() ->
    io:format("Hello from new process!~n")
end).
%% = <0.123.0>

%% spawn/3 - create with Module:Function/Args
Pid2 = spawn(mymodule, my_function, [Arg1, Arg2]).

%% spawn_link/1 - spawn and link (crash together)
Pid3 = spawn_link(fun() -> do_work() end).

%% spawn_monitor/1 - spawn and monitor (OTP 23+)
{Pid4, MonRef} = spawn_monitor(fun() -> do_work() end).

%% ตัวอย่าง: parallel computation
parallel_map(Fun, List) ->
    Parent = self(),
    Pids = [spawn(fun() ->
                Parent ! {self(), Fun(X)}
            end) || X <- List],
    [receive {Pid, Result} -> Result end || Pid <- Pids].

parallel_map(fun(X) -> X * X end, [1,2,3,4,5]).
%% = [1,4,9,16,25] (order preserved)

%% spawn กับ argument passing
start_worker(Job) ->
    spawn(fun() ->
        io:format("Processing: ~p~n", [Job]),
        Result = process_job(Job),
        io:format("Done: ~p~n", [Result])
    end).

%% spawn แบบ module function (สำหรับ hot code reload)
start_worker2(Job) ->
    spawn(?MODULE, worker_loop, [Job]).

worker_loop(Job) ->
    Result = process_job(Job),
    io:format("Result: ~p~n", [Result]).
```

---

## Process Identity

```erlang
%% PID format: <Node.Creator.ID>
%% <0.1.0> = node 0, creator 1, id 0 (Erlang VM process)
%% <0.123.0> = local process #123

%% Get current process PID
self().  %% = <0.XX.0>

%% Check if PID is alive
is_process_alive(Pid).    %% = true | false

%% PID to/from string
Pid = self().
PidStr = pid_to_list(Pid).     %% = "<0.123.0>"
Pid2 = list_to_pid(PidStr).    %% = <0.123.0>

%% Process info
process_info(self()).
%% Returns list of {Key, Value} pairs

process_info(self(), memory).              %% = {memory, 2688}
process_info(self(), heap_size).           %% = {heap_size, 233}
process_info(self(), stack_size).          %% = {stack_size, 3}
process_info(self(), message_queue_len).   %% = {message_queue_len, 0}
process_info(self(), messages).            %% = {messages, [...]}
process_info(self(), current_function).    %% = {current_function, {M,F,A}}
process_info(self(), initial_call).        %% = {initial_call, {M,F,A}}
process_info(self(), status).              %% = {status, running}
process_info(self(), trap_exit).           %% = {trap_exit, false}
process_info(self(), links).               %% = {links, [...]}
process_info(self(), monitors).            %% = {monitors, [...]}
process_info(self(), registered_name).     %% = {registered_name, my_server}

%% All processes
erlang:processes().  %% list of all PIDs
length(erlang:processes()).  %% count

%% Process list info
[{P, process_info(P, registered_name)} || P <- erlang:processes(),
                                           process_info(P, registered_name) =/= undefined].
```

---

## Message Passing

```erlang
%% Send: Pid ! Message
%% Receive: receive ... end

%% พื้นฐาน
Pid = spawn(fun() ->
    receive
        hello ->
            io:format("Got hello!~n");
        {msg, Content} ->
            io:format("Got: ~p~n", [Content])
    end
end).

Pid ! hello.
Pid ! {msg, "world"}.

%% Request-Response pattern
server() ->
    receive
        {call, From, Ref, Request} ->
            Response = handle_request(Request),
            From ! {reply, Ref, Response},
            server();
        stop ->
            ok
    end.

call(ServerPid, Request) ->
    Ref = make_ref(),
    ServerPid ! {call, self(), Ref, Request},
    receive
        {reply, Ref, Response} -> Response
    after 5000 ->
        {error, timeout}
    end.

%% ตัวอย่าง: echo server
start_echo() ->
    spawn(fun echo_loop/0).

echo_loop() ->
    receive
        {From, Msg} ->
            From ! {echo, Msg},
            echo_loop();
        stop ->
            ok
    end.

Echo = start_echo().
Echo ! {self(), "hello"}.
receive {echo, Msg} -> io:format("Echo: ~s~n", [Msg]) end.

%% Selective receive (skip messages that don't match)
wait_for_specific(Ref) ->
    receive
        {result, Ref, Value} ->
            {ok, Value}
        %% อื่นๆ จะถูก skip และรออยู่ใน mailbox
    after 1000 ->
        {error, timeout}
    end.

%% Flush mailbox
flush() ->
    receive
        _ -> flush()
    after 0 ->
        ok
    end.

%% Priority receive (receive specific patterns first)
priority_receive() ->
    receive
        {high_priority, Msg} -> {high, Msg}
    after 0 ->
        receive
            {normal, Msg} -> {normal, Msg}
        after 0 ->
            empty
        end
    end.
```

---

## Process Monitoring

```erlang
%% Monitor ทำให้รู้เมื่อ process ตาย แต่ไม่ผูกชะตากัน

%% erlang:monitor/2
Pid = spawn(fun() ->
    timer:sleep(1000),
    exit(something_wrong)
end).

Ref = erlang:monitor(process, Pid).
%% ถ้า Pid ตาย -> receive {'DOWN', Ref, process, Pid, Reason}

receive
    {'DOWN', Ref, process, Pid, Reason} ->
        io:format("Process ~p died: ~p~n", [Pid, Reason])
after 2000 ->
    io:format("Timeout~n")
end.

%% spawn_monitor/1 (convenient)
{Pid2, Ref2} = spawn_monitor(fun() ->
    do_work()
end).

receive
    {'DOWN', Ref2, process, Pid2, normal}  -> success;
    {'DOWN', Ref2, process, Pid2, Reason} -> {failed, Reason}
end.

%% Demonitor
erlang:demonitor(Ref).
erlang:demonitor(Ref, [flush]).  %% flush pending DOWN message

%% Monitor named process
Ref3 = erlang:monitor(process, my_server).
%% รับ {'DOWN', Ref3, process, my_server, noproc} ถ้าไม่มี

%% Monitor by name and get PID
case whereis(my_server) of
    undefined ->
        {error, not_registered};
    Pid ->
        Ref = erlang:monitor(process, Pid),
        {ok, Pid, Ref}
end.

%% ตัวอย่าง: worker pool with monitoring
start_workers(N, WorkFun) ->
    Workers = [
        begin
            {P, R} = spawn_monitor(fun() -> worker(WorkFun) end),
            {P, R}
        end
        || _ <- lists:seq(1, N)
    ],
    monitor_workers(Workers).

monitor_workers([]) -> done;
monitor_workers(Workers) ->
    receive
        {'DOWN', Ref, process, Pid, Reason} ->
            io:format("Worker ~p died: ~p~n", [Pid, Reason]),
            Remaining = [{P,R} || {P,R} <- Workers, R =/= Ref],
            monitor_workers(Remaining)
    end.
```

---

## Process Linking

```erlang
%% Link: two processes share fate
%% ถ้า linked process ตาย -> ส่ง exit signal ไปหา linked processes

%% spawn_link: spawn and link atomically
Pid = spawn_link(fun() ->
    timer:sleep(500),
    exit(crash)
end).
%% เมื่อ Pid crash -> current process ได้รับ exit signal -> ตายด้วย!

%% ป้องกัน: trap_exit
process_flag(trap_exit, true).
%% ตอนนี้ exit signal ถูก convert เป็น message {'EXIT', Pid, Reason}

Pid2 = spawn_link(fun() -> exit(crash) end).
receive
    {'EXIT', Pid2, Reason} ->
        io:format("Linked process crashed: ~p~n", [Reason])
end.

%% Link/Unlink manually
link(Pid).
unlink(Pid).

%% ตัวอย่าง: linked server
start_server() ->
    process_flag(trap_exit, true),
    spawn_link(fun server_loop/0).

server_loop() ->
    receive
        {From, Request} ->
            Reply = handle(Request),
            From ! Reply,
            server_loop();
        {'EXIT', _Pid, Reason} ->
            io:format("Linked process died: ~p~n", [Reason]),
            server_loop()
    end.

%% Link vs Monitor comparison
%% Link: bidirectional, crash together (unless trap_exit)
%% Monitor: unidirectional, observer gets notified only
%%
%% ใช้ Link: when processes should die together (gen_server + supervisor)
%% ใช้ Monitor: when you want to know if process died but not die yourself

%% ตัวอย่าง: supervisor-like behavior
supervisor(ChildFun) ->
    process_flag(trap_exit, true),
    restart_child(ChildFun).

restart_child(ChildFun) ->
    Pid = spawn_link(ChildFun),
    io:format("Started child: ~p~n", [Pid]),
    receive
        {'EXIT', Pid, normal} ->
            io:format("Child finished normally~n");
        {'EXIT', Pid, Reason} ->
            io:format("Child crashed (~p), restarting...~n", [Reason]),
            restart_child(ChildFun)
    end.
```

---

## Registered Processes

```erlang
%% ลงทะเบียนชื่อให้ process เพื่อส่ง message โดยไม่ต้องรู้ PID

%% Register
register(my_server, Pid).
register(my_server, self()).

%% Send to named process
my_server ! hello.
whereis(my_server) ! hello.  %% same thing

%% Check if registered
whereis(my_server).     %% = <0.123.0> | undefined

%% All registered names
registered().
%% = [my_server, error_logger, user, ...]

%% Unregister
unregister(my_server).

%% Re-register (must unregister first)
unregister(my_server),
register(my_server, NewPid).

%% ตัวอย่าง: simple named server
start() ->
    Pid = spawn(?MODULE, loop, [#{}]),
    register(key_value_store, Pid),
    {ok, Pid}.

loop(State) ->
    receive
        {put, Key, Value} ->
            loop(State#{Key => Value});
        {get, From, Key} ->
            From ! {ok, maps:get(Key, State, undefined)},
            loop(State);
        dump ->
            io:format("State: ~p~n", [State]),
            loop(State);
        stop ->
            ok
    end.

%% Usage
start().
key_value_store ! {put, name, "Alice"}.
key_value_store ! {get, self(), name}.
receive {ok, V} -> V end.  %% = "Alice"

%% Global registration (ข้าม nodes)
global:register_name(my_global_server, self()).
global:whereis_name(my_global_server).
global:unregister_name(my_global_server).
global:send(my_global_server, hello).
```

---

## Process Dictionary

```erlang
%% Each process has its own private dictionary
%% ใช้สำหรับ thread-local-like storage

%% Put
put(key, value).
put(counter, 0).

%% Get
get(key).    %% = value
get(counter). %% = 0
get(nonexistent). %% = undefined

%% Get all
get().  %% = [{key, value}, {counter, 0}, ...]

%% Erase
erase(key).     %% remove key, return old value
erase().        %% clear all, return old dict

%% ตัวอย่าง: simple counter
increment(Key) ->
    New = get(Key) + 1,
    put(Key, New),
    New.

put(visits, 0).
increment(visits).  %% = 1
increment(visits).  %% = 2
get(visits).        %% = 2

%% ตัวอย่าง: request context
handle_request(Request) ->
    put(request_id, maps:get(id, Request)),
    put(user_id, maps:get(user, Request)),
    try
        process_request(Request)
    after
        erase(request_id),
        erase(user_id)
    end.

get_request_id() ->
    get(request_id).

%% คำเตือน: process dictionary มีข้อเสีย:
%% - ยากต่อการ test (global mutable state)
%% - ไม่ชัดเจนว่า function ใช้อะไร
%% - ใช้เฉพาะเมื่อจำเป็นจริงๆ
%% - ดีกว่าสำหรับ logging context, tracing, etc.
```

---

## ตัวอย่างจริง: Concurrent Counter

```erlang
%% ไฟล์: counter.erl
-module(counter).
-export([
    start/0,
    start/1,
    increment/1,
    decrement/1,
    get/1,
    reset/1,
    stop/1
]).

%% Start counter with initial value
-spec start() -> pid().
start() -> start(0).

-spec start(integer()) -> pid().
start(Initial) ->
    spawn(?MODULE, loop, [Initial]).

%% Public API
increment(Pid) ->
    Pid ! {self(), increment},
    receive {reply, Value} -> Value end.

decrement(Pid) ->
    Pid ! {self(), decrement},
    receive {reply, Value} -> Value end.

get(Pid) ->
    Pid ! {self(), get},
    receive {reply, Value} -> Value end.

reset(Pid) ->
    Pid ! {self(), reset},
    receive {reply, Value} -> Value end.

stop(Pid) ->
    Pid ! stop,
    ok.

%% Internal loop
loop(Count) ->
    receive
        {From, increment} ->
            New = Count + 1,
            From ! {reply, New},
            loop(New);
        {From, decrement} ->
            New = Count - 1,
            From ! {reply, New},
            loop(New);
        {From, get} ->
            From ! {reply, Count},
            loop(Count);
        {From, reset} ->
            From ! {reply, 0},
            loop(0);
        stop ->
            ok
    end.
```

```erlang
%% ตัวอย่างการใช้งาน
C = counter:start(10).
counter:get(C).         %% = 10
counter:increment(C).   %% = 11
counter:increment(C).   %% = 12
counter:decrement(C).   %% = 11
counter:reset(C).       %% = 0
counter:stop(C).

%% Parallel increment test
C2 = counter:start(0).
Pids = [spawn(fun() ->
    [counter:increment(C2) || _ <- lists:seq(1, 100)]
end) || _ <- lists:seq(1, 10)].

timer:sleep(1000).
counter:get(C2).  %% = 1000 (10 * 100, no race condition!)

%% Distributed counter example
start_distributed() ->
    Nodes = nodes(),
    [rpc:call(Node, counter, start, [0]) || Node <- Nodes].
```

---

## สรุป Part 21

| Concept | API | Notes |
|---------|-----|-------|
| Spawn | `spawn(Fun)` | Create process |
| Spawn linked | `spawn_link(Fun)` | Linked lifecycle |
| Spawn monitored | `spawn_monitor(Fun)` | Get notified on death |
| Send | `Pid ! Msg` | Async, non-blocking |
| Receive | `receive ... end` | Block until match |
| Monitor | `erlang:monitor(process, Pid)` | Observe, don't link |
| Link | `link(Pid)` | Share fate |
| Register | `register(Name, Pid)` | Named process |
| Process info | `process_info(Pid)` | Inspect process |

---

*[← Part 20: File I/O](part_20_file_io.md) | [Part 22: Message Passing Advanced →](part_22_message_passing.md)*
