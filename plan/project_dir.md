# Loom — Project Directory Structure

```
loom/
│
├── README.md
├── .env.example
├── .gitignore
├── docker-compose.yml               # boots entire system: all agents + postgres + redis + langfuse
├── pyproject.toml                   # workspace-level dependencies
├── Makefile                         # shortcuts: make dev, make eval, make lint, make migrate
│
│
├── core/                            # shared across orchestrator + all agents
│   ├── __init__.py
│   ├── config.py                    # settings: env vars, retry config, cost ceilings, thresholds
│   ├── base_agent.py                # base class: tracing + cost tracking + logging wired in
│   ├── sanitizer.py                 # prompt injection defense for external content
│   ├── boundaries.py                # hard security stops (never-allowed actions list)
│   │
│   ├── a2a/                         # A2A protocol implementation
│   │   ├── __init__.py
│   │   ├── server.py                # base A2A server — all specialist agents extend this
│   │   ├── client.py                # typed A2A client used by orchestrator execute_node
│   │   └── models.py                # A2A message, task, AgentCard Pydantic schemas
│   │
│   ├── mcp/                         # MCP tool wrappers (used by specialist agents)
│   │   ├── __init__.py
│   │   ├── manifest.py              # MCPToolManifest: name, permission, side_effects, max_calls
│   │   ├── web_search.py            # Tavily / SerpAPI wrapper
│   │   ├── web_scraper.py           # HTML fetch + clean
│   │   ├── pdf_parser.py            # PDF text extraction
│   │   ├── code_exec.py             # sandboxed code execution
│   │   ├── database.py              # Postgres read-only query tool
│   │   ├── github.py                # GitHub API (read: diffs, PRs, commits)
│   │   └── slack.py                 # Slack MCP (write-permission)
│   │
│   ├── memory/                      # memory layer used by orchestrator
│   │   ├── __init__.py
│   │   ├── mem0_client.py           # Mem0 graph client wrapper
│   │   ├── episodic.py              # episodic memory: read/write to Postgres
│   │   ├── document_store.py        # pgvector document retrieval
│   │   └── tool_performance.py      # tool performance read/write
│   │
│   ├── tracing.py                   # Langfuse span propagation, @track_cost decorator
│   ├── cost_tracker.py              # token cost calculation, write to cost_tracking table
│   ├── db.py                        # async Postgres client (asyncpg), connection pool
│   ├── redis_client.py              # Redis client, tool call counter, job scheduler
│   └── logger.py                    # structlog JSON logger, per-request context
│
│
├── orchestrator/                    # orchestrator agent (LangGraph)
│   ├── __init__.py
│   ├── agent.py                     # LangGraph graph definition: nodes + edges + router fns
│   ├── state.py                     # OrchestratorState TypedDict
│   ├── llm_calls.py                 # all lightweight LLM calls (see 09_llm_calls.md)
│   ├── nodes/
│   │   ├── __init__.py
│   │   ├── validate_node.py         # TaskValidator + WriteIntentDetector (parallel)
│   │   ├── clarify_node.py          # ClarificationGenerator + interrupt
│   │   ├── plan_node.py             # TaskPlanner + cost ceiling check + persist plan
│   │   ├── confirm_node.py          # interrupt for user plan confirmation
│   │   ├── route_node.py            # agent health check + AgentCard matching
│   │   ├── execute_node.py          # A2A calls, parallel/sequential/adaptive dispatch
│   │   ├── synthesize_node.py       # ConflictDetector (pairwise) + AssumptionExtractor
│   │   ├── output_node.py           # finalize TaskReport, persist, emit SSE completed
│   │   └── memory_node.py           # post-task: Mem0 + episodic + preferences (background)
│   ├── registry.py                  # AgentCard registry: register, search (pgvector), health
│   ├── schemas.py                   # all Pydantic models (see 06_output_schema.md)
│   ├── health_check.py              # background job: stale task detection + recovery
│   ├── api.py                       # FastAPI router: POST /tasks, POST /tasks/{id}/resume,
│   │                                #                 GET /tasks/{id}, GET /tasks/{id}/events
│   └── Dockerfile
│
│
├── agents/                          # specialist agents (each is an independent A2A service)
│   │
│   ├── research/                    # web research, news, competitor analysis
│   │   ├── __init__.py
│   │   ├── agent.py                 # LangGraph graph (multi-step search-summarize-validate)
│   │   ├── tools.py                 # MCP tools: web_search, web_scraper, pdf_parser
│   │   ├── agent_card.json          # A2A AgentCard declaration
│   │   ├── schemas.py               # input/output schemas specific to this agent
│   │   ├── api.py                   # A2A FastAPI server (extends core/a2a/server.py)
│   │   ├── evals/
│   │   │   ├── test_cases.jsonl     # 20+ labeled input/output pairs
│   │   │   ├── run_evals.py         # RAGAS + LLM-as-judge scoring
│   │   │   └── results/             # eval results per version: v1.json, v2.json
│   │   ├── Dockerfile
│   │   └── README.md
│   │
│   ├── data_analyst/                # SQL gen, data profiling, statistical analysis
│   │   ├── __init__.py
│   │   ├── agent.py                 # raw Anthropic SDK (tool use loop)
│   │   ├── tools.py                 # MCP tools: database (read-only), csv_parser
│   │   ├── agent_card.json
│   │   ├── schemas.py
│   │   ├── api.py
│   │   ├── evals/
│   │   │   ├── test_cases.jsonl
│   │   │   ├── run_evals.py
│   │   │   └── results/
│   │   ├── Dockerfile
│   │   └── README.md
│   │
│   ├── code_infra/                  # code analysis, git diff, infra metrics
│   │   ├── __init__.py
│   │   ├── agent.py                 # raw Anthropic SDK
│   │   ├── tools.py                 # MCP tools: github, code_exec (sandboxed)
│   │   ├── agent_card.json
│   │   ├── schemas.py
│   │   ├── api.py
│   │   ├── evals/
│   │   │   ├── test_cases.jsonl
│   │   │   ├── run_evals.py
│   │   │   └── results/
│   │   ├── Dockerfile
│   │   └── README.md
│   │
│   ├── risk_compliance/             # regulatory lookup, risk scoring, compliance gaps
│   │   ├── __init__.py
│   │   ├── agent.py                 # DSPy — prompt optimized against labeled eval set
│   │   ├── optimizer.py             # DSPy BootstrapFewShot optimization pipeline
│   │   ├── tools.py                 # MCP tools: web_search (legal sources), pdf_parser
│   │   ├── agent_card.json
│   │   ├── schemas.py
│   │   ├── api.py
│   │   ├── evals/
│   │   │   ├── test_cases.jsonl     # 25+ labeled compliance assessments (ground truth)
│   │   │   ├── run_evals.py
│   │   │   ├── optimize.py          # run DSPy optimization → save optimized prompt
│   │   │   └── results/
│   │   ├── Dockerfile
│   │   └── README.md
│   │
│   └── _template/                   # copy this to add a new specialist agent
│       ├── agent.py
│       ├── tools.py
│       ├── agent_card.json
│       ├── schemas.py
│       ├── api.py
│       ├── evals/
│       │   └── test_cases.jsonl
│       ├── Dockerfile
│       └── README.md
│
│
├── evals/                           # cross-agent evaluation framework
│   ├── __init__.py
│   ├── harness.py                   # AgentEvalHarness base class
│   ├── scorers.py                   # RAGASScorer, LLMJudgeScorer, custom scorers
│   ├── regression.py                # version comparison + regression detection
│   ├── e2e/                         # end-to-end task eval (full orchestrator → report)
│   │   ├── test_cases.jsonl         # complex multi-domain tasks with expected structure
│   │   └── run_e2e.py
│   └── reports/                     # generated eval reports (gitignored, regenerated)
│
│
├── ui/                              # FastAPI app + minimal HTML/JS frontend
│   ├── main.py                      # mounts orchestrator router + static files
│   ├── static/
│   │   ├── index.html               # task submission + live SSE rendering
│   │   ├── report.html              # TaskReport structured view
│   │   ├── dashboard.html           # cost + latency + eval scores dashboard
│   │   └── js/
│   │       ├── sse.js               # SSE client, progressive report card rendering
│   │       └── dashboard.js         # cost charts, agent performance graphs
│   └── templates/                   # Jinja2 templates if needed
│
│
├── infra/
│   ├── docker-compose.yml           # symlinked from root
│   ├── docker-compose.dev.yml       # dev overrides: hot reload, debug ports
│   ├── prometheus.yml               # scrape configs for all agent /metrics endpoints
│   ├── grafana/
│   │   └── dashboards/
│   │       ├── loom_overview.json   # task throughput, cost, latency
│   │       └── agent_health.json    # per-agent success rate, p95 latency
│   └── k8s/                         # future: Kubernetes manifests (not for v1)
│       └── .gitkeep
│
│
├── migrations/                      # Alembic DB migrations
│   ├── env.py
│   ├── alembic.ini
│   └── versions/
│       └── 001_initial_schema.py    # tasks, plans, subtasks, task_events, cost_tracking, etc
│
│
├── scripts/
│   ├── run_evals.py                 # run all agent evals, print report
│   ├── benchmark.py                 # end-to-end task benchmark suite
│   ├── seed_registry.py             # register all AgentCards with the orchestrator registry
│   └── check_health.py              # ping all agent /health endpoints
│
│
└── docs/
    └── orchestrator/                # all 11 design docs (already written)
        ├── 00_overview.md
        ├── 01_task_lifecycle.md
        ├── 02_input_validation.md
        ├── 03_planning.md
        ├── 04_streaming_hitl.md
        ├── 05_memory.md
        ├── 06_output_schema.md
        ├── 07_failure_handling.md
        ├── 08_security.md
        ├── 09_llm_calls.md
        └── 10_langgraph_nodes.md
```

---

## Service Map (Docker Compose)

Each service runs on its own port in local dev:

| Service | Port | What it is |
|---|---|---|
| `ui` | 8000 | FastAPI app + static UI (mounts orchestrator router) |
| `orchestrator` | 8001 | Orchestrator agent (LangGraph) |
| `agent-research` | 8002 | Research specialist (A2A server) |
| `agent-data-analyst` | 8003 | Data analyst specialist (A2A server) |
| `agent-code-infra` | 8004 | Code & infra specialist (A2A server) |
| `agent-risk-compliance` | 8005 | Risk & compliance specialist (A2A server) |
| `postgres` | 5432 | Postgres (tasks, plans, memory, cost tracking) |
| `redis` | 6379 | Redis (queue, checkpoints, tool call counters) |
| `langfuse` | 3000 | Langfuse (traces, cost per hop) |

---

## Key Conventions

**Every specialist agent follows the same internal structure:**
`agent_card.json` → `api.py` (A2A server) → `agent.py` (logic) → `tools.py` (MCP) → `schemas.py` (I/O) → `evals/`

**Every specialist exposes three endpoints:**
- `POST /tasks` — A2A task submission (streaming SSE response)
- `GET /health` — health check (polled by orchestrator route_node)
- `GET /metrics` — Prometheus metrics

**Adding a new specialist = copy `agents/_template/`, fill in 5 files, register AgentCard.**
Zero changes to orchestrator required.

**All Pydantic models live in `schemas.py` within their service.**
`core/a2a/models.py` only contains A2A protocol schemas (AgentCard, A2ATask, A2AResult).
Loom domain schemas (Finding, TaskReport, etc.) live in `orchestrator/schemas.py`.