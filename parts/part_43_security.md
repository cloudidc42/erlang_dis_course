# Part 43: Security — การรักษาความปลอดภัย

## สารบัญ
1. [SSL/TLS](#ssltls)
2. [Authentication และ Authorization](#authentication-และ-authorization)
3. [Input Validation](#input-validation)
4. [Cryptography](#cryptography)
5. [Secure Coding Patterns](#secure-coding-patterns)
6. [ตัวอย่างจริง: Secure REST API](#ตัวอย่างจริง-secure-rest-api)

---

## SSL/TLS

```erlang
%% SSL/TLS ใน Erlang ผ่าน ssl module

%% Start SSL app (included in OTP)
ssl:start().

%% TLS Server
start_tls_server() ->
    Options = [
        {certfile, "/etc/ssl/certs/server.crt"},
        {keyfile, "/etc/ssl/private/server.key"},
        {cacertfile, "/etc/ssl/certs/ca.crt"},
        {verify, verify_peer},          %% ต้องการ client cert
        {fail_if_no_peer_cert, true},   %% ปฏิเสธถ้าไม่มี cert
        {versions, ['tlsv1.2', 'tlsv1.3']},
        {ciphers, strong_ciphers()},
        {secure_renegotiate, true},
        {reuse_sessions, true},
        binary,
        {active, false}
    ],
    {ok, ListenSocket} = ssl:listen(8443, Options),
    accept_loop(ListenSocket).

accept_loop(ListenSocket) ->
    {ok, Socket} = ssl:transport_accept(ListenSocket),
    {ok, SslSocket} = ssl:handshake(Socket, 5000),
    Pid = spawn(fun() -> handle_client(SslSocket) end),
    ssl:controlling_process(SslSocket, Pid),
    accept_loop(ListenSocket).

handle_client(Socket) ->
    case ssl:recv(Socket, 0, 10000) of
        {ok, Data} ->
            ssl:send(Socket, process(Data)),
            handle_client(Socket);
        {error, closed} -> ok
    end.

strong_ciphers() ->
    [
        "TLS_AES_256_GCM_SHA384",
        "TLS_CHACHA20_POLY1305_SHA256",
        "TLS_AES_128_GCM_SHA256",
        "ECDHE-RSA-AES256-GCM-SHA384"
    ].

%% TLS Client
connect_tls(Host, Port) ->
    Options = [
        {cacertfile, "/etc/ssl/certs/ca.crt"},
        {verify, verify_peer},
        {server_name_indication, Host},
        {versions, ['tlsv1.2', 'tlsv1.3']},
        binary,
        {active, false}
    ],
    {ok, Socket} = ssl:connect(Host, Port, Options, 5000),
    Socket.

%% Certificate pinning
verify_cert(Cert, ValidCerts) ->
    CertHash = crypto:hash(sha256, Cert),
    lists:member(CertHash, ValidCerts).

%% Cowboy with HTTPS
start_https() ->
    cowboy:start_tls(https_listener, [
        {port, 8443},
        {certfile, "priv/cert.pem"},
        {keyfile, "priv/key.pem"}
    ], #{env => #{dispatch => routes()}}).
```

---

## Authentication และ Authorization

```erlang
%% JWT (JSON Web Token) authentication
%% deps: {joken, "2.6.0"}

-module(auth).
-export([generate_token/1, verify_token/1, require_auth/2]).

-define(SECRET, <<"your-256-bit-secret-from-env">>).
-define(EXPIRY, 3600).  %% 1 hour

generate_token(UserId) ->
    Claims = #{
        <<"sub">> => integer_to_binary(UserId),
        <<"iat">> => erlang:system_time(second),
        <<"exp">> => erlang:system_time(second) + ?EXPIRY,
        <<"jti">> => base64:encode(crypto:strong_rand_bytes(16))
    },
    Signer = joken_config:default_generated_claims(Claims),
    {ok, Token, _} = Joken.encode_and_sign(Claims, Signer),
    Token.

verify_token(Token) ->
    Verifier = joken:verify(Token, ?SECRET, []),
    case Verifier of
        {ok, Claims} ->
            Exp = maps:get(<<"exp">>, Claims, 0),
            Now = erlang:system_time(second),
            case Now < Exp of
                true -> {ok, Claims};
                false -> {error, token_expired}
            end;
        {error, _} ->
            {error, invalid_token}
    end.

%% Cowboy middleware for authentication
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            case auth:verify_token(Token) of
                {ok, Claims} ->
                    {ok, Req, Env#{user_claims => Claims}};
                {error, Reason} ->
                    Body = jsx:encode(#{error => Reason}),
                    Req2 = cowboy_req:reply(401,
                        #{<<"content-type">> => <<"application/json">>},
                        Body, Req),
                    {stop, Req2}
            end;
        _ ->
            Req2 = cowboy_req:reply(401,
                #{<<"www-authenticate">> => <<"Bearer">>},
                <<"Unauthorized">>, Req),
            {stop, Req2}
    end.

%% Role-based access control
require_role(Req, Role, Env) ->
    Claims = maps:get(user_claims, Env, #{}),
    Roles = maps:get(<<"roles">>, Claims, []),
    case lists:member(Role, Roles) of
        true -> ok;
        false -> {error, forbidden}
    end.
```

---

## Input Validation

```erlang
%% Input validation: ป้องกัน injection attacks

-module(validator).
-export([validate/2, sanitize_html/1]).

%% Validate with schema
validate(Data, Schema) ->
    maps:fold(fun(Key, Rules, Acc) ->
        Value = maps:get(Key, Data, undefined),
        case validate_field(Key, Value, Rules) of
            ok -> Acc;
            {error, Msg} -> [{Key, Msg} | Acc]
        end
    end, [], Schema).

validate_field(Key, undefined, Rules) ->
    case lists:member(required, Rules) of
        true -> {error, <<"is required">>};
        false -> ok
    end;
validate_field(Key, Value, Rules) ->
    lists:foldl(fun
        (_, {error, _} = E) -> E;
        (required, ok) -> ok;
        ({type, integer}, ok) when is_integer(Value) -> ok;
        ({type, integer}, ok) -> {error, <<"must be integer">>};
        ({type, binary}, ok) when is_binary(Value) -> ok;
        ({type, binary}, ok) -> {error, <<"must be string">>};
        ({min, Min}, ok) when is_integer(Value), Value >= Min -> ok;
        ({min, Min}, ok) when is_integer(Value) -> 
            {error, iolist_to_binary(["must be >= ", integer_to_list(Min)])};
        ({max, Max}, ok) when is_integer(Value), Value =< Max -> ok;
        ({max, Max}, ok) when is_integer(Value) -> 
            {error, iolist_to_binary(["must be <= ", integer_to_list(Max)])};
        ({min_length, Min}, ok) when is_binary(Value), byte_size(Value) >= Min -> ok;
        ({min_length, Min}, ok) when is_binary(Value) -> {error, <<"too short">>};
        ({max_length, Max}, ok) when is_binary(Value), byte_size(Value) =< Max -> ok;
        ({max_length, Max}, ok) when is_binary(Value) -> {error, <<"too long">>};
        ({regex, Pattern}, ok) when is_binary(Value) ->
            case re:run(Value, Pattern) of
                {match, _} -> ok;
                nomatch -> {error, <<"invalid format">>}
            end;
        (_, Acc) -> Acc
    end, ok, Rules).

%% SQL injection prevention: use parameterized queries ONLY
%% Never do:
bad_query(Username) ->
    Query = "SELECT * FROM users WHERE name = '" ++ Username ++ "'",
    db:query(Query).  %% SQL INJECTION!

%% Always do:
good_query(Username) ->
    db:query("SELECT * FROM users WHERE name = $1", [Username]).

%% HTML sanitization
sanitize_html(Input) ->
    %% Escape HTML entities
    lists:foldl(fun({From, To}, Acc) ->
        binary:replace(Acc, From, To, [global])
    end, Input, [
        {<<"&">>, <<"&amp;">>},
        {<<"<">>, <<"&lt;">>},
        {<<">">>, <<"&gt;">>},
        {<<"\"">>, <<"&quot;">>},
        {<<"'">>, <<"&#x27;">>}
    ]).

%% Path traversal prevention
safe_path(Base, UserPath) ->
    Normalized = filename:nativename(
        filename:join(Base, filename:basename(UserPath))
    ),
    case string:prefix(Normalized, Base) of
        nomatch -> {error, path_traversal};
        _ -> {ok, Normalized}
    end.
```

---

## Cryptography

```erlang
%% crypto module: built-in cryptographic functions

%% Hashing
hash_password(Password) ->
    Salt = crypto:strong_rand_bytes(16),
    Hash = crypto:hash(sha256, [Salt, Password]),
    {base64:encode(Salt), base64:encode(Hash)}.

verify_password(Password, SaltB64, HashB64) ->
    Salt = base64:decode(SaltB64),
    ExpectedHash = base64:decode(HashB64),
    ActualHash = crypto:hash(sha256, [Salt, Password]),
    crypto:hash_equals(ActualHash, ExpectedHash).  %% constant time compare

%% Better: use Argon2/bcrypt/PBKDF2 for password hashing
%% {comeonin_bcrypt, "3.1.0"} or {argon2_elixir, ...}

%% HMAC: message authentication
sign_data(Data, Key) ->
    crypto:mac(hmac, sha256, Key, Data).

verify_signature(Data, Key, Signature) ->
    Expected = crypto:mac(hmac, sha256, Key, Data),
    crypto:hash_equals(Expected, Signature).

%% Symmetric encryption
encrypt(Plaintext, Key) ->
    IV = crypto:strong_rand_bytes(16),
    {Ciphertext, Tag} = crypto:crypto_one_time_aead(
        aes_256_gcm, Key, IV, Plaintext, <<>>, true
    ),
    {IV, Ciphertext, Tag}.

decrypt({IV, Ciphertext, Tag}, Key) ->
    crypto:crypto_one_time_aead(
        aes_256_gcm, Key, IV, Ciphertext, <<>>, Tag, false
    ).

%% Asymmetric: RSA
generate_keypair() ->
    {PublicKey, PrivateKey} = crypto:generate_key(rsa, {2048, 65537}),
    {PublicKey, PrivateKey}.

rsa_sign(Data, PrivateKey) ->
    public_key:sign(Data, sha256, PrivateKey).

rsa_verify(Data, Signature, PublicKey) ->
    public_key:verify(Data, sha256, Signature, PublicKey).

%% Secure random
secure_random_bytes(N) ->
    crypto:strong_rand_bytes(N).

secure_random_string(N) ->
    base64url:encode(crypto:strong_rand_bytes(N)).

%% Key derivation
derive_key(Password, Salt, Iterations, KeyLen) ->
    {ok, Key} = pbkdf2:pbkdf2(sha256, Password, Salt, Iterations, KeyLen),
    Key.
```

---

## Secure Coding Patterns

```erlang
%% 1. Secrets: ไม่เก็บใน source code หรือ config files
get_secret(Name) ->
    case os:getenv(Name) of
        false -> error({missing_env_var, Name});
        Value -> list_to_binary(Value)
    end.

db_password() -> get_secret("DB_PASSWORD").
jwt_secret() -> get_secret("JWT_SECRET").

%% 2. Rate limiting: ป้องกัน brute force
-module(rate_limiter).
-behaviour(gen_server).

init(_) ->
    {ok, #{attempts => #{}}}.

handle_call({check, Key}, _From, #{attempts := Attempts} = State) ->
    Now = erlang:system_time(second),
    Window = maps:get(Key, Attempts, []),
    Recent = [T || T <- Window, Now - T < 300],  %% 5-minute window
    case length(Recent) >= 10 of
        true ->
            {reply, {error, rate_limited}, State};
        false ->
            NewWindow = [Now | Recent],
            NewAttempts = maps:put(Key, NewWindow, Attempts),
            {reply, ok, State#{attempts := NewAttempts}}
    end.

%% 3. Sensitive data: ไม่ log passwords, tokens
safe_log(#{password := _, token := _, credit_card := _} = Data) ->
    SafeData = maps:without([password, token, credit_card, cvv], Data),
    logger:info("Request", SafeData);
safe_log(Data) ->
    logger:info("Request", Data).

%% 4. Session management
create_session(UserId) ->
    SessionId = base64url:encode(crypto:strong_rand_bytes(32)),
    Expires = erlang:system_time(second) + 86400,  %% 24h
    ets:insert(sessions, {SessionId, UserId, Expires}),
    SessionId.

validate_session(SessionId) ->
    case ets:lookup(sessions, SessionId) of
        [{_, UserId, Expires}] ->
            case erlang:system_time(second) < Expires of
                true -> {ok, UserId};
                false ->
                    ets:delete(sessions, SessionId),
                    {error, expired}
            end;
        [] -> {error, invalid_session}
    end.

%% 5. CSRF protection
generate_csrf_token() ->
    base64url:encode(crypto:strong_rand_bytes(32)).

verify_csrf(RequestToken, SessionToken) ->
    crypto:hash_equals(RequestToken, SessionToken).
```

---

## ตัวอย่างจริง: Secure REST API

```erlang
%% secure_api.erl: REST API with auth, validation, rate limiting

-module(users_handler).
-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    handle(Method, Req, State).

%% POST /api/login
handle(<<"POST">>, Req, #{route := login} = State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case jsx:decode(Body, [return_maps]) of
        #{<<"username">> := Username, <<"password">> := Password} ->
            case rate_limiter:check(Username) of
                ok ->
                    case auth:login(Username, Password) of
                        {ok, UserId} ->
                            Token = auth:generate_token(UserId),
                            reply(200, #{token => Token}, Req2, State);
                        {error, invalid_credentials} ->
                            timer:sleep(1000),  %% slow down brute force
                            reply(401, #{error => <<"Invalid credentials">>}, Req2, State)
                    end;
                {error, rate_limited} ->
                    reply(429, #{error => <<"Too many attempts">>}, Req2, State)
            end;
        _ ->
            reply(400, #{error => <<"Invalid request">>}, Req2, State)
    end;

%% GET /api/users/:id (requires auth)
handle(<<"GET">>, Req, #{route := user} = State) ->
    case get_authenticated_user(Req) of
        {ok, Claims} ->
            UserId = binary_to_integer(cowboy_req:binding(id, Req)),
            case users:get(UserId) of
                {ok, User} ->
                    reply(200, sanitize_user(User), Req, State);
                {error, not_found} ->
                    reply(404, #{error => <<"Not found">>}, Req, State)
            end;
        {error, _} ->
            reply(401, #{error => <<"Unauthorized">>}, Req, State)
    end.

get_authenticated_user(Req) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> -> auth:verify_token(Token);
        _ -> {error, no_token}
    end.

sanitize_user(User) ->
    maps:without([password_hash, salt, internal_notes], User).

reply(Status, Body, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Body), Req),
    {ok, Req2, State}.
```

---

## สรุป Part 43

| Threat | Mitigation |
|--------|-----------|
| SQL Injection | Parameterized queries |
| XSS | HTML escaping, CSP headers |
| CSRF | CSRF tokens |
| Brute force | Rate limiting + timing |
| Replay attacks | JWT exp + jti |
| Secrets exposure | Env vars, never log |
| TLS downgrade | Force TLSv1.2+ |
| Path traversal | Normalize + prefix check |

---

*[← Part 42: NIFs และ Ports](part_42_nifs_ports.md) | [Part 44: gRPC →](part_44_grpc.md)*
