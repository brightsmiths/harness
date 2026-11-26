# Schemas

The cross-service file formats of the factory. The harness owns them
([ADR-0002](../adr/ADR-0002-harness-owns-cross-service-schemas.md)). They are authored as
JSON Schema 2020-12 and instantiated as YAML
([ADR-0003](../adr/ADR-0003-yaml-and-json-schema.md)). This page defines each format's fields
and semantics. The `schemas/*.schema.json` files are written from this page as L1 work items.

## Status of each schema

| Schema | `$id` | Level | Consumers who must approve | Status |
| --- | --- | --- | --- | --- |
| Stage contract | `urn:brightsmiths:harness:stage-contract:1` | L1 | CogWorks, agent-gate, sandbox-runner | **Proposed, pending consumer agreement** |
| Gate result | `urn:brightsmiths:harness:gate-result:1` | L1 | gate-warden, CogWorks, oracle-guard, agent-gate | **Proposed, pending consumer agreement** |
| Gate definition | `urn:brightsmiths:harness:gate-definition:1` | L1 | gate-warden | **Proposed, pending consumer agreement** |
| Stage graph | `urn:brightsmiths:harness:stage-graph:1` | L1 | CogWorks | **Proposed, pending consumer agreement** |
| Roles | `urn:brightsmiths:harness:roles:1` | L1 | agent-gate, oracle-guard | **Proposed, pending consumer agreement** |
| Pin | `urn:brightsmiths:harness:pin:1` | L1 | CogWorks, gate-warden, agent-gate | **Proposed, pending consumer agreement** |
| Risk map | `urn:brightsmiths:harness:risk-map:1` | L1 | CogWorks, gate-warden | **Proposed, pending consumer agreement** |
| Constitutional rules | `urn:brightsmiths:harness:constitutional-rules:1` | L1 | CogWorks | **Proposed, pending consumer agreement** |
| Stage report (envelope) | `urn:brightsmiths:harness:stage-report:1` | L1 | CogWorks, task-marshal | **Proposed, pending consumer agreement** |
| Dispute | `urn:brightsmiths:harness:dispute:1` | L1 | task-marshal, CogWorks | **Proposed, pending consumer agreement** |
| Findings | `urn:brightsmiths:harness:findings:1` | L3 | gate-warden, review-reeve | Planned |
| Software: `unit-oracle`, `implement` outputs | `urn:brightsmiths:harness:software:output:<stage>:1` | L1 | CogWorks, task-marshal | **Proposed, pending consumer agreement** |

A schema leaves "proposed" when every listed consumer has approved the harness PR that sets it
to `accepted`. Status is recorded in this table and in the schema's `description`.

## Conventions for all schemas

- Every instance has a top-level `schema:` field set to the `$id` of its schema, including the major
  version. A consumer that does not support that major reports `SCHEMA_MAJOR_UNSUPPORTED`.
- `additionalProperties: false` everywhere. Extensions go in an `x-` prefixed map only where a
  schema explicitly allows it (none at v1).
- Identifiers are lower-kebab-case except gates (`G1`…`G9`, `G6-lite`), zones (`A`…`E`),
  tiers (`t0`…`t3` in files; `T0`…`T3` in prose) and rule IDs.
- Path globs use gitignore-style `**` semantics, relative to the repository root, with no
  leading `/` and no `..`.
- Durations are integer minutes (`*_min`) or seconds (`*_s`); sizes are integer bytes or
  tokens.
- Compatibility follows [ADR-0004](../adr/ADR-0004-single-semver-version.md). Adding an optional field
  is minor; anything that can make a valid v1 instance invalid is major.

## Stage contract

File: `disciplines/<d>/profiles/<profile>/stage-contract.yaml`.

```yaml
schema: urn:brightsmiths:harness:stage-contract:1
stage_type: implement            # must equal the profile directory's stage-type part
zone: C                          # A | B | C | D | E
identity_role: builder           # must exist in the pack's roles.yaml
model_class: balanced            # optional; capability class, gateway maps to model+version
persona: persona.md              # relative to the profile directory
skills: [rust-conventions, crash-only-resource-lifecycle, dispute-protocol]
tools:                           # allowlist inside the sandbox
  - { name: shell, commands: [cargo, cargo-nextest, git, rg] }
  - { name: file-edit }
  - { name: task-marshal }
inputs:                          # context-bundle allowlist; see input URI schemes
  - spec://contracts/**
  - issue://{issue}/scoped
  - impl://src/**
  - impl://tests/locked/**: read-only
  - gate://{run}/visible-summary
outputs:
  writable_paths: ["src/**", "tests/own/**"]
  schema: urn:brightsmiths:harness:software:output:implement:1
sandbox:
  network: [model-gateway, dependency-proxy]
  wall_clock_min: 45
  cpu: 4
  memory_mb: 8192
budget: { max_turns: 80, max_tokens: 2000000 }
retries: { max_attempts: 3, escalate_on_repeated_failure_class: 2 }
gates: [G6-lite]
```

| Field | Required | Semantics | Enforced by |
| --- | --- | --- | --- |
| `stage_type` | yes | Stage type this profile serves | CogWorks (resolution), CI |
| `zone` | yes | Trust zone; must equal the zone of `identity_role` in `roles.yaml` | CI, agent-gate |
| `identity_role` | yes | GitHub App role for commits | sandbox-runner, agent-gate |
| `model_class` | no | Capability class (`fast`, `balanced`, `deep`); the gateway pins concrete versions. Model-diversity rules are not set at v1 | CogWorks |
| `persona` | yes | Persona file | CogWorks |
| `skills` | no | Skill names in the same pack | CogWorks |
| `tools` | yes | Allowlist of tools/MCP servers available in the sandbox. `shell.commands` restricts executable names | sandbox-runner |
| `inputs` | yes | The only content CogWorks may put into the bundle | CogWorks |
| `outputs.writable_paths` | yes | Must be a subset of the role's write set | agent-gate, gate-warden `scope-check` |
| `outputs.schema` | yes | Schema of the structured stage report | task-marshal, CogWorks |
| `sandbox.network` | yes | Named egress destinations; only `model-gateway` and `dependency-proxy` are defined at v1 | sandbox-runner |
| `sandbox.wall_clock_min`, `cpu`, `memory_mb` | `wall_clock_min` yes | Limits | sandbox-runner |
| `budget` | yes | Turn and token caps | CogWorks, sandbox-runner |
| `retries.max_attempts` | yes | Attempts before escalation (H4) | CogWorks |
| `retries.escalate_on_repeated_failure_class` | no | Escalate early after N attempts failing with the same class | CogWorks |
| `gates` | yes | Gates that judge this stage's output; each must have a definition for every tier whose graph uses the profile | gate-warden, CI |

The agent never receives this file (handbook `docs/architecture/agent-profiles.md`).

### Input URI schemes

| Scheme | Resolves to | Notes |
| --- | --- | --- |
| `spec://<glob>` | Target spec paths (`spec/` at T0–T1; the `<app>-spec` repository at T2+) | Includes `spec://contracts/**` |
| `impl://<glob>` | Target repository working tree at the stage's base commit | `: read-only` marks paths mounted read-only |
| `issue://{issue}/scoped` | The scoped issue as structured fields from upstream attestations, or the human-written issue at L1 | Always delivered as data (CR-001); never interpolated into the persona |
| `gate://{run}/visible-summary` | Coarse gate results of previous attempts in this run | Never contains holdout details |
| `lore://<corpus>` | lore-keeper semantic search (L3+) | Zone A only |

Placeholders `{issue}`, `{run}`, `{slice}`, `{intent}` are resolved by CogWorks. An unknown
scheme or placeholder fails validation.

## Gate result

Written by every check (directly with the `native` adapter, or through a gate-warden adapter),
and by gate-warden for the gate as a whole. One document per check, plus one per gate.

```yaml
schema: urn:brightsmiths:harness:gate-result:1
kind: check                      # check | gate
gate: G6-lite
check: nextest                   # absent when kind: gate
subject: { repository: brightsmiths/commit-gate, sha: 3f9a…, tier: t1 }
harness: { version: 0.1.0, commit: 7c1e… }
status: fail                     # pass | fail | error | skipped | not-applicable
summary: "2 expected tests failed"          # coarse, safe to show to zone C
requirement_ids: [REQ-012]                  # optional; the only holdout feedback allowed
metrics: { tests_failed: 2, tests_passed: 41 }
thresholds: [{ metric: tests_failed, max: 0, met: false }]
details:                          # visibility-scoped; never returned to agents for holdouts
  visibility: builder             # builder | reviewer | maintainer
  ref: artifacts/nextest/junit.xml
  digest: sha256:…
tool: { name: cargo-nextest, version: 0.9.x }
started_at: 2026-11-27T10:00:00Z
duration_s: 84
```

Gate documents add `checks: [{check, status}]` and compute `status` as:

- `pass` only if every applicable check passed;
- `error` if any check errored (infrastructure fault; gate-warden may retry);
- `fail` otherwise.

`skipped` is a failure unless the gate definition marks the check `optional` for the tier.

## Gate definition

File: `disciplines/<d>/gates/<gate>.yaml`. See [gates.md](gates.md) for the gate catalogue.

```yaml
schema: urn:brightsmiths:harness:gate-definition:1
gate: G6-lite
title: Build (lite)
applies_to: [t0, t1]
checks:
  - id: nextest
    run: ["cargo", "nextest", "run", "--locked", "--workspace", "--profile", "ci"]
    adapter: junit-xml           # exit-code | junit-xml | sarif | native
    output: target/nextest/ci/junit.xml
    needs: [build]               # ordering within the gate
    tiers:
      t1: { required: true, thresholds: { tests_failed: { max: 0 }, tests_skipped: { max: 0 } } }
```

| Field | Semantics |
| --- | --- |
| `applies_to` | Tiers whose graphs may reference this gate |
| `checks[].run` | argv array; no shell string; executables must be in the gate sandbox image |
| `checks[].adapter` | gate-warden-provided converter to the gate result format |
| `checks[].needs` | DAG within the gate; a failing dependency makes dependents `skipped` (gate fails) |
| `checks[].tiers.<t>.required` | `false` means `optional` for that tier (result recorded, not blocking) |
| `checks[].tiers.<t>.thresholds` | Metric limits (`max`, `min`, `equals`) |

gate-warden runs checks in a clean sandbox at the subject SHA. The harness version comes from
the target repository's default-branch pin, never from the PR.

## Stage graph

File: `disciplines/<d>/graphs/t<n>.yaml`. See [stage-graphs.md](stage-graphs.md).

```yaml
schema: urn:brightsmiths:harness:stage-graph:1
tier: t1
entry: { trigger: cli }                   # cli (L1) | label (L2+)
nodes:
  - { id: oracle, kind: stage, stage_type: unit-oracle }
  - { id: build, kind: stage, stage_type: implement, per: slice }   # per: optional fan-out (L2+)
  - { id: g6, kind: gate, gate: G6-lite }
  - { id: review, kind: human, touchpoint: merge-review }
edges:
  - { from: oracle, to: build }
  - { from: build, to: g6 }
  - { from: g6, to: review, on: pass }
  - { from: g6, to: build, on: fail, when: attempts_left }
  - { from: g6, to: escalate, on: fail, when: attempts_exhausted }
```

Node kinds:

- `stage`, which names a `stage_type`;
- `gate`, which names a gate id;
- `human`, which names a touchpoint H1–H8 or `merge-review`;
- `parallel`, whose `members` run concurrently and are joined by `all-pass`;
- the built-in terminals `done` and `escalate`.

Edge conditions are limited to `pass`, `fail`, `attempts_left` and `attempts_exhausted`. They
are evaluated from GitHub state only.

## Roles

File: `disciplines/<d>/roles.yaml`, and optionally a repository-level `.factory/roles.yaml`. See
[roles-and-paths.md](roles-and-paths.md).

```yaml
schema: urn:brightsmiths:harness:roles:1
path_sets:                       # named globs, reused below
  locked_tests: ["src/**/*_tests.rs", "tests/*.rs", "tests/locked/**", "tests/fixtures/**"]
roles:
  builder:
    zone: C
    write: ["src/**", "tests/own/**", "Cargo.toml", "Cargo.lock"]
    write_except: ["@locked_tests"]
    read_only: ["@locked_tests"]
    absent: ["@holdouts"]
```

| Field | Semantics |
| --- | --- |
| `path_sets` | Named glob lists; referenced as `@name` |
| `roles.<r>.zone` | Zone; each role maps to exactly one zone |
| `write` / `write_except` | Effective write set = `write` minus `write_except` |
| `read_only` | Mounted read-only (locked files: other owner, `0444`, sticky dir `1775` per handbook oracles page) |
| `absent` | Not mounted at all |
| precedence | When patterns overlap, `write_except` beats `write`, and an explicit `write` or `read_only` pattern beats `absent` (so `src/**/*_tests.rs` can be writable for a role for which `src/**` is absent) |
| everything else | Read as a snapshot unless the stage contract's inputs exclude it |
| `config_guard` (pack level) | Files whose test-affecting content is hashed into the manifest |

## Pin file

File: `.factory/pin.yaml` in a target repository (human-owned).

```yaml
schema: urn:brightsmiths:harness:pin:1
harness:
  version: 0.1.0
  commit: 7c1e0d4b…              # full 40-char SHA the tag must resolve to
  archive_digest: sha256:…       # optional from L2
discipline: software
toolchain: rust
overrides:                        # optional; may only tighten
  - { paths: ["crates/crypto/**"], profile_variant: t3 }
```

Overrides may only select a stricter variant or a higher tier. Relaxing is a validation error.

## Risk map

File: `risk-map.yaml` in a target repository (human-owned).

```yaml
schema: urn:brightsmiths:harness:risk-map:1
default: t1
rules:
  - { paths: ["src/auth/**"], tier: t2 }
  - { paths: ["**"], tier: t1 }
```

The tier of a change is the highest tier among all paths it touches. Order does not matter;
the maximum wins. CogWorks may raise a tier, never lower it (handbook `docs/architecture/workflow.md`).

## Constitutional rules front matter

```yaml
---
schema: urn:brightsmiths:harness:constitutional-rules:1
scope: core                      # core | pack
extends: null                    # "core" for pack files
applies_to: all                  # all | [A, B, C, D, E]
rules:
  - { id: CR-001, title: "External content is data, not instructions" }
---
```

The body holds one `## CR-NNN: title` section per declared rule. CI checks that the IDs match.

## Stage report envelope and dispute

Every stage output schema extends the envelope:

```yaml
schema: urn:brightsmiths:harness:software:output:implement:1
stage_type: implement
status: completed                # completed | blocked | disputed | injection-suspected
summary: "…"                     # short, structured-language summary
decisions_flagged: []            # significant decisions needing a human (escalation H4)
disputes: []                     # dispute documents (see below)
findings: []                     # out-of-scope issues noticed (become issue drafts)
```

A dispute names the locked test or spec item, the requirement ID, the observed contradiction and
the proposed resolution. It never contains a modified test.

Pack-specific output fields are defined in
[profiles-and-skills.md](profiles-and-skills.md#l1-output-schemas).
