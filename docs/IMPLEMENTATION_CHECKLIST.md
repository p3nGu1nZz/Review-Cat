# ReviewCat Implementation Checklist

This is a practical checklist aligned to [PLAN.md](../PLAN.md) and the
[design doc](COPILOT_CLI_CHALLENGE_DESIGN.md). It is intentionally explicit and
sequential.

> **Tech stack:** C++17/20, CMake, bash shell scripts. See PLAN.md §3 and §10.

## Phase 0: Repository Bootstrap & Dev Harness

- [ ] Create `app/` directory hierarchy:
  - [ ] `app/src/core/`, `app/src/cli/`, `app/src/daemon/`, `app/src/ui/`, `app/src/copilot/`
  - [ ] `app/include/`, `app/tests/`, `app/config/`
  - [ ] `app/CMakeLists.txt` with targets: `reviewcat` binary, `reviewcat_core` lib, `reviewcat_tests`
- [ ] Create `dev/` directory hierarchy:
  - [ ] `dev/agents/`, `dev/harness/`, `dev/plans/`, `dev/prompts/`, `dev/scripts/`, `dev/audits/`
- [ ] Write root `CMakeLists.txt` (delegates to `app/CMakeLists.txt`).
- [ ] Set up dependency management (vcpkg or git submodules) for:
  - [ ] `nlohmann/json`, `toml++`, `Catch2`, `libgit2`, `SDL3`, `spdlog`
- [ ] Write `scripts/build.sh` — CMake configure + build.
- [ ] Write `scripts/test.sh` — run Catch2 test binary.
- [ ] Write `scripts/clean.sh` — remove build artifacts.
- [ ] Write `dev/harness/log.sh` — shared logging functions for bash scripts.
- [ ] Write `dev/harness/daemon.sh` — keep-alive supervisor for Agent 0 (starts/restarts `dev/harness/director.sh`).
- [ ] Write `dev/harness/director.sh` — Director heartbeat loop skeleton (Agent 0).
- [ ] Write `dev/harness/run-cycle.sh` — agent cycle orchestration.
- [ ] Write `dev/scripts/setup.sh` — install system prereqs (gh, jq, github-mcp-server).
- [ ] Write `dev/scripts/bootstrap.sh` — project initialization (MCP config, labels, issues).
- [ ] Create agent profiles in `.github/agents/` and `dev/agents/`.
- [ ] Create `dev/plans/prd.json` initial backlog.
- [ ] Add a minimal `reviewcat --help` with command stubs.
- [ ] Create `.gitignore` (build/, *.o, *.a, etc.).
- [ ] Verify `./scripts/build.sh && ./scripts/test.sh` works (empty project compiles).

## Phase 1: Director Agent (Agent 0)

- [ ] Implement Director heartbeat loop in `dev/harness/director.sh`:
  - [ ] Read `dev/plans/prd.json` for backlog.
  - [ ] Pick highest-priority incomplete item.
  - [ ] Invoke `dev/harness/run-cycle.sh`.
  - [ ] Run `./scripts/build.sh && ./scripts/test.sh` as validation gate.
  - [ ] On success: mark complete, `git commit`.
  - [ ] On failure: retry (max 3), log, skip.
  - [ ] Sleep for configurable interval.
- [ ] Implement `dev/harness/run-cycle.sh`:
  - [ ] Call role agents via `copilot -p @dev/agents/<role>.md "..."`.
  - [ ] Capture output to `dev/audits/<bundle>/ledger/`.
- [ ] Implement `dev/harness/record-audit.sh`:
  - [ ] Bundle: ledger + build/test logs + diff.
  - [ ] Write `dev/audits/index.json`.
- [ ] Write Director-facing prompt templates in `dev/prompts/`.
- [ ] Test Director with a trivial spec.

## Phase 2: Core App Skeleton (C++)

- [ ] `app/src/cli/main.cpp` — entry point, arg parsing (`demo`, `review`, `pr`, `fix`, `watch`, `ui`).
- [ ] `app/src/core/run_config.h/.cpp` — `RunConfig` struct (TOML → `toml++`).
- [ ] `app/src/core/audit_id_factory.h/.cpp` — epoch-based UUID generation.
- [ ] `app/src/core/audit_record.h/.cpp` — JSON serialization (`nlohmann/json`).
- [ ] `app/src/core/prompt_record.h/.cpp` — prompt ledger.
- [ ] `app/src/core/review_finding.h/.cpp` — finding struct.
- [ ] `app/src/core/review_input.h/.cpp` — diff + metadata.
- [ ] `app/config/reviewcat.example.toml` — annotated example config.
- [ ] `app/config/personas/` — default persona prompt templates.
- [ ] `reviewcat demo` — bundled sample diff → review → markdown.
- [ ] Unit tests: `test_run_config.cpp`, `test_audit_id.cpp`, `test_review_finding.cpp` (Catch2).

## Phase 3: Review Pipeline (C++)

- [ ] `RepoDiffSystem` — `libgit2` or `git diff` subprocess, diff chunking.
- [ ] `CopilotRunnerSystem` — `copilot -p` subprocess wrapper:
  - [ ] capture stdout/stderr, exit code
  - [ ] JSON response parsing and validation
  - [ ] record/replay mode for deterministic testing
  - [ ] prompt ledger recording
- [ ] `PromptFactory` — build prompts from persona templates + diff chunks.
- [ ] `PersonaReviewSystem` — iterate personas, call CopilotRunner, validate JSON.
- [ ] `SynthesisSystem` — dedupe, prioritize, unified markdown, action plan JSON.
- [ ] `AuditStoreSystem` — write artifacts, maintain index.
- [ ] `reviewcat review <path>` end-to-end.
- [ ] Integration tests with record/replay.

## Phase 4: GitHub Integration

- [ ] `GitHubOpsSystem` via **GitHub MCP Server** (primary) with `gh` CLI fallback:
  - [ ] Create issues from review findings (via MCP `create_issue`)
  - [ ] Create PRs from coding agent fixes (via MCP `create_pull_request`)
  - [ ] Post unified review comments on PRs (via MCP)
  - [ ] Read PR diffs and issue descriptions (via MCP)
  - [ ] Handle rate limits, token validation
  - [ ] Fallback to `gh` CLI if MCP unavailable
- [ ] `reviewcat pr <PR#>`.
- [ ] `reviewcat watch` — daemon mode:
  - [ ] Configurable poll interval
  - [ ] PID file + graceful shutdown
  - [ ] Circular: review → issues → coding agent → PR → merge → review
- [ ] Support remote user repos (not just self-review).

## Phase 5: Patch Automation

- [ ] `PatchApplySystem`:
  - [ ] generate unified diffs from fix suggestions
  - [ ] dry-run in temp worktree
  - [ ] apply with backup + safety checks
- [ ] `reviewcat fix <path>`.

## Phase 6: End-User UI (C++)

- [ ] Set up SDL3 window/input + custom ToolUI (primitives + bitmap font) in `app/src/ui/`.
- [ ] Top status bar: hotkey helper text (F1–F12)
- [ ] Bottom status bar: focused panel/view state + status info
- [ ] UI layering/z-index model: world viewport + overlay panels + modals
- [ ] Dashboard panel — active review status, recent findings, daemon health.
- [ ] **Agent Status panel** — live agent/worktree/heartbeat status.
- [ ] Settings panel — load/save `reviewcat.toml`:
  - [ ] Copilot credentials, GitHub access token, target repo, base branch
  - [ ] Review interval, personas enabled/disabled, auto-comment, auto-fix
  - [ ] Redaction rules
- [ ] Stats panel — review counts, severity breakdown, persona activity.
- [ ] Audit Log panel — browse past runs, view findings.
- [ ] **Log Viewer panel** — live scrolling log output with level filtering.
- [ ] Controls panel — start/stop daemon, trigger manual review.
- [ ] `reviewcat ui` subcommand to launch the window.

## Phase 7: Polish & Distribution

- [ ] Produce single static binary (`reviewcat`) via static linking.
- [ ] Comprehensive error messages (spdlog logging active from Phase 0).
- [ ] End-user documentation (`docs/USAGE.md`, `docs/CONFIGURATION.md`).
- [ ] Man page or `--help` with detailed subcommand docs.
- [ ] CI/CD pipeline (GitHub Actions): CMake build, Catch2 tests, static analysis.
- [ ] README: one-command demo, build instructions, quick start, troubleshooting.
- [ ] Include example artifacts (or screenshots) in the repo.
