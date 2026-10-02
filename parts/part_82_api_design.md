# Part 82: API Design Patterns

## สารบัญ
1. [RESTful API Design](#rest)
2. [GraphQL with Erlang](#graphql)
3. [gRPC Integration](#grpc)
4. [API Versioning](#versioning)
5. [OpenAPI/Swagger Documentation](#openapi)
6. [ตัวอย่างจริง: Multi-Protocol API Gateway](#gateway)

---

## RESTful API Design

```erlang
%% rest_api.erl — well-designed REST API

%% Resource: /api/v1/users
%% GET    /api/v1/users           — list users
%% POST   /api/v1/users           — create user
%% GET    /api/v1/users/:id       — get user
%% PUT    /api/v1/users/:id       — replace user
%% PATCH  /api/v1/users/:id       — update user fields
%% DELETE /api/v1/users/:id       — delete user

-module(users_handler).
-behaviour(cowboy_rest).
-export([init/2, allowed_methods/2, content_types_provided/2,
         content_types_accepted/2, resource_exists/2,
         delete_resource/2]).
-export([provide_json/2, accept_json/2]).

init(Req, State) ->
    {cowboy_rest, Req, State}.

allowed_methods(Req, State) ->
    {[<<"GET">>, <<"POST">>, <<"PUT">>, <<"PATCH">>, <<"DELETE">>], Req, State}.

content_types_provided(Req, State) ->
    {[{<<"application/json">>, provide_json}], Req, State}.

content_types_accepted(Req, State) ->
    {[{<<"application/json">>, accept_json}], Req, State}.

%% Check if resource exists (for GET/PUT/PATCH/DELETE)
resource_exists(Req, State) ->
    case cowboy_req:binding(id, Req) of
        undefined ->
            %% Collection endpoint
            {true, Req, State};
        UserId ->
            case user_service:get(UserId) of
                {ok, User} -> {true, Req, State#{user => User}};
                not_found -> {false, Req, State}
            end
    end.

%% GET handler
provide_json(Req, State) ->
    case cowboy_req:binding(id, Req) of
        undefined ->
            %% List with pagination
            QS = cowboy_req:parse_qs(Req),
            Page = binary_to_integer(proplists:get_value(<<"page">>, QS, <<"1">>)),
            PageSize = min(binary_to_integer(proplists:get_value(<<"page_size">>, QS, <<"20">>)), 100),
            
            {ok, Users, Total} = user_service:list(Page, PageSize),
            
            Body = jiffy:encode(#{
                data => Users,
                meta => #{
                    total => Total,
                    page => Page,
                    page_size => PageSize,
                    total_pages => ceil(Total / PageSize)
                }
            }),
            {Body, Req, State};
        
        _UserId ->
            User = maps:get(user, State),
            Body = jiffy:encode(#{data => User}),
            {Body, Req, State}
    end.

%% POST/PUT/PATCH handler
accept_json(Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    
    case jiffy:decode(Body, [return_maps]) of
        {error, _} ->
            Req3 = cowboy_req:reply(400, #{}, <<"Invalid JSON">>, Req2),
            {halt, Req3, State};
        
        Data ->
            Method = cowboy_req:method(Req2),
            handle_write(Method, Data, Req2, State)
    end.

handle_write(<<"POST">>, Data, Req, State) ->
    case validate_user_create(Data) of
        {ok, ValidData} ->
            case user_service:create(ValidData) of
                {ok, User} ->
                    UserId = maps:get(id, User),
                    Location = <<"/api/v1/users/", UserId/binary>>,
                    Req2 = cowboy_req:set_resp_headers(#{<<"location">> => Location}, Req),
                    Req3 = cowboy_req:reply(201, #{<<"content-type">> => <<"application/json">>},
                                            jiffy:encode(#{data => User}), Req2),
                    {halt, Req3, State};
                {error, Reason} ->
                    error_response(422, Reason, Req, State)
            end;
        {error, Errors} ->
            error_response(400, #{errors => Errors}, Req, State)
    end;

handle_write(<<"PATCH">>, Data, Req, State) ->
    UserId = cowboy_req:binding(id, Req),
    case user_service:update(UserId, Data) of
        {ok, User} ->
            {jiffy:encode(#{data => User}), Req, State};
        {error, Reason} ->
            error_response(422, Reason, Req, State)
    end.

delete_resource(Req, State) ->
    UserId = cowboy_req:binding(id, Req),
    user_service:delete(UserId),
    {true, Req, State}.

error_response(StatusCode, Reason, Req, State) ->
    Body = jiffy:encode(#{error => Reason}),
    Req2 = cowboy_req:reply(StatusCode,
        #{<<"content-type">> => <<"application/json">>}, Body, Req),
    {halt, Req2, State}.

validate_user_create(Data) ->
    validator:validate(Data, #{
        <<"name">> => #{type => string, required => true, min_length => 1, max_length => 100},
        <<"email">> => #{type => string, required => true, max_length => 255},
        <<"password">> => #{type => string, required => true, min_length => 8}
    }).
```

---

## GraphQL with Erlang

```erlang
%% graphql setup using ergl or absinthe-compatible approach
%% {deps, [{graphql, "0.16.0"}]}  -- graphql-erlang

-module(graphql_setup).
-export([init_schema/0, execute/2]).

%% Define schema
init_schema() ->
    Schema = #{
        query => #{
            fields => #{
                user => #{
                    type => user_type,
                    args => #{id => #{type => id}},
                    resolve => fun resolve_user/2
                },
                users => #{
                    type => #{list => user_type},
                    args => #{
                        page => #{type => int, default => 1},
                        page_size => #{type => int, default => 20}
                    },
                    resolve => fun resolve_users/2
                }
            }
        },
        mutation => #{
            fields => #{
                create_user => #{
                    type => user_type,
                    args => #{
                        name => #{type => string},
                        email => #{type => string}
                    },
                    resolve => fun resolve_create_user/2
                }
            }
        },
        types => #{
            user_type => #{
                fields => #{
                    id => #{type => id},
                    name => #{type => string},
                    email => #{type => string},
                    orders => #{
                        type => #{list => order_type},
                        resolve => fun resolve_user_orders/2
                    }
                }
            }
        }
    },
    graphql:init(Schema).

resolve_user(#{<<"id">> := Id}, _Context) ->
    user_service:get(Id).

resolve_users(#{<<"page">> := Page, <<"page_size">> := PageSize}, _Context) ->
    {ok, Users, _Total} = user_service:list(Page, PageSize),
    Users.

resolve_create_user(Args, _Context) ->
    user_service:create(Args).

resolve_user_orders(#{<<"id">> := UserId}, _Context) ->
    order_service:list_by_user(UserId).

%% GraphQL HTTP endpoint
-module(graphql_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"query">> := Query, <<"variables">> := Vars} = jiffy:decode(Body, [return_maps]),
    
    case graphql:execute(Query, Vars, #{}) of
        {ok, Result} ->
            {ok, cowboy_req:reply(200,
                #{<<"content-type">> => <<"application/json">>},
                jiffy:encode(#{data => Result}), Req2), State};
        {error, Errors} ->
            {ok, cowboy_req:reply(200,
                #{<<"content-type">> => <<"application/json">>},
                jiffy:encode(#{errors => Errors}), Req2), State}
    end.
```

---

## gRPC Integration

```erlang
%% gRPC using grpc-erl library
%% {deps, [{grpc, "0.7.0"}]}

%% 1. Define proto file (user_service.proto):
%%
%% syntax = "proto3";
%% service UserService {
%%   rpc GetUser (GetUserRequest) returns (User) {}
%%   rpc ListUsers (ListUsersRequest) returns (stream User) {}
%%   rpc CreateUser (CreateUserRequest) returns (User) {}
%% }
%% message User { string id = 1; string name = 2; string email = 3; }
%% message GetUserRequest { string id = 1; }

%% 2. Generate Erlang code from proto
%% grpc_erl:generate("user_service.proto")

%% 3. Implement service
-module(user_service_server).
-behaviour(user_service_bhvr).
-export([get_user/2, list_users/2, create_user/2]).

get_user(#{id := UserId}, _Metadata) ->
    case user_store:get(UserId) of
        {ok, User} ->
            {ok, #{id => UserId,
                   name => maps:get(name, User),
                   email => maps:get(email, User)}};
        not_found ->
            {error, {not_found, <<"User not found">>}}
    end.

%% Server streaming
list_users(#{page := Page, page_size := PageSize}, StreamRef) ->
    {ok, Users, _} = user_store:list(Page, PageSize),
    [grpc:send(StreamRef, #{id => U#user.id, name => U#user.name}) || U <- Users],
    grpc:finish(StreamRef).

create_user(#{name := Name, email := Email}, _Metadata) ->
    case user_store:create(Name, Email) of
        {ok, User} -> {ok, User};
        {error, Reason} -> {error, Reason}
    end.

%% 4. Start gRPC server
start_grpc() ->
    grpc:start_server(grpc_server, tcp, 50051, [
        {handler, user_service_server},
        {service, user_service}
    ]).

%% 5. gRPC client
call_grpc_service() ->
    {ok, Connection} = grpc_client:connect(tcp, "localhost", 50051),
    
    {ok, User} = user_service_client:get_user(Connection, #{id => <<"123">>}, #{}),
    io:format("Got user: ~p~n", [User]),
    
    grpc_client:stop_connection(Connection).
```

---

## API Versioning

```erlang
%% versioning_router.erl — URL-based versioning

routes() ->
    cowboy_router:compile([
        {'_', [
            %% v1 routes
            {"/api/v1/users", v1_users_handler, []},
            {"/api/v1/users/:id", v1_user_handler, []},
            
            %% v2 routes (new features)
            {"/api/v2/users", v2_users_handler, []},
            {"/api/v2/users/:id", v2_user_handler, []},
            
            %% Redirect old paths
            {"/api/users", redirect_handler, [{to, "/api/v2/users"}]}
        ]}
    ]).

%% v2 extends v1 with backward compatibility
-module(v2_users_handler).
-behaviour(cowboy_rest).

%% v2 adds: cursor-based pagination, field selection, expanded response
provide_json(Req, State) ->
    QS = cowboy_req:parse_qs(Req),
    
    %% New in v2: cursor pagination
    Cursor = proplists:get_value(<<"cursor">>, QS),
    PageSize = binary_to_integer(proplists:get_value(<<"page_size">>, QS, <<"20">>)),
    
    %% New in v2: field selection
    Fields = case proplists:get_value(<<"fields">>, QS) of
        undefined -> all;
        F -> binary:split(F, <<",">>)
    end,
    
    {ok, Users, NextCursor} = user_service:list_cursor(Cursor, PageSize),
    
    FilteredUsers = filter_fields(Users, Fields),
    
    Body = jiffy:encode(#{
        data => FilteredUsers,
        pagination => #{
            next_cursor => NextCursor,
            has_more => NextCursor =/= undefined
        }
    }),
    {Body, Req, State}.

filter_fields(Users, all) -> Users;
filter_fields(Users, Fields) ->
    [maps:with([binary_to_atom(F) || F <- Fields], U) || U <- Users].

%% Header-based versioning alternative
get_api_version(Req) ->
    case cowboy_req:header(<<"api-version">>, Req) of
        <<"2">> -> v2;
        <<"1">> -> v1;
        _ -> v2  %% default to latest
    end.
```

---

## ตัวอย่างจริง: Multi-Protocol API Gateway

```erlang
%% api_gateway.erl — single entry point supporting REST, GraphQL, gRPC

-module(api_gateway).
-export([start/0]).

start() ->
    %% HTTP/REST + GraphQL
    start_http_server(),
    
    %% gRPC
    start_grpc_server(),
    
    %% WebSocket
    start_ws_server(),
    
    ok.

start_http_server() ->
    Dispatch = cowboy_router:compile([
        {'_', [
            %% REST API
            {"/api/v1/[:resource]/[:id]", rest_router, []},
            
            %% GraphQL
            {"/graphql", graphql_handler, []},
            
            %% WebSocket
            {"/ws", ws_handler, []},
            
            %% Health
            {"/health", health_handler, []},
            
            %% Metrics
            {"/metrics", prometheus_cowboy2_handler, []}
        ]}
    ]),
    
    cowboy:start_clear(http, [{port, 8080}, {num_acceptors, 100}], #{
        env => #{dispatch => Dispatch},
        middlewares => [
            cowboy_router,
            auth_middleware,
            rate_limit_middleware,
            logging_middleware,
            cowboy_handler
        ]
    }).

start_grpc_server() ->
    grpc:start_server(grpc_server, tcp, 50051, [
        {handler, grpc_handler},
        {interceptors, [auth_interceptor, logging_interceptor]}
    ]).

%% REST router that dispatches to handlers
-module(rest_router).
-behaviour(cowboy_handler).

init(Req, State) ->
    Resource = cowboy_req:binding(resource, Req),
    handle_resource(Resource, Req, State).

handle_resource(<<"users">>, Req, State) ->
    users_handler:init(Req, State);
handle_resource(<<"orders">>, Req, State) ->
    orders_handler:init(Req, State);
handle_resource(<<"products">>, Req, State) ->
    products_handler:init(Req, State);
handle_resource(_, Req, State) ->
    Req2 = cowboy_req:reply(404, #{<<"content-type">> => <<"application/json">>},
                            jiffy:encode(#{error => <<"Not Found">>}), Req),
    {ok, Req2, State}.

%% Request/Response logging middleware
-module(logging_middleware).
-behaviour(cowboy_middleware).

execute(Req, Env) ->
    T1 = erlang:monotonic_time(microsecond),
    
    case cowboy_middleware:execute(Req, Env) of
        {ok, Req2, Env2} ->
            T2 = erlang:monotonic_time(microsecond),
            log_request(Req2, T2 - T1),
            {ok, Req2, Env2};
        Other ->
            Other
    end.

log_request(Req, DurationUs) ->
    logger:info("HTTP Request", #{
        method => cowboy_req:method(Req),
        path => cowboy_req:path(Req),
        status => cowboy_req:resp_status(Req),
        duration_ms => DurationUs / 1000,
        user_agent => cowboy_req:header(<<"user-agent">>, Req)
    }).
```

---

## สรุป Part 82

| API Style | Library/Tool | Use Case |
|-----------|-------------|---------|
| REST | cowboy_rest | Most APIs |
| GraphQL | graphql-erlang | Flexible queries |
| gRPC | grpc-erl | Microservices RPC |
| WebSocket | cowboy_websocket | Real-time |
| SSE | cowboy stream | Unidirectional streaming |

**Best practice**: Design API contract first (OpenAPI spec) → Generate code → Implement

---

*[← Part 81: Event Sourcing](part_81_event_sourcing.md) | [Part 83: Microservices Patterns →](part_83_microservices.md)*
