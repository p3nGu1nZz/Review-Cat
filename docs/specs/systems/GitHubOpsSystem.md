# GitHubOpsSystem

## Overview

`GitHubOpsSystem` integrates ReviewCat with GitHub via the **GitHub MCP Server**
for rich issue/PR operations, with `gh` CLI as a fallback.

This system serves two distinct contexts:

1. **Development context** — Dev agents use it to create/read/manage issues,
   PRs, and comments on the ReviewCat repo itself during self-improvement.
2. **Runtime context** — The compiled app uses it to create issues, PRs, and
   comments on the end user's target repository.

## Requirements

1. Must create, read, update, and comment on GitHub Issues.
2. Must create, read, review, and merge Pull Requests.
3. Must manage labels on issues and PRs.
4. Must fetch PR metadata and diffs.
5. Must post unified review comments on PRs.
6. Must never perform GitHub mutations unless explicitly enabled.
7. Must support GitHub MCP Server as primary integration.
8. Must fall back to `gh` CLI if MCP Server is unavailable.
9. Must respect swarm-level backpressure and rate-limit policy (see `RequestBudgetSystem`).
10. Must support the Director’s **issue-claim lock protocol** (label transitions + claim comment).

Canonical label taxonomy and claim-comment format: `docs/dev/GITHUB_LABELS.md`.

## Interfaces

### Issue operations (via GitHub MCP Server)
- `create_issue(repo, title, body, labels) -> issue_number`
- `get_issue(repo, issue_number) -> issue_data`
- `list_issues(repo, labels, state) -> issues[]`
- `add_issue_comment(repo, issue_number, body) -> comment_url`
- `update_issue(repo, issue_number, labels, state) -> void`

#### Director-only lock operations

These operations are conceptually higher-level; they are implemented using the primitives above.

- `claim_issue_lock(repo, issue_number, run_id, worker_id, release_branch?) -> void`
  - MUST remove `agent-task`
  - MUST add `agent-claimed`
  - MUST add a `ReviewCat-Claim:` comment (see `docs/dev/GITHUB_LABELS.md`)
- `reclaim_issue_lock(repo, issue_number, reason) -> void`
  - SHOULD add a `ReviewCat-Reclaim:` comment
  - SHOULD remove `agent-claimed`
  - SHOULD add `agent-task` (or `agent-blocked` if escalation is required)

### PR operations (via GitHub MCP Server)
- `create_pull_request(repo, head, base, title, body) -> pr_number`
- `get_pull_request(repo, pr_number) -> pr_data`
- `merge_pull_request(repo, pr_number, method) -> void`
- `add_pull_request_review_comment(repo, pr_number, body) -> void`
- `fetch_pr_diff(repo, pr_number) -> diff_text`

### Fallback (via `gh` CLI)
- `gh issue create`, `gh issue list`, `gh issue comment`
- `gh pr create`, `gh pr diff`, `gh pr comment`, `gh pr merge`

## Acceptance criteria

- Without explicit enablement, system performs no writes to GitHub.
- If GitHub MCP Server is unavailable, falls back to `gh` CLI gracefully.
- If `gh` is also missing or unauthenticated, system fails gracefully and
  suggests `gh auth login`.
- Created issues have correct labels from the label taxonomy.
- Issue claiming uses the lock protocol:
  - label transitions are correct (`agent-task` -> `agent-claimed`)
  - claim comment is present and machine-parseable
- Created PRs link issues for traceability:
  - worker PRs use `Refs #N`
  - release PRs aggregate `Closes #...` so issues close on merge to `main`.

## Test cases

- Replay mode stubs MCP/gh calls.
- Verify issue creation includes correct labels.
- Verify PR creation includes issue linkage.
- Verify fallback from MCP to `gh` CLI.

## Edge cases

- PR from fork.
- Large PR diff.
- Rate limiting from GitHub API (including burst/abuse protection).
- GitHub MCP Server timeout.
- Token expired or insufficient permissions.

## Related specs

- `docs/specs/systems/RequestBudgetSystem.md` — global pacing/backpressure for GitHub and model calls.

## Non-functional constraints

- No token storage in code; use environment variables.
- Redact sensitive content before posting.
- `GITHUB_PERSONAL_ACCESS_TOKEN` for MCP; `gh auth` for CLI fallback.
- MCP Server is configured via either remote MCP or a native `github-mcp-server stdio` binary (host or container-bundled).
- Three deployment options: remote server, pre-built binary, build from source.
  See PLAN.md §3.1.
