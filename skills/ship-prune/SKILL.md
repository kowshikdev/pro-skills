---
name: ship-prune
description: >
  Stale-code sweep and replacement-ledger enforcement. Finds orphaned code,
  duplicate implementations, uncontracted migrations, rotted TODO and ponytail
  markers, zombie dependencies and documentation drift, then closes ledger rows
  with proof. Use at ship pipeline gates, or when asked to find dead code,
  remove leftovers, clean up after a refactor or migration, check for
  duplicate implementations, or answer "is anything stale in here".
---

# Ship: prune

Stale code is not a tidiness problem. It is code that lies: it looks live, gets
read, gets imported by accident, and gets maintained. This skill removes it and
proves it is gone.

Six checks. Run all six at a gate; run the relevant ones on demand.

## 1. Orphans — reachable from nothing

Every symbol added or touched should be reachable from an entry point: a route,
a CLI command, a test, an exported public API, a scheduled job.

```
grep -rn "<symbol>" --include="*.<ext>" . | grep -v "<definition-file>"
```

Zero hits outside its own definition means orphaned. Delete it.

Prefer the language's own tooling where it exists and is honest — `ts-prune`,
`vulture`, `cargo +nightly udeps`, `go vet`, `knip`, the compiler's unused
warnings. Treat their output as **candidates**, not verdicts: reflection,
dynamic imports, DI containers, plugin registries and framework conventions all
create real references these tools cannot see. Confirm each before deleting.

## 2. Duplicates — two ways to do one thing

The most damaging category, because both paths look alive and drift apart.

- Two functions whose bodies differ only in naming or argument order.
- A `_v2`, `_new`, `_old`, `_legacy`, `.bak`, `.orig` sibling of a live module.
- A local helper that reimplements something the repo, the stdlib, or an
  installed dependency already provides.

For each: pick the surviving path, migrate the callers, delete the other. Two
implementations of one behavior is a bug that has not fired yet.

## 3. Uncontracted migrations — expand without contract

`migration` sequences expand → migrate → verify → contract. The contract step
is destructive and separately authorized, which is exactly why it gets left
undone and forgotten.

Look for: columns written but never read, dual-write paths whose old side has
no readers, compatibility shims past their window, API versions with no
remaining callers, feature flags permanently on.

Each one is a contract step **owed**. Either execute it now with authorization,
or record it in the ledger with a date. Not both silent.

## 4. Rotted markers

```
grep -rn "ponytail:\|TODO\|FIXME\|HACK\|XXX\|@deprecated" --include="*.<ext>" .
```

Run `ponytail-debt` if installed — it harvests `ponytail:` markers into a
proper ledger. Every marker in code this pipeline touched gets resolved: done,
scheduled with an owner, or deleted along with the assumption behind it. A
marker nobody will act on is a comment pretending to be a plan.

## 5. Zombie dependencies

Dependencies added for an approach that was abandoned mid-build, or made
redundant by the ladder.

- Every entry in the manifest has at least one real import.
- Removals happen in the manifest **and** the lockfile, regenerated with the
  project's own tooling — never hand-edited.
- Transitive weight matters: a dependency pulled in for one helper that the
  stdlib provides is a `ponytail-review` finding, not a preference.

## 6. Documentation and config drift

Prose and config go stale silently, and are the last place anyone looks.

- README, docs and comments describing removed behavior or renamed symbols.
- `.env.example` keys nothing reads; config keys with no consumer.
- Dead CI steps, scripts referencing deleted paths.
- Examples that no longer run.

## Ledger enforcement

`.ship/ledger.md` is the pipeline's record of what each change replaced:

```
| id | added | supersedes | retire-by | status |
|----|-------|------------|-----------|--------|
| R1 | src/auth/session.ts | src/auth/legacy.ts, useOldAuth() | slice-2 | CLOSED |
| R2 | api/v2/claims.py | api/v1/claims.py | release | OPEN — mobile client cutover |
```

Rules:

- A row is created the moment a change supersedes existing behavior — at plan
  time when known, at build time otherwise. Never retroactively.
- `retire-by` defaults to the slice that created it. Later needs a written
  reason in the row. There is nothing after `release`.
- **CLOSED requires proof**: the search showing zero remaining references,
  quoted. Intent does not close a row.
- Gate 4 fails on any row due at `prove` or earlier still OPEN.
- Gate 5 fails on **any** OPEN row.

## Reporting

One line per finding, most severe first:

```
path:line: <category>: <what is stale>. <what replaces it or why it goes>.
```

Then the deletions you made and the ledger rows you closed with their proof.

If a sweep is clean, say so in one line. Do not manufacture findings to look
thorough, and do not delete on suspicion — every deletion in this skill is
backed by a search you ran and can quote.
