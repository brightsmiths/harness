# ADR-0009: Profile directory equals stage type; tier variant suffix

**Status:** Proposed
**Date:** 2026-11-27

## Context

CogWorks selects a profile deterministically from
`(stage type, discipline, language/toolchain, tier) → profile@version` and records the choice in
the stage attestation (handbook `docs/architecture/agent-profiles.md`). The handbook's
illustrative path is `profiles/implementer/` for stage type `implement`, while the brief names
the profile `implement`. A naming rule is needed so that resolution is a pure lookup.

## Decision

- A profile directory is named after its stage type: `disciplines/<d>/profiles/<stage-type>/`.
- A variant directory adds the most specific qualifier as a dot-separated suffix:
  - `<stage-type>.<tier>`, e.g. `implement.t3`;
  - `<stage-type>.<toolchain>`;
  - `<stage-type>.<toolchain>.<tier>`.

  `@` is reserved for versions.
- Resolution order, first match wins:
  1. `<stage-type>.<toolchain>.<tier>`
  2. `<stage-type>.<toolchain>`
  3. `<stage-type>.<tier>`
  4. `<stage-type>`

  No other fallback. In particular, a profile for a lower tier never substitutes for a missing
  higher-tier variant unless the unqualified profile exists. If nothing matches, the result is
  `PROFILE_NOT_FOUND`.
- A variant's `stage-contract.yaml` must declare the same `stage_type`, zone and identity role
  as the unqualified profile.

## Consequences

- No lookup table to maintain; CI checks names against contract fields.
- The software pack's first toolchain is Rust, so L1 profiles may be unqualified. C# and
  TypeScript variants slot in later without renaming.

## Alternatives considered

- **Role nouns (`implementer`):** reads naturally but needs a mapping table from stage type.
- **A resolution table file:** flexible, but adds an artifact that can drift from the
  directories.
