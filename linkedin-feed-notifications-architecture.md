<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/Linkedin.png" 
    alt="Linkedin Logo" 
    width="200"
  />
</p>

# LinkedIn Home Feed + Notifications - System Architecture

**Product:** LinkedIn (Home Feed + Notifications)

**Audience:** Product Managers / Engineers

**Goal:** Describe a plausible, implementation-oriented architecture for **(a) the LinkedIn Home Feed** and **(b) the notifications system**, grounded in publicly observable product behavior and generic system design patterns (no proprietary internals).

**PRD:** https://github.com/004mayank/product-prd/blob/main/linkedin-signal-density-prd.md

---

## Table of contents

- [0) Mental model](#0-mental-model)
- [1) Non-functional requirements (NFRs) / SLOs](#1-non-functional-requirements-nfrs--slos)
- [2) Back-of-envelope sizing](#2-back-of-envelope-sizing)
- [3) High-level architecture](#3-high-level-architecture)
- [4) Data model (key entities)](#4-data-model-key-entities)
- [5) Event-driven foundation: ingestion + enrichment](#5-event-driven-foundation-ingestion--enrichment)
- [6) Feed: candidate generation → ranking → serving](#6-feed-candidate-generation--ranking--serving)
- [7) Notifications: event → recipient expansion → policy/ranking → delivery](#7-notifications-event--recipient-expansion--policyranking--delivery)
- [8) Contracts: schemas + partitioning + idempotency](#8-contracts-schemas--partitioning--idempotency)
- [9) Caches, precomputes, and fanout strategies](#9-caches-precomputes-and-fanout-strategies)
- [10) Degraded modes / load shedding](#10-degraded-modes--load-shedding)
- [11) Observability + ops playbooks](#11-observability--ops-playbooks)
- [12) Trade-offs and design choices](#12-trade-offs-and-design-choices)
- [13) Appendix: “hard cases” checklist](#13-appendix-hard-cases-checklist)

---

## 0) Mental model

LinkedIn has two tightly coupled distribution systems:

1) **Feed distribution** answers:
> “What should this member see *now* when they open Home?”

2) **Notification distribution** answers:
> “What should interrupt this member (or be batched) to pull them back?”

They share primitives:
- the **member graph** (connections, follows)
- **entities** (posts, comments, reactions, shares, jobs)
- **policy** (privacy, blocks, visibility)
- **relevance scoring** + **feedback loops**

A practical way to think about it:
- Feed = **supply matching + ranking** (limited “slots” on screen)
- Notifications = **attention management** (interrupt vs batch), using many of the same signals

**Key difference:**
- Feed is often **session-scoped ranking** (what’s best *for this session*), tolerant of slight staleness.
- Notifications are often **event-scoped** (this event happened), less tolerant of missing a critical event, but must control spam.

---

## 1) Non-functional requirements (NFRs) / SLOs

Indicative targets for a large consumer social network.

### Feed latency
- **Home feed request end-to-end:** p50 < **200 ms**, p95 < **600 ms** (server-side), excluding network
- **Candidate gather stage:** p95 < **150 ms**
- **Heavy ranking stage:** p95 < **250 ms** (model server + feature fetch)

### Notifications latency
- **Direct social events** (comment on your post, mention, connection request): p95 < **2 s** to in-app notification record
- **Push dispatch for high-priority:** p95 < **10 s** from event commit (provider throttles may extend)

### Availability / durability
- Feed serving availability **99.95%** monthly (degraded ranking allowed)
- Notification pipeline availability **99.95%** monthly
- At-least-once processing + **idempotent writes**; bounded duplication is acceptable

### Correctness & safety
- **Fail-closed** for user-respecting policy (privacy/blocks/mute). If uncertain, suppress or degrade rather than leak.
- Deletes and privacy changes should converge quickly: “soft guarantee” within **minutes**, “hard” within **hours** (via backfill sweeps).

---

## 2) Back-of-envelope sizing

Numbers are illustrative; goal is to reason about hotspots and partitioning.

### Example assumptions
- Daily active members (DAU): **50M**
- Feed opens per DAU/day: **5**
- Avg feed items served per open: **30**
- New content objects/day (posts + reshares): **50M**
- Engagement events/day (reactions + comments): **500M**
- Peak-to-average ratio: **8×**

### Derived volumes (rough)
- **Feed requests/day:** 50M × 5 = **250M** → avg **~2.9k req/s**, peak **~23k req/s**
- **Items served/day:** 250M × 30 = **7.5B items/day**
- **Event ingest (engagement):** 500M/day → avg **~5.8k events/s**, peak **~46k events/s**

Implication: hottest paths are:
- feed request fan-out (candidate gather + ranking)
- notification recipient expansion + push dispatch

---

## 3) High-level architecture

```text
+--------------------+         +---------------------+
| LinkedIn Clients   |         | Identity + Auth     |
| iOS/Android/Web    | <-----> | Member profiles     |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Read APIs          |         | Graph Service       |
| (Home, Notifs)     | <-----> | (connections,       |
+---------+----------+         |  follows, blocks)   |
          |                    +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feed Serving       | ----->  | Feed Candidate Gen  |
| (cache + paginate) |         | (multi-source)      |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feed Ranking       | <-----> | Feature Store /     |
| (multi-stage)      |         | Enrichment outputs  |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feedback + Analytics| -----> | Model Training +    |
| (events/labels)     |        | Experimentation     |
+--------------------+         +---------------------+

Write + event foundation

+--------------------+         +---------------------+
| Write APIs         | ----->  | Event Stream / Log  |
| (posts/comments)   |         | (durable pub/sub)   |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Entity Store +     |         | Enrichment Pipeline |
| Indexes            |         | (spam, topics, etc) |
+--------------------+         +---------------------+

Notifications path

Event Stream / Log -> Notification Orchestrator -> Policy/Ranking -> Delivery (push/in-app/email)
                                 |                      |
                                 v                      v
                          Prefs/Settings            Notification State Store
```

---

## 4) Data model (key entities)

- **Member**: `member_id`
- **Graph edge**: `(src_member_id, dst_member_id, edge_type)`
  - edge_type: connection, follow, block, mute
- **Post**: `post_id`, `author_id`, `visibility`, `created_at`, `language`, `topics[]`
- **Engagement**: `reaction_id`, `comment_id`, `share_id` (each links to `post_id` and `actor_id`)
- **Notification**: `notification_id`, `recipient_id`, `source_event_id`, `type`, `state`, `bundle_key?`

Stores / indexes you’d expect:
- Entity store (durable): posts/comments/reactions
- Graph store: adjacency lists + “can_view” checks
- Feature store (online): member/entity aggregates, embeddings (optional)
- Notification state store: delivered/opened/dismissed + dedupe

---

## 5) Event-driven foundation: ingestion + enrichment

### 5.1 Write path (authoring)
1. Client calls `CreatePost` / `CreateComment` / `React`.
2. Write API validates auth + basic policy (rate limits, blocks, restricted content).
3. Entity is persisted.
4. Event emitted to stream: `PostCreated`, `CommentCreated`, `ReactionCreated`.

### 5.2 Enrichment pipeline
Async consumers attach metadata used by feed/notifs:
- language detection, topic classification
- spam / low-quality scoring
- author reputation signals
- entity embeddings (optional)

Design principle: **don’t block writes** on enrichment; downstream systems should tolerate partial features.

### 5.3 “Correction” events
To handle edits/deletes/privacy changes you typically also need:
- `PostUpdated` / `PostVisibilityChanged`
- `PostDeleted` (tombstone)
- `MemberBlocked` / `Unblocked`

These are crucial for feed/notifs convergence.

---

## 6) Feed: candidate generation → ranking → serving

### 6.1 Candidate sources
For viewer `V`, candidates commonly come from:
- **Network**: posts from connections, followed people/companies
- **Neighborhood**: 2-hop graph; “people in your ecosystem”
- **Topic / interest**: hashtags, topics, inferred interests
- **Intent / session context**: job seeking, hiring, learning
- **Exploration**: controlled testing of new creators/topics

### 6.2 Feed request flow (sequence)

```text
Client
  |  GET /v1/home-feed?cursor&limit
  v
Read API / Edge
  |-- fetch experiment bucket (sticky) -------------------->
  |-- fetch viewer snapshot (prefs, locale, blocks) ------->
  |-- gather candidates (multi-source, cached indexes) ---->
  |-- policy filter (visibility/blocks/safety) ------------>
  |-- light rank (fast) ----------------------------------->
  |-- heavy rank (model + online features) ---------------->
  |-- post-process (diversity, freshness, dedupe) --------->
  |-- write impression log async -------------------------->
  v
Response (items + cursor)
```

### 6.3 Candidate generation architecture
At request time (fanout-on-read), gather candidate IDs from:
- graph-based indexes (connections/follows)
- topic/trending indexes
- “recently engaged” indexes (e.g., “new comments on posts you interacted with”)

To keep latency low:
- use **precomputed candidate pools** for some sources (e.g., “top posts in topic T today”)
- cache “viewer neighborhood” expansions (connections/follows)

### 6.4 Ranking: multi-stage
Common pattern:
1) **Policy filter**
   - visibility, blocks, spam suppression
2) **Light ranker** (fast)
   - downselect thousands → few hundred
3) **Heavy ranker** (ML + richer features)
   - score top few hundred → final top N
4) **Post-processing**
   - diversity constraints (authors/topics)
   - freshness mix
   - session-level dedupe (avoid repeat exposures)

Typical feature groups (plausible):
- viewer↔author affinity (graph distance, prior interactions)
- content quality (author reputation, spam score)
- freshness/velocity (recent engagement rate)
- topical relevance (topics, embeddings)
- negative feedback risk (hide/report propensity)

### 6.5 Feed serving + pagination
Concerns:
- stable pagination tokens (avoid duplicates/holes)
- cache top-N for short windows (seconds to minutes)
- consistent **experiment assignment** per member

#### API contract (example)
`GET /v1/home-feed?member_id=...&cursor=...&limit=30`

Response:
```json
{
  "member_id": "m_123",
  "items": [
    {"type": "post", "post_id": "p_1", "reasons": ["connection", "topic_affinity"]},
    {"type": "post", "post_id": "p_2", "reasons": ["neighborhood_trending"]}
  ],
  "next_cursor": "c_abc",
  "experiment": {"bucket": "feed_ranker_v5_treatment"}
}
```

### 6.6 Feedback loop
Capture and stream:
- impressions, dwell, clicks
- likes/comments/shares
- hides/reports
- follow/unfollow after exposure

Use these for:
- online counters (velocity, freshness)
- offline labels (training)
- member preference updates (interest model)

---

## 7) Notifications: event → recipient expansion → policy/ranking → delivery

### 7.1 Notification orchestration flow (sequence)

```text
Write API commits CommentCreated
  |
  |  publish CommentCreated(event_id, post_id, actor_id, mentions[])
  v
Event Stream
  |
  |  Notification Orchestrator consumes
  |    - expand recipients (author, mentions, thread participants)
  |    - generate candidates with dedupe_key
  v
Notification State Store (idempotent upsert)
  |
  |  Policy / Budgeting
  |    - prefs, blocks, quiet hours
  |    - per-recipient rate limits (push budget)
  v
Delivery Router
  |-- in-app record available immediately
  |-- push dispatch for eligible/high-priority
  |-- optional email/digest
  v
Provider (APNs/FCM/Email)
```

### 7.2 Recipient expansion
A **Notification Orchestrator** consumes events and expands recipients.

Examples:
- `ReactionCreated` → notify post author; potentially notify commenters (if subscribed)
- `CommentCreated` → notify post author; notify mentioned members; notify thread participants
- `ConnectionRequestCreated` → notify recipient

Output: `NotificationCandidateCreated` events.

**Fanout control:** Recipient expansion is the “blast radius” step. It typically needs:
- caps per event (e.g., thread participants capped)
- sampling for low-priority edges
- batching (write candidates in chunks)

### 7.3 Hard policy / eligibility gates
Before ranking/delivery:
- member settings (mute types)
- blocks/privacy
- rate budgets (push per hour/day)
- dedupe/idempotency
- safety policy gates (don’t push unsafe content)

### 7.4 Ranking + bundling
Notifications compete for attention.

Useful primitives:
- **Priority tier**
  - P0 security, P1 direct social, P2 network activity, P3 marketing
- **Bundle key** for aggregation
  - e.g., `bundle_key = recipient_id + ":" + post_id + ":" + type`
- **Quiet hours / digest windows**

Bundling examples:
- “X and 12 others reacted to your post”
- “New comments on a post you follow”

### 7.5 Delivery channels
- In-app: reliable and cheap
- Push: interruptive, must be budgeted and bundled
- Email: digests, alerts (user-controlled)

Delivery service responsibilities:
- provider integration (APNs/FCM)
- retries/backoff
- write delivery outcomes to notification state store

### 7.6 Notification state machine
States:
- `candidate` → `suppressed|bundled|delivered` → `opened|dismissed`

Idempotent keys:
- candidate dedupe: `recipient_id + source_event_id + type`
- push attempt dedupe: `notification_id + device_id + channel`

---

## 8) Contracts: schemas + partitioning + idempotency

Assume a durable log (Kafka/Pulsar/Kinesis-like).

### 8.1 Partitioning recommendations
- Core write events (post/comment/reaction): partition by **entity_id** (e.g., `post_id`) to preserve per-entity ordering
- Notification candidates/decisions: partition by **`recipient_id`** (per-recipient ordering + budgeting)
- Feed feedback events: partition by **`member_id`**

Ordering guarantees:
- hard ordering only within a partition
- product-level “good enough”: per-recipient notification ordering; per-post engagement ordering

### 8.2 Example events

`feed.post_created.v1`
```json
{
  "schema_version": 1,
  "event_id": "evt_01...",
  "event_type": "post_created",
  "occurred_at_ms": 1762300000000,
  "post_id": "p_123",
  "author_id": "m_9",
  "visibility": "public|connections|custom"
}
```

`notifications.candidate_created.v1`
```json
{
  "schema_version": 1,
  "event_id": "evt_02...",
  "event_type": "notification_candidate_created",
  "occurred_at_ms": 1762300001000,
  "recipient_id": "m_42",
  "source_event_id": "evt_01...",
  "type": "comment|reaction|mention|connection_request",
  "entity_refs": {"post_id": "p_123", "comment_id": "c_7"},
  "dedupe_key": "m_42:evt_01:comment"
}
```

`notifications.feedback_event.v1`
```json
{
  "schema_version": 1,
  "event_id": "evt_03...",
  "event_type": "notification_feedback",
  "occurred_at_ms": 1762300010000,
  "recipient_id": "m_42",
  "notification_id": "n_abc",
  "action": "opened|dismissed|mute_type|disable_push",
  "client": {"platform": "ios|android|web"}
}
```

---

## 9) Caches, precomputes, and fanout strategies

### 9.1 What to cache
- viewer graph neighborhood expansions (connections/follows)
- per-member preference snapshots (notification settings)
- hot entity aggregates (recent engagement velocity)
- per-recipient rate-limit buckets

TTL guidance (indicative):
- graph expansions: minutes
- prefs snapshots: 10–60s (invalidate on change events)
- velocity counters: seconds

### 9.2 Fanout options

**Fanout-on-read (typical for large feeds)**
- Pros: avoids massive write amplification; adapts ranking per session
- Cons: expensive read path; needs aggressive caching/precompute

**Fanout-on-write (classic “push feed”)**
- Pros: very fast reads; simple serving
- Cons: huge write amplification (celebrity problem), hard to re-rank

**Hybrid**
- Fanout-on-write for *bounded* audiences (e.g., your direct connections)
- Fanout-on-read for broader discovery sources

LinkedIn-like systems often land in hybrid territory.

---

## 10) Degraded modes / load shedding

Hotspots are dominated by fan-out:
- viral posts causing huge reaction/comment volume
- massive recipient expansion (mentions, thread participants)
- push provider throttling

Explicit degraded policies:
1) **Protect P0/P1 notifications**; drop/batch P2/P3
2) Widen bundling windows (e.g., 30s → 5m)
3) Switch some notifications to **in-app only** (no push)
4) Feed ranking fallback:
   - if heavy ranker down, use light ranker + cached candidate pools
5) Admission control:
   - cap per-entity fan-out work; sample low-priority recipients

---

## 11) Observability + ops playbooks

### Feed
- request latency (p50/p95), candidate count, cache hit rates
- CTR/dwell, hides/reports
- diversity metrics (author/topic concentration)

### Notifications
- candidates created/s, suppressed/bundled/delivered rates
- pushes/member/day (p50/p95), open rate by type
- provider error rates + throttling

### Pipeline health
- stream lag per consumer group
- DLQ / poison-pill rate (bad events)
- idempotency conflict rate (unexpected duplicates)

### Common playbooks
- **Provider throttling spike:** switch to in-app-only for P2/P3; widen bundling; reduce concurrency
- **Consumer lag:** scale consumers; pause low-priority topics; ensure backpressure works
- **Bad deploy in ranking:** flip experiment flag to fallback; serve cached pools; alert on guardrails

---

## 12) Trade-offs and design choices

- Fanout-on-read vs hybrid fanout-on-write
- Multi-stage ranking to balance latency and quality
- Push aggressiveness vs long-term engagement (budgets, digests)
- Eventual consistency acceptable for counts/digests; stronger consistency needed for settings and security notifications

---

## 13) Appendix: hard cases checklist

Things that break naive designs:

- **Privacy change after distribution** (visibility narrowed): ensure previously eligible items/notifs are suppressed going forward; consider retrospective cleanup sweeps.
- **Delete/tombstone:** remove from caches; invalidate feed cursors where possible; suppress pending notifications.
- **Block/mute changes:** fail-closed; propagate quickly; ensure cache invalidation.
- **Idempotency:** at-least-once streams produce duplicates; require dedupe keys + idempotent upserts.
- **Backfills:** replay events to rebuild indexes; require versioned schemas and reprocessing safety.
- **Celebrity problem:** writer has tens of millions of followers → avoid fanout-on-write explosion.
- **Notification spam:** without budgets/bundles, user churn increases; require per-recipient caps and quiet hours.

---

## Related repos
- Systems explained: https://github.com/004mayank/product-systems-explained
- Teardowns: https://github.com/004mayank/product-teardowns
- PRDs: https://github.com/004mayank/product-prd
