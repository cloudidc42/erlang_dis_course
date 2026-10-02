# Part 79: Security Hardening

## สารบัญ
1. [Input Validation and Sanitization](#validation)
2. [Authentication Patterns](#authentication)
3. [Authorization (RBAC/ABAC)](#authorization)
4. [Cryptography](#cryptography)
5. [Rate Limiting and DDoS Protection](#rate-limiting)
6. [ตัวอย่างจริง: Secure API Server](#secure-api)

---

## Input Validation and Sanitization

```erlang
%% validator.erl — comprehensive input validation

-module(validator).
-export([validate/2, sanitize_string/1, validate_email/1, validate_uuid/1]).

%% Schema-based validation
validate(Data, Schema) ->
    validate_fields(maps:to_list(Schema), Data, []).

validate_fields([], _Data, []) -> {ok, validated};
validate_fields([], _Data, Errors) -> {error, Errors};
validate_fields([{Field, Rules} | Rest], Data, Errors) ->
    Value = maps:get(Field, Data, undefined),
    case validate_value(Value, Rules, Field) of
        ok -> validate_fields(Rest, Data, Errors);
        {error, Error} -> validate_fields(Rest, Data, [Error | Errors])
    end.

validate_value(undefined, #{required := true}, Field) ->
    {error, #{field => Field, error => required}};
validate_value(undefined, _, _) -> ok;
validate_value(Value, Rules, Field) ->
    checks(Value, Rules, Field, [
        {type, fun check_type/2},
        {min_length, fun check_min_length/2},
        {max_length, fun check_max_length/2},
        {pattern, fun check_pattern/2},
        {min, fun check_min/2},
        {max, fun check_max/2},
        {enum, fun check_enum/2}
    ]).

checks(_, _, _, []) -> ok;
checks(Value, Rules, Field, [{RuleKey, CheckFun} | Rest]) ->
    case maps:get(RuleKey, Rules, undefined) of
        undefined -> checks(Value, Rules, Field, Rest);
        RuleVal ->
            case CheckFun(Value, RuleVal) of
                ok -> checks(Value, Rules, Field, Rest);
                {error, Msg} -> {error, #{field => Field, error => Msg, rule => RuleKey}}
            end
    end.

check_type(Value, string) when is_binary(Value) -> ok;
check_type(Value, integer) when is_integer(Value) -> ok;
check_type(Value, float) when is_float(Value) orelse is_integer(Value) -> ok;
check_type(Value, boolean) when is_boolean(Value) -> ok;
check_type(Value, list) when is_list(Value) -> ok;
check_type(Value, map) when is_map(Value) -> ok;
check_type(_, Type) -> {error, {invalid_type, Type}}.

check_min_length(Value, Min) when is_binary(Value), byte_size(Value) >= Min -> ok;
check_min_length(_, Min) -> {error, {too_short, Min}}.

check_max_length(Value, Max) when is_binary(Value), byte_size(Value) =< Max -> ok;
check_max_length(_, Max) -> {error, {too_long, Max}}.

check_pattern(Value, Pattern) when is_binary(Value) ->
    case re:run(Value, Pattern) of
        {match, _} -> ok;
        nomatch -> {error, pattern_mismatch}
    end.

check_min(Value, Min) when is_number(Value), Value >= Min -> ok;
check_min(_, Min) -> {error, {below_min, Min}}.

check_max(Value, Max) when is_number(Value), Value =< Max -> ok;
check_max(_, Max) -> {error, {above_max, Max}}.

check_enum(Value, Allowed) ->
    case lists:member(Value, Allowed) of
        true -> ok;
        false -> {error, {not_in_enum, Allowed}}
    end.

%% String sanitization — prevent XSS
sanitize_string(Str) when is_binary(Str) ->
    Str1 = binary:replace(Str, <<"<">>, <<"&lt;">>, [global]),
    Str2 = binary:replace(Str1, <<">">>, <<"&gt;">>, [global]),
    Str3 = binary:replace(Str2, <<"\"">>, <<"&quot;">>, [global]),
    binary:replace(Str3, <<"'">>, <<"&#x27;">>, [global]).

validate_email(Email) when is_binary(Email) ->
    case re:run(Email, "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$") of
        {match, _} -> ok;
        nomatch -> {error, invalid_email}
    end.

validate_uuid(UUID) when is_binary(UUID) ->
    case re:run(UUID, "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
                [{capture, none}, caseless]) of
        match -> ok;
        nomatch -> {error, invalid_uuid}
    end.

%% SQL injection prevention — use parameterized queries
safe_query_example() ->
    UserId = get_user_input(),
    %% NEVER: "SELECT * FROM users WHERE id = " ++ UserId
    %% ALWAYS: parameterized query
    epgsql:equery(db_conn(), "SELECT * FROM users WHERE id = $1", [UserId]).
```

---

## Authentication Patterns

```erlang
%% auth.erl — JWT authentication

-module(auth).
-export([generate_token/2, verify_token/1, refresh_token/1]).

-define(JWT_SECRET, os:getenv("JWT_SECRET")).
-define(ACCESS_TOKEN_TTL, 3600).    %% 1 hour
-define(REFRESH_TOKEN_TTL, 604800). %% 7 days

generate_token(UserId, Roles) ->
    Now = erlang:system_time(second),
    
    Claims = #{
        <<"sub">> => UserId,
        <<"roles">> => Roles,
        <<"iat">> => Now,
        <<"exp">> => Now + ?ACCESS_TOKEN_TTL,
        <<"jti">> => generate_jti()  %% unique token ID
    },
    
    %% Sign with HMAC-SHA256
    Header = #{<<"alg">> => <<"HS256">>, <<"typ">> => <<"JWT">>},
    
    HeaderB64 = base64url_encode(jsx:encode(Header)),
    PayloadB64 = base64url_encode(jsx:encode(Claims)),
    
    Signature = crypto:mac(hmac, sha256, ?JWT_SECRET,
                           <<HeaderB64/binary, ".", PayloadB64/binary>>),
    SigB64 = base64url_encode(Signature),
    
    {ok, <<HeaderB64/binary, ".", PayloadB64/binary, ".", SigB64/binary>>}.

verify_token(Token) ->
    case binary:split(Token, <<".">>, [global]) of
        [HeaderB64, PayloadB64, SigB64] ->
            %% Verify signature
            ExpectedSig = crypto:mac(hmac, sha256, ?JWT_SECRET,
                                     <<HeaderB64/binary, ".", PayloadB64/binary>>),
            
            case base64url_encode(ExpectedSig) =:= SigB64 of
                false ->
                    {error, invalid_signature};
                true ->
                    %% Decode payload
                    Claims = jsx:decode(base64url_decode(PayloadB64), [return_maps]),
                    
                    %% Check expiry
                    Exp = maps:get(<<"exp">>, Claims, 0),
                    Now = erlang:system_time(second),
                    
                    case Now < Exp of
                        false -> {error, token_expired};
                        true ->
                            %% Check if token is revoked
                            Jti = maps:get(<<"jti">>, Claims),
                            case token_blacklist:is_revoked(Jti) of
                                true -> {error, token_revoked};
                                false -> {ok, Claims}
                            end
                    end
            end;
        _ ->
            {error, malformed_token}
    end.

revoke_token(Token) ->
    case verify_token(Token) of
        {ok, Claims} ->
            Jti = maps:get(<<"jti">>, Claims),
            Exp = maps:get(<<"exp">>, Claims),
            TTL = Exp - erlang:system_time(second),
            token_blacklist:add(Jti, TTL);
        Error ->
            Error
    end.

%% Token blacklist using Redis
-module(token_blacklist).
-export([add/2, is_revoked/1]).

add(Jti, TTL) ->
    eredis:q(redis_conn(), ["SETEX", <<"jwt:revoked:", Jti/binary>>, TTL, <<"1">>]).

is_revoked(Jti) ->
    case eredis:q(redis_conn(), ["EXISTS", <<"jwt:revoked:", Jti/binary>>]) of
        {ok, <<"1">>} -> true;
        _ -> false
    end.

base64url_encode(Data) ->
    B64 = base64:encode(Data),
    B64_1 = binary:replace(B64, <<"+">>, <<"-">>, [global]),
    B64_2 = binary:replace(B64_1, <<"/">>, <<"_">>, [global]),
    re:replace(B64_2, "=+$", "", [{return, binary}]).

base64url_decode(B64) ->
    B64_1 = binary:replace(B64, <<"-">>, <<"+">>, [global]),
    B64_2 = binary:replace(B64_1, <<"_">>, <<"/">>, [global]),
    Padding = case byte_size(B64_2) rem 4 of
        0 -> <<>>;
        2 -> <<"==">>;
        3 -> <<"=">>
    end,
    base64:decode(<<B64_2/binary, Padding/binary>>).

generate_jti() ->
    base64url_encode(crypto:strong_rand_bytes(16)).
```

---

## Authorization (RBAC)

```erlang
%% rbac.erl — Role-Based Access Control

-module(rbac).
-export([authorize/3, has_permission/3, get_user_roles/1]).

%% Permission structure:
%% {resource, action} — e.g., {orders, read}, {users, delete}

%% Role permissions map
role_permissions() ->
    #{
        admin => [
            {users, read}, {users, write}, {users, delete},
            {orders, read}, {orders, write}, {orders, delete},
            {reports, read}
        ],
        manager => [
            {users, read},
            {orders, read}, {orders, write},
            {reports, read}
        ],
        customer => [
            {orders, read}
        ],
        readonly => [
            {orders, read},
            {users, read}
        ]
    }.

has_permission(UserId, Resource, Action) ->
    Roles = get_user_roles(UserId),
    RolePerms = role_permissions(),
    
    lists:any(fun(Role) ->
        Perms = maps:get(Role, RolePerms, []),
        lists:member({Resource, Action}, Perms)
    end, Roles).

authorize(UserId, Resource, Action) ->
    case has_permission(UserId, Resource, Action) of
        true -> ok;
        false ->
            logger:warning("Authorization denied", #{
                user_id => UserId,
                resource => Resource,
                action => Action
            }),
            {error, forbidden}
    end.

get_user_roles(UserId) ->
    case user_store:get_roles(UserId) of
        {ok, Roles} -> Roles;
        _ -> []
    end.

%% Cowboy middleware for authorization
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    Token = extract_token(Req),
    
    case auth:verify_token(Token) of
        {ok, Claims} ->
            UserId = maps:get(<<"sub">>, Claims),
            Roles = maps:get(<<"roles">>, Claims, []),
            
            %% Inject claims into request
            Req2 = cowboy_req:set_resp_header(<<"x-user-id">>, UserId, Req),
            NewEnv = Env#{auth_claims => Claims, user_id => UserId, roles => Roles},
            
            {ok, Req2, NewEnv};
        
        {error, Reason} ->
            Body = jsx:encode(#{error => <<"unauthorized">>, reason => Reason}),
            Req2 = cowboy_req:reply(401,
                #{<<"content-type">> => <<"application/json">>,
                  <<"www-authenticate">> => <<"Bearer">>},
                Body, Req),
            {stop, Req2}
    end.

extract_token(Req) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> -> Token;
        _ -> undefined
    end.
```

---

## Cryptography

```erlang
%% crypto_utils.erl — cryptography helpers

-module(crypto_utils).
-export([hash_password/1, verify_password/2, encrypt/2, decrypt/2,
         generate_key/0, hmac_sign/2, hmac_verify/3]).

%% Password hashing with Argon2 (via NIF) or PBKDF2
hash_password(Password) when is_binary(Password) ->
    %% Using PBKDF2-SHA256 (built into Erlang crypto)
    Salt = crypto:strong_rand_bytes(32),
    Iterations = 600000,  %% OWASP recommended 2024
    
    Hash = crypto:pbkdf2_hmac(sha256, Password, Salt, Iterations, 32),
    
    %% Store as: algorithm$iterations$base64(salt)$base64(hash)
    iolist_to_binary([
        "pbkdf2-sha256$",
        integer_to_binary(Iterations), "$",
        base64:encode(Salt), "$",
        base64:encode(Hash)
    ]).

verify_password(Password, StoredHash) ->
    case binary:split(StoredHash, <<"$">>, [global]) of
        [<<"pbkdf2-sha256">>, IterBin, SaltB64, HashB64] ->
            Iterations = binary_to_integer(IterBin),
            Salt = base64:decode(SaltB64),
            ExpectedHash = base64:decode(HashB64),
            
            ActualHash = crypto:pbkdf2_hmac(sha256, Password, Salt, Iterations, 32),
            
            %% Constant-time comparison to prevent timing attacks
            crypto:hash_equals(ActualHash, ExpectedHash);
        _ ->
            false
    end.

%% AES-256-GCM encryption (authenticated encryption)
generate_key() ->
    crypto:strong_rand_bytes(32).

encrypt(Plaintext, Key) when is_binary(Plaintext), byte_size(Key) =:= 32 ->
    IV = crypto:strong_rand_bytes(12),    %% 96-bit IV for GCM
    AAD = <<>>,                            %% Additional authenticated data
    
    {Ciphertext, Tag} = crypto:crypto_one_time_aead(
        aes_256_gcm, Key, IV, Plaintext, AAD, true
    ),
    
    %% Return: IV || Tag || Ciphertext (all base64 encoded)
    #{
        iv => base64:encode(IV),
        tag => base64:encode(Tag),
        ciphertext => base64:encode(Ciphertext)
    }.

decrypt(#{iv := IVB64, tag := TagB64, ciphertext := CiphertextB64}, Key) ->
    IV = base64:decode(IVB64),
    Tag = base64:decode(TagB64),
    Ciphertext = base64:decode(CiphertextB64),
    AAD = <<>>,
    
    case crypto:crypto_one_time_aead(
        aes_256_gcm, Key, IV, Ciphertext, AAD, Tag, false
    ) of
        error -> {error, decryption_failed};
        Plaintext -> {ok, Plaintext}
    end.

%% HMAC signing for webhooks and API signatures
hmac_sign(Payload, Secret) ->
    Mac = crypto:mac(hmac, sha256, Secret, Payload),
    <<"sha256=", (base16_encode(Mac))/binary>>.

hmac_verify(Payload, Signature, Secret) ->
    Expected = hmac_sign(Payload, Secret),
    %% Constant-time comparison
    crypto:hash_equals(Signature, Expected).

base16_encode(Bin) ->
    << <<(hex_char(H)), (hex_char(L))>> || <<H:4, L:4>> <= Bin >>.

hex_char(N) when N < 10 -> $0 + N;
hex_char(N) -> $a + N - 10.

%% Secure random token generation
generate_token(LengthBytes) ->
    base64:encode(crypto:strong_rand_bytes(LengthBytes)).

%% Encrypt PII fields at application level
encrypt_pii(UserData) ->
    EncKey = get_encryption_key(),
    Fields = [email, phone, ssn],
    
    lists:foldl(fun(Field, Data) ->
        case maps:get(Field, Data, undefined) of
            undefined -> Data;
            Value ->
                Encrypted = encrypt(Value, EncKey),
                Data#{Field => Encrypted}
        end
    end, UserData, Fields).
```

---

## Rate Limiting and DDoS Protection

```erlang
%% rate_limiter.erl — token bucket algorithm with Redis

-module(rate_limiter).
-export([check/3, check_and_increment/3]).

%% Rate limit: N requests per T seconds
%% Returns: {ok, remaining} | {error, rate_limited}

check(Key, MaxRequests, WindowSeconds) ->
    Now = erlang:system_time(second),
    WindowStart = Now - WindowSeconds,
    
    Pipeline = [
        %% Remove old entries
        ["ZREMRANGEBYSCORE", Key, "-inf", WindowStart],
        %% Count current window
        ["ZCARD", Key],
        %% Add current request timestamp
        ["ZADD", Key, Now, make_request_id()],
        %% Set expiry
        ["EXPIRE", Key, WindowSeconds * 2]
    ],
    
    case eredis:qp(redis_conn(), Pipeline) of
        [{ok, _}, {ok, CountBin}, {ok, _}, {ok, _}] ->
            Count = binary_to_integer(CountBin),
            Remaining = MaxRequests - Count,
            
            case Remaining >= 0 of
                true -> {ok, Remaining};
                false -> {error, rate_limited}
            end;
        _ ->
            {ok, MaxRequests}  %% Redis error → allow (fail open)
    end.

%% Cowboy middleware for rate limiting
-module(rate_limit_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

-define(GLOBAL_LIMIT, 1000).   %% per IP
-define(AUTH_LIMIT, 5).         %% login attempts per minute
-define(WINDOW, 60).            %% 60 seconds

execute(Req, Env) ->
    IP = get_client_ip(Req),
    Path = cowboy_req:path(Req),
    
    {Limit, Window} = rate_limit_for_path(Path),
    Key = rate_key(IP, Path),
    
    case rate_limiter:check(Key, Limit, Window) of
        {ok, Remaining} ->
            Req2 = cowboy_req:set_resp_headers(#{
                <<"x-ratelimit-limit">> => integer_to_binary(Limit),
                <<"x-ratelimit-remaining">> => integer_to_binary(Remaining),
                <<"x-ratelimit-reset">> => integer_to_binary(erlang:system_time(second) + Window)
            }, Req),
            {ok, Req2, Env};
        
        {error, rate_limited} ->
            logger:warning("Rate limit exceeded", #{ip => IP, path => Path}),
            metrics:counter(rate_limited_requests, 1),
            
            Body = jsx:encode(#{error => <<"Too Many Requests">>}),
            Req2 = cowboy_req:reply(429,
                #{<<"content-type">> => <<"application/json">>,
                  <<"retry-after">> => integer_to_binary(Window)},
                Body, Req),
            {stop, Req2}
    end.

rate_limit_for_path(<<"/auth/login">>) -> {?AUTH_LIMIT, 60};
rate_limit_for_path(<<"/auth/", _/binary>>) -> {20, 60};
rate_limit_for_path(_) -> {?GLOBAL_LIMIT, 60}.

rate_key(IP, Path) ->
    <<"rl:", IP/binary, ":", Path/binary>>.

get_client_ip(Req) ->
    %% Get real IP when behind proxy
    case cowboy_req:header(<<"x-forwarded-for">>, Req) of
        undefined -> format_ip(cowboy_req:peer(Req));
        XFF -> hd(binary:split(XFF, <<",">>))
    end.

format_ip({IP, _Port}) ->
    list_to_binary(inet:ntoa(IP)).

make_request_id() ->
    integer_to_binary(erlang:unique_integer([positive])).
```

---

## สรุป Part 79

| Security Concern | Solution |
|-----------------|---------|
| Input validation | Schema-based validator |
| SQL injection | Parameterized queries |
| XSS | HTML entity escaping |
| Authentication | JWT + token blacklist |
| Authorization | RBAC middleware |
| Password storage | PBKDF2-SHA256 |
| Data encryption | AES-256-GCM |
| Rate limiting | Redis sliding window |
| HMAC signing | crypto:mac/4 |

**OWASP Top 10 ป้องกัน**: Input validation + parameterized queries + proper auth + rate limiting = ครอบคลุม 80% ของช่องโหว่

---

*[← Part 78: ML Serving](part_78_ml_serving.md) | [Part 80: Performance Optimization →](part_80_performance.md)*
