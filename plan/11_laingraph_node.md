# LangGraph Nodes — Orchestrator Graph

> See also: `09_llm_calls.py` for the LLM calls each node makes.
> See also: `04_streaming_hitl.md` for SSE events each node emits.
> See also: `01_task_lifecycle.md` for state transitions each node triggers.

---

## Graph State

The shared state object passed between all nodes:

```python
class OrchestratorState(TypedDict):
    # Identity
    task_id: str
    trace_id: str
    user_id: str | None

    # Input
    original_input: str
    resolved_input: str | None          # set after clarification
    clarification_rounds: int           # 0–3
    confirmation_revisions: int         # 0–2

    # Validation
    validation_result: ValidationResult | None
    write_intent: WriteIntentResult | None

    # Planning
    plan: Plan | None
    plan_id: str | None
    available_agents: list[AgentCard]   # retrieved from registry

    # Memory
    mem0_context: list[str]             # injected at planning time
    tool_performance_hints: dict

    # Execution
    subtask_results: dict[str, SpecialistResult | None]  # subtask_id → result
    hitl_response: dict | None          # user's response after interrupt

    # Synthesis
    conflicts: list[Conflict]
    report: TaskReport | None

    # Error tracking
    current_node: str
    error: str | None
    retry_count: int
```

---

## Node Map

```
[entry]
    │
    ▼
[validate_node] ──(vague)──► [clarify_node] ──► loop back to [validate_node]
    │
    │(clear)
    ▼
[plan_node]
    │
    ▼
[confirm_node]  ◄── interrupt (user confirms/rejects) ──► loop back to [plan_node]
    │
    │(confirmed)
    ▼
[route_node]
    │
    ▼
[execute_node] ──(write/execute tool)──► interrupt (human approval)
    │          ──(low confidence)──────► interrupt (human context)
    │
    ▼
[synthesize_node] ──(high conflict)──► interrupt (human resolution)
    │
    ▼
[output_node]
    │
    ▼
[memory_node]  ← runs after output is returned to user (background)
```

---

## Node 1 — validate_node

**Responsibility:** Run TaskValidator and WriteIntentDetector in parallel. Route to clarify or plan.

**LLM calls:** `validate_task`, `detect_write_intent` (parallel)

**State reads:** `original_input`, `resolved_input`, `available_agents`
**State writes:** `validation_result`, `write_intent`, `current_node`

**Transitions:**
- `task_type == "clear"` → `plan_node`
- `task_type in ("vague", "unbounded")` → `clarify_node`
- `clarification_rounds >= 3` → `plan_node` (force proceed, add assumptions)

**SSE emits:**
- `{ type: "status", message: "Analyzing your task..." }`

```python
async def validate_node(state: OrchestratorState) -> OrchestratorState:
    await db.update_task_status(state.task_id, "validating")
    await emit_sse(state.task_id, {"type": "status", "message": "Analyzing your task..."})

    validation, write_intent = await asyncio.gather(
        validate_task(state.resolved_input or state.original_input,
                      state.available_agents, state.trace_id),
        detect_write_intent(state.original_input, state.trace_id)
    )

    await db.log_event(state.task_id, "validation_complete",
                       node="validate_node",
                       payload={"clarity_score": validation.clarity_score,
                                "task_type": validation.task_type})

    return {**state, "validation_result": validation, "write_intent": write_intent,
            "current_node": "validate_node"}
```

**Router function:**
```python
def route_after_validate(state: OrchestratorState) -> str:
    if state.validation_result.task_type in ("vague", "unbounded"):
        if state.clarification_rounds >= 3:
            return "plan_node"  # force proceed after max rounds
        return "clarify_node"
    return "plan_node"
```

---

## Node 2 — clarify_node

**Responsibility:** Generate clarification questions, emit them to user, interrupt. On resume, append user answers to resolved_input.

**LLM calls:** `generate_clarifications`

**State reads:** `validation_result`, `available_agents`, `clarification_rounds`
**State writes:** `resolved_input`, `clarification_rounds`

**Interrupts:** Yes — waits for user answers via `POST /tasks/{id}/resume`

**SSE emits:**
- `{ type: "clarification_needed", questions: [...], round: N }`
- `{ type: "scope_warning", ... }` (if task_type == "unbounded")

```python
async def clarify_node(state: OrchestratorState) -> OrchestratorState:
    questions = await generate_clarifications(
        task=state.original_input,
        validation=state.validation_result,
        agent_cards=state.available_agents,
        trace_id=state.trace_id
    )

    await emit_sse(state.task_id, {
        "type": "clarification_needed",
        "questions": questions.questions,
        "round": state.clarification_rounds + 1,
        "max_rounds": 3
    })
    await db.update_task_status(state.task_id, "awaiting_clarification")

    # Interrupt — resume will inject user answers into state.hitl_response
    raise NodeInterrupt("awaiting_clarification")
```

**On resume:** The resume handler injects `user_answers` into `state.hitl_response`. The node appends answers to `resolved_input` and increments `clarification_rounds`.

---

## Node 3 — plan_node

**Responsibility:** Load memory context, call TaskPlanner, persist plan to Postgres, check cost ceiling.

**LLM calls:** `plan_task`, `extract_assumptions`

**State reads:** `resolved_input`, `available_agents`, `write_intent`, `confirmation_revisions`
**State writes:** `plan`, `plan_id`

**Interrupts:** Yes — if cost ceiling exceeded, waits for approval.

**SSE emits:**
- `{ type: "status", message: "Building your plan..." }`
- `{ type: "plan_created", plan: {...} }`
- `{ type: "cost_warning", estimated_cost: X }` (if ceiling exceeded)
- `{ type: "plan_revised", plan: {...}, version: N }` (if this is a revision)

```python
async def plan_node(state: OrchestratorState) -> OrchestratorState:
    await db.update_task_status(state.task_id, "planning")

    # Load memory in parallel
    mem0_context, tool_hints = await asyncio.gather(
        get_relevant_memory(state.resolved_input, state.user_id),
        db.get_tool_performance_hints()
    )

    plan = await plan_task(
        resolved_input=state.resolved_input,
        agent_cards=state.available_agents,
        mem0_context=mem0_context,
        tool_performance_hints=tool_hints,
        trace_id=state.trace_id
    )

    # Persist before showing to user
    plan_id = await db.save_plan(state.task_id, plan)

    event_type = "plan_revised" if state.confirmation_revisions > 0 else "plan_created"
    await emit_sse(state.task_id, {
        "type": event_type,
        "plan": plan.dict(),
        "version": state.confirmation_revisions + 1
    })

    # Cost ceiling check
    if plan.estimated_cost_usd > settings.COST_CEILING_USD:
        await emit_sse(state.task_id, {"type": "cost_warning", ...})
        raise NodeInterrupt("cost_ceiling")

    return {**state, "plan": plan, "plan_id": plan_id, "mem0_context": mem0_context}
```

---

## Node 4 — confirm_node

**Responsibility:** Interrupt to wait for user plan confirmation. On resume, route to execute or re-plan.

**LLM calls:** None

**State reads:** `plan`, `confirmation_revisions`
**State writes:** `confirmation_revisions`

**Interrupts:** Yes — always (plan confirmation is always required).

**SSE emits:** None (plan_created already emitted by plan_node).

**Router function:**
```python
def route_after_confirm(state: OrchestratorState) -> str:
    response = state.hitl_response
    if response["response_type"] == "plan_confirm":
        return "route_node"
    elif response["response_type"] == "plan_reject":
        if state.confirmation_revisions >= 2:
            return "output_node"  # max revisions, return error
        return "plan_node"       # re-plan with feedback
    return "output_node"  # cancel
```

---

## Node 5 — route_node

**Responsibility:** Check agent health, match sub-tasks to healthy agents, mark degraded sub-tasks early.

**LLM calls:** None

**State reads:** `plan`
**State writes:** `available_agents` (filtered to healthy)

**SSE emits:** `{ type: "subtask_degraded", domain: X }` for any domain with no healthy agent.

---

## Node 6 — execute_node

**Responsibility:** Execute all sub-tasks according to execution_mode. Make A2A calls. Handle HITL triggers.

**LLM calls:** None (A2A calls to specialist agents)

**State reads:** `plan`, `available_agents`, `write_intent`
**State writes:** `subtask_results`

**Interrupts:** Yes — on write/execute tool approval, low confidence, or if adaptive re-plan needed.

**SSE emits:**
- `{ type: "subtask_started", agent, subtask_description }`
- `{ type: "subtask_progress", agent, message }` (forwarded from specialist SSE)
- `{ type: "subtask_completed", agent, finding_summary }`
- `{ type: "subtask_degraded", agent, domain }` (on timeout/failure)
- `{ type: "human_input_required", reason, question }` (HITL triggers)

**Execution mode dispatch:**
```python
async def execute_node(state: OrchestratorState) -> OrchestratorState:
    if state.plan.execution_mode == "parallel":
        results = await execute_parallel(state)
    elif state.plan.execution_mode == "sequential":
        results = await execute_sequential(state)
    else:  # adaptive
        results = await execute_adaptive(state)
    return {**state, "subtask_results": results}
```

---

## Node 7 — synthesize_node

**Responsibility:** Run pairwise conflict detection, synthesize all findings into final report structure.

**LLM calls:** `detect_conflict` (pairwise), `extract_assumptions`

**State reads:** `subtask_results`, `plan`
**State writes:** `conflicts`, `report`

**Interrupts:** Yes — on high-severity conflict (HITL for resolution).

**SSE emits:**
- `{ type: "synthesizing", message: "Combining findings..." }`
- `{ type: "conflict_detected", conflict: {...} }` (if any conflicts found)
- `{ type: "human_input_required", reason: "conflict" }` (if high severity)

---

## Node 8 — output_node

**Responsibility:** Finalize the TaskReport, persist to Postgres, emit `completed` or `degraded` SSE event.

**LLM calls:** None

**State reads:** `report`, `conflicts`, `plan`, `subtask_results`
**State writes:** `report` (finalized)

**SSE emits:**
- `{ type: "completed", report: TaskReport }` (status == completed or degraded)
- `{ type: "failed", error_summary: str }` (status == failed)

---

## Node 9 — memory_node

**Responsibility:** Write task memory to Mem0, episodic memory to Postgres, extract preferences. Runs AFTER output is returned to user — does not block the response.

**LLM calls:** `summarize_task`, `extract_preferences`

**State reads:** `report`, `task` (from DB)
**State writes:** Nothing in graph state (writes to Mem0 + Postgres)

**SSE emits:** None (user already has the report)

```python
# In the graph, memory_node is the final node and runs in background:
# graph.add_edge("output_node", "memory_node")
# The SSE connection can close after output_node — memory_node is fire-and-forget
```