# Part 27: gen_event — Event Manager

## สารบัญ
1. [gen_event คืออะไร](#gen_event-คืออะไร)
2. [Event Manager และ Handler](#event-manager-และ-handler)
3. [Callbacks](#callbacks)
4. [Error Logger Integration](#error-logger-integration)
5. [ตัวอย่างจริง: Application Event System](#ตัวอย่างจริง-application-event-system)

---

## gen_event คืออะไร

```erlang
%% gen_event = Event Manager + multiple Event Handlers
%% รูปแบบ Observer/Event pattern

%% Architecture:
%%   Event Manager Process
%%     ├── Handler A (callback module)
%%     ├── Handler B (callback module)
%%     └── Handler C (callback module)
%%
%% เมื่อ notify event:
%%   Manager calls Handler:handle_event/2 for each handler

%% ใช้เมื่อ:
%% - Logging (หลาย destination: file, console, remote)
%% - Monitoring (หลาย handler ดู event เดียวกัน)
%% - Plugin systems (add/remove handlers at runtime)
%% - Alarm management (SASL alarm_handler)

%% ไม่ค่อยใช้เมื่อ:
%% - ต้องการ reply จาก event
%% - Performance critical (handler run ใน manager process)
%% - ต้องการ handler แยก process
```

---

## Event Manager และ Handler

```erlang
%% สร้าง Event Manager
{ok, Pid} = gen_event:start().
{ok, Pid} = gen_event:start({local, my_events}).  %% named

%% Start manager ผ่าน supervisor
#{
    id => my_event_mgr,
    start => {gen_event, start_link, [{local, my_event_mgr}]},
    type => worker
}

%% เพิ่ม Handler
ok = gen_event:add_handler(my_events, my_handler, []).
ok = gen_event:add_handler(my_events, my_handler, InitArgs).

%% เพิ่ม Supervised Handler (monitor Handler, restart on crash)
ok = gen_event:add_sup_handler(my_events, my_handler, InitArgs).
%% Manager ส่ง {gen_event_EXIT, Handler, Reason} ไปยัง calling process

%% Notify (async, no reply)
ok = gen_event:notify(my_events, {user_login, "alice"}).
ok = gen_event:notify(my_events, {error, "something wrong"}).

%% Sync notify (wait for all handlers)
ok = gen_event:sync_notify(my_events, Event).

%% Call single handler (sync with reply)
{ok, Result} = gen_event:call(my_events, my_handler, get_stats).

%% Remove Handler
ok = gen_event:delete_handler(my_events, my_handler, stop_reason).

%% List handlers
[my_handler, another_handler] = gen_event:which_handlers(my_events).

%% Swap handlers atomically
gen_event:swap_handler(my_events,
    {old_handler, SwapArg},   %% เรียก old_handler:terminate/2 ก่อน
    {new_handler, InitArg}).  %% เรียก new_handler:init/1 ทีหลัง
```

---

## Callbacks

```erlang
%% -behaviour(gen_event).
%% Required callbacks:

%% init/1: initialize handler state
init(Args) ->
    {ok, InitialState} |
    {ok, InitialState, hibernate} |
    {error, Reason}.

%% handle_event/2: handle async event
handle_event(Event, State) ->
    {ok, NewState} |
    {ok, NewState, hibernate} |
    {swap_handler, Args1, NewState, Handler2, Args2} |  %% swap to another handler
    remove_handler.  %% remove this handler

%% handle_call/2: handle sync call
handle_call(Request, State) ->
    {ok, Reply, NewState} |
    {ok, Reply, NewState, hibernate} |
    {swap_handler, Reply, Args1, NewState, Handler2, Args2} |
    {remove_handler, Reply}.

%% handle_info/2: handle other messages
handle_info(Info, State) ->
    {ok, NewState} |
    ...

%% terminate/2: cleanup
terminate(Reason, State) ->
    ok.  %% ค่า return ส่งไปยัง swap_handler หรือ delete_handler caller

%% code_change/3
code_change(OldVsn, State, Extra) ->
    {ok, NewState}.

%% ตัวอย่าง Handler: Console Logger
-module(console_log_handler).
-behaviour(gen_event).

-export([init/1, handle_event/2, handle_call/2, handle_info/2,
         terminate/2, code_change/3]).

init([Level]) ->
    {ok, #{level => Level, count => 0}}.

handle_event({log, Severity, Msg}, #{level := MinLevel, count := N} = State) ->
    case severity_value(Severity) >= severity_value(MinLevel) of
        true ->
            io:format("[~s] ~s~n", [Severity, Msg]),
            {ok, State#{count => N + 1}};
        false ->
            {ok, State}
    end;
handle_event(_, State) ->
    {ok, State}.

handle_call(get_count, #{count := N} = State) ->
    {ok, N, State};
handle_call(reset_count, State) ->
    {ok, ok, State#{count => 0}}.

handle_info(_, State) -> {ok, State}.

terminate(_Reason, State) ->
    io:format("Console handler stopping. Total logged: ~p~n",
              [maps:get(count, State, 0)]).

code_change(_, State, _) -> {ok, State}.

severity_value(debug) -> 0;
severity_value(info) -> 1;
severity_value(warning) -> 2;
severity_value(error) -> 3;
severity_value(critical) -> 4.
```

---

## Error Logger Integration

```erlang
%% error_logger เป็น gen_event manager ที่ OTP สร้างให้
%% OTP 21+: logger เป็นตัวหลัก (error_logger ยังทำงานอยู่)

%% เพิ่ม custom handler ไปยัง error_logger
error_logger:add_report_handler(my_log_handler, []).

%% SASL alarm_handler
alarm_handler:set_alarm({disk_full, "/var"}).
alarm_handler:clear_alarm({disk_full, "/var"}).
alarm_handler:get_alarms().

%% Custom alarm handler
-module(my_alarm_handler).
-behaviour(gen_event).

init([]) ->
    {ok, #{alarms => #{}}}.

handle_event({set_alarm, {AlarmId, AlarmDesc}}, #{alarms := As} = State) ->
    io:format("ALARM SET: ~p - ~p~n", [AlarmId, AlarmDesc]),
    notify_ops_team(AlarmId, AlarmDesc),
    {ok, State#{alarms => As#{AlarmId => AlarmDesc}}};

handle_event({clear_alarm, AlarmId}, #{alarms := As} = State) ->
    io:format("ALARM CLEARED: ~p~n", [AlarmId]),
    {ok, State#{alarms => maps:remove(AlarmId, As)}};

handle_event(_, State) ->
    {ok, State}.

handle_call(get_alarms, #{alarms := As} = State) ->
    {ok, maps:to_list(As), State}.

%% ลงทะเบียน
gen_event:swap_handler(alarm_handler,
    {alarm_handler, swap},          %% ปิด default handler
    {my_alarm_handler, []}).        %% เปิด custom handler
```

---

## ตัวอย่างจริง: Application Event System

```erlang
%% ไฟล์: app_events.erl
-module(app_events).
-behaviour(gen_event).

%% API for emitting events
-export([start_link/0,
         subscribe/1,    %% subscribe(HandlerMod)
         subscribe/2,    %% subscribe(HandlerMod, Args)
         unsubscribe/1,
         emit/1,
         emit/2,
         handlers/0]).

%% gen_event callbacks
-export([init/1, handle_event/2, handle_call/2, handle_info/2,
         terminate/2, code_change/3]).

-define(MGR, ?MODULE).

start_link() ->
    gen_event:start_link({local, ?MGR}).

subscribe(HandlerMod) ->
    subscribe(HandlerMod, []).

subscribe(HandlerMod, Args) ->
    gen_event:add_sup_handler(?MGR, HandlerMod, Args).

unsubscribe(HandlerMod) ->
    gen_event:delete_handler(?MGR, HandlerMod, unsubscribe).

emit(Event) ->
    emit(Event, #{}).

emit(Event, Metadata) ->
    Envelope = #{
        event => Event,
        metadata => Metadata,
        timestamp => os:system_time(millisecond),
        node => node()
    },
    gen_event:notify(?MGR, Envelope).

handlers() ->
    gen_event:which_handlers(?MGR).

%% app_events itself can be a handler too (for logging)
init([]) ->
    {ok, #{logged => 0}}.

handle_event(#{event := Event, timestamp := T}, #{logged := N} = State) ->
    io:format("[~p] Event: ~p~n", [T, Event]),
    {ok, State#{logged => N + 1}};
handle_event(_, State) ->
    {ok, State}.

handle_call(stats, #{logged := N} = State) ->
    {ok, #{logged => N}, State}.

handle_info(_, State) -> {ok, State}.
terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.


%% ไฟล์: metrics_handler.erl
-module(metrics_handler).
-behaviour(gen_event).

-export([init/1, handle_event/2, handle_call/2, handle_info/2,
         terminate/2, code_change/3]).

init([]) ->
    {ok, #{counters => #{}, timings => #{}}}.

handle_event(#{event := {request, completed, #{duration := D, path := Path}}},
             #{counters := C, timings := T} = State) ->
    NewC = maps:update_with(Path, fun(N) -> N + 1 end, 1, C),
    NewT = maps:update_with(
        Path,
        fun(Ds) -> [D | Ds] end,
        [D],
        T
    ),
    {ok, State#{counters => NewC, timings => NewT}};

handle_event(#{event := {user, action, #{action := Action, user := User}}},
             #{counters := C} = State) ->
    Key = {user_action, Action},
    NewC = maps:update_with(Key, fun(N) -> N + 1 end, 1, C),
    _ = User,  %% could log user-specific metrics
    {ok, State#{counters => NewC}};

handle_event(_, State) ->
    {ok, State}.

handle_call(report, #{counters := C, timings := T} = State) ->
    AvgTimings = maps:map(
        fun(_, Ds) ->
            lists:sum(Ds) / length(Ds)
        end,
        T
    ),
    Report = #{
        request_counts => C,
        avg_response_ms => AvgTimings
    },
    {ok, Report, State};

handle_call(reset, State) ->
    {ok, ok, State#{counters => #{}, timings => #{}}}.

handle_info(_, State) -> {ok, State}.
terminate(_, _) -> ok.
code_change(_, S, _) -> {ok, S}.


%% ไฟล์: audit_handler.erl
-module(audit_handler).
-behaviour(gen_event).

-export([init/1, handle_event/2, handle_call/2, handle_info/2,
         terminate/2, code_change/3]).

init([{file, Filename}]) ->
    case file:open(Filename, [append, binary]) of
        {ok, Fd} -> {ok, #{fd => Fd, entries => 0}};
        {error, R} -> {error, R}
    end.

handle_event(#{event := {user, _, #{user := _}} = Event,
               timestamp := T,
               metadata := Meta},
             #{fd := Fd, entries := N} = State) ->
    Line = io_lib:format("~p ~p ~p~n", [T, Event, Meta]),
    file:write(Fd, Line),
    {ok, State#{entries => N + 1}};

handle_event(_, State) ->
    {ok, State}.

handle_call(flush, #{fd := Fd} = State) ->
    file:sync(Fd),
    {ok, ok, State}.

handle_info(_, State) -> {ok, State}.

terminate(_, #{fd := Fd}) ->
    file:close(Fd).

code_change(_, S, _) -> {ok, S}.


%% ใช้งาน
app_events:start_link().

%% Subscribe handlers
app_events:subscribe(app_events).     %% console logging
app_events:subscribe(metrics_handler).
app_events:subscribe(audit_handler, [{file, "audit.log"}]).

%% Emit events
app_events:emit({user, login, #{user => "alice"}}).
app_events:emit({request, completed, #{
    path => "/api/users",
    duration => 45,
    status => 200
}}).
app_events:emit({user, action, #{user => "alice", action => purchase}}).

%% Check metrics
gen_event:call(app_events, metrics_handler, report).
%% #{request_counts => #{"/api/users" => 1},
%%   avg_response_ms => #{"/api/users" => 45.0}}

%% Flush audit log
gen_event:call(app_events, audit_handler, flush).
```

---

## สรุป Part 27

| Function | Description |
|----------|-------------|
| `gen_event:start_link/1` | Start named event manager |
| `gen_event:add_handler/3` | Add event handler |
| `gen_event:add_sup_handler/3` | Add supervised handler |
| `gen_event:notify/2` | Async event |
| `gen_event:sync_notify/2` | Sync event (wait all handlers) |
| `gen_event:call/3` | Sync call to one handler |
| `gen_event:delete_handler/3` | Remove handler |
| `gen_event:which_handlers/1` | List handlers |

| Callback | Called when |
|----------|-------------|
| `init/1` | Handler added |
| `handle_event/2` | notify/sync_notify |
| `handle_call/2` | call/2 |
| `handle_info/2` | Other messages |
| `terminate/2` | Handler removed |

---

*[← Part 26: gen_statem](part_26_gen_statem.md) | [Part 28: Application Behaviour →](part_28_application.md)*
