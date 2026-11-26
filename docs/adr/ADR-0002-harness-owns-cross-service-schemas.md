# ADR-0002: The harness owns the cross-service schemas

**Status:** Proposed
**Date:** 2026-11-27

## Context

Handbook ADR-0007 makes the stage contract schema and the gate result format public interfaces
that need versioning. The harness brief requires these schemas to be "agreed with CogWorks,
gate-warden and agent-gate". Several consumers (gate-warden, sandbox-runner, oracle-guard) have
no specification yet, so there is no existing format to defer to.

## Decision

- The harness is the canonical home of every cross-service schema: stage contract, gate
  result, gate definition, stage graph, roles, pin, risk map, constitutional-rule front matter,
  and the stage output schemas.
- Consumers agree to a schema change by approving the harness PR that introduces it. Until
  approved by the listed consumers, a schema is marked **proposed, pending consumer agreement**
  in [schemas.md](../specs/schemas.md) and in its `description`.
- Consumers use the schema of the harness version they are pinned to, or vendor a copy
  identified by `$id`. They never fork it.

## Consequences

- One place to look for every format; one versioning scheme (ADR-0004).
- Consumer teams must review harness PRs that touch `schemas/`; CODEOWNERS enforces this.
- A consumer that needs a new field must change the harness first. This slows changes down on
  purpose.

## Alternatives considered

- **Each consumer owns its own input format:** natural for a single consumer, but the stage
  contract alone is read by three services, and the gate result is written by many tools and
  read by two services. Ownership would be split.
- **A separate schemas repository:** a second pin and release train with no benefit while the
  harness is the only producer of instances.
