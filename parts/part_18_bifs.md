# Part 18: BIFs — Built-in Functions

## สารบัญ
1. [BIFs คืออะไร](#bifs-คืออะไร)
2. [Type Testing BIFs](#type-testing-bifs)
3. [Conversion BIFs](#conversion-bifs)
4. [Process BIFs](#process-bifs)
5. [System BIFs](#system-bifs)
6. [List BIFs](#list-bifs)
7. [Tuple BIFs](#tuple-bifs)
8. [I/O BIFs](#io-bifs)
9. [Time BIFs](#time-bifs)
10. [ตัวอย่างจริง: System Monitor](#ตัวอย่างจริง-system-monitor)

---

## BIFs คืออะไร

Built-in Functions เป็น functions ที่ implement ใน C ในตัว VM:

```erlang
%% BIFs ส่วนใหญ่ใช้ได้โดยไม่ต้อง import module
%% บาง BIFs อยู่ใน erlang module

%% ตัวอย่าง BIFs ที่รู้จักกันดี
length([1,2,3]).       %% = 3
size({a,b,c}).         %% = 3
hd([1,2,3]).           %% = 1
tl([1,2,3]).           %% = [2,3]
abs(-5).               %% = 5
round(3.7).            %% = 4
trunc(3.7).            %% = 3

%% ทำไม BIFs ถึงสำคัญ:
%% 1. ทำงานเร็วมาก (C code)
%% 2. สามารถใช้ใน guards ได้ (บาง BIFs)
%% 3. Atomic operations บางอย่าง
%% 4. Access to VM internals

%% BIFs สามารถ import ได้
-import(erlang, [list_to_binary/1, binary_to_list/1]).
%% หรือใช้ module prefix
erlang:list_to_binary("hello").
```

---

## Type Testing BIFs

```erlang
%% Type check functions (ใช้ได้ใน guards ด้วย)
is_atom(hello).          %% = true
is_atom("hello").        %% = false
is_atom(true).           %% = true (true/false เป็น atoms)

is_binary(<<"hello">>).  %% = true
is_binary("hello").      %% = false (list, ไม่ใช่ binary)

is_bitstring(<<1:1>>).   %% = true (1 bit)
is_bitstring(<<"hello">>). %% = true (binary = 8-bit aligned bitstring)

is_boolean(true).        %% = true
is_boolean(false).       %% = true
is_boolean(yes).         %% = false

is_float(3.14).          %% = true
is_float(3).             %% = false

is_function(fun() -> ok end).      %% = true
is_function(fun() -> ok end, 0).   %% = true (arity 0)
is_function(fun(X) -> X end, 1).   %% = true
is_function(fun(X) -> X end, 2).   %% = false

is_integer(42).          %% = true
is_integer(42.0).        %% = false

is_list([1,2,3]).        %% = true
is_list([]).             %% = true
is_list({1,2,3}).        %% = false
is_list("hello").        %% = true (string เป็น list)

is_map(#{a => 1}).       %% = true
is_map([{a, 1}]).        %% = false

is_number(42).           %% = true
is_number(3.14).         %% = true
is_number("42").         %% = false

is_pid(self()).          %% = true
is_pid(<0.1.0>).         %% = true

is_port(Port).           %% สำหรับ port PIDs

is_record(#user{}, user).   %% = true (ถ้า define record user)

is_reference(make_ref()).  %% = true

is_tuple({1,2,3}).       %% = true
is_tuple([1,2,3]).       %% = false
```

---

## Conversion BIFs

```erlang
%% Atom conversions
atom_to_list(hello).            %% = "hello"
atom_to_binary(hello).          %% = <<"hello">>
atom_to_binary(hello, utf8).    %% = <<"hello">> (explicit encoding)

list_to_atom("hello").          %% = hello (creates atom in atom table!)
binary_to_atom(<<"hello">>).    %% = hello
binary_to_atom(<<"hello">>, utf8). %% = hello

list_to_existing_atom("erlang"). %% = erlang (ถ้ามีอยู่แล้ว)
binary_to_existing_atom(<<"erlang">>). %% = erlang

%% Integer conversions
integer_to_list(42).            %% = "42"
integer_to_list(255, 16).       %% = "FF"
integer_to_list(255, 2).        %% = "11111111"
integer_to_binary(42).          %% = <<"42">>

list_to_integer("42").          %% = 42
list_to_integer("FF", 16).      %% = 255
binary_to_integer(<<"42">>).    %% = 42
binary_to_integer(<<"FF">>, 16). %% = 255

%% Float conversions
float_to_list(3.14).            %% = "3.14000000000000012434..." (ugly)
float_to_list(3.14, [{decimals, 2}]).  %% = "3.14"
float_to_list(3.14, [compact]). %% = "3.14"
float_to_binary(3.14, [{decimals, 2}]). %% = <<"3.14">>

list_to_float("3.14").          %% = 3.14
binary_to_float(<<"3.14">>).    %% = 3.14

float(42).                      %% = 42.0

%% List/Binary conversions
list_to_binary("hello").        %% = <<"hello">>
list_to_binary([72,101,108]).   %% = <<"Hel">>

binary_to_list(<<"hello">>).    %% = "hello" = [104,101,108,108,111]

iolist_to_binary(["hello", " ", <<"world">>]).  %% = <<"hello world">>

%% Term conversions
term_to_binary({hello, [1,2,3]}).  %% = binary representation
binary_to_term(Bin).               %% = {hello, [1,2,3]}

%% Tuple/List
tuple_to_list({1,2,3}).         %% = [1,2,3]
list_to_tuple([1,2,3]).         %% = {1,2,3}

%% Type coercion
erlang:float(42).               %% = 42.0
erlang:round(3.7).              %% = 4 (integer)
erlang:trunc(3.7).              %% = 3 (integer)
erlang:abs(-5).                 %% = 5
```

---

## Process BIFs

```erlang
%% Current process
self().          %% = <0.123.0> (current PID)

%% Spawn
spawn(fun() -> io:format("Hello~n") end).
spawn(Module, Function, Args).
spawn_link(fun() -> io:format("Hello~n") end).
spawn_monitor(fun() -> do_work() end).  %% Returns {Pid, MonitorRef}

%% Send message
Pid ! Message.
erlang:send(Pid, Message).
erlang:send(Pid, Message, [noconnect, nosuspend]).  %% options

%% Process info
process_info(self()).
%% = [{current_function,...}, {initial_call,...}, {status,...},
%%    {message_queue_len,...}, {heap_size,...}, {memory,...}, ...]

process_info(self(), memory).         %% = {memory, 2688} (bytes)
process_info(self(), message_queue_len).  %% = {message_queue_len, 0}
process_info(self(), status).         %% = {status, running}
process_info(self(), current_function). %% = {current_function, {module, func, arity}}
process_info(self(), dictionary).     %% = {dictionary, [...]}

%% List all processes
erlang:processes().   %% = [<0.1.0>, <0.2.0>, ...]
length(erlang:processes()).  %% number of processes

%% Register/unregister
register(my_server, self()).
whereis(my_server).        %% = <0.123.0>
whereis(nonexistent).      %% = undefined
unregister(my_server).
registered().              %% = [error_logger, user, standard_error, ...]

%% Exit/Kill
exit(Pid, Reason).         %% send exit signal
exit(normal).              %% current process exits normally
exit(shutdown).            %% clean shutdown

erlang:kill(Pid, kill).    %% same as exit(Pid, kill)

%% Monitor
Ref = erlang:monitor(process, Pid).
erlang:demonitor(Ref).
erlang:demonitor(Ref, [flush]).  %% flush pending monitor messages

%% Link
erlang:link(Pid).
erlang:unlink(Pid).

%% Process flag
process_flag(trap_exit, true).  %% receive exit signals as messages
process_flag(priority, high).
process_flag(max_heap_size, #{size => 1000000}).
```

---

## System BIFs

```erlang
%% Node information
node().                      %% = nonode@nohost (or actual node name)
nodes().                     %% = [] (or list of connected nodes)
is_alive().                  %% = false (not distributed) or true

%% System info
erlang:system_info(schedulers).          %% = 8 (or num CPUs)
erlang:system_info(schedulers_online).   %% = 8
erlang:system_info(otp_release).         %% = "26" 
erlang:system_info(version).             %% = "13.2"
erlang:system_info(system_version).      %% = "Erlang/OTP 26..."
erlang:system_info(wordsize).            %% = 8 (64-bit)
erlang:system_info(heap_sizes).          %% = [heap size parameters]
erlang:system_info(atom_count).          %% = 4567 (current atom count)
erlang:system_info(atom_limit).          %% = 1048576 (max atoms)
erlang:system_info(process_count).       %% = 128 (running processes)
erlang:system_info(process_limit).       %% = 262144
erlang:system_info(port_count).          %% = 5
erlang:system_info(port_limit).          %% = 65536
erlang:system_info(ets_count).           %% = 12 (ETS tables)
erlang:system_info(ets_limit).           %% = 34079

%% Memory
erlang:memory().
%% = [{total, 26M}, {processes, 4M}, {system, 22M},
%%    {atom, 401K}, {binary, 187K}, {code, 10M}, {ets, 584K}]

erlang:memory(total).      %% = total bytes
erlang:memory(processes).  %% = process heap bytes

%% Garbage collection
erlang:garbage_collect().            %% GC current process
erlang:garbage_collect(Pid).         %% GC specific process

%% Runtime statistics
erlang:statistics(wall_clock).    %% = {Total, SinceLast} in ms
erlang:statistics(runtime).       %% = {Total, SinceLast} cpu time
erlang:statistics(reductions).    %% = {Total, SinceLast}
erlang:statistics(run_queue).     %% = run queue length
erlang:statistics(garbage_collection). %% = {GCs, WordsReclaimed, 0}

%% Halt
erlang:halt().           %% immediate halt
erlang:halt(0).          %% halt with exit code
erlang:halt("Reason").   %% halt with crash dump
```

---

## List BIFs

```erlang
%% Fundamental list BIFs (ทำงานใน VM)
hd([1,2,3]).            %% = 1 (head)
tl([1,2,3]).            %% = [2,3] (tail)
length([1,2,3,4,5]).    %% = 5

%% หมายเหตุ: length([]) ถ้าส่ง infinite list จะไม่จบ
%% ตัวอย่าง improper list
hd([1|2]).              %% = 1
tl([1|2]).              %% = 2

%% Concatenation
[1,2,3] ++ [4,5,6].     %% = [1,2,3,4,5,6]
[1,2,3,2,1] -- [1,2].   %% = [3,2,1] (remove first occurrence of each)
[1,2,3,2,1] -- [2,2].   %% = [1,3,1]

%% Element access
lists:nth(1, [a,b,c]).  %% = a (1-indexed!)
lists:last([a,b,c]).    %% = c

%% erlang:element/2 สำหรับ tuples
element(1, {a,b,c}).    %% = a (1-indexed)
element(2, {a,b,c}).    %% = b

%% Conversion
list_to_tuple([1,2,3]).  %% = {1,2,3}
tuple_to_list({1,2,3}).  %% = [1,2,3]
```

---

## I/O BIFs

```erlang
%% Basic I/O
io:format("Hello~n").
io:format("~p~n", [Term]).   %% print any term
io:format("~s~n", [String]). %% print string
io:format("~b~n", [Int]).    %% print binary (int in base 2)

%% Format to string (iolist)
io_lib:format("~p", [Term]).  %% = iolist

%% Write to stderr
io:format(standard_error, "Error: ~p~n", [Reason]).

%% Read from user
{ok, Term} = io:read("> ").   %% read Erlang term
Line = io:get_line("> ").     %% read line as string

%% Write term
io:write(Term).      %% output without newline
io:fwrite("~w~n", [Term]).  %% same as io:format

%% erlang:display/1 (debug, bypasses io server)
erlang:display(Term).  %% prints to stdout directly

%% read_term
file:read_file("data.erl").    %% read file
file:write_file("out.txt", Data).

%% Environment variables
os:getenv("HOME").              %% = "/home/user"
os:getenv("PATH").
os:putenv("MY_VAR", "value").
os:unsetenv("MY_VAR").
os:getenv().                    %% = all env vars as strings
```

---

## Time BIFs

```erlang
%% Wall clock time (เวลาจริง)
erlang:timestamp().
%% = {MegaSecs, Secs, MicroSecs} (seconds since epoch)
%% = {1696,123456,789012}

%% Convert to datetime
calendar:now_to_datetime(erlang:timestamp()).
%% = {{2024,1,15},{10,30,45}}

%% OS time
os:timestamp().                    %% = {MegaSecs, Secs, MicroSecs}
os:system_time(second).           %% = Unix timestamp in seconds
os:system_time(millisecond).      %% = Unix timestamp in ms
os:system_time(microsecond).      %% = Unix timestamp in µs
os:system_time(nanosecond).       %% = Unix timestamp in ns

%% Monotonic time (ไม่ย้อนกลับ, สำหรับ measurement)
erlang:monotonic_time().          %% = integer (native units)
erlang:monotonic_time(second).    %% = seconds
erlang:monotonic_time(millisecond). %% = milliseconds

%% Measure elapsed time
Start = erlang:monotonic_time(millisecond).
do_something().
End = erlang:monotonic_time(millisecond).
Elapsed = End - Start.
io:format("Took ~p ms~n", [Elapsed]).

%% timer:tc - measure function execution time
{Time, Result} = timer:tc(fun() -> lists:sort(lists:seq(1, 10000)) end).
%% Time in microseconds

{Time2, Result2} = timer:tc(M, F, Args).
{Time3, Result3} = timer:tc(fun(X) -> X * 2 end, [5]).

%% Datetime conversions
calendar:local_time().   %% = {{2024,1,15},{10,30,45}}
calendar:universal_time().  %% UTC time

calendar:date_to_gregorian_days({2024,1,15}).  %% = Gregorian day number
calendar:time_to_seconds({10,30,0}).  %% = 37800 (seconds since midnight)
calendar:datetime_to_gregorian_seconds({{2024,1,15},{0,0,0}}).

%% Day of week
calendar:day_of_the_week({2024,1,15}).  %% = 1 (Monday=1, ..., Sunday=7)

%% Is leap year
calendar:is_leap_year(2024).  %% = true

%% Date arithmetic
calendar:date_to_gregorian_days({2024,1,15}) + 30.  %% = 30 days later
calendar:gregorian_days_to_date(GDays).  %% back to {Y,M,D}
```

---

## ตัวอย่างจริง: System Monitor

```erlang
%% ไฟล์: system_monitor.erl
-module(system_monitor).
-export([
    snapshot/0,
    top_processes/1,
    memory_stats/0,
    format_snapshot/1
]).

-type process_info() :: #{
    pid := pid(),
    name := atom() | undefined,
    memory := pos_integer(),
    reductions := pos_integer(),
    message_queue_len := non_neg_integer(),
    status := atom(),
    current_function := {module(), atom(), arity()}
}.

%% Take system snapshot
-spec snapshot() -> map().
snapshot() ->
    #{
        timestamp => os:system_time(millisecond),
        node => node(),
        process_count => length(erlang:processes()),
        memory => erlang:memory(),
        statistics => #{
            run_queue => erlang:statistics(run_queue),
            reductions => element(1, erlang:statistics(reductions)),
            gc => erlang:statistics(garbage_collection)
        },
        system => #{
            otp_release => erlang:system_info(otp_release),
            schedulers => erlang:system_info(schedulers_online),
            word_size => erlang:system_info(wordsize),
            atom_count => erlang:system_info(atom_count),
            atom_limit => erlang:system_info(atom_limit)
        }
    }.

%% Get top N processes by memory usage
-spec top_processes(pos_integer()) -> [process_info()].
top_processes(N) ->
    Procs = erlang:processes(),
    Infos = lists:filtermap(
        fun(Pid) ->
            case process_info(Pid, [memory, reductions, message_queue_len,
                                    status, current_function, registered_name]) of
                undefined -> false;
                Info ->
                    {true, #{
                        pid => Pid,
                        name => proplists:get_value(registered_name, Info),
                        memory => proplists:get_value(memory, Info),
                        reductions => proplists:get_value(reductions, Info),
                        message_queue_len => proplists:get_value(message_queue_len, Info),
                        status => proplists:get_value(status, Info),
                        current_function => proplists:get_value(current_function, Info)
                    }}
            end
        end,
        Procs
    ),
    Sorted = lists:sort(
        fun(A, B) -> maps:get(memory, A) > maps:get(memory, B) end,
        Infos
    ),
    lists:sublist(Sorted, N).

%% Get memory statistics
-spec memory_stats() -> map().
memory_stats() ->
    Mem = erlang:memory(),
    Total = proplists:get_value(total, Mem),
    #{
        total => Total,
        total_human => format_bytes(Total),
        processes => format_bytes(proplists:get_value(processes, Mem)),
        system => format_bytes(proplists:get_value(system, Mem)),
        atom => format_bytes(proplists:get_value(atom, Mem)),
        binary => format_bytes(proplists:get_value(binary, Mem)),
        code => format_bytes(proplists:get_value(code, Mem)),
        ets => format_bytes(proplists:get_value(ets, Mem))
    }.

%% Format snapshot as text
-spec format_snapshot(map()) -> binary().
format_snapshot(Snap) ->
    iolist_to_binary([
        io_lib:format("=== System Snapshot ===~n"),
        io_lib:format("Node: ~p~n", [maps:get(node, Snap)]),
        io_lib:format("Processes: ~p~n", [maps:get(process_count, Snap)]),
        io_lib:format("OTP: ~s~n", [maps:get(otp_release, maps:get(system, Snap))]),
        format_memory(maps:get(memory, Snap))
    ]).

format_memory(MemList) ->
    Total = proplists:get_value(total, MemList),
    io_lib:format("Memory: ~s total~n", [format_bytes(Total)]).

format_bytes(Bytes) when Bytes < 1024 ->
    io_lib:format("~B B", [Bytes]);
format_bytes(Bytes) when Bytes < 1024*1024 ->
    io_lib:format("~.1f KB", [Bytes/1024]);
format_bytes(Bytes) when Bytes < 1024*1024*1024 ->
    io_lib:format("~.1f MB", [Bytes/(1024*1024)]);
format_bytes(Bytes) ->
    io_lib:format("~.2f GB", [Bytes/(1024*1024*1024)]).
```

```erlang
%% ใช้งาน
Snap = system_monitor:snapshot().
io:format("~s~n", [system_monitor:format_snapshot(Snap)]).

%% Top 5 processes by memory
Top5 = system_monitor:top_processes(5).
[io:format("~p: ~s~n", [maps:get(name, P), format_bytes(maps:get(memory, P))]) 
 || P <- Top5].

%% Memory breakdown
system_monitor:memory_stats().
```

---

## สรุป Part 18

| Category | BIFs ที่สำคัญ |
|----------|--------------|
| Type test | `is_integer`, `is_binary`, `is_list`, `is_map`, `is_pid` |
| Conversion | `list_to_binary`, `integer_to_list`, `atom_to_binary` |
| Process | `spawn`, `self`, `exit`, `register`, `whereis` |
| System | `erlang:system_info`, `erlang:memory`, `os:system_time` |
| Time | `erlang:monotonic_time`, `timer:tc`, `calendar:local_time` |
| Math | `abs`, `round`, `trunc`, `floor`, `ceil` |
| I/O | `io:format`, `io_lib:format` |

---

*[← Part 17: Comprehensions](part_17_comprehensions.md) | [Part 19: Exception Handling →](part_19_exceptions.md)*
