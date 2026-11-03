---
name: knowledge-base
description: Use when adding, moving, archiving, or auditing documentation in any workspace that uses the kb protocol — including questions like "is this doc current?", "where did spec X go?", "archive the old specs", or when the docs tree looks messy. Also use before writing a new spec/plan, and when a session needs the authoritative status of a design document.
---

# Knowledge base protocol (kb)

One engine, one schema, many workspaces. **The rules live in this skill directory;
a workspace holds only paths.**

- Engine: `scripts/kb.py` (stdlib only, Python 3.8+)
- Rules: `schema.yaml` (status vocabulary, per-type lifecycle, transitions, index columns)
- Decisions: `decisions.md` (why index-as-truth, why superseded/historical is a field, why final is type-scoped)

## Non-negotiables

1. **Never hand-edit the indexed state.** Change status via `kb set-status` or `kb archive`;
   the tables in `docs/INDEX.md` and `docs/archive/INDEX.md` are generated.
2. **Archived content is frozen.** Only the retrieval flow may reopen it: move back with
   `kb`/git, mark a new revision, then edit.
3. **Workspaces never copy the schema.** If a workspace needs different paths it writes
   `docs/kb.yaml` (paths only, no rules).
4. **Status vocabulary** (closed set): draft / active / final / archived.
   - `final` is allowed **only** for plans and reviews (terminal records).
   - living docs (framework, specs, knowledge, ops) go draft → active → archived.
   - `superseded` vs `historical` is the `superseded_by` column of an archived row,
     not a status.

## Commands

```bash
KB=~/.dsh/skills/knowledge-base/scripts/kb.py
python $KB check                    # validate: index<->tree, statuses, sha256 freeze, links
python $KB index                    # regenerate both tables (preserves statuses/reasons)
python $KB status  docs/specs/x.md  # what is it?
python $KB set-status docs/specs/x.md final
python $KB archive docs/plans/y.md --superseded-by docs/specs/z.md --reason superseded
python $KB hook --install           # install the git pre-commit gate (runs check)
```

Run from the workspace root (or pass `--root DIR`). No config? `kb` walks up looking
for `docs/kb.yaml`.

## New document

1. Put it in the type directory (`docs/specs/`, `docs/knowledge/`, ...).
2. `python $KB index` — it appears with the type's default status.
3. Set the real status with `kb set-status`; for a new spec the default is `draft`.

## Archiving

1. `python $KB archive <path> [--superseded-by <successor>] [--reason <vocab>]`
   (uses git mv when tracked, then rebuilds both tables and records the sha256).
2. Never edit the file after archiving; `kb check` fails on any byte change.

## When `kb check` fails

- "missing from active index" → run `kb index`, then set the intended status.
- "invalid status" → set-status to a value allowed for that type.
- "frozen archive changed" → you edited an archived file; retrieve it properly.
- "broken link" → fix the referring document (active docs only; archived links may dangle).

## Reporting

After any change, run `kb check` and paste its one-line result. Do not claim the docs
tree is consistent without that line.
