# Decisions — KB lifecycle protocol (ADR)

Append-only. Each decision records what was decided, why, and what it overrides.
Never rewrite an accepted decision; supersede it with a new numbered entry.

## KB-1. State lives in the central index, not in per-file headers

**Decision (2026-11-03).** The authoritative status of a document is its row in
`docs/INDEX.md` (active tree) or `docs/archive/INDEX.md` (archived). Files carry no
status header.

**Why.**
1. Archived files are frozen by rule; adding a header to them is a content change,
   so a per-file header cannot be the authority for archived docs anyway.
2. The repo had 88 free-text `Status:` lines in incompatible formats; a second
   copy of status in a header would drift.
3. The index is generated and checked by `kb`, so it cannot drift silently.

**Consequence.** `docs/specs/2026-09-22-judge-review-stats-design.md` proposed a
per-file `Active:` header; this ADR supersedes that part of it.

## KB-2. "superseded" vs "historical" is a field, not a status

Both mean "do not act on this"; they differ only in whether there is a successor to
redirect to. That difference is one pointer, so it is the `superseded_by` column of
an `archived` row. A publishing layer may derive the two labels later without
changing storage.

**Consequence.** The status vocabulary stays closed at four values; no `superseded`
or `historical` status exists, and adding one would be a schema break.

## KB-3. `final` is type-scoped: plans and reviews only

`final` means "terminal record: executed/observed, frozen, citable but not an
instruction". Living artifacts (framework, specs, knowledge, ops) never reach a
terminal event: they are either current authority (`active`, revisable) or replaced
(`archived`). Marking a living doc `final` would freeze the authority while reality
keeps moving.

**Why the distinction is operational, not cosmetic.**

| | `active` | `final` |
|---|---|---|
| What the reader does with it | acts on it — it is authority or an instruction | cites it / reproduces it, does not execute it again |
| May it change | yes, append a Revision | no, errata only |

For a plan this is the whole point: an `active` plan says "do this"; a `final` plan
says "this was done, do not run it again". For specs and knowledge the distinction
does not exist — a spec is the design of record, so implementation is not its
terminal event (the code moves, the spec gains a Revision).

**Consequence.** A realized plan is `final`; authority returns to the spec plus the
code. `plans` may legally go `active → archived` directly, but that is discouraged:
it loses the "record, not instruction" signal that prevents a re-run.

## KB-4. One engine, thin interfaces

The rules live in `schema.yaml` + `scripts/kb.py` in this skill directory. Workspaces
hold only a `docs/kb.yaml` with path overrides. MCP is a future `kb mcp` wrapper over
the same engine; a DSH plugin is only justified if write-time interception is needed.

**Why.** A copied schema is a second source of authority, and the two copies drift
the moment one workspace tunes a rule. Paths are the only thing that legitimately
varies per workspace.

## KB-5. Archived content is frozen; retrieval re-opens it

An archived file may only change by the retrieval flow: move it back into the active
tree, mark a new revision, then edit. `kb check` enforces the freeze with sha256.

**Why.** The archive is evidence. If archived bytes can change, "what did we believe
on 2026-09-17" stops being answerable.

## KB-6. Lifecycle state machines (per type family)

```
living  (framework / specs / knowledge / ops)
    draft ──approve──> active ──replaced|retired──> archived
      └────────────────abandon──────────────────────^

terminal (plans / reviews)
    draft ──start──> active ──done──> final ──no longer cited──> archived
      └────────────────abandon──────────────────────────────────^
```

The only backward edge is retrieval (`archived → draft|active`), and it requires a
new Revision marker. `kb check` rejects any other transition.

## KB-7. Status assignment for the 2026-11-03 migration

Applied once, to the 38 active files of this workspace:

- framework 1 → `active`
- specs 20 → `active`, except the two under review → `draft`
- plans 3 → `final` (all executed; kept as reproduction entry points)
- knowledge 10 → `active`
- reviews 2 → `final`
- ops 2 → `active`

**Why plans are `final` and not `archived`.** They are still cited by the current
campaign and by code docstrings, so they must stay in the active tree — but they must
not read as pending instructions.
