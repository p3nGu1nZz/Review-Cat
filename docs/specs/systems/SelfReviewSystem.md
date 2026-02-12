# SelfReviewSystem

## Overview

`SelfReviewSystem` implements the self-review capability where ReviewCat
reviews its own codebase, identifies issues, and creates GitHub Issues for
critical/high findings. This is the engine that drives the circular
self-improvement loop.

This system is invoked by the Director daemon when no other work is available,
or on a configurable schedule.

Self-review is intended to run **unattended** as part of the zero-touch loop;
it should not require interactive prompts.

## Requirements

1. Must generate a diff of recent changes (e.g., `HEAD~5..HEAD` or full codebase).
2. Must run persona review agents (security, performance, architecture,
   testing, docs) on the diff via CopilotRunnerSystem.
3. Must parse findings from each persona into structured JSON.
4. Must filter findings by severity threshold (default: critical + high).
5. Must create GitHub Issues for qualifying findings via GitHubOpsSystem.
6. Must add appropriate labels (`auto-review`, persona, priority).
7. Must deduplicate against existing open issues to prevent flooding.
8. Must record all interactions in the prompt ledger.

## Interfaces

- `run_self_review() -> findings[]`
- `create_issues_from_findings(findings) -> issue_numbers[]`
- `deduplicate(findings, existing_issues) -> filtered_findings[]`

## Workflow

1. Generate diff of recent changes.
2. For each persona (security, performance, architecture, testing, docs):
   - Invoke persona review agent via CopilotRunnerSystem.
   - Parse output as JSON array of findings.
3. Filter findings by severity threshold.
4. List existing open issues labeled `auto-review`.
5. Deduplicate: skip findings with titles similar to existing issues.
6. For each remaining finding:
   - Create GitHub Issue via GitHubOpsSystem.
   - Add labels: `agent-task`, `auto-review`, `<persona>`, `priority-<severity>`.

## Acceptance criteria

- Self-review produces a list of typed findings.
- Findings above severity threshold become GitHub Issues.
- Duplicate findings do not create duplicate issues.
- All persona agents run and produce parseable output.

## Test cases

- Replay mode with canned persona outputs.
- Verify deduplication against existing issues.
- Verify label application on created issues.
- Verify severity threshold filtering.

## Edge cases

- All findings are low/nit severity (no issues created).
- Persona agent produces malformed JSON.
- GitHub MCP Server unavailable during issue creation.
- Very large diff exceeds prompt budget (chunking required).
- No recent changes (empty diff).

## Non-functional constraints

- Severity threshold prevents infinite trivial issue loops.
- Deduplication uses fuzzy title matching to catch near-duplicates.
- All Copilot CLI interactions logged to prompt ledger.
- Self-review should complete within a reasonable time budget.
