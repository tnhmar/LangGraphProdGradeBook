# Chapter 32 Spec — The Tool Abstraction Layer and MCP Integration

> **Volume 1 reference:** Part 7 — Tooling and Integrations, Chapter 32
> **Part:** 7 — Tooling and Integrations
> **Depends on:** chap20 (`MCP-ARCH-001–005`, `MCP-SEC-001–005`), chap31 (framework selection matrix)
> **Java + LangGraph implementation target:** Volume 2 — Chapter 8: Tool Abstraction Layer; Chapter 9: MCP Client Integration
> **Bridges to:** chap33 (Enterprise Systems Integration), chap34 (Advanced Tool Patterns)

> *"A system is reliable not because its components never fail, but because it knows exactly what each component promises — and holds it to that promise."*
> — Attributed to distributed systems folklore

---

## Chapter Purpose

Every agent that acts on the world does so through tools. Left unmanaged, tool integrations accumulate as ad-hoc, bespoke wiring — fragile, untestable, and impossible to reason about at scale. Chapter 32 builds the **Tool Abstraction Layer** from first principles: a uniform contract between the agent's reasoning loop and every capability it can invoke.

The chapter then shows how the **Model Context Protocol (MCP)** operationalizes this abstraction as a wiring standard for production agent systems. Chapter 20 established MCP as a protocol specification; here the focus shifts to implementation — how you engineer reliable, secure, and testable tool integrations on top of that foundation.

By the end of this chapter the reader has a concrete mental model of:
- Tool contracts and the Tool Registry
- The dispatch lifecycle (7-stage pipeline)
- Error classification, retry math, and circuit breakers
- Security boundaries (authorization, sanitization, output filtering)
- MCP connection lifecycle and transport selection
- A three-level testing strategy

---

## Section 32.1 — Why Tools Need an Abstraction Layer

### The Direct Integration Problem

The simplest way to give an agent a tool is to write a function and let the agent call it. This works in a prototype. It breaks in production. Without a shared contract layer, each tool integration answers the same questions differently:

- How are arguments validated?
- What does an error look like?
- Who handles timeouts?
- What happens if the tool is temporarily unavailable?

Each developer solving these questions independently produces a system where every tool behaves differently, every error surfaces differently, and every failure mode is a surprise. The deeper problem is compositional: agents compose tools in sequences and parallel branches. When each tool in a chain has its own error semantics, reasoning about the composed behavior becomes intractable.

> ⚠️ **Agents are not forgiving of inconsistent tool behavior.** When tool A returns an empty list on failure but tool B throws an exception, the agent's reasoning loop must contain explicit special-casing for each. This couples tool implementation decisions to agent logic — a maintenance trap that compounds with every new tool added.

### What an Abstraction Layer Provides

A tool abstraction layer is the subsystem that sits between the agent's action dispatch and the actual tool implementations. Its five responsibilities:

| Responsibility | What It Enforces |
|---|---|
| **Contract enforcement** | Every tool declares inputs, outputs, and failure modes in machine-readable schema; layer validates before dispatch |
| **Lifecycle management** | Timeout, retry, and cancellation handled uniformly — not inside each tool |
| **Error normalization** | All failures translated into a standard error envelope before returning to the agent |
| **Security enforcement** | Authorization, input sanitization, output filtering run consistently for every call |
| **Observability** | Every dispatch, result, and error logged and measured with a trace ID |

The agent runtime is a **consumer** of this layer. It calls `toolLayer.invoke(name, args, context)` and receives either a result or a structured error. What happens inside the layer — retry logic, MCP transport, schema validation — is invisible to the reasoning loop. That separation is the point.

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-ABS-001` | The tool abstraction layer MUST be implemented as an explicit architectural component with a defined API — not as inline glue code scattered across the agent runtime. |
| `TOOL-ABS-002` | The agent reasoning loop MUST interact with tools only through the abstraction layer — direct tool function calls from the reasoning loop bypass all reliability and security guarantees. |
| `TOOL-ABS-003` | The abstraction layer MUST present a uniform call interface regardless of how the tool is implemented: in-process function, MCP server (stdio or HTTP), or direct HTTP endpoint. |

---

## Section 32.2 — Anatomy of a Tool Contract

### The Contract as a Specification

A tool contract is the **complete specification** of what a tool does, what it needs, what it returns, and how it can fail. It is the single source of truth used by the abstraction layer for validation, by the agent runtime for invocation, and by the LLM for tool selection and argument generation.

A complete contract has six mandatory fields:

| Field | Type | Purpose |
|---|---|---|
| `name` | string | Unique, stable, namespaced identifier (e.g., `crm.getContact`) — no collisions in multi-tool environments |
| `description` | string | Precise natural-language summary **written for the LLM** — primary signal the model uses to decide whether to invoke |
| `inputSchema` | JSON Schema | Argument names, types, constraints, required fields — validated by abstraction layer before every dispatch |
| `outputSchema` | JSON Schema | Structure of a successful response — used for result validation after dispatch |
| `sideEffects` | enum | `read-only` / `write` / `destructive` — drives authorization policy and retry safety decisions |
| `operationalParams` | object | `timeoutMs`, `maxRetries`, `idempotent` flag, `fallback` tool name, auth requirements |

### Side-Effect Classification and Its Consequences

The `sideEffects` field is not cosmetic. It is the primary input to two critical runtime decisions:

**Retry safety:**
- `read-only` tools may be retried freely on transient failure — calling a search API twice does not corrupt state
- `write` tools that are NOT explicitly `idempotent: true` MUST NOT be retried without confirmation that the first call did not succeed
- `destructive` tools MUST NOT be retried automatically and MAY require human confirmation (see chap12 HITL rules)

**Authorization:**
- Most enterprise environments apply different access control policies to read vs. write operations
- Routing authorization decisions through the side-effect classification enables consistent policy enforcement at the abstraction layer

> ⚠️ **Marking a write tool as `read-only` to bypass retry restrictions is a common shortcut that causes data corruption.** Treat the `sideEffects` field as a safety constraint, not a performance hint. The abstraction layer MUST reject any tool registration where the declared classification contradicts observed behavior in integration tests.

### Tool Contract Example (JSON)

```json
{
  "name": "kb.search",
  "description": "Search the internal knowledge base for documents relevant to a query. Use this tool when the user asks about company policies, product documentation, or internal procedures.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search query, expressed as a natural language question or keyword phrase.",
        "maxLength": 512
      },
      "topK": {
        "type": "integer",
        "description": "Number of results to return.",
        "default": 5,
        "minimum": 1,
        "maximum": 20
      }
    },
    "required": ["query"]
  },
  "outputSchema": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "docId": { "type": "string" },
        "title": { "type": "string" },
        "excerpt": { "type": "string" },
        "score": { "type": "number" }
      }
    }
  },
  "sideEffects": "read-only",
  "timeoutMs": 5000,
  "maxRetries": 2
}
```

> 💡 **Write tool descriptions from the LLM's perspective.** The model reads the `description` field to decide *when* to call the tool and *what* to put in the arguments. A description that explains the tool's purpose, the situations that warrant its use, and any constraints on its arguments produces dramatically better selection accuracy than a one-liner like "searches documents."

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-CONTRACT-001` | Every tool registered in the abstraction layer MUST have a complete contract with all six mandatory fields — partial contracts (missing output schema or sideEffects) MUST be rejected at registration time. |
| `TOOL-CONTRACT-002` | Tool names MUST use namespaced dot-notation (e.g., `crm.getContact`, `kb.search`) to prevent collisions in multi-tool environments. |
| `TOOL-CONTRACT-003` | Tool descriptions MUST be written for the LLM reader, not for human developers — the description is the LLM's primary tool-selection signal. |
| `TOOL-CONTRACT-004` | The `sideEffects` field MUST be validated against the tool's observed behavior in contract tests — misclassification is a reliability and safety defect. |
| `TOOL-CONTRACT-005` | `destructive` tools MUST declare a `requiresConfirmation: true` flag and MUST NOT be registered without a corresponding HITL confirmation gate (see chap12). |

### Java + LangGraph Target

```java
// Tool contract as an immutable value object
@Value.Immutable
public interface ToolContract {
    String name();
    String description();
    JsonSchema inputSchema();
    JsonSchema outputSchema();
    SideEffectClass sideEffects();    // READ_ONLY, WRITE, DESTRUCTIVE
    long timeoutMs();
    int maxRetries();
    boolean idempotent();
    Optional<String> fallbackTool();

    // TOOL-CONTRACT-005: destructive tools must declare confirmation requirement
    default boolean requiresConfirmation() {
        return sideEffects() == SideEffectClass.DESTRUCTIVE;
    }
}

// Registration-time validation (TOOL-CONTRACT-001)
public class ToolContractValidator {
    public void validate(ToolContract contract) {
        Preconditions.checkArgument(contract.name().contains("."),
            "Tool name must be namespaced: %s", contract.name());    // TOOL-CONTRACT-002
        Preconditions.checkArgument(!contract.description().isBlank(),
            "Tool description required for LLM selection");           // TOOL-CONTRACT-003
        Preconditions.checkNotNull(contract.inputSchema(),
            "inputSchema required");
        Preconditions.checkNotNull(contract.outputSchema(),
            "outputSchema required");
        if (contract.sideEffects() == SideEffectClass.DESTRUCTIVE) {
            Preconditions.checkArgument(contract.requiresConfirmation(),
                "Destructive tools must declare requiresConfirmation");  // TOOL-CONTRACT-005
        }
    }
}
```

---

## Section 32.3 — Tool Registration and Discovery

### Static vs. Dynamic Registration

Tool registration is the process by which tools become known to the abstraction layer. Two approaches, often used together:

| Approach | When to Use | Risk |
|---|---|---|
| **Static** | Fixed tool set known at deploy time; simple, predictable, easy to audit | None — the right default for most deployments |
| **Dynamic** | Tool availability changes at runtime (user-connected integrations, MCP server plug-in) | Consistency risk: LLM-prompted tools may differ from currently registered tools mid-session |

> ⚠️ **Dynamic tool registration introduces a consistency risk** — the set of tools the LLM was prompted with at the start of a conversation may differ from the set available when a particular tool call is dispatched. The abstraction layer MUST handle the case where a tool the agent attempts to call has been deregistered mid-session.

### The Tool Registry

The Tool Registry is the data structure inside the abstraction layer that holds all registered tool contracts and maps tool names to their execution handlers. Minimum API:

```
register(contract, handler)      — add a tool
deregister(name)                 — remove a tool
lookup(name)                     — retrieve a contract by name
list(filter?)                    — enumerate tools, optionally filtered by sideEffects or namespace
schemaFor(name)                  — return the input schema for LLM prompt injection
```

The `list` operation is used by the agent runtime to build the tool section of the system prompt. When tool count is large, passing all tool schemas to the LLM in every prompt is expensive and degrades selection accuracy. Chapter 34 addresses dynamic tool selection strategies that solve this at scale.

### Tool Discovery via MCP

When tools are served by MCP servers, discovery is part of the protocol. An MCP server exposes a `tools/list` endpoint that returns the full set of available tool contracts in the MCP schema format. The abstraction layer's MCP Client calls this endpoint at connection time and populates the registry from the response automatically.

> 💡 **Implement a capability hash on your MCP server's `tools/list` response.** The client can poll this hash at low frequency to detect when the server's tool set has changed, triggering a selective registry update without fetching the full list on every heartbeat.

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-REG-001` | The Tool Registry MUST be the single source of truth for all tool contracts — no tool call may be dispatched against a contract not present in the registry. |
| `TOOL-REG-002` | When a mid-session tool deregistration occurs, the abstraction layer MUST return a `TOOL_NOT_FOUND` error envelope to the agent — it MUST NOT throw an unchecked exception. |
| `TOOL-REG-003` | MCP tool discovery via `tools/list` MUST populate the registry at connection time AND re-populate when the server signals `tools/list/changed` — stale registry entries after server update are a silent correctness defect. |
| `TOOL-REG-004` | The `schemaFor(name)` operation MUST return a JSON Schema object suitable for direct injection into the LLM system prompt — no transformation required by the agent runtime. |

### Java + LangGraph Target

```java
@Component
public class ToolRegistry {
    private final ConcurrentHashMap<String, RegisteredTool> tools = new ConcurrentHashMap<>();
    private final ToolContractValidator validator;

    public void register(ToolContract contract, ToolHandler handler) {
        validator.validate(contract);                        // TOOL-CONTRACT-001
        tools.put(contract.name(), new RegisteredTool(contract, handler));
    }

    public void deregister(String name) {
        tools.remove(name);
    }

    // TOOL-REG-001: single source of truth
    public Optional<RegisteredTool> lookup(String name) {
        return Optional.ofNullable(tools.get(name));
    }

    public List<ToolContract> list(Predicate<ToolContract> filter) {
        return tools.values().stream()
            .map(RegisteredTool::contract)
            .filter(filter)
            .collect(Collectors.toList());
    }

    // TOOL-REG-004: schema ready for LLM prompt injection
    public JsonSchema schemaFor(String name) {
        return lookup(name)
            .map(t -> t.contract().inputSchema())
            .orElseThrow(() -> new ToolNotFoundException(name));
    }
}
```

---

## Section 32.4 — MCP as the Wiring Standard: From Protocol to Implementation

### What MCP Adds Beyond the Tool Contract

Chapter 20 established MCP's protocol semantics — the JSON-RPC 2.0 message format, the `tools/list` and `tools/call` methods, capability negotiation, and transport options. The engineering question here is different: **how do you wire MCP into the abstraction layer so that tool calls from the agent runtime cross process and network boundaries transparently?**

MCP adds three things the tool contract alone does not provide:

1. **Transport abstraction** — tools can run in the same process (in-process handler), a child process (stdio transport), or a remote service (HTTP/SSE transport). The MCP client presents a uniform call interface regardless of where the server runs.
2. **Protocol versioning** — MCP's capability negotiation handshake establishes which features both client and server support, allowing the ecosystem to evolve without breaking existing integrations.
3. **Streaming results** — for long-running tools, MCP supports progressive result streaming over SSE, letting the agent runtime receive partial results before execution completes.

### The MCP Connection Lifecycle

Every MCP connection follows the same initialization sequence before any tool can be called:

```
1. Transport establishment
   → Client opens a stdio pipe (local) OR HTTP/SSE connection (remote)

2. Initialize handshake
   → Client sends `initialize` with protocol version + capabilities
   → Server responds with its version + capabilities
   → Both sides agree on the intersection

3. Tool discovery
   → Client calls `tools/list`
   → Server returns array of tool definitions
   → Client registers them in the Tool Registry

4. Steady state
   → Connection ready for `tools/call` requests
   → Client may receive server-initiated `tools/list/changed` notifications

5. Shutdown
   → Either side may terminate
   → Client must handle mid-session disconnection gracefully
   → Attempt reconnection before failing in-flight calls
```

### Transport Selection: stdio vs. HTTP/SSE

| Dimension | stdio | HTTP/SSE |
|---|---|---|
| **Deployment model** | Server runs as child process of the agent | Server runs as independent network service |
| **Latency** | Lowest — no network overhead | Network RTT added to every call |
| **Scalability** | One server instance per agent process | Many agents share one server pool |
| **Fault isolation** | Server crash brings down agent process | Server crash is isolated — client reconnects |
| **Authentication** | OS process ownership | OAuth 2.0, API keys, mTLS |
| **Best for** | Local tools, dev environments, single-tenant | Shared enterprise tools, multi-tenant, cloud-native |

> 💡 **Use stdio transport during development for simplicity and zero configuration. Switch to HTTP/SSE for any tool that needs to be shared across multiple agent instances or deployed independently of the agent runtime.** Never use stdio in a containerized production deployment — the process lifecycle management becomes a reliability liability.

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-MCP-001` | The abstraction layer's MCP Client MUST implement the full MCP connection lifecycle — including capability negotiation, `tools/list` discovery, reconnection on disconnect, and `tools/list/changed` handling. |
| `TOOL-MCP-002` | stdio transport MUST NOT be used in containerized production deployments — use HTTP/SSE transport for any tool server that must be independently deployable. |
| `TOOL-MCP-003` | The MCP Client MUST support configurable per-connection timeouts for the initialize handshake and `tools/list` calls — unbounded connection setup blocks agent startup. |
| `TOOL-MCP-004` | The MCP Client MUST propagate W3C `traceparent` headers on all `tools/call` HTTP requests (see `OBS-TRACE-001` in chap23). |
| `TOOL-MCP-005` | Tool contracts auto-discovered from MCP `tools/list` MUST be normalized to the internal `ToolContract` schema before being registered in the Tool Registry — raw MCP schemas MUST NOT be stored directly. |

### Java + LangGraph Target

```java
@Component
public class McpToolClient {
    private final McpClient delegate;         // from io.modelcontextprotocol:sdk-client-java
    private final ToolRegistry registry;
    private final OpenTelemetry otel;

    // TOOL-MCP-001: full lifecycle
    public CompletableFuture<Void> connect(McpServerConfig config) {
        return delegate.connect(config)
            .thenCompose(conn -> conn.initialize())
            .thenCompose(conn -> conn.listTools())
            .thenAccept(tools -> tools.forEach(this::registerFromMcp));
    }

    private void registerFromMcp(McpToolDefinition mcpTool) {
        // TOOL-MCP-005: normalize to internal ToolContract
        ToolContract contract = ToolContractMapper.fromMcp(mcpTool);
        registry.register(contract, args -> callMcpTool(mcpTool.name(), args));
    }

    private ToolResult callMcpTool(String name, Map<String, Object> args) {
        Tracer tracer = otel.getTracer("mcp.client");
        Span span = tracer.spanBuilder("mcp.tool_call")
            .setAttribute("tool.name", name)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // TOOL-MCP-004 + OBS-TRACE-001: inject traceparent
            Map<String, String> headers = new HashMap<>();
            W3CTraceContextPropagator.getInstance()
                .inject(Context.current(), headers, Map::put);

            return delegate.callTool(name, args, headers);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Section 32.5 — The Tool Execution Lifecycle

### The Seven-Stage Dispatch Pipeline

When an agent's reasoning loop decides to call a tool, the call passes through a **deterministic sequence of seven stages** inside the abstraction layer. Each stage is a failure point; each must be handled explicitly.

```
Stage 1: Schema Validation
  → Validate arguments against tool's registered inputSchema
  → Invalid arguments: reject immediately (INVALIDARGS) — call never reaches tool

Stage 2: Authorization Check
  → Verify calling agent has permission for this tool + these arguments
  → Check tool-level ACL, argument-level scope, risk-based step-up (destructive)
  → Unauthorized: return UNAUTHORIZED envelope immediately

Stage 3: Pre-Call Logging & Metrics
  → Emit mcp.tool.dispatched log event (see OBS-LOG-003)
  → Increment in-flight counter for this tool
  → Record traceId on span

Stage 4: Dispatch
  → Forward call to execution handler with deadline = contract.timeoutMs
  → For MCP: inject traceparent header (TOOL-MCP-004)
  → On deadline expiry: cancel in-flight request, return TOOLTIMEOUT

Stage 5: Result Validation
  → Validate response against tool's registered outputSchema
  → Malformed responses: wrap in error envelope (TOOL_SCHEMA_VIOLATION)

Stage 6: Output Filtering
  → Apply PII stripping (chap33, OBS-LOG-004)
  → Truncate oversized outputs
  → Flag suspicious patterns (possible indirect prompt injection)

Stage 7: Post-Call Logging & Metrics
  → Emit mcp.tool.completed or mcp.tool.failed log event
  → Decrement in-flight counter
  → Update latency histogram
  → Return result or error envelope to agent runtime
```

### The Result Envelope

Every tool call returns one of two envelope types. **The agent runtime MUST never throw on a tool error — it always receives an envelope and decides what to do.**

```json
// Success envelope
{
  "status": "success",
  "traceId": "t-abc123",
  "toolName": "kb.search",
  "durationMs": 243,
  "result": { ... }
}

// Error envelope
{
  "status": "error",
  "traceId": "t-abc123",
  "toolName": "kb.search",
  "durationMs": 5001,
  "error": {
    "code": "TOOLTIMEOUT",
    "message": "Tool exceeded 5000ms deadline.",
    "retryable": true
  }
}
```

The `retryable` flag is set by the abstraction layer based on the error code AND the tool's `sideEffects` classification:
- `read-only` tools: retryable on `TOOLTIMEOUT`, `TOOLUNAVAILABLE`, `RATELIMITED`
- `write` tools: NOT retryable unless `idempotent: true`
- `destructive` tools: NEVER retryable

### Timeout and Deadline Propagation

Timeouts are not optional. An agent that calls a tool with no deadline will hang indefinitely if the tool blocks — and an LLM inference budget does not pause while the tool waits.

The tool contract's `timeoutMs` field defines the maximum wall-clock time the abstraction layer will wait for a response. When the deadline expires, the layer cancels the in-flight request (via Java `CompletableFuture` cancellation or `Thread.interrupt()`) and returns `TOOLTIMEOUT`.

> ⚠️ **In MCP, the client controls the deadline.** The MCP server is not required to honor the deadline — it is the client's responsibility to cancel the call when the deadline expires, regardless of whether the server acknowledges the cancellation.

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-EXEC-001` | Every tool call MUST pass through all seven stages of the dispatch pipeline — skipping any stage (e.g., bypassing authorization for "trusted" tools) removes a mandatory safety guarantee. |
| `TOOL-EXEC-002` | The result envelope MUST be the only return type from the abstraction layer — no unchecked exceptions may propagate to the agent reasoning loop from tool dispatch. |
| `TOOL-EXEC-003` | The `retryable` flag in error envelopes MUST be computed from both the error code AND the tool's `sideEffects` classification — `retryable: true` on a write tool is a safety defect. |
| `TOOL-EXEC-004` | Every tool dispatch MUST have a deadline derived from `contract.timeoutMs` — unbounded tool calls are not permitted in production. |
| `TOOL-EXEC-005` | Output filtering (Stage 6) MUST run before the result is returned to the agent runtime — raw tool output MUST be treated as untrusted input when constructing subsequent prompts (indirect prompt injection mitigation). |
| `TOOL-EXEC-006` | The dispatch pipeline MUST be instrumented with the four metric families defined in `OBS-METRIC-001` (chap23): latency histogram, error rate by code, in-flight gauge, and cache hit ratio. |

### Java + LangGraph Target

```java
@Component
public class ToolDispatchPipeline {
    private final ToolRegistry registry;
    private final ToolAuthorizationPolicy authPolicy;
    private final OutputFilterChain outputFilter;
    private final AgentMetrics metrics;
    private final AgentStructuredLogger logger;

    // TOOL-EXEC-001: all seven stages enforced
    public ToolResult dispatch(String toolName, Map<String, Object> args,
                                AgentCallingContext ctx) {

        // Stage 1: Schema validation
        RegisteredTool tool = registry.lookup(toolName)
            .orElse(null);
        if (tool == null) {
            return ToolResult.error(toolName, "TOOL_NOT_FOUND",
                "Tool not registered: " + toolName, false);        // TOOL-REG-002
        }
        ValidationResult validation = tool.contract().inputSchema().validate(args);
        if (!validation.valid()) {
            return ToolResult.error(toolName, "INVALIDARGS",
                validation.message(), false);
        }

        // Stage 2: Authorization
        AuthDecision decision = authPolicy.evaluate(tool.contract(), args, ctx);
        if (!decision.permitted()) {
            return ToolResult.error(toolName, "UNAUTHORIZED",
                decision.reason(), false);
        }

        // Stage 3: Pre-call logging + metrics (TOOL-EXEC-006)
        logger.logToolDispatched(ctx.agentState(), toolName,
            tool.serverUrl(), tool.contract().inputSchemaHash(), tool.contract().timeoutMs());
        metrics.incrementInFlight(toolName);
        long startMs = System.currentTimeMillis();

        try {
            // Stage 4: Dispatch with deadline (TOOL-EXEC-004)
            CompletableFuture<Object> future = CompletableFuture.supplyAsync(
                () -> tool.handler().call(args));
            Object raw = future.get(tool.contract().timeoutMs(), TimeUnit.MILLISECONDS);

            // Stage 5: Result validation
            ValidationResult outValidation = tool.contract().outputSchema().validate(raw);
            if (!outValidation.valid()) {
                return ToolResult.error(toolName, "TOOL_SCHEMA_VIOLATION",
                    outValidation.message(), false);
            }

            // Stage 6: Output filtering (TOOL-EXEC-005)
            Object filtered = outputFilter.apply(toolName, raw, ctx);

            // Stage 7: Post-call logging + metrics
            long durationMs = System.currentTimeMillis() - startMs;
            logger.logToolCompleted(ctx.agentState(), toolName, durationMs);
            metrics.recordToolCall(toolName, "success", null, durationMs, ctx.agentId());

            return ToolResult.success(toolName, filtered);                // TOOL-EXEC-002

        } catch (TimeoutException e) {
            long durationMs = System.currentTimeMillis() - startMs;
            boolean retryable = tool.contract().sideEffects() == SideEffectClass.READ_ONLY;
            logger.logToolFailed(ctx.agentState(), toolName, "TOOLTIMEOUT", retryable);
            metrics.recordToolCall(toolName, "error", "TOOLTIMEOUT", durationMs, ctx.agentId());
            return ToolResult.error(toolName, "TOOLTIMEOUT",
                "Tool exceeded " + tool.contract().timeoutMs() + "ms deadline.",
                retryable);                                               // TOOL-EXEC-003

        } finally {
            metrics.decrementInFlight(toolName);
        }
    }
}
```

---

## Section 32.6 — Error Handling and Fallback Strategies

### Error Classification

Not all tool failures are equal. The abstraction layer MUST distinguish four categories, each with distinct retry behavior:

| Error Category | Codes | Retryable? | Response |
|---|---|---|---|
| **Validation errors** | `INVALIDARGS` | No | Agent must reformulate the call |
| **Authorization errors** | `UNAUTHORIZED`, `FORBIDDEN` | No | Not retryable without credential/policy change |
| **Transient errors** | `TOOLTIMEOUT`, `TOOLUNAVAILABLE`, `RATELIMITED` | Yes (read-only only) | Retry with exponential backoff + jitter |
| **Tool errors** | `TOOLERROR` | No | Application-level failure — return to agent for reasoning |

### Retry with Exponential Backoff and Jitter

For transient errors on retryable (read-only) tools, the abstraction layer implements exponential backoff with jitter. The delay before attempt *n* is:

\[d_n = \min\!\left(b \cdot 2^n + \text{Uniform}(0,\, j),\; D\right)\]

where:
- \(b\) = base delay (typically 100 ms)
- \(D\) = ceiling on delay (e.g., 30 s) — prevents excessively long waits
- \(j\) = maximum jitter (typically 50% of the unclipped delay) — prevents thundering-herd behavior

**The deadline guard** is applied before each retry attempt. Let \(R\) be the remaining wall-clock time in the tool's deadline, and \(\bar{t}\) the mean observed call duration for this tool. A retry is only attempted when:

\[d_n + \bar{t} \leq R\]

> ⚠️ **Never retry a tool without first checking the remaining deadline budget.** An agent with a 30-second task deadline cannot afford three 5-second retries on a single tool call. The \(D\) ceiling and the deadline budget guard address different concerns — both are required.

### Fallback Tools and Circuit Breakers

**Fallback tools** are declared at registration time as part of the tool contract:

```json
{
  "name": "news.search.live",
  "fallback": "news.search.cached",
  "sideEffects": "read-only",
  "timeoutMs": 3000,
  "maxRetries": 1
}
```

When a tool fails beyond its retry budget, the abstraction layer routes to the declared fallback. A real-time search falls back to a cached index; a live database query falls back to a read replica.

**Circuit breakers** add a layer above retries. After a configurable number of consecutive failures, the circuit opens — calls return `CIRCUITOPEN` immediately without dispatching. The circuit moves to half-open after a recovery window, then closes on a successful probe call.

> 💡 **Instrument each circuit breaker with a named metric and a tracing label.** When a circuit opens in production, the alert should fire within seconds — not after the agent has silently degraded for minutes. (See `ToolCircuitBreakerOpen` alert in chap23 Section 23.4.)

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-ERR-001` | The abstraction layer MUST classify every tool error into one of the four categories (validation, authorization, transient, tool) before determining retry behavior. |
| `TOOL-ERR-002` | Retry with exponential backoff MUST only be applied to transient errors on `read-only` tools — retrying write tools without explicit `idempotent: true` is a correctness defect. |
| `TOOL-ERR-003` | Retry attempts MUST be gated by the deadline budget guard — no retry may be initiated when the remaining deadline is insufficient to complete another attempt. |
| `TOOL-ERR-004` | Fallback tools MUST satisfy the same output schema as the primary tool — the agent runtime receives the same envelope structure regardless of which implementation served the call. |
| `TOOL-ERR-005` | Circuit breaker state MUST be exported as a named metric (`agent_tool_circuit_breaker_state`) and MUST trigger a P1 alert when any tool circuit opens (see `OBS-METRIC-001`). |

---

## Section 32.7 — Security Boundaries at the Tool Layer

### The Tool Layer as an Attack Surface

The tool layer is the point where agent intent crosses into real-world side effects. It is the primary attack surface for threat classes that do not exist in conventional software:

| Threat | Mechanism |
|---|---|
| **Tool misuse via prompt injection** | Malicious payload in retrieved content instructs agent to call a destructive tool with attacker-controlled arguments |
| **Privilege escalation via tool chaining** | Agent chains a low-privilege read tool with a high-privilege write tool using the output of the first to construct arguments for the second |
| **Data exfiltration via output channels** | Tool invocation exfiltrates sensitive data by encoding it in arguments of a subsequent outbound call (e.g., an email-send tool) |
| **Overly permissive tool scopes** | Tools declared with broader permissions than required, increasing blast radius when an agent is compromised or misconfigured |

### Three-Layer Defense-in-Depth

**Layer 1 — Authorization (Stage 2 of dispatch pipeline)**

Every tool call carries a calling context: agent identity, session identity, delegated permission set. The authorization check evaluates three questions:

1. Is this agent allowed to call this tool at all? (**Tool-level ACL**)
2. Are these specific argument values within the agent's permitted scope? (**Argument-level policy** — e.g., agent may query only its own tenant's data)
3. Does the tool's side-effect class require additional confirmation for this calling context? (**Risk-based step-up** for destructive tools)

Authorization policies MUST be declared externally in a policy engine (OPA or equivalent) — not hard-coded in the tool handler. This makes them auditable, testable, and changeable without redeploying the tool.

**Layer 2 — Input Sanitization (Stage 2, after authorization)**

Input sanitization runs after schema validation and before dispatch. It applies tool-specific rules:
- Strip control characters from string arguments
- Cap numeric arguments to safe ranges
- Reject arguments that match known injection patterns

> ℹ️ **Input sanitization is a defense-in-depth measure, not a replacement for authorization.** An attacker who controls the agent's reasoning via prompt injection can craft arguments that pass sanitization. The correct mitigation for prompt injection combines output filtering on retrieved content, least-privilege tool access, and human confirmation gates on destructive actions.

**Layer 3 — Output Filtering (Stage 6 of dispatch pipeline)**

Output filtering runs on the tool's result before returning it to the agent:
- Scrub PII fields the agent does not need (e.g., SSNs returned by a CRM tool when only the name is needed)
- Truncate oversized outputs to prevent context window flooding
- Flag results containing suspicious patterns (e.g., instructions embedded in tool output that could be injected into the next reasoning step)

> ⚠️ **Passing raw tool output directly into the next LLM prompt without filtering is the primary mechanism of indirect prompt injection attacks.** Always treat tool output as untrusted user input when constructing subsequent prompts.

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-SEC-001` | Authorization policies MUST be declared in an external policy engine (OPA or equivalent) — hard-coded authorization logic in tool handlers is not auditable or testable. |
| `TOOL-SEC-002` | Argument-level scope enforcement MUST be implemented for every tool that accesses multi-tenant data — tool-level ACL alone is insufficient for tenant isolation. |
| `TOOL-SEC-003` | Input sanitization MUST run after schema validation and before dispatch for every tool call — it is a non-optional pipeline stage, not an optional middleware. |
| `TOOL-SEC-004` | Output filtering MUST run before returning any tool result to the agent runtime — raw tool output MUST be treated as untrusted. |
| `TOOL-SEC-005` | Least-privilege tool scopes MUST be enforced at registration time — tools MUST NOT be registered with broader permissions than required by their declared `sideEffects` classification. |

---

## Section 32.8 — Testing and Mocking Tool Integrations

### The Testing Problem for Tool-Using Agents

Testing agents that use tools is harder than testing conventional software because the system under test has non-deterministic outputs (the LLM) combined with stateful side effects (the tools). The practical solution is a **three-level testing strategy** that separates tool logic testing from agent reasoning testing.

### Level 1 — Mock Tools for Agent Unit Tests

A mock tool satisfies the full tool contract but returns fixture data rather than calling a real backend. Because the abstraction layer enforces the tool contract uniformly, the agent runtime cannot distinguish a mock tool from a real one. This gives agent unit tests complete control over tool responses — including error conditions — without any external dependencies.

```java
// Java mock tool registration for agent unit tests
public class TestToolRegistryBuilder {
    public static ToolRegistry buildTestRegistry() {
        ToolRegistry registry = new ToolRegistry(new ToolContractValidator());

        // Mock kb.search with fixture data
        registry.register(
            ToolContractLoader.load("kb.search"),
            args -> ToolResult.success("kb.search",
                FixtureLoader.load("kb-results.json"))
        );

        // Mock crm.getContact with fixture data
        registry.register(
            ToolContractLoader.load("crm.getContact"),
            args -> ToolResult.success("crm.getContact",
                FixtureLoader.load("contact.json"))
        );

        return registry;
    }
}

@Test
public void agentUsesKbBeforeCrm() {
    Agent agent = new Agent(TestToolRegistryBuilder.buildTestRegistry());
    AgentResult result = agent.run("Who is Jane Doe's account manager?");
    assertToolCalledBefore(result, "kb.search", "crm.getContact");
}
```

### Level 2 — Contract Tests for MCP Servers

A contract test verifies that a tool server's actual behavior matches its declared contract — independent of any agent that uses it. Contract tests run against the live tool server or a staging instance and check:

- The server's `tools/list` response matches the registered contract schemas
- Valid input produces output conforming to the output schema
- Invalid input produces a well-formed error response (not a server panic or malformed payload)
- The declared `timeoutMs` is realistic — the server responds within it under normal load
- `sideEffects` declarations are accurate — write tools produce observable state changes; read-only tools do not

> 💡 **Run contract tests in your CI pipeline against a staging tool server on every pull request that touches a tool's contract, handler, or MCP server implementation.** This catches schema drift — where deployed tool behavior has diverged from the registered contract — before it reaches production.

### Level 3 — End-to-End Integration Tests

End-to-end integration tests run the complete stack (agent runtime, abstraction layer, MCP Client, and a live or fully emulated tool server) and validate that the entire dispatch pipeline behaves correctly under realistic conditions. Minimum coverage:

| Scenario | What It Validates |
|---|---|
| Successful tool discovery | `tools/list` response populates registry correctly |
| Correct argument serialization | Arguments round-trip correctly over MCP transport |
| Timeout behavior | Server delayed beyond `timeoutMs` returns `TOOLTIMEOUT` |
| Retry on transient error | Server returns 503 once, then succeeds — abstraction layer retries correctly |
| Circuit breaker activation | Repeated failures trip the circuit breaker |
| Authorization rejection | Out-of-scope arguments are rejected at Stage 2 |
| Metric emission | Latency histograms and error counters update correctly for each outcome |

### Design Rules

| Rule ID | Rule |
|---|---|
| `TOOL-TEST-001` | Every agent that uses tools MUST have a unit test suite that uses mock tools satisfying the full tool contract — tests that call real tool servers are integration tests, not unit tests. |
| `TOOL-TEST-002` | Every MCP tool server MUST have contract tests that run in CI against a staging instance on every PR touching the tool's contract or implementation. |
| `TOOL-TEST-003` | End-to-end integration tests MUST cover all seven failure scenarios listed in Section 32.8 — partial coverage creates false confidence in the dispatch pipeline's reliability. |
| `TOOL-TEST-004` | Schema drift detection MUST be implemented as a contract test — a tool server whose `tools/list` response diverges from the registered contract without a matching contract test will cause silent production failures. |

---

## Section 32.9 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `TOOL-ABS-001` | Abstraction | Tool layer as explicit architectural component, not glue code |
| `TOOL-ABS-002` | Abstraction | Agent reasoning loop interacts with tools only through abstraction layer |
| `TOOL-ABS-003` | Abstraction | Uniform call interface regardless of tool implementation |
| `TOOL-CONTRACT-001` | Contract | Complete six-field contract required at registration |
| `TOOL-CONTRACT-002` | Contract | Namespaced dot-notation tool names |
| `TOOL-CONTRACT-003` | Contract | Descriptions written for LLM reader |
| `TOOL-CONTRACT-004` | Contract | `sideEffects` validated against observed behavior in contract tests |
| `TOOL-CONTRACT-005` | Contract | `destructive` tools require `requiresConfirmation: true` and HITL gate |
| `TOOL-REG-001` | Registry | Tool Registry is single source of truth |
| `TOOL-REG-002` | Registry | Mid-session deregistration returns `TOOL_NOT_FOUND` envelope |
| `TOOL-REG-003` | Registry | MCP `tools/list` populates registry at connect and on `tools/list/changed` |
| `TOOL-REG-004` | Registry | `schemaFor(name)` returns LLM-injectable JSON Schema |
| `TOOL-MCP-001` | MCP | Full MCP connection lifecycle including reconnection |
| `TOOL-MCP-002` | MCP | stdio transport prohibited in containerized production |
| `TOOL-MCP-003` | MCP | Configurable timeouts for initialize handshake and `tools/list` |
| `TOOL-MCP-004` | MCP | W3C `traceparent` propagated on all `tools/call` HTTP requests |
| `TOOL-MCP-005` | MCP | MCP schemas normalized to internal `ToolContract` before registration |
| `TOOL-EXEC-001` | Execution | All seven pipeline stages enforced for every call |
| `TOOL-EXEC-002` | Execution | Result envelope only return type — no unchecked exceptions |
| `TOOL-EXEC-003` | Execution | `retryable` flag computed from error code AND `sideEffects` |
| `TOOL-EXEC-004` | Execution | Every dispatch has a deadline from `contract.timeoutMs` |
| `TOOL-EXEC-005` | Execution | Output filtering runs before result returned to agent |
| `TOOL-EXEC-006` | Execution | Four metric families instrumented (chap23 `OBS-METRIC-001`) |
| `TOOL-ERR-001` | Error handling | Four-category error classification before retry decision |
| `TOOL-ERR-002` | Error handling | Retry only on transient errors and read-only tools |
| `TOOL-ERR-003` | Error handling | Retry gated by deadline budget guard |
| `TOOL-ERR-004` | Error handling | Fallback tools satisfy same output schema as primary |
| `TOOL-ERR-005` | Error handling | Circuit breaker state exported as metric + P1 alert |
| `TOOL-SEC-001` | Security | Authorization policies in external policy engine (OPA) |
| `TOOL-SEC-002` | Security | Argument-level scope enforcement for multi-tenant data |
| `TOOL-SEC-003` | Security | Input sanitization as non-optional pipeline stage |
| `TOOL-SEC-004` | Security | Output filtering before returning to agent runtime |
| `TOOL-SEC-005` | Security | Least-privilege tool scopes enforced at registration |
| `TOOL-TEST-001` | Testing | Mock tool unit test suite for every tool-using agent |
| `TOOL-TEST-002` | Testing | MCP server contract tests in CI |
| `TOOL-TEST-003` | Testing | E2E integration tests cover all seven failure scenarios |
| `TOOL-TEST-004` | Testing | Schema drift detection as contract test |

---

## Section 32.10 — Chapter Boundary: Relationship to Adjacent Chapters

| Chapter | Relationship |
|---|---|
| **chap20** (MCP Protocol) | Provides the protocol specification; chap32 is the implementation layer on top of that spec. Read chap20 before chap32. |
| **chap23** (Observability) | Defines `OBS-*` rules that chap32 satisfies: `OBS-TRACE-003` (MCP tool call spans), `OBS-LOG-003` (mandatory log events), `OBS-METRIC-001` (tool metric families). |
| **chap12** (HITL) | `TOOL-CONTRACT-005` references HITL confirmation gates for destructive tools — chap12 defines the implementation. |
| **chap33** (Enterprise Integration) | Fills the abstraction layer with concrete enterprise backends (REST APIs, databases, SaaS platforms, message queues). chap33 assumes all `TOOL-*` rules from chap32 are in place. |
| **chap34** (Advanced Tool Patterns) | Extends chap32 with parallel fan-out, tool chaining safety, dynamic tool selection, async long-running tools, and tool versioning. All advanced patterns build on the abstraction layer architecture defined here. |
