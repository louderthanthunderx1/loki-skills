# Ticket pipeline (the steps in detail)

The single rail every feature, refactor, or chore rides. Each step has a *gate* — don't open the next gate until the current one closes.

## 1. Ticket

Every piece of work gets a Jira ticket `ITAECD-NNN`. If the user kicks off work without naming one, *ask*. Two patterns:

- **Single-ticket work** — one branch, one spec, one plan, one PR, one Jira summary.
- **Multi-ticket feature** — umbrella folder `docs/superpowers/features/<feature-name>/` + per-ticket spec/plan as usual. See `docs/superpowers/features/mcp-migration/` for the canonical structure.

**Cross-check:** does the ticket already have a sibling spec under `docs/superpowers/specs/`? If yes, you're continuing existing work — read it first, don't re-brainstorm from scratch.

## 2. Branch

- Base: **always `develop`**, never `main`.
- Naming:
  - Ticketed: `feature/ITAECD-NNN-<kebab-summary>`
  - Cleanup: `chore/<topic>`
- Verify with `git rev-parse --abbrev-ref HEAD` before continuing.

`develop` is the integration branch. `main` is a release pointer — only ops/release flows touch it.

## 3. Brainstorm — `superpowers:brainstorming`

**No code, no spec text until brainstorming completes.** This is the highest-leverage step in the whole pipeline. It:

- Locks scope (what's *out* matters as much as what's in).
- Surfaces 2–3 approaches with tradeoffs.
- Presents the design in sections, then writes the spec.

Skipping this is the #1 cause of rework on this project — CLAUDE.md says so explicitly. Even small tasks earn the brainstorm pass.

## 4. Spec

Folder: `docs/superpowers/specs/<BRANCH-NAME>/`

Required files:

```
docs/superpowers/specs/<BRANCH-NAME>/
  YYYY-MM-DD-<topic>-design.md       # the spec narrative
  architecture.md                    # mermaid blocks, embedded inline
  diagrams/
    01-<descriptive-name>.mmd        # source
    01-<descriptive-name>.png        # rendered at scale 3, width 1600
    02-<descriptive-name>.mmd
    02-<descriptive-name>.png
    ...
```

**Filenames must be descriptive** — `arch-01` is not acceptable; `01-system-view`, `02-orchestrator-flow`, `03-error-paths` are.

Mermaid is **required**. Specs without diagrams will block PR review. Render details in [spec-plan-layout.md](spec-plan-layout.md).

## 5. Plan — `superpowers:writing-plans`

Folder: `docs/superpowers/plans/<BRANCH-NAME>/`

Same structure as spec, but the diagrams depict *the work*: task DAG, phase progression, breakage timeline, test growth, commit cadence. The spec answers "what is the work"; the plan answers "how will the work land".

Plan content includes:

- Bite-sized TDD steps (each step: failing test → implementation → green test).
- One atomic commit per task.
- Tests labelled precisely: **unit** (mocked) vs **integration** (real API / containers). See [testing-and-quality.md](testing-and-quality.md).

## 6. Implement

Two execution modes:

- **`superpowers:subagent-driven-development`** (preferred) — dispatches each plan task to a subagent in parallel where dependencies allow. Keeps the main context clean.
- **`superpowers:executing-plans`** — runs the plan inline in the current session. Use when tasks are tightly coupled or when subagents would thrash on shared state.

Pick the mode at the end of `writing-plans`. Don't switch mid-execution without telling the user.

Per-task discipline (load the **`tdd`** skill for red-green-refactor discipline + mocking/refactoring guardrails):

1. Read the task from the plan.
2. Write the failing test first (TDD).
3. Make it green.
4. Run `uv run ruff format <files>` + `uv run ruff check --fix <files>`.
5. Run `uv run pytest` for the relevant slice.
6. Commit atomically — one task, one commit, message references `ITAECD-NNN`.

## 7. Self-review

Before opening the PR, run **`scrutinize`** on the full diff. Catches: inconsistencies between spec and code, missing tests, scope creep, dead code, half-finished comments. Cheaper than the user catching it on PR review.

If this ticket was a prod bugfix, also queue **`post-mortem`** for after merge (writes root cause + mechanism + validation into the ticket close-out).

## 8. PR + Jira summary

- Push the branch with `-u` if it's the first push.
- Open the PR: `gh pr create --assignee theeraphan --body-file docs/superpowers/pr/<branch-name>.md` (or pipe the body via HEREDOC). PR body conventions in [pr-and-jira.md](pr-and-jira.md).
- **Do not** include the `🤖 Generated with Claude Code` footer in PR bodies.
- Jira summary draft (untracked file) is OK *before* merge — uses the template at `docs/templates/jira-feature-summary.md`.
- At merge, commit the summary under `docs/superpowers/jira/<TICKET-ID>-<KEBAB-SUMMARY>-YYYY-MM-DD-summary.md` and paste the body into the Jira ticket Description.
- PR body = engineer voice (raw). Jira Description = BA/PM voice — pipe draft through `grill-me` then `management-talk` before pasting.

See [pr-and-jira.md](pr-and-jira.md) for the full template + timing rules.

## Common deviations and how to handle them

| Situation | Right move |
|-----------|-----------|
| Tiny one-line fix on `develop` | Still ticket + branch. Brainstorm can be 2 lines, spec can be the PR description — but don't skip the rail entirely. |
| Hotfix on `main` | Ask the user first. The default rail is develop-only; main hotfixes are an exception that should be explicit. |
| Multi-ticket feature | Create the feature umbrella *before* the first ticket's spec. Subsequent specs link upward into it. |
| User says "just do it, no spec" | Confirm scope verbally, then proceed — but still ticket + branch + ruff + PR. The user can waive the spec; they cannot waive the rail. |

## Reference example

[docs/superpowers/specs/ITAECD-378-ba-case-auto-fill-expected-fields/](../../../docs/superpowers/specs/ITAECD-378-ba-case-auto-fill-expected-fields/) is the most recent canonical example — design doc + architecture.md + diagrams folder. Use it as a shape reference.
