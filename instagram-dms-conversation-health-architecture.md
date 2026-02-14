# Instagram DMs — Conversation Health Assist (Needs Reply / Requests Triage / Open Loops)

**Artifact:** System Architecture (V1)  
**Product:** Instagram (DMs)  
**Scope:** Architecture to support the PRD features shipped in PRD v3 (lightweight, policy-safe, no “deep semantic” intent)  
**Author:** Mayank Malviya (with Botty)  
**Status:** v1 — end-to-end design + key components + trade-offs  

---

## 0) What are we building (in one paragraph)
We’re adding three lightweight “conversation health” assists to Instagram DMs:
1) a **Needs Reply** indicator on eligible threads in the DM list,
2) a **Requests triage** UX in Message Requests (signals + one-tap actions + likely-spam bucket), and
3) an **Open Loops** entry module that resurfaces eligible items when the user returns after time away.

The architecture must be **safe** (avoid spam amplification), **low-latency**, **consistent across devices**, and **measurable** (experiments + event logging).

---

## 1) Design principles / constraints
- **No new “deep ML” claims**: start with explicit context + simple heuristics; optionally evolve to model-driven later.
- **Do not reorder the inbox** in V1. Add indicators/modules only.
- **Consistency across devices**: if a user marks “handled” on one device, it should generally reflect on others.
- **Privacy/data minimization**: store only state needed for the feature (e.g., handled flag + timestamps). Avoid storing derived sensitive intent.
- **Latency**: DM list must remain fast. Compute or fetch labels without blocking first paint.
- **Safety**: integrate existing spam/restriction signals; conservative eligibility.

---

## 2) High-level architecture

### Components (logical)
1) **Messaging Backend** (existing)
   - Thread list/inbox service
   - Message store
   - Requests inbox
   - Trust/safety classifiers (existing spam signals)

2) **Conversation Health Service (new or logical layer)**
   - Computes **Needs Reply eligibility + score** per thread
   - Produces **Open Loops** module payload
   - Stores per-user/per-thread “handled” state
   - Enforces caps/frequency rules

3) **Feature Flag / Experimentation Platform** (existing)
   - Gates features (`dm_needs_reply_enabled`, `dm_open_loops_enabled`, etc.)
   - Assigns variants (control vs treatment)

4) **Client Apps (iOS/Android)**
   - Renders DM list indicator
   - Renders Requests triage surfaces
   - Renders Open Loops module
   - Sends interaction events

5) **Analytics/Event Pipeline** (existing)
   - Receives events and enables dashboards/guardrails

---

## 3) Data model (minimal)

### 3.1 Per-thread computed attributes (derived)
These are computed on request and generally **not stored long-term**:
- `eligible_for_needs_reply` (bool)
- `needs_reply_score` (int)
- `needs_reply_reason` (enum): `story_reply | question | verified | graph | other`

Inputs (read-only):
- `last_inbound_ts`
- `last_outbound_ts`
- `is_story_reply_context`
- `is_request_thread`
- `is_muted | is_archived | is_restricted | is_blocked`
- `trust_signals`: verified, mutuals, etc.
- `spam_classification_bucket` (existing)

### 3.2 Stored state (new)
**UserThreadAssistState** (per user_id, thread_id):
- `handled_until_ts` (nullable) — set when user taps “Mark handled”
- `handled_reason` (optional enum) — e.g., `user_marked_handled`
- `updated_at`

**UserAssistSessionState** (per user_id, day):
- `open_loops_last_shown_ts`
- `open_loops_last_dismiss_ts`

Storage requirements:
- Small, key-value friendly, low write volume.
- TTL possible (e.g., handled state expires after 14 days).

---

## 4) Core flows

### 4.1 DM list load → Needs Reply labels
**Goal:** Render DM list quickly and overlay “Needs Reply” where applicable.

**Option A (recommended): server-augmented thread list**
1) Client requests thread list: `GET /dm/inbox?cursor=...`
2) Inbox service calls Conversation Health Service (or internal module) with thread summaries.
3) Service returns: for each thread: `{needs_reply: bool, reason, score_bucket(optional)}`.
4) Client renders list + pill if `needs_reply=true`.

Pros: consistent cross-device, centralized cap logic, easy experiments.
Cons: extra backend hop; must be cached and cheap.

**Option B: client-side heuristics**
- Client computes “Needs Reply” using local thread metadata.

Pros: no backend changes, fast.
Cons: inconsistent across devices, harder to evolve and audit, riskier for safety.

**V1 choice:** Option A.

Latency strategy:
- Compute labels on **thread summaries only** (no message body fetch).
- Use a cheap scoring function; cache results for short periods (e.g., 5 minutes).
- If assist service times out, fail open to “no label” (safe).

### 4.2 “Mark handled” action
1) User taps overflow action on a thread: “Mark handled”.
2) Client calls: `POST /dm/assist/threads/{thread_id}/handled`.
3) Service stores `handled_until_ts = now + 14 days`.
4) Client immediately hides the label (optimistic UI).

Idempotency:
- repeated calls safe.

### 4.3 Requests triage
Requests UX is primarily client rendering + existing backend actions.

1) Client loads Requests: `GET /dm/requests`.
2) Server returns request items with trust signals and spam bucket classification.
3) Client shows:
   - signals row (verified/mutuals/etc.)
   - Accept/Delete/Report actions
   - separate “Likely spam” tab

Actions:
- `POST /dm/requests/{id}/accept`
- `DELETE /dm/requests/{id}`
- `POST /dm/requests/{id}/report`

Analytics logging should include `signals_shown[]`.

### 4.4 DM entry → Open Loops module
1) Client enters DMs surface (or pulls to refresh).
2) Client calls: `GET /dm/assist/open_loops`.
3) Service:
   - checks eligibility: last session > 6h, open loops count threshold, frequency cap (24h)
   - selects top N items:
     - Needs Reply threads (eligible + not handled)
     - pending Requests (not acted)
     - explicit mentions in group threads
   - returns ordered list (simple ordering rules, not ranking)
4) Client renders module; logs impression.

Dismiss:
- `POST /dm/assist/open_loops/dismiss`

---

## 5) Eligibility + scoring (server-side)

### 5.1 Hard exclusion gates
Exclude thread from Needs Reply if any:
- muted / archived
- restricted / safety-sensitive states
- blocked relationship
- last inbound older than 7 days
- thread is in Requests (handled separately)
- spam bucket == high confidence spam

### 5.2 Scoring (lightweight)
- +3 story reply context
- +2 in Primary inbox
- +2 contains `?` (requires scanning last inbound text)
- +2 sender verified
- +1 mutual follow / graph signal
- +1 inbound age ≤ 24h

**Threshold:** score ≥ 4.

**Note on message text access:**
- If policy/performance disallows sending message text to assist service, the inbox service can compute the presence of `?` as a boolean and pass only that.

---

## 6) Capping / saturation control
Enforce caps server-side to prevent overwhelming users.

- Needs Reply: cap to `min(5, 10% of first-screen threads)`.
- Open Loops: show max 3–5 items.
- Frequency: Open Loops at most once per 24h.

Implementation:
- Cap applied per “inbox page” response.
- Prefer to label the top eligible set using deterministic ordering (e.g., most recent inbound first) to avoid flapping.

---

## 7) Experimentation & flags
Flags:
- `dm_needs_reply_enabled`
- `dm_requests_triage_enabled`
- `dm_open_loops_enabled`

Variants:
- Control: no labels/modules
- Treatment: labels + handled + caps

Assignment:
- user-level randomization
- cohort gating: creators/SMBs first

---

## 8) Observability & analytics
Minimum events (from PRD v3):
- `dm_needs_reply_impression(thread_id, reason)`
- `dm_needs_reply_mark_handled(thread_id)`
- `dm_open_loops_impression(count_threads, count_requests)`
- `dm_open_loops_dismiss`
- `dm_request_action(request_id, action, signals_shown[])`

Dashboards:
- TTFR (median/p90)
- CCR (24h)
- Reports/blocks rate
- Urgency saturation distribution
- Time-to-first-action in Requests

---

## 9) Failure modes & fallbacks
- Assist service timeout → return inbox without labels/modules.
- State store unavailable → disable “handled persistence” (labels may reappear; acceptable short-term) and log error.
- Misclassification risk (spam surfaced) → rely on spam bucket exclusions + guardrail monitoring.

---

## 10) Security / privacy notes
- Avoid storing message content.
- If using `?` heuristic, store only boolean derived feature.
- Handled state is user action-derived and low sensitivity.
- Ensure teen/safety-sensitive states are excluded at source.

---

## 11) Implementation plan (V1)
1) Implement Conversation Health Service endpoints:
   - `POST /dm/assist/threads/{thread_id}/handled`
   - `GET /dm/assist/open_loops`
   - `POST /dm/assist/open_loops/dismiss`
2) Integrate label computation into inbox response (or sidecar endpoint).
3) Client UI:
   - Needs Reply pill + Mark handled
   - Open Loops module + dismiss
   - Requests triage UI (signals + actions + spam bucket tab)
4) Instrumentation and dashboards.

---

## 12) Open questions for Architecture V2
- Where should “handled” state live for best consistency (messaging store vs separate KV)?
- Can inbox service compute `contains_question_mark` safely without shipping text elsewhere?
- How to unify identity/trust signals across Primary + Requests surfaces cleanly?
