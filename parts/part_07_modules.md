# Part 7: Modules และการจัดระเบียบโค้ด

## สารบัญ
1. [Module คืออะไร?](#module-คืออะไร)
2. [Module Declaration](#module-declaration)
3. [Export และ Import](#export-และ-import)
4. [Module Attributes](#module-attributes)
5. [Header Files (.hrl)](#header-files-hrl)
6. [Module Info](#module-info)
7. [OTP Behaviours Overview](#otp-behaviours-overview)
8. [Naming Conventions](#naming-conventions)
9. [Project Structure](#project-structure)
10. [การแบ่ง Module](#การแบ่ง-module)
11. [สรุป](#สรุป)

---

## Module คืออะไร?

Module คือหน่วยพื้นฐานของการจัดระเบียบโค้ดใน Erlang:

```
Module = ไฟล์ .erl 1 ไฟล์ = namespace 1 ชุด

my_app/
├── src/
│   ├── user_service.erl      → module: user_service
│   ├── auth_service.erl      → module: auth_service
│   ├── database.erl          → module: database
│   └── my_app_sup.erl        → module: my_app_sup
└── include/
    └── my_app.hrl            → shared records/macros
```

### ข้อกำหนด
- ชื่อ module ต้องตรงกับชื่อไฟล์ (ไม่รวม `.erl`)
- ชื่อ module เป็น atom (ตัวพิมพ์เล็กทั้งหมด, snake_case)
- ทุก function ต้อง declare ใน module

---

## Module Declaration

```erlang
%% ไฟล์: user_service.erl
%% ต้องอยู่บรรทัดแรกสุดของไฟล์

-module(user_service).

%% Erlang/OTP ต้องการ -module ก่อน attributes อื่นๆ
%% ชื่อ module ต้องตรงกับชื่อไฟล์

%% Attributes อื่นๆ ตามมาหลัง -module
-author("Your Name").
-vsn("1.0.0").
-created("2024-01-01").
```

---

## Export และ Import

### Export

```erlang
-module(math_utils).

%% Export format: -export([Name/Arity, ...]).
-export([
    add/2,
    subtract/2,
    multiply/2,
    divide/2,
    factorial/1,
    fibonacci/1
]).

%% Functions ที่ไม่ export เป็น private
%% เรียกได้แค่ภายใน module นี้เท่านั้น
add(A, B) -> A + B.
subtract(A, B) -> A - B.
multiply(A, B) -> A * B.
divide(A, 0) -> {error, division_by_zero};
divide(A, B) -> {ok, A / B}.

%% Private helper
factorial(N) -> factorial_impl(N, 1).
factorial_impl(0, Acc) -> Acc;
factorial_impl(N, Acc) -> factorial_impl(N - 1, N * Acc).

fibonacci(N) -> fibonacci_impl(N, 0, 1).
fibonacci_impl(0, A, _) -> A;
fibonacci_impl(N, A, B) -> fibonacci_impl(N - 1, B, A + B).

%% private - ไม่ได้ export
helper_function(X) -> X * 2.
```

### Export ทั้งหมด (สำหรับ debugging เท่านั้น!)

```erlang
%% -compile(export_all). %% อย่าใช้ใน production!
%% ใช้เฉพาะ debug/test

%% แทนที่จะใช้ export_all ให้ export ที่ต้องการเท่านั้น
```

### Import

```erlang
%% Import ช่วยให้เรียก function ข้าม module โดยไม่ต้องระบุ module name
-module(my_module).
-import(lists, [map/2, filter/2, foldl/3]).
-import(string, [upper/1, lower/1]).

%% หลัง import สามารถเรียกโดยตรง
process(List) ->
    map(fun(X) -> X * 2 end, List).  % ไม่ต้องเขียน lists:map

%% แต่ส่วนใหญ่ใน Erlang community ไม่นิยมใช้ -import
%% เพราะอ่านโค้ดแล้วไม่รู้ว่า function มาจาก module ไหน
%% แนะนำให้เขียน module name ตลอด:
process2(List) ->
    lists:map(fun(X) -> X * 2 end, List).  % ชัดเจนกว่า
```

---

## Module Attributes

```erlang
-module(example).

%% Standard attributes
-vsn("1.2.3").          % version
-author("Alice Smith").  % author

%% Compile options
-compile([debug_info]).                    % เก็บ debug info
-compile([warn_unused_vars]).             % เตือน unused variables
-compile([warn_unused_function]).         % เตือน unused functions
-compile([{parse_transform, my_pt}]).     % parse transform

%% Behavior declaration
-behaviour(gen_server).    % OTP GenServer
-behavior(gen_server).     % อักขระสะกดแบบ American ก็ได้

%% Defines (macros)
-define(MAX_RETRIES, 3).
-define(DEFAULT_TIMEOUT, 5000).
-define(VERSION, "1.0.0").
-define(DEBUG, true).

%% Macro with function-like syntax
-define(LOG(Msg), io:format("[LOG] ~p~n", [Msg])).
-define(LOG(Level, Msg), io:format("[~s] ~p~n", [Level, Msg])).

%% Type definitions
-type status() :: active | inactive | pending.
-type user_id() :: pos_integer().
-type email() :: binary().

%% Opaque types (internal structure hidden)
-opaque connection() :: #{socket := port(), status := status()}.

%% Record definitions
-record(user, {
    id :: user_id(),
    name :: binary(),
    email :: email(),
    status = active :: status(),
    created_at :: calendar:datetime()
}).

%% Callback declarations (สำหรับ behaviour definitions)
-callback init(Args :: term()) -> {ok, State :: term()} | {error, Reason :: term()}.
-callback handle_call(Request :: term(), State :: term()) ->
    {reply, Reply :: term(), NewState :: term()}.
```

### Custom Attributes

```erlang
%% Custom attributes สำหรับ metadata
-module(api_handler).
-api_version("v2").
-deprecated([{old_function, 1}]).

%% อ่าน custom attributes ด้วย module_info
api_handler:module_info(attributes).
% = [{vsn, [...]}, {api_version, ["v2"]}, {deprecated, [{old_function, 1}]}]
```

---

## Header Files (.hrl)

Header files ใช้เก็บ shared definitions (records, macros, types):

```erlang
%% ไฟล์: include/models.hrl

%% Record definitions
-record(user, {
    id          :: pos_integer(),
    name        :: binary(),
    email       :: binary(),
    password    :: binary(),
    role = user :: admin | user | moderator,
    active = true :: boolean(),
    created_at  :: calendar:datetime()
}).

-record(product, {
    id        :: pos_integer(),
    name      :: binary(),
    price     :: float(),
    quantity  :: non_neg_integer(),
    category  :: binary()
}).

-record(order, {
    id          :: pos_integer(),
    user_id     :: pos_integer(),
    items       :: list(),
    total       :: float(),
    status = pending :: pending | processing | shipped | delivered | cancelled,
    created_at  :: calendar:datetime()
}).
```

```erlang
%% ไฟล์: include/constants.hrl

%% Application constants
-define(APP_NAME, my_app).
-define(APP_VERSION, "1.0.0").

%% Timeouts (milliseconds)
-define(DEFAULT_TIMEOUT, 5000).
-define(LONG_TIMEOUT, 30000).
-define(CONNECT_TIMEOUT, 10000).

%% Limits
-define(MAX_RETRIES, 3).
-define(MAX_CONNECTIONS, 100).
-define(PAGE_SIZE, 20).

%% Error codes
-define(ERR_NOT_FOUND, 404).
-define(ERR_UNAUTHORIZED, 401).
-define(ERR_FORBIDDEN, 403).
-define(ERR_INTERNAL, 500).

%% Logging macros
-define(LOG_DEBUG(Msg), logger:debug(Msg)).
-define(LOG_INFO(Msg), logger:info(Msg)).
-define(LOG_WARNING(Msg), logger:warning(Msg)).
-define(LOG_ERROR(Msg), logger:error(Msg)).
-define(LOG_ERROR(Msg, Data), logger:error(Msg, Data)).
```

```erlang
%% ใช้ header file
-module(user_service).
-include("models.hrl").     % relative path
-include_lib("my_app/include/models.hrl").  % absolute with app name

%% ตอนนี้ใช้ record ที่ define ใน models.hrl ได้แล้ว
create_user(Name, Email) ->
    #user{
        id = generate_id(),
        name = Name,
        email = Email,
        created_at = calendar:local_time()
    }.

get_user_name(#user{name = Name}) ->
    Name.

update_user_email(#user{} = User, NewEmail) ->
    User#user{email = NewEmail}.
```

---

## Module Info

```erlang
%% module_info/0 - แสดงข้อมูลทั้งหมดของ module
math_utils:module_info().
% = [{module, math_utils},
%    {exports, [{add,2}, {subtract,2}, {multiply,2}, ...]},
%    {attributes, [{vsn, [...]}, {author, ["Alice"]}, ...]},
%    {compile, [{version, "7.0.4"}, {time, {...}}, ...]},
%    {md5, <<...>>},
%    {native, false}]

%% module_info/1 - ดูข้อมูลเฉพาะส่วน
math_utils:module_info(exports).
% = [{add,2}, {subtract,2}, ...]

math_utils:module_info(attributes).
% = [{vsn, [...]}, {author, [...]}]

math_utils:module_info(module).
% = math_utils

math_utils:module_info(compile).
% = [{version, "7.0.4"}, ...]
```

---

## OTP Behaviours Overview

Behaviour คือ interface ที่ต้องการ implementation:

```erlang
%% gen_server - Generic Server
-behaviour(gen_server).

%% Callbacks ที่ต้องมี:
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2, code_change/3]).

init(Args) -> {ok, InitialState}.
handle_call(Request, From, State) -> {reply, Reply, NewState}.
handle_cast(Request, State) -> {noreply, NewState}.
handle_info(Info, State) -> {noreply, NewState}.
terminate(Reason, State) -> ok.
code_change(OldVsn, State, Extra) -> {ok, NewState}.
```

```erlang
%% supervisor - Supervisor
-behaviour(supervisor).

-export([init/1]).

init(Args) ->
    {ok, {
        #{strategy => one_for_one, intensity => 5, period => 10},
        [
            #{id => worker1,
              start => {worker_module, start_link, []},
              restart => permanent,
              type => worker}
        ]
    }}.
```

```erlang
%% gen_statem - State Machine
-behaviour(gen_statem).

%% Callbacks:
init(Args) -> {ok, initial_state, InitData}.
callback_mode() -> state_functions.

initial_state(enter, _OldState, Data) -> {keep_state, Data};
initial_state({call, From}, Event, Data) ->
    {next_state, next_state, Data, [{reply, From, ok}]}.
```

---

## Naming Conventions

```erlang
%% Module Names
user_service        % service modules
user_handler        % request handlers  
user_store          % data storage
user_model          % data models
user_validator      % validation
user_utils          % utility functions
my_app_sup          % supervisors (suffix _sup)
my_app_app          % application (suffix _app)

%% Function Names
get_user/1          % retrieve
create_user/1       % create
update_user/2       % update
delete_user/1       % delete
list_users/0        % list all
find_users_by/2     % find with criteria
validate_user/1     % validate
format_user/1       % format for output
init/1              % initialization
start_link/0        % start with link
handle_request/2    % handle requests
process_message/2   % process messages

%% Variables
UserId, UserName, UserEmail  % CamelCase
HttpStatus, ApiKey           % Abbreviations ok
Pid, Ref                     % Known abbreviations
_Unused, _IgnoredVar         % Prefix _ for unused

%% Atoms
ok, error, undefined, nil    % Common
active, inactive, pending    % States
get, post, put, delete       % HTTP methods
```

---

## Project Structure

### Standard OTP Application Structure

```
my_app/
├── rebar.config           # Build configuration
├── rebar.lock             # Locked dependencies
├── README.md
├── LICENSE
├── src/
│   ├── my_app.app.src     # Application resource file
│   ├── my_app_app.erl     # Application callback
│   ├── my_app_sup.erl     # Top-level supervisor
│   │
│   ├── users/             # Feature modules
│   │   ├── user_service.erl
│   │   ├── user_store.erl
│   │   └── user_validator.erl
│   │
│   ├── products/
│   │   ├── product_service.erl
│   │   └── product_store.erl
│   │
│   └── utils/
│       ├── http_utils.erl
│       └── crypto_utils.erl
│
├── include/
│   ├── models.hrl         # Shared record definitions
│   └── constants.hrl      # Shared constants
│
├── test/
│   ├── user_service_test.erl
│   └── product_service_test.erl
│
├── priv/
│   ├── migrations/        # Database migrations
│   └── static/            # Static files
│
└── config/
    ├── sys.config         # Runtime configuration
    └── vm.args            # VM arguments
```

### Multi-App (Umbrella) Structure

```
my_umbrella/
├── apps/
│   ├── my_core/           # Core library
│   │   ├── src/
│   │   └── include/
│   │
│   ├── my_web/            # Web application
│   │   └── src/
│   │
│   └── my_worker/         # Background workers
│       └── src/
│
├── rebar.config
└── config/
```

---

## การแบ่ง Module

### Module Cohesion

```erlang
%% ดี - Module ทำสิ่งเดียวอย่างดี

%% user_service.erl - Business Logic
-module(user_service).
-export([create/1, get/1, update/2, delete/1]).

create(Params) -> ...
get(Id) -> ...
update(Id, Params) -> ...
delete(Id) -> ...

%% user_store.erl - Data Layer
-module(user_store).
-export([insert/1, find/1, update/2, remove/1]).

%% user_validator.erl - Validation
-module(user_validator).
-export([validate_create/1, validate_update/2]).
```

### Module Coupling

```erlang
%% หลีกเลี่ยง circular dependencies!
%% Module A -> Module B -> Module A  (BAD!)

%% แก้ไขด้วย:
%% 1. Introduce shared module C
%%    A -> C <- B  (GOOD)

%% 2. ใช้ Behaviour/Callback
%% 3. ส่ง function ผ่าน argument

%% ตัวอย่าง: หลีกเลี่ยง circular dependency
%% แทนที่ user_service เรียก notification_service โดยตรง:

%% user_service.erl
create_user(Params, NotifyFn) ->
    case do_create(Params) of
        {ok, User} ->
            NotifyFn(User),  % ส่ง function เข้ามา
            {ok, User};
        Error -> Error
    end.

%% เรียกจากภายนอก
notification_service:send_welcome_email(User),
user_service:create_user(Params, 
    fun(User) -> notification_service:send_welcome_email(User) end).
```

### Full Example: Complete Module

```erlang
%%% @doc
%%% User Service Module
%%%
%%% Handles user management operations including
%%% creation, retrieval, update, and deletion.
%%% @end
-module(user_service).

-include("../include/models.hrl").
-include("../include/constants.hrl").

%% Public API
-export([
    create/1,
    get/1,
    get_by_email/1,
    update/2,
    delete/1,
    list/0,
    list/1,
    activate/1,
    deactivate/1
]).

%% Types
-type user_id() :: pos_integer().
-type user_params() :: #{
    name := binary(),
    email := binary(),
    password := binary()
}.
-type user_update_params() :: #{
    name => binary(),
    email => binary()
}.
-type error() :: {error, term()}.
-type result(T) :: {ok, T} | error().

%%--------------------------------------------------------------------
%% Public API
%%--------------------------------------------------------------------

-spec create(user_params()) -> result(#user{}).
create(Params) ->
    case user_validator:validate_create(Params) of
        ok ->
            do_create(Params);
        {error, _} = Error ->
            Error
    end.

-spec get(user_id()) -> result(#user{}).
get(Id) when is_integer(Id), Id > 0 ->
    user_store:find(Id);
get(_) ->
    {error, invalid_id}.

-spec get_by_email(binary()) -> result(#user{}).
get_by_email(Email) when is_binary(Email) ->
    user_store:find_by_email(Email);
get_by_email(_) ->
    {error, invalid_email}.

-spec update(user_id(), user_update_params()) -> result(#user{}).
update(Id, Params) ->
    case get(Id) of
        {ok, User} ->
            case user_validator:validate_update(User, Params) of
                ok ->
                    do_update(User, Params);
                {error, _} = Error ->
                    Error
            end;
        Error ->
            Error
    end.

-spec delete(user_id()) -> result(ok).
delete(Id) ->
    case get(Id) of
        {ok, _} ->
            user_store:remove(Id);
        Error ->
            Error
    end.

-spec list() -> {ok, [#user{}]}.
list() ->
    list(#{}).

-spec list(map()) -> {ok, [#user{}]}.
list(Filters) ->
    user_store:find_all(Filters).

-spec activate(user_id()) -> result(#user{}).
activate(Id) ->
    update(Id, #{active => true}).

-spec deactivate(user_id()) -> result(#user{}).
deactivate(Id) ->
    update(Id, #{active => false}).

%%--------------------------------------------------------------------
%% Internal Functions
%%--------------------------------------------------------------------

-spec do_create(user_params()) -> result(#user{}).
do_create(#{name := Name, email := Email, password := Password}) ->
    HashedPassword = crypto:hash(sha256, Password),
    User = #user{
        name = Name,
        email = Email,
        password = base64:encode(HashedPassword),
        created_at = calendar:local_time()
    },
    user_store:insert(User).

-spec do_update(#user{}, user_update_params()) -> result(#user{}).
do_update(User, Params) ->
    UpdatedUser = maps:fold(
        fun(name, V, U) -> U#user{name = V};
           (email, V, U) -> U#user{email = V};
           (active, V, U) -> U#user{active = V};
           (_, _, U) -> U
        end,
        User,
        Params
    ),
    user_store:update(UpdatedUser).
```

---

## สรุป Part 7

| Concept | Description |
|---------|-------------|
| `-module(name)` | Module declaration |
| `-export([f/n])` | Make functions public |
| `-import(M, [f/n])` | Import from module (rarely used) |
| `-define(NAME, val)` | Macro definition |
| `-record(name, {...})` | Record type |
| `-type name() :: ...` | Type alias |
| `-spec f(T) -> R` | Function type spec |
| `-behaviour(name)` | OTP Behaviour |
| `.hrl` files | Shared definitions |

### ก้าวต่อไป

ใน **Part 8** เราจะเรียน Lists อย่างละเอียด:
- List Comprehensions
- List Operations
- lists module
- Efficient List Processing
- IOLists

---

## แบบฝึกหัด Part 7

```erlang
%% แบบฝึกหัดที่ 1: สร้าง module ครบวงจร
%% สร้าง product.hrl ที่มี record:
%% -record(product, {id, name, price, quantity, category})
%%
%% สร้าง product_service.erl ที่มี:
%% - create/1
%% - get/1
%% - update/2
%% - delete/1
%% - list_by_category/1

%% แบบฝึกหัดที่ 2: Module Introspection
%% สร้าง function ที่:
%% - รับ module name
%% - return list ของ exported functions
%% - ตรวจว่า function นั้นมีอยู่ใน module หรือไม่

%% แบบฝึกหัดที่ 3: สร้าง math_extra module ที่
%% - extend standard math module
%% - เพิ่ม: is_prime/1, gcd/2, lcm/2, combinations/2, permutations/2
```

---

*[← Part 6: Functions](part_06_functions.md) | [Part 8: Lists →](part_08_lists.md)*
