# Orchestrator — Design Overview

> This document is the entry point for all orchestrator design decisions.
> Every section links to a detailed doc. Read this first, then go deep on specific areas.

---

## What the Orchestrator Is

The orchestrator is the brain of Loom. It is a LangGraph-based agent that:

1. Accepts a complex multi-domain task from the user
2. Validates and clarifies the task before doing anything
3. Decomposes the task into typed sub-tasks via a lightweight LLM planning call
4. Presents the plan to the user for confirmation before executing
5. Dynamically discovers specialist agents via AgentCard registry
6. Routes sub-tasks to the right specialist agents via the A2A protocol
7. Streams progressive updates to the user via SSE throughout execution
8. Resolves conflicts between specialist outputs with confidence scoring
9. Synthesizes a structured final report (TaskReport)
10. Persists every state transition to Postgres for recovery and observability

**What it is NOT:**
- It is not a chatbot
- It is not a hardcoded pipeline
- It does not directly call tools — specialists do that via MCP
- It does not make business decisions — it routes and synthesizes

---

## File Map

| File | What it covers |
|---|---|
| `00_overview.md` | This file — entry point, architecture summary |
| `01_task_lifecycle.md` | Task state machine, Postgres schema, recovery |
| `02_input_validation.md` | Validation, clarification flow, vagueness detection |
| `03_planning.md` | Task decomposition, agent discovery, execution modes |
| `04_streaming_hitl.md` | SSE event design, plan confirmation, pause/resume |
| `05_memory.md` | Mem0 graph, episodic memory, document memory, tool memory |
| `06_output_schema.md` | TaskReport, Finding, Citation, Conflict Pydantic models |
| `07_failure_handling.md` | Retry logic, stale task recovery, graceful degradation |
| `08_security.md` | Tool permissions, prompt injection, cost ceiling, boundaries |
| `09_llm_calls.md` | Every lightweight LLM call — purpose, input, output schema |
| `10_langgraph_nodes.md` | Every LangGraph node — responsibility, inputs, outputs |

---

## Architecture in One Diagram

```
User
 │  POST /tasks + SSE connection
 ▼
┌─────────────────────────────────────────────────────────┐
│                   LangGraph Orchestrator                │
│                                                         │
│  [Validate] → [Plan] → [Confirm*] → [Route] → [Execute]│
│                  ↑                      ↓          ↓   │
│             [Clarify*]            AgentRegistry  [HITL*]│
│                                                    ↓   │
│                              [Synthesize] → [Output]   │
│                                                         │
│  * = interrupt point (user input required)              │
│                                                         │
│  llm_calls.py: Validator, Clarifier, Planner,           │
│                WriteDetector, ConflictDetector,         │
│                AssumptionExtractor                      │
└─────────────────────────────────────────────────────────┘
         │                    │                  │
         ▼                    ▼                  ▼
     Postgres              Mem0 Graph         Langfuse
   (task state,          (entities,          (traces,
    plan, events)         org memory)         cost)
                                │
                         Redis (queue,
                          checkpoints)
         │
         ▼  A2A (JSON-RPC over HTTP + SSE)
┌─────────────────────┐
│  Specialist Agent   │
│  A2A Server         │
│  → Agent Logic      │
│  → MCP Tools        │
│  AgentCard JSON     │
└─────────────────────┘
```

---

## Key Design Principles

**1. LLM decides, business logic executes.**
The LLM is responsible for reasoning: decomposing tasks, detecting conflicts, synthesizing outputs.
Python/LangGraph is responsible for mechanics: persisting state, making A2A calls, managing retries.
The LLM never directly calls an agent, writes to a DB, or manages a retry. It outputs structured Pydantic objects. The graph acts on them.

**2. Every failure writes to the TaskEvent table.**
No silent failures. Every exception, retry, timeout, and degradation is logged with timestamp, node name, and error message. This feeds the observability dashboard and tells you exactly where to improve the system.

**3. Pause is a state, not a wait.**
Human-in-the-loop does not block threads. When user input is needed, the task status is set to `awaiting_*`, LangGraph serializes its full state to Postgres via the checkpointer, and the process returns. When the user responds, the graph resumes from the exact node it paused at.

**4. Structured output everywhere.**
No LLM call returns free text that propagates downstream. Every call returns a typed Pydantic model. This prevents hallucinations from silently corrupting the execution flow and makes every call independently testable.

**5. Agent discovery is dynamic.**
The orchestrator does not have hardcoded agent URLs or names. Specialists register AgentCards at startup. The Route Node queries the vector index at runtime. Adding a new specialist requires zero changes to the orchestrator.

---

## Technology Choices

| Component | Technology | Reason |
|---|---|---|
| Graph execution | LangGraph | Stateful graph, native interrupt/resume, Postgres checkpointer |
| LLM calls | Anthropic SDK (direct) | Full control over structured output, no framework overhead for simple calls |
| Agent protocol | A2A (JSON-RPC 2.0 + SSE) | Typed task lifecycle, streaming, AgentCard discovery — the emerging standard |
| Tool protocol | MCP | Specialists use MCP for tools — decoupled from orchestrator |
| Memory | Mem0 (graph variant) | Entity-relation memory, not just vector similarity |
| Persistence | Postgres | Task state, plan, events, cost, episodic memory log |
| Checkpointing | LangGraph Postgres checkpointer | Graph state serialized at every node for crash recovery |
| Queue | Redis | Background jobs, stale task detection |
| Tracing | Langfuse | Cross-agent trace with shared trace_id |
| Streaming | FastAPI SSE | Progressive updates to user during long-running tasks |