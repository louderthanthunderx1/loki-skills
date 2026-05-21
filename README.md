# loki-skills

Agent skills loaded by Claude Code.

## Layout

Skills live under `skills/`, grouped into buckets:

- `engineering/` — daily code work
- `productivity/` — daily non-code workflow tools
- `general/` — cross-cutting behavioral guidelines
- `misc/` — kept around but rarely used

Each skill is its own directory containing a `SKILL.md` (with YAML frontmatter — `name` and `description`) and any bundled scripts or reference files under `references/`.

## Install

Symlink every shippable skill into `~/.claude/skills/`:

```bash
./scripts/link-skills.sh
```

List every `SKILL.md` in the repo:

```bash
./scripts/list-skills.sh
```

## Reference

### Engineering

- **[bpr-skill](./skills/engineering/bpr-skill/SKILL.md)** — End-to-end workflow playbook for `agentic-platform` (BPR). Pipeline (ticket → branch → brainstorm → spec → plan → implement → PR → Jira), hard rules (uv run, Alembic-only, soft-delete, ruff), and where to land changes. Platform architecture + runtime reference bundled.
- **[post-mortem](./skills/engineering/post-mortem/SKILL.md)** — Write the canonical engineering record of a fixed bug — root cause, mechanism, fix, validation, how it slipped through. Engineer-audience; refuses to draft without a reliable repro, known cause, and validated fix.
- **[scrutinize](./skills/engineering/scrutinize/SKILL.md)** — Outsider-perspective end-to-end review of a plan, PR, or code change. Questions intent (is there a simpler way?), traces the actual code path, and verifies the change does what it claims. Output is concise, actionable, with rationale.
- **[diagnose](./skills/engineering/diagnose/SKILL.md)** — Disciplined diagnosis loop for hard bugs and performance regressions: reproduce → minimise → hypothesise → instrument → fix → regression-test.
- **[tdd](./skills/engineering/tdd/SKILL.md)** — Test-driven development with red-green-refactor loop. Bundled references for deep-modules, interface design, mocking, refactoring, and tests.
- **[zoom-out](./skills/engineering/zoom-out/SKILL.md)** — Zoom out for broader context or higher-level perspective on unfamiliar code.

### Productivity

- **[management-talk](./skills/productivity/management-talk/SKILL.md)** — Rewrite engineer-to-engineer content for engineering-org leadership and shape it for the channel it's going to (JIRA, Slack, async standup, email, meeting talking-points).
- **[handoff](./skills/productivity/handoff/SKILL.md)** — Compact the current conversation into a handoff document for another agent to pick up.
- **[caveman](./skills/productivity/caveman/SKILL.md)** — Ultra-compressed communication mode. Cuts token usage ~75% by dropping filler, articles, and pleasantries while keeping full technical accuracy.
- **[grill-me](./skills/productivity/grill-me/SKILL.md)** — Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree.
- **[tva-timekeeper](./skills/productivity/tva-timekeeper/SKILL.md)** — Audit and repair the loki-skills repo structure. Sacred Timeline = the layout rules in CLAUDE.md. Detects Nexus Events (missing README entries, frontmatter drift, orphan files, broken links) across every skill and bucket, then prunes the branches on approval.

### General

- **[karpathy-guidelines](./skills/general/karpathy-guidelines/SKILL.md)** — Behavioral guidelines to reduce common LLM coding mistakes. Avoid overcomplication, make surgical changes, surface assumptions, define verifiable success criteria.

### Misc

- **[git-guardrails-claude-code](./skills/misc/git-guardrails-claude-code/SKILL.md)** — Set up Claude Code hooks to block dangerous git commands (push, reset --hard, clean, branch -D, etc.) before they execute. Fits the bpr-skill rule of letting the user push develop themselves.
