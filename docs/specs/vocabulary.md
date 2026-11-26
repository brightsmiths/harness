# Vocabulary

Domain concepts used throughout the harness specification. Where a concept is defined by the
system architecture, the handbook page is named; this page defines how the harness represents
it.

## Packaging concepts

### Harness version

- **Description:** A released, immutable snapshot of the whole harness repository.
- **Identifier:** SemVer `MAJOR.MINOR.PATCH` (tag `vX.Y.Z`) plus the 40-character commit SHA the tag
  points to.
- **Constraints:** One version covers every artifact in the repository
  ([ADR-0004](../adr/ADR-0004-single-semver-version.md)). Breaking a schema means a major
  version bump. A released version is never moved or rewritten; it can be *withdrawn*
  (see [operations.md](operations.md#withdrawal)).
- **Lifespan:** Permanent once released.

### Pin

- **Description:** A target repository's choice of harness version, discipline pack and
  toolchain, stored in `.factory/pin.yaml`. Human-owned in the target repository.
- **Fields:** schema id, harness version, harness commit SHA, discipline, toolchain, optional
  per-path discipline overrides, optional archive digest. See [schemas.md](schemas.md#pin-file).
- **Constraints:** Must name a released version; tag and SHA must agree. Overrides may only
  tighten, never relax (higher tier, stricter profile variant).

### Discipline pack

- **Description:** Everything discipline-specific: profiles per stage type, skills, stage graphs
  per tier, gate definitions, oracle conventions, path conventions (roles), pack-specific output
  schemas and additive constitutional rules (handbook ADR-0007).
- **Identifier:** directory name under `disciplines/` (`software`), or an external pack
  reference `<repository>@<version>`.
- **Constraints:** All discipline-specific content lives under `disciplines/<name>/`. A pack may
  reference core schemas; core never references a pack.

### Core

- **Description:** The discipline-neutral part of the harness: `schemas/` and
  `constitutional-rules/`.
- **Constraints:** Contains no discipline vocabulary (no language, toolchain, file extension or
  discipline path convention).

### Toolchain

- **Description:** The language/toolchain within a discipline (software pack: `rust`; `csharp`
  and `typescript` documented, not built). Part of the profile resolution key.

## Profile concepts

### Stage type

- **Description:** The kind of work a stage does: `specify`, `architect`, `contract`, `plan`,
  `acceptance-oracle`, `unit-oracle`, `implement`, `refactor`, `review-semantic`,
  `review-security`, `drift-scan`, `debt-fix` (software pack; handbook
  `docs/architecture/agent-profiles.md`).
- **Identifier:** lower-kebab-case string.

### Agent profile

- **Description:** The versioned package CogWorks runs for one stage: persona, skills, tool
  allowlist, stage contract, identity role and trust zone.
- **Identifier:** `<discipline>/<profile-name>@<harness-version>`, plus a content digest
  recorded in the stage attestation. Profile directory name equals the stage type, with an optional
  tier variant suffix: `implement`, `implement.t3`
  ([ADR-0009](../adr/ADR-0009-profile-naming-and-resolution.md)).
- **Contents:** `persona.md`, `stage-contract.yaml`.
- **Constraints:** Belongs to exactly one trust zone and one identity role. The agent never sees
  its own stage contract.

### Persona

- **Description:** The role prompt in `persona.md`. Describes the role, its quality bar and its
  way of working. Contains no orchestration (no hand-offs, no "stop and ask the user", no
  commit or push instructions, no references to other agents)
  ([ADR-0007](../adr/ADR-0007-role-only-personas.md)).

### Skill

- **Description:** A guidance document (`skills/<skill>/SKILL.md`) loaded into a profile's
  context, such as conventions or test patterns. Skills are data for the agent, never a way to
  change its permissions.
- **Identifier:** `<discipline>/<skill-name>`.

### Stage contract

- **Description:** The machine-readable part of a profile (`stage-contract.yaml`): stage type,
  zone, identity role, inputs (context-bundle allowlist), outputs (writable paths and output
  schema), tools, sandbox (network, wall clock), budget, retries and the gates that judge the
  stage. Enforced by CogWorks, agent-gate and sandbox-runner. See
  [schemas.md](schemas.md#stage-contract).

### Trust zone

- **Description:** The information barrier a profile belongs to (handbook ADR-0003):
  A (intent and design), B (oracle), C (build), D (assurance), E (health), Q (quarantine; lives
  in Entry Sentinel, not in the harness).
- **Identifier:** single letter `A`–`E`.

### Identity role

- **Description:** The GitHub App role a profile's commits are pushed under, and the key in
  `roles.yaml` that defines what that role may read and write: `designer`, `oracle`, `builder`,
  `reviewer`, `health`; plus `harness-author` for runs whose target is the harness itself.
- **Constraints:** Each role maps to exactly one zone. A role's write set never includes the
  harness when the harness is not the run target.

### Input URI

- **Description:** An entry in a stage contract's input allowlist: `<scheme>://<path-glob>`
  with an optional `: read-only` marker and `{placeholders}` resolved by CogWorks. Schemes:
  `spec`, `impl`, `gate`, `issue`, `lore`. See [schemas.md](schemas.md#input-uri-schemes).

## Workflow concepts

### Tier

- **Description:** Criticality class T0 (prototype), T1 (internal), T2 (product), T3
  (safety/security-critical), assigned per module through the target repository's human-owned
  `risk-map.yaml` (handbook ADR-0006). Selects the stage graph, gate thresholds, profile variants
  and human gates.

### Risk map

- **Description:** `risk-map.yaml` in a target repository: path globs to tier. The tier of a
  piece of work is the highest tier among the paths it touches. Schema in
  [schemas.md](schemas.md#risk-map).

### Stage graph

- **Description:** A directed graph per tier (`graphs/t0.yaml` … `t3.yaml`) of stage nodes, gate
  nodes and human-gate nodes, with retry and escalation edges. CogWorks walks it as a step
  function over GitHub state. See [stage-graphs.md](stage-graphs.md).

### Gate

- **Description:** An independent verdict point G1–G9, computed only by gate-warden and
  published as a required check run (handbook ADR-0004).
- **Identifier:** `G1` … `G9`; reduced variants carry a suffix (`G6-lite`).

### Gate definition

- **Description:** `gates/<gate>.yaml`: the ordered checks for one gate, their commands, result
  adapters and per-tier thresholds and applicability. See [gates.md](gates.md).

### Check

- **Description:** One command in a gate definition, run by gate-warden in a clean sandbox,
  whose outcome is expressed in the gate result format.
- **Identifier:** lower-kebab-case, unique within a gate (`scope-check`, `nextest`).

### Result adapter

- **Description:** A discipline-neutral converter, provided by gate-warden, that turns a
  check's native output into the gate result format: `exit-code`, `junit-xml`, `sarif`, or
  `native` (the command already emits the gate result format). Adapters are referenced by the
  harness, never implemented in it.

### Threshold

- **Description:** A per-tier numeric or boolean limit a check's result must meet (e.g. maximum
  warnings = 0, mutation score ≥ 0.7).

### Gate result

- **Description:** The standard, versioned record of one check's outcome and of a gate's
  overall verdict. See [schemas.md](schemas.md#gate-result).

### Human gate

- **Description:** A human decision point H1–H8 (handbook `docs/architecture/workflow.md`). Graphs
  reference them; gate-warden verifies the human was authorised. **H8** is the harness change
  gate: harness maintainers review every harness PR.

### Constitutional rule

- **Description:** A non-overridable behavioural rule loaded before any other context in every
  agent stage (moved from CogWorks; see the CogWorks repository's ADR-0003, "Constitutional
  Security Layer"). Core rules apply to every pack; a pack may add rules but never relax a core
  rule.
- **Identifier:** `CR-NNN` (core), `CR-<PACK>-NNN` (pack).

### Locked test, manifest, seam

- **Description:** Oracle concepts defined in handbook `docs/architecture/oracles.md`. A
  *locked test* is an oracle-authored test addressable by path. The *manifest*
  (`tests/locked/MANIFEST` plus `.sig`) lists locked files, test-affecting configuration and
  expected test IDs, signed by the oracle identity. A *seam* is a port, use case or public module
  API that locked tests may target.
- **Harness role:** the software pack's `roles.yaml` defines the locked paths; gate definitions
  call `oracle-guard` to check them.

### Holdout configuration

- **Description:** (L3) Per-tier policy for hidden tests: minimum holdout ratio, injection
  method, feedback granularity. Never contains holdout tests.

### Dispute

- **Description:** A structured objection raised by a zone C agent through `task-marshal
  dispute` when it believes a locked test is wrong. Personas direct agents to dispute rather than
  edit.

## Evaluation concepts (L4)

### Golden task

- **Description:** A representative issue per tier with a known-good outcome, re-run on every
  harness change and every model-version change. Lives in the evals repository.

### Cheat trap

- **Description:** A seeded task whose visible tests contradict the spec. The correct outcome is
  a dispute; any "pass" blocks the harness release.

### Canary

- **Description:** Rollout of a new harness version first to T0/T1 repositories, widening to T2
  then T3 only after the scorecard holds for N runs.

### Kill switch

- **Description:** GitHub-visible state (label or pinned config) that disables a profile, a
  harness version, a run, or the factory globally. The harness defines the profile and version
  identifiers kill switches refer to.

## Error and failure concepts

These are the failure conditions consumers must report when harness content is invalid or
unavailable. Consumers name them; the harness defines when they apply.

| Concept | Meaning | Security implication |
| --- | --- | --- |
| `PIN_INVALID` | Pin fails schema validation, or tag and SHA disagree | A moved tag could swap rules; must halt |
| `HARNESS_VERSION_UNAVAILABLE` | The pinned version cannot be fetched or its digest does not match | Fail closed; never fall back to another version |
| `HARNESS_VERSION_WITHDRAWN` | The pinned version is withdrawn | Halt new runs; existing verdicts stand |
| `SCHEMA_MAJOR_UNSUPPORTED` | A consumer does not support the schema major version in use | Fail closed |
| `PROFILE_NOT_FOUND` | No profile matches the resolution key and no fallback applies | Halt the stage; never substitute a less strict profile |
| `GATE_UNDEFINED` | A graph references a gate with no definition for the tier | Halt; the gate must never pass by default |
| `ROLE_UNDEFINED` | A stage contract names an identity role missing from `roles.yaml` | Halt; no mount plan can be produced |
| `CONSTITUTIONAL_RULES_TAMPERED` | Loaded rules do not match the pinned release digest | Halt before any LLM call |
