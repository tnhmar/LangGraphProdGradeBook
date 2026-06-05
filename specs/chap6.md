# Chapter 6 — Perception & Input Processing

**Part 2: Anatomy of an AI Agent**  
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

> *"What the agent cannot represent, it cannot reason about."* — Classical AI maxim

Perception is the layer that transforms raw, heterogeneous inputs into typed, grounded, context-ready representations. It is the first gate in every agentic loop. Garbage in at this layer degrades all downstream reasoning, planning, and action — no amount of prompt engineering recovers from malformed, unfiltered, or ungrounded inputs reaching the LLM.

This chapter specifies the complete perception pipeline: input surface definition, normalization, multimodal extraction, relevance filtering, streaming ingestion, grounding, and the full Java + LangGraph node architecture for `PerceiveNode`.

---

## 6.1 — The Agent's Perception Surface

### Context

An agent's perception surface is larger than the most recent user message. At a single step, the support agent may receive: the customer's original ticket, the reply history, a CRM account profile, an order record from an OMS, a batch of payment events from a ledger API, an internal knowledge base article, and the structured result of a prior tool call. Each signal has a different structure, freshness profile, trust level, and semantic density.

### What (the spec)

**Input classification by origin — four canonical classes:**

| Origin class | Examples | Trust tier | Freshness profile |
|---|---|---|---|
| **User-originated** | Messages, uploads, voice transcripts | Medium — intent is real, facts may be wrong | Current but ambiguous |
| **System-originated** | Event triggers, scheduled callbacks, pipeline notifications | High — machine-generated, schema-bound | Current, authoritative for its event |
| **Environment-originated** | Tool outputs, API responses, external data | Varies by source SLA | May lag live system by seconds to minutes |
| **State-derived** | Prior run observations, working memory summaries | Medium — derived, not primary | Age equals session duration |

**The full perception surface must be explicitly declared before any pipeline is built:**

For each input type, document:
1. **Type identifier** — a canonical string name (e.g., `CUSTOMER_MESSAGE`, `PAYMENT_EVENT_BATCH`)
2. **Origin class** — one of the four above
3. **Trust tier** — LOW / MEDIUM / HIGH
4. **Freshness profile** — expected staleness at time of ingestion
5. **Schema contract** — the typed structure the pipeline must output
6. **Failure mode** — what happens if this input is absent, malformed, or late

### Why (engineering implications)

- The trust level and staleness profile of each origin class directly determine how aggressively the perception layer filters and validates each source before passing it downstream.
- An agent that treats all inputs as equally trustworthy will reason confidently on stale CRM data while discarding a high-confidence payment ledger event.
- Failing to declare the full perception surface before building the pipeline produces a system where new input types are added ad hoc, without normalization contracts, and silently bypass relevance filtering.

### How (system design rules)

1. **Define the full perception surface document** as a first engineering artifact, before building any pipeline stage.
2. **Assign a trust tier and freshness profile to every input type** at design time. These drive filter aggressiveness and staleness handling downstream.
3. **Treat the perception surface as a versioned contract.** Adding a new input type is a schema change — it requires a pipeline update, a trust tier assignment, and a failure mode definition.
4. **Never pass an undeclared input type through the pipeline.** Unknown inputs are rejected at the pipeline boundary with a `UnknownInputTypeRejection` event.
5. **In Java + LangGraph**: `InputDescriptor` is a record type with fields `inputType`, `originClass`, `trustTier`, `freshnessProfile`, `schemaClass`, and `failurePolicy`. `PerceiveNode` validates every incoming payload against the registered `InputDescriptor` registry before processing begins.

---

## 6.2 — Input Normalization Pipeline

### Context

Normalization transforms raw inputs into stable, typed objects that downstream components can use reliably. It is not formatting cleanup — it is a structured pipeline with defined stages, typed contracts, and explicit failure modes. The output of this pipeline is never a raw string; it is a typed, annotated object addressable by field at reasoning time.

### What (the spec)

**Six mandatory pipeline stages, in order:**

| Stage | Operation | Output | Failure policy |
|---|---|---|---|
| **1. Type identification** | Classify payload as its declared input type | `InputType` enum value | Reject with `UnknownInputTypeRejection` |
| **2. Format parsing** | Extract structure from payload (parse JSON, extract email fields, parse CSV) | Raw typed map | Reject with `ParseFailure` |
| **3. Schema validation** | Verify payload conforms to the declared contract for its type | Validated record | Reject with `SchemaViolation`; log full payload |
| **4. Normalization** | Clean whitespace; standardize encodings; unify date formats to ISO 8601; normalize numeric precision | Normalized record | Log normalization deltas for audit |
| **5. Metadata injection** | Tag with: `sourceId`, `ingestionTimestamp`, `freshnessRating`, `trustTier`, `runId`, `stepId` | Enriched record | Hard failure — metadata is mandatory |
| **6. Budget management** | Apply token-limit-aware summarization or truncation for payloads exceeding working memory budget | Budget-compliant record | Log truncation events; preserve truncation boundary |

**Critical operational rule:** The pipeline runs at **every ingestion boundary**, not only at session start. When a tool returns a result at step 7 of a run, that result enters the same normalization pipeline as the original ticket at step 1.

**Pipeline output contract:** every stage produces a typed Java record. No stage passes raw strings or untyped maps to the next stage. The final output is a `NormalizedInput` record with all six metadata fields populated.

### Why (engineering implications)

- Pipeline stages implemented as composable transformers with typed inputs and outputs make each stage independently unit-testable and allow safe pipeline reuse without hidden coupling between stages.
- Skipping schema validation produces inputs that look structurally correct but contain semantic garbage (e.g., a `chargeAmount` field that is a string instead of a `BigDecimal`). These propagate silently to the reasoning layer and produce confident but wrong plans.
- The budget management stage is the most commonly omitted stage in early implementations and the most commonly demanded fix after the first context overflow in production.

### How (system design rules)

1. **Implement each pipeline stage as a separate, independently testable Java class** with typed input and output records.
2. **Never short-circuit the pipeline for "simple" inputs.** Every input, regardless of origin, runs all six stages. Exceptions create untestable bypass paths.
3. **All pipeline rejections produce a typed `NormalizationFailure` event** (not an exception) that is logged, metrics-tagged, and routed to the failure handling path — not silently swallowed.
4. **Log normalization deltas** (what changed between stage 4 input and output) for all date, encoding, and numeric transforms. These are essential for debugging grounding failures downstream.
5. **Set token budget thresholds per input type**, not as a global constant. A payment event batch has a different budget ceiling than a knowledge base article.
6. **In Java + LangGraph**: `NormalizationPipeline` is a Spring `@Component` composed of six `PipelineStage<I, O>` implementations. `PerceiveNode` calls `NormalizationPipeline.process(RawInput)` and receives `Either<NormalizationFailure, NormalizedInput>`. Budget management uses the `TokenBudgetManager` component (shared with `StateManager`).

---

## 6.3 — Multimodal Perception

### Context

Not all agent inputs are text. The support agent may receive a screenshot of a duplicate charge, a PDF receipt, a scanned invoice, or an audio message. Each requires modality-specific preprocessing before its content can enter the reasoning context. Sending raw binary or richly formatted documents to the model without extraction produces unreliable results even when the model nominally supports the modality.

### What (the spec)

**Modality extraction table — mandatory for every non-text input type:**

| Modality | Extraction path | Structured output | Risk if skipped |
|---|---|---|---|
| **Screenshot / image** | OCR + object detection + captioning | Amount, date, merchant name, auth code as typed fields | Model receives uninterpretable bytes or incorrect automatic OCR |
| **Audio** | ASR + speaker diarization | Timestamped transcript with speaker labels | Temporal structure and speaker identity lost |
| **PDF document** | Layout parser + table extractor | Per-section text + per-table typed matrix | Tables flatten incorrectly; structure destroyed |
| **Structured data (CSV, Excel)** | Schema parser + cell-level typing | Typed column map with inferred schema | Type information lost; numbers become strings |
| **Video** | Frame sampling + per-frame OCR/captioning | Timestamped caption sequence | Temporal events collapsed or lost |

**Fusion strategy (how extracted non-text content combines with text context):**

Two approaches, ordered by preference:
1. **Structured text annotation (preferred):** Convert all extracted content to typed, labelled text annotations and inject them as additional `NormalizedInput` objects in the normalization pipeline. This is modality-agnostic, token-budgeted, and always auditable.
2. **Native multimodal tokens (where verified):** For models with confirmed vision capability and where lossy extraction is unacceptable, pass image tokens directly. Requires explicit capability check at agent startup — never assume multimodal support from model name alone.

**Two-phase extraction architecture:**
- **Phase 1 — Extract:** Run modality-specific extractor. Output: raw structured map.
- **Phase 2 — Annotate:** Convert raw map to labelled `NormalizedInput` objects. Apply trust tier from source. Inject `modalityType` metadata field.

### Why (engineering implications)

- Modality-specific extraction is a required pipeline stage, not an optional enrichment. An agent that assumes its LLM "can handle PDFs" without extraction will fail inconsistently across model versions, document layouts, and file sizes.
- Extraction quality directly determines reasoning quality for multimodal inputs. A table extractor that flattens merged cells produces wrong numeric data that the agent will reason over confidently.
- The two-phase architecture (extract then annotate) keeps the downstream normalization pipeline modality-agnostic. New modalities are added by implementing a new extractor — not by modifying the normalization pipeline.

### How (system design rules)

1. **Implement a `ModalityExtractor` interface** with a `supports(InputType)` method and an `extract(RawInput) → StructuredMap` method. Register all extractors in the `ModalityExtractorRegistry`.
2. **Never pass raw binary content to the normalization pipeline.** All modality extraction happens in a pre-pipeline stage that converts binary to `StructuredMap` before stage 1.
3. **Verify multimodal model capability at agent startup**, not at inference time. If the configured model does not support a required modality, fail fast with a configuration error.
4. **Log extraction confidence scores** for OCR and ASR outputs. Low-confidence extractions trigger a `LowConfidenceExtraction` warning event and apply a reduced trust tier to the resulting `NormalizedInput`.
5. **In Java + LangGraph**: `ModalityExtractorRegistry` is a Spring-managed map of `InputType → ModalityExtractor`. `PerceiveNode` calls `ModalityExtractorRegistry.extract(rawInput)` before `NormalizationPipeline.process()`. Extracted outputs carry `extractionConfidence` as a metadata field.

---

## 6.4 — Relevance Filtering at Ingestion

### Context

Every token in the context window carries a cost. Tokens spent on irrelevant content displace useful reasoning material and increase the probability that the model attends to the wrong information. Relevance filtering at ingestion is therefore both an efficiency decision and a reasoning-quality decision. Relevance filters that are too aggressive are nearly as dangerous as no filtering — dropping a relevant document silently produces a reasoning gap that looks like model error during post-incident analysis.

### What (the spec)

**Two sequential filtering levels:**

| Level | Mechanism | Trigger | What it removes |
|---|---|---|---|
| **Coarse filter** | Rule-based; origin-class and type matching | Applied to every input after normalization | Clearly irrelevant inputs (wrong type, wrong account, wrong session) |
| **Fine filter** | Embedding similarity against active subgoal | Applied to partially relevant candidates | Low-relevance candidates below threshold score |

**Relevance signal: the active subgoal.** Fine filtering scores inputs against the current active subgoal from the `GoalStack`, not the top-level goal. When the active subgoal is `DETERMINE_REFUND_ELIGIBILITY`, payment history and refund policy score high. Shipping history from unrelated orders scores low even if technically part of the customer's record.

**Mandatory filter audit log:** Every filtering decision — both retains and discards — is written to the `FilterAuditLog` with:
- Input identifier
- Filter level (coarse / fine)
- Relevance score (fine filter only)
- Decision (RETAINED / DISCARDED)
- Active subgoal at time of filtering
- Timestamp

**Filtering thresholds are configurable per input type.** There is no single global relevance threshold. A knowledge base article and a payment ledger event have different base relevance distributions and warrant different threshold calibrations.

### Why (engineering implications)

- Silently dropped relevant documents are nearly impossible to detect without a filter audit log. The failure looks exactly like a model reasoning error: the agent reaches a wrong conclusion without any visible error signal.
- Relevance filtering based on the active subgoal (not the top-level goal) dramatically improves precision. A top-level goal of `RESOLVE_SUPPORT_TICKET` is too broad to be a useful relevance signal — every document in the customer's record is nominally relevant.
- Threshold calibration is an empirical discipline. Set initial thresholds conservatively (prefer false positives over false negatives), measure precision and recall against a golden set, and tighten only once the audit log confirms no relevant documents are being dropped.

### How (system design rules)

1. **Implement coarse and fine filtering as separate, independently configurable components.** `CoarseFilter` applies rule-based logic; `FineFilter` applies embedding similarity against the active subgoal vector.
2. **Never filter without logging.** Every `FilterDecision` — retain or discard — is written synchronously to the `FilterAuditLog` before any downstream step proceeds.
3. **Source the active subgoal from `AgentState.goalStack.activeSubgoal`** at the time of fine filtering. If the goal stack is empty, use the top-level goal. Never use a hardcoded or session-start goal.
4. **Make thresholds per-`InputType` configuration**, loaded from `AgentConfig` at startup. No magic numbers in filter logic.
5. **Build a filter precision/recall dashboard** before going to production. Use a golden set of 50–100 labeled (input, subgoal, expected decision) triples. Run it on every threshold change.
6. **In Java + LangGraph**: `RelevanceFilterChain` is composed of `CoarseFilter` and `FineFilter` implementations. `FineFilter` calls `EmbeddingService.embed(activeSubgoal)` and computes cosine similarity against each candidate. `FilterAuditLog` writes to the structured audit store via `AuditService`.

---

## 6.5 — Streaming Inputs and Real-Time Perception

### Context

Some agents receive input as a stream. A live customer chat session generates messages one at a time. A fulfillment monitoring feed emits status events continuously. A real-time payment system pushes notifications as transactions settle. Streaming perception is not simply faster ingestion — it requires segmentation policy, rolling buffer management, and tight coupling to the orchestrator's event model.

### What (the spec)

**Two additions that streaming perception requires beyond batch perception:**

**1. Segmentation** — when enough input has accumulated to justify acting:

| Strategy | Trigger | Best for |
|---|---|---|
| **Time-windowed** | Act after accumulating N seconds of events | High-frequency event streams with uniform density |
| **Event-triggered** | Act when a specific signal arrives (end-of-message, `TASK_COMPLETE`) | Chat sessions; structured event protocols |
| **Semantic heuristic** | Act when a lightweight classifier detects a complete intent has been expressed | Ambiguous streams; voice input |

Segmentation strategy is per-input-type configuration. Never use a single global segmentation strategy for all stream types.

**2. Buffer management** — controlling how much of the stream remains in active context:

| Policy | Mechanism | Token cost | Fidelity |
|---|---|---|---|
| **Full retention** | Keep all turns at full resolution | Linear growth — unsustainable | Perfect |
| **Recency window** | Keep last N turns at full resolution; discard older | Bounded | Recent turns only |
| **Sliding summary** | Keep last N turns at full resolution; summarize older into a rolling summary | Bounded | Compressed fidelity |
| **Importance-weighted** | Keep turns above importance threshold at full resolution; compress others | Bounded | Selective fidelity |

Default policy: **sliding summary** with a configurable recency window (default: last 5 turns at full resolution; older turns compressed into a rolling 200-token summary).

**Stream state object** — maintained per active stream:
- `streamId` — unique identifier for this stream session
- `segmentationStrategy` — configured strategy for this stream type
- `buffer` — current buffer contents with per-turn metadata
- `rollingS summary` — compressed summary of evicted content
- `lastSegmentedAt` — timestamp of last segmentation event
- `eventCount` — total events received

### Why (engineering implications)

- Adopting streaming perception without explicit segmentation produces an agent that either acts on every token (wasting inference budget) or never acts until the stream closes (losing real-time utility).
- Buffer growth without eviction causes context overflow on long-running streams. The failure is gradual — early turns in the session crowd out current context, and the agent begins reasoning as if earlier events are more relevant than recent ones.
- Streaming inputs shift the runtime toward an event-driven execution model. Both the perception layer and the orchestrator must be designed with this coupling in mind — a synchronous orchestrator cannot efficiently serve a streaming perception layer.

### How (system design rules)

1. **Declare segmentation strategy per stream type** in `InputDescriptor`. Never use a global default for all streams.
2. **Implement buffer eviction as a first-class operation**, not as a lazy cleanup. Eviction runs after every segmentation event, before the next cycle begins.
3. **Preserve the rolling summary** across evictions. The summary is part of the agent's working memory for that stream — losing it loses session coherence.
4. **Emit a `SegmentationEvent`** every time the segmentation policy triggers. This event is the signal to the orchestrator that a new perception cycle should begin.
5. **In Java + LangGraph**: `StreamingPerceiveNode` wraps `PerceiveNode` with a `StreamBuffer` and `SegmentationPolicy`. It emits `SegmentationEvent` to the LangGraph event bus. `StreamBuffer.evict()` is called post-segmentation; `SlidingSummaryCompressor` compresses evicted content using a secondary LLM call with a bounded token budget.

---

## 6.6 — Grounding: Connecting Perceived Input to Agent Knowledge

### Context

Grounding resolves ambiguous or implicit references in perceived inputs to the concrete entities, records, and verified facts the agent can act on. Without grounding, the reasoning layer receives language that *points to* things rather than to specific records and values. Grounding failures often look like hallucination — when an agent acts on an unresolved reference and picks the wrong entity, the behavior appears as model fabrication. In many cases, the perception layer simply never resolved the reference.

### What (the spec)

**Two forms of grounding, composed in sequence:**

**1. Referential grounding** — maps phrases to specific database entities:
- `"my order"` → `orderId=88931` via account lookup
- `"last week"` → date range `[2026-05-26, 2026-06-01]` via current date and calendar logic
- `"the charge"` → `paymentEventId=TX-44291` via payment ledger query

**2. Factual grounding** — anchors claims to authoritative data rather than model priors:
- `"I was charged twice"` → verified by querying the payment ledger and returning two captured payment events
- `"my order never arrived"` → verified by querying the OMS and returning shipment status `STALLED_IN_TRANSIT` with last scan timestamp

**Grounding pipeline output — `GroundedContext` record:**

Every material claim in the normalized input is resolved to either:
- A **verified fact** — a concrete entity or value from an authoritative source, with `sourceId`, `retrievedAt`, and `trustTier`
- An **unresolved reference** — marked explicitly with `resolutionStatus=UNRESOLVED` and the resolution attempt log
- An **assumed fact** — a model prior used when resolution is impossible, marked `confidenceTier=ASSUMED` and flagged for downstream confidence tracking

Unresolved references and assumed facts are **never silently promoted** to verified facts. They carry their resolution status into the reasoning layer, where they influence plan confidence and escalation thresholds.

**Grounding coverage requirement:** All entity references (`account`, `order`, `payment`, `user`, `product`) in the normalized input must have a grounding resolution attempt before `GroundedContext` is emitted. Missing coverage is a `GroundingIncomplete` error, not a silent pass.

### Why (engineering implications)

- The reasoning layer operates on whatever it receives from perception. If that content contains unresolved references, the LLM will silently resolve them using its priors — which may be wrong, outdated, or hallucinated. Explicit grounding prevents this by forcing resolution before reasoning begins.
- Referential and factual grounding together convert a message *about* a situation into a structured description *of* a specific situation, with real data supporting every material claim.
- Unresolved references that are passed to the reasoning layer without marking look identical to verified facts in the context window. The LLM cannot distinguish them. The consequences appear as confident, wrong decisions.

### How (system design rules)

1. **Implement a `GroundingService`** that receives `NormalizedInput` and returns `GroundedContext`. It executes referential resolution first, then factual verification for each resolved entity.
2. **All entity resolution calls are tool calls**, executed via the `ToolRegistry` (not direct database calls). This ensures every grounding operation is logged, versioned, and governed by the same validation and audit infrastructure as other tool invocations.
3. **Mark every fact in `GroundedContext`** with its resolution status: `VERIFIED`, `UNRESOLVED`, or `ASSUMED`. This status propagates to the reasoning layer and is used by `PlanValidator` to assign confidence tiers to plan steps that depend on those facts.
4. **Set a `groundingTimeout` per entity type** in `AgentConfig`. Slow resolution (e.g., OMS latency spike) should produce an `UNRESOLVED` mark with a timeout reason — not an infinite wait that blocks the cycle.
5. **Never pass an `UNRESOLVED` reference to an action node** that executes an irreversible operation. `ActionValidator` must check the confidence tier of every fact a proposed action depends on before dispatch.
6. **In Java + LangGraph**: `GroundingService` is injected into `PerceiveNode`. It calls `ToolRegistry.invoke("account.lookup")`, `ToolRegistry.invoke("order.get")`, etc. for each entity reference. Results are assembled into a `GroundedContext` with per-fact `FactResolution` records. `PerceiveNode` emits `GroundedContext` as the output to the LangGraph state channel `PERCEPTION_OUTPUT`.

---

## 6.7 — The `PerceiveNode` Architecture

### Context

All perception pipeline stages compose into a single LangGraph node: `PerceiveNode`. This node is the structural boundary between the environment and the agent's internal state. Every input enters the agent through `PerceiveNode`; nothing bypasses it. The node is stateless across cycles — its output is a `GroundedContext` written to `AgentState`, not internal state held between invocations.

### What (the spec)

**`PerceiveNode` execution sequence (eight steps):**

1. **Receive** raw input from LangGraph state channel `RAW_INPUT`
2. **Validate** input type against `InputDescriptor` registry — reject unknown types immediately
3. **Extract** modality-specific content via `ModalityExtractorRegistry` (if non-text)
4. **Normalize** via `NormalizationPipeline` (all six stages)
5. **Filter** via `RelevanceFilterChain` (coarse then fine, against active subgoal)
6. **Ground** via `GroundingService` (referential then factual)
7. **Budget-check** final `GroundedContext` token count against `AgentState.contextBudget.perceptionSlot`
8. **Write** `GroundedContext` to `AgentState.perceptionOutput` and emit to `REASONING_INPUT` channel

**`PerceiveNode` failure routing:**

| Failure type | Produced by stage | Routing |
|---|---|---|
| `UnknownInputTypeRejection` | Stage 2 (validate) | → `PERCEPTION_FAILURE` channel; orchestrator routes to error handler |
| `NormalizationFailure` | Stages 3–6 of normalization | → `PERCEPTION_FAILURE` channel |
| `GroundingIncomplete` | Grounding | → `REASONING_INPUT` with `groundingStatus=INCOMPLETE`; reasoning layer applies conservative confidence |
| `BudgetExceeded` | Budget check | → Re-run stage 6 (normalization budget management) with tighter ceiling; if still exceeded → `PERCEPTION_FAILURE` |

**Observability requirements for `PerceiveNode`:**

Every invocation emits a `PerceptionTrace` record containing:
- `runId`, `stepId`, `cycleId`
- Input type and origin class
- Normalization delta summary
- Filter decisions (count retained, count discarded, active subgoal)
- Grounding coverage (verified / unresolved / assumed counts)
- Final token count of `GroundedContext`
- Total `PerceiveNode` latency (ms) — broken down by stage

### Why (engineering implications)

- `PerceiveNode` being stateless across cycles is a deliberate design choice: it makes the node independently testable, safely retryable, and parallelizable in multi-input scenarios.
- All perception pipeline stages inside `PerceiveNode` run synchronously within the LangGraph super-step. Grounding tool calls (which are async by nature) use `CompletableFuture.allOf()` to resolve all entity references in parallel before stage completion.
- The `PerceptionTrace` record is the primary debugging tool for any agent behaviour that looks like a model error but is actually a perception failure. Without it, distinguishing "wrong model decision" from "wrong input to the model" requires log archaeology.

### How (system design rules)

1. **`PerceiveNode` is the only entry point for all external inputs.** No other node reads from `RAW_INPUT`. This is an architectural invariant enforced at compile time via LangGraph graph construction.
2. **Stage execution within `PerceiveNode` is sequential and logged.** Each stage writes its output to a stage-specific field in a transient `PerceiveNodeContext` object, not directly to `AgentState`. `AgentState` is written only once, at step 8, after all stages complete.
3. **Grounding tool calls run in parallel** using `CompletableFuture.allOf()`. Individual grounding timeouts are per-entity-type, not a single global timeout.
4. **Emit `PerceptionTrace` unconditionally** — on success and on every failure path. A `PerceptionTrace` with a failure field is more valuable than no trace.
5. **In Java + LangGraph**: `PerceiveNode` implements `NodeAction<AgentState>`. It is registered as the `perceive` node in the `StateGraph`. The LangGraph edge from `START` routes to `perceive`. The edge from `perceive` routes to `reason` (on success) or `handleError` (on `PERCEPTION_FAILURE`). `PerceptionTrace` is written to `AuditService` and emitted as an OTel span.

---

## 6.8 — Perception Failure Modes and Mitigations

### Context

Perception failures are the most commonly misdiagnosed class of agent failures in production. Because they propagate silently to the reasoning layer, they appear as model reasoning errors — the agent reaches a wrong conclusion with high confidence, and the post-incident investigation starts at the prompt. The correct first diagnostic question is always: *did the perception layer produce correct input?*

### What (the spec)

**Six canonical perception failure modes:**

| Failure mode | Symptom | Root cause | Mitigation |
|---|---|---|---|
| **Trust inversion** | Agent acts on stale data as if it were current | Freshness profile not applied; stale CRM record treated as authoritative | Inject `freshnessRating` metadata; reasoning layer applies recency weight |
| **Silent filter drop** | Agent misses a relevant fact; confident wrong conclusion | Relevance threshold too aggressive; or filter applied against wrong subgoal | `FilterAuditLog`; threshold calibration; golden-set regression tests |
| **Grounding miss** | Agent resolves an entity to the wrong record | Reference ambiguous; resolver picked wrong candidate | `GroundedContext.resolutionStatus=UNRESOLVED`; downstream confidence penalty |
| **Modality bypass** | Agent receives garbled content from a PDF or image | Extraction skipped or failed; raw bytes passed to model | `ModalityExtractor` mandatory for all non-text types; extraction confidence logging |
| **Budget truncation loss** | Agent is missing key facts that were in the original input | Token budget management truncated high-value content | Truncate by relevance rank, not by position; log truncation events |
| **Normalization delta loss** | Agent reasons on wrong numeric value or date | Normalization changed a value without audit trail | Log all normalization deltas; surface in `PerceptionTrace` |

**Perception failure diagnostic protocol (post-incident):**

1. Retrieve `PerceptionTrace` for the failing cycle by `runId` + `stepId`
2. Check `groundingCoverage` — were all entity references resolved?
3. Check `filterDecisions` — was the relevant input retained?
4. Check `normalizationDeltas` — were any values transformed unexpectedly?
5. Check `modalityExtractionConfidence` — were any non-text inputs extracted with low confidence?
6. If all four checks pass → the failure originated in the reasoning layer, not perception

### Why (engineering implications)

- Every perception failure that reaches production without being caught by the `PerceptionTrace` will be diagnosed as a model error. This leads to prompt engineering that tries to fix a data quality problem — and fails.
- The six failure modes are not theoretical. Each appears in production deployments of agents with moderate input surface complexity (3+ input types, at least one non-text modality, and at least one external data source with variable latency).
- The diagnostic protocol is only useful if `PerceptionTrace` is always written. A perception trace written on the success path but omitted on the failure path defeats the entire observability design.

### How (system design rules)

1. **Implement automated perception regression tests** using the golden-set fixture: a set of (raw input, active subgoal, expected `GroundedContext`) triples. Run on every deployment.
2. **Add a `perceptionHealth` dashboard** showing: filter drop rate by input type, grounding resolution rate by entity type, normalization delta frequency, and extraction confidence distribution.
3. **Alert on filter drop rate spikes.** A sudden increase in filter discards on a specific input type indicates a threshold miscalibration or a schema change in the upstream source.
4. **Treat `PerceptionTrace` as a first-class operational artifact**, stored with the same retention policy as action audit logs. It is required for compliance investigation of any agent action.
5. **In Java + LangGraph**: `PerceptionHealthDashboard` reads from `AuditService` and `FilterAuditLog`. Alerting thresholds are configured in `AgentConfig.observability.perception`. `PerceptionTrace` retention is governed by `AuditRetentionPolicy` (minimum 90 days).

---

## Key Takeaways for Volume 2 Implementation

- **Perception is the first gate** in every agentic loop. Every failure in perception propagates to the reasoning layer and appears as a model error. Fix perception first.
- **Declare the full perception surface** (all input types, origin classes, trust tiers, freshness profiles, schema contracts) as the first engineering artifact of any agent project.
- **The normalization pipeline has six mandatory stages**, runs at every ingestion boundary (not just session start), and always produces typed `NormalizedInput` records — never raw strings.
- **Multimodal extraction is mandatory**, not optional. Every non-text modality has a dedicated extractor in the `ModalityExtractorRegistry`. Raw binary content never enters the normalization pipeline.
- **Relevance filtering uses the active subgoal** as the relevance signal, not the top-level goal. Every filter decision — retain and discard — is written to the `FilterAuditLog`.
- **Grounding resolves all entity references** to verified facts before the reasoning layer receives the context. Unresolved references are explicitly marked and carry a confidence penalty into planning.
- **`PerceiveNode` is the only entry point** for all external inputs. It is stateless across cycles, emits a `PerceptionTrace` on every invocation (success and failure), and writes to `AgentState.perceptionOutput` only after all eight stages complete.
- **The perception diagnostic protocol** (check `PerceptionTrace` before diagnosing model errors) is the single most effective tool for reducing mean time to root cause in production agent incidents.
