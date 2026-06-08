# Chapter 17 Spec — Human-in-the-Loop as a First-Class Pattern

> **Volume 1 reference:** `chhuman-in-the-loop-as-a-first-class-pattern`  
> **Part:** 4 — Agent Architecture Patterns (third chapter)  
> **Depends on:** chap15 (control envelope, approval gates), chap16 (coordination contracts, governance failure class, `CC-005`)  
> **Java + LangGraph implementation target:** Volume 2  
> **Cross-reference:** Volume 2 Chapter 20 (Enterprise Governance, GDPR, Compliance); Volume 2 Chapter 12 (Distributed Tracing, OTel)

---

## Chapter Purpose

Chapter 16 established that governance failures must halt execution and escalate to human approval — never auto-retry (`CC-005`). Chapter 17 treats that escalation destination as a **first-class architectural domain** in its own right.

The central claim:

> *Human involvement is not a limitation of autonomy. It is the mechanism that makes autonomy governable. Inserting a human into the wrong place, with the wrong context, at the wrong time, is as dangerous as leaving the human out entirely.*

This chapter:
- Explains why human participation must be **designed**, not bolted on
- Provides a five-pattern taxonomy of human-in-the-loop (HITL) control topologies
- Specifies each pattern's architecture, failure modes, and Java + LangGraph design rules
- Defines the human control surface as an engineering artefact
- Covers HITL failure modes and the path from human-in-the-loop to human-on-the-loop

> **Compliance note:** In regulated industries (finance, healthcare, legal, government), human-in-the-loop is not a design choice — it is a compliance requirement. The architecture must express these requirements in **enforceable runtime terms**, not in prompt instructions alone.

---

## Section 17.1 — Why Human Participation Is an Architectural Choice

### Context

Most early agent projects treat human review as a fallback — added at the last moment, wired as a manual override, justified as a compliance checkbox. That framing produces brittle, unmeasurable, and politically fragile systems.

The correct frame: **human involvement is a deliberate allocation of decision authority**. The question is not whether to include a human. The question is where human judgment adds value that runtime policies, tests, retrieval, or model improvements cannot add as cheaply or consistently.

### Human Roles in Agent Systems

A human can occupy several distinct architectural roles. Conflating them into one generic fallback makes governance unoptimisable:

| Role | Function | Design Implication |
|---|---|---|
| **Approver** | Authorises a risky action before it occurs | Pre-action gate with rich context packet |
| **Reviewer** | Evaluates output after generation but before release | Post-generation review surface with structured actions |
| **Corrector** | Edits, redirects, or repairs in-flight work | State management for correction propagation |
| **Escalation endpoint** | Resolves ambiguity the system cannot safely resolve alone | Escalation trigger logic and routing |
| **Operational trainer** | Improves prompts, routing, policies, tools based on failure patterns | Feedback capture and loop-back mechanism |

### The Autonomy Spectrum

Human participation is not binary. The design space runs from:

```
Fully manual → Conditional approval → Post-hoc audit → Sampled review → Near-complete autonomy with rare escalation
```

The right point depends on: risk, reversibility, cost of error, and operator capacity.

> ⚠️ **If you do not design human authority explicitly, you do not have an autonomous system. You have unmanaged exception handling** — typically discovered when a high-stakes failure surfaces the invisible manual cleanup load.

### Design Rules

| Rule ID | Rule |
|---|---|
| `HITL-ARCH-001` | Human involvement MUST be designed as an explicit allocation of decision authority — role, context, available actions, and feedback path all specified before the first production deployment. |
| `HITL-ARCH-002` | The question "where does human judgment add value that runtime policy cannot add as cheaply?" MUST be answered and documented before any HITL pattern is chosen. |
| `HITL-ARCH-003` | In regulated domains, HITL constraints MUST be enforced in runtime code — not solely expressed in prompt instructions. Compliance requirements are not prompt engineering problems. |

---

## Section 17.2 — Taxonomy of Human-in-the-Loop Patterns

### Context

Most production systems with meaningful human involvement converge on five reusable patterns. Pattern selection is driven by a single governing question: **what is the cost of being wrong, and at which point in execution does that cost materialise?**

### Pattern Overview

| Pattern | Best Fit | Main Strength | Main Cost | Common Failure |
|---|---|---|---|---|
| **Pre-Action Approval** | Risky or irreversible actions | Strong control before side effects | Latency and queue management | Approval fatigue; low-context rubber-stamping |
| **Post-Generation Review** | Output quality and policy checks | Better release quality | Slower throughput | Rubber-stamp review; no feedback capture |
| **Exception-Only Escalation** | High-volume, mostly safe workflows | Low human load; scale-friendly | Missed escalation conditions | Silent bad autonomy; under-escalation |
| **Mixed-Initiative Collaboration** | Co-authorship tasks (research, coding, drafts) | High quality + adaptability | Complex state management | Ownership confusion; interaction thrash |
| **Sampled Audit** | High-volume systems needing quality oversight | Scalable monitoring | Delayed error detection | False confidence from low sample coverage |

### Pattern Selection Rule

```
Action irreversible or high-risk?           → Pre-Action Approval
Output quality matters more than pre-action safety? → Post-Generation Review
Most work routine, rare cases dangerous?    → Exception-Only Escalation
Task requires iterative human + agent authorship? → Mixed-Initiative
High volume + oversight needed but cannot block all paths? → Sampled Audit
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `HITL-PAT-001` | Pattern MUST be selected by failure cost and authority requirement, not by convenience or default. |
| `HITL-PAT-002` | The selected HITL pattern MUST be documented in `AgentPolicy` alongside the action class it governs. |
| `HITL-PAT-003` | HITL patterns compose — a system may use pre-action approval for destructive writes while using post-generation review for report release and exception-only escalation for everything else. |

### Java + LangGraph Target (Policy Config)

```java
public enum HitlPattern {
    PRE_ACTION_APPROVAL,
    POST_GENERATION_REVIEW,
    EXCEPTION_ESCALATION,
    MIXED_INITIATIVE,
    SAMPLED_AUDIT
}

public record HitlPolicy(
    HitlPattern pattern,
    String actionClass,         // which action class this governs
    Duration approvalTimeout,   // how long to wait before escalating stale approval
    int maxQueueDepth,          // approval fatigue prevention
    boolean captureReviewData   // must be true in production
) {}
```

---

## Section 17.3 — Pre-Action Approval and Gated Autonomy

### Context

Pre-action approval is the most recognisable HITL pattern. The agent prepares a proposed action or plan, then **halts and waits** for human authorisation before executing any side effect.

Common trigger classes: external communication, privileged writes, financial transactions, production changes, regulated decisions, destructive operations.

### The Approval Packet Standard

The difference between a strong and a weak approval packet is the difference between real governance and approval theater. A production-grade approval packet MUST contain:

| Field | Purpose |
|---|---|
| **Proposed action** | What exactly will happen |
| **Reason** | Why the agent determined this action is correct |
| **Evidence / state** | What the agent observed that led to the proposal |
| **Expected consequence** | What will change if approved |
| **Risk classification** | Severity and reversibility |
| **Rollback path** | How to undo or mitigate if approved action proves wrong |

> ❌ **Weak packet:** "Ready to send the document. Approve?" — forces the reviewer to reconstruct context from scratch under time pressure. Low-context approval is a **design failure**, not a reviewer failure.

> ✅ **Strong packet:** "Proposed: send contract amendment to counterparty. Reason: all conditions in clause 4.2 met per signed amendment received at 14:32. Consequence: triggers 30-day review period. Risk: medium. Rollback: contact counterparty to withdraw within 24 hours."

### Graded Autonomy Model

Pre-action approval is most effective when combined with a policy-class-based autonomy gradient:

| Action Class | Governance |
|---|---|
| Read operations | Fully autonomous |
| Low-risk writes | Lightweight policy check (automated) |
| Medium-risk writes | Human single-approver |
| High-risk / regulated actions | Two-person authorisation or blocked |

### Approval Fatigue

> ⚠️ **An approval gate is not a safety feature if the reviewer lacks time, context, or authority.** A weak approval process can be *worse* than no approval process — it produces false institutional confidence while providing no real governance.

Approval fatigue occurs when too many actions are routed to humans and reviewers stop reading carefully. The gate exists formally but has ceased functioning substantively.

**Prevention:** Queue depth limits, action-class scoping, and alert on declining approval turnaround time.

### Design Rules

| Rule ID | Rule |
|---|---|
| `PAA-001` | Every approval packet MUST include all six standard fields before routing to a human reviewer. |
| `PAA-002` | Approval routing MUST be based on action class, not on ad-hoc LLM judgment. The same action class always routes to the same approval tier (`CE-004` from chap15). |
| `PAA-003` | Approval queues MUST have a defined max depth; exceeding it triggers escalation rather than silent accumulation. |
| `PAA-004` | Approval timeout MUST be configured — unanswered approvals MUST escalate to a secondary reviewer or halt the task, never silently expire. |
| `PAA-005` | All pre-interrupt side effects (DB writes, API calls) in the approving node MUST be idempotent, because LangGraph resumes by re-executing the interrupting node from the beginning. |

### Java + LangGraph Target

LangGraph's `interrupt()` primitive is the canonical implementation mechanism for pre-action approval:

```java
// Approval gate node — uses LangGraph interrupt
public class PreActionApprovalNode implements NodeAction<AgentState> {
    @Override
    public AgentState apply(AgentState state) {
        ApprovalPacket packet = ApprovalPacket.builder()
            .proposedAction(state.proposedAction())
            .reason(state.actionReason())
            .evidence(state.relevantEvidence())
            .expectedConsequence(state.expectedConsequence())
            .riskClass(state.riskClassification())
            .rollbackPath(state.rollbackPlan())
            .build(); // PAA-001: all six fields required

        // Checkpoint state to durable store; execution halts here
        Map<String, Object> decision = interrupt(Map.of(
            "type", "approval",
            "packet", packet
        ));

        boolean approved = (boolean) decision.get("approved");
        if (!approved) {
            return state.withStatus(TaskStatus.REJECTED)
                        .withRejectionReason((String) decision.get("reason"));
        }
        return state.withStatus(TaskStatus.APPROVED);
    }
}

// Graded autonomy router before the approval gate
public class ActionClassRouter {
    public String route(AgentState state) {
        return switch (state.proposedAction().riskClass()) { // PAA-002
            case READ         -> "execute";          // fully autonomous
            case LOW_RISK     -> "policy_check";     // automated check
            case MEDIUM_RISK  -> "approval_gate";   // single approver
            case HIGH_RISK    -> "two_person_auth";  // or BLOCKED
        };
    }
}
```

---

## Section 17.4 — Post-Generation Review and Release Control

### Context

Many agent tasks produce **artefacts** rather than direct side effects — reports, summaries, code suggestions, policy drafts, customer-facing responses. For these, the risk is not the *act* of generating, but the risk of *releasing* something wrong. Post-generation review places the human gate **after generation but before release**.

### When to Apply

- Output is rich enough that human judgment adds real value
- The main risk is incorrect content reaching external systems or users
- Human can inspect output faster than they could produce it from scratch
- A clear release workflow with structured reviewer actions exists

### Structured Reviewer Actions

Binary approve/reject is insufficient. Reviewers offered only two options are forced into a binary resolution of nuanced quality judgments. **Production review surfaces MUST offer at minimum:**

| Action | Effect |
|---|---|
| `APPROVE` | Release artefact as-is |
| `RETURN_WITH_COMMENTS` | Return to agent with specific improvement instructions |
| `EDIT_DIRECTLY` | Human edits and takes ownership |
| `REGENERATE_WITH_CONSTRAINTS` | Agent regenerates with stated constraints added |
| `ESCALATE` | Route to specialist or policy reviewer |

### Feedback Capture Requirement

Review decisions are **architecture signals**, not one-time actions. Captured data must reveal:
- Whether failures are in facts, formatting, tone, policy compliance, or scope
- Which parts of the system need architectural fixes vs more reviewer effort
- Whether rejection rates are stable, improving, or degrading

> Without feedback capture, humans improve individual outputs but the system never learns from the review pattern. The organisation pays for oversight without compounding any gains.

### Failure Modes

| Failure | Description | Prevention |
|---|---|---|
| **Rubber-stamp review** | Overloaded reviewers approve by habit | Queue depth limits; alert on review time collapse |
| **Review without actionability** | Reviewer sees flaw but has no structured path to return task | Structured action surface (all five actions) |
| **Feedback void** | Review data not captured or surfaced | `ReviewDecisionEvent` stored and surfaced in observability stack |

### Design Rules

| Rule ID | Rule |
|---|---|
| `PGR-001` | Review surfaces MUST offer all five structured actions — approve, return with comments, direct edit, regenerate with constraints, escalate. |
| `PGR-002` | Every review decision MUST be captured as a structured `ReviewDecisionEvent` (reviewer ID, decision type, reason, timestamp, edit delta). |
| `PGR-003` | Rejection reason MUST be structured (failure category: facts / format / tone / policy / scope), not free-form text only. |
| `PGR-004` | Review metrics (rejection rate by failure category, review turnaround, edit distance) MUST be surfaced in the system's observability stack. |
| `PGR-005` | Apply post-generation review only when human inspection time < cost of releasing a wrong output. When review becomes cheaper than rework, it is still worth it. |

### Java + LangGraph Target

```java
public enum ReviewAction {
    APPROVE, RETURN_WITH_COMMENTS, EDIT_DIRECTLY,
    REGENERATE_WITH_CONSTRAINTS, ESCALATE
}

public enum FailureCategory {
    FACTS, FORMAT, TONE, POLICY, SCOPE
}

public record ReviewDecisionEvent(
    String taskId,
    String correlationId,
    String reviewerId,
    ReviewAction action,
    FailureCategory failureCategory,  // PGR-003
    String comments,
    double editDistanceFraction,       // PGR-004
    Instant decidedAt
) {}

// Post-generation review node
public class PostGenerationReviewNode implements NodeAction<AgentState> {
    @Override
    public AgentState apply(AgentState state) {
        // interrupt: present artefact to reviewer with structured actions
        Map<String, Object> decision = interrupt(Map.of(
            "type",     "review",
            "artifact", state.generatedArtifact(),
            "actions",  ReviewAction.values()  // PGR-001
        ));

        ReviewDecisionEvent event = ReviewDecisionEvent.builder()
            .taskId(state.taskId())
            .reviewerId((String) decision.get("reviewerId"))
            .action(ReviewAction.valueOf((String) decision.get("action")))
            .failureCategory(parseCategory(decision.get("failureCategory")))
            .comments((String) decision.get("comments"))
            .decidedAt(Instant.now())
            .build();

        eventStore.publish(event); // PGR-002: capture for observability

        return state.withReviewDecision(event);
    }
}
```

---

## Section 17.5 — Exception-Only Escalation

### Context

Full review of every action does not scale. For most high-volume production systems, the right HITL topology is **exception-only escalation**: the agent operates autonomously for routine cases and routes only specific classes of uncertainty or risk to a human.

### When to Apply

- Most tasks are routine and rarely ambiguous
- High-volume operation makes per-decision review operationally untenable
- Risk is real but concentrated in a defined minority of cases

### Escalation Trigger Classes

Escalation logic MUST be explicit and narrow. Common triggers:

| Trigger | Description |
|---|---|
| **Confidence below threshold** | Agent's confidence score for next action falls below `escalationThreshold` |
| **Conflicting evidence** | Retrieval returns contradictory signals with no resolution heuristic |
| **Missing authority** | Required permission not present in agent's action policy |
| **Policy boundary condition** | Action lies at the edge of a defined policy rule (not clearly in or out) |
| **Repeated failure** | Task has failed `maxRetries` times without resolution |
| **User contestation** | User explicitly disputes an agent output |

### Escalation Packet Standard

An escalation request is not "I am unsure." It must include:
1. Task and current state
2. Conflicting options and what was attempted
3. The exact reason the agent could not safely proceed
4. The consequence of each available choice
5. Recommended path (if confidence allows)

### The Two Opposing Failures

| Failure | Description | Consequence |
|---|---|---|
| **Under-escalation** | Agent proceeds through genuine ambiguity | Real-world damage; governance failure |
| **Over-escalation** | Human becomes hidden primary operator | Throughput destroyed; no benefit over manual operation |

Both failures indicate a **weak autonomy boundary** — the line between routine and exceptional is not correctly calibrated.

> ⚠️ **Escalation conditions MUST be enforced through runtime policy, confidence thresholds, retry limits, and structured failure categories — not defined only in prompt language.** Prompt-only escalation is inconsistent and unpredictable.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ESC-001` | Escalation triggers MUST be defined as runtime policy rules (confidence threshold, failure class, retry count) — not only in prompt instructions. |
| `ESC-002` | Every escalation MUST produce a structured escalation packet containing all five required fields. |
| `ESC-003` | Escalation volume and latency MUST be monitored. Over-escalation (human becomes primary operator) is a system defect, not a safety feature. |
| `ESC-004` | Escalation routes MUST be failure-class-aware — the trigger class determines which human role receives the escalation. |
| `ESC-005` | Autonomy boundaries MUST be periodically reviewed against production escalation data and tightened or relaxed based on observed failure patterns. |

### Java + LangGraph Target

```java
public record EscalationPolicy(
    double confidenceThreshold,    // ESC-001: below this → escalate
    int maxRetries,                // ESC-001: exceeded → escalate
    Set<FailureClass> escalatingClasses, // ESC-004
    Duration escalationTimeout,
    String defaultEscalationQueue
) {}

public record EscalationPacket(
    String taskId,
    String correlationId,
    AgentState currentState,
    List<String> conflictingOptions,
    String attemptedResolution,
    String exactBlockingReason,     // ESC-002
    String recommendedPath,
    EscalationPolicy policy
) {}

// Exception-only escalation node
public class EscalationGateNode implements NodeAction<AgentState> {
    private final EscalationPolicy policy;

    @Override
    public AgentState apply(AgentState state) {
        if (!shouldEscalate(state, policy)) {
            return state; // pass through — autonomous execution continues
        }
        EscalationPacket packet = buildPacket(state);
        Map<String, Object> decision = interrupt(Map.of(
            "type", "escalation",
            "packet", packet
        ));
        return state.withEscalationDecision(decision);
    }

    private boolean shouldEscalate(AgentState state, EscalationPolicy policy) {
        return state.confidence() < policy.confidenceThreshold()
            || state.retryCount() >= policy.maxRetries()
            || policy.escalatingClasses().contains(state.lastFailureClass()); // ESC-001
    }
}
```

---

## Section 17.6 — Mixed-Initiative Collaboration

### Context

Some tasks are not "agent works, human checks." Instead, the human and agent **alternate initiative** — each can propose, revise, redirect, or deepen the work. This is the mixed-initiative pattern.

Common domains: research assistance, coding copilots, planning systems, design work, document drafting, exploratory analysis.

### Three Required Conditions

Mixed-initiative works well only when all three hold:

1. **State visibility:** The user always knows the current state of the work
2. **Interruptibility:** The user can interrupt, redirect, or refine without fighting the system
3. **Cumulative context:** The system remembers accepted/rejected suggestions and active assumptions — collaboration is progressive, not repetitive

### The Conversational Contract

Mixed-initiative systems require **crisp initiative contracts** to prevent ownership confusion and interaction thrash:

| Contract Question | Required Design Decision |
|---|---|
| When does the agent *propose* vs *act*? | Proposals labelled explicitly; user retains final decision |
| What may the agent do automatically? | Limited to pre-authorised action classes only |
| What is an instruction vs discussion? | System distinguishes preference expression from command issuance |
| How are revisions tracked? | User can see what changed and who changed it |
| Which outputs require explicit human authorship? | Public statements, binding commitments, legal text always need confirmation |

### Failure Modes

| Failure | Description | Prevention |
|---|---|---|
| **Ownership confusion** | User cannot tell whether agent is proposing, deciding, or acting | Explicit proposal labels; action-class pre-authorisation |
| **Interaction thrash** | Agent keeps taking initiative not asked for; user corrects continuously | Narrow pre-authorised action scope; interaction history review |
| **Authority ambiguity** | Record unclear on who decided what | Immutable revision log with agent/human attribution |
| **Context reset** | New session loses previous preferences and rejections | Persistent interaction state across sessions (chap13 episodic memory) |

### Design Rules

| Rule ID | Rule |
|---|---|
| `MI-001` | The system MUST maintain persistent interaction state: accepted suggestions, rejected options, and active user preferences (using episodic memory, chap13). |
| `MI-002` | The agent MUST NOT act autonomously on classes of actions not explicitly pre-authorised by the user for this session or system policy. |
| `MI-003` | Every agent proposal MUST be labelled as a proposal — never presented as an executed action until confirmed. |
| `MI-004` | An immutable revision log MUST attribute every change to either the agent or the human, with timestamp. |
| `MI-005` | Output categories defined as requiring human authorship (public statements, legal text, binding commitments) MUST require explicit human confirmation before release — agent cannot self-approve. |

### Java + LangGraph Target

```java
public record InteractionState(
    List<Suggestion> acceptedSuggestions,  // MI-001
    List<Suggestion> rejectedSuggestions,  // MI-001
    Map<String, String> activePreferences, // MI-001
    Set<String> preAuthorisedActionClasses // MI-002
) {}

public record Suggestion(
    String suggestionId,
    String content,
    SuggestionStatus status,   // PROPOSED | ACCEPTED | REJECTED | EXECUTED
    String authorType,         // AGENT | HUMAN
    Instant timestamp
) {}

// Mixed-initiative proposal node
public class MixedInitiativeNode implements NodeAction<AgentState> {
    @Override
    public AgentState apply(AgentState state) {
        Suggestion proposal = Suggestion.builder()
            .content(state.generatedProposal())
            .status(SuggestionStatus.PROPOSED) // MI-003
            .authorType("AGENT")
            .timestamp(Instant.now())
            .build();

        Map<String, Object> decision = interrupt(Map.of(
            "type",     "proposal",
            "proposal", proposal,
            "context",  state.interactionState() // MI-001: show full context
        ));

        return state.withProposalDecision(decision)
                    .withRevisionLog(buildRevisionEntry(proposal, decision)); // MI-004
    }
}
```

---

## Section 17.7 — Designing the Human Control Surface

### Context

A HITL pattern is only as good as the **control surface** through which the human experiences it. The control surface is the engineering artefact that makes human participation legible, efficient, and auditable. A poor control surface creates the bureaucratic appearance of governance while producing none of the safety.

### Required Control Surface Elements

| Element | Purpose |
|---|---|
| **Clear task framing** | Reviewer immediately understands what they are deciding |
| **Concise but sufficient context** | Decision-ready representation, not raw model chatter |
| **Risk classification** | Reviewer knows the stakes before reading |
| **The exact decision requested** | One unambiguous ask per review packet |
| **Structured action set** | Not just approve/reject — full structured options |
| **Evidence trail** | What the agent observed and attempted |
| **Consequence of each choice** | Reviewer understands downstream impact |
| **Persistent audit trail** | Immutable record of all human decisions |

> The reviewer sees a **decision-ready representation** — not raw trace logs. Full traces must be available for audit, but default review surfaces compress information for effective use under time pressure.

### Control Surface as Architecture Metric

Reviewer effort is not a UX metric — it is an **architecture signal**:

| Metric | What It Reveals |
|---|---|
| Time-to-decision | Whether context load is appropriate |
| Reopen rate | Whether approvals are sticking or being reconsidered |
| Edit depth | Whether agent output quality is improving |
| Escalation rate | Whether autonomy boundary is correctly calibrated |
| Disagreement rate by category | Which failure classes recur in agent output |

### Design Rules

| Rule ID | Rule |
|---|---|
| `CS-001` | Every human review surface MUST present all required control surface elements — no element may be omitted in production. |
| `CS-002` | Default review surfaces MUST present a decision-ready summary, not raw execution traces. Full traces must be accessible but not the default view. |
| `CS-003` | Context switching between systems to complete one review is a **design failure** in the control surface, not a reviewer problem. |
| `CS-004` | Control surface metrics (time-to-decision, reopen rate, edit depth, escalation rate, disagreement rate) MUST be instrumented and surfaced in the observability stack. |
| `CS-005` | Disagreement must be **actionable**: every rejection MUST route to a defined next state (revise, escalate, cancel) — review decisions MUST NOT disappear into logs. |

---

## Section 17.8 — Failure Modes of Human-in-the-Loop Systems

### Context

Inserting a human does not make a system safer by default. It shifts the failure surface. HITL systems have distinctive failure modes that are invisible until the system is under production load.

### HITL-Specific Failure Taxonomy

| Failure Mode | Description | Detection Signal |
|---|---|---|
| **Approval theater** | Human signs off without meaningful review — queue overloaded, context poor, decision socially precommitted | Approval turnaround < meaningful review threshold; rejection rate near zero |
| **Hidden labor dependency** | Product described as autonomous; humans continuously repair outputs behind the scenes | Manual effort in cost model is zero; actual staff time on corrections is non-zero |
| **Escalation collapse** | Agent over-escalates; queue grows; turnaround time rises; operators bypass control path | Escalation volume per task type trending upward; SLA misses increasing |
| **Authority ambiguity** | Human believes agent already decided; agent assumes human will correct; no one owns the decision | Post-incident review reveals no clear decision owner |
| **Feedback without learning** | Reviewers reject and escalate; system never improves | Rejection rate for same category stable over 60+ days |

> ⚠️ **The most dangerous HITL system is the one that looks controlled to leadership while silently depending on exhausted reviewers to keep it safe.** Hidden labor dependency is invisible to the cost model and catastrophic when staff turnover occurs.

### HITL Operational Metrics

HITL systems MUST be monitored with the same rigor as agent runtime metrics:

- Approval rate by risk class
- Rejection and revision rates
- Escalation volume and latency
- Reviewer turnaround time
- Reviewer disagreement rates
- Edit distance between agent output and final released artefact
- Frequency of repeated corrections for the same failure category

### Design Rules

| Rule ID | Rule |
|---|---|
| `HF-001` | HITL operational metrics MUST be instrumented from day one — not added after failure surfaces. |
| `HF-002` | Hidden labor dependency MUST be explicitly checked for during architecture reviews: does the system's cost model account for all human effort including informal correction and cleanup? |
| `HF-003` | Escalation collapse MUST trigger an autonomy boundary review — not a reviewer headcount increase. |
| `HF-004` | Authority ambiguity MUST be resolved in the runtime contract, not in social convention or team norms. |
| `HF-005` | Repeated corrections in the same failure category (>30 days without improvement) MUST trigger a system-level architectural review, not just continued review cycles. |

---

## Section 17.9 — From Human-in-the-Loop to Human-on-the-Loop

### Context

As systems mature, the goal is to **earn reduced intervention** without losing governance. This is the human-on-the-loop (HOTL) state: the system runs autonomously in most cases while humans supervise through monitoring, policy management, audit, and sampled intervention.

### The Three Prerequisites for HOTL

| Prerequisite | Test |
|---|---|
| **Stable task class** | Routine cases are genuinely routine — rare exceptions are statistically rare, not just hoped to be |
| **Mature policy enforcement** | Anomalies are caught without direct human inspection — runtime policies, circuit breakers, and thresholds all operational |
| **Proven intervention value** | Historical review data shows that direct intervention is rarely changing outcomes relative to policy-enforced defaults |

> ⚠️ **Teams often try to make this transition too early.** They see review volume as a cost centre and remove humans from the path before the architecture has earned that trust. The correct sequence: *learn from intervention → codify lessons into runtime policy → reduce intervention gradually*.

### Human Decisions as System Improvement Input

In a well-designed HITL system, every human intervention is a training signal:

```
Repeated approvals     →  codify into autonomous policy (remove gate)
Repeated rejections    →  improve agent output quality (reduce rework)
Repeated escalations   →  tighten confidence thresholds (prevent bad autonomy)
Repeated edits         →  improve generation quality (reduce review load)
```

This is how a system becomes **more autonomous without becoming less governable** — autonomy is earned incrementally through demonstrated reliability.

### Design Rules

| Rule ID | Rule |
|---|---|
| `HOTL-001` | Transition to HOTL MUST satisfy all three prerequisites before any direct HITL gate is removed. |
| `HOTL-002` | Every human intervention captured in review data MUST feed into a system improvement backlog — rejections and edits are architecture signals, not just operational events. |
| `HOTL-003` | HOTL supervision MUST include sampling: a defined percentage of autonomous decisions are reviewed retrospectively to detect policy drift. |
| `HOTL-004` | Any degradation in autonomous decision quality (measured via sampled audit) MUST automatically re-insert the corresponding HITL gate until root cause is resolved. |

### Java + LangGraph Target (HOTL Sampler)

```java
public record HotlSamplingPolicy(
    double samplingRate,          // e.g. 0.05 = 5% sampled audit
    Duration reviewLookback,      // how far back sampled reviews go
    double qualityDegradationThreshold, // HOTL-004: triggers gate re-insertion
    String alertChannel
) {}

// HOTL sampled audit node — fires on sampling policy
public class SampledAuditNode implements NodeAction<AgentState> {
    private final HotlSamplingPolicy policy;

    @Override
    public AgentState apply(AgentState state) {
        if (!sampler.shouldSample(policy.samplingRate())) {
            return state; // not sampled — pass through
        }
        // Present to auditor without blocking live execution path
        auditQueue.enqueue(SampledAuditRecord.from(state, Instant.now()));
        return state; // HOTL-003: non-blocking sample
    }
}
```

---

## Section 17.10 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `HITL-ARCH-001` | Architecture | Human authority designed explicitly — role, context, actions, feedback all specified |
| `HITL-ARCH-002` | Architecture | Value-add question answered before pattern chosen |
| `HITL-ARCH-003` | Architecture | Regulated domain constraints enforced in runtime code |
| `HITL-PAT-001` | Pattern selection | Pattern chosen by failure cost and authority requirement |
| `HITL-PAT-002` | Pattern selection | Pattern documented in `AgentPolicy` |
| `HITL-PAT-003` | Pattern selection | Patterns compose — multiple patterns active simultaneously |
| `PAA-001` | Pre-action approval | All six packet fields required |
| `PAA-002` | Pre-action approval | Routing based on action class, not LLM judgment |
| `PAA-003` | Pre-action approval | Approval queue has defined max depth |
| `PAA-004` | Pre-action approval | Approval timeout triggers escalation, never silent expiry |
| `PAA-005` | Pre-action approval | Pre-interrupt side effects are idempotent |
| `PGR-001` | Post-generation review | Five structured reviewer actions required |
| `PGR-002` | Post-generation review | Every decision captured as structured event |
| `PGR-003` | Post-generation review | Rejection reason is structured (failure category) |
| `PGR-004` | Post-generation review | Review metrics surfaced in observability stack |
| `PGR-005` | Post-generation review | Applied when human inspection < cost of wrong release |
| `ESC-001` | Exception escalation | Triggers defined as runtime policy rules |
| `ESC-002` | Exception escalation | Structured escalation packet with all five fields |
| `ESC-003` | Exception escalation | Escalation volume monitored — over-escalation is a defect |
| `ESC-004` | Exception escalation | Failure-class-aware escalation routing |
| `ESC-005` | Exception escalation | Autonomy boundaries reviewed against production data |
| `MI-001` | Mixed-initiative | Persistent interaction state across sessions |
| `MI-002` | Mixed-initiative | Agent autonomous only on pre-authorised action classes |
| `MI-003` | Mixed-initiative | Proposals labelled explicitly as proposals |
| `MI-004` | Mixed-initiative | Immutable revision log with attribution |
| `MI-005` | Mixed-initiative | Human-authorship output categories always require confirmation |
| `CS-001` | Control surface | All required elements present |
| `CS-002` | Control surface | Decision-ready summary as default view |
| `CS-003` | Control surface | Cross-system context switching is a design failure |
| `CS-004` | Control surface | Control surface metrics instrumented in observability stack |
| `CS-005` | Control surface | Every rejection routes to defined next state |
| `HF-001` | Failure modes | HITL metrics instrumented from day one |
| `HF-002` | Failure modes | Hidden labor dependency checked in architecture reviews |
| `HF-003` | Failure modes | Escalation collapse triggers autonomy boundary review |
| `HF-004` | Failure modes | Authority ambiguity resolved in runtime contract |
| `HF-005` | Failure modes | Recurring failure category triggers system architectural review |
| `HOTL-001` | Human-on-loop | All three prerequisites met before HITL gate removed |
| `HOTL-002` | Human-on-loop | Interventions feed system improvement backlog |
| `HOTL-003` | Human-on-loop | Sampled audit defined and active |
| `HOTL-004` | Human-on-loop | Quality degradation auto-reinstates HITL gate |

---

## Section 17.11 — Transition to Chapter 18

Chapter 17 completed the governed spectrum: from fully autonomous single agents (chap15), through deliberately orchestrated multi-agent systems (chap16), to human participation as a first-class control mechanism.

**Chapter 18** moves in the opposite direction on the control axis — away from any central governing authority and toward systems where **control is distributed across many agents whose collective behaviour is not commanded but designed through local rules and interaction patterns**.

| Chapter 18 Topic | Chapter 17 Dependency |
|---|---|
| Why decentralisation arises (autonomy beyond single orchestrator) | HOTL state — the limit of reduced central control within governed systems |
| Peer-to-peer delegation networks | Authority contracts from `HITL-ARCH-001`; lineage rules from `MI-004` |
| Stigmergic coordination (shared environment signals) | Shared workspace provenance from chap16 `BB-002`; interaction state from `MI-001` |
| Market-based and incentive-driven coordination | Escalation volume monitoring (`ESC-003`) as analogue for incentive tuning |
| Swarm local-rule systems | Control surface observability (`CS-004`) as analogue for network-level monitoring |
| Protocol imperative (bridge to Part 5) | HITL contracts under external agents — motivation for MCP and A2A |

> Human-in-the-loop is the last point in the architecture where a single governing authority makes decision rights explicit. Chapter 18 examines what happens when that authority is intentionally dissolved.
