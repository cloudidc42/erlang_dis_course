# Part 95: Final Project — E-Commerce Platform

## สารบัญ
1. [System Architecture](#architecture)
2. [Product Catalog Service](#catalog)
3. [Order Processing Service](#orders)
4. [Payment Integration](#payment)
5. [Search and Recommendations](#search)
6. [ตัวอย่างจริง: Complete E-Commerce Backend](#complete)

---

## System Architecture

```
E-Commerce Platform Architecture:

  API Gateway (cowboy)
        │
        ├── /api/v1/products ──── ProductService (gen_server + PostgreSQL)
        ├── /api/v1/cart     ──── CartService (gen_server + Redis)
        ├── /api/v1/orders   ──── OrderService (Saga pattern + PostgreSQL)
        ├── /api/v1/payments ──── PaymentService (stripe/paypal integration)
        ├── /api/v1/search   ──── SearchService (elasticsearch client)
        └── /ws/events       ──── EventStream (WebSocket)

  Background Services:
        ├── InventorySync (monitors stock levels)
        ├── PriceEngine (dynamic pricing rules)
        ├── RecommendationEngine (collaborative filtering)
        └── NotificationService (email/SMS/push)

  Storage:
        ├── PostgreSQL (orders, users, products)
        ├── Redis (sessions, cart, price cache)
        ├── Elasticsearch (product search)
        └── S3 (product images, invoices)
```

---

## Product Catalog Service

```erlang
%% product_service.erl — product catalog with inventory

-module(product_service).
-behaviour(gen_server).
-export([start_link/0, get/1, list/1, create/1, update/2,
         update_inventory/2, get_inventory/1]).
-export([init/1, handle_call/3]).

-record(product, {
    id,
    sku,
    name,
    description,
    price,           %% in cents (avoid float issues)
    category,
    tags = [],
    images = [],
    active = true,
    inventory = 0,
    version = 1      %% optimistic locking
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    {ok, #{}}.

get(ProductId) ->
    case cache:get({product, ProductId}) of
        {ok, Product} -> {ok, Product};
        miss ->
            case db_products:get(ProductId) of
                {ok, Product} ->
                    cache:set({product, ProductId}, Product, 300),
                    {ok, Product};
                not_found -> not_found
            end
    end.

list(Filters) ->
    Page = maps:get(page, Filters, 1),
    PageSize = min(maps:get(page_size, Filters, 20), 100),
    Category = maps:get(category, Filters, undefined),
    MinPrice = maps:get(min_price, Filters, undefined),
    MaxPrice = maps:get(max_price, Filters, undefined),
    
    db_products:list(#{
        page => Page, page_size => PageSize,
        category => Category,
        min_price => MinPrice, max_price => MaxPrice,
        active => true
    }).

create(Data) ->
    case validate_product(Data) of
        {ok, Valid} ->
            Product = maps:merge(Valid, #{
                id => uuid:generate(),
                created_at => erlang:system_time(second)
            }),
            case db_products:create(Product) of
                {ok, Created} ->
                    %% Index in Elasticsearch
                    search_indexer:index(product, Created),
                    {ok, Created};
                Error -> Error
            end;
        {error, _} = E -> E
    end.

update(ProductId, Updates) ->
    case get(ProductId) of
        {ok, Product} ->
            case validate_product_update(Updates) of
                {ok, Valid} ->
                    Updated = maps:merge(product_to_map(Product), Valid),
                    NewVersion = Product#product.version + 1,
                    
                    case db_products:update_with_version(ProductId, Updated, Product#product.version, NewVersion) of
                        {ok, NewProduct} ->
                            cache:delete({product, ProductId}),
                            search_indexer:update(product, ProductId, NewProduct),
                            {ok, NewProduct};
                        {error, version_conflict} ->
                            %% Retry
                            update(ProductId, Updates);
                        Error -> Error
                    end;
                Error -> Error
            end;
        not_found -> not_found
    end.

update_inventory(ProductId, QuantityDelta) ->
    case db_products:atomic_update_inventory(ProductId, QuantityDelta) of
        {ok, NewQuantity} ->
            cache:delete({product, ProductId}),
            case NewQuantity < 5 of
                true -> inventory_alert:notify(ProductId, NewQuantity);
                false -> ok
            end,
            {ok, NewQuantity};
        {error, insufficient_inventory} ->
            {error, out_of_stock}
    end.

get_inventory(ProductId) ->
    db_products:get_inventory(ProductId).

validate_product(Data) ->
    validator:validate(Data, #{
        name => #{type => string, required => true, max_length => 200},
        sku => #{type => string, required => true, max_length => 50},
        price => #{type => integer, required => true, min => 1},
        category => #{type => string, required => true}
    }).

validate_product_update(Data) ->
    validator:validate(Data, #{
        name => #{type => string, max_length => 200},
        price => #{type => integer, min => 1},
        active => #{type => boolean}
    }).

product_to_map(#product{} = P) ->
    #{id => P#product.id, name => P#product.name, price => P#product.price,
      category => P#product.category, active => P#product.active}.
```

---

## Order Processing Service

```erlang
%% order_service.erl — complete order lifecycle

-module(order_service).
-export([create_order/2, get_order/1, cancel_order/2, list_by_user/2]).

-record(order, {
    id,
    user_id,
    items,
    subtotal,
    tax,
    shipping,
    total,
    status,          %% pending | confirmed | shipped | delivered | cancelled | refunded
    payment_id,
    shipping_address,
    created_at,
    updated_at
}).

create_order(UserId, #{items := Items, shipping_address := Address}) ->
    %% Use Saga pattern for distributed transaction
    OrderId = uuid:generate(),
    
    SagaSteps = [
        {reserve_inventory, fun() -> reserve_items(Items) end,
                            fun(ResId) -> cancel_reservation(ResId) end},
        
        {calculate_pricing, fun() -> calculate_order_pricing(Items, Address) end,
                            fun(_) -> ok end},  %% no compensation needed
        
        {create_order_record, fun(Pricing) ->
            create_order_in_db(OrderId, UserId, Items, Address, Pricing)
        end, fun(_) -> db:execute("DELETE FROM orders WHERE id = $1", [OrderId]) end},
        
        {process_payment, fun() ->
            Payment = get_payment_method(UserId),
            payment_service:charge(Payment, get_order_total(OrderId))
        end, fun(PayId) -> payment_service:refund(PayId) end}
    ],
    
    run_saga(SagaSteps, #{order_id => OrderId}).

run_saga(Steps, Context) ->
    run_saga(Steps, Context, []).

run_saga([], Context, _Compensations) ->
    {ok, maps:get(order_id, Context)};

run_saga([{StepName, StepFun, CompFun} | Rest], Context, Comps) ->
    logger:info("Saga step", #{step => StepName}),
    
    try
        Result = StepFun(),
        NewComps = [{CompFun, Result} | Comps],
        NewContext = Context#{StepName => Result},
        run_saga(Rest, NewContext, NewComps)
    catch Class:Reason ->
        logger:error("Saga step failed", #{step => StepName, class => Class, reason => Reason}),
        
        %% Run compensations in reverse order
        [begin
            try CompensationFun(R)
            catch _:_ -> ok  %% best effort compensation
            end
        end || {CompensationFun, R} <- Comps],
        
        {error, {StepName, {Class, Reason}}}
    end.

reserve_items(Items) ->
    %% Reserve each item atomically
    ReservationIds = [begin
        ProductId = maps:get(product_id, Item),
        Qty = maps:get(quantity, Item),
        case product_service:update_inventory(ProductId, -Qty) of
            {ok, _} -> {ProductId, Qty};
            {error, out_of_stock} -> throw({out_of_stock, ProductId})
        end
    end || Item <- Items],
    ReservationIds.

cancel_reservation(ReservationIds) ->
    [product_service:update_inventory(ProdId, Qty) || {ProdId, Qty} <- ReservationIds].

calculate_order_pricing(Items, Address) ->
    ItemsWithPrices = [begin
        {ok, Product} = product_service:get(maps:get(product_id, I)),
        Item = I#{price => maps:get(price, Product),
                   name => maps:get(name, Product)},
        Item
    end || I <- Items],
    
    Subtotal = lists:sum([maps:get(price, I) * maps:get(quantity, I) || I <- ItemsWithPrices]),
    Tax = calculate_tax(Subtotal, Address),
    Shipping = calculate_shipping(ItemsWithPrices, Address),
    
    #{
        items => ItemsWithPrices,
        subtotal => Subtotal,
        tax => Tax,
        shipping => Shipping,
        total => Subtotal + Tax + Shipping
    }.

get_order(OrderId) ->
    case db:execute(
        "SELECT id, user_id, items, subtotal, tax, shipping, total, "
        "       status, payment_id, shipping_address, created_at "
        "FROM orders WHERE id = $1",
        [OrderId]
    ) of
        {ok, [Row]} -> {ok, row_to_order(Row)};
        {ok, []} -> not_found
    end.

cancel_order(OrderId, Reason) ->
    case get_order(OrderId) of
        {ok, #order{status = Status} = Order} when Status =:= pending; Status =:= confirmed ->
            %% Refund payment
            case Order#order.payment_id of
                undefined -> ok;
                PaymentId -> payment_service:refund(PaymentId)
            end,
            
            %% Restore inventory
            [product_service:update_inventory(maps:get(product_id, I), maps:get(quantity, I))
             || I <- Order#order.items],
            
            %% Update status
            db:execute("UPDATE orders SET status = 'cancelled', "
                        "cancel_reason = $2, updated_at = NOW() WHERE id = $1",
                        [OrderId, Reason]),
            
            notification_service:send(order_cancelled, #{order => Order, reason => Reason}),
            ok;
        
        {ok, #order{status = Status}} ->
            {error, {cannot_cancel, Status}};
        not_found ->
            not_found
    end.

list_by_user(UserId, #{page := Page, page_size := PageSize}) ->
    db:execute(
        "SELECT id, status, total, created_at FROM orders "
        "WHERE user_id = $1 ORDER BY created_at DESC "
        "LIMIT $2 OFFSET $3",
        [UserId, PageSize, (Page - 1) * PageSize]
    ).

calculate_tax(Subtotal, _Address) ->
    %% Simplified: 7% flat tax
    round(Subtotal * 0.07).

calculate_shipping(Items, _Address) ->
    TotalWeight = lists:sum([maps:get(weight_g, I, 500) * maps:get(quantity, I) || I <- Items]),
    
    if TotalWeight < 500 -> 0;        %% Free shipping under 500g
       TotalWeight < 2000 -> 5000;    %% 50 THB
       true -> 10000                   %% 100 THB
    end.

get_order_total(OrderId) ->
    case db:execute("SELECT total FROM orders WHERE id = $1", [OrderId]) of
        {ok, [{Total}]} -> Total;
        _ -> 0
    end.

get_payment_method(UserId) ->
    case db:execute(
        "SELECT id, type, last_four, token FROM payment_methods "
        "WHERE user_id = $1 AND is_default = true LIMIT 1",
        [UserId]
    ) of
        {ok, [Row]} -> row_to_payment_method(Row);
        _ -> throw(no_payment_method)
    end.

row_to_order({Id, UserId, ItemsJson, Subtotal, Tax, Shipping, Total, Status, PayId, AddrJson, CreatedAt}) ->
    #order{
        id = Id, user_id = UserId,
        items = jiffy:decode(ItemsJson, [return_maps]),
        subtotal = Subtotal, tax = Tax, shipping = Shipping, total = Total,
        status = binary_to_atom(Status), payment_id = PayId,
        shipping_address = jiffy:decode(AddrJson, [return_maps]),
        created_at = CreatedAt
    }.

row_to_payment_method({Id, Type, LastFour, Token}) ->
    #{id => Id, type => Type, last_four => LastFour, token => Token}.

create_order_in_db(OrderId, UserId, _Items, Address, Pricing) ->
    db:execute(
        "INSERT INTO orders (id, user_id, items, subtotal, tax, shipping, total, "
        "                    status, shipping_address, created_at) "
        "VALUES ($1, $2, $3, $4, $5, $6, $7, 'pending', $8, NOW())",
        [OrderId, UserId,
         jiffy:encode(maps:get(items, Pricing)),
         maps:get(subtotal, Pricing),
         maps:get(tax, Pricing),
         maps:get(shipping, Pricing),
         maps:get(total, Pricing),
         jiffy:encode(Address)]
    ),
    OrderId.
```

---

## Payment Integration

```erlang
%% payment_service.erl — payment processing

-module(payment_service).
-export([charge/2, refund/1, get_payment/1]).

-define(STRIPE_API, "https://api.stripe.com/v1").

charge(#{type := stripe, token := Token}, AmountCents) ->
    case stripe_api:create_charge(#{
        amount => AmountCents,
        currency => <<"thb">>,
        source => Token,
        description => <<"E-Commerce Order">>
    }) of
        {ok, #{<<"id">> := ChargeId, <<"status">> := <<"succeeded">>}} ->
            {ok, ChargeId};
        {ok, #{<<"status">> := <<"failed">>, <<"failure_message">> := Msg}} ->
            {error, {payment_declined, Msg}};
        {error, Reason} ->
            {error, Reason}
    end;

charge(#{type := paypal, token := Token}, AmountCents) ->
    case paypal_api:capture_order(Token, AmountCents) of
        {ok, CaptureId} -> {ok, CaptureId};
        Error -> Error
    end.

refund(PaymentId) ->
    case stripe_api:create_refund(#{charge => PaymentId}) of
        {ok, _} -> ok;
        Error -> Error
    end.

%% stripe_api.erl — Stripe HTTP client
-module(stripe_api).
-export([create_charge/1, create_refund/1]).

create_charge(Params) ->
    post("/charges", Params).

create_refund(Params) ->
    post("/refunds", Params).

post(Path, Params) ->
    ApiKey = application:get_env(my_app, stripe_api_key, ""),
    
    Body = uri_string:compose_query([{atom_to_binary(K), to_string(V)}
                                      || {K, V} <- maps:to_list(Params)]),
    
    case hackney:post(<<?STRIPE_API, Path/binary>>,
                       [{<<"Authorization">>, <<"Bearer ", ApiKey>>},
                        {<<"Content-Type">>, <<"application/x-www-form-urlencoded">>}],
                       Body, [with_body]) of
        {ok, 200, _Headers, ResponseBody} ->
            {ok, jiffy:decode(ResponseBody, [return_maps])};
        {ok, StatusCode, _Headers, ResponseBody} ->
            #{<<"error">> := Error} = jiffy:decode(ResponseBody, [return_maps]),
            {error, {StatusCode, Error}};
        Error ->
            Error
    end.

to_string(V) when is_binary(V) -> V;
to_string(V) when is_integer(V) -> integer_to_binary(V);
to_string(V) when is_atom(V) -> atom_to_binary(V).
```

---

## ตัวอย่างจริง: Complete E-Commerce Backend

```erlang
%% ecommerce_app.erl — complete application setup

-module(ecommerce_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_Type, _Args) ->
    ecommerce_sup:start_link().

-module(ecommerce_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Infrastructure
        {db_pool, {db_pool, start_link, [#{size => 20}]}, permanent, 5000, worker, [db_pool]},
        {redis_pool, {redis_pool, start_link, [#{size => 10}]}, permanent, 5000, worker, [redis_pool]},
        
        %% Domain services
        {product_service, {product_service, start_link, []}, permanent, 5000, worker, [product_service]},
        {cart_service, {cart_service, start_link, []}, permanent, 5000, worker, [cart_service]},
        {notification_service, {notification_service, start_link, []}, permanent, 5000, worker, []},
        
        %% HTTP server
        {ecommerce_http, {ecommerce_http, start_link, []}, permanent, 5000, worker, []}
    ],
    
    {ok, {#{strategy => one_for_one, intensity => 5, period => 10}, 
           [maps:from_list([{id, Id}, {start, Start}, {restart, R}, {shutdown, S}, {type, T}, {modules, M}])
            || {Id, Start, R, S, T, M} <- Children]}}.

-module(ecommerce_http).
-export([start_link/0]).

start_link() ->
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/api/v1/products",           products_handler, []},
            {"/api/v1/products/:id",        product_handler, []},
            {"/api/v1/cart",               cart_handler, []},
            {"/api/v1/cart/:item_id",       cart_item_handler, []},
            {"/api/v1/orders",             orders_handler, []},
            {"/api/v1/orders/:id",          order_handler, []},
            {"/api/v1/users/me",           user_handler, []},
            {"/api/v1/users/me/orders",    user_orders_handler, []},
            {"/api/v1/search",             search_handler, []},
            {"/health",                    health_handler, []},
            {"/metrics",                   prometheus_cowboy2_handler, []}
        ]}
    ]),
    
    cowboy:start_clear(ecommerce_http, [{port, 8080}, {num_acceptors, 100}], #{
        env => #{dispatch => Dispatch},
        middlewares => [
            cowboy_router,
            request_id_middleware,   %% add X-Request-ID
            tracing_middleware,      %% OpenTelemetry
            auth_middleware,         %% JWT verification
            rate_limit_middleware,   %% Redis sliding window
            metrics_middleware,      %% Prometheus
            cowboy_handler
        ],
        request_timeout => 30000
    }).
```

---

## สรุป Part 95

| Component | Technology | Pattern |
|-----------|-----------|---------|
| API | cowboy REST | cowboy_rest behaviour |
| Auth | JWT | HMAC-SHA256 tokens |
| Orders | Saga | Compensating transactions |
| Inventory | PostgreSQL | Atomic counter updates |
| Cart | Redis | TTL-based session storage |
| Payments | Stripe API | HTTP integration |
| Search | Elasticsearch | Full-text + facets |
| Cache | Redis + ETS | Multi-tier caching |

**Production checklist**:
- [ ] Input validation at every endpoint
- [ ] Rate limiting per user and IP
- [ ] Idempotency keys for payment endpoints
- [ ] Audit log for all order changes
- [ ] Inventory alerts at low stock
- [ ] Payment webhook handlers (Stripe events)

---

*[← Part 94: Chat System](part_94_final_project_chat.md) | [Part 96: World-Class Patterns →](part_96_world_class_patterns.md)*
