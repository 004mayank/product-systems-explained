# Skyscanner — System Architecture (V2)

*(Flights-first metasearch, with Accounts + Price Alerts; adds stronger scalability, quality, and reliability patterns.)*

## 0) What changed vs V1

V2 focuses on:
- **Streaming / incremental search** (SSE/WebSocket) + clearer search lifecycle
- **Better offer quality**: canonical schema, dedupe, “best offer per itinerary”, and revalidation
- **Provider cost control**: budgets, caching strategy, and adaptive fan-out
- **Fraud & abuse**: bot defenses, click fraud, and provider protection
- **Operational maturity**: SLOs, multi-region posture, and data governance

---

## 1) Goals & scope

### In-scope
- Flight search (metasearch): query → aggregated results → filters/sort
- Redirect/deeplink to airline/OTA for booking (no checkout)
- User accounts
- Saved searches + price alerts (create, manage, notify)
- Analytics + attribution for search and clickouts

### Still out-of-scope
- Full booking/PNR management
- Hotels/cars/packages (can be added later with same patterns)

---

## 2) System overview (components)

### Clients
- Web app
- iOS/Android apps

### Edge
- CDN + edge caching
- WAF + bot protection
- TLS termination

### API tier
- API Gateway (auth, routing, quotas/rate limits)
- BFF (Web/Mobile; can be one or split)
- Feature flags + experimentation platform

### Core domain services

1) **Identity & Accounts Service**
- signup/login, sessions, OAuth linking
- profile: locale/currency, notification prefs

2) **Search Orchestrator**
- owns `searchId` and lifecycle (created → running → partial → complete/expired)
- chooses providers based on query + historical performance + cost budget
- coordinates streaming results to client

3) **Provider Connectors (Integration Layer)**
- adapter per GDS/NDC/OTA/airline
- throttling, retries, circuit breakers, idempotency
- response sampling + provider-specific parsing

4) **Offer Normalization & Enrichment**
- canonical `Itinerary`/`Offer` model
- currency conversion (FX)
- baggage/fare rules normalization
- dedupe + “same itinerary” grouping

5) **Ranking, Sorting & Facets**
- compute Best/Cheapest/Fastest
- stable facet model + pagination/cursors
- (optional) lightweight personalization/learning-to-rank later

6) **Pricing / Revalidation Service**
- optional “freshness check” before redirect (provider terms permitting)
- marks offer as stale/changed and returns next-best alternatives

7) **Redirect/Deeplink Service**
- signed click tokens, affiliate params, replay protection
- emits click event and fraud signals

8) **Saved Searches & Alerts Service**
- saved search definitions
- alert rules (threshold, direction, frequency)

9) **Notifications Service**
- email/SMS/push via providers
- templating, retries, dedupe, unsubscribe management

10) **Tracking & Attribution**
- search/click/alert events
- attribution stitching (campaigns, partner tracking)
- fraud detection hooks

### Platform/data
- **Redis**: search sessions, partial results, hot reference caches
- **Relational DB** (Postgres/MySQL): users, saved searches, alerts, preferences
- **Event bus/stream** (Kafka/PubSub): clickstream, alert triggers, provider health events
- **Warehouse/lake** (BigQuery/Snowflake/S3): analytics + price history
- Observability: logs/metrics/traces, SLO dashboards

---

## 3) Search lifecycle & APIs (V2)

### 3.1 Create search
`POST /search`
- Request: origin/destination, dates, pax, cabin, locale/currency, flexibility, filters
- Response: `{ searchId, streamUrl, expiresAt }`

**Key V2 ideas**
- Orchestrator sets a **time budget** (e.g., 8–12s) and **provider budgets** (per query class).
- Results are produced incrementally.

### 3.2 Stream results (recommended)
`GET /search/{searchId}/stream` via **SSE** (or WebSocket)
- emits:
  - `SearchStarted`
  - `ProviderStatus` (started/timeout/error)
  - `ResultBatch` (offers/itineraries delta)
  - `SearchComplete`

Fallback polling:
`GET /search/{searchId}/results?cursor=...`

### 3.3 Result shaping
To reduce payload size and duplicate offers:
- Normalize to canonical itineraries.
- Compute **OfferGroups**: “best offer per itinerary” plus alternates.
- Paginate on groups (stable cursor), not raw offers.

### 3.4 Degraded mode
- Partial provider failures → still return results.
- If budgets are exceeded, stop slow providers and mark as `timed_out`.

---

## 4) Redirect & revalidation (V2)

### 4.1 Redirect
`POST /redirect` with `{ searchId, offerId }`
- Redirect Service:
  - validates offer TTL
  - optionally requests **revalidation** for top offers
  - signs redirect token (short TTL)
  - emits click event + fraud features
- Returns 302 to airline/OTA

### 4.2 Revalidation rules
- Only revalidate when:
  - offer is near TTL
  - provider has high change rate
  - fare is unusually low (fraud/price error risk)
- If changed:
  - return updated price or next-best alternative

---

## 5) Price alerts (V2)

### 5.1 Alert evaluation pipeline
1) User creates alert: `POST /alerts`
2) Alerts Service enqueues tracking jobs (schedule/frequency)
3) **Batch Search Workers** run search in “batch mode” via Orchestrator
4) Store observations: min/median price, provider mix, confidence
5) Trigger rules:
   - absolute threshold
   - % drop over baseline
   - best-in-N-days
6) Emit `AlertTriggered` event → Notifications

### 5.2 Notification dedupe
- idempotency key: `(alertId, observedAt_bucket, triggerType)`
- per-channel retry with dedupe store

---

## 6) Data model (conceptual)

### Identity
- `User(id, email/phone, createdAt, locale, currency, consentFlags)`
- `AuthSession(userId, deviceId, refreshTokenHash, expiresAt)`
- `NotificationPreference(userId, channels, quietHours, frequency)`

### Search / offers
- `SearchSession(searchId, normalizedQuery, createdAt, status, expiresAt)`
- `Itinerary(itinId, legs[], segments[], carriers, duration, stops)`
- `Offer(offerId, itinId, providerId, price, currency, fareRules, baggage, ttl, qualityScore)`
- `OfferGroup(groupId, itinId, bestOfferId, alternateOfferIds[])`

### Alerts
- `SavedSearch(id, userId, normalizedQuery, createdAt)`
- `PriceAlert(id, userId, savedSearchId, rule, schedule, status)`
- `PriceObservation(savedSearchId, observedAt, minPrice, medianPrice, sampleCount, meta)`

---

## 7) Caching, cost controls & provider protection

### 7.1 Caching
- Redis:
  - search sessions + partial results (TTL minutes)
  - reference data (airports/airlines/FX) longer TTL
- Edge caching for reference endpoints
- Provider response caching:
  - very short TTL, provider-specific, respect terms

### 7.2 Adaptive fan-out
- Start with a **core provider set**, then expand if:
  - yield is low
  - query is long-haul/complex
  - user scrolls / requests more
- Per-provider concurrency caps + circuit breakers

---

## 8) Security, privacy, abuse & fraud

- Bot mitigation on `POST /search` (cost amplification) and `POST /redirect` (click fraud)
- Signed redirect tokens (short TTL) + replay protection
- Consent gating for analytics/ads tracking
- Encrypt sensitive fields at rest; avoid PII in logs
- Unsubscribe + notification audit trails

---

## 9) Reliability, SLOs & multi-region posture

### Targets (example)
- Search first results p95: **< 2s**
- Search completion window: **5–15s** (depending on query/providers)
- Redirect p95: **< 300ms**
- Alerts: trigger within schedule window (e.g., **< 15 min** jitter)

### Multi-region (optional)
- Active-active for stateless tiers (Gateway/BFF/Orchestrator)
- Regional connector pools (reduce latency to providers)
- Data:
  - user DB: primary + read replicas (or multi-region DB)
  - search session cache: regional Redis with TTL

---

## 10) Observability & business metrics

### Tech
- distributed tracing: BFF → orchestrator → connectors
- provider health dashboards: timeout/error/throttle rates
- cost dashboards: calls/search, provider spend proxy

### Biz
- search success rate & yield (offers/search)
- search-to-click conversion
- click validity / fraud rate
- alert trigger rate, open/click rates, unsubscribes

---

## 11) Minimal deployment view (ASCII)

```
Clients -> CDN/WAF -> API Gateway -> BFF
                          |
      +-------------------+---------------------------+
      |                   |                           |
 Search Orchestrator   Identity/Accounts          Alerts Service
      |                   |                           |
 Provider Connectors      DB (users)               DB (alerts)
      |
 Normalize/Enrich -> Rank/Facets -> Redis (search state)
      |
 Pricing/Revalidate (opt)
      |
 Redirect Service -> Tracking/Attribution -> Event Bus -> Warehouse
                                    |
                              Notification Service -> Email/SMS/Push
```
