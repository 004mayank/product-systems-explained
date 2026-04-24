# Figma - Design-System-Aware AI Generation (System Architecture)

**What this explains:** The system architecture that powers library-grounded Make Designs generation - how Figma reads an org's published component library, injects it as structured context into the generation pipeline, and ensures output uses real component instances instead of hardcoded shapes.

**PRD reference:** https://github.com/004mayank/product-prd/blob/main/figma-design-system-aware-ai-prd.md

**Version:** v1 - Initial system design
**Changes from v0:** N/A - first version.

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core architecture (5 layers), data flow, library context payload schema, component matching logic, compliance computation, failure modes, trade-offs, user journeys |

---

## 1) What this system is

Figma's existing `Make Designs` pipeline generates frames from a text prompt using a general-purpose LLM with no knowledge of the org's component library. The output looks plausible but uses hardcoded hex values, raw shapes, and arbitrary font sizes - all of which violate the org's design system on inspection.

This architecture specifies the system that makes `Make Designs` **design-system-aware**: every generation session in an org with a published library receives a structured representation of that library as part of the generation context. The LLM maps each element in its output to a real library component before the frame is placed on the canvas.

The system has five layers:

1. **Library Index Service** - Maintains a structured, queryable representation of every org's published component library.
2. **Context Injection Service** - Fetches and trims the library context for a specific generation session.
3. **Design-Aware Generation Pipeline** - The LLM generation layer that consumes library context and outputs component-first frames.
4. **Compliance Computation Service** - Post-generation analysis that scores the output frame against the org library and populates the compliance panel.
5. **Library Gap Pipeline** - Event capture and notification system for component types the generation needed but could not find.

Hard constraint: **Library context is used only for in-session generation.** It is not retained after session end and is not used to train the generation model without explicit org admin opt-in.

---

## 2) Core problem the architecture must solve

The naive approach to library-grounded generation is to include the entire library as text in the LLM prompt. This fails for three reasons:

- A 300-component library with variant metadata, color variables, and text styles exceeds any practical LLM context budget (estimated 80-150k tokens for a mature enterprise library, before the prompt itself).
- Including the full library regardless of the prompt content means the LLM is equally likely to reach for any component, not the components relevant to the specific screen being generated.
- Library data is largely stable between publishes. Fetching it fresh on every generation request is expensive and slow.

The architecture solves this with three mechanisms:
1. **Relevance filtering:** For each generation session, embed the user prompt and retrieve the top N semantically relevant components from the library index, not the full library.
2. **Caching:** Library context is cached per org library version with a 10-minute TTL. Most sessions hit the cache; fresh fetches only occur after a library publish event.
3. **Two-pass generation:** First pass extracts a component type manifest from the prompt (e.g., "Button, Input, Card, Toggle"). Second pass fetches the exact library entries for those types, enabling precise variant selection rather than fuzzy similarity matching.

---

## 3) System layers

### Layer 1: Library Index Service

**Purpose:** Maintains a structured, up-to-date representation of every org's published component library. This is the source of truth that all other layers read from.

**What it stores per org library:**
- Component catalogue: name, category, variant set, variant properties (type, state, size), semantic label (inferred from component name and description), usage frequency.
- Color variable catalogue: name, collection, light mode value, dark mode value, semantic role.
- Text style catalogue: name, font family, size, weight, line height, semantic role.
- Component embeddings: a vector embedding per component (based on name + description + semantic label) used for relevance filtering.

**Triggers for re-indexing:**
- `library_published` event fires whenever an org library is published. The index worker processes the updated library file and rebuilds the affected org's catalogue within 5 minutes.
- Freshness monitor runs hourly. Any org whose index is older than 24 hours relative to the last `library_published` event triggers a forced re-index.

**Indexing worker logic:**
```
on library_published(org_id, library_file_id, library_version):
  fetch_library_file(library_file_id)
  parse_components(file) -> component_records[]
  parse_variables(file) -> variable_records[]
  parse_text_styles(file) -> text_style_records[]
  embed_components(component_records) -> embedding_vectors[]
  upsert_catalogue(org_id, library_version, records)
  emit index_updated(org_id, library_version, component_count, latency_ms)
```

**Storage:** Postgres for structured records (component catalogue, variables, text styles). `pgvector` extension for component embeddings. For orgs with >500 components, a dedicated vector index shard per org is used to avoid cross-org embedding interference.

---

### Layer 2: Context Injection Service

**Purpose:** Given a `(org_id, prompt)` pair, produces the structured `library_context` payload that the generation pipeline consumes.

**Two-pass context assembly:**

**Pass 1 - Component type manifest extraction:**
```
prompt = "Settings page for notification preferences with toggles and save button"

manifest = extract_component_types(prompt)
# -> ["Toggle", "Button", "Input", "Card", "Divider"]
```
The manifest is extracted by a lightweight classifier that maps natural language descriptions to UI primitive categories. This does not require a full LLM call - it uses a fine-tuned classification model trained on (prompt, UI element type) pairs. Latency target: <100ms.

**Pass 2 - Targeted library fetch:**
```
for type in manifest:
  matches = vector_search(
    query = type + org_id,
    filter = {org_id: org_id},
    top_k = 5  # top 5 per component type
  )
  score by: semantic_similarity * 0.7 + usage_frequency * 0.3
  take top 2 per type
```

The result is a `library_context` payload (see Section 5 for schema) bounded at ~8,000 tokens, covering the components most relevant to the specific prompt.

**Caching:**
- Cache key: `hash(org_id + library_version + component_type_manifest)`.
- TTL: 10 minutes.
- Cache is invalidated on `library_published` event for the org.
- Cache hit rate target: >70% (most generation sessions in an active file occur within a 10-minute window of a library state that has been previously fetched).

**SLOs for this service:**
- P50 latency: <300ms (cache hit path).
- P95 latency: <800ms (cache miss, full two-pass assembly).
- Availability: 99.9%.
- `library_context_injection_failure_rate`: <0.5%.

---

### Layer 3: Design-Aware Generation Pipeline

**Purpose:** Takes the user prompt and the `library_context` payload and generates a frame where every element is mapped to a library component instance.

**Prompt structure:**

```
[System instruction]
You are Figma's Make Designs AI.
You have the user's org design system below.
Every element in your generated frame must use a component from this library.
Map each UI element to the exact component ID and variant specified in the library context.
Do not use hardcoded hex values. Reference color variables by name.
Do not use raw font sizes. Reference text styles by name.
If the library has no match for a required element, flag it as library_gap.

[Library context - up to 8,000 tokens]
{library_context_payload}

[User prompt]
{prompt_text}
```

**Output format (structured JSON, not prose):**

The generation model outputs a structured JSON frame spec rather than a Figma plugin command sequence. This allows the Compliance Computation Service to inspect and score the output before it is placed on the canvas.

```json
{
  "frame_spec": {
    "width": 375,
    "height": 812,
    "elements": [
      {
        "element_id": "el_01",
        "type": "component_instance",
        "component_id": "cmp_btn_primary",
        "variant": "Button/primary/default",
        "position": { "x": 16, "y": 760 },
        "overrides": { "label": "Save preferences" }
      },
      {
        "element_id": "el_02",
        "type": "library_gap",
        "semantic_type": "DataTable",
        "fallback": "raw_frame",
        "position": { "x": 16, "y": 200 }
      }
    ],
    "color_fills": [
      { "element_id": "el_01", "fill": "variable:semantic/primary" }
    ],
    "text_styles": [
      { "element_id": "el_03", "style": "body/md" }
    ]
  }
}
```

Elements with `type: component_instance` reference real library component IDs. The Figma canvas layer translates these IDs into actual component instances when placing the frame. Elements with `type: library_gap` are placed as styled raw frames and flagged in the compliance panel.

**Model routing:**
- Primary: Figma's internal generation model (fine-tuned on UI layout tasks).
- Fallback: GPT-4o with the same prompt structure, used when the internal model is unavailable or when `library_component_count > 300` (GPT-4o's larger context window handles larger library payloads more reliably).

---

### Layer 4: Compliance Computation Service

**Purpose:** Scores the generation output against the org library before the frame is placed on the canvas. Populates the compliance panel.

**Inputs:** `frame_spec` JSON from the generation pipeline + `library_context` payload.

**Computation:**

```
total_elements = count(frame_spec.elements where type != "container")
component_instances = count(elements where type == "component_instance")
library_gaps = count(elements where type == "library_gap")

component_coverage_pct = component_instances / total_elements * 100

color_fills = count(frame_spec.color_fills)
variable_fills = count(fills where fill starts_with "variable:")
variable_coverage_pct = variable_fills / color_fills * 100

text_nodes = count(elements where has_text = true)
styled_text = count(frame_spec.text_styles)
text_style_coverage_pct = styled_text / text_nodes * 100

non_compliant_list = [
  for each library_gap element:
    find top 3 nearest library components by semantic_similarity to element.semantic_type
    -> { element_id, semantic_type, suggested_matches: [component_id, name, similarity_score] }
]
```

**Output:** `compliance_report` struct emitted within 500ms of frame spec generation completing.

```json
{
  "session_id": "uuid",
  "component_coverage_pct": 92.0,
  "variable_coverage_pct": 100.0,
  "text_style_coverage_pct": 94.0,
  "library_gap_count": 1,
  "non_compliant_elements": [
    {
      "element_id": "el_02",
      "semantic_type": "DataTable",
      "suggested_matches": [
        { "component_id": "cmp_table_basic", "name": "Table/basic", "score": 0.61 },
        { "component_id": "cmp_list_card", "name": "List/card", "score": 0.48 }
      ]
    }
  ]
}
```

**Panel rendering:** The Figma canvas client receives the `compliance_report` alongside the frame spec and renders the compliance panel below the generated frame before the user accepts or discards.

---

### Layer 5: Library Gap Pipeline

**Purpose:** Captures `library_gap_detected` events, deduplicates them, and routes notifications to design systems leads.

**Event schema:**

```json
{
  "event": "library_gap_detected",
  "org_id": "uuid",
  "generation_session_id": "uuid",
  "missing_element_type": "DataTable",
  "prompt_context": "data table with sortable columns",
  "gap_count_this_week": 3,
  "ts": "ISO8601"
}
```

**Deduplication:** Events are deduplicated per `(org_id, missing_element_type)` within a 7-day rolling window. A single component type missing from 5 sessions in one day triggers a Tier 1 real-time Figma notification to the design systems lead. The same type missing from 3+ sessions in 7 days (but not triggering Tier 1) is included in the weekly digest.

**Why this matters architecturally:** The gap pipeline turns the generation pipeline into a design system feedback mechanism. Gaps are not just errors - they are signals about which components the org needs next. This creates a flywheel: more AI generation -> better library gap visibility -> library expands -> better AI generation quality.

---

## 4) Data flow - happy path

```
1. Designer submits prompt in Make Designs panel
   -> POST /ai/generate {org_id, file_id, prompt}

2. Context Injection Service checks cache
   -> cache miss: run two-pass assembly (type manifest + targeted library fetch)
   -> cache hit: return cached library_context (P50 latency <300ms)

3. library_context payload assembled (<8,000 tokens)

4. Design-Aware Generation Pipeline receives (prompt + library_context)
   -> generates frame_spec JSON with component_ids, variable references, text_style references

5. Compliance Computation Service scores frame_spec
   -> produces compliance_report within 500ms

6. Canvas client receives frame_spec + compliance_report
   -> translates component_ids to real Figma component instances
   -> renders frame on canvas
   -> displays compliance panel with scores and non-compliant elements list

7. Designer reviews panel, replaces any flagged elements, accepts frame

8. Events fire:
   ai_make_designs_generated (uses_org_library_components: true, outcome: edited)
   compliance_panel_viewed
   [if applicable] library_gap_detected per non-compliant element type
```

**Total latency budget:**

| Step | P50 target | P95 target |
|---|---|---|
| Context injection (cache hit) | 300ms | 800ms |
| Context injection (cache miss) | 600ms | 1,200ms |
| Frame generation (LLM) | 3,000ms | 6,000ms |
| Compliance computation | 100ms | 500ms |
| Canvas rendering | 200ms | 500ms |
| **End-to-end (cache hit)** | **~4,000ms** | **<8,000ms** |
| **End-to-end (cache miss)** | **~5,000ms** | **<10,000ms** |

---

## 5) Core data model

### LibraryComponent

```
component_id          uuid
org_id                uuid
library_file_id       uuid
library_version       string
name                  string
category              string          // "Button", "Input", "Card", etc.
description           string
variants              Variant[]
usage_frequency       integer         // times used across org files in last 30d
embedding_vector      float[1536]     // text-embedding-3-small on name+desc+semantic_label
semantic_label        string          // "primary CTA", "secondary action", "low-emphasis"
indexed_at            timestamp
```

### LibraryVariant

```
variant_id            uuid
component_id          uuid
name                  string          // "Button/primary/default"
properties            map<string,string>  // {type: primary, state: default, size: md}
semantic_label        string          // "main CTA, enabled state"
```

### LibraryColorVariable

```
variable_id           uuid
org_id                uuid
name                  string          // "semantic/primary"
collection            string          // "Brand", "Semantic", "Neutral"
value_light           string          // hex
value_dark            string          // hex
semantic_role         string          // "primary brand action"
```

### LibraryTextStyle

```
style_id              uuid
org_id                uuid
name                  string          // "body/md"
font_family           string
font_size             integer
font_weight           integer
line_height           float
semantic_role         string          // "body paragraph, medium size"
```

### GenerationSession

```
session_id            uuid
org_id                uuid
file_id               uuid
user_id               uuid
prompt_text           string
library_context_injected  boolean
library_version           string
component_count_injected  integer
cache_hit                 boolean
frame_spec_id             uuid
compliance_report_id      uuid
outcome                   enum(accepted, edited, deleted)
generation_latency_ms     integer
created_at                timestamp
```

### ComplianceReport

```
report_id             uuid
session_id            uuid
component_coverage_pct    float
variable_coverage_pct     float
text_style_coverage_pct   float
library_gap_count         integer
non_compliant_elements    NonCompliantElement[]
computed_at               timestamp
panel_viewed              boolean
panel_dismissed_without_action  boolean
```

---

## 6) Key trade-offs

### Two-pass vs. single-pass generation

**Two-pass** (type manifest extraction -> targeted library fetch -> generation) adds ~200ms latency but produces dramatically better component specificity. The manifest step lets the system fetch only the components the prompt actually needs, staying within the token budget without relevance ranking over the entire library.

**Single-pass** (embed prompt -> top-N library components by cosine similarity -> generation) is simpler but degrades for prompts where the UI element types are described implicitly ("settings page for notification preferences") rather than explicitly ("page with toggles and a save button"). The semantic similarity between "notification preferences" and "Toggle" is lower than expected.

Decision: Two-pass for v1. Monitor manifest extraction accuracy - if the classifier misses element types in >10% of sessions, add a single-pass fallback.

### Strict component matching vs. best-effort matching

**Strict:** The generation model must use an exact library component for every element. If no match exists, the element is placed as `library_gap` and flagged. This produces cleaner output but more visible warnings for libraries with coverage gaps.

**Best-effort:** The model uses the nearest library component even if the semantic match is weak (e.g., using a `Card/content` component as a table container because no `Table` exists). This reduces gap warnings but at the cost of producing system-compliant-looking frames that are semantically wrong - engineers may trust them and build from incorrect specs.

Decision: Strict matching. The compliance panel is the user's signal for gaps; best-effort matching would produce false confidence. Design systems teams should fix the library, not accept wrong component substitutions.

### Cache invalidation on library publish vs. real-time library reads

**Cache:** Library context cached for 10 minutes per org library version. A designer who publishes a new component and immediately generates a frame using it will hit the old cache for up to 10 minutes.

**Real-time:** Every generation session fetches the live library. Zero stale cache risk, but eliminates the primary latency and cost lever.

Decision: Cache with 10-minute TTL. Real-time reads add 600-800ms to every cache-miss path and are unnecessary for the vast majority of sessions where the library has not changed since the last fetch. The 10-minute window is short enough that designers who publish a new component can trigger a new generation session after a brief wait without losing trust in the system.

---

## 7) Failure modes

| Failure | Detection | Mitigation | Degraded state |
|---|---|---|---|
| Library Index Service unavailable | Health check failure; `library_context_injection_failure_rate` spike | Circuit breaker; fall back to generic generation; `library_context_injected = false` in event | Designer sees generic (non-library) Make Designs output; no error shown |
| Context injection latency exceeds 1.5s | P95 latency alert | Serve partial library context (top 30 components by usage frequency only, skipping two-pass assembly) | Lower coverage quality; generation proceeds |
| Component type manifest classifier fails (returns empty manifest) | `manifest_extraction_failure` event | Fall back to single-pass: embed prompt, return top 40 components by cosine similarity | Slightly lower component coverage; no user-visible error |
| Generation model returns malformed frame_spec | JSON parse failure | Retry once with explicit JSON schema hint in prompt; if second attempt fails, fall back to generic generation | Rare; user sees standard "Try again" prompt |
| Compliance Computation Service exceeds 500ms | P95 alert | Surface compliance scores only (no non-compliant element list) if full computation exceeds 500ms | Panel shows coverage percentages only; no suggested replacements |
| Library gap pipeline event backlog | Consumer lag > 1 min | Scale consumer horizontally; Tier 1 notifications degrade to Tier 2 during backlog recovery | Notification delay; no user-facing impact |
| Multi-library conflict (org has two published org-scoped libraries) | `library_scope_conflict` event logged | Enforce priority: most recently published org library wins; log which library was used per session | Predictable behaviour; design systems lead can inspect via audit log |

---

## 8) User journeys (architecture perspective)

### Journey 1: Designer generates a settings page (happy path)

1. File belongs to an org with a published library (140 components, 32 variables, 12 text styles).
2. Designer submits prompt. Context Injection Service checks cache - cache hit on the org's library context for the type manifest [Toggle, Button, Divider]. Returns cached payload in 280ms.
3. Generation pipeline receives prompt + 8 components from the library (Toggle/active, Toggle/inactive, Button/primary/default, Button/primary/disabled, Card/content, Divider/horizontal/full, Input/text/default, FormLabel/default).
4. frame_spec returned in 4.2s. All 14 elements use `component_instance` type with valid `component_id` references.
5. Compliance: component coverage 100%, variable coverage 100%, text style coverage 92% (two text nodes used raw 13px font not mapped to `body/sm`). Report computed in 180ms.
6. Canvas places frame, shows compliance panel with "Looks good - 2 text elements not using library styles" flagged.
7. Designer clicks suggested replacement for both text nodes, accepts frame.
8. Total time from prompt submit to frame acceptance: ~7s.

### Journey 2: Library index stale during generation

1. Design systems lead published a new `Table/sortable` component 8 minutes ago.
2. A designer generates a "data table with filters" - prompt arrives during the 10-minute cache window before the new component is indexed.
3. Context Injection Service serves the cached library context (does not include `Table/sortable`).
4. Generation pipeline sees no table component; returns `library_gap` for the table element.
5. Compliance panel shows "Table not in your library" - technically incorrect for 2 more minutes until the cache expires.
6. At the next generation attempt (after 10-minute TTL), the cache refreshes and the new `Table/sortable` component is available.

**Trade-off accepted:** The 10-minute window is a deliberate cost/accuracy trade-off. Reducing the TTL to 1 minute would eliminate this gap but increase library fetches by 10x. This scenario requires a design systems lead to publish a component and a designer to immediately generate a frame using that exact new component type - a low-frequency event.

### Journey 3: Org with no published library (Free/Professional tier without org-scoped library)

1. Generation session fires. Context Injection Service checks for published org-scoped library - none found.
2. `library_context_injected = false`. Generation proceeds with the existing generic Make Designs pipeline.
3. No compliance panel rendered (no library to compare against).
4. `ai_make_designs_generated` event fires with `library_context_injected: false`, `uses_org_library_components: false`.
5. No user-visible change from current product behaviour.

---

## 9) Open questions

1. **Manifest classifier training data:** The two-pass approach relies on a classifier that maps prompt text to UI component types. What training corpus is used? Figma likely has access to anonymised Make Designs prompt logs (with opt-in) that can provide ground truth, but this needs explicit data governance approval before training can begin.

2. **Code Connect linkage in generation:** Dev Mode's Code Connect links Figma components to their Storybook/codebase counterparts. Should AI-generated component instances automatically carry Code Connect metadata? This would make generated frames immediately engineer-ready - but it requires the generation pipeline to know which components have active Code Connect links, adding a dependency on the Dev Mode infrastructure team.

3. **Cross-file local component handling:** A file may have locally defined components (not from a published library) that the designer regularly uses. Should local components be included in the library context? Including them increases coverage but creates privacy complexity (local components may include draft patterns not ready for AI use). Recommendation: exclude local components in v1; add as opt-in toggle in v2.

4. **Variable mode context (light/dark theme):** The library context includes both `value_light` and `value_dark` for each color variable. Should the generation pipeline be aware of the file's current variable mode (light/dark/system)? If a file is set to dark mode and the compliance panel checks only dark mode variable references, coverage scores will differ from a file in light mode. Not a blocking question for v1 - default to checking against the active mode.

5. **Variant selection confidence thresholds:** The compliance panel in v2 will show a confidence score per variant selection. What is the minimum threshold below which a variant selection is flagged as low-confidence? This requires a calibration dataset (human-labelled correct/incorrect variant selections) that does not yet exist. Plan: collect labels via the thumbs-up/thumbs-down micro-feedback in Phase 2 rollout (per PRD Q5 resolution) and use them to calibrate in v2.

---

*All latency targets and cost figures are directional estimates based on publicly available LLM and vector search benchmarks, not internal Figma data.*
