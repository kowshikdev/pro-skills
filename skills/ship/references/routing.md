# Full routing table

Every skill and agent across the two wrapped repositories, the phase it belongs
to, and what to do instead when it is not installed.

Sources:
- `caveman` — https://github.com/JuliusBrussee/caveman (20 skills, 3 agents)
- `ponytail` — https://github.com/DietrichGebert/ponytail (6 skills)

## Ambient — on for the whole pipeline

| Skill | Repo | Role | Fallback if absent |
|-------|------|------|--------------------|
| `ponytail` | ponytail | YAGNI ladder on every build decision. Levels lite/full/ultra | Apply the ladder inline: needs to exist → in repo → stdlib → native → installed dep → one line → minimum that works |
| `caveman` | caveman | Compressed output. Levels lite/full/ultra | Drop hedging, preamble and restatement; keep identifiers and numbers exact |

## Phase 0 — FRAME

| Skill | Repo | Role | Fallback if absent |
|-------|------|------|--------------------|
| `caveman-compress` | caveman | Compress a long brief, `CLAUDE.md` or plan file, keeping a readable backup | Summarize to bullets; keep the original file untouched |

## Phase 1 — MAP

| Skill / agent | Repo | Role | Fallback if absent |
|---------------|------|------|--------------------|
| `caveman-explore` | caveman | Read-only cold-start orientation, returns `path:line` only. Skip when the file is already named | `Glob` + `Grep` directly, reporting `path:line` and nothing else |
| `cavecrew-investigator` | caveman (agent) | Read-only locator: where is X, what calls Y, map this directory | `Explore` agent, or inline `Grep` with `-n` |
| `cavecrew` | caveman | Decides when delegating beats working inline | Delegate when the read is broad or exploratory; inline when the target is named |
| `ponytail-audit` | ponytail | Repo-wide ranked bloat report, tagged `delete:` `stdlib:` `native:` `yagni:` `shrink:`, ending in `net: -<N> lines, -<M> deps possible.` Record that number — it is the phase 4 gate metric | Grep for duplicate helpers, unused exports, single-caller layers and wrapper-only modules; count them for a crude baseline |
| `ponytail-debt` | ponytail | Harvests `ponytail:` markers into a debt ledger | `grep -rn "ponytail:\|TODO\|FIXME\|HACK"` and tabulate |

## Phase 2 — SLICE

No dedicated skill. `ponytail` governs: each slice is the minimum that produces
observable value end to end. If a slice touches 3+ files for one behavior,
suspect it is two slices.

## Phase 3 — BUILD

Route by change shape. These are mutually exclusive per change — pick one.

| Skill / agent | Repo | Use when | Fallback if absent |
|---------------|------|----------|--------------------|
| `investigate-first` | caveman | Cause unknown, intermittent, or a performance regression. Runs *before* any edit | Separate symptom from cause, rank hypotheses by evidence, do not edit until one mechanism explains the evidence |
| `lean-build` | caveman | New behavior, product slice, integration | Derive acceptance and non-goals, trace the entry point through owning layers, deliver one coherent path, stop when acceptance passes |
| `surgical-patch` | caveman | Bug or small behavior change | Reproduce first, change the narrowest layer that owns the behavior, add only regression proof |
| `safe-refactor` | caveman | Restructure with behavior preserved | Establish verification first, move one ownership boundary at a time, keep intermediate states buildable |
| `migration` | caveman | Schema, data, API, protocol, config or dependency transition | Define forward and rollback paths, sequence expand → migrate → verify → contract, keep mixed-version operation safe |
| `cavecrew-builder` | caveman (agent) | Bounded 1-2 file edit. Refuses 3+ files | Edit inline, but treat a 3+ file spill as a signal the slice is too wide |

## Phase 4 — PROVE

| Skill / agent | Repo | Role | Fallback if absent |
|---------------|------|------|--------------------|
| `verify-and-stop` | caveman | Smallest sufficient proof set; distinguishes pass / fail / unavailable / blocked | Run focused checks before wide gates, quote commands and results, add nothing after criteria pass |
| `caveman-review` | caveman | Correctness review, one line per finding with location, problem, fix | Review the diff for correctness only, one line per finding |
| `cavecrew-reviewer` | caveman (agent) | Same, delegated and severity-tagged | Use the `code-review` skill if present |
| `ponytail-review` | ponytail | Over-engineering review of the diff, same five tags as the audit | Ask of each added symbol: what breaks if this is deleted? No answer means delete it |
| `ponytail-audit` | ponytail | Re-run for the **bloat delta**: `N` and `M` must not exceed the phase 1 baseline. A build leaving more cuttable code than it found added bloat, whatever the tests say | Re-count the phase 1 proxy and compare |

## Phase 5 — RELEASE

| Skill | Repo | Role | Fallback if absent |
|-------|------|------|--------------------|
| `migration` | caveman | Rollout sequencing and rollback for anything stateful | As above |
| `caveman-commit` | caveman | Conventional Commits compressed to intent | Conventional Commits, subject states intent not mechanics |

## Post-release — cost and observability

Optional. Route here only when the user asks about LLM spend, or when the
project itself calls an LLM in production. All six touch an external service
(Caveman Cloud); the three marked **consent** change state and must not run
without explicit approval.

| Skill | Repo | Role |
|-------|------|------|
| `caveman-setup` | caveman | Wire the repo through the gateway so requests are measured, no behavior change |
| `caveman-discover` | caveman | Find and label LLM workflows so spend groups by workflow |
| `caveman-evidence-review` | caveman | Read-only: cost, Cave Score, traces, latency, errors, routing, savings |
| `caveman-learn` | caveman | **consent** — act on a learn report, apply cost-lowering fixes per-edit |
| `caveman-optimize` | caveman | **consent** — turn an observation into a candidate with paired baseline |
| `caveman-manage` | caveman | **consent** — start, approve, cancel, promote or roll back an experiment |

## Meta

| Skill | Repo | Role |
|-------|------|------|
| `caveman-stats` | caveman | Real token usage and estimated savings for the session |
| `ponytail-gain` | ponytail | Benchmark medians: less code, less cost, more speed |
| `caveman-help` / `ponytail-help` | both | Quick-reference cards |

## Precedence when two skills could fire

1. `investigate-first` outranks every build skill when the cause is unknown.
   Editing on a guess is the most expensive thing in the pipeline.
2. `migration` outranks `lean-build` and `surgical-patch` when the change
   crosses a persistence, API or dependency boundary — reversibility wins.
3. `safe-refactor` and any feature skill never run in the same change. Split.
4. `verify-and-stop` outranks everything at gate 4: it may not edit product
   code, which is what makes it a gate rather than another build step.
