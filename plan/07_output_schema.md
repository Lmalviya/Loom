# Output Schema

> These are the canonical Pydantic models for all structured outputs.
> Source of truth: `orchestrator/schemas.py`
> See also: `09_llm_calls.md` for which calls produce each model.

---

## Why Structured Output Matters

The final report is not a markdown string. It is a structured Pydantic object that:
- Gets serialized to Postgres on completion
- Is returned via the SSE `completed` event
- Is rendered by the UI into a structured report card
- Is consumed by the eval harness for scoring
- Can be compared across versions programmatically

A markdown blob cannot be evaled, compared, or programmatically checked for citation accuracy. Structured output is what makes the report measurable.

---

## Core Models

### Citation

```python
class Citation(BaseModel):
    source_agent: str           # which specialist produced this finding
    source_tool: str            # which MCP tool retrieved this data
    source_url: str | None      # URL if from web search or external doc
    source_title: str | None    # document title if from PDF/doc
    excerpt: str                # specific piece of evidence (max 200 chars)
    retrieved_at: datetime
```

---

### Finding

The output of a single specialist agent. One per domain per task.

```python
class FindingStatus(str, Enum):
    CONFIRMED  = "confirmed"   # finding is solid, no contradictions
    CONFLICTED = "conflicted"  # contradiction detected with another agent's finding
    DEGRADED   = "degraded"    # specialist was unavailable or timed out

class Finding(BaseModel):
    domain: str                     # "research" | "data_analyst" | "compliance" | etc
    agent: str                      # agent name that produced this
    claim: str                      # the core finding in one sentence
    detail: str                     # supporting detail, 2-3 sentences max
    confidence: float               # 0.0–1.0, set by specialist agent
    citations: list[Citation]       # every claim must be backed by at least one citation
    status: FindingStatus
    raw_output: dict | None         # specialist's full output, stored but not shown in UI
```

---

### Conflict

Populated when the Synthesis Node detects contradictions between findings.

```python
class ConflictSeverity(str, Enum):
    LOW    = "low"     # minor disagreement in detail, not in core claim
    MEDIUM = "medium"  # notable disagreement, synthesis made a judgment call
    HIGH   = "high"    # direct contradiction, synthesis could not resolve — HITL triggered

class Conflict(BaseModel):
    finding_a_domain: str           # domain of first conflicting finding
    finding_b_domain: str           # domain of second conflicting finding
    claim_a: str                    # the specific claim from agent A
    claim_b: str                    # the specific claim from agent B
    conflict_description: str       # plain English description of the disagreement
    severity: ConflictSeverity
    resolution: str | None          # how synthesis resolved it (if it could)
    resolution_reasoning: str | None # why synthesis trusted one over the other
    requires_human: bool            # true if HIGH severity — HITL was/will be triggered
    human_resolution: str | None    # populated after user resolves via HITL
```

---

### PlanSummary

A read-only record of the plan that was executed — not the live Plan object.

```python
class PlanSummary(BaseModel):
    plan_id: str
    version: int                    # 1 = first plan, 2+ = revised
    execution_mode: str             # parallel | sequential | adaptive
    subtasks_total: int
    subtasks_completed: int
    subtasks_degraded: int          # failed/timed out
    agents_used: list[str]
    agents_degraded: list[str]      # agents that failed or timed out
    clarification_rounds: int       # 0–3
    confirmation_revisions: int     # 0–2
    assumptions: list[str]          # explicit assumptions made during planning
```

---

### CostSummary

```python
class CostSummary(BaseModel):
    total_cost_usd: float
    cost_by_agent: dict[str, float]   # {"research_agent": 0.04, "data_analyst": 0.03, ...}
    cost_by_call_type: dict[str, float] # {"planning": 0.01, "synthesis": 0.02, "specialist": 0.07}
    total_input_tokens: int
    total_output_tokens: int
    total_latency_ms: int
    slowest_agent: str | None
```

---

### TaskReport (root object)

```python
class TaskStatus(str, Enum):
    COMPLETED = "completed"
    DEGRADED  = "degraded"   # completed with some missing domains
    FAILED    = "failed"     # unrecoverable failure

class TaskReport(BaseModel):
    # Identity
    task_id: str
    trace_id: str               # use to construct Langfuse URL: {LANGFUSE_URL}/traces/{trace_id}

    # Timestamps
    created_at: datetime
    completed_at: datetime

    # Status
    status: TaskStatus

    # Input (preserved for reference)
    original_input: str
    resolved_input: str         # after clarification rounds

    # Core output
    summary: str                # 3-5 sentence executive summary
    findings: list[Finding]     # one per domain, ordered by confidence descending
    conflicts: list[Conflict]   # empty list if no contradictions detected
    recommendation: str | None  # synthesis agent's recommendation if task warrants one

    # Quality signals
    confidence_overall: float   # weighted average of all finding.confidence values
    completeness_score: float   # domains_completed / domains_planned (0.0–1.0)
    assumptions: list[str]      # all assumptions made during planning + execution

    # Meta
    plan: PlanSummary
    cost: CostSummary

    # Error info (populated on degraded/failed)
    degraded_domains: list[str]  # domains that failed/timed out
    error_summary: str | None    # human-readable explanation of what failed and why

    @property
    def langfuse_url(self) -> str:
        return f"{settings.LANGFUSE_URL}/traces/{self.trace_id}"
```

---

## Specialist Agent Output Schema

Every specialist agent returns a `SpecialistResult` — the orchestrator wraps this into a `Finding`.

```python
class SpecialistResult(BaseModel):
    # Required
    claim: str                  # core finding in one sentence
    detail: str                 # 2-3 sentences of supporting detail
    confidence: float           # 0.0–1.0

    # Evidence
    citations: list[Citation]

    # Optional structured data (specialist-specific)
    data: dict | None           # raw data, charts, tables (stored, not shown in summary)

    # Tool performance metadata (written to tool_performance table)
    tools_used: list[dict]      # [{"tool": "web_search", "success": true, "latency_ms": 1200}]
```

---

## Eval Harness Consumption

The eval harness reads `TaskReport` objects from Postgres and scores them:

```python
# evals/harness.py
class EvalCase(BaseModel):
    task_input: str
    expected_domains: list[str]     # which domains should have been invoked
    expected_claim_keywords: list[str]  # keywords that should appear in findings
    should_have_conflicts: bool | None  # None = don't check

scorer = RAGASScorer(
    metrics=["faithfulness", "answer_relevance", "context_precision"]
)

# Groundedness: every claim in Finding.claim is backed by at least one Citation
# Completeness: len(report.findings) / len(expected_domains) >= 0.8
# Conflict handling: if should_have_conflicts and not report.conflicts → fail
```