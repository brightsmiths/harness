# Behavioural assertions

Testable statements about harness content and how consumers must treat it. Each assertion
names how it is verified:

- **CI:** harness validation ([testing.md](testing.md));
- **Dry run:** consumer dry run;
- **Eval:** L4 behavioural evals;
- **Review:** an H8 checklist item.

## Write protection and layout

**A-001: No pack role can write the harness** *(CI)*
- Given any `disciplines/*/roles.yaml`
- When its roles' write, read-only and read sets are expanded
- Then none contains a harness path pattern, and no role is named `harness-author`.

**A-002: Harness-as-target changes do not take effect before release** *(Review, Dry run)*
- Given a `harness-author` PR merged to `main`
- When a target repository's next run starts
- Then CogWorks and gate-warden use the version in that repository's default-branch pin
- And the merged change has no effect until it is in a release that the pin names.

**A-003: Definitions only** *(CI)*
- Given the repository tree
- When files are listed
- Then every file outside `.github/` is YAML, JSON Schema, Markdown, or a repository metadata
  file (`LICENSE`, `.gitignore`, `CHANGELOG.md`)
- And `.github/` holds only CODEOWNERS, rulesets and workflow files that invoke pinned
  third-party validators.

**A-004: Discipline content stays in packs** *(CI, Review)*
- Given `schemas/` and `constitutional-rules/`
- When searched for discipline terms (toolchain names, `cargo`, `src/`, file extensions of
  languages)
- Then there are no matches.

## Schema conformance

**A-005: Every instance validates** *(CI)*
- Given any YAML file in the repository
- When validated against the schema named in its `schema:` field
- Then validation succeeds
- And a file without a `schema:` field fails CI.

**A-006: Schema ids carry the major version** *(CI)*
- Given a schema file
- When its `$id` is read
- Then it has the form `urn:brightsmiths:harness:<name>:<major>`, unique across the repository.

**A-007: Restricted YAML** *(CI)*
- Given any YAML file
- When linted
- Then it contains no anchors, aliases, merge keys or custom tags.

**A-008: Pin mismatch fails closed** *(Dry run)*
- Given a target pin whose `version` tag resolves to a commit other than `commit`
- When CogWorks or gate-warden loads the harness
- Then it reports `PIN_INVALID` and runs nothing.

## Profiles

**A-009: Profile name matches stage type** *(CI)*
- Given a profile directory `<name>[.<qualifier>…]`
- When its stage contract is read
- Then `stage_type` equals `<name>`
- And each qualifier is a known toolchain or tier.

**A-010: Role exists** *(CI)*
- Given a stage contract
- When `identity_role` is looked up in the pack's `roles.yaml`
- Then the role exists.

**A-011: Zone matches role** *(CI)*
- Given a stage contract and its role
- Then `zone` equals the role's zone.

**A-012: Writable paths within role write set** *(CI, Dry run)*
- Given a stage contract
- When `outputs.writable_paths` is compared with the role's `write` minus `write_except`
- Then every writable path pattern is covered by the effective write set.

**A-015: Zone B never reads implementation** *(CI)*
- Given a zone B stage contract
- Then no input matches `impl://src/**` except locked test files, and no input matches
  `tests/own/**`.

**A-018: Personas are role-only** *(CI, Review)*
- Given a `persona.md`
- When checked against the denylist and the persona rules P-1…P-9
- Then there are no matches, and the file is under the size limit.

**A-026: No silent fallback in resolution** *(Dry run)*
- Given a resolution key with no matching profile under the ADR-0009 lookup order
- When CogWorks resolves it
- Then it reports `PROFILE_NOT_FOUND` and does not substitute any other profile.

## Graphs and gates

**A-016: Graph stages resolve** *(CI)*
- Given `graphs/t<n>.yaml`
- When each `stage` node is resolved for tier `t<n>` and toolchain `rust`
- Then a profile is found.

**A-017: Graph gates are defined for the tier** *(CI)*
- Given a `gate` node in `graphs/t<n>.yaml`
- Then a gate definition with that id exists, and its `applies_to` includes `t<n>`.

**A-025: No stage reaches done ungated** *(CI)*
- Given a stage graph
- When every path from `entry` to `done` is enumerated
- Then each stage node on the path is followed by at least one gate node before `done`.

**A-019: G6-lite contents** *(CI, Dry run)*
- Given `gates/G6-lite.yaml`
- Then it contains exactly the checks `scope-check`, `oracle-verify`, `build`, `fmt`,
  `clippy`, `nextest` and `oracle-inventory`
- And each is required at t1.

**A-021: Skipped means failed** *(Dry run)*
- Given a gate whose required check did not run (dependency failed or tool missing)
- When gate-warden computes the gate result
- Then the gate status is `fail` (or `error` for infrastructure faults), never `pass`.

**A-013: Thresholds are monotonic by tier** *(CI)*
- Given a check with thresholds at tiers `ti` < `tj`
- Then the threshold at `tj` is equal or stricter
- And a check required at `ti` is required at `tj`.

**A-027: Holdout feedback is coarse** *(Dry run, L3)*
- Given a failing holdout check
- When the result reaches a zone C agent
- Then it contains only `status` and `requirement_ids`.

## Constitutional rules

**A-014: Pack rules only add** *(CI, Review)*
- Given a pack rule file
- Then it declares `extends: core`, its rule IDs use the `CR-<PACK>-` prefix, and no core ID
  is redefined.

**A-024: Rules load first and are verified** *(Dry run)*
- Given a stage bundle built by CogWorks
- Then core rules come first, then pack rules, then the persona
- And if the loaded rules' digest differs from the pinned release, CogWorks reports
  `CONSTITUTIONAL_RULES_TAMPERED` before any LLM call.

## Releases

**A-022: Schema breaks bump major** *(Review, CI where detectable)*
- Given a PR that removes or renames a schema field, narrows an enum or adds a required field
- Then the PR carries a breaking-change marker, and the next release bumps the major version
  (or the minor version while `0.y`, with a CHANGELOG note).

**A-023: Pin overrides only tighten** *(CI for schema, Dry run)*
- Given a pin override
- When it would select a lower tier or a less strict profile variant than the risk map gives
- Then validation fails.

**A-020: Cheat traps block release** *(Eval, L4)*
- Given an eval run for a release candidate
- When any cheat-trap task ends in a gate pass instead of a dispute
- Then the release is blocked.
