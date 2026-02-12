# ReviewCat — TODO

> Actionable task list mapped to **PLAN.md** phases.
> Each item maps to a spec under `docs/specs/` where applicable.
> Status: `[ ]` = not started, `[-]` = in progress, `[x]` = done
>
> **GitHub Issues** are the primary tracking mechanism for autonomous agents.
> This file provides the human-readable overview; `dev/plans/prd.json` and
> GitHub Issues labeled `agent-task` are the machine-readable sources.

**TODO.md lifecycle (governance):** At the start of each release cycle, archive
the previous TODO into `/archive/` (e.g., `archive/TODO-2026-02-11.md`) and keep
this file focused on the current release.

---

## Phase 0: Bootstrap & Dev Harness (PRIORITY — Get Director Running)

> **Goal:** Go from docs-only repo to a self-coding Director daemon — like
> Claude Code but autonomous. Every subsection below is ordered by dependency.
> Complete them top-to-bottom.

### 0.1 — Environment & Toolchain (prereqs before anything else)

Run `dev/scripts/setup.sh` to automate these, or do them manually:

- [ ] Install/verify **Copilot CLI**: `copilot --version` (requires GitHub Copilot subscription)
- [ ] Install **gh CLI**: `sudo apt install gh` → `gh --version`
- [ ] Authenticate gh CLI: `gh auth login` (select HTTPS, `repo` scope, confirm with browser)
- [ ] Install **jq**: `sudo apt install jq` → `jq --version`
- [ ] Install/verify **Docker** (dev harness runtime; WSL2-friendly):
  - `docker --version`
  - `docker compose version`
  - Verify non-root usage (or document `sudo docker ...` if required)
- [ ] Verify **cmake**: `cmake --version` (already installed)
- [ ] Verify **g++**: `g++ --version` (already installed)
- [ ] Install **github-mcp-server** (choose one):
  - **Remote MCP** (preferred for MVP): `https://api.githubcopilot.com/mcp/`
  - **Native binary** on host (fallback/offline)
  - **Bundled in dev harness container image** (optional; avoids host installs)
  ```bash
  # Option A: Download pre-built binary from GitHub Releases
  GH_MCP_VERSION="v0.30.3"  # check for latest
  curl -Lo /usr/local/bin/github-mcp-server \
    "https://github.com/github/github-mcp-server/releases/download/${GH_MCP_VERSION}/github-mcp-server-linux-amd64"
  chmod +x /usr/local/bin/github-mcp-server

  # Option B: Build from source (requires Go 1.21+)
  git clone https://github.com/github/github-mcp-server.git /tmp/github-mcp-server
  cd /tmp/github-mcp-server && go build -o /usr/local/bin/github-mcp-server ./cmd/github-mcp-server
  ```
- [ ] Verify MCP server: `github-mcp-server --help`
- [ ] Create a GitHub **Personal Access Token** (PAT) with `repo` scope
- [ ] Export PAT: `export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_...` (add to `~/.bashrc`)
- [ ] Verify Copilot CLI works: `copilot -p "Say hello and confirm you can see this prompt"`
- [ ] Verify gh works: `gh issue list --repo p3nGu1nZz/Review-Cat`

- [ ] Create `/archive/` directory for TODO history (see TODO.md lifecycle note)

- [ ] Create/maintain canonical dev harness config: `config/dev.toml`
  - All timings (intervals, watchdogs, TTLs, retry/backoff) live here
  - Agent-bus ports/security live here
  - No secrets in TOML (tokens remain env vars)

### 0.2 — GitHub MCP Server Configuration (gives agents GitHub superpowers)

- [ ] Verify `github-mcp-server` is installed (from setup.sh)
- [ ] Create `dev/mcp/github-mcp.json` — MCP config file:
  ```json
  {
    "mcpServers": {
      "github": {
        "command": "github-mcp-server",
        "args": ["stdio", "--toolsets", "issues,pull_requests,repos,git"],
        "env": {
          "GITHUB_PERSONAL_ACCESS_TOKEN": ""
        }
      }
    }
  }
  ```
  > **Note:** For the MVP, prefer the **remote MCP server** (`{"type": "http", "url": "https://api.githubcopilot.com/mcp/"}`)
  > to minimize host/container setup complexity. If you run the native stdio MCP server,
  > it can live on the host OR inside the dev harness container image — either way, agents
  > only need a working MCP config.
- [ ] Smoke-test MCP read: `copilot -p "List open issues on p3nGu1nZz/Review-Cat using GitHub MCP tools" --mcp-config dev/mcp/github-mcp.json`
- [ ] Smoke-test MCP write: `copilot -p "Create a GitHub Issue titled '[test] MCP smoke test' with body 'Delete me' on p3nGu1nZz/Review-Cat" --mcp-config dev/mcp/github-mcp.json`
- [ ] Close/delete the smoke-test issue
- [ ] Smoke-test agent + MCP + file write:
  ```bash
  copilot -p "Create a file called /tmp/mcp-test.txt containing 'hello from copilot'. \
    Then list open issues on p3nGu1nZz/Review-Cat via GitHub MCP." \
    --mcp-config dev/mcp/github-mcp.json --allow-tools write
  ```
  This proves the agent can **read/write files AND talk to GitHub** — the two core capabilities needed for self-coding.
- [ ] Set up label taxonomy on the repo (via `gh` or MCP):
  - `agent-task`, `agent-claimed`, `agent-review`, `agent-blocked`, `auto-review`
  - `security`, `performance`, `architecture`, `testing`, `docs`
  - `priority-critical`, `priority-high`, `priority-medium`, `priority-low`

  Reference: `docs/dev/GITHUB_LABELS.md` (taxonomy + lifecycle + claim-lock protocol).

### 0.3 — Directory Structure & Build Scaffold

- [ ] Create `app/` hierarchy: `src/core/`, `src/cli/`, `src/daemon/`, `src/ui/`, `src/copilot/`, `include/`, `tests/`, `config/`, `scripts/`
- [ ] Create `dev/` hierarchy: `agents/`, `harness/`, `plans/`, `prompts/`, `scripts/`, `audits/`, `mcp/`
- [ ] Create `.github/agents/` for Copilot CLI repo-level agent profiles
- [ ] Create `dev/harness/log.sh` — shared logging functions for all bash scripts
- [ ] Create `.gitignore`:
  ```
  build/
  *.o
  *.a
  *.so
  Review-Cat-agent-*/
  dev/audits/*/
  .env
  STATE.json
  *.log
  ```
- [ ] Write root `CMakeLists.txt` (delegates to `app/CMakeLists.txt`)
- [ ] Write `app/CMakeLists.txt` with targets: `reviewcat` binary, `reviewcat_core` lib, `reviewcat_tests`

### 0.4 — Minimal C++ Scaffold (build gate must pass from day 1)

The Director's build/test validation gate runs `scripts/build.sh && scripts/test.sh`
on every agent cycle. We need a minimal compilable project so this gate passes
immediately — even before any real app code exists.

- [ ] Write `app/src/cli/main.cpp` — Minimal main:
  ```cpp
  #include <iostream>
  int main(int argc, char* argv[]) {
      if (argc > 1 && std::string(argv[1]) == "--version") {
          std::cout << "reviewcat 0.0.1-bootstrap" << std::endl;
          return 0;
      }
      std::cout << "ReviewCat — autonomous code review daemon" << std::endl;
      std::cout << "Usage: reviewcat [demo|review|pr|fix|watch|ui]" << std::endl;
      return 0;
  }
  ```
- [ ] Write `scripts/build.sh`:
  ```bash
  #!/usr/bin/env bash
  set -e
  mkdir -p build && cd build
  cmake .. -DCMAKE_BUILD_TYPE=Debug
  cmake --build . --parallel "$(nproc)"
  ```
- [ ] Write `scripts/test.sh`:
  ```bash
  #!/usr/bin/env bash
  set -e
  if [ -f build/reviewcat_tests ]; then
      ./build/reviewcat_tests
  else
      echo "No test binary yet — scaffold pass"
      exit 0
  fi
  ```
- [ ] Write `scripts/clean.sh` — `rm -rf build/`
- [ ] Verify: `./scripts/build.sh && ./scripts/test.sh` passes (green gate)
- [ ] Verify: `./build/reviewcat --version` prints `0.0.1-bootstrap`

### 0.5 — Agent Profiles (the "system prompts" that make agents code like Claude Code)

These are the prompt files that define each agent's identity, capabilities,
rules, and output format. They are the equivalent of Claude Code's system
prompt — they tell the LLM what it is and how to behave.

- [ ] Create `dev/agents/coder.md` — **CRITICAL, this is the core coding agent**:
  - Identity: "You are a coding agent for the ReviewCat project"
  - Capabilities: read/write files, run shell commands, use GitHub MCP tools
  - Context: C++17/20, CMake, nlohmann/json, toml++, Catch2, project structure
  - Rules: only modify files in your worktree, follow specs, write tests
  - Output: working code + tests that pass `scripts/build.sh && scripts/test.sh`
  - MCP tools: `get_issue`, `create_pull_request`, `add_issue_comment`
- [ ] Create `dev/agents/implementer.md` — Writes new features from specs
- [ ] Create `dev/agents/code-review.md` — Reviews PRs, posts review comments
- [ ] Create `dev/agents/qa.md` — Writes Catch2 tests, creates fixtures
- [ ] Create `dev/agents/architect.md` — Reviews architecture, complexity
- [ ] Create `dev/agents/security.md` — Finds vulnerabilities, unsafe patterns
- [ ] Create `dev/agents/docs.md` — Maintains documentation
- [ ] Create `.github/agents/reviewcat.md` — Repo-level Copilot agent definition
- [ ] Verify agent invocation works:
  ```bash
  copilot -p @dev/agents/coder.md \
    "Read the file PLAN.md and summarize the Phase 0 tasks" \
    --mcp-config dev/mcp/github-mcp.json
  ```

### 0.6 — Core Harness Scripts (the orchestration that makes it autonomous)

These scripts are what transform Copilot CLI from a manual tool into an
autonomous coding agent. Without them, you have Claude Code with no loop.
With them, you have a self-driving development daemon.

- [ ] Write `dev/harness/worktree.sh` — Worktree lifecycle helpers:
  - [ ] `create <branch>` — `git worktree add ../Review-Cat-${branch} -b ${branch}`
  - [ ] `teardown <worktree_dir>` — `git worktree remove --force`
  - [ ] `list` — `git worktree list --porcelain`
  - [ ] `count` — count active worktrees
  - [ ] Validate MAX_WORKERS limit before creating
  - [ ] Document how worktrees map to worker containers:
    - One **shared** dev harness Docker image tag
    - One **container per worker**
    - One **worktree per container**, bind-mounted as the container workspace
- [ ] Write `dev/harness/run-cycle.sh` — Single task cycle (the "Claude Code session"):
  - [ ] Accept args: `<issue_number> <branch_name> [base_branch]`
  - [ ] `cd` into the worktree for this branch
  - [ ] Docker-first execution (recommended): run `run-cycle.sh` inside a worker container with the worktree mounted at a fixed path (e.g., `/workspace`)
  - [ ] Create audit dir: `dev/audits/$(date +%Y%m%d-%H%M%S)-${issue}/`
  - [ ] Invoke coder agent with issue context + MCP + file write:
    ```bash
    copilot -p @dev/agents/coder.md \
      "Fix issue #${ISSUE} on p3nGu1nZz/Review-Cat. \
       Read the issue via GitHub MCP. Implement the fix. Write tests. \
       Ensure scripts/build.sh && scripts/test.sh pass." \
      --mcp-config dev/mcp/github-mcp.json \
      --allow-tools write \
      2>&1 | tee "$AUDIT_DIR/ledger/coder.txt"
    ```
  - [ ] Run `./scripts/build.sh && ./scripts/test.sh` (validation gate)
  - [ ] On success: `git add -A && git commit -m "fix(#${ISSUE}): ..."`
  - [ ] On success: invoke PR creation via MCP:
    ```bash
    # NOTE: PRs should normally target the active release branch (feature/*),
    # not main. The release PR (feature/* -> main) is what closes issues.
    copilot -p "Create a PR from branch '${BRANCH}' to '${BASE_BRANCH}' on \
      p3nGu1nZz/Review-Cat. Title: 'fix(#${ISSUE}): ...' \
      Body: 'Refs #${ISSUE} (batched into current release)'. Use GitHub MCP tools." \
      --mcp-config dev/mcp/github-mcp.json \
      2>&1 | tee "$AUDIT_DIR/ledger/pr-create.txt"
    ```
  - [ ] Invoke code-review agent on the PR
  - [ ] Record audit bundle
- [ ] Write `dev/harness/review-self.sh` — Self-review (creates work for the loop):
  - [ ] Generate diff: `git diff HEAD~5..HEAD` (or full tree on first run)
  - [ ] For each persona (security, performance, architecture, testing, docs):
    ```bash
    copilot -p @dev/agents/${PERSONA}-review.md \
      "Review this code for ${PERSONA} concerns. Output JSON array: \
       [{title, description, severity, file, line_start, suggested_fix}]" \
      --stdin <<< "$DIFF" > /tmp/reviewcat-self-${PERSONA}.json
    ```
  - [ ] Filter findings: only `critical` and `high` severity
  - [ ] Deduplicate: compare titles against existing open issues (`gh issue list`)
  - [ ] Create GitHub Issues for new findings via MCP with labels
- [ ] Write `dev/harness/monitor-workers.sh` — Worker completion check:
  - [ ] List active worktrees
  - [ ] For each: check if agent process is still running
  - [ ] Track worker container state + heartbeat TTL (via agent bus + docker state)
  - [ ] Consume WorkerState heartbeats and structured error reports as defined in `docs/specs/systems/AgentBusSystem.md`
  - [ ] Apply retry/recovery/escalation policy from `docs/dev/ERROR_HANDLING.md`
  - [ ] For completed workers: validate build/test passed
  - [ ] For passing workers: merge PR **into active release branch** via MCP, teardown worktree
  - [ ] Close issues only when the **release PR** (feature/* → main) is merged (or close explicitly as part of release finalization)
  - [ ] For failing workers: increment retry counter, re-dispatch or label `agent-blocked`
- [ ] Write `dev/harness/record-audit.sh` — Audit recording:
  - [ ] Collect: ledger files, build.log, test.log, `git diff`
  - [ ] Write audit summary JSON
  - [ ] Update `dev/audits/index.json`

### 0.7 — Daemon Supervisor + Director (keep-alive + Agent 0 orchestration)

The **daemon supervisor** is responsible for keeping Agent 0 alive across
interruptions (container restarts, transient failures) and coordinating
**release-oriented upgrades**.

- [ ] Add `STATE.json` (gitignored) to the repo contract:
  - [ ] Created lazily by the daemon/Director if missing
  - [ ] Stores **local cached state** only (never committed)
  - [ ] Used to detect first-run vs resume after restart

- [ ] Write `dev/scripts/daemon.sh` — Supervisor daemon:
  - [ ] Starts `dev/harness/director.sh` as a child process
  - [ ] Keep-alive: if Director exits unexpectedly, restart with backoff
  - [ ] Writes/updates `STATE.json` with last-seen heartbeat + current release context
  - [ ] Upgrade handling:
    - [ ] When a new release is published, supervisor updates worker image tag
    - [ ] Workers may restart only at **safe points** (between commits; clean worktree)
    - [ ] If a worker is mid-edit and safe to discard, it may `git reset --hard HEAD` and restart
    - [ ] If a worker is mid-commit/rebase (unsafe), it finishes the critical section first

- [ ] Write `dev/harness/director.sh` — The main daemon:
  - [ ] Load config from `config/dev.toml` (intervals, max workers, watchdogs, agent-bus ports)
  - [ ] PID file: write `$$` to `dev/harness/director.pid`, check on startup
  - [ ] Trap `SIGTERM`/`SIGINT` for graceful shutdown (teardown all worktrees)
  - [ ] On startup, read/create `STATE.json`:
    - [ ] Determine if this is first run vs resume
    - [ ] Load current release context (if any)
  - [ ] **Main loop** (`while true`):
    1. Ensure there is an **active release plan** (or create one):
       - Create/refresh a release branch: `feature/release-<id>`
       - Create/refresh a release PR: `feature/release-<id>` → `main`
       - Store the release id/branch/PR in `STATE.json`
    2. Scan open issues labeled `agent-task` via GitHub MCP (gh fallback)
    3. Select a batch of issues for the current release (release plan)
    4. Count active worktrees via `dev/harness/worktree.sh count`
    5. For each available worker slot + unclaimed issue:
       - Claim issue: add `agent-claimed` label, remove `agent-task`
       - Create branch: `agent/${ISSUE}-$(date +%s)`
       - Create worktree: `dev/harness/worktree.sh create $BRANCH`
       - Dispatch: start a worker **container** (shared image tag) that runs:
         - `dev/harness/run-cycle.sh $ISSUE $BRANCH feature/release-<id>`
       - Register worker in swarm state (container id, worktree path, task id)
    6. Monitor completed workers: `dev/harness/monitor-workers.sh`
    7. When all issues in the release plan are merged into `feature/release-<id>`:
       - Invoke **merge agent expert** to merge the release PR into `main`
       - Resolve merge conflicts (if any) using release-cycle context + memories
       - Verify: build/test gates + release tag correctness
       - Broadcast to swarm: "new version published" (agent bus)
    8. If no issues and no PRD tasks and all workers idle:
       - Run `dev/harness/review-self.sh` (creates new issues → feeds the loop)
    9. `sleep $INTERVAL`
  - [ ] Log each heartbeat iteration to `dev/audits/director.log`

  > **Note (memory model):** Durable shared memory is **git-tracked** as engram JSON files under `/memory/st/<batch_id>/` (short-term) and `/memory/lt/<batch_id>/` (long-term), with an authoritative Director-maintained LUT at `memory/catalog.json`. The Director enforces **bounded policies** (max ST/LT files/bytes, max engram size).
  >
  > `MEMORY.md` is a **tracked**, Director-managed **shared “focus” view** derived from recent high-signal ST engrams (and optionally the latest event window). It is regenerated/updated by the Director at a controlled cadence, kept small via an LRU/size cap, and is used to inject “what matters right now” context into prompts. Durable shared memory still lives in the engram files + catalog; `MEMORY.md` is intentionally regeneratable.

### 0.8 — Setup Script (install system prereqs — run once per machine)

- [ ] Write `dev/scripts/setup.sh` — System prerequisite installer:
  - [ ] Check and install `gh` CLI (via `apt` or official installer)
  - [ ] Check and install `jq` (via `apt`)
  - [ ] Check and install `github-mcp-server` binary (download from GitHub Releases)
  - [ ] Verify `copilot` CLI is available
  - [ ] Verify `cmake` and `g++` are available
  - [ ] Prompt for `GITHUB_PERSONAL_ACCESS_TOKEN` if unset
  - [ ] Run `gh auth login` if not authenticated
  - [ ] All installs are idempotent — safe to re-run
  - [ ] Print summary of installed/verified tools with versions

### 0.9 — Bootstrap Script (initialize project — run once per clone)

- [ ] Write `dev/scripts/bootstrap.sh` — Project initialization:
  - [ ] Verify setup.sh was run (check for all required tools)
  - [ ] Verify `GITHUB_PERSONAL_ACCESS_TOKEN` is set
  - [ ] Verify gh is authenticated: `gh auth status`
  - [ ] Create `dev/mcp/github-mcp.json` if missing
  - [ ] Create label taxonomy on repo (idempotent — skip existing labels)
  - [ ] Create `dev/plans/prd.json` with initial bootstrap tasks
  - [ ] Create initial GitHub Issues for remaining Phase 0 items
  - [ ] Run `./scripts/build.sh` to verify C++ scaffold compiles
  - [ ] Run `dev/harness/review-self.sh` to seed first issues
  - [ ] Print: "Bootstrap complete. Run: ./dev/scripts/daemon.sh"

### 0.10 — Initial Backlog & First Issues

- [ ] Create `dev/plans/prd.json` — Initial task backlog mapping Phase 0 items
- [ ] Create seed GitHub Issues manually or via bootstrap:
  - Issue: "Implement RunConfig TOML parsing" → `agent-task`, `architecture`
  - Issue: "Implement AuditIdFactory" → `agent-task`, `architecture`
  - Issue: "Add --help and subcommand stubs to CLI" → `agent-task`, `docs`
  - Issue: "Write first Catch2 unit test" → `agent-task`, `testing`
  - Issue: "Implement CopilotRunnerSystem subprocess wrapper" → `agent-task`, `architecture`

### 0.11 — End-to-End Smoke Test (Director self-codes for the first time)

This is the acceptance test for Phase 0. If this passes, you have a self-coding
system equivalent to running Claude Code in an autonomous loop.

- [ ] Run `./dev/scripts/setup.sh` — verify all tools installed
- [ ] Run `./dev/scripts/bootstrap.sh` — verify clean exit
- [ ] Run `./dev/scripts/daemon.sh` — let it execute **one full heartbeat** (supervisor + Director)
- [ ] Verify: Director read open issues (printed to log)
- [ ] Verify: Director created a worktree for an issue
- [ ] Verify: Copilot CLI agent was invoked in the worktree with `--allow-tools write`
- [ ] Verify: Agent produced code changes (files modified in worktree)
- [ ] Verify: `scripts/build.sh` ran in worktree (CMake output in audit log)
- [ ] Verify: `scripts/test.sh` ran in worktree
- [ ] Verify: Agent created a PR via GitHub MCP (PR visible on GitHub)
- [ ] Verify: Code-review agent posted a review comment on the PR
- [ ] Verify: Director merged the worker PR into the active release branch
- [ ] Verify: Merge agent merged the release PR into `main` (or flagged for human review)
- [ ] Verify: Director tore down the worktree after merge
- [ ] Verify: Audit bundle exists under `dev/audits/` with ledger files
- [ ] Run `dev/harness/review-self.sh` independently — verify it creates ≥1 issue
- [ ] Let Director run for 3+ heartbeats — verify the circular loop:
  - Self-review creates issues → agent fixes them → PRs merged → self-review again
- [ ] **MILESTONE: Director is autonomously coding. Phase 0 complete.**

## Phase 1: Self-Review Loop (Self-Improvement Begins)

- [ ] Implement complete `dev/harness/review-self.sh`:
  - [ ] Run all persona agents (security, performance, architecture, testing, docs) on own code
  - [ ] Output findings as structured JSON
  - [ ] Create GitHub Issues for critical/high severity findings
  - [ ] Add appropriate labels (`auto-review`, persona, priority)
  - [ ] Deduplicate against existing open issues (avoid duplicates)
- [ ] Implement coding agent workflow:
  - [ ] Agent reads issue description via GitHub MCP
  - [ ] Agent creates fix branch `fix/<issue>-<desc>`
  - [ ] Agent implements fix in worktree
  - [ ] Agent writes tests for the fix
  - [ ] Agent runs build + test
  - [ ] Agent creates PR via GitHub MCP targeting the active release branch
  - [ ] Agent links the issue with `Refs #<issue>` (release PR closes issues)
  - [ ] Agent adds `agent-review` label for code-review
- [ ] Implement Director merge logic:
  - [ ] Code-review agent reviews PR
  - [ ] Director validates build/test pass
  - [ ] Director merges worker PR into the active release branch
  - [ ] Director tears down worktree
- [ ] Verify circular loop: review → issue → fix → PR → merge → review
- [ ] Director runs indefinitely, self-improving the codebase

## Phase 2: Core App Skeleton (C++)

> Specs: `components/RunConfig.md`, `entities/AuditIdFactory.md`, `components/AuditRecord.md`, `components/PromptRecord.md`, `components/ReviewFinding.md`, `components/ReviewInput.md`

- [ ] `app/src/cli/main.cpp` — Entry point, arg parsing (subcommands: `demo`, `review`, `pr`, `fix`, `watch`, `ui`)
- [ ] `app/src/core/run_config.h/.cpp` — `RunConfig` struct: load from TOML, CLI flag overrides
- [ ] `app/src/core/audit_id_factory.h/.cpp` — `AuditIdFactory`: epoch-based UUID generation
- [ ] `app/src/core/audit_record.h/.cpp` — `AuditRecord` struct: JSON serialization
- [ ] `app/src/core/prompt_record.h/.cpp` — `PromptRecord` struct: prompt ledger
- [ ] `app/src/core/review_finding.h/.cpp` — `ReviewFinding` struct
- [ ] `app/src/core/review_input.h/.cpp` — `ReviewInput` struct: diff + metadata
- [ ] `app/config/reviewcat.example.toml` — Example config file
- [ ] `app/config/personas/` — Default persona prompt templates
- [ ] `reviewcat demo` — Bundled sample diff → persona review → markdown output
- [ ] Unit tests (Catch2): `test_run_config.cpp`, `test_audit_id.cpp`, `test_review_finding.cpp`

## Phase 3: Review Pipeline (C++)

> Specs: `systems/RepoDiffSystem.md`, `systems/CopilotRunnerSystem.md`, `systems/PersonaReviewSystem.md`, `systems/SynthesisSystem.md`, `systems/AuditStoreSystem.md`, `systems/CodingAgentSystem.md`, `entities/PromptFactory.md`

- [ ] `RepoDiffSystem` — `libgit2` or `git diff` subprocess, diff chunking
- [ ] `CopilotRunnerSystem` — `copilot -p` subprocess wrapper with MCP config:
  - [ ] Capture stdout/stderr, exit code
  - [ ] JSON response parsing and validation
  - [ ] Record/replay mode for deterministic testing
  - [ ] Prompt ledger recording
- [ ] `PromptFactory` — Build prompts from persona templates + diff chunks
- [ ] `PersonaReviewSystem` — Iterate personas, call CopilotRunner, validate JSON
- [ ] `SynthesisSystem` — Dedupe, prioritize, unified markdown, action plan JSON
- [ ] `CodingAgentSystem` — Dispatch coding agents to fix findings:
  - [ ] Create GitHub Issue from finding (via GitHub MCP)
  - [ ] Dispatch coding agent in worktree
  - [ ] Create PR linking issue
- [ ] `AuditStoreSystem` — Write artifacts, maintain index
- [ ] `reviewcat review <path>` end-to-end
- [ ] Integration tests with record/replay

## Phase 4: GitHub Integration (Runtime)

> Specs: `systems/GitHubOpsSystem.md`

- [ ] `GitHubOpsSystem` via GitHub MCP Server:
  - [ ] Create issues from review findings
  - [ ] Create PRs from coding agent fixes
  - [ ] Post unified review comments on PRs
  - [ ] Read PR diffs, issue descriptions
  - [ ] Handle rate limits, token validation
  - [ ] Fallback to `gh` CLI if MCP unavailable
- [ ] `reviewcat pr <PR#>` — Review a PR end-to-end
- [ ] `reviewcat watch` — Daemon mode: poll, auto-review, auto-fix
  - [ ] Configurable poll interval
  - [ ] PID file + graceful shutdown
  - [ ] Circular: review → issues → coding agent → PR → merge → review
- [ ] Support remote user repos (not just self-review)

## Phase 5: Patch Automation

> Specs: `systems/PatchApplySystem.md`

- [ ] `PatchApplySystem`:
  - [ ] Generate unified diffs from Copilot fix suggestions
  - [ ] Dry-run in temp worktree
  - [ ] Apply with backup + safety checks
- [ ] `reviewcat fix <path>` — Review + auto-apply patches

## Phase 6: End-User UI (C++)

- [ ] Set up SDL3 window/input + custom ToolUI (primitives + bitmap font) in `app/src/ui/`
- [ ] Add top status bar (single-line hotkey helper text)
- [ ] Add bottom status bar (focused panel/view state + status info)
- [ ] Define UI layering/z-index model (world viewport + overlay panels + modals)
- [ ] (Optional) Swarm visualizer UI surface (3D task graph) driven by agent-bus telemetry
- [ ] Dashboard panel — active review status, recent findings, daemon health
- [ ] **Agent Status panel** — live view of:
  - [ ] Active agents and their current tasks
  - [ ] Worktree status (branch, issue, progress)
  - [ ] Worker slot utilization (`active / MAX_WORKERS`)
  - [ ] Heartbeat health indicator (time since last beat)
  - [ ] Per-agent retry counts and error states
- [ ] Settings panel — load/save `reviewcat.toml`
- [ ] Stats panel — review counts, severity breakdown, persona activity
- [ ] Audit Log panel — browse past runs, view findings
- [ ] **Log Viewer panel** — live scrolling log with level filtering (ToolUI text rendering)
- [ ] Controls panel — start/stop daemon, trigger manual review
- [ ] `reviewcat ui` subcommand

## Phase 7: Polish & Distribution

- [ ] Single static binary via static linking
- [ ] Comprehensive error messages (spdlog logging active from Phase 0)
- [ ] End-user documentation (`docs/USAGE.md`, `docs/CONFIGURATION.md`)
- [ ] CI/CD pipeline (GitHub Actions): CMake build, Catch2 tests, static analysis
- [ ] README: build instructions, quick start, screenshots

---

## Cross-Cutting Concerns

- [ ] Establish C++ code style (`clang-format` config)
- [ ] Set up `clang-tidy` for static analysis
- [ ] Add `compile_commands.json` generation in CMake
- [ ] Document WSL prerequisites (Ubuntu packages, Copilot CLI, GitHub MCP)
- [ ] Configure GitHub MCP Server for dev agents (documented in `dev/scripts/`)
- [ ] Security: never log or store tokens; redact sensitive paths in audits
- [ ] **Logging infrastructure:**
  - [ ] Dev harness: timestamped log functions in `dev/harness/log.sh`
  - [ ] Dev harness: all scripts source `log.sh` for consistent output
  - [ ] Dev harness: Director writes to `dev/audits/director.log`
  - [ ] C++ app: integrate spdlog (console + rotating file sinks)
  - [ ] C++ app: log file at `~/.reviewcat/reviewcat.log`
  - [ ] C++ app: `--log-level` CLI flag (trace/debug/info/warn/error)
  - [ ] C++ app: UI sink for Log Viewer panel (no Dear ImGui dependency)
- [ ] **Test infrastructure:**
  - [ ] Reference: `docs/dev/TESTING_STRATEGY.md` (philosophy, conventions, determinism).
  - [ ] `scripts/test.sh` runs Catch2 binary + reports results
  - [ ] Record/replay fixtures for Copilot CLI responses
  - [ ] Golden file tests for demo output
  - [ ] Integration test for Director single-heartbeat
  - [ ] Test coverage reporting (gcov/lcov)

---

## Notes

- **Dev harness** items (Phase 0–1) are all **bash/shell** — no C++ compilation needed.
- **App** items (Phase 2–7) are all **C++** — require CMake build.
- Phase 0 is the **highest priority** — get the Director running so it can autonomously implement Phases 1+.
- Phase 1 enables the **self-improvement loop** — ReviewCat reviews and fixes itself.
- All agent work is tracked via **GitHub Issues and PRs** on `p3nGu1nZz/Review-Cat`.
- Agents run in parallel via **worker containers** (shared image tag) + **git worktrees** (one worktree mounted per worker).
- The `dev/` agents use **GitHub MCP Server** via either **remote MCP** (preferred) or a **native stdio binary** (host or container-bundled).
- Setup script (`dev/scripts/setup.sh`) installs system prereqs.
- Bootstrap script (`dev/scripts/bootstrap.sh`) initializes the project.
- Mark items `[x]` as they are completed; primary tracking is via GitHub Issues.
