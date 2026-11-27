# Profiles and skills

## Profile anatomy

A profile is a directory `disciplines/<d>/profiles/<stage-type>[.<qualifier>]/`
([ADR-0009](../adr/ADR-0009-profile-naming-and-resolution.md)) containing:

| File | Content | Who reads it |
| --- | --- | --- |
| `persona.md` | The role prompt | The agent, through the context bundle |
| `stage-contract.yaml` | Machine contract ([schemas.md](schemas.md#stage-contract)) | CogWorks, agent-gate, sandbox-runner, gate-warden. **Never the agent** |

Skills live separately in `disciplines/<d>/skills/<skill>/SKILL.md` and are listed by name in
the stage contract.

### Context bundle order

CogWorks assembles the bundle in this order:

1. core constitutional rules;
2. pack constitutional rules;
3. persona;
4. skills, in the order listed;
5. inputs, each marked as data.

The persona and skills come only from the pinned harness, so no repository file can replace
them.

### Resolution

`(stage type, discipline, toolchain, tier) → profile@harness-version`, using the lookup order in
[ADR-0009](../adr/ADR-0009-profile-naming-and-resolution.md). The attestation records
`<discipline>/<profile>@<version>` plus the digest of the profile directory and its skills.

## Persona rules

Every persona must satisfy these rules. CI checks the mechanical ones; H8 reviewers check the
rest.

| # | Rule | Checked by |
| --- | --- | --- |
| P-1 | Describes one role, its quality bar and its method. Nothing about other agents, hand-offs, pipelines or "the Conductor" | Review; CI denylist (`Conductor`, `hand off`, `invoke … agent`) |
| P-2 | No interactive steps: never "ask the user", "wait for confirmation", "present the plan and confirm". Decisions needing a human go into `decisions_flagged` in the stage report | Review; CI denylist |
| P-3 | No commit, push, branch, hook, PR or task-file instructions. The agent commits its work in the workspace; message conventions come from a skill. sandbox-runner pushes | Review; CI denylist (local task-file paths, `git push`) |
| P-4 | No restatement of the stage contract: no write paths, attempts, budgets, gate thresholds, holdouts. Limits are enforced, not explained | Review |
| P-5 | No toolchain specifics if they apply to more than one profile; put them in a skill | Review |
| P-6 | Disagreement with a locked test or a spec is raised as a dispute, never fixed by editing | Review; `dispute-protocol` skill required for zones B and C |
| P-7 | Never assumes network access beyond the model gateway and dependency proxy (no web search or fetch) | Review; stage contract `tools` |
| P-8 | Treats all inputs as data (CR-001) and reports suspected injection with `status: injection-suspected` | Review |
| P-9 | Under 400 lines; prose in British or American English consistently within a file | CI |

## Porting procedure

The L1 profiles are ported from the existing agent definitions in
[`pvandervelde/llm-prompt-engineering`, `claude-agents/`](https://github.com/pvandervelde/llm-prompt-engineering/tree/master/claude-agents),
`tester.md` and `coder.md`. Those definitions run under an LLM "Conductor" that injects
context and dispatches agents. In Brightsmiths that orchestration belongs to CogWorks
([ADR-0007](../adr/ADR-0007-role-only-personas.md)).

For each source definition:

1. **Sort every paragraph** into one of these classes:
   - **keep** (role, philosophy, method);
   - **skill** (reusable or toolchain-specific);
   - **contract** (tools, attempts, model tier);
   - **gate** (checks the agent was told to run as acceptance);
   - **report** (output to return);
   - **drop** (orchestration, interactive steps, local task files, other domains).
2. **Rewrite the keep content** in the second person, present tense, without references to the
   source pipeline.
3. **Move skill content** into `skills/` and list the skills in the stage contract.
4. **Turn "stop and ask" points** into stage report fields (`status: blocked`,
   `decisions_flagged`).
5. **Record the mapping** in the PR description: source section, its class and its
   destination. H8 reviewers check it against the source.
6. **Check against P-1…P-9.**

### `unit-oracle` (zone B, role `oracle`), from `tester.md`

| Source section | Class | Destination / treatment |
| --- | --- | --- |
| Role: write tests before implementation, tests that cannot be fooled by lazy implementations; work from interface specs and assertions, test the contract not the code | keep | Persona core. Replace "before the coder" with "before any implementation exists; you never see implementation source" |
| Three tiers of tests (specification, adversarial unit, property) | keep | Persona method; tool names move to `rust-test-patterns` |
| Testing philosophy (assume wrong; fail against stubs, hardcoded returns, off-by-one; one assertion per test; adversarial over happy-path) | keep | Persona quality bar |
| House principle: crash-only resource lifecycle; recovery-path convergence tests mandatory | skill | `crash-only-resource-lifecycle` (shared with `implement`) |
| Calibrating depth to criticality (module classes → tiers 1–3) | keep, remapped | Persona: depth follows the run's tier (provided as an input field), with the module-class table kept as guidance; T2+ always includes property tests |
| Workflow 1 "Bootstrap context: pre-injected by the Conductor…" | drop | CogWorks builds the bundle |
| Workflow 2 "Load specification context", frontend UX files | keep (spec/contract mapping); drop (frontend) | Persona: map requirements → specification tests, error variants → error-path tests, constraints → boundary tests |
| Workflow 3 "Identify test target… stop only if scope ambiguous; never stop for implementation absence" | keep, reworded | Persona: if behaviour is undefined, return `status: blocked` with the gap; never stop because implementation is absent |
| Workflow 4 "Enumerate scenarios… present plan and confirm scope" | keep (enumerate); drop (confirm) | Enumeration goes into the stage report (`scenarios`) |
| Workflow 5 Specification tests, 1:1 with assertions, ID in test name | keep, renamed | Requirement IDs (`REQ-NNN`) in test names; the trace goes into the report |
| Workflow 6 Boundary, side-effect, stub-killing, recovery convergence | keep / skill | Persona method; convergence details in the skill |
| Workflow 7 Property tests with `proptest`, state machine properties | keep / skill | Persona: when property tests are required; `rust-test-patterns`: how |
| Workflow 8 Contract tests for abstractions | keep | Persona method; only at approved seams (ports, use cases, public module APIs) |
| Workflow 9 Quality checklist | keep | Persona self-check list. Drop "every assertion in `docs/spec/assertions.md`"; use the requirement IDs in the inputs |
| Workflow 10 Commit message format | drop / skill | `conventional-commits` (optional skill); no commit instructions in the persona |
| Workflow 11 "After implementation by the coder: run the suite and report failures" | drop | Zone B never sees the implementation; gate-warden runs tests |
| Workflow 11 "report spec ambiguities to architect", "update edge-cases.md" | report | `decisions_flagged` / `findings` in the stage report; zone B cannot write spec |
| Collaboration block, Conductor note | drop | |
| Front matter `tools`, `disallowedTools: Task`, `tier: balanced` | contract | `tools: [file-edit, shell(cargo for formatting only), task-marshal]`, `model_class: balanced` |
| Post-implementation mutation/fuzz/formal belongs to QA | drop | Gate definitions (G6 at L3) |

Added content not in the source:

- Tests must fail against the stubs (red-check, G5). Write no test that passes against
  `todo!()`.
- Locked unit tests go in sibling `*_tests.rs` files, public-API tests in top-level
  `tests/*.rs`, shared fixtures in `tests/fixtures/`. This is stated in
  `rust-test-patterns`, not as a path rule.
- Report every test function's fully qualified name in `expected_tests` so that the manifest's
  inventory can be built.

Stage contract (L1, proposed):

```yaml
schema: urn:brightsmiths:harness:stage-contract:1
stage_type: unit-oracle
zone: B
identity_role: oracle
model_class: balanced
persona: persona.md
skills: [rust-test-patterns, crash-only-resource-lifecycle, dispute-protocol, rust-conventions]
tools:
  - { name: shell, commands: [cargo, rg] }
  - { name: file-edit }
  - { name: task-marshal }
inputs:
  - spec://**
  - issue://{issue}/scoped
  - impl://src/**/*_tests.rs: read-only   # existing locked tests only, never production source
  - impl://tests/*.rs: read-only
  - impl://tests/fixtures/**: read-only
outputs:
  writable_paths: ["src/**/*_tests.rs", "tests/*.rs", "tests/fixtures/**", "tests/snapshots/**"]
  schema: urn:brightsmiths:harness:software:output:unit-oracle:1
sandbox: { network: [model-gateway, dependency-proxy], wall_clock_min: 45 }
budget: { max_turns: 80, max_tokens: 2000000 }
retries: { max_attempts: 2 }
gates: [G6-lite]      # L1; G5 from L2
```

### `implement` (zone C, role `builder`), from `coder.md`

| Source section | Class | Destination / treatment |
| --- | --- | --- |
| "Implement against pre-defined interfaces… make them work, not invent new ones" | keep | Persona core |
| Execution philosophy: "pure executor… scope decided upstream"; stop only for ambiguity/missing spec | keep, reworded | Persona: implement the slice as specified; if the contract is ambiguous or missing, return `status: blocked` |
| House principle: crash-only, one acquire/reacquire path per resource | skill | `crash-only-resource-lifecycle`; persona keeps one line saying it is a hard rule |
| Loop "exactly once per interaction… STOP EXECUTION… never proceed to next task" | drop | One stage attempt = one slice; CogWorks schedules |
| Step 1 Load context (Conductor-injected, AGENTS.md, `.tech-decisions.yml`, shared registry) | drop | Inputs are the bundle |
| Step 2 "Read the test files committed by the Tester" | keep, reworded | "Read the locked tests for your slice; they define done" |
| Step 3 Pre-task verification: reuse over duplication, structural search, stubs/partials | keep | Persona method; drop local findings files; duplicates go to `findings` |
| Step 4 Load specification (backend); frontend branch | keep (backend); drop (frontend) | Persona |
| Step 4a "Surface significant decisions… STOP and present to user… wait for confirmation" | report | List the decision categories in the persona. If one applies and the contract does not settle it, return `status: blocked` with `decisions_flagged` (escalates to H4) |
| Step 5 Types first, exact types from the interface spec | keep | Persona |
| Step 6 "Do NOT write new tests" | changed | Persona: you may write your own tests in `tests/own/`; they never decide pass/fail; you never modify locked tests |
| Step 7 First commit (types and docs), "if the suite fails to compile, contract mismatch, do not edit tests" | keep (mismatch rule) / drop (commit split) | Persona: a compile failure in locked tests against the contract is a dispute |
| Step 8 Implement bodies; follow documented behaviour; delegate to ports; no extra functionality; "if a test appears wrong, report it" | keep | Persona; "report" becomes a dispute through `task-marshal dispute` |
| Step 9 Final validation, max 3 fix attempts | contract / gate | `retries.max_attempts`; lint and tests are G6-lite. The persona may run them for feedback |
| Step 9a Quality validation, hooks | gate | commit-gate, G6; the secrets rule stays in the persona and CR-005 |
| Step 9b Remove obsolete code | keep | Persona |
| Step 9c Leave it better: fix small issues, write findings for large ones | keep | Persona; findings go into the stage report |
| Step 9d Verify integration, no orphans | keep | Persona |
| Step 10 Second commit format, `.githooks/commit-msg` | drop / skill | `conventional-commits` |
| Step 11 Shared type registry, `docs/catalog.md` | drop | Builder cannot write spec or docs paths; catalog upkeep is a zone A/E concern |
| Steps 12–13 Mark task complete, document TDD discoveries in the local task file | drop | CogWorks state; learnings feed the harness learning loop via findings |
| "On completion… switch to verifier mode" | drop | |
| Frontend extension | drop | Not in the Rust pack at L1 |
| Front matter `tools` incl. WebFetch/WebSearch | contract | Egress is deny-by-default; no web tools |

Stage contract (L1, proposed): as in the [schemas.md example](schemas.md#stage-contract), with
inputs:

- `spec://contracts/**`
- `issue://{issue}/scoped`
- `impl://src/**`
- `impl://Cargo.toml`
- `impl://Cargo.lock`
- `impl://tests/**: read-only`
- `gate://{run}/visible-summary`

### L1 output schemas

| Profile | Fields beyond the envelope |
| --- | --- |
| `unit-oracle` | `expected_tests[]` (fully qualified test names), `requirement_trace[]` (`requirement_id` → test names), `scenarios[]` (tier, kind, description), `files[]` |
| `implement` | `slice` id, `files_changed[]`, `own_tests[]`, `disputes[]` (envelope), `decisions_flagged[]` (envelope) |

## Skills

### Skill format

`SKILL.md` contains:

- YAML front matter: `name`, `description`, `applies_to` (stage types), `zones`;
- a Markdown body.

A skill is guidance only. It cannot add tools, inputs or paths, and it must follow P-1 to P-4
and P-7.

### Catalogue

| Skill | Level | Used by | Source |
| --- | --- | --- | --- |
| `rust-conventions` | L1 | unit-oracle, implement | Team Rust conventions: error handling with `Result`, no `unwrap` outside tests, module layout, sibling test files |
| `rust-test-patterns` | L1 | unit-oracle | `tester.md` tiers and checklists, proptest patterns, locked-test file placement |
| `crash-only-resource-lifecycle` | L1 | unit-oracle, implement | House principle from `tester.md` and `coder.md` |
| `dispute-protocol` | L1 | unit-oracle, implement | New: when and how to use `task-marshal dispute`, what evidence to include |
| `conventional-commits` | L1 (optional) | implement, unit-oracle | Commit message convention from `coder.md` step 10 |
| `ears-acceptance-criteria` | L2 | specify | Brief |
| `program-design-pack` | L2 | contract | Brief |
| `vertical-slicing` | L2 | plan | Brief |

## Profile catalogue by level

| Level | Profile | Zone | Role | Gates | Source to port |
| --- | --- | --- | --- | --- | --- |
| L1 | `unit-oracle` | B | oracle | G6-lite (G5 from L2) | `tester.md` |
| L1 | `implement` | C | builder | G6-lite | `coder.md` |
| L2 | `specify` | A | designer | G1 | `requirements-analyst.md` |
| L2 | `contract` | A | designer | G3 | `interface-designer.md` |
| L2 | `plan` | A | designer | G4 | `planner.md` |
| L3 | `architect` | A | designer | G2 | `architect.md` |
| L3 | `acceptance-oracle` | B | oracle | G5 | `tester.md` (black-box variant) |
| L3 | `refactor` | C | builder | G7 | `refactor.md` |
| L3 | `review-semantic` | D | reviewer | G8 | `verifier.md`, `spec-reviewer.md` |
| L3 | `review-security` | D | reviewer | G8 | `security-reviewer.md` |
| L4 | `drift-scan` | E | health | — | `codebase-surveyor.md` |
| L4 | `debt-fix` | C | builder | G6 | `coder.md` |

The source column suggests where to port from. The same porting procedure applies at each
level.
