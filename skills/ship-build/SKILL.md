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

`ponytail` is on. Before writing anything, walk it: does this need to exist →
already in the repo (the MAP phase answered this — use it) → in the stdlib →
a native platform feature → an installed dependency → one line → the minimum
that works.

A new dependency needs a stated tradeoff, not a preference. Adding config,
providers, modes, or extensibility that no acceptance line requires is the
default failure mode of this phase.

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
