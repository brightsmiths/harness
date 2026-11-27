# ADR-0004: One SemVer version for the whole harness

**Status:** Proposed
**Date:** 2026-11-27

## Context

The handbook writes profile selection as `→ profile@version`, and attestations record the
harness version, profile version and prompt hash (handbook
`docs/architecture/isolation-and-security.md`, provenance). Target repositories pin a harness
version. It is not stated whether profiles are versioned individually.

## Decision

- The harness has a single SemVer version. Tag `vX.Y.Z`; releases are made by release-regent
  from conventional commits.
- A profile's version is the harness version. Its identity in attestations is
  `<discipline>/<profile>@<harness-version>` plus a SHA-256 digest of the profile directory
  and the skills it loads.
- Version bumps:
  - **major:** a breaking change to any schema (removed or renamed field, narrowed enum, new
    required field), or removing a profile, gate or graph;
  - **minor:** new profiles, gates, checks, skills, optional fields, or stricter thresholds;
  - **patch:** prose changes in personas and skills that do not change contracts.
- Any change to persona, skill, gate or threshold content is still evaluated (L4) whatever the
  bump, because prompt changes change behaviour.

## Consequences

- One pin per repository; one CHANGELOG; evals run per harness version.
- A change to one profile releases everything, so canary rollout covers all profiles together.
- The digest lets attestations distinguish profiles that did not change between versions.

## Alternatives considered

- **Per-profile versions:** fine-grained pins but a combinatorial set of untested
  combinations, and a resolution step with its own failure modes.
- **Calendar versioning:** no signal for breaking schema changes.
