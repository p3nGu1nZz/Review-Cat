# DirectorDev Workflow (Self-Improving Development Coordination)

This document specifies how the ReviewCat project builds and improves itself
using GitHub Copilot CLI agents, the GitHub MCP Server, parallel git worktrees,
and a Director daemon (Agent 0) that orchestrates the circular self-improvement
loop.

At the project level, ReviewCat evolves in a deliberate loop:

> **bootstrap → dev → app → dev → app → …**

The dev harness improves the repo; the app grows features; and each informs the
other continuously.

> **Directive (MVP worker execution model):** The dev harness is Docker-first.
> Use **one shared Docker image tag** for all workers, run **one worker
> container per task**, and bind-mount **one git worktree per worker** into the
> container as `/workspace`. Prefer **scale-to-zero** (stop/remove idle worker
> containers) to keep the system cheap when idle.

> **See also:** [PLAN.md](../PLAN.md) §5 for heartbeat architecture,
> [AGENT.md](../AGENT.md) for the swarm operating contract,
> [TODO.md](../TODO.md) Phase 0–1 for implementation tasks, and
> [COPILOT_CLI_CHALLENGE_DESIGN.md](COPILOT_CLI_CHALLENGE_DESIGN.md) for full
> design context.

The key intent is not full autonomy. The intent is:

- consistent decomposition
- explicit acceptance criteria
- repeatable build/test loops
- recorded evidence of Copilot CLI usage
- autonomous forward progress with human oversight
- circular self-improvement via GitHub Issues and PRs

## Local cached state (`STATE.json`)

Every checkout (main worktree and worker worktrees) may contain a gitignored
`STATE.json` at repo root. The Director and workers create it lazily if absent.
It stores **local cached state** (first-run vs resume, current release context,
last-seen SHAs/hashes). It is never committed.

## Roles

DirectorDev coordinates these roles:

- **Director**: decomposes work, enforces scope, merges outputs, validates acceptance.
- **Architect**: reviews architecture changes, watches complexity.
- **Implementer**: writes code for spec-driven work.
- **Coder**: implements fixes for GitHub Issues (review findings), creates PRs.
- **QA**: writes tests and adds record/replay fixtures.
- **Docs**: maintains README, examples, prompt cookbook.
- **Security**: enforces safe defaults, redaction, permission policy.
- **Code-review**: reviews diffs and blocks low-signal changes.
- **Merge expert**: finalizes releases by merging the release PR into `main`.

In Copilot CLI, these roles are expressed as custom agents:

- Agent profiles live in `.github/agents/` (Copilot CLI repo-level agents).
- Role-specific prompt files live in `dev/agents/` (e.g., `architect.md`,
  `implementer.md`, `coder.md`, `qa.md`, `docs.md`, `security.md`,
  `code-review.md`).
- All agents are invoked via `copilot -p @dev/agents/<role>.md "..."`.
- Agents use the **GitHub MCP Server** for issue/PR operations when configured
  with `--mcp-config github-mcp.json`.

All development tooling is **bash shell scripts** under `dev/`.
No C++ compilation is required for the dev harness to operate.

## Inputs

- A target spec file in `docs/specs/`.
- Current repository state.
- PRD backlog (`dev/plans/prd.json`) with task items and status.
- Open GitHub Issues labeled `agent-task` (via GitHub MCP Server).
- Optional constraints:
  - time budget
  - risk tolerance (dry-run vs automation)
  - scope lock (files in scope for the target spec)

## Outputs

- Code changes implementing the spec or fixing the issue.
- Tests proving acceptance criteria.
- Updated docs.
- GitHub PRs linking source issues:
  - worker PRs use `Refs #N`
  - the release PR aggregates `Closes #N` so issues close when merged to `main`
- GitHub Issue status updates (labels, comments).
- A development audit bundle under `dev/audits/<audit_id>/`:
  - `ledger/` — prompt/response pairs per agent
  - `build.log` — build output
  - `test.log` — test output
  - `diff.patch` — before/after diff
  - `summary.md` — Director's completion note

Audit bundles are indexed in `dev/audits/index.json`.
The Director also logs each heartbeat to `dev/audits/director.log`.

## Orchestration Algorithm

The Director runs as a **bash heartbeat daemon** (`dev/harness/director.sh`).
For day-to-day operation we recommend a keep-alive supervisor wrapper
(`dev/scripts/daemon.sh`) that restarts the Director cleanly and coordinates
upgrade safe points.
See [PLAN.md](../PLAN.md) §5 for the full heartbeat loop pseudocode.

### Per-Heartbeat Algorithm

1. **Ensure release context** — Maintain an active release branch and release PR:
  - branch: `feature/release-<release_id>`
  - PR: `feature/release-<release_id>` → `main`
  - store `{release_id, branch, pr_number}` in `STATE.json`
  - if a **new release cycle** begins, archive the previous `TODO.md` into
    `/archive/` (see `AGENT.md` and the TODO lifecycle note in `TODO.md`)
2. **Scan GitHub** — List open issues labeled `agent-task` via GitHub MCP.
3. **Check PRD** — Read `dev/plans/prd.json` for spec-driven work.
4. **Select release batch** — Decide which issues are in-scope for the current release.
5. **Check capacity** — Count active worktrees vs `MAX_WORKERS`.
6. **Claim issues** — For each unclaimed issue within capacity, add
  `agent-claimed` label via GitHub MCP.
7. **Dispatch workers** — For each claimed issue:
  - create worktree via `dev/harness/worktree.sh create`
  - start a worker container that executes `dev/harness/run-cycle.sh <issue> <branch>`
    with the worktree bind-mounted as the container workspace
  - pass the active release branch name so worker PRs target the release branch
8. **Monitor workers** — Via `dev/harness/monitor-workers.sh`:
  - check completed worktrees
  - ingest worker heartbeats/status from the agent bus (real-time state)
  - correlate with Docker container state (running/exited/exit code)
  - validate build/test in completed worktrees
  - worker creates PR via GitHub MCP targeting the active release branch
  - code-review agent reviews PR
  - on pass: merge worker PR into the release branch, teardown worktree
  - on fail: retry (max 3), label `agent-blocked`

Real-time worker state is also emitted onto the agent bus. That telemetry is
intended to power an optional **swarm visualizer** client (SDL3-based) that can
render a live task graph and provide operator controls (start/stop/pause).
7. **Finalize release** — When all planned issues are merged into the release branch:
  - invoke the merge agent expert to merge the release PR into `main`
  - resolve merge conflicts if needed
  - re-run validation gate(s)
  - create/verify the release tag
  - close issues included in the release (or ensure the release PR closes them)
  - broadcast `release_published` on agent bus so workers can upgrade
8. **Self-review** — When idle (no issues, no PRD tasks):
   - Run `dev/harness/review-self.sh`.
   - Create new GitHub Issues for critical/high findings.
9. **Sleep** — Wait for configurable interval (default: 60s).

### Per-Task Algorithm (run-cycle.sh)

1. **Read issue** — Get issue details via GitHub MCP.
2. **Load spec** — If issue links a spec, load from `docs/specs/`.
3. **Invoke agents** — Run role agents sequentially in worktree:
   - Implementer/Coder: write code.
   - QA: write tests.
   - Docs: update documentation.
   - Security: review for vulnerabilities.
   - Code-review: review the diff.
4. **Sync context** — Before final validation/PR readiness, bring the worker branch up to date with `main` (Director policy):
  - this pulls in updated specs/protocols
  - this pulls in newly merged engrams under `/memory/` (durable shared memory)
5. **Validate** — `./scripts/build.sh && ./scripts/test.sh`.
6. **Create PR** — Via GitHub MCP targeting the active release branch and using
   `Refs #<issue>` (the release PR is what closes issues on merge to `main`).
7. **Record audit** — Bundle all artifacts.

If the swarm is over memory budget, the Director may also schedule a memory-maintenance task:

- compact older agent-bus event slices into structured engrams
- propose new `/memory/...` engram files via PR (memory agent)

## Parallel Execution via Worktrees

> In the Docker-first harness, parallelism is **worker containers + worktrees**:
> each worker container runs one task against its own isolated worktree.

Multiple agents work simultaneously in isolated git worktrees:

```
~/source/repos/
├── Review-Cat/                    # Main worktree (Director runs here)
├── Review-Cat-agent-42-1707600000/ # Worker 1 (implementing issue #42)
├── Review-Cat-agent-57-1707600060/ # Worker 2 (implementing issue #57)
└── Review-Cat-agent-63-1707600120/ # Worker 3 (implementing issue #63)
```

- Each worktree has its own branch, build directory, and test artifacts.
- Each worker has its own container lifecycle and environment variables.
- Director manages up to `MAX_WORKERS` concurrent worktrees.
- Operational coordination between agents is primarily via GitHub Issue/PR comments.
- Shared context is distributed via `/memory/**` engrams, the Director LUT at `memory/catalog.json`, and the tracked `MEMORY.md` focus view.
- Worktrees are torn down after PRs are merged.

## Inter-Agent Communication via GitHub

Agents communicate through GitHub's native features:

| Mechanism | Purpose | Example |
|-----------|---------|---------|
| **Issues** | Work items and findings | Review agent creates issue for a bug |
| **Issue comments** | Discussion and clarification | Architect comments on complexity |
| **PRs** | Code implementations | Coder agent creates PR fixing issue |
| **PR comments** | Review feedback | Code-review agent comments on PR |
| **Labels** | Categorization and routing | `agent-task`, `security`, `in-progress` |
| **Linked issues** | Traceability | Worker PR says "Refs #42"; release PR aggregates "Closes #42" |

Canonical reference: `docs/dev/GITHUB_LABELS.md`.

### Issue claiming (exclusive lock)

To prevent duplicate work, issue claiming is treated as a **lock acquisition**.

**Rules:**

- Only the **Director** may claim issues. Workers must never self-claim.
- Claiming MUST be done by:
  1) removing `agent-task`
  2) adding `agent-claimed`
  3) posting a machine-parseable claim comment

**Claim comment format:**

The comment MUST start with `ReviewCat-Claim:` and include the fields needed for audit + recovery.

Example:

```text
ReviewCat-Claim:
  claimed_by: director
  run_id: <run_id>
  worker_id: <worker_id>
  claimed_at: 2026-02-11T23:59:59Z
  release_branch: feature/release-20260211-235959Z
```

**Reclaiming a stale claim:**

If a worker is stuck (heartbeat TTL exceeded / no progress for a configured window), the Director may reclaim:

- add a `ReviewCat-Reclaim:` comment explaining why
- remove `agent-claimed`, re-add `agent-task`
- optionally apply `agent-blocked` if human input is required

## Self-Review Loop

The core innovation: ReviewCat reviews itself, generates issues, fixes them,
and repeats indefinitely:

```
Review own code → Create GitHub Issues → Coding agent fixes →
Create PRs → Merge PRs → Review again → ...
```

This loop runs automatically when the Director has no other work. It enables
ReviewCat to bootstrap from minimal hardcoded scripts and progressively develop
itself into a full product.

## Coding Agent Workflow

When a review finding is promoted to a GitHub Issue, the Coder agent:

1. **Reads the issue** via GitHub MCP.
2. **Creates a branch** — `fix/<issue>-<description>`.
3. **Works in a worktree** — isolated environment.
4. **Implements the fix** — C++17/20 code following project conventions.
5. **Writes tests** — Catch2 test cases validating the fix.
6. **Runs build + test** — validates everything compiles and passes.
7. **Creates a PR** — via GitHub MCP, `Refs #<issue>` (issues close when the release PR merges to `main`).
8. **Requests review** — adds `agent-review` label.

## Bootstrap Sequence

To cold-start the Director for the first time:

```bash
cd Review-Cat
./dev/scripts/setup.sh        # Install prereqs (gh, jq, github-mcp-server)
./dev/scripts/bootstrap.sh    # Configure MCP, create issues, verify build
  ./dev/scripts/daemon.sh       # Start supervisor + Director (recommended)
```

(`dev/harness/director.sh` remains the underlying heartbeat loop; `dev/scripts/daemon.sh` adds keep-alive + upgrade coordination.)

## Guardrails

- **Dry-run is default** — no `git push`, no GitHub mutations.
- **Scope lock** — Director refuses to modify files outside the spec's scope.
- **Retry budget** — Max 3 retries per sub-task before marking failed.
- **Dangerous command deny list** — `rm -rf /`, `git push --force`, etc.
- **Watchdog timeout** — Kill subprocess if a cycle exceeds time limit.
- **Permission profiles** — Copilot CLI `--allow-tools` / `--deny-tools` flags.
- **Worktree isolation** — Agents cannot modify the main worktree directly.
- **PR-gated merges** — All changes go through PRs, never direct main commits.
- **Label-based claiming** — Agents claim issues before starting work to avoid
  duplicate effort across parallel workers.
- **Severity threshold** — Auto-review only creates issues for critical/high findings.
- **Deduplication** — Self-review checks for existing open issues to prevent flooding.
- **PID file** — Single-instance enforcement for the Director daemon.

## Upgrade safety points (worker containers)

When a new version is published during an active swarm run, workers should only
restart into the new shared image tag at **safe points**:

- between commits
- not mid-rebase/merge
- ideally with a clean working tree

If a worker is mid-edit and safe to discard, it may reset its worktree back to
`HEAD` before restart. Otherwise, it should finish the critical git operation
and restart at the next safe point.

## Recursive and Circular Behavior

DirectorDev is recursive in the sense that:

- If a spec requires new subsystems, DirectorDev creates additional specs first.
- Each new spec becomes a new loop iteration.

DirectorDev is circular in the sense that:

- Self-review generates issues → coding agents fix them → PRs merge →
  self-review runs again on the updated code.
- This loop continues indefinitely while the daemon is active.

This keeps work modular and avoids monolithic changes, while enabling continuous
self-improvement of the ReviewCat codebase.

## How to Demonstrate

- Run one small DirectorDev cycle:
  ```bash
  ./dev/scripts/daemon.sh  # or run a single cycle manually
  ```
- Show:
  - the spec file or GitHub Issue being implemented
  - the worktree created for the work
  - the resulting code change (`git diff`)
  - the test run (`./scripts/test.sh`)
  - the PR created via GitHub MCP
  - the prompt ledger (`dev/audits/<bundle>/ledger/`)

The prompt ledger is the proof that Copilot CLI was used meaningfully,
not just incidentally.
