<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/slack.png" 
    alt="Slack logo" 
    width="200"
  />
</p>

# Slack Messaging — Notification Load Shaping (NYR) System Architecture (V1)

**Product:** Slack (messaging + notifications)
**Audience:** Product Managers / Engineers
**Goal:** Describe a plausible, implementation-oriented system architecture for **notification load shaping** and **Not-Yet-Relevant (NYR)**-style ranking, grounded in **publicly observable Slack behavior** and **generic system design patterns** (no proprietary internals).

**Scope in this doc (V1):**
- Message → notification candidate generation
- NYR labeling/scoring → ranking → delivery (cross-device)
- User feedback loops (Not relevant / Snooze / Mute) → preference updates
- Digest / catch-up generation
- Reliability, privacy/safety, and observability for the above

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

## 2) High-level architecture (conceptual)

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

## 3) Event flows

### Flow 1 — Message create → notification candidate generation

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

### Flow 2 — NYR labeling/scoring → ranking → delivery on devices

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

### Flow 3 — User feedback (not relevant/snooze) → preference updates → model/heuristic adjustments

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

### Flow 4 — Digest / catch-up generation

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

## 4) Data model (key entities + IDs)

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

## 5) Consistency + cross-device considerations

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

---

## 6) Failure handling + reliability

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

### Rate limiting & load shedding
- Per-user and per-workspace budgets:
  - max notifications/minute
  - max concurrent deliveries
- Degradation modes:
  - switch from per-message pushes to digest/bundles
  - disable low-importance candidate types temporarily

### Ordering
- Notification ordering should be **best-effort** and relevance-based, not strict FIFO.
- For threads/DMs, preserve intuitive order in the client feed using message timestamps.

---

## 7) Privacy & safety considerations

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

## 8) Observability (logs/metrics/traces) + guardrails

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

## 9) Key trade-offs (explicit)

- **Immediate vs relevant:** NYR enables deferring non-urgent interruptions.
- **Server intelligence vs privacy:** keep payload minimal; favor metadata and aggregates.
- **Consistency vs availability:** accept eventual convergence for counts/digests.
- **Personalization vs simplicity:** start rule-based + thresholds; add models behind flags.

---

## Related docs
- Teardowns: https://github.com/004mayank/product-teardowns
- PRDs: https://github.com/004mayank/product-prd
