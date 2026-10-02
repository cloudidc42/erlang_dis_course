# Part 76: Real-Time Bidding System

## สารบัญ
1. [RTB Architecture](#architecture)
2. [Bid Request Processing](#bid-processing)
3. [Budget Management](#budget)
4. [Frequency Capping](#frequency)
5. [Ad Selection Algorithm](#selection)
6. [ตัวอย่างจริง: Full RTB Server](#full-rtb)

---

## RTB Architecture

```
Real-Time Bidding (RTB):
- Publisher shows ad slot → sends bid request to SSP
- SSP sends bid request to multiple DSPs (in ~100ms total)
- DSPs calculate bid → respond with bid response
- Highest bidder wins → ad shown
- Entire process must complete in < 100ms

Erlang ทำไมเหมาะ:
- Concurrent bid processing (1000s simultaneous)
- Low GC pauses → consistent sub-10ms responses
- Fault tolerance → no single point of failure
- Hot code reload → deploy without downtime

Key metrics:
- Throughput: 100,000+ QPS
- P99 latency: < 10ms
- Budget accuracy: prevent overspend
```

---

## Bid Request Processing

```erlang
%% bid_handler.erl — cowboy HTTP handler for bid requests

-module(bid_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    T1 = erlang:monotonic_time(microsecond),
    
    case cowboy_req:method(Req) of
        <<"POST">> ->
            {ok, Body, Req2} = cowboy_req:read_body(Req, #{length => 65536, period => 5000}),
            handle_bid_request(Body, Req2, State, T1);
        _ ->
            {ok, cowboy_req:reply(405, Req), State}
    end.

handle_bid_request(Body, Req, State, T1) ->
    case parse_bid_request(Body) of
        {ok, BidRequest} ->
            case bid_engine:process(BidRequest) of
                {bid, BidResponse} ->
                    T2 = erlang:monotonic_time(microsecond),
                    metrics:histogram(bid_latency_us, T2 - T1),
                    metrics:counter(bids_won, 1),
                    
                    RespBody = jsx:encode(BidResponse),
                    {ok, cowboy_req:reply(200,
                        #{<<"content-type">> => <<"application/json">>},
                        RespBody, Req), State};
                
                no_bid ->
                    metrics:counter(no_bids, 1),
                    {ok, cowboy_req:reply(204, Req), State}
            end;
        
        {error, invalid} ->
            metrics:counter(invalid_requests, 1),
            {ok, cowboy_req:reply(400, Req), State}
    end.

parse_bid_request(Body) ->
    try
        #{
            <<"id">> := RequestId,
            <<"imp">> := Impressions,
            <<"site">> := Site,
            <<"user">> := User
        } = jsx:decode(Body, [return_maps]),
        
        {ok, #{
            id => RequestId,
            impressions => Impressions,
            site => Site,
            user => User,
            received_at => erlang:system_time(millisecond)
        }}
    catch _:_ ->
        {error, invalid}
    end.

%% bid_engine.erl — core bidding logic

-module(bid_engine).
-export([process/1]).

-define(MAX_PROCESS_TIME_MS, 80).  %% budget for processing

process(BidRequest) ->
    UserId = get_user_id(BidRequest),
    
    %% Parallel checks (must complete within budget)
    Deadline = erlang:monotonic_time(millisecond) + ?MAX_PROCESS_TIME_MS,
    
    #{
        eligible_campaigns := EligibleCampaigns,
        frequency_ok := FrequencyOk,
        user_segments := UserSegments
    } = parallel_lookup(UserId, BidRequest, Deadline),
    
    case FrequencyOk andalso EligibleCampaigns =/= [] of
        false -> no_bid;
        true ->
            %% Score and select best campaign
            Scored = score_campaigns(EligibleCampaigns, UserSegments, BidRequest),
            case select_winning_campaign(Scored) of
                {ok, Campaign, Bid} ->
                    record_bid(UserId, Campaign),
                    {bid, format_bid_response(BidRequest, Campaign, Bid)};
                no_candidate ->
                    no_bid
            end
    end.

parallel_lookup(UserId, BidRequest, Deadline) ->
    Parent = self(),
    
    %% Launch parallel lookups
    Ref1 = async(fun() -> campaign_store:find_eligible(BidRequest) end),
    Ref2 = async(fun() -> frequency_cap:check(UserId) end),
    Ref3 = async(fun() -> user_segmenter:get_segments(UserId) end),
    
    %% Collect results within deadline
    TimeoutMs = max(0, Deadline - erlang:monotonic_time(millisecond)),
    collect_results([
        {eligible_campaigns, Ref1, []},
        {frequency_ok, Ref2, true},
        {user_segments, Ref3, []}
    ], TimeoutMs).

async(Fun) ->
    Parent = self(),
    Ref = make_ref(),
    spawn(fun() -> Parent ! {Ref, Fun()} end),
    Ref.

collect_results([], _) -> #{};
collect_results([{Key, Ref, Default} | Rest], TimeoutMs) ->
    receive
        {Ref, Result} ->
            maps:merge(#{Key => Result}, collect_results(Rest, TimeoutMs))
    after TimeoutMs ->
        maps:merge(#{Key => Default}, collect_results(Rest, 0))
    end.
```

---

## Budget Management

```erlang
%% budget_manager.erl — prevent overspend with atomic counters

-module(budget_manager).
-behaviour(gen_server).
-export([start_link/0, reserve/3, confirm/2, release/2, get_spend/1]).
-export([init/1, handle_call/3, handle_cast/2]).

%% Uses ETS + counters for atomic operations without process bottleneck

init([]) ->
    ets:new(budgets, [named_table, public, {write_concurrency, true}]),
    ets:new(spend_log, [named_table, public, bag]),
    {ok, #{}}.

%% Try to reserve budget for a bid
reserve(CampaignId, BidAmount, RequestId) ->
    case ets:lookup(budgets, CampaignId) of
        [{CampaignId, Budget}] ->
            DailyLimit = maps:get(daily_limit, Budget),
            CurrentSpend = current_spend(CampaignId),
            
            case CurrentSpend + BidAmount =< DailyLimit of
                true ->
                    %% Atomic reservation using counter ref
                    Ref = {CampaignId, RequestId},
                    ets:insert(spend_log, {CampaignId, Ref, BidAmount, reserved}),
                    {ok, Ref};
                false ->
                    {error, budget_exhausted}
            end;
        [] ->
            {error, campaign_not_found}
    end.

%% Confirm spend (bid won)
confirm(CampaignId, Ref) ->
    ets:update_element(spend_log, {CampaignId, Ref}, {4, confirmed}),
    ok.

%% Release reservation (bid lost)
release(CampaignId, Ref) ->
    ets:delete(spend_log, {CampaignId, Ref}),
    ok.

current_spend(CampaignId) ->
    Entries = ets:lookup(spend_log, CampaignId),
    lists:sum([Amount || {_, _, Amount, _} <- Entries]).

get_spend(CampaignId) ->
    #{
        reserved => lists:sum([A || {_, _, A, reserved} <- ets:lookup(spend_log, CampaignId)]),
        confirmed => lists:sum([A || {_, _, A, confirmed} <- ets:lookup(spend_log, CampaignId)])
    }.

%% Periodic cleanup of old reservations
cleanup_stale_reservations() ->
    %% Remove reservations older than 1 second (bid response timeout)
    %% In practice, track timestamps and expire them
    ok.
```

---

## Frequency Capping

```erlang
%% frequency_cap.erl — limit ad impressions per user

-module(frequency_cap).
-export([check/1, record_impression/2, get_caps/1]).

-define(REDIS_POOL, redis_pool).
-define(CAP_TTL_SECONDS, 86400).  %% 24 hours

%% Check if user is under frequency cap for campaigns
check(UserId) ->
    %% Redis pipeline for efficiency
    Commands = [
        ["GET", user_key(UserId, daily)],
        ["GET", user_key(UserId, hourly)]
    ],
    
    case eredis:qp(redis_conn(), Commands) of
        [{ok, DailyCount}, {ok, HourlyCount}] ->
            DailyN = parse_count(DailyCount),
            HourlyN = parse_count(HourlyCount),
            
            DailyN < max_daily_impressions() andalso HourlyN < max_hourly_impressions();
        _ ->
            true  %% Redis error → allow bid (fail open)
    end.

record_impression(UserId, CampaignId) ->
    %% Increment counters atomically
    Pipeline = [
        ["INCR", user_key(UserId, daily)],
        ["EXPIRE", user_key(UserId, daily), 86400],
        ["INCR", user_key(UserId, hourly)],
        ["EXPIRE", user_key(UserId, hourly), 3600],
        ["INCR", campaign_user_key(CampaignId, UserId)],
        ["EXPIRE", campaign_user_key(CampaignId, UserId), 86400]
    ],
    eredis:qp(redis_conn(), Pipeline).

%% Check campaign-specific cap
check_campaign_cap(UserId, CampaignId, MaxImpressions) ->
    case eredis:q(redis_conn(), ["GET", campaign_user_key(CampaignId, UserId)]) of
        {ok, undefined} -> true;
        {ok, Count} -> binary_to_integer(Count) < MaxImpressions;
        _ -> true
    end.

user_key(UserId, daily) ->
    <<"fc:u:", UserId/binary, ":d">>;
user_key(UserId, hourly) ->
    <<"fc:u:", UserId/binary, ":h">>.

campaign_user_key(CampaignId, UserId) ->
    <<"fc:c:", CampaignId/binary, ":u:", UserId/binary>>.

parse_count(undefined) -> 0;
parse_count(Bin) when is_binary(Bin) -> binary_to_integer(Bin).

max_daily_impressions() -> 50.
max_hourly_impressions() -> 10.
```

---

## ตัวอย่างจริง: Full RTB Server

```erlang
%% rtb_server.erl — main application startup

-module(rtb_app).
-behaviour(application).
-export([start/2, stop/1]).

start(normal, _Args) ->
    %% Initialize caches
    campaign_store:init(),
    frequency_cap:init(),
    
    %% Start Redis pool
    {ok, _} = poolboy:start_link(
        [{name, {local, redis_pool}}, {worker_module, redis_worker}, {size, 50}],
        [{host, os:getenv("REDIS_HOST", "localhost")}]
    ),
    
    %% Start Cowboy with high performance settings
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/bid", bid_handler, []},
            {"/win", win_handler, []},     %% win notification
            {"/health", health_handler, []}
        ]}
    ]),
    
    {ok, _} = cowboy:start_clear(http, [
        {port, 8080},
        {num_acceptors, 100},  %% high concurrency
        {max_connections, 50000}
    ], #{
        env => #{dispatch => Dispatch},
        request_timeout => 100,     %% 100ms max request time
        idle_timeout => 5000
    }),
    
    rtb_sup:start_link().

%% Ad scoring algorithm
score_campaigns(Campaigns, UserSegments, BidRequest) ->
    [score_campaign(C, UserSegments, BidRequest) || C <- Campaigns].

score_campaign(Campaign, UserSegments, BidRequest) ->
    %% Base CPM bid
    BaseBid = maps:get(base_bid_cpm, Campaign),
    
    %% Audience match multiplier
    AudienceMultiplier = calculate_audience_multiplier(
        maps:get(target_segments, Campaign, []),
        UserSegments
    ),
    
    %% Contextual relevance
    SiteCategory = get_in(BidRequest, [site, cat]),
    ContextMultiplier = calculate_context_multiplier(Campaign, SiteCategory),
    
    %% Budget pacing (slow down if ahead of pace)
    PacingMultiplier = calculate_pacing(Campaign),
    
    FinalBid = BaseBid * AudienceMultiplier * ContextMultiplier * PacingMultiplier,
    
    {Campaign, FinalBid}.

calculate_audience_multiplier(TargetSegments, UserSegments) ->
    Matches = length([S || S <- TargetSegments, lists:member(S, UserSegments)]),
    1.0 + (Matches * 0.3).  %% +30% per matching segment

calculate_context_multiplier(Campaign, SiteCategories) ->
    CampaignCategories = maps:get(target_categories, Campaign, []),
    case lists:any(fun(C) -> lists:member(C, SiteCategories) end, CampaignCategories) of
        true -> 1.5;   %% +50% for contextual match
        false -> 1.0
    end.

calculate_pacing(Campaign) ->
    %% Linear pacing: if spent 50% of budget but only 40% through the day → slow down
    DayFraction = day_fraction(),
    SpendFraction = budget_manager:spend_fraction(maps:get(id, Campaign)),
    
    if
        SpendFraction > DayFraction * 1.2 -> 0.5;  %% significantly ahead → slow down
        SpendFraction > DayFraction -> 0.8;          %% slightly ahead → reduce
        true -> 1.0                                   %% on pace or behind → normal
    end.

day_fraction() ->
    {_, {H, M, S}} = calendar:universal_time(),
    (H * 3600 + M * 60 + S) / 86400.

select_winning_campaign([]) -> no_candidate;
select_winning_campaign(ScoredCampaigns) ->
    %% Second-price auction: winner pays second-highest bid + $0.01
    Sorted = lists:reverse(lists:keysort(2, ScoredCampaigns)),
    
    case Sorted of
        [{WinnerCampaign, WinnerBid} | [{_, SecondBid} | _]] ->
            ClearingPrice = SecondBid + 0.01,
            {ok, WinnerCampaign, ClearingPrice};
        [{WinnerCampaign, WinnerBid}] ->
            {ok, WinnerCampaign, WinnerBid * 0.8};  %% no competition → pay 80% of bid
        [] ->
            no_candidate
    end.

format_bid_response(BidRequest, Campaign, BidAmount) ->
    #{
        <<"id">> => maps:get(id, BidRequest),
        <<"seatbid">> => [#{
            <<"bid">> => [#{
                <<"id">> => generate_bid_id(),
                <<"impid">> => get_impression_id(BidRequest),
                <<"price">> => BidAmount,
                <<"adid">> => maps:get(ad_id, Campaign),
                <<"adm">> => maps:get(creative, Campaign),
                <<"crid">> => maps:get(creative_id, Campaign),
                <<"w">> => maps:get(width, Campaign),
                <<"h">> => maps:get(height, Campaign)
            }]
        }],
        <<"cur">> => <<"USD">>
    }.

get_in(Map, [Key]) -> maps:get(Key, Map, undefined);
get_in(Map, [Key | Rest]) ->
    case maps:get(Key, Map, undefined) of
        SubMap when is_map(SubMap) -> get_in(SubMap, Rest);
        _ -> undefined
    end.

generate_bid_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).

get_impression_id(BidRequest) ->
    [Imp | _] = maps:get(impressions, BidRequest),
    maps:get(<<"id">>, Imp).

get_user_id(BidRequest) ->
    User = maps:get(user, BidRequest, #{}),
    maps:get(<<"id">>, User, <<"unknown">>).

record_bid(UserId, Campaign) ->
    frequency_cap:record_impression(UserId, maps:get(id, Campaign)).
```

---

## สรุป Part 76

| RTB Component | Approach |
|--------------|---------|
| Bid handler | Cowboy HTTP, 100ms timeout |
| Parallel lookup | Spawn + collect pattern |
| Budget | ETS atomic reservations |
| Frequency cap | Redis INCR pipeline |
| Scoring | Pure function, multipliers |
| Auction | Second-price (Vickrey) |

**Performance targets**: < 10ms P99, 100K+ QPS, zero budget overspend

---

*[← Part 75: IoT](part_75_iot.md) | [Part 77: Zero-Downtime Deployment →](part_77_zero_downtime.md)*
