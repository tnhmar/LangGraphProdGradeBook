# Chapter 21 Spec — Layer 2: Agent Collaboration — A2A and ANP

> **Volume 1 reference:** `chlayer-2-agent-collaboration-a2a-and-anp`
> **Part:** 5 — Agent Communication Protocols
> **Depends on:** chap19 (`LAYER-001–003`, `SEC-001–005`), chap20 (`MCP-ARCH-003`), chap18 (`DEC-ARCH-002`, `FED-001–005`)
> **Java + LangGraph implementation target:** Volume 2 (A2A Client/Server, Agent Card registry, task lifecycle state machine)
> **Bridges to:** chap22 (Layer 3, Agent Web, ANS, Agent Gateway), chap33 (Enterprise integration)

---

## Chapter Purpose

Chapter 20 covered Layer 1 — the interface between an agent and its tools. That layer is **asymmetric by design**: the agent drives, the tool responds, the relationship is well-bounded. Layer 2 is fundamentally different.

When two agents communicate, **both parties reason, plan, and act**. Either may initiate. Either may delegate. Either may reject, renegotiate, or fail independently. The coordination challenges are of a different order of magnitude.

This chapter covers the protocols that address Layer 2 agent collaboration as of 2026:
- **A2A (Agent2Agent Protocol)** — the consolidated enterprise standard for structured agent collaboration, following the merger of IBM's Agent Communication Protocol (ACP) in August 2025 under Linux Foundation governance
- **ANP (Agent Network Protocol)** — the open-network collaboration protocol for scenarios where no prior organizational trust relationship exists

---

## Section 21.1 — What Layer 2 Collaboration Actually Requires

### The Symmetry Problem

In MCP, the relationship is clear: the client requests, the server responds. The server has no agenda of its own. It does not renegotiate the task, request clarification mid-execution, or push back on the client's intent.

Between two agents, none of these assumptions hold. A delegating agent issues a task. The receiving agent has its own goals, resource constraints, failure modes, and potentially its own policy about what it will and will not do. It may need to ask for more information. It may run for seconds or hours. It may produce partial results before a final answer. It may hand off to a third agent.

### Five Requirements of a Layer 2 Protocol

A Layer 2 collaboration protocol must satisfy five requirements that do not apply at Layer 1:

| Requirement | Description | Why Layer 1 Doesn't Need It |
|---|---|---|
| **Discovery** | Agents cannot assume prior knowledge of each other. A client agent must find a remote agent, understand its capabilities, and assess fit before delegating. | MCP server URL is typically hardcoded in host config |
| **Task lifecycle management** | Delegated tasks are not instantaneous. Protocol must define creation, state transitions, cancellation, resumption, and handoff. | MCP tool calls are single-shot request-response |
| **Multi-turn interaction** | Many tasks require clarification, intermediate results, or additional input mid-execution. | MCP has no multi-turn task concept |
| **Asynchrony by default** | A remote agent may take minutes or hours. The protocol must decouple task submission from result delivery. | MCP tool calls complete synchronously or stream |
| **Cross-organizational identity and trust** | Layer 2 may cross org boundaries. Protocol must carry verifiable identity claims and support trust between parties with no prior relationship. | MCP operates within a single org's trust boundary |

> Any protocol that fails to address all five requirements will produce integration gaps that must be patched at the application layer — the most expensive and least reliable fix.

### Design Rules

| Rule ID | Rule |
|---|---|
| `L2-REQ-001` | Before selecting a Layer 2 protocol, all five requirements MUST be evaluated for the target deployment context — missing any creates application-layer gaps. |
| `L2-REQ-002` | A2A MUST be used for Layer 2 collaboration when a shared organizational trust context exists — ANP is for open-network scenarios only. |
| `L2-REQ-003` | MCP MUST NOT be used to implement agent-to-agent task delegation — it is structurally insufficient for Layer 2 (lacks task lifecycle, multi-turn, asynchrony, cross-org trust). |

---

## Section 21.2 — A2A: The Agent2Agent Protocol

### Origins, Governance, and the Path to v1.0

| Milestone | Date | Significance |
|---|---|---|
| A2A announced by Google Cloud | April 2025 | 50+ technology partners at launch (Atlassian, Salesforce, SAP, ServiceNow, LangChain, MongoDB) |
| IBM's ACP merged into A2A | August 2025 | Linux Foundation governance; IBM, Google, Microsoft, AWS, Cisco, Salesforce, SAP on TSC |
| A2A v1.0 shipped | March 2026 | First production-stable release — canonical proto model, three official bindings, extensions framework |

> *A2A is vendor-neutral under Linux Foundation governance. The TSC includes representatives from Google, Microsoft, AWS, Cisco, Salesforce, ServiceNow, SAP, and IBM.*

### Three-Layer Specification Architecture

A2A v1.0's specification is organized into three layers that separate concerns cleanly:

| Spec Layer | Description |
|---|---|
| **Layer 1 — Canonical Data Model** | Core data structures (`Task`, `Message`, `AgentCard`, `Part`, `Artifact`, `Extension`) defined in the normative `a2a.proto` file. All protocol bindings derive from this source — no binding-specific schema may diverge from the proto definition. |
| **Layer 2 — Abstract Operations** | Fundamental agent capabilities — Send Message, Stream Message, Get Task, List Tasks, Cancel Task, Subscribe to Task, push notification management — described independently of any specific binding. |
| **Layer 3 — Protocol Bindings** | Concrete mappings of abstract operations to wire protocols: **JSON-RPC 2.0 over HTTP** (original binding), **gRPC**, and **HTTP/JSON/REST**. Custom bindings permitted but must not break the canonical data model. |

> The proto-first normative model ensures protocol neutrality across bindings and eliminates the specification drift that plagued earlier multi-binding protocols.

### Agent Cards: Structured Discovery

The foundational discovery mechanism in A2A is the **Agent Card** — a JSON document published at a well-known URL, typically `.well-known/agent.json`:

```json
{
  "name": "Analytics Agent",
  "version": "1.2.0",
  "provider": { "organization": "Acme Corp", "url": "https://agents.acme.com" },
  "skills": [
    {
      "id": "data.analyze",
      "name": "Data Analysis",
      "description": "Performs statistical analysis on structured datasets",
      "inputModes": ["text", "file"],
      "outputModes": ["text", "data"]
    }
  ],
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "extendedAgentCard": true
  },
  "securitySchemes": {
    "oauth2": {
      "type": "oauth2",
      "flows": { "clientCredentials": { "tokenUrl": "https://auth.acme.com/token" } }
    }
  }
}
```

**Extended Agent Cards** (`GetExtendedAgentCard` — A2A v1.0): a more detailed capability description served only to authenticated clients. This two-tier disclosure model allows agents to publish minimal public capability information while reserving sensitive skill details (pricing, internal API structure, proprietary capabilities) for verified partners.

### Design Rules

| Rule ID | Rule |
|---|---|
| `A2A-CARD-001` | Every A2A-capable agent MUST publish a complete Agent Card at `.well-known/agent.json`. |
| `A2A-CARD-002` | Agent Cards MUST include all skills, capability flags, security schemes, and supported extensions — incomplete cards cause unnecessary discovery failures. |
| `A2A-CARD-003` | Sensitive skill details (pricing, internal endpoints, proprietary capabilities) MUST use the Extended Agent Card mechanism — not the public card. |
| `A2A-CARD-004` | Agent Cards MUST be machine-readable and semantically complete — they will be consumed by registries, security scanners, and automated governance tooling, not only by other agents. |

---

## Section 21.3 — The A2A Task Lifecycle

### Eight-State Task Model

A2A v1.0 defines an eight-state task lifecycle. Every state transition is a defined protocol event:

```
                    ┌─────────────────────────────────────┐
                    │ SUBMITTED                             │
                    └─────────────────────────────────────┘
                              ↓
                    ┌─────────────────────────────────────┐
                    │ WORKING                               │
                    └─────────────────────────────────────┘
              /          |          |          \
    INPUT-REQUIRED   AUTH-REQUIRED  ↓         FAILED
         ↓               ↓       COMPLETED
    (input pause)  (authz pause)
```

| State | Meaning |
|---|---|
| `submitted` | Task accepted by remote agent; not yet started |
| `working` | Remote agent actively executing |
| `input-required` | Remote agent needs clarifying input from the client to proceed (multi-turn pause) |
| `auth-required` | Remote agent needs explicit authorization grant from the client before executing a sensitive sub-operation |
| `completed` | Task finished successfully; artifacts available |
| `failed` | Task terminated with an error; failure details available |
| `canceled` | Task canceled by client request |
| `rejected` | Remote agent refused the task (capability mismatch, policy violation, capacity) |

### Three Communication Patterns

| Pattern | A2A Operation | When to Use |
|---|---|---|
| **Blocking (synchronous)** | `SendMessage` | Short tasks where the client can wait |
| **Streaming** | `StreamMessage` (SSE) | Long-running tasks that produce incremental output |
| **Asynchronous push** | `SendMessage` + `tasks/subscribe` or webhook push notifications | Tasks lasting minutes/hours where blocking is unacceptable |

### Design Rules

| Rule ID | Rule |
|---|---|
| `A2A-TASK-001` | Every A2A task implementation MUST handle all eight states — partial state handling produces subtly broken agents that appear to work in testing and fail in production. |
| `A2A-TASK-002` | The `input-required` state MUST be implemented for any skill that requires clarification — failing the task instead of pausing for input is a protocol misuse. |
| `A2A-TASK-003` | The `auth-required` state MUST be used when a sensitive sub-operation requires explicit authorization — it is the protocol-native mechanism for per-action authorization escalation. |
| `A2A-TASK-004` | Long-running tasks (expected duration > 10 seconds) MUST use asynchronous push or streaming — blocking the client thread on a long-running delegation is a reliability defect. |
| `A2A-TASK-005` | Task lifecycle state transitions MUST be persisted to durable storage before being reported to the client — in-memory-only state machines lose task state on restart. |

### Java + LangGraph Target

```java
public enum A2ATaskState {
    SUBMITTED, WORKING, INPUT_REQUIRED, AUTH_REQUIRED,
    COMPLETED, FAILED, CANCELED, REJECTED
}

public class A2ATaskLifecycleNode implements RunnableNode<AgentState> {
    private final A2ATaskRepository taskRepo;   // A2A-TASK-005: durable persistence
    private final A2APushNotifier notifier;

    @Override
    public AgentState run(AgentState state) {
        A2ATask task = taskRepo.load(state.currentTaskId());

        return switch (task.state()) {
            case SUBMITTED  -> transitionTo(task, A2ATaskState.WORKING, state);
            case WORKING    -> executeTask(task, state);
            case INPUT_REQUIRED -> pauseForInput(task, state);  // A2A-TASK-002
            case AUTH_REQUIRED  -> pauseForAuth(task, state);   // A2A-TASK-003
            case COMPLETED  -> deliverResult(task, state);
            case FAILED     -> handleFailure(task, state);
            case CANCELED,
                 REJECTED   -> terminateGracefully(task, state);
        };
    }

    private AgentState transitionTo(A2ATask task, A2ATaskState newState, AgentState s) {
        A2ATask updated = task.withState(newState);
        taskRepo.save(updated);           // A2A-TASK-005: persist before notifying
        notifier.sendStateUpdate(updated);
        return s.withTask(updated);
    }
}
```

---

## Section 21.4 — ACP: Historical Context and Contribution to A2A

### What ACP Was

IBM Research launched the **Agent Communication Protocol (ACP)** in March 2025 to power its BeeAI Platform. ACP's design philosophy was deliberately minimal and REST-native:

- Built on standard HTTP endpoints with OpenAPI-defined schemas
- Central abstraction was the **Run** (a single agent execution) rather than A2A's Task
- **Multipart `MessagePart` model**: natively handled multimodal content (text, images, structured JSON, embeddings) without custom encoding
- SDK-optional: an ACP server could be called with `curl` without any specialized tooling

### Why the Merger Happened

When A2A appeared in April 2025, both teams recognized that parallel development of two competing standards would fragment the ecosystem. The merger was announced jointly by Kate Blair (IBM Research) and Todd Segal (Google, A2A TSC) in August 2025:

> *"By bringing the assets and expertise behind ACP into A2A, we can build a single, more powerful standard for how AI agents communicate and collaborate."* — Kate Blair, IBM Research, August 2025

### ACP's Technical Contributions to A2A v1.0

| ACP Contribution | A2A v1.0 Manifestation |
|---|---|
| Multipart `MessagePart` model | `Part` abstraction in A2A — smallest content unit in `Message`/`Artifact`, carrying text, file references, or structured data |
| Native multimodality | A2A's ability to exchange diverse content types within a single message |
| REST-native ergonomics | HTTP/JSON/REST protocol binding (third official binding in v1.0) |
| SDK-optional deployment | REST binding enables `curl`-callable agents — first-class option in unified protocol |

### Implications for Engineers

- ACP is **no longer under active development**
- The BeeAI platform is now A2A-native
- Migration from ACP to A2A is supported by official tooling
- **All new multi-agent integration work targets A2A**
- Engineers familiar with ACP will find A2A ergonomically familiar: Run → Task, MessagePart → Part

> 💡 **If you encounter ACP support listed as a capability in third-party frameworks or tooling, treat it as a proxy for A2A compatibility. Virtually all frameworks that adopted ACP have since adopted or are adopting A2A.**

---

## Section 21.5 — ANP: The Agent Network Protocol

### The Problem ANP Addresses

Both A2A and ACP assumed that the organizations deploying collaborating agents have some prior relationship — a shared identity provider, negotiated authentication mechanisms, or common governance framework. This assumption holds within an enterprise or defined partner ecosystem. It breaks in open, public, decentralized networks where any agent should be able to discover and interact with any other agent without prior coordination.

**ANP's core premise:** agents should be able to interact on the internet the way web pages interact — discoverable, linkable, and accessible to any agent that speaks the protocol, without pre-registration or bilateral trust agreements.

### ANP's Three-Layer Architecture

| ANP Layer | Description | Technology |
|---|---|---|
| **Layer 1 — Identity & Encrypted Communication** | Each agent has a DID (W3C Decentralized Identifier) as its globally unique, self-sovereign identifier. `did:wba` (Web-Based Agent DID) anchors identifiers to standard HTTPS/DNS — no blockchain required. Agents authenticate using cryptographic proofs from their DID keys. All communication is end-to-end encrypted. | W3C DIDs, `did:wba`, PKI |
| **Layer 2 — Meta-Protocol Negotiation** | Before exchanging content, two ANP agents negotiate which application protocol to use (e.g., A2A, custom). An ANP agent advertises supported protocols; its counterpart selects the best match. | ANP negotiation handshake |
| **Layer 3 — Application Protocol** | The actual content exchange using the negotiated protocol. ANP specifies the **Agent Description Protocol (ADP)** — a JSON-LD document describing an agent's skills and services in semantic, machine-readable terms, compatible with web standards and indexable by search engines. | JSON-LD, ADP |

### Discovery in an Open Network

ANP's discovery model mirrors the web:
- An agent publishes its ADP document at a well-known URL
- Other agents discover it through web search, direct URL knowledge, or structured registries
- ADP documents are JSON-LD anchored to standard HTTPS endpoints — crawlable and indexable by conventional web infrastructure

### A2A vs. ANP Decision Framework

| Criterion | Use A2A | Use ANP |
|---|---|---|
| Counterpart relationship | Known organizational counterpart with shared IdP | Unknown or untrusted external agent, no shared IdP |
| Task lifecycle | Full A2A eight-state lifecycle | ANP negotiates then typically delegates to A2A for task execution |
| Identity model | OIDC-issued tokens (OAuth 2.1) | Cryptographic DID proofs |
| Framework/tooling support | Mature — all major agent frameworks | 2026 — early stage |
| Governance maturity | Linux Foundation, v1.0 stable | W3C Community Group, specification stage |
| Production deployment | Ready | Limited — reference implementations only |

> ⚠️ **ANP is at the specification and reference implementation stage as of early 2026. Production adoption remains limited. Treat ANP as a forward-looking reference design and monitor W3C standardization progress rather than deploying it as a primary production protocol today.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `ANP-001` | ANP MUST only be adopted at the organizational perimeter — for interactions with agents outside any established trust relationship. |
| `ANP-002` | ANP deployments MUST use `did:wba` for agent identity — blockchain-dependent DID methods are not operationally viable for enterprise deployments. |
| `ANP-003` | ANP's meta-protocol negotiation MUST be implemented completely — hardcoding the application protocol defeats the open-network composability that ANP provides. |
| `ANP-004` | Production systems MUST monitor ANP W3C standardization progress before committing to deep ANP integration — the specification may evolve materially. |

---

## Section 21.6 — Combining the Protocols in a Full Stack

### A Realistic 2026 Multi-Agent System

Most production multi-agent systems in 2026 use all three protocol layers in combination:

```
Inbound request from external agent
  → ANP Layer 1: verify DID-based identity                     [Layer 2 boundary, ANP]
  → ANP Layer 2: negotiate application protocol (selects A2A)  [Layer 2 boundary, ANP]
  → Agent Gateway: validate credential + delegation chain,      [Layer 3 gateway]
                   enforce capability scope, route to agent
  → A2A Task submission + multi-turn lifecycle                  [Layer 2, A2A]
  → Internal agent calls tools via MCP                         [Layer 1, MCP]
```

**The full per-interaction protocol stack flow** (five steps, five protocol layers):

1. **ANS / `.well-known/agent.json`** — external agent discovers the org's capability (Layer 3 discovery)
2. **ANP handshake** — DID-based identity verification + protocol negotiation (Layer 2, ANP)
3. **Agent Gateway inbound** — validates credential + signed card provenance, delegation chain, capability scope (Layer 3 gateway)
4. **A2A task collaboration** — task submission, multi-turn clarification, streaming results (Layer 2, A2A)
5. **MCP tool access** — agent fetches MCP Server Card from `.well-known/mcp.json`, calls tool (Layer 1, MCP)

> *Note: In step 5, the same well-known discovery pattern used in step 1 for agent discovery is used for tool discovery — A2A Agent Cards and MCP Server Cards under `.well-known` create a unified discovery surface across both protocol layers.*

### Design Rules

| Rule ID | Rule |
|---|---|
| `STACK-001` | MCP at Layer 1 for tool access, A2A at Layer 2 for enterprise agent collaboration, ANP at the organizational perimeter — each layer's protocol addresses only its scope. |
| `STACK-002` | The Agent Gateway MUST be deployed at every organizational trust boundary before any cross-boundary A2A or ANP interactions are permitted. |
| `STACK-003` | The full five-step per-interaction flow MUST be documented as an architecture reference — it is the operational specification for cross-boundary collaboration. |

---

## Section 21.7 — Security Across Layer 2

### Three Layer 2-Specific Security Concerns

| Concern | Description | Protocol Defense |
|---|---|---|
| **Agent impersonation** | A malicious agent presents itself as a trusted specialized agent using a forged or stolen identity, receives delegated tasks, returns corrupted results, or escalates privileges | A2A: OIDC-issued tokens validated; ANP: cryptographic DID proofs — both require correct credential validation |
| **Task injection** | Adversarially crafted task payload contains instructions designed to manipulate the receiving agent's reasoning (protocol-layer prompt injection) | Treat all task content as **untrusted input** regardless of sender identity or trust level |
| **Authorization scope creep** | A client agent requests scope of action broader than its original mandate; remote agent has no mechanism to verify the delegating agent's authorization boundary | A2A skill-level authorization scopes + `auth-required` task state + application-level policy enforcement |

### The Confused Deputy in Multi-Agent Systems

The **confused deputy problem**: a component with legitimate access to a resource is manipulated by an unauthorized party into misusing that access.

In multi-agent chains:
1. An orchestrator agent with broad permissions delegates to a subagent
2. The subagent, following instructions injected by a malicious input, uses the orchestrator's authorization scope
3. Actions the original user never intended are performed under the orchestrator's authority

> ⚠️ **The confused deputy attack is not a theoretical concern. It is the most commonly exploited vulnerability pattern in multi-agent systems that have reached production. Every agent in a collaboration chain must enforce its own authorization policy. Trust in identity does not imply trust in instructions.**

### The Multi-Hop Trust Gap in A2A

A2A's security model handles the single-hop case well. For multi-hop chains — where agentA delegates to agentB which delegates to agentC — A2A does not prescribe a specific delegated authorization mechanism. Engineers must implement token chaining or propagation patterns at the application layer.

> ⚠️ **A2A's multi-hop trust gap is a real production risk. If agentB presents its own identity when calling agentC, agentC cannot verify that the original request came from agentA with appropriate authorization. Design explicit delegation token strategies before deploying A2A in chains deeper than two hops.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `A2A-SEC-001` | Every A2A agent MUST validate incoming credentials completely — signature, expiry, issuer, audience, and required scopes — regardless of the trust level of the sending party. |
| `A2A-SEC-002` | All task content MUST be treated as untrusted data — task payloads are never trusted instructions even when the sending agent's identity is verified. |
| `A2A-SEC-003` | Every agent in a delegation chain MUST enforce its own authorization policy — delegated tokens from the previous hop do not confer automatic authorization. |
| `A2A-SEC-004` | Systems deploying A2A in chains deeper than two hops MUST implement an explicit delegated authorization token strategy (aligned with `SEC-002` from chap19) before production deployment. |
| `A2A-SEC-005` | The `auth-required` task state MUST be used for sensitive sub-operations rather than granting broad permissions at connection time. |

### Java + LangGraph Target

```java
// A2A-SEC-002: task content is untrusted data regardless of sender identity
public class A2ATaskReceiverNode implements RunnableNode<AgentState> {
    private final A2ACredentialValidator credentialValidator;
    private final TaskContentSanitizer contentSanitizer;     // A2A-SEC-002
    private final AgentAuthorizationPolicy authPolicy;       // A2A-SEC-003

    @Override
    public AgentState run(AgentState state) {
        A2AInboundTask inbound = state.inboundTask();

        // A2A-SEC-001: full credential validation
        A2AIdentityClaims claims = credentialValidator.validate(
            inbound.bearerToken(),
            expectedIssuer, expectedAudience, requiredScopes
        );

        // A2A-SEC-003: this agent enforces its own policy
        authPolicy.assertPermitted(claims, inbound.requestedSkill());

        // A2A-SEC-002: sanitize content before passing to reasoning loop
        A2ATask sanitizedTask = contentSanitizer.sanitize(inbound.task());

        return state.withActiveTask(sanitizedTask);
    }
}
```

---

## Section 21.8 — A2A Protocol Bindings in Production

### Choosing a Binding

| Binding | Transport | Best For | Notes |
|---|---|---|---|
| **JSON-RPC 2.0 over HTTP** | HTTP/1.1 or HTTP/2 | Broad compatibility, web-native integrations, existing HTTP infrastructure | Original binding; most widely implemented in 2026 |
| **gRPC** | HTTP/2 + Protocol Buffers | High-throughput internal services, polyglot environments, streaming-heavy workloads | Lower overhead; requires gRPC infrastructure |
| **HTTP/JSON/REST** | HTTP REST | Browser clients, `curl`-friendly testing, teams preferring JSON without JSON-RPC envelope | ACP's REST-native contribution; SDK-optional |

### Design Rules

| Rule ID | Rule |
|---|---|
| `A2A-BIND-001` | Binding selection MUST be documented as an architecture decision with rationale — changing bindings after production deployment requires updating every integration point. |
| `A2A-BIND-002` | Agents SHOULD support at minimum two bindings (JSON-RPC and REST) to accommodate diverse client environments without requiring all clients to adopt gRPC infrastructure. |
| `A2A-BIND-003` | The canonical `a2a.proto` data model MUST be treated as normative — binding-specific data transformations that diverge from the proto definition are specification violations. |

---

## Section 21.9 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `L2-REQ-001` | Layer 2 requirements | All five requirements evaluated before protocol selection |
| `L2-REQ-002` | Layer 2 requirements | A2A for shared-trust context; ANP for open network |
| `L2-REQ-003` | Layer 2 requirements | MCP MUST NOT implement agent-to-agent delegation |
| `A2A-CARD-001` | Agent Cards | Every A2A agent publishes Agent Card at `.well-known/agent.json` |
| `A2A-CARD-002` | Agent Cards | Cards include all skills, capabilities, security schemes, extensions |
| `A2A-CARD-003` | Agent Cards | Sensitive details use Extended Agent Card mechanism |
| `A2A-CARD-004` | Agent Cards | Cards machine-readable for registries, scanners, governance tooling |
| `A2A-TASK-001` | Task lifecycle | All eight states handled |
| `A2A-TASK-002` | Task lifecycle | `input-required` implemented for clarification-capable skills |
| `A2A-TASK-003` | Task lifecycle | `auth-required` used for sensitive sub-operation authorization |
| `A2A-TASK-004` | Task lifecycle | Long tasks use async push or streaming |
| `A2A-TASK-005` | Task lifecycle | State transitions persisted before client notification |
| `ANP-001` | ANP | ANP only at organizational perimeter |
| `ANP-002` | ANP | `did:wba` for agent identity |
| `ANP-003` | ANP | Meta-protocol negotiation implemented completely |
| `ANP-004` | ANP | Monitor W3C standardization before deep ANP commitment |
| `STACK-001` | Full stack | MCP Layer 1, A2A Layer 2, ANP at perimeter — each layer addresses its scope |
| `STACK-002` | Full stack | Agent Gateway at every organizational trust boundary |
| `STACK-003` | Full stack | Five-step per-interaction flow documented as architecture reference |
| `A2A-SEC-001` | Security | Full credential validation regardless of sender trust level |
| `A2A-SEC-002` | Security | All task content treated as untrusted data |
| `A2A-SEC-003` | Security | Every agent enforces its own authorization policy |
| `A2A-SEC-004` | Security | Explicit delegated token strategy for chains > 2 hops |
| `A2A-SEC-005` | Security | `auth-required` state for sensitive sub-operations |
| `A2A-BIND-001` | Bindings | Binding selection documented as architecture decision |
| `A2A-BIND-002` | Bindings | Minimum two bindings (JSON-RPC + REST) recommended |
| `A2A-BIND-003` | Bindings | `a2a.proto` canonical data model is normative |

---

## Section 21.10 — Transition to Chapter 22

Chapter 21 has covered Layer 2 — structured agent collaboration within and across organizations. Both A2A and ANP assume some context: a known domain for Agent Card discovery, a resolvable DID, an existing deployment.

Chapter 22 removes those assumptions entirely. **Layer 3: The Agent Web** is the architecture in which autonomous agents built by different organizations, running on different infrastructure, speaking different application protocols, can discover each other, establish trust, and collaborate — without prior coordination, bilateral agreements, or shared deployment context.

| Concern | Layer 2 (A2A / ANP) | Layer 3 (Agent Web) |
|---|---|---|
| Discovery | Known domain or DID | Cold-start: task-description-first, no known domain |
| Trust establishment | Shared IdP (A2A) or DID proof (ANP) | SPIFFE/SPIRE + Verifiable Credentials + Token Exchange |
| Policy enforcement | Per-agent (distributed) | Centralized at Agent Gateway |
| Federation topology | Bilateral agreements | Hub-and-spoke (2026 dominant) or mesh |
| Governance | Linux Foundation (A2A) + W3C CG (ANP) | Per-organization policy + emerging Agent Name Service |

> Chapter 22 assembles the full Agent Web: the Layer 3 architecture that emerges when agents, tool servers, and open networks are composed into a coherent, operating whole — covering discovery (well-known endpoints + ANS), trust infrastructure (SPIFFE/SPIRE, Verifiable Credentials, OAuth Token Exchange), Agent Gateways, federation topologies, and the governance gap that is the defining operational risk of Layer 3 in 2026.
