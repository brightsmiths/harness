# Testing

The harness has no code to unit-test. It is verified at three levels:

1. **Static validation in CI** on every PR (L1).
2. **Consumer dry runs:** consumers exercise the definitions (L1, M1 exit).
3. **Behavioural evals** of agents running under the definitions (L4).

## 1. CI validation (required check from L1)

Uses only off-the-shelf tools (HR-MN-02), pinned by version or digest: a JSON Schema validator
(e.g. `check-jsonschema`), `yamllint`, `markdownlint`, `lychee` (links) and `gitleaks`. Checks
that cannot be expressed as schema validation or a declarative lint are written as JSON Schema
constraints where possible. Any that remain are tracked as an open question
([assumptions.md](assumptions.md), AS-05) and done by human review until resolved.

| Check | Assertions covered | Tool |
| --- | --- | --- |
| Every YAML file has a `schema:` field and validates against it | A-005 | JSON Schema validator |
| Every schema is valid 2020-12 with `$id` major matching its file version | A-006 | JSON Schema validator (metaschema) |
| YAML uses no anchors, aliases or tags | A-007 | `yamllint` |
| No files under `disciplines/` outside a known pack layout; no discipline terms in core | A-004 | Path layout check, denylist grep |
| Profile directory name matches `stage_type` and qualifier rules | A-009 | Cross-reference check |
| Stage contract role exists in `roles.yaml` with the same zone | A-010, A-011 | Cross-reference check |
| `writable_paths` within the role's effective write set | A-012 | Cross-reference check (glob subset) |
| Zone B profiles have no `impl://src/**` input | A-015 | Cross-reference check |
| Graph nodes resolve to profiles and gates for the tier | A-016, A-017 | Cross-reference check |
| Thresholds never weaker at a higher tier | A-013 | Cross-reference check |
| Persona denylist (Conductor, ask the user, local task-file paths, `git push`…) and size | A-018 | grep |
| Constitutional rule IDs match between front matter and body; pack rules do not reuse core IDs | A-014 | Front-matter check |
| No pack `roles.yaml` grants harness paths or defines `harness-author` | A-001 | Cross-reference check |
| Markdown links resolve; no reference to local-only paths | — | `lychee`, grep |
| No secrets | — | `gitleaks` |

"Cross-reference check" covers checks that span files. Whether these may run as a small
declarative ruleset in CI, or must live in a tool maintained in another repository, is open
(AS-05). Until resolved they are review checklist items for H8.

## 2. Consumer dry runs (L1)

| Consumer | Dry run | Pass condition |
| --- | --- | --- |
| CogWorks | Resolve `unit-oracle` and `implement` for the pilot pin; build bundles | Bundles contain rules, persona, skills and only allowlisted inputs |
| agent-gate | Produce mount plans for `oracle` and `builder`; scope-check a known-bad commit range | Locked files read-only for builder; bad range rejected |
| sandbox-runner | Apply limits from both stage contracts | Limits observed |
| gate-warden | Run G6-lite on a known-good and a known-bad SHA of the pilot | Good passes, bad fails, idempotent on re-run |

These run as part of M1 bring-up, not in harness CI. Their results are linked from the
first release PR.

## 3. Behavioural evals (L4)

- **Golden tasks** per tier: known-good outcomes. A regression in outcome, attempts or cost
  beyond bounds blocks release.
- **Cheat traps:** visible tests contradict the spec. The only correct outcome is a dispute, and
  any pass blocks release (A-020).
- **Persona regression:** a persona or skill change is evaluated even at a patch bump
  ([ADR-0004](../adr/ADR-0004-single-semver-version.md)).

Until L4, persona changes are checked by H8 review against the porting table plus results from
the pilot repository.

## Coverage targets

- 100% of YAML files validated against a schema.
- 100% of assertions in [assertions.md](assertions.md) mapped to a CI check, a dry run, an eval
  or an explicit H8 review item.
