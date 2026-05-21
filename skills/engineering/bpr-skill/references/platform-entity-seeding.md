# Adding a platform entity (Agent, Skill, Tool, MCP server)

The platform has *two* entry points for new entities:

1. **REST + Streamlit UI** at runtime — for entities the user creates as part of normal operation.
2. **`scripts/seed.py` + Alembic** — for the "ships in the box" demo + training entities that the system needs immediately after a fresh `compose up`.

This file covers the seeded path. The runtime path is the same data flow, just through the routers.

## The five tables and how they link

```
agents ──────────────< agent_registry (agent_id, skill_id) >─────────── skills
   │                                                                       │
   └────< agent_tools (agent_id, tool_id) >────── tools                     │
                                                    │  mcp_server_id        │
                                                    ▼                    skill_tools (ITAECD-406)
                                              mcp_servers                    │
                                                                             ▼
                                                                          tools (re-link)
```

Soft-delete columns (`is_deleted`) on every row. Junctions are seeded **by name** (`AGENT_TOOLS`, `SKILL_TOOLS` dicts) and resolved to IDs at seed time.

## scripts/seed.py — the source-of-truth

Read [scripts/seed.py](../../../scripts/seed.py) before adding anything. It's idempotent (upsert by `id`), runs on every `api` container start via the entrypoint:

```
alembic upgrade head && python scripts/seed.py && uvicorn app.main:app --reload
```

Section landmarks (line numbers shift, search for the symbol):

| Section | Symbol |
|---------|--------|
| Agents (list of dicts) | `AGENTS = [` |
| Skills (list of dicts) | `SKILLS = [` |
| MCP servers | `MCP_SERVERS = [` |
| Tools | `TOOLS = [` |
| Agent ↔ Tool junction | `AGENT_TOOLS: dict[str, list[str]]` |
| Skill ↔ Tool junction | `SKILL_TOOLS: dict[str, list[str]]` |
| Insert/upsert loops | bottom of file, ~`def main()` |

The canonical recent example is the **shipment-intake** agent (ITAECD-378) — agent + handler skill + 7 tool junctions, reusing existing `so_get_*` and `send_notification` tools rather than re-defining them. Search the file for `shipment-intake` to see the shape.

## Recipe: add a new Agent

1. **Pick a stable UUID.** Prefix `20000000-0000-0000-0000-...` is the agent block. Use a fresh tail (`...00050`, `...00051`, …). Stable IDs make seed re-runs idempotent.
2. **Append a dict to `AGENTS`:**

   ```python
   {
       "id": uuid.UUID("20000000-0000-0000-0000-0000000000NN"),
       "name": "kebab-case-name",
       "description": "What the agent does + when it triggers.",
       "model_tier": "agent",          # or "orchestrator" / "utility"
       "creativity_level": "precise",  # precise (0.0) | balanced (0.25) | creative (0.7)
       "persona": "You are …",         # the system prompt body — multi-line, plain text
   }
   ```

3. **If the agent has skills**, add the (agent_name, skill_name) pairs to the `AGENT_REGISTRY` block.
4. **If the agent has tools**, add `"<agent-name>": [...tool_names]` to the `AGENT_TOOLS` dict. The tool list must be a **superset** of the union of its skills' tools.
5. **Alembic** — agents are not schema; no migration needed *unless* you added a new column.
6. **Ruff** — `uv run ruff format scripts/seed.py && uv run ruff check --fix scripts/seed.py`.
7. **Smoke-test the seed locally:**

   ```bash
   docker compose down --volumes
   docker compose up -d --build
   docker compose logs -f api          # expect "Seed complete" before "Uvicorn running"
   ```

8. **Langfuse sync happens automatically** on agent insert/upsert via the router — but the seed path inserts directly. If the agent must show up in Langfuse from cold start, ensure the seed dispatches the same publish call the router does (search seed for `langfuse.create_prompt`). Don't add a separate sync step elsewhere.

## Recipe: add a new Skill

1. **Pick UUID** under the `20000000-0000-0000-0000-...` skill subrange used in `SKILLS`.
2. **Append to `SKILLS`:**

   ```python
   {
       "id": uuid.UUID("20000000-0000-0000-0000-0000000000NN"),
       "name": "kebab-case-name",
       "category": "payments",   # or "claims" / "shipment" / etc.
       "description": "Short one-liner.",
       "skill": "# Header\n\nThe full markdown body the agent will see…",
   }
   ```

3. **Wire to agents** in `AGENT_REGISTRY` block.
4. **If the skill calls tools**, add `"<skill-name>": [...tool_names]` to `SKILL_TOOLS`. Reasoning-only skills (no tool calls) stay out of `SKILL_TOOLS`.
5. **The agent owning this skill** must have *at least* the skill's tools in its `AGENT_TOOLS` list (superset rule).

## Recipe: add a new Tool

Tools today are MCP-backed. The choice is *which MCP server* serves the tool, not "do I need an MCP server".

1. **Identify the MCP server** that exposes (or will expose) the tool.
   - If reusing `payments-mcp` (`mcp_server_id = 40000000-0000-0000-0000-000000000001`), the tool already exists in the MCP server's code — you're just adding the DB row.
   - If a new MCP server, see the next recipe.
2. **Append to `TOOLS`:**

   ```python
   {
       "id": uuid.UUID("<fresh-uuid4>"),
       "name": "kebab_or_snake_tool_name",
       "description": "What the tool does + when to call.",
       "parameters": {
           "type": "object",
           "title": "<tool>Arguments",
           "required": ["arg1"],
           "properties": {
               "arg1": {"type": "string", "description": "…"},
           },
       },
       "mcp_server_id": _PAYMENTS_MCP_ID,   # or the new server's UUID
       "mcp_tool_name": "actual_tool_name_on_mcp_server",  # may differ from DB `name`
   }
   ```

3. **Wire to agents/skills** via `AGENT_TOOLS` and `SKILL_TOOLS`.
4. **Schema** is canned in seed — MCP `rediscover` overwrites it from the live server on the next health-check tick. So if the live MCP server's schema diverges from the seed, the live one wins after one tick. The seed schema is the cold-start placeholder.

## Recipe: add a new MCP server

1. **Implement the MCP server** under `mcp_servers/<name>/` (use `mcp_servers/payments/` as the template — FastMCP HTTP transport).
2. **Add to `docker-compose.yml`** as a sibling service. Mirror the payments-mcp `extra_hosts: ["host.docker.internal:host-gateway"]` if it needs to reach the host network.
3. **Append to `MCP_SERVERS`:**

   ```python
   {
       "id": uuid.UUID("40000000-0000-0000-0000-0000000000NN"),
       "name": "<name>-mcp",
       "description": None,
       "transport": "http",
       "url": "http://<service-name>:<port>/mcp",
       "auth_token": None,
   }
   ```

4. **Capture the UUID** as a module-level constant (e.g., `_NEWSVC_MCP_ID = uuid.UUID("…")`) and use it as `mcp_server_id` on each tool dict.
5. **Append the tools** to `TOOLS` with `mcp_server_id` pointing at the new server.
6. **Health-check** is automatic — the FastAPI lifespan loop probes every 60 seconds and updates `status` + `last_seen_at`. No code to add.
7. **Streamlit `/mcp-servers` UI** picks the new server up automatically once seeded.

## Validation checklist (run before committing seed changes)

- [ ] `uv run ruff format scripts/seed.py && uv run ruff check --fix scripts/seed.py` clean.
- [ ] `docker compose down --volumes && docker compose up -d --build` → `docker compose logs -f api` shows "Seed complete" with no exceptions.
- [ ] `curl localhost:8000/agents | jq '.[] | select(.name=="<your-agent>")'` returns the row.
- [ ] `curl localhost:8000/agents/<id>/prompt` returns the assembled system prompt + tool defs — read it to confirm the persona + skills + tools assembled the way you expect.
- [ ] If you added an MCP server: `curl localhost:8000/mcp-servers | jq '.[] | select(.name=="<your-server>")'` shows `status: "healthy"` within ~60s.
- [ ] If the agent is in a Flow or Team, run the flow once via `/flows/{id}/run` and confirm Langfuse shows the trace.

## Soft-delete vs replacement

If you're "replacing" an agent/skill/tool, **soft-delete the old row**, don't hard-delete. The replacement gets a new UUID; the old row stays with `is_deleted=true`. This is the only safe way — historical traces in Langfuse reference the old IDs, and FK constraints from old `agent_registry` / `agent_tools` rows still need to resolve.

## Common mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Agent's `AGENT_TOOLS` list missing a tool that one of its skills needs | Skill calls fail at runtime: "tool not bound to agent" | Add the tool to the agent's list (superset rule). |
| New tool dict missing `mcp_server_id` | Tool exists in DB but dispatch fails | Set `mcp_server_id` — every tool today is MCP-backed. |
| Same UUID reused | Seed runs as update instead of insert; old row's fields get overwritten | Use a fresh UUID for new entities. |
| Hard-delete a deprecated agent | FK constraint error or orphaned Langfuse traces | Set `is_deleted=true`, leave the row. |
| Forgot to run seed after schema change | Empty tables in fresh DB | Entrypoint runs `alembic upgrade head && python scripts/seed.py` — make sure your container actually restarted. |
