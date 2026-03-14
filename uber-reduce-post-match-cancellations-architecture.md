# Uber Ride-hailing - Reduce Post-Match Cancellations via “Pickup Quality Pack” (System Architecture V1)

**Product:** Uber (ride-hailing — Rider + Driver apps)  
**Audience:** Product Managers / Engineers  
**Goal:** Describe an implementation-oriented, **generic** (non-proprietary) system architecture to reduce **post‑match cancellations** by improving **pickup rendezvous success** *only when needed*.

**Scope in this doc**
- Eligibility + pickup difficulty scoring (rules-first v0)
- Contextual rider guided pickup confirmation (Module A)
- Driver “difficulty cue” + message templates (Module B)
- Arrival window state machine + symmetric rendezvous cues (Module C)
- API contracts, event schemas, data model
- Reliability / degraded modes / latency budgets
- Experimentation + rollout plan

**Out of scope**
- Core dispatch/pricing changes
- Full “best stop” optimization at scale (can be added later for select POIs)
- Airport/event mega-flows (separate templates/rollouts)

---

## 1) Problem: why post‑match cancellations cluster at pickup
Post‑match cancellations are expensive because they create wasted time (deadhead + waiting), re-matching churn, and trust erosion.

Pickup is failure-prone due to:
- **GPS ambiguity** (pin drift, urban canyons, multi-entrance buildings)
- **Human coordination** (finding the right person/car)
- **Local constraints** (no-stopping zones, enforcement risk, gates/security)

The architecture goal is simple:
> **Detect likely-hard pickups early and add minimal, explainable coordination UI** that reduces arrive→start time and pickup-related cancels, without harming booking conversion or driver acceptance.

---

## 2) Design principles (V1)
1. **Targeted friction only**: show extra steps only to high-confidence “hard pickup” rides.
2. **Fail open**: any dependency failure must fall back to baseline UX (never block booking).
3. **Explainable prompts**: “Hard pickup” must have a human-readable reason.
4. **Symmetry**: both rider and driver see the same rendezvous string (“where to meet”).
5. **Idempotent + event-driven**: at-least-once processing with idempotent writes.

---

## 3) Non-functional requirements (NFRs)
These are target SLOs for a large ride-hailing marketplace.

### Latency budgets
**Booking-time (pre-request) add-on**
- Difficulty scoring + POI template lookup must add **< 300ms p95** overall.
- Hard timeout: **500ms** total budget (beyond this → baseline flow).

**Post-match surfaces**
- Fetch “pickup pack” payload for trip screen: **< 200ms p95** (cache-first).

### Availability / degraded modes
- Pickup Quality Pack availability target: **99.9%** (degraded mode allowed).
- Degraded mode must preserve core ride request + trip state machine.

### Safety / privacy
- No prompts that encourage oversharing (phone number, precise outfit, etc.).
- Avoid unsafe directions (e.g., crossing highways). Copy + templates must be safety-reviewed.

---

## 4) User-facing modules (what backend must support)
### Module A — Rider Guided Pickup Confirmation (contextual)
- Entrance/gate selection (only where POI templates support it)
- Optional safe quick-note templates (3–5 max)
- Always provide a **“Looks good” skip**

### Module B — Driver Difficulty Cue + Templates
- “Easy / Medium / Hard” chip + reason
- 1‑tap message templates that incorporate rider’s entrance/landmark
- **No ‘best stop’** in MVP by default

### Module C — Pickup State Machine + Rendezvous Cues
- Rider: “Driver waiting at {spot}” card + car/plate + “I’m coming out now”
- Driver: “Waiting at {spot}” card + timer + “I’m at pickup spot”
- Pickup updates (pin edits) refresh rendezvous string and notify driver (rate-limited)

---

## 5) High-level architecture (services + data stores)
Assumed typical stack: **HTTP/gRPC microservices + Kafka + Redis + Postgres/MySQL**.

### Core services
1. **Booking Service (Ride Request API)**
2. **Pickup Difficulty Scorer Service (rules-first v0)**
3. **POI Template Service**
4. **Rendezvous Service (state store)**
5. **Dispatch/Matching Service**
6. **Trip Service (trip state machine)**
7. **Comms/Notifications Service** (in-app banners / push)
8. **Experiment/Feature Flag Service**
9. **Analytics / Event Ingestion**

### Storage
- **Postgres** (POI templates + localization)
- **Postgres** (Rendezvous state)
- **Postgres** (Scorer config: thresholds, curated POIs, reason mapping) *(or a config service)*
- **Redis** caches (templates, rendezvous reads, scorer feature cache)

---

## 6) System diagram (V1)
```mermaid
flowchart LR
  Rider[Rider App] -->|booking| Booking[Booking Service]
  Driver[Driver App] -->|trip screen| Trip[Trip Service]

  Booking -->|score(pickup)| Scorer[Pickup Difficulty Scorer]
  Booking -->|lookup(pickup)| POI[POI Template Service]
  Booking -->|upsert(request_id)| Rendezvous[Rendezvous Service]
  Booking -->|bucketing| Exp[Experiment/Flags]

  Dispatch[Dispatch/Matching] -->|TripMatched event| Kafka[(Kafka)]
  Kafka --> Rendezvous

  Trip -->|get pickup pack| Rendezvous
  Trip -->|get templates| POI
  Trip -->|optional recompute| Scorer

  Trip -->|banner/push| Comms[Comms/Notifications]

  Rider -->|pin edit after match| Trip

  Booking -->|events| Kafka
  Trip -->|events| Kafka
  Kafka --> Analytics[Analytics/Warehouse]

  Rendezvous --> Redis[(Redis cache)]
  Scorer --> Redis
  POI --> Redis

  Rendezvous --> RDB[(Postgres)]
  POI --> PTDB[(Postgres)]
  Scorer --> CFG[(Postgres/Config)]
```

---

## 7) Core flows (end-to-end)

### 7.1 Booking-time eligibility + guided flow (Module A)
**Goal:** decide eligibility quickly and safely.

1. Rider sets pickup + destination.
2. Booking Service performs **parallel** calls with strict timeouts:
   - `Scorer.score(pickup, geo, context)`
   - `POI.lookup(pickup, geo)`
3. Booking Service asks Experiment Service for bucket assignment (eligible-only A/B).
4. If in treatment and eligible:
   - send guided pickup payload to Rider app
   - Rider completes or skips
   - Booking Service writes rendezvous state (best-effort)

**Fail-open logic (mandatory)**
- If scorer/template slow or fails → treat as **not eligible**, show baseline flow.
- If rendezvous store write fails → proceed; baseline pickup string used later.

### 7.2 Match binding (request_id → trip_id)
1. Dispatch creates a match.
2. Dispatch emits `TripMatched {request_id, trip_id, driver_id, rider_id}` to Kafka.
3. Rendezvous Service consumes and binds state:
   - `bind(request_id, trip_id)` idempotently
   - warms Redis

### 7.3 Post-match driver cue (Module B)
1. Driver app requests trip details.
2. Trip Service loads pickup-pack payload:
   - rendezvous string + reason codes
   - suggested templates
3. Driver sees “Hard pickup: multi-entrance” + 1‑tap template.

### 7.4 Arrival window rendezvous state machine (Module C)
Trip Service manages canonical states:
- `ENROUTE → ARRIVED → WAITING → STARTED`

Pickup Quality Pack adds *soft* substates/flags:
- `rider_confirmed_coming_out`
- `driver_confirmed_at_spot`

These should not block trip start; they’re coordination signals.

### 7.5 Rider updates pickup after match
1. Rider edits pickup pin.
2. Trip Service updates pickup location and emits `PickupUpdated`.
3. Optionally recompute difficulty + regenerate rendezvous string.
4. Notify driver via banner (rate-limited).

---

## 8) Data model (V1)

### 8.1 PickupDifficultyResult
Stored for audit + consistency between booking and trip.
- `request_id` (PK)
- `geo_id`
- `pickup_lat`, `pickup_lng`
- `difficulty_score` (0..1)
- `difficulty_bucket` (easy|medium|hard)
- `reason_codes` (array, 1–3)
- `rules_version`
- `latency_ms`
- `created_at`

### 8.2 RendezvousState
- `request_id` (PK)
- `trip_id` (nullable until match)
- `selected_entrance_id` / `selected_entrance_label`
- `landmark_label` (optional)
- `note_template_id` (optional)
- `note_text` (optional, sanitized)
- `where_to_meet_string` (localized, derived)
- `revision` (monotonic)
- `updated_by` (rider|system)
- `updated_at`

### 8.3 POITemplate (simplified)
- `poi_id` / `geofence_id`
- `geo_id`
- `poi_type`
- `entrances[]` (id, label, lat/lng optional)
- `meeting_strings` (by locale)
- `template_version`

---

## 9) APIs (suggested V1 contracts)
These are intentionally minimal and idempotent.

### 9.1 Difficulty scoring
`POST /pickup-difficulty/score`
```json
{
  "request_id": "...",
  "pickup": {"lat": 12.9, "lng": 77.6},
  "geo_id": "blr",
  "poi_hint": {"poi_type": "MALL"},
  "rider_context": {"tenure": "new", "local_time_bucket": "night"}
}
```
Response:
```json
{
  "difficulty_score": 0.82,
  "difficulty_bucket": "hard",
  "reason_codes": ["MULTI_ENTRANCE_POI", "HISTORICAL_HIGH_FRICTION"],
  "rules_version": "v0.3",
  "latency_ms": 143
}
```

### 9.2 POI template lookup
`GET /poi-templates/lookup?lat=..&lng=..&geo_id=..`
Response:
```json
{
  "poi_type": "MALL",
  "template_version": "2026-03",
  "entrances": [{"id": "N_GATE", "label": "North Gate"}],
  "supported": true
}
```

### 9.3 Rendezvous state write (best-effort)
`PUT /rendezvous/state/{request_id}`
```json
{
  "selected_entrance_id": "N_GATE",
  "note_template_id": "NEAR_SECURITY_CABIN",
  "locale": "en-IN",
  "revision": 3
}
```
Response:
```json
{
  "ok": true,
  "revision": 4,
  "where_to_meet_string": "North Gate, near the security cabin"
}
```

### 9.4 Bind request to trip
`POST /rendezvous/bind`
```json
{ "request_id": "...", "trip_id": "..." }
```

### 9.5 Pickup pack aggregation
`GET /trips/{trip_id}/pickup-pack`
Response:
```json
{
  "difficulty_bucket": "hard",
  "reason_codes": ["MULTI_ENTRANCE_POI"],
  "where_to_meet_string": "North Gate",
  "templates": [{"id": "WAITING_AT_SPOT", "text": "I’m waiting at North Gate."}],
  "exposure_flags": {"A": true, "B": true, "C": true}
}
```

### 9.6 Pickup confirmations (soft signals)
`POST /trips/{trip_id}/pickup-confirmation`
```json
{ "side": "rider", "type": "COMING_OUT_NOW" }
```

---

## 10) Event schemas (Kafka / analytics)
Key rule: **at-least-once** delivery; consumers must be idempotent.

### 10.1 Trip matched
`TripMatched`
```json
{ "request_id": "...", "trip_id": "...", "geo_id": "...", "ts": 0 }
```

### 10.2 Pickup updated (post-match)
`PickupUpdated`
```json
{ "trip_id": "...", "old_pickup": {"lat": 0, "lng": 0}, "new_pickup": {"lat": 0, "lng": 0}, "revision": 5 }
```

### 10.3 Post-match cancel attribution
`PostMatchCancel`
```json
{
  "trip_id": "...",
  "side": "driver",
  "reason_code": "CANT_FIND_RIDER",
  "time_since_match_s": 420,
  "difficulty_bucket": "hard",
  "exposure_flags": {"A": true, "B": true, "C": true}
}
```

---

## 11) Scaling & partitioning (practical)

### Where load concentrates
- Booking-time scoring calls (high QPS, latency sensitive)
- Post-match trip screen fetches (cacheable, bursty)
- Hot POIs (airports/malls) create localized hotspots

### Recommended tactics
- **Redis caches** for POI template lookup + rendezvous reads
- Partition Kafka topics by `geo_id` or `trip_id` to reduce cross-shard joins
- Store scorer features as aggregated counters per `(geo_id, pickup_cell_id, poi_id)`

---

## 12) Degraded modes / load shedding
This is where reliability is won.

1. **Scorer timeout** → skip Module A entirely (baseline booking).
2. **POI template missing** → allow only generic note templates.
3. **Rendezvous store down** → show baseline pickup pin; suppress driver cue reason text if needed.
4. **Comms rate limiting** → coalesce pickup updates; max 1 banner / N minutes.

---

## 13) Experimentation & rollout (V1)
- Experiments are **eligible-only A/B**.
- Ramp: **1% → 5% → 25% → 50%** of eligible rides per geo.

**Stop criteria** (pause ramp if any):
- booking conversion drop beyond threshold
- driver acceptance drop beyond threshold
- pickup-related support contact spike
- safety incident uptick correlated with treatment

---

## 14) Security, privacy, and abuse considerations
- Sanitize rider notes (length caps, profanity filters, PII detection as needed).
- Don’t encourage sharing phone numbers or exact outfit.
- Avoid precise driver location disclosure beyond what baseline already provides.

---

## 15) Open questions (for V2)
- Where to compute `where_to_meet_string`: server vs client rendering?
- Should scoring results be persisted at request time or recomputed on match?
- How to maintain a high-quality curated POI template set (ops workflow + versioning)?

---

## Appendix: Reason codes (explainability)
- `MULTI_ENTRANCE_POI` → “This location has multiple entrances.”
- `NO_STOPPING_RISK` → “Stopping is limited near this pickup.”
- `HISTORICAL_HIGH_FRICTION` → “Pickups here often take longer.”
- `GPS_DRIFT_ZONE` → “GPS can be inaccurate here.”
