---
name: ship-verify
description: >
  Phase 4 of the ship pipeline: the gate before release. Builds the smallest
  sufficient proof set via verify-and-stop, runs a correctness review and an
  over-engineering review as two separate passes, triggers the stale-code
  sweep, and refuses to pass while any ledger row is open. Use when work is
  believed complete, when asked to verify or validate before shipping, or when
  the ship pipeline reaches PROVE.
---

# Ship: prove

The gate. This phase does not build; it decides whether what was built is
allowed to leave. That distinction is what makes it a gate, so it does not edit
product code unless the user's request explicitly includes fixes.

## 1. Proof, not assertion

Run `verify-and-stop`. Its rules hold here:

- Translate acceptance into the **smallest sufficient** proof set. A full suite
  run when three focused tests would settle it is waste, not rigor.
- Reuse still-current results — same repo state, same inputs — rather than
  re-running for the feeling of it.
- Focused checks before wide gates.
- Report four states exactly, never collapsing them: **pass**, **fail**,
  **unavailable** (no such check exists), **blocked** (exists, could not run).
  Reporting *unavailable* as *pass* is the single most damaging error in this
  phase.
- Quote the commands and their results. A claim of green without a command is
  not a proof.

## 2. Two reviews, different questions

Run both. Neither substitutes for the other, and running only one is how a
codebase ends up either correct-and-bloated or lean-and-broken.

| Review | Skill | Asks |
|--------|-------|------|
| Correctness | `caveman-review` or `cavecrew-reviewer` | Does it do the right thing? Edge cases, error paths, invariants, concurrency |
| Complexity | `ponytail-review` | Should it exist? Reinvented stdlib, unneeded deps, speculative abstraction, dead flexibility |

Delegate both to sub-agents where available — reviews are read-heavy and their
findings compress well.

For each complexity finding, apply the deletion test: **what breaks if this is
removed?** If the answer is nothing, or only a hypothetical future caller, it
goes. "Might need it later" is the phrase this phase exists to catch.

## 3. Bloat delta — the measured gate

Re-run `ponytail-audit` and compare against `.ship/audit-baseline.md` from
phase 1. The audit ends with `net: -<N> lines, -<M> deps possible.`, and that
number is what makes this a gate rather than a sentiment.

**The rule: `N` and `M` must not be higher than baseline.** A build that leaves
more cuttable code than it found has added bloat, whatever its diff looks like
and however well it passes its tests.

Read the delta by tag; each one has a different remedy:

| Tag | Delta means | Remedy |
|-----|-------------|--------|
| `delete:` | Dead code, unused flexibility, speculative feature shipped | Delete it. Nothing replaces it |
| `stdlib:` | Hand-rolled something the standard library ships | Swap for the named function |
| `native:` | Code or a dependency doing what the platform already does | Swap for the named feature |
| `yagni:` | Abstraction with one implementation, config nobody sets, layer with one caller | Collapse it |
| `shrink:` | Same logic, more lines than needed | Take the shorter form |

`ponytail-audit` reports and applies nothing, so its findings are input to this
gate, not the outcome of it. A rise in `N` is fixed here, in this phase, before
the gate passes — it is not a follow-up ticket, because the moment the context
to fix it cheaply exists is now.

Baseline findings **outside** the code this build touched stay out of scope.
This gate measures what the build added, not what it inherited. Inherited bloat
is a separate piece of work with its own frame.

If the audit returns `Lean already. Ship.`, the gate is met — record it and
move on.

## 4. Stale-code sweep

Invoke `ship-prune`. Its full sweep runs here — orphans, duplicate paths,
uncontracted migrations, rotted markers, zombie dependencies, doc drift.

## 5. Ledger reconciliation

Open `.ship/ledger.md`. Every row with `retire-by` of `prove` or earlier must
be CLOSED, and closing requires evidence: the search you ran showing zero
remaining references to the superseded symbol, quoted. An intention to delete
is not a closed row.

Rows legitimately deferred to `release` — a live mixed-version window, an
external consumer still cutting over — stay OPEN with their reason intact and
are settled at gate 5. Nothing survives past that.

## Gate 4 → 5

All five must hold:

1. Acceptance proof green, with commands quoted, and no *unavailable* line
   silently counted as a pass.
2. Correctness review clear, or every finding fixed and re-proved.
3. Complexity review clear, or every finding deleted or justified in one line.
4. Bloat delta at or below baseline — `N` and `M` not higher than phase 1.
5. Prune sweep clean and no ledger row past due.

If a gate item fails for something inside this phase's scope, fix and re-prove.
If it fails for something outside it — a missing test harness, an external
dependency mid-cutover, a decision that is the user's — stop and report the
blocker and what would clear it. Do not widen scope to get to green, and do not
skip, disable or quarantine a check to make a gate pass: that converts a real
failure into a hidden one and defeats the phase entirely.

## What this phase does not do

No polish after criteria pass. No new tests beyond what acceptance needs. No
refactor prompted by something noticed while reviewing — file it as a slice.
Stop the moment the proof is complete.
