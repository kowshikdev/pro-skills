---
name: ship
description: >
  End-to-end project pipeline in six gated phases: frame, map, slice, build,
  prove, release. Wraps the caveman and ponytail skill families, routing every
  task to the right specialist skill, delegating reads and reviews to cheap
  sub-agents, and enforcing a replacement ledger so superseded code dies in the
  same change that replaces it. Use when the user wants to build, plan, or ship
  a project or feature end to end, says "ship this", "/ship", "plan to
  deployment", "build this properly", "take this to production", "build it
  efficiently with no stale code", or asks to resume a pipeline in progress.
  Do NOT use for one-off questions, single obvious edits, or non-code work.
argument-hint: "[frame|map|slice|build|prove|release|prune|status]"
license: MIT
---

# Ship

A pipeline, not a vibe. Six phases, five gates. A phase may not start until the
gate behind it is green, and no gate passes on assertion — each one names the
command that proved it.

This skill routes. It does not re-implement what `caveman` and `ponytail`
already do; it decides which of them runs, when, and what must be true before
the next one starts.

## The two axes you are wrapping

- **ponytail** governs *how much* gets built. Every phase inherits its ladder:
  does this need to exist → already in repo → in stdlib → native platform →
  installed dependency → one line → minimum that works.
- **caveman** governs *how work is executed and reported*: workflow discipline
  per change type, compressed output, delegated reads.

Both stay on for the whole pipeline. They are the ambient rules, not steps.

## Phases and gates

| # | Phase | Produces | Gate to pass |
|---|-------|----------|--------------|
| 0 | FRAME | Acceptance conditions, explicit non-goals | Every acceptance line is observable and falsifiable |
| 1 | MAP | Prior art, reuse seams, constraints | Every planned capability marked reuse / extend / build, with `path:line` for the first two |
| 2 | SLICE | Ordered thin vertical slices | Slice 1 is end-to-end runnable and each slice has its own acceptance subset |
| 3 | BUILD | Working slices + ledger rows | Slice acceptance passes; every superseding change has a ledger row |
| 4 | PROVE | Proof set, two reviews, prune sweep | Focused proof green, both reviews clear, ledger has no OPEN row due by PROVE |
| 5 | RELEASE | Reversible rollout | Rollback path exercised, ledger fully closed, observability live |

**Status** (`/ship status`) is not a separate skill — read `.ship/ledger.md`
and the phase artifacts, then report in three lines: current gate, what is
blocking it, next action. Never silently skip a phase — if the user starts
mid-pipeline, say which earlier gates you are taking on trust.

## Routing

Load the phase skill; it carries the detail.

- Phase 0-2 → `ship-plan`
- Phase 3 → `ship-build`
- Phase 4 → `ship-verify`
- Phase 5 → `ship-release`
- Stale-code sweep (gates 3→4 and 4→5) → `ship-prune`

Within a phase, dispatch by change shape. Condensed table; full version with
all 26 wrapped skills is in `references/routing.md`.

| Situation | Invoke |
|-----------|--------|
| Cause of a failure is unknown | `investigate-first` — before any edit |
| New behavior, feature, integration | `lean-build` |
| Bug or small behavior change | `surgical-patch` |
| Restructure, behavior preserved | `safe-refactor` |
| Schema, API, config, protocol, dependency | `migration` |
| Validation only, last-mile proof | `verify-and-stop` |
| Cold start, or search already failed | `caveman-explore` |
| Locate code across files | `cavecrew-investigator` agent |
| Bounded 1-2 file edit | `cavecrew-builder` agent |
| Correctness review of a diff | `caveman-review` / `cavecrew-reviewer` agent |
| Over-engineering review of a diff | `ponytail-review` |
| Repo-wide bloat hunt | `ponytail-audit` |
| Deferred shortcuts owed | `ponytail-debt` |
| Commit message | `caveman-commit` |

Two reviews at gate 4 is deliberate: `caveman-review` asks *is it correct*,
`ponytail-review` asks *should it exist*. Neither answers the other's question.

## The stale-code contract

The pipeline's hard guarantee. Anything that supersedes existing behavior gets
a row in `.ship/ledger.md` at the moment it is written:

```
| id | added | supersedes | retire-by | status |
|----|-------|------------|-----------|--------|
| R1 | src/auth/session.ts | src/auth/legacy.ts, useOldAuth() | prove | OPEN |
```

Three rules, no exceptions:

1. **Delete in the same change.** The default `retire-by` is the slice that
   introduced the replacement. Deferring to `prove` needs a stated reason
   (mixed-version rollout, external consumer). Deferring past `release` is
   not available.
2. **Gate 5 fails while any row is OPEN.** "We'll clean it up later" is the
   mechanism that produces stale code, so the gate removes the option.
3. **Closing a row needs proof**, not intent: the superseded symbol has zero
   references, shown by a search you ran and quoted.

`ship-prune` runs the full sweep — orphans, duplicate paths, uncontracted
migrations, rotted markers, zombie dependencies, doc drift.

## Token discipline

Optimal here means cheap as well as correct. Main context is the scarce
resource, so:

- Reads fan out to `cavecrew-investigator` or `caveman-explore` (haiku,
  compressed) rather than pulling files into the main thread. Skip this when
  the exact file is already named — delegation has its own cost.
- Reviews fan out to `cavecrew-reviewer`.
- Bounded edits fan out to `cavecrew-builder`; it refuses 3+ file scope, which
  is a useful check on whether the slice was actually thin.
- Prose stays in `caveman` mode. Code stays in `ponytail` mode.
- A long-lived `CLAUDE.md` or plan file goes through `caveman-compress`.

## When the wrapped skills are absent

`caveman` and `ponytail` are separate marketplaces (see README). If a routed
skill is not installed, do not stall and do not silently drop the discipline:
apply the inline fallback in `references/routing.md` for that row, and say once
which skill would have run. The pipeline degrades in quality, never in shape.

## Stop conditions

Stop at the current gate and report when: acceptance is met, a gate fails for a
reason you cannot fix inside the current phase's scope, or a ledger row cannot
be closed because something outside the repo still depends on it. Report the
blocker and what would unblock it. Do not widen scope to get past a gate.
