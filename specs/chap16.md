# Chapter 16 Spec — Multi-Agent Orchestration

> **Volume 1 reference:** `chmulti-agent-orchestration`  
> **Part:** 4 — Agent Architecture Patterns (second chapter)  
> **Depends on:** chap13 (shared memory, synchronisation, conflict resolution), chap15 (single-agent patterns, graduation signals, control envelope)  
> **Java + LangGraph implementation target:** Volume 2

---

## Chapter Purpose

Chapter 15 established the five graduation signals that justify leaving single-agent architecture. Chapter 16 begins at the moment those signals appear and examines **multi-agent orchestration** — the discipline of deliberately coordinating multiple agents through governed control structures.

The central claim:

> *A multi-agent system is not just a larger single agent. It is a different architectural choice with different benefits, different risks, and different operating costs. The hard part is not adding agents — it is defining coordination contracts, maintaining observability across agent boundaries, and keeping failures local.*

This chapter:
- Identifies the four pressures that justify multi-agent orchestration
- Defines a taxonomy of five orchestration patterns
- Specifies what each pattern demands from coordination contracts, state design, and failure handling
- Closes with the conditions under which orchestration is the wrong choice

> **Scope boundary:** This chapter covers *orchestrated* systems — systems with deliberate, governed control structure. Decentralised and emergent peer behaviour is treated in chap18.

---

## Section 16.1 — Why Multi-Agent Orchestration Exists

### Context

Single-agent architecture (chap15) is the correct default. Multi-agent orchestration is justified only when centralising all reasoning and control in one execution context becomes a **measurable bottleneck**.

There are four concrete pressures that force the move.

### The Four Pressures

| Pressure | Definition | Measurable Signal |
|---|---|---|
| **Specialisation pressure** | One agent must behave like a researcher, planner, code analyst, compliance checker, and reporter simultaneously — the prompt becomes crowded, the toolset incoherent, and behaviour blurs | One prompt change improving one behaviour measurably degrades another (chap15 `GRAD-001` signal 2) |
| **Parallelism pressure** | Independent subtasks — multiple source reviews, hypothesis tests, retrieval tasks — can proceed concurrently, but one agent serialises everything | P95 latency cannot meet SLA even with tool-level optimisation (chap15 `GRAD-001` signal 3) |
| **Isolation pressure** | Some subtasks need different permissions, runtime constraints, or failure domains (privileged enterprise actions vs untrusted web content vs code execution) | Security or compliance audit flags co-located execution as non-compliant (chap15 `GRAD-001` signal 4) |
| **Control pressure** | The workflow needs routing, delegation, escalation, aggregation, and checkpointing as explicit architectural concepts — one unit of cognition is no longer the right unit of governance | Governance artefacts (approvals, audit trails) cannot be expressed without explicit coordination structure |

> **Architecture rule:** Multi-agent orchestration is a **control topology change**, not an intelligence upgrade. It is adopted when the system needs division of labour, not when it merely feels impressive with multiple agents.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MA-JUSTIFY-001` | Multi-agent orchestration MUST be justified by at least one of the four pressures measured in production. Felt complexity is not justification. |
| `MA-JUSTIFY-002` | Each agent added to an orchestrated system MUST reduce specialisation pressure, create meaningful parallelism, improve isolation, or clarify a governance boundary. Agents that do none of these are coordination overhead. |
| `MA-JUSTIFY-003` | The architecture decision record (ADR) MUST name the pressure that triggered multi-agent design and document what was attempted in single-agent first. |

---

## Section 16.2 — Orchestration Pattern Taxonomy

### Context

Most production multi-agent systems converge on a small number of recurring orchestration patterns. Pattern selection is driven by three forces: **task decomposability**, **control requirements**, and **state-sharing burden**.

### Pattern Overview

| Pattern | Best Fit | Main Strength | Main Cost | Common Failure |
|---|---|---|---|---|
| **Supervisor–Worker** | Decomposable tasks with central authority | Clear authority and trace root | Coordinator bottleneck; supervisor as single point of failure | Over-centralisation; supervisor context saturation |
| **Manager–Specialist** | Domain-diverse tasks requiring expert routing | Strong specialisation | Routing complexity | Wrong expert assigned |
| **Pipeline / Handoff** | Ordered workflows with clear stage boundaries | Predictable sequencing | Brittle upstream assumptions | Silent error propagation |
| **Blackboard / Shared Workspace** | Tasks requiring cumulative partial-result collaboration | Flexible, flexible contribution order | State consistency burden | Conflicting or unattributed updates |
| **Hierarchical Orchestration** | Large tasks with nested delegation across many subtasks | Scales across wide task trees | Harder observability; hidden failure trees | Causal attribution collapse; policy drift |

### Pattern Selection Rule

```
Ask first: what is the smallest coordination structure that lets the work decompose safely?

Central authority + independent subtasks       → Supervisor–Worker
Domain expertise drives routing                → Manager–Specialist
Ordered stages with clear hand-off contracts   → Pipeline / Handoff
Partial results must accumulate collaboratively→ Blackboard
Task tree exceeds flat supervisor capacity     → Hierarchical
```

> 💡 **Draw the work graph before the agent graph.** Many teams decide how many agents they want and then invent a decomposition to justify them. That produces architecture-shaped overhead. The work graph — its parallelism, dependencies, ownership changes, and isolation requirements — dictates the topology.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MA-PAT-001` | Pattern selection MUST be driven by the shape of the work and the required control boundary — not by framework defaults or preference for complexity. |
| `MA-PAT-002` | The work graph MUST be drawn and reviewed before agent topology is decided. |
| `MA-PAT-003` | The selected pattern MUST be documented in `OrchestratorConfig` so all operators understand expected trace shape and coordination contract semantics. |

### Java + LangGraph Target (Config)

```java
public record OrchestratorConfig(
    OrchestrationPattern pattern,        // SUPERVISOR_WORKER | MANAGER_SPECIALIST
                                         // | PIPELINE | BLACKBOARD | HIERARCHICAL
    int maxAgents,
    int maxDelegationDepth,              // for HIERARCHICAL: default 3
    boolean requireCoordinationContracts,// must be true in production
    boolean traceCorrelationRequired     // must be true in production
) {}
```

---

## Section 16.3 — The Supervisor–Worker Pattern

### Context

The Supervisor–Worker pattern is the most common entry point into multi-agent design. A central supervising agent:
1. Receives the overall goal
2. Decomposes it into subproblems
3. Assigns work to worker agents
4. Monitors progress
5. Aggregates results

Workers do not coordinate freely among themselves — control flows through the supervisor.

### When to Apply

- The top-level task can be partitioned into relatively independent subtasks
- A central authority must decide priorities or stopping conditions
- Worker outputs can be normalised into a common format
- Strong governance over delegation and termination authority is required

### Strengths and Costs

| Dimension | Effect |
|---|---|
| Governance | Strong — authority is centralised |
| Throughput | Good when subtasks are independent |
| Traceability | Moderate to strong if worker outputs are normalised |
| Fault isolation | Better than single-agent, but supervisor remains a critical dependency |
| Scalability | Limited by supervisor decision rate and aggregation quality |

### Critical Failure Modes

| Failure | Description | Prevention |
|---|---|---|
| **Supervisor failure** | Bad decomposition, context saturation, or execution error collapses the entire task | Supervisor failure is a first-class failure mode — handle explicitly in runtime |
| **Micromanagement collapse** | Supervisor rewrites every worker output in depth → architecture collapses back to one overloaded agent with messaging overhead | Design supervisors to **govern**, not micromanage |
| **Orphaned worker tasks** | Supervisor fails after delegation; workers continue executing with no one to collect results | All worker tasks carry a parent task ID; orphan detection in `OrchestrationMonitor` |
| **Weak decomposition** | Bad partitioning sends low-quality work to competent workers | Decomposition MUST be validated against the work graph before delegation |

> ⚠️ **Supervisor failure must be handled explicitly, not left as an edge case.** When the supervisor fails, workers may return valid results that no one collects, or continue running on a task the system has already abandoned.

### Design Rules

| Rule ID | Rule |
|---|---|
| `SW-001` | Use Supervisor–Worker when central authority + decomposable work are both required. |
| `SW-002` | Supervisor failure MUST be treated as a first-class failure mode with an explicit recovery path in the runtime. |
| `SW-003` | Supervisors MUST govern (assign, monitor, aggregate) — not micromanage (rewrite worker outputs). |
| `SW-004` | All delegated worker tasks MUST carry a parent task correlation ID for orphan detection and trace reconstruction. |
| `SW-005` | Decomposition MUST be validated against the work graph before any worker is invoked. |

### Java + LangGraph Target

```java
// Supervisor node: LLM-driven routing to workers or END
public class SupervisorNode implements NodeAction<OrchestratorState> {
    @Override
    public OrchestratorState apply(OrchestratorState state) {
        // LLM call: decide next worker or DONE
        String next = llm.decide(state.goal(), state.completedSubtasks());
        return state.withNextAgent(next)
                    .withCorrelationId(UUID.randomUUID().toString()); // SW-004
    }
}

// Supervisor graph
StateGraph<OrchestratorState> graph = new StateGraph<>(OrchestratorState.class)
    .addNode("supervisor", new SupervisorNode())
    .addNode("researcher",  new ResearcherSubgraph())
    .addNode("coder",       new CoderSubgraph())
    .addNode("reporter",   new ReporterSubgraph())
    .addConditionalEdges("supervisor", state -> switch(state.nextAgent()) {
        case "researcher" -> "researcher";
        case "coder"      -> "coder";
        case "reporter"   -> "reporter";
        case "DONE"       -> END;
        default           -> "error_handler"; // SW-002
    })
    .addEdge("researcher", "supervisor")  // all workers return to supervisor
    .addEdge("coder",       "supervisor")
    .addEdge("reporter",    "supervisor");
```

---

## Section 16.4 — The Manager–Specialist Pattern

### Context

The Manager–Specialist pattern differs from Supervisor–Worker in one critical way: **decomposition is driven by expertise, not just task partitioning**. The manager routes work to specialists based on domain, risk profile, tool access, or mode of reasoning — not purely by subtask independence.

### When to Apply

- One task spans multiple disciplines that produce quality or safety interference when compressed into one prompt
- Different agents need different model configurations, toolsets, or quality thresholds
- Compliance or risk posture requires a specialist to certify their domain output

### Supervisor–Worker vs Manager–Specialist

| Aspect | Supervisor–Worker | Manager–Specialist |
|---|---|---|
| Coordination basis | Task partitioning | Domain expertise / role |
| Worker similarity | Often identical worker agents | Specialists with different tools, prompts, model configs |
| Typical failure | Over-centralisation | Wrong routing |
| Integration burden | Output normalisation | Output reconciliation (coherence + consistency) |

### The Routing and Integration Problem

The manager must repeatedly answer:
- Which specialist is responsible for this subtask?
- Should work go to one specialist or several (and in what order)?
- How should conflicting specialist outputs be reconciled?

> **Routing error is the defining failure mode.** A good specialist with the wrong assignment is still a system error. Specialisation optimises local quality — the manager must supply global coherence.

> 💡 **Specialisation should reflect real differences in capability, tooling, or risk posture.** If two specialists differ only cosmetically, they are probably one agent wearing two labels — and the routing overhead is pure waste.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MS-001` | Apply Manager–Specialist when domain separation materially improves quality or safety — not when specialisation is cosmetic. |
| `MS-002` | Routing logic MUST be deterministic and auditable — routing decisions must appear in the trace with the routing reason recorded. |
| `MS-003` | The manager MUST perform integration (coherence + consistency), not just routing. Outputs from different specialists MUST be reconciled before release. |
| `MS-004` | Each specialist MUST have an explicit capability declaration (`SpecialistProfile`) that the manager uses for routing. Routing from free-form LLM judgment alone is insufficient. |
| `MS-005` | Specialists that differ only in label and not in tooling, model config, or risk posture MUST be merged into a single agent. |

### Java + LangGraph Target

```java
public record SpecialistProfile(
    String agentId,
    List<String> domains,         // MS-004: explicit capability declaration
    List<String> registeredTools,
    String modelConfig,
    RiskProfile riskProfile
) {}

public class ManagerNode implements NodeAction<OrchestratorState> {
    private final Map<String, SpecialistProfile> registry; // MS-004

    @Override
    public OrchestratorState apply(OrchestratorState state) {
        String domain = classifier.classify(state.currentSubtask());
        String specialistId = registry.entrySet().stream()
            .filter(e -> e.getValue().domains().contains(domain))
            .map(Map.Entry::getKey)
            .findFirst()
            .orElseThrow(() -> new RoutingException("No specialist for: " + domain)); // MS-002
        return state.withNextAgent(specialistId)
                    .withRoutingReason(domain); // MS-002: audit trail
    }
}
```

---

## Section 16.5 — The Pipeline and Handoff Patterns

### Context

Not all multi-agent systems are about parallel collaboration. Some are about **staged transformation**.

- **Pipeline:** Agents arranged as ordered processing stages. Each stage receives a work artefact, transforms it, and passes it downstream.
- **Handoff:** Control moves from one agent to the next because the *nature of the task* has changed (e.g., intake → planner → domain specialist).

### When to Apply

| Pattern | Best For | Main Benefit | Main Risk |
|---|---|---|---|
| Pipeline | Repeatable staged workflows | Predictability, easy approval gate insertion | Upstream error propagation |
| Handoff | Changing task ownership as nature evolves | Clear responsibility transfer | Ambiguous completion semantics |

Both patterns have strong governance advantages: stage ownership is explicit, observability maps naturally onto workflow progress, and quality gates insert cleanly between stages.

### The Inter-Stage Contract

**Every handoff needs a contract.** Upstream stages produce artefacts; downstream stages consume them with well-defined expectations. Without explicit contracts:
- A weak classification step silently poisons all later stages
- A lossy summarisation step removes facts needed downstream
- A schema-valid but semantically wrong artefact propagates error through the entire chain

Every inter-stage contract MUST define:
1. What the upstream stage **promises** (artefact type, schema, completeness)
2. What the downstream stage is **entitled to assume**
3. What constitutes a **rejection** (conditions under which downstream refuses the artefact)
4. Who to notify and what to do on rejection

### Failure Modes

| Failure | Description | Prevention |
|---|---|---|
| **Silent upstream error** | Bad artefact looks valid; error propagates downstream | Schema validation + semantic checks at each stage boundary |
| **Ambiguous completion** | Handoff recipient does not know whether upstream work is complete or partial | Explicit `HandoffStatus` enum: `COMPLETE`, `PARTIAL`, `FAILED` |
| **Backtracking debt** | Task requires correction of an earlier stage after downstream work has begun | Apply Pipeline only to tasks with linear, non-backtracking workflows |

> ⚠️ **Do not use a pipeline when the task routinely requires backtracking, cross-stage negotiation, or dynamic branching.** In those cases, apparent simplicity hides a large implicit coordination debt.

### Design Rules

| Rule ID | Rule |
|---|---|
| `PH-001` | Every inter-stage handoff MUST be governed by an explicit contract specifying upstream promises, downstream assumptions, rejection conditions, and rejection routing. |
| `PH-002` | Each handoff artefact MUST carry a `HandoffStatus` (`COMPLETE` / `PARTIAL` / `FAILED`) and a correlation ID. |
| `PH-003` | Stage boundaries MUST include schema validation before the downstream stage begins processing. |
| `PH-004` | Pipeline MUST NOT be applied to tasks that routinely require backtracking or cross-stage negotiation. |
| `PH-005` | Rejection at any stage MUST route to a defined error handler — never drop silently. |

### Java + LangGraph Target

```java
public record HandoffArtifact<T>(
    String correlationId,   // PH-002
    HandoffStatus status,   // COMPLETE | PARTIAL | FAILED
    T payload,
    String producingAgentId,
    Instant producedAt,
    String contractVersion  // PH-001: which contract version governs this artefact
) {}

public enum HandoffStatus { COMPLETE, PARTIAL, FAILED }

// Pipeline graph
StateGraph<PipelineState> graph = new StateGraph<>(PipelineState.class)
    .addNode("intake",        new IntakeNode())
    .addNode("validate_s1",   new StageContractValidator("intake→planner"))  // PH-003
    .addNode("planner",       new PlannerNode())
    .addNode("validate_s2",   new StageContractValidator("planner→specialist")) // PH-003
    .addNode("specialist",    new SpecialistNode())
    .addNode("error_handler", new PipelineErrorHandler())  // PH-005
    .addEdge(START, "intake")
    .addEdge("intake", "validate_s1")
    .addConditionalEdges("validate_s1",
        state -> state.validationPassed() ? "planner" : "error_handler")
    .addEdge("planner", "validate_s2")
    .addConditionalEdges("validate_s2",
        state -> state.validationPassed() ? "specialist" : "error_handler");
```

---

## Section 16.6 — Shared State and the Blackboard Pattern

### Context

Some tasks do not decompose into isolated subtasks or ordered stages. Multiple agents need to contribute partial insights to a **common working artefact**. This is the Blackboard pattern: agents coordinate through a shared workspace rather than through direct messaging.

The **memory foundations** for this pattern are established in chap13 (shared memory, synchronisation, conflict resolution). This section focuses on the **coordination topology** — how agents behave around that shared state, not how the state is stored.

### How the Pattern Works

```
Shared Workspace (typed, versioned, append-only)
     ↑ post partial result    ↑ post enrichment    ↑ post remediation
[Extractor Agent]        [Correlator Agent]      [Fixer Agent]
     reads raw input           reads errors             reads enriched errors
```

No agent directly messages another. All coordination passes through the workspace. The shared state is the collaboration medium.

### Structural Requirements

Blackboard architectures MUST impose explicit structure on the workspace to remain operable:

| Requirement | Purpose |
|---|---|
| **Typed artefact schemas** | Prevent structurally incompatible contributions |
| **Append-only log or version history** | Preserve full contribution lineage |
| **Provenance metadata** | Trace every field value to its producing agent and timestamp |
| **Locking or compare-and-swap semantics** | Prevent conflicting concurrent writes to sensitive fields |
| **Designated consolidation / arbitration role** | Resolve conflicting contributions before artefact is released |

> ⚠️ **Without these constraints, agents coordinate through a medium that looks shared but lacks shared meaning.** Conflicting updates, stale reads, and unattributed values are all silent failure modes.

### When to Apply

- Task genuinely requires cumulative collaboration on common artefacts
- Agents contribute partial insights that become more complete through iteration
- Contribution order is not predetermined

When **not** to apply:
- Sequential work is sufficient (use Pipeline)
- Work is independently partitionable (use Supervisor–Worker)
- Coordination cost of shared state consistency exceeds the benefit of flexible contribution

### Design Rules

| Rule ID | Rule |
|---|---|
| `BB-001` | Blackboard workspaces MUST use typed artefact schemas — untyped shared maps are not acceptable in production. |
| `BB-002` | All writes to the workspace MUST append to a versioned log with provenance metadata (producing agent ID, timestamp, correlation ID). |
| `BB-003` | Fields that can be written by more than one agent MUST use compare-and-swap (CAS) or optimistic locking semantics. |
| `BB-004` | A consolidation / arbitration node MUST be designated to resolve conflicting contributions before the artefact leaves the workspace. |
| `BB-005` | Apply Blackboard only when cumulative partial-result collaboration is genuinely required. If sequential work suffices, use Pipeline. |

### Java + LangGraph Target

```java
public record WorkspaceEntry<T>(
    String fieldKey,
    T value,
    String producingAgentId,   // BB-002
    Instant producedAt,        // BB-002
    String correlationId,      // BB-002
    long version               // BB-003: optimistic locking
) {}

public interface BlackboardWorkspace {
    <T> boolean compareAndSet(String key, T expected, T update, String agentId); // BB-003
    <T> void append(String key, T value, String agentId, String correlationId); // BB-002
    List<WorkspaceEntry<?>> getHistory(String key); // full provenance log
}

// LangGraph Blackboard graph
StateGraph<WorkspaceState> graph = new StateGraph<>(WorkspaceState.class)
    .addNode("extractor",     new ExtractorNode())
    .addNode("correlator",    new CorrelatorNode())
    .addNode("fixer",         new FixerNode())
    .addNode("consolidator",  new ConsolidationNode())  // BB-004
    .addEdge(START, "extractor")
    .addEdge("extractor", "correlator")
    .addEdge("correlator", "fixer")
    .addEdge("fixer", "consolidator")  // always consolidate before release
    .addEdge("consolidator", END);
```

---

## Section 16.7 — Hierarchical Orchestration

### Context

When tasks become large enough, a flat orchestration model stops working. One top-level supervisor cannot manage dozens of workers without becoming a throughput bottleneck and a context saturation problem. **Hierarchical orchestration** addresses this by creating multiple layers of delegation.

### Structure

```
Top-Level Orchestrator
    assigns workstream A → Mid-Level Manager A
        assigns subtask A1 → Worker A1 → structured summary
        assigns subtask A2 → Worker A2 → structured summary
        synthesises → Domain Report A
    assigns workstream B → Mid-Level Manager B
        assigns subtask B1 → Worker B1 → structured summary
        ...
        synthesises → Domain Report B
    integrates Domain Reports A, B, C, D → Final Output
```

Each layer handles a **bounded problem** and returns a **structured artefact**. No single agent carries the full task context.

### When to Apply

- Task tree is too large for one supervisor to manage directly
- Different workstreams require distinct tools, roles, and quality criteria
- Multiple parallelism levels exist (workstream-level and subtask-level)

Examples: large research synthesis (40+ sources across 4 domains), enterprise software delivery (requirements → implementation → testing → deployment review).

### Hierarchy Depth Rule

> **Design rule: Limit hierarchy depth to 3 layers maximum**, or enforce that every delegation carries a trace correlation ID that preserves parent–child lineage across all layers. Deeper trees make causal attribution exponentially harder.

### The Observability Problem

Hierarchy creates a serious observability challenge: **failure is nested**. A bad output may originate in:
- A leaf worker (execution failure)
- A local manager's decomposition (routing/integration failure)
- A middle-layer aggregation step (contract failure)
- The top-level orchestrator's original framing (goal decomposition failure)

As the tree deepens, causal attribution becomes harder. Without explicit trace lineage, root-cause analysis degrades to guesswork.

### Policy Drift Risk

Different layers may interpret goals, risk thresholds, or completion criteria differently. Without strict contracts, the system slowly becomes a set of loosely related local habits rather than one governed architecture.

Hierarchical systems require stronger governance than flat systems:
- Delegation contracts explicit at every level
- Escalation conditions standardised across layers
- Trace propagation preserving parent–child lineage
- Termination authority clear at every level
- Summary compression carefully controlled to avoid information loss

### Design Rules

| Rule ID | Rule |
|---|---|
| `HIER-001` | Hierarchy depth MUST NOT exceed 3 layers unless every delegation carries a trace correlation ID that preserves parent–child lineage. |
| `HIER-002` | Each layer MUST receive and return structured artefacts — free-form delegation is not acceptable. |
| `HIER-003` | Escalation conditions MUST be standardised and documented across all hierarchy levels. |
| `HIER-004` | Summary compression between levels MUST preserve all facts required by parent layers — lossy compression at mid-layers is a silent failure mode. |
| `HIER-005` | The hierarchy is justified only when it reduces overload at one level without making the whole system opaque. Organisational sophistication is not a justification. |

### Java + LangGraph Target

```java
public record DelegationContract(
    String taskId,
    String parentTaskId,        // HIER-001: parent-child lineage
    String traceCorrelationId,  // HIER-001
    int hierarchyDepth,         // HIER-001: enforced ≤ 3
    String goal,
    String requiredOutputSchema,// HIER-002
    EscalationPolicy escalation // HIER-003
) {}

// Hierarchical orchestrator validates depth at delegation time
public class HierarchicalOrchestrator {
    static final int MAX_DEPTH = 3; // HIER-001

    public DelegationContract delegate(String goal, DelegationContract parent) {
        int depth = parent != null ? parent.hierarchyDepth() + 1 : 0;
        if (depth > MAX_DEPTH) throw new HierarchyDepthException(
            "Delegation depth " + depth + " exceeds MAX_DEPTH=" + MAX_DEPTH);
        return new DelegationContract(
            UUID.randomUUID().toString(),
            parent != null ? parent.taskId() : null,
            parent != null ? parent.traceCorrelationId() : UUID.randomUUID().toString(),
            depth, goal, outputSchema, escalationPolicy
        );
    }
}
```

---

## Section 16.8 — Coordination Contracts, Observability, and Failure Handling

### Context

The success of a multi-agent system depends less on the capability of individual agents than on **the quality of contracts between them**. Every orchestration pattern is a bundle of contracts: task assignment, output, escalation, retry, and termination.

### The Five Contract Questions

Every coordination contract MUST answer:

1. What exactly is being delegated?
2. What inputs are guaranteed?
3. What output format is required?
4. What should the receiving agent do when blocked or uncertain?
5. Who has authority to declare work complete, failed, or escalated?

### Production Observability Requirements

In a single-agent system, one trace often tells the full story. In a multi-agent system, **trace reconstruction is an architecture feature, not an afterthought**.

Minimum observability requirements for all multi-agent production systems:

| Requirement | Implementation |
|---|---|
| Correlation IDs across all delegated work | Every node write includes `correlationId` in state |
| Parent–child relationships between agent actions | `parentTaskId` in every delegation contract |
| Structured event logs for task lifecycle | `ASSIGNED`, `STARTED`, `COMPLETED`, `RETRIED`, `ESCALATED` events |
| Artefact provenance | Every output artefact records producing agent ID, timestamp, and contract version |
| Both local and orchestration-level failure visibility | Leaf-node failures surfaced to parent layer with full context |

### Failure Class Taxonomy

| Failure Class | Example | Remediation |
|---|---|---|
| **Execution failure** | Worker throws exception | Retry with exponential backoff; escalate after N failures |
| **Contract failure** | Artefact fails schema validation at stage boundary | Reject; request clarification or re-delegation |
| **Routing failure** | Manager assigns to wrong specialist | Fallback to default specialist or supervisor override |
| **Integration failure** | Specialist outputs contradict each other | Apply conflict resolution policy (recency, source authority, arbitration node) |
| **Governance failure** | Action exceeds authorised scope | Halt execution; require human approval |

> ⚠️ **Blind retries are often the worst possible response.** Each failure class implies a different corrective action. Multi-agent systems that retry uniformly across failure classes amplify the original failure.

> ⚠️ **Multi-agent systems can appear more resilient because another agent can be called when one fails. In reality, poor orchestration turns local failure into systemic churn** — the system keeps reassigning work without fixing the underlying coordination defect.

### Design Rules

| Rule ID | Rule |
|---|---|
| `CC-001` | Every coordination contract MUST answer the five contract questions before the system handles production traffic. |
| `CC-002` | All delegated work MUST carry correlation IDs and parent-task IDs enabling full trace reconstruction. |
| `CC-003` | Failure handling MUST be failure-class-aware — blind uniform retry across failure classes is a production defect. |
| `CC-004` | Multi-agent systems MUST emit structured lifecycle events (`ASSIGNED`, `STARTED`, `COMPLETED`, `RETRIED`, `ESCALATED`) for every agent action. |
| `CC-005` | Governance failures MUST halt execution and route to human approval — they MUST NOT be retried automatically. |

### Java + LangGraph Target

```java
public enum FailureClass {
    EXECUTION, CONTRACT, ROUTING, INTEGRATION, GOVERNANCE
}

public record AgentFailureEvent(
    String taskId,
    String correlationId,
    String agentId,
    FailureClass failureClass,  // CC-003
    String failureDetail,
    Instant occurredAt
) {}

public class FailureRouter {
    public String route(AgentFailureEvent event) {
        return switch (event.failureClass()) {
            case EXECUTION    -> "retry_with_backoff";
            case CONTRACT     -> "reject_and_clarify";
            case ROUTING      -> "fallback_specialist";
            case INTEGRATION  -> "arbitration_node";
            case GOVERNANCE   -> "human_approval";   // CC-005
        };
    }
}
```

---

## Section 16.9 — When Orchestration Is the Wrong Choice

### Context

Multi-agent orchestration is the wrong choice when a simpler architecture would produce a better system. This section provides the disqualification criteria.

### Disqualification Conditions

| Condition | Why Orchestration Fails |
|---|---|
| **Task is mostly sequential** | No parallelism benefit; coordination overhead is pure cost |
| **Specialisation is cosmetic** | Two agents with the same tools, model config, and risk profile are one agent with extra messaging |
| **Parallelism too small to offset coordination overhead** | Coordination cost > parallelism gain → multi-agent is net-negative on latency |
| **Shared state more expensive than central reasoning** | Blackboard consistency cost exceeds the benefit of flexible contribution |
| **Team cannot yet observe and debug one agent reliably** | Adding agents multiplies the observability problem before a baseline exists |

> **Good litmus test:** If you removed the extra agents and replaced them with structured internal phases inside one agent, would the system become simpler without losing the core benefit? If yes, the multi-agent design is premature.

> ⚠️ **Each agent call incurs its own inference cost.** Orchestration can multiply token usage dramatically. Always budget for the worst-case number of agent invocations, and consider cost per subtask before adding parallel workers.

### The Coordination Overhead Law

As soon as multiple agents need stable, inspectable, interoperable communication semantics, the architecture begins pressing toward **standard communication layers** (chap18–19: MCP and A2A protocols). Bespoke coordination stops scaling. This pressure is the architectural bridge from orchestration patterns to protocol-level design.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MA-WRONG-001` | Before choosing multi-agent orchestration, run the litmus test: would structured internal phases inside one agent produce the same benefit? If yes, stay single-agent. |
| `MA-WRONG-002` | Cost modelling for multi-agent systems MUST include worst-case total inference cost across all agents. Orchestration that doubles cost for marginal quality gain requires explicit business justification. |
| `MA-WRONG-003` | Teams MUST be able to produce a reliable trace for a single agent before building multi-agent systems. Observability capability is a prerequisite, not a follow-on. |

---

## Section 16.10 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `MA-JUSTIFY-001` | Justification | Justify by measured pressure, not felt complexity |
| `MA-JUSTIFY-002` | Justification | Every added agent must serve a concrete architectural purpose |
| `MA-JUSTIFY-003` | Justification | ADR must name the pressure and prior single-agent attempts |
| `MA-PAT-001` | Pattern selection | Pattern driven by work shape and control boundary |
| `MA-PAT-002` | Pattern selection | Draw work graph before agent graph |
| `MA-PAT-003` | Pattern selection | Document pattern in `OrchestratorConfig` |
| `SW-001` | Supervisor–Worker | Apply for central authority + decomposable work |
| `SW-002` | Supervisor–Worker | Supervisor failure is a first-class failure mode |
| `SW-003` | Supervisor–Worker | Supervisors govern, not micromanage |
| `SW-004` | Supervisor–Worker | All delegated tasks carry parent correlation ID |
| `SW-005` | Supervisor–Worker | Decomposition validated against work graph first |
| `MS-001` | Manager–Specialist | Apply only when specialisation is real, not cosmetic |
| `MS-002` | Manager–Specialist | Routing decisions are deterministic and auditable |
| `MS-003` | Manager–Specialist | Manager integrates, not just routes |
| `MS-004` | Manager–Specialist | Explicit `SpecialistProfile` required for routing |
| `MS-005` | Manager–Specialist | Cosmetically different specialists MUST be merged |
| `PH-001` | Pipeline / Handoff | Every handoff governed by explicit contract |
| `PH-002` | Pipeline / Handoff | Artefacts carry `HandoffStatus` and correlation ID |
| `PH-003` | Pipeline / Handoff | Schema validation at every stage boundary |
| `PH-004` | Pipeline / Handoff | Not applied to backtracking workflows |
| `PH-005` | Pipeline / Handoff | Stage rejection routes to defined error handler |
| `BB-001` | Blackboard | Typed artefact schemas required |
| `BB-002` | Blackboard | Writes append to versioned provenance log |
| `BB-003` | Blackboard | Multi-writer fields use CAS or optimistic locking |
| `BB-004` | Blackboard | Consolidation node required before artefact release |
| `BB-005` | Blackboard | Applied only when cumulative collaboration is genuinely required |
| `HIER-001` | Hierarchical | Max depth 3 layers or full trace lineage required |
| `HIER-002` | Hierarchical | Each layer receives and returns structured artefacts |
| `HIER-003` | Hierarchical | Escalation conditions standardised across layers |
| `HIER-004` | Hierarchical | Summary compression must not lose parent-required facts |
| `HIER-005` | Hierarchical | Hierarchy justified by overload reduction, not org sophistication |
| `CC-001` | Contracts | All five contract questions answered before production |
| `CC-002` | Contracts | Correlation IDs and parent-task IDs on all delegated work |
| `CC-003` | Contracts | Failure handling is failure-class-aware — no blind retry |
| `CC-004` | Contracts | Structured lifecycle events on every agent action |
| `CC-005` | Contracts | Governance failures halt and escalate — never auto-retry |
| `MA-WRONG-001` | Anti-pattern | Run litmus test before choosing multi-agent |
| `MA-WRONG-002` | Anti-pattern | Worst-case total inference cost modelled before deployment |
| `MA-WRONG-003` | Anti-pattern | Single-agent observability must be proven before multi-agent |

---

## Section 16.11 — Transition to Chapter 17

Chapter 16 established that multi-agent orchestration is a **control topology change** — valuable when specialisation, parallelism, isolation, or governance requirements make one execution context insufficient, and expensive when applied prematurely.

The next dimension of control is not between agents — it is between agents and humans. **Chapter 17: Human-in-the-Loop as a First-Class Pattern** treats human participation as a deliberate architectural decision:

| Chapter 17 Topic | Chapter 16 Dependency |
|---|---|
| Why human involvement is architectural, not bolt-on | `CC-005`: governance failures always escalate to human |
| Pre-action approval and gated autonomy | `SW-002`: supervisor failure handling; `HIER-003`: escalation policy |
| Post-generation review | Pattern-level trace requirements (`CC-002`, `CC-004`) |
| Exception-only escalation | `MS-002`: routing auditability; `CC-003`: failure-class-aware handling |
| Mixed-initiative collaboration | Blackboard shared state foundation (§16.6) |
| Human-on-the-loop: earning reduced intervention | Graduation from orchestration to protocol-level design (§16.9) |

> The coordination contracts and observability infrastructure defined in Chapter 16 are the prerequisites for making human participation legible and manageable in Chapter 17. Without them, human review becomes approval theater.
