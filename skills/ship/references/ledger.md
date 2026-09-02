# The replacement ledger

`.ship/ledger.md`, in the project being built. The pipeline's record of what
each change replaced, and the mechanism that makes "no stale code" a gate
rather than an intention.

## Format

```markdown
# Replacement ledger

| id | added | supersedes | retire-by | status |
|----|-------|------------|-----------|--------|
| R1 | src/auth/session.ts | src/auth/legacy.ts, useOldAuth() | slice-2 | CLOSED |
| R2 | api/v2/claims.py | api/v1/claims.py | release | OPEN — mobile client cutover, est. 2026-09-15 |

## Closure proof

- **R1** closed at slice-2.
  `grep -rn "useOldAuth\|auth/legacy" --include="*.ts" .` → 0 hits.
  `src/auth/legacy.ts` deleted in the same commit as `session.ts`.
```

## Columns

| Column | Rule |
|--------|------|
| `id` | `R<n>`, never reused, referenced from commit messages |
| `added` | Path of the replacement |
| `supersedes` | Every path **and symbol** it replaces. Symbols matter: a file can survive while a function in it is dead |
| `retire-by` | `slice-<n>`, `prove`, or `release`. Defaults to the slice that created the row |
| `status` | `OPEN` or `CLOSED`. OPEN past its default carries the reason inline |

## Lifecycle

1. **Created** at plan time when the replacement is known, at build time
   otherwise. Never retroactively — a row written after the fact records what
   you remember, not what happened.
2. **Default retire-by is the same slice.** The replacement and the deletion
   land in one change. This is the rule that does the actual work; everything
   else is enforcement of it.
3. **Deferral needs a constraint**, written into the row: a live mixed-version
   rollout, an external consumer mid-cutover, a contract step awaiting
   authorization. Preference is not a constraint. There is nothing after
   `release`.
4. **Closure needs proof** — the search showing zero remaining references,
   quoted under the table. Intent does not close a row.

## Gate behavior

| Gate | Fails when |
|------|------------|
| Slice exit (phase 3) | A row created this slice is OPEN without a written reason |
| PROVE (gate 4→5) | Any row with `retire-by` of `prove` or earlier is OPEN |
| RELEASE (gate 5) | **Any** row is OPEN |

## Why a file and not a habit

Deferred deletion is the mechanism that produces stale code, and it never feels
like a mistake at the time — the replacement works, the old path is harmless,
cleanup is a follow-up. The follow-up does not happen, the two paths drift, and
a year later nobody knows which one is real.

Writing the row costs one line. It converts "we'll clean it up later" from a
private intention into a visible, gated commitment, and moves the decision to
the moment when the context to make it correctly still exists.

## Bootstrapping an existing project

`ship` on a repo with no ledger starts one. Do not backfill history — rows for
replacements made before the pipeline existed are guesses. Instead run
`ship-prune`'s full sweep once to establish the current stale baseline, record
it, and hold the line from there: the ledger governs changes this pipeline
makes, and the sweep governs what it inherited.
