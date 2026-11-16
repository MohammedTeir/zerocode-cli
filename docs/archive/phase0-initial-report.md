Superseded by docs/phase0-gates.md. Evidence is from dev at d432a06b9c, not the v2.3.0 baseline.

# ZeroCode CLI — Phase 0 Architecture and Fork Plan

**Status:** Phase 0 inspection complete; no source-code changes made. The report was refreshed after the fork synchronized with upstream on 2026-11-16.

**Inspected repository:** `MohammedTeir/zerocode-cli`

**Inspected branch and commit:** `dev` at `d432a06b9c07bab0be4e8d642173b1cb14137f11` (`fix(core): preserve UTF-8 across shell output pages (#63984)`)

**Remotes:**

```text
origin   https://github.com/MohammedTeir/zerocode-cli.git
upstream https://github.com/anomalyco/opencode.git
```

**Post-report synchronization state:** `origin/dev`, `upstream/dev`, and the local `dev` checkout now all point to `0354362b857beae78166d4305f06c0b2f47c251e` (`chore(site): show New on Hy4 flash Free instead of Hy4 in the Go chart (#64178)`). This synchronized tip is 14 commits ahead of the originally inspected checkout at `d432a06b9c`. The local checkout was fast-forwarded with `git merge --ff-only origin/dev` while preserving the untracked Phase 0 documentation; the verification results below still remain evidence for the original checkout until the synchronized tip is retested.

The repository is currently a renamed GitHub fork, but its contents are still functionally upstream OpenCode. The rename has not yet been applied to package metadata, source identifiers, configuration paths, environment variables, hosted-service references, updater logic, or visible UI text.

The synchronization did not introduce ZeroCode-specific changes. The 14 upstream commits changed AI media/generation middleware and tests, core SQLite storage-failure handling and recovery, Claude Agent context handling, subagent lifecycle reporting, server defect responses, and mostly application/site/desktop/UI surfaces. No root or CLI package metadata, lockfile, license, configuration-root, branding, updater, MCP OAuth, or permission-policy change was found in the synchronized commit range. The architecture, scope-cut, license, and egress recommendations below remain applicable, but the synchronized tip needs a fresh build/lint/typecheck/test run before it can replace the original evidence baseline.

## 1. Executive summary

The repository is a large TypeScript/Bun monorepo. Its strongest reusable foundation for ZeroCode is the terminal CLI/TUI, local server, provider/model layer, permissions, MCP client, plugin system, skills, commands, sessions, and Git/VCS abstractions. The repository also contains a desktop application, web application, marketing/docs site, hosted Cloudflare services, release infrastructure, and several SDK/support packages that should not be part of the first CLI-focused ZeroCode release.

The recommended approach is to freeze a stable upstream baseline, retain the `upstream` remote, and implement ZeroCode in a separate milestone branch series. Core patches should be minimized and recorded in `docs/upstream-sync.md` once M1 begins. New behavior should prefer built-in agents, commands, plugins, skills, custom tools, MCP configuration, and the existing permission engine over direct edits to core execution paths.

## 2. Proposed Phase 0 decisions

### 2.1 Upstream line

**Recommendation: use the tagged upstream release `v2.3.0` as the initial ZeroCode baseline, not the moving `dev` branch.**

Evidence from the inspection:

- The fork currently tracks the moving `dev` branch.
- The repository contains stable tags through `v2.3.0`.
- The current `dev` commit is `d432a06b9c`, dated 2026-11-14, and is not itself a stable release tag.
- The specification explicitly prefers a tagged stable release over a moving branch.

Before M1, create a ZeroCode baseline branch from the exact upstream `v2.3.0` tree, preserve the current `dev` snapshot for comparison if desired, and record the chosen commit hash in the milestone report. If the user prefers to retain newer `dev` functionality, that is a conscious exception and must be documented as a moving-baseline risk.

### 2.2 Scope cut

**Recommendation: make the first product CLI/TUI-focused.**

Keep the dependency closure needed for:

- `packages/cli` — executable, command dispatch, non-interactive mode, service lifecycle, auth, MCP and plugin commands.
- `packages/tui` — terminal interface.
- `packages/server` — local HTTP/RPC server.
- `packages/core` — sessions, agents, tools, permissions, providers, MCP, config, persistence, Git/VCS and workflows.
- `packages/client`, `packages/protocol`, `packages/schema` — typed client/server contracts.
- `packages/plugin`, `packages/plugin-browser` — extension APIs and browser-related plugin support where explicitly retained.
- `packages/ai` and `packages/codemode` — model/provider and code-mode support required by retained core functionality.
- `packages/util`, `packages/theme`, `packages/latex`, `packages/merman`, and other direct TUI dependency packages.
- `packages/simulation` for provider and integration tests, if it remains part of the verification closure.
- `packages/sdk` only if the public SDK is intentionally supported in the first release; otherwise defer it rather than silently shipping an unbranded public API.

Defer or remove from the first CLI release, after dependency-closure confirmation:

- `packages/app`, `packages/desktop`, `packages/stories`, `packages/gui-plugins`, `packages/session-ui`, and browser/application-only UI packages not required by the TUI.
- `services/site`, `services/data`, `services/files`, `services/legacy`, and `services/update`.
- Cloudflare deployment targets, hosted sharing, product marketing/legal pages, upstream release publishing infrastructure, and desktop-specific signing/update assets.

Do not delete these areas during Phase 0. M1 should make the scope cut explicit and verify that removed services cannot be imported or contacted by the CLI. Any retained desktop/web code must be treated as out of product scope and must not remain as an undiscovered source of default network egress.

### 2.3 Upstream sync strategy

Use a fork-first, milestone-branch workflow:

1. Keep `upstream` pointing to `anomalyco/opencode` and `origin` pointing to `MohammedTeir/zerocode-cli`.
2. Create one branch per milestone, based on a reviewed ZeroCode baseline branch.
3. Keep branding, privacy, agent definitions, commands, templates, and most new behavior in separate packages/configuration/plugins.
4. Maintain a patch inventory in `docs/upstream-sync.md` for every direct core modification, with reason, upstream file, compatibility risk, and an expected rebase strategy.
5. Rebase or merge upstream only at explicitly reviewed sync points; never silently mix upstream changes into a milestone branch.
6. Use small conventional commits and do not push without explicit approval.

Recommended initial cadence: sync upstream only after each completed milestone or when a security fix is required. Re-run the full Phase 0 egress, branding, license, and regression checks after every sync.

## 3. Monorepo and package map

The root workspace uses Bun 1.4.4 and Turbo 2.10.2. The workspace includes `packages/*`, `packages/plugins/*`, `services/*`, and `packages/data/*`. TypeScript uses the Bun base configuration. Root lint uses oxlint, ast-grep, SDK documentation checks, and UI grey-scale checks. Typecheck uses Turbo with 35 package tasks in the inspected run. The root `test` script intentionally fails with `do not run tests from root`; tests are run per package or through CI orchestration.

### Core runtime packages

| Package | Observed role | ZeroCode disposition |
| --- | --- | --- |
| `@opencode/cli` | `opencode`/`opencode2` binaries, command dispatch, TUI startup, server connection, non-interactive `run`, auth, MCP, plugins, services and updater | Keep; rename to ZeroCode |
| `@opencode/tui` | Terminal UI | Keep; rename visible text and imports |
| `@opencode/server` | Local server and HTTP/RPC server-side entry points | Keep |
| `@opencode/core` | Agents, sessions, tools, permissions, config, providers, MCP, persistence, Git/VCS, worktrees and execution | Keep; minimize core patches |
| `@opencode/client` | Typed Promise/Effect client and RPC/service APIs | Keep |
| `@opencode/protocol` | Generated and hand-authored protocol contracts | Keep |
| `@opencode/schema` | Shared schemas for config, permissions, providers, MCP, sessions and events | Keep |
| `@opencode/ai` | AI SDK/provider support and fixtures | Keep if required by core |
| `@opencode/codemode` | Code-mode support | Keep if enabled in the CLI scope |
| `@opencode/plugin` | Plugin API, runtime, hooks, agents, commands, skills, providers and VCS extension interfaces | Keep; primary extension point |
| `@opencode/plugin-browser` | Browser plugin support | Keep only if browser MCP/plugin support is in v0.1.0 |
| `@opencode/util` | Shared Effect/runtime/process/path/utilities and observability | Keep; audit observability |
| `@opencode/theme` | Shared themes | Keep if TUI requires it |
| `@opencode/latex`, `@opencode/merman` | TUI rendering and diagrams | Keep if required by TUI; otherwise defer |
| `@opencode/simulation` | Simulated provider and integration support | Keep for tests, not necessarily public package |

### Application and release packages

`@opencode/app`, `@opencode/desktop`, `@opencode/gui-plugins`, `@opencode/session-ui`, `@opencode/stories`, and the web/UI-only packages support the graphical/web product. They should be excluded from the first CLI-focused release unless dependency inspection proves that the TUI requires a specific shared package.

`@opencode/sdk`, `@opencode/http-recorder`, `@opencode/httpapi-codegen`, `@opencode/script`, `@opencode/data`, and `@opencode/process-daemon` require individual release decisions. `process-daemon` includes native/Rust-adjacent packaging and a bundled Ghostty license area; it must not be copied or redistributed without a separate license audit.

### Services

The repository contains `services/data`, `services/files`, `services/legacy`, `services/site`, and `services/update`. These include hosted domains, Cloudflare workers, release/update APIs, sharing compatibility, marketing/docs content, and deployment configuration. They are not required for a local-first CLI and should be removed from the first product workspace or isolated from the CLI build.

## 4. Runtime entry points

The current executable is declared in `packages/cli/package.json`:

```json
{
  "bin": {
    "opencode": "./bin/opencode.cjs",
    "opencode2": "./bin/opencode2.cjs"
  }
}
```

`packages/cli/src/index.ts` is the Bun entry point. It initializes the plugin runtime, global paths, configuration, updater, observability layer, command handlers, and runtime error handling. The default command launches the TUI. The CLI currently exposes `upgrade`, `uninstall`, `acp`, `api`, `debug`, `auth`, `mcp`, `plugin`, `models`, `stats`, `mini`, `run`, `pair`, `reload`, `session`, `service`, and `serve` commands.

The non-interactive path is implemented by `packages/cli/src/run/noninteractive.ts` and `packages/cli/src/run/run.ts`. It connects to an existing service when possible and consumes session events, permission requests, tool events, subagent activity, and usage information. This is the natural foundation for ZeroCode CI mode and structured output.

The server/client split is represented by `packages/server`, `packages/client`, `packages/protocol`, and `packages/schema`. The TUI uses the typed client to communicate with the local service. The CLI can also start or connect to a standalone/private service.

## 5. Agent system

The current system has a core agent service in `packages/core/src/agent.ts`, agent configuration and loading in `packages/core/src/config/plugin/agent.ts`, and subagent invocation through `packages/core/src/tool/plugin/subagent.ts`. The repository also retains V1 compatibility migration code under `packages/core/src/v1/config/`.

Repository-local examples currently exist under `.opencode/agent/`, `.opencode/command/`, `.opencode/skills/`, and `.opencode/plugins/`. The repository has `AGENTS.md` guidance files. This confirms that ZeroCode can implement `coder`, `planner`, `ask`, `debugger`, `reviewer`, `git-coder`, `automate`, `orchestrator`, `autonomous`, and `local` primarily as built-in agent definitions and commands rather than separate execution paths.

The current upstream built-in/plugin resources still contain OpenCode-specific names and instructions. They must be renamed or rewritten in M1 and audited for unintentional brand references.

## 6. Configuration, paths and migration surface

Configuration discovery is implemented in `packages/core/src/config/discovery.ts`, `packages/core/src/config.ts`, `packages/core/src/config/plugin/*`, and the CLI config modules. Current project config names include `opencode.json` and `opencode.jsonc`; project-local definitions are under `.opencode/`. Current global configuration is under the OpenCode global config path, with project/state/cache/data/log paths supplied by the global path service.

The current discovery order includes direct config files and `.agents`, `.claude`, and `.opencode` roots. Variable substitution is implemented in `packages/core/src/config/variable.ts`. Managed/system configuration is implemented in `packages/core/src/config/managed.ts`. The current system supports provider, agent, command, instruction, plugin, MCP, policy, skill, formatter, shell, websearch, worktree and related configuration plugins.

M1 must change the canonical names to `.zerocode/`, `zerocode.json`/`zerocode.jsonc`, `~/.config/zerocode/`, and `ZEROCODE_`. Migration from `.opencode/` and `opencode.json*` must be explicit, one-time, and opt-in; the new CLI must never silently read or modify those paths.

## 7. Permissions and security enforcement

The existing permission engine is in `packages/core/src/permission.ts`, with schemas in `packages/schema/src/permission*` and persistence in `packages/core/src/permission/saved.ts` and `packages/core/src/permission/sql.ts`. File reads/writes, shell commands, web fetches, MCP tools, subagents and skills call the permission service rather than bypassing it.

The current rule engine already supports ordered rule evaluation with actions/resources/effects in the V2 configuration. This should be extended rather than replaced for M3.5. ZeroCode must add coverage and policies for package install, network access, Git push, database migration, Docker/deployment commands, and outside-project paths. The existing hidden `--yolo` and `--dangerously-skip-permissions` flags must not become the default; they require redesign and sandbox enforcement.

Path protection currently has dedicated filesystem/access abstractions and permission checks, but symlink escape and project-boundary behavior must be tested explicitly for ZeroCode. Audit logging is not yet demonstrated as a complete append-only, redacted action ledger and is a required new capability.

## 8. Provider and model layer

Provider support is concentrated under `packages/core/src/plugin/provider/`, with files for OpenAI, OpenRouter, Anthropic-compatible/Claude paths, Azure, Bedrock, Gemini/Vertex-related providers, Ollama, LM Studio, vLLM, local and other integrations. Model discovery and refresh code exists under `packages/core/src/models-dev.ts`, `packages/core/src/model.ts`, provider refresh modules, and the CLI `models` handler.

The current provider layer is a strong extension point for M3. The current code also contains a default remote models catalog source at `https://models.opencode.ai` and provider-specific remote calls. ZeroCode should bundle a reviewed local model snapshot by default and make catalog refresh explicitly opt-in. Credentials are currently handled through integration/auth and provider credential settings; owner-only file permissions, secret redaction, and cost/rate limits need dedicated verification.

## 9. MCP, plugins, skills and commands

MCP is implemented under `packages/core/src/mcp/` and the CLI handlers under `packages/cli/src/commands/handlers/mcp/`. The code supports local stdio servers, remote HTTP/streamable HTTP through the MCP SDK, OAuth flows, server connection lifecycle, prompts, resources, tools, and experimental event streams. The CLI currently supports `mcp list`, `mcp add`, `mcp auth`, and `mcp logout`.

The current MCP OAuth client metadata uses an OpenCode-hosted URL (`https://opencode.ai/oauth/opencode/client.json`) and therefore must be replaced or made local/ZeroCode-owned before M2/M7. MCP output is untrusted input and must remain behind the permission engine, with per-server and per-tool allowlists added in M7.

Plugins are loaded through `packages/core/src/plugin/`, `packages/core/src/config/plugin/`, and `packages/cli` plugin handlers. Skills use discovery and `SKILL.md`-style files under `packages/core/src/skill/` and repository config roots. Commands and agents can be supplied as files. These are the preferred extension points for ZeroCode custom agents, commands, skills and tools.

## 10. Session storage and logging

Session and execution state is implemented in `packages/core/src/session/`, with SQLite/Drizzle persistence under `packages/core/src/database/`. The server exposes session APIs through the client/protocol packages. Logs are emitted through Effect logging and the utility observability layer. The CLI currently configures OpenTelemetry/observability from `OTEL_EXPORTER_OTLP_ENDPOINT` and related environment variables.

M2 must verify that no telemetry or trace export occurs by default. M3.5 must add a separate redacted append-only JSONL audit log for important actions, permission decisions, tool calls, and workflow state transitions. Normal logs must not include API keys or repository contents by default.

## 11. Network egress inventory

The following remote or potentially remote paths were found during static inspection:

- User-selected model/provider endpoints, including OpenAI-compatible, Anthropic, Google, Azure, Bedrock, OpenRouter, Ollama/LM Studio/vLLM/local endpoints, and provider-specific OAuth/auth endpoints.
- The models catalog default at `https://models.opencode.ai`, plus provider/logo fetches in UI build/configuration paths.
- Updater checks in `packages/cli/src/services/updater.ts`, including `https://opencode.ai/update/api/...` and Homebrew formula metadata at `https://formulae.brew.sh/api/formula/opencode.json`.
- MCP remote server URLs and OAuth metadata/token endpoints, including the OpenCode OAuth client metadata URL.
- Web fetch/search integrations and websearch plugins under `packages/core/src/plugin/websearch/`.
- Session sharing/hosted-service compatibility code under the legacy service and related share handlers.
- OpenTelemetry export when `OTEL_EXPORTER_OTLP_ENDPOINT`/headers are configured.
- GitHub API calls in repository maintenance scripts and the GitHub integration itself when explicitly invoked.
- Package registries and release infrastructure during installation/build/release, including npm and GitHub-hosted artifacts.
- Cloudflare-hosted site/data/files/update services and deployment scripts under `services/` and `.github/release/`.

M2 should disable/remove updater, sharing, analytics/telemetry, upstream-hosted service calls, and default catalog refresh. User-selected model providers, explicitly configured MCP servers, explicitly requested web tools, package installation, and GitHub operations remain opt-in network operations governed by permissions.

## 12. License and compatibility plan

The root `LICENSE` is MIT and currently reads `Copyright (c) 2025 opencode`. It must be preserved verbatim, with an additional ZeroCode copyright line added alongside it rather than replacing the upstream notice.

The tree contains additional license/notice surfaces that require review before publishing:

- `packages/http-recorder/LICENSE`
- `packages/process-daemon/LICENSE`
- `packages/process-daemon/src/ghostty/LICENSE`
- `packages/ui/LICENSE`
- `script/oxlint/anti-slop/vendor/eslint-stylistic/LICENSE` and its `UPSTREAM.md`
- Test/spec license files such as `packages/codemode/test/LICENSE.test262` and `LICENSE.wpt`
- `packages/core/src/image/notice.ts`
- `packages/plugins/figma/skills/NOTICE.md`
- Dependency trees under `.github/release/*/package-lock.json`, which visibly include Apache-2.0, LGPL-3.0-or-later, CC0 and other licenses.

The repository has no completed ZeroCode `NOTICE` or generated `THIRD_PARTY_NOTICES.md`. M1 must add both, use an actual dependency-tree license scanner, commit the generation script, and flag non-permissive or mixed-license dependencies. Any code copied from Aider, Cline, Kilo Code, OpenHands, Goose, Pi or Qwen Code is out of scope; ZeroCode should use clean-room reimplementation of ideas only.

## 13. Rename/modify/add/remove inventory

### Rename or rewrite in M1

- Root package name/description/repository metadata in `package.json`.
- All workspace package names and internal `@opencode/*` imports that remain in the retained CLI closure.
- CLI binaries in `packages/cli/package.json` and generated build scripts: `opencode`/`opencode2` to the chosen ZeroCode binary name(s).
- `OPENCODE_*` environment variables to `ZEROCODE_*` in retained source, tests, scripts and documentation.
- `.opencode/` project paths and `opencode.json*` config names to `.zerocode/` and `zerocode.json*`, with explicit migration support.
- Visible CLI/TUI strings, help text, agent instructions, built-in skills/commands, docs and retained assets.
- OpenCode-specific URLs, service names, updater identifiers, package URLs and repository links.

### Modify across M1–M2

- `packages/cli/src/index.ts`, command specifications and handlers.
- `packages/cli/src/config/*`, `packages/cli/src/services/*`, `packages/cli/src/server-process.ts`, and `packages/cli/src/run/*`.
- `packages/core/src/config/*`, `packages/core/src/permission*`, `packages/core/src/mcp/*`, `packages/core/src/plugin/*`, provider modules, model catalog logic, session logging and paths.
- `packages/schema` config/permission/provider/MCP schemas.
- Build/pack scripts and CI workflows that currently hard-code `anomalyco/opencode`, OpenCode package names, domains, release conditions or updater artifacts.
- Retained README/docs and generated SDK/client metadata.

### Add during Phase 0/M1 planning or M1 implementation

- `NOTICE`.
- `THIRD_PARTY_NOTICES.md`.
- A committed dependency license generation script, likely under `script/`.
- `docs/architecture.md` (this document).
- `docs/upstream-sync.md`.
- `docs/migration-from-opencode.md`.
- ZeroCode branding/assets and explicit non-affiliation language.
- Tests for one-time explicit config migration and absence of silent OpenCode config reads.

### Defer/remove from the first CLI release after dependency validation

- Hosted services under `services/`.
- Desktop/web/marketing/release-only packages and their OpenCode branding/assets.
- Upstream hosted sharing, updater endpoints, analytics/telemetry defaults, and service deployment manifests.
- Unneeded release signing, Homebrew/Winget/AUR and OpenCode-specific publishing scripts.
- Any vendored or bundled code with incompatible licensing or unclear provenance.

## 14. Untouched-fork verification evidence

Environment and install:

```text
Bun: 1.4.4
Node: v22.23.4
bun install --frozen-lockfile: exit code 0
1594 packages installed [5.91s]
```

The install reported upstream lockfile patch/platform warnings and three blocked postinstall scripts. These were not treated as build failures, but native/postinstall behavior should be rechecked on a clean CI runner.

Lint:

```text
bun run lint: exit code 0
Found 507 warnings and 0 errors.
Finished in 39.9s on 6803 files with 1 rules using 6 threads.
sdk-docs: ... 0 problem(s)
```

Typecheck:

```text
bun run typecheck: exit code 0
No type errors in 3,518 files across 2 projects
No type errors in 2,139 files across 2 projects
No type errors in 1,574 files across 2 projects
Tasks: 35 successful, 35 total
Time: 54.372s
```

CLI build:

```text
bun run --cwd packages/cli build: exit code 0
✓ built in 41.05s
PWA ... precache 2802 entries
building cli-linux-arm64
building cli-linux-x64
building cli-linux-x64-baseline
building cli-linux-arm64-musl
building cli-linux-x64-musl
building cli-linux-x64-baseline-musl
building cli-darwin-arm64
building cli-darwin-x64
building cli-darwin-x64-baseline
building cli-windows-arm64
building cli-windows-x64
building cli-windows-x64-baseline
```

Core test suite in the chained core+CLI command:

```text
bun run --cwd packages/core test && bun run --cwd packages/cli test: overall exit code 1
7129 pass
350 skip
15 fail
96497 expect() calls
Ran 7494 tests across 322 files. [306.38s]
```

Because the core command failed, the `&&` chain did not start `bun run --cwd packages/cli test`. The CLI test suite therefore remains unverified and must be run separately before the baseline is considered fully characterized.

The failing baseline tests were concentrated in MCP session-expiry retry, session diff, shell parsing, snapshot diff, and VCS diff/status behavior. The captured failure names included:

- MCP reconnect/retry after session expiry.
- `Session.diff` busy-period and later-turn range handling.
- `ShellParse` static directory targets.
- `Snapshot` long-selection restore/diff.
- VCS untracked-file batching.
- VCS untracked-folder listing.
- VCS working-copy diff against HEAD.
- VCS diff before the first commit.
- VCS feature-branch diff against the default branch.
- VCS committed/combined/staged/unstaged/untracked change separation.
- VCS locally undone commit visibility.
- VCS unpushed default-branch review resolution.
- VCS nested-path diff with binary/rename/deletion metadata.

The exact test output is retained in the sandbox command capture. These are baseline failures on the originally inspected untouched fork and must not be attributed to ZeroCode changes. They cannot be assumed to still reproduce on the synchronized tip until that tip is checked out and tested. A clean-machine CI run should reproduce or classify them before M1 is approved.

### 14.1 Synchronization impact and rerun requirement

The remote fork and local checkout are now synchronized at `0354362b857b`. Before accepting this tip as the Phase 0 baseline:

1. Preserve or commit the Phase 0 documentation separately; do not discard it during checkout.
2. Re-run frozen install, lint, typecheck, CLI build, the core test suite, and the CLI test suite separately.
3. Record any changed failure count, especially in core storage/recovery, subagent lifecycle, AI media, and server defect-response areas.
4. Re-run the branding/egress search against the synchronized tip; the initial audit remains directionally valid but is not a substitute for a new scan.

## 15. Phase 0 exit criteria and next gate

Phase 0 inspection is complete for the original checkout, and the local repository is now fast-forwarded to the synchronized upstream tip. The fork is located, remotes are correct, the repository architecture and egress surface are mapped, and the original untouched baseline was built and typechecked successfully. Lint passed with warnings. The original representative core test run failed with 15 baseline failures. The synchronized tip is present locally but has not yet been reverified.

**Do not begin M1 until the upstream baseline decision, scope cut, and synchronized-tip verification are reviewed.** Once approved, create the selected baseline branch from the reviewed commit, add `docs/upstream-sync.md`, and begin M1 on its own branch with small conventional commits. No push or force-push is authorized by this report.
