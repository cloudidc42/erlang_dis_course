# Part 98: Erlang for Financial Systems

## สารบัญ
1. [Decimal Arithmetic ที่แม่นยำ](#decimal)
2. [Double-Entry Accounting](#accounting)
3. [Event Sourcing สำหรับ Audit Trail](#event-sourcing)
4. [Idempotent Payments](#idempotent)
5. [Regulatory Compliance](#compliance)
6. [ตัวอย่างจริง: Payment Processing Engine](#payment-engine)

---

## Decimal Arithmetic ที่แม่นยำ

```erlang
%% NEVER use floats for money calculations
%% Use integers (smallest unit) or decimal module

%% Wrong — floating point errors
bad_money_calc() ->
    Price = 19.99,
    Tax = Price * 0.07,
    Total = Price + Tax,
    %% Total = 21.3893 (float imprecision!)
    Total.

%% Correct — use integer cents (smallest unit)
-module(money).
-export([new/2, add/2, subtract/2, multiply/2, format/1]).

%% Amount is always in smallest unit (cents for USD, satoshi for BTC)
-type money() :: #{amount := integer(), currency := atom()}.

new(AmountCents, Currency) ->
    #{amount => AmountCents, currency => Currency}.

add(#{amount := A, currency := C}, #{amount := B, currency := C}) ->
    #{amount => A + B, currency => C};
add(_, _) ->
    error(currency_mismatch).

subtract(#{amount := A, currency := C}, #{amount := B, currency := C}) when A >= B ->
    #{amount => A - B, currency => C};
subtract(_, _) ->
    error(insufficient_funds).

%% Multiply by a factor (e.g., 1.07 for 7% tax)
%% Use integer ratio: multiply(Price, {107, 100}) = 107/100 = 1.07
multiply(#{amount := A, currency := C}, {Numerator, Denominator}) ->
    %% Round half-up
    Result = (A * Numerator + Denominator div 2) div Denominator,
    #{amount => Result, currency => C}.

format(#{amount := Cents, currency := usd}) ->
    Dollars = Cents div 100,
    CentsPart = abs(Cents rem 100),
    io_lib:format("$~p.~2..0w", [Dollars, CentsPart]);
format(#{amount := Amount, currency := Currency}) ->
    io_lib:format("~p ~p", [Amount, Currency]).

%% Tax calculation: 7% = 7/100
apply_tax(Price) ->
    multiply(Price, {107, 100}).

%% Split: divide money N ways (banker's rounding)
split(#{amount := Amount, currency := C}, N) ->
    Base = Amount div N,
    Remainder = Amount rem N,
    %% First Remainder recipients get 1 extra cent
    [#{amount => Base + (if I =< Remainder -> 1; true -> 0 end),
       currency => C} || I <- lists:seq(1, N)].

%% Verify: sum of split equals original
verify_split(Original, Parts) ->
    Total = lists:foldl(fun(#{amount := A}, Acc) -> Acc + A end, 0, Parts),
    Total =:= maps:get(amount, Original).

%% Example usage
example() ->
    Price = new(1999, usd),     %% $19.99
    Tax   = multiply(Price, {7, 100}),   %% 7% = $1.40
    Total = add(Price, Tax),    %% $21.39
    
    io:format("Price: ~s~n", [format(Price)]),
    io:format("Tax:   ~s~n", [format(Tax)]),
    io:format("Total: ~s~n", [format(Total)]).
```

---

## Double-Entry Accounting

```erlang
%% Every financial transaction must balance: debits = credits
%% This prevents money creation/destruction bugs

-module(ledger).
-export([record_transaction/1, get_balance/1, verify_integrity/0]).

%% Transaction: list of journal entries that must sum to zero
-type entry() :: #{account := binary(), amount := integer()}.
-type transaction() :: #{
    id := binary(),
    entries := [entry()],
    description := binary(),
    timestamp := integer()
}.

record_transaction(#{entries := Entries} = Txn) ->
    %% Verify entries balance (sum must be zero)
    Sum = lists:sum([maps:get(amount, E) || E <- Entries]),
    case Sum of
        0 ->
            %% Persist atomically
            db:transaction(fun() ->
                TxnId = maps:get(id, Txn),
                db:insert(transactions, Txn),
                [db:insert(journal_entries, E#{txn_id => TxnId}) || E <- Entries]
            end);
        NonZero ->
            {error, {unbalanced_transaction, NonZero}}
    end.

get_balance(AccountId) ->
    %% Sum all entries for this account
    Rows = db:query(
        "SELECT COALESCE(SUM(amount), 0) as balance 
         FROM journal_entries 
         WHERE account = $1",
        [AccountId]
    ),
    case Rows of
        [{ok, Cols, [{Balance}]}] -> {ok, Balance};
        _ -> {error, query_failed}
    end.

%% Verify double-entry integrity
verify_integrity() ->
    %% Sum of ALL journal entries must be zero
    [{ok, _, [{Total}]}] = db:query(
        "SELECT COALESCE(SUM(amount), 0) FROM journal_entries",
        []
    ),
    case Total of
        0 -> ok;
        N -> {error, {ledger_imbalanced, N}}
    end.

%% Example: record a payment
record_payment(CustomerId, Amount, Description) ->
    TxnId = uuid:v4(),
    record_transaction(#{
        id => TxnId,
        description => Description,
        timestamp => erlang:system_time(second),
        entries => [
            %% Debit: customer liability decreases (positive = debit)
            #{account => <<"revenue">>, amount => Amount},
            %% Credit: cash increases (negative = credit in double-entry)
            #{account => <<"cash">>, amount => -Amount}
        ]
    }).

%% Transfer between accounts
record_transfer(FromAccount, ToAccount, Amount, Description) ->
    record_transaction(#{
        id => uuid:v4(),
        description => Description,
        timestamp => erlang:system_time(second),
        entries => [
            #{account => FromAccount, amount => Amount},
            #{account => ToAccount, amount => -Amount}
        ]
    }).
```

---

## Event Sourcing สำหรับ Audit Trail

```erlang
%% Financial systems require immutable audit trails
%% Event sourcing: store events, derive state from events

-module(account_aggregate).
-export([open/2, deposit/3, withdraw/3, get_balance/1, get_history/1]).

%% Events (never deleted, append-only)
-type event() ::
    {account_opened, #{owner := binary(), initial_balance := integer()}} |
    {deposited, #{amount := integer(), reference := binary()}} |
    {withdrawn, #{amount := integer(), reference := binary()}} |
    {frozen, #{reason := binary()}}.

open(AccountId, Owner) ->
    Event = {account_opened, #{
        owner => Owner,
        initial_balance => 0,
        timestamp => erlang:system_time(second)
    }},
    append_event(AccountId, Event).

deposit(AccountId, Amount, Reference) when Amount > 0 ->
    State = rebuild_state(AccountId),
    case maps:get(frozen, State, false) of
        true -> {error, account_frozen};
        false ->
            Event = {deposited, #{
                amount => Amount,
                reference => Reference,
                timestamp => erlang:system_time(second)
            }},
            append_event(AccountId, Event)
    end.

withdraw(AccountId, Amount, Reference) when Amount > 0 ->
    State = rebuild_state(AccountId),
    
    case State of
        #{frozen := true} ->
            {error, account_frozen};
        #{balance := Balance} when Balance < Amount ->
            {error, insufficient_funds};
        _ ->
            Event = {withdrawn, #{
                amount => Amount,
                reference => Reference,
                timestamp => erlang:system_time(second)
            }},
            append_event(AccountId, Event)
    end.

get_balance(AccountId) ->
    State = rebuild_state(AccountId),
    maps:get(balance, State, undefined).

get_history(AccountId) ->
    load_events(AccountId).

%% Rebuild current state from events (event replay)
rebuild_state(AccountId) ->
    Events = load_events(AccountId),
    lists:foldl(fun apply_event/2, #{}, Events).

apply_event({account_opened, #{initial_balance := Bal}}, _State) ->
    #{balance => Bal, frozen => false};

apply_event({deposited, #{amount := Amount}}, State) ->
    State#{balance => maps:get(balance, State) + Amount};

apply_event({withdrawn, #{amount := Amount}}, State) ->
    State#{balance => maps:get(balance, State) - Amount};

apply_event({frozen, _}, State) ->
    State#{frozen => true}.

%% Optimistic concurrency: prevent lost updates
append_event(AccountId, Event) ->
    db:transaction(fun() ->
        %% Lock the account row
        [{_, CurrentVersion}] = db:query(
            "SELECT version FROM accounts WHERE id = $1 FOR UPDATE",
            [AccountId]
        ),
        
        NewVersion = CurrentVersion + 1,
        
        %% Append event with version
        db:insert(account_events, #{
            account_id => AccountId,
            version => NewVersion,
            event => Event,
            created_at => erlang:system_time(second)
        }),
        
        %% Update version (optimistic lock)
        db:update(accounts, #{version => NewVersion},
                  #{id => AccountId, version => CurrentVersion}),
        
        {ok, NewVersion}
    end).

load_events(AccountId) ->
    db:query(
        "SELECT event FROM account_events WHERE account_id = $1 ORDER BY version ASC",
        [AccountId]
    ).
```

---

## Idempotent Payments

```erlang
%% Idempotency: same request produces same result
%% Critical for payments to prevent double-charges

-module(idempotent_payment).
-export([charge/3]).

%% Idempotency key: caller-generated unique ID for the request
charge(IdempotencyKey, CustomerId, Amount) ->
    case lookup_previous(IdempotencyKey) of
        {found, Result} ->
            %% Already processed, return same result
            logger:info("Idempotent replay", #{key => IdempotencyKey}),
            Result;
        
        not_found ->
            process_new_charge(IdempotencyKey, CustomerId, Amount)
    end.

process_new_charge(IdempotencyKey, CustomerId, Amount) ->
    %% Atomic: claim key and process
    db:transaction(fun() ->
        %% Try to claim the idempotency key
        case db:insert_unique(idempotency_keys, #{
            key => IdempotencyKey,
            status => processing,
            created_at => erlang:system_time(second)
        }) of
            ok ->
                %% We own this request, process it
                Result = do_charge(CustomerId, Amount),
                
                %% Store result for future replays
                db:update(idempotency_keys, #{
                    status => completed,
                    result => Result
                }, #{key => IdempotencyKey}),
                
                Result;
            
            {error, duplicate_key} ->
                %% Race condition: another process is handling same key
                %% Wait briefly and retry lookup
                timer:sleep(100),
                case lookup_previous(IdempotencyKey) of
                    {found, Result} -> Result;
                    not_found -> {error, concurrent_request}
                end
        end
    end).

lookup_previous(IdempotencyKey) ->
    case db:query(
        "SELECT status, result FROM idempotency_keys WHERE key = $1",
        [IdempotencyKey]
    ) of
        [{ok, _, [{<<"completed">>, Result}]}] -> {found, Result};
        [{ok, _, [_]}] -> not_found;  %% still processing
        [{ok, _, []}] -> not_found
    end.

do_charge(CustomerId, Amount) ->
    %% Actual payment processing
    case stripe:charge(CustomerId, Amount) of
        {ok, ChargeId} ->
            ledger:record_payment(CustomerId, Amount, ChargeId),
            {ok, ChargeId};
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## ตัวอย่างจริง: Payment Processing Engine

```erlang
%% Full payment processing pipeline

-module(payment_engine).
-behaviour(gen_server).
-export([start_link/0, process/1]).
-export([init/1, handle_call/3]).

-record(state, {
    fraud_model,
    rate_limiter,
    circuit_breaker
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    {ok, #state{
        fraud_model = fraud_model:load(),
        rate_limiter = rate_limiter:new(1000, 60),  %% 1000/min
        circuit_breaker = circuit_breaker:new(stripe, 5, 60000)
    }}.

process(#{
    idempotency_key := IdemKey,
    customer_id := CustomerId,
    amount := Amount,
    currency := Currency,
    metadata := Meta
} = Request) ->
    gen_server:call(?MODULE, {process, Request}, 30000).

handle_call({process, Request}, _From, State) ->
    Result = pipeline(Request, [
        fun validate_input/1,
        fun check_rate_limit/1,
        fun fraud_check/1,
        fun reserve_funds/1,
        fun execute_payment/1,
        fun record_to_ledger/1,
        fun emit_events/1
    ], State),
    {reply, Result, State}.

pipeline(Request, [], _State) ->
    {ok, Request};
pipeline(Request, [Step | Rest], State) ->
    case Step(Request) of
        {ok, UpdatedRequest} ->
            pipeline(UpdatedRequest, Rest, State);
        {error, _} = Error ->
            %% Rollback any partial work
            rollback(Request),
            Error
    end.

validate_input(#{amount := Amount}) when Amount =< 0 ->
    {error, invalid_amount};
validate_input(#{currency := Currency}) when Currency =:= <<>> ->
    {error, missing_currency};
validate_input(Request) ->
    {ok, Request}.

check_rate_limit(#{customer_id := CustomerId} = Request) ->
    case rate_limiter:check(payment_limiter, CustomerId) of
        allow -> {ok, Request};
        deny -> {error, rate_limited}
    end.

fraud_check(#{customer_id := CustomerId, amount := Amount} = Request) ->
    Score = fraud_model:score(CustomerId, Amount),
    case Score > 0.8 of  %% 80% fraud probability threshold
        true ->
            logger:warning("High fraud score", #{
                customer => CustomerId,
                score => Score
            }),
            {error, suspected_fraud};
        false ->
            {ok, Request#{fraud_score => Score}}
    end.

reserve_funds(#{customer_id := CustomerId, amount := Amount} = Request) ->
    case account_aggregate:withdraw(CustomerId, Amount, pending) of
        {ok, ReservationId} ->
            {ok, Request#{reservation_id => ReservationId}};
        {error, insufficient_funds} ->
            {error, insufficient_funds}
    end.

execute_payment(#{customer_id := CustomerId, amount := Amount,
                  idempotency_key := IdemKey} = Request) ->
    case idempotent_payment:charge(IdemKey, CustomerId, Amount) of
        {ok, ChargeId} ->
            {ok, Request#{charge_id => ChargeId}};
        {error, Reason} ->
            {error, {payment_failed, Reason}}
    end.

record_to_ledger(#{customer_id := CustomerId, amount := Amount,
                   charge_id := ChargeId} = Request) ->
    ok = ledger:record_payment(CustomerId, Amount, ChargeId),
    {ok, Request}.

emit_events(#{customer_id := CustomerId, amount := Amount,
              charge_id := ChargeId} = Request) ->
    event_bus:publish(payment_succeeded, #{
        customer_id => CustomerId,
        amount => Amount,
        charge_id => ChargeId,
        timestamp => erlang:system_time(second)
    }),
    {ok, Request}.

rollback(#{reservation_id := ReservationId, customer_id := CustomerId}) ->
    %% Release the reserved funds
    account_aggregate:deposit(CustomerId, 0, {rollback, ReservationId});
rollback(_) ->
    ok.
```

---

## สรุป Part 98

| ข้อกำหนดด้านการเงิน | วิธีแก้ใน Erlang |
|--------------------|-----------------|
| Precision arithmetic | Integer cents, not floats |
| Audit trail | Event sourcing, append-only |
| No double-charge | Idempotency keys |
| Balanced books | Double-entry ledger |
| Fraud prevention | Pipeline with scoring |
| Regulatory compliance | Immutable event log |

---

*[← Part 97: Ecosystem](part_97_ecosystem_deep_dive.md) | [Part 99: Production Operations →](part_99_production_operations.md)*
