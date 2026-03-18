# Planning, Agent Discovery & Execution Modes

> See also: `09_llm_calls.md` for TaskPlanner call details.
> See also: `01_task_lifecycle.md` for how the plan is persisted.
> See also: `08_security.md` for write intent and tool permission checks during planning.

---

## What the Planner Produces

The planner is a single lightweight LLM call (not an agent) that takes:
- `resolved_input` — the clarified task
- `available_agent_cards` — retrieved from the agent registry (top-k, not all)
- `mem0_context` — prior organizational memory relevant to this task
- `tool_performance_hints` — which tools have been unreliable recently (from Postgres)

And returns a typed `Plan` Pydantic object.

```python
class SubTaskPlan(BaseModel):
    id: str                          # UUID, generated client-side
    domain: str                      # "research" | "data_analyst" | "compliance" | etc
    assigned_agent: str              # agent name from AgentCard
    description: str                 # what this sub-task should accomplish
    input_payload: dict              # structured input the specialist will receive
    depends_on: list[str]            # list of subtask ids this depends on (empty = no dependency)
    estimated_tokens: int            # for cost ceiling check
    requires_write_approval: bool    # true if involves write-permission tools

class Plan(BaseModel):
    subtasks: list[SubTaskPlan]
    execution_mode: Literal["parallel", "sequential", "adaptive"]
    assumptions: list[str]           # explicit assumptions made during planning
    estimated_cost_usd: float        # sum of (estimated_tokens * model_cost) across all subtasks
    estimated_duration_sec: int
    reasoning: str                   # why the planner decomposed it this way (shown in UI)
```

---

## Agent Discovery — How the Route Node Finds Agents

The orchestrator does NOT have hardcoded agent URLs. Discovery is dynamic:

### Step 1 — Specialists register at startup

Every specialist agent registers its AgentCard with the registry on startup:

```python
# agents/research/agent_card.json
{
  "name": "research_agent",
  "version": "1.0.0",
  "description": "Performs web research, news aggregation, competitor analysis, and document summarization. Best for: market intelligence, competitor profiles, recent events, public information.",
  "capabilities": ["web_search", "news_aggregation", "pdf_parsing", "competitor_analysis"],
  "input_schema": {
    "query": "string",
    "geography": "string | null",
    "time_period": "string | null",
    "output_format": "finding | summary | raw"
  },
  "output_schema": "Finding",
  "sla": {
    "estimated_tokens_per_call": 4000,
    "max_latency_sec": 60,
    "tool_permission": "read"
  },
  "a2a_endpoint": "http://research-agent:8001",
  "health_endpoint": "http://research-agent:8001/health"
}
```

The registry stores the AgentCard in Postgres and embeds the `description` + `capabilities` into a vector index (pgvector).

### Step 2 — Route Node retrieves relevant agents

The Route Node does NOT pass all AgentCards to the planner. It retrieves only relevant ones:

```python
async def route_node(state: OrchestratorState) -> OrchestratorState:
    # 1. Extract domains from the plan (already decided by planner)
    domains_needed = [s.domain for s in state.plan.subtasks]

    # 2. Vector search: retrieve top-k AgentCards matching each domain
    # This means 100 registered agents costs the same as 5
    available_agents = await registry.search(
        queries=domains_needed,
        top_k=2  # top 2 per domain
    )

    # 3. Check health of retrieved agents
    healthy_agents = await check_agent_health(available_agents)

    # 4. For any domain with no healthy agent → mark subtask as degraded immediately
    for subtask in state.plan.subtasks:
        if subtask.domain not in [a.name for a in healthy_agents]:
            await db.update_subtask(subtask.id, status="degraded",
                                    result={"error": f"No healthy agent for domain: {subtask.domain}"})

    return state.update(available_agents=healthy_agents)
```

---

## Execution Modes

The planner outputs one of three execution modes. The Execute Node behaves differently for each.

### Parallel
All sub-tasks have no dependencies on each other. Execute Node fires them all simultaneously via `asyncio.gather`. Best for most tasks — fastest path.

```
Task: "Analyze churn increase"
  ├── subtask_1: research_agent — "find competitor changes in Q3"
  ├── subtask_2: data_analyst   — "analyze churn metrics Q3 vs Q2"
  └── subtask_3: compliance     — "check data retention policy compliance"

All 3 fire simultaneously → synthesis waits for all 3
```

### Sequential
Sub-tasks have explicit dependencies. Execute Node fires them in order, passing each output to the next.

```
Task: "Assess risk of microservices migration"
  ├── subtask_1: code_infra — "analyze current monolith architecture"
  └── subtask_2: research   — "find failure patterns for teams with THIS architecture"
                                                                      ▲
                                                    depends on subtask_1 output
```

### Adaptive
The planner signals that step N cannot be determined without seeing step N-1's output. The graph loops back to a lightweight re-planning call after each step.

```
Task: "Deep dive into Q3 performance"
  Step 1: data_analyst — broad analysis
  ↓ output reveals: revenue anomaly in EMEA segment
  Re-plan: step 2 = research_agent focused on EMEA market events (not pre-planned)
  ↓ output reveals: regulatory change in Germany
  Re-plan: step 3 = compliance_agent on German regulation (not pre-planned)
```

**Adaptive mode is powerful but harder to trace and eval.** Use it only when the planner explicitly sets `execution_mode = "adaptive"`. Most tasks are parallel.

---

## Cost Ceiling Check

Before execution begins, the estimated cost is compared to the configured ceiling:

```python
DEFAULT_COST_CEILING_USD = 0.50  # configurable per deployment

async def check_cost_ceiling(plan: Plan, state: OrchestratorState):
    if plan.estimated_cost_usd > DEFAULT_COST_CEILING_USD:
        # Emit warning, require user confirmation
        await emit_sse(state.task_id, {
            "type": "cost_warning",
            "estimated_cost_usd": plan.estimated_cost_usd,
            "ceiling_usd": DEFAULT_COST_CEILING_USD,
            "message": f"This task is estimated to cost ${plan.estimated_cost_usd:.2f}. Proceed?"
        })
        # Set interrupt — wait for user approval
        raise NodeInterrupt("cost_ceiling_exceeded")
```

---

## Plan Revision Loop

The user can reject the plan (max 2 revisions). On rejection:

1. User provides feedback: `"I don't need compliance checked, focus on data and research only"`
2. Planner re-runs with: `resolved_input + rejected_plan + rejection_feedback`
3. New plan is generated
4. `plans.version` increments to 2
5. New plan is shown to user for confirmation
6. If rejected again (revision 2): one more attempt
7. If rejected a third time: system returns `"Please simplify your task and resubmit"` — no more revisions

Both plans (original + revisions) are stored in Postgres for eval purposes — useful for measuring planner quality over time.
