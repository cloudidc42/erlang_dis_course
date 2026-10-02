# Part 65: Full Production System — E-commerce Platform

## สารบัญ
1. [System Architecture](#architecture)
2. [Order Processing Service](#orders)
3. [Inventory Service](#inventory)
4. [Payment Service](#payment)
5. [API Gateway](#gateway)
6. [Deployment และ Operations](#operations)

---

## System Architecture

```
                    ┌─────────────────┐
                    │   CDN/CloudFront │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │ ← rate limiting, auth, routing
                    └────┬────────┬──┘
                         │        │
              ┌──────────▼──┐  ┌──▼──────────┐
              │ Order Svc   │  │ Product Svc  │
              │ (Erlang)    │  │ (Erlang)     │
              └──────┬──────┘  └──────┬───────┘
                     │                │
         ┌───────────▼──────────────────────┐
         │              RabbitMQ            │ ← async events
         └───┬───────────────────────────┬──┘
             │                           │
    ┌────────▼──────┐          ┌─────────▼──────┐
    │ Inventory Svc │          │ Payment Svc     │
    │ (Erlang)      │          │ (Erlang)        │
    └───────┬───────┘          └────────┬────────┘
            │                           │
    ┌───────▼──────────────────────────▼────────┐
    │              PostgreSQL cluster            │
    └────────────────────────────────────────────┘

Monitoring: Prometheus + Grafana
Tracing: Jaeger
Logs: ELK Stack
```

---

## Order Processing Service

```erlang
%% order_app/src/order_server.erl

-module(order_server).
-behaviour(gen_server).
-export([start_link/0, create_order/1, get_order/1, cancel_order/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(TIMEOUT, 30000).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create_order(OrderData) ->
    gen_server:call(?MODULE, {create, OrderData}, ?TIMEOUT).

get_order(OrderId) ->
    gen_server:call(?MODULE, {get, OrderId}).

cancel_order(OrderId, Reason) ->
    gen_server:call(?MODULE, {cancel, OrderId, Reason}).

init(_) ->
    {ok, #{}}.

handle_call({create, OrderData}, From, State) ->
    %% Async: don't block gen_server for slow operations
    spawn(fun() ->
        Result = do_create_order(OrderData),
        gen_server:reply(From, Result)
    end),
    {noreply, State};

handle_call({get, OrderId}, _From, State) ->
    Result = db:query("SELECT * FROM orders WHERE id = $1", [OrderId]),
    {reply, format_order(Result), State};

handle_call({cancel, OrderId, Reason}, From, State) ->
    spawn(fun() ->
        Result = do_cancel_order(OrderId, Reason),
        gen_server:reply(From, Result)
    end),
    {noreply, State}.

%% Order creation saga
do_create_order(#{
    customer_id := CustomerId,
    items := Items,
    shipping_address := Address
} = OrderData) ->
    OrderId = generate_order_id(),
    
    %% Step 1: Reserve inventory
    case inventory_client:reserve(OrderId, Items) of
        {ok, ReservationId} ->
            %% Step 2: Process payment
            Total = calculate_total(Items),
            case payment_client:charge(CustomerId, Total, OrderId) of
                {ok, PaymentId} ->
                    %% Step 3: Persist order
                    case persist_order(OrderId, OrderData, ReservationId, PaymentId) of
                        ok ->
                            %% Step 4: Emit event
                            publish_event(order_created, #{
                                order_id => OrderId,
                                customer_id => CustomerId,
                                items => Items,
                                total => Total
                            }),
                            {ok, #{order_id => OrderId}};
                        {error, DbError} ->
                            %% Compensate: refund payment, release inventory
                            payment_client:refund(PaymentId),
                            inventory_client:release(ReservationId),
                            {error, DbError}
                    end;
                {error, PaymentError} ->
                    inventory_client:release(ReservationId),
                    {error, {payment_failed, PaymentError}}
            end;
        {error, InvError} ->
            {error, {insufficient_inventory, InvError}}
    end.

do_cancel_order(OrderId, Reason) ->
    case db:query("SELECT status, payment_id, reservation_id FROM orders WHERE id = $1", [OrderId]) of
        {ok, [{Status, PaymentId, ReservationId}]} ->
            case Status of
                <<"pending">> ->
                    payment_client:refund(PaymentId),
                    inventory_client:release(ReservationId),
                    db:execute("UPDATE orders SET status = 'cancelled', cancel_reason = $2, updated_at = NOW() WHERE id = $1",
                               [OrderId, Reason]),
                    publish_event(order_cancelled, #{order_id => OrderId, reason => Reason}),
                    ok;
                <<"shipped">> ->
                    {error, already_shipped};
                _ ->
                    {error, cannot_cancel}
            end;
        _ ->
            {error, order_not_found}
    end.

persist_order(OrderId, Data, ReservationId, PaymentId) ->
    db:transaction(fun() ->
        db:execute(
            "INSERT INTO orders(id, customer_id, status, total, reservation_id, payment_id, created_at) VALUES($1, $2, 'pending', $3, $4, $5, NOW())",
            [OrderId, maps:get(customer_id, Data), calculate_total(maps:get(items, Data)), ReservationId, PaymentId]
        ),
        Items = maps:get(items, Data),
        [db:execute(
            "INSERT INTO order_items(order_id, product_id, quantity, price) VALUES($1, $2, $3, $4)",
            [OrderId, maps:get(product_id, Item), maps:get(qty, Item), maps:get(price, Item)]
        ) || Item <- Items],
        ok
    end).

generate_order_id() ->
    <<A:32, B:16, C:16, D:16, E:48>> = crypto:strong_rand_bytes(16),
    list_to_binary(io_lib:format("~8.16.0b-~4.16.0b-~4.16.0b-~4.16.0b-~12.16.0b",
                                  [A, B, (C band 16#0fff) bor 16#4000,
                                   (D band 16#3fff) bor 16#8000, E])).

publish_event(EventType, Payload) ->
    rabbitmq:publish(<<"orders">>, atom_to_binary(EventType), jsx:encode(Payload)).

calculate_total(Items) ->
    lists:sum([maps:get(price, I) * maps:get(qty, I) || I <- Items]).

format_order({ok, [{Id, CustomerId, Status, Total, CreatedAt}]}) ->
    {ok, #{id => Id, customer_id => CustomerId, status => Status,
           total => Total, created_at => CreatedAt}};
format_order(_) -> {error, not_found}.
```

---

## Inventory Service

```erlang
%% inventory_server.erl

-module(inventory_server).
-behaviour(gen_server).
-export([start_link/0, reserve/2, release/1, adjust/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

reserve(OrderId, Items) ->
    gen_server:call(?MODULE, {reserve, OrderId, Items}, 10000).

release(ReservationId) ->
    gen_server:cast(?MODULE, {release, ReservationId}).

adjust(ProductId, Quantity, Reason) ->
    gen_server:call(?MODULE, {adjust, ProductId, Quantity, Reason}).

init(_) ->
    %% Load inventory into ETS for fast reads
    ets:new(inventory_cache, [named_table, set, {read_concurrency, true}]),
    refresh_cache(),
    timer:send_interval(60000, refresh_cache),
    {ok, #{}}.

handle_call({reserve, OrderId, Items}, _From, State) ->
    case try_reserve(OrderId, Items) of
        {ok, ReservationId} = Ok ->
            %% Update cache
            [ets:update_counter(inventory_cache, maps:get(product_id, I),
                                {3, -maps:get(qty, I)}) || I <- Items],
            {reply, Ok, State};
        Error ->
            {reply, Error, State}
    end;

handle_call({adjust, ProductId, Qty, Reason}, _From, State) ->
    db:execute("UPDATE inventory SET available = available + $2, last_adjusted_at = NOW() WHERE product_id = $1",
               [ProductId, Qty]),
    db:execute("INSERT INTO inventory_log(product_id, change, reason, ts) VALUES($1, $2, $3, NOW())",
               [ProductId, Qty, Reason]),
    ets:update_counter(inventory_cache, ProductId, {3, Qty}),
    {reply, ok, State}.

try_reserve(OrderId, Items) ->
    db:transaction(fun() ->
        %% Lock rows for update
        Results = [begin
            ProductId = maps:get(product_id, Item),
            Qty = maps:get(qty, Item),
            case db:query(
                "SELECT available FROM inventory WHERE product_id = $1 FOR UPDATE",
                [ProductId]) of
                {ok, [{Available}]} when Available >= Qty ->
                    {ok, ProductId, Qty};
                {ok, [{Available}]} ->
                    {error, {insufficient, ProductId, Available, Qty}};
                _ ->
                    {error, {not_found, ProductId}}
            end
        end || Item <- Items],
        
        case lists:all(fun({ok, _, _}) -> true; (_) -> false end, Results) of
            true ->
                ReservationId = generate_id(),
                %% Deduct from available, add to reserved
                [db:execute("UPDATE inventory SET available = available - $2, reserved = reserved + $2 WHERE product_id = $1",
                            [PId, Q]) || {ok, PId, Q} <- Results],
                db:execute("INSERT INTO reservations(id, order_id, expires_at) VALUES($1, $2, NOW() + INTERVAL '15 minutes')",
                           [ReservationId, OrderId]),
                {ok, ReservationId};
            false ->
                Errors = [E || {error, E} <- Results],
                throw({insufficient_inventory, Errors})
        end
    end).

handle_info(refresh_cache, State) ->
    refresh_cache(),
    {noreply, State}.

refresh_cache() ->
    case db:query("SELECT product_id, reserved, available FROM inventory", []) of
        {ok, Rows} ->
            [ets:insert(inventory_cache, {PId, Reserved, Available})
             || {PId, Reserved, Available} <- Rows];
        _ -> ok
    end.

generate_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).
```

---

## API Gateway

```erlang
%% gateway_handler.erl: route, auth, rate limit

-module(gateway_handler).
-export([init/2]).

init(Req, State) ->
    Path = cowboy_req:path(Req),
    Method = cowboy_req:method(Req),
    
    case authenticate(Req) of
        {ok, User} ->
            case rate_limit_check(User) of
                ok ->
                    route_request(Path, Method, User, Req, State);
                {error, limit_exceeded} ->
                    cowboy_req:reply(429, #{<<"retry-after">> => <<"60">>},
                                    <<"Rate limit exceeded">>, Req),
                    {ok, Req, State}
            end;
        {error, unauthorized} ->
            cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
            {ok, Req, State}
    end.

authenticate(Req) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            jwt_verify(Token);
        _ ->
            {error, unauthorized}
    end.

route_request(<<"/api/orders", Rest/binary>>, Method, User, Req, State) ->
    proxy_to(order_service_url(Rest), Method, User, Req, State);
route_request(<<"/api/products", Rest/binary>>, Method, User, Req, State) ->
    proxy_to(product_service_url(Rest), Method, User, Req, State);
route_request(<<"/api/inventory", Rest/binary>>, Method, User, Req, State) ->
    proxy_to(inventory_service_url(Rest), Method, User, Req, State);
route_request(_, _, _, Req, State) ->
    cowboy_req:reply(404, #{}, <<"Not Found">>, Req),
    {ok, Req, State}.

proxy_to(Url, Method, #{id := UserId}, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    Headers = [
        {"X-User-Id", binary_to_list(UserId)},
        {"Content-Type", "application/json"}
    ],
    case httpc:request(
        binary_to_atom(string:lowercase(Method)),
        {Url, Headers, "application/json", Body},
        [{timeout, 30000}], []
    ) of
        {ok, {{_, Status, _}, RespHeaders, RespBody}} ->
            cowboy_req:reply(Status, maps:from_list([
                {list_to_binary(K), list_to_binary(V)}
                || {K, V} <- RespHeaders, K /= "transfer-encoding"
            ]), RespBody, Req2),
            {ok, Req2, State};
        {error, Reason} ->
            logger:error("Proxy error", #{url => Url, reason => Reason}),
            cowboy_req:reply(502, #{}, <<"Bad Gateway">>, Req2),
            {ok, Req2, State}
    end.

rate_limit_check(#{id := UserId}) ->
    Key = <<"rate_limit:", UserId/binary>>,
    case redis:incr(Key) of
        {ok, Count} when Count =:= 1 ->
            redis:expire(Key, 60),
            ok;
        {ok, Count} when Count =< 1000 ->  %% 1000 req/min
            ok;
        _ ->
            {error, limit_exceeded}
    end.

jwt_verify(Token) ->
    Secret = application:get_env(gateway, jwt_secret, <<>>),
    case joken:verify(Token, joken_config:default_claims(), Secret) of
        {ok, Claims} ->
            {ok, #{
                id => maps:get(<<"sub">>, Claims),
                role => maps:get(<<"role">>, Claims, <<"user">>)
            }};
        _ ->
            {error, unauthorized}
    end.
```

---

## สรุป Part 65

| Service | Responsibility | Key Tech |
|---------|---------------|---------|
| API Gateway | Auth, rate limit, routing | Cowboy, JWT |
| Order Service | Order lifecycle, saga | gen_server, PostgreSQL |
| Inventory Service | Stock management | ETS cache, transactions |
| Payment Service | Charge, refund | External API, retries |
| Event Bus | Async notifications | RabbitMQ |

---

*[← Part 64: Database Patterns](part_64_database_patterns.md) | [Part 66: RabbitMQ Internals →](part_66_rabbitmq.md)*
