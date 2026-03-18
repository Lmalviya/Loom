# Memory Architecture

> See also: `03_planning.md` for how Mem0 context is injected into the planner.
> See also: `06_output_schema.md` for what gets written to memory after task completion.

---

## Memory Is Not Conversation History

These are two completely separate concerns:

| | Conversation History | Memory |
|---|---|---|
| What it is | Raw `[{role, content}]` message list | Extracted entities, facts, relationships |
| Where it lives | LangGraph state dict (session) + Postgres (archive) | Mem0 graph |
| Lifetime | Session → archived | Persistent across all sessions |
| Used for | Continuing current task | Enriching future tasks |
| Who manages it | LangGraph state | Mem0 client in orchestrator |

After each task completes, `mem0.add(messages, user_id=...)` is called. Mem0 extracts what matters and stores it. The raw messages are archived to Postgres and discarded from active memory.

---

## Five Memory Types in Loom

### 1. Entity + Relation Memory — Mem0 Graph (primary)

**What it stores:** Named entities (companies, products, people, markets) and the relationships between them, along with facts learned about each.

**Example:**
```
Task: "Analyze why churn increased for Acme Corp"

After completion, Mem0 stores:
  Entity: Acme Corp
  Entity: B2B segment
  Fact: "churn increased 18% in Q3 2025 for Acme Corp B2B segment"
  Fact: "root cause identified as competitor pricing change by CompetitorX"
  Relation: Acme Corp → competes_with → CompetitorX
  Relation: churn_increase → caused_by → CompetitorX pricing change
```

**Next task benefit:** When user asks anything about Acme Corp or churn, Mem0 injects this context automatically.

**Implementation:**
```python
# After task completion
async def save_task_memory(task_id: str, report: TaskReport, messages: list):
    await mem0_client.add(
        messages=messages,
        user_id=task.user_id,
        metadata={
            "task_id": task_id,
            "task_type": report.task_type,
            "completed_at": report.completed_at.isoformat()
        }
    )
```

**Retrieval at planning time:**
```python
async def get_relevant_memory(resolved_input: str, user_id: str) -> list[str]:
    results = await mem0_client.search(
        query=resolved_input,
        user_id=user_id,
        limit=10
    )
    return [r["memory"] for r in results]
```

---

### 2. Episodic Memory — Postgres + Mem0

**What it stores:** Past tasks and their outcomes — what was asked, what was found, when.

**Why both Postgres and Mem0:**
- Postgres is the ground truth: full task record, structured, queryable by time/type/outcome
- Mem0 is the retrieval layer: semantic search over past task summaries

**Postgres table:**
```sql
CREATE TABLE episodic_memory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    user_id         TEXT,
    task_summary    TEXT,       -- 2-3 sentence summary of what was asked and found
    domains_touched TEXT[],
    key_findings    JSONB,      -- top 3 findings from the TaskReport
    outcome         TEXT,       -- completed | degraded | failed
    created_at      TIMESTAMPTZ DEFAULT now()
);
```

**Written after every completed task:**
```python
# Summarize the task in 2-3 sentences (lightweight LLM call)
summary = await summarize_task(report)
await db.save_episodic_memory(task_id, user_id, summary, report)
# Also add to Mem0 for semantic retrieval
await mem0_client.add([{"role": "assistant", "content": summary}], user_id=user_id)
```

---

### 3. Personalization Memory — Mem0 (namespaced)

**What it stores:** User/org preferences inferred from task history and explicit feedback.

**Examples:**
- "User prefers EMEA focus for market research"
- "User's company is in B2B SaaS, always assume this context"
- "User rejects compliance sub-tasks — they have a separate legal team"
- "User prefers concise reports, not detailed"

**How it's written:**
Personalization is NOT auto-extracted. A dedicated `extract_preferences` LLM call runs after each task and looks for signals in:
- What domains the user rejected from plans
- What additional context the user provided during clarification
- Explicit preferences stated in task descriptions

```python
async def extract_and_store_preferences(task: Task, report: TaskReport):
    prefs = await extract_preferences(
        original_input=task.original_input,
        clarification_exchanges=task.clarification_history,
        rejected_plan_feedback=task.plan_rejection_feedback,
        hitl_responses=task.hitl_history
    )
    if prefs:
        await mem0_client.add(
            messages=[{"role": "user", "content": f"User preference: {p}"} for p in prefs],
            user_id=task.user_id
        )
```

---

### 4. Document / Knowledge Base Memory — pgvector on Postgres

**What it stores:** Reference documents — runbooks, architecture docs, policy files, domain knowledge files.

**This is NOT Mem0.** Document memory is for static reference material. Mem0 is for dynamic learned memory.

```sql
CREATE TABLE documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    content     TEXT NOT NULL,
    embedding   vector(1536),       -- pgvector
    doc_type    TEXT,               -- runbook | policy | architecture | domain_knowledge
    source      TEXT,               -- file path, URL, or manual
    created_at  TIMESTAMPTZ DEFAULT now(),
    updated_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);
```

**Used by specialists** (not the orchestrator directly) when they need to ground their reasoning in internal documentation. The orchestrator can also query it during planning to understand org-specific context.

---

### 5. Tool Performance Memory — Postgres

**What it stores:** Historical performance of each MCP tool — success rate, average latency, common failure types.

**Why it matters:** The planner uses this to avoid routing sub-tasks to tools that have been unreliable for similar inputs. This is the "closed loop" for tool quality.

```sql
-- Already defined in 01_task_lifecycle.md: tool_performance table
-- Query used by planner:
SELECT
    tool_name,
    COUNT(*) FILTER (WHERE success) * 1.0 / COUNT(*) AS success_rate,
    AVG(latency_ms) AS avg_latency_ms,
    MODE() WITHIN GROUP (ORDER BY error_type) AS most_common_error
FROM tool_performance
WHERE agent = 'research_agent'
AND created_at > now() - interval '7 days'
GROUP BY tool_name;
```

**Written after every tool call** inside each specialist agent — not the orchestrator. The specialist reports tool performance back via the A2A result payload's metadata field.

---

## Memory Write Sequence (After Task Completion)

```python
async def post_task_memory_pipeline(task: Task, report: TaskReport, messages: list):
    # Run in parallel — all writes are independent
    await asyncio.gather(
        save_task_memory(task, messages),           # Mem0: entities + facts
        save_episodic_memory(task, report),         # Postgres + Mem0: task summary
        extract_and_store_preferences(task, report) # Mem0: inferred preferences
        # Tool performance is written by specialists during execution — not here
    )
```

---

## Memory Read Sequence (At Planning Time)

```python
async def load_planning_context(resolved_input: str, user_id: str) -> PlanningContext:
    # Run in parallel
    entity_memory, episodic_memory, doc_context, tool_hints = await asyncio.gather(
        mem0_client.search(resolved_input, user_id=user_id, limit=10),
        db.search_episodic_memory(resolved_input, user_id=user_id, limit=5),
        db.search_documents(resolved_input, limit=5),
        db.get_tool_performance_hints()
    )
    return PlanningContext(
        entity_memory=entity_memory,
        episodic_memory=episodic_memory,
        doc_context=doc_context,
        tool_hints=tool_hints
    )
```

---

## Memory Namespacing for Multi-Tenant

All Mem0 calls include `user_id`. In the current single-user version, `user_id` is set to the organization name or a constant. When multi-tenancy is added:
- Each organization gets its own `user_id` namespace in Mem0
- Postgres tables already have `user_id` columns
- Document memory gets a `org_id` foreign key
- No architectural changes needed — just populate the user_id fields
