# ADR-0003: YAML authoring, JSON Schema 2020-12 validation

**Status:** Proposed
**Date:** 2026-11-27

## Context

Harness content is written by humans (L1) and by design-zone agents (L2+), read by services
written in different languages, and validated in CI without bespoke code (HR-MN-02). The
handbook's illustrative stage contract is YAML referencing `outputs/*.schema.json`.

## Decision

- Definitions are authored in YAML 1.2 (JSON-compatible subset: no anchors, aliases, merge
  keys or custom tags), prose in Markdown.
- Schemas are JSON Schema draft 2020-12, one file per format, with
  `$id: urn:brightsmiths:harness:<name>:<major>` and `additionalProperties: false` at every
  object level.
- Every YAML instance declares its schema in a top-level `schema:` field holding that `$id`.

## Consequences

- Off-the-shelf validators in any language can check every file; CI needs no harness code.
- Restricting YAML features avoids parser differences between consumers.
- `additionalProperties: false` makes typos fail loudly but means adding a field is a schema
  change (minor version if optional).

## Alternatives considered

- **CUE or Dhall:** stronger cross-file validation, but adds a toolchain every consumer must
  understand.
- **TOML:** poor fit for nested graphs and lists of checks.
- **JSON only:** valid, but less readable for prose-heavy fields and reviews.
