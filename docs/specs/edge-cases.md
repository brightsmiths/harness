# Edge cases and failure modes

How harness content and its consumers behave when things are not normal. The general rule is
**fail closed**: when in doubt, stop and escalate. Never fall back to a weaker rule.

## Versions and pins

| Case | Expected behaviour |
| --- | --- |
| Pinned tag moved or deleted | `PIN_INVALID` (SHA mismatch) or `HARNESS_VERSION_UNAVAILABLE`; no run. Tag protection should prevent this |
| Pinned version withdrawn | `HARNESS_VERSION_WITHDRAWN` for new runs; recorded verdicts stand ([operations.md](operations.md#withdrawal)) |
| Pin names a schema major the consumer does not support | `SCHEMA_MAJOR_UNSUPPORTED`; the consumer is upgraded first, or the pin stays |
| Pin changed in a PR | Ignored for that PR's gates; default-branch pin applies |
| No pin in the target repository | The factory does not run on that repository |
| Pin for a pack not present in the version | `PROFILE_NOT_FOUND` at the first stage |
| Harness repository before its first release | Its own PRs have no harness-driven gates; humans run checks and H8 reviews |
| Upgrading pin mid-run | The run continues on the version recorded in its first stage attestation; the next run uses the new pin. *To be confirmed with CogWorks* ([assumptions.md](assumptions.md), AS-10) |

## Profiles and graphs

| Case | Expected behaviour |
| --- | --- |
| No profile for `(stage type, discipline, toolchain, tier)` | `PROFILE_NOT_FOUND`; stage halts; escalate |
| Tier variant missing (e.g. no `implement.t3`) | Unqualified `implement` is used per [ADR-0009](../adr/ADR-0009-profile-naming-and-resolution.md). Allowed, because a tier's strictness comes from its graph and gates |
| Stage contract role missing from `roles.yaml` | `ROLE_UNDEFINED`; CI should prevent it |
| Graph references a gate not defined for the tier | `GATE_UNDEFINED`; CI should prevent it |
| Change touches paths of two tiers | Highest tier wins (risk map) |
| Change touches paths of two disciplines (pin overrides) | Out of scope until the pack interface (L4). At L1–L3 a run has one discipline, and mixed-discipline changes escalate |
| Agent output fails its output schema | Counts as a failed attempt; retry within `max_attempts` |
| Agent returns `status: injection-suspected` | Stage halts; issue goes to `factory:security-hold` (CogWorks) |
| Agent returns `decisions_flagged` with `status: blocked` | Escalate (H4); no retry |
| Same failure class repeats | Early escalation after `escalate_on_repeated_failure_class` attempts |

## Gates and checks

| Case | Expected behaviour |
| --- | --- |
| Check tool missing from the gate sandbox image | Check `error`; gate `error`; gate-warden may retry; never `pass` |
| Check produces no result file | Adapter yields `error` |
| Dependent check skipped after a failure | Gate `fail` |
| Flaky test passes on retry | Not a harness concern at L1. gate-warden policy decides; holdout and expected tests must pass in the single recorded run |
| Builder's `tests/own/` test fails | Not a gate failure ([gates.md](gates.md)); visible in the summary |
| `Cargo.toml` test-affecting section changed by builder | `oracle-verify` (config guard) fails |
| Builder deletes the `mod tests;` declaration in a source file | `oracle-inventory` fails: expected tests did not run |
| Builder adds a new top-level `tests/*.rs` | `scope-check` fails: path is oracle-owned |
| Locked test does not compile against the implementation | Builder raises a dispute; it is not allowed to edit. G6-lite `build` fails until resolved |
| Locked test passes against stubs (asserts nothing) | Not caught at L1; G5 red-check at L2 |

## Constitutional rules

| Case | Expected behaviour |
| --- | --- |
| Rule file digest differs from the release | `CONSTITUTIONAL_RULES_TAMPERED`; halt before any LLM call |
| Pack rule contradicts a core rule | Core wins; CI and review should reject the pack rule |

## Harness as target

| Case | Expected behaviour |
| --- | --- |
| `harness-author` PR touches `schemas/` | `scope-check` fails; the change must be human-authored |
| `harness-author` PR edits the profile it is running under | Allowed in the checkout. It has no effect on the running stage, which uses the pinned version |
| A harness PR is approved only by its agent author's operator | Not enough: H8 requires a maintainer from CODEOWNERS |
