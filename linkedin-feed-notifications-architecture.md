<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/linkedin.png" 
    alt="LinkedIn logo" 
    width="200"
  />
</p>

# LinkedIn Home Feed + Notifications (V1) — System Architecture (Plausible)

**Product:** LinkedIn (Home Feed + Notifications)

**Audience:** Product Managers / Engineers

**Goal:** Explain a plausible, implementation-oriented architecture for **(a) the LinkedIn Home Feed** and **(b) the notifications system**, grounded in publicly observable product behavior and common large-scale system design patterns (no proprietary internals).

**Scope in this doc (V1)**
- Content creation → ingestion → feature enrichment
- Home Feed: candidate generation → ranking → serving → feedback loops
- Notifications: event → eligibility/rules → ranking/bundling → delivery (push/in-app/email)
- Cross-cutting: identity/graph, privacy, experimentation, observability, reliability

**Non-goals (V1)**
- Exact model architectures, feature lists, or confidential thresholds
- Ads auction internals (we mention where ads plug in, but don’t design the whole ads stack)
- Full anti-abuse content moderation system (we touch policy gates where they affect feed/notifications)

---

## 0) Mental model

LinkedIn has two coupled distribution systems:

1) **Feed distribution** (supply → matching → ranking) answers:
> “What should this member see when they open Home?”

2) **Notification distribution** (events → interrupt/batch) answers:
> “What should interrupt this member now (or later) to pull them back?”

They share the same primitives:
- the **graph** (people, companies, follows)
- **entities** (posts, comments, reactions, jobs)
- **policy** (privacy, blocks, visibility)
- **relevance scoring** and **feedback loops**

---

## 1) High-level architecture

```text
+--------------------+         +---------------------+
| LinkedIn Clients   |         | Identity + Auth     |
| iOS/Android/Web    | <-----> | Member profiles     |
+---------+----------+         +----------+----------+
          |                               |
          |                               v
          |                    +---------------------+
          |                    | Graph Service       |
          |                    | (connections/follows|
          |                    | blocks/privacy)     |
          |                    +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Write APIs         | ----->  | Event Stream / Log  |
| (posts/comments)   |         | (durable pub/sub)   |
+---------+----------+         +----------+----------+
          |                               |
          |                               v
          |                    +---------------------+
          |                    | Entity Store +      |
          |                    | Indexes             |
          |                    | (posts, comments,   |
          |                    | reactions, media)   |
          |                    +----------+----------+
          |                               |
          |                               v
          |                    +---------------------+
          |                    | Enrichment Pipeline |
          |                    | (features, spam,    |
          |                    | language, topics)   |
          |                    +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feed Candidate Gen | ----->  | Feed Ranking        |
| (graph + sources)  |         | (ML + rules)        |
+---------+----------+         +----------+----------+
          |                               |
          v                               v
+--------------------+         +---------------------+
| Feed Serving API   |         | Feedback + Analytics|
| (pagination, cache)| <-----  | (impressions, clicks|
+--------------------+         | hides, follows, etc)|
                               +----------+----------+
                                          |
                                          v
                               +---------------------+
                               | Model Training /    |
                               | Experimentation     |
                               +---------------------+

Notifications path (in parallel)

Event Stream / Log  ---> Notification Orchestrator ---> Notification Ranking/Policy ---> Delivery (Push/In-app/Email)
                                          |                        |
                                          v                        v
                                   Prefs/Settings            Notification State Store
```

Key idea: **everything starts as an event** (post created, someone reacted, someone viewed a job). Feed and notifications are downstream consumers that decide what to show, to whom, and when.

---

## 2) Core data entities (conceptual)

- **Member**: `member_id`
- **Graph edges**: connections, follows, blocks, mutes
- **Post**: `post_id`, `author_id`, visibility rules, media, language, topics
- **Engagement**: `reaction_id`, `comment_id`, `share_id`
- **Notification**: `notification_id`, `member_id`, `source_event_id`, `type`, `state`

### Stores / indexes you’d expect
- **Entity store** (durable): posts/comments/reactions
- **Graph store**: adjacency lists + fast “who can see what” checks
- **Feature store** (online): cached per-member and per-entity aggregates
- **Notification state store**: delivered/opened/dismissed + dedupe keys

---

## 3) Ingestion + enrichment (shared foundation)

### 3.1 Write path (post/comment)
1. Client calls `CreatePost` / `CreateComment`.
2. Write API validates auth + basic policy (rate limits, blocks, restricted content).
3. Entity is persisted.
4. An event is emitted: `PostCreated`, `CommentCreated`.

### 3.2 Enrichment pipeline
As events flow through, an async pipeline attaches metadata used later:
- language detection, topic classification
- spam / low-quality signals
- entity embeddings / similarity vectors (optional)
- author reputation features

**Why async?** Keep write latency low; feed/notifications can run in degraded mode using partial features.

---

## 4) Home Feed V1: candidate generation → ranking → serving

### 4.1 Candidate generation (where supply comes from)
For a viewer `V`, candidate sources include:
- posts from **connections** and **followed** entities
- posts from **people you may know** (limited exploration)
- viral posts in your “neighborhood” (2-hop graph, topic communities)
- reshares liked/commented by your network
- “recommended” content based on topics and prior engagement

Implementation sketch:
- A **fanout-on-read** approach is common for social feeds with heavy personalization:
  - at request time, gather candidate IDs from multiple indexes (graph, topic feeds, trending, etc.)
- Some systems also keep a **hybrid cache** (fanout-on-write for your close network) to reduce read-time cost.

### 4.2 Ranking (multi-stage)
Typical stages:
1. **Filters / policy gates**
   - block lists, privacy visibility
   - spam/unsafe content suppression
2. **Lightweight scorer** (fast)
   - downselect from thousands → a few hundred
3. **Heavy ranker** (slower, ML)
   - rank top ~50–200 items
4. **Post-processing**
   - diversity constraints (topics, authors)
   - freshness vs evergreen balance
   - “don’t show too many from the same author”

**Outputs:** ordered list of post IDs + rendering hints.

### 4.3 Serving + pagination
Feed Serving API concerns:
- stable pagination tokens (avoid duplicates/holes)
- caching of top-N for short windows
- per-member **experiment assignment** (A/B variants)

### 4.4 Feedback loop (the real engine)
Events captured:
- impression, dwell time, click, like, comment, share
- hide/report
- follow/unfollow

These events feed:
- near-real-time counters (e.g., engagement velocity)
- offline training (label generation)
- online feature updates (e.g., “V engaged with author A recently”)

---

## 5) Notifications V1: events → eligibility → ranking → delivery

LinkedIn notifications are a mix of:
- **social events** (someone viewed your profile, reacted, commented)
- **relationship events** (connection request, accepted)
- **content events** (mentions, new posts from followed entities)
- **system events** (job alerts, security)

### 5.1 Notification candidate creation
From the stream, generate candidates such as:
- `ReactionCreated` → notify post author (and sometimes commenters)
- `CommentCreated` → notify post author, previous commenters, mentioned people
- `ConnectionRequestCreated` → notify target member

This stage can be implemented by a **Notification Orchestrator** that:
- maps event → potential recipients
- attaches minimal context (IDs, type, workspace/locale)
- produces `NotificationCandidateCreated` events

### 5.2 Eligibility rules (hard constraints)
Before ranking, enforce:
- member notification settings (muted types)
- blocks/privacies
- rate budgets (per member per time)
- dedupe (same event shouldn’t produce two pushes)

### 5.3 Ranking + bundling
Notifications compete for attention; system needs:
- **priority tiers** (security > direct social > marketing)
- **bundling** (“X people reacted to your post”)
- **quiet hours / digests** (batch low priority)

A simple V1 approach:
- Rules-based priority + light ML relevance score
- Bundle within a short time window by `bundle_key`:
  - e.g., `bundle_key = member_id + post_id + notification_type`

### 5.4 Delivery channels
- **In-app notifications**: reliable, can show many, cheap
- **Push (APNs/FCM)**: expensive + interruptive; strict budgets
- **Email**: digests, job alerts (user-controlled)

A Delivery service handles retries, provider throttles, and writes delivery outcomes.

### 5.5 Notification state machine
States:
- `candidate` → `suppressed|bundled|delivered` → `opened|dismissed`

Store idempotently with keys like:
- `dedupe_key = member_id + event_id + type`

---

## 6) How feed + notifications interact (the coupling)

Notifications are effectively **feed distribution triggers**:
- They bring the member back, where the feed must deliver value quickly.

Shared signals:
- If a member frequently opens notifications about comments on their posts, feed can boost:
  - their own post threads
  - content from people who engage with them

Shared policy and quality:
- If content is downranked in feed for spam/low-quality, notifications for it should also be suppressed or batched.

---

## 7) Reliability + degraded modes (V1)

Common failure cases:
- stream lag/backlog
- ranking service latency spike
- push provider throttling

Degraded behaviors:
- prioritize “must deliver” notification types; drop/batch low-priority
- widen bundling windows
- fall back to cached/heuristic feed ranking if heavy model is unavailable

---

## 8) Observability (what to measure)

Feed:
- request latency, cache hit rate
- CTR, dwell, hides/reports
- diversity metrics (author/topic concentration)

Notifications:
- sends per member/day, push vs in-app ratio
- open rate by type
- bundling rate
- provider errors (APNs/FCM)

Guardrails:
- notification overload (spike in pushes/member)
- feed quality regressions (hides/reports up)

---

## 9) Trade-offs (V1)

- **Fanout-on-read vs fanout-on-write**: read-time personalization vs write-time cost
- **Model complexity vs latency**: multi-stage ranking to stay fast
- **Push aggressiveness vs long-term engagement**: budgets + digests avoid fatigue
- **Consistency vs availability**: eventual consistency acceptable for counts/digests; stronger guarantees for settings (mute) and security notifications

---

## Related repos
- Systems explained: https://github.com/004mayank/product-systems-explained
- Teardowns: https://github.com/004mayank/product-teardowns
- PRDs: https://github.com/004mayank/product-prd
