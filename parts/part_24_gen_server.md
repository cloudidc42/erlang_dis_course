# Part 24: gen_server — Generic Server Behaviour

## สารบัญ
1. [gen_server คืออะไร](#gen_server-คืออะไร)
2. [Callbacks ทั้งหมด](#callbacks-ทั้งหมด)
3. [Starting gen_server](#starting-gen_server)
4. [handle_call (Synchronous)](#handle_call-synchronous)
5. [handle_cast (Asynchronous)](#handle_cast-asynchronous)
6. [handle_info (Other Messages)](#handle_info-other-messages)
7. [State Management](#state-management)
8. [Timeout และ Hibernation](#timeout-และ-hibernation)
9. [ตัวอย่างจริง: Key-Value Store พร้อม TTL](#ตัวอย่างจริง-key-value-store-พร้อม-ttl)

---

## gen_server คืออะไร

```erlang
%% gen_server = Generic Server
%% เป็น OTP behaviour ที่ใช้บ่อยที่สุด
%% จัดการ: lifecycle, error handling, debugging, code reloading
%% คุณต้องทำแค่: implement callback functions

%% แทนที่จะเขียน custom loop:
my_server() ->
    receive
        {call, From, Ref, Req} ->
            {Reply, NewState} = handle(Req, State),
            From ! {reply, Ref, Reply},
            my_server();
        ...
    end.

%% ใช้ gen_server แทน:
handle_call(Req, _From, State) ->
    {reply, handle(Req), State}.

%% gen_server ให้:
%% - Standard message protocol
%% - Timeout handling
%% - Tracing and debugging
%% - sys module integration (suspend, trace, get_state)
%% - Hibernate support
%% - Graceful termination
%% - Code upgrade (hot reload)
```

---

## Callbacks ทั้งหมด

```erlang
%% -behaviour(gen_server).
%% Required callbacks:

%% 1. init/1
init(Args) ->
    {ok, InitialState} |
    {ok, InitialState, Timeout} |  %% ส่ง timeout message หลัง Timeout ms
    {ok, InitialState, hibernate} | %% หยุดพัก (ประหยัด memory)
    {ok, InitialState, {continue, Continue}} |  %% handle_continue callback
    {stop, Reason} |  %% ไม่ start
    ignore.

%% 2. handle_call/3 (synchronous request)
handle_call(Request, From, State) ->
    {reply, Reply, NewState} |
    {reply, Reply, NewState, Timeout} |
    {reply, Reply, NewState, hibernate} |
    {reply, Reply, NewState, {continue, Continue}} |
    {noreply, NewState} |  %% จะ reply ทีหลังด้วย gen_server:reply/2
    {noreply, NewState, Timeout} |
    {noreply, NewState, hibernate} |
    {stop, Reason, Reply, NewState} |  %% reply แล้ว stop
    {stop, Reason, NewState}.

%% 3. handle_cast/2 (asynchronous request)
handle_cast(Request, State) ->
    {noreply, NewState} |
    {noreply, NewState, Timeout} |
    {noreply, NewState, hibernate} |
    {noreply, NewState, {continue, Continue}} |
    {stop, Reason, NewState}.

%% 4. handle_info/2 (other messages: erlang ! message)
handle_info(Info, State) ->
    {noreply, NewState} |
    ... (same as handle_cast)

%% 5. terminate/2 (cleanup)
terminate(Reason, State) ->
    ok.  %% return value ignored

%% 6. code_change/3 (hot code upgrade)
code_change(OldVsn, State, Extra) ->
    {ok, NewState} |
    {error, Reason}.

%% 7. handle_continue/2 (OTP 21+)
handle_continue(Continue, State) ->
    {noreply, NewState} |
    ...
```

---

## Starting gen_server

```erlang
%% start_link/3,4 - start and link to calling process
gen_server:start_link(Module, Args, Options).
gen_server:start_link(ServerName, Module, Args, Options).

%% ServerName:
%% {local, Name}  - register locally as Name
%% {global, Name} - register globally (across nodes)
%% {via, Module, Name} - register via custom registry (e.g., gproc)

%% Options:
%% {debug, Dbg}    - sys module debug options
%% {timeout, T}    - init/1 must finish in T ms
%% {hibernate_after, T} - hibernate after T ms of inactivity
%% {spawn_opt, SpawnOpts} - process spawn options

%% ตัวอย่าง
-module(my_server).
-behaviour(gen_server).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% start without link (ใน scripts/tests)
start() ->
    gen_server:start({local, ?MODULE}, ?MODULE, [], []).

%% Start ด้วย args
start_link(Config) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Config, []).

%% start_child ผ่าน supervisor
supervisor:start_child(my_sup, [Arg1, Arg2]).

%% stop gen_server gracefully
gen_server:stop(ServerRef).            %% reason = normal
gen_server:stop(ServerRef, Reason, Timeout).

%% คำแนะนำ: ทุก gen_server ควร expose start_link/0 หรือ /N
%% เพื่อให้ supervisor สามารถ start ได้
```

---

## handle_call (Synchronous)

```erlang
%% gen_server:call/2,3 - blocking call with reply

%% API function
get_count() ->
    gen_server:call(?MODULE, get_count).

get_count() ->
    gen_server:call(?MODULE, get_count, 5000).  %% explicit timeout

%% Callback
handle_call(get_count, _From, #{count := N} = State) ->
    {reply, N, State};

%% Pattern: From = {CallerPid, Ref}
handle_call(Request, {CallerPid, _Ref} = From, State) ->
    io:format("Request from ~p: ~p~n", [CallerPid, Request]),
    {reply, ok, State}.

%% Pattern: delayed reply
handle_call(heavy_computation, From, State) ->
    spawn(fun() ->
        Result = do_heavy_work(),
        gen_server:reply(From, Result)
    end),
    {noreply, State}.  %% จะ reply จาก spawned process

%% Pattern: call that may stop server
handle_call(shutdown, _From, State) ->
    {stop, normal, ok, State}.
%% ส่ง ok เป็น reply แล้ว stop

%% Pattern: call with state update
handle_call({update, Key, Value}, _From, #{data := Data} = State) ->
    NewData = Data#{Key => Value},
    {reply, ok, State#{data => NewData}};

handle_call({get, Key}, _From, #{data := Data} = State) ->
    Value = maps:get(Key, Data, undefined),
    {reply, Value, State}.
```

---

## handle_cast (Asynchronous)

```erlang
%% gen_server:cast/2 - non-blocking, no reply

%% API
increment() ->
    gen_server:cast(?MODULE, increment).

log_event(Event) ->
    gen_server:cast(?MODULE, {log, Event}).

%% Callback
handle_cast(increment, #{count := N} = State) ->
    {noreply, State#{count => N + 1}};

handle_cast({log, Event}, #{logs := Logs} = State) ->
    Timestamp = os:system_time(millisecond),
    Entry = #{event => Event, at => Timestamp},
    {noreply, State#{logs => [Entry | Logs]}};

handle_cast({broadcast, Msg}, #{subscribers := Subs} = State) ->
    lists:foreach(fun(Sub) -> Sub ! {message, Msg} end, Subs),
    {noreply, State};

handle_cast(Unknown, State) ->
    io:format("Unknown cast: ~p~n", [Unknown]),
    {noreply, State}.

%% Pattern: cast that needs to stop
handle_cast({stop, Reason}, State) ->
    {stop, Reason, State}.
```

---

## handle_info (Other Messages)

```erlang
%% handle_info จัดการ messages อื่นๆ ที่ไม่ใช่ call/cast

%% Periodic work
init([]) ->
    erlang:send_after(1000, self(), tick),
    {ok, #{count => 0}}.

handle_info(tick, #{count := N} = State) ->
    io:format("Tick ~p~n", [N]),
    erlang:send_after(1000, self(), tick),  %% schedule next tick
    {noreply, State#{count => N + 1}};

%% Process exit (when trap_exit)
init([]) ->
    process_flag(trap_exit, true),
    {ok, #{}}.

handle_info({'EXIT', Pid, Reason}, State) ->
    io:format("Linked process ~p died: ~p~n", [Pid, Reason]),
    {noreply, State};

%% Monitor DOWN
handle_info({'DOWN', Ref, process, Pid, Reason}, #{monitors := Refs} = State) ->
    case maps:find(Ref, Refs) of
        {ok, WorkerName} ->
            io:format("Worker ~p (~p) died: ~p~n", [WorkerName, Pid, Reason]),
            {noreply, State#{monitors => maps:remove(Ref, Refs)}};
        error ->
            {noreply, State}
    end;

%% TCP/gen_tcp messages
handle_info({tcp, Socket, Data}, State) ->
    handle_tcp_data(Socket, Data, State);

handle_info({tcp_closed, Socket}, State) ->
    handle_disconnect(Socket, State);

%% Timer messages
handle_info({timeout, _Ref, cleanup}, State) ->
    NewState = cleanup_expired(State),
    erlang:start_timer(60000, self(), cleanup),
    {noreply, NewState};

%% Catch-all (important for debugging)
handle_info(Unknown, State) ->
    io:format("Unexpected info: ~p~n", [Unknown]),
    {noreply, State}.
```

---

## State Management

```erlang
%% State คือ term ใดๆ ก็ได้ (map, record, tuple, list)
%% Map เป็น choice ที่แนะนำสำหรับโค้ดใหม่

%% Map state
-type state() :: #{
    count := non_neg_integer(),
    users := #{binary() => map()},
    config := map()
}.

init([Config]) ->
    {ok, #{
        count => 0,
        users => #{},
        config => Config
    }}.

%% Record state (เร็วกว่า, type-safe)
-record(state, {
    count = 0 :: non_neg_integer(),
    users = #{} :: map(),
    config = #{} :: map()
}).

init([Config]) ->
    {ok, #state{config = Config}}.

handle_call(get_count, _From, #state{count = N} = S) ->
    {reply, N, S};

handle_cast(increment, #state{count = N} = S) ->
    {noreply, S#state{count = N + 1}}.

%% State transitions
update_state(State, #{action := add_user, user := User}) ->
    Id = maps:get(id, User),
    State#{users => (maps:get(users, State))#{Id => User}};

update_state(State, #{action := remove_user, id := Id}) ->
    State#{users => maps:remove(Id, maps:get(users, State))};

update_state(State, _) ->
    State.

%% Read state (sys module)
sys:get_state(?MODULE).
sys:replace_state(?MODULE, fun(S) -> S#{debug_mode => true} end).
```

---

## Timeout และ Hibernation

```erlang
%% Timeout: gen_server ส่ง timeout message ถึงตัวเอง
%% ใช้สำหรับ periodic cleanup, expiry, etc.

%% Return timeout from handle_call/cast
handle_call(start_timer, _From, State) ->
    {reply, ok, State, 5000}.  %% timeout ใน 5 วินาที

handle_info(timeout, State) ->
    io:format("Timeout! Doing cleanup~n"),
    {noreply, State}.

%% ตัวอย่าง: idle connection cleanup
handle_info(timeout, #{connections := Conns} = State) ->
    Now = erlang:monotonic_time(second),
    Active = maps:filter(
        fun(_, #{last_seen := T}) -> Now - T < 300 end,  %% 5 min timeout
        Conns
    ),
    {noreply, State#{connections => Active}, 60000}.  %% check again in 1 min

%% Hibernation: เมื่อ process ไม่ได้ทำงาน ลด memory
handle_call(Request, _From, State) ->
    Result = handle(Request, State),
    {reply, Result, State, hibernate}.  %% hibernate after replying

%% hibernate_after option (OTP 22+)
gen_server:start_link(?MODULE, [], [{hibernate_after, 1000}]).
%% hibernate หลังจากไม่มี message 1 วินาที

%% Hibernation ลดขนาด heap แต่ต้องใช้เวลาตื่น
%% เหมาะสำหรับ: connection pools, idle workers

%% handle_continue: ทำงานหลัง init หรือ handler
init([]) ->
    {ok, #{}, {continue, load_initial_data}}.

handle_continue(load_initial_data, State) ->
    Data = db:load_all(),
    {noreply, State#{data => Data}}.

%% ดีกว่าทำใน init เพราะ: init ไม่บล็อก supervisor
```

---

## ตัวอย่างจริง: Key-Value Store พร้อม TTL

```erlang
%% ไฟล์: kv_store.erl
-module(kv_store).
-behaviour(gen_server).

%% API
-export([
    start_link/0, start_link/1,
    put/2, put/3,
    get/1,
    delete/1,
    exists/1,
    keys/0,
    size/0,
    flush/0,
    stats/0
]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

-define(DEFAULT_TTL, infinity).
-define(CLEANUP_INTERVAL, 60000).  %% 60 seconds

-record(entry, {
    value :: term(),
    inserted_at :: integer(),
    expires_at :: integer() | infinity
}).

-record(state, {
    store = #{} :: #{term() => #entry{}},
    hits = 0 :: non_neg_integer(),
    misses = 0 :: non_neg_integer(),
    evictions = 0 :: non_neg_integer()
}).

%% ====== API ======

start_link() ->
    start_link([]).

start_link(Opts) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Opts, []).

-spec put(term(), term()) -> ok.
put(Key, Value) ->
    put(Key, Value, ?DEFAULT_TTL).

-spec put(term(), term(), pos_integer() | infinity) -> ok.
put(Key, Value, TTL) ->
    gen_server:cast(?MODULE, {put, Key, Value, TTL}).

-spec get(term()) -> {ok, term()} | miss.
get(Key) ->
    gen_server:call(?MODULE, {get, Key}).

-spec delete(term()) -> ok.
delete(Key) ->
    gen_server:cast(?MODULE, {delete, Key}).

-spec exists(term()) -> boolean().
exists(Key) ->
    gen_server:call(?MODULE, {exists, Key}).

-spec keys() -> [term()].
keys() ->
    gen_server:call(?MODULE, keys).

-spec size() -> non_neg_integer().
size() ->
    gen_server:call(?MODULE, size).

-spec flush() -> ok.
flush() ->
    gen_server:cast(?MODULE, flush).

-spec stats() -> map().
stats() ->
    gen_server:call(?MODULE, stats).

%% ====== Callbacks ======

init(_Opts) ->
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {ok, #state{}}.

handle_call({get, Key}, _From, #state{store = Store} = State) ->
    Now = erlang:monotonic_time(second),
    case maps:get(Key, Store, undefined) of
        #entry{value = V, expires_at = Exp} when Exp =:= infinity; Exp > Now ->
            {reply, {ok, V}, State#state{hits = State#state.hits + 1}};
        #entry{} ->
            %% Expired
            NewStore = maps:remove(Key, Store),
            {reply, miss,
             State#state{
                 store = NewStore,
                 misses = State#state.misses + 1,
                 evictions = State#state.evictions + 1
             }};
        undefined ->
            {reply, miss, State#state{misses = State#state.misses + 1}}
    end;

handle_call({exists, Key}, _From, #state{store = Store} = State) ->
    Now = erlang:monotonic_time(second),
    Exists = case maps:get(Key, Store, undefined) of
        #entry{expires_at = infinity} -> true;
        #entry{expires_at = Exp} -> Exp > Now;
        undefined -> false
    end,
    {reply, Exists, State};

handle_call(keys, _From, #state{store = Store} = State) ->
    Now = erlang:monotonic_time(second),
    ValidKeys = [K || {K, #entry{expires_at = Exp}} <- maps:to_list(Store),
                      Exp =:= infinity orelse Exp > Now],
    {reply, ValidKeys, State};

handle_call(size, _From, #state{store = Store} = State) ->
    Now = erlang:monotonic_time(second),
    Count = maps:fold(
        fun(_, #entry{expires_at = Exp}, Acc) ->
            case Exp =:= infinity orelse Exp > Now of
                true -> Acc + 1;
                false -> Acc
            end
        end,
        0,
        Store
    ),
    {reply, Count, State};

handle_call(stats, _From, #state{store = Store, hits = H, misses = M, evictions = E} = State) ->
    Stats = #{
        total_keys => map_size(Store),
        hits => H,
        misses => M,
        evictions => E,
        hit_rate => case H + M of
            0 -> 0.0;
            Total -> H / Total
        end
    },
    {reply, Stats, State}.

handle_cast({put, Key, Value, TTL}, #state{store = Store} = State) ->
    Now = erlang:monotonic_time(second),
    ExpiresAt = case TTL of
        infinity -> infinity;
        Seconds -> Now + Seconds
    end,
    Entry = #entry{
        value = Value,
        inserted_at = Now,
        expires_at = ExpiresAt
    },
    {noreply, State#state{store = Store#{Key => Entry}}};

handle_cast({delete, Key}, #state{store = Store} = State) ->
    {noreply, State#state{store = maps:remove(Key, Store)}};

handle_cast(flush, State) ->
    {noreply, State#state{store = #{}}}.

handle_info(cleanup, #state{store = Store, evictions = E} = State) ->
    Now = erlang:monotonic_time(second),
    {Clean, Evicted} = maps:fold(
        fun(K, #entry{expires_at = Exp} = Entry, {CleanAcc, EvictCount}) ->
            case Exp =:= infinity orelse Exp > Now of
                true -> {CleanAcc#{K => Entry}, EvictCount};
                false -> {CleanAcc, EvictCount + 1}
            end
        end,
        {#{}, 0},
        Store
    ),
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {noreply, State#state{store = Clean, evictions = E + Evicted}};

handle_info(Unknown, State) ->
    io:format("kv_store unexpected: ~p~n", [Unknown]),
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

code_change(_OldVsn, State, _Extra) ->
    {ok, State}.
```

```erlang
%% ใช้งาน
kv_store:start_link().

%% Basic operations
kv_store:put(name, "Alice").
kv_store:get(name).         %% = {ok, "Alice"}
kv_store:exists(name).      %% = true
kv_store:keys().            %% = [name]
kv_store:size().            %% = 1

%% With TTL (3 seconds)
kv_store:put(session, <<"token123">>, 3).
kv_store:get(session).      %% = {ok, <<"token123">>}
timer:sleep(4000),
kv_store:get(session).      %% = miss (expired)

%% Stats
kv_store:stats().
%% = #{total_keys => 1, hits => 2, misses => 1, hit_rate => 0.667}

%% Flush
kv_store:flush().
kv_store:size().  %% = 0
```

---

## สรุป Part 24

| Callback | Returns | ใช้เมื่อ |
|----------|---------|---------|
| `init/1` | `{ok, State}` | Initialize state |
| `handle_call/3` | `{reply, R, S}` | Sync request |
| `handle_cast/2` | `{noreply, S}` | Async request |
| `handle_info/2` | `{noreply, S}` | Other messages |
| `terminate/2` | `ok` | Cleanup |
| `code_change/3` | `{ok, S}` | Hot reload |

| Return value | Effect |
|-------------|--------|
| `{reply, R, S}` | Send reply, continue with state S |
| `{noreply, S}` | No reply, continue with state S |
| `{stop, R, S}` | Terminate with reason R |
| `{noreply, S, T}` | No reply, timeout T ms |
| `{noreply, S, hibernate}` | No reply, enter hibernation |

---

*[← Part 23: OTP Overview](part_23_otp_overview.md) | [Part 25: Supervisor →](part_25_supervisor.md)*
