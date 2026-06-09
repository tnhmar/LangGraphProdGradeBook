# Chapter 2 — Termination, Resource Limits & Loop Failure Guards
**Outline** | ~13 pages | ~2,100 words prose + code | `Tex/chapters/chapter02.tex`

---

## 2.1 `TerminationPolicy`: Pydantic Config Model

**Pedagogical goal:** Show why the four termination axes — cycle count, token ceiling, wall-clock timeout, cost cap — must be a single validated, serialisable object rather than four separate constants scattered across the codebase.

**Key design decisions to resolve in prose:**
- Why `BaseModel` and not `@dataclass`: Pydantic gives free JSON serialisation for checkpointer storage (a `@dataclass` requires manual `__json__` or `asdict`); validators enforce invariants at object construction, not at first use; `model_config = {"frozen": True}` prevents mid-run mutation.
- Why all four axes are present even if some are `None`: a `TerminationPolicy` with only `max_cycles` set is valid; the orchestrator checks whichever fields are not `None`. Omitting axes entirely creates silent unbounded behaviour.
- `cost_cap` is in USD float, not tokens — explain why the distinction matters.

**Exact signatures:**
```python
class TerminationPolicy(BaseModel):
    max_cycles:         int             # required; no default
    token_ceiling:      int             # required; no default
    wall_clock_timeout: float | None    # seconds; None = no wall-clock limit
    cost_cap:           float | None    # USD; None = no cost limit
    model_config = {"frozen": True}

    @field_validator("max_cycles")
    def max_cycles_positive(cls, v) -> int: ...

    @field_validator("token_ceiling")
    def ceiling_positive(cls, v) -> int: ...
```

**Code listing:** `TerminationPolicy` full class with all validators. File: `src/agent/termination.py`. ~25 lines.

**Forward refs:** Ch.9 (cost accounting replaces `cost_cap` stub with real LiteLLM usage callbacks); Ch.4 (policy passed through `config["configurable"]` so every node can read it without being passed explicitly).

---

## 2.2 LangGraph Native Recursion Guard: `recursion_limit` and `GraphRecursionError`

**Pedagogical goal:** Establish `recursion_limit` as a runtime ceiling that LangGraph enforces regardless of orchestrator logic — and show why it is the *last* defence, not the first.

**Key design decisions to resolve in prose:**
- `recursion_limit` is set in `b.compile(recursion_limit=N)` — it is not a `config["configurable"]` key. Explain why: it is a graph-structural property, not a per-run parameter.
- `GraphRecursionError` must be caught explicitly in the runner, not allowed to propagate uncaught. An uncaught `GraphRecursionError` in production means the caller gets an exception instead of a structured `termination_status: RESOURCE_LIMIT` result.
- The right value for `recursion_limit`: set it to `max_cycles * 4` (four nodes per cycle), not to `max_cycles`. Explain the arithmetic.
- Never raise `recursion_limit` as a workaround for missing termination logic.

**Exact signatures:**
```python
# In build_graph():
b.compile(checkpointer=checkpointer, recursion_limit=policy.max_cycles * 4)

# In runner:
def run_agent(graph, initial_state, config) -> AgentRunResult:
    try:
        result = graph.invoke(initial_state, config=config)
        return AgentRunResult(state=result, stopped_by="completion")
    except GraphRecursionError:
        return AgentRunResult(state=None, stopped_by="recursion_limit")
```

**Code listing:** `run_agent()` with `try/except GraphRecursionError`, returning `AgentRunResult`. File: `src/agent/runner.py`. ~20 lines.

**Forward refs:** §2.3 (why `recursion_limit` alone is insufficient and the second guard is needed); Ch.28 (integration test that deliberately triggers `GraphRecursionError`).

---

## 2.3 Application-Level Max-Cycle Policy: The Second Independent Guard Layer

**Pedagogical goal:** Explain the two-layer guard design — LangGraph's `recursion_limit` and the orchestrator's `cycle_count` check — and why neither alone is sufficient.

**Key design decisions to resolve in prose:**
- **Why `recursion_limit` alone is not enough**: it fires only when the graph structure itself runs out of steps. It has no knowledge of semantic progress — it will not fire if a loop terminates naturally after one cycle but then re-enters via a `RETRY` branch twenty times.
- **Why application-level alone is not enough**: the orchestrator check depends on the node returning a correct `cycle_count` increment every cycle. A node bug that skips the increment means the application guard never fires. `recursion_limit` is structural — it cannot be bypassed by application code.
- **Where the application guard lives**: a `pre_cycle_gate` routing function inserted as the first step of each cycle, before `perceive_node`. It reads `state["cycle_count"]` and `state["_policy"]`, and routes to `cycle_limit_exceeded_node` if the limit is breached.
- **Increment location**: `cycle_count` is incremented inside `perceive_node` (first node of each cycle) — never inside `reflect_node` — to ensure the count reflects cycles entered, not cycles completed.

**Exact signatures:**
```python
def pre_cycle_gate(state: AgentState) -> str:
    if state["cycle_count"] >= state["_policy"].max_cycles:
        return "cycle_limit_exceeded"
    if state["token_budget"] < _MINIMUM_SAFE_BUDGET:
        return "budget_exhausted"
    return "perceive_node"
```

**Code listing:** `pre_cycle_gate` routing function + updated `build_graph()` showing the new entry-point edge. File: `src/agent/graph.py`. ~15 lines (listing focused on the delta from Ch.1 graph).

**Forward refs:** §2.8 (each failure route branches to its own recovery subgraph, not a shared `failure_node`); Ch.4 (policy injection via `config["configurable"]`).

---

## 2.4 Cumulative Token Budget: `AgentState.token_budget` Graceful Stop

**Pedagogical goal:** Show how `token_budget` in `AgentState` is decremented each cycle and how exhaustion produces a graceful routing stop, not an exception.

**Key design decisions to resolve in prose:**
- `token_budget` is set from `TerminationPolicy.token_ceiling` at graph entry via `make_initial_state`. It decrements in `reason_node` (Ch.1 §1.3). This section shows the *routing* consequence: when `token_budget < _MINIMUM_SAFE_BUDGET`, the pre-cycle gate catches it before the node fires.
- Distinguish between the hard `BudgetExhaustedError` (Ch.1 §1.3) used in tests and the graceful routing path used in production. In production the orchestrator catches budget exhaustion before the node, not inside it.
- `token_budget` is a floor, not a counter. Never let it go negative.
- The `pre_cycle_gate` combines both checks (cycle count and token budget) so there is one routing function at the loop entry, not two.

**Exact signatures:**
```python
# AgentState additions (already present from Ch.1; documented here for clarity):
token_budget: int   # decremented by reason_node each cycle

_MINIMUM_SAFE_BUDGET = 500   # from src/agent/nodes/reason.py

def pre_cycle_gate(state: AgentState) -> str:
    policy = state["_policy"]
    if state["cycle_count"] >= policy.max_cycles:
        return "cycle_limit_exceeded"
    if state["token_budget"] < _MINIMUM_SAFE_BUDGET:
        return "budget_exhausted"
    return "perceive_node"
```

**Code listing:** `pre_cycle_gate` (combined cycle + budget gate) + updated graph entry wiring. File: `src/agent/graph.py`. ~18 lines.

**Forward refs:** Ch.9 (real token accounting via LiteLLM usage callbacks replaces the char-count estimate); Ch.4 (wall-clock timeout check added to the same gate).

---

## 2.5 Infinite Loop Detector: `goal_delta` Tracking + `repeated_action_detector`

**Pedagogical goal:** Show how `goal_delta` from `ReflectNode` (Ch.1 §1.5) becomes the signal for loop detection when it appears `NO_CHANGE` across N consecutive cycles.

**Key design decisions to resolve in prose:**
- Back-reference: `goal_delta: Literal["ADVANCED","NO_CHANGE","REGRESSED"]` is already emitted by `reflect_node` into `AgentState` via `CycleTrace.goal_delta`. The detector reads `cycle_traces[-N:]` and counts consecutive `NO_CHANGE` values. N is configurable; default 3.
- `repeated_action_detector`: separately checks whether `CycleTrace.action_invoked` is identical across N consecutive cycles. A loop where `goal_delta == "NO_CHANGE"` AND the same tool is called repeatedly is a stronger signal than either alone.
- Return type: `InfiniteLoopSignal` with fields `triggered: bool`, `consecutive_no_change: int`, `repeated_action: str | None`.
- The detector is a pure function of state — no model call, no I/O.

**Exact signatures:**
```python
@dataclass
class InfiniteLoopSignal:
    triggered:             bool
    consecutive_no_change: int
    repeated_action:       str | None   # tool name if same tool N times

def detect_infinite_loop(
    state: AgentState,
    window: int = 3,
) -> InfiniteLoopSignal: ...

def detect_repeated_action(
    state: AgentState,
    window: int = 3,
) -> str | None: ...   # returns tool name or None
```

**Code listing:** Both detector functions + `loop_guard_node` that calls them and routes to `infinite_loop_recovery` if triggered. File: `src/agent/guards.py`. ~30 lines.

**Forward refs:** §2.8 (infinite loop routes to its own recovery subgraph); §2.9 (`test_infinite_loop_triggers_after_N_no_change_cycles`).

---

## 2.6 Stuck-State Detector: N Consecutive Cycles Without Valid Output

**Pedagogical goal:** Define "stuck" precisely — the model is executing but producing neither a valid `ToolRequest` nor a `FinalAnswer` — and show how to detect it from `CycleTrace.reason_output_type`.

**Key design decisions to resolve in prose:**
- A stuck state is distinct from an infinite loop. An infinite loop *makes progress in form* (it calls tools) but not in substance (goal doesn't advance). A stuck state doesn't even produce a valid action — the model repeatedly returns `ReasoningTrace` or malformed output.
- Detection signal: `CycleTrace.reason_output_type` is `"REASONING_TRACE"` (or absent) for N consecutive cycles. N is configurable; default 3.
- Why `"REASONING_TRACE"` triggers stuck-state rather than normal reasoning: a single `ReasoningTrace` is legitimate (Ch.1 §1.3 explains this). N consecutive traces with no intervening `TOOL_CALL` or `FINAL_ANSWER` indicates the model is circling without committing.
- `StuckStateSignal` mirrors `InfiniteLoopSignal` for uniform routing interface.

**Exact signatures:**
```python
@dataclass
class StuckStateSignal:
    triggered:               bool
    consecutive_non_actions: int
    last_output_types:       list[str]   # for diagnostics

def detect_stuck_state(
    state: AgentState,
    window: int = 3,
) -> StuckStateSignal: ...
```

**Code listing:** `detect_stuck_state` + `stuck_state_guard_node` (reads `cycle_traces[-N:]`, counts `reason_output_type` values, routes to `stuck_state_recovery`). File: `src/agent/guards.py`. ~25 lines.

**Forward refs:** §2.8 (stuck-state routes to clarification/handoff subgraph, not same recovery as infinite loop); §2.9 (`test_stuck_state_triggers_after_N_reasoning_trace_cycles`).

---

## 2.7 Goal Divergence Detector: Cosine Distance + Rollback

**Pedagogical goal:** Show how embedding-based cosine distance between the original goal and the current goal summary detects semantic drift — and why caching the original embedding in `AgentState` is mandatory.

**Key design decisions to resolve in prose:**
- **Cost flag**: embedding the current goal every cycle is expensive. The solution: embed the original goal once at graph entry and store it in `AgentState.goal_embedding: list[float]`. At detection time, embed only the *current* goal summary. One embedding call per detection check, not per cycle.
- **Threshold**: `DIVERGENCE_THRESHOLD = 0.25` (cosine *distance*; 0.0 = identical, 1.0 = orthogonal). Configurable via `TerminationPolicy` extension or separate config.
- **Check frequency**: goal divergence is checked every N cycles (default every 5), not every cycle, to amortise the embedding cost.
- **Rollback target**: LangGraph's `SqliteSaver` checkpointer stores state after every node. Rollback means calling `graph.get_state(config, checkpoint_id)` at the last cycle where cosine distance was below threshold.
- `GoalDivergenceSignal` must include the distance value and the checkpoint ID to roll back to.

**Exact signatures:**
```python
@dataclass
class GoalDivergenceSignal:
    triggered:           bool
    cosine_distance:     float
    rollback_checkpoint: str | None   # checkpointer thread_id/checkpoint_id

def embed_text(text: str) -> list[float]: ...   # thin wrapper; model configurable
def cosine_distance(a: list[float], b: list[float]) -> float: ...
def detect_goal_divergence(
    state: AgentState,
    threshold: float = 0.25,
) -> GoalDivergenceSignal: ...

# AgentState additions:
goal_embedding:             list[float] | None  # set once at entry; never mutated
goal_divergence_checked_at: int | None          # cycle_count of last check
```

**Code listing — two blocks:**
1. `cosine_distance` + `embed_text` + `detect_goal_divergence` (~22 lines). File: `src/agent/guards.py`.
2. `goal_divergence_guard_node` with every-N-cycles check and rollback route (~18 lines). File: `src/agent/guards.py`.

**Forward refs:** Ch.6 (embedding model selection and caching strategy); Ch.4 (SqliteSaver checkpoint retrieval API for rollback); §2.9 (`test_goal_divergence_triggers_above_threshold`).

---

## 2.8 Per-Failure-Mode LangGraph Branches

**Pedagogical goal:** Show that each failure type routes to its own recovery subgraph — not a shared `failure_node` — because recovery strategy differs fundamentally across failure modes.

**Key design decisions to resolve in prose:**
- Guard nodes (`loop_guard_node`, `stuck_state_guard_node`, `goal_divergence_guard_node`) are inserted after `reflect_node`, before `pre_cycle_gate`.
- **Why separate recovery subgraphs**: infinite loop → increment retry counter, then escalate; stuck state → clarification request or human handoff; goal divergence → rollback to last checkpoint and re-anchor. Merging them into one `failure_node` loses recovery semantics.
- Each recovery node writes a structured `termination_status` into `AgentState` and routes to `END` (or back to `perceive_node` for rollback recovery).
- Recovery nodes: `infinite_loop_recovery_node`, `stuck_state_recovery_node`, `goal_divergence_recovery_node`.
- Present updated wiring as a `(source, condition, target)` table covering all new edges.

**Exact signatures:**
```python
def infinite_loop_recovery_node(state: AgentState) -> dict:
    # increments retry counter; escalates if ceiling reached
    ...

def stuck_state_recovery_node(state: AgentState) -> dict:
    # emits ESCALATE; surfaces to human handoff
    ...

def goal_divergence_recovery_node(
    state: AgentState, config: dict
) -> dict:
    # performs checkpoint rollback, re-anchors goal in state
    ...
```

**Code listing:** Updated `build_graph()` showing all guard nodes, recovery nodes, and their conditional edges. File: `src/agent/graph.py`. ~35 lines (delta from §2.3).

**Forward refs:** Ch.3 (human handoff subgraph that `stuck_state_recovery_node` delegates to); Ch.5 (retry ceiling policy for infinite loop recovery); Ch.4 (rollback API details for `goal_divergence_recovery_node`).

---

## 2.9 Full Test Suite: Guard Triggering Tests

**Pedagogical goal:** Show that each detector is a pure function testable by constructing synthetic `CycleTrace` sequences — no compiled graph, no model calls.

**Key design decisions to resolve in prose:**
- Detectors read `state["cycle_traces"]` — a list of dicts. Tests construct this list directly with `make_cycle_trace()` helper.
- `GraphRecursionError` path requires a compiled graph with `recursion_limit=2` and a node that always returns `CONTINUE`. Show the integration test pattern.
- Each test is named for the specific trigger condition.

**Exact signatures:**
```python
# tests/test_guards.py

def make_cycle_trace(
    goal_delta: str = "NO_CHANGE",
    reason_output_type: str = "REASONING_TRACE",
    action_invoked: str | None = None,
) -> dict: ...

def test_infinite_loop_triggers_after_3_no_change_cycles(): ...
def test_infinite_loop_no_trigger_below_window(): ...
def test_stuck_state_triggers_after_3_reasoning_traces(): ...
def test_stuck_state_no_trigger_with_valid_tool_call(): ...
def test_goal_divergence_triggers_above_threshold(): ...
def test_goal_divergence_no_trigger_below_threshold(): ...

# tests/test_runner.py
def test_graphrecursion_caught_by_runner(): ...
    # compiles graph with recursion_limit=2, state loops forever,
    # asserts run_agent returns stopped_by="recursion_limit"
```

**Code listing — two blocks:**
1. `make_cycle_trace` fixture + 6 detector unit tests (~30 lines). File: `tests/test_guards.py`.
2. `test_graphrecursion_caught_by_runner` integration test (~20 lines). File: `tests/test_runner.py`.

**Forward refs:** Ch.28 (full integration test suite adding multi-cycle state evolution assertions); Ch.9 (token budget tests extended with real usage objects).

---

## Listing Inventory

| Label | What it shows | Est. lines | File |
|---|---|---|---|
| `lst:terminationpolicy` | `TerminationPolicy` Pydantic model + validators | 25 | `src/agent/termination.py` |
| `lst:runner` | `run_agent()` with `try/except GraphRecursionError` | 20 | `src/agent/runner.py` |
| `lst:precyclegate` | `pre_cycle_gate` routing function + updated graph entry wiring | 18 | `src/agent/graph.py` |
| `lst:infiniteloop` | `detect_infinite_loop` + `detect_repeated_action` + `loop_guard_node` | 30 | `src/agent/guards.py` |
| `lst:stuckstate` | `detect_stuck_state` + `stuck_state_guard_node` | 25 | `src/agent/guards.py` |
| `lst:goaldivergence-fns` | `cosine_distance` + `embed_text` + `detect_goal_divergence` | 22 | `src/agent/guards.py` |
| `lst:goaldivergence-node` | `goal_divergence_guard_node` with every-N-cycles check | 18 | `src/agent/guards.py` |
| `lst:graph-ch2` | Updated `build_graph()` with all guard + recovery nodes | 35 | `src/agent/graph.py` |
| `lst:guard-tests` | `make_cycle_trace` + 6 detector unit tests | 30 | `tests/test_guards.py` |
| `lst:runner-test` | `test_graphrecursion_caught_by_runner` integration test | 20 | `tests/test_runner.py` |

**Total listings:** 10 blocks · **Total listing lines:** ~243 · **Target prose:** ~2,100 words across 9 sections.
