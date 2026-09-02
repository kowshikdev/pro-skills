---
description: Run the end-to-end project pipeline, from planning to deployment, with no stale code
argument-hint: "[frame|map|slice|build|prove|release|prune|status] [what to ship]"
---

Invoke the `ship` skill.

Argument handling:

- **No argument** — start at the phase the project is actually in. Check for
  `.ship/ledger.md` and existing acceptance criteria first; do not restart a
  pipeline that is mid-flight. State which phase you are entering and why.
- **A phase name** (`frame`, `map`, `slice`, `build`, `prove`, `release`,
  `prune`) — enter that phase directly, and say which earlier gates you are
  taking on trust rather than silently assuming them.
- **`status`** — read `.ship/ledger.md` and the phase artifacts, then report in
  three lines: current gate, what is blocking it, next action. Change nothing.
- **Free text** (`/ship add stripe webhooks`) — treat it as the brief and start
  at FRAME.

$ARGUMENTS
