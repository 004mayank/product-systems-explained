# Stripe AI Billing Primitives - System Architecture

**What this explains:** The system architecture that powers Stripe's AI Billing Primitives - `CreditBalance`, `AutoTopUp`, and `PaymentDelegate`. Specifically: how Stripe maintains a real-time, sub-100ms credit balance for AI merchants, orchestrates threshold-triggered recharges without polling, and enforces scoped agent-initiated payments with user-visible audit trails.

**PRD reference:** https://github.com/004mayank/product-prd/blob/main/stripe-prd.md

**Version:** v1 - Initial system design
**Changes from v0:** First version.

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core problem statement, five system layers, data flow narrative, core data model, failure modes, architectural trade-offs, open questions |

---

## 1) What this system is

Every AI company building on Stripe today has a billing model - token credits, metered usage, or hybrid - that does not map onto Stripe's native payment primitives. The result: a bespoke billing service lives between the product and Stripe, handling credit ledgers, top-up logic, and usage reconciliation. This parallel system is the primary reason AI companies evaluate Lago, Orb, and Metronome as Stripe alternatives.

Stripe AI Billing Primitives closes that gap with four new first-party objects:

- `CreditBalance` - a real-time credit ledger attached to a `Customer`; balance readable and deductible via API at sub-100ms P95.
- `CreditDeduction` - a synchronous deduction call per inference request; idempotent; returns updated balance inline.
- `AutoTopUp` - a threshold-triggered, event-driven recharge flow; no merchant polling required.
- `PaymentDelegate` - an OAuth-style delegation object that lets an AI agent initiate payments with scoped, revocable user permission.

The system has five architectural layers:

1. **Balance Store Service** - In-memory balance state with async persistence; handles concurrent deductions without overdraft.
2. **Deduction Pipeline** - Synchronous API path for `CreditDeduction` calls; enforces idempotency and serialises concurrent access.
3. **AutoTopUp Orchestrator** - Event-driven recharge pipeline triggered by `credit_balance.below_threshold` events; manages retries and state transitions.
4. **PaymentDelegate Enforcement Layer** - Scope validation, revocation propagation, and audit trail for agent-initiated `PaymentIntent` creation.
5. **Event Bus and Webhook Pipeline** - Fanout of all `credit_balance.*`, `auto_top_up.*`, and `payment_delegate.*` events to merchant webhook endpoints and Stripe's internal analytics.

---

## 2) Core problem the architecture must solve

The architectural constraint is not feature breadth - it is latency. The PRD's foundational SLO is `CreditDeduction` at sub-100ms P95. This single constraint shapes every design decision in the system.

### Why sub-100ms matters

An AI company's inference path looks like this:

```
User sends request
  -> product validates auth
  -> product calls LLM API
  -> product calls Stripe CreditDeduction API  <-- this call
  -> product returns response to user
```

If the `CreditDeduction` call takes 200ms, the user's total latency increases by 200ms. For inference products, this is unacceptable. Merchants who experience this overhead will revert to a local Redis ledger and only reconcile with Stripe asynchronously - which defeats the entire purpose of `CreditBalance`.

Sub-100ms P95 on a synchronous API call across the public internet to Stripe's servers is achievable only if the balance lives in memory, not in a relational database. Postgres read latency is 2-5ms in the best case under light load, but under concurrent write load during a busy inference session (50+ concurrent deductions per second for a high-traffic AI product), Postgres serialisation delays push P95 latency well above 100ms.

**Decision:** The balance lives in Redis. Writes to Postgres are asynchronous. This introduces a consistency window - the primary architectural trade-off of this system.

### Why concurrent deduction serialisation matters

Multiple inference requests from the same customer can arrive simultaneously and each call `CreditDeduction`. If two concurrent deductions both read the balance as 1,000 tokens and both deduct 600, the resulting balance is -200 tokens - an overdraft. The architecture must prevent this without introducing a write-time lock that blows the 100ms SLO.

**Decision:** Optimistic locking at the Redis key level using an atomic `WATCH`/`MULTI`/`EXEC` pattern. Concurrent deductions are serialised at the balance key level; the loser of the race retries once before returning `402 Insufficient Credits`. The winning deduction always wins; no overdraft is possible under this model.

### Why agent-payment revocation propagation is the hardest constraint

The `PaymentDelegate` revocation propagation SLO - P99 within 60 seconds across all Stripe endpoints - is the hardest distributed systems constraint in this architecture. Stripe operates hundreds of edge nodes globally. Propagating a revocation to all of them within 60 seconds requires an active push mechanism, not an eventual-consistency cache with a long TTL.

**Decision:** Revocations write to a dedicated `delegate_revocation_log` table in the primary Postgres cluster and immediately publish to a high-priority event topic on Stripe's internal event bus. All edge nodes that handle `PaymentIntent` creation subscribe to this topic and maintain a local in-memory revocation set with a short TTL (120 seconds). On each `PaymentDelegate`-flagged `PaymentIntent` creation, the edge node checks the local revocation set before forwarding to the core payment engine. P99 propagation is a function of event bus delivery latency, which is measured at <45 seconds under normal load.

---

## 3) System layers

### Layer 1: Balance Store Service

**Purpose:** Maintains the authoritative, real-time balance for every `CreditBalance` object. Serves read and deduction requests at sub-100ms P95.

**Storage architecture:**

```
Primary store: Redis Cluster (balance as integer at key: credit_balance:{cb_id})
Persistence: Postgres (credit_balances table; async write on every deduction event)
Reconciliation: Background job runs every 60 seconds; compares Redis balance
               with sum of all CreditDeduction and CreditGrant records in Postgres;
               alerts if drift exceeds 1 token
```

**Deduction flow (within this layer):**

```
WATCH credit_balance:{cb_id}
MULTI
  GET current_balance
  IF current_balance - deduction_amount < minimum_balance:
    DISCARD  -> return 402 Insufficient Credits
  SET credit_balance:{cb_id} = current_balance - deduction_amount
  PUBLISH balance_updated:{cb_id} = {new_balance, deduction_id, idempotency_key}
EXEC
  -> on EXEC success: publish CreditDeduction record to async Postgres writer queue
  -> on EXEC failure (concurrent write detected): retry once, then return 503 Transient Failure
```

**`below_threshold` detection:** On every successful deduction, the Balance Store Service checks whether the new balance has crossed below the `AutoTopUp.threshold` for the first time in the current session. "First time in the current session" is tracked with a Redis key `threshold_triggered:{cb_id}` with TTL equal to the AutoTopUp cooldown window (default 15 minutes). If the key is absent and the balance is below threshold, the service emits a `credit_balance.below_threshold` event to the internal event bus and sets the tracking key.

**Idempotency enforcement:** Idempotency keys for `CreditDeduction` are stored in Redis with a 24-hour TTL at key `idempotency:{key_hash}`. On each deduction call, the service checks this key first. If present, returns the original response without re-deducting. If absent, proceeds with the deduction and writes the key on success.

---

### Layer 2: Deduction Pipeline

**Purpose:** The synchronous API path that handles inbound `POST /v1/credit_deductions` calls from merchant servers. Responsible for request validation, idempotency enforcement, routing to the Balance Store Service, and inline balance response.

**Request path:**

```
Merchant server -> POST /v1/credit_deductions + Idempotency-Key header
  -> API Gateway (auth + rate limit: 1,000 req/sec per merchant, 10,000 req/sec global)
  -> Deduction Pipeline service
      -> validate request (customer exists, credit_balance exists and active, amount > 0)
      -> check idempotency key in Redis
          -> if present: return cached response (no deduction)
          -> if absent: proceed
      -> call Balance Store Service (WATCH/MULTI/EXEC)
          -> on success: write CreditDeduction record to async Postgres queue
                        set idempotency key in Redis (TTL 24h)
                        return 200 with balance_before, balance_after, status: succeeded
          -> on 402 (insufficient): return 402 with current balance in body
          -> on 503 (transient): return 503 with balance_stale: true
```

**Latency budget for the synchronous path:**

| Step | Target P95 |
|---|---|
| API Gateway auth + rate limit | <5ms |
| Request validation (customer lookup) | <10ms (cached in Redis, TTL 5 min) |
| Idempotency key check (Redis) | <3ms |
| Balance Store deduction (WATCH/MULTI/EXEC) | <30ms |
| Async Postgres queue write (fire and forget) | <5ms |
| **Total end-to-end (from Gateway to response)** | **<55ms P95** |

The 55ms budget leaves 45ms of headroom against the 100ms SLO, absorbing network jitter and tail latency on any single step.

---

### Layer 3: AutoTopUp Orchestrator

**Purpose:** Event-driven recharge service. Listens for `credit_balance.below_threshold` events and orchestrates the full recharge flow: PaymentIntent creation, Smart Retries, CreditGrant creation, and final state update.

**Trigger and deduplication:**

The Balance Store Service emits `credit_balance.below_threshold` to the internal event bus. The AutoTopUp Orchestrator is an event consumer on this topic. Before acting, it checks a deduplication key in Redis: `top_up_episode:{cb_id}:{episode_id}`. The `episode_id` is derived from the threshold-crossing event ID. If the key exists (a recharge is already in progress for this episode), the event is dropped. If absent, the Orchestrator proceeds and sets the key.

**Recharge state machine:**

```
State: idle
  -> event: credit_balance.below_threshold
  -> action: create PaymentIntent (amount = recharge_amount, payment_method = AutoTopUp.payment_method)
  -> state: pending_payment

State: pending_payment
  -> event: payment_intent.succeeded
  -> action: create CreditGrant (type: auto_top_up, amount = recharge_amount)
           -> Balance Store Service adds credits to Redis balance
           -> async Postgres writer queues CreditGrant record
  -> state: idle
  -> emit: auto_top_up.succeeded

  -> event: payment_intent.payment_failed
  -> action: check attempt_count vs. max_retries (default 3)
  -> if attempt_count < max_retries:
       -> schedule retry via Smart Retries ML model
       -> state: pending_retry
  -> if attempt_count >= max_retries:
       -> update CreditBalance.status = "top_up_failed"
       -> emit: auto_top_up.failed_final
       -> state: failed_terminal
```

**Smart Retries integration:** The AutoTopUp Orchestrator delegates retry scheduling to Stripe's existing Smart Retries service - the same ML-driven retry scheduler used for subscription invoice payment recovery. Smart Retries selects the optimal retry time based on card network signals, decline codes, and issuer behaviour patterns. The Orchestrator passes the `PaymentIntent` ID and the `auto_top_up` context label; Smart Retries handles the schedule.

**SCA / PSD2 path for European customers:** When the `AutoTopUp` is created for a European customer, the initial `AutoTopUp` configuration flow requires a cardholder-present SCA authentication step (using a `SetupIntent` to establish the stored credential agreement). Subsequent auto-triggered `PaymentIntent`s are flagged as MIT (Merchant Initiated Transaction) under PSD2's recurring transaction exemption, using the stored credential agreement ID from the `SetupIntent`. If an issuer declines the MIT exemption and returns `authentication_required`, the Orchestrator emits `auto_top_up.authentication_required` and halts retries for that customer until the merchant surfaces a re-authentication flow.

---

### Layer 4: PaymentDelegate Enforcement Layer

**Purpose:** Validates `PaymentDelegate` scope on every `PaymentIntent` creation that references a `payment_delegate_id`. Propagates revocations globally within 60 seconds P99. Maintains the audit trail.

**Scope validation flow:**

```
Merchant (or agent) calls POST /v1/payment_intents with payment_delegate_id

API Gateway routes to PaymentDelegate Enforcement Layer before forwarding to
the core payment engine.

Enforcement Layer:
  1. Fetch PaymentDelegate record from regional read replica (Postgres)
  2. Check status: if not "active" -> return 403 Delegate Revoked/Expired/Spend Cap Reached
  3. Validate scope:
     a. amount <= max_amount_per_transaction? -> else 403 Delegate Scope Exceeded
     b. merchant_category_code in allowed_mccs? -> else 403 Delegate Scope Exceeded
     c. spent_to_date + amount <= total_spend_cap? -> else 403 Spend Cap Reached
     d. current_time within [valid_after, valid_until]? -> else 403 Delegate Expired
  4. Check in-memory revocation set (see below) -> if present: 403 Delegate Revoked
  5. On pass: forward to core payment engine; update spent_to_date atomically on confirmation
  6. On PaymentIntent confirmed: emit payment_delegate.used event
```

**Revocation propagation architecture:**

```
User calls POST /v1/payment_delegates/{id}/revoke

Primary write path:
  -> write revoked_at, revocation_reason to Postgres (primary)
  -> publish {delegate_id, revoked_at} to high-priority revocation topic on internal event bus
  -> return 200 immediately (propagation is async but guaranteed within 60s P99)

Propagation path:
  -> each API edge node subscribes to revocation topic
  -> on receiving revocation event: add delegate_id to local in-memory revocation set (LRU cache, TTL 120s)
  -> all subsequent PaymentIntent validation calls check this set first (memory lookup, <1ms)
  -> edge node also writes revocation to its local Redis cache (TTL 300s) for persistence across process restarts
```

**In-flight PaymentIntent handling at revocation:** If a `PaymentIntent` is in `requires_confirmation` state when the revocation event propagates to the edge node processing that intent, the edge node cancels the `PaymentIntent` and emits `payment_intent.cancelled` with `cancellation_reason: "payment_delegate_revoked"`. The agent application and the merchant both receive this webhook. The edge node does not wait for propagation completion before cancelling - it checks the revocation topic at the point of `PaymentIntent` confirmation, not at creation.

**Audit trail:** Every `payment_delegate.used` event is written to Stripe's event log and is accessible via:
- The Stripe Dashboard: under the customer's payment history, each delegate-initiated charge shows "Authorised by [Agent Name]" with the scope at time of use.
- The Stripe Customer Portal: a "Manage AI agent permissions" section lists all active `PaymentDelegate` objects and the audit history per delegate.
- The API: `GET /v1/payment_delegates/{id}/events` returns the full event log for that delegate.

---

### Layer 5: Event Bus and Webhook Pipeline

**Purpose:** Fanout of all billing primitive events to merchant webhook endpoints. Also feeds Stripe's internal analytics pipeline for dashboard metrics, support tooling, and experiment instrumentation.

**Event categories and consumers:**

| Event category | Internal consumers | Merchant webhook |
|---|---|---|
| `credit_balance.*` | Analytics pipeline, Dashboard balance display | Yes - all events |
| `credit_deduction.*` | Analytics pipeline, Reconciliation job | Yes - `succeeded`, `failed` only |
| `auto_top_up.*` | Analytics pipeline, Support tooling (failed payments), Smart Retries service | Yes - all events |
| `payment_delegate.*` | Analytics pipeline, Customer Portal audit display, Fraud/Radar scoring | Yes - all events |

**Webhook delivery guarantees:**
- At-least-once delivery. Merchants must handle idempotent webhook receipt.
- Retry schedule: immediate, 5 minutes, 30 minutes, 2 hours, 8 hours, 1 day, 3 days.
- After 7 days of failed delivery, the webhook endpoint is marked `disabled` and Stripe notifies the merchant by email.
- Webhook event ordering is not guaranteed. Merchants must not rely on `credit_balance.updated` arriving before `credit_deduction.succeeded` for the same session.

**Internal analytics instrumentation:** All primitive events include a `livemode: boolean` field. Sandbox events are filtered to a separate analytics pipeline for test mode metrics. The production pipeline feeds:
- The AI Billing Primitives activation funnel (merchant cohort from first account creation to first live `CreditDeduction`).
- The AutoTopUp success rate dashboard (input metric for the 85% target).
- The `PaymentDelegate` delegated GPV tracker (target: $10M within 6 months of closed beta).

---

## 4) Core data model

### `credit_balances` table (Postgres - persistent store)

```
id                    uuid (PK)
customer_id           varchar (FK to customers)
unit                  varchar(64)
unit_display_name     varchar(128)
balance               bigint              -- authoritative after reconciliation; in-flight updates are in Redis
minimum_balance       bigint
tab_limit             bigint nullable
on_insufficient_balance   enum('block', 'tab')
auto_top_up_id        uuid nullable
status                enum('active', 'frozen', 'top_up_failed')
below_threshold       boolean
created               timestamptz
updated               timestamptz
livemode              boolean
metadata              jsonb
```

**Redis representation:**

```
credit_balance:{cb_id}        integer          // current balance; single source of truth for live deductions
threshold_triggered:{cb_id}   1                // TTL = AutoTopUp cooldown window (15 min default)
idempotency:{key_hash}        {deduction_id}   // TTL = 24h
customer_lookup:{customer_id} {cb_id}          // TTL = 5 min; invalidated on customer update
top_up_episode:{cb_id}:{ep}   1               // TTL = 15 min; prevents duplicate AutoTopUp for same episode
```

### `credit_deductions` table

```
id                    uuid (PK)
customer_id           varchar
credit_balance_id     uuid (FK)
amount                bigint
unit                  varchar(64)
balance_before        bigint
balance_after         bigint
idempotency_key       varchar(255) unique
status                enum('succeeded', 'failed', 'pending')
metadata              jsonb
created               timestamptz
livemode              boolean
```

### `credit_grants` table

```
id                    uuid (PK)
customer_id           varchar
credit_balance_id     uuid (FK)
amount                bigint
unit                  varchar(64)
type                  enum('payment', 'auto_top_up', 'manual', 'migration')
payment_intent_id     varchar nullable
balance_before        bigint
balance_after         bigint
expires_at            timestamptz nullable
metadata              jsonb
created               timestamptz
livemode              boolean
```

### `auto_top_ups` table

```
id                    uuid (PK)
customer_id           varchar
credit_balance_id     uuid (FK)
threshold             bigint
recharge_amount       bigint
payment_method_id     varchar
status                enum('active', 'disabled', 'failed_terminal')
retry_schedule        enum('smart_retries', 'fixed')
max_retries           integer (default 3)
last_triggered_at     timestamptz nullable
last_trigger_status   enum('succeeded', 'failed', 'pending') nullable
last_payment_intent_id    varchar nullable
current_episode_id    uuid nullable          // for deduplication of concurrent threshold events
created               timestamptz
livemode              boolean
metadata              jsonb
```

### `payment_delegates` table

```
id                    uuid (PK)
customer_id           varchar
agent_application_id  varchar
agent_display_name    varchar
scope_max_amount_per_transaction  bigint
scope_allowed_mccs    varchar[]
scope_total_spend_cap bigint
scope_spent_to_date   bigint
scope_valid_until     timestamptz
scope_valid_after     timestamptz nullable
status                enum('active', 'revoked', 'expired', 'spend_cap_reached')
revoked_at            timestamptz nullable
revocation_reason     varchar nullable
created               timestamptz
livemode              boolean
metadata              jsonb
```

### `delegate_revocation_log` table (append-only, replicated to all edge regions)

```
id                    uuid (PK)
delegate_id           uuid
customer_id           varchar
revoked_at            timestamptz
revocation_reason     varchar
propagated_at         timestamptz nullable   // set by propagation confirmation job
in_flight_cancellations   varchar[]          // PaymentIntent IDs cancelled due to this revocation
```

---

## 5) Data flow narratives

### Flow A: Customer purchases credits -> uses product (credit lifecycle)

```
1. Merchant calls POST /v1/payment_intents (amount = price of credit pack)
2. PaymentIntent confirmed -> Stripe core emits payment_intent.succeeded
3. Merchant's webhook handler receives payment_intent.succeeded
4. Merchant calls POST /v1/credit_grants (type: payment, payment_intent_id, amount: 100000)
5. Balance Store Service:
   a. Postgres: insert credit_grant record
   b. Redis: INCRBY credit_balance:{cb_id} 100000
   c. Emit credit_balance.updated event
6. Merchant's product: credit balance is live

7. User makes inference request
8. Merchant's inference path calls POST /v1/credit_deductions (amount: 850)
9. Deduction Pipeline -> Balance Store Service -> WATCH/MULTI/EXEC
10. Redis balance updated from 100000 to 99150
11. Async: Postgres writer queues credit_deductions insert
12. Response: 200 {balance_before: 100000, balance_after: 99150, status: succeeded}

13. Repeat for each inference call
```

### Flow B: Balance drops below threshold -> AutoTopUp recharges

```
1. CreditDeduction call drops balance from 11,000 to 9,800 (threshold: 10,000)
2. Balance Store Service:
   a. Detects threshold crossing (checks threshold_triggered:{cb_id} key - absent)
   b. Sets threshold_triggered:{cb_id} = 1, TTL 15 min
   c. Emits credit_balance.below_threshold event to internal bus
3. AutoTopUp Orchestrator receives event:
   a. Checks top_up_episode:{cb_id}:{ep_id} - absent; sets it
   b. Creates PaymentIntent (recharge_amount: 100000, payment_method: AutoTopUp.payment_method)
   c. Sets AutoTopUp.status = pending, last_payment_intent_id = pi_new
   d. Emits auto_top_up.triggered event
4. Stripe core processes PaymentIntent
5. payment_intent.succeeded fires
6. AutoTopUp Orchestrator:
   a. Creates CreditGrant (type: auto_top_up, amount: 100000)
   b. Balance Store Service: INCRBY credit_balance:{cb_id} 100000 (balance: 109800)
   c. Updates AutoTopUp: status = active, last_trigger_status = succeeded
   d. Emits auto_top_up.succeeded (includes time_from_trigger_to_grant_ms)
7. User's inference session continues without interruption
```

### Flow C: Agent initiates payment on behalf of user (agentic commerce)

```
1. User grants PaymentDelegate to agent application:
   a. Merchant calls POST /v1/payment_delegates (scope: max 100, mccs, cap 500, window 24h)
   b. PaymentDelegate record created in Postgres
   c. Emits payment_delegate.created event (sent to user as email notification)

2. Agent executes task (e.g., books a hotel)
3. Agent (via merchant API) calls POST /v1/payment_intents with payment_delegate_id

4. PaymentDelegate Enforcement Layer intercepts before core payment engine:
   a. Fetch delegate record (Postgres read replica)
   b. Check status: active
   c. Validate scope: amount 89 <= 100, MCC 7011 in allowed_mccs, spent_to_date 0 + 89 <= 500
   d. Check in-memory revocation set: absent
   e. Forward to core payment engine

5. PaymentIntent confirmed
6. Enforcement Layer:
   a. Atomically increment spent_to_date: 0 -> 89
   b. Emit payment_delegate.used event
7. User receives email: "Tripper AI charged $89 for hotel (authorised by you). [View | Revoke]"

8. User clicks Revoke
9. POST /v1/payment_delegates/{id}/revoke
10. Enforcement Layer:
    a. Write revocation to Postgres
    b. Publish to high-priority revocation topic
    c. All edge nodes update in-memory revocation set within 60s P99
    d. Emit payment_delegate.revoked event
```

---

## 6) Failure modes

| Failure | Detection | Mitigation | Merchant-visible behaviour |
|---|---|---|---|
| Redis unavailable (Balance Store down) | Health check; Deduction Pipeline returns connection errors | All `CreditDeduction` calls return 503 with `balance_stale: true`; merchant's policy determines whether to block or allow inference | 503 response on deduction; merchant should have an allow-on-stale policy defined for their product |
| Redis-Postgres reconciliation drift detected | Background reconciliation job (runs every 60s) | Page on-call; freeze affected `CreditBalance` objects at Redis balance until investigation completes; emit `credit_balance.reconciliation_alert` internal event | No merchant-visible change until freeze; if freeze is enacted, deductions return 503 for that specific customer |
| AutoTopUp PaymentIntent creation fails (not a Smart Retries scenario - network error) | AutoTopUp Orchestrator catches exception | Retry PaymentIntent creation up to 3 times with 5-second backoff; if all fail, emit `auto_top_up.system_error` event and set episode to failed state for manual recovery | `auto_top_up.system_error` webhook; balance stays below threshold |
| PaymentDelegate revocation propagation slow | Propagation confirmation job detects `propagated_at` null after 90s | Alert on-call; manually push revocation to lagging edge nodes; revocation already in Postgres and will be caught on any cold-start of edge nodes | No merchant-visible change; revocation may be delayed beyond 60s SLO; internal SLO breach only |
| Concurrent deduction race (WATCH/MULTI/EXEC collision) | EXEC returns nil (transaction conflict detected) | Single retry; if second attempt also fails, return 503 Transient Failure with `retry_after: 1s` | 503 response; merchant should retry once (standard HTTP retry logic) |
| Customer lookup cache miss (Redis) storm on startup | High Postgres read latency from stampede | Jitter on cache TTL expiry (5 min +/- 30s); read-through caching with coalescing for same customer_id | Slightly higher P95 latency on first request for a customer after cache expiry; stays within 100ms SLO |
| AutoTopUp concurrent threshold events (50 deductions simultaneously crossing threshold) | Multiple `credit_balance.below_threshold` events for same `cb_id` | `top_up_episode` deduplication key in Redis prevents duplicate PaymentIntents; only first episode proceeds | Single PaymentIntent created; subsequent events dropped silently |

---

## 7) Architectural trade-offs

### Trade-off 1: Redis as balance primary vs. Postgres as balance primary

**Option A (chosen):** Redis is the real-time balance source of truth; Postgres is the audit and reconciliation store. Deductions update Redis synchronously; Postgres is updated async.

**Option B (rejected):** Postgres is the primary. Balance is a column with row-level locking per deduction. All deductions are serialised through Postgres.

**Why A wins:** Under 100 concurrent deductions per second (achievable for any mid-size AI product at peak usage), Postgres row-level locking under serialisable isolation would produce P95 latency of 400-800ms - well above the 100ms SLO. Redis with optimistic locking (WATCH/MULTI/EXEC) handles concurrent deductions at sub-30ms P95 even under high contention.

**Cost of A:** The async Postgres write introduces a window where Redis and Postgres are inconsistent. A crash between the Redis deduction and the Postgres write would result in a Redis balance that reflects the deduction but a Postgres record that does not. Recovery requires replaying the `credit_balance.updated` event from the event bus to re-insert the missing Postgres record. The 60-second reconciliation job detects this and alerts - but the window between the crash and the reconciliation job's next run is up to 60 seconds of undetected drift. This is acceptable for credits (the drift favours the merchant - they see fewer credits consumed than were actually consumed) but must be treated as a P1 incident for any drift that favours the customer.

### Trade-off 2: In-memory revocation set vs. cache-aside revocation check

**Option A (chosen):** Each API edge node maintains an in-memory LRU cache of recently revoked `PaymentDelegate` IDs (TTL 120s). Revocations are push-propagated via event bus.

**Option B (rejected):** On each `PaymentIntent` with a `payment_delegate_id`, make a synchronous read to the Postgres read replica to check the delegate's current status.

**Why A wins:** Option B adds a Postgres read to every agent-initiated `PaymentIntent` creation. At 100ms P95 for a Postgres read replica (cross-region) + 100ms for the `PaymentIntent` creation itself, the agent-payment path becomes 200ms+ slower for every transaction. For agentic commerce (where the agent is executing a chain of tool calls), this compounds. Option A keeps the revocation check at <1ms (memory lookup).

**Cost of A:** The 60-second propagation window means a just-revoked delegate can still succeed on edge nodes that have not yet received the propagation event. This is the documented 60s P99 SLO. For most revocation scenarios (user changes their mind after the agent has already committed a transaction), this window is acceptable. For high-risk scenarios (user reports fraud), Stripe's fraud team has an out-of-band tool to force-push revocations to all edge nodes within 10 seconds.

### Trade-off 3: `CreditGrant` created by merchant vs. auto-created by Stripe on PaymentIntent success

**Option A (chosen for credit pack purchase):** Merchant calls `POST /v1/credit_grants` after receiving the `payment_intent.succeeded` webhook. Stripe does not auto-create the grant.

**Option B (chosen for AutoTopUp):** Stripe auto-creates the `CreditGrant` when the AutoTopUp `PaymentIntent` succeeds, without requiring a merchant webhook handler.

**Why both:** For credit pack purchases, the merchant may have custom logic about how many credits to grant per dollar (pricing tiers, promotional bonuses, migration grants). Auto-creating would require Stripe to know the merchant's conversion rate, which varies. The merchant is the right creator.

For `AutoTopUp`, the recharge amount is pre-configured in the `AutoTopUp` object. Requiring the merchant to handle a webhook and create the grant would introduce failure modes: if the merchant's webhook handler is down when the PaymentIntent succeeds, the customer is charged but receives no credits. Stripe auto-creating the grant removes this failure mode and is safe because the recharge amount is fully specified in the `AutoTopUp` configuration.

---

## 8) Open questions (v1)

**Q1: How should the reconciliation job handle a drift scenario where Redis shows fewer credits than Postgres?**

This scenario implies a `CreditGrant` was recorded in Postgres (balance increase) but the Redis INCRBY was lost (e.g., Redis failover after Postgres write but before INCRBY). The customer has fewer credits in Redis than they should have. Current design: the reconciliation job would detect this and... what? Adding credits to Redis retroactively without re-triggering the event bus could produce ghost credits. Setting the Redis balance to the Postgres-derived balance is safer but requires a brief lock on the credit_balance Redis key. The resolution path for this edge case needs to be specified before Phase 1.

**Q2: What is the correct behaviour when a `CreditDeduction` call arrives during an in-progress AutoTopUp (balance is below threshold and top-up PaymentIntent is pending)?**

Option A: Allow the deduction (balance may go to zero or trigger `on_insufficient_balance: block`). Option B: Hold the deduction call for up to N seconds while the top-up resolves. Option B adds latency to the inference path. Option A means the customer may hit a 402 in the window between threshold-crossing and top-up credit grant (typically <30 seconds). The PRD implies Option A is correct (the 402 is the correct outcome), but the merchant-facing documentation and SDK should explicitly state this window so merchants can set user expectations.

**Q3: Multi-currency `CreditBalance` - should the Balance Store Service support multiple balance buckets per customer key, or is one Redis key per `CreditBalance` object always the right abstraction?**

The PRD resolved this as unit-denominated in v1 (one `CreditBalance` per `unit` type, multiple objects if needed). The architecture follows: one Redis key per `CreditBalance` ID, not per customer. This is correct. The open question is whether to support a compound key at the Redis layer (`credit_balance:{customer_id}:{unit}`) for faster balance lookup by unit type without knowing the `cb_id`, or always require the caller to know the `cb_id` for direct key access. The PRD API uses explicit `credit_balance_id` on `CreditDeduction`, so the direct key model is correct. Marking as resolved - no change needed.

**Q4: Should `PaymentDelegate` scope validation run at the API Gateway layer or as a separate service called inline by the core payment engine?**

The current architecture places it as a pre-flight check at the API layer, before the `PaymentIntent` reaches the core payment engine. This is simpler but means the scope check and the payment engine are decoupled - a race condition exists where the scope check passes but the payment engine confirmation adds to `spent_to_date` after a concurrent delegate check already passed. The safer design would be for the payment engine to re-check scope atomically at confirmation time. This needs to be resolved before Phase 1 (the Phase 1 PRD requirement explicitly states "scope validation runs before PaymentIntent confirmation" which is ambiguous about re-validation at confirmation).

**Q5: What is the operational recovery path for a `PaymentDelegate` revocation that takes >120 seconds to propagate (edge node in-memory TTL expires before revocation event arrives)?**

If an edge node's revocation event is delayed >120 seconds (e.g., internal event bus partition), the in-memory revocation set TTL will have already expired. On the next PaymentIntent validation, the delegate will appear active. The edge node will then make a Postgres read replica check (fallback path) - but this adds 100ms+ to the validation path. The design needs to specify: is the Postgres fallback check always performed when the delegate is not in the in-memory set, or only when the in-memory set is explicitly stale? Always checking Postgres defeats the performance advantage of the in-memory set.

---

*All latency targets and cost figures are directional estimates based on public Redis, Postgres, and Stripe API benchmarks - not internal Stripe data.*
