# Streaming & Human-in-the-Loop (HITL)

> See also: `01_task_lifecycle.md` for task states triggered by HITL.
> See also: `10_langgraph_nodes.md` for Confirm Node and HITL Node interrupt implementation.

---

## Core Design Principle

**Pause is a state, not a wait.**

When user input is required, the orchestrator does NOT block a thread waiting. It:
1. Sets task status to `awaiting_*` in Postgres
2. Serializes full LangGraph state to Postgres via the checkpointer
3. Emits an SSE event describing what it needs
4. Returns — the process is free

When the user responds via `POST /tasks/{id}/resume`, the graph is reconstructed from the checkpoint and continues from the exact node it paused at. No re-execution of prior nodes.

---

## SSE Event Types

All events follow this envelope:

```python
class SSEEvent(BaseModel):
    task_id: str
    trace_id: str
    type: str           # see event types below
    timestamp: datetime
    payload: dict       # event-specific data
```

### Full list of SSE event types

```
# Validation phase
clarification_needed    — validation found task vague, questions attached
scope_warning           — task too broad, cost/time estimate shown
cost_warning            — plan cost exceeds ceiling, confirmation needed

# Planning phase
status                  — generic progress update ("Analyzing your task...")
plan_created            — plan ready for user review, full plan attached
plan_revised            — revised plan after rejection, version number attached

# Execution phase
subtask_started         — a specialist agent has started a sub-task
subtask_progress        — internal progress from specialist ("Searching web...")
subtask_completed       — specialist finished, partial finding attached
subtask_degraded        — specialist failed/timed out, domain marked as degraded

# HITL triggers
human_input_required    — pause during execution, reason + question attached
conflict_detected       — synthesis found contradiction, both claims shown

# Synthesis phase
synthesizing            — synthesis has begun
completed               — full TaskReport attached
degraded                — TaskReport with partial results attached
failed                  — error summary attached
```

---

## Full Streaming Flow

```
User submits task
      │
      ▼
SSE: { type: "status", message: "Analyzing your task..." }
      │
      ▼
SSE: { type: "plan_created", plan: { subtasks: [...], reasoning: "...", estimated_cost: 0.12 } }
      │        ← UI renders plan as a card the user can inspect
      │
User reviews plan:
      │
  ┌── REJECT + feedback ──────────────────────────────────────────┐
  │                                                               │
  │   SSE: { type: "status", message: "Updating plan..." }       │
  │   SSE: { type: "plan_revised", plan: {...}, version: 2 }     │
  │   ← back to user review (max 2 revisions)                    │
  └───────────────────────────────────────────────────────────────┘
      │
  CONFIRM
      │
      ▼
SSE: { type: "subtask_started", agent: "research_agent", subtask: "Analyze competitors" }
SSE: { type: "subtask_progress", agent: "research_agent", message: "Searching web..." }
SSE: { type: "subtask_progress", agent: "research_agent", message: "Found 12 sources, summarizing..." }
SSE: { type: "subtask_completed", agent: "research_agent", finding: { claim: "...", confidence: 0.87 } }
      │      ← UI fills in the research section of the report card
      │
SSE: { type: "subtask_started", agent: "data_analyst", subtask: "Check revenue trends" }
      ...
      │
SSE: { type: "synthesizing", message: "Combining findings from 3 agents..." }
      │
SSE: { type: "conflict_detected", conflict: { claim_a: "...", agent_a: "research", claim_b: "...", agent_b: "data_analyst" } }
      │      ← optional: only if synthesis finds a contradiction
      │
SSE: { type: "completed", report: TaskReport }
```

---

## Plan Confirmation — Design Details

The plan is shown to the user as a structured card, not a wall of text. Each sub-task is a row:

```
┌─────────────────────────────────────────────────────────┐
│  Plan for: "Analyze why churn increased 18%"            │
│  Est. cost: $0.12  |  Est. time: ~45 sec                │
│  Mode: parallel                                         │
├─────────────────────────────────────────────────────────┤
│  1. research_agent   — Analyze competitor changes Q3    │
│  2. data_analyst     — Compare churn metrics Q3 vs Q2   │
│  3. compliance       — Check data retention compliance  │
├─────────────────────────────────────────────────────────┤
│  Assumptions:                                           │
│  • Time period interpreted as Q3 2025                   │
│  • Segment interpreted as B2B based on prior context    │
├─────────────────────────────────────────────────────────┤
│  [Confirm]    [Reject + Feedback]                       │
└─────────────────────────────────────────────────────────┘
```

The `reasoning` field from the Plan object is shown as a collapsible section — the user can see why the planner made each routing decision.

---

## HITL During Execution — Three Triggers

### Trigger 1 — Write-permission tool about to execute

```python
# In Execute Node, before any A2A call involving write tools:
if subtask.requires_write_approval:
    await emit_sse(task_id, {
        "type": "human_input_required",
        "reason": "write_action",
        "question": f"Agent is about to: {subtask.write_action_description}. Approve?",
        "options": ["approve", "reject"]
    })
    raise NodeInterrupt("write_approval_needed")
```

### Trigger 2 — Specialist returns low confidence

Threshold: `confidence < 0.60` (configurable)

```python
if finding.confidence < CONFIDENCE_THRESHOLD:
    await emit_sse(task_id, {
        "type": "human_input_required",
        "reason": "low_confidence",
        "question": f"Finding from {agent} has low confidence ({finding.confidence:.0%}). Additional context?",
        "current_finding": finding.dict(),
        "options": ["provide_context", "accept_as_is", "skip_domain"]
    })
    raise NodeInterrupt("low_confidence")
```

### Trigger 3 — Synthesis detects high-severity conflict

```python
if conflict.severity == "high":
    await emit_sse(task_id, {
        "type": "human_input_required",
        "reason": "conflict",
        "question": "Two agents disagree. Which do you trust more?",
        "conflict": conflict.dict(),
        "options": ["trust_research", "trust_data_analyst", "both_uncertain"]
    })
    raise NodeInterrupt("conflict_resolution")
```

---

## Resume API

```
POST /tasks/{task_id}/resume
Content-Type: application/json

{
  "response_type": "plan_confirm" | "plan_reject" | "clarification" | "hitl_approve" | "hitl_reject" | "hitl_context",
  "payload": {
    // for plan_reject: { "feedback": "Remove compliance, focus on data only" }
    // for clarification: { "answers": ["Q3 2025", "B2B segment"] }
    // for hitl_approve: { "approved": true }
    // for hitl_context: { "additional_context": "..." }
    // for conflict: { "trusted_agent": "research_agent" }
  }
}
```

The Resume endpoint:
1. Validates task is in an `awaiting_*` state
2. Loads the LangGraph checkpoint from Postgres
3. Injects the user's response into the graph state
4. Resumes execution from the interrupted node
5. Re-opens the SSE stream to the user

---

## Partial Results — Progressive UI Fill

Specialists complete at different times. The UI does not wait for all agents before showing anything. Each `subtask_completed` SSE event carries a partial `Finding` object that the UI renders immediately into the report card.

Example: research completes in 20s, data_analyst in 40s, compliance in 35s.
- At 20s: research section of report card is filled in
- At 35s: compliance section fills in
- At 40s: data_analyst section fills in
- At ~42s: synthesis completes, full report rendered

This makes a 40-second task feel responsive from second 20.

---

## Timeout on Plan Confirmation

If the user does not confirm or reject the plan within 10 minutes:
- Task status → `awaiting_confirmation` (already set)
- No action taken — the plan persists in Postgres
- When the user returns (even days later), the plan is still there via `GET /tasks/{id}`
- User can confirm, reject, or cancel the task
- The SSE connection will have dropped — user re-connects by polling `GET /tasks/{id}/events?since={timestamp}`
