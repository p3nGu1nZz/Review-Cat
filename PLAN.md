# ReviewCat: Comprehensive Project Plan

> **Golden Source of Truth** — This document, together with `TODO.md` and the
> specs under `docs/specs/`, defines what ReviewCat is, how it is built, and how
> it operates at runtime. All implementation must trace back to these docs.

## Status (repo reality check)

This repo is currently **docs/specs-first**.

- The *target* structure described below (`app/`, `dev/`, `scripts/`, `.github/`) is
  the intended end state, but the scaffold is not fully present yet.
- Phase 0’s first milestone is to create the minimal scaffold + a “green”
  build/test gate so autonomous cycles can run end-to-end.

Tracking issue: **#16 — _Bootstrap Repo Scaffold + Green Build/Test Gate_**.

## 1. Vision

ReviewCat is an **autonomous, self-improving code review and development daemon**
that:

1. **Reviews** code using persona-based Copilot CLI agents (security,
   performance, architecture, testing, docs).
2. **Synthesizes** findings into unified reviews, action plans, and GitHub Issues.
3. **Implements** fixes for review findings via coding agents that create PRs.
4. **Self-improves** by running its own review pipeline on itself, creating
   issues and PRs to iteratively enhance its own codebase.
5. **Provides** UI surfaces for settings, stats, and command-and-control (including an optional swarm visualizer).

The workflow is **circular and indefinite**: review → issues → coding agent
fixes → PR → merge → review again.

At the meta level (how the repo evolves), the loop is:

> **bootstrap → dev → app → dev → app → …**

This runs continuously while the daemon is active, enabling ReviewCat to
bootstrap from minimal hardcoded logic and progressively develop itself into a
full product.

### 1.1. Self-First MVP Strategy

The MVP focuses on ReviewCat reviewing and developing **itself** before
supporting remote user repositories:

1. **Phase A** — Bootstrap: Hardcoded scripts get the Director daemon running.
2. **Phase B** — Self-review: ReviewCat reviews its own code, creates issues.
3. **Phase C** — Self-fix: Coding agents implement fixes from self-review issues.
4. **Phase D** — Remote repos: Extend to review/fix arbitrary user repositories.

### 1.2. GitHub as the Coordination Layer

All development work — both autonomous and human — is tracked via **GitHub
Issues and Pull Requests** on the ReviewCat repository itself:

- **Issues** = work items (review findings, feature requests, bugs).
- **PRs** = implementations (code changes with linked issues).
- **PR/Issue comments** = durable discussion/traceability channel.
- **Labels** = agent ownership, priority, status, category.
- **Containers + branches/worktrees** = isolated parallel work environments.

Real-time worker telemetry (heartbeats, structured error reports, memory sync)
is handled by an **agent bus** (see §5.5), while GitHub remains the durable
coordination layer.

This is distinct from the **app's runtime behavior** which creates issues/PRs
on the end user's target repository.

## 2. Two-Part Architecture

The repository is split into two top-level sections:

```
Review-Cat/
├── app/                    # The product — what end users run
│   ├── src/                # C++ application source code
│   │   ├── core/           # Core library (components, entities, systems)
│   │   ├── cli/            # CLI frontend (main, arg parsing)
│   │   ├── daemon/         # Daemon / watch mode
│   │   ├── ui/             # UI surfaces (runtime UI and optional dev harness visualizer)
│   │   └── copilot/        # Copilot CLI subprocess bridge
│   ├── include/            # Public C++ headers
│   ├── tests/              # C++ test suite (Catch2)
│   ├── config/             # Default configs, persona templates
│   └── CMakeLists.txt      # App build system
│
├── dev/                    # The meta-tooling — what builds the product
│   ├── agents/             # Development role agent prompt files
│   ├── harness/            # Director heartbeat + helper scripts
│   │   ├── director.sh     # Director heartbeat loop (Agent 0)
│   │   ├── run-cycle.sh    # Single task cycle orchestration
│   │   ├── worktree.sh     # Worktree create/teardown helpers
│   │   ├── review-self.sh  # Self-review bootstrap script
│   │   └── record-audit.sh # Audit recording helper
│   ├── plans/              # PRD items, task graphs, progress tracking
│   ├── prompts/            # Prompt templates for dev agents
│   ├── mcp/               # MCP config files (github-mcp.json)
│   ├── scripts/            # Dev harness setup and utility scripts
│   │   ├── daemon.sh       # Keep-alive supervisor (recommended entrypoint)
│   │   ├── setup.sh        # Install system prerequisites
│   │   └── bootstrap.sh    # One-shot project initialization
│   └── audits/             # Development audit bundles
│
├── .github/
│   └── agents/             # Copilot CLI custom agent definitions
│
├── docs/                   # Design docs and specifications (golden source)
│   ├── specs/              # Component, entity, system, agent specs
│   ├── COPILOT_CLI_CHALLENGE_DESIGN.md
│   ├── DIRECTOR_DEV_WORKFLOW.md
│   ├── IMPLEMENTATION_CHECKLIST.md
│   └── PROMPT_COOKBOOK.md
│
├── scripts/                # Top-level convenience scripts
│   ├── build.sh            # Build the app
│   ├── test.sh             # Run all tests
│   └── clean.sh            # Clean build artifacts
│
├── CMakeLists.txt          # Root CMake (delegates to app/)
├── PLAN.md                 # This file
├── TODO.md                 # Actionable task list
└── README.md               # Project overview and quick start
```

### 2.1. `app/` — The Product

This is the ReviewCat application that end users build and run. It contains:

- **CLI frontend** (`reviewcat` binary: `demo`, `review`, `pr`, `fix`, `watch`)
- **Daemon mode** — persistent background process that monitors a repo
- **UI window** — lightweight native window for settings, stats, and control
- **Runtime agents** — persona review agents + coding agents via Copilot CLI
- **Audit system** — structured artifact output for every review run
- **GitHub integration** — create PRs, issues, comments via GitHub MCP Server

### 2.2. `dev/` — The Development Harness

This is the meta-tooling that builds ReviewCat autonomously. It is composed
entirely of **shell scripts** and **Copilot CLI agent profiles**:

- **Agent 0 (Director)** — the always-running orchestrator daemon (shell script)
- **Role agents** — Architect, Implementer, QA, Docs, Security, Code-Review
- **Coding agents** — implement fixes for issues generated by review agents
- **Heartbeat system** — persistent bash loop that keeps the Director alive
- **GitHub MCP integration** — agents use GitHub MCP Server to create/read
  issues, PRs, comments as their primary communication channel
- **Worker parallelism** — multiple agents work simultaneously in isolated
  Docker containers, each bind-mounted to an isolated git worktree in the same
  parent directory
- **Progress tracking** — GitHub Issues + PRD backlog + audit bundles

## 3. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Language | **C++17/20** | Fast native binary; no runtime deps; cross-platform |
| Build system | **CMake** | Industry standard for C++ projects |
| Shell scripts | **Bash** | Dev harness, agent orchestration, build/test/CI |
| UI toolkit | **SDL3** (window/input) + **custom ToolUI** (primitives + bitmap glyph font + hex colors) + optional 3D visualizer | Avoid external UI frameworks; keep UI stack minimal and fully controlled; supports runtime UI + operator-focused swarm visualizer |
| Copilot integration | **`copilot -p`** subprocess calls | Invoke Copilot CLI from C++ via `popen`/`fork+exec` or from shell scripts |
| GitHub integration | **GitHub MCP Server** | Agents use MCP tools for issues, PRs, comments, branches |
| Git ops | **libgit2** + **`git worktree`** | Programmatic git from C++; parallel worktrees for agents |
| JSON | **nlohmann/json** | De facto C++ JSON library |
| Config | **TOML** (`reviewcat.toml`) | Human-readable; parsed with `toml++` |
| Testing | **Catch2** | Mature, header-only C++ test framework. See `docs/dev/TESTING_STRATEGY.md`. |
| Logging | **spdlog** | Fast, header-only C++ logging; structured dev harness logs |
| Package manager | **vcpkg** or git submodules | Dependency management for C++ libs |
| Distribution | Single static binary + shell | No runtime dependencies for end users |
| Dev harness | **Bash shell scripts** + **Docker** | Docker-first Director/worker orchestration; one shared image tag, many containers; structured logs; easy automation and audit trail |

### 3.1. GitHub MCP Server

The **GitHub MCP Server** (`github/github-mcp-server`) provides MCP tools that
Copilot CLI agents can use to interact with GitHub.

- **Toolsets used:** `issues`, `pull_requests`, `repos`, `git`
- **Key capabilities:**
  - Create, read, update, and comment on issues
  - Create, read, review, and merge pull requests
  - Create branches, read file contents, search code
  - List and manage labels
- **Integration:** Copilot CLI agents access GitHub MCP tools natively when
  the MCP server is configured. Agents can `create_issue`, `create_pull_request`,
  `add_issue_comment`, etc. directly in their prompt context.

**Deployment options (MVP-friendly; can run on host or in containers):**

| Option | Method | Best for |
|--------|--------|----------|
| **Remote server** (preferred) | `https://api.githubcopilot.com/mcp/` | Zero install; HTTP-based; always up-to-date |
| **Pre-built binary** | Download from [GitHub Releases](https://github.com/github/github-mcp-server/releases) | Offline use; single Go binary; no build step |
| **Build from source** | `go build ./cmd/github-mcp-server` | Custom patches; development |

The dev harness may run inside Docker (WSL2-friendly). In that model, you can:
- use the **remote** MCP server (simplest), or
- bundle/mount the native `github-mcp-server` binary into the container image.

This repo keeps example MCP config files under:

- `dev/mcp/github-mcp.json` — **remote HTTP MCP** (preferred MVP)
- `dev/mcp/github-mcp-stdio.json` — **local stdio** `github-mcp-server` binary (fallback/offline)

**MCP config for native binary (`dev/mcp/github-mcp-stdio.json`):**

```json
{
  "mcpServers": {
    "github": {
      "command": "/usr/local/bin/github-mcp-server",
      "args": ["stdio"]
    }
  }
}
```

**MCP config for remote server (`dev/mcp/github-mcp.json`):**

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

The native binary is launched with `github-mcp-server stdio` and communicates
over stdin/stdout. Toolsets can be filtered via `--toolsets issues,pull_requests,repos,git`
or the `GITHUB_TOOLSETS` environment variable.

This replaces the previous `gh` CLI approach for development agents, though
`gh` CLI remains available as a fallback and for the C++ runtime app.

### 3.2. Logging Strategy

Logging is split into two layers:

1. **Dev harness logging** (bash) — Structured log output to
   `dev/audits/director.log` and per-cycle logs. Uses `tee` and timestamped
   `printf` statements. Log levels: `INFO`, `WARN`, `ERROR`.
2. **Runtime app logging** (C++) — `spdlog` with console + rotating file sinks.
   Log file at `~/.reviewcat/reviewcat.log`. Configurable log level via
   `reviewcat.toml` or `--log-level` CLI flag.

All logs follow the format: `[YYYY-MM-DD HH:MM:SS] [LEVEL] [component] message`.
Tokens and secrets are never logged.

## 4. Agent Architecture

### 4.1. Development Agents (in `dev/`)

These are **shell scripts + Copilot CLI agent profiles** that run during
development to build ReviewCat itself. They do NOT require the C++ app to be
built — they operate purely via Copilot CLI, bash, and GitHub MCP:

| Agent | Role | How It Runs |
|-------|------|-------------|
| **Director (Agent 0)** | Orchestrator | `dev/scripts/daemon.sh` (recommended) → starts `dev/harness/director.sh` heartbeat loop |
| **Architect** | Design reviewer | `copilot -p @dev/agents/architect.md "..."` |
| **Implementer** | Code writer | `copilot -p @dev/agents/implementer.md "..."` in worktree |
| **Coder** | Fix implementer | `copilot -p @dev/agents/coder.md "..."` in worktree |
| **QA** | Test engineer | `copilot -p @dev/agents/qa.md "..."` |
| **Docs** | Documentation | `copilot -p @dev/agents/docs.md "..."` |
| **Security** | Security auditor | `copilot -p @dev/agents/security.md "..."` |
| **Code-Review** | Reviewer | `copilot -p @dev/agents/code-review.md "..."` |
| **Merge Agent** | Release finalization | `copilot -p @dev/agents/merge-expert.md "..."` (invoked by Director when release is ready) |

**Key addition: Coder agent** — Takes a GitHub Issue (typically generated by
the Code-Review or Security agent), reads the issue description and linked
code, implements a fix in an isolated worktree, and creates a PR linking back
to the issue. This closes the review→fix loop.

### 4.2. Runtime Agents (in `app/`)

These run when end users use ReviewCat to review code on their target repos:

| Agent | Persona | Focus Area |
|-------|---------|-----------|
| **Security** | Vulnerability hunter | Exploitability, unsafe defaults, data leaks |
| **Performance** | Optimizer | Algorithmic complexity, hot paths, allocations |
| **Architecture** | Structural analyst | Module boundaries, naming, dependency direction |
| **Testing** | Test coverage analyst | Missing tests, concrete test case proposals |
| **Docs** | Documentation reviewer | Missing usage docs, confusing behaviors |
| **Coder** | Fix implementer | Implements fixes for review findings |

### 4.3. Copilot CLI + MCP Integration

Copilot CLI is invoked in two contexts with MCP support:

1. **Development harness** (shell scripts) — `copilot -p "prompt"` called
   directly from bash with GitHub MCP Server configured. Agents can create
   issues, PRs, and comments on the ReviewCat repo as they work.
2. **Runtime app** (C++ binary) — The compiled `reviewcat` binary invokes
   `copilot -p "prompt"` as a subprocess with MCP configured for the user's
   target repo.

In both cases, the integration is via the **Copilot CLI subprocess** with
**GitHub MCP Server** providing GitHub API capabilities.

## 5. Agent 0: The Director Daemon

### 5.1. Heartbeat System

Agent 0 is a **bash script** (`dev/harness/director.sh`) that runs in a loop:

In the MVP dev harness, the Director operates in **release cycles**:

- maintain an active release branch: `feature/release-<release_id>`
- maintain a single release PR: `feature/release-<release_id>` → `main`
- worker PRs target the release branch; the release PR closes issues on merge

Local cached process state (first-run vs resume, current release context) is
stored in a gitignored root file `STATE.json` created lazily if missing.

```
┌──────────────────────────────────────────────────────────┐
│              Director Daemon (bash)                      │
│                                                          │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Heartbeat│─▶│  Check   │─▶│ Dispatch │─▶│ Monitor  │ │
│  │  Timer  │  │ Backlog  │  │ Workers  │  │ Worktrees│ │
│  └────┬────┘  └──────────┘  └──────────┘  └────┬─────┘ │
│       │                                         │       │
│       │         ┌────────────┐                  │       │
│       └────────▶│   Sleep    │◀─────────────────┘       │
│                 │  Interval  │                          │
│                 └────────────┘                          │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │     Parallel Worker Containers (shared image)     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │   │
│  │  │Worker 1 │  │Worker 2 │  │Worker 3 │  ...      │   │
│  │  │(worktree)│ │(worktree)│ │(worktree)│         │   │
│  │  └─────────┘  └─────────┘  └─────────┘          │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Heartbeat loop (pseudocode):**

```bash
#!/usr/bin/env bash
# dev/harness/director.sh

INTERVAL=${DIRECTOR_INTERVAL:-60}
MAX_WORKERS=${DIRECTOR_MAX_WORKERS:-3}
MAX_RETRIES=3
REPO="p3nGu1nZz/Review-Cat"

while true; do
  # 0. Ensure release context exists (branch + release PR)
  # RELEASE_BRANCH="feature/release-$(date -u +%Y%m%d-%H%M%SZ)"  # example id
  # (Store release context in STATE.json)

    # 1. Check GitHub Issues for open work items
    ISSUES=$(gh issue list --repo "$REPO" --label "agent-task" \
        --state open --json number,title,labels --jq '.[].number')

    # 2. Check PRD backlog for spec-driven work
    BACKLOG=$(jq -r '.[] | select(.passes == false) | .id' \
        dev/plans/prd.json 2>/dev/null | head -5)

    # 3. Check active workers (containers) / worktrees
    # (Implementation choice: count containers, or derive from worktree list)
    ACTIVE=$(git worktree list --porcelain | grep -c "worktree")

    # 4. For each available worker slot, dispatch work
    for TASK in $ISSUES $BACKLOG; do
        if [ "$ACTIVE" -ge "$MAX_WORKERS" ]; then
            break
        fi

        # Create worktree and dispatch agent
        BRANCH="agent/${TASK}-$(date +%s)"
        ./dev/harness/worktree.sh create "$BRANCH"
        ./dev/harness/run-cycle.sh "$TASK" "$BRANCH" "$RELEASE_BRANCH" &
        ACTIVE=$((ACTIVE + 1))
    done

    # 5. Monitor completed worktrees → merge worker PRs into release branch
    ./dev/harness/monitor-workers.sh

    # 6. Self-review cycle (if no other work)
    if [ -z "$ISSUES" ] && [ -z "$BACKLOG" ]; then
        ./dev/harness/review-self.sh
    fi
            Authentication for the stdio server is provided via environment variables (never commit secrets).
            See `.env.example` and `docs/dev/ENVIRONMENT.md`.

    # 7. Sleep
    sleep "$INTERVAL"
done
```

### 5.2. Container + Worktree-Based Parallel Execution

Each coding agent runs as a **worker container** (all workers share the same
Docker image tag), and each worker container is bind-mounted to an isolated
**git worktree** in the same parent directory as the main repo.

```
~/source/repos/
├── Review-Cat/                    # Main worktree (Director runs here)
├── Review-Cat-agent-42-1707600000/ # Worker 1 (implementing issue #42)
├── Review-Cat-agent-57-1707600060/ # Worker 2 (implementing issue #57)
└── Review-Cat-agent-63-1707600120/ # Worker 3 (implementing issue #63)
```

**Worktree lifecycle:**

```bash
# dev/harness/worktree.sh

create() {
    BRANCH=$1
    WORKTREE_DIR="../Review-Cat-${BRANCH//\//-}"
    git worktree add "$WORKTREE_DIR" -b "$BRANCH"
    echo "$WORKTREE_DIR"
}

teardown() {
    WORKTREE_DIR=$1
    git worktree remove "$WORKTREE_DIR" --force
}
```

**Worker container model (dev harness):**

- **One shared image tag** for all workers (e.g., `reviewcat-dev:main`)
- **One container per task** (worker) with a deterministic name (e.g., `reviewcat-worker-<issue>-<ts>`)
- Bind-mount the worktree into the container (e.g., host `../Review-Cat-agent-...` → container `/workspace`)
- Pass task context via env vars (`ISSUE_NUMBER`, `BRANCH_NAME`, `REPO`) and/or CLI args
- Prefer **scale-to-zero**: stop/remove worker containers when idle or finished

This enables **true parallel execution**: multiple agents can compile, test,
and commit independently without conflicts. The Director manages the lifecycle
and ensures PRs are created when work is complete.

### 5.3. Agent Cycle Script

`dev/harness/run-cycle.sh` orchestrates role agents for a single task in a
worktree (typically executed *inside* the worker container with the worktree
mounted as the container workspace):

```bash
#!/usr/bin/env bash
# dev/harness/run-cycle.sh <task_id> <branch> <base_branch>

TASK=$1
BRANCH=$2
BASE_BRANCH=${3:-main}
WORKTREE_DIR="../Review-Cat-${BRANCH//\//-}"
AUDIT_DIR="dev/audits/$(date +%Y%m%d-%H%M%S)-${TASK}"

cd "$WORKTREE_DIR" || exit 1
mkdir -p "$AUDIT_DIR/ledger"

# 1. Implementer writes code (with GitHub MCP for issue context)
copilot -p @dev/agents/implementer.md \
    "Implement issue #${TASK}. Read the issue description via GitHub MCP, \
     understand the requirements, and write the code." \
    --allow-tools write \
    2>&1 | tee "$AUDIT_DIR/ledger/implementer.txt"

# 2. QA writes tests
copilot -p @dev/agents/qa.md \
    "Write tests for the changes in this branch related to issue #${TASK}." \
    --allow-tools write \
    2>&1 | tee "$AUDIT_DIR/ledger/qa.txt"

# 3. Validate (build + test)
./scripts/build.sh 2>&1 | tee "$AUDIT_DIR/build.log"
./scripts/test.sh 2>&1 | tee "$AUDIT_DIR/test.log"

if [ $? -eq 0 ]; then
    # 4. Commit and create PR via GitHub MCP
    git add -A
    git commit -m "feat(#${TASK}): implement changes"

    copilot -p "Create a pull request for branch '${BRANCH}' into '${BASE_BRANCH}' that \
      references issue #${TASK}. Use 'Refs #${TASK}' (issues close when the release PR merges to main). \
      Include a summary of changes and link the issue. Use GitHub MCP tools." \
        2>&1 | tee "$AUDIT_DIR/ledger/pr-create.txt"

    # 5. Add review comment on the PR
    copilot -p @dev/agents/code-review.md \
        "Review the diff on this branch and add a review comment \
         on the PR via GitHub MCP." \
        2>&1 | tee "$AUDIT_DIR/ledger/code-review.txt"
fi

# 6. Record audit
./dev/harness/record-audit.sh "$TASK" "$AUDIT_DIR"
```

### 5.4. Self-Review Loop (Circular Self-Improvement)

The core innovation: ReviewCat reviews itself, generates issues, fixes them,
and repeats indefinitely:

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

**`dev/harness/review-self.sh`:**

```bash
#!/usr/bin/env bash
# Review the ReviewCat codebase itself and create issues for findings

REPO="p3nGu1nZz/Review-Cat"

# 1. Generate diff of recent changes
DIFF=$(git diff HEAD~5..HEAD 2>/dev/null || git diff HEAD)

# 2. Run persona review agents on the diff
for PERSONA in security performance architecture testing docs; do
    copilot -p @dev/agents/${PERSONA}-review.md \
        "Review the following code changes for ${PERSONA} concerns. \
         Output findings as JSON array with fields: title, description, \
         severity, file, line_start, suggested_fix." \
        --stdin <<< "$DIFF" \
        > "/tmp/reviewcat-self-${PERSONA}.json" 2>&1
done

# 3. For each critical/high finding, create a GitHub Issue
for PERSONA in security performance architecture testing docs; do
    jq -c '.[] | select(.severity == "critical" or .severity == "high")' \
        "/tmp/reviewcat-self-${PERSONA}.json" 2>/dev/null | \
    while read -r FINDING; do
        TITLE=$(echo "$FINDING" | jq -r '.title')
        BODY=$(echo "$FINDING" | jq -r '.description + "\n\n**Suggested fix:** " + .suggested_fix')

        # Create issue via GitHub MCP (through copilot)
        copilot -p "Create a GitHub Issue on ${REPO} with title: \
            '[$PERSONA] ${TITLE}' and body: '${BODY}'. \
            Add labels: agent-task, ${PERSONA}, auto-review. \
            Use GitHub MCP tools."
    done
done
```

### 5.5. Inter-Agent Communication (GitHub + Agent Bus)

Agents coordinate through **two channels**:

1) **GitHub** — durable coordination (source of truth for work)
2) **Agent bus** — real-time telemetry + structured messaging (worker state, errors, memory sync)

#### GitHub (durable)

Agents communicate through GitHub's native features:

| Mechanism | Purpose | Example |
|-----------|---------|---------|
| **Issues** | Work items and findings | Review agent creates issue for a bug |
| **Issue comments** | Discussion and clarification | Architect comments on complexity concern |
| **PRs** | Code implementations | Coder agent creates PR fixing issue |
| **PR comments** | Review feedback | Code-review agent comments on PR |
| **PR reviews** | Approval/request changes | QA agent approves after tests pass |
| **Labels** | Categorization and routing | `agent-task`, `security`, `in-progress` |
| **Linked issues** | Traceability | Worker PR says "Refs #42"; release PR aggregates "Closes #..." |

#### Agent bus (real-time)

In addition to GitHub, workers maintain a lightweight socket connection to the
Director (hub-and-spoke MVP). This enables:

- **Worker state telemetry** (stage + progress + heartbeat)
- **Structured error reports** for recovery/retry (Docker/MCP/GitHub/bus)
- **ProjectState snapshots** for the future UI (live swarm graph)
- **Memory synchronization** via a bounded in-memory agent-bus event buffer, compacted into engram DTO batches under `/memory/` with an authoritative catalog at `memory/catalog.json` (see Issue #13)

All timings and ports live in `config/dev.toml` (see Issue #5).

Normative message framing and DTO field requirements live in:

- `docs/specs/systems/AgentBusSystem.md`

Retry/recovery policy and the canonical structured error DTO are documented in:

- `docs/dev/ERROR_HANDLING.md`

**Label taxonomy for agent coordination:**

| Label | Meaning |
|-------|---------|
| `agent-task` | Available for agent pickup |
| `agent-claimed` | Currently being worked on |
| `agent-review` | Needs review from another agent |
| `agent-blocked` | Needs human input |
| `auto-review` | Created by self-review process |
| `security` / `performance` / `architecture` / `testing` / `docs` | Persona category |
| `priority-critical` / `priority-high` / `priority-medium` / `priority-low` | Severity |

For the full label taxonomy, lifecycle diagrams, and the issue-claim lock protocol, see `docs/dev/GITHUB_LABELS.md`.

### 5.6. Guardrails

- **Dry-run default** — no `git push` or GitHub mutations without explicit opt-in.
- **Scope lock** — Director refuses to modify files outside the spec's scope.
- **Retry budget** — Max 3 retries per sub-task before marking failed.
- **Dangerous command deny list** — `rm -rf`, `git push --force`, etc.
- **Watchdog** — If a cycle exceeds timeout, Director kills the subprocess.
- **Permission profiles** — Copilot CLI `--allow-tools` / `--deny-tools` flags.
- **Worktree isolation** — Agents cannot modify the main worktree directly.
- **PR-gated merges** — All changes go through PRs, never direct main commits.
- **Label-based claiming** — Agents claim issues before starting work to avoid
  duplicate effort across parallel workers.

### 5.7. Setup & Bootstrap Sequence

The cold-start process is split into two scripts:

1. **`dev/scripts/setup.sh`** — Installs system prerequisites (run once per machine).
2. **`dev/scripts/bootstrap.sh`** — Initializes the project (run once per clone).

```bash
# 1. Clone and enter the repo
cd Review-Cat

# 2. Install system prerequisites
./dev/scripts/setup.sh
# This script:
#   - Installs gh CLI (if missing)
#   - Installs jq (if missing)
#   - Downloads github-mcp-server binary from GitHub Releases (if missing)
#   - Verifies: copilot, gh, jq, cmake, g++, github-mcp-server
#   - Prompts for GITHUB_PERSONAL_ACCESS_TOKEN if unset
#   - Runs gh auth login if not authenticated
#   - All installs are idempotent — safe to re-run

# 3. Bootstrap the project
./dev/scripts/bootstrap.sh
# This script:
#   - Verifies setup.sh was run (checks for all tools)
#   - Creates dev/mcp/github-mcp.json MCP config
#   - Creates dev/plans/prd.json with initial bootstrap tasks
#   - Sets up label taxonomy on the repo (agent-task, etc.)
#   - Creates initial GitHub Issues for Phase 0 tasks
#   - Runs ./scripts/build.sh to verify C++ scaffold compiles
#   - Runs initial self-review to seed first issues
#  - Prints: "Bootstrap complete. Run: ./dev/scripts/daemon.sh"

# 4. Start supervisor + Director daemon
./dev/scripts/daemon.sh
# Director will:
#   - Read open issues labeled 'agent-task'
#   - Create worktrees for parallel work
#   - Dispatch agents to implement, test, review
#   - Create PRs, get reviews, merge
#   - Run self-review to generate more issues
#   - Loop indefinitely
```

## 6. End-User UI

### 6.1. UI Window

A native window built with **SDL3** (window/input) and a **custom ToolUI** (draw primitives + bitmap font) provides:

- **Two status bars**
  - **Top bar:** single-line hotkey helper text (e.g., F1–F12)
  - **Bottom bar:** current focused panel/view state + status info
- **Layered UI**
  - A “world/scene” viewport (e.g., swarm graph visualization)
  - Overlay panels/windows with explicit **z-index** ordering (including modal confirmations)

- **Dashboard** — Active review status, recent findings, daemon health.
- **Agent Status** — Live view of active agents, worktree status, current task,
  progress indicators, heartbeat health, worker slot utilization.
- **Settings** — Copilot credentials, GitHub access token, target repo, persona
  selection, review interval, notification preferences.
- **Stats** — Review counts, finding trends, persona breakdown, agent uptime.
- **Agent Personas** — Enable/disable personas, adjust severity thresholds.
- **Audit Log** — Browse past review runs and their artifacts.
- **Log Viewer** — Live scrolling log output with level filtering (ToolUI text rendering).
- **Controls** — Start/stop daemon, trigger manual review, open audit directory.

Optionally, the UI can expose a **swarm visualizer** mode intended for the
dev harness: it connects to the agent bus (socket pub/sub) and renders a live
3D scene of workers, tasks, and message edges, with panels for start/stop,
pause/unpause, filtering, and inspecting structured status/error payloads.

The UI is compiled into the `reviewcat` binary. Running `reviewcat ui` opens the
window. The daemon can run headless (no UI) via `reviewcat watch`.

### 6.2. Settings Screen

End users configure:

| Setting | Description |
|---------|------------|
| Copilot credentials | Authentication for Copilot CLI |
| GitHub access token | PAT for creating PRs, issues, comments |
| Target repository | `OWNER/REPO` to monitor |
| Base branch | Default branch for diff comparison |
| Review interval | How often the daemon checks for changes (seconds) |
| Personas enabled | Which review personas to run |
| Auto-comment | Whether to post unified review as PR comment |
| Auto-fix | Whether to generate and apply patches via coding agents |
| Redaction rules | Glob patterns for sensitive files to exclude |

Settings are stored in `reviewcat.toml` (user-editable, no secrets in repo).

## 7. Core Review Pipeline (Runtime)

```
 Target Repo
      │
      ▼
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │ RepoDiff │───▶│ Persona  │───▶│Synthesis │───▶│  Audit   │
 │  System  │    │ Review   │    │  System  │    │  Store   │
 └──────────┘    │  System  │    └──────────┘    └──────────┘
                 └──────────┘           │
                      │                 ▼
                      │          ┌──────────┐    ┌──────────┐
                      │          │ GitHub   │───▶│  Coding  │
                      │          │   Ops    │    │  Agent   │
                      │          └──────────┘    └────┬─────┘
                      │                               │
                      │          ┌──────────┐         │
                      └─────────▶│  Patch   │◀────────┘
                                 │  Apply   │
                                 └──────────┘
```

1. **RepoDiffSystem** — Collect diffs via `libgit2` or `git` subprocess.
2. **PersonaReviewSystem** — Run persona agents via `copilot -p` subprocess.
3. **SynthesisSystem** — Deduplicate, prioritize, unify findings (C++).
4. **AuditStoreSystem** — Write artifacts to disk, update index.
5. **GitHubOpsSystem** — Create issues and PRs via GitHub MCP Server.
6. **CodingAgentSystem** — Dispatch coding agents to implement fixes.
7. **PatchApplySystem** — Generate and apply safe patches (opt-in).

All systems are implemented as C++ classes following the ECS-style separation.

**Note:** When ReviewCat reviews itself (self-improvement mode), the "Target
Repo" is `p3nGu1nZz/Review-Cat` and the GitHubOpsSystem creates issues/PRs on
itself. When reviewing a user's repo, it operates on the user's configured
repository.

## 8. Development Workflow

### 8.1. Spec-First Development

All development follows the spec-first pattern:

1. Write or update a spec in `docs/specs/`.
2. Director creates a GitHub Issue linking the spec.
3. Director assigns the issue to a coding agent in a worktree.
4. Agent implements, tests, and documents.
5. Agent creates a PR linking the issue.
6. Review agent reviews the PR.
7. Director validates via `./scripts/build.sh && ./scripts/test.sh`.
8. Director merges the PR and closes the issue.
9. Director records audit.

### 8.2. GitHub-Driven Development Loop

The Director uses GitHub as its coordination layer:

1. **Wake** — Heartbeat timer fires.
2. **Scan** — List open issues labeled `agent-task` via GitHub MCP.
3. **Claim** — For each unclaimed issue, add `agent-claimed` label.
4. **Dispatch** — Create worktree, spawn agent to work on the issue.
5. **Monitor** — Check worktree workers for completion.
6. **PR** — Agent creates PR via GitHub MCP, linking the issue.
7. **Review** — Code-review agent reviews the PR via GitHub MCP.
8. **Validate** — Run `./scripts/build.sh && ./scripts/test.sh` in worktree.
9. **Merge** — If validation passes, merge the worker PR into the active release branch.
10. **Teardown** — Remove worktree.
11. **Self-review** — When idle, review own code and create new issues.
12. **Sleep** — Wait for next heartbeat interval.

### 8.3. Parallel Execution via Worktrees

- Independent tasks run in parallel in isolated git worktrees.
- Each worktree has its own branch, build directory, and test artifacts.
- Worktrees are created in the parent directory of the main repo.
- The Director manages up to `MAX_WORKERS` concurrent worktrees.
- Each agent's work is verified by build/test results before PR creation.
- Verified worker PRs are merged to the active release branch; a release PR merges to `main`.
- Durable coordination is via GitHub Issues/PRs/comments/labels.
- Real-time telemetry and memory sync use the agent bus (see §5.5).
- Avoid ad-hoc file-based IPC between workers.

### 8.4. Coding Agent Workflow

When a review finding is promoted to a GitHub Issue, the coding agent:

1. **Reads the issue** via GitHub MCP — understands the problem and requirements.
2. **Creates a branch** — `fix/<issue-number>-<short-description>`.
3. **Works in a worktree** — isolated environment for compilation and testing.
4. **Implements the fix** — uses Copilot CLI with the codebase context.
5. **Writes tests** — ensures the fix is validated.
6. **Runs build + test** — validates everything compiles and passes.
7. **Creates a PR** — via GitHub MCP, linking `Refs #<issue-number>` (the release PR closes issues).
8. **Requests review** — adds `agent-review` label for code-review agent.

## 9. Phased Implementation Plan

### Phase 0: Bootstrap & Dev Harness (PRIORITY — Get Director Running)
- **Bootstrap scaffold milestone:** see Issue #16.
- Create `app/` and `dev/` directory structure.
- Create `config/dev.toml` (dev harness timings, ports, worker limits; no secrets).
- Write `dev/scripts/setup.sh` (install system prereqs: gh, jq,
  github-mcp-server binary, verify toolchain).
- Write `dev/scripts/bootstrap.sh` (configure MCP, create initial issues,
  set up labels, verify build).
- Write `dev/scripts/daemon.sh` (keep-alive supervisor for Agent 0).
- Write `dev/harness/director.sh` (heartbeat daemon with worktree management).
- Write `dev/harness/run-cycle.sh` (agent cycle in worktree).
- Write `dev/harness/worktree.sh` (create/teardown helpers).
- Write `dev/harness/review-self.sh` (self-review bootstrap).
- Write `dev/harness/record-audit.sh` (audit recording).
- Define agent profiles in `.github/agents/` and `dev/agents/`.
- Set up CMake build system for C++ project.
- Create `scripts/build.sh`, `scripts/test.sh`, `scripts/clean.sh`.
- Create `dev/plans/prd.json` initial backlog.
- Verify Director can: read issues, create worktrees, dispatch agents,
  create PRs, and merge — end to end.

### Phase 1: Self-Review Loop (Self-Improvement Begins)
- Implement `dev/harness/review-self.sh` that runs persona reviews on own code.
- Implement issue creation from review findings.
- Implement coding agent that reads issues and creates PRs.
- Implement Director merge logic (validate → merge → teardown).
- Verify circular loop: review → issue → fix → PR → merge → review.
- The Director should be able to run indefinitely at this point.

### Phase 2: Core App Skeleton (C++)
- Implement CLI frontend (`main.cpp`, arg parsing).
- Implement `RunConfig` component (TOML parsing via `toml++`).
- Implement `AuditIdFactory` and `AuditStoreSystem`.
- Implement prompt ledger (`PromptRecord` as JSON via `nlohmann/json`).
- Implement `ReviewFinding` and `ReviewInput` components.
- Implement `reviewcat demo` with bundled sample diff.
- Write unit tests with Catch2.

### Phase 3: Review Pipeline (C++)
- Implement `RepoDiffSystem` (git diff via `libgit2` or subprocess).
- Implement diff chunking for prompt budget management.
- Implement `CopilotRunnerSystem` (`copilot -p` subprocess wrapper).
- Implement record/replay mode for deterministic testing.
- Implement `PersonaReviewSystem` (persona loop, JSON validation, repair).
- Implement `SynthesisSystem` (dedupe, prioritize, unified markdown).
- Implement `CodingAgentSystem` (dispatch coding agents for fixes).
- Implement `reviewcat review` end-to-end.

### Phase 4: GitHub Integration (Runtime)
- Implement `GitHubOpsSystem` using GitHub MCP Server for runtime app.
- Implement `reviewcat pr` command.
- Implement watch mode daemon for continuous monitoring.
- Implement coding agent dispatch for user's target repo.

### Phase 5: Patch Automation
- Implement `PatchApplySystem` (generate patches, apply with safety checks).
- Implement `reviewcat fix` command.

### Phase 6: End-User UI (C++)
- Integrate SDL3 window/input with custom ToolUI (primitives + bitmap font + hex colors).
- Implement status bars (top hotkey helper, bottom focus/status).
- Implement dashboard, settings, stats, audit log, daemon controls.

### Phase 7: Polish & Distribution
- Produce single static binary (`reviewcat`).
- Add comprehensive error handling (spdlog logging is active from Phase 0).
- Write end-user documentation.
- Set up CI/CD (GitHub Actions with CMake).

## 10. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | **C++17/20** | Fast native binary; no runtime deps; cross-platform |
| Dev harness | **Bash scripts** | Simple, no build step needed, runs anywhere with Copilot CLI |
| Build system | **CMake** | Industry standard for C++ |
| UI framework | **SDL3 + custom ToolUI** | SDL3 handles window/input; ToolUI provides primitives + bitmap font; explicit layering/z-index; optional 3D view for swarm visualization |
| Copilot invocation | **Subprocess** (`copilot -p`) | No SDK dependency; works from both bash and C++ |
| GitHub integration | **GitHub MCP Server** | Native MCP tools for issues, PRs, comments; replaces raw `gh` CLI |
| Agent communication | **GitHub Issues/PRs/Comments** | Transparent, auditable, enables parallel agents |
| Parallel execution | **Git worktrees** | True isolation; shared parent dir; no merge conflicts |
| Daemon model | Bash loop (dev) / C++ daemon (runtime) | Dev harness needs no compilation; runtime is compiled |
| Self-improvement | **Circular review→fix loop** | ReviewCat reviews itself, creates issues, fixes them |
| Config format | **TOML** | Human-readable, C++ parsers available |
| Test framework | **Catch2** | Header-only, widely used, good CMake support |
| JSON library | **nlohmann/json** | De facto standard for C++ JSON |
| Git library | **libgit2** | C library with C++ wrappers; no subprocess needed |

## 11. Inspiration & Prior Art

| Project | What We Borrow |
|---------|---------------|
| [Ralph](https://github.com/soderlind/ralph) | Heartbeat loop, PRD-driven task picking, progress tracking, permission profiles |
| [CEO Orchestration](https://github.com/ivfarias/ceo) | Agent profiles, index-driven discovery, workflow prescription |
| [Copilot Swarm Orchestrator](https://github.com/moonrunnerkc/copilot-swarm-orchestrator) | Parallel wave execution, branch isolation, transcript verification |
| [Kodus AI](https://github.com/kodustech/kodus-ai) | Production code review agent patterns |
| [GitHub MCP Server](https://github.com/github/github-mcp-server) | MCP tools for GitHub API: issues, PRs, branches, code |

## 12. Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Copilot CLI quota limits | Record/replay mode for tests; caching per chunk/persona |
| Agent produces bad code | Build/test validation gate; max retry budget; PR review before merge |
| Daemon crashes | Watchdog with auto-restart in bash; PID file; graceful state persistence |
| Scope creep in autonomous dev | Director enforces spec scope; refuse unscoped changes |
| Credential exposure | Never store tokens in code; redaction rules; `.gitignore` for config |
| C++ compile times | Precompiled headers; modular CMake targets; incremental builds |
| WSL-specific issues | Test on WSL Ubuntu; document WSL prerequisites |
| Worktree conflicts | Branch naming conventions; label-based claiming prevents duplicates |
| Infinite loop of trivial issues | Severity threshold for auto-issue creation; dedup against existing open issues |
| MCP Server availability | Three deployment options (remote, binary, source); fallback to `gh` CLI |
| Parallel agent race conditions | Issue claiming via labels; branch naming prevents collisions |
