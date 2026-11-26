# ADR-0006: Write protection and the harness-as-target role

**Status:** Proposed
**Date:** 2026-11-27

## Context

The brief says the harness must not "be writable by any agent role". It also says harness L2–L4
are "built by the factory with human review (H8)". Both hold if the protection is defined
against the harness *in force*: an agent must never change the rules it is running under, or
the rules any other run is judged by. The handbook's visibility and write matrix gives no agent
role any access to the harness, and only maintainers write it.

## Decision

1. **No agent running under a pinned harness can write that harness.**
   - No role in any pack's `roles.yaml` lists harness paths in its read or write sets.
   - The harness is absent from every sandbox mount plan (handbook
     `docs/architecture/isolation-and-security.md`).
   - Profile contents reach the agent only through the context bundle CogWorks builds.
2. **Harness as run target.** When the harness repository itself is the target of a run, a
   dedicated `harness-author` role (zone A) may push branches and open PRs to it. The role's
   mapping is defined in the harness's own `.factory/roles.yaml`
   ([roles-and-paths.md](../specs/roles-and-paths.md#harness-repository-as-a-run-target)).
   - It may write `disciplines/**`, `docs/**` and `CHANGELOG.md` content.
   - It never writes `schemas/**`, `constitutional-rules/**`, `.factory/**`, `.github/**`,
     `risk-map.yaml` or the `docs/adr/` status of an accepted ADR. These stay human-only.
3. **Every PR to the harness needs H8.** CODEOWNERS covers every path with the harness
   maintainers team. Rulesets require at least one maintainer approval, or two for `schemas/`
   and `constitutional-rules/`. Only maintainers can merge.
4. **Changes take effect only once released and pinned.** A merge to `main` changes nothing for
   any run. Consumers read only released, pinned versions (ADR-0005).
5. **The harness judges itself only from a released version.** PRs to the harness are gated by
   the version in the harness's own `.factory/pin.yaml` on the default branch, never by the PR
   head (handbook `docs/roadmap.md`, bootstrap rule 5). Until the first release, gates on
   harness PRs are run by humans.

## Consequences

- The factory can build the harness from M3 without any path by which an agent changes rules in
  force.
- agent-gate must support a repository-level roles mapping (`.factory/roles.yaml`) that
  replaces or extends the pack's mapping. This is a requirement on agent-gate L2 and an open
  question for it ([assumptions.md](../specs/assumptions.md)).
- Schemas and constitutional rules always need a human author.

## Alternatives considered

- **No agent writes at all; humans write all harness content:** contradicts the plan for the
  factory to build L2–L4.
- **Use the pack's `designer` role for harness runs:** it writes target-repo spec paths. Reusing
  it would blur the matrix and give one role two very different write sets.
