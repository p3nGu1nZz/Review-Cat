# Role Agents (Custom Copilot CLI Agents)

## Overview

ReviewCat uses a set of custom Copilot CLI agents to represent development
roles. These agents are used by the DirectorDev workflow and communicate
via GitHub Issues/PRs through the GitHub MCP Server.

## Requirements

1. Each role agent must have a clear mandate and refusal rules.
2. Agents must be runnable via Copilot CLI by name.
3. Agents must follow repository instructions and testing requirements.
4. Agents must use GitHub MCP Server tools for issue/PR operations.
5. Agents must work in isolated git worktrees when making code changes.

## Interfaces

Repository-level agent profiles live in:

- `.github/agents/<agent_name>.md`
- `dev/agents/<agent_name>.md`

### Agent roster:

| Agent | Profile | Purpose |
|-------|---------|---------|
| `director-dev` | Orchestrator | Manages all other agents, worktrees, issues, PRs |
| `architect` | Design reviewer | Reviews architecture changes, complexity |
| `implementer` | Code writer | Writes code for spec-driven work |
| `coder` | Fix implementer | Reads GitHub Issues, implements fixes, creates PRs |
| `qa` | Test engineer | Writes tests, adds record/replay fixtures |
| `docs` | Documentation | Maintains README, examples, specs |
| `security` | Security auditor | Enforces safe defaults, redaction, permissions |
| `code-review` | Code reviewer | Reviews PR diffs, blocks low-signal changes |
| `merge-expert` | Release merge agent | Merges release PRs into `main`, resolves conflicts, verifies tag |

Each agent profile should specify:

- mission
- output format
- allowed tools guidance
- testing requirements
- GitHub MCP tools the agent may use

## GitHub MCP Integration

Agents interact with GitHub through the MCP Server:

- **Coder agent**: `create_pull_request`, `get_issue`, `add_issue_comment`
- **Code-review agent**: `create_pull_request_review`, `add_pull_request_review_comment`
- **Security agent**: reads code via MCP, reports findings as issue comments
- **Director**: `list_issues`, `create_issue`, `merge_pull_request`, label management

## Acceptance criteria

- DirectorDev can run a full cycle using these agents in parallel worktrees.
- Agents do not produce scope creep without Director approval.
- Coder agent can read an issue, implement a fix, and create a PR end-to-end.
- Code-review agent can review a PR and post feedback.
- All agents record their Copilot CLI interactions in the prompt ledger.

## Test cases

- DirectorDev replay mode covers orchestration.
- Verify each agent produces structured output.
- Verify coder agent creates PRs linking issues.

## Edge cases

- Conflicting guidance from different agents.
- GitHub MCP Server unavailable (agents should gracefully degrade).
- Agent produces output exceeding prompt budget.

## Non-functional constraints

- Safety: agents must not recommend storing secrets.
- Determinism: agent output should be structured where possible.
- Isolation: agents work in worktrees, not the main checkout.
