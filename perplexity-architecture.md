# Perplexity Verified Answers - System Architecture

**What this explains:** The system architecture that powers Perplexity Verified Answers - a claim-level citation grounding layer that verifies the semantic relationship between each LLM-generated claim and its cited source before the answer streams to the user. Specifically: how Perplexity extracts structured claim-citation pairs from a raw LLM draft, runs parallel cross-encoder scoring against cached source content, maps calibrated confidence scores to UI signals, replaces weak citations where possible, and persists a per-answer audit record for enterprise accounts - all within a 400ms P95 pre-stream gate.

**PRD reference:** https://github.com/004mayank/product-prd/blob/main/perplexity-prd.md

**Version:** v1 - Initial system design
**Changes from v0:** N/A - initial version

---

## Version history

| Version | Key additions |
|---|---|
| v1 | Core problem statement, five system layers, data flow narrative, core data model, failure modes, architectural trade-offs, open questions |

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

---

## 3) System architecture overview

The verification gate sits between the synthesis stage and the streaming stage. This position is non-negotiable: the gate must receive the complete answer draft, run all verifications, and release the enriched payload as a unit. It cannot interleave with token streaming.

The Claim Extraction Service is co-located with the synthesis stage output handler - it operates on the completed draft buffer, not on the streaming token channel. The Semantic Verifier Service is a separate microservice, independently scaled, that the verification gate calls over an internal RPC connection with a hard timeout of 500ms (Academic Focus) or 700ms (Web Focus), providing 100ms of headroom against the user-facing SLO.

The Source Content Cache Interface is a thin client around the retrieval pipeline's existing cache (Redis or equivalent key-value store). The verification layer does not own or populate this cache; it reads from it. A cache miss on a cited URL means verification cannot proceed for that citation - the claim is marked `unverified_claim` rather than triggering a live re-fetch, which would exceed the latency budget.

The Confidence Signal Layer runs on the verification gate coordinator after all batch inference results are returned. It applies score thresholds, override rules (exact match, source format), and replacement source selection. It produces the final `confidence_signal` enum for each citation and annotates the answer payload.

The Audit Persistence Layer writes the `CitationAuditRecord` asynchronously via a write queue, after the enriched answer has been released to the stream. The audit write is fire-and-forget from the perspective of the latency-critical path. Write failures are queued for retry; reconciliation jobs detect missing records.

```
Query
  |
  v
[Retrieval Pipeline] -----> [Source Content Cache]
  |                                  |
  v                                  |
[LLM Synthesis]                      |
  |                                  |
  v                                  |
[Verification Gate Coordinator]       |
  |                                  |
  +----> [Claim Extraction Service]  |
  |         |                        |
  |         v                        |
  |      claim-citation pairs        |
  |         |                        |
  +----> [Source Content Cache Interface] <---+
  |         |
  |      source passages per claim
  |         |
  +----> [Semantic Verifier Service]
  |         |
  |      citation_confidence_score per claim
  |         |
  +----> [Confidence Signal Layer]
  |         |
  |      confidence_signal + source_passage_used per claim
  |         |
  v         v
[Enriched Answer Payload] ---------> [Token Stream -> User]
                          |
                          +--> [Audit Persistence Layer] --> [CitationAuditRecord Store]
                                                         --> [Audit API / Export Service]
```

---

## 4) System layers

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

**Edge case - zero citations:** If the LLM draft contains no citation superscripts, the Claim Extraction Service returns an empty list. The verification gate skips all verifier calls and sets `verification_pass_status: skipped_no_citations`. The answer streams immediately with no signals and no audit record.

**Edge case - citation index with no URL mapping:** If a superscript references an index that is not present in the synthesis stage's source list (e.g., due to a source retrieval failure), the corresponding claim is emitted with `source_url: null` and the Confidence Signal Layer assigns `confidence_signal: unverified_claim` without calling the verifier.

**Latency target:** <20ms P95. The extraction is a synchronous in-process operation; it must not be the bottleneck in the verification gate.

---

### Layer 2: Source Content Cache Interface

**Purpose:** Retrieves the full cached text of each cited source URL to provide passage material for the cross-encoder. This layer is a read-only consumer of the retrieval pipeline's existing source cache; it does not fetch live pages.

**Cache lookup:** For each unique `source_url` in the claim list, the interface performs a key lookup against the retrieval cache (key structure: `source_text:{sha256(url)}`). The returned value is the full parsed text of the page at the time of retrieval, stored as a UTF-8 string. The cache TTL is set by the retrieval pipeline (typically 10-60 minutes depending on source type); the verification layer does not control or extend this TTL.

**Cache miss handling:** A miss means the source text is not available for verification. This happens when: (a) the source retrieval failed silently during the query pipeline, (b) the source page requires JavaScript rendering and the retrieval pipeline returned an empty body, or (c) the cache entry expired between retrieval and verification (rare within the query lifecycle, but possible for very long LLM synthesis times). On a miss, the claim is passed to the Confidence Signal Layer with `source_passages: null`; the layer assigns `confidence_signal: unverified_claim`.

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

**Batching:** All claims in a single answer are batched into a single inference call (up to 8 claim-source pairs per batch). The batch is scored in parallel across the cross-encoder model's capacity. This is the engineering requirement from the PRD: citations must be verified in parallel, not sequentially. Sequential scoring of a 4-citation answer at 110ms per claim would take 440ms - already at the Academic Focus P95 budget before any other gate overhead. Parallel batch scoring targets <200ms P95 for a 4-claim batch.

**Calibration requirements:** The Platt scaling parameters (a, b) must be re-fitted on a held-out calibration set before each model version promotion. The Expected Calibration Error (ECE) of the deployed model must be <0.05 on a 2,000-example calibration set. This is a model release gate: calibration test failure blocks production promotion.

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

The 50ms headroom above 350ms absorbs internal scheduling jitter and network variance on the RPC call to the verifier service.

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
```

**Corroborating source check:** The "corroborating_sources >= 2" condition for `verified` requires that at least two independently retrieved sources each score >= 0.65 for the same claim. The verifier is called once; the Confidence Signal Layer checks whether any other source in the answer's retrieved pool also scores >=0.65 for this claim without a separate model call - it reuses already-computed scores from the batch. If only one retrieved source covers the claim, the signal is `single_source`, not `verified`, regardless of the primary score.

**Override rules (applied before signal assignment):**

| Override | Condition | Effect |
|---|---|---|
| Exact-match override | Claim contains a specific date or proper noun that appears verbatim in the source document | Assign `verified_exact_match` flag; skip score threshold; set `confidence_signal: single_source` minimum |
| PR domain override | `source_domain` matches known press release distribution domains (prnewswire.com, businesswire.com, globenewswire.com) | Lower `low_confidence` threshold from 0.65 to 0.55 for this source type; reduces false-positives from footer-structured date announcements |

**Replacement source search:** When a citation scores <0.65 and no override applies, the layer searches the answer's already-retrieved source pool (all sources retrieved for this query, not just the ones already cited) for a better match. It reuses cached source passages and makes a targeted verifier call for up to 3 candidate replacement sources. The replacement call is time-boxed at 100ms; if no replacement scores >=0.65 within that window, `low_confidence` is assigned and the original citation is retained unmodified.

**Answer payload enrichment:** The layer writes a `confidence_annotation` object into the answer payload for each claim. The claim text is byte-identical to the original draft - enrichment adds metadata, never modifies content. The stream gate releases the enriched payload to the browser renderer, which maps `confidence_annotation` to the UI badge and source panel state.

---

### Layer 5: Audit Persistence Layer

**Purpose:** Persists a `CitationAuditRecord` for every answer that completes the verification gate. Serves the audit read API for Teams and Enterprise accounts. Powers the Spaces audit panel and the JSON/PDF export pipeline.

**Write path (asynchronous):** After the enriched answer payload is released to the stream, the verification gate coordinator publishes the `CitationAuditRecord` to an internal write queue. A background worker processes the queue and writes to the audit store (Postgres). The write is decoupled from the user-facing latency path. Write failures are retried up to 5 times with exponential backoff before being routed to a dead-letter queue and generating an alert.

**Reconciliation:** A daily reconciliation job compares the count of answers with `verification_pass_status: complete` (from the `citation_verification_completed` analytics event) against the count of persisted `CitationAuditRecord` rows for the same date. Any gap > 0.1% triggers an alert. The job replays missing records from the dead-letter queue where possible.

**Retention policy:**
- Teams and Enterprise: 5-year retention (full `CitationAuditRecord` with plain-text `claim_text`).
- Pro: 90-day retention (plain-text `claim_text`); purged and replaced with a one-way hash after 90 days.
- Free: 90-day retention; `claim_text` stored as hash only (never as plain text); no audit API access.

**Read path - Audit API:** The `GET /v1/audit/threads/{thread_id}` endpoint reads from a Postgres read replica (not the primary write path). Access is enforced at the API gateway: Teams and Enterprise keys receive 200; Pro and free keys receive 403. The API contract and full error specification are defined in the PRD section 7.

**Read path - Export service:** The `GET /v1/audit/spaces/{space_id}/export` endpoint triggers batch generation of a `CitationAuditExport` payload. JSON exports are synchronous (200 with download URL). PDF exports are asynchronous: a job is enqueued, a `202` with `poll_url` is returned immediately, and the PDF is generated within 60 seconds for sessions up to 100 answers. PDF generation uses a templated document renderer with the `methodology_disclaimer` field rendered on the cover page.

---

## 5) Core problem this architecture must solve

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

This creates a critical dependency: the retrieval pipeline must cache source full-text with a TTL long enough to survive the LLM synthesis time (which can be 2-10 seconds for long answers). If source cache TTLs are shorter than the synthesis time, cache misses at verification time are inevitable. The source cache TTL is a shared contract between the retrieval team and the verification team - it must be validated in the dependency map and monitored in production.

---

## 6) Data flow narratives

### Flow A: Happy path - Academic Focus query with all citations verified

```
1. User types: "What are the key findings from the AlphaFold2 Nature paper?"
2. Retrieval pipeline fetches 7 candidate sources; caches full text for each URL.
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

5. Source Content Cache Interface fetches:
   -> source_text:{sha256(url_1)}: 4,200-word paper abstract and introduction (cached, hit)
   -> source_text:{sha256(url_2)}: 3,100-word methods section (cached, hit)
   -> source_text:{sha256(url_3)}: 2,800-word supplementary dataset description (cached, hit)
   -> Each segmented into 512-token windows with 64-token overlap

6. Semantic Verifier Service receives one batch of 4 (claim, passages) pairs.
   Batch inference completes in 178ms P50.
   Results:
   claim_1: citation_confidence_score: 0.91, source_passage_used: "...median GDT_TS score of 92.4 on free modelling..."
   claim_2 (url_1): citation_confidence_score: 0.88
   claim_3 (url_2): citation_confidence_score: 0.86
   claim_4: citation_confidence_score: 0.84, source_passage_used: "...trained on 170,000 protein structures from PDB..."

7. Confidence Signal Layer:
   claim_1: score 0.91; only url_1 retrieved; corroborating_sources = 1 -> single_source
   claim_2 + claim_3 (same claim, two sources): both score >= 0.65; corroborating_sources = 2 -> verified
   claim_4: score 0.84; corroborating_sources = 1; contains proper nouns "UniProt", "PDB" present verbatim in source -> exact-match override -> single_source

   Answer payload annotated with confidence signals.

8. Total gate time: 22ms (extraction) + 14ms (cache) + 178ms (inference) + 8ms (signals) = 222ms.
   Well within 400ms Academic Focus SLO.

9. Enriched answer streams to user. Source panel shows highlighted passage for the Evoformer claim.

10. Async: CitationAuditRecord written to audit store.
    verified_count: 1 (Evoformer claim)
    single_source_count: 2 (GDT claim, dataset claim)
    low_confidence_count: 0
    overall_answer_accuracy_score: (0.91 + 0.87 + 0.84) / 3 = 0.87
```

---

### Flow B: Citation laundering caught - low-confidence signal prevents propagation

```
1. User types: "What percentage of startups fail within the first year?"
2. LLM generates: "Approximately 90% of startups fail within the first year of operation [2]."
   (The correct figure from the cited source is 20% fail in year 1; 90% is a commonly misquoted lifetime figure.)

3. Claim Extraction Service extracts:
   claim_1: {"claim_text": "Approximately 90% of startups fail within the first year", "source_url": url_2}

4. Source cache hit: url_2 is a CB Insights startup failure report.

5. Verifier runs on (claim_1, source_passages_from_url_2).
   The report's actual text reads: "About 20% of new businesses fail during the first year."
   No passage in url_2 supports "90% fail within the first year".
   citation_confidence_score: 0.28

6. Confidence Signal Layer:
   score 0.28 < 0.65 -> trigger replacement source search
   Replacement search across remaining 6 retrieved sources:
   -> url_4 (Forbes article): score 0.31 (mentions "90% fail" but in lifetime context - ambiguous)
   -> url_5 (BLS data): score 0.22
   -> No replacement source scores >= 0.65 for the specific "first year" claim
   replacement_source_found: false -> confidence_signal: low_confidence

7. Enriched answer streams with yellow indicator on citation [2]:
   "Approximately 90% of startups fail within the first year of operation [2 - yellow indicator]"

8. User clicks citation [2] with caution.
   Source panel: "Limited sources - The source only partially supports this claim"
   User reads the CB Insights report; sees the 20% figure; corrects the claim before sharing.

Result: The architecture prevented a wrong claim from being used without independent review.
The system did not rewrite the answer - it directed the user's attention precisely.
```

---

### Flow C: Degraded mode - verifier service unavailable

```
1. Verifier service health check fails (service restart, overload, or dependency failure).
   Verification gate coordinator receives health check failure before processing the answer.

2. Gate activates degraded mode:
   -> Skip all verifier calls
   -> Skip confidence signal assignment
   -> Release answer payload to stream WITHOUT confidence signals
   -> Set verification_pass_status: degraded in the answer metadata
   -> No CitationAuditRecord written (verification did not run)

3. Answer streams normally with no confidence badges.
   User sees the same answer they would have seen before Verified Answers launched.
   No error messages. No UI indicators of degraded mode.

4. citation_verification_completed event emitted:
   {"verification_pass_status": "degraded", "total_citations": 4, "verified_count": 0, ...}

5. On-call engineer receives alert: verifier service error rate > 1% for 1-hour window.
   Degraded mode rate alert fires if > 0.5% of queries are degraded within 15 minutes.

Recovery: Degraded mode is transient. Once the verifier service recovers, the gate automatically
resumes normal operation for subsequent queries. No user-visible recovery step is required.
```

---

## 7) Core data model

### `CitationVerificationResult` (transient - per claim per answer turn)

Produced by the Semantic Verifier Service and Confidence Signal Layer during the gate. Rolled up into `CitationAuditRecord` before async persistence. Not stored independently.

```json
{
  "claim_id": "string - UUID; unique per claim per answer turn",
  "claim_text": "string - verbatim extracted claim text from the answer draft",
  "claim_char_start": "integer - character offset of claim start in answer draft",
  "claim_char_end": "integer - character offset of claim end in answer draft",
  "source_url": "string - full URL of cited source; null if no citation",
  "source_domain": "string - domain extracted from URL for analytics",
  "source_passage_used": "string - the source passage that scored highest; max 512 chars",
  "source_passage_char_offset": "integer - byte offset of source_passage_used in the cached source document",
  "raw_cross_encoder_score": "float - raw model output before Platt scaling",
  "citation_confidence_score": "float [0.0-1.0] - calibrated probability that source supports claim",
  "confidence_signal": "enum: verified | single_source | low_confidence | unverified_claim",
  "corroborating_source_count": "integer - number of retrieved sources scoring >= 0.65 for this claim",
  "exact_match_override_applied": "boolean",
  "source_format_override_applied": "boolean - true if PR domain threshold adjustment was applied",
  "replacement_source_attempted": "boolean",
  "replacement_source_url": "string | null",
  "replacement_citation_confidence_score": "float | null",
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
  "verification_pass_status": "enum: complete | degraded | skipped_no_citations",
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
TTL: set by retrieval pipeline (typically 10-60 minutes)
```

### `confidence_annotation` (answer payload enrichment - not persisted independently)

The in-memory object attached to each citation in the answer payload before streaming. Consumed by the browser renderer to produce UI signals.

```json
{
  "citation_index": "integer - 1-based citation number in the answer",
  "confidence_signal": "enum: verified | single_source | low_confidence | unverified_claim",
  "citation_confidence_score": "float",
  "source_passage_highlight": "string - the source_passage_used, for source panel highlight",
  "source_passage_char_offset": "integer",
  "replacement_applied": "boolean - true if the original citation URL was replaced",
  "replaced_source_url": "string | null"
}
```

---

## 8) Failure modes

| Failure | Detection | Mitigation | User-visible behaviour |
|---|---|---|---|
| Verifier service timeout (gate exceeds 500ms/700ms hard timeout) | Gate coordinator receives timeout error from RPC call | Activate degraded mode: answer streams without confidence signals; `verification_pass_status: degraded` | Answer streams with no badges; no error message |
| Source cache miss for a cited URL | `source_passages: null` returned by cache interface | Assign `confidence_signal: unverified_claim` for the affected citation; continue verification for other citations | No badge on affected citation; other citations still verified |
| Claim extraction parse failure (malformed LLM draft format) | Exception in Claim Extraction Service | Return empty claim list; skip all verification; set `verification_pass_status: skipped_parse_failure` | Answer streams with no badges |
| Cross-encoder model calibration drift (false-positive rate climbs above 12% in weekly sampling) | Weekly human review job compares sampled `low_confidence` results against human labels | Alert ML Platform; investigate model drift; re-run Platt scaling calibration; do not automatically roll back (false-positives are conservative, not harmful) | Increased `low_confidence` signals that may be over-cautious; addressed in next model calibration cycle |
| Audit record write failure (Postgres write error or timeout) | Dead-letter queue growth; reconciliation job detects gap between analytics events and persisted records | Retry queue (5 attempts, exponential backoff); after exhaustion, route to dead-letter; alert on >0.1% daily gap | No user-visible impact; audit record may be missing for the affected turn |
| Replacement source search timeout (>100ms) | Internal timeout within Confidence Signal Layer | Assign `low_confidence` with original citation; skip replacement; `replacement_source_attempted: true`, `replacement_source_url: null` | `low_confidence` badge shown with original citation; user directed to verify manually |
| Source cache TTL expiry between retrieval and verification (synthesis took longer than cache TTL) | Cache miss on URL that was retrieved at query start | Assign `unverified_claim` for the affected citation | No badge on affected citation |
| Concurrent query spike causing verifier service overload | Verifier service P95 latency alert (>200ms for batch); degraded mode activation rate rising | Auto-scale verifier service replicas (horizontal); gate coordinator falls back to degraded mode if batch latency exceeds 500ms | Degraded mode activated for queries during the spike |

---

## 9) Architectural trade-offs

### Trade-off 1: Pre-stream gate vs. async annotation

**Option A (chosen):** Verification is a synchronous gate. The answer does not stream until the full verification pass is complete and confidence signals are attached to the payload.

**Option B (rejected):** Verification runs in parallel with streaming. Confidence signals are injected into the rendered UI as they arrive, while the answer text is already visible to the user.

**Why A wins for the target segment:** Knowledge workers and academic researchers use verified citations to make high-stakes decisions - pasting into decks, citing in papers, forwarding to colleagues. If the signal arrives after the claim is already read and processed, the user has formed an initial trust judgment. Post-hoc signal revision is cognitively more disruptive than pre-formed confidence from the start. For this segment, 400ms of additional latency is acceptable; a confidence signal that feels like a footnote rather than a guarantee is not.

**Cost of A:** Time-to-first-token is 400ms (Academic Focus) to 600ms (Web Focus) slower than baseline. For queries that produce simple verified answers, this overhead is detectable. The UX must signal that something trustworthy is happening during the gate delay - a "verifying citations..." indicator while the gate runs is the required compensation.

---

### Trade-off 2: Dependency on retrieval cache vs. live source re-fetch

**Option A (chosen):** The verification layer reads only from the retrieval pipeline's existing source cache. It never fetches source URLs live.

**Option B (rejected):** The verification layer fetches the latest version of each cited source URL live during the gate.

**Why A wins:** A live re-fetch adds a minimum of 200-800ms per unique source URL (network round trip + HTML parsing). For an answer with 4 citations from 4 domains, Option B adds 800ms-3.2s to the gate. This blows every latency SLO in the NFR table. The retrieval cache is already warm from the query pipeline; reading it is a <15ms operation for the full answer.

**Cost of A:** The verification layer is checking the source text as it was at retrieval time, not as it is now. For most academic and technical sources, this distinction is irrelevant within a query session. For rapidly updating sources (news, live sports, market data), the verified claim may be accurate against the cached version but stale by the time the user reads it. This is documented in the PRD as a non-goal: source staleness is a separate retrieval infrastructure workstream.

---

### Trade-off 3: Cross-encoder scoring vs. bi-encoder scoring

**Option A (chosen):** Cross-encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`). The claim text and source passage are scored jointly as a concatenated input. The model attends to the interaction between claim and passage. Accuracy: 88.9% on the labelled test set. False-positive rate: 7.2%.

**Option B (rejected):** Bi-encoder (embedding cosine similarity). The claim and source passage are encoded independently; cosine similarity between embeddings is the score. Accuracy: 81-85% on the same test set. False-positive rate: 10-14%.

**Why A wins:** The bi-encoder's false-positive rate (10-14%) is above the 8% threshold. A system that marks 10-14% of correct claims as `low_confidence` destroys user trust in the signal faster than it builds it. The cross-encoder's superior accuracy (7.2% false-positive rate) is worth the higher per-claim latency (110ms vs. 28-55ms) because batching absorbs the latency overhead.

**Cost of A:** Higher cost per claim ($0.18/1M vs. $0.04-0.09/1M) and higher latency per claim. Both costs are acceptable: the per-claim cost is economically negligible at Perplexity's query volume, and the latency is addressed through parallel batching.

---

### Trade-off 4: Confidence signal as pre-stream payload enrichment vs. separate UI API call

**Option A (chosen):** Confidence signals are embedded in the answer payload before streaming begins. The browser renderer reads them from the payload and renders badges inline.

**Option B (rejected):** The answer streams first; the browser makes a separate API call for confidence signals after the answer is fully rendered.

**Why A wins:** Option B has the same failure as async annotation - the user reads the answer without confidence signals. Additionally, Option B requires a second API round trip, which at browser network latency (20-100ms) means a visible badge "pop-in" effect. This is perceptually disruptive and signals to the user that the verification is an afterthought, undermining the "Perplexity guarantees this" positioning.

**Cost of A:** The browser renderer must be designed to consume and render `confidence_annotation` objects embedded in the answer payload. This is a renderer contract change - all platforms (web, iOS, Android) must support the new payload format before any user sees confidence signals. This increases the implementation coordination surface area.

---

### Trade-off 5: Replacement source selection as in-gate operation vs. deferred to re-query

**Option A (chosen):** When a citation scores `low_confidence`, the gate attempts to find a replacement source from the already-retrieved pool within the 100ms replacement time budget. If found, the replacement is silently applied before streaming.

**Option B (rejected):** When a citation scores `low_confidence`, display the `low_confidence` signal and offer the user a "find a better source" button that triggers a targeted re-retrieval query.

**Why A wins:** Option B adds user friction for a failure mode that the system can often self-correct without user action. A replacement source in the already-retrieved pool costs <100ms and no additional retrieval call. The user's experience is a clean answer with a `single_source` badge rather than a `low_confidence` indicator requiring manual action. Option A is the right default.

**Cost of A:** Silent source replacement means the user sees a different citation URL than the one the LLM originally wrote. The original LLM draft cited source A; the rendered answer cites source B. This is transparent in the source panel (the `source_passage_used` from source B is highlighted) but not in the answer text (the superscript number still corresponds to the original citation position). This creates a minor discrepancy between the draft and the rendered answer that must be documented in the audit record (`replacement_applied: true`, `replaced_source_url` field).

---

## 10) Open questions

**Q1: Compound claim parsing in Claim Extraction**

When the LLM produces a multi-clause sentence with a single superscript at the end - for example, "AlphaFold2 uses the Evoformer architecture, trains on PDB structural data, and achieves 92.4 GDT on CASP14 targets [1]" - how does the Claim Extraction Service split this into individual claims? If each clause becomes a separate claim verified against the same single source, the system may produce three separate `single_source` signals for what the user sees as one cited sentence. If the clauses are kept as one compound claim, the verifier may score high on one clause and average down because another clause is not well-supported. The extraction granularity policy is not yet defined; it needs a labelled test set and human evaluation before the model goes to production.

**Q2: JavaScript-rendered and paywalled source handling**

The retrieval pipeline's source cache relies on fetching the HTML of each cited URL. For sources that require JavaScript rendering (SPAs, dynamically loaded content) or that are behind paywalls (academic journals with soft paywalls, news sites with metered access), the cached content may be a stub ("Please enable JavaScript" or a registration wall) rather than the actual article text. The verifier will correctly score these as `unverified_claim` (no supporting passage), but the user may click through and find the source is actually valid. This false `unverified_claim` signal for paywalled academic journals is particularly problematic in Academic Focus mode, where the segment expects verified academic citations. A paywalled source detection heuristic (domain-level known-paywall list; cached content length < 500 words with no passage match) that assigns a distinct signal state (e.g., `paywall_blocked`) rather than `unverified_claim` would improve signal accuracy. Not yet specified; open for v2.

**Q3: Source cache TTL contract with the retrieval pipeline**

The verification layer's correctness depends on the retrieval pipeline's source cache being warm for all cited URLs at verification time. For long LLM synthesis outputs (complex multi-document answers where synthesis takes 8-12 seconds), a cache TTL shorter than the synthesis time produces cache misses at verification time. The current source cache TTL is not specified in this document; it must be confirmed with the retrieval team. The NFR table targets >95% cache hit rate at verification time - but this rate is entirely determined by retrieval pipeline configuration, not by the verification system itself. This dependency must be formalised as a shared SLO before Phase 1 launch.

**Q4: Multi-turn conversation handling for audit records**

The PRD specifies `CitationAuditRecord` per answer turn within a thread. For multi-turn conversations where the user asks a follow-up question ("What about the failure modes of that approach?") and the LLM produces an answer that partially cites sources from the previous turn's retrieval, how should the audit record be structured? Is the `CitationAuditRecord` for turn 2 self-contained (only sources retrieved in turn 2's retrieval run), or does it reference citations from turn 1's record? The current design assumes self-contained per-turn records, but a follow-up answer may cite sources that were retrieved in turn 1 and are still in the cache - this is architecturally valid but may produce audit records that are confusing to a reviewer who reads turn 2 in isolation (missing source context). Not yet resolved.

**Q5: Replacement source selection ranking**

The Confidence Signal Layer attempts replacement source selection from the already-retrieved pool when a citation scores `low_confidence`. The current design says "up to 3 candidate replacement sources" are tried. What is the ranking criterion for which 3 sources are tried first? Options: (a) ranked by the retrieval pipeline's original relevance score (a proxy for topical relevance), (b) randomly sampled from the non-cited portion of the retrieved pool, (c) re-ranked by a lightweight bi-encoder pre-screen against the claim text before sending to the cross-encoder. Option (c) is most principled but adds compute cost. Options (a) and (b) may waste cross-encoder calls on sources that are topically relevant to the query but not to the specific claim. The replacement selection strategy needs a benchmarked evaluation against a labelled set of `low_confidence` cases before Phase 1 launch.

**Q6: Confidence signal display during streaming for long answers**

The pre-stream gate holds the full answer before streaming begins. For long answers (>500 words), the gate-to-stream transition is a step change: nothing, then the full answer at once. Most Perplexity answers stream progressively; a step-change reveal at 400ms feels like a different rendering pattern. Two options: (a) stream the answer text normally but delay the rendering of confidence badges until the gate completes (badges "pop in" at gate completion), or (b) display a "verifying citations..." indicator during the gate and then stream the full answer with badges simultaneously. Option (a) is closer to the current streaming UX; option (b) makes the gate delay visible and intentional. The PRD trade-off analysis chose option (b) directionally (the "verifying" indicator compensates for the latency), but the exact UX specification is not yet designed. This is a product design question with architectural implications (the renderer must handle two rendering phases per answer) that needs resolution before Phase 1.

---

*All latency figures are directional estimates based on publicly available cross-encoder model benchmarks and industry Redis/Postgres performance profiles - not internal Perplexity data.*
