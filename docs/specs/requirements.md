# Requirements

Requirements are grouped by increment level. Each level is tied to a Brightsmiths milestone
(handbook `docs/roadmap.md`). A level is built only to what its milestone needs.

| Level | Milestone | Built by |
| --- | --- | --- |
| L1 | M1, first dark slice | Humans |
| L2 | M3, event-driven runs and agent-written specs | Factory, H8 review |
| L3 | M6, T2 and release | Factory, H8 review |
| L4 | M7, health loop and hardening | Factory, H8 review |

Requirement IDs are `HR-L<level>-NN`. "Must not" rules are `HR-MN-NN` and apply at every level.
Anything beyond L4 belongs to the harness's own roadmap and is marked **after factory role
complete**.

## Must not (all levels)

| ID | Requirement | Verified by |
| --- | --- | --- |
| HR-MN-01 | The harness is not writable by any agent role running under a pinned harness. The only agent write access is the `harness-author` role, and only when the harness repository is itself the run target. Those changes take effect only after H8 review, release and pinning. | [roles-and-paths.md](roles-and-paths.md), [security.md](security.md), A-001, A-002 |
| HR-MN-02 | The harness contains no service code. It holds definitions only: YAML, JSON Schema, Markdown. CI workflow configuration that calls off-the-shelf validators is allowed. | [constraints.md](constraints.md), A-003 |
| HR-MN-03 | No discipline-specific content outside `disciplines/<name>/`. | [architecture.md](architecture.md), A-004 |

## L1 (M1, human-led)

| ID | Requirement | Consumer(s) | Acceptance |
| --- | --- | --- | --- |
| HR-L1-01 | A **stage contract schema** (`schemas/stage-contract.schema.json`, v1), agreed with CogWorks, agent-gate and sandbox-runner. | CogWorks, agent-gate, sandbox-runner | Schema published; consumers approve the PR that introduces it; both L1 profiles validate against it. |
| HR-L1-02 | A **gate result schema** (`schemas/gate-result.schema.json`, v1), agreed with gate-warden and CogWorks. | gate-warden, CogWorks, oracle-guard | Schema published; gate-warden approves; G6-lite's adapters produce it. |
| HR-L1-03 | A **gate definition schema**, **stage graph schema**, **roles schema**, **pin schema** and **risk-map schema** (all v1, minimal). | gate-warden, CogWorks, agent-gate, target repos | Each published and used by at least one L1 artifact. |
| HR-L1-04 | Profile `unit-oracle` (zone B, role `oracle`), ported from the existing Tester agent definition with orchestration removed. | CogWorks, sandbox-runner | Persona follows [the persona rules](profiles-and-skills.md#persona-rules); contract validates; no `impl://src/**` input. |
| HR-L1-05 | Profile `implement` (zone C, role `builder`), ported from the existing Coder agent definition with orchestration removed. | CogWorks, sandbox-runner | As HR-L1-04; locked tests are read-only inputs; writable paths are within the builder write set. |
| HR-L1-06 | Skills needed by the L1 profiles: `rust-conventions`, `rust-test-patterns`, `crash-only-resource-lifecycle`, `dispute-protocol`. | CogWorks | Each skill is referenced by at least one profile; none contains orchestration. |
| HR-L1-07 | `disciplines/software/roles.yaml` with the `oracle` and `builder` roles and the Rust path conventions. | agent-gate | agent-gate produces a mount plan and scope check from it for the pilot repository. |
| HR-L1-08 | Gate definition `G6-lite`: scope-check, oracle-guard verify and inventory, build, fmt, clippy, nextest. | gate-warden | gate-warden runs it on a pilot SHA and publishes one check run per gate with per-check results. |
| HR-L1-09 | Stage graph `graphs/t1.yaml` for the pilot: human-written issue → unit-oracle → implement → G6-lite → human review and merge. | CogWorks | CogWorks L1 walks it for commit-gate issues. |
| HR-L1-10 | **Core constitutional rules** (`constitutional-rules/core.md`), moved from CogWorks, with the format and precedence rules in [ADR-0008](../adr/ADR-0008-constitutional-rules-in-core.md). | CogWorks | CogWorks L1 loads them from the pinned harness and verifies the digest before any LLM call. |
| HR-L1-11 | A **release and pin procedure** that is enough for M1: SemVer tag, CHANGELOG, pin by tag plus commit SHA. | All consumers | The pilot repository pins `v0.1.0` (or later). |
| HR-L1-12 | **CI validation**: every YAML file validates against its schema; cross-reference and layout checks per [testing.md](testing.md). | Harness maintainers | Required on every PR. |
| HR-L1-13 | Repository protections: CODEOWNERS for all paths, rulesets so only maintainers can merge, the harness's own `risk-map.yaml` mapping `**` to T2. | — | Configured before the first release. |

## L2 (M3)

| ID | Requirement | Consumer(s) |
| --- | --- | --- |
| HR-L2-01 | Profiles `specify`, `contract`, `plan` (zone A, role `designer`). | CogWorks |
| HR-L2-02 | Gate definitions `G1` (specify), `G3` (contracts), `G4` (plan). | gate-warden |
| HR-L2-03 | Skills `ears-acceptance-criteria`, `program-design-pack`, `vertical-slicing`. | CogWorks |
| HR-L2-04 | `designer` role in `roles.yaml`; output schemas for `specify`, `contract` and `plan` (including `plan.yaml`). | agent-gate, CogWorks, task-marshal |
| HR-L2-05 | T1 graph extended with the design stages, G1/G3/G4 and H1. | CogWorks |
| HR-L2-06 | `harness-author` role mapping for runs that target the harness repository itself (from M3 the factory builds harness L2+). | agent-gate |
| HR-L2-07 | Signed releases: release archive digest and Ed25519 signature; chain-chronicler DSSE when available. | CogWorks, gate-warden |
| HR-L2-08 | `G5` (oracles: red-check, requirement trace, manifest signed) added to the T1 graph. This was added beyond the brief; see [assumptions.md](assumptions.md). | gate-warden |

## L3 (M6)

| ID | Requirement | Consumer(s) |
| --- | --- | --- |
| HR-L3-01 | Profiles `architect`, `acceptance-oracle`, `refactor`, `review-semantic`, `review-security`. | CogWorks |
| HR-L3-02 | `graphs/t2.yaml` and T2 thresholds in every gate definition; `G2`, full `G6`, `G7`, `G8`. | CogWorks, gate-warden |
| HR-L3-03 | Holdout configuration (policy only): ratio, injection method, coarse feedback. | gate-warden, oracle-guard |
| HR-L3-04 | `reviewer` role; output schema for structured findings. | agent-gate, review-reeve |

## L4 (M7)

| ID | Requirement | Consumer(s) |
| --- | --- | --- |
| HR-L4-01 | Profiles `drift-scan` (zone E, `health`) and `debt-fix` (zone C, `builder`). | CogWorks |
| HR-L4-02 | Evals repository interface: golden tasks and cheat traps per tier; release blocked on any cheat-trap pass. | Harness maintainers |
| HR-L4-03 | Pinning and canary rollout procedure, with withdrawal and kill-switch identifiers. | All consumers |
| HR-L4-04 | Formal discipline-pack interface: what a pack must provide, how external packs are referenced and validated. | CogWorks, gate-warden, agent-gate |
| HR-L4-05 | `graphs/t3.yaml` and T3 thresholds and profile variants (`*.t3`). | CogWorks, gate-warden |

## After factory role complete

These items belong to the harness's own roadmap and are not scheduled by the factory milestones:

- C# and TypeScript toolchain mappings in the software pack (handbook open question 10).
- Model-diversity rules for zone B vs zone C (handbook open question 7).
- Profiles for stage types not listed above.
