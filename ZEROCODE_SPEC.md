# ZeroCode CLI — Fork Spec v2

Revised for the active upstream, `anomalyco/opencode` (MIT, TypeScript/Bun monorepo). Paste this into a local coding agent and run it **one milestone at a time**.

## 0. Role and working rules

You are a senior TypeScript/Bun engineer, AI-agent architect, terminal-UX designer, security engineer, DevOps engineer, technical writer, and open-source compliance specialist.

Working rules for every milestone:

1. Inspect before changing. Never assume upstream structure; read it and write down what you found.
2. Prefer upstream extension points (agents, commands, plugins, custom tools, skills, MCP config, permissions) over patching core. Every core patch must be listed in `docs/upstream-sync.md` with a reason.
3. Do not claim success without evidence. After each milestone run the build, lint, and tests, run the new commands for real, and paste the actual output into the milestone report. If something cannot be verified (for example no Docker available), say so plainly.
4. Work on a branch per milestone, commit in small conventional commits, and never push or force-push without my explicit approval.
5. If a requirement is ambiguous or conflicts with upstream, stop and ask instead of guessing.
6. Priorities: correctness over feature count, safety over autonomy, user control over automation, clean architecture over shortcuts, verified results over claimed ones, original implementation over copied code.

## 1. Project identity

- PROJECT_NAME = "ZeroCode CLI"
- CLI_COMMAND = "zerocode"
- REPOSITORY_NAME = "zerocode-cli"
- PACKAGE_NAME = "zerocode-cli" (npm; check availability and fall back to a scope such as `@<me>/zerocode-cli`)
- BASE_PROJECT = "OpenCode" (`https://github.com/anomalyco/opencode`)
- BASE_LICENSE = MIT, TARGET_LICENSE = MIT
- Config directory: `.zerocode/` (project) and `~/.config/zerocode/` (global). Config file: `zerocode.json` / `zerocode.jsonc`. Environment prefix: `ZEROCODE_`.
- Keep `AGENTS.md` as the rules file (it is an open convention, not upstream branding).

Before publishing, check that the name "ZeroCode" does not collide with an existing npm package, product, or trademark. Report the result; I will decide whether to rename.

## 2. Fork and license rules

1. I fork `anomalyco/opencode` manually on my GitHub account. Keep the original as an `upstream` remote.
2. Preserve the original `LICENSE` text and copyright notices. Add my copyright line alongside, never replacing theirs. Do not remove upstream notices from source files.
3. Create `NOTICE` stating that ZeroCode is based on OpenCode, links the upstream repository, and says ZeroCode is not affiliated with or endorsed by the OpenCode team.
4. Create `THIRD_PARTY_NOTICES.md` generated from the actual dependency tree (use a license-scanning tool for Bun/npm packages; commit the script that generates it). Flag any dependency whose license is not MIT, Apache-2.0, BSD, ISC, or similar permissive.
5. Rename the product, binary, package, docs, and visible UI text. Remove OpenCode names, logos, wordmarks, and brand assets. Upstream's README asks derivative projects not to imply affiliation; ZeroCode avoids the name and states non-affiliation.
6. Audit upstream for files with different licenses (assets, fonts, enterprise or hosted-service directories, vendored code) and list them in the notices. Remove anything not permissively licensed or not needed.
7. For Aider, Cline, Kilo Code, OpenHands, Goose, Pi, and Qwen Code: check each project's current LICENSE file (see the table in section 6), and default to clean-room reimplementation of ideas and workflows. Do not copy their source, prompts, UI text, or docs. If any code is ever reused, keep its license, copyright, and notices, and record it in `THIRD_PARTY_NOTICES.md`.

## 3. Phase 0 — inspect upstream (no code changes yet)

Produce `docs/architecture.md` containing a short map of the real repository:

- Monorepo layout: each package under `packages/`, what it does, and how packages depend on each other.
- Runtime and tooling: Bun, Turbo, TypeScript config, lint (e.g. oxlint), formatter, git hooks, test runner and where tests live, CI workflows, release scripts.
- Entry points: the CLI, the TUI, the server/client split, and the non-interactive `run` mode.
- Agent system: built-in agents (`build`, `plan`, `general` subagent), how agents are defined (markdown files and config), how subagents are invoked.
- Config system: file locations, merge order, variable substitution (`{env:...}`, `{file:...}`), managed settings.
- Permission system: schema and enforcement point.
- Provider/model layer: how providers and models are registered and called, how streaming and tool calls work, how auth is stored.
- MCP support: supported transports and OAuth, config shape, any existing `mcp` CLI commands.
- Plugins, custom tools, skills, commands: formats and load paths.
- Session storage and any logging.
- Network egress: every place the code contacts a remote host (model registry fetches, update checks, share links, analytics, hosted services). List each one.

Then make and record these decisions, asking me if unclear:

1. **Which upstream line to fork.** Upstream's permission config differs between the current and the V2 lines (older `permission`/`bash`/`task` keys versus a V2 ordered `permissions` rule array with `action`, `resource`, `effect`). Pick the line, branch, or tag to fork and explain why. Prefer a tagged stable release over a moving branch.
2. **Scope cut.** Decide which packages to keep for a CLI-focused product and which to drop or disable (desktop app, web/marketing site, hosted services, editor SDKs, cloud infrastructure, unrelated directories). Dropping code is allowed; leaving dead code that phones home is not.
3. **Upstream sync strategy.** Upstream moves fast. Keep ZeroCode changes in separate packages and plugins where possible, keep a list of core patches, and document in `docs/upstream-sync.md` how to merge upstream (rebase or cherry-pick cadence).

Deliver the file list: what to rename, modify, add, and remove.

**Phase 0 exit gates** (added after the first Phase 0 report). Do not start M1 until all of these hold:

- Baseline evidence comes from the exact commit that will be forked (the chosen tag), not from a moving branch. Run frozen install, lint, typecheck, CLI build, the core tests, and the CLI tests as separate commands, never chained with `&&`.
- Every failing baseline test is classified as environment (for example missing git identity or default branch), upstream bug, or flaky, with evidence. Failures in VCS, snapshot, shell-parse, and MCP areas must be understood before M5 or M7.
- The scope cut is checked against the real build. If the CLI build bundles the web app or other UI packages, record that, then either keep them as build-only inputs or change the build first.
- Config-discovery audit: list every folder the CLI reads from other tools (such as `.claude` and `.agents`) and make each one off by default or opt-in.
- The egress audit covers every CLI command (including `pair`, `acp`, `api`, `service`, and `upgrade`), not only what static search finds.
- Record whether Docker or Podman is available. If not, M10 uses the mock runtime interface.
- I have decided the rename policy (see M1).

## 4. Milestones

Each milestone ends with: build passes, lint passes, tests pass, new behavior demonstrated with real command output, docs updated, and a short report listing what was verified and what was not.

### M1 — Rename, branding, license

- Rename package, binary (`zerocode`), config paths, env vars, and UI strings. Keep a migration path: if `.opencode/` or `opencode.json` exists and no ZeroCode config does, offer a one-time, explicit import. Never silently read or modify another tool's config.
- Add `NOTICE`, `THIRD_PARTY_NOTICES.md`, and updated `LICENSE` headers.
- Remove upstream branding and assets.
- `zerocode` launches the TUI; `zerocode --version` works.
- Rename policy (default, to be confirmed at the Phase 0 review): change everything users see or depend on (binary, published package name, env vars, config paths, UI text, URLs, docs). Keep internal workspace package scopes such as `@opencode/*` so upstream merges stay cheap, and list every remaining internal `opencode` identifier in `docs/upstream-sync.md`. Check in an allowlist of the places the old name may still appear, and fail CI on any other occurrence.
- Config roots belonging to other tools (`.claude`, `.agents`) are not read by default.

### M2 — Privacy and egress lockdown

- From the Phase 0 egress list: disable or remove sharing, analytics, telemetry, auto-update checks, and any call to upstream-hosted services. Nothing is sent anywhere by default.
- If a remote call is genuinely needed (for example a model catalog), bundle a local snapshot and make any refresh opt-in and documented.
- Add a test that fails if new default network calls appear.

### M3 — Provider layer and local models

Extend the existing provider system; do not rewrite it.

- Ensure support for: OpenAI-compatible APIs, Anthropic, Google Gemini, OpenRouter, Azure OpenAI, AWS Bedrock, Ollama, LM Studio, llama.cpp server, vLLM, and any custom OpenAI-compatible endpoint. Reuse what upstream already provides and add presets for the rest.
- BYOK via config, environment variables, and `.env`. Store credentials with owner-only file permissions (or the OS keychain if upstream supports it).
- Model routing by agent, fallback model chains, streaming, tool calling.
- Token usage and cost tracking per session, with configurable cost and rate limits that stop the agent when exceeded.
- Secret redaction in all logs and audit output. No API keys or repository contents in logs by default.
- `local` preset works with no cloud account. Handle local models that lack native tool calling gracefully (clear error or a documented fallback).
- Open-weight models use different tool-call formats (for example Qwen-Coder). Add per-model parsers or normalizers with recorded-response fixtures, and tests for malformed tool calls.

### M3.5 — Permission system

Extend upstream's rule engine; do not build a second one.

- Gate these actions: file write, file delete, shell command, package install, network access, git push, database migration, Docker command, deployment command, access outside the project directory.
- Permission modes as presets over the rule engine: `ask` (default), `auto-safe` (allow reads and an allowlist of safe commands, ask for everything else), `yolo`.
- `yolo` is never the default, requires an explicit flag every run, and refuses to start unless a sandbox is active.
- A hard-deny list no mode can override: force push, history rewrite, deleting branches, pushing to `main`/`master`/protected branches, deploying to production, reading secret files without approval.
- Path traversal and symlink-escape protection on every file operation; filesystem boundary at the project root unless explicitly extended.
- Clear prompts showing exact command, working directory, and reason.
- Session audit log (append-only JSONL, redacted) recording every important action and decision.

### M4 — Core agents: `coder`, `ask`, `planner`, `debugger`, `reviewer`

- Map onto upstream: `build` becomes `coder`, `plan` becomes `planner`. Define new agents using upstream's agent mechanism (markdown or config files, shipped as built-ins) rather than new code paths.
- `ask`: read-only Q&A. `debugger`: reproduce, isolate, fix, verify. `reviewer`: read-only review with structured findings.
- Each agent has an explicit tool and permission profile.
- CLI: `zerocode coder|plan|debug|review|ask [prompt]` as thin wrappers that select the agent and its defaults. Non-interactive mode for CI (`-p`/`run` style, JSON output option, exit codes).

### M5 — Git workflow and `git-coder`

Implement as a separate package.

- Repository map: file tree plus symbol outline (a parser such as tree-sitter is fine), ranked and size-limited for the model context. Clean-room design.
- Detect language, framework, package manager, build system, and test framework.
- Focused minimal diffs, diff review before applying, test-aware editing (run relevant tests after changes).
- Conventional commits, branch creation, staging, committing, PR description generation, and git-history awareness.
- Safe Git wrapper: a typed API over the `git` binary that cannot express force-push, branch deletion, history rewrite, or push without an approval token.
- `zerocode git` launches the `git-coder` agent.

### M6 — Workflow engine

Plan → approve → edit → run → verify → report, as a state machine.

- Plan step produces a structured plan the user can edit or reject.
- Edits happen only after approval; commands go through the permission engine; verification runs tests, type checks, and lint.
- Failure handling: bounded retries, then stop and report.
- Final report lists changes, commands run, test results, and anything unverified.
- Unit-test the state machine with a mock model provider.

### M7 — MCP and `automate`

Extend upstream's MCP client.

- Transports: stdio, HTTP, SSE, streamable HTTP, and OAuth. Verify which exist already and add only what is missing.
- CLI: `zerocode mcp add <name> [options] -- <command...>`, `mcp add --url <url>`, `mcp list`, `mcp remove <name>`, `mcp auth <name>` (reuse any existing upstream `mcp` commands).
- Templates for: GitHub, GitLab, Filesystem, Browser, Playwright, PostgreSQL, SQLite, Slack, Linear, Jira, Sentry, Docker, Kubernetes, and CI/CD. Templates are data files with placeholders for secrets; they never contain real tokens. Verify each package name before shipping a template.
- MCP tool output is untrusted input (see Security). Per-server tool allowlists and per-tool permission rules.
- `automate` agent plus saved, reusable automation workflows (prompt + tools + permissions) runnable with `zerocode automate <name>`.
- Follow the open standards: MCP and `AGENTS.md` are stewarded by the Linux Foundation's Agentic AI Foundation, which also hosts goose. Keep `AGENTS.md` as the rules file and track MCP spec changes.
- Saved automations (goose calls them recipes) are plain files: prompt, tools, permissions, parameters. Optional later item: Agent Client Protocol (ACP) support for editor integration, only if upstream lacks it.

### M8 — Plugins, skills, custom agents

- A user can add an agent by dropping a file in `.zerocode/agents/`, a command in `commands/`, a skill in `skills/`, a custom tool in `tools/`, or a plugin in `plugins/`, with no core code changes.
- Project-level config is untrusted by default. A repository can define plugins, MCP servers, and commands that execute code, so ZeroCode must show what a project's config will enable and require a one-time trust approval per repository (stored by repo path and config hash; re-prompt when the config changes).
- Document formats with working examples.
- Skills use the Agent Skills format (`SKILL.md` with frontmatter), which Pi also follows. Verify that upstream's skill format matches before adding anything.
- Optional compatibility: read rules files from other tools (such as `CLAUDE.md` or `QWEN.md`) only when the user turns it on, and treat them as untrusted project content.

### M9 — `orchestrator`

- Splits a task into subtasks with explicit dependencies, assigns each to an agent, runs independent subtasks in parallel only when they cannot conflict (read-only work, or separate git worktrees), and merges results.
- Reuse upstream's subagent mechanism. Cap depth, parallelism, tokens, and cost.
- Produces a final report with per-subtask status, diffs, and test results.

### M10 — `autonomous` (sandboxed)

- Runs in a Docker or Podman container, or an isolated git worktree when no container runtime exists (with a clear warning about weaker isolation).
- Flow: create plan, confirm the plan (unless I started it with an explicit approved non-interactive flag), implement incrementally, run tests after meaningful changes, keep a task journal, recover from failures.
- Mandatory budgets: wall-clock timeout, token and cost cap, maximum steps, maximum consecutive failures. No unlimited mode.
- Stops safely on repeated failures, permission walls, or ambiguous requirements, and says why.
- Never: push to `main`, deploy to production, access secrets without approval, touch files outside the workspace, or run without budgets.
- Outputs a reviewable diff or a pull-request branch plus a final report.
- Network inside the sandbox is off by default; allowlist explicitly.
- If Docker/Podman is unavailable in the dev environment, build against a mock runtime interface, test that, and mark real-container behavior as unverified.
- Sandbox backends sit behind one interface: Docker/Podman first, with optional OS-level isolation (bubblewrap on Linux) as a lighter option. Kilo's CLI bundles bubblewrap, which is LGPL-2, so any bundled sandbox binary needs a license check and an entry in `THIRD_PARTY_NOTICES.md`.

### M11 — Quality, packaging, release

- Tests: unit tests for core logic (permissions, git wrapper, workflow state machine, redaction, path protection); integration tests for each CLI command using mock providers and temp git repos; a security test suite (traversal, symlink escape, prompt-injection fixtures, secret leakage).
- CI (GitHub Actions): lint, typecheck, test, build, license scan, release.
- Release: semantic versioning, `CHANGELOG.md`, npm publish with provenance, standalone binaries if upstream's build supports it, and a one-line install (`npm i -g zerocode-cli`, plus an install script if upstream has one).
- Docker image for running ZeroCode and for the autonomous sandbox.
- `.github` issue and PR templates, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`.
- Tag `v0.1.0` only when the Definition of Done below is met.
- Supply chain: commit lockfiles, install with the frozen lockfile and lifecycle scripts disabled in CI, review any dependency that needs an install script, publish with provenance, and run a scheduled dependency audit (Pi does similar for its npm releases).

## 5. Agents and commands

| Command | Agent | Notes |
|---|---|---|
| `zerocode` | default (`coder`) | Interactive TUI |
| `zerocode coder` | `coder` | Maps to upstream `build` |
| `zerocode plan` | `planner` | Maps to upstream `plan`, read-only |
| `zerocode git` | `git-coder` | Git-first editing and commits |
| `zerocode debug` | `debugger` | Reproduce, fix, verify |
| `zerocode review` | `reviewer` | Read-only review |
| `zerocode autonomous` | `autonomous` | Sandboxed, budgeted |
| `zerocode automate` | `automate` | MCP-first workflows |
| `zerocode local` | `local` | Local-model defaults |
| `zerocode ask` | `ask` | Read-only Q&A |
| `zerocode orchestrator` | `orchestrator` | Coordinates the others |

Plus `zerocode mcp ...`, and non-interactive prompt mode for CI and scripts.

## 6. Inspiration map (ideas only, clean-room)

- Aider: git-aware editing, repository map, focused diffs, test-aware edits, auto-commits.
- Cline: plan, approve, edit, run, verify, and permission controls.
- Kilo Code: Architect/Code/Debug/Ask/Orchestrator style modes and orchestration.
- OpenHands: autonomous, sandboxed, long-running tasks.
- Goose: MCP-first extensibility and reusable automations.
- Pi: lightweight, local-first, BYOK.
- Qwen Code: strong open-weight and local model support (Ollama, vLLM).

License check, from public sources on Nov 15, 2026. Re-read each repo's `LICENSE` before relying on any row.

| Project | Current repo | License | What ZeroCode may do |
| --- | --- | --- | --- |
| OpenCode (base) | `anomalyco/opencode` | MIT | Fork; keep notices |
| OpenCode Go line | `opencode-ai/opencode` | MIT, archived Sep 2025 | Not the base; development moved to Crush |
| Crush | `charmbracelet/crush` | FSL-1.1-MIT: source-available, blocks competing commercial use, converts to MIT after two years | Do not fork or copy |
| Aider | `Aider-AI/aider` | Apache-2.0 | Ideas only; development slowed (last release v0.86.x, Aug 2025) |
| Cline | `cline/cline` | Apache-2.0 | Ideas only; its hosted service has separate terms, so use none of it |
| Kilo Code | `Kilo-Org/kilocode` | Apache-2.0 core; new Kilo CLI reported as MIT; bundled third-party parts vary, including LGPL-2 | Ideas only; check each package |
| OpenHands | `OpenHands/OpenHands` | MIT, except `enterprise/` (source-available, commercial license) | Ideas only; never copy from `enterprise/` |
| Goose | `aaif-goose/goose` (moved from `block/goose`) | Apache-2.0 | Ideas only |
| Pi | `earendil-works/pi` (pi-mono) | MIT | Ideas only |
| Qwen Code | `QwenLM/qwen-code` | Apache-2.0 | Ideas only; began as a Gemini CLI fork |

- If code from any MIT or Apache-2.0 project is ever reused, keep its notices and list it in `THIRD_PARTY_NOTICES.md`. Never reuse code from FSL, source-available, or enterprise-only directories.
- Use no project's names, logos, hosted services, or terms of service.
- Repo names and homes change (goose and Pi both moved). Open the repo's own `LICENSE` and README before citing it in docs.

Docs must credit the inspiration for each feature without implying endorsement or copied code.

## 7. Security requirements

- Treat repository files, terminal output, issue text, web content, and MCP tool output as untrusted data, never as instructions. Wrap such content in clearly delimited blocks, tell the model it is data, and never let it change permissions, approve actions, or alter system prompts.
- Tool-call confirmation through the permission engine; no tool bypasses it.
- Command sandboxing for autonomous mode and `yolo`; filesystem boundary; path traversal and symlink protection.
- Secret detection and redaction in prompts sent to models where feasible, logs, audit trails, and error messages. Block reading of common secret files (`.env`, key files) without approval.
- No telemetry, analytics, or automatic uploads by default. No update pings by default.
- No automatic destructive Git operations. No automatic package installation without approval.
- Rate limits and cost limits. Session audit logs.
- Trust prompt for project-level config (see M8).
- Safe defaults everywhere; the least-privileged option is the default.

## 8. Documentation to create or rewrite

`README.md`, `docs/architecture.md`, `docs/configuration.md`, `docs/agents.md`, `docs/mcp.md`, `docs/security.md`, `docs/contributing.md`, `docs/roadmap.md`, `docs/upstream-sync.md`, `docs/migration-from-opencode.md`, `NOTICE`, `THIRD_PARTY_NOTICES.md`, `CHANGELOG.md`.

Docs must explain: ZeroCode is based on OpenCode and is not affiliated with it; which upstream capabilities were retained; which features are new and what inspired each; how to configure cloud and local models; how permissions and sandboxing work; how to add MCP tools; how custom agents, skills, and plugins work; and what is verified versus experimental in the current release.

## 9. Engineering standards

- Strict TypeScript, modular packages, interfaces at module boundaries (providers, tools, agents, UI, sandbox runtime, git).
- Follow upstream's existing lint, formatting, naming, test, and monorepo conventions.
- Make external integrations mockable: model providers, git, Docker/Podman, MCP servers, filesystem.
- Avoid monolithic files; keep new features in separate packages or plugins.
- Every new dependency is justified and license-checked.

## 10. Definition of done (v0.1.0)

A developer can, on a clean machine:

1. Install `zerocode` with one command.
2. Run `zerocode` in any Git repository.
3. Choose a cloud or local model.
4. Ask it to implement a feature.
5. Review the plan and diffs.
6. Approve safe commands.
7. Run tests.
8. Create a clean commit.
9. Use MCP tools.
10. Add a custom agent without changing core code.
11. Run an autonomous task in a sandbox.
12. Inspect an audit log of every important action.

And, with evidence: build, lint, tests, and license scan pass in CI; no default network egress beyond the user's chosen model provider; every claim in the README matches a verified behavior.

## 11. First deliverables

Start with Phase 0 only, then stop for my review:

1. The architecture map (`docs/architecture.md`).
2. The recorded decisions: upstream line, scope cut, sync strategy.
3. The fork and license-compatibility plan.
4. The rename/modify/add/remove file list.
5. Whether the project builds and tests pass on the untouched fork, with actual output.

Only after I approve do you begin M1, and continue one milestone at a time.
