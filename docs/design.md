# spec-gate — design

Status: all 25 decisions in [§10](#10-decisions) are decided; the operator's approval is quoted
there.

## 1. Purpose and the one invariant

When an agent writes the code, the human's product is an executable specification: scenarios
in domain language that run against the product and go red when it is wrong. `spec-gate` is the
runner that makes that specification authoritative.

**Invariant: the generator may not touch the measurement.** The agent that changes the product
must not be able to change the specs, choose which spec version applies, edit the drivers that
observe the product, record evidence, or post the verdict. Every component below exists to hold
this invariant; a design choice that weakens it is a defect.

Grounding, in the KBE (`~/kbe`):

- `sources/2026-10-21-farley-industry-catching-up-ai-code-that-works.md` — the four-layer
  executable specification, red-first, specs out of the agent's reach, mutation testing for spec
  holes ([video](https://www.youtube.com/watch?v=EkH_uoybj8k)).
- `designs/software-factory/10-designing-verifiers.md` — strongest-oracle decomposition and the
  anti-gaming catalog.
- `designs/software-factory/11-testing-the-spec.md` — spec mutation coverage.
- `findings/spec-authoritative-layer-needs-a-gate.md` — a spec→test link nothing checks drifts
  where it matters most.
- `guides/writing-specifications.md` — EARS, Gherkin, witness and near-miss.

## 2. Two repositories per product

| Repository | Owner | Holds | The agent |
|---|---|---|---|
| product | agent and operator | the code; its public protocol | writes |
| spec | operator | requirements, scenarios, DSL, drivers, ledger, judge pin, gate config | reads only |

The product repository holds no gate configuration that counts. Its own CI may run anything as a
pre-check for the agent; the only verdict that gates a merge is the commit status posted by the
spec-side runner, with a credential the agent never holds.

## 3. The spec workspace — four layers

```
spec/
  <product>.req.md
  requirements.toon
  features/*.feature
  crates/dsl
  crates/driver-port
  crates/driver-<channel>
  crates/acceptance
  crates/model
  ledger/
  spec-gate.toml
```

| Layer | Crate | Knows | Must not know |
|---|---|---|---|
| 1 — scenarios | `features/` + `acceptance` step definitions | domain words | protocol, product types |
| 2 — DSL | `dsl` | domain operations, aliases, defaults | protocol |
| 3 — drivers | `driver-port` (trait) + `driver-<channel>` (adapters) | the product's public protocol | product internals |
| 4 — product | product repository | everything | the spec repository |

The dependency graph is the layering: `acceptance → dsl → driver-port ← driver-<channel>`, and
`dsl` depends on no driver adapter. The spec workspace carries `hex-lint` roles from its first
commit, so a layering violation is a build failure.

The DSL has its own vocabulary types; drivers translate them to and from the product's protocol
(D-02). Negative outcomes are typed (D-08). Drivers live in this workspace; an agent may draft a
driver or a DSL operation only as a spec-repository PR the operator approves, never in the task
that changes the product it observes, and each driver carries a self-test against a fake product
that lies, which the driver must detect (D-09, D-11).

Sketch of layers 2 and 3 for a shop product (illustrative):

```rust
pub trait ShopDriver {
    async fn create_basket(&mut self, id: &BasketId) -> Result<(), DriverError>;
    async fn add_item(&mut self, id: &BasketId, sku: &Sku, price: Money) -> Result<(), DriverError>;
    async fn apply_code(&mut self, id: &BasketId, code: &str) -> Result<CodeOutcome, DriverError>;
    async fn total(&mut self, id: &BasketId) -> Result<Money, DriverError>;
}

impl<D: ShopDriver> BasketOps<'_, D> {
    pub async fn add(&mut self, item: &str, price: Money) {
        let sku: Sku = self.shop.aliases.sku(item);
        self.shop.driver.add_item(&self.id, &sku, price).await.expect_ok("add item");
    }
}
```

Aliasing maps a scenario's name (`"book"`) to a value unique to that scenario run
(`book-7f3a`), so scenarios run in parallel against one product instance without colliding.
Each gate run starts one fresh instance (D-03). Test data is set up only through the product's
public protocol (D-04).

## 4. Requirements and binding

Requirements live in `<product>.req.md` in `req` format (`~/code/tools/req`); `req mint` assigns
ids and `req generate` writes `requirements.toon`. Each requirement carries a witness and a
near-miss (`guides/writing-specifications.md`); the near-miss is the run the scenario must be
seen failing on.

Each scenario carries `@covers:<criterion-id>` and an `@id:<scenario-id>` minted once by
`spec-gate mint` with the `hashid` crate; the ledger is keyed by `@id` (D-20). The binding check
(no tag, a tag naming an absent criterion) belongs to `binding-contract`, extended to read
Gherkin tags; `spec-gate` v0 has no binding stage (D-24). Zero scenarios fails the run
(`sg-zf8.2`).

## 5. The three loops

**Author (operator).** Write a requirement and a scenario; record the scenario red against the
current product commit with `spec-gate red`; merge to spec `main`.

**Agent.** Reads a read-only checkout of the spec repository; runs `spec-gate run --sut <path>`;
reads the report; changes the product; repeats. A local run writes nothing to the ledger and
posts nothing.

**Gate (runner).** A product push triggers the spec-side runner. It checks out the product
commit, builds and starts it per `spec-gate.toml`, runs every configured gate from spec `main`,
appends to the ledger, and posts one commit status naming the product commit and the spec
commit.

## 6. Gates

`spec-gate` orchestrates; it reuses existing tools rather than reimplementing them. All
configuration that decides a verdict lives in the spec repository.

| Stage | Tool | Spec-side configuration |
|---|---|---|
| acceptance | `cucumber` harness in `crates/acceptance` | `fail_on_skipped`, suite path |
| red-first | `spec-gate` ledger | `ledger/` |
| binding | `req check`, then `binding-contract` | `requirements.toon` |
| holdout | `spec-gate` | `holdout/`, absent from every agent checkout |
| structure | `hex-lint` on the product | expected role map and exceptions file held in the spec repository |
| purity | `effect-audit` | expected kernel/domain crate set |
| API and wire | `cargo-regime-check`, `contract-suite` | baselines |
| spec holes | `cargo-mutants` + `mutation-triage`, rebuilding and restarting the product per mutant | packages, equivalence ledger |
| properties, model | `hegeltest`, `proptest-state-machine`, `quint-connect` in `crates/model` | — |

A product's `Cargo.toml` role tags are agent-editable; the structure stage compares them against
the spec-side role map and fails on any difference.

Holdout scenarios start once the visible set holds about 20 scenarios. A holdout failure does not
block a merge; it is reported to the operator only, never in the agent-visible status, and the
failing scenario is promoted into the visible set and replaced with a new holdout (D-12, D-13).

The model-based tier in `crates/model` uses the DSL operations as its action alphabet and starts
once about 20 acceptance scenarios pass (D-10).

The runner is a dedicated OS user on a homelab host the agents do not run on; it alone holds the
status credential (D-16). The agent's spec checkout is owned by another user and read-only, and a
`rule-gate` hook refuses agent writes to it at tool-call time (D-22).

## 7. The report

JSON, one entry per scenario, written for an agent's next change:

```json
{
  "product": "<sha>",
  "spec": "<sha>",
  "judge": "<release>@<sha256>",
  "verdict": "fail",
  "failures": [
    {
      "scenario": "A valid discount code reduces the total",
      "covers": ["sg-xxx.1"],
      "step": "Then the basket total is 18.00",
      "expected": "18.00",
      "actual": "20.00",
      "trace": [{ "request": "…", "response": "…" }]
    }
  ]
}
```

A void verdict (build failed, product did not start, judge pin mismatch) reports the cause and
no pass or fail counts.

## 8. Dog-fooding

`spec-gate` is the first product judged by `spec-gate`.

- Product repository: `../spec-gate`. Spec repository: `../spec-gate-specs`.
- Channel: CLI. The driver runs the `spec-gate` binary under test against fixture spec and
  product repositories it creates in a temporary directory, and observes exit codes, the JSON
  report and ledger bytes.
- **The judge is never the commit under test.** A gate that judges itself can pass itself. The
  binary that judges product commit `P` is a released build whose checksum `spec-gate.toml`
  pins (`sg-i86`).
- **Release.** A release is an operator-signed tag `vX.Y.Z` on `spec-gate`. The runner builds it
  from the tag with `cargo build --release --locked` and publishes the binary with its sha256.
- **Promotion.** Moving the `[judge]` pin to a new release is a spec-repository PR, allowed only
  after that release has passed the full suite under the current judge. Each judge is certified
  by its predecessor (D-18).
- **Bootstrap.** The first judge is `v0`: acceptance, red-first and the checksum refusal, nothing
  else. The operator reviews it by hand, runs the seed scenarios with it once, and records the
  result. `v0` is trusted on review alone; it is the one unchecked assumption at the base of the
  chain, kept small on purpose (D-19).
- The `spec-gate` repository's own CI is a pre-check and counts for nothing.

Seed requirements: `../spec-gate-specs/spec-gate.req.md` (7 requirements, 13 criteria). Seed
scenarios: `../spec-gate-specs/features/` (one per criterion).

## 9. Adoption contract for a product

A product judged by `spec-gate` must:

1. Build from a commit with one command, and start with one command plus a readiness probe.
2. Expose the setup operations its scenarios need through its public protocol.
3. Expose a controllable clock through its protocol if any requirement depends on time.
   Asynchrony elsewhere is handled by `eventually(timeout)` assertions in the DSL.
4. Have branch protection requiring the `spec-gate` status context.
5. Adopt in ratchet mode: the first authoritative run's pass count is the floor; a gate moves from
   warning to error when its backlog reaches zero.
6. Drive its specs through the channel each requirement is about; a second driver is added only
   when a requirement names that channel.

A spec repository serves one system, which may comprise several products (D-21).

## 10. Decisions

### Operator decision

> "accept all" — operator, 2026-10-21, in reply to the recommendation for each of D-01 to D-25.

Approves: the 25 decisions in the table below, as worded there. This covers architecture (D-01
to D-11, D-20, D-23), policy (D-12 to D-15, D-19, D-25), infrastructure (D-16, D-18, D-21, D-22)
and two pieces of follow-up work: a `spec-gate mint` command (D-20) and extending
`binding-contract` to read Gherkin tags (D-24). Nothing else is approved: no change to any
existing tool beyond D-24, no gate or lint configuration in `~/code/tools`, and no scope outside
these two repositories.

| Id | Decision | Decided |
|---|---|---|
| D-01 | Scenario form | Gherkin via `cucumber` for acceptance; plain Rust tests for the model tier |
| D-02 | DSL vocabulary | The DSL has its own types; drivers translate. It never imports the product's domain crate |
| D-03 | Test isolation | Aliasing against one shared instance, with one fresh instance per gate run. No reset endpoint |
| D-04 | Test data setup | Through the product's public protocol only |
| D-05 | Time | `eventually(timeout)` assertions everywhere; a controllable clock required only from products with time-dependent requirements. `spec-gate` needs none |
| D-06 | Where assertions live | DSL operations for acceptance; the model tier asserts against the model |
| D-07 | Channels | The channel each requirement is about; a second driver only when a requirement names it. `spec-gate`: CLI |
| D-08 | Negative outcomes | Typed outcomes; a `DriverError` fails as a broken rig |
| D-09 | Who extends the DSL | The agent proposes as a spec-repository PR, the operator approves. Each new operation ships with a scenario using it and a near-miss proving it can fail |
| D-10 | Model tier shares DSL vocabulary | Yes; DSL operations are the action alphabet. Starts once about 20 acceptance scenarios pass |
| D-11 | Where drivers live | In the spec repository. An agent may draft one through the D-09 path, never in the task that changes the product it observes. Each driver carries a self-test against a lying fake product |
| D-12 | Holdout set | Yes, from about 20 visible scenarios; held in a directory no agent checkout contains |
| D-13 | Holdout failure | Flags only, reported to the operator only; the failing scenario is promoted to the visible set and replaced |
| D-14 | Spec sign-off | The operator's merge under branch protection |
| D-15 | Agent drafts scenarios | Yes; a draft counts only after the operator's merge and a recorded red |
| D-16 | Runner host | A dedicated OS user on a homelab host the agents do not run on; it alone holds the status credential |
| D-17 | Mutation through external specs | Rebuild and restart the product per mutant; proven first with a planted mutant the specs must kill |
| D-18 | Judge release and promotion | Operator-signed tag `vX.Y.Z`; runner builds it with `--locked` and publishes binary and sha256. The pin moves by spec-repository PR only after the candidate passes the full suite under the current judge |
| D-19 | Bootstrap | A minimal `v0` (acceptance, red-first, checksum refusal), hand-reviewed by the operator, run once on the seed scenarios with the result recorded; it becomes the first judge |
| D-20 | Scenario identity | An `@id` tag minted once by `spec-gate mint` with the `hashid` crate; the ledger key |
| D-21 | Products per spec repository | One spec repository per system |
| D-22 | Agent writes to the spec checkout | Read-only checkout owned by another user, plus a `rule-gate` hook refusing writes at tool-call time |
| D-23 | BDD crate | `cucumber` 0.23 |
| D-24 | Owner of the binding check | `binding-contract`, extended to read Gherkin tags; `spec-gate` v0 has no binding stage |
| D-25 | Adoption on an existing product | Ratchet from the first authoritative run |

## 11. Non-goals

- A new BDD framework, mutation tool, model checker or prover.
- An LLM judge on any correctness criterion.
- Hosting or multi-tenant service; `spec-gate` is a CLI plus a runner.

## 12. Flagged and unverified

- `binding-contract`'s `@covers:` tag syntax is not fixed in its repository; `@covers:<id>` here
  is an assumption.
- With `cargo mutants --test-package acceptance`, the product binary may not be rebuilt per
  mutant, so every mutant survives. Unproven either way until D-17's planted-mutant check runs.
- `cucumber` reports an undefined step as skipped; `fail_on_skipped` is required and must be seen
  failing once on a deliberately undefined step.
- Holdout sets as practised by StrongDM are known to this design second-hand only.
- Whether `hegeltest` supports stateful testing is unchecked.
- The seed scenarios carry no `@id` tags: `spec-gate mint` (D-20) does not exist yet, and ids are
  never written by hand.
- `[judge] sha256` in `spec-gate-specs/spec-gate.toml` is `PENDING` until `v0` is built and
  reviewed (D-19).
