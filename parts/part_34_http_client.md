# Part 34: HTTP Client

## สารบัญ
1. [httpc — Built-in HTTP Client](#httpc--built-in-http-client)
2. [hackney — Popular HTTP Client](#hackney--popular-http-client)
3. [Gun — HTTP/2 Client](#gun--http2-client)
4. [JSON Handling](#json-handling)
5. [ตัวอย่างจริง: REST API Client](#ตัวอย่างจริง-rest-api-client)

---

## httpc — Built-in HTTP Client

```erlang
%% httpc เป็นส่วนหนึ่งของ inets application (OTP stdlib)
%% ไม่ต้องติดตั้งเพิ่ม

inets:start().
ssl:start().

%% GET request
{ok, {{Version, StatusCode, ReasonPhrase}, Headers, Body}} =
    httpc:request(get, {"https://api.example.com/users", []}, [], []).

%% POST request
{ok, Response} = httpc:request(post,
    {"https://api.example.com/users",
     [{"Content-Type", "application/json"}],
     "application/json",
     <<"{'name':'Alice'}">>},
    [],    %% HTTP options
    []).   %% Request options

%% PUT, DELETE, PATCH
{ok, Response} = httpc:request(put,
    {"https://api.example.com/users/1",
     [{"Authorization", "Bearer token123"}],
     "application/json",
     Body},
    [], []).

{ok, Response} = httpc:request(delete,
    {"https://api.example.com/users/1",
     [{"Authorization", "Bearer token123"}]},
    [], []).

%% Options
HttpOpts = [
    {timeout, 5000},        %% total timeout
    {connect_timeout, 2000}, %% connection timeout
    {ssl, [{verify, verify_peer}, {cacertfile, "/etc/ssl/certs/ca-certificates.crt"}]},
    {autoredirect, true}
].

Opts = [
    {body_format, binary},   %% return body as binary
    {full_result, true}      %% return full response tuple
].

{ok, {{_, 200, _}, _, Body}} =
    httpc:request(get, {Url, Headers}, HttpOpts, Opts).

%% Async request
httpc:request(get, {Url, []}, [], [{sync, false}]).
receive
    {http, {RequestId, Result}} ->
        handle_result(Result)
end.

%% Stream response (large files)
httpc:request(get, {Url, []}, [],
    [{stream, self}]).
%% Receives:
%% {http, {RequestId, stream_start, Headers}}
%% {http, {RequestId, stream, Body}} (multiple)
%% {http, {RequestId, stream_end, Headers}}
```

---

## hackney — Popular HTTP Client

```erlang
%% hackney: popular, feature-rich HTTP client
%% deps: {hackney, "1.18.1"}

%% Simple GET
{ok, StatusCode, Headers, ClientRef} =
    hackney:get("https://api.example.com/users", [], <<>>, []).

{ok, Body} = hackney:body(ClientRef).

%% One-shot (auto-read body)
{ok, StatusCode, _Headers, Body} =
    hackney:get("https://api.example.com/users", [], <<>>,
                [with_body]).

%% POST with JSON
Headers = [{<<"Content-Type">>, <<"application/json">>}],
Body = jsx:encode(#{name => <<"Alice">>, age => 30}),
{ok, 201, _, RespBody} =
    hackney:post("https://api.example.com/users", Headers, Body, [with_body]).

%% Connection pool (reuse connections)
Options = [
    {pool, default},         %% use default pool
    {recv_timeout, 5000},
    {connect_timeout, 2000},
    {follow_redirect, true},
    {max_redirect, 5}
].

{ok, 200, _, Body} =
    hackney:get(Url, [], <<>>, [{with_body} | Options]).

%% Configure pool
hackney_pool:start_pool(my_pool, [
    {timeout, 150000},
    {max_connections, 100}
]).

hackney:get(Url, Headers, Body, [{pool, my_pool}, with_body]).

%% Multipart upload
Parts = [
    {<<"file">>, <<"filename.csv">>,
     {file, "/path/to/file.csv"},
     [{<<"content-type">>, <<"text/csv">>}]},
    {<<"description">>, <<"My file">>}
],
{ok, 200, _, _} = hackney:post(
    Url,
    [],
    {multipart, Parts},
    [with_body]
).

%% Streaming response
{ok, StatusCode, Headers, Ref} =
    hackney:get(Url, [], <<>>, [async]).

stream_response(Ref) ->
    case hackney:stream_body(Ref) of
        {ok, Data} ->
            process_chunk(Data),
            stream_response(Ref);
        done ->
            finish();
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## Gun — HTTP/2 Client

```erlang
%% gun: HTTP/1.1, HTTP/2, WebSocket client
%% deps: {gun, "2.0.1"}

%% Connect
{ok, ConnPid} = gun:open("api.example.com", 443, #{
    transport => tls,
    tls_opts => [{verify, verify_peer}],
    protocols => [http2, http]   %% prefer HTTP/2
}).

%% Wait for connection
{ok, Protocol} = gun:await_up(ConnPid).
io:format("Connected with ~p~n", [Protocol]).

%% GET request
StreamRef = gun:get(ConnPid, "/api/users", [
    {<<"authorization">>, <<"Bearer token123">>}
]).

%% Wait for response
{response, nofin, 200, Headers} = gun:await(ConnPid, StreamRef).
{data, fin, Body} = gun:await(ConnPid, StreamRef).

%% POST request
StreamRef2 = gun:post(ConnPid, "/api/users",
    [{<<"content-type">>, <<"application/json">>}],
    jsx:encode(#{name => <<"Bob">>})
).
{response, nofin, 201, _Headers} = gun:await(ConnPid, StreamRef2).
{data, fin, RespBody} = gun:await(ConnPid, StreamRef2).

%% WebSocket upgrade
StreamRef3 = gun:ws_upgrade(ConnPid, "/ws").
receive
    {gun_upgrade, ConnPid, StreamRef3, [<<"websocket">>], Headers} ->
        io:format("WebSocket ready~n")
end.

%% Send WebSocket message
gun:ws_send(ConnPid, StreamRef3, {text, <<"hello">>}).
gun:ws_send(ConnPid, StreamRef3, {binary, Data}).
gun:ws_send(ConnPid, StreamRef3, close).

%% Receive WebSocket messages
receive
    {gun_ws, ConnPid, StreamRef3, {text, Msg}} ->
        handle_ws_message(Msg);
    {gun_ws, ConnPid, StreamRef3, {binary, Data}} ->
        handle_binary(Data);
    {gun_down, ConnPid, _, _, _} ->
        reconnect()
end.

%% Close connection
gun:close(ConnPid).
```

---

## JSON Handling

```erlang
%% jsx: popular JSON library
%% deps: {jsx, "3.1.0"}

%% Encode
Json = jsx:encode(#{name => <<"Alice">>, age => 30, active => true}).
%% <<"{"active":true,"age":30,"name":"Alice"}">>

jsx:encode([1, 2, <<"three">>]).
%% <<"[1,2,"three"]">>

jsx:encode(#{nested => #{a => 1}}).
%% <<"{"nested":{"a":1}}">>

%% Decode
{ok, Map} = jsx:decode(<<"{"name":"Alice","age":30}">>, [return_maps]).
%% {ok, #{<<"age">> => 30, <<"name">> => <<"Alice">>}}

%% Note: keys เป็น binary เสมอ
maps:get(<<"name">>, Map).  %% <<"Alice">>

%% Decode list
{ok, [1, 2, 3]} = jsx:decode(<<"[1,2,3]">>).

%% Error handling
case jsx:decode(InvalidJson, [return_maps]) of
    {ok, Data} -> handle(Data);
    {error, _} -> {error, invalid_json}
end.

%% json (OTP 27+): built-in JSON module (ไม่ต้อง deps!)
%% Encode
Json = json:encode(#{name => <<"Alice">>, age => 30}).

%% Decode
{ok, Map} = json:decode(Json).

%% thoas: faster, pure Erlang JSON
%% deps: {thoas, "1.0.0"}
thoas:encode(Data).
thoas:decode(Json).
```

---

## ตัวอย่างจริง: REST API Client

```erlang
%% ไฟล์: api_client.erl
-module(api_client).

-export([
    get_users/0, get_user/1,
    create_user/1, update_user/2, delete_user/1,
    get_paginated/2
]).

-define(BASE_URL, "https://jsonplaceholder.typicode.com").
-define(TIMEOUT, 10000).

%% ====== Users API ======

get_users() ->
    request(get, "/users", #{}).

get_user(Id) ->
    request(get, "/users/" ++ integer_to_list(Id), #{}).

create_user(UserData) ->
    request(post, "/users", UserData).

update_user(Id, UserData) ->
    request(put, "/users/" ++ integer_to_list(Id), UserData).

delete_user(Id) ->
    request(delete, "/users/" ++ integer_to_list(Id), #{}).

get_paginated(Resource, Opts) ->
    Page = maps:get(page, Opts, 1),
    Limit = maps:get(limit, Opts, 10),
    Path = "/" ++ Resource ++ "?_page=" ++ integer_to_list(Page) ++
           "&_limit=" ++ integer_to_list(Limit),
    request(get, Path, #{}).

%% ====== Internal ======

request(Method, Path, Body) ->
    Url = ?BASE_URL ++ Path,
    Headers = [
        {"Content-Type", "application/json"},
        {"Accept", "application/json"}
    ],
    HttpOpts = [
        {timeout, ?TIMEOUT},
        {connect_timeout, 5000},
        {ssl, [{verify, verify_none}]}   %% dev only
    ],
    Opts = [{body_format, binary}, {full_result, true}],

    Request = case Method of
        get -> {Url, Headers};
        delete -> {Url, Headers};
        post -> {Url, Headers, "application/json", encode_body(Body)};
        put -> {Url, Headers, "application/json", encode_body(Body)};
        patch -> {Url, Headers, "application/json", encode_body(Body)}
    end,

    case httpc:request(Method, Request, HttpOpts, Opts) of
        {ok, {{_, StatusCode, _}, RespHeaders, RespBody}} ->
            handle_response(StatusCode, RespHeaders, RespBody);
        {error, Reason} ->
            {error, {network_error, Reason}}
    end.

encode_body(#{} = Map) when map_size(Map) =:= 0 -> <<>>;
encode_body(Body) -> jsx:encode(Body).

handle_response(StatusCode, _Headers, Body) when StatusCode >= 200, StatusCode < 300 ->
    case Body of
        <<>> -> {ok, #{}};
        _ ->
            case jsx:decode(Body, [return_maps]) of
                {ok, Data} -> {ok, Data};
                {error, _} -> {ok, Body}  %% return raw if not JSON
            end
    end;

handle_response(404, _, _) ->
    {error, not_found};

handle_response(401, _, _) ->
    {error, unauthorized};

handle_response(429, Headers, _) ->
    RetryAfter = proplists:get_value("retry-after", Headers, "60"),
    {error, {rate_limited, list_to_integer(RetryAfter)}};

handle_response(StatusCode, _, Body) ->
    ErrorMsg = case jsx:decode(Body, [return_maps]) of
        {ok, #{<<"message">> := Msg}} -> Msg;
        _ -> Body
    end,
    {error, {http_error, StatusCode, ErrorMsg}}.


%% ใช้งาน
inets:start(), ssl:start().

%% Get all users
{ok, Users} = api_client:get_users().
length(Users).  %% 10

%% Get specific user
{ok, User} = api_client:get_user(1).
maps:get(<<"name">>, User).  %% <<"Leanne Graham">>

%% Create user
{ok, NewUser} = api_client:create_user(#{
    name => <<"Alice">>,
    email => <<"alice@example.com">>,
    username => <<"alice">>
}).

%% Pagination
{ok, Page1} = api_client:get_paginated("posts", #{page => 1, limit => 5}).

%% Error handling
case api_client:get_user(99999) of
    {ok, User} -> process_user(User);
    {error, not_found} -> io:format("User not found~n");
    {error, Reason} -> io:format("Error: ~p~n", [Reason])
end.
```

---

## สรุป Part 34

| Library | Features | ใช้เมื่อ |
|---------|---------|---------|
| `httpc` | Built-in, sync/async | Simple requests, no deps |
| `hackney` | Pool, streaming, multipart | Production REST APIs |
| `gun` | HTTP/2, WebSocket | Modern APIs, ws |

| JSON Library | Note |
|-------------|------|
| `json` (OTP 27+) | No deps needed |
| `jsx` | Popular, stable |
| `thoas` | Fast, pure Erlang |

---

*[← Part 33: Networking](part_33_networking.md) | [Part 35: HTTP Server with Cowboy →](part_35_cowboy.md)*
