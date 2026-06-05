# Chapter 5 — The Memory Layer

**Part 2: Anatomy of an AI Agent**  
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

> *"An intelligent agent is not defined by what it can compute. It is defined by what it can remember."*

Memory is the architectural property that separates a reactive function from an intelligent system. A stateless agent that discards all context after each call is executing sophisticated computation — but it accumulates nothing. Every agent that retains, retrieves, and builds on past experience is doing something qualitatively different: it is accumulating intelligence over time.

This chapter specifies the complete memory architecture a production agent requires: the five-type taxonomy, the mandatory in-context/external hierarchy, the subsystem interface contract, the four-stage lifecycle, and all operational requirements for a Java + LangGraph implementation.

---

## 5.1 — Why Memory Is the Defining Feature

### Context

Most early production agent deployments were effectively stateless. Each API call carried a fresh context window populated with the current conversation and nothing else. That is not memory — it is context injection, and it breaks the moment the conversation exceeds the context window, the session ends, or a different user starts a new thread.

### What (the spec)

Memory is a **structural architectural property**, not a feature that can be added after deployment. The four engineering consequences of absent memory:

| Absent memory type | Capability permanently lost |
|---|---|
| No episodic memory | Cannot resume interrupted tasks; cannot learn from past failures |
| No semantic memory | Cannot accumulate domain knowledge beyond training data cutoff |
| No procedural memory | Cannot store, reuse, or refine workflows |
| No external memory (any type) | Effective knowledge horizon capped by context window size |

**Memory determines four agent quality axes:**
1. **Information availability** — what the agent knows at any given moment
2. **Retrieval quality** — how quickly and accurately relevant context is surfaced
3. **Adaptation** — how agent behaviour changes as knowledge accumulates
4. **Continuity** — how reliably the agent operates across sessions, users, and failure modes

### Why (engineering implications)

- Two agents can share the same LLM backbone, the same tool set, and the same planning algorithm. What distinguishes the capable one from the naive one is almost always the quality of its memory architecture.
- Context window sizes have grown significantly — frontier models support 200K to 2M tokens as of 2026. This reduces urgency for external memory in short sessions. It does not eliminate it. Long-running agents, multi-session continuity, user modelling, and cost management all demand external memory regardless of window size.
- The most common memory architecture mistake is building the agent first and adding memory later. Memory added as an afterthought produces context windows that overflow under moderate load, retrieval that returns irrelevant records, inconsistent updates, and no operational visibility into what the agent actually remembers.

### How (system design rules)

1. **Design the memory architecture before building any agent component** that depends on it.
2. **Treat memory as a subsystem** with its own interface, operational SLAs, and observability — not as a utility function scattered through the codebase.
3. **Ask this diagnostic question before each implementation decision**: *Does this agent's output quality improve if it knows what happened in previous interactions?* If yes, the answer is statefulness — design it deliberately.
4. **Never substitute context injection for memory architecture.** Prepending conversation history to every prompt is not a memory system. It is a buffer that fails at scale.

---

## 5.2 — The Five Memory Types

### Context

The taxonomy draws from cognitive psychology — Atkinson and Shiffrin's multi-store model (1968) and Tulving's episodic/semantic distinction (1972) — adapted here as functional engineering archetypes. Each type solves a distinct engineering problem. Without a precise vocabulary, architectural decisions about storage, retrieval, consolidation, and persistence are made informally and expensively.

### What (the spec)

**The Part 3 architecture map — canonical reference for all memory design decisions:**

| Type | Storage tier | Lifetime | Access pattern | Primary engineering concern |
|---|---|---|---|---|
| **Sensory** | Perception buffer (transient) | Milliseconds | Implicit pre-filter | Signal-to-noise at ingestion |
| **Working** | Context window (in-context) | Single session | Zero-latency | Token budget overflow and eviction order |
| **Episodic** | External hot/warm vector DB | Days to months | Similarity retrieval | Volume growth, freshness, consolidation |
| **Semantic** | External hot vector DB + graph | Months to indefinite | Similarity + graph traversal | Contradiction, confidence, freshness |
| **Procedural** | System prompt + tool library + external store | Stable | Direct injection or retrieval | Inconsistency, undocumented drift |

**Type definitions:**

- **Sensory memory**: Transient buffering of raw inputs at the perception layer. Short-lived, high-bandwidth, discarded after relevance filtering. Rarely modelled explicitly; appears in any system that buffers streaming input before loading it into the context window. Primary concern: signal-to-noise — raw external content that reaches working memory unchecked degrades all downstream reasoning.

- **Working memory**: The active, in-context information available during a single reasoning step or session. Maps directly to the context window. Zero-latency; session-scoped. Primary concern: token budget management — what gets loaded, what gets evicted, how it is summarised when the window fills.

- **Episodic memory**: Specific events, interactions, and experiences with their temporal context. Answers *"What happened, and when?"* Event-based; freshness-indexed; contextually rich; accumulating. External store required. Primary concern: volume growth and freshness — episodic records are the primary input to the consolidation pipeline.

- **Semantic memory**: General, context-free knowledge — facts, concepts, relationships, preferences — without binding to a specific event. Answers *"What do I know in general?"* Where episodic memory says *"Last week the user told me they prefer TypeScript"*, semantic memory says *"This user codes in TypeScript"*. Primary concern: contradiction and over-generalisation.

- **Procedural memory**: Skills, workflows, and action sequences. Answers *"How do I do this?"* Structurally separated from the other four types because it does not participate in the retrieve-and-inject pipeline — its access pattern is direct execution, not similarity search.

### Why (engineering implications)

- Episodic and semantic memory are easy to conflate. The distinction is critical: episodic records expire (they become stale); semantic facts are durable (they represent current truth). The consolidation pipeline converts episodic records into semantic facts before they expire. A system that treats all external memory as a single store will degrade retrieval quality as stale episodic records pollute the semantic search space.
- Procedural memory is the most underengineered type in production agents. Its absence is invisible: the agent re-derives the same steps from scratch on every task, fails inconsistently on edge cases that a stored heuristic would catch, and silently degrades when the system prompt is edited carelessly.
- Sensory memory appears in every system that processes streaming input. The difference between a system that engineers it and one that does not is the difference between a noise-filtered context window and a polluted one.

### How (system design rules)

1. **Use the Part 3 architecture map as the canonical design reference** for every memory implementation decision. Return to it when type-specific design decisions arise.
2. **Treat episodic and semantic memory as separate stores** with separate TTL policies, separate retrieval pipelines, and a consolidation process connecting them.
3. **Version procedural memory as source code.** System prompt instructions, tool library definitions, and workflow templates must all be versioned and change-controlled.
4. **Implement sensory filtering as the first step** in `PerceiveNode` — not as an optional optimisation.
5. **In Java + LangGraph**: declare memory type as a mandatory classification field (`MemoryType` enum) on every `MemoryRecord`. The storage backend is selected by type at write time.

---

## 5.3 — Episodic vs. Semantic Memory

### Context

The episodic/semantic distinction is operationally significant. These two types require different storage backends, different retrieval strategies, different TTL policies, and a structured handoff process between them (consolidation). Treating them as a single undifferentiated external store is the most common memory design error in production agents.

### What (the spec)

| Property | Episodic | Semantic |
|---|---|---|
| Answers | *What happened?* | *What is true in general?* |
| Binds to time | Yes | No |
| Example record | `User auth failed at 14:02 on 2026-03-14` | `This user authenticates via OAuth2` |
| Typical store | Vector DB with timestamps | Vector DB or knowledge graph |
| Grows via | Event logging | Consolidation from episodes |
| Primary risk | Staleness, volume growth | Contradiction, over-generalisation |
| TTL | Days to months | Months to indefinite |
| Consolidation role | Source | Target |

**Consolidation pipeline (episodic → semantic):**

At session close (or on a scheduled consolidation run), episodic records are evaluated by importance score, filtered, and distilled into durable semantic facts:
1. **Importance filter** — only records above the minimum importance threshold proceed.
2. **Deduplication** — records semantically equivalent to existing semantic facts are discarded.
3. **Conflict check** — records that contradict existing semantic facts enter the conflict resolution pipeline.
4. **Promotion** — surviving records are written to the semantic store with updated metadata.
5. **Archive/delete** — source episodic records are archived or deleted after successful promotion.

### Why (engineering implications)

- A semantic store that accumulates stale episodic records will degrade retrieval quality progressively. Stale records compete with current ones during similarity search. The degradation is gradual and silent — by the time it is visible in agent behaviour, it has been accumulating for weeks.
- The consolidation pipeline is idempotent. A session-close consolidation that runs twice must produce the same result as one that runs once. This is a hard requirement for retry safety.
- Contradiction detection is the most important quality gate in the consolidation pipeline. A semantic store full of contradictory beliefs produces inconsistent, confidence-degrading agent behaviour.

### How (system design rules)

1. **Run consolidation at session close AND on a scheduled batch cycle** (at minimum daily). Do not rely on session-close-only consolidation — sessions can end abnormally.
2. **Make the consolidation pipeline idempotent.** Use content hashing and idempotency keys to prevent duplicate semantic facts.
3. **Implement a four-policy conflict resolution hierarchy** for contradictions: (1) source authority, (2) recency, (3) confidence, (4) human arbitration.
4. **Set TTL policies per type at write time** — not as a global configuration. Different episodic record categories warrant different TTLs.
5. **In Java + LangGraph**: a `ConsolidationService` runs as a scheduled job and is also called by the session-close handler. It reads from the episodic store, applies the importance filter and conflict check, and writes surviving records to the semantic store with updated `MemoryRecord` metadata.

---

## 5.4 — Procedural Memory: The Most Underengineered Type

### Context

Procedural memory encodes skills, workflows, and action sequences — not what to know, but how to do. Agents without deliberately designed procedural memory re-derive the same steps from scratch on every task, fail inconsistently on edge cases a stored heuristic would catch, and silently degrade when the system prompt is edited carelessly by an engineer who did not know which lines were load-bearing.

### What (the spec)

Procedural memory includes:
- **Tool usage patterns** — how to call a specific API, in what order, with what error handling and retry logic
- **Workflow templates** — the standard step sequence for a class of tasks
- **Action heuristics** — rules such as *"if a query returns more than 1,000 rows, add a LIMIT clause before executing"*
- **Learned strategies** — generalised approaches distilled from many past episodes

**Four implementation approaches:**

| Approach | Mechanism | Flexibility | Update cost | Best for |
|---|---|---|---|---|
| **System prompt injection** | Procedures encoded directly in system prompt | Low — same for all users | Prompt change = re-deploy | Stable, universal procedures |
| **Tool library** | Tool schemas, descriptions, and pre/post-condition checks | Medium — per tool | Tool redeploy | Skill-level procedures |
| **Workflow templates** | Named templates in external retrieval store | High — per user/task | Write to store | Task-specific or personalised procedures |
| **Fine-tuning** | Procedures baked into model weights | None — model-level | Full training run | High-frequency, highly stable procedures |

### Why (engineering implications)

- Procedural memory does not map cleanly onto a single storage mechanism and does not participate in the retrieve-and-inject pipeline that governs episodic and semantic memory. This structural difference justifies treating it as a separate design domain.
- The absence of explicit procedural memory design does not mean the absence of procedural memory — it means it is scattered informally across system prompts, tool definitions, and undocumented engineering knowledge that lives in no version control system.
- System prompt injection is the most common form of procedural memory in production agents. It is also the most dangerous: a single careless edit can silently remove a load-bearing heuristic that was protecting against a class of failures no one remembers.

### How (system design rules)

1. **Start with system prompt injection for stable, universal procedures** and workflow templates in an external store for task-specific or user-personalised ones. Reserve fine-tuning for procedures that are both high-frequency and highly stable.
2. **Version system prompt content as source code.** Treat every line that encodes a procedure or heuristic as a versioned, tested rule — not as free text.
3. **Document which system prompt lines are load-bearing.** Annotate each procedural instruction with its purpose, the failure mode it prevents, and its test coverage.
4. **Store workflow templates as named, versioned records** in the external memory store. Retrieve them by task type at cycle start.
5. **In Java + LangGraph**: `ToolRegistry` entries carry typed pre/post-condition schemas. Workflow templates are stored as `ProceduralRecord` objects with `taskType`, `version`, `steps`, and `heuristics` fields. `PerceiveNode` retrieves the relevant workflow template at cycle start and injects it into the system prompt slot.

---

## 5.5 — In-Context Memory vs. External Memory: The Mandatory Hierarchy

### Context

The single most important architectural insight in memory design: in-context and external memory are not alternatives — they are mandatory layers of the same hierarchy. Every production agent uses both. The engineering challenge is managing what crosses the boundary between them, in which direction, and at what cost.

### What (the spec)

The boundary is a hard architectural constraint imposed by the LLM's token limit:

| Factor | In-context | External |
|---|---|---|
| Access latency | ~1ms (directly attended) | 50–500ms (retrieval step required) |
| Record size limit | ~10K tokens per session | No practical upper limit |
| Persistence required | No — session-scoped acceptable | Yes — must survive session end |
| Cross-session access | No | Yes |
| Cost sensitivity | High — billed per token | Low — billed per GB stored |
| Retrieval precision | Exact (full attention mechanism) | Approximate (top-k similarity) |
| Typical content | System prompt, current task, recent turns | Past episodes, domain facts, user profiles, task history |

**The memory injection pattern (standard six-step process):**

1. **Query** — formulate a retrieval query from the current task description
2. **Retrieve** — fetch top-20 most relevant records from the external store
3. **Rerank** — order candidates by relevance to current context
4. **Select** — pick top-5 results
5. **Format** — convert each selected record into a compact, labelled summary
6. **Inject** — insert formatted summaries into the context window `EVIDENCE` slot

**Labelling rule (mandatory):** Inject memories as clearly labelled context, never as instructions:
```
Memory [episodic 2026-01-15]: User prefers async Java patterns over thread pools.
```

### Why (engineering implications)

- The retrieval step is a **point of failure**. A poor retrieval query injects wrong records and the agent reasons on incorrect context. Memory retrieval quality directly controls agent behaviour quality.
- **Prompt drift** is the subtlest and most dangerous memory failure mode. It is not a crash or a data loss event. It is a slow behavioural change driven by retrieved memories that are treated as implicit instructions. Explicit labelling is the primary defence.
- Injecting raw memory records verbatim wastes tokens, introduces formatting noise, and makes it harder for the LLM to distinguish remembered facts from live context. Compact, labelled summaries cost a fraction of the tokens and are unambiguous.
- The MemGPT tiering model (Packer et al., 2023) formalises this as an OS analogy: context window is RAM, external stores are disk. The agent uses explicit function calls to promote records from disk into RAM (retrieval) and demote stale in-context content to disk (archival).

### How (system design rules)

1. **Define the in-context/external boundary explicitly** before building any retrieval pipeline. Decide what content belongs in each tier, what retrieval latency is acceptable, and how the injection pipeline mediates the crossing.
2. **Never inject raw records into the context window.** Always format as compact, labelled summaries with type and freshness indicated.
3. **Use the four-question injection policy** before including any content: (1) Is it required for this cycle? (2) Is it more relevant than what it would displace? (3) Is it in the most compressed useful form? (4) Does it belong in the correct slot?
4. **Limit injected records per cycle.** A default of top-5 is a reasonable starting point; tune based on measured task quality vs. context cost trade-offs.
5. **In Java + LangGraph**: a `MemoryRetrievalService` is called by `PerceiveNode` at each cycle. It executes the six-step injection pipeline and returns a `List<FormattedMemoryRecord>` for insertion into the `EVIDENCE` slot. Retrieval latency is measured and emitted as a metric.

---

## 5.6 — The Memory Subsystem Interface

### Context

A properly architected memory system exposes a well-defined interface to the rest of the agent. Every other component — the planner, the action layer, the reflection module — interacts with memory only through this interface. No component reaches directly into the database. The interface is the boundary that makes the memory subsystem independently testable, independently deployable, and independently observable.

### What (the spec)

**Core interface operations:**

| Operation | Signature | Responsibility |
|---|---|---|
| `write` | `(content, type, metadata) → RecordId` | Persist a new record after write-gate evaluation |
| `retrieve` | `(query, type, topK) → List<MemoryRecord>` | Return top-K most relevant records ranked by similarity + importance |
| `update` | `(id, content) → void` | Revise an existing record; produces audit record |
| `delete` / `expire` | `(id) / (id, ttl) → void` | Remove immediately or schedule; both produce audit records |
| `consolidate` | `(sessionId) → ConsolidationResult` | Distil episodic records into durable semantic facts |

**Write-gate evaluation (five mandatory checks before any `write`):**
1. **Significance** — is this record likely to be useful in a future retrieval?
2. **Type classification** — is this episodic, semantic, or procedural? (Determines backend and TTL)
3. **Sensitivity** — does this record contain PII or confidential content that must not be persisted?
4. **Duplication risk** — does a semantically equivalent record already exist?
5. **TTL assignment** — what is the appropriate time-to-live given the record's type?

**Operational SLAs (non-negotiable for production):**

| Requirement | Engineering implication |
|---|---|
| Retrieval latency SLA | Memory retrieval is on the critical path of every reasoning step; p99 latency must be budgeted in end-to-end SLA |
| Write durability | Lost writes corrupt agent behaviour silently; use at-least-once semantics with idempotency keys |
| Consistency model | Multiple agent instances sharing memory must not diverge on critical facts; define model before first multi-agent deployment |
| TTL and expiry | Unbounded growth degrades retrieval quality; expiry must be scheduled, not ad hoc |
| Observability | Inspect what the agent remembers, when it was written, why it was retrieved — for debugging and compliance |
| Access control | In multi-user deployments, memory must be isolated per user/tenant at the storage layer |

### Why (engineering implications)

- Memory observability is consistently cut in early implementations and consistently demanded during the first production incident. An agent that misbehaves because of a stale or incorrect memory record is nearly impossible to debug without memory inspection tooling.
- Access control is not an application-layer concern — it is a storage-layer requirement. Enforcing user isolation only in application code allows a compromised agent component to read another user's memories directly from the database.
- The `consolidate` operation is listed in the interface to complete the contract. Full consolidation logic (importance scoring, summarisation, conflict resolution) is a separate implementation concern.

### How (system design rules)

1. **Define the memory interface as a Java interface** (`MemoryStore`) before implementing any storage backend. All agent components depend on the interface, not the implementation.
2. **Implement write-gate evaluation as a `WriteGate` component** called before every `write` operation. Gate failures return a `WriteRejection` event, not an exception.
3. **Enforce operational SLAs via observable metrics**: retrieval p99 latency, write durability (retry count), TTL expiry job success rate, and per-user record count.
4. **Build memory observability from day one**: expose a `MemoryInspector` API that returns the agent's memory state (by type, by session, by importance score) for debugging and compliance.
5. **In Java + LangGraph**: `MemoryStore` is a Spring-managed interface with `VectorMemoryStore` (episodic/semantic) and `ProceduralMemoryStore` (templates, heuristics) implementations. `WriteGate` is a Spring component injected into `MemoryStore`. All operations emit `MemoryAuditRecord` events.

---

## 5.7 — The Memory Lifecycle: Write, Store, Retrieve, Expire

### Context

Every memory record passes through four stages: Write, Store, Retrieve, and Expire. Failures at any stage corrupt the memory system's integrity — and those failures are almost always silent. The lifecycle is a pipeline, not a point-in-time event. Each stage is independently engineered and independently observable.

### What (the spec)

**Stage 1 — Write (gatekeeping):**
- Five-gate evaluation (Section 5.6).
- On pass: classify type, enrich metadata, assign TTL, compute initial importance score.
- On fail: emit `WriteRejection` event, discard silently.

**Stage 2 — Store (encoding and indexing):**
- Generate embedding for unstructured content (vector similarity retrieval).
- Insert structured metadata: `createdAt`, `source`, `importanceScore`, `ttlExpiresAt`, `tags`, `memoryType`.
- Write to the correct backend (vector DB for episodic/semantic; structured store for procedural).
- Emit `WriteAuditRecord`.

**Stage 3 — Retrieve (quality-critical):**
- Execute the six-step injection pipeline (Section 5.5).
- Rank by composite score: similarity × freshness weight × importance score.
- Format as compact labelled summaries before injection — never inject raw records.
- Emit `RetrievalAuditRecord` with query, topK candidates, selected records, and latency.

**Stage 4 — Expire (discipline of forgetting):**

Four expiry mechanisms, used in combination:

| Mechanism | Trigger | Granularity | Primary use |
|---|---|---|---|
| **TTL-based expiry** | Wall-clock time | Per record | Session-scoped episodic records |
| **Freshness decay** | Continuous (retrieval scoring) | Score adjustment | Makes old records effectively unreachable |
| **Importance-threshold pruning** | Scheduled job (daily minimum) | Per record | Removes records below minimum importance |
| **Consolidate-then-delete** | Session close + scheduled batch | Per session | Converts episodic to semantic, then deletes source |

### Why (engineering implications)

- Expiry is the most underimplemented stage in production memory systems. Without it, stores grow without bound, stale records compete with current ones during retrieval, and storage costs accumulate silently.
- **Forgetting is a design requirement, not a failure mode.** But forgetting the wrong records is a hard-to-diagnose failure mode. Implement expiry conservatively at first (archive before delete), measure retrieval quality before and after each pruning run, and tune aggressively only once observability is in place.
- Freshness decay in scoring is a complement to, not a substitute for, physical expiry. Decayed records that still exist in the store consume storage and can be retrieved by unusual query patterns.

### How (system design rules)

1. **Implement all four expiry mechanisms.** TTL-based expiry and importance-based pruning are separate mechanisms with separate triggers, separate implementations, and separate operational schedules.
2. **Archive before delete for the first 90 days of production.** Move expired records to cold storage (S3, GCS) before physical deletion. This provides a recovery path when expiry is misconfigured.
3. **Run a daily importance-threshold pruning job.** Measure retrieval quality (precision@5 on a golden set) before and after each run to detect when thresholds are too aggressive.
4. **Make expiry observable**: emit `ExpiryAuditRecord` for every physical deletion; dashboard the store's record count by type and age daily.
5. **In Java + LangGraph**: `ExpiryScheduler` is a Spring `@Scheduled` component running at configurable intervals per memory type. `ConsolidationService` is called by `ExpiryScheduler` before any episodic delete. Archive writes are routed to the cold tier (`S3ArchiveStore`) before deletion.

---

## 5.8 — Stateful vs. Stateless Agent Architecture

### Context

Statefulness is an emergent property of a correctly designed and operated memory architecture, not a mode an agent enters. When memory is correct, complete, and consistently maintained, the agent becomes stateful as a consequence. When memory is absent, inconsistent, or poorly retrieved, the agent remains stateless regardless of the complexity of its reasoning engine.

### What (the spec)

**When to choose stateless:**
- Task is isolated and atomic: single-turn classification, one-shot code explanation, standalone document summarisation.
- Agent output quality does not improve with knowledge of previous interactions.
- Horizontal scaling simplicity and stateless operational model outweigh continuity benefits.

**When to choose stateful:**
- Agent's output quality improves with accumulated interaction history.
- Tasks span multiple sessions, require user modelling, or involve resumable long-horizon execution.
- Personalisation, task continuity, or multi-agent coordination is a requirement.

**Five failure modes of stateful architecture (design for all five before deployment):**

| Failure mode | Cause | Prevention |
|---|---|---|
| **Parallel overwrites** | Two agents update the same record simultaneously | Optimistic locking with version-checked writes |
| **Partial updates** | Session-close consolidation interrupted mid-run | Idempotent consolidation pipeline with idempotency keys |
| **Race conditions** | Concurrent access to shared memory without locking | Record-level locking or conflict-resolution policy |
| **Prompt drift** | Retrieved memories injected as implicit instructions | Explicit labelling; memories never override system prompt |
| **Lost state on retry** | Task fails mid-execution without checkpoint | Task-state checkpoint written at each significant milestone |

### Why (engineering implications)

- Statelessness is a legitimate, correct architectural choice for atomic tasks — not a default that stateful agents always supersede. Choose deliberately based on whether accumulated history improves output quality.
- The five failure modes of stateful architecture are not theoretical. They appear in the first production deployment of any multi-step or multi-session agent. Designing for them before they appear is the difference between a planned migration and a production incident.
- Prompt drift is the subtlest failure mode. It is a slow behavioural change, not a crash. By the time it is visible, it has been corrupting agent behaviour for weeks.

### How (system design rules)

1. **Choose statelessness deliberately for atomic tasks.** Document the choice and its rationale.
2. **Choose statefulness deliberately for persistent agents.** Document the five failure modes and their mitigations in the architecture record.
3. **Implement optimistic locking** (`version`-checked writes) for all shared memory write operations.
4. **Write task-state checkpoints at each significant execution milestone** — not only at session close.
5. **Enforce memory labelling as a non-negotiable architectural rule**: injected memories are always labelled context; they never appear as system instructions.
6. **In Java + LangGraph**: `MemoryStore.write` is version-checked via `OptimisticLockingMemoryStore`. Task checkpoints are written to `AgentState.taskCheckpoint` and persisted by the LangGraph `PostgreSQLCheckpointer`. Prompt drift prevention is enforced in `PromptAssembler` — memories are always placed in the `EVIDENCE` slot, never in the `ROLE` or `ACTIVE_GOAL` slots.

---

## Key Takeaways for Volume 2 Implementation

- Memory is a **first-class architectural subsystem** with a defined interface (`MemoryStore`), operational SLAs, and observability from the first deployment.
- **Five memory types** — sensory, working, episodic, semantic, procedural — each solve a distinct engineering problem and map to distinct storage backends and access patterns.
- **Episodic and semantic memory are separate stores** connected by a scheduled, idempotent `ConsolidationService`. Treating them as one store is the most common memory design error.
- **Procedural memory is the most underengineered type.** Every workflow template and action heuristic must be versioned, annotated, and stored in a retrievable form.
- The **in-context/external boundary** is a hard architectural constraint. The six-step injection pipeline mediates every crossing. Records are always injected as compact labelled summaries — never raw.
- The **write gate** (five checks) is the first line of defence against memory pollution. The **four expiry mechanisms** are the last line. Both must be implemented from day one.
- **Statefulness is an emergent property** of correct memory architecture, not a flag to set. Design for the five failure modes before deployment.
- **Memory observability** (`MemoryInspector`, `MemoryAuditRecord`, retrieval metrics) is non-negotiable. It will be the most-demanded debugging tool after the first production incident.
