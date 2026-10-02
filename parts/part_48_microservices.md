# Part 48: Microservices Architecture กับ Erlang

## สารบัญ
1. [Microservices คืออะไร](#microservices-คืออะไร)
2. [Service Discovery](#service-discovery)
3. [API Gateway](#api-gateway)
4. [Service Mesh Concepts](#service-mesh-concepts)
5. [Inter-Service Communication](#inter-service-communication)
6. [ตัวอย่างจริง: E-Commerce Platform](#ตัวอย่างจริง-e-commerce-platform)

---

## Microservices คืออะไร

```
Microservices: สถาปัตยกรรมที่แยก application เป็น small services
- แต่ละ service: เล็ก, focused, deployable อิสระ
- สื่อสารผ่าน APIs (REST, gRPC, Message Queue)
- แต่ละ service มี database ของตัวเอง
- Scale แต่ละ service อิสระ

Erlang/OTP เหมาะกับ Microservices:
- Lightweight processes (แทน Docker containers ภายใน service)
- Pattern: ใช้ Erlang nodes เป็น services
- OTP supervision: self-healing
- Hot code reload: zero-downtime updates
```

---

## Service Discovery

```erlang
%% Consul integration สำหรับ service discovery
%% deps: {consul, "0.1.0"}

-module(service_registry).
-behaviour(gen_server).
-export([start_link/0, register/2, deregister/1, discover/1, watch/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(CONSUL_HOST, "localhost").
-define(CONSUL_PORT, 8500).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Register this service with Consul
register(ServiceName, Config) ->
    gen_server:call(?MODULE, {register, ServiceName, Config}).

deregister(ServiceId) ->
    gen_server:cast(?MODULE, {deregister, ServiceId}).

%% Discover service instances
discover(ServiceName) ->
    gen_server:call(?MODULE, {discover, ServiceName}).

%% Watch for service changes
watch(ServiceName, Callback) ->
    gen_server:call(?MODULE, {watch, ServiceName, Callback}).

init(_) ->
    State = #{services => #{}, watchers => #{}},
    {ok, State}.

handle_call({register, Name, Config}, _From, State) ->
    Port = maps:get(port, Config, 8080),
    ServiceId = iolist_to_binary([Name, "-", atom_to_list(node())]),
    Body = jsx:encode(#{
        <<"ID">> => ServiceId,
        <<"Name">> => Name,
        <<"Address">> => get_host_ip(),
        <<"Port">> => Port,
        <<"Check">> => #{
            <<"HTTP">> => iolist_to_binary(["http://", get_host_ip(), ":", 
                                             integer_to_list(Port), "/health"]),
            <<"Interval">> => <<"10s">>,
            <<"Timeout">> => <<"3s">>
        },
        <<"Tags">> => maps:get(tags, Config, [])
    }),
    consul_http:put("/v1/agent/service/register", Body),
    schedule_deregistration_on_shutdown(ServiceId),
    {reply, {ok, ServiceId}, State};

handle_call({discover, ServiceName}, _From, State) ->
    case consul_http:get("/v1/health/service/" ++ binary_to_list(ServiceName) ++ "?passing") of
        {ok, Services} ->
            Instances = [extract_endpoint(S) || S <- Services],
            {reply, {ok, Instances}, State};
        {error, Reason} ->
            {reply, {error, Reason}, State}
    end.

extract_endpoint(#{<<"Service">> := #{<<"Address">> := Addr, <<"Port">> := Port}}) ->
    #{host => Addr, port => Port}.

get_host_ip() ->
    {ok, Hostname} = inet:gethostname(),
    {ok, {A, B, C, D}} = inet:getaddr(Hostname, inet),
    iolist_to_binary(io_lib:format("~B.~B.~B.~B", [A, B, C, D])).

%% DNS-based service discovery
discover_via_dns(ServiceName) ->
    DnsName = binary_to_list(ServiceName) ++ ".service.consul",
    case inet_res:resolve(DnsName, in, a) of
        {ok, Msg} ->
            Addresses = inet_dns:msg(Msg, anlist),
            [{inet_dns:rr(RR, data), default_port(ServiceName)} || RR <- Addresses];
        {error, Reason} ->
            {error, Reason}
    end.

schedule_deregistration_on_shutdown(ServiceId) ->
    %% Register SIGTERM handler
    erlang:process_flag(trap_exit, true).

consul_http:get(Path) ->
    Url = "http://" ++ ?CONSUL_HOST ++ ":" ++ integer_to_list(?CONSUL_PORT) ++ Path,
    case httpc:request(get, {Url, []}, [{timeout, 5000}], []) of
        {ok, {{_, 200, _}, _, Body}} ->
            {ok, jsx:decode(list_to_binary(Body), [return_maps])};
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## API Gateway

```erlang
%% api_gateway.erl: reverse proxy + routing + auth

-module(gateway_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Path = cowboy_req:path(Req),
    Method = cowboy_req:method(Req),
    
    %% Route to backend service
    case route(Path, Method) of
        {ok, Service, ServicePath} ->
            proxy_request(Service, ServicePath, Req, State);
        {error, not_found} ->
            reply(404, #{error => <<"Route not found">>}, Req, State)
    end.

route(<<"/api/users", Rest/binary>>, _) -> {ok, user_service, Rest};
route(<<"/api/orders", Rest/binary>>, _) -> {ok, order_service, Rest};
route(<<"/api/inventory", Rest/binary>>, _) -> {ok, inventory_service, Rest};
route(_, _) -> {error, not_found}.

proxy_request(ServiceName, ServicePath, Req, State) ->
    %% Get service instance
    {ok, Instances} = service_registry:discover(atom_to_binary(ServiceName)),
    Instance = load_balance(Instances),
    
    %% Build target URL
    Host = maps:get(host, Instance),
    Port = maps:get(port, Instance),
    Url = iolist_to_binary(["http://", Host, ":", integer_to_list(Port), ServicePath]),
    
    %% Forward headers (add tracing)
    Headers0 = cowboy_req:headers(Req),
    RequestId = cowboy_req:header(<<"x-request-id">>, Req, generate_id()),
    Headers = Headers0#{
        <<"x-request-id">> => RequestId,
        <<"x-forwarded-for">> => get_client_ip(Req),
        <<"x-forwarded-host">> => cowboy_req:host(Req)
    },
    
    %% Forward body
    {ok, Body, _} = cowboy_req:read_body(Req),
    Method = binary_to_atom(string:lowercase(cowboy_req:method(Req))),
    
    %% Make upstream request
    case httpc:request(Method, {binary_to_list(Url), 
                                headers_to_list(Headers), 
                                "application/json", Body},
                      [{timeout, 30000}], [{body_format, binary}]) of
        {ok, {{_, Status, _}, RespHeaders, RespBody}} ->
            Req2 = cowboy_req:reply(Status,
                list_to_map(RespHeaders),
                RespBody, Req),
            {ok, Req2, State};
        {error, timeout} ->
            reply(504, #{error => <<"Gateway timeout">>}, Req, State);
        {error, _} ->
            reply(502, #{error => <<"Bad gateway">>}, Req, State)
    end.

%% Round-robin load balancing
load_balance(Instances) ->
    N = ets:update_counter(gateway_counters, lb_counter, 1),
    Index = (N rem length(Instances)) + 1,
    lists:nth(Index, Instances).

%% Rate limiting per client IP
check_rate_limit(Req) ->
    Ip = get_client_ip(Req),
    case rate_limiter:check(Ip, 100, 60) of  %% 100 req/min
        ok -> ok;
        {error, rate_limited} -> {error, rate_limited}
    end.

get_client_ip(Req) ->
    case cowboy_req:header(<<"x-forwarded-for">>, Req) of
        undefined -> cowboy_req:peer(Req);
        Ip -> Ip
    end.

reply(Status, Body, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Body), Req),
    {ok, Req2, State}.
```

---

## Inter-Service Communication

```erlang
%% service_client.erl: resilient service-to-service communication
-module(service_client).
-export([call/3, call/4]).

-define(DEFAULT_TIMEOUT, 5000).
-define(MAX_RETRIES, 3).

call(Service, Path, Opts) ->
    call(Service, Path, Opts, ?DEFAULT_TIMEOUT).

call(Service, Path, Opts, Timeout) ->
    Method = maps:get(method, Opts, get),
    Body = maps:get(body, Opts, <<>>),
    Headers = maps:get(headers, Opts, #{}),
    
    call_with_retry(Service, Method, Path, Body, Headers, Timeout, ?MAX_RETRIES).

call_with_retry(_, _, _, _, _, _, 0) ->
    {error, max_retries_exceeded};
call_with_retry(Service, Method, Path, Body, Headers, Timeout, Retries) ->
    %% Circuit breaker check
    case circuit_breaker:call(Service, fun() ->
        do_call(Service, Method, Path, Body, Headers, Timeout)
    end) of
        {ok, Result} -> {ok, Result};
        {error, circuit_open} -> {error, service_unavailable};
        {error, Reason} when Retries > 0 ->
            %% Retry with exponential backoff
            Delay = trunc(100 * math:pow(2, ?MAX_RETRIES - Retries)),
            timer:sleep(Delay),
            call_with_retry(Service, Method, Path, Body, Headers, Timeout, Retries - 1);
        {error, Reason} ->
            {error, Reason}
    end.

do_call(Service, Method, Path, Body, Headers, Timeout) ->
    {ok, Instances} = service_registry:discover(atom_to_binary(Service)),
    #{host := Host, port := Port} = load_balance(Instances),
    Url = iolist_to_binary(["http://", Host, ":", integer_to_list(Port), Path]),
    
    case gun:open(binary_to_list(Host), Port) of
        {ok, ConnPid} ->
            {ok, _} = gun:await_up(ConnPid, Timeout),
            StreamRef = gun:Method(ConnPid, binary_to_list(Path), 
                                   maps:to_list(Headers), Body),
            case gun:await(ConnPid, StreamRef, Timeout) of
                {response, fin, Status, _Headers} ->
                    gun:close(ConnPid),
                    {ok, #{status => Status, body => <<>>}};
                {response, nofin, Status, _Headers} ->
                    {ok, RespBody} = gun:await_body(ConnPid, StreamRef, Timeout),
                    gun:close(ConnPid),
                    {ok, #{status => Status, body => RespBody}};
                {error, Reason} ->
                    gun:close(ConnPid),
                    {error, Reason}
            end;
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## ตัวอย่างจริง: E-Commerce Platform

```erlang
%% e_commerce architecture overview:
%%
%% ┌─────────────────────────────────────────────────────┐
%% │                  API Gateway :8080                   │
%% │   Auth  │  Rate Limit  │  Routing  │  Load Balance  │
%% └────────┬───────────────────────────────────────────┘
%%          │
%%    ┌─────┼──────────────────────────────────────┐
%%    │     │                                       │
%% ┌──▼──┐ ┌▼──────────┐ ┌──────────┐ ┌──────────┐
%% │User │ │  Order    │ │Inventory │ │ Payment  │
%% │Svc  │ │  Service  │ │ Service  │ │ Service  │
%% │:8081│ │  :8082    │ │  :8083   │ │  :8084   │
%% └──┬──┘ └─────┬─────┘ └────┬─────┘ └────┬─────┘
%%    │          │             │             │
%%    └──────────┴─────────────┴─────────────┘
%%                       │
%%              ┌─────────┴──────────┐
%%              │    Kafka Cluster    │
%%              │  order.created      │
%%              │  payment.processed  │
%%              │  inventory.reserved │
%%              └────────────────────┘

%% Each service is an Erlang OTP application
%% user_service_app.erl
-module(user_service_app).
-behaviour(application).
-export([start/2, stop/1]).

start(normal, _) ->
    %% Register with Consul
    service_registry:register(<<"user-service">>, #{port => 8081}),
    
    %% Start HTTP server
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/health", health_handler, []},
            {"/users", users_collection_handler, []},
            {"/users/:id", users_resource_handler, []}
        ]}
    ]),
    {ok, _} = cowboy:start_clear(user_listener, [{port, 8081}],
        #{env => #{dispatch => Dispatch},
          middlewares => [auth_middleware, cors_middleware, cowboy_router, cowboy_handler]}),
    
    user_service_sup:start_link().

stop(_) ->
    service_registry:deregister(<<"user-service">>).
```

---

## สรุป Part 48

| Concern | Solution |
|---------|----------|
| Service Discovery | Consul / DNS |
| Load Balancing | Round-robin / least-connections |
| API Gateway | Cowboy reverse proxy |
| Circuit Breaker | gen_statem (Part 37) |
| Retry Logic | Exponential backoff |
| Distributed Tracing | OpenTelemetry (Part 40) |
| Configuration | env vars + Consul KV |
| Secrets | Vault / env vars |

---

*[← Part 47: Event-Driven](part_47_event_driven.md) | [Part 49: Kubernetes Deployment →](part_49_kubernetes.md)*
