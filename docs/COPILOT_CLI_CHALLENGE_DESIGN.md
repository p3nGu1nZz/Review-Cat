# ReviewCat: Autonomous Self-Improving Code Review Daemon

This is the primary design doc for ReviewCat — an autonomous, self-improving
code review and development daemon powered by GitHub Copilot CLI and the
GitHub MCP Server.

> **See also:** [PLAN.md](../PLAN.md) for the comprehensive project plan and
> [TODO.md](../TODO.md) for the actionable task list.

Design principles:

- **Circular self-improvement** — ReviewCat reviews itself, creates issues,
  fixes them via coding agents, merges PRs, and repeats indefinitely.
- **GitHub as coordination layer** — All work is tracked via GitHub Issues
  and Pull Requests. Agents communicate through comments and labels.
- **GitHub MCP Server** — Agents use MCP tools for issue/PR operations,
  replacing raw `gh` CLI calls for dev agents.
- **Parallel worktrees** — Multiple agents work simultaneously in isolated
  git worktrees in the same parent directory.
- **Copilot CLI subprocess** — Copilot CLI is invoked via subprocess calls
  from both bash scripts (dev harness) and C++ (runtime app).
- **Safe by default** — Dry-run, allowlists, explicit opt-in for mutations.
  All changes go through PRs, never direct main commits.
- **Two-part architecture** — `app/` (the product) and `dev/` (the meta-tooling).
- **Bootstrap-first** — Hardcoded scripts cold-start the Director daemon;
  the self-improvement loop progressively builds the rest.
- **WSL-native** — Developed and tested on WSL/Linux with Copilot CLI.

## Problem statement

Developers want fast, repeatable, high-signal code review feedback before
pushing a PR. Today, Copilot CLI can help review code, but the workflow is
ad-hoc: prompt quality varies, outputs are inconsistent, and review artifacts
are not stored or comparable over time. Furthermore, no existing tool closes
the loop from review finding to automated fix.

ReviewCat turns Copilot CLI into an opinionated, repeatable code-review
workflow that produces:

- structured persona findings
- a single unified review
- an actionable change plan
- automated coding agent fixes via GitHub Issues and PRs
- an auditable record of how Copilot CLI was used
- a circular self-improvement loop that runs indefinitely

## Goals

Product goals:

- Provide a high-quality, repeatable review experience on:
  - local diffs
  - GitHub pull requests
  - the ReviewCat codebase itself (self-review)
- Close the review→fix loop by dispatching coding agents that implement
  fixes and create PRs from review findings.
- Create artifacts that are:
  - readable (markdown)
  - machine-parseable (JSON)
  - diffable and archivable
- Run autonomously as a daemon, self-improving indefinitely.

Development goals:

- Use Copilot CLI meaningfully during development and at runtime.
- Use GitHub MCP Server for agent coordination (issues, PRs, comments).
- Demonstrate parallel agent execution via git worktrees.
- Bootstrap from minimal hardcoded scripts to a full self-building system.

## Non-goals

- Fully replacing human review.
- Auto-merging changes without validation (build + test gate required).
- Requiring external cloud services beyond GitHub (everything runs locally).
- Shipping without a UI (a lightweight native window is part of the product).

The UI is a lightweight native window built with **SDL3** (window/input) and a
**custom ToolUI** (draw primitives + bitmap glyph font) providing settings,
stats, and command-and-control — not a full GUI-first product.

For dev harness operations, an optional **swarm visualizer** mode can render a
live task graph driven by agent-bus telemetry.

## Target users

- Solo developers who want a fast pre-PR review.
- Maintainers who want structured triage on incoming PRs.
- Teams that want consistent, persona-based feedback and an artifact trail.
- The ReviewCat project itself (self-review and self-improvement).

## User experience

### Command surface

ReviewCat is a terminal-first CLI.

- `reviewcat demo`
  - Runs on a bundled sample diff.
  - Produces a full audit directory.
  - Requires no GitHub authentication.

- `reviewcat review [--base <ref>] [--head <ref>] [--include <glob>] [--exclude <glob>]`
  - Reviews a local git diff.
  - Writes artifacts under `docs/audits/<audit_id>/`.

- `reviewcat pr <pr_url|number> [--repo OWNER/REPO] [--comment]`
  - Fetches PR diff via GitHub MCP Server.
  - Generates artifacts.
  - Optionally posts a unified comment.

- `reviewcat fix [--apply] [--branch <name>] [--pr] [--run-tests]`
  - Generates patch suggestions.
  - Optionally applies them.
  - Optionally opens a PR.

- `reviewcat watch [--branch <name>] [--interval <seconds>]`
  - Polls and runs `review` on new commits.
  - Dispatches coding agents for critical/high findings.
  - Circular: review → issues → coding agent → PR → merge → review.

- `reviewcat dev director`
  - Runs the DirectorDev daemon that coordinates Copilot CLI agents to build
    and improve the ReviewCat codebase itself.
  - This is the "self-building" mode — the circular self-improvement loop.
  - The Director runs as a persistent daemon with a heartbeat loop.
  - See `PLAN.md §5` for heartbeat architecture.

### Output and artifacts

Each run creates a directory:

`docs/audits/<audit_id>/`

Minimum artifacts:

- `audit.json` (machine-readable record)
- `unified_review.md` (human-readable)
- `action_plan.json` (issue-ready tasks)
- `persona/` (one JSON per persona)
- `ledger/copilot_prompts.jsonl` (exact Copilot CLI calls)
- `ledger/copilot_raw_outputs/` (raw text output per call)

Summary behavior:

- Print a one-screen summary at the end:
  - top 5 findings
  - GitHub Issues created (if any)
  - next command suggestions
  - where artifacts were written

## Core review workflow

1. Collect input
   - Determine repo root.
   - Determine diff range:
     - default: `git diff --merge-base origin/main...HEAD`
     - configurable `--base` and `--head`.
   - Apply include/exclude filters.

2. Build review context
   - For each changed file:
     - include diff hunks
     - optionally include small surrounding context windows
   - Chunk content to stay under prompt budgets.

3. Persona passes
   - Run a set of persona prompts via Copilot CLI programmatic mode.
   - Validate responses against a strict JSON schema.
   - If validation fails, re-prompt with a repair prompt.

4. Synthesis
   - Deduplicate findings.
   - Prioritize.
   - Produce unified markdown review.
   - Produce action plan JSON.

5. Issue creation and coding agent dispatch
   - For critical/high findings, create GitHub Issues via GitHub MCP Server.
   - Dispatch coding agents to worktrees to implement fixes.
  - Coding agents create worker PRs that reference issues (`Refs #N`) and target
    the active release branch; the release PR aggregates `Closes #...` when it
    merges to `main`.

6. Persist audit
   - Write `audit.json`.
   - Update `docs/audits/index.json`.

7. Circular continuation
   - After merges, review again to find new issues or verify fixes.
   - Loop indefinitely while daemon is active.

## Architecture (components, entities, systems)

The repository is split into two top-level sections:

- **`app/`** — The ReviewCat product (CLI, daemon, UI, runtime agents).
- **`dev/`** — The development harness (Director daemon, role agents,
  coding agents, worktree management, GitHub MCP coordination).

To match an ECS-style separation and keep modules testable:

- Components are pure data structures.
- Entities are factories that construct validated instances of components.
- Systems contain all logic and side-effects.

### Components (pure data)

Representative components:

- `RunConfig`
- `ReviewInput`
- `PersonaDefinition`
- `PersonaFinding`
- `UnifiedFinding`
- `UnifiedReview`
- `ActionItem`
- `AuditRecord`
- `PromptRecord`

### Entities (factories)

Representative entities:

- `AuditIdFactory` (timestamp + commit hash)
- `PromptFactory` (persona template + chunk injection)
- `PatchPlanFactory` (suggested diffs gated by policy)

### Systems (logic)

Representative systems:

- `RepoDiffSystem` (collect diffs, file lists, context)
- `CopilotRunnerSystem` (invoke Copilot CLI with MCP and guardrails)
- `PersonaReviewSystem` (persona loop and JSON validation)
- `SynthesisSystem` (dedupe, prioritize, unify)
- `CodingAgentSystem` (dispatch coding agents to fix findings)
- `PatchApplySystem` (optional, safe patch application)
- `GitHubOpsSystem` (GitHub MCP Server for issues/PRs/comments)
- `AuditStoreSystem` (write artifacts, update index, archive)
- `DirectorRuntimeSystem` (orchestrate end-to-end review + fix flow)

## Copilot CLI + GitHub MCP integration

ReviewCat uses Copilot CLI as the analysis and generation engine, with
the **GitHub MCP Server** providing GitHub API capabilities.

### Invocation mode

- **CLI subprocess mode** — Invoke `copilot -p "..."` as a subprocess.
  - From **bash scripts** (dev harness): direct `copilot -p` calls with MCP.
  - From **C++ binary** (runtime app): `popen`/`fork+exec` subprocess wrapper.
- **GitHub MCP Server** — Configured via `--mcp-config` to give agents
  access to GitHub tools (create_issue, create_pull_request, add_issue_comment,
  list_issues, get_pull_request, etc.).
- Plan mode for complex end-to-end tasks (optional).
- No SDK or npm dependency — Copilot CLI + GitHub MCP Server (native binary
  or remote) are the only external requirements.

### GitHub MCP Server configuration

The GitHub MCP Server (`github/github-mcp-server`) provides MCP tools for
agents to interact with GitHub:

- **Toolsets:** `issues`, `pull_requests`, `repos`, `git`
- **Authentication:** `GITHUB_PERSONAL_ACCESS_TOKEN` env var
- **Launch:** Native binary (`github-mcp-server stdio`) or remote server
  (`https://api.githubcopilot.com/mcp/`). In the Docker-first dev harness model,
  agents run inside worker containers, while the MCP server can be remote (preferred)
  or a native stdio binary on the host / bundled in the container image.
- **Agent integration:** `copilot -p "..." --mcp-config github-mcp.json`

This replaces raw `gh` CLI for development agents. The `gh` CLI remains as
a fallback and for simple operations in the C++ runtime app.

### Guardrails

- Default is dry-run.
- Prefer minimal allowlists for tools.
- Explicitly deny destructive commands by default.
- Worktree isolation — agents cannot modify the main worktree directly.
- PR-gated merges — all changes go through PRs, never direct main commits.
- Label-based claiming — agents claim issues before starting work to avoid
  duplicate effort across parallel workers.

ReviewCat treats safety as part of UX: users can opt into broader permissions,
but the tool should make the risks clear.

### Prompt ledger

Every Copilot CLI interaction is recorded as an append-only log.

This is critical for auditability:

- reviewers can see Copilot CLI usage clearly
- runs can be reproduced deterministically
- improvements can be made to prompts empirically

## Self-improving development workflow (DirectorDev)

A core differentiator is that ReviewCat is designed to **build and improve
itself indefinitely** through a circular self-improvement loop.

The project includes a DirectorDev workflow that coordinates multiple Copilot
CLI agents to handle typical software project responsibilities. All development
tooling lives under `dev/`. Agents communicate via **GitHub Issues and PRs**.

### Concept

- **Agent 0 (Director)** runs as a persistent daemon with a heartbeat loop
  (see `PLAN.md §5`).
- The Director owns:
  - scope control
  - planning and task decomposition
  - worktree management (parallel agents)
  - acceptance criteria
  - integration quality (build/test gate)
  - progress tracking (GitHub Issues + PRD)
- Specialized role agents execute sub-tasks:
  - architect, implementer, coder, QA, docs, security, code-review
- The **Coder agent** closes the review→fix loop by reading GitHub Issues,
  implementing fixes in worktrees, and creating PRs.

The Director recursively delegates until a feature is complete, then reviews
its own changes and generates new issues — indefinitely.

### Self-improvement loop

```
┌────────────────────────────────────────────────────────────┐
│                  Self-Improvement Loop                      │
│                                                            │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │  Review  │───▶│  Create  │───▶│  Coding  │             │
│  │  Own     │    │  GitHub  │    │  Agent   │             │
│  │  Code    │    │  Issues  │    │  Fixes   │             │
│  └──────────┘    └──────────┘    └────┬─────┘             │
│       ▲                               │                    │
│       │          ┌──────────┐         │                    │
│       └──────────│  Merge   │◀────────┘                    │
│                  │  PRs     │                              │
│                  └──────────┘                              │
│                                                            │
│  This loop runs continuously while the daemon is active.  │
└────────────────────────────────────────────────────────────┘
```

### Parallel execution via worktrees

Multiple agents work simultaneously in isolated git worktrees:

```
~/source/repos/
├── Review-Cat/                    # Main worktree (Director runs here)
├── Review-Cat-agent-42-1707600000/ # Worker 1 (implementing issue #42)
├── Review-Cat-agent-57-1707600060/ # Worker 2 (implementing issue #57)
└── Review-Cat-agent-63-1707600120/ # Worker 3 (implementing issue #63)
```

- Each worktree has its own branch, build directory, and test artifacts.
- Director manages up to `MAX_WORKERS` concurrent worktrees.
- Communication between agents is via GitHub Issue/PR comments, not files.
- Worktrees are torn down after PRs are merged.

### Inter-agent communication via GitHub

| Mechanism | Purpose | Example |
|-----------|---------|---------|
| **Issues** | Work items and findings | Review agent creates issue for a bug |
| **Issue comments** | Discussion and clarification | Architect comments on complexity |
| **PRs** | Code implementations | Coder agent creates PR fixing issue |
| **PR comments** | Review feedback | Code-review agent comments on PR |
| **Labels** | Categorization and routing | `agent-task`, `security`, `in-progress` |
| **Linked issues** | Traceability | Worker PR says "Refs #42"; release PR aggregates "Closes #..." |

**Label taxonomy:**

| Label | Meaning |
|-------|---------|
| `agent-task` | Available for agent pickup |
| `agent-claimed` | Currently being worked on |
| `agent-review` | Needs review from another agent |
| `agent-blocked` | Needs human input |
| `auto-review` | Created by self-review process |
| `security` / `performance` / `architecture` / `testing` / `docs` | Persona |
| `priority-critical` / `priority-high` / `priority-medium` / `priority-low` | Severity |

### How this maps to Copilot CLI features

- Custom agents are defined in `.github/agents/` (repository-level).
- Copilot CLI is invoked via subprocess (`copilot -p`) from bash scripts.
- GitHub MCP Server is configured via `--mcp-config` (native binary or remote).
- The heartbeat daemon runs agents in a repeatable, auditable loop.
- Record/replay mode enables deterministic testing without live Copilot calls.

### DirectorDev heartbeat loop

The Director daemon runs continuously with a configurable interval:

1. **Wake** — Heartbeat timer fires.
2. **Scan** — List open issues labeled `agent-task` via GitHub MCP.
3. **Check PRD** — Read `dev/plans/prd.json` for spec-driven work.
4. **Claim** — For each unclaimed issue, add `agent-claimed` label.
5. **Dispatch** — Create worktree, spawn agent to work on the issue.
6. **Monitor** — Check worktree workers for completion.
7. **PR** — Agent creates PR via GitHub MCP, linking the issue.
8. **Review** — Code-review agent reviews the PR via GitHub MCP.
9. **Validate** — Run `./scripts/build.sh && ./scripts/test.sh` in worktree.
10. **Merge** — If validation passes, merge the worker PR into the active release
  branch.
11. **Teardown** — Remove worktree.
12. **Self-review** — When idle, review own code and create new issues.
13. **Sleep** — Wait for next heartbeat interval.

### Bootstrap sequence

To cold-start the Director for the first time:

```bash
cd Review-Cat
./dev/scripts/setup.sh        # Install system prereqs (gh, jq, github-mcp-server)
./dev/scripts/bootstrap.sh    # Configure MCP, create issues, verify build
./dev/harness/daemon.sh       # Start keep-alive supervisor (starts Director loop)
```

`setup.sh` installs:
- gh CLI, jq, github-mcp-server binary (from GitHub Releases)
- Verifies copilot, cmake, g++ are available
- Prompts for GITHUB_PERSONAL_ACCESS_TOKEN if unset

`bootstrap.sh` performs:
- Verify setup.sh prerequisites are met
- Configure GitHub MCP Server for Copilot CLI
- Create `dev/plans/prd.json` with initial bootstrap tasks
- Create initial GitHub Issues for Phase 0 tasks
- Set up labels on the repo
- Run initial self-review to seed first issues

### Development audit artifacts

DirectorDev generates auditable evidence for every cycle:

- Prompt ledger (every Copilot CLI subprocess call logged)
- Before/after diffs
- Build/test output logs
- Agent output transcripts
- GitHub Issue and PR links

Artifacts are stored under `dev/audits/<audit_id>/`.

## Implementation plan

> **Canonical plan:** See [PLAN.md](../PLAN.md) §9 for the phased plan
> and [TODO.md](../TODO.md) for the full task list.

The implementation language is **C++17/20**. The dev harness is **bash shell
scripts**. Copilot CLI + GitHub MCP Server are the primary external deps.

### Phase 0: Bootstrap & dev harness (PRIORITY)

- Create `app/` and `dev/` directory structure.
- Write `dev/scripts/setup.sh` (install system prereqs: gh, jq,
  github-mcp-server binary).
- Write `dev/scripts/bootstrap.sh` (configure MCP, create initial issues,
  set up labels, verify build).
- Write Director daemon, run-cycle, worktree, review-self, and audit scripts.
- Set up CMake build system for C++ project.
- Define agent profiles in `.github/agents/` and `dev/agents/`.

### Phase 1: Self-review loop (self-improvement begins)

- Implement self-review: persona agents review own code, create issues.
- Implement coding agent: read issues, fix in worktrees, create PRs.
- Implement Director merge logic: validate → merge → teardown.
- Verify circular loop: review → issue → fix → PR → merge → review.

### Phase 2: Core app skeleton (C++)

- CLI frontend with command stubs.
- Config, audit, prompt ledger components.
- `reviewcat demo` with bundled sample diff.

### Phase 3: Review pipeline (C++)

- RepoDiffSystem, CopilotRunnerSystem, PersonaReviewSystem, SynthesisSystem.
- CodingAgentSystem for automated fix dispatch.
- `reviewcat review` end-to-end.

### Phase 4: GitHub integration (runtime)

- GitHubOpsSystem via GitHub MCP Server for runtime app.
- `reviewcat pr`, `reviewcat watch` with circular review→fix loop.
- Support remote user repos (not just self-review).

### Phase 5: Patch automation

- PatchApplySystem, `reviewcat fix`.

### Phase 6: End-user UI

- SDL3 + custom ToolUI surfaces: dashboard, agent status panel, settings, stats,
  audit log, log viewer, daemon controls (optionally including a swarm visualizer view).
  Include a top/bottom status bar and explicit UI layer/z-index handling.

### Phase 7: Polish and distribution

- Single static binary, error handling, spdlog logging (active from Phase 0),
  documentation.

## Testing strategy

- Unit tests:
  - schema validation
  - chunking
  - synthesis dedupe
  - audit serialization

- Integration tests:
  - record/replay mode that stubs Copilot CLI responses
  - demo snapshot tests (golden files)
  - GitHub MCP mock server for issue/PR operations

This ensures the project remains testable even if Copilot CLI or GitHub MCP
Server are unavailable.

## Security and privacy

- Do not read or upload secrets.
- `GITHUB_PERSONAL_ACCESS_TOKEN` is set via environment, never stored in code.
- Redact sensitive file globs.
- Avoid default network actions.
- PR-gated merges: no direct pushes to main.
- Require explicit opt-in for:
  - posting comments
  - creating issues
  - opening PRs
  - auto-merging

## Risks and mitigations

- Prompt budget limits → chunking and summarization
- Copilot CLI request quotas → caching per chunk/persona; record/replay tests
- Worktree conflicts → branch naming conventions; label-based claiming
- Infinite trivial self-review issues → severity threshold; dedup against open issues
- MCP Server unavailability → three deployment options (remote, binary, source);
  fallback to `gh` CLI; graceful degradation
- Parallel agent race conditions → issue claiming via labels; branch naming
- Agent produces bad code → build/test validation gate; PR review before merge

## Distribution checklist

- Repo contains:
  - Clear README with quick start and installation steps
  - Demo mode (`reviewcat demo`)
  - Sample output artifacts
  - Prompt ledger evidence of autonomous development
  - GitHub Issues/PRs showing self-improvement history
  - Comprehensive specs in `docs/specs/`

- Package includes:
  - Single static binary (`reviewcat`) — no runtime dependencies
  - SDL3 + custom ToolUI built into the binary (`reviewcat ui`)
  - Default config template (`reviewcat.toml`)
  - Bootstrap scripts for self-improvement setup
