# CodingAgentSystem

## Overview

`CodingAgentSystem` dispatches coding agents to implement fixes for review
findings or GitHub Issues. It manages the lifecycle from issue reading to
PR creation, working in isolated git worktrees.

This system is used both by the dev harness (self-improvement) and the
runtime app (fixing issues on user repos).

## Requirements

1. Must read GitHub Issue details via GitHubOpsSystem / GitHub MCP Server.
2. Must create an isolated git worktree for each fix.
3. Must invoke the coding agent via CopilotRunnerSystem.
4. Must run build + test validation before PR creation.
5. Must create a PR via GitHubOpsSystem linking the source issue.
6. Must add `agent-review` label to created PRs.
7. Must handle retry logic (max 3 attempts).
8. Must record all interactions in the prompt ledger.

## Interfaces

- `dispatch_fix(issue_number, worktree_path) -> pr_number | error`
- `validate_fix(worktree_path) -> bool` (runs build + test)
- `create_fix_pr(issue_number, branch, worktree_path) -> pr_number`

## Workflow

1. Read issue via GitHubOpsSystem.
2. Create branch `fix/<issue>-<short-desc>`.
3. Invoke coder agent in worktree via CopilotRunnerSystem.
4. Run `./scripts/build.sh && ./scripts/test.sh`.
5. On success: commit, push, create PR via GitHubOpsSystem.
6. On failure: retry up to 3 times, then mark `agent-blocked`.

## Acceptance criteria

- Given a GitHub Issue, produces code changes and a PR.
- Build and test pass before PR creation.
- PR body contains `Refs #<issue>` (issue closure is handled by the release PR when merged to `main`).
- Prompt ledger records all coding agent interactions.

## Test cases

- Replay mode with fixture issue and canned responses.
- Verify PR creation with correct linkage.
- Verify retry on build failure.
- Verify `agent-blocked` label on max retries exceeded.

## Edge cases

- Issue description too vague for implementation.
- Fix requires changes across unrelated modules.
- Worktree creation fails (e.g., branch name collision).
- Build system not yet set up (early bootstrap).

## Non-functional constraints

- Worktree isolation: never modify main worktree.
- Scope: implement only what the issue describes.
- Auditability: all interactions logged.
