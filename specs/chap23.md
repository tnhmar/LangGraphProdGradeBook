# Chapter 23 Spec — Observability Across the Protocol Stack

> **Volume 1 reference:** Part 6 — The Framework Landscape 2026 (intro chapter); cross-cutting observability concerns referenced in chap19–22 and chap32
> **Part:** 6 — The Framework Landscape 2026
> **Depends on:** chap20 (`MCP-ARCH-001–005`), chap21 (`A2A-CARD-001–004`, `A2A-SEC-001–005`), chap22 (`L3-GW-001–006`, `L3-GOV-001–006`)
> **Java + LangGraph implementation target:** Volume 2 — Chapter 12: Distributed Tracing, Structured Logging, and OTel
> **Bridges to:** chap24–29 (per-framework observability depth), chap33 (enterprise audit trail), chap34 (regulatory compliance logging)

> *"Emit all metrics through OpenTelemetry from day one. The trace ID on every log event and the span context on every metric emission are the integration points for the distributed tracing architecture. Retrofitting OTel instrumentation onto a production tool layer is significantly harder than building it in from the start."*
> — Volume 1, Chapter 32

---

## Chapter Purpose

Observability is not a post-deployment concern. In agent systems it is a **structural requirement** that must be designed into every layer from the beginning — because the failure modes of agents are fundamentally different from those of conventional software.

A conventional HTTP service fails loudly: it returns 500, it times out, it panics. An agent system can fail silently: the LLM produces plausible-sounding output that calls the wrong tool, the tool returns a valid-shaped result that contains subtly wrong data, the agent loops 47 times before timing out, or a prompt injection attack redirects the agent's behavior while all metrics remain green.

**Conventional monitoring is necessary but not sufficient for agent systems.** This chapter defines the full observability architecture for a production agent system operating across all three protocol layers: MCP (Layer 1), A2A (Layer 2), and the Agent Gateway / Agent Web (Layer 3).

The chapter is organized around four pillars:
1. **Distributed tracing** — correlating spans across LangGraph nodes, MCP tool calls, A2A task delegations, and gateway traversals into a single end-to-end trace
2. **Structured logging** — the mandatory log events at each protocol boundary and agent lifecycle stage
3. **Metrics** — the metric families that capture aggregate behavior and drive alerting
4. **Behavioral signals** — the agent-specific observability concerns that have no analog in conventional service monitoring

---

## Section 23.1 — Why Agent Observability Is Architecturally Distinct

### The Silent Failure Problem

Conventional service monitoring answers three questions: Is it up? Is it fast? Is it returning errors? Agent systems can be **fully up, reasonably fast, and returning no errors while producing completely wrong outputs.**

The failure modes unique to agent systems include:

| Failure Mode | Why Conventional Monitoring Misses It |
|---|---|
| **Reasoning loop** | Agent repeatedly calls the same tool or same LLM step — metrics show normal call volume until timeout |
| **Prompt injection** | External content redirects agent behavior — all tool calls succeed with valid outputs, goal is silently altered |
| **Token budget exhaustion** | Agent truncates reasoning mid-task, returns partial result — HTTP 200, no errors emitted |
| **Tool selection drift** | LLM starts choosing wrong tools after model update — individual calls succeed, aggregate behavior degrades |
| **Context window overflow** | Agent silently drops earlier reasoning steps — outputs are coherent but factually incorrect |
| **Silent A2A delegation failure** | Remote subagent returns `completed` status with an artifact that is a polite error message, not a real result |

### The Three-Layer Observability Problem

An agent system that spans all three protocol layers generates spans, logs, and metrics across:
- **LangGraph graph execution** — node entry/exit, state transitions, conditional edges
- **MCP client/server boundary** — `tools/call` dispatch, server-side execution, result delivery
- **A2A client/server boundary** — task creation, multi-turn updates, streaming events, task completion
- **Agent Gateway** — inbound/outbound policy decisions, delegation chain validation, rate limiting
- **LLM inference** — prompt construction, token counts, model latency, finish reason

Without explicit trace context propagation at every boundary, these are **five independent trace trees** that cannot be correlated during incident investigation. This is the single most expensive observability gap to fix after the fact.

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-STRUCT-001` | OpenTelemetry (OTel) MUST be the sole telemetry SDK for all agents, MCP clients/servers, A2A services, and gateways — no proprietary telemetry SDKs that cannot export to a standard OTel collector. |
| `OBS-STRUCT-002` | A single OTel collector endpoint MUST be configured across all frameworks and services in the system — independent collector instances per framework produce unconnected telemetry pipelines. |
| `OBS-STRUCT-003` | Observability instrumentation MUST be implemented before the first production deployment — retrofitting distributed tracing onto a running multi-framework system requires touching every boundary simultaneously. |
| `OBS-STRUCT-004` | Conventional service monitoring (uptime, latency, error rate) is necessary but not sufficient — behavioral signals (loop detection, token budget, tool selection accuracy) MUST be instrumented alongside infrastructure metrics. |

---

## Section 23.2 — Distributed Tracing Architecture

### The W3C Trace Context Standard

The foundation of cross-boundary trace correlation is the **W3C Trace Context** specification (`traceparent` header). Every agent interaction — regardless of which framework, transport, or protocol layer it crosses — must carry a `traceparent` header that encodes:

```
traceparent: 00-{traceId}-{spanId}-{flags}
```

- `traceId` — 16-byte globally unique identifier for the entire request chain (same value from user request to final tool call result)
- `spanId` — 8-byte identifier for the current span (changes at each boundary)
- `flags` — sampling decision bit

The `traceId` is the **correlation key** that the OTel collector uses to stitch all spans from all services into a single distributed trace tree.

### Span Hierarchy in an Agent System

A complete agent invocation produces a span tree with a defined hierarchy:

```
[ROOT SPAN] user-request (LangGraph host / agent runtime)
  ├── [SPAN] graph.node: InputProcessorNode
  ├── [SPAN] graph.node: LlmNode
  │     └── [SPAN] llm.inference (model API call)
  ├── [SPAN] graph.node: ToolDispatchNode
  │     ├── [SPAN] mcp.tool_call: kb.search        ← Layer 1
  │     └── [SPAN] mcp.tool_call: crm.getContact   ← Layer 1
  ├── [SPAN] graph.node: A2ADelegationNode
  │     └── [SPAN] a2a.task: financialResearch      ← Layer 2
  │           ├── [SPAN] graph.node: RemoteAgentLlmNode
  │           └── [SPAN] mcp.tool_call: analytics.historicalData
  └── [SPAN] graph.node: ResponseFormatterNode
```

**Key invariant:** The `traceId` is identical at every level of this tree. Only `spanId` changes at each boundary.

### Cross-Boundary Propagation Requirements

Trace context does **not** propagate automatically across protocol boundaries. It requires explicit injection and extraction at each crossing:

| Boundary | Inject At | Extract At | Header |
|---|---|---|---|
| **MCP HTTP call** | MCP client (before `tools/call` POST) | MCP server middleware (on request receipt) | `traceparent` |
| **A2A task submission** | A2A client (before task create/send) | A2A server executor (on task receive) | `traceparent` (HTTP header or A2A extension field) |
| **Agent Gateway (outbound)** | Internal agent before gateway call | Gateway middleware | `traceparent` |
| **Agent Gateway (inbound)** | Gateway after policy check (forward to internal) | Internal agent | `traceparent` |
| **LLM API call** | Agent before model API request | Not applicable (model is a leaf span) | `traceparent` to model provider if supported |

> ⚠️ **Trace context propagation across A2A and MCP boundaries is not automatic.** It requires explicit header injection and extraction at every boundary crossing. Without it, you have multiple independent traces that cannot be correlated during incident investigation. Implement trace propagation before going to production — not as a post-incident improvement.

### Sampling Strategy

Not every agent invocation should be traced at full fidelity in production — the volume is too high and the storage cost is significant. The correct sampling strategy for agent systems combines:

| Strategy | When to Use | Implementation |
|---|---|---|
| **Head-based sampling (rate)** | Default for routine invocations | OTel SDK probabilistic sampler (e.g., 10% of all requests) |
| **Tail-based sampling (error)** | Capture all failed invocations regardless of head decision | OTel Collector tail sampling processor: always sample if any span in trace has `error=true` |
| **Tail-based sampling (anomaly)** | Capture all invocations with behavioral anomalies | Always sample if loop counter > threshold, token budget exceeded, or tool selection mismatch detected |
| **Force-sample (debug)** | Targeted debugging of specific agents or sessions | `traceparent` with `flags=01` injected by debug tooling |

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-TRACE-001` | W3C `traceparent` MUST be propagated across every MCP, A2A, and gateway boundary — missing propagation at any single boundary breaks the end-to-end trace correlation. |
| `OBS-TRACE-002` | Every LangGraph node execution MUST produce a span with `node.name`, `node.input_schema_hash`, entry timestamp, exit timestamp, and outcome (`success` / `error` / `conditional_branch`). |
| `OBS-TRACE-003` | Every MCP `tools/call` span MUST include: `tool.name`, `tool.server_url`, `tool.input_schema_hash`, `tool.timeout_ms`, `span.status`, latency in ms, and `tool.result_status` (`success` / `timeout` / `error_code`). |
| `OBS-TRACE-004` | Every A2A task span MUST include: `a2a.task_id`, `a2a.remote_agent_url`, `a2a.skill`, `a2a.task_state` (final), `a2a.artifact_count`, and total wall-clock duration. |
| `OBS-TRACE-005` | Tail-based sampling MUST be configured to capture 100% of traces containing at least one error span, one loop-detection event, or one behavioral anomaly signal. |
| `OBS-TRACE-006` | The root span of every agent invocation MUST include `session.id`, `agent.id`, `agent.version`, and `user.id` (or tenant identifier) — these are the primary correlation keys for multi-session and multi-tenant debugging. |

### Java + LangGraph Target

```java
// Trace context injection at MCP boundary
public class TracedMcpClient {
    private final McpClient delegate;
    private final OpenTelemetry otel;

    public ToolResult callTool(String toolName, Map<String, Object> args, Context otelCtx) {
        Tracer tracer = otel.getTracer("mcp.client");
        Span span = tracer.spanBuilder("mcp.tool_call")
            .setParent(otelCtx)
            .setAttribute("tool.name", toolName)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Inject traceparent into MCP HTTP headers (OBS-TRACE-001)
            Map<String, String> headers = new HashMap<>();
            W3CTraceContextPropagator.getInstance()
                .inject(Context.current(), headers, Map::put);

            ToolResult result = delegate.callTool(toolName, args, headers);
            span.setAttribute("tool.result_status", result.status().name());
            return result;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}

// LangGraph node with OTel span (OBS-TRACE-002)
public class ObservableNode implements RunnableNode<AgentState> {
    private final Tracer tracer;
    private final String nodeName;
    private final RunnableNode<AgentState> delegate;

    @Override
    public AgentState run(AgentState state) {
        Span span = tracer.spanBuilder("graph.node")
            .setAttribute("node.name", nodeName)
            .setAttribute("session.id", state.sessionId())      // OBS-TRACE-006
            .setAttribute("agent.id", state.agentId())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            AgentState result = delegate.run(state);
            span.setStatus(StatusCode.OK);
            return result;
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

## Section 23.3 — Structured Logging

### Why Structured Logging Is Mandatory

Free-text log lines are not sufficient for agent systems. The failure investigation workflow for agents requires querying logs by:
- `traceId` — "show me everything that happened in this invocation"
- `toolName` + `errorCode` — "show me all TOOLTIMEOUT events for kb.search in the last 24 hours"
- `agentId` + `loopCount` — "show me all sessions where the agent looped more than 5 times"
- `taskState` — "show me all A2A tasks that completed with `failed` status"

None of these queries are possible on unstructured text. **Every log event emitted by the agent system must be a structured JSON object with a defined schema.**

### Mandatory Log Events by Layer

#### Layer 0 — Agent Lifecycle

| Event Name | Trigger | Mandatory Fields |
|---|---|---|
| `agent.session.started` | New agent session begins | `traceId`, `sessionId`, `agentId`, `agentVersion`, `userId`, `timestamp` |
| `agent.session.completed` | Session ends (success or failure) | + `outcome`, `totalDurationMs`, `totalTokensUsed`, `nodeCount`, `toolCallCount` |
| `agent.loop.detected` | Same node/tool called N times in single session | + `loopNodeName`, `loopCount`, `loopThreshold` |
| `agent.goal.abandoned` | Agent exits without achieving stated goal | + `abandonReason`, `lastNodeName`, `stepsCompleted` |

#### Layer 1 — MCP Tool Calls

| Event Name | Trigger | Mandatory Fields |
|---|---|---|
| `mcp.tool.dispatched` | Tool call exits abstraction layer toward MCP server | `traceId`, `sessionId`, `toolName`, `serverUrl`, `inputSchemaHash`, `timeoutMs`, `timestamp` |
| `mcp.tool.completed` | Tool call result received | + `resultStatus`, `durationMs`, `outputSchemaHash` |
| `mcp.tool.failed` | Tool call returns error | + `errorCode`, `retryable`, `retryAttempt` |
| `mcp.circuit_breaker.opened` | Circuit breaker trips for a tool | `traceId`, `toolName`, `failureCount`, `threshold`, `cooldownMs` |

#### Layer 2 — A2A Task Delegation

| Event Name | Trigger | Mandatory Fields |
|---|---|---|
| `a2a.task.submitted` | Task submitted to remote A2A agent | `traceId`, `sessionId`, `taskId`, `remoteAgentUrl`, `skill`, `timestamp` |
| `a2a.task.status_update` | `TaskStatusUpdateEvent` received | + `taskState`, `progressMessage`, `artifactCount` |
| `a2a.task.completed` | Task reaches terminal state | + `finalState`, `totalDurationMs`, `artifactCount`, `artifactTypes` |
| `a2a.task.failed` | Task reaches `failed` or `cancelled` state | + `failureReason`, `lastKnownState` |

#### Layer 3 — Gateway Events

| Event Name | Trigger | Mandatory Fields |
|---|---|---|
| `gateway.request.permitted` | PDP permits a cross-boundary request | `traceId`, `agentId`, `requestedOperation`, `policyVersion`, `timestamp` |
| `gateway.request.rejected` | PDP rejects a cross-boundary request | + `rejectionReason`, `policyRule` |
| `gateway.chain.validated` | Delegation chain validated successfully | + `chainLength`, `chainOrigin`, `tokenIssuers` |
| `gateway.chain.rejected` | Delegation chain validation fails | + `rejectionPoint`, `rejectionReason` |

#### LLM Inference

| Event Name | Trigger | Mandatory Fields |
|---|---|---|
| `llm.inference.started` | LLM API call dispatched | `traceId`, `sessionId`, `modelId`, `promptTokens`, `maxTokens`, `timestamp` |
| `llm.inference.completed` | LLM API call returns | + `completionTokens`, `totalTokens`, `finishReason`, `durationMs` |
| `llm.inference.failed` | LLM API call errors | + `errorCode`, `retryable` |

### Log Correlation Requirements

Every log event MUST carry `traceId` and `sessionId`. This is the **minimum correlation contract** — it enables the query "show me all events from this trace" and "show me all events from this session" without any other join key.

> 💡 **Inject `traceId` from the OTel span context, not from a separate logging context.** When the OTel span is the source of truth for `traceId`, the value in logs automatically matches the value in traces — no synchronization required.

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-LOG-001` | All log events MUST be structured JSON with `traceId`, `sessionId`, `agentId`, `timestamp` (ISO 8601), and `eventName` as mandatory fields on every event. |
| `OBS-LOG-002` | `traceId` in log events MUST be sourced from the active OTel span context — not from a separate request-scoped logging variable. |
| `OBS-LOG-003` | The mandatory log events defined in Section 23.3 MUST be emitted at every corresponding lifecycle trigger — missing a mandatory event is a observability gap, not an optimization. |
| `OBS-LOG-004` | Tool call arguments and LLM prompt content MUST NOT be logged in full at the default log level — they may contain PII or secrets. Log `inputSchemaHash` and `promptTokens` at INFO; log sanitized argument summaries only at DEBUG. |
| `OBS-LOG-005` | Log events MUST be shipped to the same observability backend as traces and metrics — siloed log storage that cannot be correlated with trace data defeats the purpose of structured logging. |

### Java + LangGraph Target

```java
@Component
public class AgentStructuredLogger {
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private final Logger log = LoggerFactory.getLogger(AgentStructuredLogger.class);

    // OBS-LOG-001: structured JSON with mandatory correlation fields
    public void logToolDispatched(AgentState state, String toolName, String serverUrl,
                                   String inputSchemaHash, long timeoutMs) {
        Map<String, Object> event = new LinkedHashMap<>();
        event.put("eventName", "mcp.tool.dispatched");
        event.put("traceId", currentTraceId());            // OBS-LOG-002: from OTel span
        event.put("sessionId", state.sessionId());
        event.put("agentId", state.agentId());
        event.put("timestamp", Instant.now().toString());
        event.put("toolName", toolName);
        event.put("serverUrl", serverUrl);
        event.put("inputSchemaHash", inputSchemaHash);     // OBS-LOG-004: hash not args
        event.put("timeoutMs", timeoutMs);
        log.info(MAPPER.writeValueAsString(event));
    }

    public void logLoopDetected(AgentState state, String loopNodeName,
                                 int loopCount, int threshold) {
        Map<String, Object> event = new LinkedHashMap<>();
        event.put("eventName", "agent.loop.detected");
        event.put("traceId", currentTraceId());
        event.put("sessionId", state.sessionId());
        event.put("agentId", state.agentId());
        event.put("timestamp", Instant.now().toString());
        event.put("loopNodeName", loopNodeName);
        event.put("loopCount", loopCount);
        event.put("loopThreshold", threshold);
        log.warn(MAPPER.writeValueAsString(event));        // loop = WARN level
    }

    private String currentTraceId() {
        Span span = Span.current();
        return span.getSpanContext().isValid()
            ? span.getSpanContext().getTraceId()
            : "no-active-span";
    }
}
```

---

## Section 23.4 — Metrics

### The Five Essential Metric Families

Metrics capture aggregate behavior over time — they are the signals that drive alerting and capacity planning. Five metric families cover the essential observability surface of a production agent system.

#### 1. Tool Execution Metrics (Layer 1)

| Metric Name | Type | Labels | Purpose |
|---|---|---|---|
| `agent_tool_call_duration_ms` | Histogram | `tool_name`, `server_url`, `status` | p50/p95/p99 latency per tool; detect backend degradation before error rates rise |
| `agent_tool_call_total` | Counter | `tool_name`, `status`, `error_code` | Total call volume; error rate by code (TOOLTIMEOUT vs INVALIDARGS vs UNAUTHORIZED) |
| `agent_tool_call_inflight` | Gauge | `tool_name` | In-flight calls; detect tool call pile-up from slow backends |
| `agent_tool_cache_hit_ratio` | Gauge | `tool_name` | Cache hit fraction when caching is active; anomalously low = TTL too aggressive |
| `agent_tool_circuit_breaker_state` | Gauge | `tool_name` | 0=closed, 1=half-open, 2=open; alert when any tool CB opens |

#### 2. A2A Task Metrics (Layer 2)

| Metric Name | Type | Labels | Purpose |
|---|---|---|---|
| `agent_a2a_task_duration_ms` | Histogram | `remote_agent`, `skill`, `final_state` | End-to-end task duration by remote agent and outcome |
| `agent_a2a_task_total` | Counter | `remote_agent`, `skill`, `final_state` | Task volume and success/failure rates per remote agent |
| `agent_a2a_task_inflight` | Gauge | `remote_agent` | In-flight delegated tasks; detect remote agent bottlenecks |
| `agent_a2a_turn_count` | Histogram | `remote_agent`, `skill` | Number of multi-turn exchanges per task; spikes indicate remote agent confusion |

#### 3. LLM Inference Metrics

| Metric Name | Type | Labels | Purpose |
|---|---|---|---|
| `agent_llm_inference_duration_ms` | Histogram | `model_id`, `finish_reason` | Inference latency by model and finish reason |
| `agent_llm_tokens_total` | Counter | `model_id`, `token_type` (`prompt`/`completion`) | Token consumption for cost attribution and budget alerting |
| `agent_llm_finish_reason_total` | Counter | `model_id`, `finish_reason` | Distribution of `stop` / `length` / `tool_calls` / `content_filter`; spikes in `length` signal truncation |
| `agent_llm_error_total` | Counter | `model_id`, `error_code` | API error rate; rate limit hits vs. server errors |

#### 4. Agent Behavioral Metrics

| Metric Name | Type | Labels | Purpose |
|---|---|---|---|
| `agent_session_duration_ms` | Histogram | `agent_id`, `outcome` | End-to-end session duration by outcome; detect runaway sessions |
| `agent_loop_detection_total` | Counter | `agent_id`, `loop_node` | Loop events by agent and node; systematic looping = prompt or tool design issue |
| `agent_tool_calls_per_session` | Histogram | `agent_id` | Tool invocations per session; anomalous high values signal reasoning loops or injection |
| `agent_goal_abandonment_total` | Counter | `agent_id`, `abandon_reason` | Sessions that exit without achieving goal; distinguish timeout, error, and policy abandonment |
| `agent_token_budget_exhausted_total` | Counter | `agent_id` | Sessions where token limit was reached before task completion |

#### 5. Gateway Metrics (Layer 3)

| Metric Name | Type | Labels | Purpose |
|---|---|---|---|
| `gateway_request_total` | Counter | `direction` (inbound/outbound), `outcome` (permitted/rejected) | Cross-boundary request volume and rejection rate |
| `gateway_rejection_total` | Counter | `direction`, `rejection_reason` | Rejection breakdown by reason (auth failure, scope violation, chain invalid) |
| `gateway_rate_limit_total` | Counter | `agent_id`, `direction` | Rate limit events per agent; spikes signal misbehavior or attack |
| `gateway_chain_validation_duration_ms` | Histogram | `chain_length` | Delegation chain validation latency; important for multi-hop performance budgeting |

### Alerting Rules

The following alert conditions represent the minimum viable alerting surface for a production agent system:

| Alert Name | Condition | Severity | Interpretation |
|---|---|---|---|
| `ToolCircuitBreakerOpen` | `agent_tool_circuit_breaker_state{state="open"} > 0` | P1 | A tool is unavailable; agent functionality impaired |
| `ToolLatencyP99Spike` | p99 of `agent_tool_call_duration_ms` > 3× baseline for > 5 min | P2 | Backend degradation beginning; not yet causing errors |
| `AgentLoopRateHigh` | `rate(agent_loop_detection_total[5m]) > threshold` | P2 | Systematic reasoning loop; likely prompt or tool design issue |
| `LlmTruncationRateHigh` | `rate(agent_llm_finish_reason_total{finish_reason="length"}[10m]) > 5%` | P2 | Agents hitting token limits; context management issue |
| `GatewayRejectionSpike` | `rate(gateway_rejection_total[5m]) > 3× baseline` | P1 | Policy violation spike; possible attack or misconfiguration |
| `A2ATaskFailureRateHigh` | `agent_a2a_task_total{final_state="failed"} / total > 10%` | P2 | Remote agent reliability issue |
| `TokenBudgetExhaustionRate` | `rate(agent_token_budget_exhausted_total[10m]) > 1%` | P2 | Agent sessions consistently running out of tokens |

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-METRIC-001` | All five metric families MUST be instrumented before production deployment — partial metric coverage creates blind spots in the alerting surface. |
| `OBS-METRIC-002` | Tool latency histograms MUST use consistent bucket boundaries across all tool names (e.g., 50ms, 100ms, 250ms, 500ms, 1s, 2s, 5s, 10s) to enable cross-tool comparison. |
| `OBS-METRIC-003` | Token metrics (prompt + completion) MUST be attributed to `agent_id` and `session_id` to enable per-agent cost accounting — unatttributed token metrics are operationally useless. |
| `OBS-METRIC-004` | Behavioral metrics (`agent_loop_detection_total`, `agent_goal_abandonment_total`, `agent_token_budget_exhausted_total`) MUST be treated as first-class alerting signals with defined thresholds — not as informational-only metrics. |
| `OBS-METRIC-005` | All metrics MUST be exported to the OTel collector using the OTel Metrics SDK — not via a parallel Prometheus client that bypasses the OTel pipeline. |

### Java + LangGraph Target

```java
@Component
public class AgentMetrics {
    private final LongHistogram toolDuration;
    private final LongCounter toolCallTotal;
    private final ObservableLongGauge toolInFlight;
    private final LongCounter loopDetectionTotal;
    private final LongCounter tokenTotal;
    private final AtomicLong inFlightCount = new AtomicLong(0);

    public AgentMetrics(OpenTelemetry otel) {
        Meter meter = otel.getMeter("agent.metrics");

        // OBS-METRIC-002: consistent histogram buckets
        toolDuration = meter.histogramBuilder("agent_tool_call_duration_ms")
            .ofLongs()
            .setExplicitBucketBoundariesAdvice(
                List.of(50L, 100L, 250L, 500L, 1000L, 2000L, 5000L, 10000L))
            .build();

        toolCallTotal = meter.counterBuilder("agent_tool_call_total").build();
        loopDetectionTotal = meter.counterBuilder("agent_loop_detection_total").build();

        // OBS-METRIC-003: token attribution requires agent_id label
        tokenTotal = meter.counterBuilder("agent_llm_tokens_total").build();

        toolInFlight = meter.gaugeBuilder("agent_tool_call_inflight")
            .ofLongs()
            .buildWithCallback(obs -> obs.record(inFlightCount.get()));
    }

    public void recordToolCall(String toolName, String status, String errorCode,
                                long durationMs, String agentId) {
        Attributes attrs = Attributes.of(
            AttributeKey.stringKey("tool_name"), toolName,
            AttributeKey.stringKey("status"), status,
            AttributeKey.stringKey("error_code"), errorCode != null ? errorCode : "none"
        );
        toolDuration.record(durationMs, attrs);
        toolCallTotal.add(1, attrs);
    }

    public void recordLoopDetection(String agentId, String loopNodeName) {
        // OBS-METRIC-004: behavioral metric as first-class signal
        loopDetectionTotal.add(1, Attributes.of(
            AttributeKey.stringKey("agent_id"), agentId,
            AttributeKey.stringKey("loop_node"), loopNodeName
        ));
    }

    public void recordTokens(String agentId, String modelId,
                              long promptTokens, long completionTokens) {
        // OBS-METRIC-003: attributed to agentId
        tokenTotal.add(promptTokens, Attributes.of(
            AttributeKey.stringKey("agent_id"), agentId,
            AttributeKey.stringKey("model_id"), modelId,
            AttributeKey.stringKey("token_type"), "prompt"
        ));
        tokenTotal.add(completionTokens, Attributes.of(
            AttributeKey.stringKey("agent_id"), agentId,
            AttributeKey.stringKey("model_id"), modelId,
            AttributeKey.stringKey("token_type"), "completion"
        ));
    }
}
```

---

## Section 23.5 — Behavioral Observability

### What Behavioral Observability Means

Behavioral observability is the discipline of detecting **semantic failures** — cases where the agent system is technically functional but operationally wrong. It has no direct analog in conventional service monitoring and requires agent-specific instrumentation.

The three primary behavioral signals are:

#### Loop Detection

An agent that calls the same node or the same tool more than a threshold number of times within a single session is very likely stuck in a reasoning loop. The loop may be caused by:
- A tool returning a result the agent does not know how to interpret, causing it to retry
- A prompt that fails to constrain the agent's exit condition
- A prompt injection attack that creates a goal the agent cannot achieve

**Detection rule:** If the same `(node_name, input_hash)` pair appears more than `loopThreshold` times in a session's span tree, emit `agent.loop.detected` and increment `agent_loop_detection_total`.

**Response rule:** The LangGraph graph MUST include a loop-detection conditional edge that interrupts the execution and routes to a `LoopBreakNode` when the threshold is exceeded. This is a structural graph design requirement, not just a monitoring concern.

#### Token Budget Management

An LLM has a finite context window. An agent that operates across many tool calls and multi-turn exchanges will eventually exhaust its token budget if it does not actively manage the context. When `finish_reason = "length"` is returned by the model API, the agent has been truncated mid-reasoning — any subsequent outputs are unreliable.

**Detection rule:** When `llm.inference.completed` is logged with `finishReason = "length"`, emit `agent_token_budget_exhausted_total` and flag the session for review.

**Preventive rule:** The agent's context management strategy (rolling window, summarization, selective retention) must be implemented in the `ContextManagerNode` before calling the LLM — truncation is a design failure, not an operational edge case. (See chap03 specs for context management rules.)

#### Tool Selection Accuracy

When the agent's underlying model is updated, prompt is modified, or a new tool is added to the registry, the agent may start selecting different tools than intended. This is not an error — all tool calls may succeed — but the behavioral output changes.

**Detection rule:** Maintain a `toolSelectionBaseline` for each agent: the expected distribution of tool calls for a given class of input. When the observed distribution deviates by more than a threshold over a rolling window, emit an alert. This requires:
1. A session replay test suite that exercises each agent with canonical inputs
2. A baseline tool selection profile per agent version
3. A comparison job that runs on each deployment

### The Observability Node Pattern

The cleanest way to implement behavioral observability in LangGraph is the **Observability Node Pattern**: a dedicated node in every graph that inspects the current state and emits behavioral signals, inserted at key decision points.

```java
public class BehavioralObserverNode implements RunnableNode<AgentState> {
    private final AgentMetrics metrics;
    private final AgentStructuredLogger logger;
    private final int loopThreshold;

    @Override
    public AgentState run(AgentState state) {
        // Loop detection (OBS-BEHAV-001)
        String currentNode = state.currentNodeName();
        int loopCount = state.nodeInvocationCount(currentNode);
        if (loopCount >= loopThreshold) {
            logger.logLoopDetected(state, currentNode, loopCount, loopThreshold);
            metrics.recordLoopDetection(state.agentId(), currentNode);
            return state.withLoopBreakFlag(true);   // triggers conditional edge → LoopBreakNode
        }

        // Token budget check (OBS-BEHAV-002)
        if (state.lastFinishReason() != null &&
            state.lastFinishReason().equals("length")) {
            metrics.recordTokenBudgetExhausted(state.agentId());
            logger.logTokenBudgetExhausted(state);
        }

        return state;
    }
}
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-BEHAV-001` | Every LangGraph graph MUST include a loop-detection conditional edge that routes to a `LoopBreakNode` when `nodeInvocationCount` exceeds `loopThreshold` — this is a structural graph requirement, not just a monitoring concern. |
| `OBS-BEHAV-002` | Every LLM inference result with `finishReason = "length"` MUST increment `agent_token_budget_exhausted_total` and log `agent.token.budget.exhausted` — truncated reasoning produces unreliable outputs and must be flagged. |
| `OBS-BEHAV-003` | Tool call volume per session MUST be monitored against a per-agent baseline — anomalous high volume (> 3× baseline) is a diagnostic signal for prompt injection, reasoning loops, or runaway retry chains. |
| `OBS-BEHAV-004` | A session replay test suite with canonical inputs MUST be maintained for each agent to detect tool selection drift after model updates or prompt changes. |
| `OBS-BEHAV-005` | Behavioral observability signals (`agent.loop.detected`, `agent.goal.abandoned`, `agent.token.budget.exhausted`) MUST be visible in the same observability dashboard as infrastructure metrics — behavioral signals in a separate system are not actionable during incident response. |

---

## Section 23.6 — OTel Collector Configuration

### The Collector as the Observability Hub

The OTel Collector is the central aggregation and routing point for all telemetry in the agent system. Every agent runtime, MCP client, MCP server, A2A service, and Agent Gateway exports to the same collector instance (or collector cluster in high-volume deployments).

The collector performs:
- **Reception** — accepts OTLP (gRPC or HTTP) from all instrumented services
- **Processing** — tail-based sampling decisions, attribute enrichment, PII scrubbing
- **Export** — routes to backend(s): Jaeger/Tempo (traces), Prometheus/Mimir (metrics), Loki/Elasticsearch (logs)

### Minimal Collector Pipeline for Agent Systems

```yaml
# otel-collector-config.yaml (minimum viable agent observability)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Tail-based sampling: always capture errors and behavioral anomalies (OBS-TRACE-005)
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: always-sample-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: always-sample-loops
        type: string_attribute
        string_attribute: { key: agent.loop.detected, values: ["true"] }
      - name: probabilistic-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

  # PII scrubbing: never log raw tool arguments (OBS-LOG-004)
  attributes/scrub_pii:
    actions:
      - key: tool.input_args
        action: delete
      - key: llm.prompt_content
        action: delete

  batch:
    timeout: 5s

exporters:
  otlp/jaeger:
    endpoint: jaeger-collector:4317
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, attributes/scrub_pii, batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [attributes/scrub_pii, batch]
      exporters: [loki]
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `OBS-OTEL-001` | All agent services MUST export via OTLP to the shared OTel Collector — direct backend exports (e.g., writing directly to Prometheus or Jaeger from the agent) bypass the tail sampling and PII scrubbing pipeline. |
| `OBS-OTEL-002` | The PII scrubbing processor MUST be applied to all pipelines before export — raw tool arguments and LLM prompt content MUST be deleted before leaving the collector. |
| `OBS-OTEL-003` | The collector MUST be deployed in HA mode (minimum 2 instances behind a load balancer) for any production system — a single collector instance is an observability single point of failure. |
| `OBS-OTEL-004` | The collector configuration MUST be version-controlled alongside the agent system code — undocumented collector configurations are an operational risk at every infrastructure change. |

---

## Section 23.7 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `OBS-STRUCT-001` | Structure | OTel as sole telemetry SDK across all services |
| `OBS-STRUCT-002` | Structure | Single shared OTel collector endpoint |
| `OBS-STRUCT-003` | Structure | Instrumentation before first production deployment |
| `OBS-STRUCT-004` | Structure | Behavioral signals alongside infrastructure metrics |
| `OBS-TRACE-001` | Tracing | W3C `traceparent` propagated across every MCP, A2A, and gateway boundary |
| `OBS-TRACE-002` | Tracing | Every LangGraph node produces a span with defined attributes |
| `OBS-TRACE-003` | Tracing | Every MCP `tools/call` span includes full attribute set |
| `OBS-TRACE-004` | Tracing | Every A2A task span includes full attribute set |
| `OBS-TRACE-005` | Tracing | Tail-based sampling captures 100% of error, loop, and anomaly traces |
| `OBS-TRACE-006` | Tracing | Root span carries `session.id`, `agent.id`, `agent.version`, `user.id` |
| `OBS-LOG-001` | Logging | All events are structured JSON with mandatory correlation fields |
| `OBS-LOG-002` | Logging | `traceId` sourced from active OTel span context |
| `OBS-LOG-003` | Logging | Mandatory log events emitted at every defined lifecycle trigger |
| `OBS-LOG-004` | Logging | Tool args and LLM prompt content not logged at INFO level |
| `OBS-LOG-005` | Logging | Logs shipped to same backend as traces and metrics |
| `OBS-METRIC-001` | Metrics | All five metric families instrumented before production |
| `OBS-METRIC-002` | Metrics | Consistent histogram bucket boundaries across all tool names |
| `OBS-METRIC-003` | Metrics | Token metrics attributed to `agent_id` and `session_id` |
| `OBS-METRIC-004` | Metrics | Behavioral metrics treated as first-class alerting signals |
| `OBS-METRIC-005` | Metrics | All metrics exported via OTel Metrics SDK |
| `OBS-BEHAV-001` | Behavioral | Loop-detection conditional edge in every LangGraph graph |
| `OBS-BEHAV-002` | Behavioral | `finish_reason=length` flagged and counted |
| `OBS-BEHAV-003` | Behavioral | Tool call volume per session monitored against baseline |
| `OBS-BEHAV-004` | Behavioral | Session replay test suite for tool selection drift detection |
| `OBS-BEHAV-005` | Behavioral | Behavioral signals in same dashboard as infrastructure metrics |
| `OBS-OTEL-001` | Collector | All services export via OTLP to shared collector |
| `OBS-OTEL-002` | Collector | PII scrubbing applied before export |
| `OBS-OTEL-003` | Collector | Collector in HA mode (min 2 instances) |
| `OBS-OTEL-004` | Collector | Collector config version-controlled with agent code |

---

## Section 23.8 — Transition: From Observability Foundation to Framework-Specific Depth

Chapter 23 establishes the observability architecture that applies uniformly across all frameworks covered in Chapters 24–29. Each subsequent framework chapter addresses **framework-specific observability gaps** — the places where a given framework's built-in telemetry falls short of the `OBS-*` rules above, and the specific instrumentation code required to close those gaps.

| Chapter | Framework | Key Observability Gap vs. OBS Rules |
|---|---|---|
| Chapter 24 | LangGraph | Node-level span emission requires custom `ObservableNode` wrapper; loop detection is not built-in |
| Chapter 25 | OpenAI Agents SDK | Runner-based execution model has no explicit node spans; requires custom run hooks |
| Chapter 26 | Google ADK | Runner event stream provides partial observability; A2A boundary propagation requires explicit injection |
| Chapter 27 | Mastra | In-process embedding means trace context must be propagated through application layer, not transport |
| Chapter 28 | AutoGen | Multi-agent group chat produces deeply nested delegation chains; chain length alerting is critical |
| Chapter 29 | FastMCP | Server-side is a leaf span; `traceparent` extraction from HTTP headers requires middleware |

The `OBS-*` rules in this chapter are the acceptance criteria for the observability sections of each framework chapter. A framework chapter's observability section is complete when all applicable `OBS-*` rules are satisfied.
