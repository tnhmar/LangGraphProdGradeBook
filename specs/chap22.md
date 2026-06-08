# Chapter 22 Spec — Layer 3: The Agent Web

> **Volume 1 reference:** `chlayer-3-the-agent-web`
> **Part:** 5 — Agent Communication Protocols
> **Depends on:** chap19 (`PROTO-001–006`), chap20 (`MCP-ARCH-001–005`), chap21 (`A2A-CARD-001–004`, `A2A-SEC-001–005`, `ANP-001–004`, `STACK-001–003`)
> **Java + LangGraph implementation target:** Volume 2 (Agent Gateway, SPIFFE/SPIRE integration, Agent inventory, VC validation, ANS resolver)
> **Bridges to:** chap23 (Observability across protocol layers), chap33 (Enterprise governance), chap34 (Regulatory compliance)

> *"The internet was not designed for agents. The Agent Web must be."*
> — Agent Network Protocol White Paper, 2025

---

## Chapter Purpose

Chapters 20 and 21 covered the two inner layers of the agent communication stack — MCP at Layer 1 for tool access, and A2A and ANP at Layer 2 for agent collaboration. **Both layers assumed that the parties involved have some prior relationship** — a shared identity provider, a pre-negotiated protocol, or at minimum a known endpoint.

Layer 3 removes that assumption entirely.

The **Agent Web** is the emerging architecture in which autonomous agents built by different organizations, running on different infrastructure, speaking different application protocols, can discover each other, establish trust, and collaborate — **without prior coordination, bilateral agreements, or shared deployment context**.

This is the hardest problem in agent protocol design, and it is largely unsolved in production as of 2026. This chapter maps the current state: the infrastructure being built, the federation models in use, the trust frameworks being standardized, and the governance gaps that deployment is already exposing.

---

## Section 22.1 — From Closed Ecosystems to an Open Agent Web

### The Closed Ecosystem Starting Point

Every agent system begins closed. An enterprise builds agents for internal use. They call internal tools via MCP. They collaborate via A2A within a governance boundary the organization controls. Discovery is handled by deployment configuration. Trust is handled by a shared corporate identity provider. The system works precisely because everything is known in advance.

This closed model has a ceiling:
- A procurement agent that can only consult internal tools is less capable than one that can reach external supplier agents, logistics networks, and financial clearing systems
- A research agent confined to internal knowledge bases is less capable than one that can delegate to specialized scientific literature agents run by universities or publishers
- The value of an agent is partly a function of the **network of agents and tools it can reach**

The Agent Web is the infrastructure that extends this reach beyond organizational boundaries without requiring bilateral pre-integration agreements for every pair of parties.

### Three Structural Shifts

Moving from a closed ecosystem to an open agent web requires three structural shifts that do not arise inside a single organization:

| Shift | Description | Why It Doesn't Exist Inside One Org |
|---|---|---|
| **Open discovery** | Agents must find other agents they have never interacted with before, learn their capabilities, and assess trustworthiness — all at runtime, without human-operator configuration | Deployment config handles discovery internally |
| **Decentralized trust** | The common corporate IdP that anchors trust within one org does not extend to unknown external agents. Trust requires cryptographically verifiable credentials that don't depend on a shared authority | Shared Okta/Entra tenant handles identity |
| **Governed interoperability** | Open networks without governance become vectors for abuse: spam, impersonation, unauthorized data access, coordinated manipulation. Governance must be protocol-aware — enforced at the communication boundary, not after the fact | Internal policy engines handle governance |

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-STRUCT-001` | Agent systems MUST be designed with an explicit Layer 3 strategy before any cross-organizational interactions are permitted — retrofitting open-network infrastructure onto a closed system is one of the most expensive rewrites in agent architecture. |
| `L3-STRUCT-002` | All three structural shifts (open discovery, decentralized trust, governed interoperability) MUST be addressed independently — solving one without the others leaves the system unsecured at the unsolved boundary. |
| `L3-STRUCT-003` | The value ceiling of a closed system MUST be documented as a known architectural constraint when making the decision to remain closed — this makes the future cost of opening explicit. |

---

## Section 22.2 — Agent Discovery at Scale

### The Discovery Gap

As of 2026, **no universal agent registry exists**. An agent that wants to find an appropriate counterpart for a task cannot issue a structured query against a global directory and receive a verified response. The ecosystem is addressing this through two parallel efforts approaching production readiness.

### Well-Known Endpoints: A Converging Standard

The foundational Layer 3 discovery mechanism builds on a **well-known endpoint convention** that both A2A and MCP are now adopting, creating a convergent discovery surface across both layers of the protocol stack:

| Protocol | Well-Known Endpoint | Document Type | Content |
|---|---|---|---|
| **A2A** | `.well-known/agent.json` | Agent Card | Capabilities, skills, auth requirements, extensions |
| **MCP** (2026 roadmap) | `.well-known/mcp.json` | Server Card | Tools, resources, prompts, auth requirements |

Both are discoverable by the same crawlers, registries, and security scanners — **unifying discovery infrastructure across Layers 1 and 2**.

> **Architectural significance:** When a client agent knows a domain, it can fetch both the Agent Card and the MCP Server Card from that domain using the same HTTP GET pattern, learning what collaboration capabilities and what tool capabilities that domain exposes — before initiating any live connection.

> ⚠️ **The well-known endpoint convention solves the known-domain discovery problem** — given a domain, find what agent or tool services it exposes. It does **not** solve the cold-start discovery problem — given only a task description, find an appropriate agent with no known domain to start from. Cold-start discovery requires the Agent Name Service.

### The Agent Name Service (ANS)

The **Agent Name Service (ANS)**, submitted to the IETF as a draft in May 2025, proposes a dedicated discovery layer modeled explicitly on DNS:

| ANS Component | Description |
|---|---|
| **DNS-inspired naming** | Agents register under structured namespaces such as `procurement.acme.ans` encoding organizational hierarchy and capability domain — enables structured query by function rather than by known endpoint |
| **PKI-anchored identity** | Registry records are signed with PKI certificates — any querying agent can validate without contacting the registrant |
| **Capability-aware resolution** | Unlike DNS (returns IP addresses), ANS returns structured capability descriptors: protocols spoken, skills offered, auth mechanisms required — a querying agent can filter on capability before initiating contact |
| **Protocol-agnostic** | Registry records include protocol tags (A2A, MCP, ANP) — client agents select a counterpart that speaks a compatible protocol |
| **Federated registries** | Multiple registry operators can federate their namespaces, with DNSSEC-equivalent signing providing tamper-evident records |

> *GoDaddy launched a public ANS API in 2025, positioning registry operation as an extension of the domain registrar model.*

> 📝 **ANS is an IETF draft as of early 2026, not a ratified standard.** Implementations exist but the specification is still evolving. Use well-known endpoints + Agent Cards for production today. Monitor ANS progress for cold-start discovery capability.

**ANS and A2A Agent Cards are complementary:** ANS handles naming and capability-first discovery; A2A handles the interaction protocol once the counterpart is found.

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-DISC-001` | Every agent exposed to cross-organizational interaction MUST publish a complete, machine-readable Agent Card at `.well-known/agent.json` — this is the minimum viable Layer 3 discovery surface. |
| `L3-DISC-002` | Every MCP server exposed beyond the organizational boundary MUST publish a Server Card at `.well-known/mcp.json` when the 2026 MCP roadmap endpoint stabilizes — monitor for specification release. |
| `L3-DISC-003` | Agent Card and Server Card metadata MUST be treated as security-sensitive artifacts — incomplete or stale cards create invisible agents that bypass governance tooling. |
| `L3-DISC-004` | ANS MUST NOT be used as the sole discovery mechanism in production in 2026 — it is an IETF draft. Use as supplementary capability-first discovery alongside well-known endpoints. |
| `L3-DISC-005` | Cold-start discovery (task-description-first, no known domain) MUST be explicitly handled in the architecture — either through ANS integration, curated marketplace catalogs, or documented human-configuration fallback. |

### Java + LangGraph Target

```java
public class AgentDiscoveryNode implements RunnableNode<AgentState> {
    private final WellKnownCardFetcher cardFetcher;       // L3-DISC-001
    private final AnsResolver ansResolver;                 // L3-DISC-004: supplementary only
    private final AgentCardValidator cardValidator;

    @Override
    public AgentState run(AgentState state) {
        String targetDomain = state.targetAgentDomain();

        // Step 1: well-known endpoint (primary, production-ready)
        Optional<AgentCard> card = cardFetcher.fetchAgentCard(targetDomain);

        // Step 2: ANS fallback for capability-first cold-start (supplementary)
        if (card.isEmpty() && state.hasCapabilityQuery()) {
            card = ansResolver.resolveByCapability(state.capabilityQuery()); // L3-DISC-005
        }

        AgentCard validated = card
            .map(cardValidator::validateSignature)   // L3-TRUST-002: verify signed card
            .orElseThrow(() -> new AgentDiscoveryException("No agent found for: " + targetDomain));

        return state.withDiscoveredAgent(validated);
    }
}
```

---

## Section 22.3 — Trust Infrastructure for Open Networks

### Why Organizational Trust Boundaries Do Not Compose

Inside a single organization, trust is anchored to a corporate identity provider — an Okta tenant, a Microsoft Entra directory, or equivalent. Agents authenticate with tokens issued by that provider.

**Across organizations, no shared anchor exists.** Organization A trusts Entra. Organization B trusts a different Okta tenant. When an agent from A calls an agent from B, B's authorization server cannot verify A's identity token because it was issued by an authority B does not trust.

The solution is not to create a universal identity provider — that shifts the centralization problem. The solution is to **compose trust from verifiable cryptographic primitives** that each party can validate independently.

### Four Components of Cross-Org Trust

| Component | What It Provides | Standard | Trust Lifecycle Point |
|---|---|---|---|
| **Signed Agent Cards** | Tamper-evident discovery documents with pre-handshake identity verification | A2A v1.0 | Discovery time — before any connection |
| **SPIFFE/SPIRE** | Cryptographic workload identity at the transport layer (X.509 SVIDs / JWTs, mTLS) | CNCF SPIFFE spec | Transport layer — before any application message |
| **Verifiable Credentials** | Signed capability and compliance claims, self-contained and verifiable without contacting issuer | W3C VC Data Model + DIDs | Application layer — before any task is accepted |
| **OAuth 2.0 Token Exchange** | Cross-organizational authorization propagation across delegation chains (RFC 8693 + RAR RFC 9396) | IETF RFC 8693, 9396 | Delegation layer — as tasks are delegated across orgs |

### How the Components Compose

Each component addresses trust at a different point in an interaction:

1. **Signed Agent Cards** establish organizational identity at discovery time, before any connection is made
2. **SPIFFE SVIDs** establish workload identity at the transport layer, before any application message is exchanged
3. **Verifiable Credentials** establish claimed capabilities and compliance posture at the application layer, before any task is accepted
4. **OAuth Token Exchange** propagates authorization across organizational boundaries as tasks are delegated

An agent interaction that traverses all four components gives the receiving party high confidence in:
- **Who is calling** (signed Agent Card + SPIFFE SVID)
- **That the calling process is a legitimate workload** rather than a spoofed client (SPIFFE)
- **What the caller claims to be capable of and compliant with** (Verifiable Credentials)
- **What the caller is specifically authorized to request** (OAuth Token Exchange with RAR scopes)

No single component provides this full picture — the layered composition does.

### The Delegation Chain Problem

Chapter 21 identified multi-hop trust chains as a known gap in A2A's security model. The cross-org trust infrastructure above addresses this systematically:

When **agentA (orgX)** delegates to **agentB (orgY)** which calls **agentC (orgZ)**, the delegation chain is encoded as a sequence of OAuth token exchanges:
- A obtains a token for B's authorization server
- B presents that token along with its own identity to obtain a token for C's authorization server
- Each hop produces an auditable token that encodes the full delegation history

> ⚠️ **This mechanism is technically sound but adds latency.** In high-frequency, low-latency agent collaboration scenarios, the overhead of multiple synchronous token exchanges may be prohibitive. Engineers must evaluate whether full delegation chain integrity is necessary for their risk profile.

> 💡 **Do not implement cross-organizational trust from scratch.** SPIFFE/SPIRE is production-grade open-source infrastructure that major cloud providers support natively. OAuth Token Exchange is a well-tested IETF standard. Signed Agent Cards are a built-in capability of A2A v1.0. Build on these primitives rather than designing bespoke credential formats.

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-TRUST-001` | Cross-organizational agent interactions MUST use all four trust components — signed Agent Cards, SPIFFE/SPIRE, Verifiable Credentials, and OAuth Token Exchange — as a layered stack. Omitting any component leaves a trust gap at the corresponding lifecycle point. |
| `L3-TRUST-002` | Signed Agent Card signatures MUST be verified before any A2A interaction with an external agent is initiated — unsigned or unverifiable cards MUST be rejected. |
| `L3-TRUST-003` | SPIFFE/SPIRE MUST be deployed for all agent workloads that participate in cross-organizational interactions — ad-hoc mTLS certificate management is not a substitute. |
| `L3-TRUST-004` | Verifiable Credentials MUST be verified against the issuer's public key before accepting any capability or compliance claim from an external agent. |
| `L3-TRUST-005` | OAuth Token Exchange (RFC 8693) with Rich Authorization Requests (RFC 9396) MUST be used for cross-organizational task delegation — forwarding tokens issued by one org to another org's authorization server is a security violation. |
| `L3-TRUST-006` | Multi-hop delegation chains MUST be explicitly designed before deploying agents in chains > 2 organizational hops — the latency cost of full chain integrity must be evaluated against the risk profile. |

### Java + LangGraph Target

```java
// Cross-org inbound request validation: all four trust components
public class CrossOrgTrustValidationNode implements RunnableNode<AgentState> {
    private final AgentCardSignatureVerifier cardVerifier;         // L3-TRUST-002
    private final SpiffeIdentityVerifier spiffeVerifier;           // L3-TRUST-003
    private final VerifiableCredentialVerifier vcVerifier;         // L3-TRUST-004
    private final OAuthTokenExchangeValidator tokenValidator;       // L3-TRUST-005

    @Override
    public AgentState run(AgentState state) {
        CrossOrgRequest req = state.inboundCrossOrgRequest();

        // Layer 1: signed Agent Card provenance (discovery-time trust)
        cardVerifier.verifySignature(req.agentCard());

        // Layer 2: SPIFFE workload identity (transport-time trust)
        SpiffeIdentity workloadId = spiffeVerifier.verify(req.mtlsCertificate());

        // Layer 3: Verifiable Credentials (application-time capability trust)
        VerifiedClaims claims = vcVerifier.verify(req.verifiableCredential());

        // Layer 4: OAuth Token Exchange chain (delegation-time authorization)
        DelegationChain chain = tokenValidator.validateExchangeChain(req.accessToken()); // L3-TRUST-005

        return state.withValidatedCrossOrgIdentity(workloadId, claims, chain);
    }
}
```

---

## Section 22.4 — Agent Gateways: The Policy Enforcement Point

### The Need for a Mediation Layer

Even with strong discovery and cryptographic trust infrastructure, open agent networks require a point at which an organization's policies are enforced on all cross-boundary agent interactions. Distributing policy enforcement across every individual agent is impractical: policy changes require redeploying every agent, and consistent enforcement cannot be verified.

An **Agent Gateway** centralizes this function.

> An Agent Gateway is a policy-enforcing intermediary that sits at the boundary of an organizational trust domain and mediates **all** agent communication crossing that boundary — both inbound (external agents calling internal agents) and outbound (internal agents calling external agents). It is the agent equivalent of an API Gateway, extended to handle the specific properties of agentic interactions: multi-turn sessions, streaming results, delegation chains, and capability-scoped authorization.

### Six Gateway Functions

| Function | Description | Why Agents Must Not Do This Individually |
|---|---|---|
| **Mutual authentication** | Verifies identity of inbound callers (SPIFFE SVIDs or OAuth tokens); presents own identity to outbound targets | Agents behind the gateway do not perform their own transport-level auth |
| **Authorization policy enforcement** | Evaluates authorization policies (agent identity + requested operation + context) via PDP/PEP before forwarding any request | Policy changes require redeployment of every agent individually |
| **Delegation chain validation** | Inspects token chain on inbound requests; rejects broken or suspicious delegation chains before they reach internal agents | Agents cannot see the full chain — only their immediate predecessor |
| **Rate limiting and abuse prevention** | Enforces per-agent and per-organization rate limits; detects anomalous patterns; blocks over-threshold requests | Individual agents see only their own request stream — cannot detect cross-agent abuse |
| **Audit logging** | Logs every cross-boundary interaction with full request context; authoritative because it cannot be altered by monitored agents | Agent-side logs can be corrupted by a compromised agent |
| **Protocol translation** | Translates between compatible protocol bindings (e.g., A2A JSON-RPC inbound → gRPC outbound) | Eliminates requirement for every agent to implement multiple bindings |

> ⚠️ **An Agent Gateway is not a performance optimization or a convenience — it is a security and governance requirement for any agent system that crosses organizational trust boundaries.** Operating without a gateway means distributing policy enforcement across every agent individually — an approach that fails consistently under the pressure of real deployments, version drift, and evolving requirements.

### Hub-and-Spoke vs. Mesh Federation

| Dimension | Hub-and-Spoke | Mesh |
|---|---|---|
| **Policy enforcement** | Centralized at gateway | Distributed per agent |
| **Revocation** | Single revocation point — disable gateway severs all external connections | No centralized revocation mechanism |
| **Debugging** | Gateway logs provide full cross-boundary visibility | Requires distributed tracing across all parties |
| **Availability** | Gateway is a single point of failure | No single point of failure |
| **Scalability** | Gateway may become bottleneck at very high request volume | Scales linearly with agent count |
| **Operational maturity** | Well-understood, dominant pattern in enterprise 2026 | Aspirational for most organizations |

**Hub-and-spoke federation is the operationally mature choice in 2026.** Mesh federation is technically possible using SPIFFE mTLS, signed Agent Cards, and DID-VC attestations instead of a shared broker — but requires distributed policy enforcement, no centralized revocation, and substantially harder cross-boundary interaction debugging.

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-GW-001` | An Agent Gateway MUST be deployed at every organizational trust boundary before any cross-boundary A2A or ANP interactions are permitted. |
| `L3-GW-002` | All six gateway functions (mutual auth, authorization enforcement, delegation chain validation, rate limiting, audit logging, protocol translation) MUST be implemented — partial gateway implementations create the false confidence of security theater. |
| `L3-GW-003` | Hub-and-spoke federation MUST be the default topology for 2026 production deployments — mesh federation requires organizational capability in distributed policy and tracing that most organizations do not yet have. |
| `L3-GW-004` | Gateway audit logs MUST be stored in a tamper-evident store separate from the agents the gateway monitors — agent-alterable audit logs are not audit logs. |
| `L3-GW-005` | The gateway MUST be the single point of revocation for all external agent access — individual agent credential revocation is insufficient; the gateway must also terminate active sessions. |
| `L3-GW-006` | Protocol translation at the gateway MUST preserve the canonical A2A proto data model — translations that silently alter task state, artifacts, or identity claims are security violations. |

### Java + LangGraph Target

```java
public class AgentGatewayNode implements RunnableNode<AgentState> {
    private final MutualAuthenticator mutualAuth;               // L3-GW-002
    private final PolicyDecisionPoint pdp;                      // L3-GW-002
    private final DelegationChainValidator chainValidator;       // L3-GW-002
    private final RateLimiter rateLimiter;                      // L3-GW-002
    private final TamperEvidentAuditLogger auditLogger;         // L3-GW-004
    private final ProtocolTranslator protocolTranslator;        // L3-GW-006

    @Override
    public AgentState run(AgentState state) {
        CrossBoundaryRequest req = state.pendingCrossBoundaryRequest();

        // 1. Mutual authentication
        AuthenticatedIdentity identity = mutualAuth.authenticate(req);

        // 2. Delegation chain validation
        chainValidator.validate(req.delegationChain());         // L3-TRUST-006

        // 3. Policy decision
        PolicyDecision decision = pdp.evaluate(identity, req.requestedOperation());
        if (!decision.isPermitted()) {
            auditLogger.logRejection(req, identity, decision);  // L3-GW-004
            return state.withGatewayRejection(decision.reason());
        }

        // 4. Rate limit check
        rateLimiter.checkAndRecord(identity.agentId());

        // 5. Protocol translation (if needed)
        ForwardableRequest translated = protocolTranslator.translate(req); // L3-GW-006

        // 6. Audit log (before forwarding — authoritative record)
        auditLogger.logPermitted(req, identity, decision);

        return state.withForwardedRequest(translated);
    }
}
```

---

## Section 22.5 — Governance: The Gap Between Deployment and Control

### The Governance Gap

RSA 2026 produced a striking finding: **more than two-thirds of organizations cannot clearly distinguish which AI agents are operating in their environment.** Agents are being deployed faster than governance structures can track them.

This is not primarily a technology failure — it is an **organizational and architectural failure** with serious security and compliance consequences.

The governance gap has three dimensions:

| Dimension | Description | Consequence |
|---|---|---|
| **Visibility** | If an organization cannot enumerate its own agents, it cannot enforce policy on them, audit their behavior, or respond when one misbehaves | Invisible agents = unmanaged agents = uncontrolled liability |
| **Ownership** | Agents without assigned human owners operate without accountability | When an agent causes harm, absence of a responsible party makes remediation and liability resolution difficult |
| **Boundary definition** | Agents need explicit, machine-enforceable definitions of what they are permitted to do, what resources they may access, and with which external parties they may interact | Underdefined boundaries enable scope creep and privilege escalation |

### Engineering Responsibilities for Governance

Governance at Layer 3 is not solely a management concern. Engineers carry specific technical responsibilities:

| Responsibility | Description | Technical Implementation |
|---|---|---|
| **Agent inventory** | Every agent in production must be registered with metadata covering identity, owner, protocol capabilities, resource access scope, and deployment environment | Registry service with mandatory registration at deploy time |
| **Capability boundary enforcement** | Tools an agent can call, agents it can delegate to, and data it can access must be explicitly defined and technically enforced — not documented but unenforced | Declarative capability manifests evaluated by the gateway PDP |
| **Audit trail completeness** | Every cross-boundary action must produce a tamper-evident, attributable audit record retained for a policy-defined period | Append-only audit log with cryptographic chaining |
| **Revocation readiness** | The ability to revoke an agent's credentials and terminate its active sessions at short notice is an operational requirement, not an edge case | SPIFFE SVID rotation + gateway session termination API |

> 💡 **Build governance infrastructure before scale forces it.** An agent inventory, capability boundary enforcement, and revocation capability are cheapest to implement at design time. Retrofitting them onto a production system with hundreds of deployed agents is substantially more expensive and operationally disruptive.

### Regulatory Trajectory

As of 2026, no jurisdiction has enacted comprehensive regulations specifically governing agent-to-agent communication across organizational boundaries. The EU AI Act, US executive orders on AI safety, and emerging sector-specific regulations (financial services, healthcare) impose obligations on AI systems generally, but their application to autonomous agent networks operating at Layer 3 is still being interpreted.

What is clear: **liability for agent actions travels with the instruction chain.** An organization whose agent initiates a cross-boundary action that causes harm retains legal risk even if the harm occurs in another organization's system.

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-GOV-001` | Every agent deployed to production MUST be registered in an agent inventory before it is permitted to initiate or receive cross-boundary interactions. |
| `L3-GOV-002` | Agent capability boundaries MUST be declarative, machine-readable, and evaluated by the gateway PDP — undocumented or unenforced capability boundaries do not satisfy this requirement. |
| `L3-GOV-003` | Audit trails for cross-boundary agent interactions MUST be tamper-evident, attributable to a specific agent identity, and retained for a minimum policy-defined period (default: 90 days; regulate-industry minimum may differ). |
| `L3-GOV-004` | Every agent MUST have an assigned human owner registered in the agent inventory — ownerless agents MUST NOT be permitted in production. |
| `L3-GOV-005` | Credential revocation and active session termination for any agent MUST be achievable within 15 minutes of a revocation decision — agents that cannot be rapidly revoked are security liabilities. |
| `L3-GOV-006` | Agent inventory, capability boundary definitions, and audit trail completeness MUST be verified at each deployment pipeline stage — governance cannot be a post-deployment activity. |

### Java + LangGraph Target

```java
@Component
public class AgentGovernanceEnforcer {
    private final AgentInventoryService inventory;           // L3-GOV-001
    private final CapabilityBoundaryEvaluator boundaries;    // L3-GOV-002
    private final TamperEvidentAuditLogger auditLog;         // L3-GOV-003
    private final RevocationService revocationService;       // L3-GOV-005

    // Called at agent deploy time — L3-GOV-001, L3-GOV-004, L3-GOV-006
    public void registerAgent(AgentRegistration registration) {
        requireNonNull(registration.humanOwner(), "Agent must have a human owner (L3-GOV-004)");
        inventory.register(registration);
        boundaries.registerCapabilityManifest(registration.agentId(),
                                               registration.capabilityManifest()); // L3-GOV-002
        auditLog.logRegistration(registration); // L3-GOV-003
    }

    // Called at runtime — checks every cross-boundary interaction
    public void assertPermitted(String agentId, CrossBoundaryOperation op) {
        AgentRecord record = inventory.lookup(agentId)
            .orElseThrow(() -> new UnregisteredAgentException(agentId)); // L3-GOV-001

        if (revocationService.isRevoked(agentId)) {
            throw new RevokedAgentException(agentId); // L3-GOV-005
        }
        boundaries.assertOperationPermitted(record, op); // L3-GOV-002
    }
}
```

---

## Section 22.6 — Assembling the Full Protocol Stack

### The Complete Stack

Every production agent system that participates in the Agent Web uses some or all of these layers:

| Layer | Protocol(s) | Function | 2026 Maturity |
|---|---|---|---|
| **Layer 1 — Tool Access** | MCP (Nov 2025 spec) | Agent ↔ tool communication; JSON-RPC 2.0; STDIO + Streamable HTTP | Production-stable; broad ecosystem adoption |
| **Layer 2 — Agent Collaboration** | A2A v1.0 (enterprise), ANP (open network) | Agent ↔ agent task delegation; eight-state lifecycle; three bindings | A2A: production-stable; ANP: specification stage |
| **Layer 3 — Agent Web** | Well-known endpoints, ANS, SPIFFE/SPIRE, VCs, OAuth Token Exchange, Agent Gateway | Discovery, cross-org trust, policy enforcement, governance | Partially available; three known gaps |

### The Well-Known Discovery Convergence

A2A Agent Cards (`.well-known/agent.json`) and MCP Server Cards (`.well-known/mcp.json`, 2026 roadmap) are converging under the same discovery convention:

```
https://agents.acme.com/.well-known/agent.json   → A2A Agent Card (what agents can do)
https://tools.acme.com/.well-known/mcp.json      → MCP Server Card (what tools expose)
```

The same crawlers, registries, and security scanners can index both. This makes the two discovery mechanisms **operationally unified** — one infrastructure serves both Layer 1 and Layer 2 discovery.

### Three Current Gaps Requiring Application-Layer Engineering

| Gap | Status | Engineering Response |
|---|---|---|
| **Cold-start discovery** | ANS is an IETF draft — no universal registry in production | Use well-known endpoints for known-domain discovery; document human-configuration fallback for cold-start; monitor ANS maturity |
| **Standardized multi-hop delegation** | OAuth Token Exchange provides the mechanism; composing it correctly across arbitrary org chains without pre-agreed trust anchors is still bespoke per deployment | Design explicit delegation token strategies before deploying chains > 2 hops (`L3-TRUST-006`) |
| **Mature governance tooling** | Requirements are clear; off-the-shelf tooling for automated agent inventory, policy-as-code for capability boundaries, and real-time session revocation is still maturing | Build to the `L3-GOV-001–006` rules using available primitives (SPIFFE, OPA, append-only audit stores); do not wait for turnkey tooling |

### Design Rules

| Rule ID | Rule |
|---|---|
| `L3-STACK-001` | MCP MUST be used for Layer 1 (tool access), A2A for Layer 2 (enterprise agent collaboration), and the Layer 3 infrastructure (well-known endpoints, gateway, trust primitives) for cross-boundary interactions — protocol substitutions at any layer break the composability guarantee. |
| `L3-STACK-002` | All three current gaps (cold-start discovery, multi-hop delegation, governance tooling) MUST be explicitly addressed in the architecture documentation — unaddressed gaps in production are vulnerabilities, not TODOs. |
| `L3-STACK-003` | The well-known endpoint convergence (Agent Cards + Server Cards) MUST be used as the unified discovery surface — separate discovery pipelines for agents and tools add operational complexity with no architectural benefit. |

---

## Section 22.7 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `L3-STRUCT-001` | Structure | Explicit Layer 3 strategy before any cross-org interactions |
| `L3-STRUCT-002` | Structure | All three structural shifts addressed independently |
| `L3-STRUCT-003` | Structure | Value ceiling of closed system documented as architectural constraint |
| `L3-DISC-001` | Discovery | Agent Card at `.well-known/agent.json` for every externally-facing agent |
| `L3-DISC-002` | Discovery | MCP Server Card at `.well-known/mcp.json` when spec stabilizes |
| `L3-DISC-003` | Discovery | Agent/Server Card metadata treated as security-sensitive artifacts |
| `L3-DISC-004` | Discovery | ANS supplementary only in 2026 — not sole discovery mechanism |
| `L3-DISC-005` | Discovery | Cold-start discovery explicitly handled in architecture |
| `L3-TRUST-001` | Trust | All four trust components used as layered stack |
| `L3-TRUST-002` | Trust | Signed Agent Card signatures verified before any external A2A interaction |
| `L3-TRUST-003` | Trust | SPIFFE/SPIRE for all workloads in cross-org interactions |
| `L3-TRUST-004` | Trust | Verifiable Credentials verified against issuer public key |
| `L3-TRUST-005` | Trust | OAuth Token Exchange (RFC 8693 + RAR RFC 9396) for cross-org delegation |
| `L3-TRUST-006` | Trust | Multi-hop delegation chains explicitly designed before chains > 2 hops |
| `L3-GW-001` | Gateway | Agent Gateway at every organizational trust boundary |
| `L3-GW-002` | Gateway | All six gateway functions implemented |
| `L3-GW-003` | Gateway | Hub-and-spoke default topology for 2026 production |
| `L3-GW-004` | Gateway | Tamper-evident audit logs in separate store |
| `L3-GW-005` | Gateway | Gateway is single point of revocation for external access |
| `L3-GW-006` | Gateway | Protocol translation preserves canonical A2A proto data model |
| `L3-GOV-001` | Governance | Agent inventory registration before any cross-boundary interaction |
| `L3-GOV-002` | Governance | Capability boundaries declarative, machine-readable, gateway-evaluated |
| `L3-GOV-003` | Governance | Tamper-evident audit trails, 90-day minimum retention |
| `L3-GOV-004` | Governance | Every agent has assigned human owner |
| `L3-GOV-005` | Governance | Revocation + session termination within 15 minutes |
| `L3-GOV-006` | Governance | Governance verified at each pipeline stage |
| `L3-STACK-001` | Full stack | MCP/A2A/Layer 3 at correct layers — no protocol substitution |
| `L3-STACK-002` | Full stack | All three current gaps explicitly addressed in architecture |
| `L3-STACK-003` | Full stack | Well-known convergence as unified discovery surface |

---

## Section 22.8 — Transition: From Protocol Stack to Operational System

Chapter 22 closes Part 5. The full three-layer protocol stack is now specified:

| Layer | Chapter | Status |
|---|---|---|
| Layer 1 — Tool Access (MCP) | Chapter 20 | Complete |
| Layer 2 — Agent Collaboration (A2A + ANP) | Chapter 21 | Complete |
| Layer 3 — The Agent Web | Chapter 22 | Complete |

The remaining parts of Volume 1 apply this protocol foundation to the operational, observability, and governance challenges that production systems face:

- **Chapter 23** — Observability across the protocol stack: tracing, metrics, and logging at each layer boundary
- **Chapter 33** — Enterprise integration: deploying the full stack inside regulated, compliance-governed environments
- **Chapter 34** — Regulatory compliance: mapping the three-layer stack to EU AI Act, sector-specific, and emerging agent-network regulations

| Concern | Protocol Layer That Surfaces It | Operational Chapter That Resolves It |
|---|---|---|
| End-to-end trace correlation across MCP + A2A + gateway | All three layers | Chapter 23 |
| Enterprise SSO integration at the Agent Gateway | Layer 3 (gateway) | Chapter 33 |
| Capability boundary policy-as-code | Layer 3 (governance) | Chapter 33 |
| Audit trail format for regulatory submission | Layer 3 (governance) | Chapter 34 |
| Agent inventory for AI Act Article 13 transparency | Layer 3 (governance) | Chapter 34 |

> The protocol stack composes cleanly when each layer addresses only its own concerns. The operational chapters that follow assume the stack is in place and build the production engineering layer on top of it.
