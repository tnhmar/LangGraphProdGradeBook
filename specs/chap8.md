# Chapter 8 — The State Manager: The Heart of the Agent

**Part 2: Anatomy of an AI Agent**
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

> *"Without state, the agent is not an agent. It is a sequence of independent API calls wearing the same name." — Engineering maxim*

Every component in the agent loop reads from or writes to state. Inputs are stored as state. Plans are state. Tool results update state. Beliefs are state. Termination conditions are evaluated against state. Without a disciplined state manager, the agent has no continuity between steps.

This chapter is about **operational state within and immediately around a run** — not long-term memory architecture. State answers *"what is active now?"* Memory answers *"what should persist?"* Those are distinct architectural questions. Memory architecture begins in Part 3. This chapter covers the six structural concerns of the state manager: state schema and classification, working memory management, context window management, goal stack tracking, belief state, state serialization and checkpointing, and the explicit state machine model.

---

## 8.1 — What Is Agent State? Scope and Mutability Classification

### Context

Agent state is the complete set of information that determines how the agent behaves at the current moment in a run. It includes the active goal, subgoal stack, working context, belief state, recent observations, pending action queue, and execution metadata. Before designing any state schema or storage backend, classify every state component by two dimensions: **scope** and **mutability**.

### What (the spec)

**Four scope tiers:**

| Tier | Name | Lifetime | Examples | Boundary |
|---|---|---|---|---|
| **S0** | Step-local | Single step; discarded after completion | Raw tool call arguments before schema normalization | `PerceiveNode` → `ReasonNode` only |
| **S1** | Run state | All steps within one execution run | Active goal, subgoal stack, belief state, working memory, tool observation history | Full `AgentState` in LangGraph |
| **S2** | Session state | Multiple runs within a user session | Conversation history, partial task progress, session-scoped preferences | Session store (Redis or equivalent) |
| **S3** | Persistent state | Survives session boundaries | User model, episodic memory, long-term semantic facts | External memory stores (Part 3) |

> **Architectural boundary rule:** This chapter covers S0 and S1 only. S2 and S3 are covered in Part 3 (Memory Architecture). Mixing these concerns in the same component produces a system that is hard to test, hard to reason about, and difficult to scale. Never store S3 concerns in `AgentState`.

**Two mutability classes:**

| Class | Description | Examples | Write rule |
|---|---|---|---|
| **Immutable reference** | Set at run start; never updated during the run | Original goal statement, initial user request, policy version loaded at session start | Write-once; reads are always consistent |
| **Mutable working data** | Continuously updated as the run progresses | Tool results, belief state, subgoal completion status, cycle counter | All writes go through typed `AgentState` reducers; no raw dict mutation |

**Treating all state as a single mutable dictionary erases these distinctions** and makes state corruption difficult to detect, belief provenance impossible to track, and context window management unreliable.

### Why (engineering implications)

- An immutable goal statement that is accidentally overwritten mid-run causes goal drift — the most insidious loop failure mode (Section 2.6). Declaring it immutable at the schema level prevents this class of bug structurally.
- S2 (session) and S1 (run) state have different TTLs, different backends, and different failure modes. Storing them in the same object creates a system where a session expiry silently clears active run state, or a run crash poisons session history.
- The scope classification is the input to every other decision in this chapter: what goes in the context window (S1 mutable), what gets checkpointed (S1 mutable + S1 immutable reference), and what routes to memory architecture (S2, S3).

### How (system design rules)

1. **Define `AgentState` as a typed Java record** with explicit fields for every S1 component. No untyped `Map<String, Object>` fields. Every field has a declared `MutabilityClass` annotation (`@ImmutableRef` or `@MutableWorkingData`).
2. **Enforce immutability at the reducer level:** LangGraph state reducers for `@ImmutableRef` fields throw `ImmutableFieldViolationException` on any write after initialization.
3. **Annotate every `AgentState` field with its scope tier** (`@StateScope(S1)`) so that serialization and checkpointing logic can filter by tier correctly.
4. **Never pass `AgentState` directly to sub-agents or delegation targets.** Only explicitly typed `DelegationInput` records cross agent boundaries (Section 7.8).
5. **In Java + LangGraph**: `AgentState` is a Java record. `AgentStateReducer` is a `BinaryOperator<AgentState>` passed to `StateGraph.addNode`. Immutability violations are checked in `AgentStateReducer.apply()` before merging. `AgentStateSchema` is a JUnit 5 test class that validates every field has a declared mutability class and scope tier annotation.

---

## 8.2 — Working Memory: The In-Session Scratchpad

### Context

Working memory is the scratchpad the agent uses throughout a run. It holds the active goal, current subgoal, recent tool results, active hypotheses, and the most contextually relevant facts for the current reasoning step. Working memory is bounded by the context window and must be **actively managed**. The correct metaphor is a workbench, not an archive.

### What (the spec)

**Working memory contents at any given cycle:**

| Component | Description | Eviction policy |
|---|---|---|
| `activeGoal` | Current top-level goal statement (immutable ref) | Never evicted |
| `activeSubgoal` | Current subgoal from the goal stack | Replaced on subgoal transition |
| `recentObservations` | Last N tool results (configurable, default N=5) | FIFO eviction after N |
| `activeBeliefs` | Beliefs directly relevant to current subgoal | Evicted when subgoal completes |
| `pendingHypotheses` | Tentative conclusions awaiting confirmation | Evicted on confirmation or refutation |
| `activeConstraints` | Policy constraints active for current step | Refreshed from constraint store per cycle |
| `workingFacts` | Key facts extracted from observations | Evicted when superseded by belief consolidation |

**Working memory is append-only by default — this is a common and costly mistake.** Once a tool response has been incorporated into a belief state entry, the raw response should be replaced by a summary that preserves the key facts at a fraction of the token cost.

**Eviction triggers (in priority order):**
1. **Belief consolidation:** When a raw observation has been incorporated into `activeBeliefs`, remove the raw observation from `recentObservations`
2. **Subgoal completion:** Evict all working facts and hypotheses scoped to the completed subgoal
3. **Context high-water mark:** When context window utilization exceeds 70%, trigger pruning (Section 8.3)
4. **Explicit summarization:** Before eviction of content that may still matter downstream, summarize and store the summary (Section 8.3)

**Instrumentation requirement:** Emit a `WorkingMemoryMetrics` event at every cycle recording: total tokens in working memory, tokens by component, eviction events since last cycle, and current high-water mark percentage.

### Why (engineering implications)

- A working memory that grows unboundedly across cycles is the primary cause of context window saturation failures in long-horizon agents. These failures are silent: the agent starts truncating its own context, losing facts it previously knew, and producing subtly wrong reasoning — without any explicit error.
- Instrumenting working memory size at every step reveals accumulation patterns and context pressure points before a context overflow incident occurs. Teams that add this instrumentation after their first production saturation incident universally wish they had added it at day one.
- Eviction policy must be tested explicitly. A unit test that runs 20 simulated cycles and asserts that working memory tokens never exceed the configured budget is a first-class test, not an optional one.

### How (system design rules)

1. **`WorkingMemory` is a typed component of `AgentState`** with explicit fields for each component listed above and a configurable `maxRecentObservations` (default 5).
2. **`WorkingMemoryManager` is a Spring `@Component`** called by `PerceiveNode` at the start of each cycle to apply eviction and summarization policies before assembling the context window.
3. **Belief consolidation is a named operation** in `WorkingMemoryManager.consolidate(observation, belief)` — not implicit state mutation. Every consolidation emits a `ConsolidationEvent` to the audit log.
4. **Eviction is logged** with the evicted component type, token count recovered, and eviction trigger reason.
5. **In Java + LangGraph**: `WorkingMemory` is a field of `AgentState`. `WorkingMemoryManager` is called in `PerceiveNode.apply()` before context assembly. `WorkingMemoryMetrics` is an OTel `Gauge` instrument updated each cycle. The eviction policy is configurable via `WorkingMemoryConfig` (a `@ConfigurationProperties` bean).

---

## 8.3 — Context Window Management: Filling, Pruning, Summarizing

### Context

The context window determines what the model can reason about in a single inference call. It is finite, and managing it well is one of the highest-leverage engineering decisions in a production agent. Three active operations govern context quality: **filling**, **pruning**, and **summarizing**. These are not automatic behaviors — they are engineering responsibilities of the state manager.

### What (the spec)

**Three context window operations:**

| Operation | When to apply | Mechanism | Risk if misapplied |
|---|---|---|---|
| **Filling** | Every cycle — select what enters the context | Filter by current subgoal relevance; use tiered relevance scoring | Irrelevant context included → reasoning degrades |
| **Pruning** | When window exceeds 70% capacity | Remove oldest-irrelevant content first; validate no outstanding dependencies | Still-needed context removed → silent reasoning gaps |
| **Summarizing** | Before pruning, when content may still matter | LLM-based summarization with explicit "preserve these facts" instruction | Lossy summaries may drop facts needed later |

**Tiered context model (three tiers, managed by `ContextWindowManager`):**

| Tier | Content | Resolution | Token budget |
|---|---|---|---|
| **Active tier** | Current subgoal inputs, most recent tool result, active constraints | Full resolution | 40% of window |
| **Recent history tier** | Last 3 subgoal results (summarized form) | Summarized | 30% of window |
| **Reference tier** | Goal statement, policy snapshot, session metadata | Compressed reference | 15% of window |
| **Reserved** | LLM reasoning output buffer | — | 15% of window |

**Filling selection algorithm (per cycle):**
1. Always include: `activeGoal` (ref), `activeSubgoal`, `activeConstraints`
2. Score all `recentObservations` and `activeBeliefs` by relevance to `activeSubgoal` (using BM25 + recency score)
3. Fill active tier to budget with highest-scoring items
4. Fill recent history tier with summarized forms of last 3 completed subgoal results
5. Fill reference tier with goal statement + policy snapshot
6. Validate total token count ≤ 85% of context window limit before calling `ReasonNode`

**Summarization instruction template (for LLM-based summarization):**
> *"Summarize the following tool result for an agent resuming the task '{activeSubgoal}'. Preserve: all numeric values, all entity identifiers, all status flags. Omit: raw JSON structure, repeated field names, metadata unrelated to the subgoal. Write for a reader with a fresh context window."*

**High-water mark alert thresholds:**
- **70%**: Trigger pruning of lowest-relevance content
- **85%**: Trigger summarization of recent history tier
- **95%**: Emergency pruning — drop all S0 step-local content and oldest reference tier items; log `CONTEXT_EMERGENCY_PRUNE`

### Why (engineering implications)

- Filling selection by subgoal relevance (not by recency alone) is the difference between a context window that supports good reasoning and one that overwhelms it with stale, irrelevant tool results. A payment result from cycle 2 is irrelevant when the current subgoal is "format output" — but a naive append-only context includes it anyway.
- Pruning without dependency validation is dangerous. Content that was incorporated into a belief ("order is confirmed as duplicate") can be safely pruned. Content that is still an unresolved dependency ("awaiting delivery scan result") cannot. The dependency check must be explicit.
- The summarization instruction matters. A generic "summarize this" produces a summary optimized for a human reader. The resume-oriented instruction above produces a summary optimized for an agent with a fresh context window — which is what is actually needed.

### How (system design rules)

1. **`ContextWindowManager` is a Spring `@Component`** called by `PerceiveNode` after `WorkingMemoryManager`. It assembles the final context window array passed to `ReasonNode`.
2. **Implement `RelevanceScorer`** as a named component that scores each content item by BM25 against the current subgoal description + recency decay. Scores are logged per cycle for observability.
3. **Implement `DependencyChecker`** that maintains a `Set<String> pendingDependencies` in `AgentState`. Pruning candidates are filtered against this set before removal.
4. **Summarization is a dedicated LangGraph node** (`SummarizeNode`) called on demand by `ContextWindowManager` when the 85% threshold is reached. It is not embedded in `PerceiveNode`.
5. **Emit `ContextWindowMetrics`** (OTel Gauge) at every cycle: total tokens, tokens per tier, fill score statistics, and prune/summarize events.
6. **In Java + LangGraph**: `ContextWindowManager.assemble(AgentState)` returns a `List<Message>` passed directly to `ChatModel.call()`. `RelevanceScorer` uses Apache Lucene BM25 for text scoring + a recency decay function. `SummarizeNode` calls the LLM with the resume-oriented summarization prompt.

---

## 8.4 — Goal Stack and Subgoal Tracking

### Context

As the agent decomposes a task into subgoals, it needs a structure that tracks what it is doing now, which parent goal that subgoal belongs to, and where to resume when the subgoal finishes. The goal stack provides that structure. Without it, completing a nested subgoal has no visible connection to the original mission, and the agent may not know what to do next.

### What (the spec)

**Goal stack entry schema (every entry must have all fields):**

| Field | Type | Description |
|---|---|---|
| `goalId` | `UUID` | Stable identifier for this goal entry |
| `parentGoalId` | `UUID?` | Reference to parent entry; null for top-level goal |
| `description` | `String` | Precise, bounded goal statement |
| `status` | `GoalStatus` enum | `PENDING`, `ACTIVE`, `COMPLETED`, `FAILED`, `DEFERRED` |
| `successCriteria` | `List<String>` | Measurable, evaluable conditions for `COMPLETED` |
| `knownDependencies` | `List<UUID>` | Other goal IDs that must be `COMPLETED` before this can be `ACTIVE` |
| `tokenBudget` | `int` | Maximum tokens allocated to this subgoal |
| `stepBudget` | `int` | Maximum cycles allocated to this subgoal |
| `createdAt` | `Instant` | Goal creation timestamp |
| `completedAt` | `Instant?` | Completion timestamp (for performance analysis) |
| `resumeCondition` | `String?` | For `DEFERRED` status: the condition that re-activates this goal |

**Example goal stack during a complex billing dispute run:**

| Level | Goal | Status |
|---|---|---|
| Top-level | Resolve case 44821 fairly and accurately | `ACTIVE` |
| Level 2 | Resolve billing complaint | `ACTIVE` |
| Level 3 | Verify duplicate charge | `COMPLETED` |
| Level 3 | Apply refund and compensation policy | `ACTIVE` |
| Level 4 | Confirm authorization limit | `ACTIVE` |

**Goal stack operations:**

| Operation | Trigger | Effect |
|---|---|---|
| `push(goal)` | `ReasonNode` generates a new subgoal | Add entry with `PENDING` status; validate `parentGoalId` exists |
| `activate(goalId)` | All `knownDependencies` are `COMPLETED` | Set status → `ACTIVE`; inject into `activeSubgoal` in working memory |
| `complete(goalId)` | `ReflectNode` evaluates `successCriteria` as met | Set status → `COMPLETED`; set `completedAt`; trigger `activate` on any dependent goals |
| `fail(goalId, reason)` | `ReflectNode` determines goal is infeasible | Set status → `FAILED`; trigger failure escalation path |
| `defer(goalId, condition)` | `ReasonNode` determines goal cannot proceed now | Set status → `DEFERRED` with `resumeCondition`; do not evict from stack |
| `pop()` | `ReflectNode` finalizes `COMPLETED` or `FAILED` | Remove from stack; update parent goal's dependency set |

**Intent anchoring (mandatory for runs > 5 cycles):** At every 5th cycle, the orchestrator performs an **intent anchor check** — it evaluates whether the current `ACTIVE` leaf goal is still aligned with the top-level goal statement. If the current active subgoal cannot be traced back to the top-level goal via the parent chain, emit `GOAL_DRIFT_DETECTED` and trigger `ReflectNode` immediately.

### Why (engineering implications)

- `DEFERRED` goals belong on the stack, not dropped and reconstructed from context. An agent that drops a deferred goal loses the information about why that goal existed and what its success criteria were. Reconstructing it from context is unreliable, especially in long-horizon runs where the original context has been pruned.
- The `successCriteria` field is the hardest field to write well, and the most important. Open-ended tasks ("resolve the dispute fairly") must be decomposed into evaluable criteria ("refund issued OR dispute escalated to human with documented rationale"). `ReflectNode` evaluates these criteria — if they are unmeasurable, `ReflectNode` cannot determine completion, and the loop continues indefinitely.
- The `tokenBudget` and `stepBudget` per subgoal are the fine-grained resource controls that prevent one runaway subgoal from consuming the entire run budget. A top-level 15-cycle budget distributed as 5 cycles per Level 2 subgoal is more robust than a single global cycle counter.

### How (system design rules)

1. **`GoalStack` is a typed field of `AgentState`** — a `List<GoalEntry>` with `push`, `activate`, `complete`, `fail`, `defer`, and `pop` operations implemented as pure functions on immutable lists (new list returned; old list not mutated in place).
2. **`GoalStackManager` is a Spring `@Component`** called by `ReflectNode` after each cycle to evaluate `successCriteria` for the current `ACTIVE` leaf goal and determine the next stack operation.
3. **Intent anchor check** is implemented in `OrchestrationNode` as a conditional check at every 5th cycle. `GOAL_DRIFT_DETECTED` routes the graph to `ReflectNode` with a `DRIFT_RECOVERY` signal.
4. **`successCriteria` is validated at goal creation time** — `GoalStackManager.push()` rejects goals with empty `successCriteria` with a `MissingSuccessCriteriaException`.
5. **In Java + LangGraph**: `GoalEntry` is a Java record. `GoalStack` wraps `List<GoalEntry>` with named operations. `GoalStackManager` is called in `ReflectNode.apply()`. LangGraph conditional edge: if `GoalStackManager.evaluate()` returns `GOAL_DRIFT_DETECTED` → `reflectNode` with `DRIFT_RECOVERY`; if `COMPLETED` → pop and activate next; if `FAILED` → `handleError`.

---

## 8.5 — Belief State: The Agent's Model of the World

### Context

Belief state is the agent's current model of the relevant world. It is **not** the same as raw tool output. A payment ledger may return twelve events. The belief derived from those events is: *"two successful charges were captured for order-88931 within 18 seconds; neither has been reversed."* That belief is what drives the refund decision — not the raw JSON.

### What (the spec)

**Belief entry schema:**

| Field | Type | Description |
|---|---|---|
| `beliefId` | `UUID` | Stable identifier |
| `category` | `BeliefCategory` enum | `FACTUAL`, `INFERRED`, `HYPOTHETICAL`, `CONTRADICTED` |
| `content` | `String` | Precise, declarative statement of the belief |
| `confidence` | `float` [0.0–1.0] | Confidence score (see calibration rules below) |
| `provenance` | `List<BeliefSource>` | Tool calls + timestamps that produced this belief |
| `derivedFrom` | `List<UUID>` | Belief IDs that this belief was inferred from |
| `createdAt` | `Instant` | When this belief was formed |
| `expiresAt` | `Instant?` | TTL for time-sensitive beliefs (e.g., stock prices, session tokens) |
| `supersededBy` | `UUID?` | If this belief has been revised, the ID of the superseding belief |

**Three belief properties that make belief state trustworthy in production:**

1. **Provenance tracking** — Each belief links to the evidence that produced it: source tool name, call timestamp, and result hash. *"Duplicate charge confirmed"* cites `paymentQuery/order-88931` at `2026-03-30T14:03` — not model inference from prior context.

2. **Confidence annotation** — Confidence is calibrated by source type:
   - `1.0`: Live authoritative source (real-time API with no caching, strong consistency)
   - `0.85`: Cached authoritative source (TTL < 5 min)
   - `0.70`: Inferred from multiple corroborating T0 sources
   - `0.50`: Inferred from a single source or LLM reasoning
   - `0.30`: Hypothetical / speculative

3. **Conflict detection** — When new evidence contradicts an existing belief, `BeliefManager` surfaces the conflict explicitly rather than silently overwriting. The agent receives a `BELIEF_CONFLICT` signal and must resolve it before proceeding.

**Belief conflict resolution protocol (three steps):**
1. Mark existing belief as `CONTRADICTED`; create new belief as `HYPOTHETICAL`
2. Route to `ReasonNode` with `CONFLICT_RESOLUTION` signal; inject both beliefs
3. `ReasonNode` resolves by selecting higher-confidence belief, requesting confirmation tool call, or escalating to human review

**Belief expiry:** Time-sensitive beliefs (live API responses, session tokens) carry `expiresAt`. `PerceiveNode` checks for expired beliefs at every cycle and marks them `CONTRADICTED` with `content = "EXPIRED"` before context assembly.

### Why (engineering implications)

- Belief state is not the same as the context window. It is a structured, queryable representation that exists separately from the token stream. Conflating the two makes belief updates invisible, conflicts undetectable, and provenance impossible to audit.
- Silent belief overwrite — new tool result replaces old belief without conflict detection — is the primary cause of agents making contradictory decisions within the same run. A refund agent that initially believes "no refund issued" and then silently overwrites to "refund issued" without audit trail has no way to detect or recover from the overwrite if it was caused by a tool error.
- The confidence threshold for action is a configurable policy: `actNode` can be configured to require minimum confidence `0.70` for T2 actions and `0.85` for T3 actions. This turns confidence into a safety gate.

### How (system design rules)

1. **`BeliefState` is a typed field of `AgentState`** — a `Map<UUID, BeliefEntry>` indexed by `beliefId`, with named operations: `add`, `update`, `contradict`, `expire`, `resolve`.
2. **`BeliefManager` is a Spring `@Component`** with `addOrUpdate(observation, subgoal)` that either creates a new belief or detects a conflict with an existing belief and emits `BELIEF_CONFLICT`.
3. **Implement `ConfidenceCalibrator`** that assigns confidence scores based on source type. Scores are declared in `ConfidencePolicy` (a `@ConfigurationProperties` bean) — not hard-coded.
4. **`ActNode` enforces confidence gates** for T2 (minimum 0.70) and T3 (minimum 0.85) actions. A tool call proposed by `ReasonNode` that depends on a belief below the threshold routes to `ReasonNode` with a `LOW_CONFIDENCE_BLOCK` signal.
5. **In Java + LangGraph**: `BeliefEntry` is a Java record. `BeliefManager` is called in `ReflectNode.apply()` after each tool observation. `BELIEF_CONFLICT` is a signal in `AgentState.signals` that `OrchestrationNode` reads and routes to `ReasonNode` with the conflict payload injected into working memory.

---

## 8.6 — State Serialization and Checkpointing

### Context

Long-running agents are exposed to infrastructure failures. Serialization converts the full run state to a durable, portable format. Checkpointing is the policy that determines **when** to serialize. Without checkpointing, every infrastructure failure loses all run progress, and post-incident forensic analysis is impossible.

### What (the spec)

**Checkpoint trigger policy (five mandatory triggers):**

| Trigger | Condition | Priority |
|---|---|---|
| **T1 — Subgoal completion** | Every time `GoalStack.complete()` is called | High |
| **T2 — Pre-irreversible action** | Before any T3 (irreversible) tool dispatch | Critical — must complete before dispatch begins |
| **T3 — Context high-water mark** | When context window utilization crosses 85% | Medium |
| **T4 — Material plan change** | When `ReflectNode` produces a plan revision (not just progress) | Medium |
| **T5 — Time-based** | Every N minutes for long-horizon runs (configurable, default 5 min) | Low |

**Checkpoint record schema (complete; partial checkpoints are forbidden):**

| Field | Type | Description |
|---|---|---|
| `checkpointId` | `UUID` | Unique checkpoint identifier |
| `runId` | `UUID` | The run this checkpoint belongs to |
| `stepIndex` | `int` | Cycle count at checkpoint time |
| `schemaVersion` | `String` | `AgentState` schema version (for migration) |
| `integrityHash` | `String` | SHA-256 of the serialized state (detect corruption) |
| `agentState` | `SerializedAgentState` | Complete S1 state snapshot (all fields) |
| `goalStackSnapshot` | `List<GoalEntry>` | Full goal stack at checkpoint time |
| `beliefStateSnapshot` | `Map<UUID, BeliefEntry>` | Full belief state at checkpoint time |
| `contextSummary` | `String` | Resume-oriented summary (see instruction below) |
| `resumeNode` | `String` | LangGraph node name to execute first on resume |
| `triggerType` | `CheckpointTrigger` enum | Which trigger caused this checkpoint |
| `createdAt` | `Instant` | Checkpoint timestamp |

**Context summary instruction (mandatory for every checkpoint):**
> *"Write for an agent resuming this task with a fresh context window. State: (1) the top-level goal, (2) what has been confirmed as true, (3) what step was in progress, (4) what the next step is, and (5) any open questions. Maximum 300 tokens."*

Write this summary at **checkpoint time** (while full context is available), not at resume time. A summary written with full context is dramatically more useful than one reconstructed from partial information.

**Partial checkpoints are forbidden.** A checkpoint that does not include the complete `AgentState`, `GoalStack`, and `BeliefState` is more dangerous than no checkpoint. An agent resuming from an incomplete snapshot may believe it has completed subgoals it has not, producing duplicate actions or missed work.

**Four-step resume injection sequence (execute in order on `runId` load):**
1. Inject `contextSummary` as the first system message
2. Inject completed steps from `goalStackSnapshot` (all `COMPLETED` entries) as established facts
3. Inject next steps from `goalStackSnapshot` (all `PENDING` / `ACTIVE` entries) as the active work queue
4. Retrieve and inject related memories from the memory store (filtered by `runId` and `taskId` tags, top-10 by relevance)

### Why (engineering implications)

- The T2 trigger (pre-irreversible action) is the most critical. When the agent issues an incorrect T3 action, the ability to restore the pre-dispatch checkpoint and replay the exact step with corrected validation logic is the difference between a forensically diagnosable incident and an irreproducible mystery.
- Checkpointing on every cycle is prohibitively expensive. The five-trigger policy checkpoints at risk boundaries (before irreversible actions, on major state changes) rather than uniformly — concentrating storage cost where it has the highest diagnostic value.
- Schema versioning on the checkpoint record (`schemaVersion`) is mandatory because `AgentState` schemas evolve. A checkpoint written with schema `v1.3` must be loadable by an agent running schema `v1.5`. `CheckpointMigrationService` handles version-to-version upgrades.

### How (system design rules)

1. **LangGraph's built-in checkpointing mechanism is the implementation foundation.** `StateGraph` is compiled with a `CheckpointSaver` that writes to a configured backend (PostgreSQL for production; in-memory for testing).
2. **`CheckpointPolicy` is a Spring `@Component`** that evaluates the five triggers after every cycle and calls `CheckpointSaver.save()` when any trigger fires.
3. **Integrity hash validation** is performed on every checkpoint read (resume): `SHA-256(serializedState)` must match `checkpointId.integrityHash`. Hash mismatch triggers `CHECKPOINT_CORRUPTION` alert and falls back to the previous valid checkpoint.
4. **`CheckpointMigrationService`** maintains a migration chain: `v1.0 → v1.1 → ... → vCurrent`. Each version step is a pure function `migrate(oldState) → newState`. Applied sequentially on checkpoint load.
5. **`ResumeOrchestrator`** executes the four-step resume injection sequence as the first operation after checkpoint load, before the first `PerceiveNode` call.
6. **In Java + LangGraph**: `CheckpointSaver` is a LangGraph `JdbcSaver` (PostgreSQL) for production or `InMemorySaver` for tests. `CheckpointPolicy` listens to `AgentStateEvents` published by `ReflectNode` and `ActNode`. `ResumeOrchestrator` is called in `AgentRuntime.resume(runId)`.

---

## 8.7 — State Transitions and the Explicit State Machine

### Context

An agent run moves through a finite set of identifiable execution states. Treating those transitions as implicit control flow buried in orchestrator logic makes them untestable and impossible to diagnose when something goes wrong. The **explicit state machine model** makes every transition a first-class artifact: defined, guarded, testable, and auditable.

### What (the spec)

**Agent run state machine — states and transitions:**

| From State | Triggering Event | Guard Condition | To State |
|---|---|---|---|
| `INITIALIZED` | Goal received | Goal schema valid | `PLANNING` |
| `PLANNING` | Plan produced | Plan passes pre-validation | `EXECUTING` |
| `PLANNING` | Plan invalid | — | `REPLANNING` |
| `REPLANNING` | Revised plan produced | Plan passes pre-validation | `EXECUTING` |
| `REPLANNING` | Replan budget exhausted | — | `DEGRADED` |
| `EXECUTING` | Tool call proposed | Action passes all validators | `AWAITING_RESULT` |
| `EXECUTING` | All subgoals done | — | `COMPLETING` |
| `EXECUTING` | Safety violation | — | `ABORTED` |
| `AWAITING_RESULT` | Result received | Result schema valid | `EXECUTING` |
| `AWAITING_RESULT` | Timeout | Retry budget remains | `EXECUTING` (retry) |
| `AWAITING_RESULT` | Retry budget exhausted | — | `DEGRADED` |
| `AWAITING_RESULT` | Authorization failure | — | `ABORTED` |
| `COMPLETING` | Output validated | — | `TERMINATED` |
| `COMPLETING` | Output invalid | — | `REPLANNING` |
| `DEGRADED` | Recovery action available | — | `EXECUTING` |
| `DEGRADED` | No recovery path | — | `ABORTED` |
| `ABORTED` | — | — | *(terminal — no outbound transitions)* |
| `TERMINATED` | — | — | *(terminal — no outbound transitions)* |

**Key policy decisions embedded in the state machine:**
- There is **no transition from `DEGRADED` directly to `TERMINATED`** — a degraded run must either recover or be explicitly aborted. Degraded runs that are silently abandoned without an `ABORTED` event are invisible in audit logs.
- There is **no transition from `ABORTED` back to `EXECUTING`** — aborted runs are final. Recovery means starting a new run from a checkpoint, not resuming the aborted one.
- The `REPLANNING` state is a first-class state, not a silent internal retry — it has its own entry/exit actions and its own step budget counter.

**State machine implementation requirements:**
- Typed `RunState` enum with all states above
- Explicit `TransitionTable` mapping `(RunState, TransitionEvent) → (RunState, List<SideEffect>)` 
- Guard functions evaluated synchronously before each transition
- Entry actions (e.g., emit `StateTransitionEvent`, reset subgoal step counter) executed on every state entry
- Exit actions (e.g., flush working memory tier on `COMPLETING`) executed on every state exit

**Observability requirement:** Every state transition emits a `StateTransitionEvent` to the audit log with: `runId`, `fromState`, `toState`, `triggeringEvent`, `guardsPassed`, `sideEffects`, `timestamp`.

### Why (engineering implications)

- The state machine table reveals design decisions that prose descriptions obscure. The explicit absence of `DEGRADED → TERMINATED` and `ABORTED → EXECUTING` transitions are policy choices. When they are buried in `if/else` orchestrator logic, no one notices them — and future developers accidentally add those paths. Explicit tables make the policy visible and reviewable.
- Each transition can be unit-tested by: constructing the source state, firing the event, and asserting the resulting state and emitted side effects. This is dramatically simpler than testing orchestrator logic end-to-end.
- The `REPLANNING` state being first-class means it has its own step budget (`maxReplanningCycles`, default 3). Without this, a run can oscillate between `PLANNING` and `REPLANNING` indefinitely, exhausting the global cycle budget on planning alone.

### How (system design rules)

1. **Implement `RunStateMachine` as a first-class Spring `@Component`** with a typed `RunState` enum, a `TransitionTable` (an immutable `Map<Pair<RunState, TransitionEvent>, Transition>`), and a `dispatch(event, context)` method.
2. **`OrchestrationNode` calls `RunStateMachine.dispatch()`** on every cycle-end event. It does not contain its own `if/else` state logic — all routing is delegated to the state machine.
3. **Guard functions are named, independently testable Java `Predicate<TransitionContext>`** — not anonymous lambdas embedded in the transition table.
4. **LangGraph conditional edges are generated from `TransitionTable`** — a `GraphBuilder` utility reads the transition table and adds the corresponding conditional edges to the `StateGraph` during startup.
5. **`StateTransitionEvent` is emitted via `AuditService`** and as an OTel span event on the active trace for the run.
6. **Unit test every transition** in `RunStateMachineTest` — 18+ test cases, one per row in the transition table.
7. **In Java + LangGraph**: `RunState` is a Java `enum`. `TransitionTable` is a `Map<Pair<RunState, String>, BiFunction<AgentState, TransitionContext, TransitionResult>>`. `GraphBuilder.buildFromTransitionTable(TransitionTable)` creates LangGraph conditional edges automatically. `RunStateMachine` is the single source of truth for all routing logic.

---

## 8.8 — State Observability and Replay

### Context

A production agent whose state is not fully observable cannot be reliably debugged, audited, or improved. State observability means every state change is logged as a structured event, every run can be replayed deterministically from its checkpoints, and every production incident is diagnosable from the state transition history without access to live systems.

### What (the spec)

**Minimum state observability record per cycle:**

| Event type | Contents | Retention |
|---|---|---|
| `CycleStartEvent` | `runId`, `cycleId`, `stepIndex`, `runState`, `activeSubgoal`, `workingMemoryTokens` | 30 days |
| `StateTransitionEvent` | `runId`, `fromState`, `toState`, `event`, `guards`, `sideEffects`, `timestamp` | 90 days |
| `BeliefUpdateEvent` | `runId`, `cycleId`, `operation` (add/contradict/expire/resolve), `beliefId`, `content`, `confidence`, `provenance` | 90 days |
| `GoalStackEvent` | `runId`, `cycleId`, `operation` (push/activate/complete/fail/defer/pop), `goalId`, `description`, `status` | 90 days |
| `WorkingMemoryEvent` | `runId`, `cycleId`, `operation` (evict/consolidate/summarize), `component`, `tokenDelta` | 30 days |
| `ContextWindowEvent` | `runId`, `cycleId`, `totalTokens`, `tokensByTier`, `fillScore`, `pruneEvents`, `highWaterPct` | 30 days |
| `CheckpointEvent` | `runId`, `checkpointId`, `triggerType`, `stateSize`, `integrityHash`, `timestamp` | 1 year |

**Deterministic replay requirements:**
- Load checkpoint → inject resume sequence (Section 8.6) → re-execute from `resumeNode` with original inputs
- Replay must produce the same `GoalStack` and `BeliefState` transitions as the original run (given the same tool responses)
- Replay uses **mock tool handlers** that return the logged tool responses from the original run — no live API calls
- `ReplayOrchestrator.verify(runId, checkpointId)` executes replay and asserts that state transitions match the `StateTransitionEvent` log

**Forensic investigation workflow (post-incident):**
1. Load the checkpoint immediately before the incorrect action (T2 trigger checkpoint)
2. Inspect `BeliefStateSnapshot` — identify which beliefs drove the action
3. Inspect `GoalStackSnapshot` — verify the active subgoal was correctly scoped
4. Replay the cycle using mock tools with the original responses
5. Test a corrected validator or belief update against the replayed sequence — without touching live data or incurring API costs

### Why (engineering implications)

- The T2 checkpoint (pre-irreversible action) combined with the `BeliefUpdateEvent` log is the forensic kit for every T3 action incident. Without both, reconstructing why the agent took a destructive action requires inference from incomplete evidence — which is often impossible under time pressure.
- Deterministic replay with logged tool responses is the most powerful debugging tool in an agent engineer's toolkit. Teams that invest in this from day one debug incidents in minutes. Teams that add it after the third production incident wish they had built it first.
- 1-year retention for `CheckpointEvent` and checkpoint records is driven by compliance requirements in regulated domains (financial services, healthcare). In non-regulated domains, 90 days is typically sufficient — but the retention policy must be explicit and documented.

### How (system design rules)

1. **All state observability events go through `StateEventPublisher`**, a Spring `@Component` that emits events to both the audit log (`AuditService`) and the OTel trace (`Tracer.getCurrentSpan().addEvent()`).
2. **`StateEventPublisher` is injected into** `WorkingMemoryManager`, `BeliefManager`, `GoalStackManager`, `ContextWindowManager`, `CheckpointPolicy`, and `RunStateMachine`. No component writes state silently.
3. **`ReplayOrchestrator` is a first-class production component**, not a testing utility. It is used for post-incident investigation, regression testing, and checkpoint validation.
4. **Mock tool registry for replay** is built from the `ActionTrace` log (Section 7.9): every tool call in the original run has a logged request and response; `ReplayToolRegistry` serves these as deterministic mock handlers.
5. **Retention policy is declared in `ObservabilityConfig`** (`@ConfigurationProperties`) and enforced by a scheduled `RetentionCleanupJob`.
6. **In Java + LangGraph**: `StateEventPublisher` is a Spring `@Component` with typed `publish(StateEvent)` method. `ReplayOrchestrator` loads a `CheckpointRecord` from `CheckpointSaver`, builds a `ReplayToolRegistry`, and calls `AgentRuntime.replayFrom(checkpoint, replayToolRegistry)`. Replay results are compared against the original `StateTransitionEvent` log using `StateTransitionLogComparator`.

---

## Key Takeaways for Volume 2 Implementation

- **State is the foundation of agent continuity.** Every component reads from or writes to `AgentState`. Without disciplined state management, the agent has no continuity, no observability, and no recoverability.
- **Classify every state component by scope (S0–S3) and mutability (immutable ref / mutable working data)** before designing schemas. These classifications drive every other state management decision.
- **Working memory is a managed workbench, not an append-only log.** Eviction, consolidation, and summarization are first-class engineering operations — not optimizations to add later.
- **Context window management (filling, pruning, summarizing)** is an active cycle-by-cycle responsibility of the state manager. Fill by subgoal relevance; prune only after dependency validation; summarize with a resume-oriented instruction.
- **The goal stack is the agent's navigational instrument.** Every subgoal has `successCriteria`, `tokenBudget`, `stepBudget`, and a parent reference. `DEFERRED` goals stay on the stack with a `resumeCondition`. Intent anchoring checks drift every 5 cycles.
- **Belief state is the agent's model of the world** — structured, provenance-tracked, and separate from the context window. Conflict detection is mandatory; silent overwrite is a production defect.
- **Checkpoint before every risk boundary** (pre-irreversible action, subgoal completion, plan revision, context high-water mark). Never checkpoint partially. Write the resume-oriented `contextSummary` at checkpoint time, not at resume time.
- **The explicit state machine** turns runtime transitions from hidden control flow into testable, auditable artifacts. Every transition is defined, guarded, and emits a `StateTransitionEvent`. `DEGRADED` requires explicit `ABORTED` — not silent abandonment.
- **State observability and replay** are first-class production capabilities, not debugging tools. The T2 checkpoint + `BeliefUpdateEvent` log is the forensic kit for every T3 action incident.
