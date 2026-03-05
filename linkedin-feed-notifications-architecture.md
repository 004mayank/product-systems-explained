<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/linkedin.png" 
    alt="LinkedIn logo" 
    width="200"
  />
</p>

# LinkedIn Home Feed + Notifications — System Architecture (Plausible)

**Product:** LinkedIn (Home Feed + Notifications)

**Audience:** Product Managers / Engineers

**Goal:** Describe a plausible, implementation-oriented architecture for **(a) the LinkedIn Home Feed** and **(b) the notifications system**, grounded in publicly observable product behavior and generic system design patterns (no proprietary internals).

**Version history**
- **V1:** Conceptual decomposition + end-to-end flows + trade-offs
- **V2 (current):** Adds explicit **NFRs/SLOs**, **sizing assumptions**, **API/event contracts**, **partitioning/idempotency**, **cache strategy**, and more concrete **degraded-mode** policies.

**Scope (V2)**
- Content creation → ingestion → enrichment
- Feed: candidate generation → ranking → serving → feedback
- Notifications: event → recipient expansion → eligibility/rules → ranking/bundling → delivery → feedback
- Cross-cutting: identity/graph, privacy/policy, experimentation, observability, reliability

**Non-goals**
- Exact model architectures, feature lists, or confidential thresholds
- Ads auction internals (we note where ads plug in)
- Full trust & safety moderation system design (we only include policy gates relevant to distribution)

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
- Feed = **supply matching + ranking**
- Notifications = **attention management** (interrupt vs batch) using many of the same signals

---

## 1) Non-functional requirements (NFRs) / SLOs (V2)

These are indicative targets for a large consumer social network.

### Feed latency
- **Home feed request end-to-end:** p50 < **200 ms**, p95 < **600 ms** (server-side), excluding network
- **Candidate gather stage:** p95 < **150 ms**
- **Heavy ranking stage:** p95 < **250 ms** (often via model server + feature fetch)

### Notifications latency
- **Direct social events (comment on your post, mention, connection request):** p95 < **2 s** to in-app notification record
- **Push dispatch for high-priority:** p95 < **10 s** from event commit (provider throttles may extend)

### Availability / durability
- Feed serving availability **99.95%** monthly (degraded ranking allowed)
- Notification pipeline availability **99.95%** monthly
- At-least-once stream processing + **idempotent writes**; bounded duplication is acceptable

### Cost constraints
- Avoid per-request heavy joins; rely on **precomputed indexes + caches**
- Prefer metadata-only push payloads; clients fetch full content via authenticated APIs

---

## 2) Back-of-envelope sizing (assumptions → derived QPS)

Numbers are illustrative; the goal is to reason about hotspots and partitioning.

### Example assumptions
- Daily active members (DAU): **50M**
- Feed opens per DAU/day: **5**
- Avg feed items served per open: **30**
- New content objects/day (posts + reshares): **50M**
- Engagement events/day (reactions + comments): **500M**
- Peak-to-average ratio: **8×** (diurnal + spikes)

### Derived volumes (rough)
- **Feed requests/day:** 50M × 5 = **250M** → avg **~2.9k req/s**, peak **~23k req/s**
- **Items served/day:** 250M × 30 = **7.5B items/day**
- **Event ingest (engagement):** 500M/day → avg **~5.8k events/s**, peak **~46k events/s**
- **Notification candidates:** depends heavily on recipient expansion (comments/reactions can notify multiple parties); design must handle **fan-out amplification**.

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
| Feed Serving        | -----> | Feed Candidate Gen  |
| (cache + paginate)  |        | (multi-source)      |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feed Ranking        | <----> | Feature Store /     |
| (multi-stage)       |        | Enrichment outputs  |
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
| Entity Store +      |       | Enrichment Pipeline |
| Indexes             |       | (spam, topics, etc) |
+--------------------+         +---------------------+

Notifications path

Event Stream / Log -> Notification Orchestrator -> Notif Policy/Ranking -> Delivery (push/in-app/email)
                                 |                         |
                                 v                         v
                          Prefs/Settings             Notification State Store
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

### 5.1 Write path
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

Design principle: **don’t block writes** on enrichment; downstream systems should handle partial features.

---

## 6) Feed (V2): candidate generation → ranking → serving

### 6.1 Candidate sources
For viewer `V`, candidates typically come from multiple sources:
- **Network**: posts from connections, followed people/companies
- **Neighborhood**: 2-hop graph; “people in your ecosystem”
- **Topic / interest**: communities, hashtags, inferred interests
- **Search / intent**: if the session implies job seeking, hiring, learning, etc.
- **Exploration**: controlled testing of new creators/topics

### 6.2 Candidate generation architecture
At request time (fanout-on-read), gather candidate IDs from:
- graph-based indexes (connections/follows)
- topic/trending indexes
- recent-engagement indexes (e.g., “new comments on posts you engaged with”)

To keep latency low:
- use **precomputed candidate pools** for some sources (e.g., “top posts in topic T today”)
- cache “viewer neighborhood” expansions (connections/follows)

### 6.3 Ranking: multi-stage
A common pattern:
1) **Policy filter**
   - visibility, blocks, spam suppression
2) **Light ranker** (fast)
   - downselect thousands → few hundred
3) **Heavy ranker** (ML + richer features)
   - score top few hundred → final top N
4) **Post-processing**
   - diversity constraints (authors/topics)
   - freshness mix
   - session-level dedupe (don’t show same post repeatedly)

### 6.4 Feed serving + pagination
Concerns:
- stable pagination tokens (avoid duplicates/holes)
- cache top-N short window (seconds to minutes)
- consistent **experiment assignment** for member

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

### 6.5 Feedback loop
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

## 7) Notifications (V2): event → recipient expansion → policy/ranking → delivery

### 7.1 Notification candidate creation (orchestration)
A **Notification Orchestrator** consumes events and expands recipients.

Examples:
- `ReactionCreated` → notify post author; potentially notify commenters (if subscribed)
- `CommentCreated` → notify post author; notify mentioned members; notify thread participants
- `ConnectionRequestCreated` → notify recipient

Output: `NotificationCandidateCreated` events.

### 7.2 Hard policy / eligibility gates
Before ranking/delivery:
- member settings (mute types)
- blocks/privacy
- rate budgets (push per hour/day)
- dedupe/idempotency
- safety policy gates (don’t push unsafe content)

### 7.3 Ranking + bundling
Notifications compete for attention.

V2 recommends explicit primitives:
- **Priority tier** (P0 security, P1 direct social, P2 network activity, P3 marketing)
- **Bundle key** for aggregation:
  - e.g., `bundle_key = recipient_id + ":" + post_id + ":" + type`
- **Quiet hours / digest windows**

### 7.4 Delivery channels
- In-app: reliable and cheap
- Push: interruptive, must be budgeted and bundled
- Email: digests, alerts (user-controlled)

Delivery service responsibilities:
- provider integration (APNs/FCM)
- retries/backoff
- write delivery outcomes to notification state store

### 7.5 Notification state machine
States:
- `candidate` → `suppressed|bundled|delivered` → `opened|dismissed`

Idempotent keys:
- candidate dedupe: `recipient_id + event_id + type`
- push attempt dedupe: `notification_id + device_id + channel`

---

## 8) Contracts (V2): event schemas + partitioning/idempotency

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

## 9) Cache strategy (V2)

Caches are essential to hit SLOs.

What to cache:
- viewer graph neighborhood expansions (connections/follows)
- per-member preference snapshots (notification settings)
- hot entity aggregates (recent engagement velocity)
- per-recipient rate-limit buckets

TTL guidance (indicative):
- graph expansions: minutes
- prefs snapshots: 10–60s (invalidate on change events)
- velocity counters: seconds

Consistency posture:
- be **fail-closed** on user-respecting suppressions (if mute/DND is uncertain, prefer suppress/batch over push)

---

## 10) Degraded modes / load shedding (V2)

Hotspots are dominated by fan-out:
- viral posts causing huge reaction/comment volume
- massive recipient expansion (mentions, thread participants)
- push provider throttling

Explicit degraded policies:
1) **Protect P0/P1 notifications**; drop/batch P2/P3
2) Widen bundling windows (e.g., 30s → 5m)
3) Switch some notifications to **in-app only** (no push)
4) Feed ranking fallback:
   - if heavy ranker down, use light ranker + cached top posts
5) Admission control:
   - cap per-entity fan-out work; sample low-priority recipients

---

## 11) Observability (V2)

Feed:
- request latency (p50/p95), candidate count, cache hit rates
- CTR/dwell, hides/reports
- diversity metrics (author/topic concentration)

Notifications:
- candidates created/s, suppressed/bundled/delivered rates
- pushes/member/day (p50/p95), open rate by type
- provider error rates + throttling

Guardrails:
- sudden increase in pushes/member or hides/reports
- stream lag, ranking latency spikes

---

## 12) Trade-offs

- Fanout-on-read vs hybrid fanout-on-write
- Multi-stage ranking to balance latency and quality
- Push aggressiveness vs long-term engagement (budgets, digests)
- Eventual consistency acceptable for counts/digests; stronger consistency needed for settings and security notifications

---

## Related repos
- Systems explained: https://github.com/004mayank/product-systems-explained
- Teardowns: https://github.com/004mayank/product-teardowns
- PRDs: https://github.com/004mayank/product-prd
