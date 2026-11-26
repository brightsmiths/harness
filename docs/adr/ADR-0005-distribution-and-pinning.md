# ADR-0005: Distribution by git tag; pin by tag and commit SHA

**Status:** Proposed
**Date:** 2026-11-27

## Context

Consumers must fetch an exact, immutable harness version. Brightsmiths runs anywhere (cloud or
on-prem) and avoids cloud-specific dependencies. Chain of custody requires signatures: Ed25519
today, in-toto/DSSE through chain-chronicler later. Git tags can be moved by anyone with push
rights.

## Decision

- A harness release is a git tag `vX.Y.Z` with a GitHub Release.
- `.factory/pin.yaml` records both `version` and the full 40-character `commit` SHA. Consumers
  fetch by SHA and check that the tag still resolves to it. A mismatch is `PIN_INVALID`
  (fail closed).
- From L2 (HR-L2-07), each release also has a deterministic `tar` archive of the tree, its
  SHA-256 digest and an Ed25519 signature by the harness release key. The pin may record
  `archive_digest`; consumers that use the archive verify the signature and digest. When
  chain-chronicler L1 is available, the signature becomes a DSSE attestation.
- Tags matching `v*` are protected by ruleset: no deletion, no update.

## Consequences

- Works with any git server and air-gapped mirrors; no registry needed.
- Pin updates are explicit, human-owned PRs in target repositories.
- Signing key custody becomes an operational concern (KMS or Vault; see
  [security.md](../specs/security.md)).

## Alternatives considered

- **OCI artifacts:** good signing tooling, but needs a registry in every deployment.
- **Tag only:** simplest, but a moved tag silently swaps rules.
- **Git submodule in each target repo:** couples harness fetches to target-repo checkout and
  puts harness files inside agent-visible workspaces.
