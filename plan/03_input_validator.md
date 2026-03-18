# Input Validation & Clarification

> See also: `09_llm_calls.md` for the TaskValidator and ClarificationGenerator call details.
> See also: `10_langgraph_nodes.md` for the Validate Node and Clarify Node implementation.

---

## Why Validation Exists

A vague task submitted to the planner causes one of three bad outcomes:
1. The planner hallucinates a plan based on wrong assumptions
2. The planner generates an over-broad plan that invokes every specialist (expensive, slow)
3. The plan is technically valid but wrong for what the user actually needed

Validation is cheaper than fixing a bad plan. A single classification call costs ~$0.001. A failed 10-agent plan costs $2+ and 5 minutes.

---

## Two Types of Bad Input

### Type 1 — Too vague
No clear goal, no domain signal, no measurable outcome.

Examples:
- `"analyze our business"`
- `"help me understand things"`
- `"look into competitors"`

### Type 2 — Too broad (unbounded scope)
Has a goal but scope will explode into 20+ sub-tasks.

Examples:
- `"do a full company audit"`
- `"review everything about our product launch"`
- `"analyze all our data"`

These are technically actionable but will run for 10+ minutes and cost $5+. The system warns and asks the user to scope down before proceeding.

---

## Validation Output Schema

```python
class ValidationResult(BaseModel):
    clarity_score: float            # 0.0–1.0
    is_actionable: bool
    task_type: Literal[
        "clear",                    # proceed to planning
        "vague",                    # needs clarification
        "unbounded",                # needs scoping
        "write_heavy",              # contains write operations, flag for security
    ]
    missing_context: list[str]      # e.g. ["time period?", "which product?", "which market?"]
    estimated_domains: list[str]    # e.g. ["research", "data_analyst"] — used for clarification
    estimated_complexity: Literal["low", "medium", "high", "unbounded"]
    suggested_clarifications: list[str]  # human-readable questions
```

### Thresholds
- `clarity_score >= 0.75` AND `task_type != "unbounded"` → proceed to planning
- `clarity_score < 0.75` OR `task_type == "vague"` → trigger clarification
- `task_type == "unbounded"` → trigger scope warning (different flow from clarification)

---

## Clarification Flow

```
Task submitted
      │
      ▼
[Validate Node] calls TaskValidator LLM call
      │
      ├── clarity_score >= 0.75 and not unbounded ──► [Plan Node]
      │
      ├── clarity_score < 0.75 (vague) ──────────────────────────────────┐
      │                                                                    ▼
      │                                                    [Clarify Node]
      │                                                    calls ClarificationGenerator
      │                                                    emits SSE: clarification_needed
      │                                                    task status: awaiting_clarification
      │                                                    ◄── user answers (max 3 rounds)
      │                                                    resolved_input = original + answers
      │                                                    ──► back to [Validate Node]
      │
      └── task_type == unbounded ──────────────────────────────────────────┐
                                                                           ▼
                                                           emits SSE: scope_warning
                                                           task status: awaiting_clarification
                                                           user can: narrow scope OR confirm
                                                           if confirm → proceed with warning
                                                           in assumptions: ["Scope is broad — estimated X min, $Y cost"]
```

### Max clarification rounds: 3

After 3 rounds, the system proceeds with its best interpretation and makes assumptions explicit in the plan. It never forces a 4th clarification. Assumption example:
```
"Assuming Q3 2025, B2B segment, German market — based on context clues in your task.
Correct these via plan rejection if wrong."
```

---

## Clarification Questions Are AgentCard-Aware

Questions are not generic. The ClarificationGenerator receives the `estimated_domains` from the ValidationResult and the AgentCards for those domains. It generates questions based on what each specialist actually needs.

Example — if `estimated_domains = ["data_analyst", "research"]`:

Data analyst AgentCard requires: `time_period`, `data_source`, `metric_name`
Research AgentCard requires: `geography`, `industry`, `competitor_names`

Generated questions:
- "What time period are you analyzing? (e.g. Q3 2025, last 90 days)"
- "Which data source should be used? (e.g. our Postgres DB, public data)"
- "Which specific competitors should be included?"
- "Are you focused on a specific geography?"

This feels intelligent — not like a generic form.

---

## Resolved Input Construction

After clarification, the original task and all answers are combined into a `resolved_input` that the planner receives:

```
Original: "Analyze why churn increased"

Clarification round 1:
  Q: "What time period?"
  A: "Q3 2025"
  Q: "Which segment?"
  A: "B2B enterprise customers"

resolved_input:
"Analyze why churn increased in Q3 2025 for B2B enterprise customers.
[User clarified: time period = Q3 2025, segment = B2B enterprise]"
```

The `resolved_input` is stored on the Task row alongside `original_input`. Both are visible in the Langfuse trace and in the final TaskReport.

---

## Scope Warning Flow (Unbounded Tasks)

Unlike clarification (which asks questions), a scope warning shows the user what they're committing to:

SSE event emitted:
```json
{
  "type": "scope_warning",
  "estimated_agents": 6,
  "estimated_subtasks": 14,
  "estimated_duration_min": 12,
  "estimated_cost_usd": 3.20,
  "message": "This task will invoke all available agents and may take ~12 minutes and cost ~$3.20. You can narrow the scope or proceed.",
  "narrowing_suggestions": [
    "Focus on a specific department or product line",
    "Limit to Q3 2025 only",
    "Focus on the top 3 competitors only"
  ]
}
```

User options:
- Narrow scope → task goes back to input with suggestions pre-filled
- Proceed → task continues with a `scope_warning` flag on the plan

---

## Write Intent Detection

The WriteDetector LLM call runs alongside validation (not sequentially — parallel). It flags whether the task contains explicit write intent.

Write intent keywords trigger elevated security mode:
- "save", "store", "update", "create", "send", "commit", "write", "post", "notify", "deploy"

If write intent is detected:
- All `write` and `execute` permission tools require human approval even at high confidence
- The plan confirmation step explicitly lists write actions that will be performed
- The user must confirm write actions separately from the plan confirmation

See `08_security.md` for full tool permission model.