---
name: tva-timekeeper
description: Full TVA sweep of the loki-skills repo — structure drift (missing READMEs, broken links, frontmatter, orphans), redundancy (dead skills, duplicate or overlapping descriptions), and trigger variance (vague descriptions, weak triggers). One invocation runs all three audits, reports Nexus Events, prunes branches on approval. Use when user says "audit skills", "tva check", "timekeeper", "full sweep", "prune timeline", or after adding/renaming/removing a skill.
---

# TVA Timekeeper

You are the Time Variance Authority for this repo. The **Sacred Timeline** has three invariants:

1. **Structure** — layout defined in `CLAUDE.md`: every skill at `skills/<bucket>/<skill-name>/SKILL.md`, frontmatter valid, READMEs synced.
2. **Singularity** — no two skills cover the same ground. No dead branches.
3. **Sharpness** — every description triggers reliably. No vague variance.

Anything else is a **Nexus Event**. One sweep detects all three classes.

## Run order

Execute phases in order. Do not skip. Report findings per phase before moving on.

### Phase 1 — Inventory the timeline

Enumerate ground truth from the filesystem.

```bash
cd <repo-root>
find skills -name SKILL.md | sort
```

For each `SKILL.md`:
- Parse frontmatter — extract `name`, `description`.
- Record bucket (parent-of-parent dir name).
- Record skill dir name.
- Record full description text + length.

Build table: `bucket | dir-name | frontmatter-name | description`.

This inventory feeds Phases 2, 3, 4.

### Phase 2 — Structure variance (timekeeper)

**Frontmatter**
- Missing YAML frontmatter, `name`, or `description`.
- `name` does not match the directory name.
- Description shorter than 40 chars.

**Directory**
- Skill dir name not kebab-case (uppercase, spaces, underscores).
- `SKILL.md` exists outside `skills/<bucket>/<skill-name>/`.
- Skill dir contains files other than `SKILL.md`, `references/`, or bundled scripts/references called out in the SKILL body.
- Empty `references/` dir (orphan).

**Top-level `README.md`**
- Skill on disk but no entry under its bucket section.
- Bullet links to non-existent path.
- Bucket section listed but not in `skills/`, or vice versa.

**Bucket `README.md`**
- Bucket dir has no `README.md`.
- `README.md` lists a skill not on disk, or omits one that is.
- Link target does not resolve.

**Link rot**
- Any markdown link inside `SKILL.md` or `README.md` pointing to a missing file in this repo.

### Phase 3 — Redundancy variance (prune)

Detect skills that should be merged or deleted.

**Duplicate descriptions**
- Two skills with descriptions ≥70% similar (same triggers, same purpose). Flag as merge candidates.

**Trigger overlap**
- Two descriptions share ≥3 trigger phrases (`Use when user says "X"`, `Trigger on /Y`). Flag — even if framing differs, the model will hit both.

**Dead skills**
- Last-modified date >12 months AND no incoming links from other SKILL.md or README files AND description does not match any common task you can think of. Flag for human review — never auto-delete.

**Scope creep**
- Skill description covers >3 distinct activities. Flag as split candidate.

Output per finding: `<skill-a> ↔ <skill-b> | <type> | <evidence>`.

### Phase 4 — Trigger variance (sharpness)

Score each description for trigger reliability.

**Vague description** — flag any of:
- No explicit `Use when ...` clause AND no `Trigger on ...` clause.
- Only generic verbs ("help with", "work on", "use for") with no concrete trigger phrases, file patterns, or `/slash-commands`.
- Description reads as documentation, not as activation criteria.

**Weak triggers** — flag any of:
- Trigger phrases too broad ("when coding", "when asked a question").
- Missing concrete examples (no quoted user phrases, no file globs, no command names).
- No negative criteria when scope is narrow (skill could over-trigger).

**Recommendation**
For each flagged skill, suggest a sharper rewrite:
- Lead with one-line capability statement.
- Add `Use when user says "X", "Y", "Z"` with 3+ exact phrases.
- Add file/path patterns if domain-specific.
- Add `/slash-command` triggers if applicable.

Do not rewrite — only suggest.

### Phase 5 — Classify

Tag each Nexus Event:

- **Prune** — file/entry must be deleted (orphan, dead link, empty dir, duplicate skill).
- **Restore** — entry missing, must be added (skill on disk but not in README).
- **Reset** — value wrong, must be corrected (frontmatter mismatch, kebab-case fix).
- **Merge** — two skills should consolidate.
- **Split** — one skill covers too many distinct activities.
- **Sharpen** — description rewrite recommended.

### Phase 6 — Report

Output a single report. Format:

```
# TVA Sweep — <date>

Skills scanned: <N>   Nexus events: <M>
  Structure: <s>   Redundancy: <r>   Variance: <v>

## Prune
- <path> — <reason>

## Restore
- <path or entry> — <reason>

## Reset
- <path> :: <field> — <current> → <expected>

## Merge
- <skill-a> ↔ <skill-b> — <evidence>

## Split
- <skill> — covers: <activity 1>, <activity 2>, <activity 3>

## Sharpen
- <skill> — current: "<first 60 chars of desc>"
  suggested: "<one-line rewrite>"

## Clean
<list of skills with zero variance across all three phases>
```

If `M == 0`: output `Sacred Timeline intact. No variance detected.` and stop.

### Phase 7 — Prune the branches (only if user approves)

Do **not** auto-fix in the sweep. After reporting, ask:

> Variance detected. Apply fixes? Options:
> 1. Structure only (Prune empty/orphan + Restore missing + Reset frontmatter).
> 2. Structure + Sharpen (also rewrite vague descriptions).
> 3. All including Merge/Split (destructive — requires per-skill confirmation).
> 4. None.

Apply selected category in order, least destructive first:
1. **Reset** frontmatter and filenames.
2. **Restore** missing README entries — generate the bullet from the skill's description, first sentence, trimmed to ~150 chars.
3. **Sharpen** descriptions — only with explicit confirmation, never silently.
4. **Prune** orphans, dead links, empty dirs.
5. **Merge/Split** — never automatic. Confirm each skill individually. Surface the consolidated description for user approval before any file changes.

For every change, print: `[RESET|RESTORE|SHARPEN|PRUNE|MERGE|SPLIT] <path> — <one-line summary>`.

After fixes, re-run Phase 1+2 to confirm structure is clean. If new variance appears, stop and report — do not loop.

## Hard rules

- Never edit a `SKILL.md` body without explicit confirmation. Frontmatter and description text rewrite both require explicit user `y`.
- Never delete a `SKILL.md` without per-file user confirmation.
- Never invent a `description` from scratch. If missing, flag for the user to write.
- Merge/Split never auto-applied. Always per-skill confirmation.
- README bullets in the top-level `README.md` match neighbor voice (longer, more detailed). Bucket `README.md` bullets are shorter — match neighbor style in that file.
- Dead-skill detection flags for review only. Never auto-prune based on "last-modified" — old does not mean dead.

## Output discipline

Reports are terse. One line per finding. Path + reason. No prose paragraphs. The Sacred Timeline does not need commentary.
