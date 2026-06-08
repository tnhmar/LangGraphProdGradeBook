# Production-Grade Agentic AI: LangGraph + Python Frameworks
## Table of Contents — v2

**Subtitle:** *Implementing 609 Spec Rules — Zero Theory, All Code*
**Language:** Python
**Target pages:** ~500

---

## Framework Stack — Quick Reference

| Layer | Framework | Notes |
|---|---|---|
| Graph Orchestration | LangGraph | Core engine |
| LLM Abstraction | LiteLLM | Universal provider router, cost tracking |
| Structured Output | Instructor + Pydantic v2 | Self-healing retries, minimal boilerplate |
| Context Assembly | LangChain ChatPromptTemplate + tiktoken | Typed slots, token budgeting |
| Short-Term Memory | LangGraph MessagesState + trim_messages() | Native, zero-overhead |
| Long-Term Memory (Episodic + Semantic) | mem0 | Auto-extracts facts, deduplicates, conflict-resolves |
| Procedural Memory | LangSmith Prompt Hub + versioned config | Version-controlled workflow templates |
| Vector Store | Qdrant (via mem0 config) | Native integrated filtering, HNSW, payload indexing |
| Hybrid Retrieval | LangChain retrievers + rank_bm25 + flashrank | BM25 + dense + reranker |
| Knowledge Graph | Neo4j + langchain-neo4j | Cypher, property graph |
| Semantic Cache | LangChain RedisSemanticCache | Redis-backed, cosine threshold |
| Tool Layer | LangChain StructuredTool + ToolNode | Native LangGraph integration |
| MCP | FastMCP + langchain-mcp-adapters | Server + client, minimal code |
| A2A Protocol | Google A2A Python SDK | Official SDK, TaskSender/Receiver |
| HITL | LangGraph interrupt() | Native, no extra lib |
| Observability + Tracing | Langfuse | Tracing, cost, datasets — single platform |
| Evaluation | Langfuse Evals + RAGAS + DeepEval | Online (Langfuse) + offline CI (RAGAS/DeepEval) |
| Security / Guardrails | Guardrails AI + Presidio | Input/output validators + PII redaction |
| Resilience | Tenacity | Retry, backoff, circuit breaker |
| Serving | LangGraph Platform + FastAPI | Platform for managed; FastAPI for custom |
| Config Management | Pydantic Settings + LangGraph configurable | Type-safe, no hardcodes |

---

## Front Matter (~8 pages)

- Companion repo structure and how to run every chapter's code
- One-page framework stack reference card
- `pyproject.toml` with all pinned dependencies
- Environment bootstrap: `.env` template, Langfuse / Qdrant / mem0 setup

---

## Part I — The Agentic Runtime Core (~85 pages)

### Chapter 1 — The Core Loop
*Implements: specs/chap2 — Core Agentic Loop*

- 1.1 `AgentState` TypedDict + Pydantic v2: mandatory fields — `goal`, `beliefs`, `intentions`, `token_budget`, `cycle_count`
- 1.2 `PerceiveNode`: sensory filtering, schema validation, Presidio PII redaction at ingestion
- 1.3 `ReasonNode`: LiteLLM call → Instructor `patch()` → typed `ToolRequest | FinalAnswer | ReasoningTrace`
- 1.4 `ActionNode`: `ToolNode` dispatch, `ToolInvocationRecord` audit log emission
- 1.5 `ReflectNode`: strict Pydantic enum output — `CONTINUE | REVISE | RETRY | TERMINATE | ESCALATE` + `goalDelta`
- 1.6 Conditional edges wiring all four nodes into a `StateGraph`
- 1.7 Embedded vs. explicit reflection: config flag, when to promote to explicit node
- 1.8 Langfuse span emission per node via `@observe` decorator — zero extra boilerplate
- 1.9 Cycle trace record: all 13 mandatory fields emitted as Langfuse span attributes
- 1.10 `pytest` suite: each node tested in isolation with mocked dependencies

### Chapter 2 — Termination, Resource Limits & Loop Failure Guards
*Implements: specs/chap2 §2.6, §2.7*

- 2.1 `TerminationPolicy` Pydantic config: max cycles, token ceiling, wall-clock timeout, cost cap
- 2.2 Hard cycle guard: enforced in orchestrator, never in prompts
- 2.3 Cumulative token budget: tracked in `AgentState.token_budget`, graceful stop branch
- 2.4 Infinite loop detector: `goalDelta` tracking across N cycles + `repeated_action_detector`
- 2.5 Stuck-state detector: N consecutive cycles without valid `ToolRequest` or `FinalAnswer` (N configurable, default 3)
- 2.6 Goal divergence detector: cosine distance between current goal embedding and original — rollback on threshold breach
- 2.7 Per-failure-mode LangGraph branches: each failure type routes to its own recovery subgraph
- 2.8 Full test suite: each detector triggered by synthetic state sequences

### Chapter 3 — State Schema, Checkpointing & Recovery
*Implements: specs/chap3 — State Management*

- 3.1 BDI fields in `AgentState`: `beliefs`, `goal`, `intentions` as mandatory named fields
- 3.2 `SqliteSaver` (dev) → `AsyncPostgresSaver` (production): setup and connection pooling
- 3.3 Checkpoint strategy: checkpoint-every-cycle vs. milestone config policy
- 3.4 State reducers: `add_messages`, append-only custom reducers
- 3.5 `RecoveryNode`: resumes from last checkpoint, re-injects goal, re-anchors context
- 3.6 State schema migration: versioned TypedDict with backward-compatible fields

### Chapter 4 — Structured Output, Validation & Self-Healing
*Implements: specs/chap4 — Output Validation*

- 4.1 Pydantic v2 output models as contracts — one model per node output type
- 4.2 Instructor `patch()` modes: `TOOLS`, `JSON`, `MD_JSON` — decision guide for each
- 4.3 `ValidationNode` as a dedicated graph node, not inline
- 4.4 Repair loop: parse-fail → error feedback injected → retry — max retry ceiling enforced
- 4.5 Rejection branch: validation failures beyond ceiling → `ESCALATE`
- 4.6 Langfuse logging of validation failures as events for offline analysis

### Chapter 5 — LLM Abstraction, Reliability & Cost Enforcement
*Implements: specs/chap0 §P.4 — LLM Infrastructure*

- 5.1 LiteLLM router: provider fallback chain, model version pinning
- 5.2 Tenacity decorators: exponential backoff, per-exception retry strategies
- 5.3 Circuit breaker: open/half-open/closed state machine wrapping LiteLLM calls
- 5.4 Cost tracking: LiteLLM `success_callback` → `AgentState.token_budget` update
- 5.5 Budget circuit breaker: `BudgetExceededError` → graceful stop branch
- 5.6 Fallback routing: primary model fails → cheaper model, logged to Langfuse
- 5.7 `RunnableConfig` + Pydantic Settings: all model config externalised, no hardcodes

### Chapter 6 — Prompt Architecture & Context Assembly
*Implements: specs/chap0 §P.3 — Context Window as a Resource*

- 6.1 `ChatPromptTemplate` with named slots: `ROLE`, `ACTIVE_GOAL`, `EVIDENCE`, `TOOLS`, `HISTORY`, `HEADROOM`
- 6.2 `tiktoken` token-budget enforcement per slot — trim before overflow, never after
- 6.3 `ContextAssemblyNode`: builds prompt from `AgentState`, enforces per-slot budgets
- 6.4 KV-cache-friendly structure: stable prefix, variable content at end
- 6.5 LangSmith prompt hub: version-pinned templates, A/B via `configurable`
- 6.6 Goal re-injection: compact structured `goalSummary` in every `ACTIVE_GOAL` slot

---

## Part II — Memory Architecture (~80 pages)

### Chapter 7 — Short-Term Memory & Working Context
*Implements: specs/chap5 §5.2, §5.5 — Working Memory & In-Context Hierarchy*

- 7.1 `MessagesState` + `add_messages` reducer: LangGraph native
- 7.2 `trim_messages()`: token-budget sliding window, eviction order policy
- 7.3 Summarization node: LangChain summarization chain → compressed history into `HISTORY` slot
- 7.4 Six-step injection pipeline as `MemoryInjectionNode`
- 7.5 Compact labelled memory format enforced: `Memory [episodic 2026-01-15]: ...` — never raw records
- 7.6 Four-question injection policy as a pre-inject check function

### Chapter 8 — Long-Term Memory with mem0
*Implements: specs/chap5 §5.2–5.7 — Episodic, Semantic, Lifecycle, Write-Gate, Expiry*

- 8.1 mem0 architecture: `Memory` class, `MemoryClient` (cloud) vs. self-hosted with Qdrant backend
- 8.2 Production config: Qdrant vector store + LiteLLM + embedder wired through `Memory.from_config()`
- 8.3 `MemoryType` enum in `AgentState`: `EPISODIC | SEMANTIC | PROCEDURAL`
- 8.4 `MemoryWriteNode`: five-check write gate before every `memory.add()` — significance, type classification, PII sensitivity (Presidio pre-check), dedup query, TTL metadata assignment
- 8.5 Episodic writes: `memory.add(content, user_id, metadata={"type":"episodic","ttl_expires_at":...,"importance_score":...})`
- 8.6 Semantic extraction: mem0 auto-extracts durable facts, deduplicates, conflict-resolves via internal graph — no custom consolidation pipeline
- 8.7 `MemoryReadNode`: six-step injection pipeline — `memory.search(query, user_id, limit=20)` → rerank → top-5 → labelled format → `EVIDENCE` slot
- 8.8 TTL expiry: APScheduler daily job calls `memory.delete()` on records past `ttl_expires_at`
- 8.9 Importance-threshold pruning: scheduled job → archive to S3 → delete
- 8.10 `MemoryInspector`: `memory.get_all(user_id)` exposed as a Langfuse event for compliance + debugging
- 8.11 Optimistic locking: version-checked writes via mem0 `metadata.version` field
- 8.12 Langfuse audit events: `WriteAuditRecord`, `RetrievalAuditRecord`, `ExpiryAuditRecord` per spec
- 8.13 Multi-tenancy: `user_id` isolation enforced at every `memory.add()` / `memory.search()` call
- 8.14 `pytest` suite: write-gate all 5 checks, retrieval scoring, TTL expiry, conflict resolution, prompt drift prevention

### Chapter 9 — Procedural Memory
*Implements: specs/chap5 §5.4 — Procedural Memory*

- 9.1 Versioned system prompts as source code: annotated load-bearing lines, change-controlled
- 9.2 LangSmith Prompt Hub: workflow templates as named, versioned `ProceduralRecord`
- 9.3 `PerceiveNode` retrieval: task-type → fetch matching workflow template → inject into `ROLE` slot
- 9.4 Tool pre/post-condition schemas as typed Pydantic models in `ToolRegistry`
- 9.5 `pytest`: procedural memory retrieval, version rollback test

### Chapter 10 — Qdrant Production Setup & Vector Store Architecture
*Implements: specs/chap7 §vector-store*

- 10.1 HNSW index config: `m`, `ef_construct`, `ef` tuning for agent memory workloads
- 10.2 Metric selection: cosine for text, dot product for normalised embeddings — locked at collection creation
- 10.3 Payload indexing: `created_at`, `memory_type`, `importance_score`, `user_id`, `ttl_expires_at`
- 10.4 Integrated filtering: filter-aware HNSW traversal — no post-filter recall collapse
- 10.5 Multi-tenancy: payload-level user isolation, not collection-per-user
- 10.6 Qdrant as mem0's backend: `Memory.from_config()` collection config reference

### Chapter 11 — Hybrid Retrieval Pipeline
*Implements: specs/chap7 §hybrid-retrieval, §retrieval-quality*

- 11.1 Two-stage architecture: BM25 recall → cross-encoder reranker for precision
- 11.2 BM25 first stage: `rank_bm25` with tokenization pipeline
- 11.3 Dense vector stage: mem0 `memory.search()` + Qdrant `query_points` with payload filters
- 11.4 RRF fusion: `reciprocal_rank_fusion()` combining BM25 + dense scores
- 11.5 `flashrank` cross-encoder reranker: `RerankRequest` → ranked candidates
- 11.6 HyDE: hypothetical answer generation → embed → use as query vector
- 11.7 CRAG: retrieval grader node → re-retrieval branch in LangGraph
- 11.8 Quality metrics as `pytest` assertions: Precision@5 ≥ 0.7, Recall@5 ≥ 0.8, MRR ≥ 0.6

### Chapter 12 — Knowledge Graph Retrieval
*Implements: specs/chap9 — Knowledge Graph*

- 12.1 Neo4j + `langchain-neo4j`: `GraphCypherQAChain`, `Neo4jGraph`
- 12.2 Entity extraction pipeline: LLM → `(subject, predicate, object)` triples → Neo4j write
- 12.3 Multi-hop Cypher queries: agent-generated traversals for relational questions
- 12.4 Routing policy: vector search vs. graph traversal — classifier node in LangGraph
- 12.5 Cross-store consistency: graph write triggers mem0 tag update

### Chapter 13 — Semantic Caching
*Implements: specs/chap8 — Caching*

- 13.1 `LangChain RedisSemanticCache`: cosine threshold, embedding model config
- 13.2 Exact cache: Redis exact key, TTL per cache tier
- 13.3 Cache invalidation: tool-dependent outputs keyed with tool-result hash
- 13.4 Cache hit/miss rate → Langfuse event + Prometheus counter

---

## Part III — Tools, Actions & Protocol Stack (~85 pages)

### Chapter 14 — Tool Design: The Production Action Interface
*Implements: specs/chap10 — Tool Layer*

- 14.1 `StructuredTool` with full Pydantic input/output schemas
- 14.2 `ToolRegistry`: per-run scoped tool inventory, least-privilege assignment
- 14.3 `PolicyConfig` Pydantic model: allowed tools per agent type/run — enforced in code, never in prompts
- 14.4 Tool sandboxing: subprocess timeout, resource limits for code execution tools
- 14.5 `ToolException` → fallback tool branch in graph
- 14.6 `ToolInvocationRecord`: every call, result, latency, agent ID — emitted to Langfuse
- 14.7 `ToolNode` with parallel execution: `asyncio.gather` for multi-tool calls
- 14.8 `pytest`: policy enforcement, sandboxing, audit emission

### Chapter 15 — ReAct: Tool Execution Loops
*Implements: specs/chap12 — ReAct Pattern*

- 15.1 `create_react_agent` internals: pre-built vs. custom graph — when to switch
- 15.2 `ToolMessage` injection, multi-step history management
- 15.3 Tool-result validation: `ValidationNode` before next reasoning step
- 15.4 Langfuse trace per tool call — automatic via `@observe`

### Chapter 16 — MCP: Hosts, Clients & Servers
*Implements: specs/chap20 — MCP Deep Dive*

- 16.1 MCP roles: Host (LangGraph agent), Client (adapter), Server (FastMCP)
- 16.2 FastMCP server: `@mcp.tool`, `@mcp.resource`, `@mcp.prompt` decorators
- 16.3 Transport: stdio for local dev, Streamable HTTP for production
- 16.4 `langchain-mcp-adapters`: `MCPToolkit` → LangGraph `ToolNode`
- 16.5 Multi-server aggregation: N FastMCP servers behind one `MCPToolkit`
- 16.6 OAuth2 token scoping for MCP server authentication
- 16.7 Full code: three FastMCP servers + LangGraph agent consuming all three

### Chapter 17 — A2A: Agent-to-Agent Protocol
*Implements: specs/chap21 — A2A Protocol*

- 17.1 A2A task lifecycle in code: `submitted → working → input-required → completed`
- 17.2 `AgentCard` generation and `/.well-known/agent.json` endpoint
- 17.3 Google A2A Python SDK: `TaskSender`, `TaskReceiver`, streaming SSE updates
- 17.4 Scoped delegation tokens: parent agent credentials → child agent scope
- 17.5 Error propagation: subagent failure → parent `ESCALATE` branch
- 17.6 Full code: orchestrator + two subagents with live A2A task streaming

### Chapter 18 — ANP & Decentralized Agent Identity
*Implements: specs/chap22 — The Agent Web / ANP*

- 18.1 W3C DID creation: `did:web` method with Python `did` library
- 18.2 DIDComm v2 message envelope: `didcomm-py`
- 18.3 Verifiable Credentials: capability attestation for cross-org agent calls
- 18.4 ANP SDK: publishing agent identity, resolving remote agents
- 18.5 Full code: cross-organization agent handshake with DID verification

---

## Part IV — Multi-Agent Orchestration (~65 pages)

### Chapter 19 — Supervisor Pattern
*Implements: specs/chap13 — Multi-Agent Coordination*

- 19.1 Supervisor graph: orchestrator LLM → `Command` handoff to subgraphs
- 19.2 `graph.add_node(subgraph.compile())`: subgraph composition pattern
- 19.3 State namespacing: `AgentState.subagent_state` isolation, no bleed
- 19.4 Supervisor fallback: subagent error → supervisor retry → escalation path
- 19.5 Langfuse: parent trace with child spans per subagent — automatic nesting
- 19.6 Full code: three-subagent supervisor with typed `Command` handoffs

### Chapter 20 — Parallel Execution & Fan-Out/Fan-In
*Implements: specs/chap15 — Parallelism*

- 20.1 `Send` API: dynamic fan-out with per-worker state slices
- 20.2 `asyncio.gather` for concurrent `ainvoke` across subgraphs
- 20.3 Custom reducers: `operator.add` for lists, `max` for scores, conflict-safe merges
- 20.4 Map-reduce topology: parallel workers → aggregation node → synthesis
- 20.5 Full code: parallel research agent — 5 concurrent searchers, RRF-merged synthesis

### Chapter 21 — Human-in-the-Loop
*Implements: specs/chap16 — HITL*

- 21.1 `interrupt()`: pause graph, expose state to human
- 21.2 `Command(resume=value)`: inject human decision back into state
- 21.3 State editing at breakpoint: `graph.update_state()` before resume
- 21.4 Time-travel: `get_state_history()` → replay from a past checkpoint
- 21.5 `PolicyConfig`: declarative approval rules — which `ActionNode` outputs require human gate
- 21.6 FastAPI + SSE endpoint: surface `interrupt` events to frontend, receive `resume` payload
- 21.7 Full code: approval-gated agent, state edit, time-travel replay

### Chapter 22 — Durable Execution & Long-Running Workflows
*Implements: specs/chap34 — Durable Execution*

- 22.1 `AsyncPostgresSaver`: cross-restart persistence
- 22.2 `RecoveryNode`: load last checkpoint, re-inject goal, replay from milestone
- 22.3 Celery + Redis: background task queue for deferred graph runs
- 22.4 LangGraph Platform background runs API
- 22.5 Full code: workflow surviving three simulated process restarts

---

## Part V — Observability, Evaluation, Security & Deployment (~75 pages)

### Chapter 23 — Observability with Langfuse
*Implements: specs/chap17 — Observability*

- 23.1 Langfuse SDK: `@observe` decorator, `langfuse_context`, manual span control
- 23.2 LangGraph integration: `LangfuseCallbackHandler` for automatic trace per node
- 23.3 Cycle trace record: all 13 mandatory fields emitted as Langfuse span attributes
- 23.4 Cost tracking: token usage per node → Langfuse `usage` field → cost dashboard
- 23.5 Memory audit events: `WriteAuditRecord`, `RetrievalAuditRecord`, `ExpiryAuditRecord` all routed to Langfuse
- 23.6 Prometheus metrics: custom counters for cycle count, error rate, latency p99 per node
- 23.7 Grafana dashboard: token usage, error rates, HITL interrupt frequency, per-node latency
- 23.8 Full code: agent with complete Langfuse + Prometheus instrumentation

### Chapter 24 — Evaluation with Langfuse, RAGAS & DeepEval
*Implements: specs/chap18 — Evaluation Framework*

- 24.1 Langfuse datasets: curating golden test cases, scoring runs online
- 24.2 Langfuse online evals: LLM-as-judge scorers attached to production traces
- 24.3 RAGAS: `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall` — offline suite
- 24.4 DeepEval: `pytest`-style assertions — `assert_faithful()`, `assert_answer_relevancy()`
- 24.5 3+ machine-checkable success criteria defined before any implementation
- 24.6 CI/CD gate: GitHub Actions — RAGAS scores must meet threshold to merge
- 24.7 Full code: complete eval suite wired to CI, Langfuse dataset sync

### Chapter 25 — Resilience & Error Handling
*Implements: specs/chap19 — Resilience*

- 25.1 Tenacity: `@retry` with `wait_exponential`, `stop_after_attempt`, per-exception strategies
- 25.2 Hard vs. soft failure branches: separate LangGraph routing per error class
- 25.3 Dead-letter state: `AgentState.dead_letter` snapshot on terminal failure — Langfuse trace linked
- 25.4 Error budget tracking: per-node error rates in Prometheus, alert on SLO breach
- 25.5 Graceful degradation cascade: full model → cheap model → static fallback → `ESCALATE`
- 25.6 Full code: resilient agent with all fallback tiers and dead-letter handling

### Chapter 26 — Security, Guardrails & Audit
*Implements: specs/chap20 — Security*

- 26.1 Trust boundary ADR template: input channels, tool surfaces, delegation paths
- 26.2 `PerceiveNode` security layer: Presidio PII redaction → Guardrails AI input validator
- 26.3 Output security: Guardrails AI output validator before `ActionNode`
- 26.4 `PolicyConfig` enforcement: tool authorization at runtime — rejected calls → `WriteRejection` event
- 26.5 Immutable audit log: every security-relevant event → Langfuse event + append-only DB record
- 26.6 Prompt injection mitigations: input sanitization, trust-level tagging on all external content
- 26.7 Full code: hardened agent with Presidio + Guardrails wrapping the full loop

### Chapter 27 — Deployment, Serving & Configuration
*Implements: specs/chap22 — Production Deployment*

- 27.1 LangGraph Platform: `langgraph.json` config, cron triggers, webhooks, Studio debugging
- 27.2 FastAPI: custom SSE streaming endpoint, `/invoke`, `/stream`, health probes
- 27.3 Docker + Kubernetes: multi-stage `Dockerfile`, `livenessProbe`, rolling deploy
- 27.4 Pydantic Settings + `.env`: all config externalised, no hardcodes
- 27.5 Feature flags: `Flagsmith` SDK — runtime agent behaviour toggles
- 27.6 Model A/B testing: `configurable` field in `RunnableConfig` → LiteLLM router

---

## Part VI — Framework Integration Deep Dives (~40 pages)

*Install → configure → integrate with LangGraph → known limits. No theory.*

### Chapter 28 — LangGraph Platform & Studio
- `langgraph.json` full reference, background runs, webhooks
- LangGraph Studio: live state inspection, visual debug

### Chapter 29 — Pydantic AI & Instructor Integration
- `pydantic-ai` agent as a typed `ReasonNode` replacement inside a LangGraph graph
- Instructor `patch()` all modes — decision guide per use case

### Chapter 30 — CrewAI as a LangGraph Subgraph
- Wrapping a CrewAI `Crew` as a compiled LangGraph subgraph node
- When CrewAI role-based abstraction is worth the overhead vs. raw supervisor

### Chapter 31 — FastMCP Server Patterns
- Production FastMCP server: auth, rate-limiting, multi-resource patterns
- Composing multiple FastMCP servers behind a single LangGraph agent

### Chapter 32 — Framework Selection Scorecard
*Implements: specs/chap31 — Decision Guide*

- 10-axis rubric applied to every Python framework in the stack
- One-page decision flowchart: requirements → framework choice per layer
- Anti-patterns: what each framework is explicitly wrong for

---

## Part VII — 7 Production Usecases (~110 pages, ~15 pages each)

*Each usecase: architecture → spec rules by ID → full Python code → eval suite → deploy config*

### Usecase 1 — Customer Support Agent with Memory & Escalation
**Stack:** LangGraph loop + mem0 (episodic + semantic) + Qdrant RAG + HITL interrupt + Langfuse
- Full `PerceiveNode → ReasonNode → ActionNode → ReflectNode` loop
- mem0 for user preference memory and interaction history
- Qdrant hybrid retrieval for product documentation
- HITL `interrupt()` on escalation threshold with FastAPI frontend SSE

### Usecase 2 — Parallel Research & Report Generation Agent
**Stack:** `Send` fan-out + Qdrant + BM25+RRF + HyDE + Instructor structured report + Langfuse
- `Send` API fan-out to 5 parallel searchers (Tavily + Qdrant)
- HyDE retrieval for abstract queries
- RRF-merged synthesis node with Pydantic report schema (Instructor)

### Usecase 3 — Code Review & Refactoring Multi-Agent System
**Stack:** Supervisor + Analyzer/Refactorer/Validator subagents + sandboxed tool + Langfuse
- Supervisor → three typed subgraphs with `Command` handoffs
- Sandboxed code execution tool with subprocess timeout
- Full LangSmith + Langfuse trace per review cycle

### Usecase 4 — Financial Analysis Agent (Utility-Based)
**Stack:** Explicit utility function + SQL agent (SQLAlchemy) + Neo4j + LiteLLM cost tracking + Langfuse
- Explicit utility function: `score = w1*accuracy + w2*speed - w3*cost`
- SQL agent over time-series financial data
- Neo4j for entity relationships, per-node cost tracking in Langfuse

### Usecase 5 — Content Moderation Pipeline (Multi-Agent Parallel)
**Stack:** Parallel specialists + Guardrails AI + Presidio + dead-letter queue + Tenacity
- Parallel specialist agents: policy, context, severity
- Guardrails AI validators on all ingestion and output boundaries
- Dead-letter queue for edge cases requiring human review

### Usecase 6 — DevOps Incident Response Agent (Durable)
**Stack:** FastMCP servers (GitHub/Slack/PagerDuty) + HITL approval gate + Celery durable execution + Langfuse
- MCP servers: GitHub API, Slack, PagerDuty, kubectl
- Durable execution: Celery-backed, resumes after restart
- HITL `interrupt()` approval gate for all destructive actions

### Usecase 7 — Enterprise Document Intelligence Platform (Full Stack)
**Stack:** Unstructured.io ingestion + Qdrant hybrid retrieval + A2A + mem0 full memory + Langfuse online evals + CI/CD gate
- Multi-modal ingestion: PDF, tables, images via `unstructured`
- Full hybrid retrieval: BM25 + mem0/Qdrant + RRF + flashrank reranker
- A2A protocol for inter-department agent collaboration
- Complete observability: Langfuse traces + Prometheus + Grafana
- RAGAS + DeepEval eval suite wired to GitHub Actions CI/CD

---

## Appendices (~8 pages)

### Appendix A — `pyproject.toml` Reference
All dependencies with pinned versions for reproducible builds.

### Appendix B — Spec Rule ID → Chapter Cross-Reference
All 609 spec rules mapped to the implementing chapter and section.

### Appendix C — LangGraph API Quick Reference
Node types, edge types, state reducers, `Send` API, `interrupt()`, `Command`.

### Appendix D — Langfuse Instrumentation Cheatsheet
`@observe`, `CallbackHandler`, `langfuse_context`, dataset sync, online eval setup.

---

## Page Budget

| Section | Pages |
|---|---|
| Front Matter | 8 |
| Part I — Runtime Core (Ch. 1–6) | 85 |
| Part II — Memory (Ch. 7–13) | 80 |
| Part III — Tools & Protocols (Ch. 14–18) | 85 |
| Part IV — Multi-Agent (Ch. 19–22) | 65 |
| Part V — Observability, Security, Deploy (Ch. 23–27) | 75 |
| Part VI — Framework Deep Dives (Ch. 28–32) | 40 |
| Part VII — 7 Usecases | 110 |
| Appendices | 8 |
| **Total** | **~556 pages** |

> **Compression to ~500:** Merge Ch.10+11 (Qdrant + Hybrid Retrieval) into one chapter; trim each usecase shared infrastructure setup by sharing a common bootstrap section (~25 pages saved). Final target: **~500 pages**.

---

*v2 — 2026-06-08 — mem0 (episodic + semantic), Langfuse (observability + eval), production framework stack locked.*
