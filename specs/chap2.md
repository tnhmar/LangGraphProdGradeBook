# Chapter 2 — The Core Agentic Loop

**Part 1: Foundations of Agentic AI**  
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

This chapter defines the operational loop that makes an agent an agent. A single LLM call can answer once; an agent sustains goal-directed behavior by cycling through **Perceive → Reason → Act → Reflect** under the control of an orchestrator. This file captures the production-grade specifications that Volume 2 must implement in Java with LangGraph.

---

## 2.1 — OODA as a Historical Frame

### Context

The chapter begins with John Boyd's OODA loop — Observe, Orient, Decide, Act — because it is a useful historical model for iterative decision-making under uncertainty. It explains why an agent is not a static responder but a feedback-driven system that repeatedly interprets a changing environment and updates its next move.

### What (the spec)

OODA maps directly to modern agent engineering, but only as an **incomplete first approximation**.

| OODA Phase | Agent Equivalent | Primary Responsibility |
|---|---|---|
| Observe | Perception | Ingest user input, tool output, and environment state |
| Orient | Memory + context assembly | Interpret current state relative to goal |
| Decide | Reasoning | Select the next action or final response |
| Act | Action | Execute tool, API call, file operation, or delegation |

### Why (engineering implications)

- OODA is useful because it emphasizes cycle speed, feedback quality, and the fact that each action changes the next decision context.
- OODA is **not sufficient** for LLM-based agents because it omits two production-critical concerns: **reflection** and **termination**.
- A system built only with Observe–Orient–Decide–Act may execute repeatedly, but it lacks a structured mechanism for evaluating whether progress is real and for deciding when to stop.

### How (system design rules)

1. **Use OODA only as a conceptual orientation tool**, not as the final implementation model.
2. **Extend OODA into the Perceive–Reason–Act–Reflect model** by making reflection explicit and by implementing termination as orchestrator logic.
3. **Never treat the model call as the whole loop.** The model participates only in the Decide/Reason phase. The rest is software architecture.
4. **Instrument cycle latency by phase**, because OODA's real operational value is that it encourages thinking in terms of loop speed and decision quality, not just answer quality.

---

## 2.2 — The Perceive–Reason–Act–Reflect Cycle

### Context

The book refines OODA into the modern loop used for LLM-based agents: **Perceive → Reason → Act → Reflect**. Reflection is the missing evaluation phase that converts action results into controlled next-step decisions. Without it, systems repeat failures, declare success prematurely, or drift away from the original task.

### What (the spec)

The loop is composed of four engineered phases, each with distinct inputs, outputs, and failure modes.

| Phase | Input | Output | Core Responsibility |
|---|---|---|---|
| **Perceive** | User task, tool results, memory retrievals, prior state | Normalized context package | Filter, normalize, and assemble what the model will see |
| **Reason** | Normalized context package | Tool call, final response, or reasoning trace | Choose the next step |
| **Act** | Structured decision from Reason | Tool result, side effect, or final delivery | Execute safely and log what happened |
| **Reflect** | Action outcome + current goal state | Continue, revise, retry, or terminate | Evaluate progress and choose the next loop transition |

### Why (engineering implications)

- **Perceive** is not passive intake. It is filtering, formatting, and relevance control. Poor perception pollutes all later phases.
- **Reason** depends as much on context assembly quality as on model quality. A strong model operating on a noisy or incomplete prompt still reasons badly.
- **Act** is the safety-critical phase because it creates external effects that may be irreversible.
- **Reflect** is what prevents blind repetition. It transforms raw outcomes into structured control decisions.

### How (system design rules)

1. **Implement each phase as a separate runtime responsibility.** In Java + LangGraph: `PerceiveNode`, `ReasonNode`, `ActionNode`, `ReflectNode` — not a single monolithic loop method.
2. **Model phase transitions explicitly in the graph.** `perceiveNode -> reasonNode -> actionNode -> reflectNode`, with conditional routing from `reflectNode` back to `perceiveNode` or to a termination path.
3. **Treat reflection as a control function, not a narrative feature.** Reflection must output one of a bounded set of next-step decisions: `CONTINUE`, `REVISE`, `RETRY`, `TERMINATE`, `ESCALATE`.
4. **Make all phase inputs and outputs typed.** Reflection must not return free-form prose that the orchestrator must interpret heuristically.
5. **Start with embedded reflection for simple agents** and promote to explicit `ReflectNode` when observability, auditability, or independent recovery requirements demand it.

---

## 2.3 — Embedded vs. Explicit Reflection

### Context

Reflection can be implemented in two ways. In simple systems, the next Reason step reads the last action result and implicitly decides what to do next. In more advanced systems, reflection is a separate, explicit evaluation step with its own prompt, output schema, and audit trail.

### What (the spec)

| Reflection Style | Mechanism | Advantages | Costs |
|---|---|---|---|
| **Embedded reflection** | The next Reason call incorporates evaluation of the last action result in the same model invocation | Simpler; cheaper; fewer model calls | Less inspectable; harder to audit independently |
| **Explicit reflection** | Separate `ReflectNode` decides whether to continue, revise, retry, or terminate before the next Reason call | Better observability, auditability, and controllability | More latency; more per-cycle cost; more orchestration complexity |

### Why (engineering implications)

- Embedded reflection is sufficient for many multi-step workflows and is the correct default starting point.
- Explicit reflection becomes necessary when the system must independently log why a transition was taken, when action safety requires a separate gate, or when failure recovery logic is becoming opaque inside the model prompt.
- Reflection style is a **production design choice**, not a theoretical purity decision.

### How (system design rules)

1. **Default to embedded reflection in first implementations** to keep cost and orchestration complexity low.
2. **Introduce explicit `ReflectNode` when at least one of these is true**: independent audit of self-evaluation is required; action safety requires a separate gate; recovery logic is becoming opaque.
3. **If explicit reflection is used, it must emit a strict schema**, e.g.:
   - `technicalSuccess: boolean`
   - `goalProgress: enum { ADVANCED, NO_CHANGE, REGRESSED }`
   - `nextDecision: enum { CONTINUE, REVISE, RETRY, TERMINATE, ESCALATE }`
   - `reasonCode: string` (structured code, not free text)
4. **Allow model selection per phase.** A smaller, cheaper model may be used for reflection if the evaluation question is narrow and well-bounded.

---

## 2.4 — Task Depth Classification

### Context

Not every task requires the same loop architecture. One of the most common design mistakes is building full agent machinery for tasks that are actually single-step, or building single-step systems for tasks that are long-horizon and therefore unstable without checkpointing, memory management, and explicit recovery.

### What (the spec)

Tasks are classified by **loop depth** into three practical engineering categories.

| Task Class | Typical Loop Depth | Characteristics | Required Components |
|---|---|---|---|
| **Single-step** | 1 cycle | All required information fits in one context window; no cross-step dependency | Prompt + one model call; no full agent loop required |
| **Multi-step** | 2–10 cycles (practical rule) | Later steps depend on earlier results; tool outcomes shape next decisions | Goal tracking, reflection, termination logic |
| **Long-horizon** | 10+ cycles, or cross-session, or checkpoint-dependent | Loop stability itself becomes a first-class engineering problem | Checkpointing, recovery, active memory management, divergence detection |

### Why (engineering implications)

- A single-step task does not need a full reflective loop or durable goal state.
- A multi-step task requires goal tracking and explicit termination because the next step depends on observed results.
- A long-horizon task introduces new failure modes: goal drift, context saturation, cumulative cost, and interruption recovery.
- The boundary between multi-step and long-horizon is an engineering signal that loop stability has become a first-class concern, not a hard numerical rule.

### How (system design rules)

1. **Classify task depth before implementation**, as part of requirements analysis, not after failures surface in production.
2. **Do not use a full agent runtime for single-step tasks** unless there is a clear, documented future need for extension.
3. **Require explicit goal state and reflection for all multi-step tasks.**
4. **Require LangGraph `checkpointer` persistence for all long-horizon tasks.** A long-horizon task that cannot recover from interruption is not production-ready.
5. **Make loop-depth classification a named property of each workflow type** in the system's architecture documentation.

---

## 2.5 — The LLM's Role Inside the Loop

### Context

A critical conceptual shift is required: in a chat application the model can feel like the entire system; in an agent it is one component inside a larger runtime. The LLM occupies the Reason phase. The orchestrator owns the loop.

### What (the spec)

The LLM is a **loop component**, not a loop controller.

**What the LLM does inside the loop:**
- Interprets the context window assembled and delivered by the orchestrator.
- Produces one of three structured outputs: tool call specification, final response, or reasoning trace.
- Contributes reasoning quality inside the Reason phase.

**What the LLM does NOT do:**
- Control when the next cycle starts.
- Decide which tools are registered or available.
- Enforce permissions or authorisation rules.
- Manage retry count or retry logic.
- Assemble or filter the context window.
- Enforce termination or resource ceilings.
- Guarantee valid output schemas.

### Why (engineering implications)

- Encoding critical control logic in prompt instructions instead of software is one of the most costly architectural errors in agent systems.
- Prompt instructions can influence model behavior; they cannot reliably enforce permissions, termination, or retry ceilings.
- Treating the model as a bounded decision engine makes the system robust to model error instead of dependent on model perfection.

### How (system design rules)

1. **Implement all control logic in the orchestrator.** Permissions, retries, branching, timeouts, and termination belong in code, not in prompts.
2. **Parse model outputs into typed command objects** — `ToolRequest`, `FinalAnswer`, or `ReasoningTrace` — before acting on them.
3. **Reject unregistered tools and malformed output structures at the orchestrator layer.** The model may request them; the runtime must refuse them safely.
4. **Support model selection per phase.** Reasoning may use a stronger model; reflection may use a cheaper model. This is a runtime configuration, not a loop design change.
5. **Make model identity a runtime configuration**, not a hard-coded dependency inside loop logic.
6. **Design the runtime assuming the model will occasionally be wrong.** Every critical control path must behave safely when that happens.

---

## 2.6 — Termination as a First-Class Design Problem

### Context

A loop that does not know how to stop is not an autonomous agent. It is an unbounded process. Termination is one of the most underdesigned parts of early agent systems, and the absence of explicit termination paths is a common cause of runaway costs and silent failures in production.

### What (the spec)

Every production agent loop must support **four termination categories**.

| Termination Type | Meaning | Nature |
|---|---|---|
| **Goal completion** | Success criteria have been met | Happy-path completion |
| **Hard resource limit** | Cycle count, token budget, wall-clock time, or cost ceiling has been reached | Safety stop — not success |
| **Failure escalation** | Too many consecutive failures, parse errors, or invalid actions | Controlled failure path |
| **Explicit model declaration** | Model emits a structured infeasibility or completion signal | Model-assisted — still enforced by the orchestrator |

### Why (engineering implications)

- Goal completion requires an explicit, machine-checkable definition of success — not just "the model said it's done."
- Resource limits are mandatory because model-driven loops can continue expensively without making real progress.
- Failure escalation prevents infinite retry behavior that burns budget without outcome.
- Model declaration is useful, but cannot be the only termination path because model confidence is not a guarantee of correctness.

### How (system design rules)

1. **Deploy every agent with a hard max cycle count from day one.** Never ship a production agent without one.
2. **Define success criteria as machine-checkable conditions** wherever possible; use model evaluation as a fallback, not the primary gate.
3. **Make termination reasons observable** — log them as structured events with cycle count, token usage, goal-state summary, and final disposition.
4. **Differentiate goal completion from safety stop in all metrics and logs.** A task stopped by a resource limit is incomplete even if partial output exists.
5. **Treat model-declared infeasibility as advisory until validated by runtime policy.**
6. **Expose all termination thresholds as configuration**, not hard-coded constants. Different task types require different budgets.

---

## 2.7 — Loop Failure Modes

### Context

Loop failures differ from conventional application failures. They are often gradual and silent: the system remains active, consumes tokens, and produces outputs, but does not make meaningful progress. By the time the failure becomes obvious, cost and latency have already compounded.

### What (the spec)

Three primary loop failure modes must be explicitly detected and handled.

| Failure Mode | Signature | Root Cause Pattern | Required Detection |
|---|---|---|---|
| **Infinite loop** | Repeated or superficially varied action sequence with no goal advancement | Unhandled tool failure; retry without state change; output variation masking no real progress | Goal-state delta tracking across cycles; repeated-action detector |
| **Stuck state** | Successful step execution but no valid next action; oscillation between ambiguous outputs | Unanticipated task boundary; no valid branch for current state | Consecutive cycles with neither a valid tool call nor a final response |
| **Goal divergence** | System continues acting and progressing, but toward the wrong objective | Original goal recedes from active context; no periodic goal re-injection | Goal-state comparison across checkpoint intervals; divergence threshold |

### Why (engineering implications)

- Infinite loops are not always obvious because the surface form of each cycle may vary slightly while the state does not advance.
- Stuck states occur at the boundaries of task design — where no path forward has been explicitly defined.
- Goal divergence is the most dangerous failure mode because the system appears healthy and productive while solving the wrong problem.
- None of these failure modes is reliably self-detectable by the model alone.

### How (system design rules)

1. **Track goal advancement explicitly per cycle** — do not infer progress only from successful tool execution.
2. **Detect repeated no-progress cycles** and escalate before hard resource limits are reached.
3. **Detect stuck-state oscillation** — N consecutive cycles where the model produces neither a valid tool call nor a final answer (N configurable, default 3).
4. **Re-inject a compact, structured goal representation into every Reason call** to reduce goal divergence risk.
5. **Checkpoint goal state periodically** so divergence recovery can roll back to the last known-aligned state.
6. **Implement distinct recovery strategies per failure mode**:
   - Infinite loop → increment retry counter → escalate at ceiling.
   - Stuck state → clarification request, fallback branch, or human handoff.
   - Goal divergence → rollback to last checkpoint and re-anchor goal in context.

---

## 2.8 — The Loop Trace as a Production Artifact

### Context

The chapter closes with a full cycle trace of the earnings-report agent: Cycle 1 — retrieve report; Cycle 2 — retrieve prior thresholds; Cycle 3 — compare and produce structured output. The trace makes every loop transition visible, attributable, and diagnosable. Without structured traces, debugging becomes narrative reconstruction from incomplete logs.

### What (the spec)

A loop trace is a **first-class operational output** of the runtime — not an optional debugging aid.

**Minimum required fields per cycle trace record:**

| Field | Type | Purpose |
|---|---|---|
| `cycleNumber` | int | Position in the loop |
| `goalSummary` | string | Active goal at cycle start |
| `contextPackageSummary` | string/hash | What the model was given |
| `reasonOutputType` | enum | `TOOL_CALL`, `FINAL_ANSWER`, `REASONING_TRACE` |
| `actionInvoked` | string | Tool name or null |
| `validationResult` | enum | `PASSED`, `REJECTED`, `SKIPPED` |
| `toolResultSummary` | string | Outcome of execution |
| `reflectionDecision` | enum | `CONTINUE`, `REVISE`, `RETRY`, `TERMINATE`, `ESCALATE` |
| `goalDelta` | enum | `ADVANCED`, `NO_CHANGE`, `REGRESSED` |
| `terminationStatus` | enum | `RUNNING`, `COMPLETED`, `RESOURCE_LIMIT`, `FAILURE`, `ESCALATED` |
| `tokenUsage` | object | Input tokens, output tokens, cumulative total |
| `wallTimeMs` | long | Cycle wall-clock duration |

### Why (engineering implications)

- The loop trace is the most effective diagnostic tool for production agent systems.
- It is also the primary input to cost optimization — cycle count, tool frequency, and token consumption are only visible when traced.
- Trace records enable evaluation replay: the same cycle can be re-run against a different model or prompt to measure improvement.

### How (system design rules)

1. **Emit a structured trace record for every cycle** as a typed Java record — not ad-hoc log strings.
2. **Store traces with replay fidelity** — enough content to reconstruct inputs, but with sensitive fields redacted or access-controlled.
3. **Use traces as the basis for cost analysis, evaluation, and optimization** — not just for incident debugging.
4. **Treat trace generation as part of the runtime contract.** A production runtime that cannot explain what it did per cycle is not acceptable.
5. **Link every termination event to its cycle trace record** so operators can inspect the exact path, state, and context that ended the task.

---

## Key Takeaways for Volume 2 Implementation

- The core loop in Volume 2 is implemented as a typed LangGraph `StateGraph` with explicit `PerceiveNode`, `ReasonNode`, `ActionNode`, and `ReflectNode` phases connected by conditional edges.
- The **orchestrator is the control plane** for cycle execution; the model is a bounded decision component inside the Reason phase.
- **Reflection and termination are not optional embellishments** — they are structural requirements for any production loop.
- **Loop-depth classification** (single-step / multi-step / long-horizon) drives the architecture choice before any code is written.
- **Three loop failure modes** — infinite loop, stuck state, goal divergence — require dedicated runtime detectors with per-mode recovery strategies.
- Every production execution must produce a **structured, cycle-by-cycle trace record** conforming to the minimum field schema above.
