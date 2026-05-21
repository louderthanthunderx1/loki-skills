---
name: tva-timekeeper
description: Audit and repair the loki-skills repo structure — every SKILL.md registered in its bucket README and the top-level README, every bucket README listed in the root, frontmatter valid, no orphan files, no broken links. Use when user says "audit skills", "check skills repo", "tva check", "timekeeper", "prune timeline", or after adding/renaming/removing a skill.
---

# TVA Timekeeper

You are the Time Variance Authority for this repo. The **Sacred Timeline** is the structure defined in `CLAUDE.md`:

- Every skill lives at `skills/<bucket>/<skill-name>/SKILL.md`.
- Every `SKILL.md` has frontmatter with `name` and `description`.
- Every skill has an entry in the top-level `README.md`.
- Every skill has an entry in its bucket `README.md`.
- Reference material lives under the skill's `references/` dir.

Anything else is a **Nexus Event**. Detect, report, prune.

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

Build a table: `bucket | dir-name | frontmatter-name | description (first 80 chars)`.

### Phase 2 — Detect Nexus Events

Check every invariant. List every violation. Do not fix yet.

**Frontmatter variance**
- `SKILL.md` missing YAML frontmatter.
- Missing `name` or `description` field.
- `name` field does not match the skill directory name.
- `description` shorter than 40 chars (too vague to trigger reliably).

**Directory variance**
- Skill dir name uses uppercase, spaces, or non-kebab-case.
- `SKILL.md` exists outside `skills/<bucket>/<skill-name>/` (orphan).
- Skill dir contains files other than `SKILL.md`, `references/`, or bundled scripts called out in the SKILL.

**README variance — top-level `README.md`**
- A skill exists on disk but has no entry under its bucket section.
- A bullet links to a path that does not exist.
- Bucket section listed but not in `skills/`, or vice versa.

**README variance — bucket `README.md`**
- Bucket dir has no `README.md`.
- `README.md` lists a skill not on disk, or omits one that is.
- Link target does not resolve.

**Link rot**
- Any markdown link inside a `SKILL.md` or `README.md` pointing to a missing file in this repo.

### Phase 3 — Classify

Tag each Nexus Event:

- **Prune** — file/entry must be deleted (orphan, dead link, duplicate).
- **Restore** — entry missing, must be added (skill on disk but not in README).
- **Reset** — value wrong, must be corrected (frontmatter `name` mismatch, kebab-case fix).

### Phase 4 — Report

Output a single report. Format:

```
# TVA Audit — <date>

Skills scanned: <N>   Nexus events: <M>

## Prune
- <path> — <reason>

## Restore
- <path or entry> — <reason>

## Reset
- <path> :: <field> — <current> → <expected>

## Clean
<list of skills with zero variance>
```

If `M == 0`: output `Sacred Timeline intact. No variance detected.` and stop.

### Phase 5 — Prune the branches (only if user approves)

Do **not** auto-fix in the audit run. After reporting, ask:

> Variance detected. Prune? [y/N]

On `y`, apply fixes in this order:
1. **Reset** frontmatter and filenames first (least destructive).
2. **Restore** missing README entries — generate the bullet from the skill's `description`, first sentence, trimmed to ~150 chars.
3. **Prune** orphans and dead links last (most destructive — confirm each delete individually).

For every change, print: `[RESET|RESTORE|PRUNE] <path> — <one-line summary>`.

After fixes, re-run Phase 1+2 to confirm the timeline is clean. If new variance appears, stop and report — do not loop.

## Hard rules

- Never edit a `SKILL.md`'s body. Only frontmatter, only when `name` mismatches the directory.
- Never delete a `SKILL.md` without explicit user confirmation for that specific file.
- Never invent a `description` from scratch. If a skill has no description, flag it for the user to write — do not autogenerate.
- README bullets in the top-level `README.md` should match the existing voice (see neighbors). Copy structure, not content.
- Bucket `README.md` bullets are shorter than top-level bullets — match the neighbor style in that file.

## Output discipline

Reports are terse. One line per finding. Path + reason. No prose paragraphs. The Sacred Timeline does not need commentary.
