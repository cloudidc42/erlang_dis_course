# Part 54: Metaprogramming — Parse Transforms และ Macros

## สารบัญ
1. [Macros และ -define](#macros-และ--define)
2. [Parse Transforms](#parse-transforms)
3. [Abstract Format](#abstract-format)
4. [Code Generation](#code-generation)
5. [Compile-time Checks](#compile-time-checks)
6. [ตัวอย่างจริง: Auto-generating REST Handlers](#ตัวอย่างจริง-auto-generating-rest-handlers)

---

## Macros และ -define

```erlang
%% Basic macros
-define(PI, 3.14159265).
-define(MAX_RETRIES, 3).
-define(TIMEOUT, 5000).

%% Function-like macros
-define(LOG(Level, Msg), logger:Level(Msg, #{module => ?MODULE, line => ?LINE})).
-define(LOG(Level, Msg, Meta), logger:Level(Msg, maps:merge(#{module => ?MODULE}, Meta))).

%% Predefined macros
?MODULE      %% current module name
?MODULE_STRING  %% module name as string
?FILE        %% current filename
?LINE        %% current line number
?FUNCTION_NAME  %% current function name (OTP 19+)
?FUNCTION_ARITY %% current function arity
?OTP_RELEASE    %% OTP release number

%% Conditional compilation
-ifdef(DEBUG).
-define(DEBUG_LOG(Msg), io:format("[DEBUG] ~p~n", [Msg])).
-else.
-define(DEBUG_LOG(_), ok).
-endif.

%% Record macros (common pattern)
-define(RECORD_TO_MAP(Record, Fields),
    maps:from_list(lists:zip(Fields, tuple_to_list(Record)))).

-define(MAP_TO_RECORD(RecName, Map, Fields),
    list_to_tuple([RecName | [maps:get(F, Map) || F <- Fields]])).

%% Assert macros (for tests)
-define(ASSERT(Cond),
    case (Cond) of
        true -> ok;
        false ->
            error({assert_failed, ??Cond, ?FILE, ?LINE})
    end).

%% Stringification: ??Macro converts macro to string
-define(SHOW(X), io:format("~s = ~p~n", [??X, X])).

%% String macros
-define(STR(X), iolist_to_binary(io_lib:format("~p", [X]))).

%% Measurements
-define(TIMED(Name, Expr),
    (fun() ->
        _Start = erlang:monotonic_time(microsecond),
        _Result = (Expr),
        _End = erlang:monotonic_time(microsecond),
        io:format("~s took ~p µs~n", [Name, _End - _Start]),
        _Result
    end)()).
```

---

## Parse Transforms

```erlang
%% Parse transform: modify AST at compile time
%% รัน before compilation, แก้ไข abstract syntax tree

%% ตัวอย่าง: auto-add logging to all functions

-module(logging_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    [transform_form(F) || F <- Forms].

transform_form({function, Line, Name, Arity, Clauses}) ->
    {function, Line, Name, Arity, [transform_clause(Name, C) || C <- Clauses]};
transform_form(Other) ->
    Other.

transform_clause(FuncName, {clause, Line, Patterns, Guards, Body}) ->
    %% Add log statement at beginning of each function
    LogCall = {call, Line,
        {remote, Line, {atom, Line, logger}, {atom, Line, debug}},
        [{atom, Line, FuncName}]},
    {clause, Line, Patterns, Guards, [LogCall | Body]}.

%% Using parse transform in your module:
%% -compile({parse_transform, logging_transform}).

%% More common: Exometer, Prometheus, lager use parse transforms
%% lager: -compile({parse_transform, lager_transform}).
%% transforms ?INFO(...) into efficient log calls with metadata

%% Write parse transform:
-module(my_transform).
-export([parse_transform/2]).

parse_transform(Forms, Options) ->
    %% Options from -compile({parse_transform, Options})
    Module = proplists:get_value(module, Options),
    io:format("Transforming module: ~p~n", [Module]),
    walk_forms(Forms, []).

walk_forms([], Acc) -> lists:reverse(Acc);
walk_forms([F | Rest], Acc) ->
    Transformed = transform(F),
    walk_forms(Rest, [Transformed | Acc]).

transform(Form) ->
    %% Use erl_syntax for easier AST manipulation
    case erl_syntax:type(Form) of
        function ->
            transform_function(Form);
        _ ->
            Form
    end.

transform_function(Form) ->
    Name = erl_syntax:atom_value(erl_syntax:function_name(Form)),
    io:format("  Function: ~p/~p~n", [Name, erl_syntax:function_arity(Form)]),
    Form.  %% return unchanged in this example
```

---

## Abstract Format

```erlang
%% Erlang Abstract Format: AST representation

%% เปิดดู AST ของ code
parse_to_ast(Code) ->
    {ok, Tokens, _} = erl_scan:string(Code),
    {ok, Form} = erl_parse:parse_form(Tokens),
    Form.

example() ->
    Code = "f(X) -> X + 1.",
    AST = parse_to_ast(Code),
    io:format("~p~n", [AST]).

%% Output:
%% {function, 1, f, 1,
%%  [{clause, 1,
%%    [{var, 1, 'X'}],
%%    [],
%%    [{op, 1, '+', {var, 1, 'X'}, {integer, 1, 1}}]}]}

%% Common AST node types:
%% {integer, Line, Value}
%% {float, Line, Value}
%% {atom, Line, Value}
%% {string, Line, Value}
%% {var, Line, Name}
%% {tuple, Line, [Elements]}
%% {list, Line, [Elements]}
%% {op, Line, Op, Left, Right}
%% {call, Line, {atom, Line, Fun}, [Args]}
%% {remote, Line, Module, Function}
%% {match, Line, Pattern, Expr}
%% {function, Line, Name, Arity, Clauses}
%% {clause, Line, Patterns, Guards, Body}

%% erl_syntax: higher-level API
-include_lib("syntax_tools/include/merl.hrl").

build_if_expr(Cond, Then, Else) ->
    erl_syntax:if_expr([
        erl_syntax:clause([Cond], none, [Then]),
        erl_syntax:clause([erl_syntax:atom(true)], none, [Else])
    ]).

%% merl: template-based AST generation
fun_with_merl(Name, Body) ->
    ?Q("_@Name() -> _@Body.").  %% merl quote macro
```

---

## Code Generation

```erlang
%% Generate modules at compile time or runtime

%% Compile-time: generate repetitive code
-define(MAKE_GETTER(Field),
    Field(#{Field := V}) -> V).

%% Runtime: generate and compile new modules
generate_module(ModName, Functions) ->
    %% Build module forms
    ModForm = {attribute, 1, module, ModName},
    ExportForms = {attribute, 1, export, [{Name, Arity} || {Name, Arity, _} <- Functions]},
    FuncForms = [make_function(Name, Arity, Body) || {Name, Arity, Body} <- Functions],
    
    Forms = [ModForm, ExportForms | FuncForms],
    
    %% Compile and load
    case compile:forms(Forms, []) of
        {ok, ModName, Binary} ->
            code:load_binary(ModName, atom_to_list(ModName) ++ ".erl", Binary),
            {ok, ModName};
        {error, Errors, _} ->
            {error, Errors}
    end.

make_function(Name, _Arity, BodyFun) ->
    %% Build function AST
    {function, 1, Name, 1, [
        {clause, 1, [{var, 1, 'X'}], [], [body_to_ast(BodyFun)]}
    ]}.

body_to_ast(Body) when is_function(Body) ->
    %% Convert fun result to AST
    {atom, 1, ok}.  %% simplified

%% Generate accessor functions from records
generate_record_accessors(RecordName, Fields) ->
    ModName = list_to_atom(atom_to_list(RecordName) ++ "_api"),
    Functions = lists:flatmap(fun({Field, Index}) ->
        [
            %% Getter
            {Field, 1, fun(Rec) -> element(Index + 1, Rec) end},
            %% Setter
            {list_to_atom("set_" ++ atom_to_list(Field)), 2,
             fun(Rec, Val) -> setelement(Index + 1, Rec, Val) end}
        ]
    end, lists:zip(Fields, lists:seq(1, length(Fields)))),
    generate_module(ModName, Functions).

%% Dynamic dispatch table
build_dispatch(Handlers) ->
    %% Generate if/case chain at runtime
    maps:fold(fun(Pattern, Handler, Acc) ->
        [{Pattern, Handler} | Acc]
    end, [], Handlers).
```

---

## Compile-time Checks

```erlang
%% Static checks at compile time

%% Dialyzer: type checking
%% -spec annotations
-spec add(integer(), integer()) -> integer().
add(A, B) -> A + B.

%% Type definitions
-type user_id() :: pos_integer().
-type user() :: #{
    id := user_id(),
    name := binary(),
    email := binary(),
    created_at := integer()
}.
-type error() :: {error, atom()}.

-spec get_user(user_id()) -> {ok, user()} | error().
get_user(UserId) ->
    db:query("SELECT * FROM users WHERE id = $1", [UserId]).

%% Opaque types: hide internal representation
-opaque token() :: binary().

-spec generate_token() -> token().
generate_token() ->
    base64:encode(crypto:strong_rand_bytes(32)).

%% Xref: find undefined/unused functions
check_xref() ->
    xref:start(s),
    xref:add_application(s, my_app),
    xref:analyze(s, undefined_function_calls),
    xref:analyze(s, unused_local_functions),
    xref:stop(s).

%% Erlang compiler warnings: useful flags
-compile([
    warn_unused_vars,
    warn_unused_import,
    warn_missing_spec,     %% missing -spec
    warn_export_all,
    warn_shadow_vars
]).

%% Strict mode warning in newer OTP
-compile([{feature, maybe_expr, enable}]).  %% OTP 25+

%% behaviours: compile-time callback verification
-behaviour(gen_server).
%% Compiler warns if required callbacks are missing
```

---

## ตัวอย่างจริง: Auto-generating REST Handlers

```erlang
%% rest_gen.erl: generate CRUD handlers from schema

%% Define schema at module level
-module(users_handler).
-rest_resource(#{
    path => "/api/users",
    schema => #{
        id => {integer, required},
        name => {binary, required, [{min_length, 1}, {max_length, 100}]},
        email => {binary, required, [email_format]},
        age => {integer, optional, [{min, 0}, {max, 150}]}
    },
    operations => [list, get, create, update, delete]
}).

%% Parse transform processes -rest_resource and generates:
%%   init/2 (route handler)
%%   list/2 (GET /users)
%%   get/2 (GET /users/:id)
%%   create/2 (POST /users)
%%   update/2 (PUT /users/:id)
%%   delete/2 (DELETE /users/:id)
%%   validate/1 (input validation)

%% The generated code looks like:
init(Req, State) ->
    Method = cowboy_req:method(Req),
    Id = cowboy_req:binding(id, Req),
    route(Method, Id, Req, State).

route(<<"GET">>, undefined, Req, State) -> list(Req, State);
route(<<"GET">>, Id, Req, State) -> get(binary_to_integer(Id), Req, State);
route(<<"POST">>, _, Req, State) -> create(Req, State);
route(<<"PUT">>, Id, Req, State) -> update(binary_to_integer(Id), Req, State);
route(<<"DELETE">>, Id, Req, State) -> delete(binary_to_integer(Id), Req, State).

%% Parse transform implementation
-module(rest_gen_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Opts) ->
    %% Find -rest_resource attribute
    case find_rest_attribute(Forms) of
        {ok, Config} ->
            %% Generate handler functions
            Generated = generate_handlers(Config),
            %% Inject into module
            inject_forms(Forms, Generated);
        not_found ->
            Forms
    end.

find_rest_attribute(Forms) ->
    Attrs = [A || {attribute, _, rest_resource, A} <- Forms],
    case Attrs of
        [Config | _] -> {ok, Config};
        [] -> not_found
    end.

generate_handlers(#{schema := Schema, operations := Ops} = Config) ->
    [generate_handler(Op, Schema) || Op <- Ops].

generate_handler(list, _Schema) ->
    %% Generate: list(Req, State) -> ...
    erl_syntax:function(
        erl_syntax:atom(list),
        [erl_syntax:clause(
            [erl_syntax:variable('Req'), erl_syntax:variable('State')],
            none,
            [generate_list_body()]
        )]
    );
generate_handler(get, _Schema) ->
    %% Similar for get, create, update, delete
    erl_syntax:function(erl_syntax:atom(get), []).

generate_list_body() ->
    %% Generate: {ok, Items, _} = db:list(?MODULE), reply(200, Items, Req, State)
    erl_syntax:atom(ok).  %% simplified
```

---

## สรุป Part 54

| Tool | Purpose |
|------|---------|
| `-define` | Compile-time constants and macros |
| Parse transforms | AST manipulation at compile time |
| `erl_syntax` | High-level AST building |
| `merl` | Template-based code generation |
| `compile:forms/2` | Runtime module compilation |
| `-spec` | Type annotations for Dialyzer |
| `xref` | Dead code and undefined call detection |
| `-behaviour` | Compile-time callback verification |

---

*[← Part 53: Concurrency Patterns](part_53_concurrency_patterns.md) | [Part 55: Interoperability →](part_55_interoperability.md)*
