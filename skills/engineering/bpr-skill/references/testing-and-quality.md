# Testing + quality

The mechanical rules around code quality, test layout, and how to invoke them. Short and dense — keep this open while you work.

For the TDD **discipline** itself (red-green-refactor loop, mocking guardrails, interface design via tests, refactoring rules), load the **`tdd`** skill. This file covers the *mechanics* (where tests live, how to run them, marker rules); `tdd` covers the *practice*.

## `uv run` — every Python invocation

The project is `uv`-managed (`uv.lock` is committed). Bare `python` / `alembic` / `pytest` / `uvicorn` skip the locked environment and can pick up the wrong interpreter or stale dependencies. **Prefix every Python command with `uv run`:**

```bash
uv sync                                    # install deps from uv.lock
uv run uvicorn app.main:app --reload       # API on :8000
uv run streamlit run streamlit_app/Home.py # UI on :8501
uv run alembic upgrade head
uv run pytest
uv run python scripts/seed.py
```

The one exception is `docker compose` — that's the docker daemon, not Python, so no `uv run` prefix.

## Alembic — the only schema path

```bash
uv run alembic revision --autogenerate -m "describe change"
uv run alembic upgrade head
uv run alembic downgrade -1
```

**Never** call `init_db()` or `Base.metadata.create_all()`. Both bypass the migration history and leave the DB in a state that no other environment will ever match. The container entrypoint runs `alembic upgrade head && python scripts/seed.py` on every start — your local DB should follow the same path.

Soft-delete (`is_deleted: bool`) is on every model. Hard `DELETE` is forbidden:

- Mutations flip the flag.
- Queries filter `is_deleted.is_(False)`.

## Ruff — run after every `.py` edit

```bash
uv run ruff format <file-or-path>
uv run ruff check --fix <file-or-path>
```

- **Line length: 120** (set in `pyproject.toml`).
- The README mentions `black` / `flake8` — **ignore that**. Ruff only.
- Run *both* `format` then `check --fix`. Format handles whitespace/quotes/imports; check handles lint rules and applies safe autofixes.

CI runs `uv run ruff format --check ...` then `uv run ruff check ...` and **fails the PR** on any deviation. Run locally before pushing.

## Test taxonomy — label precisely

Two buckets. Get the markers right or CI either runs nothing relevant or runs live tests by accident:

| Bucket | Location | Marker | Dependencies | CI |
|--------|----------|--------|--------------|----|
| **Unit** | `tests/<engine\|training>/test_*.py` | none (default) | All dependencies mocked. No DB, no network. | Runs in CI. |
| **Integration** | `tests/<engine\|training>/integration/test_*.py` | `@pytest.mark.integration` | Real DB, real MCP servers, real containers. | Skipped in CI per ITAECD-398. Operator-run. |
| **Live** | same as integration | `@pytest.mark.live` | Hits real external services (Bedrock, Azure OpenAI, Langfuse cloud). Costs money. | Skipped in CI. Confirm with user before running. |

The marker config from `pyproject.toml`:

```python
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "session"
markers = [
    "live: integration test that hits real external services (skipped by default)",
    "integration: integration test marked for opt-in operator runs (skipped by default in CI per ITAECD-398)",
]
```

CI command (`.github/workflows/app-validate.yml`):

```bash
uv run pytest -m "not live and not integration"
```

## What "unit" means here

A unit test in this codebase is one where:

- The DB session is mocked (or uses an in-memory SQLite fixture, never the real Postgres).
- MCP clients are mocked.
- LLM clients are mocked.
- Langfuse client is mocked.

If your test imports `asyncpg`, opens a real session, or makes an HTTP call out, it's **not** a unit test — move it under `integration/` and add `@pytest.mark.integration`.

This precision matters because saved feedback explicitly calls it out: **"unit test = mocked dependencies; integration test = real API calls; label precisely in specs/plans."** Mis-labelling causes either flaky CI or false confidence.

## Running tests

```bash
uv run pytest                                  # everything except live + integration
uv run pytest tests/engine                     # engine only
uv run pytest tests/training                   # training only
uv run pytest tests/engine/test_graph_nodes.py # single file
uv run pytest tests/engine/test_graph_nodes.py::test_name   # single test
uv run pytest -k "graph"                       # by keyword
uv run pytest -m integration                   # opt-in integration
uv run pytest -m live                          # only if user confirmed (costs money)
```

**Don't auto-launch `-m live` or `-m integration`** without explicit user OK — they hit real services. This is a saved feedback memory ("Confirm before costly runs").

## Model-tier picking when writing tests for agents

The orchestrator and selector nodes hard-code their `LLMRole`. Per-agent role comes from `agents.model_tier` (precise / balanced / creative → temp 0.0 / 0.25 / 0.7). When you mock the LLM in a unit test:

- **Precise tier:** mock returns deterministic JSON / tool calls. No "creative" variation needed.
- **Balanced / creative:** mock can return multiple variants; assert structural properties, not exact strings.

Don't mock the temperature itself — the temperature is set via `with_config` on the cached LLM client; the mock just intercepts the call.

## Bedrock model ID gotcha

Saved feedback memory: **"Don't invent date/version suffix on Bedrock Anthropic IDs; copy verbatim from AWS Console + add region prefix for cross-region."**

When wiring tests or env vars that touch Bedrock model IDs:

- Copy the ID verbatim from the AWS Console. Don't append a date suffix that "looks like" the one on Claude.ai.
- Cross-region inference profiles need the region prefix (`us.anthropic.claude-...`).
- `Settings` validator fails fast at boot if Bedrock is selected without all required vars — that's intentional, don't loosen it.

## Pre-commit / CI mirror

The PR-template self-check mirrors `.github/workflows/app-validate.yml`:

1. `uv sync --extra dev`
2. `uv run ruff format --check <changed paths>`
3. `uv run ruff check <changed paths>`
4. `uv run pytest -m "not live and not integration"`
5. `docker build` (the api image)

Run these locally before pushing. If `docker build` fails locally it'll fail in CI too — debug locally rather than pushing and waiting.

## Common mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Bare `pytest` instead of `uv run pytest` | Picks up wrong env / missing deps | Always `uv run`. |
| Unit test hitting real DB | Flaky CI, slow runs | Move under `integration/`, mark `@pytest.mark.integration`. |
| Forgot to mark a live test | Costs $ when CI accidentally runs it | Add `@pytest.mark.live`. |
| Edited `.py` without ruff | CI fails the PR | `uv run ruff format <path> && uv run ruff check --fix <path>` before commit. |
| Used `Base.metadata.create_all` to "just get it working" | Schema diverges from migrations | Write the migration. There's no shortcut here. |
| Hard `DELETE` on a deprecated row | FK errors / orphaned Langfuse traces | `is_deleted=true` instead; filter in queries. |
| Bedrock model ID with invented suffix | Boot fails with "model not found" | Copy verbatim from AWS Console. |
