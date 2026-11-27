# Architecture decision records

Decisions about the harness repository's own structure and content. System-level decisions
(GitHub as system of record, trust zones, gate-warden verdicts, microVM isolation, tiers,
discipline packs) live in the handbook repository,
[brightsmiths/handbook `docs/adr/`](https://github.com/brightsmiths/handbook/tree/feat/initial/docs/adr).
The ADRs here cite them and do not restate them.

Any change to harness *behaviour* (what agents are told, what gates check, what schemas
require) needs an ADR or an update to an existing one, reviewed under H8.

| ADR | Title | Status |
| --- | --- | --- |
| [0001](ADR-0001-single-repository-with-discipline-packs.md) | One repository with `disciplines/<name>/`; external packs by version | Proposed |
| [0002](ADR-0002-harness-owns-cross-service-schemas.md) | The harness owns the cross-service schemas | Proposed |
| [0003](ADR-0003-yaml-and-json-schema.md) | YAML authoring, JSON Schema 2020-12 validation | Proposed |
| [0004](ADR-0004-single-semver-version.md) | One SemVer version for the whole harness | Proposed |
| [0005](ADR-0005-distribution-and-pinning.md) | Distribution by git tag; pin by tag and commit SHA | Proposed |
| [0006](ADR-0006-write-protection-model.md) | Write protection and the harness-as-target role | Proposed |
| [0007](ADR-0007-role-only-personas.md) | Personas are role-only; orchestration is stripped | Proposed |
| [0008](ADR-0008-constitutional-rules-in-core.md) | Constitutional rules live in core and load first | Proposed |
| [0009](ADR-0009-profile-naming-and-resolution.md) | Profile directory equals stage type; tier variant suffix | Proposed |

New ADRs use [ADR_TEMPLATE.md](ADR_TEMPLATE.md) (the handbook template).
