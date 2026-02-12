# ReviewCat

**Autonomous AI-powered code review daemon** — uses GitHub Copilot CLI to
provide persona-based code review, unified synthesis, and optional patch
automation for any Git repository.

ReviewCat is also **self-building**: a development harness powered by Copilot
CLI autonomously develops, tests, and iterates on its own codebase.

The workflow is intentionally cyclic:

> **bootstrap → dev → app → dev → app → …**

## Project Status (Docs-first → Code)

This repository is currently **planning/specs-first**: the docs and specs are the
source of truth, and the implementation is being bootstrapped in Phase 0.

- The planned scaffold (`app/`, `dev/`, `scripts/`, `.github/`) is **not fully
	present yet**.
- The next milestone is to create the minimal repo scaffold + a “green”
	build/test gate so autonomous cycles can run end-to-end.

Tracking issue: **#16 — _Bootstrap Repo Scaffold + Green Build/Test Gate_**.

## Key Features

- **Persona-based review** — Security, performance, architecture, testing, and docs agents.
- **Unified synthesis** — Deduplicated, prioritized findings in markdown + JSON.
- **Audit trail** — Every Copilot CLI call recorded in a prompt ledger.
- **GitHub integration** — PR comments, issues, and patches via GitHub MCP Server.
- **Daemon mode** — Continuous monitoring with configurable intervals.
- **Native UI** — SDL3-based UI surfaces (settings, stats, control) and an optional 3D swarm visualizer.
- **Self-building** — Director daemon (Agent 0) that orchestrates development.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | C++17/20 |
| Build system | CMake |
| Dev harness | Bash shell scripts |
| UI | SDL3 (window/input) + custom ToolUI (primitives + bitmap glyph font + hex colors) + optional 3D swarm visualizer |
| Copilot integration | `copilot -p` subprocess |
| GitHub integration | GitHub MCP Server (remote MCP recommended; native binary or containerized for dev harness) |
| JSON | nlohmann/json |
| Config | TOML (toml++) |
| Testing | Catch2 |
| Logging | spdlog (C++), structured bash logging (dev) |
| Git | libgit2 + `gh` CLI |

## Repository Structure

**Target structure (planned):**

```
Review-Cat/
├── app/                    # The product (C++ binary)
│   ├── src/                # C++ source code
│   │   ├── core/           # Core library
│   │   ├── cli/            # CLI frontend
│   │   ├── daemon/         # Daemon / watch mode
│   │   ├── ui/             # SDL3 + custom ToolUI surfaces (and optional 3D visualizer client)
│   │   └── copilot/        # Copilot CLI subprocess bridge
│   ├── include/            # Public headers
│   ├── tests/              # Catch2 test suite
│   ├── config/             # Default configs, persona templates
│   └── CMakeLists.txt      # App build system
│
├── dev/                    # Development harness (bash scripts)
│   ├── agents/             # Role agent prompt files
│   ├── harness/            # Director daemon & cycle scripts
│   ├── plans/              # PRD backlog (prd.json)
│   ├── prompts/            # Prompt templates for dev agents
│   ├── mcp/               # MCP config files (github-mcp.json)
│   ├── scripts/            # Setup & bootstrap scripts
│   └── audits/             # Development audit bundles
│
├── docs/                   # Design docs & specifications
│   ├── INDEX.md            # Documentation landing page
│   ├── ARCHITECTURE.md     # High-level architecture overview
│   ├── dev/                # Dev harness docs
│   ├── app/                # Runtime app docs
│   └── specs/              # Legacy monolithic spec tree (migration in progress)
│
├── scripts/                # Top-level build/test/clean scripts
├── config/                 # Dev harness config (e.g., config/dev.toml)
├── PLAN.md                 # Comprehensive project plan
└── TODO.md                 # Actionable task list
```

**Current structure (today):** docs/specs + planning files + `config/dev.toml`.
See [TODO.md](TODO.md) Phase 0 and Issue #16 for the bootstrapping path.

## Quick Start (after Phase 0 scaffold exists)

> Note: the commands below assume Phase 0 scaffolding has been implemented.
> If you’re looking for what to do right now, start with [TODO.md](TODO.md)
> Phase 0 and Issue #16.

```bash
# Prerequisites: CMake, C++17 compiler, Copilot CLI, gh CLI
# Dev harness prereq (recommended/required for autonomous loop): Docker (WSL2-friendly)

# Clone
git clone https://github.com/<owner>/Review-Cat.git
cd Review-Cat

# Install system prereqs (gh, jq, github-mcp-server)
./dev/scripts/setup.sh

# Build
./scripts/build.sh

# Run demo (no auth required)
./build/reviewcat demo

# Review local changes
./build/reviewcat review

# Review a PR
./build/reviewcat pr 42 --repo OWNER/REPO

# Start daemon mode
./build/reviewcat watch --interval 60

# Open UI
./build/reviewcat ui
```

## Development (Self-Building)

The **dev harness** (once implemented) assumes a Docker-capable environment
(Linux, or Windows + WSL2 + Docker Desktop integration): one shared image tag,
many worker containers, one git worktree mounted per worker.

Recommended host health checks:

- `docker version`
- `docker info`
- `docker run --rm hello-world`

GitHub integration for agents is via the GitHub MCP Server. Example MCP configs live in `dev/mcp/`:

- `dev/mcp/github-mcp.json` (remote HTTP MCP; preferred MVP)
- `dev/mcp/github-mcp-docker.json` (local Docker container; stdio; offline-friendly)
- `dev/mcp/github-mcp-stdio.json` (local stdio binary; fallback/offline)

Secrets (PATs) are provided via environment variables. See `.env.example` and `docs/dev/ENVIRONMENT.md`.

The Director’s scheduling, timeouts, worker telemetry, and agent-bus networking are configured via `config/dev.toml`.

Project memory is shared in two layers (see `AGENT.md` and Issue #13):

- **Durable engrams** committed under `/memory/` (with `memory/catalog.json` as the authoritative LUT and engrams stored in timestamped batches)
- A **shared, bounded focus view** (`MEMORY.md`, tracked) maintained by the Director and derived from recent high-signal ST/LT context

The development harness runs autonomously via the Director daemon (once the
Phase 0 scaffold + scripts exist):

```bash
# Install system prereqs
./dev/scripts/setup.sh

# Bootstrap dev environment
./dev/scripts/bootstrap.sh

# Start daemon supervisor + Director (Agent 0)
./dev/scripts/daemon.sh
```

The dev harness operates in **release cycles**:

- the Director maintains a release branch/PR (`feature/release-*` → `main`)
- worker PRs target the release branch
- a dedicated merge agent finalizes the release into `main` and verifies the tag

Each checkout may contain a gitignored root `STATE.json` used for **local cached
state** (first-run vs resume, active release context). It is created lazily and
never committed.

The Director reads `dev/plans/prd.json`, delegates to Copilot CLI role agents,
validates via build + test, and commits. See [PLAN.md](PLAN.md) §5 for details.

## Documentation

| Document | Purpose |
|---------|---------|
| [PLAN.md](PLAN.md) | Comprehensive project plan (golden source of truth) |
| [TODO.md](TODO.md) | Actionable task list by phase |
| [AGENT.md](AGENT.md) | Agent system overview: roles, guardrails, agent bus, memory protocol |
| [Docs Index](docs/INDEX.md) | Documentation landing page (dev vs app) |
| [Design Doc](docs/dev/COPILOT_CLI_CHALLENGE_DESIGN.md) | Architecture, UX, integration design |
| [Director Workflow](docs/dev/DIRECTOR_DEV_WORKFLOW.md) | DirectorDev recursive development spec |
| [Implementation Checklist](docs/dev/IMPLEMENTATION_CHECKLIST.md) | Step-by-step checklist |
| [Prompt Cookbook](docs/dev/PROMPT_COOKBOOK.md) | Curated prompt patterns |
| [Specs (Dev)](docs/dev/specs/SPECS.md) | Dev harness agent + orchestration specifications |
| [Specs (App)](docs/app/specs/SPECS.md) | Runtime app component/entity/system specifications |

## Specs

Specs are split by concern:

- Dev harness specs index: `docs/dev/specs/SPECS.md`
- Runtime app specs index: `docs/app/specs/SPECS.md`

During migration, many specs still live physically under `docs/specs/`.
Use the indices above for navigation.

## License

See [LICENSE](LICENSE).
