# pro-skills

**proship** — an end-to-end project pipeline for Claude Code. It wraps the
[`caveman`](https://github.com/JuliusBrussee/caveman) and
[`ponytail`](https://github.com/DietrichGebert/ponytail) skill families into six
gated phases from planning to deployment, and enforces a replacement ledger so
no superseded code survives a release.

## Why a wrapper

Between them, caveman and ponytail cover 26 skills and 3 sub-agents. They are
strong and they compose — but they cover the *middle* of a project:

- **ponytail** governs *how much* gets built (YAGNI, stdlib first, one line
  over fifty).
- **caveman** governs *how a change is executed and reported* (workflow
  discipline per change type, compressed output, delegated reads).

Neither has a planning phase, a release phase, or any mechanism that stops a
superseded implementation from living on next to its replacement. `proship`
adds those three and routes everything else to the skill that already does it
better.

## Install

```bash
/plugin marketplace add kowshikdev/pro-skills
/plugin install proship@pro-skills
```

Then install the wrapped families — `proship` routes to them:

```bash
/plugin marketplace add JuliusBrussee/caveman
/plugin install caveman@caveman

/plugin marketplace add DietrichGebert/ponytail
/plugin install ponytail@ponytail
```

Both are optional. Each routing entry has an inline fallback, so `proship`
degrades in quality, never in shape — it will say once which skill would have
run.

## Use

```bash
/ship add stripe webhooks     # start at FRAME with a brief
/ship                          # resume at whatever phase the project is in
/ship build                    # enter a phase directly
/ship prune                    # stale-code sweep on demand
/ship status                   # current gate, blocker, next action
```

Or just describe the work — the skill triggers on "build this end to end",
"take this to production", "plan to deployment", "with no leftover code".

## The pipeline

| # | Phase | Produces | Gate to pass |
|---|-------|----------|--------------|
| 0 | FRAME | Acceptance conditions, explicit non-goals | Every acceptance line is observable and falsifiable |
| 1 | MAP | Prior art, reuse seams, constraints | Every capability marked reuse / extend / build, with `path:line` for the first two |
| 2 | SLICE | Ordered thin vertical slices | Slice 1 is end-to-end runnable; each slice has its own acceptance subset |
| 3 | BUILD | Working slices + ledger rows | Slice acceptance passes; every superseding change has a ledger row |
| 4 | PROVE | Proof set, two reviews, prune sweep | Proof green, both reviews clear, no ledger row past due |
| 5 | RELEASE | Reversible rollout | Rollback exercised, ledger fully closed, observability live |

Phases 0–2 shrink the build before it starts. That is where most of the
efficiency comes from — the cheapest code remains the code the MAP phase
found already written.

## No stale code

The hard guarantee, and the reason the pipeline has a file instead of a habit.

Anything that supersedes existing behavior gets a row in `.ship/ledger.md` the
moment it is written:

```
| id | added | supersedes | retire-by | status |
|----|-------|------------|-----------|--------|
| R1 | src/auth/session.ts | src/auth/legacy.ts, useOldAuth() | slice-2 | CLOSED |
```

- **Delete in the same change.** `retire-by` defaults to the slice that created
  the row.
- **Deferral needs a written constraint** — a mixed-version window, an external
  consumer mid-cutover. Preference is not a constraint, and there is nothing
  after `release`.
- **Closing needs proof**: the search showing zero remaining references,
  quoted. Intent does not close a row.
- **Gate 5 fails while any row is OPEN.** "We'll clean it up later" is the
  mechanism that produces stale code, so the gate removes the option.

`ship-prune` runs the full sweep at gates 3→4 and 4→5: orphans, duplicate
implementations, uncontracted migrations, rotted markers, zombie dependencies,
doc drift.

## Skills

| Skill | Phase | Role |
|-------|-------|------|
| `ship` | all | Orchestrator, routing table, gate definitions |
| `ship-plan` | 0–2 | Frame acceptance, map prior art, cut thin slices |
| `ship-build` | 3 | Route each change to the right workflow skill; write ledger rows |
| `ship-verify` | 4 | Smallest sufficient proof, two reviews, prune sweep |
| `ship-release` | 5 | Reversible rollout, ledger closure, commit, observability |
| `ship-prune` | gates | Stale-code sweep and ledger enforcement |

References: [`routing.md`](skills/ship/references/routing.md) maps all 26
wrapped skills to phases with per-skill fallbacks;
[`ledger.md`](skills/ship/references/ledger.md) specifies the ledger format and
gate behavior.

## Routing at a glance

| Situation | Routes to |
|-----------|-----------|
| Cause of a failure is unknown | `investigate-first` — before any edit |
| New behavior, feature, integration | `lean-build` |
| Bug or small behavior change | `surgical-patch` |
| Restructure, behavior preserved | `safe-refactor` |
| Schema, API, config, dependency | `migration` |
| Validation only | `verify-and-stop` |
| Cold start or failed search | `caveman-explore`, `cavecrew-investigator` |
| Bounded 1–2 file edit | `cavecrew-builder` |
| Correctness review | `caveman-review`, `cavecrew-reviewer` |
| Over-engineering review | `ponytail-review` |
| Repo-wide bloat hunt | `ponytail-audit` |
| Deferred shortcuts owed | `ponytail-debt` |
| Commit message | `caveman-commit` |

Two reviews at gate 4 is deliberate: `caveman-review` asks *is it correct*,
`ponytail-review` asks *should it exist*. Neither answers the other's question.

## Token discipline

Optimal means cheap as well as correct. Reads fan out to haiku sub-agents and
return `path:line` only; reviews fan out too; prose stays in `caveman` mode and
code stays in `ponytail` mode. Main context is the scarce resource.

## License

MIT. `caveman` and `ponytail` are separate projects under their own licenses.
