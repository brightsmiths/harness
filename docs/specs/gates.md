# Gates

Gate definitions tell gate-warden which checks make up each gate G1–G9 and the thresholds per
tier (handbook `docs/architecture/workflow.md`, gate catalogue; handbook ADR-0004). gate-warden
is the only source of verdicts. The harness only says *what* to run. The format is in
[schemas.md](schemas.md#gate-definition), and checks report in the
[gate result format](schemas.md#gate-result).

## Principles

1. **Checks are commands.** Every check is an argv array. Its output is turned into the gate
   result format by a gate-warden adapter (`exit-code`, `junit-xml`, `sarif`) or emitted
   directly (`native`). The harness never contains adapter code.
2. **Gate config is never read from the PR being judged.** gate-warden uses the harness version
   in the target repository's default-branch pin.
3. **Missing means fail.** A check that does not run is `skipped`, and `skipped` fails the gate
   unless the definition marks the check optional for that tier. A gate with no definition for
   the tier is `GATE_UNDEFINED` and never passes.
4. **Coarse feedback to agents.** Only `summary` and `requirement_ids` from check results reach
   zone C. Holdout results are reduced to requirement IDs.
5. **Idempotent.** Same (SHA, gate, harness version) gives the same check runs (handbook
   `docs/architecture/state-model.md`).

## G6-lite (L1)

The reduced build gate for the M1 pilot (T1, Rust). File:
`disciplines/software/gates/G6-lite.yaml`.

| # | Check id | Command | Adapter | Needs | T1 threshold |
| --- | --- | --- | --- | --- | --- |
| 1 | `scope-check` | `agent-gate scope-check --roles <pinned roles.yaml> --range <base>..<head>` | native | — | 0 out-of-scope paths |
| 2 | `oracle-verify` | `oracle-guard verify` (manifest signature, file hashes, config guard) | native | — | pass |
| 3 | `build` | `cargo build --locked --workspace --all-targets` | exit-code | 1, 2 | exit 0 |
| 4 | `fmt` | `cargo fmt --all --check` | exit-code | — | exit 0 |
| 5 | `clippy` | `cargo clippy --locked --workspace --all-targets -- -D warnings` | exit-code (sarif when available) | 3 | 0 warnings |
| 6 | `nextest` | `cargo nextest run --locked --workspace --profile ci` | junit-xml | 3 | 0 failed, 0 skipped |
| 7 | `oracle-inventory` | `oracle-guard inventory --junit <nextest junit>` | native | 6 | every expected test ran and passed; none skipped; no extra locked tests |

Notes:

- `scope-check` and `oracle-verify` run first because they are cheap. If they fail, the rest is
  `skipped` and the gate fails.
- `nextest` failures that come only from the builder's `tests/own/` tests do **not** fail the
  gate. The gate definition passes `--profile ci` and the pack's `.config/nextest.toml` (an
  oracle-owned file covered by the config guard) defines the `ci` profile to exclude the `own`
  binary from pass/fail. Own tests still count toward mutation score from L3. *Alternative if
  nextest filtering proves fragile:* a separate optional `nextest-own` check.
- G6-lite has no sniff check, mutation, ratchet, SAST, secret scan or commit-gate. Those come in
  full G6 (L3) as their tools exist (handbook roadmap M2–M3).

## Gates by level

| Gate | Level | Checks (software pack) | Tiers |
| --- | --- | --- | --- |
| G6-lite | L1 | As above | t0, t1 |
| G1 Specify | L2 | Spec schema valid; criteria in EARS form (lint); requirement IDs unique; no open questions marked blocking; upstream attestations verified (gate-warden built-in) | t1+ |
| G3 Contracts | L2 | Stubs compile (`cargo check` on `spec/contracts`, see [assumptions.md](assumptions.md)); `cargo-semver-checks` for breaking changes; architecture rules (`cargo-deny`, crate-boundary check) on stubs | t1+ |
| G4 Plan | L2 | `plan.yaml` schema valid; DAG acyclic; each slice within its size budget; 100% requirement coverage | t1+ |
| G5 Oracles | L2 (added; see [assumptions.md](assumptions.md)) | `oracle-guard red-check` (every new test fails against stubs); requirement trace 100%; manifest signed | t1+ |
| G2 Architecture | L3 | ADR template complete; threat-model delta present; risk-map patch valid | t2+ (t1 deterministic only) |
| G6 full | L3 | G6-lite plus `oracle-guard sniff`, holdout injection and run, mutation on changed code (`cargo-mutants`), property tests with fresh seed, fuzz smoke (t2+), `ratchet`, Semgrep, gitleaks, `cargo-deny` licences and advisories, commit-gate, todo-gate | t2 (t1 subset) |
| G7 Refactor | L3 | All G6; ratchet flat or improving; no locked-test change; seam changes rejected | t1+ |
| G8 Assurance | L3 | No open blocking findings; every dispute decided | t2+ (t1 advisory) |
| G9 Release | — | Owned by release-ratifier; not defined in the harness unless release-ratifier requests it | — |

## Thresholds per tier

Initial values. Each change is a minor harness release and goes through evals (L4).

| Metric | T0 | T1 | T2 (L3) | T3 (L4) |
| --- | --- | --- | --- | --- |
| Out-of-scope paths | 0 | 0 | 0 | 0 |
| Clippy warnings | 0 | 0 | 0 | 0 |
| Failed or skipped expected tests | 0 | 0 | 0 | 0 |
| Mutation score on changed code | — | — | ≥ 0.70 | ≥ 0.85 |
| Holdout ratio | — | — | ≥ 0.20 | ≥ 0.30 |
| Fuzz smoke | — | — | 60 s, 0 crashes | 600 s, 0 crashes |

## Holdout configuration (L3)

Policy only, in `disciplines/software/gates/holdouts.yaml`:

- minimum ratio per tier;
- injection method (`oracle-guard holdout-inject`: copy into `tests/`, or replace a locked
  placeholder behind `#[cfg(test)] mod __holdout;`);
- feedback granularity (`requirement_id`);
- maximum attempts before rotation.

Holdout tests themselves live in a private repository per deployment (handbook open question 6).
