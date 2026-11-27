# Trade-offs

Alternatives considered for the main design choices. The decisions themselves are recorded as
ADRs in [`docs/adr/`](../adr/README.md).

## Repository structure ([ADR-0001](../adr/ADR-0001-single-repository-with-discipline-packs.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **One repository, `disciplines/<name>/`** (chosen) | One pin, one release, atomic core and pack changes, one eval run | Pack cannot release on its own |
| Core repository plus per-pack repositories | Independent ownership and release | Needs the pack interface now; two pins; cross-repository compatibility matrix |

## Schema ownership ([ADR-0002](../adr/ADR-0002-harness-owns-cross-service-schemas.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **Harness owns all shared schemas** (chosen) | Single source and version; consumers agree by review | Consumers depend on harness PRs for format changes |
| Each consumer owns its input format | Close to the code that parses it | Stage contract has three readers; gate result has many writers; ownership splits |
| Separate schemas repository | Clear purpose | A second release train and pin |

## Format ([ADR-0003](../adr/ADR-0003-yaml-and-json-schema.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **YAML + JSON Schema** (chosen) | Universal tooling; reviewable | Cross-file rules need extra checks (see AS-05) |
| CUE | Cross-file constraints built in | New toolchain for every consumer |
| JSON only | No YAML ambiguity | Harder to read and review |

## Versioning ([ADR-0004](../adr/ADR-0004-single-semver-version.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **One SemVer for the whole harness** (chosen) | Simple pin; evals per version | Small change releases everything |
| Per-profile versions | Fine-grained rollout | Combinatorial, untested combinations; resolution complexity |

## Distribution ([ADR-0005](../adr/ADR-0005-distribution-and-pinning.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **Git tag + SHA, signed archive from L2** (chosen) | Works anywhere, including air-gapped mirrors; no registry | Signing key management |
| OCI artifact | Mature signing (cosign) | Registry required per deployment |
| Submodule in target repos | Familiar | Puts the harness inside agent-visible workspaces |

## Write protection ([ADR-0006](../adr/ADR-0006-write-protection-model.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **No write to the harness in force; `harness-author` for harness-as-target runs** (chosen) | Factory can build L2–L4; rules in force never change mid-run | Needs repository-level roles in agent-gate |
| Humans write everything | Simplest | Contradicts the roadmap; humans bottleneck |
| Reuse `designer` role | No new role | One role with two unrelated write sets |

## Personas ([ADR-0007](../adr/ADR-0007-role-only-personas.md))

| Option | Pros | Cons |
| --- | --- | --- |
| **Role-only, ported with a mapping table** (chosen) | Keeps proven content; contract enforced not explained | Porting effort and review |
| Verbatim port | Fast | Instructions the agent cannot follow; conflicts with contracts |
| Rewrite from scratch | Clean | Loses proven test-design content |

## Gate ordering inside G6-lite

| Option | Pros | Cons |
| --- | --- | --- |
| **Cheap integrity checks first** (chosen: scope-check, oracle-verify before build) | Fast failure; no wasted compute on tampered commits | Feedback shows only the first failure class |
| Run everything | Full feedback per attempt | Wastes compute; more information to a possibly misbehaving agent |

## Builder's own tests in nextest

| Option | Pros | Cons |
| --- | --- | --- |
| **Exclude through oracle-owned nextest profile** (chosen, AS-19) | One test run | Relies on nextest filter expressiveness |
| Separate optional `nextest-own` check | Clear separation | A second test run per attempt |
