# AGENTS.md — spec-gate (product repository)

`spec-gate` judges a product against an executable specification held in a separate spec
repository. This repository is the product. Its spec repository is `../spec-gate-specs`.

## The invariant

The agent that changes this repository may not touch the measurement.

- Never write to `../spec-gate-specs`. Read it, run its acceptance harness against your build,
  and report a spec you believe is wrong; never work around it.
- Never write ledger entries, judge pins or verdicts by hand.
- This repository's own CI is a pre-check. Only the verdict from `../spec-gate-specs` counts.

## Read first

1. `docs/design.md`: the system and decisions D-01 to D-25 (§10).
2. The epic you are working: `br show <epic>`. Its **Operator decision** section is your only
   source of authority for anything not in §10.
3. `~/kbe/guides/rust-native-architecture-template.md`.
4. `~/.claude/CLAUDE.md`: house rules, cited by id (`cm-*`).

## Beads

Beads live in `.beads/` (`br`). Your beads are labelled `side:product`. Work them one at a
time in `br ready` order.

1. `br show <id>`; confirm the status is `open` before claiming (cm-2tb); `br update <id> --claim`.
2. Work on a branch named `<id>`. Never commit to `main`; the operator merges with `--no-ff` or a
   fast-forward, never a squash.
3. When every acceptance criterion is proven (cm-nvh), leave the bead `in_progress` with a note:
   the branch, its tip, the evidence, any SHA the bead asks for, and that it awaits the operator
   merge. It closes after the merge, with the merge commit in the close reason. If a criterion is
   unproven, label the bead `unproven` and record the outstanding half in the notes.
4. A policy choice not covered by §10 or the epic's Operator decision: stop and ask (cm-vph).

## Code

- Functional core, effects behind ports (cm-rlh, cm-hkg). `hex-lint` roles on every crate.
- Lifecycles with more than two states are sum types with a pure transition function and
  exhaustive matches; no wildcard arm (cm-prk).
- No code comments; names and types carry the what, `docs/` carries the why (cm-k7c).
- Explicit type annotations on signatures, fields, constants and non-obvious locals (cm-vmv).

## Tests

- Entities: `hegeltest` properties (cm-eqj).
- Use cases: nextest unit tests against fake adapters at the ports (cm-wah).
- Boundary: a few integration tests (cm-33k).
- Zero, one and many (cm-srn); no loops or branches in a test (cm-o9o).
- Every new gate or check is seen red once on a planted violation, then green (cm-yzv).

## Before closing a bead

1. The workspace gate command passes.
2. The bead's scenarios pass when `../spec-gate-specs`'s harness runs against your build.
3. `cargo mutants` on the changed code; survivors triaged with `mutants-triage` from
   `~/code/tools/mutation-triage`, which is not on PATH: install it per that README's Install
   section (cm-1qk).

Every cargo call gets a 900000 ms timeout (cm-lpb). Only one agent on this machine runs cargo
at a time (cm-e51).

## Report

Per bead:
- branch and commits;
- each acceptance criterion, with the command that proved it and its result;
- mutation summary: caught, missed, equivalent;
- open questions.
