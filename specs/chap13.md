# Chapter 13 Spec — Memory as the Foundation for Statefulness

> **Volume 1 reference:** `chmemory-as-the-foundation-for-statefulness`  
> **Part:** 3 — Memory Architecture  
> **Depends on:** chap10 (taxonomy), chap11 (storage & retrieval), chap12 (lifecycle)  
> **Java + LangGraph implementation target:** Volume 2

---

## Chapter Purpose

Chapter 12 built and operationalised the memory infrastructure.  
Chapter 13 asks: **what agent behaviours does correctly managed memory make possible?**

The answer is **statefulness** — the emergent property that an agent's behaviour at any moment reflects accumulated interaction history, current held goals, and a persistent model of the user or environment.

> Statefulness is not a mode an agent enters. It is an emergent property of a correctly designed and operated memory architecture.

---

## Section 13.1 — Stateless vs. Stateful Agents: The Architectural Divide

### Context

Early production agent deployments were effectively stateless: each API call carried a fresh context window populated with the current conversation and nothing else. That is not memory — it is context injection, and it breaks the moment the conversation exceeds the context window, the session ends, or a different user starts a new thread.

### What

| Factor | Stateless | Stateful |
|---|---|---|
| Memory between calls | None — all context passed per request | Persistent, retrieved from external store |
| Context source | Current request only | Current request + retrieved history |
| Scaling model | Trivial horizontal scaling | Sticky sessions or partitioned state stores |
| Implementation complexity | Low — simple request handler | High — storage, retrieval, consistency, lifecycle |
| Failure modes | Prompt overflow / token limit / lost context | Stale state, stale reads, race conditions, prompt drift, state loss on retry |
| Correct for | Atomic, isolated, one-shot tasks | Multi-step workflows, personalised assistants, long-running tasks |

### Why

Statelessness is the **correct deliberate choice** for agents performing isolated, atomic tasks (single-turn classification, standalone summarisation). For those workloads, statefulness is overhead without benefit. The engineering question is: *does this agent's output quality improve if it knows what happened in previous interactions?* If yes, stateless is a design deficiency.

Stateful behaviour requires three things:
1. **Persistent storage** — the tiered hot/warm/cold architecture (chap12)
2. **Retrieval** — hybrid pipelines that surface relevant history at each reasoning step (chap11)
3. **Lifecycle management** — consolidation, pruning, consistency (chap12)

Without all three, a system can persist data without being genuinely stateful — it becomes an agent with an increasingly noisy, degrading memory store.

### Five Production Failure Modes of Stateful Agents

| # | Failure Mode | Root Cause | Prevention |
|---|---|---|---|
| 1 | **Stale state from parallel overwrites** | Two concurrent instances read, compute, write conflicting results; last write wins | Optimistic locking or write-ahead logging at the memory store layer |
| 2 | **Partial updates** | Session-close consolidation pipeline writes some facts then fails; non-idempotent restart duplicates records | Design all memory write pipelines as idempotent operations with deduplication keys |
| 3 | **Race conditions on shared memory** | Two agents attempt to update the same record simultaneously | Record-level locking or conflict-resolution policies applied atomically |
| 4 | **Prompt drift** | Retrieved memories injected into context window gradually shift the agent's effective instruction set away from system prompt design intent | Inject memories as clearly labelled context (`Memory [preference 2026-01-15]`), not as instructions |
| 5 | **Lost state across retries** | Agent task fails mid-execution; retry starts from scratch — no checkpoint was written | Write task-state checkpoints at each significant execution milestone |

> ⚠️ **Prompt drift is the subtlest and most dangerous failure mode.** It is not a crash or a data loss event. It is a slow behavioural change driven by retrieved memories that are treated as implicit instructions. Always label injected memories explicitly.

### Design Rules

| Rule ID | Rule |
|---|---|
| `SF-001` | Choose statelessness deliberately for atomic tasks. The test: does output quality improve with accumulated history? If no, stateless is correct. |
| `SF-002` | Stateful architectures require all three foundations: persistent storage, retrieval, and lifecycle management. Presence of storage alone is not statefulness. |
| `SF-003` | All memory write pipelines MUST be idempotent with deduplication keys. |
| `SF-004` | Injected memories MUST be labelled as context, not instructions. Use a consistent prefix: `Memory [type date]:`. |
| `SF-005` | Task checkpoints MUST be written at each significant execution milestone before any potentially failing operation. |

### Java + LangGraph Target

```java
// AgentState fields that distinguish stateless vs. stateful nodes
public record AgentState(
    String sessionId,
    String userId,
    List<String> currentMessages,           // working memory
    List<RetrievedRecord> injectedMemories, // labelled, confidence-filtered
    Optional<TaskCheckpoint> taskCheckpoint,
    StatefulPolicy policy
) {}

// StatefulPolicy governs what the agent is allowed to persist
public record StatefulPolicy(
    boolean persistEpisodicMemory,
    boolean persistUserModel,
    boolean checkpointTasks,
    double minInjectionConfidence  // STAT-001: filter before injection
) {}
```

---

## Section 13.2 — Session State vs. Cross-Session Persistence

### Context

Within a session, working memory is available in the context window and all reasoning steps share the same accumulated context. The **session boundary** is the critical engineering transition: what happens to session-scoped state when the session ends?

### What

- **Session state** — context valuable during the session but not beyond it: working context, intermediate reasoning steps, in-progress drafts. Discarded at session end or after a configurable timeout. Backend: **Redis** (sub-millisecond reads, TTL-based automatic expiry).
- **Cross-session persistence** — information that must survive session boundaries: user preferences, task progress, established facts. Written to the tiered external store at session close via the consolidation pipeline (chap12 §12.1).

### The Four-Step Session-Close Handoff

The order is non-negotiable:

```
1. CONSOLIDATE  → run session-close pipeline (chap12) to promote episodic records to semantic store
2. CHECKPOINT   → write current task progress to task store for any in-progress tasks
3. FLUSH        → clear the context window
4. EXPIRE       → delete or allow TTL expiry of session-scoped Redis state
```

> Expiring session state **before** consolidating means consolidation has no source data.  
> Checkpointing **before** consolidation means the task checkpoint may reference episodic records not yet promoted, leaving dangling references.

### Why

Redis is the production standard for in-session state: sub-millisecond reads, TTL-based automatic expiry at session close, and atomic operations for concurrent update safety.

### Design Rules

| Rule ID | Rule |
|---|---|
| `SS-001` | Session state MUST use a fast, TTL-governed backend (Redis or equivalent). Default TTL: 2 hours of inactivity. |
| `SS-002` | The four-step session-close handoff sequence MUST be executed in order: consolidate → checkpoint → flush → expire. |
| `SS-003` | All session-close steps MUST be transactional or compensatable; a failed consolidation step MUST NOT allow expiry of the source session state. |
| `SS-004` | Session state keys MUST be namespaced by `sessionId` and MUST NOT be shared across sessions. |

### Java + LangGraph Target

```java
public interface SessionStateManager {
    void write(String sessionId, Map<String, Object> state);
    Optional<Map<String, Object>> read(String sessionId);
    void update(String sessionId, Map<String, Object> updates);
    void expire(String sessionId); // triggers consolidation hook first
}

// LangGraph node: SessionCloseNode
// Executes the four-step handoff in order, wrapped in a try/compensate block
public class SessionCloseNode implements Node<AgentState> {
    // Step 1: consolidator.consolidateSession(sessionId, userId)
    // Step 2: taskStore.checkpointSession(sessionId, userId)
    // Step 3: contextWindow.flush()
    // Step 4: sessionMgr.expire(sessionId)
}
```

---

## Section 13.3 — User Modelling: Building a Persistent Model of the User

### Context

A user model is the agent's persistent, structured representation of a specific user — their preferences, working patterns, expertise level, communication style, active goals, and constraints. It is the **semantic memory layer for person-specific knowledge** and the primary driver of agent personalisation.

### What — User Model Schema

```
UserModelFact
  factId        : UUID
  userId        : String           -- GDPR subject
  category      : enum { preference, constraint, expertise, pattern, goal }
  key           : String
  value         : String
  confidence    : ConfidenceLevel  -- STATED(1.0), VERIFIED(0.85), INFERRED(0.60), SPECULATIVE(0.35)
  source        : String           -- "user-explicit" | "tool-verified" | "agent-inferred"
  createdAt     : Instant
  updatedAt     : Instant
  accessCount   : int
  ttlDays       : int
```

**Confidence levels:**

| Level | Value | Meaning |
|---|---|---|
| `STATED` | 1.00 | User said it explicitly |
| `VERIFIED` | 0.85 | Confirmed by tool or repeated behaviour |
| `INFERRED` | 0.60 | Derived by agent from observation |
| `SPECULATIVE` | 0.35 | Single-instance inference, low confidence |

### Three-Case Merge Protocol

When new information conflicts with an existing user model fact:

1. **Higher-confidence source wins** — `STATED` overrides `INFERRED` always
2. **Equal-source recency wins** — the more recent record is canonical
3. **Lower-confidence new fact is discarded** — do not overwrite a `VERIFIED` fact with a `SPECULATIVE` inference

### Context Injection at Session Start

- Retrieve facts filtered by `confidence >= MIN_INJECTION_CONFIDENCE` (default: `0.60` — INFERRED and above)
- Sort by confidence descending, then by `accessCount` descending
- Inject at most `maxFacts` (default: 15) to limit token spend
- Label every injected fact: `UserModel [category conf=0.85]: key = value`

> ⚠️ **Never inject SPECULATIVE-confidence facts (< 0.40) as established knowledge.** An agent acting on a low-confidence inference as though it were a stated preference makes incorrect personalisations that users notice immediately.

### Design Rules

| Rule ID | Rule |
|---|---|
| `UM-001` | User model facts MUST carry a `ConfidenceLevel`. Facts below `INFERRED (0.60)` MUST NOT be injected at session start. |
| `UM-002` | Merge conflicts MUST apply the three-case merge protocol in order: source authority → recency → discard. |
| `UM-003` | All injected user model facts MUST be labelled with type, confidence, and date. |
| `UM-004` | User model facts MUST participate in the importance-scoring pipeline from chap12 §12.3. |
| `UM-005` | A user-accessible "What does this assistant know about me?" query interface MUST be exposed (GDPR Art. 15). |

### Java + LangGraph Target

```java
public enum ConfidenceLevel {
    STATED(1.00), VERIFIED(0.85), INFERRED(0.60), SPECULATIVE(0.35);
    final double value;
    ConfidenceLevel(double v) { this.value = v; }
}

public record UserModelFact(
    String factId, String userId, String category,
    String key, String value, ConfidenceLevel confidence,
    String source, Instant createdAt, Instant updatedAt,
    int accessCount, int ttlDays
) {}

public interface UserModelStore {
    void merge(UserModelFact incoming);          // applies three-case merge
    List<UserModelFact> getForInjection(        // confidence >= threshold
        String userId, double minConfidence, int maxFacts);
    List<UserModelFact> getAllForUser(String userId); // GDPR Art.15
}

// LangGraph node: UserModelInjectionNode
// Runs at session start; injects filtered facts into AgentState.injectedMemories
```

---

## Section 13.4 — Task Continuity: Resuming Long-Running Tasks Across Interruptions

### Context

Long-running tasks — multi-day research workflows, software development tasks spanning many sessions, complex analysis processes — cannot complete in a single session. Conversation history captures *what was said*. Task continuity requires preserving *what was decided, what was completed, what remains open, and why.*

### The Task Checkpoint Schema

```
TaskCheckpoint
  taskId          : String
  userId          : String
  sessionId       : String         -- session that wrote this checkpoint
  status          : TaskStatus     -- IN_PROGRESS | SUSPENDED | AWAITING_INPUT | COMPLETED | FAILED
  goal            : String         -- original task statement
  completedSteps  : List<String>   -- what has been done
  nextSteps       : List<String>   -- what remains to be done
  openQuestions   : List<String>   -- unresolved blockers
  artefacts       : Map<String,String>  -- key → store reference
  contextSummary  : String         -- compact LLM-generated resume-oriented summary
  checkpointAt    : Instant
  version         : int            -- incremented on each update
```

### The Context Summary Contract

The `contextSummary` field is the most important field at resume time. It MUST be generated with the following LLM instruction:

> *"Summarise the current state of this task in 150 words or fewer. Include what has been decided, what was last attempted, what the immediate next step is, and any open blockers. Write for an agent resuming this task with a fresh context window."*

Write it at **suspension time** — while full context is available — not at resume time.

### Four-Step Resume Injection Sequence

```
1. Inject contextSummary  → reconstructs the full state picture
2. Inject completedSteps  → prevents re-doing finished work
3. Inject nextSteps       → directs the first reasoning step
4. Retrieve related memories → surface relevant episodic memory for this taskId
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `TC-001` | Task checkpoints MUST use the canonical `TaskCheckpoint` schema — conversation history alone is insufficient for task resumption. |
| `TC-002` | The `contextSummary` MUST be generated at suspension time with the resume-oriented LLM instruction. |
| `TC-003` | Checkpoints MUST be written before the operation that may fail, not after. |
| `TC-004` | The four-step resume injection sequence MUST be executed in order: summary → completed steps → next steps → related memories. |
| `TC-005` | `TaskCheckpoint.version` MUST be incremented on every update; stale-version writes MUST be rejected. |

### Java + LangGraph Target

```java
public enum TaskStatus { IN_PROGRESS, SUSPENDED, AWAITING_INPUT, COMPLETED, FAILED }

public record TaskCheckpoint(
    String taskId, String userId, String sessionId,
    TaskStatus status, String goal,
    List<String> completedSteps, List<String> nextSteps,
    List<String> openQuestions, Map<String, String> artefacts,
    String contextSummary, Instant checkpointAt, int version
) {}

public interface TaskStore {
    void writeCheckpoint(TaskCheckpoint checkpoint);     // TC-003: call before risky ops
    Optional<TaskCheckpoint> read(String taskId);        // returns latest version
    void updateStatus(String taskId, TaskStatus status);
}

// LangGraph node: TaskResumeNode
// Loads checkpoint, executes four-step injection, sets state.taskCheckpoint
// LangGraph node: TaskSuspendNode
// Generates contextSummary via LLM, increments version, writes checkpoint
```

---

## Section 13.5 — Shared Memory in Multi-Agent Systems

### Context

Shared memory in multi-agent systems is **necessary only when agents coordinate on genuinely shared evolving state**. If agents perform isolated, independent subtasks with results aggregated only at the end, shared memory is overhead without benefit.

### When Shared Memory Becomes Necessary

- Agents must coordinate on the same evolving task without duplicating work
- Agents must persist decisions across steps and sessions so that later agents can build on earlier agents' conclusions
- Agents must scale in parallel without contradicting each other's outputs

### Three Shared Memory Architecture Patterns

| Pattern | Structure | Best For | Primary Risk |
|---|---|---|---|
| **Centralised shared store** | Single memory store; full isolation via namespace/tenant | Orchestrator–subagent hierarchies, tightly coupled workflows | Single point of failure; write contention under high agent concurrency |
| **Hierarchical private + shared** | Each agent has private memory; promoted shared tier holds only explicitly published knowledge | Parallel specialist agents, privacy-sensitive multi-user deployments | Merge conflicts at promotion time; stale private state |
| **Distributed with consistency layer** | Each agent has local store; consistency layer propagates updates with a defined consistency model | High-throughput, geographically distributed agent networks | Eventual consistency window; complex conflict resolution |

> The **hierarchical private + shared** pattern is the correct default for most production multi-agent deployments.

### Access Control Requirements

```
WRITE_ROLES  : orchestrator, specialist, coordinator
READ_ROLES   : orchestrator, specialist, coordinator, observer
```

All shared memory writes MUST be audited (agent ID, timestamp, payload hash).

### Core Memory Alignment

In multi-agent systems, agents may read from the shared store at different times and operate on different versions of shared facts. **Core memory alignment** is the practice of explicitly synchronising the most critical shared facts into each agent's working context at task start.

Core memory tags: `{ taskGoal, currentPlan, decided, constraint }`

### Design Rules

| Rule ID | Rule |
|---|---|
| `SM-001` | Introduce shared memory only when a concrete coordination failure demands it — not by architectural preference. |
| `SM-002` | Use the hierarchical private + shared pattern as default. |
| `SM-003` | Role-based write access control MUST be enforced from the first deployment. |
| `SM-004` | All shared memory write operations MUST be audited (agent ID, timestamp, payload hash). |
| `SM-005` | Core memory alignment MUST be performed at agent task start by injecting shared facts tagged with `CORE_MEMORY_TAGS`. |
| `SM-006` | Shared memory without write access control is a security liability (see chap14 for poisoning attack vectors). |

### Java + LangGraph Target

```java
public interface SharedMemoryStore {
    void write(MemoryRecord record, String agentId, AgentRole role);
    List<MemoryRecord> read(String query, String agentId, AgentRole role, int topK);
    // throws PermissionException if role not in WRITE_ROLES / READ_ROLES
}

public class CoreMemoryAlignmentNode implements Node<AgentState> {
    // Injects records tagged with CORE_MEMORY_TAGS for this taskId
    // into AgentState.injectedMemories at node entry
    static final Set<String> CORE_MEMORY_TAGS =
        Set.of("taskGoal", "currentPlan", "decided", "constraint");
}
```

---

## Section 13.6 — Memory Synchronisation and Conflict Resolution in Multi-Agent Systems

### Context

Single-agent memory conflict resolution (chap12 §12.6) addresses divergence between two stores owned by one agent. Multi-agent conflict resolution addresses agents with different authority levels disagreeing, or parallel agents producing contradictory conclusions.

### Consistency Models

| Model | Guarantee | Best For |
|---|---|---|
| **Session consistency** | A single agent always reads its own writes within a session | Per-agent task memory, private stores |
| **Causal consistency** | An agent sees writes that causally precede its read | Orchestrator–subagent workflows where order matters |
| **Eventual semantic consistency** | All agents converge to the same view eventually; conflicts resolved by policy | Parallel specialist agents, distributed agent networks |
| **Strong consistency** | All agents see the same view at the same time; writes block until all replicas confirm | High-stakes, safety-critical shared facts, compliance records |

> **Causal consistency** is the correct target for most production multi-agent deployments. Strong consistency should be reserved for safety-critical facts and compliance records.

### Optimistic Locking — Preventing Parallel Overwrites

All shared memory writes MUST include the `expectedVersion` field. A write with a stale version MUST be rejected with a `ConflictException`, triggering a read–merge–retry cycle with exponential back-off.

### Four-Level Multi-Agent Conflict Resolution Hierarchy

Apply in priority order:

1. **Agent authority** — a designated authoritative agent (orchestrator or domain specialist) overrides a general-purpose agent for domain-specific facts
2. **Temporal precedence** — for equal-authority agents, the earlier write is canonical unless explicitly superseded
3. **Consensus** — if multiple agents independently arrive at the same conclusion, promote to shared tier with higher confidence
4. **Human arbitration** — safety-critical conflicts, or conflicts unresolvable within max retry budget, escalate to a human operator

> **Version control on shared memory records** — tracking every write with agent ID, timestamp, and version — enables rollback when a conflict creates an inconsistent state. Design this rollback capability in from the first multi-agent deployment.

### Design Rules

| Rule ID | Rule |
|---|---|
| `SYNC-001` | Choose a consistency model explicitly before the first multi-agent deployment. Causal consistency is the default; strong consistency for safety-critical facts only. |
| `SYNC-002` | All shared memory writes MUST use optimistic locking with `expectedVersion` to prevent parallel overwrite failure mode. |
| `SYNC-003` | The four-level conflict resolution hierarchy MUST be applied in priority order: agent authority → temporal precedence → consensus → human arbitration. |
| `SYNC-004` | Every shared memory record MUST carry a version number, agent ID, and timestamp to enable rollback. |
| `SYNC-005` | Maximum retry budget for conflict resolution MUST be explicitly bounded; exhausted budget triggers human arbitration, not infinite retry. |

### Java + LangGraph Target

```java
public record SharedMemoryRecord(
    String recordId, String content, String agentId,
    AgentRole agentRole, Instant writtenAt, int version
) {}

public interface MultiAgentConflictResolver {
    Resolution resolve(SharedMemoryRecord existing, SharedMemoryRecord incoming);
    // Applies: agentAuthority → temporalPrecedence → consensus → humanArbitration
}

// LangGraph pattern: optimistic locking node
// 1. Read record + version
// 2. Compute update
// 3. Write with expectedVersion
// 4. On ConflictException: back off, re-read, re-merge, retry (max 3 attempts)
// 5. On exhausted retries: escalate to HumanArbitrationNode
```

---

## Section 13.7 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `SF-001` | Stateful/stateless choice | Choose statelessness deliberately for atomic tasks |
| `SF-002` | Stateful foundations | All three foundations required: storage, retrieval, lifecycle |
| `SF-003` | Write pipelines | All memory write pipelines MUST be idempotent with dedup keys |
| `SF-004` | Context injection | Injected memories MUST be labelled, never treated as instructions |
| `SF-005` | Checkpointing | Write checkpoints before potentially failing operations |
| `SS-001` | Session state backend | TTL-governed Redis (or equivalent), 2-hour inactivity default |
| `SS-002` | Session-close order | consolidate → checkpoint → flush → expire — non-negotiable |
| `SS-003` | Session-close safety | Failed consolidation MUST NOT allow expiry of source session state |
| `SS-004` | Session key namespacing | Keys MUST be namespaced by `sessionId`, never shared across sessions |
| `UM-001` | Confidence filtering | Facts below `INFERRED (0.60)` MUST NOT be injected at session start |
| `UM-002` | Merge conflict | Three-case merge: source authority → recency → discard |
| `UM-003` | Injection labelling | Injected user model facts MUST carry type, confidence, and date |
| `UM-004` | Importance scoring | User model facts MUST participate in chap12 importance-scoring pipeline |
| `UM-005` | GDPR Art.15 | User-accessible "What does this assistant know about me?" interface MUST be exposed |
| `TC-001` | Checkpoint schema | Use canonical `TaskCheckpoint` schema — conversation history alone is insufficient |
| `TC-002` | Context summary | Generated at suspension time with resume-oriented LLM instruction |
| `TC-003` | Checkpoint timing | Written before the operation that may fail |
| `TC-004` | Resume sequence | summary → completed steps → next steps → related memories — non-negotiable order |
| `TC-005` | Version enforcement | Stale-version checkpoint writes MUST be rejected |
| `SM-001` | Shared memory decision | Introduce only when a concrete coordination failure demands it |
| `SM-002` | Default pattern | Hierarchical private + shared |
| `SM-003` | Access control | Role-based write access enforced from first deployment |
| `SM-004` | Audit | All shared memory write operations MUST be audited |
| `SM-005` | Core alignment | Core memory alignment MUST run at agent task start |
| `SM-006` | Security | Shared memory without write ACL is a security liability |
| `SYNC-001` | Consistency model | Explicit selection required before first multi-agent deployment |
| `SYNC-002` | Optimistic locking | All shared writes use `expectedVersion` |
| `SYNC-003` | Conflict hierarchy | agent authority → temporal precedence → consensus → human arbitration |
| `SYNC-004` | Version tracking | Every record carries version, agent ID, timestamp |
| `SYNC-005` | Retry budget | Bounded retry budget; exhausted → human arbitration |

---

## Section 13.8 — Transition to Chapter 14

Chapters 10–13 have built a complete, production-grade memory architecture:

- **Taxonomy** (chap10) — five types, in-context vs. external hierarchy, injection pattern, lifecycle
- **Storage & retrieval** (chap11) — embeddings, vector databases, knowledge graphs, hybrid pipelines
- **Operational lifecycle** (chap12) — consolidation, importance scoring, pruning, tiered persistence, consistency, audit
- **Statefulness** (chap13 — this chapter) — stateless/stateful divide, session management, user modelling, task continuity, shared memory, synchronisation

**Chapter 14** extends this foundation into the **advanced dynamics** that emerge once the architecture is live under real production workloads:

| Topic | Engineering Challenge |
|---|---|
| Associative memory & spreading activation | Retrieval that surfaces records the agent didn't know to ask for |
| Hierarchical summarisation trees | Scaling without degrading retrieval quality |
| Prospective memory | Scheduling future recalls; acting on time-sensitive knowledge without human prompting |
| Memory-augmented reasoning patterns | Tight integration of memory into the reasoning loop, not just as a pre-generation step |
| Memory as an attack surface | Poisoning attack vectors and defence-in-depth architecture |
| Memory system evaluation | The measurement harness that closes the feedback loop for the entire architecture |
