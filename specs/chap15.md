# Chapter 15 Spec — Single-Agent Patterns

> **Volume 1 reference:** `chsingle-agent-patterns`  
> **Part:** 4 — Agent Architecture Patterns (first chapter)  
> **Depends on:** Parts 1–3 (runtime, perception, reasoning, action, state, cognitive arch, memory)  
> **Java + LangGraph implementation target:** Volume 2

---

## Chapter Purpose

Part 3 established the memory architecture that transforms a reactive loop into a stateful, intelligent system. Part 4 lifts the view from internal agent structure to **how agents are composed, shaped, and deployed as architectural patterns**.

Chapter 15 opens Part 4 with its most important claim:

> *A single-agent architecture is not a primitive baseline that serious systems quickly outgrow. It is often the strongest default design because it centralises reasoning, state, control, and observability in one execution context.*

This chapter defines four single-agent patterns, the conditions that determine which to apply, the control envelope that makes any pattern production-safe, and the five measurable graduation signals that justify moving to multi-agent orchestration (Chapter 16).

---

## Section 15.1 — Why Single-Agent Should Be the Default

### Context

Every additional agent in a system introduces a **coordination boundary**, and every coordination boundary carries cost: messaging overhead, synchronisation complexity, ambiguous traces, and partial-failure modes that do not exist in single-agent systems. This cost is real and often underestimated.

### The Default Rule

A single-agent design is the correct default when **most** of the following properties hold:

| Property | Engineering Implication |
|---|---|
| Work is mostly sequential | No parallelism benefit from distribution |
| Task fits within one coherent context | No context-saturation pressure |
| Narrow domain | One prompt and one toolset remain internally consistent |
| Latency target achievable serially | No parallel subtask execution required |
| Failure surface manageable centrally | Single failure boundary, single recovery path |
| Single trace desired | One decision point, one approval boundary |

> After Part 3, this rule is stronger than it sounds. Once the reader understands how difficult **memory** already is in one agent, it becomes obvious why distributing state too early is a poor engineering decision. Shared-memory coordination, conflicting summaries, cross-agent state drift, and multi-agent synchronisation bugs (chap13 §13.5–13.6) are all costs that a single agent avoids by design.

> ⚠️ **Single-agent does not mean simple.** A single agent can still use memory (Parts 3), planning, retrieval, reflection, and approval gates. The architectural question is whether coordination remains *internal* rather than distributed.

### Design Rules

| Rule ID | Rule |
|---|---|
| `SA-DEFAULT-001` | Begin every new agent project with a single-agent design. The burden of proof is on multi-agent. |
| `SA-DEFAULT-002` | Do not add a second agent because the system "feels complex". Add it only when a concrete, measurable bottleneck forces the change (§15.6). |

### Java + LangGraph Target

```java
// A single-agent LangGraph graph is the canonical starting point
// One StateGraph<AgentState>, one entry node, one set of edges
// All coordination is internal: no MessageChannel, no sub-graph delegation

StateGraph<AgentState> graph = new StateGraph<>(AgentState.class)
    .addNode("perceive",    new PerceptionNode())
    .addNode("retrieve",   new MemoryRetrievalNode())
    .addNode("reason",     new ReasoningNode())
    .addNode("act",        new ToolDispatchNode())
    .addNode("reflect",    new SelfReflectiveUpdateNode())
    .addEdge(START, "perceive")
    .addEdge("perceive", "retrieve")
    .addEdge("retrieve", "reason")
    .addConditionalEdges("reason", router) // routes to act or END
    .addEdge("act", "reflect")
    .addEdge("reflect", "reason"); // loop or END
```

---

## Section 15.2 — The Four Single-Agent Pattern Taxonomy

### Context

Single-agent architecture is not one pattern. Production systems converge on four recurring forms. Pattern selection is both a **capability decision** (what the agent can do) and an **operating decision** (what the trace looks like, what it costs, and where governance hooks attach).

### The Four Patterns

| Pattern | Best Fit | Main Strength | Main Cost | Common Failure Mode |
|---|---|---|---|---|
| **Direct Tool-Using** | Short, procedural tasks with clear action paths | Low latency, clean trace | Limited adaptability | Wrong tool or wrong sequence |
| **ReAct** | Open-ended multi-step tasks under uncertainty | Adaptive step-by-step reasoning | Loop risk, token growth | Circular behaviour, runaway loop |
| **Planner–Executor** | Long workflows with dependent steps | Global coherence, inspectable plan | Upfront planning cost | Stale or hallucinated plan |
| **Reflective** | Tasks with evaluable outputs | Quality improvement through revision | Extra latency and retries | Self-confirmed wrong answers |

### Pattern Selection by Task Shape

```
Is the next action obvious and workflow short?      → Direct Tool-Using
Is the task uncertain and path discovered in-run?   → ReAct
Are there many dependent steps needing ordering?    → Planner–Executor
Is output evaluable and quality critical?           → Add Reflection wrapper
```

### Pattern Selection by Operating Constraints

| Constraint | Preferred Pattern |
|---|---|
| Cleanest trace needed | Direct Tool-Using or Planner–Executor |
| Deterministic replay required | Avoid pure ReAct adaptive loops |
| Token consumption must be tight | Avoid Reflection or deep planning |
| Approval hooks must insert cleanly | Planner–Executor (explicit plan review point) |

> 💡 **Patterns compose.** A production agent may use Direct Tool-Using as base, escalate to ReAct under uncertainty, generate a plan for long-horizon work, and invoke Reflection only when output quality is questionable.

### Design Rules

| Rule ID | Rule |
|---|---|
| `PAT-001` | Select the pattern whose **failure mode** is easiest to tolerate for the task at hand, not by novelty. |
| `PAT-002` | Document the selected pattern in `AgentConfig` so all operators understand the expected trace shape. |
| `PAT-003` | Treat pattern selection as a runtime design decision — it determines trace structure, retry behaviour, token growth, and approval insertion points. |

---

## Section 15.3 — The Direct Tool-Using Agent

### Context

The direct tool-using agent is the most operationally efficient single-agent form. The agent receives a goal, selects one or more tools, executes them, and returns a result with minimal internal deliberation. No iterative reasoning loop — just a compact, auditable action path.

### When to Apply

Apply when:
- The environment is well understood and the action path is stable
- The task is mostly procedural: retrieve data → transform → validate → return
- Common examples: structured lookups, controlled API workflows, deterministic reporting, bounded content transformations

### Operational Advantages

- **Trace is short** — tool calls are easy to inspect
- **Failure attribution is straightforward** — the failing node is immediately visible
- **Runtime budgets are easy to estimate** — no loop risk
- **Human review is simpler** — fewer intermediate turns

### Failure Modes

| Failure | Root Cause | Prevention |
|---|---|---|
| Wrong tool selected | Task underspecified | Tighter tool descriptions (chap action layer) |
| Premature completion | Agent mistakes partial for full | Explicit completion criteria in `AgentPolicy` |
| Rigid sequence despite branch signal | No conditional routing | Add `ConditionalEdge` on tool result |
| Over-reliance on prompt instructions | Tool descriptions do insufficient control work | Raise tool contract quality bar |

> ⚠️ **Do not mistake low reasoning visibility for high reliability.** A short trace may look clean while masking brittle decision logic. Simplicity reduces surface area but does not automatically improve correctness.

> Since the agent does less deliberative work, **tool quality is the primary correctness lever**. Tool descriptions, argument schemas, and error semantics must compensate for reduced internal reasoning. When this pattern is chosen, the quality bar on tool contracts rises.

### Design Rules

| Rule ID | Rule |
|---|---|
| `DTA-001` | Use the Direct Tool-Using pattern for short, bounded, mostly procedural tasks only. |
| `DTA-002` | Tool contracts MUST meet the specification from the Action Layer chapter before this pattern is considered production-ready. |
| `DTA-003` | Add `ConditionalEdge` routing on tool results whenever the next action depends on what the tool returned. |

### Java + LangGraph Target

```java
// Direct tool-using graph: no explicit reasoning loop
StateGraph<AgentState> graph = new StateGraph<>(AgentState.class)
    .addNode("perceive", new PerceptionNode())
    .addNode("dispatch", new ToolDispatchNode())   // selects and calls tools
    .addNode("validate", new OutputValidationNode())
    .addEdge(START, "perceive")
    .addEdge("perceive", "dispatch")
    .addConditionalEdges("dispatch", new ToolResultRouter()
        // COMPLETE → validate → END
        // NEEDS_MORE → dispatch (bounded retry)
    )
    .addEdge("validate", END);
```

---

## Section 15.4 — The ReAct Single Agent

### Context

The ReAct pattern (chap cognitive architectures, §12) is the default architecture for tasks whose action path **cannot be known in advance**. The agent alternates between local reasoning and local action — each observation changes the next decision. It is the correct pattern for exploratory, open-ended, or partially observable tasks.

### When to Apply

- Environment must be interrogated before the right action sequence becomes clear
- Research tasks where evidence must be gathered iteratively
- Troubleshooting tasks where each observation narrows the diagnosis
- Retrieval-heavy tasks where the first search rarely provides sufficient coverage
- Mixed-tool workflows where branching depends on intermediate outputs

### Direct Tool-Using vs ReAct Trade-Offs

| Dimension | Direct Tool-Using | ReAct |
|---|---|---|
| Latency | Lower | Higher |
| Trace clarity | Higher | Lower |
| Adaptability | Lower | Higher |
| Token growth | More predictable | Less predictable |
| Loop risk | Low | Significant |
| Recovery from partial failure | Limited | Stronger |

### Mandatory Production Constraints for ReAct

The same adaptive loop that makes ReAct powerful makes it dangerous without constraints. **All four must be enforced before ReAct serves production traffic:**

| Constraint | Default | Purpose |
|---|---|---|
| `maxSteps` | 15 | Terminates runaway loops |
| `maxTokens` | Context window × 0.80 | Prevents context overflow |
| `loopDetector` | Detects repeated (thought, action) pairs | Catches circular behaviour |
| `stopCondition` | Quality evaluator on last output | Enables early exit when goal is met |

> ⚠️ **ReAct is a generator of activity unless explicitly constrained.** Without step budgets, token budgets, loop detection, and stop conditions, a ReAct agent produces work rather than results.

### Design Rules

| Rule ID | Rule |
|---|---|
| `REACT-001` | All four ReAct production constraints MUST be set before the first production deployment. |
| `REACT-002` | ReAct MUST NOT be applied to tasks where the action path is already known — pay for adaptability only when needed. |
| `REACT-003` | Loop detection MUST compare (thought, action) pairs, not just action calls, to catch semantic repetition. |
| `REACT-004` | When `maxSteps` is reached without a `stopCondition` trigger, execution MUST terminate with a structured partial-result response, never silently. |

### Java + LangGraph Target

```java
public record ReActPolicy(
    int maxSteps,           // default: 15
    int maxTokens,          // default: contextWindowSize * 0.8
    int loopWindowSize,     // number of (thought, action) pairs to track (default: 4)
    double stopThreshold    // quality score to trigger early exit (default: 0.85)
) {}

// LangGraph ReAct graph with loop guard
StateGraph<AgentState> graph = new StateGraph<>(AgentState.class)
    .addNode("reason", new ReasoningNode(policy))
    .addNode("act",    new ToolDispatchNode())
    .addNode("guard",  new LoopGuardNode(policy))   // enforces maxSteps, loop detection
    .addEdge(START, "reason")
    .addConditionalEdges("reason", new ReActRouter()
        // ACT → act, DONE → END, BUDGET_EXCEEDED → guard
    )
    .addEdge("act", "reason")
    .addEdge("guard", END); // emits structured partial result
```

---

## Section 15.5 — The Planner–Executor Single Agent

### Context

The Planner–Executor pattern addresses a structural weakness of ReAct: a reactive loop is good at deciding what to do *next*, but poor at preserving a **coherent global strategy** across a long chain of dependent actions. Planner–Executor separates whole-task organisation from step-level execution.

> This is still a **single-agent** design. One LLM-backed agent performs both phases. What changes is the internal control structure, not the agent count.

### When to Apply

| Property | Signal |
|---|---|
| Many dependent steps | Ordering matters; premature execution is costly or irreversible |
| Operator wants plan inspection | User or operator reviews/approves plan before work begins |
| Long task horizon | Local greedy choices accumulate into bad global behaviour |
| Structured checkpointing needed | Plan doubles as task checkpoint (chap13 §13.4) |

Examples: migration workflows, long-form report generation, multi-phase coding tasks, complex API orchestration, enterprise processes with approval points.

### Structured Plan Schema

Plans MUST be represented as structured data, not free-form prose. Each step carries:

```json
{
  "id": 1,
  "goal": "Retrieve recent financial filings",
  "tool": "documentSearch",
  "prereqs": [],
  "expectedOutput": "List of filing documents from past 12 months"
}
```

Structured plans are:
- **Validatable** before execution (tool references checked against registry)
- **Persistable** as a task checkpoint (chap13 `TC-001`)
- **Presentable** to a human reviewer before any action is taken

### Planner–Executor Failure Modes

| Failure Mode | Description | Mitigation |
|---|---|---|
| **Stale plan** | Environment changes during execution | Re-evaluate plan after steps with external dependencies |
| **Over-planning** | Deep plans for simple tasks waste tokens and latency | Apply planning only when task exceeds `minPlanSteps` (default: 5) |
| **Plan hallucination** | Steps reference non-existent tools | Validate all tool references against tool registry before execution |

> ⚠️ **The failure mode of a weak plan is not inefficiency — it is false confidence.** A bad plan makes a system look organised while producing worse outcomes than a simpler reactive loop.

### Design Rules

| Rule ID | Rule |
|---|---|
| `PE-001` | Plans MUST use the structured step schema — not free-form prose. |
| `PE-002` | All tool references in the plan MUST be validated against the tool registry before execution begins. |
| `PE-003` | Plans MUST be stored as task checkpoints (chap13 `TC-001`) to enable resumption. |
| `PE-004` | Apply planning only when the task exceeds `minPlanSteps` (default: 5 dependent steps). |
| `PE-005` | After each step with an external dependency, re-validate the remaining plan steps for staleness. |

### Java + LangGraph Target

```java
public record PlanStep(
    int id, String goal, String tool,
    List<Integer> prereqs, String expectedOutput
) {}

public record ExecutionPlan(
    String taskId, List<PlanStep> steps, Instant createdAt
) {}

// LangGraph Planner–Executor graph
StateGraph<AgentState> graph = new StateGraph<>(AgentState.class)
    .addNode("plan",     new PlannerNode())         // generates ExecutionPlan
    .addNode("validate", new PlanValidationNode())  // PE-002: tool registry check
    .addNode("execute",  new ExecutorNode())        // executes one step at a time
    .addNode("replan",   new ReplanNode())          // PE-005: staleness check
    .addEdge(START, "plan")
    .addEdge("plan", "validate")
    .addConditionalEdges("validate",
        // VALID → execute, INVALID → plan (replan with error context)
    )
    .addConditionalEdges("execute",
        // NEXT_STEP → replan, DONE → END, FAILED → error handler
    )
    .addEdge("replan", "execute");
```

---

## Section 15.6 — The Reflective Single Agent

### Context

The Reflective pattern adds a **review phase** to execution. The agent produces an output, evaluates it against explicit criteria, and revises if the evaluation falls short. It is a bounded quality-improvement pattern — not a substitute for evidence, testing, or verification.

### When Reflection Works

| Task Type | Reflection Value | Why |
|---|---|---|
| Code generation | **High** | Output can be executed and tested |
| Long-form synthesis | **High** | Structure, coverage, evidence can be reviewed |
| Simple retrieval answer | **Low** | Minimal gain, extra latency |
| Factual lookup without tools | **Low** | Reflection cannot invent missing facts |
| Policy-constrained drafting | **Medium–High** | Useful for checking omissions and violations |

### When Reflection Fails

- Evaluation criteria are vague
- The model lacks the external knowledge needed to correct its own mistake
- The revision budget is unlimited (becomes a runaway process)

### Conditional Reflection Rule

Reflection MUST be **conditional**, not universal. Apply it when:
1. Evaluation is real and objective (test suite, rubric, evidence check)
2. Output quality matters enough to justify a second pass
3. Likely errors are repairable by the agent using its available tools

> 💡 **Reflection is strongest when paired with external checks.** The best reflective systems ask "Did this pass the test, satisfy the rubric, or remain consistent with the retrieved evidence?" — not just "Is this good?"

### Design Rules

| Rule ID | Rule |
|---|---|
| `REF-001` | Reflection MUST be conditional — activated only for tasks where evaluation is real and output quality justifies the second pass. |
| `REF-002` | `maxRevisions` MUST be set before deployment (default: 3). Unlimited retries are a production defect. |
| `REF-003` | The evaluator MUST produce a structured assessment (pass/fail + specific failure reason) — not a qualitative impression. |
| `REF-004` | When `maxRevisions` is exhausted, execution MUST escalate (human review or structured partial output) — not silently return the last revision. |

### Java + LangGraph Target

```java
public record ReflectionPolicy(
    int maxRevisions,        // default: 3
    double qualityThreshold, // score to trigger early stop (default: 0.85)
    String evaluatorPrompt   // structured pass/fail + reason
) {}

// LangGraph Reflective wrapper — adds two nodes to any pattern
StateGraph<AgentState> graph = baseGraph
    .addNode("evaluate", new EvaluatorNode(policy))   // scores output
    .addNode("revise",   new RevisionNode(policy))    // rewrites on fail
    .addConditionalEdges("evaluate",
        // PASS → END, FAIL + attempts < maxRevisions → revise,
        // FAIL + attempts == maxRevisions → escalate
    )
    .addEdge("revise", "evaluate");
```

---

## Section 15.7 — The Control Envelope for Single Agents

### Context

Architectural patterns are not only about task completion — they are also about **control**. A production single-agent system must operate inside a defined control envelope: what it may do autonomously, what it may not do, what requires approval, and how abnormal behaviour is interrupted.

Single-agent systems have a decisive governance advantage over multi-agent systems: **one place to enforce action policy, one approval boundary, one place to attach observability**.

### Five Control Envelope Components

| Component | Purpose | Enforced By |
|---|---|---|
| **Input constraints** | Reject or route disallowed requests before agent starts | `PerceptionNode` filter (chap perception) |
| **Action constraints** | Limit tool scope, write permissions, spend thresholds, destination domains | `ToolDispatchNode` policy (chap action layer) |
| **Output constraints** | Catch unsafe or policy-violating responses before release | `OutputValidationNode` |
| **Approval gates** | Require human or policy engine to authorise specific actions | `HumanApprovalNode` (chap17) |
| **Circuit breakers** | Terminate execution when step limits, token budgets, retry thresholds, or anomaly rules are exceeded | `CircuitBreakerNode` |

> ⚠️ **Prompt instructions are not a sufficient control mechanism for high-risk actions.** Approval policy, write permissions, and circuit breakers belong in runtime enforcement layers, not only in natural-language instructions.

### Governance Ease by Pattern

| Pattern | Governance Ease | Why |
|---|---|---|
| Direct Tool-Using | **High** | Few steps, few branches — approval insertion is easy |
| ReAct | **Medium** | More branches and loop risk increase enforcement points |
| Planner–Executor | **High** | Plans create explicit, inspectable review points before action |
| Reflective | **Medium** | Additional revision iterations require bounded retry policy |

### Design Rules

| Rule ID | Rule |
|---|---|
| `CE-001` | All five control envelope components MUST be configured before the agent serves production traffic. |
| `CE-002` | Action constraints MUST be enforced in the runtime layer (code), not only in prompt instructions. |
| `CE-003` | Circuit breakers MUST write a task checkpoint (chap13 `TC-003`) before terminating execution. |
| `CE-004` | Approval gates MUST be deterministic — the same action class always triggers the same gate. No probabilistic approval routing. |

### Java + LangGraph Target

```java
public record ControlEnvelope(
    InputPolicy inputPolicy,
    ActionPolicy actionPolicy,
    OutputPolicy outputPolicy,
    ApprovalGatePolicy approvalPolicy,
    CircuitBreakerPolicy circuitBreakerPolicy
) {}

public record CircuitBreakerPolicy(
    int maxSteps,
    int maxTokens,
    int maxRetries,
    double anomalyThreshold,
    boolean checkpointOnBreak  // CE-003: always true in production
) {}

// Injected into every node via AgentConfig
// All nodes check circuitBreakerPolicy.shouldBreak(state) before executing
```

---

## Section 15.8 — Graduation Signals: When One Agent Stops Being Enough

### Context

The wrong time to move to multi-agent is when the system *feels complex*. The right time is when single-agent architecture creates a **clear, measurable bottleneck** that better prompting, memory management, tools, or control structure cannot solve.

> The most common premature mistake: adding a second coordinator agent without a real decomposition. That creates a multi-agent system in topology but not in capability. If the second agent does not reduce specialisation pressure, create parallelism, improve isolation, or reduce blast radius, it is overhead disguised as architecture.

### The Five Graduation Signals

| Signal | Definition | Measurable Indicator |
|---|---|---|
| **Context saturation** | Agent cannot retain enough relevant state to perform well, even after pruning, summarisation, retrieval, and memory optimisation | Important facts repeatedly forgotten or inconsistently reintroduced despite chap12–13 measures |
| **Specialisation mismatch** | Task requires different expert behaviours that interfere with each other when forced into one prompt and one execution persona | One prompt change improving one behaviour measurably degrades another |
| **Throughput ceiling** | Task contains independent work that could run in parallel, but a single agent serialises it | P95 latency cannot meet SLA even with tool-level optimisation |
| **Isolation requirement** | Some subtasks need different permissions, runtime constraints, or failure domains (code execution, untrusted content, regulated data) | Security or compliance audit flags co-located execution as non-compliant |
| **Blast-radius reduction** | One sub-workflow is brittle, expensive, risky, or externally unstable; isolation would improve reliability | Single sub-workflow failure rate causes system-level failure rate > tolerance |

### Graduation Decision Protocol

```
1. Identify which graduation signal(s) are present
2. Attempt resolution within single-agent:
   - Context saturation → H-MEM (chap14 §14.2), pruning (chap12)
   - Specialisation mismatch → prompt segmentation, tool contract tightening
   - Throughput ceiling → tool-level async optimisation
3. If the signal persists after single-agent remediation → proceed to chap16
4. Document which signal forced the graduation and what was attempted first
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `GRAD-001` | Moving to multi-agent MUST be justified by at least one of the five graduation signals measured in production. |
| `GRAD-002` | At least one single-agent remediation attempt MUST be documented before multi-agent design is begun. |
| `GRAD-003` | The graduation reason MUST be recorded in the architecture decision record (ADR) for the system. |

---

## Section 15.9 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `SA-DEFAULT-001` | Default | Begin with single-agent; burden of proof is on multi-agent |
| `SA-DEFAULT-002` | Default | Add second agent only for measurable bottleneck, not complexity perception |
| `PAT-001` | Pattern selection | Choose by tolerable failure mode, not novelty |
| `PAT-002` | Pattern selection | Document selected pattern in `AgentConfig` |
| `PAT-003` | Pattern selection | Pattern determines trace, retry, token growth, approval insertion |
| `DTA-001` | Direct Tool-Using | Apply to short, bounded, procedural tasks only |
| `DTA-002` | Direct Tool-Using | Tool contracts MUST meet Action Layer specification |
| `DTA-003` | Direct Tool-Using | Add `ConditionalEdge` when next action depends on tool result |
| `REACT-001` | ReAct | All four production constraints set before deployment |
| `REACT-002` | ReAct | Do not apply where action path is already known |
| `REACT-003` | ReAct | Loop detection compares (thought, action) pairs |
| `REACT-004` | ReAct | Budget exhaustion → structured partial result, never silent termination |
| `PE-001` | Planner–Executor | Plans use structured step schema — no free-form prose |
| `PE-002` | Planner–Executor | Tool references validated against registry before execution |
| `PE-003` | Planner–Executor | Plans stored as task checkpoints (chap13 `TC-001`) |
| `PE-004` | Planner–Executor | Planning applied only when task exceeds `minPlanSteps` |
| `PE-005` | Planner–Executor | Re-validate remaining steps after each external-dependency step |
| `REF-001` | Reflective | Reflection is conditional, not universal |
| `REF-002` | Reflective | `maxRevisions` set before deployment (default: 3) |
| `REF-003` | Reflective | Evaluator produces structured pass/fail + specific failure reason |
| `REF-004` | Reflective | Exhausted revisions → escalation, not silent return |
| `CE-001` | Control Envelope | All five components configured before production traffic |
| `CE-002` | Control Envelope | Action constraints in runtime code, not prompt instructions |
| `CE-003` | Control Envelope | Circuit breakers write checkpoint before terminating |
| `CE-004` | Control Envelope | Approval gates are deterministic — same action class, same gate |
| `GRAD-001` | Graduation | Multi-agent justified by at least one measured graduation signal |
| `GRAD-002` | Graduation | At least one single-agent remediation attempt documented first |
| `GRAD-003` | Graduation | Graduation reason recorded in ADR |

---

## Section 15.10 — Transition to Chapter 16

Chapter 15 established that single-agent architecture is the strongest default because it maximises clarity, control, and operational simplicity. It also established the five measurable graduation signals that identify when a single-agent design is no longer sufficient.

**Chapter 16** begins where those signals end. It examines **multi-agent orchestration** — the deliberate, governed coordination of multiple agents:

| Chapter 16 Topic | Memory/Single-Agent Dependency |
|---|---|
| Why multi-agent orchestration exists (four pressures) | Graduation signals from §15.8 |
| Supervisor–Worker pattern | chap13 shared memory, chap15 control envelope |
| Manager–Specialist pattern | chap15 specialisation mismatch signal |
| Pipeline and Handoff patterns | chap15 Planner–Executor as single-agent precursor |
| Blackboard / Shared State pattern | chap13 §13.5 shared memory architecture |
| Hierarchical orchestration | chap13 §13.6 synchronisation and conflict resolution |

> The memory foundations of Part 3 and the single-agent discipline of Chapter 15 are the substrate on which multi-agent orchestration patterns rest. Without them, multi-agent systems reduce to stateless function composition.
