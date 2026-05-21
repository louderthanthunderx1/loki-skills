# PR + Jira summary

The closing third of the pipeline. Two artefacts, two timing rules, one common-failure pattern.

## Where bodies live

- **PR body drafts:** [docs/superpowers/pr/](../../../docs/superpowers/pr/) — one `.md` per PR. Naming has drifted historically (`pr_body_itaecd403.md`, `ITAECD-411-pr-body.md`); pick either pattern but stay consistent within the PR.
- **Jira summaries (committed):** [docs/superpowers/jira/](../../../docs/superpowers/jira/) — `ITAECD-NNN-<KEBAB-SUMMARY>-YYYY-MM-DD-summary.md`. *Flat filename, one file per ticket — not a folder.*
- **Jira template:** [docs/templates/jira-feature-summary.md](../../../docs/templates/jira-feature-summary.md) — Atlassian wiki markup. Copy, fill, save.

## Opening the PR — the one-liner

```bash
gh pr create \
  --assignee theeraphan \
  --body-file docs/superpowers/pr/<branch-name>.md \
  --title "ITAECD-NNN — <one-line description, <70 chars>"
```

Non-negotiables (user feedback memory, not optional):

- **`--assignee theeraphan`** every time. Don't defer to the operator UI to assign later.
- **`--body-file`** with the markdown draft you wrote in `docs/superpowers/pr/`. Don't paste a long body via `-b`.
- **No `🤖 Generated with Claude Code` footer** in the body. The `Co-Authored-By:` trailer in *commits* stays (per CLAUDE.md), but PR bodies and docs stay footer-free.
- **Title < 70 chars.** Detail goes in the body, not the title.

## PR body structure

Follow the shape used in [docs/superpowers/pr/pr_body_itaecd405.md](../../../docs/superpowers/pr/pr_body_itaecd405.md) — it's the most polished recent example. The shape:

```markdown
## Summary

- Bullet 1 — what changed, in one sentence.
- Bullet 2 — why (the constraint, deadline, or stakeholder driver).
- Bullet 3 — link to spec + plan: [spec](docs/superpowers/specs/<branch>/...) and [plan](docs/superpowers/plans/<branch>/...).

## <Section per major area: Schema change / Runtime change / API surface / Seed / Tests / Docs>

<Inline tables, ASCII diagrams, code blocks. Reviewers should be able to grok the change without opening the spec.>

## Test plan

- [ ] Concrete test 1 with the command (`uv run pytest tests/engine/test_x.py`).
- [ ] Concrete test 2.
- [ ] Manual smoke: `curl localhost:8000/<endpoint>` → expected shape.
```

For multi-phase migrations (like ITAECD-405), include a **deploy sequence** section that names every step in order (Alembic forward + script + Alembic forward). Reviewers shouldn't have to reconstruct the deploy order from the diff.

## When to push and when not to

- **Feature branches** (`feature/ITAECD-NNN-...`): push them yourself with `git push -u origin <branch>` before `gh pr create`.
- **`develop`**: **the user pushes develop themselves.** When you commit on develop locally (Jira summary, hotfix, one-off), *stop*. Don't `git push`. Tell the user the commit is ready and let them push.

This is a saved feedback memory — breaking it loses the user's control over when develop moves.

## Jira summary — draft vs commit timing

Two rules that look like one. Get them right and the timing of artefacts stays clean:

### Rule A — Draft pre-merge is OK as an untracked file

While the PR is open, you can draft the summary at the eventual final path:

```
docs/superpowers/jira/ITAECD-NNN-<KEBAB>-YYYY-MM-DD-summary.md
```

…and *not stage it*. Keep it as an untracked working file. That way the content is ready when the PR merges, without the file showing up in PR diffs prematurely.

### Rule B — Only commit AT merge time

The summary is committed (and pushed) in a separate develop commit immediately after the PR merges. The date in the filename is the merge date.

**If the timing was wrong** (e.g., you committed too early on the feature branch), *correct the timing* — move the file back to untracked and recommit at merge — but **do not delete the draft content**. The draft has value; only the commit timing was off. This is another saved feedback memory.

## Jira summary content — what to keep

The template at [docs/templates/jira-feature-summary.md](../../../docs/templates/jira-feature-summary.md) is the canonical shape (Atlassian wiki markup: `h1.`, `h2.`, `*bold*`, `{{code}}`, `*` bullet, `#` numbered). It enumerates:

- **Summary** — 2-3 sentences. What changed, why, headline result. Mention if cliff cutover or staged migration.
- **Why** — motivation: prior pain, stakeholder ask. One paragraph.
- **Scope delivered** — Database / Core runtime / REST + UI / Reference services / Seed / Cliff cutover / Tests / Docs subsections.
- **Result / impact** — concrete observable change.
- **Follow-up tickets** — `*ITAECD-NNN*` references.
- **Out of scope (deferred)** — what was explicitly punted + where it's captured.
- **Verification performed before merge** — numbered list of commands + expected results.

[docs/superpowers/jira/ITAECD-405-db-normalization-training-system-2026-05-14-summary.md](../../../docs/superpowers/jira/ITAECD-405-db-normalization-training-system-2026-05-14-summary.md) is a recent complete example — use it as a length and depth reference.

### Jira Cloud vs Server/DC

The template is wiki markup (Server/DC). Jira Cloud accepts markdown for most fields:

- `h2.` → `##`
- `*bold*` → `**bold**`
- `{{code}}` → `` `code` ``
- `*` bullets → `-` or `1.`

Convert if the destination is Cloud.

## Shaping prose for the audience

PR body and Jira Description target **different readers** — different voice register required:

| Artefact | Reader | Voice |
|----------|--------|-------|
| PR body | Engineers reviewing the diff | Engineer-to-engineer. Code identifiers welcome. Terse, technical, no business framing. |
| Jira Description | BAs, PMs, engineering leadership | Outcome-first. Business framing on *why*. Code identifiers minimal — translate to concepts. Headline result up top. |

Two skills help here:

- **`grill-me`** — stress-test the draft before posting. Surfaces weak claims, missing rationale, unresolved decisions. Run on the Jira Description and on the PR Summary section.
- **`management-talk`** — translate engineer-prose → leadership-prose for the Jira Description. Use after the draft is technically accurate but reads too "in the weeds" for non-engineers. Does *not* apply to PR body (engineer audience).

Workflow:

```
1. Draft Jira summary (untracked file, engineer voice).
2. /grill-me on it       → resolve weak spots.
3. /management-talk      → reshape Description for BA/PM/leadership.
4. Paste into Jira Description at merge time.
```

PR body skips step 3 — engineers read it raw.

## Common mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `🤖 Generated with Claude Code` footer slipped in | User flags it on review | Edit body, push, re-open PR if needed. |
| Forgot `--assignee theeraphan` | PR sits unassigned | `gh pr edit <num> --add-assignee theeraphan`. |
| Pushed develop yourself | User flags it | Avoid in future. The user pushes develop. |
| Jira summary committed on feature branch | Shows up in PR diff | Move file to untracked, drop commit, recommit on develop at merge time. |
| Deleted the draft on timing correction | Content is gone | Don't delete — only fix the timing. Restore from reflog if needed. |
| Title > 70 chars | GitHub truncates in lists | Trim title; push detail into body. |
| Pasted long body inline via `-b` | Hard to revise | Re-do with `--body-file`. |

## End-to-end skeleton

```bash
# 1. Verify branch + clean tree
git status

# 2. Push feature branch
git push -u origin feature/ITAECD-NNN-foo

# 3. Open PR with assignee + body file
gh pr create \
  --assignee theeraphan \
  --title "ITAECD-NNN — short description" \
  --body-file docs/superpowers/pr/ITAECD-NNN-pr-body.md

# 4. Capture URL, share with user. Wait for review/merge.

# 5. After merge — switch to develop locally, pull, drop summary in
git checkout develop && git pull
cp /path/to/draft docs/superpowers/jira/ITAECD-NNN-<kebab>-YYYY-MM-DD-summary.md
git add docs/superpowers/jira/ITAECD-NNN-<kebab>-YYYY-MM-DD-summary.md
git commit -m "docs(jira): ITAECD-NNN summary"

# 6. STOP. Let the user push develop themselves.
```
