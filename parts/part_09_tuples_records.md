# Part 9: Tuples และ Records

## สารบัญ
1. [Tuples ทบทวน + เพิ่มเติม](#tuples-ทบทวน--เพิ่มเติม)
2. [Records คืออะไร?](#records-คืออะไร)
3. [การสร้าง Record](#การสร้าง-record)
4. [การใช้ Record](#การใช้-record)
5. [Record Pattern Matching](#record-pattern-matching)
6. [Records กับ Functions](#records-กับ-functions)
7. [Nested Records](#nested-records)
8. [Records vs Maps](#records-vs-maps)
9. [Record Macros](#record-macros)
10. [ตัวอย่างจริง: User Management System](#ตัวอย่างจริง-user-management-system)
11. [สรุป](#สรุป)

---

## Tuples ทบทวน + เพิ่มเติม

```erlang
%% Tuples = fixed-size, ordered sequence ของ values
%% ใช้ {} syntax
%% เข้าถึง element ด้วย element/2 หรือ pattern matching

%% สร้าง Tuples
Empty = {}.
Pair = {hello, world}.
Triple = {1, 2, 3}.
Nested = {{1, 2}, {3, 4}}.
Mixed = {alice, 30, <<"alice@example.com">>, [friend1, friend2]}.

%% Tagged Tuples (ที่นิยมมาก)
{ok, Value}                    %% success result
{error, Reason}                %% failure result
{point, X, Y}                  %% 2D point
{rect, Left, Top, Right, Bottom}  %% rectangle
{http_request, <<"GET">>, <<"/">>}  %% HTTP request

%% Tuple Operations
T = {a, b, c, d, e}.

element(1, T).          %% = a (1-indexed!)
element(3, T).          %% = c
tuple_size(T).          %% = 5

setelement(2, T, z).    %% = {a, z, c, d, e} (สร้าง tuple ใหม่!)
T.                      %% = {a, b, c, d, e} (ไม่เปลี่ยน)

tuple_to_list(T).       %% = [a, b, c, d, e]
list_to_tuple([a,b,c]). %% = {a, b, c}
```

### เมื่อไรใช้ Tuple

```erlang
%% ใช้ Tuple เมื่อ:
%% 1. ข้อมูลมีโครงสร้างคงที่ที่รู้จำนวน fields
%% 2. ต้องการ pattern matching ที่ทรงพลัง
%% 3. เป็น temporary grouping

%% ตัวอย่าง: Geographic coordinates
Location = {lat, 13.7563, lon, 100.5018}.  %% Bangkok

process_location({lat, Lat, lon, Lon}) ->
    io:format("Lat: ~f, Lon: ~f~n", [Lat, Lon]).

%% ตัวอย่าง: RGB Color
Color = {255, 128, 0}.  %% orange

blend_colors({R1, G1, B1}, {R2, G2, B2}) ->
    {(R1 + R2) div 2, (G1 + G2) div 2, (B1 + B2) div 2}.

%% ตัวอย่าง: Result tuples
divide(_, 0) -> {error, division_by_zero};
divide(A, B) -> {ok, A / B}.

case divide(10, 2) of
    {ok, Result} -> io:format("Result: ~p~n", [Result]);
    {error, Reason} -> io:format("Error: ~p~n", [Reason])
end.
```

---

## Records คืออะไร?

Record คือ named tuple ที่มี field names:

```erlang
%% ปัญหาของ plain tuple:
User1 = {"Alice", 30, "alice@example.com", true}.
%% ไม่รู้ว่า element ที่ 3 คืออะไร ต้องจำเอาเอง!
%% ถ้าเพิ่ม field ใหม่ต้องแก้ทุกที่!

%% แก้ด้วย Record:
-record(user, {name, age, email, active}).
User2 = #user{name = "Alice", age = 30, email = "alice@example.com", active = true}.

%% ดีกว่าเยอะ! Field names ชัดเจน
```

### เปรียบเทียบ

```erlang
%% Tuple:
get_user_name(User) ->
    element(1, User).  %% ไม่ชัดเจน

%% Record:
get_user_name(#user{name = Name}) ->
    Name.  %% ชัดเจนมาก!
```

---

## การสร้าง Record

### Record Definition

```erlang
%% ใน .erl file หรือ .hrl file
%% Format: -record(Name, {Field1 [= Default] [:: Type], ...}).

%% Basic record
-record(point, {x, y}).

%% Record พร้อม defaults
-record(config, {
    host = "localhost",
    port = 8080,
    debug = false,
    timeout = 5000
}).

%% Record พร้อม types
-record(user, {
    id          :: pos_integer(),
    name        :: binary(),
    email       :: binary(),
    age         :: non_neg_integer(),
    role = user :: admin | moderator | user,
    active = true :: boolean(),
    created_at  :: calendar:datetime()
}).

%% Record พร้อม nested types
-record(address, {
    street :: binary(),
    city   :: binary(),
    country :: binary()
}).

-record(person, {
    name    :: binary(),
    age     :: pos_integer(),
    address :: #address{}
}).
```

### Record ใน Header File

```erlang
%% ไฟล์: include/models.hrl
-ifndef(MODELS_HRL).
-define(MODELS_HRL, true).

-record(user, {
    id          :: pos_integer(),
    name        :: binary(),
    email       :: binary(),
    password    :: binary(),
    salt        :: binary(),
    role = user :: admin | moderator | user,
    active = true :: boolean(),
    created_at  :: calendar:datetime(),
    updated_at  :: calendar:datetime()
}).

-record(product, {
    id           :: pos_integer(),
    name         :: binary(),
    description  :: binary(),
    price        :: float(),
    quantity = 0 :: non_neg_integer(),
    category     :: binary(),
    active = true :: boolean()
}).

-record(order, {
    id          :: pos_integer(),
    user_id     :: pos_integer(),
    items       :: [#{}],
    total       :: float(),
    status = pending :: pending | confirmed | shipped | delivered | cancelled,
    created_at  :: calendar:datetime()
}).

-endif.
```

---

## การใช้ Record

### Creating Records

```erlang
%% สร้าง record ด้วย defaults
Config = #config{}.
%% = #config{host="localhost", port=8080, debug=false, timeout=5000}

%% สร้าง record กับ custom values
User = #user{
    id = 1,
    name = <<"Alice">>,
    email = <<"alice@example.com">>,
    age = 30
}.
%% role = user (default)
%% active = true (default)

%% สร้าง record บางส่วน - fields ที่ไม่ระบุจะเป็น undefined
MinimalUser = #user{name = <<"Bob">>}.
%% id = undefined, email = undefined, ...
```

### Accessing Fields

```erlang
User = #user{id = 1, name = <<"Alice">>, email = <<"alice@example.com">>}.

%% Syntax: Record#RecordType.FieldName
User#user.name.   %% = <<"Alice">>
User#user.id.     %% = 1
User#user.email.  %% = <<"alice@example.com">>

%% Pattern matching (ดีกว่า)
#user{name = Name} = User.
Name.  %% = <<"Alice">>
```

### Updating Records

```erlang
User = #user{id = 1, name = <<"Alice">>, email = <<"alice@example.com">>}.

%% สร้าง record ใหม่พร้อม update field
UpdatedUser = User#user{email = <<"new@example.com">>}.
%% = #user{id=1, name=<<"Alice">>, email=<<"new@example.com">>}

%% User ยังคงเป็น record เดิม (immutable!)
User#user.email.        %% = <<"alice@example.com">> (ไม่เปลี่ยน)
UpdatedUser#user.email. %% = <<"new@example.com">>

%% Update หลาย fields พร้อมกัน
DeactivatedUser = User#user{active = false, updated_at = calendar:local_time()}.
```

---

## Record Pattern Matching

```erlang
%% Pattern matching กับ record

%% ใน function clause
get_name(#user{name = Name}) -> Name.
get_email(#user{email = Email}) -> Email.

%% ใน case
describe_role(User) ->
    case User of
        #user{role = admin} -> "Administrator";
        #user{role = moderator} -> "Moderator";
        #user{role = user} -> "Regular User"
    end.

%% ตรวจสอบ active status
process_user(#user{active = true} = User) ->
    do_process(User);
process_user(#user{active = false, name = Name}) ->
    {error, {inactive_user, Name}}.

%% Partial matching - เข้าถึงหลาย fields
format_user(#user{name = Name, email = Email, role = Role}) ->
    io_lib:format("~s (~s) - ~s", [Name, Email, Role]).

%% Alias pattern - เก็บทั้ง record และ fields
process(#user{id = Id} = User) ->
    log(Id),      %% ใช้ Id
    save(User).   %% ใช้ User ทั้งหมด
```

---

## Records กับ Functions

```erlang
-module(user_record).
-include("models.hrl").
-export([create/3, activate/1, deactivate/1, update_email/2, to_map/1]).

%% Create a new user
create(Name, Email, Password) when is_binary(Name), 
                                    is_binary(Email),
                                    is_binary(Password) ->
    Salt = crypto:strong_rand_bytes(16),
    HashedPassword = hash_password(Password, Salt),
    #user{
        name = Name,
        email = Email,
        password = HashedPassword,
        salt = Salt,
        created_at = calendar:local_time(),
        updated_at = calendar:local_time()
    }.

%% Activate user
activate(#user{} = User) ->
    User#user{
        active = true,
        updated_at = calendar:local_time()
    }.

%% Deactivate user
deactivate(#user{} = User) ->
    User#user{
        active = false,
        updated_at = calendar:local_time()
    }.

%% Update email
update_email(#user{} = User, NewEmail) when is_binary(NewEmail) ->
    User#user{
        email = NewEmail,
        updated_at = calendar:local_time()
    }.

%% Convert to map (for serialization)
to_map(#user{id = Id, name = Name, email = Email, role = Role,
              active = Active, created_at = CreatedAt}) ->
    #{
        id => Id,
        name => Name,
        email => Email,
        role => Role,
        active => Active,
        created_at => CreatedAt
    }.

%% Compare users
same_user(#user{id = Id1}, #user{id = Id2}) ->
    Id1 =:= Id2.

%% Filter active users
filter_active(Users) ->
    [U || #user{active = true} = U <- Users].

%%---

hash_password(Password, Salt) ->
    crypto:hash(sha256, <<Password/binary, Salt/binary>>).
```

---

## Nested Records

```erlang
-record(address, {
    street :: binary(),
    city   :: binary(),
    state  :: binary(),
    zip    :: binary(),
    country = <<"TH">> :: binary()
}).

-record(contact_info, {
    phone   :: binary(),
    address :: #address{}
}).

-record(employee, {
    id          :: pos_integer(),
    name        :: binary(),
    email       :: binary(),
    contact     :: #contact_info{},
    department  :: binary(),
    salary      :: float()
}).

%% สร้าง nested record
Employee = #employee{
    id = 1,
    name = <<"Alice Smith">>,
    email = <<"alice@company.com">>,
    contact = #contact_info{
        phone = <<"+66-81-234-5678">>,
        address = #address{
            street = <<"123 Main St">>,
            city = <<"Bangkok">>,
            zip = <<"10100">>
        }
    },
    department = <<"Engineering">>,
    salary = 80000.0
}.

%% เข้าถึง nested fields
Employee#employee.contact#contact_info.phone.
%% = <<"+66-81-234-5678">>

Employee#employee.contact#contact_info.address#address.city.
%% = <<"Bangkok">>

%% Pattern matching กับ nested
get_city(#employee{contact = #contact_info{
    address = #address{city = City}}}) ->
    City.

get_city(Employee).  %% = <<"Bangkok">>

%% Update nested record
update_city(#employee{contact = Contact} = Emp, NewCity) ->
    NewAddress = Contact#contact_info.address#address{city = NewCity},
    NewContact = Contact#contact_info{address = NewAddress},
    Emp#employee{contact = NewContact}.
```

---

## Records vs Maps

เมื่อไรใช้อะไร?

```erlang
%% Records:
%% + ตรวจสอบ field names ณ compile time (dialyzer)
%% + เร็วกว่าในการเข้าถึง fields
%% + pattern matching ที่ชัดเจน
%% - ต้อง define ไว้ล่วงหน้า
%% - ไม่ยืดหยุ่น (ไม่สามารถเพิ่ม field dynamically)
%% - ต้องรู้ record type ณ compile time

%% Maps:
%% + ยืดหยุ่น (เพิ่ม/ลบ keys ได้ runtime)
%% + ไม่ต้อง define ล่วงหน้า
%% + ง่ายกว่าในการ serialize/deserialize
%% - ช้ากว่า records เล็กน้อย
%% - ไม่มี compile-time checking

%% สรุปคำแนะนำ:
%% Records: internal data structures, OTP state, fixed schema
%% Maps: JSON-like data, external interfaces, dynamic schema

%% ตัวอย่าง: OTP GenServer state ใช้ Record
-record(state, {
    db_conn    :: pid(),
    cache      :: ets:table(),
    config     :: map(),
    started_at :: calendar:datetime()
}).

%% API response ใช้ Map
format_user_response(#user{} = User) ->
    #{
        <<"id">> => User#user.id,
        <<"name">> => User#user.name,
        <<"email">> => User#user.email
    }.
```

---

## Record Macros

```erlang
%% Helper macros สำหรับ records

%% ?FIELD macro - เข้าถึง field จาก atom variable
-define(FIELD(Rec, Field), element(
    (#Rec.Field),  %% record field index
    Rec
)).

%% ?UPDATE macro - update field
-define(UPDATE(Rec, Field, Value),
    Rec#record_type{Field = Value}
).

%% record_info BIF
-record(user, {id, name, email}).

record_info(fields, user).
%% = [id, name, email]

record_info(size, user).
%% = 4 (3 fields + 1 for record tag)

%% ตรวจสอบว่าเป็น record ไหม
is_record(#user{}, user).     %% = true
is_record({user, 1, a, b}, user).  %% = true (tuple ที่มี tag)
is_record({other}, user).    %% = false
```

---

## ตัวอย่างจริง: User Management System

```erlang
%% ไฟล์: include/user_models.hrl
-ifndef(USER_MODELS_HRL).
-define(USER_MODELS_HRL, true).

-record(user, {
    id              :: pos_integer() | undefined,
    username        :: binary(),
    email           :: binary(),
    hashed_password :: binary(),
    salt            :: binary(),
    first_name      :: binary(),
    last_name       :: binary(),
    role = user     :: admin | moderator | user,
    active = true   :: boolean(),
    email_verified = false :: boolean(),
    created_at      :: calendar:datetime(),
    updated_at      :: calendar:datetime(),
    last_login      :: calendar:datetime() | undefined
}).

-record(session, {
    token       :: binary(),
    user_id     :: pos_integer(),
    ip_address  :: binary(),
    user_agent  :: binary(),
    created_at  :: calendar:datetime(),
    expires_at  :: calendar:datetime()
}).

-record(password_reset, {
    token       :: binary(),
    user_id     :: pos_integer(),
    created_at  :: calendar:datetime(),
    expires_at  :: calendar:datetime(),
    used = false :: boolean()
}).

-endif.
```

```erlang
%% ไฟล์: src/user_service.erl
-module(user_service).
-include("../include/user_models.hrl").

-export([
    register/4,
    login/2,
    logout/1,
    get_user/1,
    update_profile/3,
    change_password/3,
    deactivate/1,
    verify_email/2
]).

-type registration_params() :: #{
    username := binary(),
    email    := binary(),
    password := binary(),
    first_name := binary(),
    last_name  := binary()
}.

-type error() :: {error, term()}.
-type result(T) :: {ok, T} | error().

%% Register new user
-spec register(binary(), binary(), binary(), map()) -> result(#user{}).
register(Username, Email, Password, Profile) ->
    case validate_registration(Username, Email, Password) of
        ok ->
            case check_username_available(Username) of
                true ->
                    case check_email_available(Email) of
                        true ->
                            create_user(Username, Email, Password, Profile);
                        false ->
                            {error, email_already_taken}
                    end;
                false ->
                    {error, username_already_taken}
            end;
        {error, _} = Error ->
            Error
    end.

%% Login
-spec login(binary(), binary()) -> result(#session{}).
login(Username, Password) ->
    case get_user_by_username(Username) of
        {ok, #user{active = false}} ->
            {error, account_inactive};
        {ok, #user{hashed_password = Hash, salt = Salt} = User} ->
            case verify_password(Password, Hash, Salt) of
                true ->
                    create_session(User);
                false ->
                    {error, invalid_credentials}
            end;
        {error, not_found} ->
            {error, invalid_credentials}
    end.

%% Logout
-spec logout(binary()) -> result(ok).
logout(Token) ->
    delete_session(Token).

%% Get user by ID
-spec get_user(pos_integer()) -> result(#user{}).
get_user(UserId) ->
    user_store:find_by_id(UserId).

%% Update profile
-spec update_profile(pos_integer(), binary(), map()) -> result(#user{}).
update_profile(UserId, RequestUserId, Updates) ->
    case can_update_profile(UserId, RequestUserId) of
        true ->
            case get_user(UserId) of
                {ok, User} ->
                    UpdatedUser = apply_profile_updates(User, Updates),
                    user_store:save(UpdatedUser);
                Error ->
                    Error
            end;
        false ->
            {error, permission_denied}
    end.

%% Change password
-spec change_password(pos_integer(), binary(), binary()) -> result(ok).
change_password(UserId, OldPassword, NewPassword) ->
    case get_user(UserId) of
        {ok, #user{hashed_password = Hash, salt = Salt} = User} ->
            case verify_password(OldPassword, Hash, Salt) of
                true ->
                    {NewHash, NewSalt} = hash_password(NewPassword),
                    UpdatedUser = User#user{
                        hashed_password = NewHash,
                        salt = NewSalt,
                        updated_at = calendar:local_time()
                    },
                    user_store:save(UpdatedUser);
                false ->
                    {error, invalid_current_password}
            end;
        Error ->
            Error
    end.

%% Deactivate account
-spec deactivate(pos_integer()) -> result(ok).
deactivate(UserId) ->
    case get_user(UserId) of
        {ok, User} ->
            UpdatedUser = User#user{
                active = false,
                updated_at = calendar:local_time()
            },
            user_store:save(UpdatedUser);
        Error ->
            Error
    end.

%% Verify email
-spec verify_email(pos_integer(), binary()) -> result(ok).
verify_email(UserId, Token) ->
    case verify_email_token(UserId, Token) of
        ok ->
            case get_user(UserId) of
                {ok, User} ->
                    UpdatedUser = User#user{
                        email_verified = true,
                        updated_at = calendar:local_time()
                    },
                    user_store:save(UpdatedUser);
                Error ->
                    Error
            end;
        Error ->
            Error
    end.

%%--------------------------------------------------------------------
%% Internal Functions
%%--------------------------------------------------------------------

validate_registration(Username, Email, Password) ->
    Validations = [
        {fun() -> validate_username(Username) end, invalid_username},
        {fun() -> validate_email(Email) end, invalid_email},
        {fun() -> validate_password(Password) end, weak_password}
    ],
    run_validations(Validations).

run_validations([]) -> ok;
run_validations([{Fun, Error} | Rest]) ->
    case Fun() of
        true -> run_validations(Rest);
        false -> {error, Error}
    end.

validate_username(Username) ->
    is_binary(Username) andalso
    byte_size(Username) >= 3 andalso
    byte_size(Username) =< 30.

validate_email(Email) ->
    is_binary(Email) andalso
    binary:match(Email, <<"@">>) /= nomatch.

validate_password(Password) ->
    is_binary(Password) andalso byte_size(Password) >= 8.

create_user(Username, Email, Password, Profile) ->
    {Hash, Salt} = hash_password(Password),
    Now = calendar:local_time(),
    User = #user{
        username = Username,
        email = Email,
        hashed_password = Hash,
        salt = Salt,
        first_name = maps:get(first_name, Profile, <<"">>),
        last_name  = maps:get(last_name,  Profile, <<"">>),
        created_at = Now,
        updated_at = Now
    },
    user_store:insert(User).

hash_password(Password) ->
    Salt = crypto:strong_rand_bytes(32),
    Hash = crypto:hash(sha256, <<Password/binary, Salt/binary>>),
    {base64:encode(Hash), base64:encode(Salt)}.

verify_password(Password, StoredHash, StoredSalt) ->
    Salt = base64:decode(StoredSalt),
    Hash = base64:encode(crypto:hash(sha256, <<Password/binary, Salt/binary>>)),
    Hash =:= StoredHash.

create_session(#user{id = UserId}) ->
    Token = base64:encode(crypto:strong_rand_bytes(32)),
    Now = calendar:local_time(),
    Expires = add_hours(Now, 24),
    Session = #session{
        token = Token,
        user_id = UserId,
        created_at = Now,
        expires_at = Expires,
        ip_address = <<>>,
        user_agent = <<>>
    },
    case session_store:insert(Session) of
        ok -> {ok, Session};
        Error -> Error
    end.

can_update_profile(UserId, RequestUserId) ->
    UserId =:= RequestUserId.

apply_profile_updates(User, Updates) ->
    maps:fold(
        fun(first_name, V, U) -> U#user{first_name = V};
           (last_name, V, U) -> U#user{last_name = V};
           (_, _, U) -> U  % ignore unknown fields
        end,
        User#user{updated_at = calendar:local_time()},
        Updates
    ).

check_username_available(Username) ->
    case user_store:find_by_username(Username) of
        {ok, _} -> false;
        {error, not_found} -> true
    end.

check_email_available(Email) ->
    case user_store:find_by_email(Email) of
        {ok, _} -> false;
        {error, not_found} -> true
    end.

get_user_by_username(Username) ->
    user_store:find_by_username(Username).

delete_session(Token) ->
    session_store:delete(Token).

verify_email_token(UserId, Token) ->
    case email_token_store:find(Token) of
        {ok, #{user_id := UserId, expires_at := ExpiresAt}} ->
            case calendar:local_time() < ExpiresAt of
                true -> ok;
                false -> {error, token_expired}
            end;
        _ ->
            {error, invalid_token}
    end.

add_hours({Date, {H, M, S}}, Hours) ->
    TotalSeconds = calendar:datetime_to_gregorian_seconds({Date, {H, M, S}}),
    NewSeconds = TotalSeconds + Hours * 3600,
    calendar:gregorian_seconds_to_datetime(NewSeconds).
```

---

## สรุป Part 9

| Concept | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Tuple | `{a, b, c}` | Fixed structure, pattern matching |
| Record | `#name{field = val}` | Named fields, compile-time checking |
| Record access | `R#type.field` | Get field value |
| Record update | `R#type{field = new}` | Create updated copy |
| Pattern match | `#type{field = Var}` | Destructure in function/case |
| Nested records | `R#t1.f#t2.g` | Access nested fields |

### ก้าวต่อไป

ใน **Part 10** เราจะเรียน Maps อย่างละเอียด:
- Map Operations
- Map Comprehensions
- Nested Maps
- Maps vs Records vs Proplists
- ใช้ Maps ใน Real Applications

---

*[← Part 8: Lists](part_08_lists.md) | [Part 10: Maps →](part_10_maps.md)*
