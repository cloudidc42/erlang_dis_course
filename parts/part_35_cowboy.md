# Part 35: HTTP Server ด้วย Cowboy

## สารบัญ
1. [Cowboy คืออะไร](#cowboy-คืออะไร)
2. [Setup และ Routing](#setup-และ-routing)
3. [Handler Types](#handler-types)
4. [REST Handler](#rest-handler)
5. [WebSocket Handler](#websocket-handler)
6. [Middleware และ Hooks](#middleware-และ-hooks)
7. [ตัวอย่างจริง: JSON REST API](#ตัวอย่างจริง-json-rest-api)

---

## Cowboy คืออะไร

```erlang
%% Cowboy: HTTP server สำหรับ Erlang
%% - HTTP/1.1, HTTP/2, WebSocket
%% - High performance, low overhead
%% - เป็น standard ใน Erlang/Elixir ecosystem

%% rebar.config
{deps, [
    {cowboy, "2.10.0"},
    {jsx, "3.1.0"}
]}.

%% src/my_app.app.src: applications
{applications, [kernel, stdlib, cowboy]}.
```

---

## Setup และ Routing

```erlang
%% ไฟล์: src/my_app_app.erl
start(normal, _) ->
    %% Define routes
    Dispatch = cowboy_router:compile([
        {'_', [                         %% match any host
            {"/", root_handler, []},
            {"/api/users", users_handler, []},
            {"/api/users/:id", user_handler, []},
            {"/api/users/:id/posts", user_posts_handler, []},
            {"/ws", ws_handler, []},
            {"/static/[...]", cowboy_static, {priv_dir, my_app, "static"}}
        ]}
    ]),

    %% Start HTTP server
    {ok, _} = cowboy:start_clear(http_listener,
        [{port, 8080}],
        #{env => #{dispatch => Dispatch}}
    ),

    %% Start HTTPS server
    {ok, _} = cowboy:start_tls(https_listener,
        [
            {port, 8443},
            {certfile, "priv/cert.pem"},
            {keyfile, "priv/key.pem"}
        ],
        #{env => #{dispatch => Dispatch}}
    ),

    my_app_sup:start_link().

%% Route patterns
%% "/api/users"           - exact match
%% "/api/users/:id"       - binding, req:binding(id, Req)
%% "/api/:resource/[:id]" - optional binding
%% "/static/[...]"        - wildcard path
%% "/api/:_/info"         - unnamed binding (ignore value)
```

---

## Handler Types

```erlang
%% 1. Plain Handler (handle everything manually)
-module(plain_handler).

-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    Path = cowboy_req:path(Req),
    io:format("~s ~s~n", [Method, Path]),

    Resp = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain">>},
        <<"Hello World!">>,
        Req),
    {ok, Resp, State}.

%% 2. Reading request
init(Req0, State) ->
    %% Headers
    ContentType = cowboy_req:header(<<"content-type">>, Req0),
    Auth = cowboy_req:header(<<"authorization">>, Req0, undefined),

    %% Query string
    Qs = cowboy_req:parse_qs(Req0),
    %% [{<<"name">>, <<"alice">>}, {<<"age">>, <<"30">>}]

    %% Path bindings
    Id = cowboy_req:binding(id, Req0),
    Id2 = cowboy_req:binding(id, Req0, undefined),  %% with default

    %% Request body (for POST/PUT)
    {ok, Body, Req1} = cowboy_req:read_body(Req0),
    {ok, Body, Req1} = cowboy_req:read_body(Req0, #{length => 64000, period => 5000}),

    %% URL-encoded form
    {ok, PostVals, Req2} = cowboy_req:read_urlencoded_body(Req0),

    %% Multipart
    multipart_loop(Req0, []).

multipart_loop(Req0, Acc) ->
    case cowboy_req:read_part(Req0) of
        {ok, Headers, Req1} ->
            {ok, Data, Req2} = cowboy_req:read_part_body(Req1),
            multipart_loop(Req2, [{Headers, Data} | Acc]);
        {done, Req1} ->
            {ok, lists:reverse(Acc), Req1}
    end.

%% 3. Response types
%% Simple reply
Req1 = cowboy_req:reply(200, #{}, <<"body">>, Req0).

%% Chunked response (streaming)
Req1 = cowboy_req:stream_reply(200, #{}, Req0).
cowboy_req:stream_body(<<"chunk1">>, nofin, Req1).
cowboy_req:stream_body(<<"chunk2">>, fin, Req1).

%% Headers only (no body)
Req1 = cowboy_req:reply(204, #{}, Req0).

%% Redirect
Req1 = cowboy_req:reply(302,
    #{<<"location">> => <<"/new-path">>},
    Req0).
```

---

## REST Handler

```erlang
%% Cowboy REST: implements HTTP content negotiation
%% - Handles GET/POST/PUT/PATCH/DELETE
%% - Content-type negotiation
%% - Caching (ETag, Last-Modified)
%% - Conditional requests

-module(users_handler).
-behaviour(cowboy_rest).

-export([init/2]).
-export([
    allowed_methods/2,
    content_types_accepted/2,
    content_types_provided/2,
    resource_exists/2,
    delete_resource/2
]).
-export([provide_json/2, accept_json/2]).

init(Req, State) ->
    {cowboy_rest, Req, State}.

%% Which HTTP methods are allowed
allowed_methods(Req, State) ->
    {[<<"GET">>, <<"POST">>, <<"DELETE">>], Req, State}.

%% What response content types we can produce
content_types_provided(Req, State) ->
    {[
        {<<"application/json">>, provide_json},
        {<<"text/html">>, provide_html}
    ], Req, State}.

%% What request content types we accept
content_types_accepted(Req, State) ->
    {[
        {<<"application/json">>, accept_json},
        {<<"application/x-www-form-urlencoded">>, accept_form}
    ], Req, State}.

%% Check if resource exists (for GET/DELETE)
resource_exists(Req, State) ->
    case cowboy_req:binding(id, Req, undefined) of
        undefined -> {true, Req, State};  %% collection
        Id ->
            case users_db:find(Id) of
                {ok, User} -> {true, Req, State#{user => User}};
                not_found -> {false, Req, State}
            end
    end.

%% Handler for GET (provides JSON)
provide_json(Req, #{user := User} = State) ->
    Body = jsx:encode(User),
    {Body, Req, State};

provide_json(Req, State) ->
    Users = users_db:all(),
    Body = jsx:encode(Users),
    {Body, Req, State}.

%% Handler for POST/PUT (accepts JSON)
accept_json(Req0, State) ->
    {ok, Body, Req1} = cowboy_req:read_body(Req0),
    case jsx:decode(Body, [return_maps]) of
        {ok, UserData} ->
            case users_db:create(UserData) of
                {ok, User} ->
                    Id = maps:get(id, User),
                    Req2 = cowboy_req:set_resp_header(
                        <<"location">>,
                        <<"/api/users/", (integer_to_binary(Id))/binary>>,
                        Req1
                    ),
                    Req3 = cowboy_req:set_resp_body(jsx:encode(User), Req2),
                    {true, Req3, State};
                {error, Reason} ->
                    Req2 = cowboy_req:reply(422, #{}, jsx:encode(#{error => Reason}), Req1),
                    {stop, Req2, State}
            end;
        {error, _} ->
            Req2 = cowboy_req:reply(400, #{}, <<"Invalid JSON">>, Req1),
            {stop, Req2, State}
    end.

delete_resource(Req, #{user := #{id := Id}} = State) ->
    users_db:delete(Id),
    {true, Req, State}.
```

---

## WebSocket Handler

```erlang
%% WebSocket handler
-module(ws_handler).

-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, State) ->
    {cowboy_websocket, Req, State,
     #{
         idle_timeout => 60000,    %% 60s idle disconnect
         compress => true           %% compress messages
     }}.

websocket_init(State) ->
    io:format("WebSocket connected~n"),
    %% Subscribe to topic
    gproc:reg({p, l, chat_room}),
    {ok, State#{connected_at => os:system_time(second)}}.

%% Handle messages from client
websocket_handle({text, <<"ping">>}, State) ->
    {reply, {text, <<"pong">>}, State};

websocket_handle({text, JsonMsg}, State) ->
    case jsx:decode(JsonMsg, [return_maps]) of
        {ok, #{<<"type">> := <<"chat">>, <<"msg">> := Msg}} ->
            %% Broadcast to all clients
            gproc:send({p, l, chat_room}, {chat, self(), Msg}),
            {ok, State};
        {ok, _} ->
            {reply, {text, jsx:encode(#{error => <<"unknown_type">>})}, State};
        {error, _} ->
            {reply, {text, <<"invalid json">>}, State}
    end;

websocket_handle({binary, _Data}, State) ->
    {ok, State};

websocket_handle(_Frame, State) ->
    {ok, State}.

%% Handle Erlang messages sent to this process
websocket_info({chat, _From, Msg}, State) ->
    Response = jsx:encode(#{type => <<"chat">>, msg => Msg}),
    {reply, {text, Response}, State};

websocket_info({timeout, _, ping}, State) ->
    %% Send ping to client to keep connection alive
    {reply, ping, State};

websocket_info(_Info, State) ->
    {ok, State}.

terminate(normal, _Req, _State) ->
    io:format("WebSocket disconnected~n"),
    ok;
terminate(Reason, _Req, _State) ->
    io:format("WebSocket error: ~p~n", [Reason]),
    ok.
```

---

## Middleware และ Hooks

```erlang
%% Cowboy Middleware: run code before/after handlers

%% ตัวอย่าง: Authentication middleware
-module(auth_middleware).
-behaviour(cowboy_middleware).

-export([execute/2]).

execute(Req, Env) ->
    case cowboy_req:path(Req) of
        <<"/api/", _/binary>> ->
            %% Require auth for /api/* routes
            case cowboy_req:header(<<"authorization">>, Req, undefined) of
                <<"Bearer ", Token/binary>> ->
                    case verify_token(Token) of
                        {ok, Claims} ->
                            Req2 = cowboy_req:set_resp_header(
                                <<"x-user-id">>,
                                maps:get(<<"sub">>, Claims, <<>>),
                                Req
                            ),
                            Env2 = Env#{auth_claims => Claims},
                            {ok, Req2, Env2};
                        {error, _} ->
                            Req2 = cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
                            {stop, Req2}
                    end;
                _ ->
                    Req2 = cowboy_req:reply(401, #{}, <<"Missing token">>, Req),
                    {stop, Req2}
            end;
        _ ->
            %% Public routes, pass through
            {ok, Req, Env}
    end.

verify_token(Token) ->
    %% Verify JWT or session token
    case jwt:decode(Token, <<"secret">>) of
        {ok, Claims} -> {ok, Claims};
        {error, Reason} -> {error, Reason}
    end.

%% Register middleware in cowboy start
cowboy:start_clear(http,
    [{port, 8080}],
    #{
        env => #{dispatch => Dispatch},
        middlewares => [
            cowboy_router,
            auth_middleware,   %% your middleware
            cowboy_handler
        ],
        stream_handlers => [cowboy_compress_h, cowboy_stream_h]
    }
).

%% CORS middleware
-module(cors_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req0, Env) ->
    Req1 = cowboy_req:set_resp_headers(#{
        <<"access-control-allow-origin">> => <<"*">>,
        <<"access-control-allow-methods">> => <<"GET, POST, PUT, DELETE, OPTIONS">>,
        <<"access-control-allow-headers">> => <<"content-type, authorization">>
    }, Req0),
    case cowboy_req:method(Req1) of
        <<"OPTIONS">> ->
            Req2 = cowboy_req:reply(204, #{}, Req1),
            {stop, Req2};
        _ ->
            {ok, Req1, Env}
    end.
```

---

## ตัวอย่างจริง: JSON REST API

```erlang
%% ไฟล์: src/api_handler.erl
%% Full-featured JSON API handler

-module(api_handler).

-export([init/2]).

init(Req0, State) ->
    Method = cowboy_req:method(Req0),
    Path = cowboy_req:path_info(Req0),  %% list of path segments after base
    {Req1, Result} = try
        handle(Method, Path, Req0, State)
    catch
        error:{badmatch, {error, not_found}} ->
            json_error(404, <<"Not found">>, Req0);
        error:{badmatch, {error, Reason}} ->
            json_error(400, Reason, Req0);
        _:Error:Stack ->
            io:format("Error: ~p~n~p~n", [Error, Stack]),
            json_error(500, <<"Internal server error">>, Req0)
    end,
    {ok, Req1, Result}.

%% GET /api/products
handle(<<"GET">>, [], Req, State) ->
    Qs = cowboy_req:parse_qs(Req),
    Page = binary_to_integer(proplists:get_value(<<"page">>, Qs, <<"1">>)),
    Limit = binary_to_integer(proplists:get_value(<<"limit">>, Qs, <<"20">>)),
    Products = products_db:list(#{page => Page, limit => Limit}),
    json_ok(200, Products, Req, State);

%% GET /api/products/:id
handle(<<"GET">>, [IdBin], Req, State) ->
    Id = binary_to_integer(IdBin),
    {ok, Product} = products_db:get(Id),  %% badmatch if not_found
    json_ok(200, Product, Req, State);

%% POST /api/products
handle(<<"POST">>, [], Req0, State) ->
    {ok, Body, Req1} = cowboy_req:read_body(Req0),
    {ok, Data} = jsx:decode(Body, [return_maps]),
    {ok, Validated} = validate_product(Data),
    {ok, Product} = products_db:create(Validated),
    Id = maps:get(id, Product),
    Req2 = cowboy_req:set_resp_header(
        <<"location">>, <<"/api/products/", (integer_to_binary(Id))/binary>>, Req1),
    json_ok(201, Product, Req2, State);

%% PUT /api/products/:id
handle(<<"PUT">>, [IdBin], Req0, State) ->
    Id = binary_to_integer(IdBin),
    {ok, _, _} = products_db:get(Id),  %% ensure exists
    {ok, Body, Req1} = cowboy_req:read_body(Req0),
    {ok, Data} = jsx:decode(Body, [return_maps]),
    {ok, Updated} = products_db:update(Id, Data),
    json_ok(200, Updated, Req1, State);

%% DELETE /api/products/:id
handle(<<"DELETE">>, [IdBin], Req, State) ->
    Id = binary_to_integer(IdBin),
    ok = products_db:delete(Id),
    json_ok(204, #{}, Req, State);

handle(_, _, Req, State) ->
    json_error(405, <<"Method not allowed">>, Req).

%% Helpers
json_ok(Status, Data, Req0, State) ->
    Body = jsx:encode(Data),
    Req1 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        Body, Req0),
    {Req1, State}.

json_error(Status, Message, Req) ->
    Body = jsx:encode(#{error => Message}),
    Req1 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        Body, Req),
    {Req1, #{}}.

validate_product(#{<<"name">> := Name, <<"price">> := Price} = Data)
  when is_binary(Name), is_number(Price), Price >= 0 ->
    {ok, Data};
validate_product(_) ->
    {error, <<"Invalid product data">>}.
```

---

## สรุป Part 35

| Handler Type | ใช้เมื่อ |
|-------------|---------|
| Plain handler | Custom logic |
| cowboy_rest | REST API with content negotiation |
| cowboy_websocket | Real-time bidirectional |
| cowboy_static | Serve static files |

| cowboy_req function | Returns |
|--------------------|---------|
| `method/1` | <<"GET">>, <<"POST">>, ... |
| `path/1` | Full path |
| `binding/2,3` | URL parameter |
| `parse_qs/1` | Query string as list |
| `header/2,3` | Request header |
| `read_body/1,2` | Body as binary |
| `reply/3,4` | Response |

---

*[← Part 34: HTTP Client](part_34_http_client.md) | [Part 36: Database Access →](part_36_database.md)*
