# ADR-0001: One repository with `disciplines/<name>/`; external packs referenced by version

**Status:** Proposed
**Date:** 2026-11-27

## Context

The handbook leaves open whether the harness is one repository with `disciplines/<name>/`, or a
core harness plus one repository per discipline pack (handbook `docs/open-questions.md`, item 2).
Handbook ADR-0007 says packs may live in other repositories and be referenced by version, and
that the formal pack interface is defined only once the software pack is proven (M7).
Brightsmiths maintains only the software pack.

## Decision

- The harness is a single repository holding the core (`schemas/`, `constitutional-rules/`)
  and the software pack under `disciplines/software/`.
- Every discipline-specific file lives under `disciplines/<name>/` (HR-MN-03).
- Packs maintained elsewhere will be referenced by repository and version from the harness.
  The reference format is part of the formal pack interface (HR-L4-04) and is not defined
  before L4.

## Consequences

- One pin, one release, one eval run covers core and software pack together; core schema
  changes and the pack changes that use them land atomically.
- The software pack cannot be released independently of core. This is acceptable while it is
  the only pack.
- Splitting later is a mechanical move because the directory boundary and the dependency rules
  ([architecture.md](../specs/architecture.md#dependency-rules)) already hold.

## Alternatives considered

- **Core repository plus one repository per pack:** cleaner ownership but needs the pack
  interface and cross-repository version resolution now, before the software pack has proven
  what the interface should be. Two pins per target repository.
- **Packs as branches:** rejected; branches are not immutable release units.
