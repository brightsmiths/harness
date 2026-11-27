# Roles and path conventions

`disciplines/software/roles.yaml` is agent-gate's single source for the visibility and write
matrix (handbook `docs/architecture/agent-profiles.md`). The same file drives:

- sandbox-runner mount plans, produced by agent-gate;
- the commit-range `scope-check` that gate-warden runs;
- push rulesets.

The format is in [schemas.md](schemas.md#roles).

## Roles by level

| Role | Zone | Level | Used by profiles |
| --- | --- | --- | --- |
| `oracle` | B | L1 | `unit-oracle`; `acceptance-oracle` (L3) |
| `builder` | C | L1 | `implement`; `refactor` (L3), `debt-fix` (L4) |
| `designer` | A | L2 | `specify`, `contract`, `plan`; `architect` (L3) |
| `reviewer` | D | L3 | `review-semantic`, `review-security` |
| `health` | E | L4 | `drift-scan` |
| `harness-author` | A | L2 | Harness-as-target runs only; defined in the harness's `.factory/roles.yaml` |

Each role maps to exactly one zone. A profile's `zone` must equal its role's zone (A-011).

## Rust path conventions (software pack, L1)

These follow the locking conventions in handbook `docs/architecture/oracles.md`.

| Path | Meaning | Owner |
| --- | --- | --- |
| `src/**` (excluding `*_tests.rs`) | Production source | builder |
| `src/**/*_tests.rs` | Locked unit tests in sibling files, included with `#[cfg(test)] #[path = "foo_tests.rs"] mod tests;` | oracle |
| `tests/*.rs` | Locked public-API integration tests (top-level files only) | oracle |
| `tests/own/main.rs`, `tests/own/**` | Builder's own tests; compiled as one integration-test target through `tests/own/main.rs`. They never decide pass/fail | builder |
| `tests/fixtures/**`, `tests/snapshots/**` | Shared test infrastructure | oracle |
| `tests/locked/MANIFEST`, `MANIFEST.sig` | Oracle manifest and signature | oracle identity (see open item below) |
| `spec/**` | Spec at T0–T1, including `spec/contracts/**` | designer (L2+); humans at L1 |
| `.factory/**`, `risk-map.yaml`, `AGENTS.md`, `CLAUDE.md` | Factory configuration | humans only |
| `.factory/baseline/**` | Ratchet baselines | gate-warden only |

**Why `tests/own/main.rs`:** Cargo discovers only `tests/*.rs` and `tests/*/main.rs`. A loose
`tests/own/foo.rs` would never be compiled. A `[[test]]` entry in `Cargo.toml` would change
test configuration, which is oracle-owned.

**Test-affecting configuration** is in `config_guard`. `oracle-guard config-guard` hashes it into
the manifest:

- `.cargo/config.toml`
- `build.rs`
- `.config/nextest.toml`
- `rust-toolchain.toml`
- `clippy.toml`, `rustfmt.toml`
- the `Cargo.toml` sections `[dev-dependencies]`, `[[test]]`, `[features]`, `[profile.test]` and
  `[lints]`

`Cargo.toml` and `Cargo.lock` stay writable by the builder so that dependencies can be added.
Changes to guarded sections are caught by the config guard, not by path.

## Matrix (software pack)

| Path set | oracle (B) | builder (C) | designer (A, L2) | reviewer (D, L3) | health (E, L4) |
| --- | --- | --- | --- | --- | --- |
| `spec/**` excluding contracts | R | R (slice only via inputs) | **W** | R | R |
| `spec/contracts/**` | R | R | **W** | R | R |
| `src/**` (production) | absent | **W** | R (snapshot) | R | R |
| locked tests, fixtures, snapshots | **W** | read-only | absent | R | R |
| `tests/own/**` | absent | **W** | absent | R | R |
| `tests/locked/**` (manifest) | **W** (via lock step) | read-only | absent | R | R |
| `config_guard` files | **W** (test sections) | R (`Cargo.toml` W, guarded) | absent | R | R |
| holdouts (L3) | **W** (acceptance-oracle) | absent | absent | absent | absent |
| `.factory/**`, `risk-map.yaml` | R | R | R | R | R |
| the harness | absent | absent | absent | absent | absent |

The illustrative YAML for L1:

```yaml
schema: urn:brightsmiths:harness:roles:1
path_sets:
  locked_tests: ["src/**/*_tests.rs", "tests/*.rs", "tests/fixtures/**", "tests/snapshots/**"]
  manifest: ["tests/locked/**"]
  own_tests: ["tests/own/**"]
  factory: [".factory/**", "risk-map.yaml", "AGENTS.md", "CLAUDE.md", "**/AGENTS.md", "**/CLAUDE.md"]
config_guard:
  files: [".cargo/config.toml", "build.rs", ".config/nextest.toml", "rust-toolchain.toml",
          "clippy.toml", "rustfmt.toml"]
  sections: { "Cargo.toml": ["dev-dependencies", "test", "features", "profile.test", "lints"] }
roles:
  oracle:
    zone: B
    write: ["@locked_tests", "@manifest"]
    absent: ["src/**", "@own_tests"]
    read_only: ["spec/**", "@factory"]
  builder:
    zone: C
    write: ["src/**", "@own_tests", "Cargo.toml", "Cargo.lock"]
    write_except: ["@locked_tests"]
    read_only: ["@locked_tests", "@manifest", "spec/**", "@factory"]
```

## Harness repository as a run target

From M3 the factory builds harness L2+ ([ADR-0006](../adr/ADR-0006-write-protection-model.md)).
When the run's target repository is the harness itself, agent-gate uses the harness's own
`.factory/roles.yaml` instead of a pack's `roles.yaml`:

```yaml
schema: urn:brightsmiths:harness:roles:1
path_sets:
  human_only: ["schemas/**", "constitutional-rules/**", "disciplines/*/constitutional-rules/**",
               ".factory/**", ".github/**", "risk-map.yaml", "AGENTS.md", "CLAUDE.md"]
roles:
  harness-author:
    zone: A
    write: ["disciplines/**", "docs/specs/**", "docs/adr/**"]
    write_except: ["@human_only"]
    read_only: ["@human_only"]
```

Rules:

1. `harness-author` exists only in the harness's own `.factory/roles.yaml`. No pack's
   `roles.yaml` may define it or grant any harness path (A-001).
2. Its PRs need H8 review like every other harness PR. Its changes take effect only once
   released and pinned.
3. The run itself uses the profiles of the harness version pinned in the harness's
   `.factory/pin.yaml`. The agent edits a *checkout* of the harness. It never edits the copy its
   own instructions came from.
4. New ADRs may be proposed with status `Proposed`. Only humans change an ADR to `Accepted`.

This requires agent-gate to support a repository-level roles file. That is a dependency on
agent-gate L2, recorded in [assumptions.md](assumptions.md).

## Open items for consumers

- **Manifest authorship.** The oracle identity's signing key cannot be inside the microVM
  (no secrets). Proposed: after the `unit-oracle` agent finishes, sandbox-runner runs
  `oracle-guard lock` outside the agent's control and signs with the oracle role's key, held in
  the KMS. Owner: oracle-guard and sandbox-runner.
- **What zone B sees at T0–T1.** Zone B never sees implementation source, so unit-oracle writes
  tests against `spec/contracts/**` only. At L1 humans write those contracts. Whether the oracle
  may also see a stub-only snapshot of `src/`, so that it can compile its tests, is an open
  question (see [assumptions.md](assumptions.md), AS-07).
