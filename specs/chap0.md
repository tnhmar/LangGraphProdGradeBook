# Prologue — LLM Foundations for Agent Engineers

This chapter establishes the **specification baseline** for the entire book: before you design agents, you must treat the LLM as a precisely specified computational component, not as a magical chatbot. Volume 2 will implement these specs as concrete Java + LangGraph patterns; this file captures the contracts Volume 1 defines.

---

## P.1 — LLM as a Stateless Function

### Context

An LLM is a **stateless, parameterized function** that maps a sequence of tokens to a probability distribution over the next token, then samples from that distribution to generate output tokens. There is no built‑in notion of a "session" or conversation; every call is structurally identical to the first.

### What (the spec)

1. **Functional interface**
   - Input: a bounded sequence of tokens (prompt) plus configuration parameters (temperature, max tokens, etc.).
   - Processing: prefill pass over all input tokens, followed by stepwise autoregressive generation.
   - Output: a sequence of output tokens produced probabilistically.

2. **Statelessness**
   - The model holds **no persistent memory** between calls.
   - There is **no background process** updating weights or knowledge during inference.
   - The model has **no awareness** that previous calls ever happened.

3. **Knowledge cutoff**
   - Model weights encode compressed knowledge up to a fixed training cutoff date.
   - Events, APIs, and data after that date are absent from weights and must be supplied as context.

### Why (engineering implications)

- You **cannot rely on the model** for long‑term memory, stateful workflows, or real‑time knowledge.
- Every piece of information needed to perform a task must be explicitly present in the input tokens of the current call.
- Any behavior that looks like memory is an illusion created by repeatedly re‑supplying past state in the prompt.

### How (system design rules)

1. **State lives outside the LLM**
   - All persistent state must be stored in external systems: databases, vector stores, key‑value stores, or LangGraph state.
   - Agent code is responsible for assembling the relevant slice of state into the prompt for each call.

2. **Model invocations are pure from the system’s perspective**
   - Treat `callLLM(prompt, params)` as a **pure, referentially transparent operation**: same inputs → distribution over outputs.
   - All side effects (state writes, tool calls) must be driven by the agent layer, not by the model internals.

3. **Design for explicit context reconstruction**
   - For every agent step, you must be able to reconstruct the exact prompt that the model saw (for debugging, replay, and evals).
   - Persist:
     - User inputs.
     - Retrieved documents.
     - Tool outputs.
     - System instructions.

4. **KV‑cache awareness (performance spec)**
   - Shared prefixes (system prompt, stable instructions) should be kept stable to maximize cache hits.
   - Avoid injecting highly variable content at the very start of prompts; it defeats caching and increases latency.

---

## P.2 — Capability and Limitation Profile

### Context

Every component in a production system has a spec: what it does well, what it cannot do, and what guarantees it offers. The LLM is no different. Mis‑specifying this leads directly to brittle agent architectures.

### What (the spec)

The model’s observable behavior can be summarized as a table of **capabilities**, **structural limitations**, and **required system responses**:

- Instruction following → degrades with ambiguity → enforce prompt discipline and input normalization.
- Structured output → occasional schema violations → add output validation and repair layers.
- Code synthesis → does not execute code → run all code in external sandboxes and capture results as new context.
- Language understanding → no live data / knowledge cutoff → integrate retrieval and tool layers for fresh information.
- Extended reasoning → degrades over long chains → decompose tasks and use multi‑step workflows.
- Memory → no persistence between calls → build an explicit memory architecture (short‑term + long‑term).
- Stochastic output → sampling variance → add retry, n‑best, and consensus strategies where needed.
- Hallucinations → confident but incorrect answers under uncertainty → design grounding and verification pipelines.
- Action → cannot act on the world → expose tools and actions behind explicit, audited interfaces.

### Why (engineering implications)

- LLM limitations are **design requirements**, not annoyances to be wished away.
- Each limitation maps **directly** to an architectural layer: memory, retrieval, tools, orchestration, verification.
- Ignoring a limitation simply moves the failure to production, where it is more expensive to fix.

### How (system design rules)

1. **Always design with a mitigation column**
   - For any new capability you rely on, explicitly write down: "What if the model fails here? How do we detect and contain it?"

2. **Separate concerns into layers**
   - Memory layer: implements read/write interfaces and retrieval strategies; the LLM only *reads* memory via prompts.
   - Tool layer: defines a typed, audited surface for side‑effects (APIs, DB updates, external calls).
   - Orchestration layer (e.g., LangGraph graphs): defines control flow, retries, fallbacks, and human‑in‑the‑loop boundaries.
   - Verification layer: post‑checks outputs against schemas, business rules, or secondary models.

3. **Make limitations explicit in contracts**
   - Document for each agent: assumptions about instruction clarity, schema rigidity, maximum reasoning depth, and allowed failure modes.
   - Feed these constraints into graph design (e.g., when to branch to a sub‑agent vs. when to call a tool vs. when to escalate to a human).

---

## P.3 — Context Window as a Managed Resource

### Context

The context window is the **binding runtime constraint** of LLM‑based systems. Everything the model can condition on during inference must fit here: instructions, history, tools, retrieved documents, and space for the generated answer.

### What (the spec)

1. **Context slots**
   - System prompt: stable instructions, tool definitions, and formatting requirements.
   - Conversation history: selected prior user/agent turns.
   - Tool results: raw or summarized outputs from external calls.
   - Retrieved knowledge: chunks from vector search or databases.
   - Current input: the user’s latest request or task.
   - Output headroom: reserved tokens for the model’s reply.

2. **Cost and latency coupling**
   - Input token count drives **latency** and **monetary cost**.
   - In multi‑step agent workflows, both cost and latency compound linearly (or worse) with the number and size of prompts.

3. **KV‑cache behavior**
   - Prefix caching optimizes repeated prefixes across calls.
   - Large or unstable prefixes reduce cache effectiveness.

### Why (engineering implications)

- Treat context as a scarce, high‑value resource, not as an infinite dumping ground.
- Over‑stuffing prompts with "maybe useful" content causes context dilution: signal gets buried under noise.
- Prompt structure is both a **quality control** mechanism and a **cost control** mechanism.

### How (system design rules)

1. **Allocate, don’t accumulate**
   - Explicitly budget tokens per slot (instructions, history, retrieved docs, headroom).
   - Implement trimming strategies for history (e.g., summarize older turns, keep only decision points).

2. **Design prompt templates as contracts**
   - Define a rigid skeleton for prompts: fixed header, explicit sections, clearly marked tool outputs.
   - Guarantee that all required fields can fit within `max_context_tokens` minus headroom.

3. **Summarization and condensation**
   - Use summarization agents to compress long histories or documents into dense, task‑aligned representations.
   - Prefer task‑specific summaries over generic ones (e.g., "legal‑risk summary" vs. "generic summary").

4. **Cache‑friendly structure**
   - Keep system prompts and global instructions stable over time.
   - Avoid changing the order or wording of top‑level sections without reason; each change has a performance cost.

---

## P.4 — LLM Serving and Infrastructure

### Context

From an infrastructure perspective, the LLM is just another network service dependency. It has latency, throughput limits, costs, failure modes, and version drift. The agent system must treat it as such.

### What (the spec)

1. **Deployment modes**
   - Hosted API: vendor‑managed, billed per token, subject to external rate limits and privacy constraints.
   - Local / self‑hosted inference: infra‑managed, billed in hardware and ops, with full control over models and data locality.

2. **Operational properties**
   - Latency: time‑to‑first‑token and total generation time.
   - Availability: SLAs or SLOs for error budgets and uptime.
   - Versioning: model versions, deprecations, and behavior changes.
   - Cost: per‑call and per‑workflow cost, especially for long‑running graphs.

3. **Failure modes**
   - Hard failures: network errors, timeouts, rate‑limit errors, server errors.
   - Soft failures: obviously incorrect responses, schema violations, or degenerate outputs.

### Why (engineering implications)

- Agent workflows are only as robust as their weakest dependency; the LLM is usually the most expensive and least predictable.
- Without explicit cost and reliability modeling, agents will behave acceptably in prototypes and fail in production under load.

### How (system design rules)

1. **Treat LLM calls as untrusted remote calls**
   - Always wrap them with timeouts, retries with backoff, and circuit breakers.
   - Distinguish between transient errors (retryable) and permanent ones (immediate failure or fallback path).

2. **Track costs at the workflow level**
   - Instrument each graph run with token usage per node and per tool.
   - Define per‑task budgets and enforce them (e.g., stop or degrade gracefully when exceeded).

3. **Design for rate limits and quotas**
   - Implement centralized throttling, queueing, and load‑shedding strategies.
   - Prefer graceful degradation (smaller contexts, cheaper models) over hard failures.

4. **Version‑aware orchestration**
   - Persist which model version was used for each decision or output.
   - When changing versions, run A/B or shadow tests, not big‑bang switches.

5. **Hosted vs. local trade‑offs**
   - Prefer hosted APIs during exploration and early iteration.
   - Move latency‑sensitive, high‑volume, or sensitive‑data workloads to self‑hosted inference when the cost and governance case justifies it.

---

This file defines the **Prologue‑level specification contract** for Volume 1. All later parts (agent anatomy, memory, multi‑agent orchestration, protocols, and frameworks) build directly on these assumptions about how LLMs behave, what they can and cannot do, and how they must be treated as infrastructure components in production systems.