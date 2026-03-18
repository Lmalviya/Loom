# Failure Handling & Recovery

> See also: `01_task_lifecycle.md` for the full state machine and TaskEvent table.
> See also: `10_langgraph_nodes.md` for which node handles each failure type.

---

## Core Rule

**Every failure writes to the TaskEvent table before any recovery action.**
No silent failures. No phantom states. If something goes wrong, it is logged with:
- Which node failed
- What the error was
- How many retries have been attempted
- What recovery action is being taken

---

## Failure Point 1 — Planning LLM Call Fails

**Cause:** Network timeout, rate limit hit, malformed structured output returned.

**Response:**
```python
# In Plan Node
MAX_PLANNING_RETRIES = 3
RETRY_BACKOFF = [1, 3, 9]  # seconds (exponential)

for attempt in range(MAX_PLANNING_RETRIES):
    try:
        plan = await plan_task(...)
        break
    except (TimeoutError, RateLimitError) as e:
        await db.log_event(task_id, "retry_attempt", node="plan_node",
                           payload={"attempt": attempt + 1, "error": str(e)})
        if attempt < MAX_PLANNING_RETRIES - 1:
            await asyncio.sleep(RETRY_BACKOFF[attempt])
        else:
            await db.update_task_status(task_id, "failed")
            await db.log_event(task_id, "error", node="plan_node",
                               payload={"error": "planning_failed_max_retries"})
            await emit_sse(task_id, {"type": "failed",
                                     "error_summary": "Planning failed after 3 attempts. Please retry."})
            return  # exit graph
    except ValidationError as e:
        # LLM returned malformed output — retry with stricter prompt
        await db.log_event(task_id, "retry_attempt", node="plan_node",
                           payload={"attempt": attempt + 1, "error": "malformed_output"})
```

---

## Failure Point 2 — Graph State Corruption (Checkpoint Write Fails)

**Cause:** Postgres briefly unavailable during a node transition, disk full.

**Key principle:** Retry the checkpoint write, not the node logic. Never re-execute a node that already succeeded just because persistence failed.

```python
# LangGraph Postgres checkpointer handles this internally
# Configure with retry logic:
checkpointer = PostgresSaver(
    conn_string=settings.POSTGRES_URL,
    retry_on_error=True,
    max_retries=3,
    retry_backoff=2.0
)
```

If checkpoint write ultimately fails after retries → task moves to `stale` (caught by health check).

---

## Failure Point 3 — Specialist Agent Fails or Times Out

**Cause:** Specialist service is down, A2A call times out, specialist returns an error.

**Response — per sub-task:**
```python
# In Execute Node
SPECIALIST_TIMEOUT_SEC = 60
MAX_SPECIALIST_RETRIES = 2

async def call_specialist(subtask: SubTaskPlan, agent: AgentCard) -> SpecialistResult | None:
    for attempt in range(MAX_SPECIALIST_RETRIES):
        try:
            result = await a2a_client.send_task(
                endpoint=agent.a2a_endpoint,
                payload=subtask.input_payload,
                trace_id=state.trace_id,
                timeout=SPECIALIST_TIMEOUT_SEC
            )
            await db.update_subtask(subtask.id, status="completed", result=result)
            await emit_sse(task_id, {"type": "subtask_completed", "agent": agent.name})
            return result

        except (TimeoutError, A2AError) as e:
            await db.log_event(task_id, "retry_attempt", subtask_id=subtask.id,
                               payload={"attempt": attempt + 1, "error": str(e)})
            if attempt < MAX_SPECIALIST_RETRIES - 1:
                await asyncio.sleep(5)

    # All retries exhausted — mark as degraded, do NOT fail the whole task
    await db.update_subtask(subtask.id, status="degraded")
    await db.log_event(task_id, "subtask_failed", subtask_id=subtask.id)
    await emit_sse(task_id, {
        "type": "subtask_degraded",
        "agent": agent.name,
        "domain": subtask.domain,
        "message": f"{agent.name} unavailable — this domain will be marked as incomplete"
    })
    return None  # Synthesis handles missing findings
```

**Rule:** A single specialist failing never fails the whole task. The report is returned as `degraded` with an explicit note on which domains are missing.

---

## Failure Point 4 — Synthesis LLM Call Fails

**Cause:** All specialists completed but the synthesis step crashes.

**Why this is recoverable without re-running specialists:**
All specialist results are persisted to `subtasks.result` in Postgres on completion. Synthesis reads from Postgres, not from in-memory state. So synthesis can be retried in isolation.

```python
MAX_SYNTHESIS_RETRIES = 3

for attempt in range(MAX_SYNTHESIS_RETRIES):
    try:
        report = await synthesize(
            findings=await db.get_completed_findings(task_id),
            conflicts=[],
            plan=state.plan
        )
        break
    except Exception as e:
        await db.log_event(task_id, "retry_attempt", node="synthesize_node",
                           payload={"attempt": attempt + 1, "error": str(e)})
        await emit_sse(task_id, {"type": "status", "message": f"Retrying synthesis... (attempt {attempt+2})"})
        if attempt == MAX_SYNTHESIS_RETRIES - 1:
            # Return a degraded report with raw findings but no synthesis
            report = build_degraded_report(state, error="synthesis_failed")
```

---

## Failure Point 5 — Task Stuck in Intermediate State (Process Crash)

**Cause:** The orchestrator process dies while a task is in `planning`, `executing`, or `synthesizing`.

**Detection:** Background health check job.

```python
# Background job — runs every 60 seconds via Redis scheduler
async def health_check_stale_tasks():
    stuck_tasks = await db.query("""
        SELECT id, status, updated_at
        FROM tasks
        WHERE status IN ('planning', 'executing', 'synthesizing')
        AND updated_at < now() - interval '5 minutes'
    """)

    for task in stuck_tasks:
        await db.update_task_status(task.id, 'stale')
        await db.log_event(task.id, 'stale_detected',
                           payload={'was_status': task.status,
                                    'stuck_for_sec': (now() - task.updated_at).seconds})

        # Attempt resume from last LangGraph checkpoint
        checkpoint = await checkpointer.get(task.id)
        if checkpoint:
            logger.info(f"Resuming stale task {task.id} from checkpoint")
            await resume_task_from_checkpoint(task.id, checkpoint)
            await db.update_task_status(task.id, checkpoint.last_status)
        else:
            # No checkpoint found — cannot recover automatically
            await notify_user(task.id, {
                "type": "failed",
                "error_summary": "Task was interrupted and could not be automatically recovered. Please resubmit."
            })
            await db.update_task_status(task.id, 'failed')
```

---

## Catastrophic Failure — Degraded Report

When all retries are exhausted and the task cannot complete normally, a degraded report is always returned — never a 500 error.

```python
def build_degraded_report(state: OrchestratorState, error: str) -> TaskReport:
    # Collect whatever findings did complete
    completed_findings = [
        Finding(**s.result) for s in state.plan.subtasks
        if s.status == "completed" and s.result
    ]
    degraded_domains = [
        s.domain for s in state.plan.subtasks
        if s.status in ("degraded", "failed", "pending")
    ]

    return TaskReport(
        task_id=state.task_id,
        trace_id=state.trace_id,
        status=TaskStatus.DEGRADED if completed_findings else TaskStatus.FAILED,
        summary="Task completed partially. Some domains could not be analyzed.",
        findings=completed_findings,
        conflicts=[],
        confidence_overall=mean([f.confidence for f in completed_findings]) if completed_findings else 0.0,
        completeness_score=len(completed_findings) / len(state.plan.subtasks),
        degraded_domains=degraded_domains,
        error_summary=f"The following domains failed: {', '.join(degraded_domains)}. Reason: {error}",
        plan=build_plan_summary(state),
        cost=build_cost_summary(state)
    )
```

---

## Retry Configuration (Centralized)

All retry settings live in `core/config.py` — not scattered across nodes:

```python
class RetryConfig:
    planning_max_retries: int = 3
    planning_backoff_sec: list[int] = [1, 3, 9]
    specialist_max_retries: int = 2
    specialist_timeout_sec: int = 60
    synthesis_max_retries: int = 3
    checkpoint_max_retries: int = 3
    stale_threshold_min: int = 5
    health_check_interval_sec: int = 60
```
