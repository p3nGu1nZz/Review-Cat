# Testing Strategy (ReviewCat)

This document defines how ReviewCat testing is expected to work so that autonomous agent changes can be validated reliably via a **green build/test gate**.

## Goals

- **Deterministic**: no flaky tests.
- **Fast**: suitable for running inside worker containers on every cycle.
- **Layered**: unit tests first, integration tests where needed, E2E tests sparingly.
- **Auditable**: failures should be obvious from logs/audit bundles.

## Test pyramid

1. **Unit tests** (most): pure logic, DTO validation, parsing, small components.
2. **Integration tests**: boundaries (git operations, filesystem, subprocess wrappers), but still hermetic.
3. **E2E tests** (least): full review pipeline scenarios; prefer record/replay.

## Framework: Catch2 (C++)

Catch2 is the planned test framework for `app/`.

### Conventions

- File naming: `app/tests/test_<area>.cpp`
- Test case naming: descriptive sentence (what/when/then)
- Tags: use stable tags like `[core]`, `[git]`, `[mcp]`, `[integration]`
- Fixtures live under: `app/tests/fixtures/`

## Determinism: record/replay

External calls must be recordable and replayable. This includes:

- Copilot CLI invocations
- GitHub operations (via MCP or gh fallback)

**Rule:** tests should default to **replay mode**.

A replay should capture:

- prompt/input
- invocation args
- stdout/stderr
- exit code
- (optional) duration and environment summary

Replays should be treated as golden fixtures and stored in-repo (small, scrubbed of secrets).

## Fixture organization (recommended)

```
app/tests/fixtures/
  configs/       # TOML configs
  diffs/         # sample patches / diffs
  replays/       # record/replay golden files
  repos/         # tiny seed repos for git tests (or generated at runtime)
```

## The green gate

The Director’s validation contract is:

- `./scripts/build.sh` then `./scripts/test.sh`

Until the build scaffold exists, these remain *planned* paths; once implemented, the gate must be:

- idempotent
- suitable for container execution
- fast enough for tight iteration

## CI expectations (future)

When CI is added, PRs must be blocked on:

- build success
- unit tests
- integration tests (where present)

## Cross-references

- Issue #8 (this document)
- `PLAN.md` (tech stack and phases)
- `TODO.md` (testing infrastructure tasks)
- `docs/specs/systems/CopilotRunnerSystem.md` (record/replay mode)
