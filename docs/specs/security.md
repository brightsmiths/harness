# Security

The harness is "the rules": what every agent is told, what every gate checks, who may write
where. If an attacker or a misbehaving agent controls the harness, it controls the factory. It is
a control-plane repository and is treated as T2+ whoever builds it (handbook `docs/roadmap.md`,
bootstrap rule 5).

## Assets

| Asset | Why it matters |
| --- | --- |
| Gate definitions and thresholds | Weakening one lets bad changes pass everywhere that pins the version |
| `roles.yaml` | Widening a write set breaks oracle separation |
| Personas, skills, constitutional rules | Prompt content reaches every agent in a privileged position |
| Schemas | Loosening one lets invalid contracts or outputs through |
| Release tags and signatures | Swapping a release swaps all of the above |
| Release signing key | Forging releases |

## Threats and mitigations

| # | Threat | Mitigation | Residual risk |
| --- | --- | --- | --- |
| T-1 | An agent edits the rules it runs under | No role grants harness paths; the harness is absent from mounts; context bundle only ([ADR-0006](../adr/ADR-0006-write-protection-model.md)); A-001 | None known |
| T-2 | A harness PR weakens a gate and is judged by itself | gate-warden uses the default-branch pin's released version, never the PR head; H8 on every PR | A maintainer approving a harmful PR. Mitigated by two approvals on `schemas/` and `constitutional-rules/`, and by evals (L4) |
| T-3 | The `harness-author` agent smuggles a weakening into a large PR | Human-only paths (schemas, rules, `.factory`, `.github`); CI cross-reference checks (A-005 to A-012); threshold-monotonic check (A-013); evals and cheat traps (L4) | Subtle persona wording. Review guidance: diff personas against the porting table |
| T-4 | Prompt injection through skill or persona content (e.g. copied text) | All content is reviewed; CR-001 on inputs; personas contain no untrusted text; skills cannot add tools or paths | Low |
| T-5 | Tag moved or release replaced | Tag protection; pin records the commit SHA; archive digest and signature from L2 ([ADR-0005](../adr/ADR-0005-distribution-and-pinning.md)) | Repository admin compromise |
| T-6 | Consumer fetches a different version than pinned (MITM, mirror) | Verify SHA and, from L2, signature; fail closed (`HARNESS_VERSION_UNAVAILABLE`) | — |
| T-7 | Holdout leakage through the harness | The harness holds holdout *policy* only; never test content or holdout repository credentials | — |
| T-8 | Discipline pack overrides core rules | Pack rules can only add; CI checks that pack rule files declare `extends: core` and do not reuse core IDs (A-014) | Semantic contradiction. Caught by review |
| T-9 | Pin override relaxes a tier in a target repo | Pin schema allows only tightening overrides; consumers reject relaxing values | — |
| T-10 | Signing key theft | Key in KMS or Vault; signing in the release workflow with OIDC-scoped access; rotation documented in the release procedure | — |
| T-11 | Credentials committed to the harness | gitleaks in CI; CR-005 for agent authors | — |
| T-12 | Supply chain of CI validators | Pin action and tool versions by digest; renovate with human review | — |

## Repository controls (HR-L1-13)

- **CODEOWNERS:**
  - `*` → harness maintainers;
  - `schemas/**` → maintainers plus consumer owners (CogWorks, gate-warden, agent-gate,
    sandbox-runner);
  - `disciplines/software/roles.yaml` → maintainers plus agent-gate owners;
  - `disciplines/*/gates/**` → maintainers plus gate-warden owners.
- **Rulesets on `main`:**
  - PR required, no direct pushes, no force-push;
  - 1 maintainer approval in general and 2 for `schemas/` and `constitutional-rules/`;
  - required checks: CI validation;
  - from the first release, gate-warden checks driven by the harness's own pin.
- **Tag ruleset:** `v*` immutable.
- **Merge rights:** maintainers only. No GitHub App role used by agents has merge or admin rights.
- **Own risk map:** `risk-map.yaml` maps `**` to T2.

## Content rules

- Every persona and skill must satisfy the persona rules P-1…P-9
  ([profiles-and-skills.md](profiles-and-skills.md#persona-rules)). They must not tell an agent
  to seek, read or reveal its stage contract, holdouts, other roles' outputs or secrets.
- No secrets, tokens, internal hostnames or organisation-specific defaults anywhere in the
  harness. Network destinations are symbolic (`model-gateway`, `dependency-proxy`).
- Public repository: assume all content is read by attackers. Security must not depend on
  personas or gate definitions being secret. Holdout content is the only secret oracle material,
  and it is never here.
