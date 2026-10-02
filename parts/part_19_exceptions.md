# Part 19: Exception Handling — try, catch, throw, error

## สารบัญ
1. [Exception Types](#exception-types)
2. [throw/1](#throw1)
3. [error/1 and error/2](#error1-and-error2)
4. [exit/1](#exit1)
5. [try-catch Expression](#try-catch-expression)
6. [catch Expression](#catch-expression)
7. [Error Handling Patterns](#error-handling-patterns)
8. [Let It Crash Philosophy](#let-it-crash-philosophy)
9. [ตัวอย่างจริง: Safe Operations Library](#ตัวอย่างจริง-safe-operations-library)

---

## Exception Types

Erlang มี exception 3 ประเภท:

```erlang
%% 1. error - Runtime errors (เช่น badmatch, badarg, badarith)
%% Class: error
%% เกิดจาก: BIF failures, pattern match failures, explicit error/1,2

%% 2. exit - Process termination signals
%% Class: exit
%% เกิดจาก: exit/1 explicit call หรือ process exit

%% 3. throw - Programmer-defined non-local jumps
%% Class: throw
%% เกิดจาก: throw/1 explicit call

%% รูปแบบ exception:
%% {Class, Reason, Stacktrace}

%% Class = error | exit | throw
%% Reason = any term
%% Stacktrace = list of {Module, Function, Arity, [{file, ...}, {line, ...}]}

%% Common error reasons:
%% badmatch   - pattern match failure
%% badarg     - bad argument to BIF
%% badarith   - arithmetic error
%% function_clause - no matching function clause
%% undef      - undefined function
%% noproc     - process not found
%% system_limit - system resource limit exceeded
```

---

## throw/1

```erlang
%% throw สำหรับ non-local exits ที่คาดไว้
%% ใช้สำหรับ control flow ไม่ใช่ errors จริงๆ

%% ตัวอย่าง: early return จาก nested code
find_first_positive(Lists) ->
    try
        lists:foreach(
            fun(List) ->
                lists:foreach(
                    fun(X) when X > 0 -> throw({found, X});
                       (_) -> ok
                    end,
                    List
                )
            end,
            Lists
        ),
        not_found
    catch
        throw:{found, Value} -> {ok, Value}
    end.

find_first_positive([[-1, -2], [-3, 5, 6], [7]]).
%% = {ok, 5}

%% ตัวอย่าง: abort deep computation
search_tree(Tree, Target) ->
    try
        search_impl(Tree, Target),
        not_found
    catch
        throw:found -> found
    end.

search_impl({node, Val, _, _}, Val) ->
    throw(found);
search_impl({node, _, Left, Right}, Target) ->
    search_impl(Left, Target),
    search_impl(Right, Target);
search_impl(nil, _) ->
    ok.

%% throw กับ complex reason
validate_all(Items) ->
    try
        lists:foreach(
            fun(Item) ->
                case validate_item(Item) of
                    ok -> ok;
                    {error, Reason} ->
                        throw({validation_failed, Item, Reason})
                end
            end,
            Items
        ),
        ok
    catch
        throw:{validation_failed, Item, Reason} ->
            {error, {invalid_item, Item, Reason}}
    end.
```

---

## error/1 and error/2

```erlang
%% error/1 - raise error exception
error(badarg).
error({invalid_input, "must be positive"}).
error({my_error, #{code => 404, message => "not found"}}).

%% error/2 - with stack trace info (OTP 24+)
error(badarg, [{?MODULE, my_function, [Arg], []}]).

%% error/3 - with error info map (OTP 24.1+)
error(badarg, [Arg], [{error_info, #{
    module => ?MODULE,
    cause => #{1 => "must be positive integer"}
}}]).

%% Common error patterns
safe_divide(_, 0) ->
    error(badarg, "Divisor cannot be zero");
safe_divide(A, B) when not is_number(A); not is_number(B) ->
    error(badarg, "Arguments must be numbers");
safe_divide(A, B) ->
    A / B.

%% Raise structured error
-define(RAISE(Code, Msg), error({Code, Msg})).

validate_age(Age) when not is_integer(Age) ->
    ?RAISE(invalid_type, <<"Age must be an integer">>);
validate_age(Age) when Age < 0 ->
    ?RAISE(out_of_range, <<"Age cannot be negative">>);
validate_age(Age) when Age > 150 ->
    ?RAISE(out_of_range, <<"Age seems unreasonably large">>);
validate_age(Age) ->
    Age.

%% Runtime errors (automatic)
1 = 2.          %% error: {badmatch, 2}
hd([]).         %% error: badarg
1 / 0.          %% error: badarith (actually allowed: 1/0 = infinity in some)
undefined:f().  %% error: undef
X.              %% error: {unbound, 'X'} (unbound variable)
```

---

## exit/1

```erlang
%% exit/1 - terminate current process or send exit signal
exit(normal).       %% normal exit (no crash)
exit(shutdown).     %% clean shutdown
exit(killed).       %% forcefully killed

exit(Pid, Reason).  %% send exit signal to another process

%% exit กับ supervisor
%% child process ที่ exit(normal) -> supervisor ไม่ restart
%% child process ที่ exit(shutdown) -> supervisor ไม่ restart
%% child process ที่ exit(AnyOther) -> supervisor restart!

%% ตัวอย่าง
stop_server(ServerPid) ->
    ServerPid ! stop.
    %% หรือ
    exit(ServerPid, shutdown).

%% Process ที่ trap_exit:
process_flag(trap_exit, true).
%% exit signal จะ convert เป็น {'EXIT', Pid, Reason} message
receive
    {'EXIT', _Pid, normal}   -> ok;
    {'EXIT', _Pid, shutdown} -> ok;
    {'EXIT', Pid, Reason}    -> handle_crash(Pid, Reason)
end.

%% erlang:exit vs throw
%% exit: terminate the process
%% throw: non-local jump (catchable)
%% error: indicate bug/runtime error
```

---

## try-catch Expression

```erlang
%% รูปแบบพื้นฐาน
try
    Expression
catch
    Class:Pattern -> Handler
end.

%% ตัวอย่าง catch all
try
    some_risky_operation()
catch
    _:_Reason -> {error, failed}  %% catch any class, any reason
end.

%% Catch specific classes
try
    operation()
catch
    error:badarg -> {error, bad_argument};
    error:{badmatch, _} -> {error, match_failed};
    exit:shutdown -> ok;
    throw:{my_error, Reason} -> {error, Reason}
end.

%% With 'of' clause (pattern match on success)
try
    compute()
of
    {ok, Value} -> process(Value);
    undefined -> default_value()
catch
    error:Reason -> {error, Reason}
end.

%% With 'after' clause (always executed)
try
    open_resource(),
    do_work()
catch
    _:Reason -> {error, Reason}
after
    close_resource()  %% always runs!
end.

%% Full form
try
    Expression
of
    Pattern1 -> Expr1;
    Pattern2 -> Expr2
catch
    Class1:Pattern3 -> Expr3;
    Class2:Pattern4 when Guard -> Expr4
after
    AlwaysRun
end.

%% Capturing stacktrace
try
    dangerous()
catch
    Class:Reason:Stacktrace ->
        io:format("Error: ~p:~p~n~p~n", [Class, Reason, Stacktrace]),
        {error, {Class, Reason}}
end.

%% ตัวอย่าง: safe call
safe_call(Fun) ->
    try Fun() of
        Result -> {ok, Result}
    catch
        Class:Reason:Stack ->
            {error, {Class, Reason, Stack}}
    end.

safe_call(fun() -> 1 / 0 end).
%% = {error, {error, badarith, [...]}}

safe_call(fun() -> 42 end).
%% = {ok, 42}
```

---

## catch Expression

```erlang
%% catch - older, simpler form
%% catch Expression

%% ถ้า normal: ให้ค่าของ expression
catch 1 + 1.    %% = 2
catch [1,2,3].  %% = [1,2,3]

%% ถ้า throw: ให้ค่าที่ throw
catch throw(hello).  %% = hello

%% ถ้า error: ให้ {'EXIT', {Reason, Stacktrace}}
catch error(badarg).
%% = {'EXIT', {badarg, [{erlang,error,[badarg,...]}]}}

catch 1/0.
%% = {'EXIT', {badarith, [...]}}

%% ถ้า exit: ให้ {'EXIT', Reason}
catch exit(shutdown).
%% = {'EXIT', shutdown}

%% catch ไม่ capture class หรือ stacktrace -> ใช้ try-catch ดีกว่า

%% ตัวอย่าง: legacy code pattern
result = catch some_function(),
case result of
    {'EXIT', Reason} -> {error, Reason};
    Value -> {ok, Value}
end.

%% คำแนะนำ: ใช้ try-catch แทน catch เสมอในโค้ดใหม่
```

---

## Error Handling Patterns

```erlang
%% Pattern 1: {ok, Value} | {error, Reason}
open_file(Path) ->
    case file:open(Path, [read]) of
        {ok, Device} -> {ok, Device};
        {error, Reason} -> {error, {file_open_failed, Path, Reason}}
    end.

%% Pattern 2: Chaining with case
process(Input) ->
    case validate(Input) of
        {ok, Valid} ->
            case transform(Valid) of
                {ok, Result} -> {ok, Result};
                {error, _} = E -> E
            end;
        {error, _} = E ->
            E
    end.

%% Pattern 3: Railway-oriented (function chain)
-spec and_then({ok, A} | {error, E}, fun((A) -> {ok, B} | {error, E})) ->
    {ok, B} | {error, E}.
and_then({ok, Value}, F) -> F(Value);
and_then({error, _} = E, _F) -> E.

%% Usage
and_then(validate(Input),
fun(Valid) -> and_then(transform(Valid),
fun(Transformed) -> save(Transformed)
end)
end).

%% Pattern 4: ok_pipe (apply functions while ok)
ok_pipe(Value, []) -> {ok, Value};
ok_pipe(Value, [F|Funs]) ->
    case F(Value) of
        {ok, Next} -> ok_pipe(Next, Funs);
        {error, _} = E -> E;
        Next -> ok_pipe(Next, Funs)
    end.

ok_pipe(Input, [
    fun validate/1,
    fun transform/1,
    fun save/1
]).

%% Pattern 5: try-catch for third-party code
safe_json_decode(Json) ->
    try
        {ok, jsx:decode(Json, [return_maps])}
    catch
        error:badarg -> {error, invalid_json};
        _:Reason -> {error, {decode_failed, Reason}}
    end.

%% Pattern 6: Error normalization
normalize_error({error, enoent}) ->
    {error, not_found};
normalize_error({error, eacces}) ->
    {error, permission_denied};
normalize_error({error, _} = E) ->
    E;
normalize_error(Other) ->
    {error, {unexpected, Other}}.
```

---

## Let It Crash Philosophy

```erlang
%% หลักการสำคัญของ Erlang:
%% "Let it crash" - ปล่อยให้ process crash และ supervisor จัดการ

%% BAD: defensive programming ที่มากเกินไป
get_user_bad(Id) ->
    try
        case db:find_user(Id) of
            undefined -> default_user();
            User ->
                try
                    validate_user(User)
                catch
                    _:_ -> default_user()
                end
        end
    catch
        _:_ -> default_user()
    end.

%% GOOD: let it crash แล้วให้ supervisor restart
get_user_good(Id) ->
    case db:find_user(Id) of
        undefined -> {error, not_found};
        User -> validate_user(User)
    end.
%% ถ้า db:find_user crash -> process crash -> supervisor restart

%% ควร catch exception เฉพาะเมื่อ:
%% 1. คาดว่าจะเกิดและ handle ได้
%% 2. เรียก third-party code ที่ไม่แน่ใจ
%% 3. เป็น boundary ของ external system

%% ไม่ควร catch exception เมื่อ:
%% 1. ซ่อน bugs จาก developer
%% 2. ป้องกัน supervisor จาก restart ที่ควรเกิด
%% 3. Handle errors ที่ไม่รู้จัก

%% ตัวอย่าง: OTP gen_server
%% ถ้า handle_call crash -> supervisor restart server
%% ไม่ต้อง wrap ทุก function ด้วย try-catch

handle_call({get_user, Id}, _From, State) ->
    %% ถ้า crash ที่นี่ -> gen_server crash -> supervisor restart
    User = maps:get(Id, State#state.users),  %% may throw if not found
    {reply, User, State}.

%% เพิ่ม error handling เฉพาะ business logic
handle_call({get_user, Id}, _From, State) ->
    case maps:find(Id, State#state.users) of
        {ok, User} -> {reply, {ok, User}, State};
        error -> {reply, {error, not_found}, State}
    end.
```

---

## ตัวอย่างจริง: Safe Operations Library

```erlang
%% ไฟล์: safe_ops.erl
-module(safe_ops).
-export([
    safe/1,
    safe/2,
    attempt/2,
    retry/3,
    with_timeout/2,
    ensure/2
]).

%% Wrap function call safely
-spec safe(fun()) -> {ok, term()} | {error, {term(), term(), list()}}.
safe(Fun) ->
    try
        {ok, Fun()}
    catch
        Class:Reason:Stack ->
            {error, {Class, Reason, Stack}}
    end.

%% Wrap with custom error handler
-spec safe(fun(), fun()) -> term().
safe(Fun, OnError) ->
    try Fun()
    catch
        Class:Reason ->
            OnError({Class, Reason})
    end.

%% Attempt N times
-spec attempt(fun(), pos_integer()) -> {ok, term()} | {error, term()}.
attempt(Fun, MaxTries) ->
    attempt(Fun, MaxTries, undefined).

attempt(_Fun, 0, LastError) ->
    {error, {max_attempts_reached, LastError}};
attempt(Fun, TriesLeft, _) ->
    case safe(Fun) of
        {ok, _} = Ok -> Ok;
        {error, {_Class, Reason, _Stack}} ->
            attempt(Fun, TriesLeft - 1, Reason)
    end.

%% Retry with exponential backoff
-spec retry(fun(), pos_integer(), pos_integer()) -> {ok, term()} | {error, term()}.
retry(Fun, MaxRetries, BaseDelayMs) ->
    retry(Fun, MaxRetries, BaseDelayMs, 0).

retry(_Fun, MaxRetries, _, Attempt) when Attempt >= MaxRetries ->
    {error, max_retries_exceeded};
retry(Fun, MaxRetries, BaseDelay, Attempt) ->
    case safe(Fun) of
        {ok, _} = Ok ->
            Ok;
        {error, {_Class, Reason, _Stack}} ->
            Delay = BaseDelay * (1 bsl Attempt),  %% exponential backoff
            timer:sleep(Delay),
            io:format("Retry ~p/~p after ~p ms: ~p~n",
                      [Attempt+1, MaxRetries, Delay, Reason]),
            retry(Fun, MaxRetries, BaseDelay, Attempt + 1)
    end.

%% Execute with timeout
-spec with_timeout(fun(), pos_integer()) -> {ok, term()} | {error, timeout}.
with_timeout(Fun, TimeoutMs) ->
    Parent = self(),
    Ref = make_ref(),
    Pid = spawn(fun() ->
        try
            Result = Fun(),
            Parent ! {Ref, {ok, Result}}
        catch
            _:Reason ->
                Parent ! {Ref, {error, Reason}}
        end
    end),
    receive
        {Ref, Result} -> Result
    after TimeoutMs ->
        exit(Pid, kill),
        {error, timeout}
    end.

%% Ensure cleanup runs
-spec ensure(fun(), fun()) -> term().
ensure(Fun, Cleanup) ->
    try
        Result = Fun(),
        Cleanup(),
        Result
    catch
        Class:Reason ->
            Cleanup(),
            erlang:raise(Class, Reason, erlang:get_stacktrace())
    end.
```

```erlang
%% ตัวอย่างใช้งาน

%% Safe call
safe_ops:safe(fun() -> 1 / 0 end).
%% = {error, {error, badarith, [...]}}

safe_ops:safe(fun() -> 42 end).
%% = {ok, 42}

%% Retry flaky operation
safe_ops:retry(
    fun() ->
        case rand:uniform(3) of
            1 -> ok;
            _ -> error(flaky)
        end
    end,
    3,
    100  %% 100ms base delay
).

%% With timeout
safe_ops:with_timeout(
    fun() ->
        timer:sleep(200),
        result
    end,
    100  %% 100ms timeout
).
%% = {error, timeout}

%% Ensure cleanup
safe_ops:ensure(
    fun() ->
        io:format("Doing work~n"),
        error(something_went_wrong)
    end,
    fun() ->
        io:format("Cleanup!~n")  %% runs even on error
    end
).
```

---

## สรุป Part 19

| Exception | เกิดจาก | Catch ด้วย |
|-----------|---------|-----------|
| `error` | BIF failures, `error/1`, badmatch | `catch error:Reason` |
| `exit` | `exit/1`, process death | `catch exit:Reason` |
| `throw` | `throw/1` | `catch throw:Value` |

| Pattern | ใช้เมื่อ |
|---------|---------|
| `{ok,V} \| {error,R}` | Function ที่อาจ fail ได้ตามปกติ |
| `try-catch` | Code ที่ might throw, third-party |
| Let it crash | Unexpected errors, let supervisor handle |
| `after` | Cleanup ที่ต้องทำเสมอ |

---

*[← Part 18: BIFs](part_18_bifs.md) | [Part 20: File I/O →](part_20_file_io.md)*
