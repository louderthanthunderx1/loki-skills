# Platform architecture + runtime (agentic-platform)

Mirror of the platform repo's own `CLAUDE.md`. Read this before touching `app/engine/core/`, `app/training/`, or anything runtime-load-bearing.

## Project overview

Enterprise agentic orchestration platform: FastAPI backend + Streamlit management UI. Users compose **Flows** (business processes) from ordered **Agents** (personas) that use **Skills** (prompt fragments) and **Tools** (OpenAI function-calling definitions). A LangGraph state machine executes flows; Langfuse v3 versions the prompts and traces every run.

Seeded demo flows: *Pre-Payment Customer Purchase* and *Insurance Claim Processing*.

## Merged app layout (ITAECD-379)

Two formerly-separate services in one FastAPI process on port 8000:

- **Engine** — original runtime: `app/`, mounted at the existing paths (`/agents`, `/flows`, `/skills`, `/tools`, `/teams`, `/chat`, `/personas`, `/mcp-servers`).
- **Training** — formerly `bpr-ai-training-system`, now at `app/training/`, mounted under `/training/...`.

Both halves share the same Postgres DB (`agentic_platform`, `public` schema) and the same Alembic environment. Training calls engine in-process via `app.training.services.runtime_adapter.RuntimeAdapter` — no HTTP self-call.

Tests split: `tests/engine/` and `tests/training/`.

## Services

Langfuse v3 lives in sibling repo `../bpr-ai-observation/` and must be up before this stack so the api can publish traces.

```bash
# Terminal 1 — Langfuse stack (start once, leave running)
cd ../bpr-ai-observation
docker compose up -d
curl http://localhost:3000/api/public/health      # verify ready

# Terminal 2 — this app
docker compose up -d --build                       # postgres, payments-mcp, api
docker compose logs -f api
docker compose down                                # stop
docker compose down --volumes                      # reset app DB volume
docker compose up -d --build api                   # rebuild API only
```

The api reaches Langfuse via `LANGFUSE_HOST` (default `http://host.docker.internal:3000`). Host-gateway alias wired in compose (`extra_hosts: ["host.docker.internal:host-gateway"]` on `api`) so cross-stack hop works on native dockerd in WSL2.

On first `api` container start, entrypoint runs `alembic upgrade head && python scripts/seed.py && uvicorn app.main:app --reload`.

Local backend (outside Docker):

```bash
uv sync
uv run uvicorn app.main:app --reload                # API on :8000 (docs at /docs)
uv run streamlit run streamlit_app/Home.py          # UI on :8501
```

## Architecture — the big picture

Read `app/engine/core/` as a unit; no single file tells the full story.

### Domain hierarchy (`app/engine/models/`)

```
Flow ──< flow_agents (step_order) >── Agent ──< agent_registry >── Skill
                                        └─────< agent_tools   >── Tool
AgentTeam ──< agent_team_members >── Agent            (alternative to Flow for team-based execution)
```

All models inherit from `Base` in `app/engine/models/base.py`. Soft-delete invariant covered below.

### Prompt assembly → LangGraph execution

Critical path for `/flows/{id}/run`, `/flows/auto/run`, `/teams/{id}/run`:

1. **Pre-load** (`run_loader.py`, `loader_helpers.py`, `agent_loader.py`) — `load_run_data(target)` single entry; `target: RunTarget(kind, target_id)` from router. Internal resolvers `_resolve_flow` / `_resolve_team` + shared helpers `load_master_block`, `build_agents_config`, `build_orchestrator_tools`, `merge_orchestrator_tools`. Auto-flow selector listing in `flow_listing.py`. Graph nodes never touch the DB.
2. **Compose graph** (`stages.py`, `graph_builder.py`) — execution shapes are lists of `Stage` objects passed to `compose()`. Two shapes: `compose([ORCHESTRATOR_LOOP])` for flow/team, `compose([SELECTOR, ORCHESTRATOR_LOOP])` for auto-flow. New shapes add a stage list, not a new builder.
3. **State** (`graph_state.py`) — `FlowState` TypedDict carries messages, pre-loaded `agents_config`, decisions log, tool results.
4. **Nodes** — three thin runners delegate to policy modules:
   - `selector_node.py` `flow_selector_node` — picks flow from user input (auto-flow only).
   - `orchestrator_node/node.py` — supervisor LLM with `route_to_agent` / `complete_flow`. Tool calls routed through `dispatch_tool_call` against handler registry in `handlers.py`. New orchestrator tool type = `@register(...)` handler.
   - `agent_node/node.py` — ReAct loop per specialist agent. 5 cross-cutting concerns (tool list, prompt, LLM temperature, result parsing, step logging) each in own policy file under `agent_node/`. New agent capability = policy file update, never `node.py`.
5. **LLM** (`agent_node/llm.py` for per-agent binding, `llm.py` for role-keyed factory) — provider env-switched (`LLM_PROVIDER=azure|bedrock`), dispatched through `get_llm(role)` on three roles (`LLMRole.{ORCHESTRATOR, AGENT, UTILITY}`). Per-role models from env per provider (`AZURE_OPENAI_DEPLOYMENT_*` / `BEDROCK_MODEL_*`); `Settings` validator fails fast at boot if Bedrock selected without required vars. Orchestrator and selector hard-code roles; `agent_node` and chat router read `agent.model_tier`. Temperature from `creativity_level` (precise/balanced/creative → 0.0/0.25/0.7), mapped in `app/engine/schemas/agent.py` via `CREATIVITY_LEVEL_TEMPERATURE`, applied per-call via `with_config` so cached LLM client reused. Temperature is **per-agent**, not global.
6. **Observability** (`langfuse.py`, `langfuse_callback.py`) — singleton Langfuse client attached as LangGraph callback. Flushed on FastAPI shutdown via lifespan handler in `app/main.py`.

Graph checkpointing uses `MemorySaver` (in-memory). No multi-turn flow persistence — flagged as gap in `docs/agent-design-review.md`.

### Langfuse prompt sync

Creating/updating Agent or Skill via API auto-pushes versioned prompt to Langfuse:

| Entity | Langfuse prompt prefix |
|--------|------------------------|
| Agent  | `agent-<slug>`         |
| Skill  | `skill-<slug>`         |

Happens inside router handler (`app/engine/routers/agent.py`, `app/engine/routers/skill.py`) — do not add separate sync step. Langfuse client env-driven via `LANGFUSE_HOST` / `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY`.

### Tools layer (post-ITAECD-385 — MCP framework)

Tools dispatch through `app/engine/tools/mcp_client.py` `MCPClientPool`, process-singleton wrapper around `langchain-mcp-adapters.MultiServerMCPClient`. Initialized in FastAPI lifespan; closed at shutdown. Tools pre-loaded into `agents_config[name].tools: dict[str, ToolConfig]` by `tool_loader.py` `load_tools_for_agent`. Runtime calls dispatch via `mcp_pool.call_tool` (orchestrator) or `mcp_pool.get_langchain_tools` (agent_node).

MCP servers managed via:
- DB tables: `mcp_servers` (registry) + `tools.mcp_server_id` FK.
- REST: `/mcp-servers/preview`, `POST /mcp-servers`, `GET /mcp-servers`, `POST /mcp-servers/{id}/rediscover`, `DELETE /mcp-servers/{id}`.
- Streamlit: `pages/10_Register_MCP_Server.py`.
- Health-check loop: 60-second probe in lifespan, updates `status` + `last_seen_at`.

Reference MCP server: `mcp_servers/payments/` — Docker container, FastMCP HTTP transport, port 8080. Implementations canned (real backends migrate in ITAECD-386). Insurance + refund tools have `mcp_server_id NULL` until ITAECD-387/388 ship.

### Two API styles coexist

- **REST CRUD** per entity (`/flows`, `/agents`, `/skills`, `/tools`, `/teams` + junctions like `/agents/{id}/skills`).
- **Chat / run**: `/chat/completions` (single-agent, OpenAI-compatible), `/flows/{id}/run` + `/flows/auto/run` (orchestrated), `/teams/{id}/run` (teams).

`GET /agents/{id}/prompt` returns assembled system prompt + tool defs — canonical introspection endpoint.

## Architecture invariants

Cross these and the runtime breaks:

- **Graph nodes pure w.r.t. DB.** New data needs pre-loaded via `run_loader.load_run_data` (extend a resolver) or helper in `loader_helpers.py`. Never import a session into a node.
- **New agent capability → policy file** under `app/engine/core/agent_node/`. Never inside `node.py`.
- **New orchestrator tool type → `@register("name")` handler** in `app/engine/core/orchestrator_node/handlers.py`. Never an `elif` inside `node.py`.
- **New execution shape → new `Stage`** in `app/engine/core/stages.py` + `compose()` call in `graph_builder.py`. Never clone a builder.
- **Error responses must be CORS-safe JSON.** Routes raise `HTTPException(...)` (handled by `ExceptionMiddleware`) or rely on catch-all middleware from `install_catch_all_middleware` in `app/main.py`. Never let exception escape past catch-all — `CORSMiddleware` cannot decorate the bare `text/plain` 500 Starlette's outer `ServerErrorMiddleware` falls back to. Call order in `app/main.py` load-bearing: catch-all first, then `CORSMiddleware`, so CORS wraps catch-all and headers attach to JSON response. See ITAECD-416.
- **Alembic only — never `init_db` / `Base.metadata.create_all`.** Schema only moves through migration.
- **Soft delete only — never hard `DELETE`.** Flip `is_deleted`, filter in every query.

## CI/CD

Three workflows under `.github/workflows/`:

- `app-validate.yml` — runs on every PR to `develop`/`main`. Steps: `uv sync --extra dev` → `uv run ruff format --check ...` → `uv run ruff check ...` → `uv run pytest -m "not live and not integration"` → `docker build`. Mirrors PR template self-check.
- `deploy-stg.yml` — `workflow_dispatch` only. Builds image, pushes to ECR `bpr-stg/ai-core-engine` (tags `sha-<sha>`, `latest`), then `kubectl set image deployment/ai-core-engine ... -n bpr-stg` against `bpr-stg-eks`.
- `deploy-prd.yml` — same as stg, against `bpr-prd/ai-core-engine` / `bpr-prd-eks` / `bpr-prd`.

Deploys gated behind manual dispatch until `bpr-ai-infrastructure` resolves the k8s manifest placeholders (`<password>`, `<rds-core-endpoint>`, `<auth-token>`) into Kubernetes Secrets. After that lands, flip both workflow triggers from `workflow_dispatch` to `push` in a 1-line follow-up PR.

Common ops:

```bash
gh workflow run "Deploy STG" --ref develop
gh workflow run "Deploy PRD" --ref main
gh run watch
gh run view <run-id> --log-failed
```

Required GitHub environment secrets (repo Settings → Environments):

- `stg` env: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ACCOUNT_ID`
- `prd` env: same three secrets

IAM principal needs ECR push on `bpr-{stg,prd}/ai-core-engine` plus EKS access entry / `aws-auth` mapping for `patch deployments` in target namespace.

## Streamlit UI (`streamlit_app/`)

Multi-page app. All pages call FastAPI through `streamlit_app/api_client.py` — add new endpoints there, not inline `requests`. Base URL from `streamlit_app/config.py`.

## Key docs

- `docs/langgraph-workflow.md` — state/nodes/routing walkthrough
- `docs/flow-run-workflow.md` — end-to-end orchestrator trace
- `docs/agent-loader-workflow.md` — prompt assembly rules
- `docs/api-examples.md` — curl / Python request samples
- `docs/agent-design-review.md` — known gaps and design rationale
