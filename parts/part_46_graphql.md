# Part 46: GraphQL กับ Erlang

## สารบัญ
1. [GraphQL คืออะไร](#graphql-คืออะไร)
2. [Absinthe (GraphQL for Erlang/Elixir)](#absinthe)
3. [Schema Definition](#schema-definition)
4. [Resolvers](#resolvers)
5. [Subscriptions](#subscriptions)
6. [ตัวอย่างจริง: Blog API](#ตัวอย่างจริง-blog-api)

---

## GraphQL คืออะไร

```
GraphQL: query language สำหรับ APIs
- Client ระบุ field ที่ต้องการเอง (ไม่ได้รับมากหรือน้อยเกินไป)
- Single endpoint: POST /graphql
- Schema-first: strongly typed
- Mutations สำหรับ write operations
- Subscriptions สำหรับ real-time

เทียบกับ REST:
REST: GET /users/1/posts/5/comments → over-fetching
GraphQL: { user(id:1) { posts(id:5) { comments { author text } } } } → exact data
```

---

## Absinthe

```erlang
%% Absinthe: GraphQL implementation สำหรับ Erlang ecosystem
%% Note: Absinthe เป็น Elixir library แต่ concept เหมือนกัน
%% สำหรับ pure Erlang ใช้: graphql-erlang หรือ erql

%% rebar.config
%% {deps, [
%%     {graphql, "0.16.0"},  %% shopify/graphql-erlang
%%     {cowboy, "2.10.0"}
%% ]}

%% Schema definition file: priv/schema.graphql
%% type Query {
%%   user(id: ID!): User
%%   users(page: Int, pageSize: Int): [User!]!
%%   post(id: ID!): Post
%% }
%%
%% type Mutation {
%%   createUser(input: CreateUserInput!): User!
%%   updateUser(id: ID!, input: UpdateUserInput!): User!
%%   deleteUser(id: ID!): Boolean!
%% }
%%
%% type Subscription {
%%   userCreated: User!
%%   postPublished(authorId: ID): Post!
%% }
%%
%% type User {
%%   id: ID!
%%   name: String!
%%   email: String!
%%   posts: [Post!]!
%%   createdAt: String!
%% }
%%
%% type Post {
%%   id: ID!
%%   title: String!
%%   body: String!
%%   author: User!
%%   comments: [Comment!]!
%%   publishedAt: String
%% }
%%
%% type Comment {
%%   id: ID!
%%   text: String!
%%   author: User!
%% }
%%
%% input CreateUserInput {
%%   name: String!
%%   email: String!
%%   password: String!
%% }
%%
%% input UpdateUserInput {
%%   name: String
%%   email: String
%% }
```

---

## Schema Definition

```erlang
%% schema.erl: GraphQL schema definition using graphql-erlang
-module(schema).
-export([load/0]).

load() ->
    SchemaData = {ok, Bin} = file:read_file("priv/schema.graphql"),
    {ok, Schema} = graphql:load_schema(#{
        scalars => #{
            'DateTime' => scalar_datetime,
            'JSON' => scalar_json
        }
    }, Bin),
    graphql:validate_schema(Schema),
    graphql:load_schema(Schema).

%% Custom scalar: DateTime
-module(scalar_datetime).
-export([input/2, output/2]).

input(_Type, Value) when is_binary(Value) ->
    case calendar:rfc3339_to_system_time(binary_to_list(Value)) of
        {ok, Time} -> {ok, Time};
        error -> {error, bad_datetime}
    end;
input(_, _) ->
    {error, not_string}.

output(_Type, Value) when is_integer(Value) ->
    Str = calendar:system_time_to_rfc3339(Value),
    {ok, list_to_binary(Str)}.
```

---

## Resolvers

```erlang
%% resolvers.erl: field resolvers
-module(resolvers).
-export([execute/4]).

%% execute(Ctx, Obj, Field, Args) -> {ok, Value} | {error, Reason}

%% Query: user(id: ID!)
execute(Ctx, null, <<"user">>, #{<<"id">> := Id}) ->
    UserId = binary_to_integer(Id),
    case users_db:get(UserId) of
        {ok, User} -> {ok, User};
        {error, not_found} -> {error, <<"User not found">>}
    end;

%% Query: users(page, pageSize)
execute(_Ctx, null, <<"users">>, Args) ->
    Page = maps:get(<<"page">>, Args, 1),
    PageSize = maps:get(<<"pageSize">>, Args, 20),
    {ok, Users} = users_db:list(Page, PageSize),
    {ok, Users};

%% User.posts: load posts for a user (N+1 problem!)
execute(_Ctx, #{id := UserId}, <<"posts">>, _Args) ->
    {ok, Posts} = posts_db:by_user(UserId),
    {ok, Posts};

%% Post.author: load author for a post
execute(_Ctx, #{author_id := AuthorId}, <<"author">>, _Args) ->
    users_db:get(AuthorId);

%% Mutation: createUser
execute(Ctx, null, <<"createUser">>, #{<<"input">> := Input}) ->
    case auth:get_current_user(Ctx) of
        {ok, _} ->  %% Must be authenticated
            Name = maps:get(<<"name">>, Input),
            Email = maps:get(<<"email">>, Input),
            Password = maps:get(<<"password">>, Input),
            {SaltB64, HashB64} = auth:hash_password(Password),
            case users_db:create(Name, Email, SaltB64, HashB64) of
                {ok, User} ->
                    graphql_subscriptions:publish(user_created, User),
                    {ok, User};
                {error, duplicate_email} ->
                    {error, <<"Email already taken">>}
            end;
        {error, _} ->
            {error, <<"Authentication required">>}
    end;

%% Mutation: deleteUser
execute(Ctx, null, <<"deleteUser">>, #{<<"id">> := Id}) ->
    case auth:get_current_user(Ctx) of
        {ok, #{id := AdminId, roles := Roles}} ->
            case lists:member(<<"admin">>, Roles) of
                true ->
                    users_db:delete(binary_to_integer(Id)),
                    {ok, true};
                false ->
                    {error, <<"Admin required">>}
            end;
        _ ->
            {error, <<"Unauthorized">>}
    end;

execute(_Ctx, _Obj, Field, _Args) ->
    {error, {not_implemented, Field}}.

%% DataLoader: solve N+1 problem
%% Batch multiple user lookups into single query
-module(user_loader).
-export([load/2, load_many/2]).

load(LoaderPid, UserId) ->
    dataloader:load(LoaderPid, users, UserId).

load_many(LoaderPid, UserIds) ->
    dataloader:load_many(LoaderPid, users, UserIds).

%% Batch function: called once per request with all IDs
batch_users(UserIds) ->
    {ok, Users} = users_db:get_many(UserIds),
    maps:from_list([{U#{id}, U} || U <- Users]).
```

---

## Subscriptions

```erlang
%% graphql_subscriptions.erl: GraphQL subscriptions via WebSocket
-module(graphql_subscriptions).
-behaviour(gen_server).
-export([start_link/0, subscribe/3, unsubscribe/2, publish/2]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

subscribe(SubscriptionId, Topic, WsPid) ->
    gen_server:call(?MODULE, {subscribe, SubscriptionId, Topic, WsPid}).

unsubscribe(SubscriptionId, WsPid) ->
    gen_server:cast(?MODULE, {unsubscribe, SubscriptionId, WsPid}).

publish(Topic, Data) ->
    gen_server:cast(?MODULE, {publish, Topic, Data}).

init(_) ->
    ets:new(subs, [named_table, bag]),
    {ok, #{}}.

handle_call({subscribe, SubId, Topic, WsPid}, _From, State) ->
    ets:insert(subs, {Topic, SubId, WsPid}),
    monitor(process, WsPid),
    {reply, ok, State}.

handle_cast({unsubscribe, SubId, WsPid}, State) ->
    ets:match_delete(subs, {'_', SubId, WsPid}),
    {noreply, State};

handle_cast({publish, Topic, Data}, State) ->
    Encoded = jsx:encode(#{
        type => <<"data">>,
        payload => #{data => #{Topic => Data}}
    }),
    Subscribers = ets:lookup(subs, Topic),
    [WsPid ! {send, Encoded} || {_, _, WsPid} <- Subscribers],
    {noreply, State}.

%% GraphQL WebSocket Handler (Apollo protocol)
-module(graphql_ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2, websocket_info/2]).

init(Req, Opts) ->
    {cowboy_websocket, Req, Opts}.

websocket_init(State) ->
    {ok, State#{subscriptions => #{}}}.

websocket_handle({text, Data}, State) ->
    Msg = jsx:decode(Data, [return_maps]),
    handle_graphql_message(Msg, State).

%% Apollo WebSocket protocol
handle_graphql_message(#{<<"type">> := <<"connection_init">>}, State) ->
    Reply = jsx:encode(#{type => <<"connection_ack">>}),
    {[{text, Reply}], State};

handle_graphql_message(#{
    <<"type">> := <<"start">>,
    <<"id">> := SubId,
    <<"payload">> := #{<<"query">> := Query, <<"variables">> := Vars}
}, State) ->
    case graphql:execute(Query, Vars) of
        {subscription, Topic} ->
            graphql_subscriptions:subscribe(SubId, Topic, self()),
            Subs = maps:put(SubId, Topic, maps:get(subscriptions, State)),
            {ok, State#{subscriptions := Subs}};
        {ok, Result} ->
            Reply = jsx:encode(#{type => <<"data">>, id => SubId,
                                 payload => #{data => Result}}),
            {[{text, Reply}], State}
    end;

handle_graphql_message(#{<<"type">> := <<"stop">>, <<"id">> := SubId}, State) ->
    graphql_subscriptions:unsubscribe(SubId, self()),
    Subs = maps:remove(SubId, maps:get(subscriptions, State)),
    {ok, State#{subscriptions := Subs}}.

websocket_info({send, Data}, State) ->
    {[{text, Data}], State}.
```

---

## ตัวอย่างจริง: Blog API

```erlang
%% blog_api.erl: Complete GraphQL blog API

%% Cowboy handler for /graphql
-module(graphql_handler).
-export([init/2]).

init(Req, Opts) ->
    Method = cowboy_req:method(Req),
    handle(Method, Req, Opts).

handle(<<"POST">>, Req, Opts) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"query">> := Query} = jsx:decode(Body, [return_maps]),
    Variables = maps:get(<<"variables">>, jsx:decode(Body, [return_maps]), #{}),

    %% Build context (auth, dataloader)
    Ctx = build_context(Req2),
    
    case graphql:execute(#{
        document => Query,
        variables => Variables,
        context => Ctx,
        schema => blog_schema:get()
    }) of
        {ok, Result} ->
            reply(200, #{data => Result}, Req2, Opts);
        {error, Errors} ->
            reply(200, #{errors => Errors}, Req2, Opts)
    end;

handle(<<"GET">>, Req, Opts) ->
    %% GraphiQL playground
    Html = graphiql_html(),
    Req2 = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/html">>},
        Html, Req),
    {ok, Req2, Opts}.

build_context(Req) ->
    User = case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            case auth:verify_token(Token) of
                {ok, Claims} -> {ok, Claims};
                _ -> nil
            end;
        _ -> nil
    end,
    #{current_user => User,
      loader => dataloader:new(#{}),
      request_id => generate_id()}.

reply(Status, Body, Req, Opts) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Body), Req),
    {ok, Req2, Opts}.

%% Example queries:
%%
%% query GetBlog {
%%   posts(published: true, limit: 10) {
%%     id title
%%     author { name }
%%     comments { count }
%%   }
%% }
%%
%% mutation CreatePost($input: CreatePostInput!) {
%%   createPost(input: $input) {
%%     id title publishedAt
%%   }
%% }
%%
%% subscription OnNewPost {
%%   postPublished {
%%     id title
%%     author { name }
%%   }
%% }
```

---

## สรุป Part 46

| Feature | GraphQL | REST |
|---------|---------|------|
| Data fetching | Client-specified | Server-specified |
| Endpoints | Single /graphql | Multiple |
| Versioning | Schema evolution | URL versioning |
| Type system | Built-in | Optional (OpenAPI) |
| Real-time | Subscriptions | SSE/WebSocket |
| Over-fetching | Never | Common |
| Learning curve | Higher | Lower |

---

*[← Part 45: WebSockets](part_45_websockets.md) | [Part 47: Event-Driven Architecture →](part_47_event_driven.md)*
