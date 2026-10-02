# Part 91: Parse Transforms and Metaprogramming

## สารบัญ
1. [Parse Transform Basics](#basics)
2. [Abstract Format](#abstract-format)
3. [Writing a Parse Transform](#writing)
4. [Macros and Compile-Time Code](#macros)
5. [Code Generation](#codegen)
6. [ตัวอย่างจริง: Logger Transform](#logger-transform)

---

## Parse Transform Basics

```erlang
%% Parse transforms run at compile time, transform the AST
%% They are powerful but should be used sparingly

%% How to enable a parse transform:
-module(my_module).
-compile({parse_transform, my_transform}).

%% Or in rebar.config:
%% {erl_opts, [{parse_transform, lager_transform}]}

%% Famous parse transforms:
%% - lager_transform (lager logging library — now deprecated in favor of logger)
%% - exprecs (record generation)
%% - cth_readable (Common Test formatting)
%% - meck_transform (mocking)

%% The parse_transform/2 function receives the abstract form of the module
%% and returns a (potentially modified) abstract form

-module(my_transform).
-export([parse_transform/2]).

parse_transform(Forms, Options) ->
    %% Forms is a list of abstract syntax trees
    %% Options is the compiler options
    %% Return modified or unmodified Forms
    io:format("Transform called with ~p forms~n", [length(Forms)]),
    
    %% Transform each form
    [transform_form(F) || F <- Forms].

transform_form(Form) ->
    Form.  %% identity transform
```

---

## Abstract Format

```erlang
%% Understanding Erlang's abstract format

%% Compile to abstract format:
%% erlc +to_pp source.erl   (pretty-printed)
%% epp:parse_file("source.erl", [], [])

%% Example: the code
%%   foo(X) -> X + 1.
%%
%% Becomes abstract form:
%%   {function, 5, foo, 1,
%%     [{clause, 5,
%%       [{var, 5, 'X'}],    %% parameter: variable X
%%       [],                  %% no guards
%%       [{op, 5, '+',        %% body: X + 1
%%          {var, 5, 'X'},
%%          {integer, 5, 1}}]
%%      }]}

%% Common abstract form nodes:
abstract_forms_reference() ->
    %% Variables
    {var, Line, Name},              %% {var, 1, 'X'}
    
    %% Literals
    {integer, Line, Value},         %% {integer, 1, 42}
    {float, Line, Value},           %% {float, 1, 3.14}
    {atom, Line, Value},            %% {atom, 1, hello}
    {string, Line, Value},          %% {string, 1, "hello"}
    
    %% Compound
    {cons, Line, Head, Tail},       %% [H | T]
    {nil, Line},                    %% []
    {tuple, Line, Elements},        %% {a, b, c}
    
    %% Calls
    {call, Line, Fun, Args},        %% f(a, b)
    {remote, Line, Mod, Fun},       %% Mod:Fun
    
    %% Operations
    {op, Line, Op, Left, Right},    %% A + B, A =:= B
    {op, Line, Op, Operand},        %% not X
    
    %% Control
    {'case', Line, Expr, Clauses},
    {'if', Line, Clauses},
    {'receive', Line, Clauses},
    {'fun', Line, {clauses, Clauses}},
    
    %% Function definitions
    {function, Line, Name, Arity, Clauses},
    {clause, Line, Patterns, Guards, Body},
    
    %% Module attributes
    {attribute, Line, module, Name},
    {attribute, Line, export, [{Name, Arity}]},
    {attribute, Line, compile, Options}.
```

---

## Writing a Parse Transform

```erlang
%% timing_transform.erl — automatically add timing to all functions

-module(timing_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    Module = get_module_name(Forms),
    [transform_form(F, Module) || F <- Forms].

get_module_name(Forms) ->
    case lists:keyfind(module, 3, Forms) of
        {attribute, _, module, Name} -> Name;
        _ -> unknown
    end.

transform_form({function, Line, Name, Arity, Clauses}, Module) ->
    %% Skip internal functions starting with _
    case atom_to_list(Name) of
        [$_ | _] ->
            {function, Line, Name, Arity, Clauses};
        _ ->
            %% Wrap each clause to add timing
            NewClauses = [wrap_clause(C, Module, Name) || C <- Clauses],
            {function, Line, Name, Arity, NewClauses}
    end;
transform_form(Form, _Module) ->
    Form.

wrap_clause({clause, Line, Patterns, Guards, Body}, Module, FunName) ->
    %% Generate: T1 = erlang:monotonic_time(microsecond), ...Body..., log timing
    T1Var = {var, Line, '_T1'},
    T2Var = {var, Line, '_T2'},
    ResultVar = {var, Line, '_Result'},
    
    StartTime = {match, Line, T1Var,
                  {call, Line, {remote, Line, {atom, Line, erlang}, {atom, Line, monotonic_time}},
                   [{atom, Line, microsecond}]}},
    
    OrigBody = case Body of
        [Single] ->
            {match, Line, ResultVar, Single};
        Multiple ->
            {match, Line, ResultVar, {block, Line, Multiple}}
    end,
    
    EndTime = {match, Line, T2Var,
               {call, Line, {remote, Line, {atom, Line, erlang}, {atom, Line, monotonic_time}},
                [{atom, Line, microsecond}]}},
    
    %% logger:debug("Function timing", #{module => M, function => F, duration_us => D})
    LogCall = {call, Line, {remote, Line, {atom, Line, logger}, {atom, Line, debug}},
               [{string, Line, "function_timing"},
                {map, Line, [
                    {map_field_assoc, Line, {atom, Line, module}, {atom, Line, Module}},
                    {map_field_assoc, Line, {atom, Line, function}, {atom, Line, FunName}},
                    {map_field_assoc, Line, {atom, Line, duration_us},
                     {op, Line, '-', T2Var, T1Var}}
                ]}]},
    
    NewBody = [StartTime, OrigBody, EndTime, LogCall, ResultVar],
    {clause, Line, Patterns, Guards, NewBody}.
```

---

## Macros and Compile-Time Code

```erlang
%% macros.erl — advanced macro usage

-module(macros_demo).

%% Simple value macros
-define(VERSION, "1.2.3").
-define(MAX_CONNECTIONS, 1000).
-define(PI, 3.14159265358979).

%% Function-like macros
-define(ASSERT(Cond),
    case (Cond) of
        true -> ok;
        false -> error({assertion_failed, ??Cond, ?FILE, ?LINE})
    end).

-define(ASSERT_EQUAL(A, B),
    case (A) =:= (B) of
        true -> ok;
        false -> error({not_equal, {expected, B}, {got, A}, ?FILE, ?LINE})
    end).

%% Logging macro with source location
-define(LOG(Level, Msg, Meta),
    logger:Level(Msg, maps:merge(#{
        file => ?FILE,
        line => ?LINE,
        function => ?FUNCTION_NAME,
        arity => ?FUNCTION_ARITY
    }, Meta))).

-define(LOG_INFO(Msg, Meta), ?LOG(info, Msg, Meta)).
-define(LOG_ERROR(Msg, Meta), ?LOG(error, Msg, Meta)).

%% Record helpers
-define(RECORD_TO_MAP(Record, RecordName),
    maps:from_list(lists:zip(
        record_info(fields, RecordName),
        tl(tuple_to_list(Record))
    ))).

%% Timing macro
-define(TIME(Label, Expr),
    begin
        __T1 = erlang:monotonic_time(microsecond),
        __Result = (Expr),
        __T2 = erlang:monotonic_time(microsecond),
        io:format("~s: ~p μs~n", [Label, __T2 - __T1]),
        __Result
    end).

%% Usage
demo() ->
    %% Compile-time constants
    io:format("Version: ~s~n", [?VERSION]),
    
    %% Assertion
    ?ASSERT(1 + 1 =:= 2),
    
    %% Timing
    Result = ?TIME("fibonacci", fib(30)),
    io:format("fib(30) = ~p~n", [Result]),
    
    %% Logging with location
    ?LOG_INFO("Starting demo", #{pid => self()}).

fib(0) -> 0;
fib(1) -> 1;
fib(N) -> fib(N-1) + fib(N-2).
```

---

## Code Generation

```erlang
%% code_generator.erl — generate Erlang modules programmatically

-module(code_generator).
-export([generate_crud_module/2, generate_validator/2]).

%% Generate a CRUD module for a given resource
generate_crud_module(ResourceName, Fields) ->
    ModuleName = list_to_atom(atom_to_list(ResourceName) ++ "_service"),
    TableName = ResourceName,
    
    %% Generate module source as string
    Source = io_lib:format(
        "-module(~w).~n"
        "-export([create/1, get/1, list/0, update/2, delete/1]).~n"
        "~n"
        "create(Data) ->~n"
        "    Id = uuid:generate(),~n"
        "    Record = Data#{id => Id, created_at => erlang:system_time(second)},~n"
        "    ets:insert(~w, {Id, Record}),~n"
        "    {ok, Record}.~n"
        "~n"
        "get(Id) ->~n"
        "    case ets:lookup(~w, Id) of~n"
        "        [{Id, Record}] -> {ok, Record};~n"
        "        [] -> not_found~n"
        "    end.~n"
        "~n"
        "list() ->~n"
        "    [R || {_, R} <- ets:tab2list(~w)].~n"
        "~n"
        "update(Id, Updates) ->~n"
        "    case get(Id) of~n"
        "        {ok, Record} ->~n"
        "            Updated = maps:merge(Record, Updates),~n"
        "            ets:insert(~w, {Id, Updated}),~n"
        "            {ok, Updated};~n"
        "        not_found -> not_found~n"
        "    end.~n"
        "~n"
        "delete(Id) ->~n"
        "    ets:delete(~w, Id),~n"
        "    ok.~n",
        [ModuleName, TableName, TableName, TableName, TableName, TableName]
    ),
    
    %% Compile to binary
    {ok, ModuleName, Binary} = compile:forms(
        parse_forms(Source),
        [binary]
    ),
    
    %% Load into VM
    code:load_binary(ModuleName, atom_to_list(ModuleName) ++ ".erl", Binary),
    
    {ok, ModuleName}.

parse_forms(Source) ->
    {ok, Tokens, _} = erl_scan:string(Source),
    {ok, Forms} = erl_parse:parse_forms(Tokens),
    Forms.

%% Generate validator module from schema
generate_validator(ModuleName, Schema) ->
    Clauses = [{Field, maps:get(Field, Schema)} || Field <- maps:keys(Schema)],
    
    Source = io_lib:format(
        "-module(~w).~n"
        "-export([validate/1]).~n"
        "~n"
        "validate(Data) ->~n"
        "    Errors = lists:filtermap(fun({Field, Value}) ->~n"
        "        case validate_field(Field, Value) of~n"
        "            ok -> false;~n"
        "            {error, E} -> {true, {Field, E}}~n"
        "        end~n"
        "    end, maps:to_list(Data)),~n"
        "    case Errors of~n"
        "        [] -> {ok, Data};~n"
        "        _ -> {error, Errors}~n"
        "    end.~n"
        "~n"
        "~s~n",
        [ModuleName, generate_validators(Clauses)]
    ),
    
    parse_forms(Source).

generate_validators(Clauses) ->
    [io_lib:format(
        "validate_field(~w, Value) ->~n"
        "    ~s;~n",
        [Field, generate_validation(Rules)]
    ) || {Field, Rules} <- Clauses] ++
    ["validate_field(_, _) -> ok."].

generate_validation(#{type := string, required := true, min_length := Min}) ->
    io_lib:format(
        "if is_binary(Value) andalso byte_size(Value) >= ~p -> ok;~n"
        "   true -> {error, invalid}~n"
        "    end", [Min]).
```

---

## ตัวอย่างจริง: Logger Transform

```erlang
%% logger_transform.erl — auto-add module/function to log calls

-module(logger_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Opts) ->
    Module = get_module(Forms),
    walk_forms(Forms, Module).

get_module(Forms) ->
    case [N || {attribute, _, module, N} <- Forms] of
        [N | _] -> N;
        [] -> undefined
    end.

walk_forms(Forms, Module) ->
    [walk_form(F, Module) || F <- Forms].

walk_form({function, Line, Name, Arity, Clauses}, Module) ->
    NewClauses = [walk_clause(C, Module, Name) || C <- Clauses],
    {function, Line, Name, Arity, NewClauses};
walk_form(Form, _) ->
    Form.

walk_clause({clause, Line, Pats, Guards, Body}, Module, Fun) ->
    NewBody = [walk_expr(E, Module, Fun) || E <- Body],
    {clause, Line, Pats, Guards, NewBody}.

%% Transform logger:Level(Msg) → logger:Level(Msg, #{module => M, function => F})
walk_expr({call, Line, {remote, _, {atom, _, logger}, {atom, _, Level}}, [Msg]}, Module, Fun)
  when Level =:= info; Level =:= warning; Level =:= error; Level =:= debug ->
    Meta = {map, Line, [
        {map_field_assoc, Line, {atom, Line, module}, {atom, Line, Module}},
        {map_field_assoc, Line, {atom, Line, function}, {atom, Line, Fun}}
    ]},
    {call, Line, {remote, Line, {atom, Line, logger}, {atom, Line, Level}}, [Msg, Meta]};

walk_expr({call, Line, Fun, Args}, Module, FunName) ->
    NewArgs = [walk_expr(A, Module, FunName) || A <- Args],
    {call, Line, walk_expr(Fun, Module, FunName), NewArgs};

walk_expr(Expr, _, _) ->
    Expr.
```

---

## สรุป Part 91

| Feature | Use Case |
|---------|---------|
| Parse transforms | Compile-time code transformation |
| Abstract format | AST manipulation |
| Macros (-define) | Compile-time constants/functions |
| Code generation | Dynamic module creation |
| compile:forms/2 | Compile AST to binary |
| code:load_binary/3 | Load compiled module |

**Warning**: Parse transforms are powerful but dangerous — they make code harder to understand and debug. Use them only when macros or behaviors cannot solve the problem.

---

*[← Part 90: NIFs](part_90_nifs.md) | [Part 92: Advanced OTP Patterns →](part_92_advanced_otp.md)*
