# System Architecture: CRED Pay (UPI) — Offline Trust Pack

**Product:** CRED Pay (UPI)  
**Author:** Mayank Malviya  
**Date:** 19 Feb 2026  
**Status:** v3 (architecture V1→V3 in one doc)

**PRD (v3):** https://github.com/004mayank/product-prd/blob/main/cred-pay-upi-prd.md

---

## Version history
- **v1 (this doc):** Baseline architecture: components, core flows, data model, state machine.
- **v2 (this doc):** Deep dive on pending resolution, retry guardrails, receipt caching, observability.
- **v3 (this doc):** Hardening: idempotency, monotonic-state guarantees, failure handling, launch gates, runbooks.

---

## 0. Scope & design goals
### In-scope
- Offline **scan & pay** (static + dynamic QR) for UPI payments.
- User-facing states: `processing`, `pending`, `success`, `failed`, `reversed`.
- Receipt-first UX + caching.
- Safe retry guardrails.
- Push notify subscription for pending resolution.
- Instrumentation + support payload attachment.

### Non-goals
- Rewards economics changes.
- Risk/AML rule changes.
- Collect requests, online deeplinks.

### Design goals
1. **Monotonic state**: client never sees a regressive state; server never emits one.
2. **Idempotency**: retries (client/network) must not create duplicate attempts.
3. **Fast path stays fast**: add minimal hops in the happy path.
4. **Receipts are durable**: user can access proof even during network drops.
5. **Measurable & operable**: every “trust” improvement must be observable and reversible.

---

## 1. High-level component map (C4-lite)
### Client (Mobile App)
- **Scanner UI**: QR acquisition + parsing.
- **Merchant Preview + Confirm UI**: identity cues, amount entry, funding selection.
- **UPI Auth UI**: PIN entry (UPI SDK / bank app / in-app module depending on integration).
- **Payment State UI**: `processing`/`pending`/terminal.
- **Receipt UI**: share/copy/receipt caching.
- **Retry Guardrail UI**: decision modal + long-press confirm.
- **Event Emitter**: analytics events with `attempt_id`.
- **Local Receipt Cache**: encrypted storage (attempt_id → receipt blob).

### Backend
- **Payments Gateway (CRED)**: API edge for attempt create/status; auth; orchestration.
- **Attempt Service (State Machine)**: canonical `PaymentAttempt` store + monotonic transitions.
- **PSP Connector**: integration layer to PSP / UPI rails; maps callbacks & status checks.
- **Receipt Service**: normalizes receipt fields; ensures UTR presence before `success`.
- **Retry Eligibility Service**: server-side decision for guardrail/block.
- **Notification Service**: push subscription + terminal state pushes.
- **Support Ingestion Service**: creates tickets with structured payload.
- **Analytics Pipeline**: event validation + dashboards + alerts.

### External systems
- **PSP/UPI rails**: status, callbacks, reversals.
- **Bank systems**: PIN auth, authorization outcomes.
- **Push provider**: FCM/APNs.
- **Support tooling**: CRM/ticketing.

---

## 2. Core data model
### 2.1 PaymentAttempt (canonical)
**Primary key:** `attempt_id` (UUID)

Key fields:
- `user_id_hash`
- `mode=offline_scan`
- `qr_type` (`static|dynamic|unknown`)
- `merchant`: `merchant_id?`, `display_name?`, `vpa`, `identity_confidence`
- `amount_paise`, `currency=INR`
- `funding_source`: `bank_id`, `masked_account?`
- `state`: `created|auth_started|processing|pending|success|failed|reversed`
- `failure_taxonomy?`
- `utr?` (required for `success`)
- `created_at_ms`, `updated_at_ms`

### 2.2 Receipt (derived but persisted)
- `attempt_id`
- `merchant_display`
- `vpa`
- `amount_paise`
- `funding_source_display`
- `timestamp_ms`
- `utr`
- `receipt_complete` (bool)

**Rule:** receipt must be derivable from `PaymentAttempt` + PSP fields, but persisted separately to avoid regressions during incident response.

---

## 3. State machine & invariants
### 3.1 Allowed transitions (server enforced)
```
created → auth_started → processing → pending → success
                                     ↘ failed
                                     ↘ reversed
processing → success|failed|reversed   (if fast confirmation)
```

### 3.2 Invariants
- **Monotonicity**: `state_rank(next) >= state_rank(current)`.
- **Success requires UTR**: if `state=success` then `utr != null`.
- **Failure taxonomy completeness**: if `state=failed` then `failure_taxonomy != null` (use `unknown`).
- **Idempotent writes**: repeated callbacks/status updates are safe.

### 3.3 Terminality
Terminal states: `success`, `failed`, `reversed`.
- Once terminal, all subsequent updates must be ignored or logged as anomalies.

---

## 4. APIs (contract-level)
> Mirror the PRD v3 contracts; architecture emphasis is on **idempotency and monotonicity guarantees**.

### 4.1 Create attempt (idempotent)
`POST /v1/payments/upi/attempts`

Idempotency:
- Client sends `Idempotency-Key` (UUID) scoped to (user_id, merchant_key, amount, timestamp_bucket).
- Server stores `(idempotency_key → attempt_id)` for TTL (e.g., 15 minutes).
- If a duplicate create arrives, server returns the same `attempt_id`.

### 4.2 Get attempt
`GET /v1/payments/upi/attempts/{attempt_id}`

Server guarantees:
- State monotonic.
- If terminal, includes receipt fields required for client receipt rendering.

### 4.3 Subscribe to pending updates
`POST /v1/payments/upi/attempts/{attempt_id}/subscriptions`

### 4.4 Retry eligibility
`POST /v1/payments/upi/retry/eligibility`

---

## 5. Key flows (sequence diagrams)
Legend:
- App = Mobile client
- PG = Payments Gateway
- AS = Attempt Service
- PSP = PSP Connector / Rails
- NS = Notification Service
- RS = Receipt Service

### 5.1 Happy path (fast rails)
```
App → PG: POST /attempts (idempotency-key)
PG → AS: create attempt (created)
AS → PSP: initiate payment
App → PG: begin auth (PIN)
PSP → AS: auth ok + confirm success (callback)
AS → RS: build receipt (requires UTR)
RS → AS: receipt ok
AS: state → success
App → PG: GET /attempts/{id}
PG → AS: fetch
AS → App: state=success + receipt fields
App: render receipt + cache locally
```

### 5.2 Slow rails: pending then success
```
... up to processing
AS: if no confirm within T=3s → state=pending
App: shows Pending UI
App → PG: POST /subscriptions

Later:
PSP → AS: success callback (or status poll confirms)
AS → RS: build receipt (UTR)
AS: state → success
NS → App: push "Payment confirmed" (attempt_id)
App → PG: GET /attempts/{id}
AS → App: success + receipt
App: render + cache
```

### 5.3 Network drop after PIN
```
App: loses network
App: enters Pending (offline variant)
App: queues status poll with backoff
When network returns:
App → PG: GET /attempts/{id}
AS → App: terminal state
App: render
```

### 5.4 Retry guardrail
```
App (pending) → PG: POST /retry/eligibility
PG → Eligibility: evaluate (same merchant+amount bucket, pending, age<=5m)
Eligibility → App: should_guardrail=true, allowed_actions=[...]
App: shows guardrail modal
User picks:
- wait_notify → subscription
- retry_different_account → funding picker → create new attempt (new idempotency key)
- continue_anyway → long-press → create new attempt
```

### 5.5 Receipt share & support
```
App: share receipt card (local render)
App → Analytics: receipt_shared

If "merchant didn't receive":
App: shows receipt (UTR prominent)
App → Support: POST /support/tickets (attempt_id + payload)
```

---

## 6. Pending resolution strategy (v2/v3 detail)
### 6.1 How pending resolves
Sources of truth (preferred order):
1. **PSP callback/webhook** for definitive terminal.
2. **PSP status poll** (server-side scheduled poll for pending attempts).
3. **Client pull** (GET attempt) + server updates.

### 6.2 Polling & load control
- Server schedules a poll job for pending attempts using exponential backoff (e.g., 10s, 30s, 60s, 120s, 300s) capped.
- Client polls with backoff too; must not DDoS.

### 6.3 Timeouts
- If PSP is unresponsive beyond max window, keep `pending` and continue background reconciliation.
- Never auto-fail solely on timeout unless rails guarantee (rare).

---

## 7. Receipt caching architecture
### 7.1 Storage
- Encrypted local store keyed by `attempt_id`.
- Stored fields limited to receipt essentials (avoid extra PII).

### 7.2 Read strategy
- On receipt screen open:
  1) Render cached receipt immediately if present.
  2) Fetch server attempt; if mismatch, update cache & UI.

### 7.3 Consistency rule
- If server says `success` but no UTR, treat as anomaly (block share; show “updating”). This should be caught by the **Success requires UTR** invariant.

---

## 8. Observability & analytics (operability)
### 8.1 Event guarantees
- `attempt_id` must be attached to 99%+ events.
- `pay_state_changed` emitted exactly once per UI-visible transition.

### 8.2 Dashboards
- Pending rate over time
- Time-to-terminal percentiles
- Panic retry rate
- Duplicate attempts per 1k
- Receipt completeness + share/copy rates
- Support tickets per 1k (tagged)

### 8.3 Alerts
- Success rate regression
- State regression detection
- Duplicate debit spike
- Receipt missing UTR spike

---

## 9. Security & privacy considerations
- Hash user identifiers in analytics.
- Receipt cache is encrypted at rest.
- Support payload must be policy-reviewed (avoid storing raw QR payload or full account details).
- Rate limit subscriptions + status checks.

---

## 10. Failure modes & mitigations (v3)
1. **PSP callback duplication** → idempotent callback handler keyed by (attempt_id, event_type, psp_ref).
2. **Out-of-order events** → state_rank check; ignore regressive writes; log `state_regression_detected`.
3. **Duplicate attempt creation** (user taps pay twice) → server idempotency key + client UI disabling in `processing`.
4. **Receipt missing UTR** → enforce invariant before `success`.
5. **Push delayed/missed** → app still polls/get status; push is optimization.

---

## 11. Launch gates & kill switches (architecture view)
- Feature flags correspond to separable components:
  - Pending UI renderer
  - Retry guardrail UI + eligibility call
  - Receipt auto-open + cache write
  - Push subscription

Kill switch strategy:
- Flags evaluated server-side for safety defaults; client respects server config on launch.

---

## 12. Runbooks (architecture view)
### PSP outage
- Increase pending copy severity via config.
- Disable “continue anyway” retry.
- Slow polling intervals.

### Duplicate debit spike
- Server returns `block=true` in retry eligibility for in-window.

### Missing UTR
- Stop marking success without UTR; roll back receipt-first success until fixed.

---

## Appendix A: State ranking (example)
`created(10) < auth_started(20) < processing(30) < pending(40) < success(50) < failed(60) < reversed(70)`

> Note: ordering between terminal states is arbitrary; monotonicity is what matters.
