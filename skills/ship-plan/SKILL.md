---
name: ship-plan
description: >
  Phases 0-2 of the ship pipeline: frame acceptance and non-goals, map what
  already exists before building anything new, and cut the work into thin
  end-to-end slices. Use when starting a project or feature, when asked to plan
  before building, when a request arrives vague or large, or when the ship
  pipeline reaches FRAME, MAP or SLICE. Produces a plan whose default answer to
  every capability is reuse, not build.
---

# Ship: plan

Three phases before a line of product code. Their combined job is to shrink the
build, not describe it.

## Phase 0 — FRAME

Write acceptance conditions and non-goals. Nothing else.

- Each acceptance line is **observable and falsifiable**: a command, a
  response, a state change someone can check. "Works reliably" is not a line;
  "POST /claims with an expired token returns 401" is.
- Non-goals are as load-bearing as goals. Anything the request implies but does
  not require goes here explicitly, because unstated non-goals get built.
- Ambiguity that changes the build gets asked now. Ambiguity that does not gets
  an assumption recorded on the line it affects.
- If the brief or `CLAUDE.md` is long-lived, run it through `caveman-compress`.

**Gate 0 → 1:** every acceptance line names how it would be checked.

## Phase 1 — MAP

Find what already does this. This is the phase that decides the size of the
build, so it is the phase people skip and should not.

- Orient with `caveman-explore` or `cavecrew-investigator` — delegated reads,
  `path:line` back, main context untouched. If the exact file is already known,
  read it directly; delegation is not free.
- Run `ponytail-audit` and **record its baseline**. It ends with a measurable
  line — `net: -<N> lines, -<M> deps possible.` — and that number is the gate
  metric phase 4 compares against. Save it plus the findings to
  `.ship/audit-baseline.md`. Without it there is nothing to compare to later,
  and "we didn't make it worse" stays an opinion.
- Findings that land **in the code this build will touch** are pre-work, not
  backlog. Settling a `stdlib:` or `yagni:` finding in a file you are about to
  edit is cheaper now than building on top of it and cheaper than routing
  around it. Findings elsewhere in the tree stay in the baseline, untouched —
  they are not this build's scope.
- Run `ponytail-debt` for shortcuts already owed. A `ponytail:` marker in code
  you are about to touch names a ceiling someone already hit; read it before
  you design against it.
- Trace the entry point through the layers that own the invariants. A plan that
  does not know which layer owns what will produce a patch in the wrong place.

Mark every capability in the frame as exactly one of:

| Mark | Means | Must carry |
|------|-------|------------|
| **reuse** | Exists, fits as-is | `path:line` |
| **extend** | Exists, needs a seam widened | `path:line` + what changes |
| **build** | Genuinely absent | Why the three above failed |

**Gate 1 → 2:** no capability is marked *build* until reuse and extend have
been checked and named. A plan of all-build is a plan that skipped this phase.

## Phase 2 — SLICE

Cut into thin vertical slices, ordered.

- A slice is **end-to-end**: it crosses every layer it needs and produces
  observable value on its own. Horizontal slices ("all the models", "all the
  endpoints") defer integration risk to the end, which is where it is most
  expensive.
- Slice 1 is runnable. If nothing runs until slice 4, re-cut.
- Each slice inherits its own subset of the acceptance lines. A slice with no
  acceptance line is not a slice — it is scope creep with a plan around it.
- Order by risk, not by comfort: the slice most likely to invalidate the design
  goes first.
- Every slice that replaces existing behavior gets its `.ship/ledger.md` row
  written **now**, at plan time, with `retire-by` defaulting to that same slice.
  Deciding what dies while planning is what prevents the parallel-implementation
  problem later.

**Gate 2 → 3:** slice 1 is end-to-end runnable, every slice has acceptance, and
every replacement has a ledger row.

## What this phase does not do

No product code. No scaffolding, no "while I'm here" setup, no directory trees
created in advance of a slice needing them. Hand off to `ship-build` with the
frame, the map and the slice list, and stop.
