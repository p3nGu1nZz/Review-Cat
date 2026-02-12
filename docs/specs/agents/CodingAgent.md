# Coding Agent Spec

## Overview

The Coding Agent (Coder) closes the review→fix loop in ReviewCat's circular
self-improvement workflow. It reads a GitHub Issue (typically created by a
review agent), understands the problem, implements a fix in an isolated git
worktree, writes tests, and creates a Pull Request via GitHub MCP Server.

This agent is invoked via `copilot -p @dev/agents/coder.md "..."` with
GitHub MCP Server configured for issue/PR operations.

## Requirements

1. Must read issue context via GitHub MCP Server tools (`get_issue`).
2. Must work exclusively in an isolated git worktree (never modify main).
3. Must create a fix branch following the naming convention `fix/<issue>-<desc>`.
4. Must implement a fix that addresses the issue description.
5. Must write tests that validate the fix (Catch2 for C++, or appropriate for the change).
6. Must ensure `./scripts/build.sh && ./scripts/test.sh` pass before creating a PR.
7. Must create a PR via GitHub MCP Server linking the issue:
  - worker PRs SHOULD use `Refs #<issue>` (issues close when the release PR is merged to `main`)
  - if release cycles are disabled and the PR targets `main` directly, the PR MAY use `Closes #<issue>`
8. Must add the `agent-review` label to the PR to trigger code-review.
9. Must record all Copilot CLI interactions in the prompt ledger.
10. Must be non-interactive: no local prompts for clarification; if issue context
  is insufficient, ask via GitHub issue comment and apply `agent-blocked`.

## Interfaces

- Invocation: `copilot -p @dev/agents/coder.md "Fix issue #<N>"`
- GitHub MCP tools used:
  - `get_issue` — read issue title, body, labels
  - `create_pull_request` — create PR from fix branch
  - `add_issue_comment` — post progress updates
  - `update_issue` — add/remove labels
- Input: GitHub Issue number + worktree path
- Output: Code changes + tests + PR

## Workflow

1. **Read issue** — Use GitHub MCP to get issue details (title, body, labels).
2. **Understand context** — Read referenced files and code via the worktree.
3. **Create branch** — `git checkout -b fix/<issue>-<short-desc>`.
4. **Implement fix** — Write C++17/20 code following project conventions.
5. **Write tests** — Add Catch2 test cases validating the fix.
6. **Build and test** — Run `./scripts/build.sh && ./scripts/test.sh`.
7. **Commit** — `git add -A && git commit -m "fix(#<issue>): <description>"`.
8. **Push** — `git push origin fix/<issue>-<short-desc>`.
9. **Create PR** — Via GitHub MCP: title `fix(#<issue>): <desc>`, body `Refs #<issue>`.
10. **Label PR** — Add `agent-review` label to trigger code-review agent.

## Acceptance criteria

- Given a GitHub Issue describing a bug or improvement:
  - Coder produces code changes that address the issue.
  - Coder produces tests that validate the fix.
  - Build and test pass in the worktree.
  - A PR is created via GitHub MCP linking the issue.
  - The PR includes `Refs #<issue>` (the release PR is responsible for closing issues on merge to `main`).

## Test cases

- Replay mode: given a fixture issue and canned Copilot responses, verify
  the agent creates the expected files and PR.
- Verify branch naming convention is followed.
- Verify PR body contains `Refs #<issue>`.

## Edge cases

- Issue description is vague or incomplete — agent should comment asking
  for clarification and add `agent-blocked` label.
- Fix requires changes across multiple modules — agent should scope the
  fix narrowly and note limitations in the PR description.
- Build or test fails — agent should retry up to 3 times, then add
  `agent-blocked` label to the issue with failure details.
- GitHub MCP Server unavailable — agent should fall back to `gh` CLI.

## Non-functional constraints

- Safety: never modify files outside the worktree.
- Scope: implement only what the issue describes; no unrelated changes.
- Determinism: structured commit messages and PR format.
- Auditability: all Copilot CLI interactions logged to prompt ledger.
