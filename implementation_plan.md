# Nexus — Cross-Framework Agent Mesh for Complex Knowledge Work

> **An open-source, protocol-native multi-agent system that accepts any complex multi-domain task, dynamically routes it to specialist agents via A2A, synthesizes results with conflict resolution and confidence scoring, and produces explainable structured reports — with full distributed tracing, cost attribution, and an evaluation harness.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![A2A Protocol](https://img.shields.io/badge/A2A-Protocol-green.svg)](https://github.com/a2aproject/A2A)
[![MCP](https://img.shields.io/badge/MCP-Enabled-purple.svg)](https://modelcontextprotocol.io)

---

## The Problem

Every organization faces tasks that cross domain boundaries — a question that starts as a business problem, requires data analysis, research, engineering assessment, and risk evaluation, and ends in a structured decision or report.

**Today, this takes:**
- Multiple people across teams
- Multiple disconnected tools
- Hours to days of coordination
- No single source of truth for the reasoning chain

**The root cause isn't capability — it's architecture.** No single AI agent has the right specialization and tools for all domains. Existing multi-agent frameworks hardcode which agents talk to which, put all agents in one process, and skip the hard parts: distributed failure handling, cross-boundary tracing, conflict resolution, and measurable output quality.

**Nexus solves this.** It is a production-grade, protocol-native agent mesh that:

1. Accepts a complex task in natural language
2. Understands which domains it touches using an LLM-powered planner
3. Dynamically routes sub-tasks to specialist agents via the A2A protocol — no hardcoding
4. Runs specialist agents in parallel, each with its own MCP tools
5. Resolves contradictions between specialist outputs with confidence scoring
6. Produces a structured, cited, explainable final report
7. Traces every hop, attributes every token cost, and measures output quality with RAGAS evals

---

## Example Tasks Nexus Can Handle

```
"Analyze why our churn rate increased 18% last quarter. Check if it correlates
with recent product changes, support ticket spikes, pricing changes, or
competitor moves. Give me a structured root cause report with confidence scores."
```

```
"Before we launch in the German market: summarize regulatory requirements,
analyze our top 3 competitors, estimate market size, flag any IP conflicts,
and give a go/no-go recommendation."
```

```
"Our API latency spiked 3x since yesterday's deployment. Identify root cause
across: recent code changes, infra metrics, DB query patterns, and third-party
dependency status. Prioritize by likely impact."
```

```
"We're migrating our monolith to microservices. Assess the risk: review our
architecture docs, check industry failure patterns, estimate engineering effort,
and flag compliance implications."
```

These tasks share a structure: **no single agent or tool handles them, but a mesh of specialists can.**

---

## Architecture

### Overview

```
User Task
    │
    ▼
┌─────────────────────────────────────┐
│         Orchestrator Agent          │  ← LangGraph · Task planner + synthesizer
│   - Decomposes task into sub-tasks  │
│   - Discovers agents via AgentCards │
│   - Routes via A2A protocol         │
│   - Owns task lifecycle state       │
│   - Mem0 cross-session memory       │
└──────────┬──────────────────────────┘
           │  A2A (JSON-RPC 2.0 over HTTP)
    ┌──────┴──────────────────────────────────────────────┐
    │              Specialist Agent Mesh                   │
    │                                                      │
    │  ┌────────────────┐   ┌────────────────┐            │
    │  │  Data Analyst  │   │    Research    │            │
    │  │   Raw SDK      │   │   LangGraph    │            │
    │  │  MCP: Postgres │   │  MCP: Web,PDF  │            │
    │  │  BigQuery, CSV │   │  Scraper, News │            │
    │  └────────────────┘   └────────────────┘            │
    │                                                      │
    │  ┌────────────────┐   ┌────────────────┐            │
    │  │ Code & Infra   │   │  Risk &        │            │
    │  │   Raw SDK      │   │  Compliance    │            │
    │  │  MCP: GitHub   │   │    DSPy        │            │
    │  │  Code Exec,    │   │  MCP: Legal DB │            │
    │  │  Metrics API   │   │  Policy Docs   │            │
    │  └────────────────┘   └────────────────┘            │
    │                                                      │
    │  ┌────────────────┐                                  │
    │  │   Synthesis    │  ← Conflict resolution +         │
    │  │   LangGraph    │    confidence scoring            │
    │  └────────────────┘                                  │
    └──────────────────────────────────────────────────────┘
           │
           ▼
    Structured Report
    + Confidence Scores
    + Source Citations
    + Full Trace (Langfuse)
    + Cost Attribution
```

### Why Each Framework Was Chosen Differently

This is intentional, not incidental. A production mesh must work across frameworks because real organizations build agents on different stacks.

| Agent | Framework | Reason |
|---|---|---|
| Orchestrator | LangGraph | Stateful graph — task lifecycle management, conditional routing, parallel fan-out |
| Data Analyst | Raw Anthropic SDK | Full control over tool use loops, no framework overhead for deterministic SQL tasks |
| Research | LangGraph | Multi-step search-summarize-validate cycles benefit from graph state |
| Code & Infra | Raw Anthropic SDK | Tight control over code execution sandboxing |
| Risk & Compliance | DSPy | Prompt optimization against labeled compliance eval set — accuracy measurably improves |
| Synthesis | LangGraph | Structured multi-input reasoning with explicit conflict detection node |

---

## The Hard Engineering Problems Solved

### 1. Dynamic Routing via Agent Discovery

The orchestrator does **not** have hardcoded agent URLs. At startup, each specialist registers itself with an AgentCard describing its capabilities, input/output schema, and SLA. The orchestrator's planner reads the task, generates a domain decomposition, and matches sub-tasks to agents by querying the registry at runtime.

This means: add a new specialist agent → it becomes available to the mesh without touching the orchestrator.

### 2. Distributed Tracing Across A2A Boundaries

Each task is assigned a `trace_id` at entry. This ID is propagated through A2A message headers to every specialist. Langfuse receives spans from all agents and assembles a single waterfall trace showing:
- Task decomposition time
- Each A2A call with latency
- Every tool call within each agent
- Token counts and cost per agent
- Final synthesis time

One URL in Langfuse → full picture of what happened across 5 services.

### 3. Conflict Resolution in Synthesis

When specialist outputs contradict each other — research agent says competitor X is growing, data agent shows declining revenue — the synthesis agent does not silently pick one. It:
- Flags the contradiction explicitly
- Assigns confidence scores to each claim based on source reliability
- Requests clarification sub-tasks if confidence delta exceeds a threshold
- Surfaces disagreements in the final report with attribution

### 4. Partial Failure Handling with Graceful Degradation

If a specialist agent is unavailable or times out:
- The orchestrator marks that domain as `DEGRADED`
- The synthesis agent produces the report with an explicit `"compliance check unavailable — timed out after 30s"` note
- The task completes, not fails
- The degradation is logged and alerted via the observability layer

Silent failures that produce hallucinated answers are treated as bugs, not acceptable behavior.

### 5. Persistent Cross-Task Organizational Memory

Nexus uses Mem0 (graph variant) on the orchestrator. After each completed task:
- Key findings are summarized and stored as memory nodes
- Entities (competitors, products, risks) are linked in the memory graph
- Future tasks on similar topics are enriched with prior context automatically

The system gets smarter about an organization's domain over time — without retraining.

### 6. Eval-Driven Improvement with DSPy

The Risk & Compliance agent uses DSPy for its internal reasoning chain. This means:
- 25 labeled historical compliance assessments are the eval set
- DSPy `BootstrapFewShot` optimizes the reasoning prompt against RAGAS faithfulness and answer relevance scores
- Prompt versions are stored, scored, and compared
- The eval dashboard shows accuracy improvement over versions

This is the closed loop: build → evaluate → optimize → measure improvement.

---

## Observability Stack

| Layer | Tool | What It Tracks |
|---|---|---|
| Trace | Langfuse | Full waterfall per task across all agents, prompt versions, latency breakdown |
| Metrics | Prometheus + Grafana | Request rate, agent latency p50/p95/p99, queue depth, error rate |
| Cost | Custom (Postgres) | Token cost per agent, per task, per domain — queryable via API |
| Evals | RAGAS + LLM-as-judge | Groundedness, completeness, conflict resolution quality, citation accuracy |
| Logs | structlog (JSON) | Structured per-event logs with trace_id, agent_id, task_id |

---

## Agent Evaluation Framework

Every agent is evaluated independently before contributing to the mesh.

### Metrics per Agent

| Agent | Primary Metrics |
|---|---|
| Data Analyst | SQL correctness, query execution success rate, result accuracy on labeled datasets |
| Research | Citation accuracy, hallucination rate (RAGAS faithfulness), source diversity |
| Code & Infra | Finding precision/recall on labeled CVE and performance issues |
| Risk & Compliance | Assessment accuracy on labeled historical cases, false positive rate |
| Synthesis | Contradiction detection rate, confidence calibration, user preference score |

### Eval Harness

```python
# Every agent has a run_evals.py
from nexus.evals import AgentEvalHarness, RAGASScorer, LLMJudge

harness = AgentEvalHarness(
    agent=ResearchAgent(),
    test_cases="evals/research/test_cases.jsonl",  # 25+ labeled cases
    scorers=[
        RAGASScorer(metrics=["faithfulness", "answer_relevance"]),
        LLMJudge(criteria="citation_accuracy"),
    ]
)

results = harness.run()
results.save("evals/research/results_v2.json")
results.compare_to("evals/research/results_v1.json")  # regression detection
```

---

## Repository Structure

```
nexus/
├── orchestrator/              # Orchestrator agent (LangGraph)
│   ├── agent.py               # Main LangGraph graph
│   ├── planner.py             # Task decomposition + domain detection
│   ├── router.py              # AgentCard registry + dynamic routing
│   ├── memory.py              # Mem0 integration
│   ├── synthesizer.py         # Conflict resolution + confidence scoring
│   ├── api.py                 # FastAPI router (SSE streaming)
│   └── schemas.py             # Pydantic task/result models
│
├── agents/
│   ├── data_analyst/          # Raw SDK agent
│   │   ├── agent.py
│   │   ├── tools.py           # MCP: Postgres, BigQuery, CSV
│   │   ├── agent_card.json    # A2A AgentCard declaration
│   │   ├── evals/
│   │   └── Dockerfile
│   ├── research/              # LangGraph agent
│   ├── code_infra/            # Raw SDK agent
│   ├── risk_compliance/       # DSPy agent
│   └── synthesis/             # LangGraph agent
│
├── core/
│   ├── a2a/                   # A2A client + server base classes
│   │   ├── server.py          # Base A2A server (all agents extend this)
│   │   ├── client.py          # Typed A2A client for orchestrator
│   │   └── models.py          # A2A message/task/AgentCard schemas
│   ├── mcp/                   # MCP tool wrappers
│   │   ├── web_search.py
│   │   ├── code_exec.py
│   │   ├── database.py
│   │   └── github.py
│   ├── tracing.py             # Langfuse span propagation across A2A
│   ├── cost_tracker.py        # Token cost calculation + storage
│   └── base_agent.py          # Base class: tracing + cost + logging wired in
│
├── evals/
│   ├── harness.py             # AgentEvalHarness base class
│   ├── scorers.py             # RAGAS + LLM-as-judge scorers
│   └── regression.py          # Version comparison + regression detection
│
├── ui/                        # FastAPI + minimal HTML/JS
│   ├── main.py                # Mounts all agent routers
│   ├── static/                # Task input UI, trace viewer, cost dashboard
│   └── templates/
│
├── infra/
│   ├── docker-compose.yml     # One command: boots all agents + Postgres + Redis + Langfuse
│   ├── prometheus.yml
│   └── grafana/
│
├── docs/
│   ├── architecture.md        # Deep dive: A2A message flow, state machine
│   ├── adding_an_agent.md     # How to add a new specialist to the mesh
│   ├── evaluation_guide.md    # How to write and run evals
│   └── deployment.md          # Production deployment guide
│
└── scripts/
    ├── run_evals.py            # Run all agent evals, generate report
    └── benchmark.py            # End-to-end task benchmark suite
```

---

## Getting Started

### Prerequisites

- Python 3.11+
- Docker + Docker Compose
- API keys: Anthropic (required), optional: Tavily (web search), GitHub token

### Quickstart

```bash
git clone https://github.com/yourusername/nexus
cd nexus

# Copy env template
cp .env.example .env
# Add your ANTHROPIC_API_KEY (minimum required)

# Boot the entire mesh
docker compose up

# Open the UI
open http://localhost:8000

# Submit your first task
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"task": "Analyze the top 3 risks of migrating a Python 2 monolith to microservices in 2026."}'
```

### Running Evals

```bash
# Run evals for all agents
python scripts/run_evals.py --all

# Run evals for a specific agent
python scripts/run_evals.py --agent research

# Compare two versions
python scripts/run_evals.py --agent risk_compliance --compare v1 v2
```

---

## Technical Decisions & Tradeoffs

Every non-obvious architectural decision is documented in `docs/architecture.md`. Key decisions:

**Why A2A over direct HTTP calls?**
A2A provides typed task lifecycle (submitted → working → completed/failed), structured streaming via SSE, and AgentCard-based discovery. Direct HTTP calls have none of this — they're fire-and-forget with no standard for partial results or task state.

**Why not put all agents in one process?**
Process isolation means a specialist agent crashing doesn't take down the mesh. It also means specialists can be scaled independently — if research tasks spike, add research agent replicas without touching anything else.

**Why Mem0 graph variant over vector-only memory?**
Graph memory stores entity relationships, not just text chunks. When the system has seen "Competitor X" across 10 tasks, it builds a connected knowledge graph of what was learned — not 10 isolated embeddings. Retrieval is relationship-aware.

**Why separate Synthesis agent from Orchestrator?**
The orchestrator handles routing and task lifecycle. The synthesis agent handles reasoning over N inputs. Separating them means synthesis logic can be tested, evaluated, and improved independently — without touching the routing layer.

**Why DSPy specifically for Risk & Compliance?**
Compliance assessment has a ground truth: historical cases with known outcomes. DSPy can optimize the reasoning prompt against this eval set automatically. Other agents handle more open-ended tasks where DSPy's optimization target is less clear.

---

## Roadmap

- [ ] Agent authentication via A2A's `securitySchemes`
- [ ] Horizontal scaling: Redis-backed task queue for orchestrator
- [ ] Additional specialists: Financial modeling agent, Legal document agent
- [ ] Streaming partial results from specialists to UI as they complete
- [ ] Multi-tenant support: per-organization Mem0 namespacing
- [ ] MCP server for Nexus itself — expose the mesh as an MCP tool for Claude Desktop

---

## Contributing

Contributions welcome. The most impactful areas:

1. **New specialist agents** — follow `docs/adding_an_agent.md`, open a PR
2. **Eval test cases** — labeled test cases for any agent improve the harness for everyone
3. **MCP tool wrappers** — new data source integrations in `core/mcp/`
4. **Bug reports with traces** — include Langfuse trace URL when reporting issues

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

---

## Why This Project Exists

Most multi-agent demos solve one of two fake problems:
- "Look, multiple agents can chat with each other"
- "Here is a notebook where Agent A calls Agent B"

Nexus is built around a real problem: **complex knowledge work that crosses domain boundaries and currently requires multiple people, days of time, and disconnected tools.** The architecture follows from the problem — A2A because agents must be independent services, multiple frameworks because real organizations don't standardize on one, DSPy because one domain has a clear optimization target, Mem0 because organizations accumulate knowledge over time.

Every design decision has a reason. Every component is independently testable. Every output is measurable.

---

## License

MIT — see [LICENSE](LICENSE)

---

*Built with the A2A Protocol (Linux Foundation), MCP (Anthropic), LangGraph (LangChain), DSPy (Stanford NLP), Mem0, Langfuse, and RAGAS.*
