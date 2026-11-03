# kb — knowledge-base lifecycle for docs trees

![ci](https://github.com/HalloFei/Knowledge-Base-Management/actions/workflows/ci.yml/badge.svg)

A tiny, dependency-free engine that keeps a documentation tree honest: one authoritative
index, a closed status vocabulary, frozen archives, and a single command that fails when
reality and the index disagree.

Built for repos where docs accumulate: specs, plans, knowledge notes, reviews, ops runbooks.
Rules live in one `schema.yaml`; every workspace only supplies paths (`docs/kb.yaml`).

## What it enforces

| Rule | Check |
|---|---|
| every doc is indexed exactly once, with a status legal for its type | `kb check` |
| archived files are frozen (any byte change fails) | sha256 recorded in the archive index |
| `superseded_by` points at a real successor | path must exist |
| no dangling links in active docs | bare `docs/...` and relative links |
| status transitions are legal | `kb set-status` refuses illegal ones |

Status vocabulary (closed set): **draft / active / final / archived**.

- `final` is a **terminal record** state and is allowed only for types with a real terminal
  event — plans (executed) and reviews (observed). Living docs (framework / specs / knowledge /
  ops) never reach final: they are current authority (`active`) or replaced (`archived`).
- `superseded` vs `historical` is not a status; it is the `superseded_by` column of an
  `archived` row (pointer present = superseded).

## Install (per machine)

Copy or symlink this directory to `~/.dsh/skills/knowledge-base` (DSH skill discovery path),
or point `KB_SCHEMA` at `schema.yaml`. No Python dependencies (stdlib only, 3.8+).

```
git clone <this-repo> ~/.dsh/skills/knowledge-base
```

If the skill is already installed, replace the directory with a symlink so one edit changes
every workspace:

```bash
mv ~/.dsh/skills/knowledge-base ~/.dsh/skills/knowledge-base.bak
ln -s /path/to/kb-skill ~/.dsh/skills/knowledge-base
```

## Adopt in a workspace

1. `docs/kb.yaml` — paths only:

    schema_version: 1
    types:
      framework: docs/framework
      specs: docs/specs
      plans: docs/plans
      knowledge: docs/knowledge
      reviews: docs/reviews
      ops: docs/ops
    archive: docs/archive
    index: docs/INDEX.md
    archive_index: docs/archive/INDEX.md
    root_files: [AGENTS.md, README.md]

2. Add the two markers to `docs/INDEX.md` and `docs/archive/INDEX.md` (see
   `schema.yaml: index_markers`).
3. `python .../kb.py index --root .` then `... check --root .`.

## Commands

```bash
KB=<path>/scripts/kb.py
python $KB check                        # the gate: exit 1 on any inconsistency
python $KB index                        # regenerate both tables (preserves statuses/reasons)
python $KB status  docs/specs/x.md
python $KB set-status docs/specs/x.md final
python $KB archive docs/plans/y.md --superseded-by docs/specs/z.md --reason superseded
python $KB hook --install               # git pre-commit hook running `check`
```

## Design decisions

See `decisions.md` (ADR): index-as-truth, superseded-as-field, final-is-type-scoped,
one-engine/thin-interfaces, archived-content-frozen.

## Layout

    schema.yaml            the single source of rules (statuses, types, transitions, columns)
    scripts/kb.py          the engine (check / index / status / set-status / archive / hook)
    SKILL.md               DSH skill instructions (when/how to use)
    decisions.md           ADRs for the protocol itself
    tests/test_kb.py       unittest suite (python -m unittest discover -s tests -v)

## License

MIT — see `LICENSE`.
