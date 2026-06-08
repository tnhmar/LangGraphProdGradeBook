# Chapter 18 Spec — Emergent Behaviors in Decentralized Networks

> **Volume 1 reference:** `chemergent-behaviors-in-decentralized-networks`
> **Part:** 4 — Agent Architecture Patterns (fourth and final chapter)
> **Depends on:** chap15 (control envelope, single-agent autonomy), chap16 (orchestration contracts, blackboard pattern, `CC-005`), chap17 (HOTL, authority contracts, `HOTL-001–004`)
> **Java + LangGraph implementation target:** Volume 2
> **Bridges to:** Part 5 — Agent Communication Protocols (MCP, A2A, ANP)

---

## Chapter Purpose

Chapter 17 concluded the governed spectrum: from single agents (chap15), through centrally orchestrated multi-agent systems (chap16), to human participants as first-class architectural components (chap17). Chapter 18 moves in the opposite direction on the control axis.

The central claim:

> *Decentralization trades direct control for scale and adaptability. The price is that the system's behavior is designed, not commanded. Useful global behavior must emerge from carefully designed local rules — it cannot be specified directly.*

This chapter:
- Explains the architectural pressures that make decentralization attractive
- Provides a five-pattern taxonomy of decentralized coordination
- Specifies each pattern's mechanics, failure modes, and Java + LangGraph design rules
- Covers network-level failure modes that arise from interaction effects, not component failures
- Establishes why decentralized agent ecosystems create irreversible pressure for standard communication protocols — the bridge into Part 5

---

## Section 18.1 — Why Decentralized Patterns Exist

### Context

A centralized orchestrator is not always the right control point. As the number of agents grows, workload becomes more dynamic, or the environment becomes more distributed, central coordination can become a bottleneck. One planner may not have enough context. One supervisor may not respond fast enough. One authority boundary may be too rigid for the task.

### Four Pressures Toward Decentralization

| Pressure | Description |
|---|---|
| **Local autonomy** | Agents should not wait for central assignment when local state is sufficient to act correctly |
| **Scale** | Agent populations exceed what a single orchestrator can manage directly |
| **Resilience** | Single-point-of-failure risk from a central coordinator is architecturally unacceptable |
| **Local responsiveness** | Faster reaction to local events than a central coordinator can achieve |
| **Open ecosystem** | Agents are built by different teams, run on different infrastructure, governed by different policies |

### The Control Trade-off

In an orchestrated system the designer controls the whole topology. In a decentralized network the designer often controls only:
- Local interaction rules
- Communication semantics
- Participation requirements

> *Decentralized does not mean chaotic. It means control is distributed rather than concentrated. Good decentralized systems still have rules, boundaries, and participation constraints — they just do not all flow through one node.*

### The Three Substitution Questions

Before any decentralized pattern is chosen, these three questions MUST be answerable:

| Question | What it Tests |
|---|---|
| What substitutes for a central scheduler? | Discovery and allocation mechanism |
| What substitutes for a central memory authority? | State consistency and provenance |
| What substitutes for a central stop condition? | Termination semantics |

If those substitutes are not explicitly designed, the system will drift.

### Design Rules

| Rule ID | Rule |
|---|---|
| `DEC-ARCH-001` | Decentralized patterns MUST be adopted only when a concrete system-level benefit — scale, autonomy, resilience — cannot be achieved with a simpler orchestrated design. |
| `DEC-ARCH-002` | Before committing to any decentralized pattern, the three substitution questions (scheduler, memory authority, stop condition) MUST be explicitly answered and documented. |
| `DEC-ARCH-003` | Hybrid designs (local autonomy within bounded regions, stronger central control around identity, policy enforcement, and settlement) are preferred over ideological full decentralization. |

---

## Section 18.2 — Taxonomy of Decentralized Coordination Patterns

### Context

Decentralized agent networks are not all the same. They differ in how agents discover each other, how they decide who should do what, and how they converge on a shared useful outcome. In an orchestrated system those questions are answered by design-time architecture. In a decentralized system they are answered by local policy, protocol convention, incentive structures, or environmental cues.

### Pattern Overview

| Pattern | Core Idea | Main Strength | Main Cost | Common Failure |
|---|---|---|---|---|
| **Peer-to-Peer Delegation** | Agents discover and delegate directly to one another | Flexible collaboration | Harder governance | Unbounded delegation chains |
| **Stigmergic Coordination** | Agents coordinate through shared traces or artifacts | Indirect, scalable cooperation | State ambiguity | Conflicting or stale signals |
| **Market-Based Coordination** | Tasks and resources allocated through bids or scores | Efficient allocation under competition | Incentive tuning complexity | Strategic or wasteful behavior |
| **Swarm Local-Rule Systems** | Global behavior emerges from simple local rules | High scalability and adaptation | Weak interpretability | Instability or oscillation |
| **Federation of Autonomous Agents** | Independent agents interact under shared conventions | Ecosystem growth | Uneven trust and capability | Interoperability and trust failures |

### Pattern Selection Rule

```
Dynamic task routing across independently governed agents?  → Peer-to-Peer Delegation
Agents need to cooperate without direct messaging?          → Stigmergic Coordination
Heterogeneous capability + variable workload?               → Market-Based Coordination
Exploration, coverage, or broad parallel sampling needed?   → Swarm Local-Rule Systems
Agents from different orgs / runtimes must interoperate?    → Federation
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `DEC-PAT-001` | Pattern selection MUST be driven by the three substitution questions (18.1), not by preference for novelty or by framework defaults. |
| `DEC-PAT-002` | The chosen pattern MUST be documented with its failure mode profile and the monitoring strategy that will make those failures observable. |

---

## Section 18.3 — Peer-to-Peer Delegation Networks

### Context

In a peer-to-peer delegation network, agents communicate directly with one another without routing through a central authority. Any agent may discover another agent, request help, delegate a subtask, return results, or redirect the task onward.

The advantage is flexibility. Agents can route around local failures, find specialized collaborators dynamically, and adapt to changing task needs without waiting for a central scheduler. This makes the network more resilient and more open to ecosystem growth, particularly when agents are developed and operated by independent teams.

### Three Immediate Problems

| Problem | Description |
|---|---|
| **Authority ambiguity** | Who is allowed to delegate what to whom? |
| **Lineage ambiguity** | Who owns the final result? |
| **Termination ambiguity** | Who decides the task is done? |

### Local Delegation Contract

Every delegation in a P2P network MUST define:

1. What the receiving agent is authorized to do
2. What context must be provided (input contract)
3. What output guarantees are expected
4. Under what conditions the task may be further re-delegated
5. How responsibility is traced across handoffs

### Failure Mode: Delegation Recursion

One agent passes a task to a second, the second reframes it and passes part of it to a third, and the chain continues until no agent retains a clear view of the original objective. The system generates activity but not useful progress.

> ⚠️ **Peer-to-peer freedom without bounded delegation rules does not create agility. It creates coordination debt that compounds faster than most teams can observe.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `P2P-001` | Every P2P delegation MUST carry a correlation ID that traces the full delegation chain back to the originating task. |
| `P2P-002` | Delegation MUST be bounded by an explicit contract: authorized scope, input guarantees, output format, re-delegation permission, and responsibility tracing. |
| `P2P-003` | Maximum delegation chain depth MUST be enforced at runtime — not only in prompt instructions. Exceeding the depth limit MUST cause the task to terminate or escalate. |
| `P2P-004` | Trust in P2P networks MUST be encoded in discovery rules, identity verification, reputation policy, or protocol semantics — not assumed from capability claims alone. |
| `P2P-005` | Termination authority MUST be explicitly assigned per task — either to the originating agent, a designated coordinator, or by time-to-live expiry. |

### Java + LangGraph Target

```java
public record DelegationContract(
    String taskId,
    String correlationId,          // P2P-001: full chain trace
    String delegatingAgentId,
    String receivingAgentId,
    String authorizedScope,        // P2P-002: what the agent may do
    Map<String, Object> inputGuarantees,
    String requiredOutputFormat,
    boolean reDelegationPermitted, // P2P-002
    int maxChainDepth,             // P2P-003
    int currentChainDepth,
    Duration taskTtl,              // P2P-005: time-to-live termination
    TrustLevel trustLevel          // P2P-004
) {}

public record DelegationResult(
    String taskId,
    String correlationId,
    String producingAgentId,
    Object output,
    TaskStatus status,
    Instant completedAt
) {}

// P2P delegation node — enforces chain depth and contract
public class P2PDelegationNode implements NodeAction<AgentState> {
    private final AgentDiscoveryService discovery;

    @Override
    public AgentState apply(AgentState state) {
        DelegationContract contract = state.pendingDelegation();

        // P2P-003: enforce chain depth
        if (contract.currentChainDepth() >= contract.maxChainDepth()) {
            return state.withStatus(TaskStatus.CHAIN_DEPTH_EXCEEDED)
                        .withError("Max delegation depth reached; task terminated");
        }

        AgentHandle peer = discovery.findCapableAgent(
            contract.authorizedScope(),
            contract.trustLevel()  // P2P-004
        );

        DelegationContract childContract = contract.toBuilder()
            .currentChainDepth(contract.currentChainDepth() + 1)
            .delegatingAgentId(state.agentId())
            .receivingAgentId(peer.agentId())
            .build();

        DelegationResult result = peer.execute(childContract);
        return state.withDelegationResult(result);
    }
}
```

---

## Section 18.4 — Stigmergic Coordination and Shared Environmental Signals

### Context

Some decentralized systems coordinate without direct negotiation. Instead, agents leave traces in a shared environment, and other agents react to those traces. This is **stigmergic coordination** — indirect cooperation mediated by environmental modification rather than explicit messaging.

A trace can be many things: a task posted to a shared queue, a partially completed artifact, a status marker, a confidence score, a shared hypothesis, a vector of priorities, or an event appended to a common stream. No agent has to command the others. Coordination emerges because agents inspect the environment and act on what they find.

### Concrete Pattern Example

A distributed research pipeline where ten agents work against a shared task board:

```
Agent A: retrieves document → appends with status UNPROCESSED
Agent B: finds UNPROCESSED → extracts claims → marks EXTRACTED
Agent C: finds EXTRACTED → writes structured summary → marks SUMMARISED
Agent D: finds SUMMARISED (confidence > threshold) → adds to final synthesis
```

No agent directly messages another. Each reads the environment and writes back to it. Coordination is entirely mediated by shared structure.

> *Note:* Stigmergic systems are decentralized even when they use a common substrate such as a shared database or message queue. The shared object is not an orchestrator — it is an **environment carrying signals that influence local behavior**.

### The Temporal Problem and Required Environmental Structure

Shared traces decay in value. A stale task marker can trigger irrelevant work. The shared environment must carry explicit semantics to remain trustworthy:

| Required Structural Element | Purpose |
|---|---|
| Versioned artifacts with timestamps | Prevent stale signal consumption |
| Status semantics with defined transitions | Agents interpret signals consistently |
| Provenance metadata (producing agent) | Traceability and conflict attribution |
| Expiration rules for time-sensitive signals | Prevent stale anchoring |
| Conflict resolution policy for concurrent writes | Prevent duplication and contradiction |

### Design Rules

| Rule ID | Rule |
|---|---|
| `STIG-001` | Every artifact in the shared environment MUST carry provenance metadata: producing agent ID, timestamp, version, and status. |
| `STIG-002` | Status field transitions MUST be defined as an explicit finite state machine — agents MUST NOT write arbitrary status strings. |
| `STIG-003` | Artifact expiry rules MUST be defined and enforced — no agent may consume an expired artifact as if it were current. |
| `STIG-004` | Concurrent write conflicts MUST be resolved by a defined policy (last-writer-wins with version check, compare-and-swap, or append-only log) — never silently ignored. |
| `STIG-005` | Agents MUST be idempotent consumers of the shared environment — processing the same artifact twice MUST NOT produce different state. |

### Java + LangGraph Target

```java
public enum ArtifactStatus {
    UNPROCESSED, EXTRACTED, SUMMARISED, SYNTHESISED, EXPIRED
}

public record SharedArtifact(
    String artifactId,
    String correlationId,
    ArtifactStatus status,         // STIG-002
    String producingAgentId,       // STIG-001
    int version,                   // STIG-004: optimistic concurrency
    Instant createdAt,
    Instant expiresAt,             // STIG-003
    Object payload
) {}

// Stigmergic consumer node — reads and updates shared environment
public class StigmergicConsumerNode implements NodeAction<AgentState> {
    private final SharedEnvironmentStore store;

    @Override
    public AgentState apply(AgentState state) {
        List<SharedArtifact> candidates = store.findByStatus(
            state.consumableStatus(),
            Instant.now()  // STIG-003: filter expired artifacts
        );

        for (SharedArtifact artifact : candidates) {
            if (artifact.expiresAt().isBefore(Instant.now())) continue; // STIG-003

            Object processed = process(artifact);

            SharedArtifact updated = artifact.toBuilder()
                .status(state.outputStatus())
                .producingAgentId(state.agentId())   // STIG-001
                .version(artifact.version() + 1)
                .payload(processed)
                .build();

            // STIG-004: compare-and-swap on version
            boolean written = store.compareAndSwap(artifact.version(), updated);
            if (!written) {
                // Conflict: another agent wrote concurrently — skip or retry
                continue;
            }
        }
        return state.withProcessedCount(candidates.size());
    }
}
```

---

## Section 18.5 — Market-Based and Incentive-Driven Coordination

### Context

Some decentralized systems allocate work using **market-based coordination**. Tasks are announced, and agents bid, score themselves, or otherwise signal fitness to perform the work. Allocation emerges from those signals rather than from a manager's decision.

### Typical Market Mechanism

```
1. Task broker publishes task: capability requirement, complexity score, deadline
2. Agents submit bids: claimed capability match, estimated completion time, confidence score
3. Broker selects highest-scoring bid (multi-attribute scoring function)
4. Winning agent performs work and returns result
5. Settlement: completion recorded, agent performance history updated
```

This is useful when capability is heterogeneous, workload is variable, and the system benefits from decentralized resource allocation. Rather than asking one orchestrator to know which agent should do what at all times, the system lets agents self-select based on **local knowledge of their own capacity and fitness**.

### Incentive Misalignment Failure Modes

Incentives are never neutral. The moment the system rewards some behaviors more than others, agents will optimize toward the reward structure.

| Failure Mode | Description |
|---|---|
| **Over-claiming** | Agents bid on tasks beyond their actual capability to win the assignment |
| **Cherry-picking** | Easy tasks preferred; difficult or ambiguous tasks avoided |
| **Bidding overhead** | High bid volume on popular tasks wastes compute without improving allocation |
| **Priority inversion** | Most important work is not the most attractive under the scoring function |
| **Local optimization at global cost** | Each agent maximizes throughput while the overall task graph stalls on bottlenecks |

> *A scoring function is an architecture decision, not a configuration detail.* Ask what behavior it actually incentivizes — not what you hope it incentivizes. Model the adversarial case before deployment.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MKT-001` | The scoring function MUST be designed explicitly as an architecture artifact — documented, version-controlled, and reviewed for incentive alignment before deployment. |
| `MKT-002` | Bidding MUST be bounded: maximum bid volume per task and maximum concurrent bids per agent MUST be enforced to prevent bidding overhead. |
| `MKT-003` | Task priority MUST be encoded in the scoring function — high-priority tasks MUST be more attractive, not merely louder. |
| `MKT-004` | Agent performance history MUST be maintained and factored into bid scoring — over-claiming without delivery MUST reduce future bid weight. |
| `MKT-005` | Priority inversion and cherry-picking patterns MUST be monitored as operational metrics and trigger scoring function review when detected. |

### Java + LangGraph Target

```java
public record TaskAnnouncement(
    String taskId,
    String correlationId,
    String capabilityRequirement,
    TaskPriority priority,          // MKT-003
    int complexityScore,
    Instant deadline,
    int maxBids                     // MKT-002
) {}

public record AgentBid(
    String taskId,
    String biddingAgentId,
    double capabilityMatchScore,
    Duration estimatedCompletion,
    double confidenceScore,
    Instant submittedAt
) {}

public class BidScoringFunction {
    // MKT-001: explicit architecture artifact
    public double score(AgentBid bid, TaskAnnouncement task,
                        AgentPerformanceHistory history) {
        double base = bid.capabilityMatchScore() * 0.4
            + (1.0 / bid.estimatedCompletion().toSeconds()) * 0.3
            + bid.confidenceScore() * 0.2
            + task.priority().weight() * 0.1; // MKT-003
        double reputationFactor = history.deliveryRate(); // MKT-004
        return base * reputationFactor;
    }
}

// Market broker node
public class MarketBrokerNode implements NodeAction<AgentState> {
    private final BidScoringFunction scorer;
    private final AgentPerformanceStore performanceStore;

    @Override
    public AgentState apply(AgentState state) {
        List<AgentBid> bids = state.receivedBids();

        // MKT-002: enforce max bid count
        if (bids.size() > state.currentTask().maxBids()) {
            bids = bids.stream()
                .sorted(Comparator.comparing(AgentBid::submittedAt))
                .limit(state.currentTask().maxBids())
                .toList();
        }

        AgentBid winner = bids.stream()
            .max(Comparator.comparingDouble(bid ->
                scorer.score(bid, state.currentTask(),
                    performanceStore.get(bid.biddingAgentId()))))
            .orElseThrow(() -> new NoEligibleBidderException(state.currentTask().taskId()));

        return state.withAssignedAgent(winner.biddingAgentId());
    }
}
```

---

## Section 18.6 — Swarm Systems and Local Rules

### Context

The most extreme decentralized pattern is the **swarm**. In a swarm system, agents follow relatively simple local rules. No agent sees the whole task. No agent centrally plans the collective outcome. Yet under the right conditions, useful global behavior emerges from the interaction of many local decisions.

### When Swarms Are Valuable

Swarm-style systems are powerful for tasks that reward **statistical coverage and adaptability** over per-transaction correctness:

- Broad hypothesis exploration across a large search space
- Distributed document corpus coverage
- Parallel sampling of candidate solutions in optimization
- Adversarial probing — many independent agents stress different system components simultaneously

**Concrete example:** Fifty agents, each following one rule — take the least-explored hypothesis from a shared pool, generate two supporting and two refuting arguments, score confidence, return results, draw a new hypothesis. No agent sees the full frontier. Yet the collective result is a comprehensive, balanced exploration that no single planner could organize as efficiently.

### Why Swarms Are Hard to Govern

Three engineering consequences flow directly from emergent dynamics:

| Consequence | Implication |
|---|---|
| **Emergent outcomes are hard to predict before runtime** | Testing requires running the system — static reasoning is insufficient |
| **Interpretability is weaker than orchestrated systems** | No single agent is responsible for the global outcome |
| **Corrective action is indirect** | You change rules rather than directly assigning tasks |

### Swarm-Specific Failure Modes

| Failure | Description |
|---|---|
| **Oscillation** | Agents repeatedly undo or counter each other's contributions; no convergence |
| **Over-concentration** | Network focuses on a subset of the problem space, missing broad coverage |
| **Premature convergence** | Local signals reinforce each other too strongly before sufficient exploration |
| **Fragmentation** | Network splits into disconnected subpopulations that optimize incompatible objectives |
| **Wrong objective** | Large coordinated activity optimizing the wrong metric entirely |

> ⚠️ **Do not confuse distributed activity with collective intelligence. A swarm can produce a large amount of coordinated motion while optimizing the wrong objective entirely.**

### When NOT to Use Swarms

Swarms are the wrong pattern when:
- Clear accountability is required for each individual action
- The workflow is naturally sequential
- Per-transaction explainability is required by regulators or users
- High-accountability enterprise workflows require audit trails to individual agents

### Design Rules

| Rule ID | Rule |
|---|---|
| `SWARM-001` | Swarm local rules MUST be formally specified before deployment and version-controlled as architecture artifacts. |
| `SWARM-002` | Convergence criteria MUST be defined explicitly — the system MUST have a stop condition that does not depend on a central coordinator. |
| `SWARM-003` | Oscillation and over-concentration MUST be detected via network-level metrics and trigger rule adjustment, not agent-level retries. |
| `SWARM-004` | Swarm systems MUST NOT be used for workflows requiring per-transaction accountability, audit trails to individual agents, or deterministic per-action control. |
| `SWARM-005` | Swarm rule sets MUST be tested via simulation before production deployment — static reasoning about emergent behavior is unreliable. |

### Java + LangGraph Target

```java
public record SwarmRule(
    String ruleId,
    String version,                         // SWARM-001: version-controlled
    Predicate<SharedArtifact> trigger,      // when to apply
    Function<SharedArtifact, Object> action,// what to produce
    Predicate<AgentState> convergenceCheck  // SWARM-002: stop condition
) {}

public record SwarmConvergencePolicy(
    int maxCycles,                          // SWARM-002: hard cycle limit
    double coverageThreshold,              // stop when coverage reached
    double oscillationDetectionWindow,     // SWARM-003: detect thrash
    int maxOscillationCount                // SWARM-003: trigger rule review
) {}

// Swarm agent node
public class SwarmAgentNode implements NodeAction<AgentState> {
    private final List<SwarmRule> rules;
    private final SwarmConvergencePolicy convergencePolicy;
    private final SwarmMetrics metrics;

    @Override
    public AgentState apply(AgentState state) {
        // SWARM-002: check convergence before acting
        if (isConverged(state, convergencePolicy)) {
            return state.withStatus(TaskStatus.CONVERGED);
        }

        // SWARM-003: detect oscillation
        if (metrics.oscillationCount(state.agentId()) 
                > convergencePolicy.maxOscillationCount()) {
            metrics.flagForRuleReview(state.agentId()); // rule change, not retry
            return state.withStatus(TaskStatus.OSCILLATION_DETECTED);
        }

        // Apply first matching rule
        for (SwarmRule rule : rules) {
            if (rule.trigger().test(state.currentArtifact())) {
                Object output = rule.action().apply(state.currentArtifact());
                return state.withOutput(output);
            }
        }
        return state.withStatus(TaskStatus.NO_RULE_MATCHED);
    }
}
```

---

## Section 18.7 — Federation of Autonomous Agents

### Context

The federation pattern addresses a reality that most other decentralized patterns assume away: that all agents in the network are designed, deployed, and governed by the same authority. In many real-world settings, agents are built by different teams, run on different infrastructure, and operate under different policies.

A **federation** is a decentralized network of autonomous agents that agree on a **minimal set of shared conventions** — capability discovery, message formats, trust verification, and error semantics — while retaining full control over their internal behavior.

### Federation vs Other Decentralized Patterns

| Dimension | Federation | P2P Delegation | Swarm |
|---|---|---|---|
| Agent ownership | Multiple independent orgs | Usually single org | Usually single org |
| Shared authority | None — only shared conventions | Partial | None |
| Trust model | Explicit per-agent trust verification | Assumed within org | Assumed within system |
| Protocol dependency | HIGH — conventions are the architecture | Low | Low |
| Interoperability scope | Cross-org, cross-framework | Same framework preferred | Same framework preferred |

### Minimal Shared Convention Set

Federated agents MUST agree on at minimum:

| Convention | Purpose |
|---|---|
| **Capability advertisement format** | Agents can discover what each other can do |
| **Task delegation message format** | Agents can request work from each other |
| **Result delivery format** | Agents can receive and interpret outputs |
| **Trust verification mechanism** | Agents can verify identity and authorization |
| **Error semantics** | Agents can interpret failure signals consistently |

### Trust in Federated Networks

In orchestrated systems, the orchestrator decides which agents are trustworthy. In federated systems, trust must be encoded at the protocol level:

- Discovery rules (which agents are visible to which)
- Identity verification (cryptographic or registry-based)
- Reputation systems (historical performance)
- Policy constraints (what a federated agent may be asked to do)
- Protocol semantics (what capabilities must be proven, not claimed)

> *Federation is the pattern that makes the agent web possible.* It is also the most demanding pattern operationally — shared conventions must be minimal, well-tested, versioned, and backward-compatible or the ecosystem fragments.

### Design Rules

| Rule ID | Rule |
|---|---|
| `FED-001` | Federated agents MUST implement a standard capability advertisement protocol — capability claims MUST be verifiable, not self-asserted prose. |
| `FED-002` | Every federated interaction MUST verify agent identity before task delegation — identity claims without cryptographic or registry verification MUST be rejected. |
| `FED-003` | Shared message format conventions MUST be versioned and backward-compatible — format changes MUST NOT silently break existing federated agents. |
| `FED-004` | Each federated agent MUST retain the right to reject work from untrusted or underperforming peers — rejection is a first-class protocol response, not a failure. |
| `FED-005` | Federated protocol conventions MUST be documented as architecture artifacts and reviewed when upgraded — they are the federation's language. |

### Java + LangGraph Target

```java
public record FederatedCapabilityCard(
    String agentId,
    String agentVersion,
    List<String> capabilities,              // FED-001: verifiable
    String publicKeyFingerprint,            // FED-002: identity verification
    String protocolVersion,                 // FED-003: format versioning
    Map<String, String> trustAnchors,       // FED-002
    Instant issuedAt,
    Instant expiresAt
) {}

public record FederatedDelegationRequest(
    String requestId,
    String correlationId,                   // P2P-001 inherited
    String requestingAgentId,
    String targetCapability,
    String protocolVersion,                 // FED-003
    Map<String, Object> taskPayload,
    String delegatingAgentSignature         // FED-002: cryptographic proof
) {}

// Federated capability discovery node
public class FederatedDiscoveryNode implements NodeAction<AgentState> {
    private final FederatedRegistryClient registry;
    private final TrustVerifier trustVerifier;

    @Override
    public AgentState apply(AgentState state) {
        List<FederatedCapabilityCard> candidates =
            registry.findByCapability(state.requiredCapability());

        // FED-002: verify identity before delegation
        List<FederatedCapabilityCard> trusted = candidates.stream()
            .filter(card -> trustVerifier.verify(card))
            .filter(card -> card.expiresAt().isAfter(Instant.now()))
            .toList();

        if (trusted.isEmpty()) {
            return state.withStatus(TaskStatus.NO_TRUSTED_AGENT_FOUND);
        }

        return state.withSelectedFederatedAgent(
            selectBestMatch(trusted, state.requiredCapability())
        );
    }
}
```

---

## Section 18.8 — Emergence, Network-Level Failure Modes, and Control Limits

### Context

Decentralized systems fail differently from orchestrated systems. In an orchestrated system, a bad outcome can often be traced to a bad routing decision, a bad plan, or a failed worker. In a decentralized network, the failure may not live in any one component. It may live in the **interaction pattern itself**.

That makes the failure surface broader, less intuitive, and harder to attribute.

### Network-Level Failure Taxonomy

| Failure Mode | Description | Detection Signal |
|---|---|---|
| **Runaway propagation** | Local action triggers cascade of unnecessary activity | Activity volume per task growing without bound |
| **Duplication** | Many agents independently do nearly identical work | Output similarity ratio high; token cost per unique output rising |
| **Oscillation** | Agents repeatedly undo or counter each other's contributions | Artifact version churn without convergence |
| **Coordination dead zones** | No agent takes responsibility because local signals too weak | Tasks stuck in UNPROCESSED state beyond TTL |
| **Premature convergence** | Network settles too quickly on a suboptimal path | Coverage metrics plateau before threshold reached |
| **Trust contamination** | Unreliable signals spread through the network and degrade otherwise competent agents | Quality metrics degrading across agents that share a common input source |

> *No single agent may be wrong in isolation. The system-level outcome is wrong because the local rules interact badly.*

### Required Network-Level Observability Views

Per-agent traces are not sufficient in decentralized systems. These **network-level views** are required:

| View | What it Reveals |
|---|---|
| Activity distribution across the network | Over- or under-utilized agents |
| Propagation paths for specific artifacts | Whether work flows correctly |
| Artifact lineage and provenance chains | Attribution and conflict tracing |
| Concentration or bottleneck patterns | Throughput constraints |
| Consensus and disagreement dynamics | Convergence quality |
| Stability of global outcomes over time | Rule set effectiveness |

### Control Shift

In decentralized systems, corrective action is fundamentally different from orchestrated systems:

```
Orchestrated: fix a routing decision, fix a plan, replace a worker
Decentralized: change participation rules, communication semantics,
               trust thresholds, incentive functions, or environmental structure
```

You do not fix the system by correcting one task. You fix it by **changing the rules**.

### Design Rules

| Rule ID | Rule |
|---|---|
| `NET-001` | Decentralized systems MUST instrument network-level observability views — per-agent traces alone are insufficient. |
| `NET-002` | Runaway propagation MUST be bounded by activity limits per task — agents MUST NOT be able to generate unbounded cascades. |
| `NET-003` | Duplication MUST be detected via output similarity monitoring and trigger participation rule review, not agent-level retries. |
| `NET-004` | Trust contamination pathways MUST be traceable via artifact provenance — the propagation path of an unreliable signal MUST be reconstructable. |
| `NET-005` | Network-level failure modes MUST trigger **rule changes**, not component restarts. Direct corrective action at the component level does not address interaction-pattern failures. |

### Java + LangGraph Target (Network Monitor)

```java
public record NetworkHealthMetrics(
    Map<String, Long> activityByAgent,       // NET-001: distribution
    Map<String, Integer> artifactVersionChurn, // oscillation detection
    Map<String, Double> outputSimilarityRatio, // duplication detection
    Map<String, Long> stuckArtifactCount,    // coordination dead zones
    double globalCoverageRatio,              // convergence quality
    Instant sampledAt
) {}

public class NetworkHealthMonitor {
    private final SharedEnvironmentStore store;
    private final MetricsPublisher metricsPublisher;

    public NetworkHealthMetrics sample() {
        NetworkHealthMetrics metrics = buildMetrics(store.snapshot());
        metricsPublisher.publish(metrics);     // NET-001

        // NET-002: detect runaway propagation
        metrics.activityByAgent().forEach((agentId, count) -> {
            if (count > MAX_ACTIVITY_PER_AGENT_PER_WINDOW) {
                ruleEngine.flagForParticipationReview(agentId); // NET-005
            }
        });

        // Oscillation detection
        metrics.artifactVersionChurn().forEach((artifactId, churn) -> {
            if (churn > OSCILLATION_THRESHOLD) {
                ruleEngine.flagOscillation(artifactId);          // NET-005
            }
        });

        return metrics;
    }
}
```

---

## Section 18.9 — When Decentralization Is the Right Pattern

### Context

Decentralization is a justified design choice when the problem genuinely demands it. It is the wrong choice when orchestration or human control would be simpler, more accountable, or more reliable.

### Use Decentralization When

- The agent population is large enough that central routing is a measurable throughput bottleneck
- Agents are independently owned or administered by different teams or organizations
- Local responsiveness matters more than global consistency
- The work benefits from broad exploration, adaptation, or self-organization
- The system must remain functional under partial partition, local failure, or changing participation

### Do NOT Use Decentralization When

- Clear accountability is required for each individual action
- The workflow is naturally sequential
- State consistency is critical and difficult to maintain across participants
- Human approval must sit directly and traceably in the decision path (chap17)
- The team cannot yet observe and debug a simpler orchestrated system reliably

### The Hybrid Principle

> *In practice, many real systems should not be fully decentralized. They should be partially decentralized — local autonomy within a bounded region of the system, combined with stronger control around identity, policy enforcement, trust, or settlement.*

The correct engineering posture: **decentralize only the parts of the system that truly benefit from local autonomy. Keep everything else explicit.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `USE-001` | Decentralization MUST be chosen for specific system-level benefits — scale, autonomy, resilience — with each benefit documented before adoption. |
| `USE-002` | Any part of the system requiring clear per-action accountability MUST remain orchestrated or HITL-governed — decentralized patterns MUST NOT be applied to that scope. |
| `USE-003` | Hybrid designs (partial decentralization) are the default recommendation — full decentralization requires explicit justification per subsystem. |
| `USE-004` | A system team MUST be able to observe and debug its current architecture reliably before adding decentralized components. Observability debt blocks safe decentralization. |

---

## Section 18.10 — The Protocol Imperative: Bridge to Part 5

### Context

Part 4 began with a single agent. In that world, communication is mostly internal — between the agent's reasoning loop, its tools, and its memory. By Chapter 16 (multi-agent orchestration), multiple agents required explicit coordination contracts. By Chapter 17, humans became explicit participants with structured approval packets and escalation paths. In Chapter 18, decentralized networks have pushed the coordination problem further still.

Agents that do not share a runtime, a governance structure, or even an organizational owner must nonetheless:
- Discover one another
- Express capabilities
- Delegate work
- Exchange artifacts
- Signal trust
- Terminate interactions reliably

At that point, **informal coordination stops scaling**. Ad hoc message formats, framework-specific conventions, and implicit assumptions about tool access or task semantics become architectural liabilities.

### Why Protocols Are a Correctness Concern, Not Just an Optimization

An agent that cannot reliably express what it can do, or that cannot interpret another agent's capabilities and limitations in a standard way, will **miscoordinate in ways that are difficult to diagnose** — precisely because the failure is at the interface, not inside either component.

When ten agents from three different frameworks interact in a decentralized network, privately negotiated conventions are no longer sufficient.

### The Architectural Progression

| Chapter | Control Scope | Communication Model |
|---|---|---|
| chap15 | Single agent | Internal — reasoning, tools, memory |
| chap16 | Orchestrated multi-agent | Explicit contracts between cooperating agents |
| chap17 | Human-in-the-loop | Structured approval packets, escalation paths |
| chap18 | Decentralized network | Local rules, shared signals, incentives |
| **Part 5** | **Open ecosystem** | **Standard protocol layer — MCP, A2A, ANP** |

> *The move from orchestrated to decentralized agent systems is a move from designing control flows to designing interaction rules. Both require engineering discipline — but the tools, the failure modes, and the governance mechanisms are fundamentally different.*

### Design Rules

| Rule ID | Rule |
|---|---|
| `PROTO-001` | Any decentralized system that must operate across independently governed agent runtimes MUST adopt a standard communication protocol rather than a privately negotiated convention. |
| `PROTO-002` | Protocol version MUST be explicitly negotiated at connection time — silent format assumptions across independently deployed agents are a correctness risk, not merely an incompatibility risk. |
| `PROTO-003` | Federation patterns (Section 18.7) MUST be implemented using the standard protocols defined in Part 5 (MCP for tool access; A2A for agent-to-agent delegation; ANP for open network participation). |

---

## Section 18.11 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `DEC-ARCH-001` | Architecture | Decentralized only when concrete system-level benefit justified |
| `DEC-ARCH-002` | Architecture | Three substitution questions answered before pattern chosen |
| `DEC-ARCH-003` | Architecture | Hybrid designs preferred over full decentralization |
| `DEC-PAT-001` | Pattern selection | Driven by three substitution questions, not framework defaults |
| `DEC-PAT-002` | Pattern selection | Pattern documented with failure profile and monitoring strategy |
| `P2P-001` | P2P delegation | Correlation ID traces full delegation chain |
| `P2P-002` | P2P delegation | Delegation bounded by explicit five-field contract |
| `P2P-003` | P2P delegation | Max chain depth enforced at runtime |
| `P2P-004` | P2P delegation | Trust encoded in discovery rules, not capability claims |
| `P2P-005` | P2P delegation | Termination authority explicitly assigned per task |
| `STIG-001` | Stigmergic | Artifact carries full provenance metadata |
| `STIG-002` | Stigmergic | Status transitions defined as explicit FSM |
| `STIG-003` | Stigmergic | Expiry rules defined and enforced |
| `STIG-004` | Stigmergic | Concurrent write conflicts resolved by defined policy |
| `STIG-005` | Stigmergic | Agents are idempotent consumers of shared environment |
| `MKT-001` | Market-based | Scoring function is version-controlled architecture artifact |
| `MKT-002` | Market-based | Bidding bounded by max volume per task and per agent |
| `MKT-003` | Market-based | Priority encoded in scoring function |
| `MKT-004` | Market-based | Performance history factors into bid scoring |
| `MKT-005` | Market-based | Priority inversion and cherry-picking monitored operationally |
| `SWARM-001` | Swarm | Local rules formally specified and version-controlled |
| `SWARM-002` | Swarm | Convergence criteria and stop condition defined explicitly |
| `SWARM-003` | Swarm | Oscillation and over-concentration detected at network level |
| `SWARM-004` | Swarm | Not used where per-transaction accountability required |
| `SWARM-005` | Swarm | Rule sets tested via simulation before production |
| `FED-001` | Federation | Capability advertisement protocol standard and verifiable |
| `FED-002` | Federation | Identity verified before delegation — no unverified claims |
| `FED-003` | Federation | Message format versioned and backward-compatible |
| `FED-004` | Federation | Agents retain right to reject untrusted peers |
| `FED-005` | Federation | Protocol conventions are architecture artifacts |
| `NET-001` | Network health | Network-level observability views required |
| `NET-002` | Network health | Runaway propagation bounded by activity limits |
| `NET-003` | Network health | Duplication triggers participation rule review |
| `NET-004` | Network health | Trust contamination pathways traceable via provenance |
| `NET-005` | Network health | Network failures fixed by rule changes, not component restarts |
| `USE-001` | Adoption | Decentralization chosen for documented specific benefits |
| `USE-002` | Adoption | Per-action accountability scope remains orchestrated or HITL |
| `USE-003` | Adoption | Hybrid designs are the default recommendation |
| `USE-004` | Adoption | Observability debt blocks safe decentralization |
| `PROTO-001` | Protocol | Cross-runtime decentralized systems use standard protocols |
| `PROTO-002` | Protocol | Protocol version explicitly negotiated at connection time |
| `PROTO-003` | Protocol | Federation implemented via MCP, A2A, ANP (Part 5) |

---

## Section 18.12 — Transition to Part 5

Chapter 18 completes Part 4. The architectural progression across Part 4 moved from the single execution unit (chap15), through deliberate orchestration (chap16), to human governance (chap17), and finally to distributed, emergent coordination (chap18).

Each step increased the scope of what the system must coordinate and decreased the degree of direct central control. The final step — decentralized open networks — surfaces an unavoidable architectural requirement: **standard communication protocols**.

**Part 5 topics and their Chapter 18 dependencies:**

| Part 5 Chapter | Title | Chapter 18 Dependency |
|---|---|---|
| chap19 | The Protocol Imperative | `PROTO-001–003`; the full DEC-ARCH argument |
| chap20 | MCP (Model Context Protocol) | Federation (`FED-001–005`); tool access contracts |
| chap21 | A2A (Agent-to-Agent) | P2P delegation (`P2P-001–005`); federated delegation |
| chap22 | ANP (Agent Network Protocol) | Open ecosystem federation; trust verification at scale |

> Part 4 established why agents must coordinate. Part 5 establishes the language in which they do it.
