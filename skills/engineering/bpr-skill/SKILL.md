---
name: bpr-skill
description: Use when working on the agentic-platform repo (Loki). Triggers on ITAECD-* tickets, branch creation, spec/plan/brainstorm requests, edits under app/engine/, app/training/, mcp_servers/, streamlit_app/, alembic/, docs/superpowers/, PR/Jira work, ruff, Alembic, soft-delete, mermaid diagrams. Triggers even on quick questions — conventions are surprising enough that one wrong default (init_db, missing mermaid, hard-deletion, missing Langfuse sync, missing assignee) costs more than the read.
---

# Loki — agentic-platform workflow

CLAUDE.md = architecture source of truth. This skill = workflow source of truth. Conflict → CLAUDE.md wins on architecture, this skill wins on workflow.

## Pipeline

```
Ticket → Branch → Brainstorm → Spec → Plan → Implement → Self-review → PR → Jira summary
```

1. **Ticket** `ITAECD-NNN` — ask before touching code if not named.
2. **Branch off `develop`** — `feature/ITAECD-NNN-<kebab>` or `chore/<topic>`. Never off `main`.
3. **Brainstorm** via `superpowers:brainstorming`. No code, no spec before this completes.
4. **Spec** `docs/superpowers/specs/<BRANCH-NAME>/` — design doc + `architecture.md` + `diagrams/NN-<name>.mmd` + `.png`. Folder name = branch name **without** `feature/` prefix.
5. **Plan** `docs/superpowers/plans/<BRANCH-NAME>/` — same shape, diagrams show the *work* not the end-state.
6. **Implement** via `superpowers:subagent-driven-development` (preferred) or `superpowers:executing-plans`. TDD per task (load `tdd` skill for the discipline), one atomic commit per task.
7. **Self-review** with `scrutinize` on the full diff before opening the PR. Catches inconsistencies, missing tests, scope creep that you missed mid-stream.
8. **PR** `gh pr create --assignee theeraphan --body-file …` — no `🤖 Generated with Claude Code` footer in body. After merge, commit Jira summary to `docs/superpowers/jira/<TICKET-ID>-<KEBAB>-YYYY-MM-DD-summary.md`. PR body is engineer-voice; Jira Description targets BAs/PMs — pipe the draft through `grill-me` then `management-talk` before pasting. If the ticket was a prod bugfix, run `post-mortem` before closing.

## Hard rules

| Rule | Detail |
|------|--------|
| `uv run` prefix | Every Python/Alembic command. Bare `python`/`alembic` skips locked env. |
| Alembic only | Never `init_db` or `Base.metadata.create_all`. Migration is the only schema path. |
| Soft-delete only | Flip `is_deleted=True`. Hard `DELETE` forbidden. Filter `is_deleted.is_(False)` every query. |
| Ruff after every `.py` | `uv run ruff format <path>` then `uv run ruff check --fix <path>`. Line length 120. Ignore README's black/flake8. |
| English only | Docs, commits, comments. |
| No AI footer in PRs | No `🤖 Generated with Claude Code`. `Co-Authored-By:` in commits stays. |
| User pushes `develop` | Commit locally, stop. Don't `git push` develop. |
| Confirm costly runs | Live tests, Bedrock calls, container builds — ask first. |
| Wait for explicit pick | Paste-back logs ≠ approval for numbered options. |

## Reference lookup

| About to… | Read |
|-----------|------|
| Start a ticket, brainstorm, spec, or plan | [references/ticket-pipeline.md](references/ticket-pipeline.md) + [references/spec-plan-layout.md](references/spec-plan-layout.md) |
| Render mermaid → PNG | [references/spec-plan-layout.md](references/spec-plan-layout.md) |
| Touch `app/engine/core/` or `app/training/`, start services, ship a deploy | [references/platform-architecture.md](references/platform-architecture.md) |
| Add Agent / Skill / Tool / MCP server | [references/platform-entity-seeding.md](references/platform-entity-seeding.md) |
| Open PR or write Jira summary | [references/pr-and-jira.md](references/pr-and-jira.md) |
| Write tests, run ruff, pick model tier | [references/testing-and-quality.md](references/testing-and-quality.md) |
