---
name: ship-release
description: >
  Phase 5 of the ship pipeline: reversible release. Sequences rollout through
  the migration skill, verifies the rollback path actually works before it is
  needed, writes the commit via caveman-commit, confirms observability, and
  makes a final ledger check that no superseded code is shipping. Use when
  deploying, releasing, cutting over, or when the ship pipeline reaches
  RELEASE.
---

# Ship: release

Reversibility first. A release you cannot undo is a bet, and the phase's job is
to make sure it is not one.

## 1. Rollback before rollout

Run `migration` for anything stateful — schema, data, API contract, protocol,
config, dependency. Its sequencing is the rule here:

- Define the forward path **and** the rollback path before executing either.
- **Exercise the rollback.** A rollback path that has never been run is a
  hypothesis, not a safety net. This is the step most often skipped and the one
  that matters most at 3am.
- Sequence expand → migrate → verify → contract. Contract is destructive and
  separately authorized; never fold it into the same step implicitly.
- Keep mixed-version operation safe for the whole window in which old and new
  can both be live.
- Make retries idempotent and partial failure observable.

## 2. Final ledger check

Open `.ship/ledger.md`. **Every row must be CLOSED.** This is the hard stop of
the pipeline: rows deferred at gate 4 for a mixed-version window are settled
here, once the window closes.

A row that cannot be closed because an external consumer still depends on the
old path is not a pass — it is a contract step owed. Say so explicitly, name
the consumer, and record when it can be closed. Shipping with an OPEN row means
shipping stale code, which is the one outcome this pipeline promises against.

Then confirm the contract holds in the tree, not just in the file:

- The superseded paths are actually deleted, not merely unreferenced.
- Dependencies added for abandoned approaches are removed from the manifest
  *and* the lockfile.
- Feature flags used only to gate this cutover have a removal date, or go now.

## 3. Commit

`caveman-commit` — Conventional Commits, compressed to intent. The subject says
what changed for a user of the code, not which files moved. Deletions are part
of the change, not a follow-up commit: the replacement and the removal ship
together, which is what makes the history honest about what happened.

## 4. Observability

Before calling it released, confirm you can tell whether it worked:

- The acceptance conditions from phase 0 are checkable **in the deployed
  environment**, not only in tests. If nothing there can answer "is it working",
  the release is unverifiable and the gate is not met.
- Errors surface somewhere a person will see them.
- If the project calls an LLM in production, route to `caveman-setup` to put
  requests behind the measuring gateway, then `caveman-discover` to label
  workflows so spend groups by workflow rather than one bucket.

## 5. Post-release

Optional, and only when the user asks or the project's own LLM spend is in
question: `caveman-evidence-review` for read-only cost, trace, latency and
routing evidence. `caveman-learn`, `caveman-optimize` and `caveman-manage`
change state and need explicit approval before each run — never invoke them to
"check something".

## Release gate

Released means: rollout done, rollback exercised, ledger fully closed,
acceptance checkable in the deployed environment, commit written.

Report what shipped, what was deleted, what rolled back if anything, and any
contract step still owed with its date. Then stop — the pipeline is complete,
and follow-on ideas noticed along the way start a new frame, not an extension
of this one.
