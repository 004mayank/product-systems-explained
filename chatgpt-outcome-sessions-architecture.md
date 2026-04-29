# ChatGPT - Outcome-First Guided Sessions: System Architecture

**Version:** v1 - Initial system design
**Changes from v0:** First version - establishes core component topology, data model, and primary workflows for the Outcome-First Guided Sessions feature.

| Version | Key additions |
|---|---|
| v1 | Core component topology, data model, primary API contracts, key workflows, trade-offs, open questions |

**Source PRD:** chatgpt-prd.md (v3 - Outcome-First Guided Sessions)
**Date:** 2026-04-29
**Author:** Staff Engineering + Product

---

## 1. Product Context

The PRD targets a specific funnel break: users treat ChatGPT as single-turn search and drop off before the second turn. The architecture must support five product pillars:

1. **Outcome templates** - persona-aware starters that pre-fill intent, constraints, and format
2. **Iteration coaching** - contextual follow-up suggestions after each LLM response
3. **Trust and confidence UX** - surfacing model assumptions and unresolved needs inline
4. **Session context sidebar** - lightweight persistence of goal, constraints, sources, decisions
5. **Shippable handoff** - copy/export artifacts in multiple formats

The system lives entirely within the ChatGPT web + mobile surfaces. It does not replace the core LLM inference path - it wraps and enriches it.

---

## 2. Goals and Non-Functional Requirements

### Goals

| Goal | Target | Measurement |
|---|---|---|
| Increase weekly successful outcomes per active user | +25% | Export + save events per WAU |
| Activation rate (users who reach second turn) | +15pp | Turn 2 rate on day 0 cohort |
| Second-turn rate across all sessions | +20pp | Turns per session P50 |
| Template adoption among new users | >40% of day-0 sessions | Template selection event |

### Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| Template load latency (P95) | <200ms | Served from edge cache, personalized ranking in background |
| Iteration suggestion latency (P95) | <1.5s after response streams complete | Non-blocking, appended after main response |
| Session context write latency (P99) | <100ms | Fire-and-forget from client, async persistence |
| Export generation latency (P95) | <3s | Markdown, DOCX, plain text; PDF async |
| System availability | 99.9% | Degrade gracefully - all pillars are additive, not blocking |
| Session context storage per user | <500KB active sessions | Pruned after 30 days of inactivity |
| Trust label extraction accuracy | >85% assumption recall | Evaluated on internal annotation set |

### Graceful Degradation Contract

Every pillar is additive. If the template service is unavailable, users see a blank prompt box (current baseline). If iteration coaching fails, the response renders without suggestions. Session context silently stops persisting but does not break the chat. The core LLM response path must never be gated on any of these services.

---

## 3. System Components

### 3.1 Component Overview

The feature introduces five new or extended services on top of the existing ChatGPT inference stack:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client (Web / Mobile)                       │
│   Template Picker  │  Session Sidebar  │  Artifact Export UI    │
└─────────┬──────────┴────────┬──────────┴───────────┬────────────┘
          │                   │                       │
          ▼                   ▼                       ▼
┌─────────────────┐  ┌────────────────┐  ┌───────────────────────┐
│ Template Service│  │ Session Context│  │  Artifact + Export    │
│  (ranked list,  │  │    Store       │  │     Service           │
│  slot schema)   │  │ (goal/constr/  │  │ (format, deliver,     │
│                 │  │  src/decisions)│  │  track)               │
└────────┬────────┘  └───────┬────────┘  └──────────┬────────────┘
         │                   │                       │
         └───────────────────┼───────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Conversation   │
                    │  Orchestrator   │  (existing, extended)
                    │  (prompt build, │
                    │   context inject│
                    │   turn routing) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────────┐
       │ LLM        │ │  Trust     │ │  Iteration     │
       │ Inference  │ │  Extractor │ │  Coach         │
       │ (existing) │ │  (post-proc│ │  (post-proc    │
       │            │ │  pipeline) │ │  suggestions)  │
       └────────────┘ └────────────┘ └────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Observability  │
                    │  (events, logs, │
                    │   dashboards)   │
                    └─────────────────┘
```

### 3.2 Template Service

**Responsibility:** Serve ranked outcome templates to the client on session start or when the user clears the prompt.

**Key behaviors:**
- Returns a ranked list of up to 6 templates based on: user persona (inferred from history), time of day, entry point (direct URL, referred from /explore, mobile vs. web), recent session patterns
- Each template includes a `slot_schema` - the required and optional fill-in fields before the prompt is submitted
- Slot validation is client-side only; the service does not block on slot completion
- Template ranking is updated offline (daily batch job, not real-time) to avoid latency spikes

**Personalization tiers:**
- Tier 0 (new user): Rule-based defaults by entry point (e.g., /code entry -> developer templates)
- Tier 1 (>5 sessions): Frequency-based ranking on past template categories
- Tier 2 (>30 sessions): Collaborative filtering against similar-behavior cohort

**Template catalog** is maintained in a CMS-backed config (not a ML model), reviewed by the content team. New templates are A/B testable via feature flag on `template_id`.

### 3.3 Session Context Store

**Responsibility:** Persist and serve the lightweight session context sidebar state: goal, constraints, sources, and decisions.

**Key behaviors:**
- Client writes context updates on user edits (debounced 500ms)
- Context is injected into the system prompt at each turn by the Conversation Orchestrator
- Sessions are soft-deleted after 30 days of inactivity; users can manually pin sessions
- Context is scoped to a single conversation ID; cross-session summarization is a v3 feature

**Context injection format (system prompt append):**
```
[Session Context]
Goal: {goal}
Constraints: {constraints}
Sources: {sources}
Decisions so far: {decisions}
```

This block is injected as a second system prompt segment, after the base persona instructions. Token budget: max 400 tokens. If the user-written context exceeds 400 tokens, the orchestrator truncates by priority: goal > constraints > decisions > sources.

### 3.4 Trust Extractor

**Responsibility:** Post-process each LLM response to identify and surface assumptions the model is making and unresolved needs.

**Key behaviors:**
- Runs as a lightweight secondary LLM call (fast, small model - e.g., GPT-4o-mini) immediately after the primary response streams complete
- Outputs a structured list: `assumptions[]` (things the model assumed true) and `needs[]` (information the model would benefit from)
- Client renders these as a collapsible "Assumptions & Needs" section below the response
- Trust extraction is skipped if: response is <50 tokens, turn is a clarifying question (model already asked), or `trust_labels_enabled` feature flag is off

**Trade-off:** Running a secondary LLM call adds ~200-400ms and ~$0.002 per response. This is acceptable at v1 given the conversion lift target. If cost scales poorly, v2 will explore in-context extraction (no second call) via structured output from the primary model.

**Edge case - hallucinated assumptions:** If the trust extractor itself hallucinates assumptions, users may lose trust in the feature. Mitigation: cap at 3 assumptions max, use a conservative prompt that prefers precision over recall, add user feedback signal ("Was this helpful?") to improve extraction quality over time.

### 3.5 Iteration Coach

**Responsibility:** Generate 2-3 contextual follow-up suggestions after each LLM response to guide users toward the next productive turn.

**Key behaviors:**
- Runs in parallel with the Trust Extractor, after the primary response completes streaming
- Uses a small model with a prompt that takes: the session goal (from context store), the last user message, and the last 2 assistant turns
- Returns an ordered list of `suggestion_text` strings, rendered as clickable chips below the response
- Clicking a chip pre-fills the prompt box (does not auto-submit); user can edit before sending
- Suggestions are refreshed on each new response. Previous suggestions are discarded.

**Suggestion quality heuristics (enforced in prompt):**
- Each suggestion must advance the session goal, not just continue conversation
- Suggestions must be differentiated (not variations of the same ask)
- At least one suggestion should push toward closure (e.g., "Make this ready to send")

**Edge case - open-ended creative sessions:** For creative writing or brainstorming, the "advance toward goal" heuristic produces poor suggestions. Detection: if no goal is set in session context and the template category is `creative`, switch to a different prompt variant that focuses on exploration rather than closure.

### 3.6 Artifact and Export Service

**Responsibility:** Track generated artifacts within a session and handle formatting + delivery for export actions.

**Key behaviors:**
- An artifact is any contiguous assistant response that the user has explicitly marked (via copy, save, or export action) or that matches an auto-detect heuristic (response >300 tokens, contains a structured document pattern)
- Export formats at v1: Markdown (instant), plain text (instant), DOCX (async, <5s), clipboard HTML (instant)
- PDF export is a v2 feature (requires headless rendering pipeline)
- Artifacts are linked to the conversation ID and session context snapshot at the time of export
- Artifacts are stored for 90 days; full conversation history retention follows existing ChatGPT policy

---

## 4. Data Model

### Session

```json
{
  "session_id": "uuid",
  "conversation_id": "uuid",        // maps to existing ChatGPT conversation
  "user_id": "uuid",
  "template_id": "string | null",   // null if no template used
  "template_slots": {               // key-value of filled slot names → values
    "role": "PM",
    "context": "Q2 planning brief"
  },
  "context": {
    "goal": "string | null",
    "constraints": "string | null",
    "sources": ["string"],
    "decisions": ["string"]
  },
  "artifacts": ["artifact_id"],
  "created_at": "ISO8601",
  "updated_at": "ISO8601",
  "last_active_at": "ISO8601",
  "pinned": false
}
```

### OutcomeTemplate

```json
{
  "template_id": "string",          // e.g., "pm-brief-v2"
  "slug": "string",
  "category": "pm | developer | writer | student | ops | creative",
  "title": "string",
  "description": "string",
  "prompt_pattern": "string",       // Jinja-style template with {{slot_name}} vars
  "slots": [
    {
      "name": "string",
      "label": "string",
      "required": true,
      "default": "string | null",
      "input_type": "text | dropdown | multiline"
    }
  ],
  "tags": ["string"],
  "active": true,
  "ab_test_cohort": "string | null"
}
```

### Artifact

```json
{
  "artifact_id": "uuid",
  "session_id": "uuid",
  "conversation_id": "uuid",
  "turn_index": 3,                  // which assistant turn produced this
  "content_snapshot": "string",     // raw assistant response at time of save
  "context_snapshot": {             // session context at time of save
    "goal": "string | null",
    "constraints": "string | null"
  },
  "export_events": [
    {
      "format": "markdown | plaintext | docx | clipboard",
      "exported_at": "ISO8601"
    }
  ],
  "auto_detected": false,
  "created_at": "ISO8601",
  "expires_at": "ISO8601"           // 90 days from created_at
}
```

### TrustExtractionResult

```json
{
  "extraction_id": "uuid",
  "session_id": "uuid",
  "turn_index": 3,
  "assumptions": [
    {
      "text": "string",
      "confidence": "high | medium | low"
    }
  ],
  "needs": [
    {
      "text": "string"
    }
  ],
  "model_used": "gpt-4o-mini",
  "latency_ms": 312,
  "skipped": false,
  "skip_reason": "string | null"
}
```

### IterationSuggestion

```json
{
  "suggestion_set_id": "uuid",
  "session_id": "uuid",
  "turn_index": 3,
  "suggestions": [
    {
      "text": "string",
      "type": "deepen | handoff | explore | clarify"
    }
  ],
  "model_used": "gpt-4o-mini",
  "latency_ms": 280
}
```

---

## 5. API Contracts (v1)

### Template Service

**GET /v1/sessions/templates**

Returns ranked templates for the session start surface.

Request headers:
- `X-User-Id`: user UUID
- `X-Entry-Point`: `direct | explore | share | mobile`

Response:
```json
{
  "templates": [
    {
      "template_id": "pm-brief-v2",
      "title": "Write a PM brief",
      "description": "Draft a concise PM brief with goals, scope, and open questions.",
      "category": "pm",
      "slots": [...],
      "rank": 1
    }
  ],
  "personalization_tier": 1,
  "ab_cohort": "template_picker_v2"
}
```

### Session Context

**PUT /v1/sessions/{session_id}/context**

Upserts the session context sidebar state. Fire-and-forget from client.

Request body:
```json
{
  "goal": "string | null",
  "constraints": "string | null",
  "sources": ["string"],
  "decisions": ["string"]
}
```

Response: `204 No Content`

**GET /v1/sessions/{session_id}/context**

Returns the current context state. Called on session resume.

Response:
```json
{
  "goal": "string | null",
  "constraints": "string | null",
  "sources": ["string"],
  "decisions": ["string"],
  "updated_at": "ISO8601"
}
```

### Artifact Export

**POST /v1/sessions/{session_id}/artifacts/{artifact_id}/export**

Triggers export of an artifact in the requested format.

Request body:
```json
{
  "format": "markdown | plaintext | docx | clipboard"
}
```

Response (sync formats - markdown, plaintext, clipboard):
```json
{
  "export_id": "uuid",
  "format": "markdown",
  "content": "string",
  "download_url": null
}
```

Response (async format - docx):
```json
{
  "export_id": "uuid",
  "format": "docx",
  "status": "processing",
  "poll_url": "/v1/exports/{export_id}/status"
}
```

---

## 6. Key Workflows

### 6.1 Session Start with Template

```
1. Client loads /chat → GET /v1/sessions/templates (cached at edge, <200ms)
2. User selects template → client renders slot form
3. User fills slots → client constructs prompt from prompt_pattern + slot values
4. Client creates session: POST /v1/sessions with template_id + slot fills
5. Prompt submitted → Conversation Orchestrator injects context block (if any) + sends to LLM
6. LLM streams response → client renders
7. Post-stream: Trust Extractor and Iteration Coach run in parallel
8. Trust labels + suggestions appended below response
```

### 6.2 Mid-Session Context Update

```
1. User opens sidebar → clicks "Add constraint"
2. Client writes to context store: PUT /v1/sessions/{id}/context (debounced 500ms)
3. Server persists; returns 204
4. On next user turn, Conversation Orchestrator fetches updated context: GET /v1/sessions/{id}/context
5. Context block injected into system prompt for this turn
```

### 6.3 Artifact Export (DOCX)

```
1. User clicks "Export as Word"
2. Client calls POST /v1/sessions/{id}/artifacts/{artifact_id}/export {format: "docx"}
3. Server enqueues export job; returns poll_url
4. Client polls /v1/exports/{export_id}/status every 1s (max 10 attempts)
5. On completion, server returns signed download URL (TTL: 1 hour)
6. Client triggers browser download
7. Export event logged: artifact_exported {format: "docx", latency_ms: ...}
```

---

## 7. Event Schemas

```json
// Template selected
{
  "event": "template_selected",
  "session_id": "uuid",
  "template_id": "pm-brief-v2",
  "entry_point": "direct",
  "slots_filled": 2,
  "slots_required": 2,
  "ts": "ISO8601"
}

// Trust label viewed
{
  "event": "trust_label_viewed",
  "session_id": "uuid",
  "turn_index": 3,
  "assumption_count": 2,
  "needs_count": 1,
  "expanded": true,
  "ts": "ISO8601"
}

// Iteration suggestion clicked
{
  "event": "iteration_suggestion_clicked",
  "session_id": "uuid",
  "turn_index": 3,
  "suggestion_rank": 1,
  "suggestion_type": "handoff",
  "ts": "ISO8601"
}

// Artifact exported
{
  "event": "artifact_exported",
  "session_id": "uuid",
  "artifact_id": "uuid",
  "format": "docx",
  "turn_index": 5,
  "latency_ms": 2100,
  "ts": "ISO8601"
}

// Session context updated
{
  "event": "session_context_updated",
  "session_id": "uuid",
  "fields_updated": ["goal", "constraints"],
  "turn_index": 2,
  "ts": "ISO8601"
}
```

---

## 8. Competitive Positioning

| Capability | ChatGPT (this design) | Claude | Gemini | Copilot |
|---|---|---|---|---|
| Outcome templates at session start | Yes - ranked, slot-driven | No - blank box | No | Partial - fixed task types |
| Mid-session context persistence | Yes - sidebar, injected each turn | Projects (separate, not inline) | No | No |
| Trust/assumption surfacing | Yes - post-process extraction | Minimal hedging in response | No | No |
| Iteration coaching | Yes - 2-3 typed suggestions | No | No | Partial - in-line Copilot suggestions |
| Multi-format export | Yes - MD, TXT, DOCX | MD only (copy) | No | Word integration only |

**Win condition vs. Claude:** Claude has better baseline instruction-following but no guided session scaffolding. Users who want to go from "I need a brief" to "done artifact" in one session have no structural support in Claude. This design fills that gap.

**Loss condition vs. Copilot:** For users already inside Microsoft 365, Copilot's deep Word/Excel/Outlook integration is a stronger handoff story. This architecture does not attempt to match native document editing - the export is one-way, not a live connection.

---

## 9. Trade-offs

| Decision | Chosen approach | Alternative considered | Reason |
|---|---|---|---|
| Trust extraction via second LLM call | Yes - small model post-process | In-context via structured output | Second call is more reliable and model-agnostic; revisit in v2 for cost |
| Session context as system prompt injection | Append to system prompt | Store in conversation messages | System prompt injection is invisible to user; doesn't pollute conversation history |
| Template ranking offline (daily batch) | Yes - batch job | Real-time personalization | Latency budget is <200ms at edge; real-time ranking not feasible without dedicated ML serving |
| Artifact auto-detection | Heuristic (>300 tokens + doc pattern) | User-explicit only | Auto-detection increases export discoverability; false positives are low-cost (user ignores the indicator) |
| DOCX export async, not streaming | Async polling | Streaming download | DOCX generation requires server-side rendering; streaming adds complexity without meaningful UX gain |

---

## 10. Failure Modes

| Failure | Detection | Degraded behavior |
|---|---|---|
| Template service down | Health check + error rate | Show blank prompt box; log `template_service_unavailable` |
| Trust extractor timeout (>2s) | P95 latency alert | Skip trust labels for that turn; no user-visible error |
| Iteration coach timeout (>2s) | P95 latency alert | Skip suggestions; no user-visible error |
| Session context store write failure | 5xx rate on PUT /context | Client retries once; if failed, context not persisted but session continues |
| DOCX export job fails | Job queue dead-letter | Return error in poll response; offer plain text fallback |
| Context injection exceeds token budget | Token count check in orchestrator | Truncate by priority: goal > constraints > decisions > sources |

---

## 11. Open Questions

1. **Template catalog ownership:** Who owns template content quality - product, content design, or a community submission model? What is the release cadence?

2. **Session context cross-device sync:** If a user starts a session on mobile and continues on web, does the context sidebar sync? v1 assumes yes (server-side), but latency and conflict resolution are unspecified.

3. **Template vs. Custom Instructions interaction:** ChatGPT already has Custom Instructions that prepend context to every conversation. How do template-generated system prompt additions stack with Custom Instructions? Risk of context collision or token budget overflow.

4. **Trust extractor model selection:** GPT-4o-mini is assumed for cost efficiency. What is the accuracy vs. cost curve? Is there an in-house model candidate that reduces OpenAI API spend?

5. **Artifact expiry policy:** 90-day expiry is assumed. Should pinned artifacts be exempt? What is the user communication model for expiry?

6. **Iteration suggestion abuse vector:** Could suggestions be used to probe for jailbreak-adjacent prompts if the coach is not constrained by safety filters? Needs safety review before launch.

7. **Mobile UX for session sidebar:** The sidebar is designed for web. Mobile surface needs a different interaction pattern (bottom sheet? Separate tab?). Not specified in v1 - deferred to mobile PM/design.
