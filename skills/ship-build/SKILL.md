---
name: ship-build
description: >
  Phase 3 of the ship pipeline: execute slices one at a time, routing each
  change to the right caveman workflow skill (investigate-first, lean-build,
  surgical-patch, safe-refactor, migration) under ponytail's YAGNI ladder, and
  recording every superseded path in the replacement ledger before moving on.
  Use when implementing planned work, when the ship pipeline reaches BUILD, or
  when asked to build a slice or feature that already has acceptance criteria.
---

# Ship: build

One slice at a time. A slice is done when its acceptance subset passes and its
ledger rows are closed — not when the code appears to work.

## Route before you edit

Pick exactly one workflow skill per change, by shape:

| Shape | Skill |
|-------|-------|
| Cause unknown, intermittent, perf regression | `investigate-first` **first**, then re-route |
| New behavior, product slice, integration | `lean-build` |
| Bug or small behavior change | `surgical-patch` |
| Restructure, behavior preserved | `safe-refactor` |
| Schema, data, API, protocol, config, dependency | `migration` |

Precedence when two fit: `investigate-first` beats everything (editing on a
guess is the most expensive move available); `migration` beats the feature
skills at a persistence or API boundary, because reversibility wins; refactor
never shares a change with a feature — split it.

## The ladder, on every added symbol

`ponytail` is the authority on how much gets written. Stop at the first rung
that holds:

1. Does this need to exist at all?
2. Already in this codebase? — MAP answered this. Use its `path:line`.
3. Stdlib does it?
4. Native platform feature covers it?
5. Already-installed dependency solves it?
6. Can it be one line?
7. Only then: the minimum code that works.

**The ladder shortens the solution, never the reading.** It runs *after* you
understand the change — trace every file it touches and the real flow first.
A minimal diff in the wrong place is not lazy, it is a second bug wearing
efficiency as a costume. This is why `investigate-first` outranks the ladder
whenever the cause is unknown.

For a bug fix, that means the root cause, not the symptom: grep every caller of
the function before editing. One guard in the shared function is a smaller diff
than a guard in each caller, *and* it fixes the siblings the ticket did not
name.

A new dependency needs a stated tradeoff, not a preference. Adding config,
providers, modes, or extensibility that no acceptance line requires is the
default failure mode of this phase.

Intensity is the user's call — `lite` names the lazier alternative and lets
them pick, `full` enforces the ladder, `ultra` challenges the requirement
itself. Default `full`. Never simplify away input validation at trust
boundaries, error handling that prevents data loss, security, accessibility, or
anything explicitly requested.

## Deliberate shortcuts get marked, not forgotten

A simplification that cuts a real corner with a known ceiling — a global lock,
an O(n²) scan, a naive heuristic — carries a `ponytail:` comment naming both
the ceiling and the upgrade path:

```python
# ponytail: global lock, per-account locks if throughput matters
```

`ponytail-debt` harvests these later, which only works if they were written.
Note the division of labour: `.ship/ledger.md` tracks what a change **replaced**
and must be deleted; a `ponytail:` marker tracks what a change **deferred** and
may be upgraded. Different questions, different lifecycles — do not merge them.

## Lazy code without its check is unfinished

Non-trivial logic — a branch, a loop, a parser, a money or security path —
leaves exactly one runnable check behind: the smallest thing that fails if the
logic breaks. An assert-based self-check or one small test. No frameworks, no
fixtures, no per-function suites unless asked. Trivial one-liners need none;
YAGNI applies to tests too.

## Delegate what is bounded

- 1-2 file edits → `cavecrew-builder`. It hard-refuses 3+ files, and that
  refusal is information: the slice was wider than planned. Re-cut rather than
  overriding.
- Locating anything → `cavecrew-investigator`, not main-context reads.

## Ledger rows are written here, not later

The moment a change supersedes existing behavior, its row exists in
`.ship/ledger.md`:

```
| id | added | supersedes | retire-by | status |
|----|-------|------------|-----------|--------|
| R3 | src/api/claims_v2.py | src/api/claims.py::list_claims | slice-3 | OPEN |
```

Default `retire-by` is **this slice**: the replacement and the deletion land in
the same change. Deferring to `prove` requires a real constraint — a
mixed-version rollout window, an external consumer mid-cutover — written into
the row. There is no defer past `release`.

This is the whole stale-code mechanism. A parallel implementation that outlives
its slice is how codebases rot, and the ledger exists so that outliving is a
gate failure rather than a habit.

## Per-slice exit

Before the next slice starts:

1. Slice acceptance passes — the actual command, run, quoted.
2. The nearest affected gate passes (lint, typecheck, unit tests for the
   touched package). Not the full suite; that is phase 4.
3. Ledger rows for this slice are CLOSED, each with the search proving zero
   remaining references — or explicitly deferred with a reason.
4. The repo is runnable. Never end a slice with the tree in a broken state.

Then stop. Report what the slice does, what it replaced and deleted, and any
material tradeoff. Do not start the next slice's work while reporting this one.

## What this phase does not do

No polish, no cleanup outside the slice, no unrelated tests, no drive-by
renames. Those are either their own slice or they are scope creep. When
acceptance passes, the slice is finished — continuing past it is the most
common way this phase produces the bloat that phase 4 then has to find.
