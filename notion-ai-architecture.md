# Notion AI - Contextual Workspace Intelligence (System Architecture v1)

**What this explains:** the system architecture for grounding Notion AI generation in the user's own workspace content - turning a generic LLM wrapper into a workspace-aware AI that reads your pages before it writes.

**PRD reference:** https://github.com/004mayank/product-prd/blob/main/notion-ai-prd.md

**Version:** v1 - Initial system design
**Changes from v0:** First version. Covers core architecture: context assembly layer, RAG pipeline, generation layer, index maintenance pipeline, surfaces, data model, and key trade-offs.

---

## 1) What this system is

Notion AI in its current state is a **generic LLM invocation** triggered from inside a workspace. It produces grammatically correct text with no knowledge of the workspace it lives in. Users edit the output heavily or discard it - the feedback loop that would build AI habit never forms.

This architecture specifies **Contextual Workspace Intelligence**: every AI generation request is preceded by a context assembly step that injects the current page and retrieves semantically relevant workspace pages. The LLM generates grounded output the user actually keeps.

The system has three pillars (from the PRD):
1. **Page Context Injection** - inject current page content, DB properties, and @mentioned pages.
2. **Workspace Context Retrieval (RAG)** - retrieve top-N relevant pages from the workspace vector index.
3. **AI Page Starter** - surface a contextual structure suggestion on blank page open.

The hard constraint running through all three pillars: **permission model is never relaxed**. A user can only receive AI-generated content grounded in pages they have read access to.

---

## 2) Why the architecture is non-trivial

A naive implementation is: inject the full page into the LLM prompt and call the API. This fails because:

- Pages can be very large (1,000+ blocks). Token budget is finite. Truncation strategy matters.
- Workspace retrieval requires a per-workspace, permission-scoped vector index. This is not a general-purpose search index.
- Context injection must be fast (<800ms P95 added latency). RAG adds a round trip to the vector store.
- The system must degrade gracefully: if RAG times out, generation must still proceed on page-only context.
- Index freshness matters. If the AI retrieves a stale version of a page, it generates incorrect facts.
- Multiple workspaces at different scales (5-person startup to 5,000-person enterprise). Index size and retrieval time vary by 3-4 orders of magnitude.

The architecture is shaped by these constraints more than by the feature set.

---

## 3) High-level architecture

Four layers:

1. **Context Assembly Layer**: given a generation request, builds the context bundle (page content + DB properties + linked pages + RAG results).
2. **Vector Index Layer**: stores per-workspace embeddings; supports permission-scoped similarity search; updated incrementally as pages change.
3. **Generation Layer**: sends assembled context + user prompt to the LLM; streams output back to the editor.
4. **Index Maintenance Pipeline**: background jobs that keep the vector index fresh and consistent.

A fifth component - **AI Surfaces** in the editor - is the UX layer that triggers requests and renders output. It is largely frontend and is described in terms of what it sends and receives.

---

## 4) Core data model

### Page (existing Notion object, relevant fields)

```
page_id         uuid
workspace_id    uuid
created_by      user_id
permissions     ACL[]          // who can read/write
block_count     int
token_estimate  int            // approximate; updated on edit
updated_at      timestamp
```

### VectorIndexEntry

```
entry_id             uuid
page_id              uuid
workspace_id         uuid
embedding_vector     float[1536]    // text-embedding-3-small dimension
content_hash         sha256         // detect change without re-reading full content
token_count          int
is_archived          bool
is_deleted           bool
last_indexed_at      timestamp
page_updated_at      timestamp      // from page metadata; detect staleness
index_version        int            // schema version for migrations
```

### AIGenerationRequest

```
request_id           uuid
user_id              uuid
workspace_id         uuid
page_id              uuid
command              enum(write, summarise, draft, improve, translate, fix_spelling, page_starter)
prompt_text          string
context_bundle_id    uuid           // FK to assembled context snapshot (for audit/debug)
rag_enabled          bool
fallback_triggered   bool           // true if RAG was skipped due to timeout
model                enum(gpt-4o, claude-3-5-sonnet)
status               enum(pending, generating, completed, failed, fallback)
created_at           timestamp
completed_at         timestamp
latency_ms           int
```

### ContextBundle (assembled per request, short-lived)

```
bundle_id            uuid
request_id           uuid
current_page_tokens  int
db_properties_tokens int
linked_pages_tokens  int
rag_pages_tokens     int
total_tokens         int
rag_page_ids         uuid[]         // which pages were retrieved
linked_page_ids      uuid[]         // which @mention pages were resolved
fallback_reason      string?        // if any layer was skipped
created_at           timestamp
ttl_s                int            // default 300; context snapshots are ephemeral
```

### AIGenerationOutcome

```
outcome_id           uuid
request_id           uuid
user_id              uuid
workspace_id         uuid
page_id              uuid
outcome              enum(accepted, discarded, partially_accepted)
edit_delta_pct       float          // character-level edit after acceptance
sources_opened       bool
db_properties_filled bool
time_to_outcome_ms   int
ts                   timestamp
```

---

## 5) Context assembly layer

The context assembly layer runs on every generation request. It is a synchronous service with a hard latency budget of 800ms P95.

### Steps

**Step 1 - Fetch current page content**

Retrieve all text blocks for `page_id`. The page block content API is an existing internal API.

Token management:
- If `token_count <= 3,000`: inject full content.
- If `token_count > 3,000`: truncate from the bottom (preserve intro + most recent content); append `[page truncated]` marker.

**Step 2 - Fetch DB properties (if page is a DB entry)**

If the page is a row in a database, fetch all typed properties. Serialize as structured JSON. Inject up to 400 tokens.

Edge case: formula properties are injected as their computed value. Relation and rollup properties are injected as referenced titles only (not resolved recursively - would blow the token budget).

**Step 3 - Resolve @mention links**

Extract up to 3 `@mention` page links from the current page content. For each:
- Check read permission for the requesting user.
- If permitted: fetch page content and extractively summarise to 400 tokens.
- If not permitted: skip silently. Do not expose page existence.

**Step 4 - RAG retrieval (async, with timeout)**

If `rag_enabled = true` and command is in `[write, draft, summarise]`:
- Embed prompt + page title using `text-embedding-3-small`.
- Query the workspace vector index with permission filter.
- Return top-3 results by cosine similarity; exclude `page_id` itself.
- For each retrieved page: extractively summarise to 600 tokens.
- Hard timeout: 600ms. If exceeded, skip RAG; set `fallback_triggered = true`; emit `ai_rag_timeout` event.

**Step 5 - Assemble and validate token budget**

| Layer | Max tokens | Priority |
|---|---|---|
| System prompt | 800 | Fixed |
| Current page content | 3,000 | Highest |
| DB properties | 400 | High |
| @mention linked pages (3 x 400) | 1,200 | Medium |
| RAG pages (3 x 600) | 1,800 | Medium |
| User prompt | 500 | Fixed |
| Generation budget | 2,000 | Fixed |
| **Total** | **~9,700** | Within GPT-4o 128k window |

If total exceeds budget (rare with these caps): reduce RAG from 3 to 2 pages, then 1, then drop RAG entirely.

---

## 6) Vector index layer

### Architecture

The vector index is a per-workspace, permission-aware similarity search service. Implementation options:

| Option | Trade-off |
|---|---|
| Dedicated vector DB (Pinecone, Weaviate) per workspace | Strong isolation; high operational cost at scale |
| Shared vector DB with workspace-scoped namespaces | Lower cost; requires careful permission enforcement at query time |
| Postgres + pgvector (for smaller workspaces) | Simpler ops; query performance degrades at 100k+ embeddings |

**Recommendation for v1:** Shared vector store with workspace-namespaced collections. Permission enforcement is applied at query time via an ACL-filtered search: the query is run against only `entry_id` values where the requesting user has read access. This is an inner join between the vector store results and the user's permission set - expensive to do naively, and the primary scaling concern for v2.

### Index operations

**Upsert (on page create or significant edit)**

Trigger: page block content changes by >50 tokens.

Steps:
1. Read page content via block content API.
2. Compute `content_hash`. If unchanged: no-op.
3. Embed content using `text-embedding-3-small`.
4. Upsert `VectorIndexEntry` with new vector + updated timestamps.

Latency: up to 24h SLA for background indexing. Session-level cache refresh for pages edited in the current session (reduces staleness for the most active users).

**Delete / archive**

On page delete: mark `is_deleted = true`; exclude from all queries; hard delete after 30 days.
On page archive: mark `is_archived = true`; still indexed but flagged so retrieval can optionally exclude archived pages.

**Permission change propagation**

When a page's ACL changes (share permission revoked), the index entry is not re-embedded - the vector is unchanged. Permission is enforced at query time, not at index time. This means permission changes take effect immediately in retrieval (no re-index needed for privacy).

---

## 7) Generation layer

The generation layer takes the assembled context bundle and the user prompt, calls the LLM, and streams output back to the editor surface.

### Prompt structure

```
[System prompt]
You are Notion AI, an assistant that helps users with their Notion workspace.
You have been provided with context from the user's workspace pages below.
Only generate content grounded in the provided context. Do not invent facts about the workspace.
When referencing retrieved pages, mention them by name.

[Current page context]
Page title: {title}
Breadcrumb: {breadcrumb}
Content:
{page_content}

[Database properties] (if DB entry)
{db_properties_json}

[Linked pages] (from @mentions)
{linked_page_summaries}

[Retrieved workspace pages] (from RAG)
{rag_page_summaries}

[User request]
{user_prompt}
```

### Streaming

Generation output is streamed token-by-token to the editor. This is not a UX nicety - it is required because P95 latency for full generation can exceed 5s for longer outputs. Streaming allows the user to see output forming, which preserves the perception of responsiveness.

### Model selection

Current: GPT-4o as primary; Claude 3.5 Sonnet as fallback or optional override.

The model selection is not exposed to users in v1. It is a server-side config. This is intentional: model characteristics (context window, cost, latency) affect context budget decisions and should not be exposed as a user-facing knob.

---

## 8) Index maintenance pipeline

A background pipeline handles all index write operations. It is decoupled from the online serving path.

### Components

**Change detector**

Subscribes to the Notion page event stream (page created, page updated, page deleted/archived).

For updates: computes token count change. Ignores edits below 50-token threshold (minor formatting changes don't change semantic content enough to warrant re-embedding).

**Embedding worker**

Reads page content. Calls the embedding API. Upserts to vector store. Emits `page_indexed` event.

Failure handling: retry with exponential backoff (max 3 retries). If all retries fail: emit `index_failure` event; continue with stale index (serving degrades gracefully - stale retrieval is better than no retrieval).

**Index freshness monitor**

Background job that computes median lag between `page_updated_at` and `last_indexed_at` across the workspace.

Triggers:
- Median lag > 24h: emit `index_staleness_alert`; reindex job.
- Any page lag > 7 days: force re-embed regardless of change threshold.

**Deleted page cleaner**

Runs hourly. Finds entries where `is_deleted = true` and `last_indexed_at` > 30 days ago. Hard deletes from vector store.

---

## 9) AI surfaces

### 9.1 Inline editor (slash command + Ask AI button)

User triggers AI via:
- `/AI` slash command (creates an AI block at cursor)
- "Ask AI" button that appears on text selection
- Keyboard shortcut (TBD in v2)

On trigger:
1. Editor sends `ai_session_started` event.
2. Context assembly service called with `page_id` + `command` + `prompt_text`.
3. Generation streams to the AI block.
4. On stream complete: "Accept / Discard / Try again" controls appear.

On accept: content is inserted as blocks. `ai_generation_outcome` event emitted.
On discard: block removed. `ai_generation_outcome` with `outcome = discarded` emitted.

### 9.2 Sources disclosure

For every RAG-enabled response:
- Below generated output, render a collapsed "Sources" section.
- On expand: list of page titles + links to the retrieved pages.
- Collapsed by default to avoid visual noise; expandable for trust verification.
- `ai_sources_disclosure_opened` event emitted on expand.

### 9.3 AI page starter

Trigger: user creates a new page, enters a title, and page has 0 blocks after 1.5s.

Shown only to AI add-on subscribers. Shown at most once per page (persisted via `ai_starter_dismissed` property).

Steps:
1. Context assembly with lightweight variant: title + parent section type + 3 most recent sibling pages.
2. LLM generates 3-7 heading blocks (no body content).
3. Starter prompt rendered inline (where first block would appear).

Accept: headings inserted as real blocks. `ai_page_starter_accepted` event emitted.
Dismiss: starter disappears. `ai_page_starter_dismissed` event emitted. Never shown again on this page.
Auto-dismiss: if user types before 10s, starter dismissed silently. `dismiss_method = typing`.

### 9.4 DB property population

Trigger: AI generation completes on a page that is a DB entry with >2 empty properties.

A "Fill properties" button appears below the output. On click:
- LLM extracts candidate property values from the generated output.
- Maps to property types (text, date, select only for v1).
- Shows preview with confirm/reject per property.
- Multi-select and relation properties: suggestions only; never auto-populated.

---

## 10) User journeys

### Happy path - PM writes a spec using workspace context

1. PM opens a blank page titled "Q3 Search Redesign Spec" inside the Product section.
2. AI page starter appears (page is blank, user has AI add-on). Starter suggests headings: "Overview", "Problem", "Goals", "Solution", "Success Metrics", "Open Questions."
3. PM accepts. Headings inserted. PM starts typing under "Problem".
4. PM types `/AI Write` -> "Draft the Problem section based on our past specs and user research."
5. Context assembly:
   - Injects current page: title + existing headings + partial text.
   - RAG retrieves: "User Research - Search Q2", "Q2 Spec - Homepage Redesign", "Design Principles - Notion Search."
   - Injects summaries of each.
6. LLM generates a Problem section that references the retrieved context (specific findings from research, prior spec patterns).
7. PM accepts with <15% edit. `ai_generation_outcome.edit_delta_pct = 14`. Sources disclosure shows 3 referenced pages.

**Key outcome:** PM accepted a draft that referenced real workspace content in the first session. This is the aha moment.

---

### Edge case - RAG retrieval timeout

1. User triggers `/AI Summarise` on a large meeting notes page.
2. Context assembly starts. RAG call to vector store times out at 600ms.
3. System sets `fallback_triggered = true`. Emits `ai_rag_timeout` event.
4. Generation proceeds with page-only context (current page content, no retrieved pages).
5. Output is generated from meeting notes content alone. No sources disclosure shown (no RAG results).
6. User sees no error message. AI generates normally; output is lower quality but not broken.
7. `ai_context_fallback` event emitted for monitoring.

**What is sacrificed:** Generation quality degrades (workspace context missing). **What is preserved:** User sees generation complete without failure. Graceful degradation is the correct trade-off here - a timeout error would break the core user flow.

---

### Edge case - User requests AI on a page they only partially own

1. Workspace guest (limited access) opens a project page they can read.
2. Project page has `@mention` links to three internal strategy pages the guest cannot read.
3. User triggers `/AI Summarise`.
4. Context assembly:
   - Current page: injected fully (user can read it).
   - @mention links: permission check fails for all three. Skipped silently. No exposure of page titles or content.
   - RAG: query runs against only user-readable pages in the workspace. Returns results the guest can access.
5. Generation proceeds on page-only context + any permitted RAG results.
6. No mention of restricted pages in output. Sources disclosure only shows permitted pages.

**Key constraint:** Permission enforcement must happen at context assembly time, not at display time. The LLM must never receive content the user cannot access.

---

## 11) Trade-offs

| Decision | Optimised for | Sacrificed | Why |
|---|---|---|---|
| Truncate from the bottom (keep intro + recent) | Most relevant content injected for typical docs (intro sets context; recent blocks are freshest) | Very long early-document content may be cut | Most Notion pages are structured intro-first; bottom truncation is least harmful for typical structures |
| Hard RAG timeout at 600ms with silent fallback | User-perceived latency; no broken generations | RAG quality when vector store is slow | A failed generation is worse than a non-RAG generation. Silent fallback is the right default |
| Permission check at query time, not index time | Immediate effect when permissions change; no re-index on ACL changes | Query-time permission filtering is expensive at scale | Re-indexing on every permission change would make index jobs unbounded; query-time is more correct |
| Extractive summarisation of retrieved pages (not full injection) | Token budget preservation; faster generation | Nuance lost in summarisation; AI may miss the most relevant passage | 600 tokens of extractive summary captures most semantic value; full injection would hit budget limits with 3 sources |
| No user-facing model selection | Simpler UX; server controls cost/performance trade-offs | Power user flexibility | Model selection exposes LLM internals and creates expectation management problems; not a v1 concern |
| Page starter shows headings only (no body content) | Low risk of AI presuming too much; gives structure without overwriting intent | Less value per starter invocation | Body content generation on a blank page is high-variance; headings are universally appropriate |

---

## 12) What v2 will add

- Detailed event schemas and API contracts for all context assembly calls.
- Competitive analysis: how Coda AI, Confluence AI, and Google Workspace AI approach context.
- Acceptance criteria with edge cases for each requirement.
- Permission-scoped retrieval scaling analysis (the primary hard problem).
- Full metrics framework with target values and instrumentation spec.
- Session-level index cache design for high-frequency editors.
- Context budget experimentation plan (is 3,000 tokens for current page optimal?).
- Mermaid system diagram (required in v2).
