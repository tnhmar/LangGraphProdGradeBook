# Chapter 12 — Architecting Memory Consolidation, Pruning, and Long-Term Persistence

> **Part 3 — Chapter 3 of 5**
> *"A memory system without a lifecycle is not a system. It is an accumulation." — Practitioner maxim, production agent engineering*

Chapter 11 built the storage and retrieval machinery — embeddings, indexes, hybrid pipelines, and production vector databases. This chapter confronts the harder operational question: once memory accumulates in production, how do you keep it **correct, compact, and useful** over time?

The chapter is exclusively about **operational lifecycle**. Retrieval mechanics belong to Chapter 11. Stateful agent behaviour belongs to Chapter 13. Advanced compression and evaluation belong to Chapter 14.

**Sections in this chapter:**
- 12.1 The Memory Consolidation Problem
- 12.2 Summarisation-Based Consolidation Strategies
- 12.3 Importance Scoring — The Load-Bearing Decision Function
- 12.4 Forgetting by Design — TTL and Importance-Based Eviction
- 12.5 Tiered Persistence — Hot, Warm, Cold
- 12.6 Cross-Store Consistency and Conflict Resolution
- 12.7 Memory Audit, Inspection, and GDPR Compliance
- 12.8 Operations Schedule
- 12.9 Design Rules Summary
- 12.10 Transition to Chapter 13

---

## 12.1 — The Memory Consolidation Problem

### Context

Without consolidation, an agent's memory is a growing append-only log. Every session adds records. The log never shrinks. At 10,000 records, retrieval latency is acceptable. At 1,000,000 records, valuable memories are buried under hundreds of thousands of low-importance records, retrieval has degraded, and storage costs have grown without a corresponding improvement in agent quality.

The numbers are concrete: a 1,536-dimensional float32 embedding occupies approximately 6 KB. At 10 million records, embedding storage alone is 60 GB before metadata and payload. HNSW query latency grows logarithmically with corpus size and degrades under concurrent load as the graph structure exceeds available RAM. **Memory growth is isomorphic to technical debt** — invisible at demo scale, catastrophic at production scale.

Mem0 production benchmarks (2026) demonstrate that structured memory pipelines with explicit consolidation outperform full-context prompting baselines by **91% on p95 latency** and **90% on token consumption**, with multi-hop retrieval quality (LOCOMO J-score) improving from 0.22 to 0.51.

### What — The Session-Close Consolidation Pipeline

The most common consolidation trigger is **session close**. The pipeline runs after each session terminates, processing the session's accumulated episodic records. It must be **idempotent** — if interrupted and restarted, it must produce the same result without duplicating records.

**Seven-step session-close pipeline:**

| Step | Operation | Key concern |
|---|---|---|
| 1 | Load all episodic records for `sessionId` | Retrieve from hot-tier vector store |
| 2 | Score each record for importance | Uses formula from Section 12.3 |
| 3 | Discard records below `minImportanceThreshold` | **Cost gate + quality gate** — filters before LLM extraction |
| 4 | Apply selected consolidation strategy (rolling summary / extraction-merge / reflection-synthesis) | LLM extraction pass |
| 5 | Merge extracted facts into the semantic store — deduplicate and conflict-resolve | Calls Section 12.6 protocol |
| 6 | Write task-state checkpoint for any open tasks | Consumed by Chapter 13 |
| 7 | Apply TTL expiry to raw episodic records processed in this session | Triggers Section 12.4 |

> ⚠️ **Step 3 is both a quality gate and a cost gate.** Running LLM extraction over every raw episodic record — including low-value ones — wastes inference budget and introduces noise into the semantic store. Always filter by importance score before extraction.

### Why

Consolidation serves three simultaneous purposes:
- **Compression** — twenty episodic records about a user's coding preferences become three semantic facts. Storage footprint shrinks; retrieval precision improves.
- **Abstraction** — specific time-bound events are generalised into stable, reusable knowledge. The agent retrieves the conclusion, not the source event.
- **Durability promotion** — short-lived episodic records containing genuinely important information are promoted to long-lived semantic memory before their TTL expires.

### How (Java + LangGraph)

```java
// SPEC RULE CON-001
// ConsolidationPipeline — idempotent session-close consolidation
@Component
public class ConsolidationPipeline {

    public ConsolidationReport consolidate(String sessionId, String userId) {
        // Step 1: load episodes
        List<EpisodicRecord> episodes = episodicStore.loadBySession(sessionId);

        // Step 2: score
        List<ScoredEpisode> scored = episodes.stream()
            .map(ep -> new ScoredEpisode(ep, importanceScorer.score(ep)))
            .toList();

        // Step 3: discard below threshold
        List<EpisodicRecord> candidates = scored.stream()
            .filter(se -> se.score() >= config.minImportanceThreshold())
            .map(ScoredEpisode::record)
            .toList();

        // Step 4: apply consolidation strategy
        List<SemanticFact> facts = consolidationStrategy.extract(candidates);

        // Step 5: merge into semantic store
        int written = 0;
        for (SemanticFact fact : facts) {
            if (semanticStore.mergeWithConflictResolution(fact)) written++;
        }

        // Step 6: write task checkpoint (Chapter 13)
        taskCheckpointer.checkpoint(sessionId, userId);

        // Step 7: expire raw episodic records
        int deleted = episodicStore.applyTtl(sessionId, config.episodicTtlDays());

        return new ConsolidationReport(sessionId, episodes.size(), written, deleted);
    }
}

public record ConsolidationReport(
    String sessionId,
    int episodesProcessed,
    int factsWritten,
    int recordsExpired
) {}
```

**Design rules:**
- `CON-001` — `ConsolidationPipeline.consolidate()` MUST be idempotent. Running it twice on the same `sessionId` MUST produce no duplicate records in the semantic store.
- `CON-002` — The pipeline MUST be triggered at every session close, whether normal termination or timeout. A missed consolidation run MUST be detectable via `consolidation_missed_total` Prometheus counter.
- `CON-003` — Step 3 (importance filter before LLM extraction) MUST NOT be bypassed. Bypassing it is a cost and quality defect.
- `CON-004` — `ConsolidationReport` MUST be written to the audit log (Section 12.7) for every run.

---

## 12.2 — Summarisation-Based Consolidation Strategies

### Context

Three strategies cover the production space. They differ in how much original structure they preserve, how much LLM compute they require, and what quality of output they produce. They are not mutually exclusive — a production pipeline typically combines all three at different timescales.

### What — Three Strategies

| Strategy | Mechanism | Best for | Primary risk |
|---|---|---|---|
| **Rolling summary** | Append new episodes to a growing summary; compress when buffer exceeds token threshold | Short-horizon agents, conversational memory, low LLM budget | Irreversible information loss; temporal attribution lost |
| **Extraction-merge** | LLM extracts discrete facts from episodes; each fact merged into semantic store independently | User modelling, preference extraction, structured fact accumulation | Hallucinated extractions; over-generalisation |
| **Reflection-synthesis** | Agent enters a dedicated reasoning pass over many episodes, generating higher-level insights about patterns and trends | Long-horizon agents, behavioural pattern recognition, strategic planning | LLM compute cost; requires high-quality episodic input |

**Combining strategies by timescale (recommended production configuration):**
- **At session close:** Rolling summaries for conversational context + extraction-merge for user profile updates.
- **Weekly scheduled job:** Reflection-synthesis for behavioural pattern reviews across 7 days of episodic records.

### What — Rolling Summary

The rolling summary maintains a compact summary of recent memory, updated incrementally as new episodes arrive. It is the **cheapest strategy** and appropriate for agents that need short-horizon continuity without the overhead of full extraction.

> ⚠️ **Rolling summaries are irreversible.** Once an episode is compressed into the summary, the original detail cannot be recovered. Appropriate for conversational context where individual events have low intrinsic value. **Not** appropriate for compliance-relevant records or high-value episodic events that may need to be retrieved individually later.

### What — Extraction-Merge

Extraction-merge uses an LLM to extract discrete, structured facts from episodic records, then merges each extracted fact into the semantic store with deduplication and conflict resolution.

**Extraction prompt contract (Java `PromptTemplate` implementation):**
```
System: Extract durable, generalisable facts from the following episodic records.
Rules:
- One fact per line.
- Each fact must be a complete, self-contained statement.
- Do not include time-bound events ("user did X yesterday").
- Do not include speculation or inferred intent.
- Output format: FACT: <statement>
Episodic records:
{episodes}
```

**Merge protocol for extracted facts:**
1. Embed the extracted fact.
2. Query semantic store for nearest neighbours (top-3, cosine threshold ≥ 0.92).
3. If a near-duplicate exists: compare confidence scores; keep the higher-confidence record; update `updatedAt` and `accessCount`.
4. If no near-duplicate: write as a new semantic record.
5. If a direct contradiction is detected: invoke conflict resolution (Section 12.6).

### What — Reflection-Synthesis

Reflection-synthesis is a dedicated LLM reasoning pass over a corpus of episodic records (typically one week's worth) to generate higher-level insights:

```
System: You are a memory consolidation agent. Review the following episodic records from 
the past {days} days for user {userId}. Generate:
1. Three to five generalised behavioural patterns you observe.
2. Any preferences or constraints that have strengthened or changed.
3. Any recurring failures or difficulties.
Output each insight as a structured fact with a confidence score (0.0–1.0).
```

Output is written to the semantic store as high-importance records with `source: agent-reflection` and `confidenceScore` from the LLM output.

### How (Java + LangGraph)

```java
// SPEC RULE CON-010
// ConsolidationStrategy — strategy interface
public interface ConsolidationStrategy {
    List<SemanticFact> extract(List<EpisodicRecord> episodes);
}

// SemanticFact — output of any consolidation strategy
public record SemanticFact(
    String       content,
    String       userId,
    String       source,          // "rolling-summary" | "extraction-merge" | "reflection-synthesis"
    double       confidenceScore, // 0.0–1.0
    double       importanceScore,
    List<String> tags,
    Instant      createdAt
) {}
```

**Design rules:**
- `CON-010` — `ConsolidationStrategy` MUST be declared as a Spring `@Bean` and injected into `ConsolidationPipeline`. Hardcoding strategy selection inside the pipeline is prohibited.
- `CON-011` — Extraction-merge MUST include a hallucination guard: any extracted fact with a `confidenceScore < 0.6` (LLM-reported) MUST be discarded before writing to the semantic store.
- `CON-012` — Reflection-synthesis MUST be gated by `consolidation.reflection.enabled` (default `false`) and scheduled via `@Scheduled` at a configurable interval (default: weekly). It MUST NOT run synchronously in the session-close pipeline.
- `CON-013` — All three strategies MUST write `source` metadata to output `SemanticFact` records. The audit log (Section 12.7) uses `source` to trace which consolidation pass produced each semantic record.

---

## 12.3 — Importance Scoring: The Load-Bearing Decision Function

### Context

Importance scoring is the **load-bearing decision function** across the entire memory lifecycle. It drives:
- Step 3 of the consolidation pipeline (which episodes are worth extracting).
- Retrieval ranking (records with higher importance score rank higher when similarity scores are close).
- Pruning decisions (records whose importance score has fallen below threshold are candidates for deletion).
- Storage tier promotion and demotion (Section 12.5).

A poorly calibrated importance scorer silently corrupts all four functions simultaneously.

### What — The Composite Importance Formula

The importance score \( I(r, t) \) for record \( r \) at evaluation time \( t \) is a weighted composite of five signals:

\[ I(r, t) = w_1 \cdot F(r, t) + w_2 \cdot A(r) + w_3 \cdot S(r) + w_4 \cdot R(r) + w_5 \cdot T(r) \]

where:

| Symbol | Signal | Formula | Description |
|---|---|---|---|
| \( F(r, t) \) | **Freshness** | \( e^{-\lambda (t - t_0)} \) | Exponential decay from write time \( t_0 \); \( \lambda \) is the decay rate (typ. 0.01 per day for episodic) |
| \( A(r) \) | **Access frequency** | \( \log(1 + n_{\text{access}}) / \log(1 + n_{\text{max}}) \) | Normalised log of retrieval access count |
| \( S(r) \) | **Salience** | LLM-assigned score at write time \in [0, 1] | How significant is the content intrinsically? |
| \( R(r) \) | **Source reliability** | Lookup table: `user-explicit=1.0`, `tool-verified=0.9`, `agent-inferred=0.6`, `document-extracted=0.8` | Discount records from less reliable sources |
| \( T(r) \) | **Task relevance** | \in [0, 1], computed at consolidation time against current task profile | Is this record relevant to the agent's current goal class? |

**Default weights (calibrate against domain golden set before production):**

\[ w_1 = 0.30,\quad w_2 = 0.20,\quad w_3 = 0.25,\quad w_4 = 0.15,\quad w_5 = 0.10 \]

All weights must satisfy \( \sum_{i=1}^{5} w_i = 1.0 \).

### What — Score Decay Over Time

Freshness \( F(r, t) \) decays exponentially. The half-life is:

\[ t_{1/2} = \frac{\ln 2}{\lambda} \]

**Default decay rates by memory type:**

| Memory type | \( \lambda \) (per day) | Half-life |
|---|---|---|
| Episodic (session-scoped) | 0.07 | ~10 days |
| Episodic (user event) | 0.02 | ~35 days |
| Semantic (user preference) | 0.005 | ~139 days |
| Semantic (domain fact) | 0.001 | ~693 days |
| Procedural (workflow template) | 0.0 | Does not decay |

Procedural memory **does not carry a freshness component** — procedures are replaced by explicit update, not aged out by decay.

### What — Calibration Requirement

The default weights are a starting point. They MUST be calibrated against a **golden importance labelled set** — a collection of (record, human-importance-label) pairs drawn from your actual memory corpus. Calibration adjusts the weights until the model's ranking matches the human labels on held-out data.

**Minimum calibration set:** 300 labelled records across at least three memory types.

### Why

An uncalibrated importance scorer produces the same degraded outcomes as a mis-tuned retrieval system: the wrong records are promoted to semantic memory, the wrong records are pruned, and the wrong records are ranked first at retrieval time. These failures are silent and compound over time.

### How (Java + LangGraph)

```java
// SPEC RULE IMP-001
// ImportanceScorer — computes I(r, t)
@Component
public class ImportanceScorer {

    private final ImportanceScorerConfig config; // weights + lambda values

    public double score(MemoryRecord record) {
        Instant now = Instant.now();
        double freshness  = Math.exp(-config.lambda(record.type()) *
                            Duration.between(record.createdAt(), now).toDays());
        double access     = Math.log1p(record.accessCount()) /
                            Math.log1p(config.maxAccessCount());
        double salience   = record.salienceScore();      // LLM-assigned at write time
        double reliability = config.reliabilityWeight(record.source());
        double taskRel    = record.taskRelevanceScore(); // computed at consolidation time

        return config.w1() * freshness
             + config.w2() * access
             + config.w3() * salience
             + config.w4() * reliability
             + config.w5() * taskRel;
    }
}

@ConfigurationProperties(prefix = "memory.importance")
public record ImportanceScorerConfig(
    double w1, double w2, double w3, double w4, double w5,
    Map<MemoryType, Double> lambdaByType,
    Map<String, Double> reliabilityBySource,
    int maxAccessCount
) {
    // Validates weights sum to 1.0 at startup
    @PostConstruct
    public void validate() {
        double sum = w1 + w2 + w3 + w4 + w5;
        if (Math.abs(sum - 1.0) > 0.001)
            throw new IllegalStateException("Importance weights must sum to 1.0, got: " + sum);
    }
}
```

**Design rules:**
- `IMP-001` — `ImportanceScorer` MUST be a Spring `@Bean`. No component computes importance scores inline.
- `IMP-002` — All five weight values (`w1`–`w5`) MUST be declared in `application.yml`. Hardcoding weights in Java code is prohibited.
- `IMP-003` — Weight validation (`sum == 1.0`) MUST be enforced at startup via `@PostConstruct`. Misconfigured weights are a fatal startup error.
- `IMP-004` — Importance scores MUST be re-computed at consolidation time (not cached from write time), because freshness \( F(r, t) \) is time-dependent.
- `IMP-005` — A golden labelled calibration set of ≥ 300 records MUST exist before the scorer is promoted to production. The `ImportanceScorerCalibrationJob` compares scorer rankings against labels and reports Spearman rank correlation. Correlation < 0.70 blocks production promotion.

---

## 12.4 — Forgetting by Design: TTL and Importance-Based Eviction

### Context

Expiry is the **most underimplemented stage** in production memory systems. Without it, stores grow without bound, stale records compete with current ones during retrieval, and storage costs accumulate silently. **Forgetting is a design requirement, not a failure mode.**

### What — Four Expiry Mechanisms

| Mechanism | Trigger | Granularity | Appropriate for |
|---|---|---|---|
| **TTL-based expiry** | Wall-clock `expiresAt` field | Per-record | Session-scoped episodic records; predictable, coarse |
| **Freshness decay in scoring** | Continuous — \( F(r,t) \) decays with age | Per-record (soft) | Makes old records unreachable before physical deletion; no storage reduction |
| **Importance-threshold pruning** | Scheduled job; removes records with \( I(r,t) < \theta_{\min} \) | Batch | All types; the primary storage reduction mechanism |
| **Consolidate-then-delete** | Post-consolidation; delete source episodic records after semantic promotion | Session-scoped | Episodic records that have been successfully consolidated |

These are **separate mechanisms with separate triggers, separate implementations, and separate operational schedules.** Conflating them produces systems where some records are never expired and others are deleted too aggressively.

### What — TTL Policy by Memory Type

| Memory type | Default TTL | Rationale |
|---|---|---|
| Sensory buffer | 60 seconds | Transient; no value beyond immediate perception |
| Session-scoped episodic | 7 days | Consolidation window; deleted after consolidation pipeline confirms success |
| User-event episodic | 90 days | Long enough to be consolidation input; short enough to limit volume |
| Semantic (preference) | 365 days | Refreshed by consolidation; expires only if access drops to zero |
| Semantic (domain fact) | 730 days | High durability; explicitly versioned when content changes |
| Procedural (workflow) | No TTL | Explicitly versioned; deleted only by operator action |

### What — Importance-Threshold Pruning

Pruning removes records whose importance score has fallen below a minimum threshold \( \theta_{\min} \):

\[ \text{prune}(r) \iff I(r, t_{\text{now}}) < \theta_{\min} \]

**Pruning safety protocol (archive before delete):**
1. Compute current importance scores for all records in the target tier.
2. Collect records with \( I < \theta_{\min} \).
3. **Archive** to cold-tier object storage (S3/GCS) with a 30-day archive retention period.
4. Measure retrieval quality (Precision@5, Recall@20) on the golden retrieval set *before* physical deletion.
5. If quality metrics degrade > 5%, halt deletion and alert.
6. After 30 days without quality regression, physically delete the archived records.

> ⚠️ **Archive before delete.** Implement expiry conservatively — measure retrieval quality before and after each pruning run. Tune thresholds aggressively only once observability is in place.

### How (Java + LangGraph)

```java
// SPEC RULE EXP-001
// MemoryExpiryScheduler — owns all four expiry mechanisms
@Component
public class MemoryExpiryScheduler {

    // Runs every hour — TTL-based expiry
    @Scheduled(fixedRate = 3_600_000)
    public void expireByTtl() {
        int expired = vectorStore.expireByTtl(Instant.now());
        metricsPublisher.recordTtlExpired(expired);
    }

    // Runs nightly — importance-threshold pruning
    @Scheduled(cron = "0 2 * * *")  // 02:00 UTC daily
    public void pruneByImportance() {
        List<RecordId> candidates = vectorStore.findBelowImportanceThreshold(
            config.pruningThreshold());
        archiver.archive(candidates);       // step 3: cold-tier archive
        qualityGate.evaluateBeforeDeletion(candidates); // step 4–5
        // physical deletion deferred 30 days by archiver TTL
        metricsPublisher.recordPruningCandidates(candidates.size());
    }
}
```

**Design rules:**
- `EXP-001` — `MemoryExpiryScheduler` MUST own all four expiry mechanisms as `@Scheduled` methods. Ad-hoc expiry calls scattered through application code are prohibited.
- `EXP-002` — Importance-threshold pruning MUST follow the archive-before-delete protocol. Physical deletion without prior archiving and quality gate evaluation is prohibited.
- `EXP-003` — `pruningThreshold` MUST be configurable in `application.yml`. Default value: 0.15. Reducing below 0.10 requires documented justification.
- `EXP-004` — Retrieval quality MUST be measured (Section 11.6 `RetrievalQualityEvaluator`) before and after every pruning run that removes > 0.5% of the corpus. A quality regression > 5% halts physical deletion.
- `EXP-005` — Prometheus metrics MUST be emitted: `memory_ttl_expired_total`, `memory_pruning_candidates_total`, `memory_physically_deleted_total`, `memory_archive_size_bytes`.

---

## 12.5 — Tiered Persistence: Hot, Warm, Cold

### Context

Not all memory records need the same performance, cost, and durability profile. A record accessed every session deserves low-latency, high-cost storage. A record that has not been accessed in six months should not consume hot-tier HNSW RAM. **Tiered storage matches cost and performance to access patterns.**

Tiering is not a post-launch optimisation. At production scale, failing to tier memory results in hot-tier stores that grow without bound until HNSW query latency degrades to unacceptable levels.

### What — Three-Tier Architecture

| Tier | Storage type | Index type | Retrieval latency | Cost | Content |
|---|---|---|---|---|---|
| **Hot** | Vector DB (Qdrant/Weaviate) + knowledge graph | HNSW | < 20 ms p99 | High | Active episodic + semantic records; \( I(r) \geq 0.50 \); accessed in last 30 days |
| **Warm** | Vector DB (IVF-FLAT) or managed DB | IVF-FLAT | < 100 ms p99 | Medium | Episodic records aged 30–180 days; \( I(r) \in [0.20, 0.50) \) |
| **Cold** | Object storage (S3/GCS) + metadata index | Sequential scan or sparse ANN | Seconds | Low | Archived records; \( I(r) < 0.20 \) or age > 180 days; compliance archive |

### What — Promotion and Demotion Policies

**Promotion (warm → hot):** Triggered on access. A record retrieved from the warm tier is promoted to hot if its recomputed importance score \( I(r, t_{\text{now}}) \geq 0.50 \).

**Demotion (hot → warm):** Scheduled nightly. Records in the hot tier with \( I(r, t_{\text{now}}) < 0.50 \) AND last accessed > 30 days ago are demoted to warm.

**Demotion (warm → cold):** Scheduled weekly. Records in the warm tier with \( I(r, t_{\text{now}}) < 0.20 \) OR age > 180 days are archived to cold.

**Hot-tier size constraint:** The hot-tier HNSW index MUST be sized to fit in available RAM. **The hot tier must be kept small.** The HNSW graph structure adds 20–40% overhead to raw vector storage. Capacity plan: `(maxHotRecords × dimensions × 4 bytes × 1.35) ≤ availableRAM × 0.70`.

### What — The MemGPT Tiering Analogy

The MemGPT model (Packer et al., 2023) formalises this hierarchy as an operating system analogy: the context window is RAM, external stores are disk. The agent uses explicit function calls to promote records from disk into RAM (retrieval) and demote stale in-context content to disk (archival). This model makes tiering a first-class architectural decision, not an infrastructure afterthought.

### How (Java + LangGraph)

```java
// SPEC RULE TIER-001
// TieredMemoryRouter — routes reads and writes to correct tier
@Component
public class TieredMemoryRouter {

    public List<ScoredRecord> query(float[] queryVector, MetadataFilter filter, int topK) {
        // Always query hot tier first
        List<ScoredRecord> hotResults = hotVectorStore.query(queryVector, topK, filter);

        if (hotResults.size() >= topK) return hotResults;

        // Fall through to warm tier if hot results insufficient
        List<ScoredRecord> warmResults = warmVectorStore.query(
            queryVector, topK - hotResults.size(), filter);

        // Promote warm hits with importance >= threshold to hot tier
        warmResults.stream()
            .filter(r -> importanceScorer.score(r.record()) >= config.promotionThreshold())
            .forEach(r -> hotVectorStore.upsert(toVectorRecord(r)));

        return Stream.concat(hotResults.stream(), warmResults.stream())
            .sorted(Comparator.comparingDouble(ScoredRecord::score).reversed())
            .limit(topK)
            .toList();
    }

    // Scheduled nightly — hot → warm demotion
    @Scheduled(cron = "0 1 * * *")
    public void demoteHotToWarm() {
        hotVectorStore.findForDemotion(config.hotDemotionThreshold(), config.hotDemotionAgeDays())
            .forEach(r -> {
                warmVectorStore.upsert(toVectorRecord(r));
                hotVectorStore.delete(r.id());
            });
    }
}
```

**Design rules:**
- `TIER-001` — `TieredMemoryRouter` MUST be the single entry point for all memory reads. Components MUST NOT query individual tier stores directly.
- `TIER-002` — Hot-tier size MUST be monitored via `memory_hot_tier_record_count` gauge. A capacity alert MUST fire when count exceeds 80% of `maxHotRecords`.
- `TIER-003` — Promotion from warm to hot MUST happen synchronously on access (within the read path). Demotion MUST be asynchronous (scheduled, never blocking reads).
- `TIER-004` — Cold-tier records MUST be stored in a format that allows full re-ingestion into the warm tier (raw content + metadata + original embedding). Object storage format: newline-delimited JSON (`*.ndjson`).
- `TIER-005` — Tiering configuration (thresholds, capacities, schedules) MUST be declared in `application.yml` under `memory.tiers.*`. No tier parameters are hardcoded.

---

## 12.6 — Cross-Store Consistency and Conflict Resolution

### Context

When the same semantic fact lives in both a vector store and a knowledge graph, both must be updated together or the system exhibits **split-brain**: the vector store says one thing and the knowledge graph says another. Additionally, the semantic store itself accumulates conflicts over time as the world changes but older records are not updated.

### What — The Daily Cross-Store Consistency Check

**Four-step consistency protocol (runs daily):**

| Step | Operation |
|---|---|
| 1 | **Entity count comparison** — compare entity counts in vector store vs. knowledge graph for each `userId`. Log discrepancies. |
| 2 | **Spot-check sampling** — sample 1% of records from the vector store; verify each has a corresponding node in the knowledge graph. |
| 3 | **Conflict detection** — query semantic store for record pairs that are near-duplicates (cosine > 0.92) but have contradictory content. |
| 4 | **Alert and resolve** — emit `memory_consistency_drift_total` counter for each discrepancy; trigger conflict resolution (below). |

### What — Semantic Conflict Detection

A conflict is defined as two records \( r_1, r_2 \) in the semantic store such that:
- Both are tagged for the same `userId` and same entity.
- Their embeddings are near-duplicates: \( \cos(e_1, e_2) \geq 0.92 \).
- Their contents assert contradictory facts (detected by LLM contradiction classifier).

### What — Four Resolution Policies (Applied in Priority Order)

| Priority | Policy | Rule |
|---|---|---|
| 1 | **Source authority** | `user-explicit` beats `agent-inferred`; `tool-verified` beats `document-extracted`. If one record has higher source authority, it wins. |
| 2 | **Recency** | If source authority is equal, the more recently written record wins. |
| 3 | **Confidence score** | If recency is within 24 hours, the record with higher `confidenceScore` wins. |
| 4 | **Human arbitration** | If policies 1–3 cannot resolve, write a `ConflictPendingResolution` record to the operator review queue. Do not silently discard either record. |

> ⚠️ **Semantic conflicts are invisible to standard database tooling.** They do not produce key collisions or constraint violations. They are detected only by semantic similarity + contradiction classification. Schedule the cross-store consistency check before the first multi-store deployment.

### How (Java + LangGraph)

```java
// SPEC RULE CON-020
// ConflictResolver — applies four resolution policies in order
@Component
public class ConflictResolver {

    public ResolutionOutcome resolve(SemanticFact incumbent, SemanticFact challenger) {
        // Policy 1: source authority
        int authorityDiff = sourceAuthority(incumbent.source())
                          - sourceAuthority(challenger.source());
        if (authorityDiff != 0)
            return authorityDiff > 0
                ? ResolutionOutcome.keep(incumbent)
                : ResolutionOutcome.keep(challenger);

        // Policy 2: recency
        long ageDiff = ChronoUnit.HOURS.between(
            incumbent.createdAt(), challenger.createdAt());
        if (Math.abs(ageDiff) > 24)
            return ageDiff < 0
                ? ResolutionOutcome.keep(incumbent)
                : ResolutionOutcome.keep(challenger);

        // Policy 3: confidence
        if (Math.abs(incumbent.confidenceScore() - challenger.confidenceScore()) > 0.05)
            return incumbent.confidenceScore() > challenger.confidenceScore()
                ? ResolutionOutcome.keep(incumbent)
                : ResolutionOutcome.keep(challenger);

        // Policy 4: human arbitration
        operatorQueue.submitForReview(incumbent, challenger);
        return ResolutionOutcome.pendingArbitration();
    }

    private int sourceAuthority(String source) {
        return switch (source) {
            case "user-explicit"       -> 4;
            case "tool-verified"       -> 3;
            case "document-extracted" -> 2;
            case "agent-inferred"      -> 1;
            default                    -> 0;
        };
    }
}
```

**Design rules:**
- `CON-020` — `ConflictResolver` MUST apply the four policies in priority order. Skipping to recency without checking source authority is prohibited.
- `CON-021` — Policy 4 (human arbitration) MUST write both conflicting records to an operator review queue. Silently discarding either record is prohibited.
- `CON-022` — The daily consistency check MUST be implemented as a `@Scheduled` Spring component. It MUST emit `memory_consistency_drift_total` (counter) and `memory_conflicts_detected_total` (counter) to Prometheus.
- `CON-023` — Conflict detection MUST use the LLM contradiction classifier, not string comparison. String-identical records are duplicates, not conflicts. Conflicts require semantic understanding.

---

## 12.7 — Memory Audit, Inspection, and GDPR Compliance

### Context

Three independent operational requirements mandate a memory audit system:
1. **Incident debugging** — an agent that misbehaves because of a stale, incorrect, or poisoned memory record is nearly impossible to diagnose without a write-immutable log of every memory operation.
2. **GDPR compliance** — Article 15 (right of access) requires producing all personal data held about a user within 30 days. Article 17 (right to erasure) requires demonstrating all personal data has been deleted. Neither obligation can be met without a complete, queryable record of all memory operations per user.
3. **Security review** — detecting memory poisoning (an attacker injecting records from an untrusted source to manipulate agent behaviour) requires querying for writes that originated from unexpected sources. The audit log is the detection surface.

> **Memory auditing is not observability added after the fact. It is structural infrastructure. Build it before the first personal data record is stored.**

### What — Audit Record Schema

Every memory operation MUST produce an audit record. The audit log is **write-immutable** — records are appended, never modified.

```java
// SPEC RULE AUD-001
public record MemoryAuditRecord(
    String    auditId,       // UUID, immutable primary key
    String    operation,     // "write" | "update" | "delete" | "expire" | "consolidate"
    String    recordId,      // ID of the memory record affected
    String    userId,        // data subject (GDPR)
    String    agentId,       // which agent instance performed the operation
    String    source,        // origin of the triggering action
    String    contentHash,   // SHA-256 of the record content (not the content itself)
    Instant   operationAt,   // UTC timestamp
    String    reason,        // human-readable reason for the operation
    Map<String,Object> metadata
) {}
```

**Why `contentHash` instead of full content:** The audit log must not duplicate PII-containing content. The hash allows detecting content changes without storing the content itself.

### What — Common Debug Query Patterns

| Debug goal | Audit query |
|---|---|
| Why did the agent respond this way? | `SELECT * FROM audit WHERE userId=? AND operation='write' ORDER BY operationAt DESC LIMIT 50` |
| What was deleted and when? | `SELECT * FROM audit WHERE userId=? AND operation IN ('delete','expire')` |
| Who wrote this record? | `SELECT agentId, source, reason FROM audit WHERE recordId=?` |
| Did consolidation run for this session? | `SELECT * FROM audit WHERE operation='consolidate' AND metadata->>'sessionId'=?` |
| GDPR Article 15 — all data for user | `SELECT * FROM audit WHERE userId=?` |
| Memory poisoning detection | `SELECT * FROM audit WHERE source NOT IN ('user-explicit','tool-verified','policy') AND operation='write'` |

### What — GDPR Compliance Obligations

| Article | Obligation | Engineering implementation |
|---|---|---|
| Art. 15 (access) | Produce all personal data within 30 days | `GdprDataExportJob` queries audit log + all stores by `userId`; produces a structured JSON export |
| Art. 17 (erasure) | Delete all personal data on request | `GdprErasureJob` deletes all records by `userId` across hot, warm, and cold tiers; writes deletion confirmation audit records |
| Art. 5(1)(e) (storage limitation) | Do not retain data longer than necessary | TTL policies (Section 12.4) + maximum retention cap per memory type |
| Art. 25 (data minimisation) | Collect only data necessary for the purpose | Write gate (Chapter 10 Stage 1) — salience and stability checks before persistence |

> ⚠️ **GDPR erasure must cover all three storage tiers** (hot, warm, cold) and the knowledge graph. A deletion that covers the hot-tier vector store but not cold-tier object storage does not satisfy Article 17. Implement `GdprErasureJob` as a cross-tier operation before the first personal data record is stored.

### How (Java + LangGraph)

**Design rules:**
- `AUD-001` — Every memory write, update, delete, expiry, and consolidation operation MUST produce a `MemoryAuditRecord` appended to the audit log.
- `AUD-002` — The audit log MUST be stored in an append-only, write-immutable store (e.g., Apache Kafka with log compaction disabled, AWS DynamoDB Streams, or a PostgreSQL table with `INSERT`-only row-level security policy). No component holds `UPDATE` or `DELETE` rights on the audit table.
- `AUD-003` — `GdprDataExportJob` MUST be implemented and tested before the first personal data record is stored. Export MUST complete within 24 hours for any single user.
- `AUD-004` — `GdprErasureJob` MUST cover all three tiers (hot, warm, cold) and the knowledge graph. Erasure MUST produce audit records confirming deletion. Erasure MUST complete within 7 days of request.
- `AUD-005` — The `contentHash` field MUST be SHA-256 of the UTF-8 record content. It MUST be computed before write and stored atomically with the audit record.

---

## 12.8 — Operations Schedule

The operations schedule is the **operational contract** for the entire memory system. All jobs MUST be scheduled, monitored, and alerted on independently.

| Job | Schedule | Trigger | Alert threshold |
|---|---|---|---|
| `ConsolidationPipeline` | At every session close | Session termination event | `consolidation_missed_total > 0` |
| `MemoryExpiryScheduler.expireByTtl()` | Every hour | `@Scheduled(fixedRate=3600000)` | `memory_ttl_expired_total` spike |
| `MemoryExpiryScheduler.pruneByImportance()` | Nightly at 02:00 UTC | `@Scheduled(cron="0 2 * * *")` | Retrieval quality regression > 5% |
| `TieredMemoryRouter.demoteHotToWarm()` | Nightly at 01:00 UTC | `@Scheduled(cron="0 1 * * *")` | `memory_hot_tier_record_count > 80% capacity` |
| `CrossStoreConsistencyMonitor` | Daily at 03:00 UTC | `@Scheduled(cron="0 3 * * *")` | `memory_consistency_drift_total > 0` |
| `ReflectionSynthesisJob` | Weekly (Sunday 04:00 UTC) | `@Scheduled(cron="0 4 * * 0")` | Job failure |
| `GdprErasureJob` | On-demand + weekly sweep | Operator request + `@Scheduled` | Erasure overdue > 7 days |
| Vector store backup | Nightly | Infrastructure scheduler | Backup failure |

---

## 12.9 — Design Rules Summary

| Rule ID | Scope | Rule |
|---|---|---|
| `CON-001` | Consolidation | `ConsolidationPipeline.consolidate()` MUST be idempotent. |
| `CON-002` | Consolidation | Pipeline MUST be triggered at every session close; missed runs detectable via counter. |
| `CON-003` | Consolidation | Step 3 importance filter MUST NOT be bypassed. |
| `CON-004` | Consolidation | `ConsolidationReport` MUST be written to audit log for every run. |
| `CON-010` | Strategies | `ConsolidationStrategy` MUST be a Spring `@Bean`. |
| `CON-011` | Strategies | Extraction-merge MUST discard facts with `confidenceScore < 0.6`. |
| `CON-012` | Strategies | Reflection-synthesis MUST be gated by config and run asynchronously. |
| `CON-013` | Strategies | All strategies MUST write `source` metadata to output `SemanticFact` records. |
| `IMP-001` | Importance | `ImportanceScorer` MUST be a Spring `@Bean`. |
| `IMP-002` | Importance | All weights MUST be declared in `application.yml`. |
| `IMP-003` | Importance | Weight validation (`sum == 1.0`) enforced at startup. |
| `IMP-004` | Importance | Importance scores MUST be re-computed at consolidation time. |
| `IMP-005` | Importance | Golden calibration set of ≥ 300 records required before production. Spearman ≥ 0.70. |
| `EXP-001` | Expiry | `MemoryExpiryScheduler` MUST own all four expiry mechanisms. |
| `EXP-002` | Expiry | Archive-before-delete protocol is mandatory for importance-threshold pruning. |
| `EXP-003` | Expiry | `pruningThreshold` MUST be in `application.yml`; default 0.15. |
| `EXP-004` | Expiry | Retrieval quality MUST be measured before and after pruning runs > 0.5% of corpus. |
| `EXP-005` | Expiry | Five Prometheus metrics MUST be emitted by `MemoryExpiryScheduler`. |
| `TIER-001` | Tiering | `TieredMemoryRouter` MUST be the single entry point for all memory reads. |
| `TIER-002` | Tiering | Hot-tier capacity alert at 80% of `maxHotRecords`. |
| `TIER-003` | Tiering | Promotion synchronous on access; demotion asynchronous. |
| `TIER-004` | Tiering | Cold-tier records MUST be stored as re-ingestible NDJSON. |
| `TIER-005` | Tiering | All tier configuration in `application.yml` under `memory.tiers.*`. |
| `CON-020` | Consistency | `ConflictResolver` MUST apply four policies in priority order. |
| `CON-021` | Consistency | Policy 4 (arbitration) MUST write both records to operator queue; no silent discard. |
| `CON-022` | Consistency | Daily consistency check MUST emit two Prometheus counters. |
| `CON-023` | Consistency | Conflict detection MUST use LLM classifier, not string comparison. |
| `AUD-001` | Audit | Every memory operation MUST produce a `MemoryAuditRecord`. |
| `AUD-002` | Audit | Audit log MUST be append-only, write-immutable. |
| `AUD-003` | Audit | `GdprDataExportJob` MUST be implemented before first personal data record. |
| `AUD-004` | Audit | `GdprErasureJob` MUST cover all three tiers and knowledge graph. |
| `AUD-005` | Audit | `contentHash` MUST be SHA-256, computed and stored atomically with audit record. |

---

## 12.10 — Transition to Chapter 13

This chapter answered the operational half of the external memory engineering challenge: how to keep memory correct, compact, and useful over time. The session-close consolidation pipeline, importance scoring formula, TTL and pruning mechanisms, tiered storage architecture, cross-store consistency protocol, and audit system together constitute the **operational lifecycle** of a production-grade agent memory tier.

Chapter 13 builds directly on this lifecycle. With memory correctly managed — consolidated, pruned, tiered, consistent, and audited — Chapter 13 asks: **what agent behaviours does that make possible?**

Chapter 13 covers:
- **The stateless vs. stateful architectural divide** — when each is the correct choice and the five failure modes of stateful architectures.
- **Session state vs. cross-session persistence** — what belongs in Redis vs. the tiered external store; how the session boundary is managed at session close.
- **Persistent user modelling** — how the semantic store becomes a continuously updated model of the user's preferences, constraints, and goals.
- **Task resumption across interruptions** — how checkpoint-based task state enables agents to resume mid-execution after failures or session boundaries.
- **Shared memory in multi-agent systems** — how multiple agents sharing a common memory tier avoid race conditions, divergence, and prompt drift.
- **Memory synchronisation** — how changes in one agent's memory are propagated to agents that share the same user context.
