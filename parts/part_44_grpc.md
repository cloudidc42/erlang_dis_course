# Part 44: gRPC กับ Protocol Buffers

## สารบัญ
1. [gRPC คืออะไร](#grpc-คืออะไร)
2. [Protocol Buffers](#protocol-buffers)
3. [gRPC Server ใน Erlang](#grpc-server-ใน-erlang)
4. [gRPC Client](#grpc-client)
5. [Streaming](#streaming)
6. [ตัวอย่างจริง: Microservice Communication](#ตัวอย่างจริง-microservice-communication)

---

## gRPC คืออะไร

```
gRPC = Google Remote Procedure Call
- สื่อสารระหว่าง services ด้วย HTTP/2
- ใช้ Protocol Buffers (protobuf) สำหรับ serialization
- รองรับ 4 ประเภท: Unary, Server Streaming, Client Streaming, Bidirectional
- เร็วกว่า REST/JSON (binary protocol)
- strongly typed (schema-first)

Libraries สำหรับ Erlang:
- grpcbox (official gRPC implementation)
- grpc (alternative)
```

---

## Protocol Buffers

```protobuf
// users.proto
syntax = "proto3";

package users;

option erlang_module_prefix = "users_pb";

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (stream User);
    rpc CreateUser (CreateUserRequest) returns (User);
    rpc WatchUser (WatchUserRequest) returns (stream UserEvent);
}

message GetUserRequest {
    int64 user_id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
    string filter = 3;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    int32 age = 3;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    int64 created_at = 5;
}

message WatchUserRequest {
    int64 user_id = 1;
}

message UserEvent {
    enum EventType {
        UNKNOWN = 0;
        CREATED = 1;
        UPDATED = 2;
        DELETED = 3;
    }
    EventType type = 1;
    User user = 2;
    int64 timestamp = 3;
}
```

```bash
# Generate Erlang code from .proto
# Using rebar3 with grpcbox plugin

# rebar.config
# {plugins, [{grpcbox_plugin, "0.17.0"}]}
# {grpc, [{protos, "proto"}, {out_dir, "src/generated"}]}

$ rebar3 grpc gen
# Generates:
# src/generated/users_pb.erl     (message encoding/decoding)
# src/generated/users_grpc.erl   (service stubs)
```

---

## gRPC Server ใน Erlang

```erlang
%% rebar.config
%% {deps, [{grpcbox, "0.17.0"}, {cthr, "0.1.3"}]}

%% ไฟล์: users_service.erl (implements UserService)
-module(users_service).
-behaviour(grpcbox_service).
-export([get_user/2, list_users/2, create_user/2, watch_user/2]).

%% Unary RPC: GetUser
get_user(Ctx, #{user_id := UserId}) ->
    case users_db:get(UserId) of
        {ok, User} ->
            {ok, user_to_pb(User), Ctx};
        {error, not_found} ->
            {grpc_error, {?GRPC_STATUS_NOT_FOUND, <<"User not found">>}}
    end.

%% Server Streaming: ListUsers
list_users(Ctx, #{page := Page, page_size := PageSize}) ->
    case users_db:list(Page, PageSize) of
        {ok, Users} ->
            lists:foreach(fun(User) ->
                grpcbox_stream:send(user_to_pb(User), Ctx)
            end, Users),
            {ok, Ctx};
        {error, Reason} ->
            {grpc_error, {?GRPC_STATUS_INTERNAL, term_to_binary(Reason)}}
    end.

%% Unary: CreateUser
create_user(Ctx, #{name := Name, email := Email, age := Age}) ->
    case validate_user(Name, Email, Age) of
        ok ->
            {ok, UserId} = users_db:create(Name, Email, Age),
            User = #{id => UserId, name => Name, email => Email, age => Age,
                     created_at => erlang:system_time(second)},
            {ok, User, Ctx};
        {error, Reason} ->
            {grpc_error, {?GRPC_STATUS_INVALID_ARGUMENT, Reason}}
    end.

%% Bidirectional Streaming: WatchUser
watch_user(Ctx, _Req) ->
    %% Subscribe to user events
    user_events:subscribe(self()),
    watch_loop(Ctx).

watch_loop(Ctx) ->
    receive
        {user_event, Event} ->
            grpcbox_stream:send(event_to_pb(Event), Ctx),
            watch_loop(Ctx);
        stop ->
            {ok, Ctx}
    after 30000 ->
        %% Keepalive
        grpcbox_stream:send(#{type => 0}, Ctx),
        watch_loop(Ctx)
    end.

user_to_pb(#{id := Id, name := Name, email := Email, age := Age}) ->
    #{id => Id, name => Name, email => Email, age => Age,
      created_at => erlang:system_time(second)}.

validate_user(Name, Email, Age) when
    byte_size(Name) > 0,
    byte_size(Email) > 3,
    Age >= 0, Age =< 150 ->
    ok;
validate_user(_, _, _) ->
    {error, <<"Invalid user data">>}.

%% ไฟล์: users_app.erl
start_grpc_server() ->
    Services = #{
        <<"users.UserService">> => {users_service, users_grpc}
    },
    ServerConfig = #{
        grpc_opts => #{services => Services},
        transport_opts => #{socket_opts => [{port, 50051}]}
    },
    grpcbox:start_server(ServerConfig).
```

---

## gRPC Client

```erlang
%% gRPC Client ใน Erlang

start_client() ->
    {ok, _} = grpcbox_channel:start_link(
        users_channel,
        [{http, "localhost", 50051, []}],
        #{codec => grpcbox_codec_proto}
    ).

%% Unary call
get_user(UserId) ->
    Request = #{user_id => UserId},
    case users_grpc_client:get_user(Request, #{channel => users_channel}) of
        {ok, User, _Headers} ->
            {ok, User};
        {error, {Status, Message}, _} ->
            {error, {Status, Message}}
    end.

%% Server streaming
list_all_users() ->
    Request = #{page => 1, page_size => 100},
    {ok, Stream} = users_grpc_client:list_users(Request, #{channel => users_channel}),
    collect_stream(Stream, []).

collect_stream(Stream, Acc) ->
    case grpcbox_client:recv_data(Stream) of
        {ok, User} ->
            collect_stream(Stream, [User | Acc]);
        {ok, _Headers, _Trailers} ->
            {ok, lists:reverse(Acc)};
        {error, Reason} ->
            {error, Reason}
    end.

%% Client streaming
batch_create_users(Users) ->
    {ok, Stream} = users_grpc_client:create_user(#{channel => users_channel}),
    lists:foreach(fun(User) ->
        grpcbox_client:send(Stream, User)
    end, Users),
    grpcbox_client:close_send(Stream),
    case grpcbox_client:recv_data(Stream) of
        {ok, Response} -> {ok, Response};
        {error, Reason} -> {error, Reason}
    end.

%% Deadline/timeout
get_user_with_timeout(UserId, TimeoutMs) ->
    Request = #{user_id => UserId},
    Options = #{
        channel => users_channel,
        timeout => TimeoutMs,
        metadata => #{<<"x-request-id">> => generate_id()}
    },
    users_grpc_client:get_user(Request, Options).
```

---

## Streaming

```erlang
%% Bidirectional streaming example
%% Chat service

-module(chat_service).
-behaviour(grpcbox_service).
-export([chat/2]).

%% chat/2: bidirectional streaming
chat(Ctx, _InitialReq) ->
    %% Register in chat room
    chat_room:join(self()),
    chat_loop(Ctx).

chat_loop(Ctx) ->
    receive
        %% Incoming message from client
        {data, #{message := Msg, user := User}} ->
            %% Broadcast to all in room
            chat_room:broadcast(Msg, User),
            chat_loop(Ctx);

        %% Message from another user (via chat_room)
        {room_message, FromUser, Msg} ->
            Response = #{from => FromUser, message => Msg,
                        timestamp => erlang:system_time(second)},
            grpcbox_stream:send(Response, Ctx),
            chat_loop(Ctx);

        stop ->
            chat_room:leave(self()),
            {ok, Ctx}
    end.

%% Interceptor: logging, auth, tracing
-module(grpc_auth_interceptor).
-export([unary/4]).

unary(Ctx, Req, ServerInfo, Handler) ->
    case grpcbox_metadata:get(<<"authorization">>, Ctx) of
        {ok, [<<"Bearer ", Token/binary>> | _]} ->
            case auth:verify_token(Token) of
                {ok, Claims} ->
                    Ctx2 = grpcbox_ctx:with_value(Ctx, user_claims, Claims),
                    Handler(Ctx2, Req, ServerInfo);
                {error, _} ->
                    {grpc_error, {?GRPC_STATUS_UNAUTHENTICATED, <<"Invalid token">>}}
            end;
        _ ->
            {grpc_error, {?GRPC_STATUS_UNAUTHENTICATED, <<"Missing auth">>}}
    end.
```

---

## ตัวอย่างจริง: Microservice Communication

```erlang
%% order_service.erl: ใช้ gRPC เรียก inventory + payment services

-module(order_service).
-behaviour(grpcbox_service).
-export([create_order/2]).

create_order(Ctx, #{user_id := UserId, items := Items, payment := Payment}) ->
    %% Step 1: Validate inventory via gRPC
    case check_inventory(Items) of
        {ok, _} ->
            %% Step 2: Reserve items
            {ok, ReservationId} = reserve_inventory(Items),
            %% Step 3: Process payment
            case process_payment(UserId, total_price(Items), Payment) of
                {ok, PaymentId} ->
                    %% Step 4: Confirm order
                    {ok, OrderId} = confirm_order(UserId, Items, ReservationId, PaymentId),
                    {ok, #{order_id => OrderId, status => <<"confirmed">>}, Ctx};
                {error, payment_failed} ->
                    %% Rollback inventory reservation
                    cancel_reservation(ReservationId),
                    {grpc_error, {?GRPC_STATUS_FAILED_PRECONDITION, <<"Payment failed">>}}
            end;
        {error, out_of_stock} ->
            {grpc_error, {?GRPC_STATUS_UNAVAILABLE, <<"Items out of stock">>}}
    end.

check_inventory(Items) ->
    Request = #{items => Items},
    inventory_grpc_client:check(Request, #{
        channel => inventory_channel,
        timeout => 3000
    }).

reserve_inventory(Items) ->
    Request = #{items => Items},
    case inventory_grpc_client:reserve(Request, #{channel => inventory_channel}) of
        {ok, #{reservation_id := Id}, _} -> {ok, Id};
        {error, _, _} -> {error, reservation_failed}
    end.

process_payment(UserId, Amount, PaymentInfo) ->
    Request = #{user_id => UserId, amount => Amount, info => PaymentInfo},
    case payment_grpc_client:charge(Request, #{
        channel => payment_channel,
        timeout => 5000
    }) of
        {ok, #{payment_id := Id}, _} -> {ok, Id};
        {error, _, _} -> {error, payment_failed}
    end.

%% Graceful degradation: fallback to HTTP if gRPC fails
call_with_fallback(GrpcFun, HttpFallback) ->
    case catch GrpcFun() of
        {ok, Result} -> {ok, Result};
        {error, {?GRPC_STATUS_UNAVAILABLE, _}} ->
            logger:warning("gRPC unavailable, trying HTTP fallback"),
            HttpFallback();
        {'EXIT', _} ->
            HttpFallback()
    end.
```

---

## สรุป Part 44

| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 |
| Serialization | Protobuf (binary) | JSON (text) |
| Schema | Required (proto) | Optional |
| Streaming | Native 4 types | Limited |
| Type safety | Strong | Weak |
| Performance | Fast | Slower |
| Browser support | Limited | Full |

**เลือกใช้ gRPC เมื่อ**: internal service-to-service communication, need streaming, high performance

---

*[← Part 43: Security](part_43_security.md) | [Part 45: WebSockets Advanced →](part_45_websockets.md)*
