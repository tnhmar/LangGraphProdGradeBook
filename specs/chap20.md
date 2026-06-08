# Chapter 20 Spec — Layer 1: Tool Access — The Model Context Protocol (MCP)

> **Volume 1 reference:** `chlayer-1-tool-access-the-model-context-protocol`
> **Part:** 5 — Agent Communication Protocols
> **Depends on:** chap19 (`LAYER-001`, `LAYER-002`, `SEC-001–005`, `PROTO-DEF-001–003`)
> **Java + LangGraph implementation target:** Volume 2 (MCP Client/Server wiring, Tool Abstraction Layer)
> **Bridges to:** chap21 (A2A, Layer 2), chap33 (Enterprise system integration), Volume 2 chap6 (sandboxing)

---

## Chapter Purpose

Chapter 19 established *why* protocols are required. Chapter 20 opens the first concrete layer — **Layer 1: Tool Access** — the interface between an agent and the external tools, data sources, and capability providers it depends on.

**MCP is the standard for Layer 1.** Introduced by Anthropic in November 2024 and adopted by OpenAI, Google DeepMind, LangChain, and a broad ecosystem of tool providers by 2025, MCP provides a universal JSON-RPC 2.0-based interface that replaces the O(n×m) integration problem with a single well-specified contract.

This chapter covers:
- Why a dedicated tool protocol is necessary and what architectural constraints shaped MCP
- The host / client / server three-role model
- The two official transports (STDIO, Streamable HTTP)
- The three-phase connection lifecycle and capability negotiation
- The three server-side primitives (tools, resources, prompts)
- The three client-side primitives (sampling, roots, elicitation)
- The OAuth 2.1 security model and the most common production security gaps
- Production considerations: versioning, observability, error handling
- The 2026 MCP roadmap: stateless transport, Server Cards, native task management, enterprise extensions

---

## Section 20.1 — Why a Dedicated Tool Protocol

### Context

Before MCP, every tool a developer wanted to expose to an agent required a custom adapter. A web search tool had one interface, a database query tool had another, a code execution sandbox had a third. Agents built on different frameworks used different calling conventions. A tool built for one agent could not be reused by another without rewriting the adapter. The integration cost was **O(n × m)**.

MCP changes the economics: a tool provider implements one MCP server; any agent that speaks MCP can call it. The cost shifts to **O(n + m)** — each agent adopts the protocol once, each tool provider implements it once.

### Inspiration: The Language Server Protocol Parallel

MCP draws explicit inspiration from the **Language Server Protocol (LSP)**, developed by Microsoft in 2016. Before LSP, adding language support to a code editor required a custom plugin per editor-language pair. LSP defined a standard so that a language server built once for Go, Rust, or Python works with any conforming editor. MCP applies the same architectural insight to the agent domain: a capability provider built once as an MCP server can serve any conforming agent host.

### Three Design Constraints

MCP was designed under three explicit constraints that shaped every architectural decision:

| Constraint | Description |
|---|---|
| **Transport agnosticism** | Protocol must work over multiple transport mechanisms (local child processes and remote network services) without changing the application-level protocol |
| **Capability negotiation** | Not every server supports every feature; parties must declare what they support at connection time so clients can adapt without failing on unsupported operations |
| **Human-in-the-loop compatibility** | Servers may request model completions from the client (not directly from the model API), keeping the human and host application in the control path |

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-ARCH-001` | MCP MUST be adopted as the Layer 1 standard for all agent-tool integrations — custom bespoke adapters are permitted only for tools that cannot be wrapped as MCP servers (and this exception MUST be documented as architectural debt). |
| `MCP-ARCH-002` | The O(n + m) composability model MUST be the architectural target — shared MCP tool servers MUST be preferred over per-agent tool implementations. |
| `MCP-ARCH-003` | MCP MUST NOT be used for Layer 2 (agent-to-agent collaboration) — A2A is the correct protocol for that scope (see chap21). |

---

## Section 20.2 — Architecture: Hosts, Clients, and Servers

### The Three-Role Model

MCP defines three distinct roles. The terms *client* and *server* carry different meanings than in most network protocols — understanding the separation is critical:

| Role | Responsibility | Key Constraint |
|---|---|---|
| **Host** | The application the user interacts with (AI IDE, chat interface, agent framework runtime). Manages user experience, conversation state, LLM orchestration, and overall session. Instantiates clients. Enforces user-level permissions. | Owns the control path — all tool calls are mediated through the host |
| **Client** | Protocol-level component instantiated by the host. Maintains a **one-to-one** connection with a single MCP server. Handles protocol mechanics: request formatting, response handling, connection lifecycle, capability contract enforcement. | One client per server connection; a single host may instantiate many clients |
| **Server** | Process that exposes capabilities (tools, resources, prompts). Lightweight and focused — does not manage conversation state or orchestrate the LLM. May be a local child process, remote HTTP service, or in-process library. | Responds to requests; does not initiate tool calls autonomously |

### The Critical Architectural Rule

> **The LLM never communicates with the MCP server directly.** The host mediates all interactions, preserving the control path and enabling the host to intercept, log, or reject any tool call. MCP is a protocol between a host application (acting through a client) and a capability provider (the server) — not between an LLM and a tool.

### Flow of a Single Tool Call

```
Agent reasoning loop (LLM inside Host)
  → determines tool call needed
  → invokes MCP Client
    → formats tools/call request (JSON-RPC 2.0)
    → sends over transport (STDIO or HTTP)
      → MCP Server receives request
      → executes tool function
      → returns JSON-RPC response
    → Client delivers result to Host
  → Host injects result into LLM context
→ Next reasoning step
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-ROLE-001` | The host MUST mediate all MCP interactions — no code path in the agent runtime may call an MCP server directly, bypassing the client. |
| `MCP-ROLE-002` | Each MCP Client instance MUST maintain a one-to-one connection with a single server — multiplexing multiple servers over a single client connection is a protocol violation. |
| `MCP-ROLE-003` | The host MUST enforce user-level permissions before dispatching any tool call through a client — authorization is a host responsibility, not a server responsibility alone. |

### Java + LangGraph Target

```java
/**
 * MCP-ROLE-001: Host mediates all MCP interactions.
 * Each McpClientConnection is a one-to-one client-server channel (MCP-ROLE-002).
 */
public class McpHostNode implements RunnableNode<AgentState> {
    private final Map<String, McpClientConnection> clients; // serverName → client
    private final HostAuthorizationPolicy authPolicy;       // MCP-ROLE-003

    @Override
    public AgentState run(AgentState state) {
        ToolCallRequest req = state.pendingToolCall();
        McpClientConnection client = clients.get(req.serverName());

        // MCP-ROLE-003: enforce host-level authorization before dispatch
        authPolicy.assertAllowed(state.agentIdentity(), req);

        ToolCallResult result = client.call(req.toolName(), req.arguments());
        return state.withToolResult(result);
    }
}
```

---

## Section 20.3 — The Transport Layer

### JSON-RPC 2.0 as the Application Layer

All MCP communication uses **JSON-RPC 2.0** as the application-layer message format — a lightweight RPC protocol encoding each call as a JSON object with a method name, parameters, and a request ID. Responses carry the result or an error object. Notifications (messages that do not require a response) use the same format without an ID.

> ⚠️ **JSON-RPC 2.0 batch requests were removed from the MCP specification in the June 2025 update.** Any implementation that relied on batching multiple calls into a single JSON-RPC array MUST be updated to issue calls individually.

### Two Official Transports

| Transport | When to Use | Direction | Notes |
|---|---|---|---|
| **STDIO** | Local tool server running as a managed subprocess of the host | Bidirectional (stdin/stdout pipes) | No network config, no port allocation, no auth infrastructure needed. Process-level OS isolation. Cannot serve remote clients. |
| **Streamable HTTP** | Remote tool server, shared across multiple hosts, cloud-native deployment | HTTP POST (client→server) + SSE stream (server→client) | Standard HTTP POST for single-result calls; SSE for streaming/incremental results. No WebSocket required. |

> 💡 **Use STDIO for local tool servers during development and for production deployments where the tool runs as a managed subprocess of the agent runtime. Use Streamable HTTP for shared tool services, third-party integrations, and any scenario where the tool must be accessible from multiple host applications.**

The MCP specification mandates **exactly two** official transports. This constraint is intentional — limiting to two well-defined mechanisms ensures any conforming client and server can interoperate out of the box. Custom transports are permitted for specialized deployments but **break the interoperability guarantee**.

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-TRANS-001` | Production deployments MUST use only STDIO or Streamable HTTP — custom transport implementations break the ecosystem interoperability guarantee and MUST be documented as architectural debt. |
| `MCP-TRANS-002` | STDIO transport MUST NOT be used in containerized production deployments — process lifecycle management becomes a reliability liability. |
| `MCP-TRANS-003` | Streamable HTTP deployments MUST be designed for stateless horizontal scaling — sticky-session requirements are a design defect to be eliminated (aligned with 2026 roadmap). |
| `MCP-TRANS-004` | JSON-RPC batch requests MUST NOT be used — they were removed from the specification in June 2025. |

---

## Section 20.4 — The Connection Lifecycle

### Three Phases

An MCP connection passes through **three phases**: Initialization, Operation, and Shutdown. Each phase has a defined set of valid messages — sending a message in the wrong phase is a protocol error.

### Phase 1: Initialization and Capability Negotiation

The initialization phase follows a strict **three-step sequence**:

```
1. Client  →  Server:   initialize(protocolVersion, clientCapabilities)
2. Server  →  Client:   initialize response(protocolVersion, serverCapabilities)
3. Client  →  Server:   notifications/initialized  (session open confirmed)
```

This handshake **locks in the capability contract** for the session:
- Neither party may use a feature the other party did not declare
- Version mismatch → server returns error → connection terminates
- Capability negotiation prevents calling methods the server does not implement — without it, failures surface at runtime as confusing errors

### Phase 2: Operation

During operation, client and server exchange application-level messages within the bounds of their negotiated capabilities. Either party may send requests, responses, or notifications. The server may send **unsolicited notifications**:
- `notifications/tools/list_changed` — tool list has changed at runtime
- Progress updates for long-running tool calls

### Phase 3: Shutdown

Shutdown is initiated by the client: stop sending new requests → wait for in-flight requests to complete → close transport.

> ⚠️ **MCP does not define a session resumption protocol. A broken connection requires full re-initialization. Design tool calls to be idempotent where possible and implement retry logic at the host layer. Do not assume the transport layer will recover silently.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-LIFE-001` | The three-step initialization handshake MUST complete before any application-level message is sent — skipping capability negotiation is a protocol violation. |
| `MCP-LIFE-002` | Tool calls MUST be designed as idempotent operations where semantically possible — MCP's absence of session resumption means re-execution on reconnect must be safe. |
| `MCP-LIFE-003` | The host MUST implement retry logic at the connection layer — a transport failure MUST trigger re-initialization, not a silent hang. |
| `MCP-LIFE-004` | The client MUST track in-flight request IDs and cancel or time out any request whose originating connection was terminated. |

### Java + LangGraph Target

```java
public class McpClientConnection {
    private final McpTransport transport;
    private McpCapabilityContract negotiatedContract;
    private final Map<String, CompletableFuture<JsonRpcResponse>> inFlight = new ConcurrentHashMap<>();

    // MCP-LIFE-001: initialization before any operation
    public void initialize(McpClientCapabilities clientCaps) {
        JsonRpcResponse response = transport.send(
            JsonRpcRequest.of("initialize", Map.of(
                "protocolVersion", MCP_VERSION,
                "capabilities", clientCaps.toJson()
            ))
        );
        this.negotiatedContract = McpCapabilityContract.from(response);
        transport.sendNotification("notifications/initialized", Map.of());
    }

    public ToolCallResult call(String toolName, Map<String, Object> args) {
        assertCapability("tools"); // MCP-LIFE-001: only use negotiated capabilities
        String requestId = UUID.randomUUID().toString();
        CompletableFuture<JsonRpcResponse> future = new CompletableFuture<>();
        inFlight.put(requestId, future);
        try {
            transport.send(JsonRpcRequest.of("tools/call", Map.of(
                "name", toolName, "arguments", args
            ), requestId));
            return ToolCallResult.from(future.get(negotiatedContract.timeoutMs(), MILLISECONDS));
        } catch (TimeoutException e) {
            // MCP-LIFE-004: cancel in-flight on timeout
            inFlight.remove(requestId);
            return ToolCallResult.timeout(toolName);
        }
    }
}
```

---

## Section 20.5 — Server Primitives: Tools, Resources, and Prompts

### Three Primitives, Three Roles

MCP servers expose capabilities through three primitives. Each has a distinct role — conflating them produces poorly designed servers:

| Primitive | Controlled By | Read/Write | Primary Use | MCP Methods |
|---|---|---|---|---|
| **Tools** | Model (via host) | Write — has side effects | Execute actions (query DB, call API, run code, send message) | `tools/list`, `tools/call` |
| **Resources** | Host application | Read only | Inject context into LLM (open document, DB schema, config file, code repo) | `resources/list`, `resources/read`, `resources/subscribe` |
| **Prompts** | User (explicit) | Read only | Initiate workflows (slash-command-style templates, reusable interaction patterns) | `prompts/list`, `prompts/get` |

### Tools: Model-Controlled Actions

A tool has a **name**, a **description**, and an **input schema** (JSON Schema). When the model decides to use a tool, the host calls `tools/call` with the tool name and arguments; the server executes the function and returns a result (text, structured data, images, or embedded resources).

Servers declare available tools in response to `tools/list` and may send `notifications/tools/list_changed` when the list changes at runtime.

### Resources: Application-Controlled Context

Resources are **read-only data objects** the host retrieves to inject into the model's context. A resource has a URI, MIME type, and content. The model may suggest reading a resource, but the **host executes the retrieval** — resources are application-controlled, not model-controlled.

Servers may expose **resource templates** — URI patterns with variables (e.g., `file://{path}` or `db://{table}/{row_id}`) that allow clients to request parameterized resources.

### Prompts: User-Controlled Templates

Prompts are **reusable interaction templates** the user explicitly selects and invokes. A prompt has a name, description, optional input parameters, and generates a structured sequence of messages. They serve two purposes:
- Encode best-practice interaction patterns centrally in the server — updates propagate to all clients without code changes
- Provide slash-command-style UX in host applications

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-PRIM-001` | Tools MUST only be used for operations with side effects (actions) — read-only context injection belongs in Resources. |
| `MCP-PRIM-002` | Resources MUST be read-only from the protocol perspective — a Resource that mutates state is a design defect; use a Tool instead. |
| `MCP-PRIM-003` | Tool descriptions MUST be written for LLM consumption — they are the primary signal the model uses for tool selection and MUST accurately describe what the tool does, when to use it, and what it returns. |
| `MCP-PRIM-004` | Tool input schemas MUST use JSON Schema with explicit `required` fields and `description` on every property — incomplete schemas produce incorrect LLM argument generation. |
| `MCP-PRIM-005` | Servers MUST send `notifications/tools/list_changed` when the tool list changes at runtime — clients that do not receive this notification will operate against a stale tool catalog. |

### Java + LangGraph Target

```java
// MCP-PRIM-003 + MCP-PRIM-004: well-described tool definition
public class KnowledgeBaseSearchTool implements McpTool {

    @Override
    public McpToolDefinition definition() {
        return McpToolDefinition.builder()
            .name("kb.search")
            .description("""
                Search the internal knowledge base for documents matching a query.
                Use when the user asks about company policies, product documentation,
                procedures, or any internal reference material.
                Returns ranked document excerpts with source references.
                """)
            .inputSchema(JsonSchema.object()
                .required("query")
                .property("query", JsonSchema.string()
                    .description("Natural language search query"))
                .property("top_k", JsonSchema.integer()
                    .description("Maximum number of results to return")
                    .defaultValue(5))
                .build())
            .build();
    }

    @Override
    public McpToolResult execute(Map<String, Object> args) {
        String query = (String) args.get("query");
        int topK = (int) args.getOrDefault("top_k", 5);
        List<SearchResult> results = knowledgeBase.search(query, topK);
        return McpToolResult.text(formatResults(results));
    }
}
```

---

## Section 20.6 — Client Primitives: Sampling, Roots, and Elicitation

### An Asymmetric Protocol with Inverse Capabilities

MCP is primarily client-to-server, but defines three **server-to-client capabilities** that invert the flow — allowing the server to request services from the client:

| Primitive | Direction | Purpose | Security Concern |
|---|---|---|---|
| **Sampling** | Server → Client | Request an LLM completion from the host (without the server needing model API credentials) | Capability escalation: unconstrained sampling allows server to make arbitrary LLM calls |
| **Roots** | Client → Server | Client declares which filesystem paths/URIs the server is permitted to access | Convention only — not OS-level enforcement; must be combined with sandboxing |
| **Elicitation** | Server → Client | Server requests specific structured input from the human user, mediated through the host | Must be governed — servers should not elicit sensitive input outside the interaction contract |

### Sampling: Server-Initiated Model Calls

The flow:
1. Server sends `sampling/createMessage` request to client
2. Client presents request to human user for approval (interactive) OR applies host policy (automated)
3. If approved, client runs the completion, reviews the result, returns it to server

This design preserves the human-in-the-loop contract — **servers never have direct model access**. The November 2025 specification extended sampling to support tool definitions and parallel tool calls, enabling server-side agent loops.

> ⚠️ **Sampling creates a capability escalation risk if the host applies no policy. A malicious or compromised MCP server could make arbitrary LLM calls at the host's expense, exfiltrate context, or manipulate the model's reasoning. Always enforce a sampling policy at the host layer: rate limits, content inspection, and explicit approval for sensitive operations.**

### Roots: Filesystem Scope Declaration

Roots are a **protocol-level convention**, not an OS-level security boundary. A well-behaved server respects declared roots; a malicious server ignores them. For strong filesystem isolation, roots **must be combined with OS-level sandboxing** (Volume 2, Chapter 6).

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-CLIENT-001` | The host MUST enforce an explicit sampling policy before forwarding any `sampling/createMessage` request — rate limits, content inspection, and scope boundaries MUST be applied. |
| `MCP-CLIENT-002` | Sampling MUST NOT be enabled without explicit policy configuration — the default MUST be to reject all sampling requests until a policy is defined. |
| `MCP-CLIENT-003` | Roots declarations MUST be combined with OS-level process sandboxing for any server that accesses the filesystem — roots alone are not a security boundary. |
| `MCP-CLIENT-004` | Elicitation requests MUST be rendered to the user through the host's native UI — servers MUST NOT receive raw user credentials or sensitive data through the elicitation channel without explicit host-level validation. |

---

## Section 20.7 — Security and Authorization

### Local vs. Remote Security Postures

| Transport | Security Model | Primary Threat |
|---|---|---|
| **STDIO** | Process-level (implicit) — server runs as child of host, inherits OS permissions | What the server process can do on the host machine (filesystem, network, credentials) |
| **Streamable HTTP** | Explicit — OAuth 2.1 required; server is a network service accessible by multiple clients | Authentication bypass, token misuse, scope escalation |

### OAuth 2.1 for Remote Deployments

The June 2025 and November 2025 specification updates hardened MCP's authorization model. Remote MCP servers **MUST** implement **OAuth 2.1**:

- Clients use OAuth access tokens as bearer credentials in HTTP request headers
- **Dynamic Client Registration** — clients can register themselves at connection time with an OIDC-compatible identity provider (Okta, Entra, custom authorization server) without out-of-band pre-registration
- **Resource Indicators (RFC 8707)** — tokens are bound to a specific resource server, preventing a token issued for one MCP server from being used against a different server

### Capability Confinement via OAuth Scopes

OAuth handles authentication (who is this client?) and coarse-grained authorization (is this client allowed to connect?). **Fine-grained capability confinement** is implemented through **OAuth scopes**:

| Scope Example | Permission |
|---|---|
| `tools:read` | May list tools, may not call them |
| `tools:execute:search` | May call the `search` tool only |
| `resources:read:docs` | May read documentation resources |
| `sampling:disabled` | Sampling requests rejected |

Designing a coherent scope taxonomy is a significant architecture decision for MCP deployments serving multiple clients with different permission levels.

### The Four Most Common Production Security Gaps

| Gap | Description | Fix |
|---|---|---|
| **Absent token validation** | Server accepts bearer tokens without verifying signature, expiry, or issuer | Validate every token: signature, expiry, issuer, audience |
| **Missing scope enforcement** | Token is validated but scopes not checked before executing tool calls — every authenticated client gets full access | Enforce scopes on every operation |
| **Unrestricted sampling** | Host applies no policy to `sampling/createMessage` — any connected server makes unconstrained model calls | Apply host-level sampling policy (MCP-CLIENT-001) |
| **Overprivileged STDIO servers** | Local servers run with full host application OS privileges rather than in a restricted sandbox | Combine roots declarations with OS sandboxing |

> 💡 **Treat every MCP server — including local ones — as a potential attack surface. Validate tokens completely. Enforce scopes on every operation. Apply a sampling policy at the host. Sandbox STDIO servers. None of these are optional for production deployments.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-SEC-001` | Remote MCP servers MUST implement OAuth 2.1 with dynamic client registration and Resource Indicators (RFC 8707). |
| `MCP-SEC-002` | Every OAuth access token MUST be validated for signature, expiry, issuer, and audience before any operation is executed. |
| `MCP-SEC-003` | OAuth scopes MUST be enforced on every operation — a valid token without the required scope MUST be rejected with HTTP 403. |
| `MCP-SEC-004` | The scope taxonomy MUST be documented as an architecture artifact and reviewed whenever new tools or client types are added. |
| `MCP-SEC-005` | STDIO servers MUST be sandboxed with OS-level privilege restriction in production — running with host-application-level privileges is a security defect. |

### Java + LangGraph Target

```java
// MCP-SEC-002 + MCP-SEC-003: token validation and scope enforcement
public class McpAuthorizationFilter implements McpRequestFilter {
    private final OAuthTokenValidator tokenValidator;
    private final ScopeEnforcer scopeEnforcer;

    @Override
    public McpRequestContext authorize(McpRequestContext ctx) {
        String bearerToken = ctx.httpHeaders().get("Authorization")
            .replace("Bearer ", "");

        // MCP-SEC-002: full token validation
        OAuthTokenClaims claims = tokenValidator.validate(bearerToken,
            expectedIssuer, expectedAudience);

        // MCP-SEC-003: scope enforcement per operation
        String requiredScope = scopeForOperation(ctx.method(), ctx.toolName());
        scopeEnforcer.assertScope(claims.scopes(), requiredScope);

        return ctx.withAuthenticatedIdentity(claims.clientId());
    }

    private String scopeForOperation(String method, String toolName) {
        return switch (method) {
            case "tools/list"  -> "tools:read";
            case "tools/call"  -> "tools:execute:" + toolName;
            case "resources/read" -> "resources:read";
            default -> throw new UnsupportedOperationException(method);
        };
    }
}
```

---

## Section 20.8 — Production Considerations

### Protocol Versioning

MCP uses **date-based version identifiers** (e.g., `2025-11-25`). The client declares its version in `initialize`; the server responds with the version it will use for the session.

**Version drift risk:** Upgrading an MCP server to a new protocol version may break clients that have not yet been updated. Servers that support multiple protocol versions simultaneously (responding at the version each client declared) provide a safer upgrade path but require more implementation complexity.

| Rule ID | Rule |
|---|---|
| `MCP-VER-001` | Protocol version MUST be pinned in production deployments and upgraded deliberately through a tested migration path. |
| `MCP-VER-002` | Shared MCP servers serving multiple client versions MUST implement multi-version support to allow safe rolling upgrades. |

### Observability at the Protocol Boundary

MCP does not define a built-in tracing protocol, but the JSON-RPC message envelope provides natural instrumentation points:
- Every request has a unique ID that can be propagated as a trace ID into downstream observability systems
- Every tool call, resource read, and sampling request is a discrete message that can be logged at the client or server

> **The correct observability posture: log every protocol message at the client layer** — where the full request-response pair is visible. Server-side logging alone is insufficient because it cannot capture messages that fail to reach the server.

| Rule ID | Rule |
|---|---|
| `MCP-OBS-001` | Every MCP request and response MUST be logged at the client layer with: trace ID, tool name, sanitized arguments, latency, and outcome. |
| `MCP-OBS-002` | MCP request IDs MUST be propagated as trace context headers into all downstream observability systems. |
| `MCP-OBS-003` | Per-tool latency histograms (p50, p95, p99) MUST be maintained — silent latency degradation is the primary early signal of tool backend failure. |

### Error Handling

JSON-RPC 2.0 defines a standard error object with a numeric code and message string. MCP extends this with application-level error codes for protocol-specific failures.

> 💡 **Do not tunnel domain-level errors through JSON-RPC error responses. Return domain errors as structured content within a successful tool result, and reserve JSON-RPC errors for protocol-level failures (malformed request, unsupported method, unauthorized).**

| Error Class | Use JSON-RPC Error? | Correct Approach |
|---|---|---|
| Protocol-level failure (malformed request, unsupported method) | ✅ Yes | JSON-RPC error object |
| Authorization failure | ✅ Yes | JSON-RPC error with code -32001 |
| Domain-level failure (search returned no results) | ❌ No | Structured content in successful tool result |
| Tool execution error (API downstream error) | ❌ No | Structured error envelope in tool result |

---

## Section 20.9 — The 2026 MCP Roadmap

### Four Architectural Directions Engineers Must Track

| Direction | What Changes | Engineering Impact |
|---|---|---|
| **Stateless transport** | Current Streamable HTTP is stateful (persistent bidirectional channel). Roadmap moves toward stateless transport where each request carries enough context for any server instance to process it. Sessions become application-data-layer constructs (analogous to cookies), not transport-layer state. | Eliminates sticky-session requirements; enables true horizontal auto-scaling on managed cloud infrastructure and serverless platforms. Design MCP servers today with this transition in mind. |
| **Server Cards (`.well-known/mcp.json`)** | Today a client must complete a full initialization handshake to learn what an MCP server offers. Server Cards expose capabilities, auth requirements, and primitives as a static document at a well-known URL — discoverable without a live connection. | Enables pre-connection capability checks, security validation, and registry indexing. Connects MCP to Layer 3 discovery infrastructure (chap22). Design server metadata to be machine-readable and complete now. |
| **Native task management (Tasks primitive)** | `Tasks` primitive (SEP-1686) allows MCP servers to initiate, track, and complete multi-step work items rather than only responding to single-shot calls. With retry semantics and expiry policies being developed by the Agent Communication Working Group, MCP servers are evolving toward autonomous task participants. | Narrows the boundary between Layer 1 tool access and Layer 2 agent collaboration. Systems that cleanly separate tool servers from agent collaborators today may need to accommodate servers that play both roles. |
| **Enterprise extensions** | Audit trails, SSO-integrated authorization, gateway-compatible behavior, and configuration portability addressed through the extensions ecosystem (not the core specification). | Engineers in regulated industries should monitor the Enterprise Working Group and plan to adopt relevant extensions as they stabilize. Core MCP stays lean; enterprise behavior is layered on. |

> 💡 **The single most impactful near-term change to watch is Server Cards. When `.well-known/mcp.json` becomes a standard discovery endpoint, automated tooling for MCP server discovery, cataloguing, and security scanning will follow quickly. Design your MCP server metadata to be machine-readable and complete now.**

### Design Rules

| Rule ID | Rule |
|---|---|
| `MCP-ROAD-001` | New MCP server deployments MUST be designed for stateless transport from the start — session state MUST be stored in application-layer data structures, not in transport-layer connection affinity. |
| `MCP-ROAD-002` | Every production MCP server MUST expose a `.well-known/mcp.json` Server Card — even before the formal specification mandates it. |
| `MCP-ROAD-003` | Systems integrating MCP servers and agent collaborators MUST track the Tasks primitive SEP-1686 evolution and plan for servers that operate at both Layer 1 and Layer 2. |

---

## Section 20.10 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `MCP-ARCH-001` | Architecture | MCP is the Layer 1 standard; bespoke adapters are architectural debt |
| `MCP-ARCH-002` | Architecture | O(n+m) composability target; shared servers preferred |
| `MCP-ARCH-003` | Architecture | MCP is NOT for Layer 2 (agent-to-agent) |
| `MCP-ROLE-001` | Roles | Host mediates all MCP interactions — no bypass |
| `MCP-ROLE-002` | Roles | One client per server connection |
| `MCP-ROLE-003` | Roles | Host enforces user-level permissions before dispatch |
| `MCP-TRANS-001` | Transport | Only STDIO or Streamable HTTP in production |
| `MCP-TRANS-002` | Transport | STDIO not in containerized production |
| `MCP-TRANS-003` | Transport | Streamable HTTP designed for stateless scaling |
| `MCP-TRANS-004` | Transport | No JSON-RPC batch requests (removed June 2025) |
| `MCP-LIFE-001` | Lifecycle | Three-step handshake before any operation |
| `MCP-LIFE-002` | Lifecycle | Tool calls idempotent where semantically possible |
| `MCP-LIFE-003` | Lifecycle | Host implements reconnect-and-reinitialize on transport failure |
| `MCP-LIFE-004` | Lifecycle | Client cancels in-flight requests on connection termination |
| `MCP-PRIM-001` | Primitives | Tools for side-effecting actions only |
| `MCP-PRIM-002` | Primitives | Resources are read-only |
| `MCP-PRIM-003` | Primitives | Tool descriptions written for LLM consumption |
| `MCP-PRIM-004` | Primitives | Tool schemas use JSON Schema with full property descriptions |
| `MCP-PRIM-005` | Primitives | Servers send `tools/list_changed` on runtime changes |
| `MCP-CLIENT-001` | Client primitives | Explicit sampling policy enforced at host |
| `MCP-CLIENT-002` | Client primitives | Default: reject all sampling until policy defined |
| `MCP-CLIENT-003` | Client primitives | Roots + OS sandboxing for filesystem access |
| `MCP-CLIENT-004` | Client primitives | Elicitation validated at host before data reaches server |
| `MCP-SEC-001` | Security | OAuth 2.1 + dynamic client registration + RFC 8707 for remote |
| `MCP-SEC-002` | Security | Full token validation (signature, expiry, issuer, audience) |
| `MCP-SEC-003` | Security | Scope enforcement on every operation |
| `MCP-SEC-004` | Security | Scope taxonomy documented as architecture artifact |
| `MCP-SEC-005` | Security | STDIO servers OS-sandboxed in production |
| `MCP-VER-001` | Versioning | Protocol version pinned in production |
| `MCP-VER-002` | Versioning | Multi-version support for shared servers |
| `MCP-OBS-001` | Observability | Every request/response logged at client layer |
| `MCP-OBS-002` | Observability | Request IDs propagated as trace context |
| `MCP-OBS-003` | Observability | Per-tool latency histograms (p50/p95/p99) |
| `MCP-ROAD-001` | Roadmap | New servers designed for stateless transport |
| `MCP-ROAD-002` | Roadmap | Every server exposes `.well-known/mcp.json` |
| `MCP-ROAD-003` | Roadmap | Track Tasks primitive (SEP-1686) for dual Layer 1/2 evolution |

---

## Section 20.11 — Transition to Chapter 21

Chapter 20 has fully covered Layer 1 — the protocol between an agent and its tools. That layer is **asymmetric by design**: the agent drives, the tool responds, the relationship is well-bounded.

Chapter 21 moves to **Layer 2: Agent Collaboration** — where both parties reason, plan, and act. Either may initiate. Either may delegate. Either may reject, renegotiate, or fail independently. The coordination challenges are of a fundamentally different order:

| Concern | Layer 1 (MCP) | Layer 2 (A2A / ANP) |
|---|---|---|
| Who initiates | Client (agent) always | Either party |
| Who reasons | Agent only | Both parties |
| Task duration | Milliseconds to seconds | Seconds to hours |
| Organizational boundary | Typically same org | May cross orgs |
| Discovery | Server URL / Server Card | Agent Card / DID |
| Trust model | OAuth 2.1 scopes | OAuth 2.1 + cryptographic identity |
| State management | Stateless preferred | Task lifecycle required |

> Chapter 21 covers the Agent2Agent Protocol (A2A) — its three-layer specification architecture (canonical proto model, abstract operations, protocol bindings), Agent Cards, the full task lifecycle, the A2A/ACP merger history, and the Agent Network Protocol (ANP) for open-network agent collaboration.
