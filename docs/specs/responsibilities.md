# Responsibilities

CRC-style cards for each artifact kind in the harness. In a definitions-only repository the
"components" are artifact kinds. *Knows* is the information an artifact carries. *Does* is the
decision it lets a consumer make deterministically. *Collaborators* are the other artifacts it
references and the services that read it.

## Core

### Schema (`schemas/*.schema.json`)

- **Knows:** the shape, required fields, enumerations and version (`$id` with major) of one
  cross-service file format.
- **Does:** lets any consumer reject invalid harness content or agent output before acting on
  it; defines the compatibility boundary for releases.
- **Collaborators:** every other artifact (validated by it); every consumer service.
- **Roles:** contract, validator.

### Constitutional rules (`constitutional-rules/core.md`)

- **Knows:** the non-overridable behavioural rules and their IDs.
- **Does:** gives CogWorks the privileged first block of every agent context; sets the floor that
  packs can only raise.
- **Collaborators:** pack rule files (additive); CogWorks (loader, digest check).
- **Roles:** policy, trust anchor.

## Software pack

### Profile (`profiles/<stage-type>/`)

- **Knows:**
  - persona (role prompt);
  - stage type, zone, identity role;
  - input allowlist, writable paths, output schema;
  - tools, sandbox network, budget, retries;
  - the gates that judge it.
- **Does:** lets CogWorks build a context bundle and lets agent-gate and sandbox-runner apply
  limits without interpreting prose.
- **Collaborators:** skills, output schema, `roles.yaml` (role must exist; paths must be within
  its write set), graphs (reference it by stage type), gates (referenced by id).
- **Roles:** unit of agent behaviour, unit of attestation (`profile@version` plus digest).

### Persona (`persona.md`)

- **Knows:** the role, its quality bar and its method.
- **Does:** shapes what the model produces. It decides nothing about the workflow.
- **Collaborators:** the profile's skills.
- **Roles:** prompt content (data the harness controls).

### Skill (`skills/<skill>/SKILL.md`)

- **Knows:** reusable guidance: conventions, patterns, checklists.
- **Does:** keeps personas short and shares guidance across profiles in the same pack.
- **Collaborators:** profiles that list it.
- **Roles:** prompt content.

### Stage graph (`graphs/t<n>.yaml`)

- **Knows:** for one tier: the nodes (stages, gates, human gates), their order, parallel groups,
  retry limits and escalation edges.
- **Does:** lets CogWorks compute the next step from GitHub state alone.
- **Collaborators:** profiles (by stage type), gate definitions (by id), human gates H1–H8.
- **Roles:** workflow definition.

### Gate definition (`gates/<gate>.yaml`)

- **Knows:** for one gate: the ordered checks, each check's command, working directory, result
  adapter, applicability per tier, and thresholds per tier.
- **Does:** lets gate-warden compute a verdict reproducibly, idempotent per (SHA, gate, harness
  version).
- **Collaborators:** gate result schema; check tools (agent-gate, oracle-guard, cargo…); graphs.
- **Roles:** verdict policy.

### Roles (`roles.yaml`)

- **Knows:** each identity role's zone, read set, write set and read-only set, and the pack's
  path conventions (locked paths, own-test paths, test-configuration files).
- **Does:** gives agent-gate the single source for mount plans, scope checks and push rulesets.
- **Collaborators:** profiles (must name a defined role); gate `scope-check`.
- **Roles:** access policy.

### Output schema (`schemas/outputs/*`, `disciplines/*/schemas/outputs/*`)

- **Knows:** the structured result a stage must return through task-marshal.
- **Does:** lets CogWorks and reviewers consume stage results without parsing prose.
- **Collaborators:** profiles; task-marshal; CogWorks.

## Repository-level artifacts

### CHANGELOG and release

- **Knows:** what changed per version, and whether it is breaking.
- **Does:** lets repositories decide whether to move their pin; lets consumers detect schema
  majors.
- **Collaborators:** release-regent (creates release PRs, tags, GitHub Releases).

### Harness factory files (`.factory/pin.yaml`, `.factory/roles.yaml`, `risk-map.yaml`)

- **Knows:** which released harness judges PRs to the harness itself; the `harness-author` role;
  the harness's tier (T2).
- **Does:** keeps the harness under the same gates as other control-plane repositories without
  letting a PR judge itself.

## Consumer matrix

| Artifact | CogWorks | agent-gate | sandbox-runner | gate-warden | task-marshal | oracle-guard | Target repo |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Stage contract | R | R (role, paths) | R (sandbox, budget) | R (gates) | — | — | — |
| Persona, skills | R | — | — | — | — | — | — |
| Output schemas | R | — | — | — | R | — | — |
| Stage graph | R | — | — | — | — | — | — |
| Gate definition | — | — | — | R | — | — | — |
| Gate result schema | R | W (scope-check) | — | R/W | — | W | — |
| Roles | — | R | — | R (scope-check) | — | R (locked paths) | — |
| Pin, risk-map schemas | R | R | — | R | — | — | instances |
| Constitutional rules | R | — | — | — | — | — | — |

R = reads, W = emits data in that format.
