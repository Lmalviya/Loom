# Lightweight LLM Calls (llm_calls.py)

> All calls in this file are plain async Python functions — not agents, not classes.
> They have: one input, one structured Pydantic output, no tools, no memory, no state.
> They are independently testable without spinning up the full graph.
> Source: `orchestrator/llm_calls.py`

---

## Design Rules (apply to every call in this file)

1. **Structured output only** — every call uses `response_format` or tool-use forcing. No free text.
2. **Retry on malformed output** — if the LLM returns output that fails Pydantic validation, retry once with a stricter prompt. Max 2 retries total.
3. **Independently testable** — each function takes plain Python types in, returns a Pydantic model out. No mocking of agents or graph state required.
4. **Cost tracked** — every call logs to `cost_tracking` table via the `@track_cost` decorator.
5. **Traced** — every call is a Langfuse child span of the parent task trace.

---

## Decorator Applied to All Calls

```python
# core/tracing.py
def track_cost(call_type: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            with langfuse.span(name=func.__name__, trace_id=kwargs.get("trace_id")):
                result = await func(*args, **kwargs)
                # cost tracked inside the LLM client wrapper
            return result
        return wrapper
    return decorator
```

---

## 1. TaskValidator

**Purpose:** Determine if the task is clear enough to plan, or needs clarification/scoping.

**When called:** First node, immediately after task submission.

**Runs in parallel with:** WriteIntentDetector (both run before any other node).

```python
@track_cost("validation")
async def validate_task(
    task: str,
    agent_cards: list[AgentCard],   # used to infer what context is missing
    trace_id: str
) -> ValidationResult:
    ...

class ValidationResult(BaseModel):
    clarity_score: float
    is_actionable: bool
    task_type: Literal["clear", "vague", "unbounded", "write_heavy"]
    missing_context: list[str]
    estimated_domains: list[str]
    estimated_complexity: Literal["low", "medium", "high", "unbounded"]
    suggested_clarifications: list[str]
```

**System prompt key instruction:**
- Return `task_type = "vague"` if the task cannot be decomposed into concrete sub-tasks without guessing
- Return `task_type = "unbounded"` if executing this task would require more than 8 sub-tasks
- `estimated_domains` must only include domains covered by the provided AgentCards
- `missing_context` must be specific to what the AgentCards need, not generic

---

## 2. ClarificationGenerator

**Purpose:** Generate targeted questions based on what the task is missing AND what each relevant specialist needs.

**When called:** When ValidationResult.task_type == "vague".

```python
@track_cost("clarification")
async def generate_clarifications(
    task: str,
    validation: ValidationResult,
    agent_cards: list[AgentCard],   # only cards for estimated_domains
    trace_id: str
) -> ClarificationQuestions:
    ...

class ClarificationQuestions(BaseModel):
    questions: list[str]            # max 4 questions, ordered by importance
    per_domain_needs: dict[str, list[str]]  # {"data_analyst": ["time_period", "metric"]}
```

**System prompt key instruction:**
- Questions must be specific to what the provided AgentCards declare as required inputs
- Maximum 4 questions — prioritize by impact on plan quality
- Questions must be answerable in a single sentence
- Do NOT ask generic questions like "can you be more specific"

---

## 3. TaskPlanner

**Purpose:** Decompose the resolved task into typed sub-tasks, assign agents, set execution mode.

**When called:** After validation passes (or clarification completes). This is the most expensive call in the orchestrator — model choice matters.

```python
@track_cost("planning")
async def plan_task(
    resolved_input: str,
    agent_cards: list[AgentCard],       # top-k retrieved from registry
    mem0_context: list[str],            # relevant memory from prior tasks
    tool_performance_hints: dict,       # unreliable tools to avoid
    trace_id: str
) -> Plan:
    ...

class SubTaskPlan(BaseModel):
    id: str
    domain: str
    assigned_agent: str
    description: str
    input_payload: dict
    depends_on: list[str]
    estimated_tokens: int
    requires_write_approval: bool

class Plan(BaseModel):
    subtasks: list[SubTaskPlan]
    execution_mode: Literal["parallel", "sequential", "adaptive"]
    assumptions: list[str]
    estimated_cost_usd: float
    estimated_duration_sec: int
    reasoning: str                  # shown to user in plan confirmation UI
```

**System prompt key instructions:**
- Only assign agents that exist in the provided agent_cards list — no hallucated agent names
- Mark `execution_mode = "adaptive"` ONLY if step N's output genuinely changes what step N+1 should do
- `input_payload` for each sub-task must conform to the agent's `input_schema` from its AgentCard
- `assumptions` must be explicit — list every interpretation made about the task
- Avoid `requires_write_approval = true` unless the agent's tools explicitly include write-permission tools
- Do NOT create sub-tasks for domains not covered by the provided agent_cards

---

## 4. WriteIntentDetector

**Purpose:** Determine if the task contains explicit intent to write, modify, send, or execute anything.

**When called:** In parallel with TaskValidator (both run as first step).

```python
@track_cost("validation")
async def detect_write_intent(
    task: str,
    trace_id: str
) -> WriteIntentResult:
    ...

class WriteIntentResult(BaseModel):
    has_write_intent: bool
    write_signals: list[str]    # specific words/phrases that indicate write intent
    # e.g. ["'send to team' implies Slack notification", "'save the report' implies DB write"]
    confidence: float
```

**System prompt key instructions:**
- Write intent keywords: save, store, update, create, send, commit, write, post, notify, deploy, modify, delete, remove, schedule
- `has_write_intent = true` even if the write is implied, not explicit
- `confidence` reflects how certain the classification is — low confidence → conservative, assume write intent

---

## 5. ConflictDetector

**Purpose:** Determine if two findings from different specialist agents contradict each other.

**When called:** In the Synthesize Node, pairwise across all findings (O(n²) but n is small, max 8).

```python
@track_cost("synthesis")
async def detect_conflict(
    finding_a: Finding,
    finding_b: Finding,
    trace_id: str
) -> ConflictResult:
    ...

class ConflictResult(BaseModel):
    has_conflict: bool
    severity: Literal["none", "low", "medium", "high"]
    conflict_description: str | None
    # "Research says competitor X is growing. Data analyst shows X revenue down 18%."
    resolution: str | None
    # "Trust data_analyst — quantitative data from primary source takes precedence over research summary"
    resolution_reasoning: str | None
    requires_human: bool            # True if severity == "high"
```

**System prompt key instructions:**
- A conflict requires the two claims to be factually incompatible — not just different perspectives
- Prefer resolution via source reliability: primary data (DB query) > secondary research > general web search
- Set `severity = "high"` only if the conflict meaningfully changes the overall recommendation
- `requires_human = true` ONLY for high severity — do not escalate low/medium conflicts to the user

---

## 6. AssumptionExtractor

**Purpose:** Extract implicit assumptions made during planning and execution to surface in the final report.

**When called:** After Plan is generated AND after Synthesis completes (called twice).

```python
@track_cost("synthesis")
async def extract_assumptions(
    context: str,   # the plan's reasoning field + all clarification exchanges
    trace_id: str
) -> list[str]:
    ...

# Returns plain list of assumption strings:
# [
#   "Time period interpreted as Q3 2025 based on 'last quarter' in task",
#   "B2B segment assumed based on prior organizational memory",
#   "German market assumed based on 'DACH region' mentioned in clarification"
# ]
```

**System prompt key instructions:**
- Only extract assumptions that a user might disagree with or want to correct
- Do not extract obvious interpretations (e.g. "interpreted 'analyze' as meaning 'analyze'")
- Maximum 5 assumptions — prioritize the ones with highest impact on the output

---

## 7. TaskSummarizer (post-completion)

**Purpose:** Generate a 2-3 sentence summary of the task and its key findings for episodic memory storage.

**When called:** After task completion, as part of the memory write pipeline.

```python
@track_cost("memory")
async def summarize_task(
    original_input: str,
    report: TaskReport,
    trace_id: str
) -> str:
    ...

# Returns a plain string like:
# "Analyzed Q3 2025 churn increase for B2B segment. Root cause identified as
#  CompetitorX pricing change and product gap in enterprise tier. Confidence: 0.82.
#  Completed 2025-10-15, 3 domains analyzed."
```

---

## 8. PreferenceExtractor (post-completion)

**Purpose:** Infer user/org preferences from task history, clarification exchanges, and plan rejection feedback.

**When called:** After task completion, in parallel with TaskSummarizer.

```python
@track_cost("memory")
async def extract_preferences(
    original_input: str,
    clarification_history: list[dict],
    plan_rejection_feedback: list[str],
    hitl_history: list[dict],
    trace_id: str
) -> list[str]:
    ...

# Returns list of preference strings for Mem0 storage:
# [
#   "User always wants EMEA focus for market research tasks",
#   "User's organization is B2B SaaS — assume this context",
#   "User does not want compliance sub-tasks — has separate legal team"
# ]
```

**System prompt key instructions:**
- Only extract preferences that are reusable across future tasks
- Do not extract one-off specifics (e.g. "user wanted Q3 data" — that's task-specific, not a preference)
- Preferences must be stated as facts, not inferences: "User prefers X" not "User might prefer X"
- Maximum 3 preferences per task — quality over quantity