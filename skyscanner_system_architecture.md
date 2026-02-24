# Skyscanner — System Architecture (V1)

*(Flights-only, with Accounts + Price Alerts)*

## 0) Goals & scope

### In-scope (V1)
- Flight search (metasearch): query → aggregated results → filters/sort
- Redirect/deeplink to airline/OTA for booking (no checkout)
- User accounts (email/phone + OAuth optional)
- Saved searches + price alerts (create, manage, notify)
- Analytics + attribution for search and clickouts

### Out-of-scope (V1)
- Full booking/PNR management
- Hotels/cars/packages
- Heavy ML personalization (basic ranking heuristics ok)

---

## 1) System overview (components)

### Clients
- Web app
- iOS/Android apps

### Edge
- CDN (static + edge caching)
- WAF / bot protection
- TLS termination

### API tier
- API Gateway (auth, routing, rate limits)
- BFF (Backend-for-Frontend): Web BFF, Mobile BFF (optional split; can be one BFF in V1)
- Feature Flag / Experiment service (or SaaS)

### Core services
1. **Identity & Accounts Service**
   - signup/login, sessions, OAuth linking
   - profile, preferences (currency, locale, notification prefs)
2. **Search Orchestrator Service**
   - validates query, creates `searchId`, manages search lifecycle
   - fans out to provider connectors and aggregates partial results
3. **Provider Integration Layer (Connectors)**
   - adapters per GDS/NDC/OTA/airline API
   - throttling, retries, circuit breakers
4. **Offer Normalization Service**
   - normalize itineraries/offers into canonical schema
   - currency conversion, dedupe, fare/baggage normalization
5. **Ranking + Filters Service**
   - compute “Best/Cheapest/Fastest”, sorting, facets
6. **Redirect/Deeplink Service**
   - generates trackable deeplinks, validates offer TTL
7. **Tracking & Attribution Service**
   - search/click events, affiliate params, fraud signals
8. **Saved Searches & Alerts Service**
   - store saved search definitions
   - price tracking jobs + notification triggers
9. **Notification Service**
   - email/SMS/push sending (via providers), templates, retries, unsubscribes

### Data/platform
- Redis (hot cache + ephemeral search sessions)
- Relational DB (Postgres/MySQL) for accounts, saved searches, alert configs
- Event bus/stream (Kafka/PubSub) for clickstream + alert events
- Warehouse/lake (BigQuery/Snowflake/S3) for analytics + long-term price history
- Observability (logs/metrics/traces)

---

## 2) Main flows

### 2.1 Flight search → results
1. Client calls `POST /search` (route, dates, pax, cabin, locale/currency).
2. BFF → Search Orchestrator:
   - normalize (airport/city resolution, date/timezone rules)
   - check short-lived query cache (optional; careful with dynamic fares)
   - create `searchId`, store initial session in Redis
3. Orchestrator fans out to Provider Connectors (parallel) with a time budget.
4. Connectors return provider-specific offers → Offer Normalization → canonical `Offer` + `Itinerary`.
5. Ranking/Filters computes facets and sorted result sets.
6. Client fetches incremental results: `GET /search/{searchId}/results?cursor=...` until complete/timeout.

**Degraded mode:** return partial results if some providers fail/timeout.

### 2.2 Offer click → redirect
1. Client `POST /redirect` with `searchId`, `offerId`.
2. Redirect Service:
   - validates offer freshness / required fields
   - creates signed redirect token + affiliate params
   - emits click event to Tracking/Attribution
3. 302 to airline/OTA.

### 2.3 Accounts (signup/login)
- Email magic link or OTP; optional password.
- Sessions via secure httpOnly cookies (web) + token storage (mobile).
- Consent + privacy settings stored per user.

### 2.4 Price alerts (saved search → track → notify)
1. User saves a search / creates an alert: `POST /alerts` (query + thresholds + channel).
2. Alerts Service persists:
   - alert definition
   - schedule (e.g., daily / price-drop)
   - user notification preferences
3. **Price Tracking Workers** run on schedule:
   - re-run search (same normalized query) via Search Orchestrator in “batch mode”
   - store observed prices/time-series in warehouse (and/or compact store)
4. Decision engine:
   - detect price drop, best price in N days, or threshold met
   - emit `AlertTriggered` event
5. Notification Service sends email/push/SMS with deep links back to results.

---

## 3) Data model (conceptual)

### User / Identity
- `User(id, email/phone, createdAt, locale, currency, consentFlags)`
- `AuthSession(userId, deviceId, refreshTokenHash, expiresAt)`
- `NotificationPreference(userId, emailOptIn, smsOptIn, pushOptIn, quietHours, frequency)`

### Search
- `SearchSession(searchId, normalizedQuery, createdAt, status, ttl)`
- `Itinerary(itinId, legs[], segments[], carriers, duration, stops)`
- `Offer(offerId, itinId, providerId, price, currency, fareRules, baggage, deeplinkTemplate, ttl)`

### Alerts
- `SavedSearch(id, userId, normalizedQuery, createdAt)`
- `PriceAlert(id, userId, savedSearchId, threshold, direction, schedule, status)`
- `PriceObservation(savedSearchId, observedAt, minPrice, providerMixMeta...)` (warehouse-friendly)

---

## 4) Caching & freshness
- Redis:
  - search sessions + partial results (TTL minutes)
  - reference lookups (airports, airlines, FX rates) with longer TTL
- Edge cache:
  - static reference endpoints
- Provider response caching:
  - very short TTL; provider-specific; respect terms

---

## 5) Reliability & performance targets (V1)
- Search initial results p95: **< 2s**, full aggregation **5–15s**
- Redirect p95: **< 300ms**
- Alert job completion: within schedule window (e.g., < 15 min jitter)

Resilience:
- per-provider circuit breakers + bulkheads
- global time budget for search fan-out
- idempotency keys for `POST /redirect` and notification sends (avoid duplicates)

---

## 6) Security, privacy, abuse
- Strong bot mitigation on:
  - `POST /search` (scraping, cost amplification)
  - `POST /redirect` (click fraud)
- GDPR/consent gating for analytics + ads tracking
- Encrypt sensitive fields at rest; no PII in logs
- Signed redirect tokens; short TTL; replay protection
- Unsubscribe + notification audit trails

---

## 7) Observability & business metrics

### Tech
- distributed tracing BFF → orchestrator → connectors
- provider health dashboards (timeouts, error rates, throttles)

### Biz
- search success rate, results yield (offers/search)
- search-to-click conversion
- click validity / fraud rate
- alert trigger rate, open/click rates, unsubscribe rates

---

## 8) Minimal deployment view (ASCII)

```
Clients -> CDN/WAF -> API Gateway -> BFF
                          |
      +-------------------+--------------------+
      |                   |                    |
 Search Orchestrator   Identity/Accounts   Alerts Service
      |                   |                    |
 Provider Connectors      DB (users)        DB (alerts)
      |
 Offer Normalize -> Rank/Filter -> Redis (search state)
      |
 Redirect Service -> Tracking/Attribution -> Event Bus -> Warehouse
                                    |
                              Notification Service -> Email/SMS/Push
```
