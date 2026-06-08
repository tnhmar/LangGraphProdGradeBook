# Chapter 34 Spec — Advanced Tool Patterns

> **Volume 1 reference:** Part 7 — Tooling and Integrations, Chapter 34
> **Depends on:** chap32 (`TOOL-ABS-*`, `TOOL-CONTRACT-*`, `TOOL-EXEC-*`, `TOOL-ERR-*`, `TOOL-SEC-*`, `TOOL-TEST-*`), chap33 (`ENT-*`)
> **Java + LangGraph implementation target:** Volume 2 — Chapter 3: Advanced Tool Patterns; Chapter 4: Tool Middleware; Chapter 12: Distributed Tracing and OTel
> **Bridges to:** chap35 (RAG Pipelines and Multi-Agent Retrieval)

> *"Composing unreliable parts into a reliable whole is the central discipline of distributed systems engineering. Tools are no different."*
> — Attributed to distributed systems practitioners

---

## Chapter Purpose

Chapters 32 and 33 established the foundations: a uniform tool contract, a reliable dispatch pipeline, and integration patterns for enterprise backends. This chapter assumes those foundations are in place and addresses what comes next.

Production agents rarely call a single tool at a time. They fan out across multiple tools in parallel, chain tool outputs into subsequent calls, select dynamically from registries containing hundreds of tools, and trigger operations that complete minutes or hours after the call returns. Each of these patterns introduces failure modes, safety requirements, and engineering decisions that the basic tool abstraction layer does not address on its own.

By the end of this chapter the reader can:
- Implement parallel fan-out with structured partial-failure semantics and global deadline enforcement
- Design dynamic tool selection pipelines using semantic retrieval over large tool registries
- Build safe tool chains with explicit handoff validation, taint tracking, and depth limits
- Handle long-running asynchronous operations via submit-poll, webhook callbacks, and structured suspension
- Compose a tool middleware pipeline for logging, cost tracking, caching, cache invalidation, and rate limiting
- Manage tool versioning and schema evolution without breaking deployed agents

---

## Section 34.1 — Parallel Tool Execution: Fan-Out and Fan-In

### Why Parallelism Matters

Sequential tool execution is the natural default — the agent calls a tool, waits for the result, then decides what to do next. For single, dependent operations this is correct. But many agent tasks decompose into independent sub-queries with no data dependency between them: a research agent gathering from three knowledge bases, a financial agent retrieving four account balances simultaneously, a DevOps agent querying multiple monitoring systems at once.

Running three tools with 300 ms latency each takes 900 ms sequentially. In parallel it takes 300 ms plus coordination overhead. At the scale of a multi-step agent loop with many such decision points, this difference determines whether the agent feels responsive or sluggish.

### Fan-Out / Fan-In Implementation

Fan-out dispatches multiple tool calls simultaneously. Fan-in collects and aggregates their results before returning to the agent. The abstraction layer implements both as a single composite operation.

```java
@Component
public class FanOutDispatcher {

    private final ToolLayer toolLayer;
    private final AgentMetrics metrics;

    public record FanOutCall(String name, Map<String, Object> args) {}
    public record FanOutResult(String name, Object result, ToolError error) {
        public boolean succeeded() { return error == null; }
    }

    /**
     * Fan-out dispatcher with partial-failure handling.
     * requireAll=true  → fail atomically if any call fails (deeply interdependent results)
     * requireAll=false → return mixed success/error results (independent enrichment queries)
     */
    public List<FanOutResult> fanOut(List<FanOutCall> calls,
                                      boolean requireAll,
                                      Duration globalDeadline) {

        // ENT-FAN-002: global fan-out deadline = min(1.2 * slowest tool timeout, remaining budget)
        Instant deadline = Instant.now().plus(globalDeadline);

        List<CompletableFuture<FanOutResult>> futures = calls.stream()
            .map(call -> CompletableFuture
                .supplyAsync(() -> {
                    try {
                        Object result = toolLayer.invoke(call.name(), call.args());
                        return new FanOutResult(call.name(), result, null);
                    } catch (ToolExecutionException e) {
                        return new FanOutResult(call.name(), null, e.toError());
                    }
                })
                .orTimeout(
                    Duration.between(Instant.now(), deadline).toMillis(),
                    TimeUnit.MILLISECONDS
                )
                .exceptionally(ex ->
                    new FanOutResult(call.name(), null,
                        ToolError.of("TOOL_TIMEOUT", ex.getMessage(), false))
                ))
            .toList();

        List<FanOutResult> results = futures.stream()
            .map(CompletableFuture::join)
            .toList();

        if (requireAll) {
            results.stream()
                .filter(r -> !r.succeeded())
                .findFirst()
                .ifPresent(r -> { throw new FanOutPartialFailureException(r.error()); });
        }

        return results;
    }
}
```

### Partial Failure Semantics

Parallel execution introduces a failure mode sequential calls do not have: some calls succeed and others fail within the same fan-out batch. The agent must receive the complete picture — not a binary success or failure.

| `requireAll` | Behaviour | Use When |
|---|---|---|
| `true` | Fail atomically if any call fails | Results are deeply interdependent; partial result set is meaningless |
| `false` | Return all outcomes (success + error mixed) | Independent enrichment queries; partial results are still useful |

> ⚠️ **Never pass a partial fan-out result to the LLM without an explicit annotation for each missing result.** An agent that receives three out of four tool results and is not told the fourth failed will often hallucinate the missing data rather than flag the gap. Annotate clearly: `crm.search: UNAVAILABLE — circuit open`.

### Global Fan-Out Deadline

Each tool in a fan-out has its own `timeoutMs` from its contract. The fan-out dispatcher MUST NOT blindly wait for the slowest tool. Instead it enforces a global deadline:

\[T_{\text{fanout}} = \min\!\left(B,\ 1.2 \cdot \max_i T_i\right)\]

where \(B\) is the remaining session deadline budget and \(T_i\) is the declared timeout of tool \(i\). The 1.2× multiplier provides a modest buffer beyond the slowest tool's declared timeout; after this deadline, any still-running tasks are cancelled and their slots filled with `TOOL_TIMEOUT` errors.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-FAN-001` | Fan-out results MUST include a per-tool status annotation for every call in the batch — the agent reasoning loop MUST NOT infer missing data from absent fields. |
| `ADV-FAN-002` | Fan-out dispatchers MUST enforce a global deadline computed as `min(session_budget, 1.2 × max_tool_timeout)` — individual tool timeouts are not sufficient to bound total fan-out latency. |
| `ADV-FAN-003` | The `requireAll` policy MUST be declared explicitly at the fan-out call site — implicit partial-failure behaviour is a correctness defect. |
| `ADV-FAN-004` | Each tool in a fan-out batch MUST run in its own isolated execution context — a failure in one tool MUST NOT cancel or corrupt in-flight calls to other tools in the same batch. |

---

## Section 34.2 — Dynamic Tool Selection at Scale

### The Scale Problem

Chapter 32 noted that the agent runtime populates the tool section of the system prompt from the Tool Registry's `list` operation. This works correctly when the registry contains fewer than roughly twenty tools. Beyond that, two problems emerge:

- **Context window cost:** A registry of 200 tools averaging 150 tokens per schema consumes 30 000 tokens per call — before the task, conversation history, or retrieved context contribute a single token.
- **Selection degradation:** LLM tool selection accuracy decreases as the number of tools in the prompt grows. Benchmark studies on function-calling tasks consistently show selection accuracy dropping materially above 30–50 tools in context.

### Semantic Tool Retrieval

The solution is to treat tool selection as a **retrieval problem** rather than an enumeration problem. Each tool's name and description are embedded at registration time and stored in an in-memory vector index. At inference time, the agent's current task is embedded and used as a query; the top-*k* most similar tools are injected into the system prompt.

```java
@Component
public class SemanticToolSelector {

    private final EmbeddingModel embeddingModel;   // e.g., all-MiniLM-L6-v2 (22M params, <5ms CPU)
    private final VectorIndex toolIndex;
    private final ToolRegistry registry;

    // Called at tool registration time — ENT-DYN-001
    public void indexTool(ToolContract contract) {
        String text = contract.name() + ": " + contract.description();
        float[] embedding = embeddingModel.embed(text);
        toolIndex.upsert(contract.name(), embedding, Map.of("contract", contract));
    }

    // Called at inference time — returns top-k tool names
    public List<String> select(String taskQuery, int topK) {
        float[] queryVec = embeddingModel.embed(taskQuery);
        return toolIndex.query(queryVec, topK).stream()
            .map(hit -> (String) hit.metadata().get("name"))
            .toList();
    }

    // Tool injection with pinned + retrieved tools (ENT-DYN-002)
    public List<ToolContract> buildToolSet(String taskQuery, int topK,
                                            Set<String> pinnedTools) {
        List<String> retrieved = select(taskQuery, topK);
        // Union: pinned always included, no duplicates
        LinkedHashSet<String> selected = new LinkedHashSet<>(pinnedTools);
        selected.addAll(retrieved);
        return selected.stream()
            .map(registry::lookup)
            .filter(Objects::nonNull)
            .toList();
    }
}
```

> 💡 **Keep the embedding model used for tool retrieval small and fast.** The `all-MiniLM-L6-v2` model (22M parameters) runs in under 5 ms on CPU for a registry of a few hundred tools. Do not use the same large LLM used for agent reasoning to perform tool retrieval — the latency cost is prohibitive and the accuracy improvement is marginal for this specific task.

### Mandatory Tool Pinning

Semantic retrieval is probabilistic. A tool the agent needs may score below the top-*k* threshold for an unusual query. Safety tools, error-reporting tools, and human escalation tools MUST always be included in the injected set regardless of retrieval score — this is **mandatory tool pinning**.

### Two-Stage Selection for Very Large Registries

For registries of 500 or more tools, a single embedding query may not be precise enough. Two-stage selection adds a lightweight classification step before semantic retrieval:

1. **Category routing** — a fast classifier (small embedding model or keyword matching) assigns the current task to one or more tool categories (e.g., `crm`, `analytics`, `messaging`)
2. **Semantic retrieval within category** — semantic search runs only over tools in the matched categories, dramatically reducing the candidate pool and improving precision

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-DYN-001` | Tool descriptions and names MUST be embedded and indexed at registration time — lazy indexing on first query is not acceptable for production registries. |
| `ADV-DYN-002` | Safety-critical tools (human escalation, error reporting, session abort) MUST be pinned — they MUST always be included in the injected tool set regardless of semantic retrieval score. |
| `ADV-DYN-003` | The embedding model used for tool retrieval MUST be a compact, low-latency model (≤30M parameters) — using the agent's reasoning LLM for tool retrieval is prohibited. |
| `ADV-DYN-004` | Two-stage selection (category routing → semantic retrieval within category) MUST be used for registries exceeding 500 tools. |
| `ADV-DYN-005` | Tool description changes (even without schema changes) MUST trigger re-embedding and re-indexing — description drift causes silent selection regressions. |

---

## Section 34.3 — Tool Chaining and Data Flow Safety

### What Tool Chaining Is

Tool chaining occurs when the output of one tool call is used — directly or transformed — as the input to a subsequent tool call. This is the most natural multi-step pattern: look up a customer ID, use that ID to query order history, use the order history to generate a summary.

Chaining is powerful and it is also where the most subtle failures in agent systems occur. Data from an untrusted source — a retrieved document, a user-supplied value, a third-party API response — flows through the chain and can contaminate inputs to high-privilege tools if the data flow is not controlled.

### Schema Validation at Handoff Points

The simplest safety measure for tool chains is strict schema validation at every handoff. Before the output of Tool A is used as input to Tool B, the abstraction layer validates that the relevant fields conform to Tool B's input schema. The construction step is a natural validation point:

```java
@Component
public class ToolChainExecutor {

    private final ToolLayer toolLayer;

    public Map<String, Object> executeChain(String sessionId) throws ChainException {
        ToolChainContext ctx = new ToolChainContext(sessionId);

        // Step 1 — look up contact
        ctx.enter("crm.getContact");
        Map<String, Object> contactResult = toolLayer.invoke("crm.getContact",
            Map.of("email", "user@example.com"));
        ctx.exit();

        // ENT-CHAIN-001: explicit field extraction with type guard — never raw dict passthrough
        String accountId = (String) contactResult.get("accountId");
        if (accountId == null || accountId.isBlank()) {
            throw new ChainException("crm.getContact returned no accountId");
        }

        // Step 2 — use extracted field only, not raw output
        ctx.enter("erp.getOrders");
        Map<String, Object> ordersResult = toolLayer.invoke("erp.getOrders",
            Map.of("accountId", accountId, "limit", 20));
        ctx.exit();

        return ordersResult;
    }
}
```

> ⚠️ **Never pass the raw output dictionary of one tool call directly as the `args` of the next.** Tool output schemas evolve — a field rename in one tool will silently break a downstream tool that consumed the raw dictionary. Explicit field extraction with type checks makes handoffs robust and the chain auditable.

### Taint Tracking for Security-Critical Chains

In security-sensitive chains, it matters not just whether data has the right shape but **where it came from**. Data retrieved from an external source — a web search result, a user document, a third-party API — is *tainted*: it may contain adversarial content designed to manipulate downstream tool calls via prompt injection.

```java
public enum TaintLevel {
    CLEAN,      // Trusted internal sources
    EXTERNAL,   // External APIs, user input
    HOSTILE     // Flagged as likely adversarial
}

@Component
public class TaintPolicy {

    private final ToolRegistry registry;

    public void check(String toolName, Map<String, Object> args,
                       Map<String, TaintLevel> taintMap) {
        ToolContract contract = registry.lookup(toolName);
        boolean isWrite = Set.of("write", "destructive")
            .contains(contract.sideEffects());

        if (isWrite) {
            taintMap.forEach((argName, taint) -> {
                if (taint == TaintLevel.HOSTILE) {
                    throw new TaintPolicyViolation(
                        "Hostile-tainted value '" + argName + "' cannot be passed " +
                        "to write tool '" + toolName + "' without human review.");
                }
            });
        }
    }
}
```

> 📝 Taint tracking is a defense-in-depth mechanism, not a complete solution to prompt injection. The broader prompt injection defence strategy — content filtering, output sanitization, least-privilege tool access — is covered in Volume 2, Chapter 18.

### Chain Depth Limits

Unbounded tool chains create a denial-of-service vector: a chain that calls Tool A, whose output leads to Tool B, whose output leads back to Tool A. Cycle detection and chain depth limits are mandatory in any agent that executes multi-step tool chains autonomously.

```java
public class ToolChainContext {
    private static final int MAX_DEPTH = 10;
    private int depth = 0;
    private final List<String> callHistory = new ArrayList<>();
    private final String sessionId;

    public ToolChainContext(String sessionId) { this.sessionId = sessionId; }

    public void enter(String toolName) {
        if (depth >= MAX_DEPTH) {
            throw new ChainDepthExceeded(
                "Tool chain exceeded " + MAX_DEPTH + " calls. History: " + callHistory);
        }
        callHistory.add(toolName);
        depth++;
    }

    public void exit() { depth--; }
}
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-CHAIN-001` | Tool chain handoffs MUST use explicit field extraction with type guards — raw output dictionary passthrough between chained tools is prohibited. |
| `ADV-CHAIN-002` | Data originating from external sources (web, user input, third-party APIs) MUST be taint-labelled at the point of retrieval and the label MUST be propagated through the chain. |
| `ADV-CHAIN-003` | Hostile-tainted data MUST NOT be passed as arguments to write or destructive tools without explicit human review (chap12 HITL gate). |
| `ADV-CHAIN-004` | All tool chains MUST enforce a maximum depth limit (default: 10 calls) — unbounded recursion is a denial-of-service vector. |
| `ADV-CHAIN-005` | Chain depth and call history MUST be stored in the agent's session state (chap8) — this data is required for debugging, audit logging, and structured suspension. |

---

## Section 34.4 — Long-Running and Asynchronous Tools

### The Long-Running Tool Problem

The standard tool contract assumes a synchronous call-return model: the tool is invoked, the result arrives within the tool's declared `timeoutMs`, and the agent proceeds. This model breaks down for operations that take minutes or hours: document processing pipelines, code compilation jobs, ML batch inference runs, regulatory approval workflows.

A tool that takes five minutes cannot declare `timeoutMs: 300000` — holding a connection open for five minutes is operationally irresponsible, and the agent's entire reasoning loop would suspend waiting for it. Long-running operations require an explicit asynchronous handoff.

### The Submit-Poll Pattern

The most widely applicable pattern is **submit-poll**: the tool call initiates the operation and returns a job token immediately. The agent stores the token and polls for completion using a separate `checkStatus` tool.

```java
// Tool 1: submit — sideEffects: write, returns jobId immediately
@Component
public class PipelineSubmitHandler implements ToolHandler {

    private final EtlPipelineClient pipelineClient;

    @Override
    public Map<String, Object> handle(Map<String, Object> args, CallingContext ctx) {
        String jobId = pipelineClient.submit(
            (String) args.get("source"),
            (String) args.get("target")
        );
        // Returns immediately — job runs asynchronously
        return Map.of("jobId", jobId, "status", "submitted");
    }
}

// Tool 2: getStatus — sideEffects: read-only, polls by jobId
@Component
public class PipelineStatusHandler implements ToolHandler {

    private final EtlPipelineClient pipelineClient;

    @Override
    public Map<String, Object> handle(Map<String, Object> args, CallingContext ctx) {
        String jobId = (String) args.get("jobId");
        JobStatus status = pipelineClient.getStatus(jobId);
        // status.state: pending | running | done | failed
        return Map.of(
            "jobId", jobId,
            "status", status.state(),
            "outputLocation", status.outputLocation(),   // null unless done
            "error", status.errorMessage()               // null unless failed
        );
    }
}
```

The submit and poll tools are two separate entries in the Tool Registry, each with its own schema and contract. The agent learns to use them as a pair from their descriptions. The job token is stored in the agent's working memory (chap8) between calls.

### Webhook Callback Pattern

When the execution environment supports it, webhook callbacks are more efficient than polling — the backend notifies the agent runtime when the operation completes rather than being asked repeatedly.

| Step | Action |
|---|---|
| 1 | Agent runtime exposes a webhook receiver endpoint: `POST /agent/callbacks/{sessionId}` |
| 2 | Submit tool includes the callback URL in its request payload |
| 3 | Backend POSTs the result to that URL when complete |
| 4 | Webhook receiver writes result to the session's state store, keyed by job token |
| 5 | On the agent's next reasoning cycle, a `checkCallbacks` tool reads pending results from state |

> 💡 **Webhook callbacks require the agent runtime to be network-reachable from the backend system** — a constraint that does not hold in all deployment environments (e.g., agents in isolated VPCs). Design submit-poll as the fallback that always works, and add webhook support as an optimization where the network topology permits.

### Structured Suspension and Resumption

The deepest form of long-running tool integration is **structured suspension**: the agent serializes its full state (goal stack, working memory, in-progress tool calls) to persistent storage and terminates. An external event — a timer, a webhook, a human action — resumes the agent from exactly the point of suspension.

This pattern requires the agent runtime to support state checkpointing (chap8), and at the tool layer requires that every in-flight job token be part of the checkpointed state.

> ⚠️ **Structured suspension is complex to implement correctly.** Partial implementations — where some state is checkpointed but not all, or where in-flight tool calls are not correctly represented in the resumed state — produce agents that silently forget work in progress. If your runtime does not support full state serialization, prefer submit-poll with explicit polling intervals over attempting incomplete suspension.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-ASYNC-001` | Operations that may take longer than 30 seconds MUST use the submit-poll pattern — synchronous tool calls with `timeoutMs > 30000` are not acceptable for production long-running operations. |
| `ADV-ASYNC-002` | Submit tools MUST return a durable job token that survives agent restart — ephemeral in-memory job IDs are not acceptable for operations spanning minutes or hours. |
| `ADV-ASYNC-003` | In-flight job tokens MUST be stored in the agent's checkpointed state (chap8) — loss of the job token on agent restart is equivalent to losing the submitted work. |
| `ADV-ASYNC-004` | The poll tool MUST be declared `sideEffects: read-only` — polling for status MUST NOT trigger side effects on the backend. |
| `ADV-ASYNC-005` | Webhook callback receivers MUST validate the callback payload's authenticity (HMAC signature or bearer token) before writing to session state — unauthenticated callbacks are a spoofing vector. |

---

## Section 34.5 — Tool Middleware: Cross-Cutting Concerns

### The Middleware Model

Individual tool handlers should contain exactly one concern: executing the tool's declared operation against its backend. Cross-cutting concerns — logging, metrics, caching, rate limiting, result transformation — should not be duplicated inside each handler.

The middleware pipeline applies these concerns uniformly by wrapping each tool invocation in a chain of composable interceptors. The pattern is identical to middleware stacks in HTTP server frameworks, applied to the tool dispatch path.

```java
// Middleware interface
@FunctionalInterface
public interface ToolMiddleware {
    Map<String, Object> apply(String toolName, Map<String, Object> args,
                               ToolHandler next) throws ToolExecutionException;
}

// Pipeline composer — middlewares applied outer-to-inner (last in list = innermost)
@Component
public class MiddlewarePipeline {

    public ToolHandler compose(ToolHandler handler, List<ToolMiddleware> middlewares) {
        ToolHandler current = handler;
        for (int i = middlewares.size() - 1; i >= 0; i--) {
            final ToolHandler next = current;
            final ToolMiddleware mw = middlewares.get(i);
            current = (name, args) -> mw.apply(name, args, next);
        }
        return current;
    }
}
```

### Logging Middleware

Every tool call MUST emit structured log events at entry and exit through OpenTelemetry so that trace IDs and span contexts are propagated correctly to the distributed tracing backend (Volume 2, Chapter 12).

```java
@Component
public class LoggingMiddleware implements ToolMiddleware {

    private final Tracer tracer;
    private final Logger log = LoggerFactory.getLogger(LoggingMiddleware.class);

    @Override
    public Map<String, Object> apply(String toolName, Map<String, Object> args,
                                      ToolHandler next) throws ToolExecutionException {
        Span span = tracer.spanBuilder("tool.call." + toolName).startSpan();
        long start = System.currentTimeMillis();

        try (Scope ignored = span.makeCurrent()) {
            log.info("tool.call.start tool={} argKeys={}", toolName, args.keySet());
            Map<String, Object> result = next.handle(args, null);
            long ms = System.currentTimeMillis() - start;
            log.info("tool.call.success tool={} durationMs={}", toolName, ms);
            span.setStatus(StatusCode.OK);
            return result;

        } catch (ToolExecutionException e) {
            long ms = System.currentTimeMillis() - start;
            log.error("tool.call.error tool={} durationMs={} error={}", toolName, ms, e.getMessage());
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Cost Tracking Middleware

Every LLM-adjacent system has a cost structure. Cost tracking middleware records the estimated cost of each tool call against the session's budget and can enforce a per-session cost ceiling.

```java
@Component
public class CostTrackingMiddleware implements ToolMiddleware {

    // Estimated cost per tool call in USD — configured externally
    private final Map<String, Double> toolCostUsd;
    private final SessionBudgetStore budgetStore;

    @Override
    public Map<String, Object> apply(String toolName, Map<String, Object> args,
                                      ToolHandler next) throws ToolExecutionException {
        double estimated = toolCostUsd.getOrDefault(toolName, 0.0);
        SessionBudget budget = budgetStore.get(CallingContext.current().sessionId());

        if (!budget.canAfford(estimated)) {
            throw new BudgetExceededException(
                "Session budget exhausted. Tool " + toolName +
                " costs $" + String.format("%.4f", estimated));
        }

        Map<String, Object> result = next.handle(args, null);
        budget.deduct(estimated);
        return result;
    }
}
```

### Caching Middleware

Read-only tools with stable outputs are candidates for response caching. The cache key is a hash of the tool name and the normalized argument dictionary.

```java
@Component
public class CachingMiddleware implements ToolMiddleware {

    private final Cache cache;
    private final ToolRegistry registry;

    @Override
    public Map<String, Object> apply(String toolName, Map<String, Object> args,
                                      ToolHandler next) throws ToolExecutionException {
        ToolContract contract = registry.lookup(toolName);

        // ENT-CACHE-001: only cache read-only tools
        if (!"read-only".equals(contract.sideEffects())) {
            return next.handle(args, null);
        }

        String cacheKey = sha256(Map.of("tool", toolName, "args", args));
        Optional<Map<String, Object>> cached = cache.get(cacheKey);
        if (cached.isPresent()) return cached.get();

        Map<String, Object> result = next.handle(args, null);

        // Tag cache entry with entity identifiers for event-driven invalidation
        Set<String> tags = extractEntityTags(toolName, args, result);
        cache.set(cacheKey, result, contract.cacheTtlSeconds(), tags);
        return result;
    }
}
```

> 💡 **Cache tool results conservatively.** A 5-minute TTL on knowledge base search results is reasonable. A 5-minute TTL on a `getAccountBalance` tool is not. Set TTLs based on how quickly the underlying data changes — a stale cache hit that causes the agent to reason from outdated data is worse than a cache miss.

### Cache Invalidation

TTL-based expiry is sufficient for slowly-changing data but fails silently when underlying data changes before the TTL expires. Three invalidation patterns:

| Pattern | Mechanism | Best For |
|---|---|---|
| **Event-driven tag eviction** | Change event on message bus evicts cache entries by entity tag | CRM records, policy documents — any data changed via known write operations |
| **Dependency-key tagging** | Write-time tags include all entity IDs the result depends on; any change to any entity evicts the entry | Multi-entity lookups (contact + account + orders) |
| **Stale-while-revalidate** | Return cached value immediately; refresh asynchronously in background when within revalidation window | High-frequency, slowly-changing read tools |

```java
// Event-driven cache invalidation consumer
@Component
public class CacheInvalidationConsumer {

    private final Cache cache;

    @KafkaListener(topics = "crm.changes")
    public void handleCrmChange(ChangeEvent event) {
        String tag = event.entityType() + ":" + event.entityId();
        int evicted = cache.evictByTag(tag);
        log.info("cache.invalidated tag={} entriesEvicted={}", tag, evicted);
    }
}
```

> ⚠️ **Never rely on TTL expiry alone for data that has compliance or safety implications** — account statuses, permission flags, policy documents. Combine a short TTL with event-driven invalidation so that changes are reflected within seconds, not minutes.

### Rate-Limiting Middleware

Rate limiting at the tool layer controls the **agent's own call rate**, preventing runaway reasoning loops, prompt-injection-driven tool flooding, and accidental DoS of shared backends. The pattern is a token bucket per `(tool, session)` pair:

```java
@Component
public class RateLimitingMiddleware implements ToolMiddleware {

    // Per-(tool, session) token bucket — capacity and refill rate configured per tool
    private final ConcurrentHashMap<String, TokenBucket> buckets =
        new ConcurrentHashMap<>();

    @Override
    public Map<String, Object> apply(String toolName, Map<String, Object> args,
                                      ToolHandler next) throws ToolExecutionException {
        String sessionId = CallingContext.current().sessionId();
        String key = sessionId + ":" + toolName;

        TokenBucket bucket = buckets.computeIfAbsent(key,
            k -> new TokenBucket(
                toolRateLimitConfig.capacity(toolName),
                toolRateLimitConfig.refillRate(toolName)
            ));

        if (!bucket.consume()) {
            throw new RateLimitedException(
                "Tool " + toolName + " rate limit exceeded for session " + sessionId);
        }

        return next.handle(args, null);
    }
}
```

> 💡 **Set rate limit capacity and refill rate per tool based on expected legitimate usage.** A knowledge base search tool may legitimately be called 10 times in a complex research task — calling it 200 times in a single session is almost certainly a reasoning loop or an attack. Alert on `RATE_LIMITED` errors in production — they are a diagnostic signal for both agent behaviour regressions and security events.

### Recommended Middleware Stack Order

| Position | Middleware | Reason for Order |
|---|---|---|
| 1 (outermost) | Rate-limiting | Reject runaway sessions before any resource is consumed |
| 2 | Logging | Capture all calls including those rejected by cache or rate limit |
| 3 | Caching | Return cached results before authorization or dispatch |
| 4 | Cost tracking | Deduct budget only for calls that reach dispatch |
| 5 (innermost) | Handler | Execute the actual tool operation |

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-MW-001` | Logging middleware MUST emit all events through OpenTelemetry — trace IDs and span contexts MUST be propagated to the distributed tracing backend (Vol 2, chap12). |
| `ADV-MW-002` | Caching middleware MUST only cache tools declared `sideEffects: read-only` — caching write or destructive tool results is prohibited. |
| `ADV-MW-003` | Cache invalidation MUST use event-driven tag eviction for compliance-sensitive data — TTL-only expiry is insufficient for account statuses, permission flags, and policy documents. |
| `ADV-MW-004` | Rate-limiting middleware MUST use a per-(tool, session) token bucket — per-tool-only buckets allow a single session to exhaust shared tool capacity. |
| `ADV-MW-005` | `RATE_LIMITED` errors MUST be recorded as named metrics and MUST trigger alerts — they are a diagnostic signal for both reasoning loop regressions and prompt injection attacks. |
| `ADV-MW-006` | The middleware stack order MUST follow: rate-limiting → logging → caching → cost tracking → handler — deviations require explicit architectural justification. |

---

## Section 34.6 — Tool Versioning and Schema Evolution

### Why Tool Schemas Change

Enterprise systems evolve. A CRM API adds a field, renames a parameter, or changes the type of a response property. A security review mandates removing a sensitive field from a tool's output schema. Every one of these changes is a potential breaking change for agents that rely on the old schema.

In a conventional software system, schema changes are managed through API versioning and migration guides. In an agent system, the consumer of the schema is partly the LLM — which was prompted with the old description — and partly tool chain code — which extracted specific fields by name. **Both must be managed.**

### Versioning at the Contract Level

Every tool contract carries a `version` field following semantic versioning conventions:

| Version Type | Example | Schema Changes | Migration Required |
|---|---|---|---|
| **Patch** | `1.0.0 → 1.0.1` | Description clarifications only | No — safe to deploy without agent changes |
| **Minor** | `1.0.0 → 1.1.0` | New optional input/output fields | No — existing agents continue to work |
| **Major** | `1.0.0 → 2.0.0` | Renamed/removed fields, changed types, changed `sideEffects` | Yes — explicit agent migration required |

```json
{
  "name": "crm.getContact",
  "version": "2.0.0",
  "description": "Retrieve a contact record by email. Returns contact name, account ID, and tier. Note: the 'phone' field was removed in v2.0.0 — use crm.getContactDetails for full contact data.",
  "inputSchema": {
    "type": "object",
    "properties": { "email": { "type": "string", "format": "email" } },
    "required": ["email"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "contactId": { "type": "string" },
      "name": { "type": "string" },
      "accountId": { "type": "string" },
      "tier": { "type": "string" }
    }
  }
}
```

### Tool Aliasing

Tool aliasing allows multiple names in the registry to resolve to the same handler. Three distinct use cases:

| Use Case | Mechanism |
|---|---|
| **Backward compatibility** | Old name (`search.kb`) aliased to new handler — agents prompted with the old name continue to work during migration |
| **Context-specific naming** | `findPolicy` (customer-facing copilot) and `policy.search` (analyst agent) both resolve to the same handler |
| **A/B routing** | Two aliases point to two different handler implementations — configurable percentage of calls route to the experimental handler for canary testing |

```java
@Component
public class ToolRegistry {

    private final Map<String, ToolContract> contracts = new ConcurrentHashMap<>();
    private final Map<String, String> aliases = new ConcurrentHashMap<>();

    public void registerAlias(String alias, String target) {
        if (!contracts.containsKey(target)) {
            throw new IllegalArgumentException("Alias target '" + target + "' not registered.");
        }
        aliases.put(alias, target);
    }

    public ToolContract lookup(String name) {
        // Resolve alias before lookup
        String resolved = aliases.getOrDefault(name, name);
        return contracts.get(resolved);
    }
}
```

> 📝 Aliases are resolved at lookup time, not at registration time. In dynamic registries with out-of-order registration, validate alias targets as part of the startup health check.

### Running Multiple Versions Simultaneously

When a major version is released, the Tool Registry can hold both `crm.getContact.v1` and `crm.getContact.v2` simultaneously. The naming convention is `namespace.operation.vN` for explicitly versioned names, with `namespace.operation` aliased to the current stable major version.

> ⚠️ **Running multiple tool versions simultaneously increases registry size and the complexity of the semantic retrieval index.** Prune deprecated versions aggressively — set a sunset date for each deprecated version at the time of deprecation and enforce it. A registry that accumulates stale versions indefinitely degrades tool selection quality over time.

### Version Migration Checklist

| Step | Action |
|---|---|
| 1 | Run existing v1 contract tests against the v2 handler — expect failures that document exactly what broke |
| 2 | Write v2 contract tests that pass against the v2 handler |
| 3 | Run all agent integration tests with v2 tools injected — fix chain code that relied on removed or renamed fields |
| 4 | Deploy v2 alongside v1; alias `namespace.operation` to v2 for a canary percentage of traffic |
| 5 | Monitor error rates and tool selection accuracy during the canary window |
| 6 | Roll back to v1 if regressions appear; promote v2 to 100% when stable |
| 7 | Set and enforce the v1 sunset date |

> 💡 **Treat tool descriptions as prompts that must be regression-tested.** When a tool's description changes significantly — even without a schema change — re-run LLM selection accuracy tests to confirm the model still selects the correct tool for the target query set. Description drift causes silent selection errors that are easy to miss without targeted tests.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ADV-VER-001` | Every tool contract MUST carry a semantic version field — versionless contracts are not permitted in production registries. |
| `ADV-VER-002` | Major version changes (breaking schema changes, `sideEffects` classification changes) MUST follow the migration checklist: run v1 contract tests against v2, deploy alongside, canary, sunset. |
| `ADV-VER-003` | Tool aliases MUST be used to maintain backward compatibility during major version migrations — agents MUST NOT be required to update tool names before the migration window closes. |
| `ADV-VER-004` | Deprecated tool versions MUST have a declared sunset date set at the time of deprecation — undated deprecations accumulate indefinitely and degrade retrieval index quality. |
| `ADV-VER-005` | Tool description changes (patch or minor) MUST trigger re-run of LLM tool selection accuracy tests — description drift causes silent selection regressions that schema tests do not catch. |

---

## Section 34.7 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `ADV-FAN-001` | Fan-out | Per-tool status annotation for every call in the batch |
| `ADV-FAN-002` | Fan-out | Global deadline: `min(session_budget, 1.2 × max_tool_timeout)` |
| `ADV-FAN-003` | Fan-out | `requireAll` policy declared explicitly at call site |
| `ADV-FAN-004` | Fan-out | Isolated execution context per tool in batch |
| `ADV-DYN-001` | Dynamic selection | Embed and index at registration time |
| `ADV-DYN-002` | Dynamic selection | Safety-critical tools always pinned |
| `ADV-DYN-003` | Dynamic selection | Compact embedding model (≤30M params) for retrieval |
| `ADV-DYN-004` | Dynamic selection | Two-stage selection for registries >500 tools |
| `ADV-DYN-005` | Dynamic selection | Re-embed on description change |
| `ADV-CHAIN-001` | Chaining | Explicit field extraction at all handoff points |
| `ADV-CHAIN-002` | Chaining | Taint-label external data at point of retrieval |
| `ADV-CHAIN-003` | Chaining | Hostile-tainted data → HITL gate before write tools |
| `ADV-CHAIN-004` | Chaining | Maximum chain depth: 10 calls |
| `ADV-CHAIN-005` | Chaining | Chain depth and call history in checkpointed state |
| `ADV-ASYNC-001` | Async | Submit-poll for operations >30s |
| `ADV-ASYNC-002` | Async | Durable job tokens — survive agent restart |
| `ADV-ASYNC-003` | Async | Job tokens in checkpointed state |
| `ADV-ASYNC-004` | Async | Poll tool declared `sideEffects: read-only` |
| `ADV-ASYNC-005` | Async | Webhook callbacks require HMAC/bearer authentication |
| `ADV-MW-001` | Middleware | Logging via OpenTelemetry — trace ID propagation |
| `ADV-MW-002` | Middleware | Cache only `read-only` tools |
| `ADV-MW-003` | Middleware | Event-driven tag eviction for compliance data |
| `ADV-MW-004` | Middleware | Per-(tool, session) token bucket for rate limiting |
| `ADV-MW-005` | Middleware | `RATE_LIMITED` metrics + alerts |
| `ADV-MW-006` | Middleware | Stack order: rate-limit → log → cache → cost → handler |
| `ADV-VER-001` | Versioning | Semantic version field on every contract |
| `ADV-VER-002` | Versioning | Major version migration checklist enforced |
| `ADV-VER-003` | Versioning | Aliases for backward compatibility during migrations |
| `ADV-VER-004` | Versioning | Sunset date set at time of deprecation |
| `ADV-VER-005` | Versioning | LLM selection accuracy tests on description changes |

---

## Section 34.8 — Chapter Boundary: Relationship to Adjacent Chapters

| Chapter | Relationship |
|---|---|
| **chap32** (Tool Abstraction Layer) | chap34 assumes all `TOOL-*` rules are in place. Fan-out uses the Stage-2 authorization check; middleware wraps the Stage-4 dispatch; taint tracking extends Stage-5 result validation; versioning governs the contract lifecycle across all stages. |
| **chap33** (Enterprise Systems Integration) | Enterprise backend adapters are the tool handlers that chap34's middleware pipeline wraps. The fan-out pattern is the primary mechanism for querying multiple enterprise backends in parallel. The cache invalidation consumer integrates with the message queue patterns from chap33 Section 33.5. |
| **chap8** (State Manager) | Structured suspension (Section 34.4) and chain call history (Section 34.3) both require chap8 state checkpointing. Job tokens for long-running tools MUST be stored in the checkpointed state. |
| **chap12** (HITL) | `ADV-CHAIN-003` requires a HITL confirmation gate when hostile-tainted data flows into write tools. chap12 defines the implementation of confirmation gates. |
| **chap35** (RAG Pipelines) | Chapter 35 builds the RAG pipeline directly on chap34 primitives: fan-out for parallel dense + sparse retrieval, tool chaining for the query-transform → retrieve → rerank → assemble pipeline, and middleware caching for frequent queries. |
| **Vol 2, chap3** | Java implementations of the fan-out dispatcher, semantic tool selector, taint policy, async submit-poll handlers, and middleware pipeline. |
| **Vol 2, chap12** | OTel trace propagation from the logging middleware connects to distributed tracing architecture. |
