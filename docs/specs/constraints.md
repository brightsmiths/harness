# Constraints

Hard rules for everything committed to the harness. A PR that breaks one is rejected in H8
review even if CI passes.

## Content types

1. **Definitions only** (HR-MN-02). Allowed:
   - YAML instances;
   - JSON Schema files;
   - Markdown (personas, skills, rules, docs);
   - repository metadata;
   - `.github/` configuration that calls pinned third-party tools.

   Not allowed: scripts, programs, adapters, or code embedded in YAML to be executed. Gate
   `run` entries are argv arrays naming tools that exist elsewhere.
2. **YAML 1.2, restricted subset:** no anchors, aliases, merge keys or tags; two-space indent;
   lower-kebab-case keys except where a schema defines otherwise
   ([ADR-0003](../adr/ADR-0003-yaml-and-json-schema.md)).
3. **JSON Schema 2020-12:**
   - `$id` of the form `urn:brightsmiths:harness:<name>:<major>`;
   - `additionalProperties: false`;
   - every property has a `description`;
   - enumerations are closed.
4. **Every instance declares `schema:`.**

## Boundaries

5. **Core is discipline-neutral.** `schemas/` and `constitutional-rules/` contain no discipline,
   language or toolchain terms (HR-MN-03).
6. **Dependencies point inward:** pack → core, never core → pack, never pack → pack
   ([architecture.md](architecture.md#dependency-rules)).
7. **No cross-version references.** Every reference resolves within the same harness commit.

## Naming

8. Profile directory = stage type, plus optional `.<toolchain>` and `.<tier>` qualifiers
   ([ADR-0009](../adr/ADR-0009-profile-naming-and-resolution.md)).
9. Gate files are named after the gate id (`G6-lite.yaml`). Graph files are `t0.yaml`…`t3.yaml`.
   Skills are lower-kebab-case directories with a `SKILL.md`.
10. Rule IDs: `CR-NNN` (core), `CR-<PACK>-NNN` (pack). Requirement IDs in this spec:
    `HR-L<n>-NN`, `HR-MN-NN`. Assertions: `A-NNN`.

## Personas and skills

11. Personas follow P-1…P-9 ([profiles-and-skills.md](profiles-and-skills.md#persona-rules)).
12. Skills give guidance only. They never grant tools, inputs or paths.
13. No secrets, internal hostnames, organisation-specific defaults or personal data anywhere.

## Change process

14. Every change is a PR with H8 approval from a maintainer.
    - Two approvals for `schemas/**` and `constitutional-rules/**`.
    - Consumer-owner approval for schema, roles and gate files (CODEOWNERS).
15. Behaviour changes need an ADR or an ADR update: anything that changes what agents are told,
    what gates check, thresholds, roles or schemas.
16. Conventional commits. Breaking schema changes carry `!` or a `BREAKING CHANGE:` footer
    ([ADR-0004](../adr/ADR-0004-single-semver-version.md)).
17. Human-only paths: `schemas/**`, `constitutional-rules/**`,
    `disciplines/*/constitutional-rules/**`, `.factory/**`, `.github/**`, `risk-map.yaml`,
    `AGENTS.md`, `CLAUDE.md` ([ADR-0006](../adr/ADR-0006-write-protection-model.md)).
18. Committed files never reference local-only, git-ignored paths. Cite the handbook
    repository instead.

## Gate semantics

19. A gate never passes by default. Missing definitions, missing tools and skipped required
    checks all fail ([gates.md](gates.md#principles)).
20. Thresholds are monotonic by tier: a higher tier is never weaker (A-013).
21. Agent-visible feedback is coarse; holdout feedback is requirement IDs only.

## Performance and size budgets

There is no runtime, so the budgets are on content size, which affects context cost:

- Persona under 400 lines.
- Each skill under 300 lines.
- The default bundle overhead (rules, persona and skills) for any L1 profile under 30k tokens.
- CI validation completes in under 2 minutes.
