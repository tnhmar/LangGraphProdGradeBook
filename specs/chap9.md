# Chapter 9 — Cognitive Architectures: The Brain of the Agent

**Part 2: Anatomy of an AI Agent — Closing Chapter**
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

> *"Components do not make a system. The architecture that connects them does." — Engineering maxim*

The previous chapters described the internal components of an agent runtime: perception, reasoning, action, and state. This chapter asks the higher-level design question: **how should those components be arranged, sequenced, and connected?** That is the role of a cognitive architecture. It is not a framework and it is not a prompt strategy. It is the structural blueprint that determines whether the agent behaves as a linear reason-act loop, a self-correcting feedback system, a search-based planner, or a commitment-driven goal executor.

Choosing the right cognitive architecture before writing code is one of the highest-leverage decisions in agent system design. This chapter provides the complete specification for making that choice: what cognitive architectures are, the classical foundations (SOAR, ACT-R, BDI) whose concepts recur in modern systems, the four primary LLM-native architectures (ReAct, Reflexion, LATS, BDI-influenced hybrid), and a structured selection framework driven by measured task properties.

---

## 9.1 — What Is a Cognitive Architecture?

### Context

A cognitive architecture is a **stable structural specification** for how an intelligent agent processes information, makes decisions, updates its state, and produces behavior. It defines which functional components exist, how information flows between them, and which control principles govern their interaction.

This is explicitly distinct from:
- A **framework** — which provides implementation infrastructure (LangGraph, LangChain, Spring AI)
- A **design pattern** — which describes a reusable local solution (e.g., retry-on-failure, fan-out/fan-in)
- A **prompt strategy** — which influences model output within a fixed architecture

The same architecture can be built using different frameworks. The same framework can host multiple architectures. Architecture selection is a behavioral decision; framework selection is an implementation decision. They must not be conflated.

### What (the spec)

**Three dimensions that characterize any cognitive architecture:**

| Dimension | Description | Options |
|---|---|---|
| **Component model** | Which functional units exist and what each is responsible for | Perception, working memory, planner, action dispatcher, belief updater, feedback processor |
| **Control flow model** | Whether the architecture is deliberative, reactive, or hybrid | Deliberative (plan-first-then-act), Reactive (stimulus-response), Hybrid (layered) |
| **Adaptation model** | Whether and how the architecture changes behavior from evidence | Within-run only, across-run (Reflexion), across-run with tree search (LATS), none |

**Control flow model comparison:**

| Dimension | Deliberative | Reactive | Hybrid |
|---|---|---|---|
| Planning | Explicit, upfront | None | Layered: fast reaction + slow deliberation |
| Latency to act | High | Low | Variable by layer |
| Adaptability | Requires replan | Immediate behavioral change | Both, by layer |
| Failure recovery | Structured replan | Behavioral change on signal | Mixed by layer |

LLM-based agents are predominantly **hybrid**: the model provides deliberative planning capability while the runtime injects reactive behaviors such as safety gates, timeout overrides, and human interrupts.

**Cognitive architecture is the primary design artifact for an agent system.** Document it explicitly before writing the first `StateGraph` node. Changes to the cognitive architecture after implementation require restructuring the graph topology, not just modifying node logic.

### Why (engineering implications)

- Teams that select a cognitive architecture based on framework defaults (e.g., "LangGraph is a loop so we'll use ReAct") without evaluating the behavioral implications of that choice are making an accidental architectural decision. Accidental architectural decisions are discovered through production failures, not design reviews.
- The framework-vs-architecture distinction also matters for portability. A system whose cognitive architecture is clearly documented can be reimplemented in a different framework without loss of behavioral properties. A system where the architecture is implicit in framework-specific code cannot.
- The adaptation model dimension is the most frequently overlooked. "This agent doesn't need to learn" is often wrong — even within a single run, the agent's behavior should change as it accumulates evidence. Whether that adaptation is explicit (Reflexion self-critique) or structural (LATS branch pruning) must be a conscious design choice.

### How (system design rules)

1. **Produce an `AgentArchitectureDocument`** as a required pre-implementation artifact. It declares: selected architecture name, component model (which nodes implement which components), control flow model (deliberative/reactive/hybrid and rationale), and adaptation model. This document is committed to the repository alongside the `StateGraph` definition.
2. **The `AgentArchitectureDocument` is the source of truth** for LangGraph graph topology. New nodes proposed without reference to this document require an architecture review before implementation.
3. **Never select an architecture based on framework defaults.** Evaluate at least ReAct and BDI-influenced hybrid before committing to a choice.
4. **In Java + LangGraph**: `AgentArchitectureDocument` is a Markdown file at `src/main/resources/architecture/agent-architecture.md`. It is read and displayed in the Spring Boot Actuator `/agent/architecture` endpoint for runtime inspection.

---

## 9.2 — Classical Foundations: SOAR and ACT-R

### Context

Classical cognitive architectures predate LLMs by decades, but their core ideas appear repeatedly in modern agent systems — often under different names. Understanding SOAR and ACT-R as foundational architectures helps reason about *why* modern designs work and *where* they will break, because the underlying mechanisms are the same.

### What (the spec)

**SOAR (State, Operator And Result) — Carnegie Mellon / University of Michigan, 1980s onward:**

SOAR is organized around a **recognize-decide-act cycle** operating over a working memory of symbolic facts and a production memory of if-then rules. At each cycle, all rules whose conditions match working memory fire and modify working memory.

**SOAR's two key contributions for modern agent design:**

| Concept | SOAR mechanism | Modern LLM-agent equivalent |
|---|---|---|
| **Impasse handling** | When the system cannot select an action due to missing knowledge or conflicting rules, it creates a subgoal dedicated to resolving the impasse | `ReflectNode` detecting uncertainty → `GoalStack.push()` with a clarification subgoal; task decomposition under incomplete information |
| **Chunking** | Successful problem-solving episodes are compiled into new production rules for future reuse | Procedural memory (Chapter 10); workflow templates extracted from successful runs |

**ACT-R (Adaptive Control of Thought — Rational) — Carnegie Mellon, John Anderson:**

ACT-R models cognition as an interaction between a central procedural module and a set of specialist declarative memory modules.

**ACT-R's two key contributions for modern agent design:**

| Concept | ACT-R mechanism | Modern LLM-agent equivalent |
|---|---|---|
| **Activation-based retrieval** | Memories compete for retrieval based on an activation score combining recency and frequency of prior use — not deterministic key lookup | Relevance scoring in `ContextWindowManager` (Section 8.3); embedding similarity search in vector memory stores (Chapter 11) |
| **Spreading activation** | Retrieved memories propagate activation to structurally linked memories via association graph | Knowledge graph traversal in hybrid retrieval (Chapter 11); the SYNAPSE architecture's associative recall pattern |

**The shared lesson from both architectures:** Separating **procedural knowledge** (how to act) from **declarative knowledge** (what is true) produces more maintainable and more general agents than conflating both into a single undifferentiated context. This separation appears in Chapter 10's memory taxonomy as the procedural/semantic distinction.

### Why (engineering implications)

- SOAR's impasse-driven subgoal generation is the theoretical foundation for the `GoalStack.push()` pattern specified in Section 8.4. When an agent hits uncertainty, the correct response is to create a resolution subgoal — not to halt or to guess. Teams that understand the SOAR lineage of this pattern are less likely to short-circuit it under deadline pressure.
- ACT-R's activation-based retrieval explains why static, deterministic context injection is inferior to relevance-scored retrieval. A context window that includes all past tool results equally degrades reasoning quality in exactly the way ACT-R predicts: low-activation (irrelevant) items compete for attention with high-activation (relevant) ones.
- ACT-R's spreading activation also explains why flat vector search misses important context in long-running agents: it retrieves records directly similar to the query but not records that are *associatively linked* to those records. Chapter 11's hybrid retrieval architecture addresses this directly.

### How (system design rules)

1. **Document the SOAR/ACT-R mappings** in the `AgentArchitectureDocument`. For each LangGraph node that implements a classical mechanism, add an inline comment: `// ACT-R: activation-based retrieval` or `// SOAR: impasse-driven subgoal generation`.
2. **Implement `ImpasHandler`** as a named `@Component` called by `ReflectNode` when confidence falls below threshold. `ImpasHandler.handle(reason)` creates a typed clarification subgoal and pushes it onto the `GoalStack`. It does not let the orchestrator guess through ambiguity.
3. **Separate procedural from declarative content** in all prompt templates. Declarative content (what is known) is assembled by `ContextWindowManager`. Procedural content (how to act) is injected by `SystemPromptBuilder` from the `ProceduralMemoryStore`. These are never assembled by the same component.
4. **In Java + LangGraph**: `ImpasHandler` is called in `ReflectNode.apply()` when `BeliefState` contains beliefs with confidence < 0.5 and no resolution path is available. `ProceduralMemoryStore` is a `@Repository` backed by the system prompt store, injected into `SystemPromptBuilder`.

---

## 9.3 — The BDI Model: Beliefs, Desires, Intentions

### Context

The Belief-Desire-Intention (BDI) model provides a formal framework for agent decision-making built around three separable mental state components. It is the most directly applicable classical architecture to production LLM agents because it provides formal separation between *what the agent knows* (beliefs), *what it could pursue* (desires), and *what it has committed to doing* (intentions).

### What (the spec)

**Three BDI components:**

| Component | Definition | Agent system equivalent | Mutability |
|---|---|---|---|
| **Beliefs** | What the agent holds to be true about the world; may be incomplete or incorrect | `BeliefState` in `AgentState` (Section 8.5) | Continuously updated from observations |
| **Desires** | Objectives the agent could pursue; many may coexist simultaneously | Candidate subgoals generated by `ReasonNode`; option space before commitment | Evaluated but not all executed |
| **Intentions** | Desires the agent has committed to executing; pursued until success, impossibility, or explicit supersession | Active entries in `GoalStack` with `ACTIVE` status (Section 8.4) | Not re-evaluated without cause |

**BDI execution cycle (five steps, maps directly to the agent runtime loop):**

1. **Update beliefs** from new observations → `PerceiveNode` + `BeliefManager.addOrUpdate()`
2. **Generate candidate options** from current beliefs and existing intentions → `ReasonNode` produces candidate subgoals
3. **Filter to a consistent option set** → `ReasonNode` evaluates feasibility and belief consistency
4. **Adopt new intentions** → `GoalStack.push()` + `GoalStack.activate()`
5. **Execute one step of the current intention plan** → `ActNode` dispatches one tool call per cycle

**The intention commitment principle — the most important engineering rule in BDI:**

> Once the agent adopts an intention, it does **not** re-evaluate the underlying desire on every cycle. It pursues the intention until it succeeds, proves impossible, or is explicitly superseded.

Agents that constantly re-evaluate all goals under minor new information exhibit **chaotic, unpredictable behavior** — the hallmark of missing BDI structure. The commitment principle is what gives BDI agents behavioral coherence under variable inputs.

**BDI intention revision triggers (the only conditions under which an intention is abandoned):**

| Trigger | Condition | Action |
|---|---|---|
| **Success** | All `successCriteria` for the intention evaluate to true | `GoalStack.complete(goalId)` |
| **Impossibility** | `ReflectNode` determines the intention is infeasible given current beliefs | `GoalStack.fail(goalId, reason)` |
| **Explicit supersession** | A higher-priority intention is adopted that conflicts with the current one | `GoalStack.defer(goalId, resumeCondition)` → push new intention |
| **Belief invalidation** | A critical belief supporting the intention is contradicted with confidence > 0.85 | `BELIEF_CONFLICT` signal → `GoalStack.defer()` + resolve conflict first |

**BDI diagnostic lens:** If an agent has no stable intention representation — if it re-evaluates its goal on every step — it will exhibit inconsistent, hard-to-predict behavior under variable input conditions. The presence of `GoalStack` with commitment semantics is the structural indicator of BDI influence.

### Why (engineering implications)

- The BDI intention commitment principle directly prevents goal drift (Section 8.4's intent anchor check is the defensive mechanism for when commitment is violated). Goal drift is the most common long-horizon failure mode in production agents, and it is a direct consequence of missing BDI structure.
- The desires/intentions distinction provides the framework for "option generation without commitment": `ReasonNode` generates multiple candidate subgoals (desires), but only one is activated at a time (intention). This prevents the agent from spreading effort across incompatible paths simultaneously.
- BDI is the correct architectural influence for rule-governed domains with explicit commitments and stable state: financial compliance, legal document processing, medical triage. In these domains, the predictability of commitment-based execution is a feature, not a limitation.

### How (system design rules)

1. **Map all three BDI components to named `AgentState` fields** in the `AgentArchitectureDocument`: beliefs → `BeliefState`, desires → `candidateSubgoals` (transient, step-local), intentions → `GoalStack` (active `ACTIVE` entries).
2. **`ReasonNode` generates desires, not intentions.** `ReasonNode.apply()` returns a `List<CandidateSubgoal>`. The `GoalStackManager` evaluates and selects the next intention to activate. This separation is explicit in the code, not implicit in a single "plan" output.
3. **Implement `IntentionRevisionGuard`** — a `@Component` that validates any proposed `GoalStack.defer()` or `GoalStack.fail()` against the four revision triggers above. Revisions without a matching trigger are rejected with `UnauthorizedIntentionRevisionException`.
4. **In Java + LangGraph**: `CandidateSubgoal` is a Java record returned by `ReasonNode`. `GoalStackManager.selectNextIntention(List<CandidateSubgoal>, AgentState)` selects and activates one. `IntentionRevisionGuard` is called before any `GoalStack.defer()` or `GoalStack.fail()` operation.

---

## 9.4 — The ReAct Architecture

### Context

ReAct (Reasoning + Acting) is the baseline cognitive architecture for LLM agents. It specifies a unified context in which reasoning traces and action outputs coexist, an interleaved control flow alternating thought and action, a linear forward trajectory with no built-in backtracking, and an immediate observation feedback loop after each action step. It is examined here as a **complete cognitive architecture**, not only as a prompting pattern.

### What (the spec)

**ReAct structural specification:**

| Structural property | Description | Engineering implication |
|---|---|---|
| **Unified context** | Reasoning traces and action outputs coexist in the same context window | Agent always reasons with full access to prior trajectory |
| **Interleaved control flow** | Alternating Thought → Action → Observation cycles | Every plan step is grounded in evidence retrieved during execution |
| **Linear forward trajectory** | No built-in backtracking; each cycle advances from the last | Control flow is simple, deterministic, fully auditable |
| **Immediate observation loop** | Observation (tool result) is injected immediately after each action | Reasoning is always grounded in the most recent evidence |

**ReAct cycle (four nodes in LangGraph):**

```
PerceiveNode → ReasonNode (Thought) → ActNode (Action) → PerceiveNode (Observation) → ...
```

Termination: when `ReasonNode` produces a `FINAL_ANSWER` signal (no `Action` in output), route to `OutputNode`.

**ReAct architectural limitations (each is a direct consequence of its structural choices):**

| Limitation | Root cause | When it becomes operationally relevant |
|---|---|---|
| **Forward error propagation** | Linear trajectory with no backtracking | Wrong reasoning step corrupts all subsequent steps |
| **Context accumulation** | Unified context grows linearly | Long runs saturate context window (mitigated by Chapter 8 context management) |
| **No cross-run learning** | No reflection or memory update mechanism | Agent repeats the same errors across different runs on structurally similar tasks |
| **Single trajectory** | No exploration of alternative paths | High-ambiguity tasks have no mechanism to consider alternative interpretations |

**ReAct is the correct baseline.** These limitations become operationally relevant only in specific task classes. Other architectures should be adopted only when ReAct's structural limitations produce measurable failure rates on real task workloads.

**ReAct LangGraph topology (minimal):**

| Node | Responsibility | Input | Output |
|---|---|---|---|
| `PerceiveNode` | Normalize input / inject observation into working memory | Raw input or tool result | Updated `AgentState` |
| `ReasonNode` | Generate Thought + Action proposal or FINAL_ANSWER | `AgentState` | `ThoughtActionResult` |
| `ActNode` | Validate and dispatch tool call | `ThoughtActionResult` | `ToolObservation` |
| `OutputNode` | Format and validate final answer | `AgentState` | `AgentOutput` |

**LangGraph conditional edge from `ReasonNode`:**
- If `ThoughtActionResult.type == FINAL_ANSWER` → `outputNode`
- If `ThoughtActionResult.type == TOOL_CALL` → `actNode`
- If `ThoughtActionResult.type == ERROR` → `handleError`

### Why (engineering implications)

- ReAct's linear, auditable trajectory is its most important production property. Every thought, action, and observation is recorded in the context and in the action log, making forensic investigation straightforward. Architectures that sacrifice this auditability (e.g., internal chain-of-thought that is discarded before logging) trade a debugging advantage for a marginal performance gain that rarely justifies the cost.
- The "no backtracking" property means that prompt quality matters enormously for ReAct. A malformed or ambiguous input that triggers an incorrect first Thought will propagate that error forward. This makes `PerceiveNode`'s grounding and normalization (Chapter 5) load-bearing for ReAct reliability in a way that is less true for Reflexion or LATS.
- ReAct's context accumulation limitation is fully addressed by Chapter 8's context window management (Section 8.3). Teams that implement Section 8.3 correctly can run ReAct on long-horizon tasks without context saturation. Teams that skip Section 8.3 hit context overflow on tasks exceeding ~15 cycles.

### How (system design rules)

1. **ReAct is the mandatory baseline for all new agent implementations.** Every agent in Volume 2 starts with ReAct topology. Architectural upgrades (Reflexion, LATS) require documented justification referencing measured failure rates.
2. **`ThoughtActionResult` is a sealed Java interface** with three implementing records: `FinalAnswer(String content)`, `ToolCallProposal(String toolName, Map<String, Object> args)`, `ErrorResult(String reason)`. `ReasonNode` returns exactly one.
3. **The ReAct loop termination condition is a configurable policy** (`ReactLoopConfig`): `maxCycles` (default 15), `maxTokens` (default 80% of model context limit), `terminateOnFinalAnswer` (default true). Exceeding `maxCycles` routes to `DEGRADED` state, not silent truncation.
4. **Emit a `ReActCycleEvent`** (via `StateEventPublisher`) at every cycle with: `cycleId`, `thought` (sanitized), `actionType`, `toolName`, `observationTokens`, `cumulativeTokens`.
5. **In Java + LangGraph**: `StateGraph<AgentState>` with nodes `perceiveNode`, `reasonNode`, `actNode`, `outputNode`, `handleError`. Conditional edge from `reasonNode` reads `AgentState.lastThoughtActionResult` and routes via `ReactRouter` (a named `Function<AgentState, String>`).

---

## 9.5 — The Reflexion Architecture

### Context

Reflexion (Shinn et al., NeurIPS 2023) addresses ReAct's inability to learn from failures within or across runs. Its core insight is that language models can generate useful self-critical analysis in natural language, and that this **verbal feedback** can serve as a lightweight substitute for weight updates when modifying the model is impractical. Reflexion adds structured self-reflection to a ReAct base.

### What (the spec)

**Two structural components added to ReAct base:**

| Component | Mechanism | Engineering role |
|---|---|---|
| **Evaluator** | After each run attempt, assesses the outcome — binary signal, heuristic score, or model critique. Produces a structured `EvaluationResult` | Determines whether to retry and what failed |
| **Self-reflection module** | Agent generates textual self-reflection based on `EvaluationResult`. Analyzes root cause of failure, identifies the specific failing step, proposes a different approach | Stored in episodic memory; injected at start of next attempt |

**Reflexion execution flow:**

```
Attempt N:
  ReAct loop → EvaluatorNode → [PASS → OutputNode] / [FAIL → ReflectNode → store reflection → Attempt N+1]
```

**Reflection record schema:**

| Field | Type | Description |
|---|---|---|
| `reflectionId` | `UUID` | Stable identifier |
| `attemptNumber` | `int` | Which attempt this reflects on |
| `evaluationResult` | `EvaluationResult` | What the evaluator determined |
| `failingStep` | `String` | The specific step or reasoning chain that failed |
| `rootCause` | `String` | Why it failed |
| `proposedApproach` | `String` | What to do differently next attempt |
| `confidence` | `float` | How confident the model is in this reflection |
| `createdAt` | `Instant` | Reflection timestamp |

**Reflexion context injection at attempt start:**

```
System: You have previously attempted this task. Your self-reflection from attempt {N}:
  - Failing step: {failingStep}
  - Root cause: {rootCause}
  - Proposed approach: {proposedApproach}
Apply this learning to your current attempt.
```

**Reflexion termination policy (all three conditions are MANDATORY — Reflexion without these produces infinite loops):**

| Condition | Threshold | Action |
|---|---|---|
| **Maximum attempts** | `maxAttempts` (default 3, configurable) | Route to `DEGRADED` → escalate |
| **Quality threshold** | `evaluator.score >= successThreshold` | Route to `OutputNode` |
| **No-progress detection** | Two consecutive reflections with identical `rootCause` | Route to `DEGRADED` → escalate (not retry) |

**Evaluator quality requirements:**
- The evaluator must be capable of detecting failure. An evaluator that always returns PASS disables Reflexion's learning mechanism.
- For the support agent: evaluator checks whether the resolution action taken matches the complaint category, whether a refund was issued when policy requires one, and whether the customer communication is factually consistent with the tool results.
- Evaluator is implemented as a separate `EvaluatorNode` — not as a prompt appended to `ReasonNode`.

### Why (engineering implications)

- Reflexion is correct when failure signals are clear, bounded retries are operationally acceptable, and the root cause of failure is **internal to the agent's reasoning** rather than external (tool unavailability, data quality). When the root cause is external, Reflexion retries will fail identically — triggering the no-progress detection and escalating correctly.
- The "no-progress detection" termination condition is the most important safety mechanism in Reflexion. Without it, an agent facing an unresolvable task (e.g., a tool that is persistently unavailable) will exhaust its retry budget on identical failures. With it, two identical `rootCause` strings trigger immediate escalation.
- Reflexion adds retry latency. For standard ticket resolution, this is unacceptable. Reserve Reflexion for task classes where the failure rate on a ReAct baseline exceeds the retry overhead cost — typically tasks with complex multi-step reasoning where the first-attempt failure rate exceeds 20%.

### How (system design rules)

1. **Extend the ReAct `StateGraph`** by adding: `EvaluatorNode`, `ReflectNode`, and a `ReflexionController` that manages attempt count, reflection injection, and termination.
2. **`EvaluatorNode` is a typed, independently testable `@Component`** that implements `Function<AgentState, EvaluationResult>`. It does not call the LLM for standard evaluation — it uses a rule-based evaluator for defined criteria and falls back to an LLM critique only for open-ended quality assessment.
3. **`ReflexionController`** tracks attempt number in `AgentState.reflexionAttempts` and enforces all three termination conditions before allowing another attempt. It logs every attempt transition as a `ReflexionAttemptEvent`.
4. **Reflection records are stored in episodic memory** (Chapter 11) via `EpisodicMemoryStore.write()` — not only in `AgentState`. This makes cross-run learning available when Reflexion is used in conjunction with memory architecture.
5. **`ReflexionConfig` (`@ConfigurationProperties`)** declares: `maxAttempts`, `successThreshold`, `noProg­ressWindowSize` (default 2), and `evaluatorMode` (`RULE_BASED` / `LLM_CRITIQUE` / `HYBRID`).
6. **In Java + LangGraph**: `StateGraph` adds `evaluatorNode`, `reflectNode` nodes. Conditional edge from `evaluatorNode`: `PASS` → `outputNode`; `FAIL + attempts < maxAttempts + progress` → `reflectNode`; `FAIL + (exhausted OR no-progress)` → `handleDegraded`.

---

## 9.6 — The LATS Architecture: Language Agent Tree Search

### Context

Language Agent Tree Search (LATS, Zhou et al., ICML) adapts **Monte Carlo Tree Search (MCTS)** for LLM agents. Where ReAct produces a single linear trajectory and Reflexion produces a sequence of retried linear trajectories, LATS explores a **tree of candidate trajectories simultaneously**, using principled search to find high-quality action sequences rather than committing immediately to the first plausible path.

### What (the spec)

**Four structural components of LATS:**

| Component | Mechanism | LangGraph equivalent |
|---|---|---|
| **LM as actor and evaluator** | Model generates candidate next steps at each tree node AND produces value estimates for those candidates | `LATSExpansionNode` (generates candidates) + `LATSEvaluatorNode` (scores them) |
| **MCTS integration** | Selection → Expansion → Evaluation → Backpropagation cycle over language trajectories | `LATSController` orchestrating `LATSSelectionNode`, `LATSExpansionNode`, `LATSEvaluatorNode`, `LATSBackpropNode` |
| **Real feedback during search** | Tool-call results and environment responses are incorporated as evaluation signals during search — not only at trajectory end | `ActNode` called during expansion; results feed `LATSEvaluatorNode` |
| **Self-reflection on failure** | Failed branches generate self-reflective annotations that improve value estimates for sibling branches | `LATSReflectNode` called on failed leaf nodes; annotations stored on tree node |

**LATS MCTS cycle (four phases):**

| Phase | Operation | Description |
|---|---|---|
| **Selection** | `LATSSelectionNode` | Choose the tree node with highest UCB1 score: \( \text{UCB1}(n) = Q(n) + c\sqrt{\frac{\ln N(p)}{N(n)}} \) where \( Q(n) \) is value estimate, \( N(n) \) is visit count, \( N(p) \) is parent visit count, \( c \) is exploration constant |
| **Expansion** | `LATSExpansionNode` | Generate K candidate next steps from selected node (K = `lats.branchingFactor`, default 3) |
| **Evaluation** | `LATSEvaluatorNode` + `ActNode` | Execute one step of each candidate; score result using model value estimate + tool feedback |
| **Backpropagation** | `LATSBackpropNode` | Update value estimates up the tree from evaluated node to root |

**LATS tree node schema:**

| Field | Type | Description |
|---|---|---|
| `nodeId` | `UUID` | Stable identifier |
| `parentNodeId` | `UUID?` | Parent in the search tree; null for root |
| `depth` | `int` | Depth in tree from root |
| `thought` | `String` | The reasoning step at this node |
| `action` | `ToolCallProposal?` | Action taken (if any) |
| `observation` | `String?` | Tool result (if action was taken) |
| `valueEstimate` | `float` | Current value estimate \( Q(n) \) |
| `visitCount` | `int` | Times this node has been selected \( N(n) \) |
| `reflection` | `String?` | Self-reflection annotation (if this is a failed leaf) |
| `status` | `LATSNodeStatus` enum | `OPEN`, `EXPANDED`, `TERMINAL_SUCCESS`, `TERMINAL_FAILURE` |

**LATS termination conditions:**

| Condition | Threshold | Action |
|---|---|---|
| **Success** | Any leaf reaches `EvaluationResult.PASS` | Return that trajectory; stop search |
| **Budget exhausted** | Total tool calls across all branches > `lats.maxToolCalls` | Return best trajectory found so far |
| **Max depth** | Any branch reaches `lats.maxDepth` (default 8) | Mark as `TERMINAL_FAILURE`; backpropagate |
| **Max iterations** | MCTS iterations > `lats.maxIterations` (default 20) | Return best trajectory found so far |

**Architecture comparison:**

| Property | ReAct | Reflexion | LATS |
|---|---|---|---|
| **Search strategy** | Linear, greedy | Sequential retry | MCTS tree |
| **Backtracking** | None | At run boundary | Any node |
| **Self-correction** | None | Cross-trial | Within search |
| **Inference cost** | Low (1x) | Medium (2–4x) | High (K × depth × iterations) |
| **Best for** | Standard tasks, clear paths | Tasks with recoverable reasoning errors | High-ambiguity, high-stakes tasks |

### Why (engineering implications)

- LATS is prohibitively expensive for standard ticket resolution (K=3, depth=8, iterations=20 = up to 480 tool calls for one task). It becomes **justifiable only when the cost of a wrong committed path exceeds the cost of systematic exploration** — typically complex fraud investigations, policy-conflict resolution, or multi-system inconsistency analysis.
- The UCB1 exploration-exploitation balance is the key tuning parameter. High `c` (exploration constant) causes LATS to explore broadly but shallowly. Low `c` causes it to exploit the best-looking branch aggressively. For the support agent, start with `c = 1.0` and tune from production data.
- LATS's real-feedback-during-search property means that tool calls are made during the search process itself. This has security implications: T3 (irreversible) tools must be blocked during LATS search phases. Only T0 (read-only) and T1 (idempotent write) tools are permitted during MCTS expansion. T2/T3 tools are only dispatched on the selected final trajectory.

### How (system design rules)

1. **LATS requires an explicit `LATSSearchTree`** stored in `AgentState` alongside (not replacing) the `GoalStack` and `BeliefState`. The search tree is the working artifact of the MCTS process.
2. **T3 tool guard during search**: `LATSExpansionNode` checks every `ToolCallProposal` against `ToolRegistry.getClassification()`. Any T3 proposal during expansion is rejected with `IrreversibleToolInSearchException`. Only the final committed trajectory may dispatch T3 tools.
3. **`LATSConfig` (`@ConfigurationProperties`)** declares: `branchingFactor` (default 3), `maxDepth` (default 8), `maxIterations` (default 20), `maxToolCalls` (default 100), `explorationConstant` (default 1.0), `evaluatorMode`.
4. **Cost budget enforcement**: `LATSController` tracks cumulative inference cost across all branches. If projected cost exceeds `lats.maxBudgetUSD`, terminate search early and return best trajectory found.
5. **The selected final trajectory is replayed as a standard ReAct run** after search completes — all nodes, edges, and logging from the ReAct architecture apply to the committed trajectory for auditability.
6. **In Java + LangGraph**: `LATSSearchTree` is a field of `AgentState`. `LATSController` orchestrates a sub-graph with nodes: `latsSelectionNode`, `latsExpansionNode`, `latsEvaluatorNode`, `latsBackpropNode`, `latsReflectNode`. The sub-graph is invoked by `OrchestrationNode` when task classification returns `LATS_REQUIRED`.

---

## 9.7 — Architecture Selection: A Decision Framework

### Context

Architecture selection should be driven by **measured task properties** — not by architectural novelty, framework defaults, or benchmark performance on tasks unlike your own. The most reliable selection process is empirical: instrument the agent with the simplest architecture (ReAct), measure failure rates on a representative task sample, and escalate architectural complexity only for task classes where the data justifies the additional cost and operational burden.

### What (the spec)

**Five decision dimensions for architecture selection:**

| Dimension | Low value → simpler architecture | High value → more complex architecture |
|---|---|---|
| **Task complexity** | Few plausible paths at key decision points | Many plausible paths; high ambiguity |
| **Latency tolerance** | User/system requires fast response | Deliberative latency is acceptable |
| **Cost per task** | Tight inference budget | Higher budget available for quality |
| **Failure consequences** | Low cost of incorrect first path | High cost of wrong committed path (irreversible actions) |
| **Observability requirements** | Basic audit trail sufficient | Every reasoning step must be provenance-traceable |

**Architecture selection decision matrix:**

| Scenario | Recommended architecture | Rationale |
|---|---|---|
| Standard ticket: clear complaint, single plausible path | **ReAct** | Linear, auditable, low cost |
| Complex billing dispute: misclassification common, retries acceptable | **Reflexion** | Self-correction on retry without tree search overhead |
| Fraud / policy-conflict investigation: high ambiguity, high stakes, irreversible actions | **LATS** | Exploration quality justifies inference cost |
| Rule-governed domain: explicit commitments, stable state, regulatory compliance | **BDI-influenced hybrid** | Commitment semantics + reactive safety layers |
| Multi-agent orchestration with specialist subagents | **BDI orchestrator + ReAct subagents** | Deliberative orchestration + simple execution |

**Empirical selection process (four steps — required before any Reflexion or LATS adoption):**

1. **Baseline measurement:** Run ReAct on a representative sample (minimum 100 tasks) of the target task class. Measure: first-attempt success rate, average cycle count, average cost, failure mode distribution.
2. **Failure analysis:** Classify failures by root cause: reasoning error (Reflexion candidate), ambiguity/multi-path (LATS candidate), tool failure (infrastructure fix, not architecture change), input quality (perception fix, not architecture change).
3. **Architecture upgrade decision:** Upgrade to Reflexion if reasoning-error failures > 20% of total. Upgrade to LATS if ambiguity failures > 15% AND failure consequence is high AND latency budget allows.
4. **Post-upgrade measurement:** After upgrade, re-measure all five dimensions. Document the improvement (or lack thereof) in the `AgentArchitectureDocument`. If improvement is < 10 percentage points, revert to simpler architecture.

**Architecture is not a one-time decision.** Teams that start with ReAct, observe failure patterns in production, and upgrade specific task classes to Reflexion or LATS based on those observations make better decisions than teams that choose the most sophisticated architecture upfront.

### Why (engineering implications)

- The empirical requirement before Reflexion/LATS adoption is a budget and operational discipline, not bureaucracy. LATS at K=3, depth=8 can cost 50–100x more per task than ReAct. Adopting it without data-driven justification is a material cost decision made without evidence.
- Architecture escalation should be **per task class**, not per agent. A single agent handling multiple task classes should route standard tasks through ReAct and complex investigations through LATS using a `TaskClassifier` at `PerceiveNode`. This is more cost-efficient than running the entire agent workload through LATS.
- The failure mode classification in step 2 is critical. Teams that misclassify tool failures as reasoning errors add Reflexion to a problem that requires infrastructure fixes. The reflection will always reach the same `rootCause` (tool unavailable), triggering no-progress escalation — which is the correct behavior, but wastes retry budget before escalating.

### How (system design rules)

1. **Implement `TaskClassifier`** as a `@Component` called in `PerceiveNode`. It classifies each incoming task as `STANDARD`, `COMPLEX_REASONING`, or `HIGH_AMBIGUITY` using a rule-based classifier (keyword matching + input complexity heuristics) backed by a configurable `TaskClassificationConfig`.
2. **`OrchestrationNode` routes by task class:** `STANDARD` → ReAct sub-graph; `COMPLEX_REASONING` → Reflexion sub-graph; `HIGH_AMBIGUITY` → LATS sub-graph.
3. **Baseline metrics are collected automatically** via `ArchitectureMetricsCollector` (`@Component`): first-attempt success rate, failure mode tags (`REASONING_ERROR`, `AMBIGUITY`, `TOOL_FAILURE`, `INPUT_QUALITY`), average cycle count, average inference cost per task. Published to Prometheus with labels `architecture=react|reflexion|lats` and `taskClass=standard|complex|high_ambiguity`.
4. **`AgentArchitectureDocument` records the empirical justification** for any architecture above ReAct: baseline metrics, failure analysis, and post-upgrade delta.
5. **In Java + LangGraph**: `TaskClassifier.classify(AgentState)` returns a `TaskClass` enum. `OrchestrationNode` reads `AgentState.taskClass` and routes to the appropriate sub-graph via a conditional edge. `ArchitectureMetricsCollector` is a Spring `@EventListener` on `AgentRunCompletedEvent`.

---

## 9.8 — Implementing LangGraph Topology for Each Architecture

### Context

Each cognitive architecture maps to a distinct LangGraph `StateGraph` topology. The nodes defined in previous chapters (Chapters 4–8) are the building blocks; the architecture determines how they are connected, which conditional edges exist, and what signals route between them. This section specifies the canonical LangGraph topology for each architecture.

### What (the spec)

**ReAct topology (canonical):**

```
START
  └─► perceiveNode
        └─► reasonNode ──[TOOL_CALL]──► actNode ──► perceiveNode (loop)
                      └─[FINAL_ANSWER]─► outputNode ──► END
                      └─[ERROR]────────► handleError ──► END
```

**Reflexion topology (extends ReAct):**

```
START
  └─► reflexionController ──[inject_reflection]──► perceiveNode
                                                      └─► [ReAct loop]
                                                            └─► evaluatorNode
                                                                  ├─[PASS]──────────────────► outputNode ──► END
                                                                  ├─[FAIL + retry_allowed]──► reflectNode ──► reflexionController (next attempt)
                                                                  └─[FAIL + exhausted]──────► handleDegraded ──► END
```

**LATS topology (extends ReAct with search sub-graph):**

```
START
  └─► taskClassifier
        ├─[STANDARD]──────────────► [ReAct sub-graph]
        ├─[COMPLEX_REASONING]─────► [Reflexion sub-graph]
        └─[HIGH_AMBIGUITY]────────► latsController
                                        └─► [MCTS loop: latsSelection → latsExpansion → latsEvaluator → latsBackprop]
                                              └─[SEARCH_COMPLETE]──► commitTrajectory ──► [ReAct replay of committed trajectory] ──► outputNode ──► END
```

**BDI-influenced hybrid topology:**

```
START
  └─► perceiveNode (update beliefs)
        └─► desireGenerationNode (generate candidate subgoals)
              └─► intentionSelectionNode (activate one intention)
                    └─► [ReAct execution of active intention]
                          └─► intentionRevisionNode (check success/fail/defer)
                                ├─[COMPLETED]──► intentionSelectionNode (next)
                                ├─[ALL_DONE]───► outputNode ──► END
                                └─[FAILED]─────► handleError
```

**Common LangGraph implementation rules for all topologies:**

1. **Every sub-graph is a named `StateGraph<AgentState>`** compiled independently and invoked as a node in the parent graph. Sub-graphs are not inlined.
2. **All conditional edges are named `Function<AgentState, String>`** implementing `RouterFunction` interface. Router names match the signal they read from `AgentState`.
3. **`END` is only reached via `outputNode` or `handleError`/`handleDegraded`.** No node routes directly to `END` without passing through one of these terminal nodes.
4. **The `handleError` and `handleDegraded` nodes** are defined in Chapter 4 (Agent Runtime). They are shared across all architectures — not duplicated per topology.
5. **All nodes implement `NodeFunction<AgentState>`** (a `Function<AgentState, AgentState>`) — a single interface regardless of architecture. Architecture topology is in the graph wiring, not in node logic.

### Why (engineering implications)

- Specifying canonical topologies prevents teams from inventing arbitrary graph structures that happen to work but are not maintainable. A developer who reads the LATS topology diagram above can immediately understand the control flow without reading the implementation. This is the primary value of the canonical topology specification.
- The "sub-graphs as nodes" pattern (rule 1) enables topology composition: a LATS sub-graph can be dropped into a BDI orchestrator as a single node. Without this pattern, architectures cannot be composed without rewriting graph wiring.
- The `END` routing rule (rule 3) is an observability invariant. Every run ends through a terminal node that emits a `RunCompletedEvent`. If a run can reach `END` without passing through `outputNode` or `handleError`, that run is invisible to the audit log.

### How (system design rules)

1. **`GraphFactory` is a Spring `@Component`** with factory methods: `buildReActGraph()`, `buildReflexionGraph()`, `buildLATSGraph()`, `buildBDIHybridGraph()`. Each returns a compiled `StateGraph<AgentState>`.
2. **`GraphTopologyValidator`** runs at Spring startup and validates that: every graph has exactly one `START` entrypoint, every terminal path passes through `outputNode` or `handleError`/`handleDegraded`, and no node has an unconditional edge to `END`.
3. **`RouterFunction` implementations are Spring `@Component`s** named by convention: `ReactRouter`, `ReflexionRouter`, `LATSRouter`, `BDIRouter`. They are injected into `GraphFactory` and are independently unit-testable.
4. **In Java + LangGraph**: `NodeFunction<S>` is `java.util.function.Function<S, S>`. `StateGraph<AgentState>` is built using LangGraph's Java API with `.addNode()`, `.addEdge()`, `.addConditionalEdge()`. `GraphTopologyValidator` implements `ApplicationRunner` and calls `graph.validate()` on each built graph at startup.

---

## Key Takeaways for Volume 2 Implementation

- **A cognitive architecture is a structural decision, not a prompt strategy.** Document it in `AgentArchitectureDocument` before writing the first node. The architecture determines the LangGraph topology; the topology cannot be changed without changing the architecture.
- **Three dimensions characterize any architecture:** component model (which nodes exist), control flow model (deliberative/reactive/hybrid), and adaptation model (within-run/across-run/none). All three must be declared explicitly.
- **SOAR's impasse-driven subgoal generation → `ImpasHandler`; ACT-R's activation-based retrieval → `RelevanceScorer`.** Classical mechanisms appear in modern systems under different names. Knowing the lineage helps predict failure modes.
- **BDI's intention commitment principle** is the structural antidote to goal drift. Intentions are not re-evaluated on every cycle. Revision requires one of four explicit triggers.
- **ReAct is the mandatory baseline.** Every agent in Volume 2 starts with ReAct. Reflexion and LATS are adopted only when empirical failure analysis justifies the cost.
- **Reflexion requires three termination conditions:** maximum attempts, quality threshold, and no-progress detection. Any Reflexion implementation missing these produces infinite retry loops.
- **LATS blocks T3 tools during search.** Only the committed final trajectory may dispatch irreversible actions. This is a hard safety rule, not a soft guideline.
- **Architecture selection is empirical and per task class.** Use `TaskClassifier` to route different task classes to different architectures within the same agent. Measure and document the empirical justification for every architecture above ReAct.
- **Canonical topologies compose.** Sub-graphs are nodes. BDI + LATS is built by dropping the LATS sub-graph into the BDI intention execution slot — no graph rewriting required.
