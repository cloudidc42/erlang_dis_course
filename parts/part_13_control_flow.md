# Part 13: Control Flow — if, case, cond, receive

## สารบัญ
1. [if Expression](#if-expression)
2. [case Expression](#case-expression)
3. [cond Expression](#cond-expression)
4. [receive Expression](#receive-expression)
5. [Pattern Matching เป็น Control Flow](#pattern-matching-เป็น-control-flow)
6. [Nested Control Flow](#nested-control-flow)
7. [ตัวอย่างจริง: HTTP Router](#ตัวอย่างจริง-http-router)

---

## if Expression

```erlang
%% if ใน Erlang ต่างจาก if ในภาษาอื่น
%% ทุก branch ต้องเป็น Guard expressions เท่านั้น!
%% ไม่ใช่ boolean expressions ทั่วไป

X = 5.

%% รูปแบบพื้นฐาน
Result = if
    X > 10 -> big;
    X > 5  -> medium;
    X > 0  -> small;
    true   -> non_positive  %% "true" เป็น catch-all (เหมือน else)
end.
%% Result = small

%% หมายเหตุ: ถ้าไม่มี branch ที่ match จะ throw exception!
%% ดังนั้น if เกือบทุกครั้งต้องมี "true ->" เป็น fallback

%% if กับหลาย conditions
classify_number(N) ->
    if
        N < 0     -> negative;
        N =:= 0   -> zero;
        N < 10    -> small_positive;
        N < 100   -> medium_positive;
        true      -> large_positive
    end.

classify_number(-5).   %% = negative
classify_number(0).    %% = zero
classify_number(7).    %% = small_positive
classify_number(50).   %% = medium_positive
classify_number(200).  %% = large_positive

%% if กับ compound conditions
check_range(X, Min, Max) ->
    if
        X >= Min, X =< Max -> in_range;  %% , = AND
        X < Min            -> too_low;
        true               -> too_high
    end.

check_range(5, 1, 10).   %% = in_range
check_range(-1, 1, 10).  %% = too_low
check_range(15, 1, 10).  %% = too_high

%% Guards ที่ใช้ใน if ได้
%% is_integer, is_float, is_atom, is_list, is_binary, is_map
%% is_pid, is_tuple, is_function
%% ==, /=, =:=, =/=, <, >, =<, >=
%% +, -, *, div, rem, band, bor, bxor, bnot, bsl, bsr
%% abs, element, float, hd, tl, length, node, round, self
%% size, tl, trunc, tuple_size, map_size, byte_size, bit_size

validate(X) ->
    if
        is_integer(X), X > 0 -> {ok, X};
        is_integer(X)        -> {error, must_be_positive};
        is_float(X)          -> {error, must_be_integer};
        true                 -> {error, invalid_type}
    end.

%% คำแนะนำ: case มักดีกว่า if ในหลายกรณี
%% if ใช้เมื่อ: conditions เป็น math/type checks ง่ายๆ
```

---

## case Expression

```erlang
%% case เป็น control flow หลักใน Erlang
%% ใช้ Pattern Matching แบบเต็ม

%% รูปแบบพื้นฐาน
Value = {ok, 42}.

Result = case Value of
    {ok, N}    -> N;
    {error, E} -> throw({error, E})
end.
%% Result = 42

%% case กับ literal patterns
describe_number(N) ->
    case N of
        0 -> "zero";
        1 -> "one";
        2 -> "two";
        _ -> "other"  %% _ = wildcard/catch-all
    end.

%% case กับ Guards
classify(N) ->
    case N of
        N when N < 0 -> negative;
        0            -> zero;
        N when N < 100 -> small;
        _            -> large
    end.

%% case กับ nested patterns
handle_response(Response) ->
    case Response of
        {ok, #{status := 200, body := Body}} ->
            {success, Body};
        {ok, #{status := 404}} ->
            not_found;
        {ok, #{status := Status}} when Status >= 500 ->
            {server_error, Status};
        {error, Reason} ->
            {error, Reason}
    end.

%% case กับ list patterns
process_list(List) ->
    case List of
        [] ->
            empty;
        [Single] ->
            {singleton, Single};
        [First, Second | _Rest] ->
            {starts_with, First, Second}
    end.

process_list([]).          %% = empty
process_list([1]).         %% = {singleton, 1}
process_list([1,2,3,4]).   %% = {starts_with, 1, 2}

%% case กับ binary patterns
parse_command(Cmd) ->
    case Cmd of
        <<"quit">>        -> quit;
        <<"help">>        -> help;
        <<"get ", Key/binary>> -> {get, Key};
        <<"set ", Rest/binary>> ->
            case binary:split(Rest, <<" ">>) of
                [Key, Val] -> {set, Key, Val};
                _          -> {error, bad_set_format}
            end;
        _ ->
            {error, unknown_command}
    end.

parse_command(<<"quit">>).           %% = quit
parse_command(<<"get username">>).   %% = {get, <<"username">>}
parse_command(<<"set key value">>).  %% = {set, <<"key">>, <<"value">>}

%% case กับ map patterns
handle_event(Event) ->
    case Event of
        #{type := click, x := X, y := Y} ->
            {clicked_at, X, Y};
        #{type := keydown, key := Key} ->
            {key_pressed, Key};
        #{type := Type} ->
            {unknown_event, Type}
    end.
```

---

## cond Expression

```erlang
%% cond เป็น syntactic sugar สำหรับ if-else chain
%% มาจาก Lisp, เพิ่มใน OTP 26

%% รูปแบบ
X = 75.
Grade = cond
    X >= 90 -> "A";
    X >= 80 -> "B";
    X >= 70 -> "C";
    X >= 60 -> "D";
    true    -> "F"
end.
%% Grade = "C"

%% cond กับ complex conditions
describe_time(Hour, Minute) ->
    cond
        Hour =:= 0, Minute =:= 0 -> "midnight";
        Hour =:= 12, Minute =:= 0 -> "noon";
        Hour < 12 -> io_lib:format("~p:~2..0pam", [Hour, Minute]);
        Hour =:= 12 -> io_lib:format("~p:~2..0ppm", [Hour, Minute]);
        true -> io_lib:format("~p:~2..0ppm", [Hour - 12, Minute])
    end.

%% เปรียบเทียบ: if vs cond vs case
%% ทั้ง 3 รูปแบบทำสิ่งเดียวกัน

classify_if(N) ->
    if
        N < 0 -> negative;
        N =:= 0 -> zero;
        true -> positive
    end.

classify_cond(N) ->
    cond
        N < 0 -> negative;
        N =:= 0 -> zero;
        true -> positive
    end.

classify_case(N) ->
    case N of
        0 -> zero;
        N when N < 0 -> negative;
        _ -> positive
    end.

%% กฎ: ใช้ case เมื่อ match patterns, ใช้ if/cond เมื่อ conditions
```

---

## receive Expression

```erlang
%% receive ใช้รับ messages จาก process mailbox
%% เป็น control flow สำคัญใน concurrent programming

%% รูปแบบพื้นฐาน
loop() ->
    receive
        stop ->
            ok;
        {print, Msg} ->
            io:format("~p~n", [Msg]),
            loop();
        Other ->
            io:format("Unknown: ~p~n", [Other]),
            loop()
    end.

%% receive กับ timeout
receive_with_timeout() ->
    receive
        {data, Value} ->
            {ok, Value}
    after 5000 ->
        {error, timeout}
    end.

%% after 0 -> check mailbox แล้วทำงานต่อทันที
flush_mailbox() ->
    receive
        _ -> flush_mailbox()
    after 0 ->
        ok
    end.

%% receive กับ pattern ซับซ้อน
handle_message() ->
    receive
        %% Match specific pattern
        {call, From, Ref, Request} ->
            Response = process(Request),
            From ! {reply, Ref, Response};
        %% Match with guard
        {event, Type, Data} when is_atom(Type) ->
            handle_event(Type, Data);
        %% Match any
        Msg ->
            io:format("Unexpected: ~p~n", [Msg])
    after 1000 ->
        idle
    end.

%% selective receive - ข้าม messages ที่ไม่ match
wait_for_reply(Ref) ->
    receive
        {reply, Ref, Value} ->
            Value
        %% messages อื่นๆ ที่ไม่ match Ref จะถูก skip
        %% และรอ messages ใหม่
    after 5000 ->
        {error, timeout}
    end.
```

---

## Pattern Matching เป็น Control Flow

```erlang
%% Pattern matching ที่ function level เป็น control flow ที่ดีที่สุดใน Erlang

%% แทน if/case ด้วย function clauses
describe(0) -> "zero";
describe(N) when N < 0 -> "negative";
describe(_) -> "positive".

%% Router pattern
handle(get, "/") -> home_page();
handle(get, "/about") -> about_page();
handle(get, "/users") -> list_users();
handle(post, "/users") -> create_user();
handle(_, _) -> not_found().

%% Error handling pattern
process({ok, Value}) ->
    transform(Value);
process({error, Reason}) ->
    log_error(Reason),
    {error, Reason}.

%% List processing pattern
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

%% State machine pattern
state(idle, start) -> running;
state(running, pause) -> paused;
state(running, stop) -> stopped;
state(paused, resume) -> running;
state(paused, stop) -> stopped;
state(State, _) -> State.  %% ignore invalid transitions
```

---

## Nested Control Flow

```erlang
%% Nested case
classify_pair({A, B}) ->
    case A of
        ok ->
            case B of
                {value, V} when is_integer(V) -> {ok, integer, V};
                {value, V} when is_binary(V)  -> {ok, binary, V};
                {value, _}                    -> {ok, other};
                _                             -> {ok, no_value}
            end;
        error ->
            {error, A}
    end.

%% ดีกว่าด้วย flat patterns
classify_pair2({ok, {value, V}}) when is_integer(V) -> {ok, integer, V};
classify_pair2({ok, {value, V}}) when is_binary(V)  -> {ok, binary, V};
classify_pair2({ok, {value, _}})                    -> {ok, other};
classify_pair2({ok, _})                             -> {ok, no_value};
classify_pair2({error, E})                          -> {error, E}.

%% case กับ let-like binding
process_data(Input) ->
    case validate(Input) of
        {ok, Valid} ->
            case transform(Valid) of
                {ok, Result} ->
                    case save(Result) of
                        ok -> {ok, Result};
                        {error, E} -> {error, save_failed, E}
                    end;
                {error, E} ->
                    {error, transform_failed, E}
            end;
        {error, E} ->
            {error, validation_failed, E}
    end.

%% ดีกว่าด้วย do-notation style (ไม่ใช่ native แต่ implement ได้)
%% หรือใช้ railway-oriented programming
do([]) -> {ok, undefined};
do([Step | Rest]) ->
    case Step of
        {ok, Value} -> do_with(Value, Rest);
        Error -> Error
    end.

%% Pattern: pipeline of failable steps
process_pipeline(Input) ->
    Steps = [
        fun validate/1,
        fun transform/1,
        fun enrich/1
    ],
    lists:foldl(
        fun(_, {error, _} = Err) -> Err;
           (F, {ok, Val}) -> F(Val);
           (F, Val) -> F(Val)
        end,
        {ok, Input},
        Steps
    ).
```

---

## ตัวอย่างจริง: HTTP Router

```erlang
%% ไฟล์: router.erl
-module(router).
-export([route/1, dispatch/2]).

-type method() :: get | post | put | delete | patch.
-type path() :: binary().
-type request() :: #{
    method := method(),
    path := path(),
    headers := #{binary() => binary()},
    body := binary(),
    params := #{binary() => binary()}
}.
-type response() :: {integer(), #{}, binary()}.

%% Main route function
-spec route(request()) -> response().
route(#{method := Method, path := Path} = Request) ->
    case match_route(Method, Path) of
        {ok, Handler, Params} ->
            try
                NewRequest = Request#{params => Params},
                Handler(NewRequest)
            catch
                error:function_clause -> not_found_response();
                Class:Reason ->
                    error_response(500, Class, Reason)
            end;
        not_found ->
            not_found_response()
    end.

%% Route matching
match_route(get, <<"/">>) ->
    {ok, fun handle_home/1, #{}};
match_route(get, <<"/health">>) ->
    {ok, fun handle_health/1, #{}};
match_route(get, <<"/users">>) ->
    {ok, fun handle_list_users/1, #{}};
match_route(post, <<"/users">>) ->
    {ok, fun handle_create_user/1, #{}};
match_route(get, <<"/users/", IdBin/binary>>) ->
    case catch binary_to_integer(IdBin) of
        Id when is_integer(Id), Id > 0 ->
            {ok, fun handle_get_user/1, #{<<"id">> => Id}};
        _ ->
            not_found
    end;
match_route(put, <<"/users/", IdBin/binary>>) ->
    case catch binary_to_integer(IdBin) of
        Id when is_integer(Id) ->
            {ok, fun handle_update_user/1, #{<<"id">> => Id}};
        _ ->
            not_found
    end;
match_route(delete, <<"/users/", IdBin/binary>>) ->
    case catch binary_to_integer(IdBin) of
        Id when is_integer(Id) ->
            {ok, fun handle_delete_user/1, #{<<"id">> => Id}};
        _ ->
            not_found
    end;
match_route(_, _) ->
    not_found.

%% Handlers
handle_home(_Request) ->
    json_response(200, #{message => <<"Welcome to Erlang API">>}).

handle_health(_Request) ->
    json_response(200, #{status => <<"ok">>, uptime => erlang:monotonic_time()}).

handle_list_users(_Request) ->
    Users = get_all_users(),
    json_response(200, #{users => Users}).

handle_create_user(#{body := Body}) ->
    case decode_json(Body) of
        {ok, #{<<"name">> := Name, <<"email">> := Email}} ->
            case create_user(Name, Email) of
                {ok, User} ->
                    json_response(201, User);
                {error, duplicate_email} ->
                    json_response(409, #{error => <<"Email already exists">>});
                {error, Reason} ->
                    json_response(400, #{error => format_error(Reason)})
            end;
        {ok, _} ->
            json_response(400, #{error => <<"Missing required fields: name, email">>});
        {error, _} ->
            json_response(400, #{error => <<"Invalid JSON">>})
    end.

handle_get_user(#{params := #{<<"id">> := Id}}) ->
    case get_user(Id) of
        {ok, User} ->
            json_response(200, User);
        not_found ->
            not_found_response()
    end.

handle_update_user(#{params := #{<<"id">> := Id}, body := Body}) ->
    case decode_json(Body) of
        {ok, Updates} ->
            case update_user(Id, Updates) of
                {ok, User} -> json_response(200, User);
                not_found  -> not_found_response();
                {error, R} -> json_response(400, #{error => format_error(R)})
            end;
        _ ->
            json_response(400, #{error => <<"Invalid JSON">>})
    end.

handle_delete_user(#{params := #{<<"id">> := Id}}) ->
    case delete_user(Id) of
        ok        -> json_response(204, #{});
        not_found -> not_found_response()
    end.

%% Dispatch based on content type
dispatch(Request, <<"application/json">>) ->
    route(Request);
dispatch(_Request, ContentType) ->
    {415, #{}, iolist_to_binary(["Unsupported media type: ", ContentType])}.

%% Helpers
json_response(Status, Data) ->
    {Status, #{<<"content-type">> => <<"application/json">>}, encode_json(Data)}.

not_found_response() ->
    json_response(404, #{error => <<"Not found">>}).

error_response(Status, Class, Reason) ->
    json_response(Status, #{
        error => <<"Internal server error">>,
        detail => iolist_to_binary(io_lib:format("~p: ~p", [Class, Reason]))
    }).

format_error(Reason) when is_atom(Reason) ->
    atom_to_binary(Reason);
format_error(Reason) ->
    iolist_to_binary(io_lib:format("~p", [Reason])).

%% Placeholder functions (implement กับ database จริง)
get_all_users() -> [].
get_user(_Id) -> not_found.
create_user(_Name, _Email) -> {ok, #{}}.
update_user(_Id, _Updates) -> not_found.
delete_user(_Id) -> not_found.
decode_json(<<>>) -> {error, empty};
decode_json(JSON) ->
    try {ok, jsx:decode(JSON, [return_maps])}
    catch _:_ -> {error, invalid_json}
    end.
encode_json(Data) ->
    jsx:encode(Data).
```

---

## สรุป Part 13

| Expression | ใช้เมื่อ | หมายเหตุ |
|-----------|---------|---------|
| `if` | Conditions เป็น guards อย่างง่าย | ต้องมี `true ->` เป็น fallback |
| `case` | Pattern matching + guards | ใช้บ่อยที่สุด |
| `cond` | If-else chain ยาว (OTP 26+) | เหมือน if แต่อ่านง่ายกว่า |
| `receive` | รับ messages | เฉพาะ concurrent programming |
| Function clauses | Pattern matching ที่ซับซ้อน | Idiomatic Erlang |

---

*[← Part 12: Arithmetic](part_12_arithmetic.md) | [Part 14: Guards →](part_14_guards.md)*
