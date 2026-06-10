# Perplexity Verified Answers - System Architecture

**What this explains:** The system architecture that powers Perplexity Verified Answers - a claim-level citation grounding layer that verifies the semantic relationship between each LLM-generated claim and its cited source before the answer streams to the user. Specifically: how Perplexity extracts structured claim-citation pairs from a raw LLM draft, runs parallel cross-encoder scoring against cached source content, maps calibrated confidence scores to UI signals, replaces weak citations where possible, and persists a per-answer audit record for enterprise accounts - all within a 400ms P95 pre-stream gate.

**PRD reference:** https://github.com/004mayank/product-prd/blob/main/perplexity-prd.md

**Version:** v3 - Final system design
**Changes from v2:** Resolved all open questions (Q4 cross-turn source attribution via `source_retrieval_turn_index`; Q6 streaming UX specification for the pre-stream gate delay indicator). Added multi-region deployment topology diagram, phased rollout plan with per-phase launch gates and kill-switch conditions, experiment backlog with rollout owners and end-state decisions, operational runbook index, and complete version history.

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core problem statement, five system layers, data flow narrative, core data model, failure modes, architectural trade-offs, open questions |
| v2 | Mermaid diagrams (system + sequence), API contracts with error codes, instrumentation event schemas, competitive architecture comparison, scaling model, inter-service communication patterns, circuit breaker design, deeper failure mode runbooks, Q2 and Q5 resolved |
| v3 | Resolved all remaining open questions (Q4 cross-turn attribution, Q6 streaming UX), multi-region topology diagram, phased rollout plan with launch gates and kill switches, experiment backlog with acceptance criteria and rollout owners, operational runbook index |

---

## 1) What this system is

Perplexity's answer pipeline has three stages: retrieve, synthesize, stream. The retrieval stage fetches and caches the full text of candidate source pages. The synthesis stage hands that content to an LLM which produces a prose answer with inline citation superscripts. The streaming stage sends tokens to the browser as they arrive.

Verified Answers inserts a fourth stage between synthesis and streaming: a pre-stream verification gate. The gate holds the completed answer draft for up to 400ms (Academic Focus) or 600ms (Web Focus), runs a semantic similarity check on every claim-citation pair, and enriches the answer payload with confidence signals before the first token is sent. The user sees a verified answer - not a raw LLM output annotated after the fact.

The gate architecture is the core design constraint. It cannot be asynchronous. If confidence signals arrive after the answer text, the user has already read the first citation before seeing its verification status. Late signals feel like an afterthought; pre-stream signals feel like a product guarantee. The latency budget is the price of that guarantee.

The system has five architectural layers:

1. **Claim Extraction Service** - Parses the LLM answer draft into structured (claim_text, citation_index) pairs before the answer is released to the stream.
2. **Source Content Cache Interface** - Serves the full cached page text for each cited URL from the retrieval pipeline's existing cache; the verification layer is a consumer, not an owner, of this cache.
3. **Semantic Verifier Service** - Runs the cross-encoder model on batches of (claim, source_passage) pairs in parallel; returns calibrated `citation_confidence_score` values and the specific `source_passage_used` for each claim.
4. **Confidence Signal Layer** - Maps scores to the `verified` / `single_source` / `low_confidence` / `unverified_claim` enum; applies exact-match and source-format overrides; attempts replacement source selection for `low_confidence` citations.
5. **Audit Persistence Layer** - Writes the `CitationAuditRecord` asynchronously to the persistence store; serves the audit read API and export pipeline for Teams and Enterprise accounts.

---

## 2) Non-functional requirements

| Requirement | Target | Measurement | Alert threshold |
|---|---|---|---|
| Verification gate P95 latency - Academic Focus | <400ms total verification pass | `verification_pass_total_latency_ms` in `CitationAuditRecord`; 7-day rolling | >450ms for >5 min rolling - page on-call; degrade to passthrough mode |
| Verification gate P95 latency - Web Focus | <600ms total verification pass | Same field | >700ms for >5 min rolling - page on-call |
| Cross-encoder inference P95 (per claim) | <150ms | Per-claim span in verifier service telemetry | >200ms for >1 min sustained - investigate model serving |
| Batch inference P95 (up to 8 claim-source pairs) | <200ms | Batch inference span | >300ms - investigate batching overload |
| Verifier service availability | 99.5% | Rolling 7 days | <99% - auto-activate degraded mode (passthrough; no signals) |
| False-positive rate (verified claim marked `low_confidence`) | <8% | Weekly human review of 200 sampled `low_confidence` results against labelled ground truth | >12% - pause rollout; investigate model calibration drift |
| `CitationAuditRecord` write success rate | >99.9% | Reconciliation job: compare `verification_pass_status: complete` answer count vs. persisted audit record count | <99.5% for 1h - alert; activate audit write retry queue |
| Source cache hit rate (cited URLs present in retrieval cache) | >95% | Cache miss events per 1,000 verifications | <90% - alert retrieval team; investigate cache TTL regression |
| Degraded mode activation rate | <0.5% of queries | `verification_pass_status: degraded` event rate | >1% of queries - investigate verifier service health |
| Replacement source search P95 | <100ms | Replacement search span in gate coordinator | >120ms - log and assign `low_confidence` with original source; do not extend budget |
| Verifier service circuit breaker trip rate | <0.1% of 5-min windows | Circuit breaker state transition events | >0.5% - investigate sustained error rate; review scaling headroom |
| Cross-region replication lag (audit records) | <30s P99 | `CitationAuditRecord` replication lag monitoring | >60s - alert data platform on-call |

---

## 3) System architecture diagram

```mermaid
flowchart TD
    subgraph Query["Query Pipeline (existing)"]
        A[User Query]
        B[Retrieval Pipeline]
        C[(Source Content Cache\nRedis - key: source_text:{sha256(url)}\nTTL: 10-60 min)]
        D[LLM Synthesis\nAcademic or Web Focus model]
    end

    subgraph Gate["Verification Gate - pre-stream blocker"]
        E[Gate Coordinator\ncontrols latency budget\nenforces hard timeout]
        F[Claim Extraction Service\n< 20ms P95\nrule-based parser]
        G[Source Content Cache Interface\n< 15ms P95\nbatched parallel reads]
        H[Semantic Verifier Service\nbatch cross-encoder inference\n< 200ms P95 for 8-pair batch]
        I[Confidence Signal Layer\nscore-to-signal mapping\noverride rules\nreplacement source search]
    end

    subgraph Persist["Persistence and Delivery"]
        J[(CitationAuditRecord Store\nPostgres)]
        K[Audit Write Queue\nasync; decoupled from latency path]
        L[Token Stream\nbrowser renderer]
        M[Audit API\nGET /v1/audit/threads/{thread_id}]
        N[Export Service\nJSON + PDF generation]
    end

    A --> B
    B --> C
    B --> D
    D --> E
    C --> G
    E --> F
    F -->|claim-citation pairs| G
    G -->|source passages per claim| H
    H -->|citation_confidence_score per claim| I
    I -->|enriched answer payload| L
    I -->|CitationAuditRecord payload| K
    K --> J
    J --> M
    J --> N
    E -.->|degraded mode: passthrough if verifier unavailable| L
```

---

## 4) Verification gate - sequence diagram

The sequence diagram shows the critical timing structure of the pre-stream gate for a 4-citation Academic Focus query.

```mermaid
sequenceDiagram
    participant RP as Retrieval Pipeline
    participant SC as Source Cache (Redis)
    participant LLM as LLM Synthesis
    participant GC as Gate Coordinator
    participant CES as Claim Extraction Service
    participant SCCI as Source Cache Interface
    participant SVS as Semantic Verifier Service
    participant CSL as Confidence Signal Layer
    participant APL as Audit Persistence Layer
    participant STR as Token Stream (Browser)

    RP->>SC: cache source_text for each retrieved URL (TTL 10-60min)
    RP->>LLM: invoke synthesis with retrieved sources
    LLM->>GC: complete answer draft + source list (t=0ms)
    GC->>CES: parse draft into claim-citation pairs (t=0ms)
    CES-->>GC: [claim_1..claim_4] (t=20ms)
    GC->>SCCI: batch fetch source passages for 4 unique URLs (t=20ms)
    SCCI->>SC: parallel GET source_text keys
    SC-->>SCCI: source texts + segmented windows (t=35ms)
    SCCI-->>GC: passages per claim (t=35ms)
    GC->>SVS: batch inference request [{claim, passages}] x 4 (t=35ms)
    SVS-->>GC: citation_confidence_score + source_passage_used x 4 (t=235ms)
    GC->>CSL: map scores to signals; apply overrides (t=235ms)
    CSL-->>GC: confidence_annotation x 4 (t=245ms)
    GC->>STR: release enriched answer payload (t=245ms)
    GC->>APL: publish CitationAuditRecord to write queue (async, t=245ms)
    APL->>APL: write to Postgres (decoupled, t=245ms+)
    STR->>STR: render answer + confidence badges
    note over GC,STR: Total gate latency: 245ms < 400ms Academic Focus SLO
```

---

## 5) System layers

### Layer 1: Claim Extraction Service

**Purpose:** Parses the completed LLM answer draft into a structured list of (claim_text, citation_index, char_start, char_end) tuples. Each tuple represents one factual assertion that carries a citation superscript.

**Input:** The raw answer draft as a string, with citation superscripts encoded in a known markup format (e.g., `[1]`, `[2]`).

**Output:** An ordered list of `ClaimExtractionResult` objects:

```json
{
  "claim_id": "string - UUID generated per claim per answer turn",
  "claim_text": "string - the verbatim text of the factual assertion carrying this citation",
  "citation_index": "integer - 1-based index of the citation superscript in the answer",
  "source_url": "string - the URL mapped to citation_index from the synthesis stage source list",
  "char_start": "integer - character offset of claim start in the draft",
  "char_end": "integer - character offset of claim end in the draft"
}
```

**Extraction strategy:** The service uses a rule-based parser tuned to Perplexity's answer structure (not a separate LLM call, which would add unacceptable latency). The parser identifies sentence boundaries, locates citation markers, and associates each marker with the immediately preceding sentence or clause. Where one superscript number appears at the end of a multi-clause sentence, each clause is treated as a separate claim with the same `citation_index` - this avoids over-crediting a single citation for claims it may not individually support.

**Compound claim handling policy:** When the LLM produces a multi-clause sentence ending with a single superscript (e.g., "AlphaFold2 uses the Evoformer architecture, trains on PDB structural data, and achieves 92.4 GDT on CASP14 targets [1]"), the parser splits at clause boundaries using a heuristic: commas followed by a coordinating conjunction ("and", "but", "or") and independent verb phrases are treated as clause boundaries. Each resulting clause becomes a separate `ClaimExtractionResult` with the same `citation_index`. The cross-encoder then scores each clause independently. The final `confidence_signal` for citation [1] uses the minimum score across all clauses from the same source - if the source supports "uses Evoformer" (score 0.89) but not "achieves 92.4 GDT on CASP14 targets" (score 0.41), the citation receives `low_confidence`. This is the conservative choice: one unverified clause in a compound claim taints the whole citation.

**Edge case - zero citations:** If the LLM draft contains no citation superscripts, the Claim Extraction Service returns an empty list. The verification gate skips all verifier calls and sets `verification_pass_status: skipped_no_citations`. The answer streams immediately with no signals and no audit record.

**Edge case - citation index with no URL mapping:** If a superscript references an index that is not present in the synthesis stage's source list (e.g., due to a source retrieval failure), the corresponding claim is emitted with `source_url: null` and the Confidence Signal Layer assigns `confidence_signal: unverified_claim` without calling the verifier.

**Latency target:** <20ms P95. The extraction is a synchronous in-process operation; it must not be the bottleneck in the verification gate.

---

### Layer 2: Source Content Cache Interface

**Purpose:** Retrieves the full cached text of each cited source URL to provide passage material for the cross-encoder. This layer is a read-only consumer of the retrieval pipeline's existing source cache; it does not fetch live pages.

**Cache lookup:** For each unique `source_url` in the claim list, the interface performs a key lookup against the retrieval cache (key structure: `source_text:{sha256(url)}`). The returned value is the full parsed text of the page at the time of retrieval, stored as a UTF-8 string. The cache TTL is set by the retrieval pipeline (typically 10-60 minutes depending on source type); the verification layer does not control or extend this TTL.

**Source cache TTL contract:** The verification layer requires the retrieval pipeline to maintain a minimum cache TTL of 120 seconds for all cited source URLs, measured from the moment the URL is first cached at retrieval time. This 120-second floor covers the worst-case LLM synthesis time (8-12 seconds for complex answers) plus the gate latency budget (600ms max), with comfortable headroom. The retrieval team owns the TTL configuration; the verification team monitors compliance via the `source_cache_miss_at_verification` event (see Section 9). The 95th-percentile cache hit rate target (>95%) is a shared SLO between the retrieval team and the verification team. If this SLO is missed for three consecutive days, the retrieval team is paged. The TTL floor of 120 seconds does not override the retrieval team's upper TTL limits for rapidly-updating sources (e.g., live news: typically 10 minutes; academic papers: typically 60 minutes).

**Cache miss handling:** A miss means the source text is not available for verification. This happens when: (a) the source retrieval failed silently during the query pipeline, (b) the source page requires JavaScript rendering and the retrieval pipeline returned an empty body, (c) the cache entry expired between retrieval and verification (TTL regression or very long synthesis), or (d) the cached content is too short to segment meaningfully (<100 words). On a miss, the claim is passed to the Confidence Signal Layer with `source_passages: null`; the layer assigns `confidence_signal: unverified_claim`.

**Paywall and JavaScript-render detection:** When cached content length is <500 words and the content string matches known paywall signature phrases ("subscribe to continue", "sign in to read", "enable JavaScript"), the interface assigns `cache_quality: paywall_blocked` or `cache_quality: js_required` instead of a simple miss. The Confidence Signal Layer maps these quality flags to a distinct signal state (`unverified_paywall` or `unverified_js_blocked`) rather than `unverified_claim`. This distinction is exposed in the audit record and in the source panel copy ("Source requires a subscription - could not verify this claim") so that Academic Focus users understand that the unverified signal reflects an access limitation, not a quality failure.

**Source text segmentation:** The interface does not return the full page text as a single string to the verifier. It segments the cached text into overlapping windows of 512 tokens with 64-token overlap, returning up to 20 candidate passages per source. This windowing is done at this layer - before the verifier call - to reduce the payload size and to align with the cross-encoder's token input limit.

**Latency target:** <15ms P95 for cache hit across up to 8 unique source URLs (typical answer citation count). Cache reads are batched as parallel key lookups.

---

### Layer 3: Semantic Verifier Service

**Purpose:** The core inference service. Receives batches of (claim_text, candidate_source_passages) pairs and returns a `citation_confidence_score` (0.0-1.0, Platt-scaled) and the specific `source_passage_used` that produced the highest score for each claim.

**Model:** `cross-encoder/ms-marco-MiniLM-L-6-v2` with Platt scaling calibration for Academic Focus Phase 1. See the PRD section 5 for the full benchmark and model selection rationale. The service is model-version-aware: the deployed model version is recorded in every `CitationVerificationResult` record, enabling post-hoc analysis of accuracy drift across model versions.

**Inference flow:**

```
Input: [{claim_text, source_passages: [passage_1, passage_2, ..., passage_20]}]

For each (claim, passage) pair:
  -> cross-encoder produces raw logit score
  -> select the passage with the highest raw score as source_passage_used
  -> apply Platt scaling: citation_confidence_score = sigmoid(a * raw_score + b)
     where a, b are learned from the 2,000-example calibration set (ECE target <0.05)

Output: [{claim_id, citation_confidence_score, source_passage_used, source_passage_char_offset, raw_score, verification_latency_ms}]
```

**Batching:** All claims in a single answer are batched into a single inference call (up to 8 claim-source pairs per batch). The batch is scored in parallel across the cross-encoder model's capacity. This is the engineering requirement from the PRD: citations must be verified in parallel, not sequentially.

**Calibration requirements:** The Platt scaling parameters (a, b) must be re-fitted on a held-out calibration set before each model version promotion. The Expected Calibration Error (ECE) of the deployed model must be <0.05 on a 2,000-example calibration set. This is a model release gate: calibration test failure blocks production promotion.

**Inter-service communication contract:** The Gate Coordinator calls the Semantic Verifier Service over an internal gRPC connection. The gRPC service definition:

```protobuf
service SemanticVerifier {
  rpc VerifyBatch(VerifyBatchRequest) returns (VerifyBatchResponse);
}

message ClaimPassagePair {
  string claim_id = 1;
  string claim_text = 2;
  repeated string source_passages = 3;      // up to 20 passages per claim
  string source_url = 4;
  string source_domain = 5;
}

message VerifyBatchRequest {
  string answer_turn_id = 1;
  string focus_mode = 2;                    // "academic" | "web"
  repeated ClaimPassagePair pairs = 3;      // up to 8 pairs per batch
  string verifier_model_version = 4;
}

message ClaimVerificationResult {
  string claim_id = 1;
  float citation_confidence_score = 2;      // Platt-calibrated [0.0, 1.0]
  float raw_cross_encoder_score = 3;
  string source_passage_used = 4;           // highest-scoring passage text
  int32 source_passage_char_offset = 5;
  int32 verification_latency_ms = 6;
}

message VerifyBatchResponse {
  repeated ClaimVerificationResult results = 1;
  string model_version_used = 2;
  int32 batch_inference_latency_ms = 3;
}
```

**Timeout policy:**
- Academic Focus: Gate Coordinator enforces a 350ms hard timeout on the gRPC call (leaving 50ms headroom to the 400ms P95 SLO). If the timeout fires, the Gate Coordinator activates degraded mode for this answer.
- Web Focus: Gate Coordinator enforces a 550ms hard timeout (leaving 50ms headroom to the 600ms P95 SLO).
- The Semantic Verifier Service itself has an internal per-batch timeout of 250ms (Academic) / 450ms (Web). If a batch exceeds this internal limit, the service returns partial results for completed pairs and `null` for remaining pairs. The Gate Coordinator treats `null` results as `unverified_claim`.

**Retry policy:** The Gate Coordinator does NOT retry failed Semantic Verifier Service calls within the verification gate. The latency budget has no room for retries. A single failure activates degraded mode for the affected answer. Retry logic applies only at the service health-check level (circuit breaker, described below).

**Latency budget within the gate:**

| Step | Target P95 | Cumulative |
|---|---|---|
| Claim extraction | <20ms | 20ms |
| Source cache fetch (batched) | <15ms | 35ms |
| Verifier batch inference | <200ms | 235ms |
| Score mapping + confidence signal assignment | <10ms | 245ms |
| Replacement source search (if triggered) | <100ms | 345ms |
| Answer payload enrichment | <5ms | 350ms |
| **Total gate overhead (Academic Focus)** | **<350ms P95** | **within 400ms SLO** |

The 50ms headroom above 350ms absorbs internal scheduling jitter and network variance on the gRPC call to the verifier service.

---

### Layer 4: Confidence Signal Layer

**Purpose:** Maps calibrated `citation_confidence_score` values to the four-state `confidence_signal` enum, applies override rules for specific source formats and content patterns, attempts replacement source selection for `low_confidence` citations, and annotates the answer payload with the final signal for each claim.

**Score-to-signal mapping:**

```
citation_confidence_score >= 0.82 AND corroborating_sources >= 2  -> verified
citation_confidence_score >= 0.65 AND corroborating_sources == 1  -> single_source
citation_confidence_score  < 0.65 AND replacement_source_found   -> single_source (replacement applied)
citation_confidence_score  < 0.65 AND no_replacement_found       -> low_confidence
source_url == null OR source_passages == null                     -> unverified_claim
cache_quality == paywall_blocked                                  -> unverified_paywall
cache_quality == js_required                                      -> unverified_js_blocked
```

**Corroborating source check:** The "corroborating_sources >= 2" condition for `verified` requires that at least two independently retrieved sources each score >= 0.65 for the same claim. The verifier is called once; the Confidence Signal Layer checks whether any other source in the answer's retrieved pool also scores >=0.65 for this claim without a separate model call - it reuses already-computed scores from the batch. If only one retrieved source covers the claim, the signal is `single_source`, not `verified`, regardless of the primary score.

**Override rules (applied before signal assignment):**

| Override | Condition | Effect |
|---|---|---|
| Exact-match override | Claim contains a specific date or proper noun that appears verbatim in the source document | Assign `verified_exact_match` flag; skip score threshold; set `confidence_signal: single_source` minimum |
| PR domain override | `source_domain` matches known press release distribution domains (prnewswire.com, businesswire.com, globenewswire.com) | Lower `low_confidence` threshold from 0.65 to 0.55 for this source type; reduces false-positives from footer-structured date announcements |

**Replacement source ranking:** When a citation scores <0.65 and no override applies, the layer searches the answer's already-retrieved source pool for a better match. Candidate replacement sources are ranked by the following priority:
1. Sources from the same top-level domain as the original citation (domain continuity indicates topical relevance).
2. Remaining sources ranked by the retrieval pipeline's original relevance score (a proxy for topical relevance to the query, not to the specific claim).
Ranked candidates are tried in order, up to 3, each using a targeted cross-encoder call (single (claim, passages) pair). The first candidate that scores >=0.65 is selected as the replacement source. If none of the top 3 score >=0.65 within the 100ms time budget, `low_confidence` is assigned and the original citation is retained. This ranked strategy outperforms random sampling on the internal evaluation set (hit rate 31% vs. 22% random) while keeping the replacement call count bounded.

**Answer payload enrichment:** The layer writes a `confidence_annotation` object into the answer payload for each claim. The claim text is byte-identical to the original draft - enrichment adds metadata, never modifies content. The stream gate releases the enriched payload to the browser renderer, which maps `confidence_annotation` to the UI badge and source panel state.

---

### Layer 5: Audit Persistence Layer

**Purpose:** Persists a `CitationAuditRecord` for every answer that completes the verification gate. Serves the audit read API for Teams and Enterprise accounts. Powers the Spaces audit panel and the JSON/PDF export pipeline.

**Write path (asynchronous):** After the enriched answer payload is released to the stream, the verification gate coordinator publishes the `CitationAuditRecord` to an internal write queue (Kafka topic: `citation-audit-records`). A background worker consumes the topic and writes to the audit store (Postgres). The write is decoupled from the user-facing latency path. Write failures are retried up to 5 times with exponential backoff before being routed to a dead-letter queue and generating an alert.

**Reconciliation:** A daily reconciliation job compares the count of answers with `verification_pass_status: complete` (from the `citation_verification_completed` analytics event) against the count of persisted `CitationAuditRecord` rows for the same date. Any gap > 0.1% triggers an alert. The job replays missing records from the dead-letter queue where possible.

**Retention policy:**
- Teams and Enterprise: 5-year retention (full `CitationAuditRecord` with plain-text `claim_text`).
- Pro: 90-day retention (plain-text `claim_text`); purged and replaced with a one-way hash after 90 days.
- Free: 90-day retention; `claim_text` stored as hash only (never as plain text); no audit API access.

**Read path - Audit API:** The `GET /v1/audit/threads/{thread_id}` endpoint reads from a Postgres read replica (not the primary write path). Access is enforced at the API gateway: Teams and Enterprise keys receive 200; Pro and free keys receive 403. The full API contract is specified in Section 7 below.

**Read path - Export service:** The `GET /v1/audit/spaces/{space_id}/export` endpoint triggers batch generation of a `CitationAuditExport` payload. JSON exports are synchronous (200 with download URL). PDF exports are asynchronous: a job is enqueued, a `202` with `poll_url` is returned immediately, and the PDF is generated within 60 seconds for sessions up to 100 answers.

---

## 6) Circuit breaker design

The Semantic Verifier Service is the most failure-sensitive dependency in the verification gate. A slow or unavailable verifier service would hold the gate open, blocking all answer streaming. The circuit breaker prevents cascading failure.

**Circuit breaker configuration (Gate Coordinator -> Semantic Verifier Service):**

| Parameter | Value | Rationale |
|---|---|---|
| Failure threshold | >10% error rate in a 60-second sliding window | 10% error rate at Perplexity's query volume means thousands of degraded answers per minute - trip early |
| Slow-call threshold | >300ms P90 in a 60-second window | A P90 above 300ms means P95 is likely above the 350ms gate budget; latency degradation should trip the breaker before timeouts dominate |
| Minimum call volume | 20 calls per 60-second window | Prevent tripping on low-traffic test periods with sporadic errors |
| Open state duration | 30 seconds | Breaker stays open for 30 seconds; during this time, all gate coordinator requests skip the verifier and activate degraded mode immediately |
| Half-open probe | 1 probe request per 10 seconds | After the open state, the breaker sends 1 in 100 requests to the verifier; if the probe succeeds, the breaker transitions to closed |
| Re-close threshold | 3 consecutive successful probes | Require consistency before full re-close; avoid re-closing on a single lucky probe |

**Degraded mode behaviour during open circuit:**
- All gate coordinator instances detect the open breaker via a shared flag in Redis (key: `verifier_circuit_breaker_state`; TTL = 120s; updated by the instance that trips the breaker).
- During the open state, all answers stream without confidence signals. `verification_pass_status: degraded` is recorded in the audit record. No `CitationAuditRecord` is written for degraded answers.
- Users see no error message. The `citation_verification_completed` event is still emitted with `verification_pass_status: degraded` so the monitoring dashboard tracks the degradation window.

**Alert chain:**
1. Circuit breaker trips -> Immediate PagerDuty alert to on-call ML Platform engineer.
2. If open state persists >5 minutes -> Escalate to on-call backend platform engineer.
3. If open state persists >15 minutes -> Auto-page engineering lead + create incident.

---

## 7) API contracts

### Semantic Verifier Service internal API (gRPC)

Defined in full in Section 5, Layer 3. Summary:

- **Endpoint:** gRPC `SemanticVerifier.VerifyBatch`
- **Timeout:** 350ms (Academic Focus) / 550ms (Web Focus) enforced by Gate Coordinator
- **Max batch size:** 8 (claim, source_passages) pairs per call
- **Auth:** mTLS between Gate Coordinator and Verifier Service (internal service mesh)
- **Failure handling:** No retry within gate; circuit breaker at Gate Coordinator level

### Audit API - external REST

#### GET /v1/audit/threads/{thread_id}

Returns the `CitationAuditRecord` for a specific answer thread. Available to Teams and Enterprise accounts only.

**Authentication:** Bearer token (account-scoped API key)

**Request:**
```
GET /v1/audit/threads/{thread_id}?include_passages=true
Authorization: Bearer <api_key>
Accept: application/json
```

**Path parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `thread_id` | string | Yes | Thread ID from the Perplexity UI (e.g. `thr_9aQ1rL`) |

**Query parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `turn_index` | integer | No | (all turns) | Return audit record for a specific turn only |
| `include_passages` | boolean | No | false | If true, include `source_passage_used` in each `CitationVerificationResult`; increases payload size |

**Success response (200):**
```json
{
  "thread_id": "thr_9aQ1rL",
  "audit_records": [
    {
      "audit_record_id": "aud_7xK2mP",
      "answer_turn_index": 1,
      "focus_mode": "academic",
      "total_citations": 6,
      "verified_count": 4,
      "single_source_count": 1,
      "low_confidence_count": 1,
      "unverified_count": 0,
      "overall_answer_accuracy_score": 0.81,
      "verification_pass_total_latency_ms": 248,
      "verifier_model_version": "cross-encoder-msmarco-MiniLM-L6-v2-platt-v1",
      "verification_pass_status": "complete",
      "verification_results": [
        {
          "claim_id": "clm_3aB9kR",
          "claim_text": "AlphaFold2 achieved a median GDT score of 92.4 on CASP14 free modelling targets",
          "source_url": "https://www.nature.com/articles/s41586-021-03819-2",
          "citation_confidence_score": 0.91,
          "confidence_signal": "single_source",
          "source_passage_used": "...median GDT_TS score of 92.4 on free modelling...",
          "source_retrieval_turn_index": 1,
          "replacement_source_attempted": false,
          "verifier_model_version": "cross-encoder-msmarco-MiniLM-L6-v2-platt-v1",
          "verification_latency_ms": 112
        }
      ],
      "created_at": "2026-06-10T09:12:34Z"
    }
  ],
  "thread_verified_citation_pct": 0.83,
  "methodology_version": "claim-verifier-v1.0"
}
```

**Error responses:**

| Status | Code | Description |
|---|---|---|
| 401 | `unauthorized` | Missing or invalid API key |
| 403 | `plan_insufficient` | Account plan is free or Pro; audit API requires Teams or Enterprise |
| 403 | `thread_access_denied` | Thread belongs to a different account or Space the caller cannot access |
| 404 | `thread_not_found` | Thread ID does not exist or has been deleted |
| 404 | `audit_record_not_found` | Thread exists but was created before verification rollout; no audit record available |
| 429 | `rate_limited` | Audit API rate limit exceeded (100 requests/minute per account); `Retry-After` header included |
| 503 | `audit_store_unavailable` | Read replica is unhealthy; retry with exponential backoff |

**Rate limiting:** 100 requests per minute per API key. Enforced at the API gateway via token bucket. Response headers: `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Unix timestamp).

---

#### GET /v1/audit/spaces/{space_id}/export

Triggers export generation for a Space's citation audit records.

**Authentication:** Bearer token; Space admin role required

**Request body:**
```json
{
  "format": "json",
  "date_range_start": "2026-05-01T00:00:00Z",
  "date_range_end": "2026-06-09T23:59:59Z",
  "include_passages": true,
  "thread_ids": ["thr_9aQ1rL", "thr_4bX7mN"]
}
```

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `format` | enum | Yes | `json` or `pdf` |
| `date_range_start` | ISO 8601 | No | If omitted with no `thread_ids`, defaults to 30 days ago |
| `date_range_end` | ISO 8601 | No | If omitted, defaults to now |
| `include_passages` | boolean | No | Default: false |
| `thread_ids` | array of strings | No | Scope export to specific threads; overrides date range |

**Success response for JSON (200 - synchronous):**
```json
{
  "export_id": "exp_3bM9pX",
  "format": "json",
  "download_url": "https://exports.perplexity.ai/exp_3bM9pX?sig=...",
  "download_url_expires_at": "2026-06-11T09:12:34Z",
  "total_answers_included": 47,
  "total_citations_included": 282,
  "verified_citation_pct": 0.84,
  "methodology_version": "claim-verifier-v1.0"
}
```

**Success response for PDF (202 - async):**
```json
{
  "export_id": "exp_3bM9pX",
  "format": "pdf",
  "status": "processing",
  "poll_url": "/v1/audit/exports/exp_3bM9pX/status",
  "estimated_ready_in_seconds": 30
}
```

**Error responses:**

| Status | Code | Description |
|---|---|---|
| 400 | `invalid_date_range` | `date_range_end` is before `date_range_start` |
| 400 | `date_range_too_large` | Date range exceeds 365 days; use `thread_ids` to scope large exports |
| 403 | `export_access_denied` | Caller does not have admin role in the Space |
| 404 | `space_not_found` | Space ID does not exist or caller has no membership |
| 429 | `rate_limited` | Export rate limit exceeded (5 exports per hour per Space); `Retry-After` header included |

---

#### GET /v1/audit/exports/{export_id}/status

Poll endpoint for async PDF export status.

**Response when processing:**
```json
{
  "export_id": "exp_3bM9pX",
  "status": "processing",
  "progress_pct": 62,
  "estimated_ready_in_seconds": 12
}
```

**Response when ready:**
```json
{
  "export_id": "exp_3bM9pX",
  "status": "ready",
  "download_url": "https://exports.perplexity.ai/exp_3bM9pX.pdf?sig=...",
  "download_url_expires_at": "2026-06-11T09:14:12Z"
}
```

**Response when failed:**
```json
{
  "export_id": "exp_3bM9pX",
  "status": "failed",
  "error_code": "pdf_generation_timeout",
  "error_message": "PDF generation exceeded the 60-second limit. The session contains more than 100 answers; please scope by thread_ids.",
  "retry_allowed": true
}
```

---

## 8) Core data model

### `CitationVerificationResult` (transient - per claim per answer turn)

Produced by the Semantic Verifier Service and Confidence Signal Layer during the gate. Rolled up into `CitationAuditRecord` before async persistence. Not stored independently.

**v3 addition:** `source_retrieval_turn_index` field added to resolve Q4 (multi-turn conversation handling). This field records which turn in the thread originally retrieved the source used for this claim, enabling reviewers to understand cross-turn source reuse without inspecting multiple audit records.

```json
{
  "claim_id": "string - UUID; unique per claim per answer turn",
  "claim_text": "string - verbatim extracted claim text from the answer draft",
  "claim_char_start": "integer - character offset of claim start in answer draft",
  "claim_char_end": "integer - character offset of claim end in answer draft",
  "source_url": "string - full URL of cited source; null if no citation",
  "source_domain": "string - domain extracted from URL for analytics",
  "cache_quality": "enum: ok | paywall_blocked | js_required | too_short | miss",
  "source_passage_used": "string - the source passage that scored highest; max 512 chars",
  "source_passage_char_offset": "integer - byte offset of source_passage_used in the cached source document",
  "source_retrieval_turn_index": "integer | null - the turn_index in this thread that originally retrieved this source URL. null if the source was retrieved in the current turn. Populated when a follow-up answer cites a source cached in a prior turn.",
  "raw_cross_encoder_score": "float - raw model output before Platt scaling",
  "citation_confidence_score": "float [0.0-1.0] - calibrated probability that source supports claim",
  "confidence_signal": "enum: verified | single_source | low_confidence | unverified_claim | unverified_paywall | unverified_js_blocked",
  "corroborating_source_count": "integer - number of retrieved sources scoring >= 0.65 for this claim",
  "exact_match_override_applied": "boolean",
  "source_format_override_applied": "boolean - true if PR domain threshold adjustment was applied",
  "compound_claim_min_score_applied": "boolean - true if compound claim minimum-score rule determined the signal",
  "replacement_source_attempted": "boolean",
  "replacement_source_url": "string | null",
  "replacement_citation_confidence_score": "float | null",
  "replacement_ranking_method": "string | null - domain_continuity | retrieval_relevance_score",
  "verifier_model_version": "string - e.g. cross-encoder-msmarco-MiniLM-L6-v2-platt-v1",
  "verification_latency_ms": "integer",
  "created_at": "timestamp ISO 8601"
}
```

### `CitationAuditRecord` (persisted - per answer turn)

```json
{
  "audit_record_id": "string - UUID",
  "thread_id": "string - maps to the answer thread",
  "answer_turn_index": "integer - which turn in the thread produced this answer",
  "user_id": "string - hashed identifier; plan-gated (see retention policy)",
  "account_plan": "enum: free | pro | teams | enterprise",
  "space_id": "string | null - Spaces context if applicable",
  "focus_mode": "enum: web | academic | writing | video | social | math",
  "synthesis_model": "string - which LLM generated the answer",
  "total_citations": "integer",
  "verified_count": "integer",
  "single_source_count": "integer",
  "low_confidence_count": "integer",
  "unverified_count": "integer",
  "overall_answer_accuracy_score": "float - mean citation_confidence_score across all citations",
  "verification_results": ["CitationVerificationResult - one per citation in the answer"],
  "verification_pass_total_latency_ms": "integer",
  "verification_pass_status": "enum: complete | degraded | skipped_no_citations | skipped_parse_failure",
  "circuit_breaker_tripped": "boolean - true if degraded mode was caused by circuit breaker open state",
  "verifier_model_version": "string",
  "created_at": "timestamp ISO 8601",
  "retention_until": "timestamp ISO 8601 - plan-dependent retention policy"
}
```

### Retrieval source cache entry (read-only dependency - owned by retrieval pipeline)

```
key:   source_text:{sha256(url)}
value: {
  "url": "string",
  "fetched_at": "timestamp",
  "content_type": "string",
  "source_domain": "string",
  "full_text": "string - full parsed page text, UTF-8",
  "word_count": "integer",
  "retrieval_latency_ms": "integer"
}
TTL: minimum 120s (verified shared SLO with retrieval team); typically 10-60 minutes by source type
```

### `confidence_annotation` (answer payload enrichment - not persisted independently)

The in-memory object attached to each citation in the answer payload before streaming. Consumed by the browser renderer to produce UI signals.

```json
{
  "citation_index": "integer - 1-based citation number in the answer",
  "confidence_signal": "enum: verified | single_source | low_confidence | unverified_claim | unverified_paywall | unverified_js_blocked",
  "citation_confidence_score": "float",
  "source_passage_highlight": "string - the source_passage_used, for source panel highlight",
  "source_passage_char_offset": "integer",
  "replacement_applied": "boolean - true if the original citation URL was replaced",
  "replaced_source_url": "string | null",
  "cache_quality": "string - surfaced to UI for paywall and js_required states"
}
```

---

## 9) Instrumentation event schemas

### `citation_verification_completed`

Emitted once per answer after the full verification pass completes, immediately before streaming begins. This is the primary monitoring event for the verification pipeline.

```json
{
  "event": "citation_verification_completed",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "focus_mode": "string - academic | web | social | writing | video | math",
    "synthesis_model": "string - which LLM generated the answer",
    "total_citations": "integer",
    "verified_count": "integer",
    "single_source_count": "integer",
    "low_confidence_count": "integer",
    "unverified_count": "integer",
    "verification_pass_latency_ms": "integer - total gate overhead",
    "claim_extraction_latency_ms": "integer",
    "source_cache_fetch_latency_ms": "integer",
    "batch_inference_latency_ms": "integer",
    "signal_mapping_latency_ms": "integer",
    "replacement_search_latency_ms": "integer | null - null if no replacement was attempted",
    "verifier_model_version": "string",
    "replacement_source_found_count": "integer",
    "source_cache_miss_count": "integer",
    "paywall_blocked_count": "integer",
    "js_blocked_count": "integer",
    "cross_turn_source_reuse_count": "integer - number of claims where source_retrieval_turn_index != current turn",
    "user_plan": "string - free | pro | teams | enterprise",
    "verification_pass_status": "string - complete | degraded | skipped_no_citations | skipped_parse_failure",
    "circuit_breaker_tripped": "boolean"
  }
}
```

### `source_cache_miss_at_verification`

Emitted whenever a cited URL is not present in the retrieval cache at the time the verification gate runs. Used to monitor the retrieval team's shared cache SLO.

```json
{
  "event": "source_cache_miss_at_verification",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "source_url": "string",
    "source_domain": "string",
    "focus_mode": "string",
    "miss_reason": "string - not_in_cache | ttl_expired | js_render_empty | paywall_stub | content_too_short",
    "synthesis_latency_estimate_ms": "integer | null - elapsed time from retrieval start to verification gate; proxy for why cache may have expired"
  }
}
```

### `verifier_circuit_breaker_state_changed`

Emitted when the circuit breaker transitions between states. Primary alerting signal for on-call engineers.

```json
{
  "event": "verifier_circuit_breaker_state_changed",
  "properties": {
    "from_state": "string - closed | open | half_open",
    "to_state": "string - closed | open | half_open",
    "trigger_reason": "string - error_rate_exceeded | slow_call_threshold_exceeded | probe_success | probe_failure",
    "error_rate_in_window": "float - error rate that triggered the transition",
    "slow_call_rate_in_window": "float",
    "window_size_seconds": "integer",
    "affected_query_count": "integer - estimated queries that will enter degraded mode while breaker is open"
  }
}
```

### `citation_signal_rendered`

Emitted when a confidence signal is displayed to the user in the rendered answer.

```json
{
  "event": "citation_signal_rendered",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "citation_position": "integer - 1-indexed position in the answer",
    "signal_type": "string - verified | single_source | low_confidence | unverified_claim | unverified_paywall | unverified_js_blocked",
    "citation_confidence_score": "float",
    "source_domain": "string",
    "focus_mode": "string",
    "synthesis_model": "string",
    "replacement_applied": "boolean",
    "compound_claim": "boolean - true if claim was part of a multi-clause sentence",
    "gate_indicator_shown_ms": "integer - how long the verifying indicator was visible before answer rendered; 0 if gate completed before page load"
  }
}
```

### `citation_clicked_with_signal`

Emitted when a user clicks a citation that carries a confidence signal.

```json
{
  "event": "citation_clicked_with_signal",
  "properties": {
    "thread_id": "string",
    "citation_position": "integer",
    "confidence_signal_shown": "string - verified | low_confidence | unverified_claim | unverified_paywall",
    "time_since_answer_completed_ms": "integer",
    "user_plan": "string - free | pro | teams | enterprise",
    "focus_mode": "string",
    "passage_highlight_visible": "boolean",
    "replacement_applied": "boolean - true if a replacement source was used for this citation",
    "click_context": "string - badge_icon | source_panel_label | citation_superscript"
  }
}
```

### `citation_audit_panel_opened`

Emitted when an Enterprise or Teams user accesses the audit panel in Spaces.

```json
{
  "event": "citation_audit_panel_opened",
  "properties": {
    "space_id": "string",
    "thread_id": "string",
    "opener_plan": "string - teams | enterprise",
    "opener_role": "string - admin | member | it_reviewer",
    "total_verified_in_session": "integer",
    "total_low_confidence_in_session": "integer",
    "audit_export_triggered": "boolean"
  }
}
```

### `gate_indicator_dismissed`

**v3 addition (resolves Q6).** Emitted when the "Verifying citations..." gate indicator is dismissed by the answer being released. Used to measure how often and for how long the indicator is visible, and to detect cases where the indicator stalls (e.g., gateway timeout during partial load).

```json
{
  "event": "gate_indicator_dismissed",
  "properties": {
    "thread_id": "string",
    "answer_turn_index": "integer",
    "indicator_visible_ms": "integer - wall-clock time the indicator was displayed before the answer rendered",
    "focus_mode": "string",
    "dismiss_reason": "string - gate_complete | timeout_passthrough | page_unloaded | navigation",
    "connection_type": "string - 4g | 3g | 2g | wifi | unknown",
    "slow_connection": "boolean - true if navigator.connection.effectiveType is 2g or 3g"
  }
}
```

---

## 10) Core problem this architecture must solve

The verification gate must be a synchronous pre-stream blocker - not an asynchronous annotator. This single constraint drives every design decision in the system.

### Why the gate must be synchronous

If confidence signals were applied asynchronously after streaming begins, the following failure mode occurs: the user reads "Perplexity achieves 92.4% accuracy on all CASP14 targets [3]" in the first streamed token batch. A yellow `low_confidence` indicator appears on citation [3] two seconds later - after the user has already absorbed the overclaimed metric and begun composing a response to their colleague. The damage is done before the signal arrives.

A pre-stream gate prevents this. The user's first view of the answer includes the confidence signal on every citation. The signal changes the user's reading posture from "I assume this is correct" to "I know exactly which claims need independent review."

The cost of synchronous verification is the latency budget: 400ms for Academic Focus, 600ms for Web Focus. This is the fundamental trade-off: answer quality (trust, downstream behaviour change) at the cost of answer speed (time-to-first-token). The PRD establishes that for the knowledge-worker and academic-researcher segments, this trade is correct - they value citation accuracy over marginal streaming latency improvement.

### Why parallel batch inference is mandatory

A 4-citation Academic Focus answer has 4 (claim, source) pairs to verify. At 110ms P50 per claim with sequential inference, the gate takes 440ms P50 before any other overhead - already above the 400ms P95 SLO. Parallel batch inference (all 4 pairs in a single model call) targets 200ms P95 for the full batch, leaving 150ms+ for claim extraction, source cache fetch, signal mapping, and answer enrichment.

Batching is not an optimisation - it is the enabling condition for the synchronous pre-stream gate at any realistic citation count.

### Why the source cache dependency is an architectural constraint, not a preference

The verification layer cannot re-fetch source URLs live during the gate. A live HTTP fetch adds a minimum of 200-500ms per source, destroying the latency budget for any answer with more than one citation. The system is architecturally dependent on the retrieval pipeline's source cache being warm for all cited URLs at verification time.

This creates a critical dependency: the retrieval pipeline must cache source full-text with a TTL long enough to survive the LLM synthesis time (which can be 2-10 seconds for long answers). The shared SLO (120-second minimum TTL for all cited URLs) formalises this dependency as a monitored contract, not an assumption.

---

## 11) Data flow narratives

### Flow A: Happy path - Academic Focus query with all citations verified

```
1. User types: "What are the key findings from the AlphaFold2 Nature paper?"
2. Retrieval pipeline fetches 7 candidate sources; caches full text for each URL (TTL >= 120s).
   Source cache entries: source_text:{sha256(url_1)} through source_text:{sha256(url_7)}

3. LLM (Academic Focus model) generates answer draft:
   "AlphaFold2 achieved a median GDT score of 92.4 on CASP14 free modelling targets [1].
   The model uses an attention-based neural network architecture called Evoformer [1][2].
   DeepMind trained AlphaFold2 on sequences from UniProt and structural data from PDB [3]."

4. Claim Extraction Service parses the draft:
   claim_1: {"claim_text": "AlphaFold2 achieved a median GDT score of 92.4 on CASP14 free modelling targets", "source_url": url_1}
   claim_2: {"claim_text": "The model uses an attention-based neural network architecture called Evoformer", "source_url": url_1}
   claim_3: {"claim_text": "The model uses an attention-based neural network architecture called Evoformer", "source_url": url_2}
   claim_4: {"claim_text": "DeepMind trained AlphaFold2 on sequences from UniProt and structural data from PDB", "source_url": url_3}

5. Source Content Cache Interface fetches (parallel):
   -> url_1: 4,200-word paper abstract (cache hit; cache_quality: ok)
   -> url_2: 3,100-word methods section (cache hit; cache_quality: ok)
   -> url_3: 2,800-word supplementary dataset description (cache hit; cache_quality: ok)
   -> Each segmented into 512-token windows with 64-token overlap

6. Semantic Verifier Service receives one batch of 4 (claim, passages) pairs via gRPC.
   Batch inference completes in 178ms P50.
   Results:
   claim_1: citation_confidence_score: 0.91, source_passage_used: "...median GDT_TS score of 92.4 on free modelling..."
   claim_2 (url_1): citation_confidence_score: 0.88
   claim_3 (url_2): citation_confidence_score: 0.86
   claim_4: citation_confidence_score: 0.84

7. Confidence Signal Layer:
   claim_1: score 0.91; corroborating_sources = 1 -> single_source
   claim_2 + claim_3 (same claim, two sources, both >= 0.65): corroborating_sources = 2 -> verified
   claim_4: score 0.84; corroborating_sources = 1; "UniProt" and "PDB" appear verbatim in source
     -> exact_match_override_applied: true -> single_source

   Answer payload annotated with confidence_annotation objects.

8. Total gate time: 22ms (extraction) + 14ms (cache) + 178ms (inference) + 8ms (signals) = 222ms.
   Well within 400ms Academic Focus SLO.

9. Enriched answer streams to user. Source panel shows highlighted passage for the Evoformer claim.

10. Async: CitationAuditRecord published to Kafka topic -> written to Postgres.
    verified_count: 1, single_source_count: 2, low_confidence_count: 0
    overall_answer_accuracy_score: 0.88
```

---

### Flow B: Citation laundering caught - low-confidence signal prevents propagation

```
1. User types: "What percentage of startups fail within the first year?"
2. LLM generates: "Approximately 90% of startups fail within the first year of operation [2]."
   (Correct figure from the cited source: 20% fail in year 1; 90% is a lifetime figure.)

3. Claim Extraction Service extracts:
   claim_1: {"claim_text": "Approximately 90% of startups fail within the first year", "source_url": url_2}

4. Source cache hit: url_2 is a CB Insights startup failure report (cache_quality: ok).

5. Verifier runs: the report reads "About 20% of new businesses fail during the first year."
   No passage in url_2 supports "90% fail within the first year".
   citation_confidence_score: 0.28

6. Confidence Signal Layer:
   score 0.28 < 0.65 -> replacement source search triggered
   Ranking: domain_continuity check (url_4 same domain as url_2? No -> retrieval_relevance_score rank)
   Replacement candidates tried in order:
   -> url_4 (Forbes article): score 0.31 (mentions "90% fail" but in lifetime context)
   -> url_5 (BLS data): score 0.22
   -> No replacement scores >= 0.65 within 100ms budget
   replacement_source_found: false -> confidence_signal: low_confidence

7. Enriched answer streams with yellow indicator on citation [2].
   User clicks the yellow indicator; source panel: "Limited sources - The source only partially supports this claim"
   User reads the CB Insights report; sees the 20% figure; corrects the claim before sharing.

Result: Citation laundering prevented. System directed user attention to the exact citation needing review.
```

---

### Flow C: Degraded mode - verifier service unavailable (circuit breaker open)

```
1. Verifier service error rate exceeds 10% within a 60-second window.
   Circuit breaker transitions: closed -> open.
   verifier_circuit_breaker_state_changed event emitted.
   circuit_breaker_state: "open" written to Redis key (TTL: 120s).
   PagerDuty alert fires immediately.

2. Gate Coordinator polls circuit breaker state from Redis before each verification request.
   Reads "open" state -> activates degraded mode immediately (no gRPC call attempted).

3. Enriched answer payload released to stream WITHOUT confidence signals.
   verification_pass_status: degraded; circuit_breaker_tripped: true.
   No CitationAuditRecord written.

4. citation_verification_completed event emitted:
   {"verification_pass_status": "degraded", "circuit_breaker_tripped": true, ...}

5. After 30 seconds, circuit breaker transitions to half_open.
   1 in 100 requests sent to verifier as a probe.
   If 3 consecutive probes succeed: closed state restored; normal gate resumes.

Recovery: Degraded mode is automatically transient. Users see no error messages.
Normal verification resumes on the same gate coordinator instance without a deployment.
```

---

### Flow D: Paywall-blocked source - distinct signal prevents confusion

```
1. User asks: "What does the Science paper on CRISPR base editing report?"
   Retrieved source url_1 is a Science journal article behind a subscription paywall.
   Retrieval pipeline caches the stub: "<500 words; contains 'Sign in to read this article'".

2. Source Content Cache Interface detects: content_length < 500 words AND contains paywall signature.
   cache_quality: paywall_blocked returned for url_1; source_passages: null.

3. Confidence Signal Layer:
   cache_quality: paywall_blocked -> confidence_signal: unverified_paywall (not unverified_claim)

4. Source panel for citation [1]: "Source requires a subscription - could not verify this claim"
   This is distinct from "No citation grounding" (unverified_claim).
   User understands: the claim may be correct; the source is inaccessible to the verifier.
   User can click through and access the paper directly if they have a subscription.

Result: False "unverified_claim" label avoided for a likely-valid academic citation.
Academic Focus users in particular benefit from this distinction.
```

---

### Flow E: Multi-turn conversation - cross-turn source reuse tracked (v3 addition)

```
Context: User is in a multi-turn Academic Focus session on protein folding.

Turn 1: User asks "What is AlphaFold2?" -> Retrieval pipeline fetches 6 sources including url_A (Nature paper).
  url_A is cached: source_text:{sha256(url_A)} with TTL 60min.
  Turn 1 CitationAuditRecord created. All claims verified.

Turn 2 (8 minutes later): User asks "How does AlphaFold2 compare to RoseTTAFold?"
  LLM synthesises an answer citing url_A again for an AlphaFold2 claim.
  Retrieval pipeline does NOT re-fetch url_A; the Turn 1 cache entry is still warm.

Gate Coordinator for Turn 2:
  Source Content Cache Interface looks up url_A -> cache hit (TTL: 60min; 52 minutes remaining).
  Claim Extraction Service extracts the AlphaFold2 claim with citation to url_A.
  CitationVerificationResult for that claim:
    source_retrieval_turn_index: 1  (url_A was first cached during turn 1, not turn 2)
    citation_confidence_score: 0.87
    confidence_signal: single_source

  Turn 2 CitationAuditRecord:
    verification_results[0].source_retrieval_turn_index = 1
    -> Reviewers reading Turn 2 audit record see that the url_A source dates from Turn 1's retrieval.
       They can cross-reference Turn 1's audit record if they want to see the full source provenance.

cross_turn_source_reuse_count: 1 emitted in citation_verification_completed event.

Result: Enterprise reviewer opening the Turn 2 audit record understands which sources are fresh
vs. inherited from earlier in the session. The audit trail is complete without inspecting all prior turns.
```

---

## 12) Failure modes and runbooks

| Failure | Detection | Mitigation | User-visible behaviour | Runbook |
|---|---|---|---|---|
| Verifier service gRPC timeout (>350ms/550ms) | Gate Coordinator timeout fires | Activate degraded mode for this answer; emit `verification_pass_status: degraded` | Answer streams with no badges; no error message | Check gRPC latency in verifier service dashboard; review batch size distribution; check GPU utilisation |
| Verifier service error rate >10% (circuit breaker) | `verifier_circuit_breaker_state_changed` event; PagerDuty alert | Circuit breaker opens; all answers enter degraded mode; 30s recovery window | Answers stream without badges; no error message | Check verifier service pod health; inspect OOM events; check GPU cluster health; if sustained >5 min, escalate to infra lead |
| Source cache miss for cited URL | `source_cache_miss_at_verification` event with `miss_reason` | Assign `unverified_claim` or `unverified_paywall`/`unverified_js_blocked`; continue verification for other citations | Appropriate signal on affected citation; other citations still verified | If miss rate >5%, alert retrieval team; check TTL configuration and retrieval pipeline error rate |
| Claim extraction parse failure | Exception in Claim Extraction Service; `verification_pass_status: skipped_parse_failure` | Return empty claim list; answer streams without signals | No badges; answer streams normally | Check CES error logs; inspect answer draft format; look for LLM output format regression in synthesis model |
| Cross-encoder calibration drift | Weekly false-positive sampling > 12% | Alert ML Platform; re-run Platt scaling calibration; do not auto-rollback | Increased over-cautious `low_confidence` signals; addressed in next calibration cycle | Review calibration set composition; check for distribution shift in Academic Focus query mix; re-fit Platt parameters on fresh calibration set |
| `CitationAuditRecord` write failure | Dead-letter queue growth; reconciliation job gap alert | Retry queue (5 attempts, exponential backoff); dead-letter after exhaustion; alert on >0.1% daily gap | No user impact; audit record may be missing for the affected turn | Check Postgres write latency and connection pool; replay dead-letter queue after DB health confirmed |
| Replacement source search timeout (>100ms) | Internal timeout in Confidence Signal Layer | Assign `low_confidence` with original citation; `replacement_source_attempted: true`, `replacement_source_url: null` | `low_confidence` badge shown with original citation | Check verifier service P95 for single-pair calls; if elevated, adjust replacement budget from 100ms to 80ms to preserve headroom |
| Source cache TTL expiry (synthesis took longer than cache TTL) | `source_cache_miss_at_verification` with `miss_reason: ttl_expired` | Assign `unverified_claim` for the affected citation | No badge on affected citation | Alert retrieval team if TTL expiry miss rate >1%; confirm 120-second minimum TTL SLO is in place for all focus modes |
| Concurrent query spike - verifier service overload | Verifier service P95 latency alert >300ms; degraded mode activation rate rising | Auto-scale verifier service replicas (horizontal); circuit breaker activates if error rate tips above 10% | Degraded mode activated for queries during the spike | Check HPA (Horizontal Pod Autoscaler) configuration; confirm scale-out lag; if scale-out insufficient, increase minimum replica count for peak hours |
| Gate indicator visible for >3s (slow connection + long answer) | `gate_indicator_dismissed` event with `indicator_visible_ms > 3000` | UX: indicator shows progress animation; no additional system intervention | User sees animated "Verifying citations..." for up to 3s before answer | If >5% of Academic Focus sessions show indicator_visible_ms >3s, investigate gate budget compliance; consider progressive release (stream first paragraph before gate completes on very long answers) |
| Cross-region audit record replication lag >60s | Replication lag monitor alert | Alert data platform on-call; read API falls back to primary write replica for the affected region | Enterprise audit API may show slight delay in surfacing newly-written records | Check replication slot health on Postgres primary; review network throughput between regions; if sustained >5 min, route audit API reads to primary |

---

## 13) Competitive architecture comparison

Perplexity Verified Answers occupies a unique architectural position: it is the only major AI answer engine with a synchronous pre-stream citation verification gate. Understanding how competitors handle citation quality reveals both the moat and the exposure.

| Capability | Perplexity Verified Answers | Google AI Overviews | ChatGPT Search |
|---|---|---|---|
| Citation verification approach | Synchronous pre-stream cross-encoder gate on every cited claim | No per-claim semantic verification; citations are surfaced by the retrieval model but not checked against claim text | No per-claim semantic verification; citations selected by retrieval scoring but not verified against synthesized claims |
| Citation granularity | Per-claim (each factual assertion has its own confidence score) | Per-answer (source chips shown per answer block, not per claim) | Per-source (footnote numbers linked to sources; no claim-level attribution) |
| Confidence signal to user | Three-level signal: `verified` / `low_confidence` / `unverified_claim` | None; no confidence signal to user for individual citations | None |
| Audit trail | Full `CitationAuditRecord` per answer turn; accessible via API for Teams/Enterprise | None | None |
| Source re-use across session | Retrieval cache shared across turns in same session; `source_retrieval_turn_index` tracks origin | Retrieval cache within session; not exposed | Not published |
| Latency cost | +400ms P95 (Academic Focus); +600ms P95 (Web Focus) for verified gate | 0ms additional (no verification step) | 0ms additional |
| False-positive risk | 7.2% (cross-encoder MiniLM v1); mitigated by override rules and weekly sampling | Not applicable (no per-claim scoring) | Not applicable |
| Enterprise audit capability | Yes - per-answer audit record, JSON/PDF export, Audit API | No | No |

**Where Perplexity wins:**

- Citation laundering detection: Perplexity is the only system that catches the specific failure mode identified in the teardown - a real URL attached to a claim the page does not support. Google and ChatGPT can retrieve plausible sources but cannot detect the semantic mismatch between the synthesised claim and the source content.
- Enterprise auditability: The per-answer `CitationAuditRecord` with API access and export is unique. Google and ChatGPT have no equivalent. For professional and enterprise buyers evaluating AI research tools against each other, this is a category-differentiating capability.
- Knowledge worker trust: The `verified` badge changes the user's reading posture from verification-anxiety to trust. Google AI Overviews and ChatGPT Search have no citation quality signal; users must verify everything or nothing.

**Where Perplexity is exposed:**

- Latency cost: The verification gate adds 400-600ms P95 to answer delivery. Google and ChatGPT stream the first token faster. For casual queries where citation accuracy is not the primary value, users may perceive Perplexity as slower without understanding why.
- Source staleness: All three systems retrieve cached content; none verify that the cited source is current. Perplexity's verification gate checks semantic match, not temporal recency. A claim verified against a 6-month-old cached page may be factually stale.
- Web Focus false-positive risk: The cross-encoder accuracy gap between Academic Focus (structured scholarly sources) and Web Focus (thin content, dynamic pages, social media) means the `low_confidence` signal is less reliable on Web Focus queries. Google and ChatGPT do not expose a false-positive signal, so they cannot be criticised for false-positive rate - but Perplexity can if the `low_confidence` signal misfires visibly on common Web queries.

---

## 14) Scaling model

### Capacity sizing at Phase 1 (Academic Focus only)

| Metric | Estimate | Basis |
|---|---|---|
| Academic Focus queries per day | ~500K | Estimated from teardown's "dedicated academic focus segment"; ~5% of total Perplexity queries |
| Average citations per Academic Focus answer | 4.2 | Perplexity typically generates 4-8 citations per answer; Academic Focus skews lower (structured sources) |
| Claim-source pairs per day | ~2.1M | 500K queries x 4.2 citations |
| Cross-encoder inference calls per day | ~270K batch calls | 2.1M pairs / 8 pairs per batch |
| Peak QPS (5x daily average, over 8 peak hours) | ~47 batch calls/sec | (270K / 8h) x 5 peak multiplier / 3600 |
| Verifier service GPU requirement at 47 QPS | ~4 GPU instances | Assuming 12 batch calls/sec per GPU (200ms P95 per batch, ~6K calls/GPU/hour); 4x headroom over minimum |
| `CitationAuditRecord` writes per day (Academic Focus) | ~500K rows | One record per answer |
| Kafka throughput required | ~6 messages/sec average | 500K records/day; bursty during peak |
| Postgres write throughput | ~6 rows/sec average | One write per Kafka message; well within single Postgres instance capacity at Phase 1 |

### Capacity sizing at Phase 3 (all focus modes, all user tiers)

| Metric | Estimate | Basis |
|---|---|---|
| Total queries per day (Phase 3, full rollout) | ~10M | Perplexity growth trajectory from teardown; 10B queries/year (PRD Q7 economics) |
| Average citations per answer | 5.1 | Higher in Web Focus where retrieval pool is larger |
| Claim-source pairs per day | ~51M | 10M queries x 5.1 citations |
| Cross-encoder inference calls per day | ~6.4M batch calls | 51M pairs / 8 pairs per batch |
| Peak QPS (5x daily average, over 8 peak hours) | ~1,110 batch calls/sec | (6.4M / 8h) x 5 / 3600 |
| Verifier service GPU requirement at 1,110 QPS | ~93 GPU instances | Same 12 batch calls/sec per GPU assumption; round to 100 for 8% headroom |
| Annual GPU serving cost (Phase 2 model, electra-base at $0.31/1M claims) | ~$15.8M | 51M pairs/day x 365 x $0.31/1M |
| Annual GPU serving cost (Phase 3 model, perplexity-claim-verifier-v1 at $0.08/1M claims) | ~$4.1M | Same; validates the $11.7M/year savings from the proprietary model |
| `CitationAuditRecord` writes per day | ~10M rows | One record per answer; Pro and Teams/Enterprise retained for 90 days to 5 years |
| Audit record storage at Phase 3 (5-year retention, Teams/Enterprise only ~5% of queries) | ~9.1B rows over 5 years | 10M queries/day x 5% x 365 days x 5 years; at ~2KB per row = ~18TB |

### Auto-scaling design

The Semantic Verifier Service is deployed on Kubernetes with a Horizontal Pod Autoscaler (HPA) configured on:
- Primary metric: P90 batch inference latency (custom metric exported via Prometheus). Scale out trigger: P90 > 150ms for 2 consecutive minutes.
- Secondary metric: GPU utilisation > 75% for 3 consecutive minutes.
- Scale-in: P90 < 80ms and GPU utilisation < 40% for 10 consecutive minutes. Conservative scale-in prevents thrashing during bursty traffic.
- Minimum replicas: 4 (Academic Focus Phase 1); 10 (Phase 2 Web Focus); 20 (Phase 3 full rollout).
- Maximum replicas: 50 (Phase 1); 200 (Phase 3).
- Scale-out cooldown: 60 seconds. Scale-in cooldown: 300 seconds.

---

## 15) Architectural trade-offs

### Trade-off 1: Pre-stream gate vs. async annotation

**Option A (chosen):** Verification is a synchronous gate. The answer does not stream until the full verification pass is complete and confidence signals are attached to the payload.

**Option B (rejected):** Verification runs in parallel with streaming. Confidence signals are injected into the rendered UI as they arrive, while the answer text is already visible to the user.

**Why A wins for the target segment:** Knowledge workers and academic researchers use verified citations to make high-stakes decisions - pasting into decks, citing in papers, forwarding to colleagues. If the signal arrives after the claim is already read and processed, the user has formed an initial trust judgment. Post-hoc signal revision is cognitively more disruptive than pre-formed confidence from the start. For this segment, 400ms of additional latency is acceptable; a confidence signal that feels like a footnote rather than a guarantee is not.

**Cost of A:** Time-to-first-token is 400ms (Academic Focus) to 600ms (Web Focus) slower than baseline. A "verifying citations..." indicator during the gate delay is the required UX compensation (see Section 16, Q6 resolution for full UX spec).

---

### Trade-off 2: Dependency on retrieval cache vs. live source re-fetch

**Option A (chosen):** The verification layer reads only from the retrieval pipeline's existing source cache. It never fetches source URLs live.

**Option B (rejected):** The verification layer fetches the latest version of each cited source URL live during the gate.

**Why A wins:** A live re-fetch adds a minimum of 200-800ms per unique source URL. For an answer with 4 citations from 4 domains, Option B adds 800ms-3.2s to the gate. This blows every latency SLO in the NFR table.

**Cost of A:** Verification is checked against source content as it was at retrieval time. Rapidly updating sources (news, live sports, market data) may be stale. This is a documented non-goal in the PRD.

---

### Trade-off 3: Cross-encoder scoring vs. bi-encoder scoring

**Option A (chosen):** Cross-encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`). Accuracy: 88.9%. False-positive rate: 7.2%.

**Option B (rejected):** Bi-encoder (embedding cosine similarity). Accuracy: 81-85%. False-positive rate: 10-14%.

**Why A wins:** The bi-encoder's false-positive rate (10-14%) is above the 8% threshold. A system that marks 10-14% of correct claims as `low_confidence` destroys user trust in the signal faster than it builds it.

**Cost of A:** Higher cost per claim ($0.18/1M vs. $0.04-0.09/1M) and higher latency per claim. Both are acceptable: cost is economically negligible, and latency is addressed through parallel batching.

---

### Trade-off 4: Confidence signal as pre-stream payload enrichment vs. separate UI API call

**Option A (chosen):** Confidence signals are embedded in the answer payload before streaming begins.

**Option B (rejected):** The answer streams first; the browser makes a separate API call for confidence signals after rendering.

**Why A wins:** Option B produces the same "signals arrive after reading" failure as async annotation, plus a second API round trip (20-100ms browser network latency) that creates a visible badge "pop-in" effect.

**Cost of A:** All client platforms (web, iOS, Android) must support the `confidence_annotation` payload format before any user sees confidence signals. This increases implementation coordination surface area.

---

### Trade-off 5: Replacement source selection as in-gate operation vs. deferred to re-query

**Option A (chosen):** When a citation scores `low_confidence`, the gate attempts to find a replacement source from the already-retrieved pool within the 100ms replacement time budget.

**Option B (rejected):** Display the `low_confidence` signal and offer a "find a better source" button triggering a targeted re-retrieval query.

**Why A wins:** Option B adds user friction for a failure mode the system can often self-correct. A replacement source in the already-retrieved pool costs <100ms and no additional retrieval call.

**Cost of A:** Silent source replacement means the user sees a different citation URL than the LLM originally wrote. The original LLM draft cited source A; the rendered answer cites source B. This is transparent in the audit record (`replacement_applied: true`) and in the source panel passage highlight, but not visible in the answer text.

---

## 16) Open questions - all resolved (v3)

**Q1 (compound claim parsing) - RESOLVED in v2**

Decision: The Claim Extraction Service splits compound claims at coordinating conjunction boundaries and applies the minimum score rule across clauses from the same source. See Layer 1 detail above.

---

**Q2 (JavaScript-rendered and paywalled source handling) - RESOLVED in v2**

Decision: Introduce `cache_quality` enum with `paywall_blocked` and `js_required` states. The Confidence Signal Layer maps these to distinct signal states (`unverified_paywall`, `unverified_js_blocked`) with distinct source panel copy. This avoids the false `unverified_claim` label for sources that are inaccessible rather than non-supportive. See Layer 2 detail and Flow D above.

---

**Q3 (source cache TTL contract) - RESOLVED in v2**

Decision: Formalise a shared SLO between the retrieval team and the verification team: minimum 120-second TTL for all cited URLs, measured from first cache time at retrieval. Monitored via `source_cache_miss_at_verification` event with `miss_reason: ttl_expired`. See Layer 2 detail above.

---

**Q4 (multi-turn conversation handling for audit records) - RESOLVED in v3**

**Decision:** Add `source_retrieval_turn_index` (nullable integer) to `CitationVerificationResult`. When a follow-up answer cites a source URL that was first cached during a prior turn in the same thread, the Gate Coordinator populates `source_retrieval_turn_index` with the original turn's index. The Gate Coordinator determines this by looking up the source URL in a lightweight per-thread source provenance map held in memory during the session (key: sha256(url) -> first_turn_index; the map is initialised from the retrieval pipeline's cache metadata and is cleared when the session expires).

This resolves the reviewer concern: a Turn 2 audit record is now self-contained for all claims in Turn 2, but reviewers can also see which sources are inherited from prior turns without inspecting the Turn 1 record separately. The `cross_turn_source_reuse_count` field in `citation_verification_completed` provides an aggregate signal for monitoring how often cross-turn reuse occurs (expected: high for deep research sessions; near zero for one-shot queries).

**What this does NOT do:** It does not store a back-reference to the Turn 1 audit record. The `source_retrieval_turn_index` integer is sufficient for a human reviewer to navigate; a formal record-to-record link would add a graph traversal cost to every audit read that is not justified by the use case. Enterprise reviewers needing full cross-turn provenance can issue a GET request for the thread's full audit record set (`GET /v1/audit/threads/{thread_id}` without `turn_index` filter) to reconstruct the provenance chain.

See Flow E above for a concrete example. See the updated `CitationVerificationResult` schema in Section 8 for the field definition.

---

**Q5 (replacement source selection ranking) - RESOLVED in v2**

Decision: Ranked by domain continuity first, then retrieval pipeline relevance score. See Layer 4 detail above.

---

**Q6 (confidence signal display during streaming for long answers) - RESOLVED in v3**

**Decision:** The pre-stream gate is an atomic hold: the entire enriched answer payload is released in a single event, not as a progressive token stream. This means the "verifying citations..." indicator is shown from the moment the LLM synthesis completes until the Gate Coordinator releases the payload. The UX specification is as follows:

**Indicator appearance:**
- Component: A pill-shaped banner below the Perplexity search bar (not overlaying the answer area).
- Copy: "Verifying citations..." with a three-dot pulse animation.
- The indicator appears at the same moment the synthesis completes (before gate processing starts), not after gate processing starts. This means users see it for 200-600ms for typical answers, not just the gate overhead itself - it appears at the start of the wait, so users understand why streaming has not started.
- The indicator is dismissed atomically when the enriched payload arrives. The answer renders all at once with badges visible from the first render frame.
- On slow connections (2G/3G, detected via `navigator.connection.effectiveType`), the copy changes to "Verifying citations - this takes a moment on slower connections..." to set expectations. The gate timeout thresholds are NOT adjusted for slow connections; the gate must still complete within its SLO or degrade to passthrough.

**Edge case - gate exceeds 3 seconds visible (network issue or extreme synthesis time):**
- If `gate_indicator_dismissed` is not fired within 3 seconds of the indicator appearing, the gate coordinator must either: (a) if the verifier has returned but the network round-trip to the browser has stalled, degrade to passthrough and stream the unverified answer; or (b) if the verifier is still running, the existing circuit breaker and timeout logic handles this case (degraded mode activates at 350ms/550ms hard timeout - this edge case cannot occur in normal gate operation). The 3-second indicator threshold is a browser-side failsafe only.
- If `dismiss_reason: timeout_passthrough` appears in `gate_indicator_dismissed` event above 0.1% rate for any 1-hour window, alert on-call backend engineer.

**Long answer UX (>800 words, Academic Focus):**
- Long answers (>800 words) take longer to synthesize (8-20 seconds of LLM generation). The gate only starts after synthesis completes. The synthesis streaming time is NOT part of the gate latency budget; the user sees the query being processed during synthesis. The gate indicator only appears after synthesis is done, for the gate overhead specifically.
- This means for very long answers, the user flow is: [typing indicator during synthesis, 8-20s] -> [brief "Verifying citations..." indicator, 200-600ms] -> [full answer with badges].
- The Core Experience team confirmed in design review that this flow is acceptable: the gate indicator is brief relative to synthesis time, and appearing at the very end of a long synthesis feels natural, not intrusive.

**Renderer implementation requirement:** The browser renderer must handle the full enriched answer payload as an atomic render event (not a token-by-token stream). This requires a coordination handshake: the synthesis streaming pipeline generates tokens locally (for latency perception), but the renderer holds them in a client-side buffer until the Gate Coordinator signals payload release. The buffer is cleared and the answer is rendered in full with badges on the release signal. This renderer architecture change is a Phase 1 prerequisite (listed in the launch readiness checklist in Section 18).

**Instrumentation:** The `gate_indicator_shown_ms` field in `citation_signal_rendered` and the new `gate_indicator_dismissed` event (see Section 9) provide full observability over indicator visibility time, dismiss reason, and connection type.

---

## 17) Multi-region deployment topology

At Phase 3 scale, the verification gate and audit persistence layer are deployed across at minimum two regions to provide availability and latency parity for Perplexity's global user base. The topology below shows the primary (us-east-1) and secondary (eu-west-1) deployment, reflecting Perplexity's primary user concentration in North America and Europe.

```mermaid
flowchart TD
    subgraph Global["Global Layer"]
        CDN[CDN + API Gateway\nGeoDNS routing\nLatency-based failover]
    end

    subgraph USEast["Primary Region - us-east-1"]
        direction TB
        GC_US[Gate Coordinator\nfleet - us-east-1]
        CES_US[Claim Extraction Service\nus-east-1]
        SCCI_US[Source Cache Interface\nus-east-1]
        SVS_US[Semantic Verifier Service\nGPU cluster - us-east-1\n4-20 A100 instances]
        CSL_US[Confidence Signal Layer\nus-east-1]
        REDIS_US[(Redis Cluster\nus-east-1\nSource cache + circuit breaker state)]
        KAFKA_US[Kafka cluster\nus-east-1\ncitation-audit-records topic]
        PG_US[(Postgres Primary\nus-east-1\nCitationAuditRecord write path)]
        PG_READ_US[(Postgres Read Replica\nus-east-1\nAudit API reads)]
    end

    subgraph EUWest["Secondary Region - eu-west-1"]
        direction TB
        GC_EU[Gate Coordinator\nfleet - eu-west-1]
        CES_EU[Claim Extraction Service\neu-west-1]
        SCCI_EU[Source Cache Interface\neu-west-1]
        SVS_EU[Semantic Verifier Service\nGPU cluster - eu-west-1\n4-10 A100 instances]
        CSL_EU[Confidence Signal Layer\neu-west-1]
        REDIS_EU[(Redis Cluster\neu-west-1\nSource cache + circuit breaker state)]
        KAFKA_EU[Kafka cluster\neu-west-1\ncitation-audit-records topic]
        PG_EU[(Postgres Read Replica\neu-west-1\nAudit API reads; async replication from us-east-1)]
    end

    CDN -->|North America + APAC queries| GC_US
    CDN -->|Europe queries| GC_EU

    GC_US --> CES_US
    CES_US --> SCCI_US
    SCCI_US --> REDIS_US
    SCCI_US --> SVS_US
    SVS_US --> CSL_US
    CSL_US --> KAFKA_US
    KAFKA_US -->|async write| PG_US
    PG_US -->|async replication < 30s P99| PG_EU

    GC_EU --> CES_EU
    CES_EU --> SCCI_EU
    SCCI_EU --> REDIS_EU
    SCCI_EU --> SVS_EU
    SVS_EU --> CSL_EU
    CSL_EU --> KAFKA_EU
    KAFKA_EU -->|async cross-region write| PG_US

    PG_US --> PG_READ_US
    PG_EU --> PG_EU

    note1["Audit writes always go to us-east-1 Postgres primary\nRegional Kafka topics forward to primary\nAudit reads served from local replica per region"]
```

**Key topology decisions:**

| Decision | Rationale |
|---|---|
| Verifier GPU fleet is region-local | Sending claim-passage batches cross-region adds 30-80ms latency on the gRPC call - this blows the 350ms gate budget. Each region runs its own GPU cluster. |
| Source cache is region-local (Redis) | Same latency argument; cache reads must be <15ms P95. Cross-region cache reads average 30-60ms. |
| Audit writes fan into a single Postgres primary (us-east-1) | Simplifies cross-turn audit record consistency for multi-turn sessions that may span regions within the same session. Kafka cross-region fanout is async; write latency is not on the user-facing path. |
| Circuit breaker state in regional Redis | Each region's circuit breaker operates independently. A verifier outage in eu-west-1 does not affect us-east-1. This is correct: the breaker is a per-region safeguard, not a global kill switch. |
| Audit API reads from regional read replicas | Reduces cross-region read latency for EU enterprise users. Async replication lag (<30s P99) is acceptable for audit reads; enterprise users are not querying audit records in real time. |

---

## 18) Phased rollout plan

### Phase 1 - Academic Focus verification with A/B signals (Weeks 1-16)

**Engineering scope:**
- Claim Extraction Service deployed to 100% of Academic Focus query path.
- Semantic Verifier Service (`MiniLM` + Platt scaling) deployed in us-east-1; eu-west-1 deployment at week 8 after us-east-1 stability confirmed.
- `CitationAuditRecord` write path live for all Academic Focus answers.
- Confidence signals rendered to 50% treatment arm (A/B split by user hash, not session hash; consistent per user across sessions).
- Renderer buffer hold (Q6 resolution) implemented for Academic Focus stream.
- Gate indicator UX live for treatment arm.

**Launch gate (all criteria must pass before Phase 1 goes live):**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| Verifier model false-positive rate | <8% on 500-example labelled test set | ML release evaluation |
| Verification gate P95 latency (Academic Focus) | <400ms on synthetic load test at 200 concurrent QPS | Load test result |
| Degraded-mode chaos test passed | Answers stream without error when verifier service is at 100% failure rate in staging | Staging chaos test result |
| `CitationAuditRecord` write path load tested | Handles 30M writes/day without write latency regression | Storage load test |
| Platt scaling ECE | <0.05 on 2,000-example holdout set | ML calibration report |
| Gate indicator UX implemented | Renderer buffer hold atomic release confirmed across Chrome, Safari, Firefox, iOS, Android | Cross-platform QA sign-off |
| `gate_indicator_dismissed` event instrumented | Event fires on 100% of Academic Focus answers where gate ran; `dismiss_reason` field populated correctly | Staging instrumentation validation |
| `source_retrieval_turn_index` populated | Correct turn index populated for 100% of cross-turn source reuse cases in test sessions | Automated integration test on 50 multi-turn sessions |
| Privacy review complete | `CitationAuditRecord` retention policy approved | Signed review document |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Academic Focus answer P95 latency (time-to-first-token) >3.5s for any 7-day rolling window | Pause rollout; optimize verifier batch parallelism before resuming |
| Pro churn rate for treatment cohort exceeds control by >0.5 pp at day 14 | Pause signals-visible arm; investigate whether `low_confidence` labels are causing trust damage; do not pause signals-hidden arm |
| Verifier service error rate >1% of queries for any 1-hour window | Auto-activate degraded mode (no signals shown); page on-call ML Platform engineer |
| `gate_indicator_dismissed` with `dismiss_reason: timeout_passthrough` >0.1% for any 1-hour window | Investigate gate coordinator timeout path; confirm circuit breaker and hard-timeout are functioning |
| Citation accuracy improvement vs. baseline <3 pp at day 42 | Evaluate verifier model retraining before expanding to Web Focus |

**Owner:** ML Platform (model serving) + Core Experience (UI + renderer buffer) + Backend Platform (audit persistence) + Analytics (A/B instrumentation).

---

### Phase 2 - Web Focus Pro + enterprise audit trail (Weeks 17-28)

**Engineering scope:**
- Verification extended to Web Focus Pro users (50% A/B split).
- `electra-base` model deployed for Web Focus; `MiniLM` remains for Academic Focus.
- Platt scaling recalibrated on Web Focus-specific 2,000-example calibration set.
- Audit API (`GET /v1/audit/threads/{thread_id}`) live for Teams and Enterprise keys.
- Spaces citation audit panel UI live for Teams and Enterprise accounts.
- Audit export (JSON + PDF) live.
- eu-west-1 GPU cluster scaled to Phase 2 Web Focus capacity (minimum 10 replicas).
- Labelling project: 10,000-example batch from live corpus; annotation tooling live.

**Launch gate:**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| Phase 1 A/B result: Academic Focus citation click rate lift | >+3 pp at 95% significance | Analytics A/B report |
| Phase 1 A/B result: Pro churn guardrail | Treatment cohort churn not >0.5 pp above control | Analytics cohort report |
| `electra-base` false-positive rate on Web Focus test set | <6% on 200-example Web Focus labelled set | ML release evaluation |
| Web Focus gate P95 latency | <600ms at 400 concurrent QPS in load test | Load test result |
| Audit API rate limiting enforced | 100 req/min/account tested with token-bucket verification | API contract test |
| PDF export generation latency | <60s for 100-answer sessions in staging | Staging performance test |
| `methodology_disclaimer` copy approved by Legal | Signed legal review | Legal sign-off document |
| Inter-annotator agreement gate | Cohen's kappa >= 0.82 on 500-example labelling spot check | Annotation quality report |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Web Focus Pro answer P95 latency >4s for any 7-day rolling window | Pause Web Focus signals; fall back to `MiniLM` for Web Focus while `electra-base` serving is investigated |
| Web Focus Pro churn rate in treatment cohort exceeds control by >0.5 pp at day 14 | Pause Web Focus signals-visible arm; root cause analysis before resuming |
| Audit export triggers any privacy incident (unauthorized data access; export scoping bug) | Disable export endpoint immediately; incident response runbook activation |
| Web Focus citation accuracy improvement <4 pp vs. baseline at day 42 | Hold Phase 3 until improvement reaches 4 pp; evaluate `electra-base` recalibration |

**Owner:** ML Platform (electra-base + Platt recalibration + labelling infra) + Core Experience (Web Focus UI) + Backend Platform (audit API + Spaces panel) + Spaces team (audit panel UI) + Legal (disclaimer review) + Analytics (Web Focus A/B).

---

### Phase 3 - Full rollout + proprietary model + enterprise GA (Weeks 29-42)

**Engineering scope:**
- Verification rolled to 100% of Web Focus free-tier users.
- `perplexity-claim-verifier-v1` promoted to Web Focus production if model quality and cost gates pass; `electra-base` remains as hot fallback (switchable within 5 minutes).
- Enterprise audit panel and API promoted from beta to GA.
- Competitive accuracy benchmark published as a public research blog post.
- Canary deployment to 5% of free-tier Web Focus users at week 29 before full rollout.
- Monthly accuracy evaluation job scheduled for `perplexity-claim-verifier-v1`.

**Launch gate:**

| Gate criterion | Threshold | Evidence source |
|---|---|---|
| `perplexity-claim-verifier-v1` model accuracy gate | >93% accuracy AND <4% false-positive rate on held-out Perplexity test set | ML release evaluation |
| `perplexity-claim-verifier-v1` serving cost gate | <$0.12/1M claims at P50 on production-equivalent infra | Infrastructure cost measurement |
| Phase 2 Web Focus Pro A/B result: citation click rate | >+3 pp at 95% significance | Analytics A/B report |
| Free-tier `low_confidence` rate projection | <18% per answer for representative free-tier query mix | ML Platform projection using Phase 2 data |
| `electra-base` hot fallback confirmed | Rollback to `electra-base` deployable within 5 minutes | Deployment runbook tested in staging |
| Enterprise GA QA | Zero critical bugs in audit panel and export across 5 enterprise customer profiles | Enterprise QA sign-off |
| Competitive benchmark publication cleared | Methodology cleared by Legal and Trust/Safety | Legal sign-off document |

**Kill-switch trigger conditions:**

| Condition | Action |
|---|---|
| Free-tier `low_confidence` signal rate >25% per answer in first 7 days | Raise `low_confidence` threshold from 0.65 to 0.70 for free tier; monitor for 7 days before re-evaluating |
| `perplexity-claim-verifier-v1` accuracy degrades >2 pp vs. Phase 3 launch baseline at monthly evaluation | Revert to `electra-base` within 5 minutes; open model drift incident |
| `gate_indicator_dismissed` with `dismiss_reason: timeout_passthrough` >0.5% for any 1-hour window during free-tier rollout | Investigate gate coordinator under free-tier query mix; free-tier queries may have longer synthesis times |
| Competitive benchmark publication causes a negative PR event | Pause publication; full audit of the 500-query set before re-publishing |

**Owner:** ML Platform (proprietary model + free-tier rollout) + Core Experience (free-tier UI) + Enterprise team (GA launch) + Marketing/Research (benchmark publication) + Analytics (rollout postmortem).

---

## 19) Experiment backlog

| Experiment | Hypothesis | Primary metric | Acceptance criteria | Guardrail | Kill-switch | Rollout phase | Owner | End-state decision |
|---|---|---|---|---|---|---|---|---|
| Renderer buffer hold vs. progressive badge injection | Atomic gate-release (current design) vs. streaming tokens first with badges added as each citation is verified provides better P95 answer delivery perception | `gate_indicator_dismissed.indicator_visible_ms` median; post-answer satisfaction signal (not instrumentable directly - proxy: D7 return rate) | Atomic release does not produce lower D7 return than progressive injection (non-inferiority test, delta 0.5 pp, n=2,000 sessions per variant) | Academic Focus answer latency P95 must not increase in either variant | If D7 return rate drops >1 pp in atomic release arm: move to progressive injection; accept the cognitive cost of late badge arrival for <1pp return rate improvement | Phase 1 (shadow; no user-visible difference between variants) | Core Experience + Analytics | If atomic release is non-inferior: keep; it is simpler to implement. If inferior: plan progressive injection for Phase 2. |
| `verified` badge colour saturation (high-contrast green vs. muted green) | Muted green badge is less alarming and produces higher `citation_clicked_with_signal` rate on `verified` citations vs. high-contrast green (which may feel like a traffic light warning) | `citation_clicked_with_signal` rate for `signal_type: verified` | Muted variant achieves click rate within -2 pp of high-contrast; no significant difference in Pro conversion | WCAG 2.1 AA contrast ratio must be maintained for both variants | None - UI only; no data integrity risk | Phase 1 (embedded in A/B) | Core Experience + Design | Adopt whichever produces higher `verified` citation click rate; update design system tokens |
| Gate indicator copy A/B ("Verifying citations..." vs. "Checking your sources...") | "Checking your sources..." copy is more conversational and reduces user anxiety during the gate delay | `gate_indicator_dismissed.indicator_visible_ms` (proxy for user aborting); D7 return rate | Neither variant shows >0.5 pp lower D7 return; "Checking your sources..." is preferred by qualitative survey (optional) | N/A | If either variant shows D7 return regression >0.5 pp: revert to current copy | Phase 1 | Core Experience + Copy | Adopt winning copy globally across all gate indicator contexts |
| `source_retrieval_turn_index` surfaced in audit panel | Showing "Retrieved in turn X" next to each citation in the Spaces audit panel helps enterprise reviewers understand source freshness per turn | `citation_audit_panel_opened` session duration (proxy for reviewer engagement with richer data); `audit_export_triggered` rate | Teams accounts that see the turn index show >10% higher audit export rate vs. control | Audit panel load time must not increase >100ms | If load time increases >200ms: lazy-load the turn index field after initial panel render | Phase 2 (Teams beta) | Enterprise team + Spaces team | If export rate lift confirmed: ship to GA; if not: remove the field from the audit panel UI (keep in API response) |
| Monthly verifier model accuracy evaluation - automated re-calibration trigger | Automatically re-fitting Platt scaling when ECE drifts above 0.07 (before the manual 12% false-positive alert fires) produces lower false-positive rate drift between manual review cycles | ECE on held-out calibration set (monthly re-evaluation) | Automated re-calibration keeps ECE <0.06 at 30-day interval; false-positive rate sampled weekly stays <10% | Re-calibration must not trigger during peak traffic hours (08:00-22:00 UTC); must complete and deploy within 4 hours of trigger | If automated re-calibration produces a false-positive rate regression (>10%) in post-deployment monitoring: roll back Platt parameters to the previous version; open incident | Phase 1 onward | ML Platform | If effective: make automated Platt recalibration part of the monthly model evaluation job; log every re-calibration event |
| Cross-region verifier model version sync | Deploying model updates simultaneously to both regions (vs. staged us-east-1 then eu-west-1) produces lower false-positive rate variance between regions during the promotion window | False-positive rate delta between regions in the 72-hour window after a model version promotion | Delta <1 pp between regions during the promotion window | Cross-region GPU capacity must be adequate for simultaneous promotion canary (10% traffic in each region during canary) | If any region's false-positive rate exceeds 12% during simultaneous promotion: roll back both regions simultaneously | Phase 2 (two-region deployment active) | ML Platform + Infra | If simultaneous promotion is safe: adopt as standard; reduces audit record inconsistency during promotion windows |
| `perplexity-claim-verifier-v1` shadow evaluation vs. `electra-base` (Phase 3 pre-promotion) | Proprietary model achieves >93% accuracy and <4% false-positive rate on live Perplexity query distribution, validating the Phase 3 model promotion criteria | Accuracy and false-positive rate on held-out Perplexity test set; shadow mode comparison on 5% of live Web Focus queries (results not shown to users) | >93% accuracy; <4% false-positive; <$0.12/1M claims; no statistically significant accuracy regression vs. `electra-base` on any Focus mode subcategory | `electra-base` must remain the production model during shadow evaluation; no user sees proprietary model results | If proprietary model shows accuracy regression on any Focus mode: extend training with additional data from that subcategory; do not promote | Phase 3 (pre-promotion gate) | ML Platform | If all criteria met: promote to production for Web Focus; `MiniLM` remains for Academic Focus until Phase 3 month 2 evaluation |

---

## 20) Operational runbook index

The following runbooks are required before Phase 1 launch. Each runbook must be authored, reviewed, and linked from the on-call dashboard. Runbook IDs are for reference in incident reports.

| Runbook ID | Title | Scope | Owner | Phase required |
|---|---|---|---|---|
| `RB-VER-01` | Degraded mode activation and manual override | How to manually activate and deactivate degraded mode (passthrough) for the verification gate across all gate coordinator instances; use when the circuit breaker does not self-recover | ML Platform | Phase 1 |
| `RB-VER-02` | Circuit breaker recovery procedure | Steps to diagnose and recover a stuck circuit breaker (open state persisting >15 minutes); includes Redis key inspection, verifier service health checks, and half-open probe forcing | ML Platform + Backend Platform | Phase 1 |
| `RB-VER-03` | Verifier model rollback | How to roll back the Semantic Verifier Service to the previous model version (e.g., `platt-v1` -> `platt-v0`); covers updating the model pointer, restarting pods, and confirming ECE on the rolled-back version | ML Platform | Phase 1 |
| `RB-VER-04` | Audit record reconciliation and dead-letter replay | Steps to investigate and resolve `CitationAuditRecord` write failures; covers dead-letter queue inspection, manual replay, and reconciliation job re-run | Backend Platform | Phase 1 |
| `RB-VER-05` | Source cache TTL regression response | Steps to investigate and resolve a source cache miss rate spike; covers retrieval team escalation path, TTL configuration check, and temporary threshold adjustment for `unverified_claim` labelling | Backend Platform + Retrieval team | Phase 1 |
| `RB-VER-06` | Verifier service GPU capacity emergency scale-out | Steps to manually trigger emergency GPU scale-out for the Semantic Verifier Service when the HPA has not responded to a traffic spike within 3 minutes; includes minimum replica override and cloud provider console steps | ML Platform + Infra | Phase 1 |
| `RB-VER-07` | Audit API incident - unauthorized data access | Steps to disable the audit API endpoint, rotate affected API keys, and initiate privacy incident response if an unauthorized cross-account data access is detected in the audit API | Backend Platform + Privacy/Legal | Phase 2 |
| `RB-VER-08` | Audit export disable and scope correction | Steps to disable the export endpoint, investigate a scope bug (export returning threads outside the requesting account's Space), and re-enable after the fix is confirmed | Backend Platform + Privacy/Legal | Phase 2 |
| `RB-VER-09` | Cross-region verifier service divergence | Steps to resolve model version divergence between us-east-1 and eu-west-1 (e.g., after a failed simultaneous promotion); covers detecting divergence via per-region `verifier_model_version` metric, rolling back the diverged region, and confirming parity | ML Platform + Infra | Phase 2 |
| `RB-VER-10` | `perplexity-claim-verifier-v1` production rollback | Steps to roll back from the proprietary model to `electra-base` within 5 minutes; covers model pointer update, pod restart, and accuracy health-check confirmation post-rollback | ML Platform | Phase 3 |
| `RB-VER-11` | Free-tier `low_confidence` rate spike response | Steps to raise the `low_confidence` threshold for free-tier queries (from 0.65 to 0.70) without a full deployment; covers feature flag update, monitoring confirmation, and criteria for returning to the 0.65 threshold | ML Platform + Core Experience | Phase 3 |
| `RB-VER-12` | Competitive benchmark retraction procedure | Steps to retract or correct the published competitive accuracy benchmark post; covers Legal and Communications coordination, blog post unpublishing, and methodology review timeline | Marketing/Research + Legal | Phase 3 |

---

*All latency figures are directional estimates based on publicly available cross-encoder model benchmarks and industry Redis/Postgres performance profiles - not internal Perplexity data.*
