# spec-gate

Runs a product's executable specification from a separate spec repository and posts the verdict
as a commit status the product repository cannot influence. Agents implement against the specs;
they cannot edit them, choose their version, or record evidence.

- Judges product commit `P` against spec `main`, and names both in the verdict.
- Counts a scenario only after the ledger has seen it fail.
- Fails on skipped steps, untagged scenarios and dangling `@covers:` tags.
- Reports each failure with step, expected, actual and driver trace, for the agent's next change.
- Local runs give the same report and record nothing.

`spec-gate` is judged by its own spec repository, [`../spec-gate-specs`](../spec-gate-specs), using
a released build. This repository's own CI is a fast pre-check and counts for nothing.

Design and open decisions: [`docs/design.md`](docs/design.md).
