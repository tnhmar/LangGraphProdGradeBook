# Chapter 10 — The Taxonomy of Agent Memory

> **Part 3 opens here.** Every term defined in this chapter is used without re-introduction in Chapters 11–14.

---

## 10.0 Chapter Intent

### Context
Part 2 produced a fully decomposed single-run agent (runtime, perception, reasoning, action, state manager, cognitive architecture). Every component was run-bounded — state was discarded at session end. Chapter 10 opens Part 3 by treating memory as a first-class architectural subsystem: the component that converts a reactive loop into a system capable of learning, personalising, and improving across every run it completes.

### What
This chapter establishes:
1. The five-type memory taxonomy (sensory, working, episodic, semantic, procedural).
2. The mandatory two-tier hierarchy (in-context vs. external) and the injection pattern that bridges them.
3. The procedural memory spec — structurally separate from the other four types.
4. The memory subsystem interface contract — the boundary every other component must respect.
5. The four-stage memory lifecycle (Write → Store → Retrieve → Expire).

### Why
Without a precise taxonomy, architectural decisions about storage, retrieval, consolidation, and persistence are made informally and expensively. Without a subsystem interface, memory is scattered across the codebase as utility functions that collapse in production.

### How (Java + LangGraph)
All memory operations are mediated through a single `MemorySubsystem` interface. No agent node reaches directly into a database. LangGraph state carries `WorkingMemory`; all durable memory crosses the subsystem boundary via explicit method calls.

---

## 10.1 Why Memory Is the Defining Feature of Intelligent Agents

### Context
Early production agents were effectively stateless: each API call received a fresh context window manually populated with conversation history. That is context injection — not memory — and it breaks when the conversation exceeds the context window, the session ends, or a different user starts a new thread.

### What
Memory is the structural property that converts a reactive function into an intelligent system. The dependency is not philosophical — it is a hard engineering constraint:

| Missing memory type | Consequence |
|---|---|
| No episodic memory | Cannot resume interrupted tasks or learn from past failures |
| No semantic memory | Knowledge horizon permanently capped by training data cutoff |
| No procedural memory | Cannot store, reuse, or refine workflows |
| No external memory (any type) | Effective knowledge horizon capped by context window size |

> **Note — context window sizes (2026):** Frontier models support 200 K–2 M tokens. This reduces urgency for external memory in short sessions but does not eliminate it. Long-running agents, multi-session continuity, user modelling, and cost management all demand external memory regardless of window size.

### Why
Two agents can share the same LLM backbone, the same tool set, and the same planning algorithm. What distinguishes a sophisticated agent from a naive one is almost always the quality of its memory architecture. Memory determines information availability, retrieval quality, adaptation, and continuity.

### How
```java
// SPEC RULE MEM-001
// An agent that cannot answer
// "What do I know, how did I learn it, and how will I remember it tomorrow?"
// does not have a memory system. It has a buffer.
// Replace that buffer with a deliberately designed, production-grade memory architecture.
```

**Design rules:**
- `MEM-001` — Every production agent MUST implement at least working memory (in-context) + one external memory tier. Context-injection-only is prohibited above demo scale.
- `MEM-002` — Memory architecture MUST be designed before the agent components that depend on it. Never add memory as an afterthought.

---

## 10.2 The Memory Taxonomy — Five Types

### Context
The taxonomy draws from cognitive psychology (Atkinson & Shiffrin multi-store model, 1968; Tulving episodic/semantic distinction, 1972) but is adapted as functional engineering archetypes, not neuroscience metaphors. Each type solves a distinct engineering problem.

### What — Part 3 Architecture Map

This table is the canonical reference for all type-specific design decisions across Chapters 11–14.

| Type | Storage tier | Lifetime | Access pattern | Primary engineering concern | Chapter |
|---|---|---|---|---|---|
| **Sensory** | Perception buffer (transient) | Milliseconds | Implicit pre-filter | Signal-to-noise at ingestion | Ch. 5 |
| **Working** | Context window (in-context) | Single session | Zero-latency | Token budget overflow / eviction order | Ch. 8 |
| **Episodic** | External hot/warm vector DB | Days to months | Similarity retrieval | Volume growth, freshness, consolidation | Ch. 11–12 |
| **Semantic** | External hot vector DB + graph | Months to indefinite | Similarity + graph traversal | Contradiction, confidence, freshness | Ch. 11–12 |
| **Procedural** | System prompt + tool library + external store | Stable | Direct injection or retrieval | Inconsistency, undocumented drift | Ch. 10, 12 |

### Five-type definitions

**Sensory memory** — Transient pre-cognitive buffering of raw input signals before relevance filtering. Extremely short-lived, high-bandwidth, discarded after filtering. Primary concern: signal-to-noise. Full engineering treatment belongs to Chapter 5 (Perception). Included in taxonomy for completeness.

**Working memory** — The active, in-context information available during a single reasoning step or session. Maps directly to the context window. Zero-latency, session-scoped. Implementation (eviction, summarisation) belongs to Chapter 8 (State Manager).

**Episodic memory** — Specific events, interactions, and experiences with temporal context. Answers *"What happened, and when?"* Key characteristics:
- Event-based, each record captures a discrete episode with full temporal context.
- Freshness-indexed — timestamps allow recency-weighted retrieval and TTL-based expiry.
- Contextually rich — records include goal state, participants, attempted actions, and outcomes.
- External and accumulating — lives in vector databases or structured stores.
- Primary input to the consolidation pipeline (Chapter 12).

**Semantic memory** — General, context-free knowledge (facts, concepts, relationships, preferences). Answers *"What do I know in general?"* The specific origin event has been abstracted away.

| Property | Episodic | Semantic |
|---|---|---|
| Answers | What happened? | What is true in general? |
| Binds to time | Yes | No |
| Example record | User auth failed at 14:02 on 2026-03-14 | This user authenticates via OAuth2 |
| Typical store | Vector DB with timestamps | Vector DB or knowledge graph |
| Grows via | Event logging | Consolidation from episodes |
| Primary risk | Staleness / volume growth | Contradiction / over-generalisation |

**Procedural memory** — Skills, workflows, and action sequences. Encodes *how to act*, not *what to know*. Structurally separated from the other four types (see §10.4). Includes: tool usage patterns, workflow templates, action heuristics, learned strategies.

### Design rules
- `MEM-010` — All five memory types MUST be explicitly accounted for in the agent architecture document. Absence of a type is a deliberate decision, not an oversight.
- `MEM-011` — The Part 3 architecture map table above is the canonical type reference. Engineers MUST consult it before making any type-specific storage, retrieval, or TTL decision.

---

## 10.3 In-Context Memory vs. External Memory

### Context
The single most important architectural insight in memory design: in-context and external memory are not alternatives — they are mandatory layers of the same hierarchy. Every production agent uses both.

### What

| Factor | In-context | External |
|---|---|---|
| Access latency needed | ≤ 1 ms required | 50–500 ms acceptable |
| Record size | ≤ 10 K tokens per session | No practical upper limit |
| Persistence required | No / session-scoped acceptable | Yes — survives session end |
| Cross-session access needed | No | Yes |
| Cost sensitivity | High — billed per token | Low — billed per GB stored |
| Retrieval precision | Exact (full attention mechanism) | Approximate (top-k similarity) |
| Typical content | System prompt, current task, recent conversation turns | Past episodes, domain facts, user profiles, task history |

### The Memory Injection Pattern

External memory is always mediated. The agent never reads directly from a database. It retrieves a curated slice and loads that slice into working memory for one reasoning step.

**Standard injection pipeline (7 steps):**
1. **Query** — Formulate a retrieval query from the current task description.
2. **Retrieve** — Fetch the top-20 most relevant records from the external store.
3. **Rerank** — Order the candidates by relevance to the current context.
4. **Select** — Pick the top-5 results.
5. **Format** — Convert each selected record into a compact, labelled summary.
6. **Inject** — Insert the formatted summaries into the context window.
7. **LLM call** — Invoke the LLM with the enriched context.

> **MemGPT tiering model (Packer et al., 2023):** Context window = RAM, external stores = disk. The agent uses explicit function calls to promote records from disk into RAM (retrieval) and demote stale in-context content to disk (archival).

### Why
Retrieval quality is not a secondary concern. A poor retrieval query injects wrong records and the agent reasons on incorrect context. This is on the critical path of every agent reasoning step.

### How (Java + LangGraph)
```java
// SPEC RULE MEM-020
// MemoryInjectionNode — standard LangGraph node for all memory injection
public class MemoryInjectionNode implements Node<AgentState> {

    private final MemorySubsystem memory;
    private static final int TOP_K_RETRIEVE = 20;
    private static final int TOP_K_INJECT   = 5;

    @Override
    public AgentState invoke(AgentState state) {
        String query   = state.currentTask().toRetrievalQuery();
        List<MemoryRecord> candidates = memory.retrieve(query, TOP_K_RETRIEVE);
        List<MemoryRecord> selected   = rerank(candidates).subList(0, TOP_K_INJECT);
        List<String> formatted        = selected.stream()
            .map(r -> String.format("[Memory %s %s] %s",
                r.type(), r.createdAt().toLocalDate(), r.content()))
            .toList();
        return state.withInjectedMemories(formatted);
    }
}
```

**Design rules:**
- `MEM-020` — External memory MUST be injected via a dedicated `MemoryInjectionNode`. No LangGraph node other than this node reads directly from the external store.
- `MEM-021` — Injected records MUST be formatted as compact, labelled summaries with type and date. Raw records MUST NOT be injected verbatim into the context window.
- `MEM-022` — The injection pipeline MUST be instrumented with p99 latency metrics. Memory injection latency counts against the end-to-end response SLA.
- `MEM-023` — The `MemoryInjectionNode` MUST implement a fallback: if the external store is unavailable, the agent proceeds with in-context memory only and emits a `MEMORY_UNAVAILABLE` event.

---

## 10.4 Procedural Memory — Knowing How to Do Things

### Context
Procedural memory is the most underengineered memory type in production agents. Its absence does not manifest as a missing database — it manifests as workflows scattered informally across system prompts, undocumented engineering knowledge in no version control system, and agents that re-derive the same steps from scratch on every task.

### What — Four Implementation Approaches

| Approach | Storage | Update mechanism | Personalisation | Use when |
|---|---|---|---|---|
| **1. System prompt injection** | In-context (system prompt) | Prompt change (global) | None — all users share | Stable, universal procedures |
| **2. Tool library** | Tool definition + schema | Tool versioning | None — schema is shared | Procedures encoded as tool pre/post conditions |
| **3. Workflow templates in external store** | External store (retrieval) | Record update per template | Per-user / per-task | Task-specific or user-personalised procedures |
| **4. Fine-tuning** | Model weights | Full training run | None — global | High-frequency, highly stable procedures only |

**Selection heuristic:**
- Start with **system prompt injection** for stable, universal procedures.
- Use **workflow templates in external store** for task-specific or user-personalised ones.
- Reserve **fine-tuning** for procedures that are both high-frequency and highly stable.

### Why Procedural Memory Gets Overlooked
Engineers overlook it because there is no procedural memory database to provision, no vector index to size, no TTL policy to configure. But the absence of explicit design means it is scattered informally — and agents without deliberately designed procedural memory:
- Re-derive the same steps from scratch on every task.
- Fail inconsistently on edge cases a stored heuristic would catch.
- Silently degrade when the system prompt is edited carelessly by an engineer who did not know which lines were load-bearing.

### How (Java + LangGraph)
```java
// SPEC RULE MEM-030
// ProceduralMemoryStore — retrieves workflow templates by task type
public interface ProceduralMemoryStore {

    /**
     * Retrieve the most relevant workflow template for the given task type.
     * Returns Optional.empty() if no template is available.
     */
    Optional<WorkflowTemplate> retrieveTemplate(String taskType, String userId);

    /**
     * Persist a new or updated workflow template (versioned).
     */
    void writeTemplate(WorkflowTemplate template);
}

// WorkflowTemplate schema
public record WorkflowTemplate(
    String id,
    String taskType,
    String userId,          // null = universal
    int    version,
    String content,         // markdown or structured JSON
    Instant lastUpdated
) {}
```

**Design rules:**
- `MEM-030` — Every agent MUST have an explicit procedural memory inventory: a list of all workflows, heuristics, and action patterns encoded in the system prompt, tool library, or external store.
- `MEM-031` — The system prompt MUST be version-controlled as source code. Every line that encodes a procedure MUST have a comment identifying it as load-bearing.
- `MEM-032` — Task-specific and user-personalised procedures MUST be stored as versioned `WorkflowTemplate` records in an external store, not hardcoded in the system prompt.
- `MEM-033` — Fine-tuning for procedural memory is prohibited unless: (a) the procedure is high-frequency (≥ 10 % of all tasks), (b) it has been stable for ≥ 3 months, and (c) the decision is documented in the agent architecture document.

---

## 10.5 Memory as a First-Class Architectural Component

### Context
The most common memory architecture mistake is building the agent first and adding memory later. Memory added as an afterthought typically collapses in production: context windows overflow under moderate load, retrieval returns irrelevant records, memory updates are inconsistent across components, and there is no operational visibility into what the agent actually remembers.

### What — The Memory Subsystem Interface

A properly architected memory system exposes a well-defined interface. Every other component interacts with memory only through this interface. No component reaches directly into the database.

| Operation | Signature | Description |
|---|---|---|
| `write` | `write(content, type, metadata) → RecordId` | Persist a new record with type classification and metadata (source, timestamp, importance score, TTL). |
| `retrieve` | `retrieve(query, type, topK) → List<MemoryRecord>` | Return the most relevant records for a query and memory type, ranked by similarity + importance score. |
| `update` | `update(id, content) → void` | Revise an existing record. Produces an audit record. |
| `delete` / `expire` | `delete(id)` / `expire(id, ttl)` | Remove immediately or schedule removal. Both produce audit records. |
| `consolidate` | `consolidate(sessionId) → void` | Distil episodic records from a session into durable semantic facts. (Full implementation: Chapter 12.) |

### What — Operational Requirements

| Requirement | Engineering implication |
|---|---|
| Retrieval latency SLA | Memory retrieval is on the critical path of every agent reasoning step. p99 latency MUST be budgeted as part of the agent's end-to-end response SLA. |
| Write durability | Lost writes corrupt agent behaviour silently. Use at-least-once write semantics with idempotency keys. |
| Consistency model | Multiple agent instances sharing memory MUST NOT diverge on critical facts. Define the consistency model before the first multi-agent deployment. |
| TTL and expiry | Unbounded growth degrades retrieval quality and increases cost. Expiry policies MUST be scheduled, not ad hoc. |
| Observability | Engineers MUST be able to inspect what the agent remembers, when it was written, and why it was retrieved — for debugging and compliance. |
| Access control | In multi-user/multi-tenant deployments, memory MUST be isolated per user or tenant at the storage layer, not only in application code. |

### How (Java + LangGraph)
```java
// SPEC RULE MEM-040
// MemorySubsystem — the single interface all agent nodes use
public interface MemorySubsystem {

    RecordId write(String content, MemoryType type, MemoryMetadata metadata);

    List<MemoryRecord> retrieve(String query, MemoryType type, int topK);

    void update(RecordId id, String newContent);

    void delete(RecordId id);

    void expire(RecordId id, Duration ttl);

    void consolidate(String sessionId);
}

// MemoryType enum
public enum MemoryType {
    SENSORY, WORKING, EPISODIC, SEMANTIC, PROCEDURAL
}

// MemoryMetadata
public record MemoryMetadata(
    String   source,           // "user", "tool", "agent_inferred", "document_extracted"
    double   importanceScore,  // composite [0.0, 1.0]
    Instant  expiresAt,        // null = no TTL
    List<String> tags
) {}
```

**Design rules:**
- `MEM-040` — `MemorySubsystem` MUST be defined and stabilised before any agent node is implemented. It is infrastructure, not an afterthought.
- `MEM-041` — No agent node (planner, action layer, reflection module, etc.) MUST reach directly into the memory database. All access MUST be mediated by `MemorySubsystem`.
- `MEM-042` — `MemorySubsystem.retrieve()` MUST expose a Spring Actuator endpoint `/agent/memory/query` for inspection in production.
- `MEM-043` — All `write`, `update`, `delete`, and `expire` operations MUST be logged to a write-immutable audit log. Audit records MUST include: `auditId`, `operation`, `agentId`, `contentHash` (SHA-256), `timestamp`.
- `MEM-044` — The memory subsystem MUST publish Prometheus metrics: `memory_write_total`, `memory_retrieve_latency_ms` (histogram), `memory_retrieve_empty_rate`, `memory_store_record_count` (gauge per type).

### Subsystem Dependencies
Memory has explicit dependencies with every other agent subsystem:
- **Perception layer (Ch. 5)** → writes new observations to sensory and working memory.
- **Reasoning engine (Ch. 6)** → reads from working memory and queries external stores for context.
- **Action layer (Ch. 7)** → logs outcomes back to episodic memory after each action.
- **Reflection module** → drives consolidation from episodic to semantic.
- **State manager (Ch. 8)** → manages in-context working memory directly and owns context window eviction.

> ⚠️ **Memory observability warning:** An agent that misbehaves because of a stale or incorrect memory record is nearly impossible to debug without memory inspection tooling. Build observability into the subsystem from the first deployment, not as a retrofit.

---

## 10.6 The Memory Lifecycle — Write, Store, Retrieve, Expire

### Context
Every memory record passes through four stages. Failures at any stage corrupt the memory system's integrity — and those failures are almost always silent.

### What — Stage 1: Write (The Gatekeeping Decision)

Without an explicit write gate, agents either store everything (memory pollution → retrieval degradation) or store nothing (context lost permanently at session end).

**Five write-gate dimensions (all must pass):**
1. **Significance** — Is this record likely to be useful in a future retrieval?
2. **Type classification** — Episodic, semantic, or procedural? Determines storage backend and TTL policy.
3. **Sensitivity** — Does this record contain PII or confidential content that should not be persisted?
4. **Duplication risk** — Does a semantically equivalent record already exist?
5. **TTL assignment** — What is the appropriate time-to-live given the record's type and expected shelf life?

```java
// SPEC RULE MEM-050
// WriteGate — minimum viability check before persistence
public class WriteGate {

    private static final Set<String> STABLE_TAGS = Set.of(
        "preference", "constraint", "profile", "goal", "decision", "outcome"
    );
    private static final Set<String> TRUSTED_SOURCES = Set.of(
        "user", "tool", "policy", "verified_document"
    );

    /**
     * Returns true only if the record passes all five gate conditions.
     * Any single failure causes the record to be discarded.
     */
    public boolean worthStoring(String content, List<String> tags,
                                String source, String sensitivity) {
        if (content == null || content.strip().length() < 20)  return false; // too short
        if ("high".equals(sensitivity))                        return false; // PII / confidential
        if (!TRUSTED_SOURCES.contains(source))                 return false; // untrusted source
        if (tags.stream().noneMatch(STABLE_TAGS::contains))    return false; // transient, no durable value
        return true;
    }
}
```

### What — Stage 2: Store (Encoding and Indexing)

Once the write decision is made, the record must be encoded for efficient retrieval. For unstructured content, this means generating an embedding (a high-dimensional numeric vector encoding semantic meaning). For structured facts, it means inserting a row with appropriate indexes.

**Mandatory metadata fields at write time:**

| Field | Type | Description |
|---|---|---|
| `createdAt` | `Instant` (UTC, immutable) | Write timestamp |
| `source` | `String` | `user_explicit`, `tool_verified`, `agent_inferred`, `document_extracted` |
| `importanceScore` | `double [0.0, 1.0]` | Composite score computed at write time (formula: Chapter 12) |
| `expiresAt` | `Instant` (nullable) | Wall-clock expiry time derived from TTL policy for this memory type |
| `tags` | `List<String>` | Memory type + domain tags for filtered retrieval |

### What — Stage 3: Retrieve (The Quality-Critical Step)

Retrieval is where memory systems most visibly succeed or fail. Retrieved records are injected into the context window and directly influence what the LLM generates in the next reasoning step.

> ⚠️ **Never inject raw memory records verbatim into the context window.** Format them as compact, labelled summaries with type and freshness indicated. Example: `[Memory preference 2026-01-15] User prefers async Java patterns over threading.`

Full retrieval engineering (similarity metrics, vector database architecture, hybrid retrieval pipelines, retrieval quality measurement) is the exclusive subject of Chapter 11.

### What — Stage 4: Expire (The Discipline of Forgetting)

Expiry is the most underimplemented stage in production memory systems. Without it, stores grow without bound, stale records compete with current ones during retrieval, and storage costs accumulate silently.

**Four expiry mechanisms:**

| Mechanism | Description | Appropriate for |
|---|---|---|
| **TTL-based expiry** | Records deleted or archived after a fixed duration. Predictable and coarse. | Session-scoped episodic records |
| **Freshness decay in retrieval scoring** | Importance scores decay exponentially with age, making old records effectively unreachable before physical deletion. | All external memory types |
| **Importance-threshold pruning** | Scheduled jobs remove records whose importance score has fallen below a minimum threshold. | Long-lived semantic stores |
| **Consolidate-then-delete** | Episodic records distilled into semantic facts (Chapter 12) then deleted. | Episodic → semantic promotion |

> ⚠️ **Forgetting is a design requirement, not a failure mode.** But forgetting the wrong records is a hard-to-diagnose failure mode. Implement expiry conservatively at first — archive before delete, measure retrieval quality before and after each pruning run, and tune thresholds aggressively only once observability is in place.

### How (Java + LangGraph)
```java
// SPEC RULE MEM-060
// MemoryLifecycleScheduler — scheduled expiry jobs
@Component
public class MemoryLifecycleScheduler {

    private final MemorySubsystem memory;

    // Run TTL expiry every hour
    @Scheduled(cron = "0 0 * * * *")
    public void runTtlExpiry() {
        memory.expireAllPastTtl();
    }

    // Run importance-threshold pruning nightly
    @Scheduled(cron = "0 2 * * *")
    public void runImportancePruning() {
        memory.pruneByImportanceThreshold(0.05); // threshold: Chapter 12
    }

    // Run session-close consolidation (triggered by session end event)
    public void onSessionClose(String sessionId) {
        memory.consolidate(sessionId);
    }
}
```

**Design rules:**
- `MEM-050` — The `WriteGate` MUST be the first operation in every `MemorySubsystem.write()` call. Records that fail the gate MUST be silently dropped with a `WRITE_GATE_REJECT` counter increment.
- `MEM-051` — Embedding model selection is a permanent contract with the data. It MUST be documented in the agent architecture document. Switching models requires re-embedding the entire corpus.
- `MEM-052` — All embedding vectors MUST include metadata fields: `createdAt`, `source`, `importanceScore`, `expiresAt`, `tags`. Vectors without metadata are prohibited.
- `MEM-053` — The `MemoryLifecycleScheduler` MUST be deployed from the first production release. Expiry schedules MUST NOT be deferred to a post-launch iteration.
- `MEM-054` — All expiry operations MUST be performed archive-before-delete during the first 90 days of operation. Hard deletion before that window requires an explicit override documented in the architecture decision log.
- `MEM-055` — Retrieval quality MUST be measured (recall@5, MRR, empty-result rate) before and after every pruning run. Pruning that degrades recall@5 by more than 5 % MUST be rolled back.

---

## 10.7 Shared Terminology for Part 3

The following terms are used consistently across Chapters 10–14. Definitions are established here and not repeated later.

| Term | Definition |
|---|---|
| **record** | A single unit of stored memory — an event, a fact, a preference, a task checkpoint, or a workflow template. |
| **freshness** | The recency component of a record's value, modelled as exponential decay from write time. |
| **importance score** | A composite signal combining freshness, access frequency, salience, source reliability, and task relevance. Drives consolidation filtering and pruning. |
| **consolidation** | The process of promoting episodic records into durable semantic facts. |
| **retrieval** | The act of querying stored memory and injecting selected records into the context window for a reasoning step. |
| **persistence** | Durable storage that survives session termination. |
| **conflict** | Two records that assert contradictory facts about the same entity. |
| **audit** | The write-immutable log of every memory operation. |

---

## 10.8 Design Rules Summary

| Rule ID | Scope | Rule |
|---|---|---|
| `MEM-001` | Subsystem | Every production agent MUST implement at least working memory (in-context) + one external memory tier. Context-injection-only is prohibited above demo scale. |
| `MEM-002` | Subsystem | Memory architecture MUST be designed before the agent components that depend on it. |
| `MEM-010` | Taxonomy | All five memory types MUST be explicitly accounted for in the agent architecture document. |
| `MEM-011` | Taxonomy | The Part 3 architecture map table is the canonical type reference. |
| `MEM-020` | Injection | External memory MUST be injected via a dedicated `MemoryInjectionNode`. No other node reads directly from the external store. |
| `MEM-021` | Injection | Injected records MUST be formatted as compact, labelled summaries. Raw records MUST NOT be injected verbatim. |
| `MEM-022` | Injection | Injection pipeline MUST be instrumented with p99 latency metrics. |
| `MEM-023` | Injection | `MemoryInjectionNode` MUST implement a fallback for store unavailability. |
| `MEM-030` | Procedural | Every agent MUST have an explicit procedural memory inventory. |
| `MEM-031` | Procedural | System prompt MUST be version-controlled as source code. |
| `MEM-032` | Procedural | Task-specific and user-personalised procedures MUST be stored as versioned `WorkflowTemplate` records. |
| `MEM-033` | Procedural | Fine-tuning for procedural memory is prohibited unless three criteria are met and documented. |
| `MEM-040` | Interface | `MemorySubsystem` MUST be defined and stabilised before any agent node is implemented. |
| `MEM-041` | Interface | No agent node MUST reach directly into the memory database. |
| `MEM-042` | Interface | `MemorySubsystem.retrieve()` MUST expose a Spring Actuator inspection endpoint. |
| `MEM-043` | Interface | All write/update/delete/expire operations MUST be logged to a write-immutable audit log. |
| `MEM-044` | Interface | Memory subsystem MUST publish four Prometheus metrics. |
| `MEM-050` | Lifecycle | `WriteGate` MUST be the first operation in every `write()` call. |
| `MEM-051` | Lifecycle | Embedding model selection is permanent. Switching requires full re-embedding. |
| `MEM-052` | Lifecycle | All embedding vectors MUST include the five mandatory metadata fields. |
| `MEM-053` | Lifecycle | `MemoryLifecycleScheduler` MUST be deployed from the first production release. |
| `MEM-054` | Lifecycle | All expiry operations MUST be archive-before-delete for the first 90 days. |
| `MEM-055` | Lifecycle | Retrieval quality MUST be measured before and after every pruning run. |

---

## 10.9 Transition to Chapter 11

This chapter established the vocabulary, architecture map, and subsystem contract that all of Part 3 is built on. Every term defined here — record, freshness, importance score, consolidation, retrieval, persistence, conflict, audit — is used without redefinition in Chapters 11–14.

Chapter 11 moves from taxonomy to implementation. It builds the concrete mechanisms that make the external memory tier queryable at production scale:
- How embedding models encode meaning as geometry.
- How similarity metrics define distance between vectors.
- How vector databases index and serve nearest-neighbour queries at millions of records.
- How knowledge graphs complement vector search for relational semantic memory.
- How hybrid retrieval pipelines combine all three into the 2026 production standard.
- How to select among the four dominant production vector database options.
