<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/slack.png" 
    alt="Slack logo" 
    width="200"
  />
</p>

# Slack Messaging - Notification Load Shaping (NYR) System Architecture

**Product:** Slack (messaging + notifications)
**Audience:** Product Managers / Engineers
**Goal:** Describe a plausible, implementation-oriented system architecture for **notification load shaping** and **Not-Yet-Relevant (NYR)**-style ranking, grounded in **publicly observable Slack behavior** and **generic system design patterns** (no proprietary internals).

**Scope in this doc :**
- Message → notification candidate generation
- NYR labeling/scoring → ranking → delivery (cross-device)
- User feedback loops (Not relevant / Snooze / Mute) → preference updates
- Digest / catch-up generation
- Reliability, privacy/safety, and observability for the above
- API contracts + event schemas for candidate generation, NYR scoring, delivery, preferences, and feedback
- **NEW in V3:** scalability sizing assumptions, queue/partitioning design, hotspot mitigation, cache strategy, and degraded-mode/load-shedding policies

**Version history**
- **V1:** Conceptual decomposition + flows + reliability/privacy/observability
- **V2:** Adds API/event contract layer, non-functional requirements, and explicit trade-offs/decisions
- **V3 (current):** Adds sizing + partitioning/ordering semantics, hotspot mitigation, cache strategy, and operational degraded modes

---

## 1) What problem “notification load shaping” solves

Slack’s messaging is high volume and multi-context (workspaces, channels, threads, DMs). A notification system must:
- **Notify quickly** for urgent and explicitly addressed events (mentions, DMs)
- **Avoid overload** (too many pings → user disables notifications)
- **Personalize** relevance by user, device, time, and workspace norms
- **Stay consistent across devices** (desktop + mobile + web)

NYR in this context means:
> A message can be “important for the workspace/channel,” but **not the right time** or **not relevant enough** to interrupt this user on this device.

---

## 2) Non-functional requirements (NFRs)

These NFRs make the design constraints explicit. Values are indicative targets for a large SaaS messaging product.

### Latency SLOs (end-to-end)
- **Mention/DM push eligibility decision:** p50 < **150 ms**, p95 < **500 ms** from `MessageCreated` event ingestion
- **Push dispatch (to APNs/FCM):** p95 < **2 s** from message commit for eligible high-urgency events
- **In-app (websocket) fanout:** p95 < **300 ms** to active clients (subject to network)
- **Preference change propagation (mute/snooze/DND):** p95 < **5 s** cross-device effect
- **Digest generation:** within **5–15 min** of scheduled window; freshness > strict ordering

### Availability / durability
- **Notification pipeline availability:** **99.95%** monthly (degraded mode allowed)
- **No data loss** for notification state transitions (candidate/delivered/opened) beyond configured retention windows
- **At-least-once** event processing with idempotent writes; bounded duplication

### Cost constraints (practical)
- Favor **metadata-only** ranking and minimal push payloads (reduces storage + egress)
- Push provider calls are expensive at scale → aggressive **bundling** and **rate budgets**
- Feature computation should be **cacheable** and/or **pre-aggregated**; avoid per-candidate heavy joins

### Multi-tenancy constraints
- Enforce per-workspace quotas and isolation:
  - noisy workspace cannot starve global queues
  - admin policies must be applied consistently

### (V3) Back-of-envelope sizing (assumptions → derived QPS)
These numbers are **illustrative** and intentionally rounded. The goal is to size the pipeline, pick partitioning strategies, and define degraded-mode triggers.

#### Example assumptions
- **Workspaces (tenants):** 1,000,000 total; 200,000 WAU (weekly active)
- **WAU users:** 30,000,000 (across all active workspaces)
- **Messages/day:** 2,000,000,000 (2B)
- **Peak-to-average ratio (diurnal + incident spikes):** 8×
- **% messages producing any candidate notification:** 20% (DMs, mentions, followed threads, “all messages” channels)
- **Avg candidates per eligible message:** 2.5 (many are 1:1 DM; some are channel mentions/thread participants)
- **Avg delivery targets per candidate:** 1.2 (in-app to active clients + push sometimes)
- **Avg push notifications per delivered candidate:** 0.6 (presence-aware suppression + bundling)

#### Derived volumes (rough)
- **Message ingest QPS (avg):** 2B / 86,400 ≈ **23k msg/s**
- **Message ingest QPS (peak):** 23k × 8 ≈ **185k msg/s**
- **Candidate generation QPS (avg):** 23k × 20% × 2.5 ≈ **11.5k candidates/s**
- **Candidate generation QPS (peak):** 185k × 20% × 2.5 ≈ **92k candidates/s**
- **Scoring QPS (peak):** same order as candidates; typically **~100k/s** (batch scoring per user can reduce RPC overhead)
- **Delivery attempt QPS (peak):** candidates × delivery_targets ≈ **110k attempts/s**
- **Push provider QPS (peak, post-bundling):** attempts × push fraction ≈ **~60k push sends/s**

#### Implications
- The system must be designed for **burst absorption** (queues + budgets) and **multi-tenant fairness**.
- Hotspots are dominated by **fan-out amplification** (big channels, @here/@channel, incident rooms) rather than baseline chat volume.

---

## 3) High-level architecture (conceptual)

This is a **conceptual decomposition** consistent with typical SaaS messaging systems.

```text
+-------------------+        +-------------------+        +-----------------------+
| Slack Clients     |        | Messaging APIs    |        | Event / Stream Bus    |
| iOS/Android/Web   |<-----> | (Gateway, Auth)   | -----> | (pub/sub, log, queue) |
+-------------------+        +-------------------+        +-----------------------+
          |                               |                          |
          |                               |                          v
          |                               |             +---------------------------+
          |                               |             | Notification Orchestrator |
          |                               |             | (candidate generation)    |
          |                               |             +------------+--------------+
          |                               |                          |
          |                               |                          v
          |                               |             +---------------------------+
          |                               |             | Ranking / Classification  |
          |                               |             | (NYR scoring + rules)     |
          |                               |             +------------+--------------+
          |                               |                          |
          |                               |                          v
          |                               |             +---------------------------+
          |                               |             | Delivery Service          |
          |                               |             | (push, in-app, email)     |
          |                               |             +------------+--------------+
          |                               |                          |
          v                               v                          v
+-------------------+        +-------------------+        +--------------------------+
| Preferences       |        | Feature Store     |        | Experimentation / Flags  |
| (user/workspace   |        | (conceptual:      |        | (rollouts, A/Bs)         |
|  settings, mutes) |        |  embeddings,      |        +--------------------------+
+-------------------+        |  aggregates)      |
          ^                  +-------------------+
          |
          |         +----------------------------+      +--------------------------+
          +---------| Feedback + Analytics       |----->| Analytics Pipeline       |
                    | (events, labels, outcomes) |      | (warehouse, dashboards)  |
                    +----------------------------+      +--------------------------+
```

### Component responsibilities (PM/Eng view)

- **Messaging APIs**: authenticate, accept message writes/reads, expose conversation context.
- **Event/Stream Bus**: durable fan-out of “message created / reaction / thread reply” events.
- **Notification Orchestrator**: converts events into **notification candidates** for users/devices.
- **Ranking/Classification (NYR)**:
  - rule-based + model-based scoring
  - assigns an **urgency/relevance score** and **NYR label** per candidate
- **Preferences**: per-user/workspace notification settings, mutes, snoozes, channel-level overrides.
- **Feature Store (conceptual)**: derived signals used for scoring (e.g., relationship strength, recent activity).
- **Experimentation/Flags**: gradual rollouts, A/B tests of thresholds and models.
- **Delivery Service**: push gateways (APNs/FCM), websocket/in-app, email digests.
- **Feedback + Analytics**: collects outcomes (opened, dismissed, “not relevant”) and powers tuning.

---

## 4) Event flows

### Flow 1 - Message create → notification candidate generation

**Trigger:** a message is created (channel, DM, thread reply).

1. **Client → Messaging API:** `CreateMessage(workspace_id, channel_id, thread_ts?, text, attachments...)`
2. **Message persisted** in primary message store (plus indexes for search).
3. **Event emitted**: `MessageCreated` to the stream bus (includes IDs + minimal metadata).
4. **Notification Orchestrator consumes** `MessageCreated` and determines **candidate recipients** using:
   - explicit addressing: `@mentions`, `@here`, `@channel`
   - DM / group DM membership
   - channel membership + per-user notification preference (e.g., “mentions only”)
   - thread participants / subscribers (publicly observable behavior)
5. For each recipient, the orchestrator creates a **NotificationCandidate** with context:
   - message metadata (sender_id, channel_id, is_thread_reply, has_mention)
   - recipient state (notification settings snapshot, device availability)

**Output:** a set of candidates `{(recipient_user_id, message_id, candidate_type, context)}`.

---

### Flow 2 - NYR labeling/scoring → ranking → delivery on devices

1. **Orchestrator → Ranking service:** send candidate(s) for scoring.
2. **Ranking service** computes:
   - **Eligibility** (hard rules): muted channel, DND/snooze window, blocked sender, etc.
   - **Relevance score** (soft): likelihood user engages now
   - **Urgency class** (e.g., “must interrupt” vs “batch/digest”)
   - **NYR label**: “not yet relevant” if it’s worth surfacing later but not now
3. **Policy engine** applies workspace- and user-level constraints:
   - rate limits (max pushes per minute)
   - bundling windows (collapse multiple messages)
   - device-aware routing (mobile push vs desktop badge)
4. **Delivery service** routes to channels:
   - **Real-time in-app** (websocket) and **badges** for active clients
   - **Mobile push** via APNs/FCM with compact payload
   - **Email** only for opted-in/digest contexts
5. **Client rendering**:
   - shows notification
   - groups multiple notifications
   - allows quick actions (mute, snooze, mark as read)

**Important design note:**
- Actual message content is fetched by the client via authenticated APIs; notification payloads should contain **minimal necessary metadata**.

---

### Flow 3 - User feedback (not relevant/snooze) → preference updates → model/heuristic adjustments

**Feedback sources (observable in many products):**
- “Mute channel,” “Snooze notifications,” “Mark as read,” “Not relevant” (or implicit signals: dismissal without open)

1. **Client emits FeedbackEvent**: `(user_id, workspace_id, notification_id, action, timestamp)`
2. **Preferences service updates** durable settings when applicable:
   - channel muted until time
   - workspace notification level changed
   - thread follow/unfollow
3. **Feedback + Analytics pipeline** records:
   - outcome labels (opened, ignored, dismissed)
   - time-to-open, session context (device, local time bucket)
4. **Ranking service update loop** (batch or near-real-time):
   - adjust heuristic weights / thresholds
   - retrain models offline using aggregated, privacy-compliant data
   - ship new model versions via experimentation system

---

### Flow 4 - Digest / catch-up generation

A digest is a **batched summary** that trades immediacy for lower interruption cost.

1. **Scheduler** (or stream-triggered batcher) selects users eligible for digest:
   - user opted in, or system chooses catch-up during quiet hours
2. **Digest builder** queries:
   - NYR-labeled candidates accumulated during a time window
   - “high-signal” events (mentions, thread replies) that were not delivered as push
3. **Ranking service** orders digest items (top N) and produces explanations/sections:
   - “Mentions,” “Threads you follow,” “Channels with spikes,” etc.
4. **Delivery service** sends digest via:
   - in-app catch-up UI
   - email (optional)

---

## 5) Data model (key entities + IDs)

IDs shown are conceptual; Slack publicly uses workspace/channel identifiers and message timestamps in some contexts, but the architecture should not depend on a specific internal format.

### Core entities

- **User**
  - `user_id`
  - belongs to many `workspace_id`
  - has devices: `device_id[]`

- **Workspace**
  - `workspace_id`
  - policies: retention, notification defaults, admin rules

- **Channel**
  - `channel_id`
  - `workspace_id`
  - membership list or membership index

- **Message**
  - `message_id` (or `(channel_id, message_ts)` style key)
  - `workspace_id`, `channel_id`
  - `sender_user_id`
  - `thread_id` (nullable)
  - metadata flags: `has_mention`, `is_bot`, `is_file_share`, etc.

- **Thread**
  - `thread_id` (or root message key)
  - participant/subscriber index

- **Notification**
  - `notification_id`
  - `user_id`, `workspace_id`
  - `source_event_id` (e.g., message event)
  - `message_id` / `thread_id`
  - `type` (mention, dm, thread_reply, channel_activity, digest_item)
  - `state` (candidate, suppressed, delivered, opened, dismissed)
  - `nyr_label` (boolean or enum)
  - `score` (float), `policy_reason[]` (why suppressed/batched)
  - `delivery_targets` (mobile_push, desktop_badge, email, in_app)

### Indexes / derived views
- `ChannelMembership(workspace_id, channel_id) -> user_id[]`
- `UserNotificationPrefs(user_id, workspace_id) -> settings`
- `ThreadSubscriptions(thread_id) -> user_id[]`

---

## 6) API contracts + event schemas (V3)

This section turns the conceptual architecture into a **contract-first** design so teams can build services independently. The interfaces are intentionally generic (HTTP/JSON shown; gRPC equivalents are straightforward).

### Design principles
- **Events are the source of truth** for fan-out; services are **stateless where possible**.
- Use **at-least-once delivery** + **idempotent writes** (dedupe keys are part of the contract).
- Prefer **metadata and IDs** over message bodies; fetch full content via authenticated read APIs.

---

### 6.1 Core topics (stream bus)

#### (V3) Queue/topic design conventions (partitioning, ordering, replay)
Assume the stream bus is a **durable log** (Kafka/Pulsar/Kinesis-like) plus supporting queues (SQS/Rabbit-like) for specialized workloads.

**Partition keys (recommended)**
- `messaging.message_created.v1`: partition by **`workspace_id`** (baseline fairness), optionally sub-partition by `channel_id` for very large workspaces.
- Per-recipient downstream topics (candidates/decisions/delivery): partition by **`workspace_id:user_id`** to preserve per-user ordering and simplify budgeting.

**Ordering semantics (what we guarantee)**
- **Hard guarantee:** preserve ordering **within a partition**.
- **Product-aligned guarantee:** best-effort ordering for **a user’s notifications** (partition on `workspace_id:user_id`).
- **Not guaranteed:** global ordering across workspaces/channels.

**Idempotency + dedupe keys**
- Events: `event_id` is globally unique.
- Candidate creation dedupe key: `dedupe_key = workspace_id + ":" + user_id + ":" + event_id + ":" + candidate_type`.
- Delivery attempt dedupe key: `dedupe_key = notification_id + ":" + device_id + ":" + channel`.
- Storage writes must be **idempotent upserts** keyed by these dedupe keys.

**Replay strategy**
- Keep a **hot retention** window (e.g., 24–72h) on core topics for quick reprocessing.
- Persist a **cold archive** (object store) for longer replay/debug (e.g., 14–30d) depending on privacy/retention.
- Consumers must be able to:
  - rewind offsets for incident recovery
  - re-emit derived events with a new `schema_version` or `reprocess_job_id`
- Because processing is at-least-once, replay correctness relies on **idempotent state transitions** and **monotonic writes**.

#### `messaging.message_created.v1`
Emitted when a message is committed.

```json
{
  "schema_version": 1,
  "event_id": "evt_01J...",
  "event_type": "message_created",
  "occurred_at_ms": 1739000000000,
  "workspace_id": "w_123",
  "channel_id": "c_456",
  "message_id": "m_789",
  "thread_id": "t_001",
  "sender_user_id": "u_222",
  "message_kind": "channel|dm|group_dm",
  "flags": {
    "has_mentions": true,
    "is_thread_reply": true,
    "is_bot": false,
    "has_files": false
  },
  "mention_user_ids": ["u_111"],
  "mention_scope": "user|here|channel|null"
}
```

#### `notifications.feedback_event.v1`
Emitted by clients when a user interacts with a notification.

```json
{
  "schema_version": 1,
  "event_id": "evt_01J...",
  "event_type": "notification_feedback",
  "occurred_at_ms": 1739000005000,
  "workspace_id": "w_123",
  "user_id": "u_111",
  "notification_id": "n_abc",
  "action": "opened|dismissed|muted_channel|snoozed|marked_read|not_relevant",
  "client": {
    "platform": "ios|android|web|desktop",
    "app_version": "1.2.3",
    "device_id": "d_777",
    "timezone": "Asia/Kolkata"
  },
  "context": {
    "channel_id": "c_456",
    "thread_id": "t_001",
    "message_id": "m_789"
  }
}
```

#### (V3) Suggested derived topics
These topics keep services loosely coupled and enable replay/backfill.

- `notifications.candidate_created.v1`
  - Produced by orchestrator
  - **Partition key:** `workspace_id:user_id` (per-user ordering + budgeting)
  - **Payload:** candidate metadata + `dedupe_key`
- `notifications.decision_made.v1`
  - Produced by NYR scorer/policy engine
  - **Partition key:** `workspace_id:user_id`
  - **Payload:** decision + reasons + `review_after_ms` for NYR
- `notifications.delivery_attempted.v1`
  - Produced by delivery service
  - **Partition key:** `workspace_id:user_id` (or `notification_id`)
  - **Payload:** attempt state transitions + provider error codes

---

### 6.2 Notification Orchestrator (candidate generation)

**Purpose:** convert messaging events into per-user notification candidates.

#### API: `POST /v1/notification-candidates:generate`
Usually invoked by a stream consumer (not a public API).

**Request**
```json
{
  "request_id": "req_01J...",
  "source_event": {
    "event_type": "message_created",
    "event_id": "evt_01J...",
    "workspace_id": "w_123",
    "channel_id": "c_456",
    "message_id": "m_789",
    "thread_id": "t_001",
    "sender_user_id": "u_222",
    "mention_user_ids": ["u_111"],
    "mention_scope": "user"
  },
  "idempotency_key": "w_123:evt_01J...:candidate-gen"
}
```

**Response**
```json
{
  "request_id": "req_01J...",
  "candidates": [
    {
      "candidate_id": "cand_01J...",
      "workspace_id": "w_123",
      "user_id": "u_111",
      "message_id": "m_789",
      "thread_id": "t_001",
      "candidate_type": "mention|dm|thread_reply|channel_activity|digest",
      "features_hint": {
        "has_direct_mention": true,
        "is_thread_participant": true,
        "channel_is_muted": false
      }
    }
  ]
}
```

**Notes**
- Candidate generation can be implemented as a pure function over (event + membership + prefs snapshot).
- Any expensive joins should be pushed into **indexes** (membership, thread subscriptions) or caches.

---

### 6.3 NYR scoring / ranking service

#### API: `POST /v1/nyr:score`
Scores candidates and returns decisions + policy explanations.

**Request**
```json
{
  "request_id": "req_01J...",
  "workspace_id": "w_123",
  "user_id": "u_111",
  "candidates": [
    {
      "candidate_id": "cand_01J...",
      "message_id": "m_789",
      "thread_id": "t_001",
      "candidate_type": "mention",
      "event_id": "evt_01J..."
    }
  ],
  "context": {
    "now_ms": 1739000006000,
    "device_state": {
      "active_clients": ["desktop"],
      "push_enabled": true
    },
    "prefs_snapshot_etag": "prefs_etag_9f..."
  },
  "idempotency_key": "w_123:u_111:evt_01J...:score"
}
```

**Response**
```json
{
  "request_id": "req_01J...",
  "decisions": [
    {
      "candidate_id": "cand_01J...",
      "eligible": true,
      "decision": "deliver_now|defer_nyr|suppress|bundle",
      "nyr": {
        "label": "not_yet_relevant|relevant_now|never_relevant",
        "review_after_ms": 1739000306000
      },
      "score": {
        "relevance": 0.92,
        "urgency": "high|medium|low",
        "model_version": "nyr_ranker_2026_02_01"
      },
      "policy_reasons": [
        "direct_mention",
        "within_rate_budget",
        "user_not_in_dnd"
      ]
    }
  ]
}
```

**Decision semantics**
- `deliver_now`: route to delivery service immediately
- `defer_nyr`: store as NYR for later surfacing (catch-up, digest, in-app inbox)
- `suppress`: do not surface (e.g., muted channel, blocked sender)
- `bundle`: eligible but will be collapsed into an existing bundle key

---

### 6.4 Delivery service

#### API: `POST /v1/notifications:deliver`
Creates delivery attempts for targets and dispatches.

**Request**
```json
{
  "request_id": "req_01J...",
  "notification": {
    "notification_id": "n_abc",
    "workspace_id": "w_123",
    "user_id": "u_111",
    "source_event_id": "evt_01J...",
    "candidate_id": "cand_01J...",
    "message_ref": {"channel_id": "c_456", "message_id": "m_789", "thread_id": "t_001"},
    "type": "mention",
    "render_hint": {"title": "Mention", "body_preview": "@you …"}
  },
  "targets": [
    {"channel": "in_app", "client_types": ["desktop", "web"]},
    {"channel": "push", "provider": "apns|fcm", "device_id": "d_777"}
  ],
  "idempotency_key": "n_abc:deliver"
}
```

**Response**
```json
{
  "request_id": "req_01J...",
  "delivery_attempts": [
    {
      "attempt_id": "att_01J...",
      "notification_id": "n_abc",
      "target": {"channel": "push", "device_id": "d_777"},
      "state": "queued|sent|failed",
      "provider_message_id": "apns_..."
    }
  ]
}
```

**Bundling contract**
- Delivery may accept a `bundle_key` (e.g., `w_123:u_111:c_456`) to collapse multiple events.

---

### 6.5 Preferences service (settings, mutes, DND)

#### API: `GET /v1/preferences/{workspace_id}/{user_id}`
Returns a normalized view used by orchestrator and scorer.

```json
{
  "workspace_id": "w_123",
  "user_id": "u_111",
  "etag": "prefs_etag_9f...",
  "dnd": {"enabled": true, "until_ms": 1739003600000},
  "channel_overrides": {
    "c_456": {"muted": false, "notify": "mentions_only|all|none"}
  },
  "digest": {"enabled": true, "cadence": "daily|weekly", "hour_local": 9}
}
```

#### API: `POST /v1/preferences:apply`
Applies user actions (mute/snooze) with audit-friendly semantics.

```json
{
  "request_id": "req_01J...",
  "workspace_id": "w_123",
  "user_id": "u_111",
  "change": {
    "type": "mute_channel|snooze|set_dnd|set_channel_level",
    "channel_id": "c_456",
    "until_ms": 1739010000000
  },
  "idempotency_key": "w_123:u_111:mute:c_456:1739010000000"
}
```

---

### 6.6 State store contracts (notification state machine)

To support retries + cross-device convergence, treat state updates as **monotonic**.

Example conditional transition (conceptual):
- `PUT /v1/notification-state/{notification_id}` with `if_state=delivered` → `opened`

State enum:
- `candidate` → `suppressed|deferred_nyr|bundled|delivered` → `opened|dismissed`

---

## 7) Consistency + cross-device considerations

### What must be consistent
- **Read state / badge counts**: should converge across mobile/desktop/web.
- **Mute/snooze**: should apply quickly across devices.
- **Deduplication**: avoid double-notifying the same event across channels.

### Practical consistency model
- **Eventual consistency** is acceptable for:
  - badge count convergence
  - digest content (built from a window)
- **Stronger consistency** (or at least monotonicity) preferred for:
  - “delivered/opened” tracking (don’t regress states)
  - DND/snooze enforcement

### Patterns to achieve this
- Per-user **notification state store** with monotonic transitions:
  - `candidate -> delivered -> opened` (append-only event log or conditional updates)
- **Client-side reconciliation**:
  - clients fetch authoritative counts on app open
  - push notifications carry only hints (“new activity”) not authoritative counts

### (V3) Cache strategy (what, where, TTLs) + consistency impact
Caching is essential because candidate generation and scoring touch high-cardinality state (membership, prefs, presence).

#### What to cache
- **Workspace + channel membership indexes**
  - Key: `(workspace_id, channel_id)` → member shards / bloom / compressed bitsets
  - TTL: minutes; invalidate on join/leave events
- **Thread subscription/participant index**
  - Key: `thread_id` → subscribers
  - TTL: minutes; invalidate on follow/unfollow
- **User preference snapshots** (mute/snooze/DND/channel overrides)
  - Key: `(workspace_id, user_id)` → prefs + `etag`
  - TTL: 10–60s hot cache; hard invalidation on pref-change events
- **Presence / active-client hints**
  - Key: `(workspace_id, user_id)` → active clients, last_seen
  - TTL: 5–15s (very short)
- **Notification bundle state**
  - Key: `bundle_key` → current bundle counters + last_sent
  - TTL: bundling window + a safety margin (e.g., 10m)

#### Where to cache
- **In-process LRU** for per-host hot keys (lowest latency, small memory)
- **Distributed cache** (Redis/Memcached-like) for shared reads and rate-limit buckets
- **Edge/client caches** for read-only UX hints (badge count hints, last digest cursor)

#### Consistency impact (what can go wrong)
- Membership/prefs caches can be **stale** → risk false positives (notify someone who muted) or false negatives (miss a newly joined user).
- Mitigations:
  - Prefer **fail-closed** for suppressions that users expect (if unsure about mute/DND, err on suppress or in-app only)
  - Use **ETags** and conditional reads for prefs snapshots
  - Propagate invalidations via events (pref_changed, membership_changed)
  - Keep short TTLs on correctness-sensitive caches (prefs/presence)

---

## 8) Failure handling + reliability

### Idempotency
- Treat notification creation as idempotent by keying on:
  - `(user_id, source_event_id, notification_type)`
- Delivery attempts are idempotent by:
  - `(notification_id, device_id, delivery_channel)`

### Deduplication
- Dedup within a time window to prevent storms:
  - collapse multiple channel messages into one push “X new messages in #channel”
  - suppress if user is already active in the channel (client presence signal)

### Retries & backoff
- Retries for transient failures:
  - stream consumer retries with exponential backoff
  - push gateway retries with provider-aware limits
- Dead-letter queues for poison events (schema issues, bad payloads)

### (V3) Hotspot mitigation, rate limiting & load shedding
Hotspots typically come from **fan-out amplification** rather than baseline traffic:
- **Big channels** with thousands of members
- **Incident / war-room channels** with extremely high message rate
- **Spam storms** (bots/integrations gone wild; mention abuse)

#### Mitigation patterns (before “degraded mode”)
- **Presence-aware suppression:** if the user is active in the channel/thread, prefer in-app updates; avoid push.
- **Per-channel bundling keys:** collapse bursts into “N new messages in #channel” with a short **bundling window** (e.g., 15–60s).
- **Mention throttles:** strict limits for `@here/@channel` and app/bot mentions; require workspace policy + quotas.
- **Fan-out control:** for extremely large channels, generate a lightweight “channel activity” candidate and let the **client pull** details (reduces per-message recipient expansion).
- **Spam classification gate:** if sender/app is suspected spam, route to a quarantine path and require stricter thresholds.

#### Priority tiers (decide what to protect)
Define a small set of tiers that map to user expectations:
- **P0:** direct messages, direct @mentions, security/admin critical
- **P1:** thread replies you follow, small group DMs
- **P2:** channel activity (non-mentions), reactions, low-signal events
- **P3:** “FYI” / bulk events that can be digest-only

#### Budgets (fairness across tenants + users)
Budgets are enforced at multiple layers:
- **Per-user budgets:** max pushes/minute, max bundles/minute, max total notifications/day (soft)
- **Per-workspace budgets:** cap total push sends/s and candidate generation/s per workspace
- **Global budgets:** cap calls to external providers (APNs/FCM), cap ranking RPCs/s

Implementation sketch:
- Token buckets keyed by `workspace_id` and `workspace_id:user_id`
- Admission control checks **before** expensive scoring and before push dispatch

#### Degraded modes (explicit product behavior)
When queues back up, providers throttle, or downstream dependencies fail, switch modes by tier:
1. **Widen bundling windows** (e.g., 60s → 5m) for P2/P3
2. **Drop low-priority pushes:**
   - suppress P2/P3 **non-mentions** first
   - keep in-app/websocket updates best-effort
3. **Digest-only fallback:** route P2/P3 into digest/catch-up; deliver at scheduled windows
4. **Sampling:** for extremely hot channels, sample channel-activity events for non-engaged users
5. **Graceful ranking fallback:** if ML scorer is down, use rules + static thresholds

**Never degrade** (as much as possible): P0 events. If push providers are down, fall back to:
- in-app banner/badge (active clients)
- email/SMS only if explicitly configured and policy-approved

#### Operational triggers (examples)
- Consumer lag > X seconds on candidate/decision topics
- Push provider error rate > Y% or sustained throttling
- Ranking service latency p95 > threshold
- Workspace-level anomaly detector flags unusual mention rates

All degraded-mode transitions should be:
- controlled by **feature flags**
- recorded in an **audit log** (who/what/when)
- visible in dashboards (so PMs understand behavior changes)

### Ordering
- Notification ordering should be **best-effort** and relevance-based, not strict FIFO.
- For threads/DMs, preserve intuitive order in the client feed using message timestamps.

---

## 9) Privacy & safety considerations

### Least privilege
- Notification services should access:
  - message metadata needed for routing (mentions, channel/thread IDs)
  - preference settings
- Avoid broad access to message bodies; clients can fetch full content with user auth.

### On-device vs server-side
- **On-device** signals (local time, focus mode, notification permission state) can be used for last-mile decisions (e.g., whether to show a banner) without sending raw data server-side.
- **Server-side** handles:
  - routing, suppression, ranking using coarse/aggregated features

### Data retention
- Store only what’s needed for:
  - delivery reliability (short-lived queues)
  - user-configurable history (notification center)
  - model training (aggregated and policy-compliant)
- Respect workspace retention policies where applicable.

### Abuse / safety
- Protect against:
  - notification spam from bots/apps (quotas, app-level limits)
  - mention abuse (rate limit @channel/@here)
- Admin controls:
  - workspace policies for app notifications and escalation paths

---

## 10) Observability (logs/metrics/traces) + guardrails

### What to instrument
- **Counters**
  - candidates created
  - candidates suppressed (by reason)
  - notifications delivered (by channel: push/in-app/email)
  - user actions: opened/dismissed/snoozed/muted
- **Latency histograms**
  - message_create → candidate
  - candidate → score
  - score → delivered
- **Quality metrics** (proxy relevance)
  - open rate by type
  - dismiss-without-open rate
  - “not relevant” feedback rate
  - notification volume per user/day (p50/p95)
- **Reliability metrics**
  - dedupe hit rate
  - retry counts
  - DLQ size
  - push provider error rates (APNs/FCM)

### Tracing
- Distributed trace from `MessageCreated` through:
  - orchestrator → ranking → delivery
- Attach correlation IDs:
  - `event_id`, `notification_id`, `request_id`

### Guardrail alerts (examples)
- Sudden spike in notifications/user (possible regression)
- Suppression rate jumps (ranking outage or prefs read failure)
- Delivery latency p95 > threshold
- Push provider error rate increase
- Digest generation backlog

---

## 11) Trade-offs & decisions (V3)

### Decisions we commit to in V1/V2/V3
- **Event-driven fan-out** with a durable stream/log as the integration boundary.
- **At-least-once processing** everywhere, paired with **idempotency keys** and monotonic state transitions.
- **Metadata-first notifications:** delivery payloads contain IDs + small render hints; clients fetch content with user auth.
- **Policy before ML:** hard constraints (mutes, DND, admin policy, budgets) gate any model-based scoring.
- **Degraded modes are product features:** bundling/digests are first-class fallbacks during spikes/outages.

### Trade-offs (why these choices)
- **Immediate vs relevant:** NYR defers non-urgent interruptions to protect long-term engagement.
- **Consistency vs availability:** accept eventual convergence for badge counts and digests to keep the pipeline highly available.
- **Server intelligence vs privacy:** keep raw content out of the pipeline; use aggregates and coarse features.
- **Complexity vs iteration speed:** contract-first APIs add upfront work, but enable parallel development and safer rollouts.

### What we intentionally *don’t* do (yet)
- **No “exactly-once” end-to-end guarantees.** We prefer simpler at-least-once + dedupe because push providers and clients are inherently unreliable.
- **No per-recipient heavy online feature joins** (e.g., large multi-table queries) on the critical path. Those belong in precomputed indexes/feature stores.
- **No fully personalized text generation in notifications** (beyond previews). Summarization can be added later, but raises cost and privacy concerns.
- **No strict global ordering of notifications.** Users care about relevance and thread/DM coherence, not total FIFO across the workspace.
- **No cross-workspace learning by default.** Any model sharing should be opt-in and privacy-reviewed.

---

## Related docs
- Teardowns: https://github.com/004mayank/product-teardowns
- PRDs: https://github.com/004mayank/product-prd
