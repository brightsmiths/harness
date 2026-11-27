# ADR-0007: Personas are role-only; orchestration is stripped

**Status:** Proposed
**Date:** 2026-11-27

## Context

The L1 profiles are ported from existing agent definitions (the Tester and Coder definitions in
`pvandervelde/llm-prompt-engineering`, `claude-agents/`). Those definitions assume an LLM
Conductor that injects context, dispatches agents and hands off work. They tell agents to:

- stop and ask the user;
- commit with particular messages;
- update local task files;
- run hooks;
- report to other agents.

In Brightsmiths, CogWorks does all orchestration deterministically (handbook ADR-0002). The agent
never sees its stage contract. sandbox-runner pushes commits, gate-warden runs the checks, and
`task-marshal` is the agent's only interface to its stage.

## Decision

A persona contains only:

- the role;
- the quality bar;
- the method for producing the stage's artifact.

Everything else is removed or moved:

| Source content | Destination |
| --- | --- |
| Who hands off to whom, "the Conductor injects…", collaboration blocks | Removed (graph, CogWorks) |
| Tool lists, model tier front matter | Stage contract `tools`, `model_class` |
| "Stop and ask the user", confirmation checkpoints | Structured output field (`decisions_flagged`) leading to escalation (H4) |
| Retry and attempt counts ("max 3 attempts") | Stage contract `retries`, `budget` |
| Commit message formats, push, hooks, task-file updates | Removed (sandbox-runner, commit-gate, CogWorks); commit convention may stay as a skill |
| Running lint, tests, mutation as acceptance | Gate definitions; the agent may run tools for feedback, but gates decide |
| Reporting a wrong test or spec gap | `task-marshal dispute`, described in the `dispute-protocol` skill |
| Toolchain specifics (cargo, proptest) | Pack skills |
| House principles reused across roles | Pack skills |

A persona never mentions its own write paths, gate thresholds or holdouts. The persona rules
are in [profiles-and-skills.md](../specs/profiles-and-skills.md#persona-rules).

## Consequences

- Personas are short, reviewable and reusable across orchestration changes.
- Behaviour that used to be an instruction becomes an enforced limit. For example, "do not
  edit tests" becomes a read-only mount plus scope check, and the persona keeps only the
  "dispute instead" guidance.
- Porting is a review task with a checklist, not a rewrite.

## Alternatives considered

- **Port verbatim:** the agent would try to do things it cannot (ask users, commit, hand off) and
  would carry instructions that contradict the stage contract.
- **Write personas from scratch:** discards proven content (adversarial test design, the
  crash-only rule).
