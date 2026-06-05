# Chapter 7 — The Action Layer: Tools & Delegation

**Part 2: Anatomy of an AI Agent**
**Volume 1 Specification — for Java + LangGraph implementation in Volume 2**

> *"Action is where the agent meets the world. It is also where the agent can break it."*

The action layer is the highest-risk layer of every agentic system. A model hallucination produces wrong text. A wrong action can delete data, trigger an external workflow, send a notification that cannot be unsent, or process a refund twice. That asymmetry is the central engineering fact that governs every design decision in this chapter.

This chapter specifies the complete action layer: the tool contract, the `ToolRegistry`, the eight-phase dispatch pipeline, safety classification, validation gates, idempotency, MCP wiring, delegation to sub-agents, and the full `ActNode` architecture.

---

## 7.1 — The Action Surface and the Least-Privilege Principle

### Context

An agent's action surface is the complete set of operations it can perform on the external world. The action surface is a security boundary, an audit boundary, and an operational cost boundary simultaneously. An agent with an unlimited action surface is an agent that can cause unlimited damage under adversarial input, hallucination, or misconfiguration.

### What (the spec)

**Action classification by side-effect class — four tiers:**

| Tier | Class | Examples | Reversibility | Autonomy permitted |
|---|---|---|---|---|
| **T0** | Read-only | Database queries, file reads, API GETs | Fully reversible | Yes — no confirmation required |
| **T1** | Idempotent write | Upserts, cache invalidation, idempotent API calls | Reversible with correct key | Yes — with idempotency key enforced |
| **T2** | Non-idempotent write | Create-new, charge, send message, post notification | Not reversible without compensating transaction | Conditional — requires plan-level confidence gate |
| **T3** | Destructive / irreversible | Delete, cancel order, initiate refund, send external communication | Irreversible | No — requires human approval gate |

**Least-privilege principle (three-part rule):**
1. Every agent instance is granted the **minimum set of tools** required for its declared task scope — not the full registry.
2. Every tool is granted the **minimum argument scope** required — an account-lookup tool receives only the account ID, never a full credentials bundle.
3. Every T3 action requires a **human-in-the-loop approval gate** before dispatch, regardless of model confidence.

**Permitted action surface document:** Before any agent goes to production, produce a document listing every tool it can invoke, its side-effect tier, and the justification for including it in that agent's scope. This document is a versioned engineering artifact.

### Why (engineering implications)

- An over-permissioned agent under prompt injection can call T3 tools with attacker-controlled arguments. Least-privilege is the primary structural defense against this.
- The side-effect tier determines retry policy, idempotency requirements, approval gate requirements, and audit log retention policy. Tools without a declared tier cannot be dispatched.
- The permitted action surface document is the primary artifact for security review and compliance audit. Agents deployed without one have no verifiable blast-radius boundary.

### How (system design rules)

1. **Declare the side-effect tier (`SideEffectClass`) on every `ToolContract`** at registration time. Undeclared-tier tools are rejected by `ToolRegistry` at registration.
2. **Enforce least-privilege scoping at the registry level**: `ToolRegistry` supports per-agent scope sets. An agent's `ActNode` is initialized with a scoped view of the registry — it cannot call tools outside its declared scope.
3. **Produce the permitted action surface document** as a required pre-production artifact. Store it in version control alongside the agent configuration.
4. **Never expand an agent's tool scope at runtime** based on model output. Scope changes require a configuration change and a re-deployment.
5. **In Java + LangGraph**: `ToolContract` is a Java record with fields `toolName`, `namespace`, `sideEffectClass`, `inputSchema`, `outputSchema`, `timeoutMs`, `idempotent`, and `requiresApproval`. `ToolRegistry` enforces scope via a `ScopedToolRegistry` wrapper that filters by a per-agent `Set<String> permittedTools`.

---

## 7.2 — The Tool Contract

### Context

A tool contract is the complete specification of what a tool does, what it accepts, what it returns, how it behaves under failure, and what side effects it produces. It is the interface contract between the agent's reasoning layer and the action layer. Every tool must have a contract before it can be registered.

### What (the spec)

**Mandatory tool contract fields:**

| Field | Type | Description |
|---|---|---|
| `toolName` | `String` | Globally unique, namespace-qualified (e.g., `crm.getContact`) |
| `namespace` | `String` | Logical grouping (e.g., `crm`, `payments`, `orders`) |
| `description` | `String` | LLM-facing description; precise enough to drive correct tool selection |
| `sideEffectClass` | `SideEffectClass` enum | T0 / T1 / T2 / T3 |
| `inputSchema` | `JsonSchema` | Full JSON Schema for all tool arguments |
| `outputSchema` | `JsonSchema` | Full JSON Schema for the success response |
| `errorEnvelope` | `ErrorEnvelopeSchema` | Standard error structure with `code`, `message`, `retryable` |
| `timeoutMs` | `int` | Maximum wall-clock time before `TOOL_TIMEOUT` |
| `idempotent` | `boolean` | Whether safe to retry without a deduplication key |
| `requiresApproval` | `boolean` | Whether T3 gate applies |
| `version` | `String` | Semantic version string (e.g., `v2.1.0`) |
| `deprecatedSince` | `Instant?` | If set, tool is in sunset; use `successorTool` instead |

**Tool description quality requirements (LLM-facing):**
- State **what** the tool does in one sentence.
- State **when** to use it (vs. similar tools) in one sentence.
- State **what it does not do** if ambiguity with a sibling tool exists.
- State the **key argument constraints** that are not obvious from the schema.

Poor description → wrong tool selection → wrong action. Tool descriptions are part of the system's operational correctness, not documentation.

**Contract versioning rule:**
- **Patch version** (`v2.1.x`): backward-compatible bug fix. No agent migration required.
- **Minor version** (`v2.x.0`): backward-compatible schema addition. Agents continue to work; new fields are optional.
- **Major version** (`vX.0.0`): breaking change. Old and new versions coexist in registry during migration window. Agents must be migrated before old version is deprecated.

### Why (engineering implications)

- The `description` field is the primary driver of the model's tool selection. A description that is too vague causes the model to call the wrong tool. A description that is too long crowds out context budget. Treating descriptions as production-quality operational text — not documentation afterthoughts — is the highest-return investment in tool layer quality.
- The `errorEnvelope` contract exists so that the agent runtime never receives an untyped exception from a tool. Every failure, regardless of cause, is wrapped in a standard envelope. This makes error handling in `ActNode` uniform and testable.
- Running two major versions simultaneously requires the registry to hold both `crm.getContact/v1` and `crm.getContact/v2`. Without explicit sunset dates and contract tests as migration guards, deprecated versions accumulate indefinitely and degrade tool selection quality.

### How (system design rules)

1. **Write the tool contract before writing the tool handler.** The contract is the specification; the handler is the implementation.
2. **Review all tool `description` fields** as part of every agent deployment review. Poor descriptions are production defects.
3. **Assign a `deprecatedSince` timestamp** at the moment a major version is released, not after migration is complete. Enforce removal of deprecated versions by that date.
4. **Contract-test every tool** against its declared `inputSchema`, `outputSchema`, and `errorEnvelope` on every CI run.
5. **In Java + LangGraph**: `ToolContract` is a Java record. `ToolContractLoader` reads contracts from versioned JSON files in the tool library at startup. `ContractTestSuite` is a parameterized JUnit 5 test that validates every registered contract against its schema and error envelope.

---

## 7.3 — The ToolRegistry

### Context

The `ToolRegistry` is the central data structure that holds all registered tool contracts and maps tool names to their execution handlers. It is the agent's entire action surface in one place. Every tool the agent can invoke passes through the registry. Tools that are not in the registry cannot be called — by design.

### What (the spec)

**Core registry operations:**

| Operation | Signature | Description |
|---|---|---|
| `register` | `(contract, handler) → void` | Register a tool; validates contract completeness; enforces no duplicate names |
| `deregister` | `(toolName) → void` | Remove a tool; in-flight calls are allowed to complete |
| `lookup` | `(toolName) → ToolContract?` | Retrieve contract by name; resolves aliases |
| `list` | `(filter?) → List<ToolContract>` | Enumerate available tools; filter by namespace, tier, or tag |
| `schemaFor` | `(toolName) → JsonSchema` | Return input schema for LLM prompt injection |
| `registerAlias` | `(alias, target) → void` | Register a name alias resolved at lookup time |

**Tool discovery via MCP:**
When tools are served by MCP servers, the `MCPClient` calls the `tools/list` endpoint at connection time and auto-populates the registry. New MCP servers plug in by advertising their tools via `tools/list` — the registry requires no prior knowledge.

**Capability hash polling:** Implement a `capabilityHash` on each MCP server's `tools/list` response. The `MCPClient` polls this hash at low frequency (configurable, default 60s) to detect tool set changes, triggering a selective registry update without fetching the full list on every heartbeat.

**Dynamic registration consistency rule:** When a tool the agent attempts to call has been deregistered mid-session (dynamic registry), `ToolRegistry.lookup` returns `null`. `ActNode` must handle this as a `TOOL_NOT_FOUND` error — not as an unhandled exception — and route to the `handleError` node.

### Why (engineering implications)

- Centralizing the registry gives a single place to audit, monitor, and govern the entire action surface. Agents that scatter tool registration across multiple modules have no auditable action surface.
- The `list` operation is used by the agent runtime to build the tool section of the system prompt. Passing all tool schemas to the LLM in every prompt is expensive and degrades selection accuracy when tool count is large. Dynamic tool selection strategies (based on `list` filtered by active subgoal) are the production solution.
- Aliases enable context-specific naming (different agent personas using different names for the same operation) and A/B routing (two handler implementations of the same contract for canary testing).

### How (system design rules)

1. **`ToolRegistry` is a singleton Spring bean** initialized at application startup. All tool registrations happen during the startup lifecycle, not at request time (except dynamic MCP discovery).
2. **Startup health check validates all alias targets** are registered before the agent accepts any requests.
3. **Prune deprecated tool versions aggressively.** Set sunset dates at major version release time; enforce removal. A registry that accumulates stale versions degrades tool selection quality over time.
4. **Limit tool schema injection to the active subgoal scope.** Use `list(filter: activeSubgoalNamespaces)` to select relevant tool schemas for each prompt assembly — not the full registry.
5. **In Java + LangGraph**: `ToolRegistry` is a `@Component`. `MCPClientAdapter` implements `ToolRegistryPopulator` and is called during `@PostConstruct`. `ScopedToolRegistry` is a per-agent wrapper initialized in `AgentConfig`. The LangGraph `ToolNode` (used for standard tool dispatch) is initialized with the scoped registry view.

---

## 7.4 — The Eight-Phase Dispatch Pipeline

### Context

Every tool call passes through a deterministic sequence of stages inside the action layer. Each stage is a failure point; each must be handled explicitly. The dispatch pipeline is the enforcement mechanism for all contracts, authorizations, validations, and audit requirements. Nothing bypasses it.

### What (the spec)

**Eight mandatory dispatch stages, in order:**

| # | Stage | Operation | Failure type | Failure routing |
|---|---|---|---|---|
| 1 | **Schema validation** | Validate arguments against `inputSchema` | `INVALID_ARGS` | Reject; return to `ReasonNode` for reformulation |
| 2 | **Authorization check** | Verify agent has permission for this tool with these arguments | `UNAUTHORIZED` / `FORBIDDEN` | Hard stop; escalate |
| 3 | **Approval gate** | For T3 tools: pause, emit `HumanApprovalRequest`, await approval | `APPROVAL_DENIED` | Abort action; log decision |
| 4 | **Idempotency key** | For T1+ tools: generate or verify deduplication key | `DUPLICATE_CALL` | Return cached result from prior identical call |
| 5 | **Pre-call audit log** | Write `ToolCallAuditRecord` with traceId, tool name, sanitized args, timestamp | — | Hard failure if audit write fails |
| 6 | **Dispatch** | Forward call to handler with deadline derived from `timeoutMs` | `TOOL_TIMEOUT` / `TOOL_UNAVAILABLE` | Retry policy by tier; circuit breaker |
| 7 | **Result validation** | Validate response against `outputSchema` | `INVALID_RESULT` | Wrap in error envelope; do not pass raw malformed result upstream |
| 8 | **Post-call audit log** | Write result status, latency, and outcome to audit log | — | Hard failure if audit write fails |

**Authorization check — three questions (all must pass):**
1. Is this agent allowed to call this tool at all? (Tool-level ACL)
2. Are these specific argument values within the agent's permitted scope? (Argument-level policy — e.g., agent may query only its own tenant's data)
3. Does the tool's side-effect class require additional confirmation for this calling context? (Risk-based step-up)

**Result envelope (mandatory standard structure):**
```json
// Success
{ "status": "success", "traceId": "t-abc123", "toolName": "payments.refund",
  "durationMs": 243, "result": { ... } }

// Error
{ "status": "error", "traceId": "t-abc123", "toolName": "payments.refund",
  "durationMs": 5001, "error": { "code": "TOOL_TIMEOUT", "message": "...", "retryable": false } }
```

The `retryable` flag is set by the dispatch layer based on error code and side-effect class. Read-only (T0) tools are retryable on `TOOL_TIMEOUT`. Write tools (T1+) are not retryable unless the tool contract explicitly declares `idempotent: true`.

### Why (engineering implications)

- The pre-call audit log (stage 5) must be written **before** dispatch (stage 6). An audit log written only on success is not an audit log — it is a success log. The pre-call record proves the call was attempted; the post-call record proves how it resolved.
- Schema validation at stage 1 prevents the single most common tool failure: the model generates a structurally valid-looking tool call with wrong field names, missing required fields, or incorrect types. Catching this before dispatch prevents a write tool from receiving garbage arguments.
- The approval gate (stage 3) is not optional for T3 tools. An agent that bypasses the approval gate for "efficiency" in development and adds it later in production will deploy it incorrectly the first time. Design it in from day one.

### How (system design rules)

1. **Implement the dispatch pipeline as a chain of eight `DispatchStage<I,O>` components** wired in fixed order. No stage is optional; all are configured, not conditionally included.
2. **Approval gate uses LangGraph's `interrupt()` mechanism** to pause the graph, write an `ApprovalRequest` to the `HITL_QUEUE`, and await a human `approve` / `deny` signal before resuming the dispatch chain.
3. **Idempotency keys are content-addressed:** SHA-256 of `(toolName + canonicalized arguments + sessionId)`. Store in a Redis-backed `IdempotencyStore` with TTL equal to the tool's `timeoutMs` × 10.
4. **Authorization policies are declared in OPA (Open Policy Agent)**, not hard-coded in tool handlers. `AuthorizationStage` calls the OPA policy engine per call.
5. **All audit writes go through `AuditService`**, which writes asynchronously to a write-ahead log. If `AuditService` is unavailable, the dispatch pipeline halts — audit is not optional.
6. **In Java + LangGraph**: `ToolDispatchPipeline` is a Spring `@Component` with eight injected `DispatchStage` beans. `ActNode` calls `ToolDispatchPipeline.dispatch(toolName, args, callingContext)` and receives `Either<ToolError, ToolResult>`. The LangGraph interrupt for HITL approval is handled by `HumanApprovalNode`.

---

## 7.5 — Error Classification and Retry Policy

### Context

Not all tool failures are equal. The dispatch layer must distinguish four error categories and apply the correct handling policy to each. A system that retries all failures uniformly will either fail to recover from transient errors or generate duplicate writes on non-idempotent tools — both are production disasters.

### What (the spec)

**Four error categories and their handling policies:**

| Category | Error codes | Retryable? | Agent action |
|---|---|---|---|
| **Validation error** | `INVALID_ARGS`, `SCHEMA_VIOLATION` | No — fix the call | Return to `ReasonNode`; inject error detail into next context |
| **Authorization error** | `UNAUTHORIZED`, `FORBIDDEN` | No — change credentials or policy | Hard stop; escalate to human reviewer |
| **Transient error** | `TOOL_TIMEOUT`, `TOOL_UNAVAILABLE`, `RATE_LIMITED` | Yes — on T0/T1 only | Exponential backoff with jitter; circuit breaker |
| **Tool error** | `TOOL_ERROR`, `NOT_FOUND`, `BUSINESS_RULE_VIOLATION` | No — application failure | Return result to agent; agent reasons about next step |

**Retry policy for transient errors (T0/T1 only):**

Delay before attempt `n`:
\[ d_n = \min(b \cdot 2^n + \text{Uniform}(0, j),\ d_{\max}) \]

Where `b` = base delay (default 100ms), `j` = jitter ceiling (default 50ms), `d_max` = max delay (default 30s).

- **Maximum retry count** is per-tool configuration (`maxRetries`, default 3 for T0, 1 for T1).
- **T2 and T3 tools are never automatically retried.** A failed T2/T3 call returns the error envelope to the agent; the agent reasons about whether to attempt a compensating action.

**Circuit breaker (mandatory for all external tool backends):**

| State | Condition | Behavior |
|---|---|---|
| **Closed** | Normal | All calls pass through |
| **Open** | Failure rate > threshold over rolling window | All calls fail immediately with `CIRCUIT_OPEN`; no backend calls |
| **Half-open** | After `resetTimeout` | One probe call permitted; success → Closed; failure → Open |

Default thresholds: open at 50% failure rate over 10 calls in 60s; `resetTimeout` = 30s. Thresholds are per-tool configuration.

### Why (engineering implications)

- Retrying a non-idempotent T2 write tool under `TOOL_TIMEOUT` produces duplicate side effects. This is one of the fastest ways to turn a tool transient failure into a production incident (e.g., two refunds issued, two emails sent).
- The circuit breaker prevents a degraded backend from cascading across the entire agent loop. Without it, one slow tool monopolizes the session budget with repeated timeouts.
- Returning tool errors to the agent (rather than throwing exceptions) is a deliberate design: the agent can reason about a `NOT_FOUND` result and decide to try a different approach. Throwing an exception collapses that reasoning opportunity.

### How (system design rules)

1. **Implement retry policy per tool, not globally.** `ToolContract.retryPolicy` carries `maxRetries`, `baseDelayMs`, `jitterMs`, and `maxDelayMs`. T2/T3 tools have `maxRetries=0`.
2. **Implement `CircuitBreaker` as a per-tool-backend component**, not per-call. Share state across all calls to the same backend within the same JVM instance.
3. **Never retry a call that failed at stage 1 (schema validation) or stage 2 (authorization).** These are deterministic failures — retrying them wastes budget.
4. **Log every retry attempt** with attempt number, delay, and error code. Circuit breaker state transitions (`CLOSED→OPEN`, `OPEN→HALF_OPEN`, `HALF_OPEN→CLOSED`) are emitted as structured audit events.
5. **In Java + LangGraph**: `RetryPolicy` is a per-tool `@ConfigurationProperties` bean. `CircuitBreakerRegistry` (Resilience4j) provides per-tool circuit breakers. `DispatchStage` for dispatch (stage 6) wraps handler calls in `CircuitBreaker.decorateCheckedSupplier`. Retry is applied at the `ActNode` level, outside the dispatch pipeline, using a `RetryTemplate`.

---

## 7.6 — Idempotency and Exactly-Once Semantics

### Context

Idempotency is the property that calling a tool multiple times with the same arguments produces the same effect as calling it once. It is the enabling condition for safe retry on T1 tools and the primary defense against duplicate side effects from network-level retries, agent loop restarts, and checkpoint replay.

### What (the spec)

**Idempotency key generation (mandatory for T1+ tools):**

- Key = `SHA-256(toolName + canonicalizedArguments + sessionId + intentId)`
- `intentId` is assigned by the `ReasonNode` to each tool call it generates. It is stable across retries of the same logical intent.
- Canonical argument representation: sort object keys alphabetically; normalize numeric precision; strip whitespace.

**Idempotency store contract:**
- Backend: Redis with TTL = `tool.timeoutMs × 10` (minimum 60s)
- On hit (key exists): return the stored result immediately; skip all dispatch stages 3–8
- On miss (key absent): proceed with dispatch; store result at stage 8

**Exactly-once delivery for T3 (irreversible) tools:**
- T3 tools require **idempotency key + human approval + pre-commit audit record** before dispatch
- The pre-commit audit record is written to the write-ahead log before the approval gate is opened
- If the agent crashes between approval and dispatch, replay uses the pre-commit record to skip re-soliciting approval

**Idempotency is not the same as safety:**
- An idempotent T2 tool is safe to retry. A non-idempotent T2 tool is not, even if it carries an idempotency key at the infrastructure level (e.g., Stripe's idempotency-key header). **Declare idempotency in `ToolContract.idempotent` only when the tool handler verifies and stores the key end-to-end.**

### Why (engineering implications)

- The `intentId` that travels from `ReasonNode` to `ActNode` is the bridge between the agent's reasoning intent and the tool call's idempotency key. Without it, retries of the same intent generate different keys because session timestamps differ.
- Idempotency store TTL must be long enough to cover the maximum realistic retry window. A TTL of 60s is too short for a tool with a 30s timeout and three retries with exponential backoff.
- Exactly-once semantics for T3 tools are a compliance requirement in regulated domains. The pre-commit audit record is the evidence that the action was authorized before it was executed, even if the execution happened in a separate JVM restart.

### How (system design rules)

1. **`ReasonNode` assigns an `intentId`** (UUID) to each tool call in its output. `ActNode` uses this `intentId` as part of the idempotency key computation.
2. **`IdempotencyStore` is a Redis-backed Spring bean** with per-tool TTL configuration. Cache hits bypass the dispatch pipeline entirely — they are logged as `IDEMPOTENCY_HIT` events.
3. **T3 pre-commit audit records** are written to `AuditService` before the `HumanApprovalNode` is entered. The record carries `approvalStatus=PENDING` and is updated to `APPROVED` or `DENIED` after the HITL decision.
4. **Never mark a tool as `idempotent: true` in `ToolContract`** unless the handler implementation verifies the key against its own store and returns the prior result on a duplicate.
5. **In Java + LangGraph**: `IntentId` is a field in `AgentState.pendingToolCall`. `DispatchStage` for idempotency (stage 4) calls `IdempotencyStore.checkOrReserve(key)`. On `DUPLICATE_CALL`, the stage returns the cached `ToolResult` and exits the pipeline at stage 4.

---

## 7.7 — MCP as the Action Wiring Standard

### Context

The Model Context Protocol (MCP) defines a vendor-neutral interface through which agents discover and invoke tools and data sources. MCP transforms tool integration from bespoke wiring into a plug-in architecture: any process that implements the MCP server protocol becomes a first-class tool provider. The `MCPClient` handles discovery, version negotiation, transport abstraction, and streaming results — the agent runtime interacts only with tool names and schemas.

### What (the spec)

**MCP connection lifecycle (five steps, mandatory in order):**

1. **Transport establishment** — open stdio pipe (local) or HTTP/SSE connection (remote)
2. **Initialize handshake** — client sends `initialize` with protocol version and capability set; server responds with its version and capabilities; both sides agree on the intersection
3. **Tool discovery** — client calls `tools/list`; server returns array of tool definitions; client registers them in `ToolRegistry`
4. **Steady state** — connection ready for `tools/call` requests; client may also receive `tools/list/changed` server-initiated notifications
5. **Shutdown** — either side may terminate; client handles mid-session disconnection gracefully; reconnects before failing in-flight calls

**Two transport modes:**

| Transport | Use case | Notes |
|---|---|---|
| **stdio** | Local tools (same machine, child process) | Low latency; process lifecycle managed by client |
| **HTTP/SSE** | Remote tools (network, container, cloud) | Supports streaming results; requires stable endpoint |

**MCP tool call flow:**
- Client serializes arguments to JSON-RPC `tools/call` request
- Server executes handler and returns JSON-RPC response
- Client deserializes and validates against `outputSchema`
- Result is returned through the standard `ToolDispatchPipeline` result path

**Streaming results (SSE):** For long-running tools, MCP supports progressive result streaming over SSE. `ActNode` receives partial results and can update `AgentState.lastToolObservation` incrementally, allowing the reasoning layer to begin processing before the tool completes.

### Why (engineering implications)

- MCP's `tools/list` mechanism turns external tool servers into self-describing capability providers. The `ToolRegistry` does not need to be pre-configured with tool contracts — they are negotiated at connection time. This is the enabling condition for dynamic tool discovery in multi-tenant and plugin-based deployments.
- The capability hash optimization (polling `tools/list` hash instead of the full list) reduces discovery overhead by two orders of magnitude for registries with hundreds of tools.
- Graceful reconnection is a non-optional requirement. An MCP server restart mid-session must not cause unrecoverable `TOOL_NOT_FOUND` errors — the `MCPClient` must reconnect and re-populate the registry transparently.

### How (system design rules)

1. **Implement `MCPClient` as a Spring `@Component`** with configurable transport (stdio / HTTP/SSE), `connectTimeout`, `readTimeout`, and `maxReconnectAttempts`.
2. **Compute and cache `capabilityHash`** on every `tools/list` response. Poll the hash at 60s intervals (configurable). On hash change, fetch the full `tools/list` and perform a diff-based registry update.
3. **Handle `tools/list/changed` server notifications** as an immediate registry refresh trigger (do not wait for the next polling interval).
4. **All MCP tool calls are dispatched through the same `ToolDispatchPipeline`** as in-process tools. MCP is a transport layer, not a bypass of the dispatch pipeline.
5. **In Java + LangGraph**: `MCPClientAdapter` implements `ToolRegistryPopulator`. It is called during `@PostConstruct` for each configured MCP server endpoint. `MCPToolHandler` wraps the `MCPClient.toolsCall()` method and implements the `ToolHandler` interface registered in `ToolRegistry`.

---

## 7.8 — Delegation: Agent-to-Agent Actions

### Context

Not all actions are tool calls. An agent can delegate a subtask to another agent — a specialized sub-agent, a parallel worker, or a peer in a multi-agent system. Delegation is an action with higher complexity, higher risk, and fewer established engineering patterns than tool invocation. It is also the fastest-growing action type in 2026 production deployments.

### What (the spec)

**Three delegation patterns:**

| Pattern | Structure | Use case | Risk |
|---|---|---|---|
| **Subgraph delegation** | Parent graph calls a compiled child `StateGraph` | Isolated subtask with private state; parent waits for result | Low — child state is isolated; parent controls resume |
| **Supervisor delegation** | Supervisor node routes to specialized worker agents | Task fan-out to domain specialists; results aggregated by supervisor | Medium — coordination overhead; shared state conflict |
| **Peer delegation (A2A)** | Agent sends task message to peer agent via A2A protocol | Deliberation, collaboration, parallel independent workstreams | High — peer has autonomous decision-making; trust boundary required |

**Delegation contract (mandatory for all patterns):**

Every delegation act must specify:
- `delegateTarget` — the sub-agent ID or subgraph name
- `taskDescription` — a precise, bounded task statement (not the full parent goal)
- `inputContext` — the minimum necessary context for the sub-agent (not the full parent state)
- `outputSchema` — the expected result structure the parent will validate against
- `timeout` — maximum wall-clock time before delegation is cancelled
- `fallbackPolicy` — what the parent does if delegation fails: `ABORT`, `RETRY`, `PROCEED_WITHOUT`, `ESCALATE`

**Delegation isolation rule:** Sub-agents receive only the `inputContext` explicitly passed to them — never a reference to the parent's `AgentState`. Cross-agent state contamination is a hard architectural violation.

**A2A protocol integration:**
- Delegation via A2A follows the same `ToolDispatchPipeline` as tool calls — `delegateToAgent` is a registered T2 tool in the `ToolRegistry`
- A2A task messages carry `taskId`, `parentRunId`, `inputContext`, and `outputSchema`
- The A2A response is validated against `outputSchema` before being injected into the parent's `AgentState`

### Why (engineering implications)

- Subgraph delegation is the lowest-risk form of delegation because the child graph is compiled, its state is isolated, and the parent's `StateGraph` retains full control over resume. Use it as the default for subtask decomposition.
- Supervisor delegation introduces shared state coordination risk. Two workers updating the same field in a shared `AgentState` without optimistic locking produces race conditions that manifest as inconsistent agent behavior, not errors.
- Peer delegation (A2A) is the highest-risk pattern because the peer has autonomous decision-making authority. Trust boundary enforcement — the peer cannot read or write the parent's memory, and the parent validates the peer's output before acting on it — is non-negotiable.

### How (system design rules)

1. **Default to subgraph delegation** for all subtask decomposition. Use supervisor delegation only when tasks genuinely require parallel domain specialists. Use peer A2A delegation only when deliberation or independent autonomous execution is required.
2. **Enforce the delegation isolation rule at compile time**: `SubgraphBuilder` does not accept a parent `AgentState` reference — only an explicitly typed `DelegationInput` record.
3. **Register `delegateToAgent` as a T2 tool** in the `ToolRegistry` with a strict `inputSchema` enforcing the delegation contract fields.
4. **Validate all delegation results** against `outputSchema` in the dispatch pipeline before injecting into parent state. A sub-agent that returns malformed results must produce a `INVALID_RESULT` error, not a silent state corruption.
5. **Set delegation timeouts aggressively.** A sub-agent that hangs holds the parent's execution budget. Default delegation timeout = 3× the expected single-step latency.
6. **In Java + LangGraph**: Subgraph delegation uses `CompiledGraph.invoke(delegationInput)`. Supervisor delegation uses a `SupervisorNode` that maintains a `workerResultMap` in `AgentState`. A2A delegation uses `A2AClient.sendTask(taskMessage)` wrapped in `A2AToolHandler`, registered as a T2 tool.

---

## 7.9 — The `ActNode` Architecture

### Context

All action layer components compose into a single LangGraph node: `ActNode`. This node is the structural boundary between the agent's internal state and the external world. Every action the agent takes passes through `ActNode`. Nothing bypasses it. `ActNode` is stateless across cycles — it reads its inputs from `AgentState.pendingToolCall` and writes its outputs to `AgentState.lastToolObservation`.

### What (the spec)

**`ActNode` execution sequence (seven steps):**

1. **Read** pending tool call from `AgentState.pendingToolCall` (set by `ReasonNode`)
2. **Resolve** tool name in `ScopedToolRegistry` — reject `TOOL_NOT_FOUND` immediately
3. **Classify** side-effect tier — select dispatch pipeline variant (T0/T1/T2/T3)
4. **Dispatch** via `ToolDispatchPipeline` (eight stages, Section 7.4)
5. **Receive** `Either<ToolError, ToolResult>` from pipeline
6. **Write** result to `AgentState.lastToolObservation` with `toolName`, `status`, `result`/`error`, `durationMs`, `traceId`
7. **Emit** `ActionTrace` (OTel span + structured log) — unconditionally on success and failure

**`ActNode` failure routing:**

| Failure type | Source | Routing |
|---|---|---|
| `TOOL_NOT_FOUND` | Step 2 | → `handleError` node |
| `INVALID_ARGS` | Pipeline stage 1 | → `reasonNode` (reformulate call) |
| `UNAUTHORIZED` | Pipeline stage 2 | → `handleError` node (escalate) |
| `APPROVAL_DENIED` | Pipeline stage 3 | → `reasonNode` (choose alternative) |
| `TOOL_TIMEOUT` / `CIRCUIT_OPEN` | Pipeline stage 6 | → `reflectNode` (assess impact) |
| `TOOL_ERROR` | Pipeline stage 6–7 | → `reflectNode` (agent reasons about failure) |

**`ActionTrace` contents (mandatory per invocation):**
- `runId`, `stepId`, `cycleId`
- Tool name, namespace, side-effect tier
- Dispatch pipeline stage outcomes (pass/fail per stage)
- Approval status (T3 only)
- Idempotency status (hit/miss/reserved)
- Result status and error code if applicable
- Total `ActNode` latency broken down by pipeline stage

### Why (engineering implications)

- `INVALID_ARGS` routing back to `ReasonNode` (not `handleError`) is intentional: the agent can reformulate the tool call with corrected arguments. Routing it to `handleError` would terminate the task for a recoverable error.
- `TOOL_ERROR` routing to `ReflectNode` (not `handleError`) is intentional: application-level tool errors (e.g., `NOT_FOUND`, `BUSINESS_RULE_VIOLATION`) are information for the agent's reasoning, not system failures. The agent should decide its next step.
- `ActionTrace` written unconditionally is the single most important observability investment in the action layer. Every production incident involving a wrong action, a duplicate action, or a missing action is diagnosable in minutes from a complete `ActionTrace` log — and takes hours without one.

### How (system design rules)

1. **`ActNode` is the only node that calls `ToolDispatchPipeline`.** No other node invokes tools directly.
2. **`ActNode` is stateless.** It reads from `AgentState.pendingToolCall` and writes to `AgentState.lastToolObservation`. It holds no instance state between cycles.
3. **Route failures by error type, not by exception type.** `ActNode` catches all `ToolError` variants from the pipeline and routes to the correct next node via LangGraph conditional edges.
4. **Emit `ActionTrace` before writing to `AgentState`** — the trace write is not contingent on the state write succeeding.
5. **In Java + LangGraph**: `ActNode` implements `NodeAction<AgentState>`. It is registered as the `act` node. Conditional edges from `act` are defined in `GraphBuilder`: `INVALID_ARGS → reason`, `UNAUTHORIZED → handleError`, `APPROVAL_DENIED → reason`, `TOOL_ERROR → reflect`, `SUCCESS → reflect`. `ActionTrace` is written via `AuditService` and emitted as an OTel span using `Tracer.spanBuilder("act.tool-dispatch")`.

---

## 7.10 — Action Layer Security

### Context

The action layer is the primary attack surface for prompt injection, tool-result poisoning, and privilege escalation in production agents. Security at this layer is not a policy document — it is a set of architectural controls enforced at the dispatch pipeline level, not at the prompt level.

### What (the spec)

**Four mandatory security controls at the action layer:**

| Control | Mechanism | What it prevents |
|---|---|---|
| **Least-privilege scoping** | `ScopedToolRegistry` per agent instance | Agent cannot call tools outside its declared scope |
| **Argument-level policy** | OPA policy check in dispatch stage 2 | Agent cannot pass out-of-scope values (e.g., another tenant's account ID) |
| **Taint tracking** | `TaintLevel` field on argument values derived from untrusted inputs | Hostile-tainted values blocked from T2/T3 tools without human review |
| **Chain depth limit** | `ToolChainContext.MAX_DEPTH` enforced per session | Prevents unbounded recursive tool chaining; default MAX_DEPTH = 10 |

**Taint tracking policy:**
- Inputs classified as `TaintLevel.HOSTILE` (from untrusted external sources, prompt injection candidates) cannot be passed to T2 or T3 tools without explicit human review approval.
- `TaintLevel` is assigned by `PerceiveNode` based on origin class and trust tier (Section 6.1).
- `TaintLevel` propagates through argument construction: if a T2 tool argument is derived from a hostile-tainted field, the argument inherits `HOSTILE` taint.

**Output filtering (mandatory for all tool results before `AgentState` write):**
- Strip PII from tool results before injection into working memory (if configured for the tool's namespace)
- Validate that no tool result contains embedded instruction patterns (`{{`, `<instruction>`, `SYSTEM:`) that could constitute prompt injection via tool output
- Results failing output filtering are logged as `OUTPUT_FILTER_VIOLATION` and returned to the agent as a `TOOL_ERROR`

### Why (engineering implications)

- Prompt injection via tool results is qualitatively different from prompt injection via user input. The agent trusts its own tool results more than user input — a hostile payload in a tool result is more likely to override system instructions than a hostile payload in the original message.
- Chain depth limits are a denial-of-service defense, not a correctness concern. An agent instructed (via prompt injection) to call ToolA → ToolB → ToolA recursively will exhaust the session budget in seconds without a chain depth guard.
- Argument-level policy in OPA is more reliable than argument validation in application code because OPA policies are externalized, versioned, testable, and auditable independently of the agent codebase.

### How (system design rules)

1. **Implement taint tracking as a `TaintContext` object** passed alongside `AgentState`. `PerceiveNode` initializes `TaintContext` with taint levels per input field. `ActNode` checks `TaintContext` before dispatch stage 2.
2. **Deploy OPA as a sidecar** (or call the OPA REST API) for argument-level policy evaluation. Policy bundles are versioned and deployed independently of the agent.
3. **Implement output filtering as a mandatory `OutputFilterStage`** added as stage 7.5 in the dispatch pipeline (between result validation and post-call audit log).
4. **`ToolChainContext` is maintained per session** in `AgentState.toolChainContext`. `ActNode` increments depth on every tool call and checks against `MAX_DEPTH` before dispatch stage 1.
5. **In Java + LangGraph**: `TaintContext` is a field in `AgentState`. `OPAAuthorizationStage` calls `OPAClient.evaluate(policy, {agent, tool, args, taintContext})`. `OutputFilterService` runs regex + embedding-based instruction-pattern detection. `ToolChainContext` throws `ChainDepthExceededException` at `MAX_DEPTH`, routing to `handleError`.

---

## Key Takeaways for Volume 2 Implementation

- **The action layer is the highest-risk layer.** A model hallucination produces wrong text. A wrong action produces real-world side effects. Every design decision follows from this asymmetry.
- **Side-effect classification (T0–T3)** drives retry policy, idempotency requirements, approval gate requirements, and audit log retention. Declare it on every `ToolContract`.
- **The `ToolContract`** is the interface specification between reasoning and action. The `description` field is production-quality operational text that directly drives tool selection quality.
- **The eight-phase dispatch pipeline** (schema validate → authorize → approve → idempotency → pre-audit → dispatch → result validate → post-audit) is the enforcement point for all contracts, authorizations, and audit requirements. Nothing bypasses it.
- **Retry policy is per-tool and per-tier.** T2/T3 tools are never automatically retried. Circuit breakers protect external backends from cascade failure.
- **Idempotency keys** are content-addressed and session-scoped. The `intentId` from `ReasonNode` is the stable cross-retry identity.
- **MCP is the standard wiring protocol.** All external tool servers are MCP servers. The `MCPClient` handles discovery, transport, and reconnection.
- **Delegation is a typed action** with a mandatory contract (target, task description, input context, output schema, timeout, fallback policy). Subgraph delegation is the default; A2A peer delegation is the highest-risk pattern.
- **`ActNode` is stateless, singular, and the only action entry point.** All failures route by error type to the correct next node via conditional edges.
- **Security controls** (least-privilege scoping, argument-level OPA policy, taint tracking, chain depth limit, output filtering) are architectural — not prompt-level instructions.
