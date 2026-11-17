# ZeroCode CLI — Phase 0 Gates

**Status:** Part A report only. Part B/M1 has not started.

**Branch:** `zerocode/baseline`
**Chosen baseline:** tag `v2.3.0`
**Baseline commit:** `a6b15d33daf6ab7d35569f6d711fdb630b9f04e7`
**Tag subject:** `release: v2.3.0`
**Distance from fetched `upstream/dev`:** 66 commits behind
**Toolchain observed:** Bun 1.4.4, Git 2.43.0

This report supersedes the earlier moving-branch inspection numbers. The live governing files were read from `dev` because the exact release tag predates them; no product source was changed during this Part A run. Raw command logs remain uncommitted under `reports/baseline/`.

## 1. Baseline evidence

The baseline branch was created from the exact release tag. The repository was clean before the branch switch; the only current untracked path is the intentionally uncommitted raw evidence directory `reports/`.

The required commands were run separately, with full output saved as follows:

| Gate | Command | Result | Evidence |
|---|---|---:|---|
| Frozen install | `bun install --frozen-lockfile` | PASS, exit 0 | [`reports/baseline/install.txt`](../reports/baseline/install.txt) |
| Lint | `bun run lint` | PASS, exit 0; 510 warnings, 0 errors | [`reports/baseline/lint.txt`](../reports/baseline/lint.txt) |
| Typecheck | `bun run typecheck` | PASS, exit 0; Turbo 35/35 tasks successful | [`reports/baseline/typecheck.txt`](../reports/baseline/typecheck.txt) |
| CLI build | `bun run --cwd packages/cli build` | PASS, exit 0; Vite/PWA built in 21.97s; 2,801 precache entries; Linux/macOS/Windows variants built | [`reports/baseline/cli-build.txt`](../reports/baseline/cli-build.txt) |
| Core tests | `bun run --cwd packages/core test` | FAIL, exit 1; 7,108 pass, 350 skip, 14 fail, 97,051 expectations across 7,472 tests/322 files | [`reports/baseline/core-test.txt`](../reports/baseline/core-test.txt) |
| CLI tests | `bun run --cwd packages/cli test` | PASS, exit 0; 399 pass, 9 skip, 0 fail, 1,482 expectations across 408 tests/58 files | [`reports/baseline/cli-test.txt`](../reports/baseline/cli-test.txt) |

The exact core test command took 240.22s. The CLI test command took 272.22s. The lint command reported `Found 510 warnings and 0 errors.` and SDK documentation reported `0 problem(s)`.

### Core failure classification

All 14 failures from the untouched baseline were captured and then rerun in the affected files. The first controlled rerun used a temporary Git configuration with identity and `init.defaultBranch=main`, but the sandbox-injected `GIT_CONFIG_COUNT`/`color.ui=always` still forced ANSI output. A second rerun explicitly overrode the injected Git configuration and `CLICOLOR` variables:

```text
GIT_CONFIG_COUNT=0 CLICOLOR=0 CLICOLOR_FORCE=0
```

with temporary global settings for identity, `init.defaultBranch=main`, `color.ui=false`, and `core.quotepath=false`. Its result was:

```text
95 pass, 1 skip, 1 fail, 410 expect() calls
Ran 97 tests across 5 files
```

Evidence: [`reports/baseline/core-failing-rerun-controlled.txt`](../reports/baseline/core-failing-rerun-controlled.txt) and [`reports/baseline/core-failing-rerun-no-color.txt`](../reports/baseline/core-failing-rerun-no-color.txt).

| Failure(s) | Classification | Evidence and decision |
|---|---|---|
| 2 failures in `packages/core/test/git.test.ts` (`Git trees`) | Environment-induced | Failures disappear when Git color is disabled. The sandbox injects `color.ui=always`, `CLICOLOR=1`, `CLICOLOR_FORCE=1`, and `GIT_PAGER=cat`; ANSI sequences enter patch output and break exact patch assertions. Not an upstream product failure demonstrated by the controlled no-color rerun. |
| 9 failures in `packages/core/test/vcs.test.ts` (`Vcs`) | Environment-induced | Same controlled no-color rerun removes all nine failures. The tests compare Git-produced patch/text output and are sensitive to forced ANSI color. |
| 1 failure in `packages/core/test/session-diff.test.ts` (`Session.diff`) | Environment-induced | Disappears with explicit no-color overrides; it consumes the same Git diff path. |
| 1 failure in `packages/core/test/snapshot.test.ts` (`Snapshot`) | Environment-induced | Disappears with explicit no-color overrides; snapshot text was contaminated by forced ANSI output. |
| 1 failure in `packages/core/test/shell-parse.test.ts` (`extracts static directory targets across escapes, end-of-options, literal dollars, and precommands`) | Upstream behavior defect / bug candidate | The remaining failure is the `~root/secret` case: expected `/root/secret`, received the temporary test-home path `/tmp/.../secret`. The source contains named-user expansion logic, but this test still resolves `~root` relative to the temporary runner home. It remains reproducible after Git identity, default branch, and color variables are controlled. It must be fixed or explicitly accepted before the shell-parser gate is considered clean. |

No MCP session-expiry failure appeared in the exact v2.3.0 core test run. No failure was classified as flaky: the 13 Git/session/snapshot/VCS failures are repeatably explained by the forced-color environment, and the shell parser failure remains after that environment is removed.

The Git environment checks were:

```text
git version 2.43.0
user.name=MohammedTeir
user.email=8.0924323e+07+MohammedTeir@users.noreply.github.com
init.defaultBranch: not configured
```

## 2. CLI build dependency closure and scope implications

The CLI build is not CLI/TUI-only. `packages/cli/script/build.ts:13` imports `buildAppArchive`; `packages/cli/script/app-assets.ts:8-21` runs `bun run build` in `packages/app` and embeds the generated `packages/app/dist` into the executable. `packages/cli/script/build.ts:17` also bundles the models snapshot, and the build includes native PTY, filesystem watcher, plugin-runtime, WebSocket, simulation-graph, and OpenTUI plugins.

The following is the full workspace closure rooted at `@opencode/cli`, `@opencode/tui`, and the embedded `@opencode/app`. All rows are required inputs to the current CLI build or TUI closure; “build-only” means not necessarily part of the first CLI/TUI product experience but still required by the current build graph.

| Workspace package | Needed by current CLI build/TUI? | Why / scope note |
|---|---:|---|
| `@opencode/cli` | Yes | Executable entry point and build script. |
| `@opencode/tui` | Yes | Interactive terminal UI, config, editor, mini UI, model preferences. |
| `@opencode/app` | Yes, build-only | Vite/PWA app is built and archived into the CLI binary. |
| `@opencode/core` | Yes | Session, agent, provider, config, Git, tools, permissions, and runtime core used by TUI/server. |
| `@opencode/server` | Yes | Local API/background server launched or contacted by CLI/TUI. |
| `@opencode/client` | Yes | Typed client used by CLI/TUI/app to call the local or explicitly configured server. |
| `@opencode/schema` | Yes | Shared API/config/session/provider schemas. |
| `@opencode/protocol` | Yes | Server, PTY, MCP, pairing, and API protocol definitions. |
| `@opencode/plugin` | Yes | Plugin host/runtime and plugin-facing types. |
| `@opencode/util` | Yes | Global paths, filesystem, process, HTTP, logging, npm, and platform utilities. |
| `@opencode/ai` | Yes, transitive | Provider/model execution layer reached through core/plugin. |
| `@opencode/codemode` | Yes, transitive | Core tool/codemode integration. |
| `@opencode/http-recorder` | Yes, transitive | Core/AI HTTP recording support and test/runtime dependency. It is in the workspace closure even though it is not a user-facing CLI feature. |
| `@opencode/plugin-browser` | Yes, transitive | Plugin/browser integration imported by core and plugin paths. |
| `@opencode/simulation` | Yes, transitive/build-verified | Imported by server/TUI and explicitly checked by the CLI simulation graph build plugin. |
| `@opencode/latex` | Yes, TUI transitive | TUI rendering support. |
| `@opencode/merman` | Yes, TUI transitive | TUI diagram/mermaid rendering support. |
| `@opencode/theme` | Yes, TUI transitive | Shared TUI/UI themes. |
| `@opencode/ui` | Yes, TUI/app transitive | Shared Solid components, styles, icons, and assets. |
| `@opencode/session-ui` | Yes, app build-only | Embedded web app session UI. |
| `@opencode/gui-plugins` | Yes, app build-only | Embedded web app GUI plugin surface. |
| `@opencode/patchmark` | Yes, app/TUI transitive | Session UI diff/patch rendering. |
| `@opencode/httpapi-codegen` | Yes, client transitive | Generated API client input. |
| `@opencode/script` | Yes, CLI build-only | Build helpers, asset/highlight embedding, and script utilities. |

Direct manifest evidence includes `packages/cli/package.json:28-52`, `packages/tui/package.json:51-72`, `packages/app/package.json:56-86`, `packages/session-ui/package.json:72-89`, and `packages/ui/package.json:132-149`. The closure was computed from all workspace dependency edges; it contains 24 workspace packages. A CLI-focused scope cut can remove or isolate unrelated services only after the build is changed and rerun. Removing `app`, `ui`, or `session-ui` now would break the documented CLI build.

## 3. Config-discovery audit

### Roots and filenames actually read

The exact v2.3.0 source discovers these roots and filenames:

| Root/source | Source evidence | Current behavior | Phase 0 recommendation |
|---|---|---|---|
| System config | `packages/core/src/config/discovery.ts:29-31` | `/etc/opencode` on Unix; `%ProgramData%\\opencode` on Windows. On macOS, managed preferences are also considered. | Rename to ZeroCode in M1; retain only if an explicit enterprise/system mode is approved. |
| macOS managed preferences | `packages/core/src/config/discovery.ts:34-38`, `130-135`; `packages/core/src/config.ts:250-267` | Reads `ai.opencode.managed.plist` from `/Library/Managed Preferences`, converting it through `plutil`. | Keep as a separately documented opt-in/managed deployment surface; do not silently inherit another product's managed policy. |
| Global config directory | `packages/util/src/global-roots.ts:4-19` derives XDG roots; `packages/core/src/config/discovery.ts:61-74` uses `global.config` | With default global roots, this is `~/.config/opencode`; it loads `opencode.json` and `opencode.jsonc`. | Rename to `~/.config/zerocode` and `zerocode.json`/`zerocode.jsonc`. |
| Project `.opencode` directories | `packages/core/src/config/discovery.ts:82`, `143-150`; `packages/core/src/config.ts:335-341` | Walks upward from the project location, discovers `.opencode`, and loads both config filenames from each present directory. | Rename to `.zerocode`; preserve explicit migration only. |
| Direct project config files | `packages/core/src/config/discovery.ts:82`, `121-143` | The upward walk also considers direct `opencode.json`/`opencode.jsonc` files. | Rename to `zerocode.json`/`zerocode.jsonc`; do not read legacy files by default. |
| Global `.claude` | `packages/core/src/config/discovery.ts:69-74`, `152-160`; compatibility plugin `packages/core/src/config/plugin/compatibility.ts:41-57` | Reads `~/.claude/skills` and can load compatible skills. | Off by default in ZeroCode; explicit compatibility flag only. |
| Project `.claude` | `packages/core/src/config/discovery.ts:82`, `152-160`; compatibility plugin lines 41-57 | Upward project discovery includes `.claude`; its `skills` directory is scanned. | Off by default; opt-in and show the trust boundary. |
| Global `.agents` | `packages/core/src/config/discovery.ts:69-74`, `161-165`; compatibility plugin lines 41-57 | Reads `~/.agents/skills` and compatible skills. | Off by default; explicit compatibility flag only. |
| Project `.agents` | `packages/core/src/config/discovery.ts:82`, `161-165`; compatibility plugin lines 41-57 | Upward project discovery includes `.agents`; its `skills` directory is scanned. | Off by default; explicit compatibility flag only. |
| `AGENTS.md` global file | `packages/core/src/config/plugin/instruction.ts:36-42`, `51-57`, `78-93` | Reads `<global config>/AGENTS.md` when global discovery is enabled. | Keep `AGENTS.md` as the rules-file convention, but the global root should be ZeroCode's root after rename. |
| `AGENTS.md` project ancestors | `packages/core/src/config/plugin/instruction.ts:51-57`, `84-94` | Walks from the current location to project root and loads ancestor `AGENTS.md` files. Reading a file can also inject nearby instructions through `packages/core/src/tool/plugin/read.ts:129-144`. | Keep; treat project content as untrusted and apply M8 trust controls. |
| Explicit config file | `packages/core/src/config/discovery.ts:138-143`; `packages/core/src/config.ts:285-290` | `options.file` loads a specifically requested path last among ordinary file sources. | Keep as explicit user intent only. |
| CLI state/config | `packages/cli/src/config/config.ts:28-49` | Loads `<global.config>/cli.json`, with content override from `OPENCODE_CLI_CONFIG_CONTENT:32-34`; writes with mode 0600. | Rename to ZeroCode paths/env in M1. |
| CLI config directory override | `packages/cli/src/index.ts:141-147` | `OPENCODE_CONFIG_DIR` replaces the global config root for CLI state/config. | Rename to `ZEROCODE_CONFIG_DIR`; keep explicit override semantics. |

`packages/core/src/config.ts:60-70` documents the discovery controls: `global:false` skips system/global roots plus `~/.claude` and `~/.agents`; `claude:false` skips Claude roots; `project:false` skips the upward project walk. The current default is permissive: `ConfigDiscovery.discover(options)` is called at `packages/core/src/config.ts:380`, and compatibility roots are subsequently watched/reloaded at `407-412`.

### Merge order and substitutions

The load order is explicit in `packages/core/src/config.ts:305-377`:

1. Managed policies from system configuration and macOS preferences, reversed so managed policies have earliest policy authority.
2. Integration/well-known documents, subject to managed policy decisions.
3. Global directory documents.
4. Explicit file documents.
5. Direct project files.
6. Project `.opencode` directory documents, ordered from lower to higher project priority.
7. Inline `content` configuration.
8. Managed settings again at the end so managed settings override ordinary settings.

A config file is read and passed through variable substitution at `packages/core/src/config.ts:173-180`. `{env:VAR}` uses the supplied environment or `process.env` at `packages/core/src/config/variable.ts:25-38`; `{file:path}` resolves relative paths from the config file directory, expands `~/`, reads the file, JSON-escapes its trimmed content, and substitutes empty text on missing/unreadable files at `40-80`.

The compatibility roots are **not** off by default in this baseline. The report therefore recommends a concrete M1 decision: ZeroCode should not automatically read `.claude`, `.agents`, or another product's global roots. They should require an explicit opt-in compatibility/migration action, with a visible list of files and a trust prompt.

## 4. Egress audit

### Method and limitations

Static inspection covered CLI/core/server/util sources for `fetch`, Effect `HttpClient`, WebSocket, updater, OAuth, model catalog, MCP, pairing, sharing, telemetry, GitHub, and hosted-service references. The static audit is not a claim that all remote behavior is disabled: model providers, remote MCP, explicitly configured instruction URLs, and explicitly selected server URLs are expected network features.

For runtime checks, the built artifact `packages/cli/dist/cli-linux-arm64/bin/opencode` was run with temporary XDG/HOME directories and `HTTP_PROXY`, `HTTPS_PROXY`, and `ALL_PROXY` set to `http://127.0.0.1:9`, with `NO_PROXY` cleared. All required top-level commands received a real `--help` probe. Selected non-destructive commands were also attempted.

The source entry now passes. The exact command was `bun run --cwd packages/cli src/index.ts --help`; it exited 0 and printed the real CLI usage, including `opencode <subcommand> [flags] [<directory>]`, the `--standalone`, `--server`, and `--print-logs` flags, and the full subcommand list. The complete output is [`reports/baseline/source-entry-probe.txt`](../reports/baseline/source-entry-probe.txt), ending with `source_entry_exit=0`. The earlier `Cannot find module 'react/jsx-dev-runtime'` error is not reproducible after the workspace dependency/module-linking state was restored; no product source fix was made. The earlier failure remains preserved in [`reports/baseline/egress-command-checks.txt`](../reports/baseline/egress-command-checks.txt) as historical evidence of the transient environment state.

The full built-artifact probe log is [`reports/baseline/egress-built-checks.txt`](../reports/baseline/egress-built-checks.txt). Additional 20-second probes and the PTY capture are documented below and remain raw, uncommitted evidence.

### Command matrix

| Command | Static remote-capable path(s) | Offline probe result / what is and is not verified |
|---|---|---|
| `upgrade` | `packages/cli/src/commands/handlers/upgrade.ts:5,15-35` calls `Updater`; `packages/cli/src/services/updater.ts` contains Homebrew formula and `https://opencode.ai/update/api/...` plus installer URLs. | `opencode upgrade --help` passed offline. The actual upgrade was not run because it changes the installation and is destructive/externally mutating. Remote update checks remain present and must be disabled or made explicit in M2. |
| `uninstall` | `packages/cli/src/commands/handlers/uninstall.ts:8-22,64` uses local service shutdown/updater removal and filesystem cleanup; no direct fetch in the handler. | `--help` passed. Actual uninstall was not run because it removes installation/cache. No claim is made about all updater removal paths beyond static inspection. |
| `acp` | `packages/cli/src/commands/handlers/acp.ts:6,26-33` builds an HTTP client to the local/resolved server; the server can invoke configured model providers and MCP. | `--help` passed. ACP server execution was not started; remote provider/MCP behavior is therefore static-only. |
| `api` | `packages/cli/src/commands/handlers/api.ts:21,37,72` uses `fetch` against the resolved server URL and `/openapi.json`; `--server` permits an explicit remote URL. | `--help` passed. No API request was issued. Explicit remote server use is user-selected, not an autonomous default. |
| `debug` | `debug/agents` and related handlers resolve the server and call client APIs; heap/paths/config are local or local-server operations. | `debug --help` and `debug paths` passed; `debug paths` printed temporary local roots. Other subcommands were not run against a live server. |
| `auth` | `auth/login.ts:65,140,160,177-218,242-321` calls integration/well-known, key, command, and OAuth APIs and can open a browser URL; logout/export/import/switch call credential APIs. | `auth --help` and `auth list` passed. `auth list` printed only `OpenAI OPENAI_API_KEY environment`. Login and OAuth were not run because they intentionally contact providers and may open a browser. |
| `mcp` | `mcp/list.ts:12-14` calls the server; `mcp/auth.ts:24-71` performs OAuth-capable remote MCP authentication; `add` accepts remote URLs; `logout` removes remote credentials. | `mcp --help` passed. `mcp list` timed out (exit 124) under the blackhole proxy, so no successful network isolation result is claimed. Auth/add were not run because they can contact user-specified remote servers or mutate config. |
| `plugin` | `plugin/list.ts:19-26,28-42` calls local server and npm resolution; `add.ts`, `check.ts`, and `update.ts` use the npm service and plugin update APIs. | `plugin --help` passed. **Unknown: command did not finish within 20s; cause not identified.** Install/update/remove were not run because they mutate config/packages. |
| `models` | `models.ts:13-19` calls the server model list; core has provider/model HTTP paths, and `packages/core/src/models-dev.ts:505` names `https://models.opencode.ai`. | `models --help` passed. **Unknown: command did not finish within 20s; cause not identified.** |
| `stats` | `stats.ts:19-32` calls the server location/session stats APIs; data is local unless the resolved server/provider path is remote. | `stats --help` passed. **Unknown: command did not finish within 20s; cause not identified.** |
| `mini` | `mini.ts:4,13-18` resolves a server and launches the minimal UI; model/MCP/instruction features can egress through that server. | `mini --help` passed. Interactive execution was not attempted; no TTY/model credentials were supplied. |
| `run` | `run.ts:4,10` resolves a server; the run then uses model providers, tools, MCP, and configured remote instructions as selected by config. | `run --help` passed. A real model run was not attempted because it would require model credentials and could execute tools. |
| `pair` | `pair.ts:9,28-30,119-131` calls server pairing/info and `RemoteTunnel`; `remote-tunnel.ts:29` constructs `https://${route}.${connection.tunnel.hostname}`. `--remote` explicitly exposes the server through OpenTunnel. | `pair --help` passed. Pairing was not run because it can expose a service or create external links. Remote tunneling is clearly an opt-in egress path but remains present. |
| `reload` | `reload.ts:7-15` resolves the local/resolved server and calls `client.location.reload`. | `reload --help` passed. Actual reload was not run without a server. |
| `session` | List/delete/export call local/resolved server APIs. `session/import.ts:17` accepts a URL and `40` posts to the server; export/import can transfer session data. | `session --help` passed. **Unknown: command did not finish within 20s; cause not identified.** Import/export and delete were not run because they can transfer or mutate data. |
| `service` | Service start/restart/status/stop/get/set/unset are local process/config operations. Starting the service launches the local API; its configured remote/tunnel behavior is a separate server-process concern. | `service --help` and `service status` passed. Status printed `stopped`; no service was started. |
| `serve` | `serve.ts` starts the local HTTP/API/web server. `packages/cli/src/server-process.ts:198-220` can run `RemoteTunnel` and constructs local HTTP URLs. | `serve --help` passed. The server was not started; real listening/remote-tunnel behavior remains unverified. |
| Default `zerocode` / TUI | Default handler launches TUI/server; model providers, MCP, remote instructions, and explicit server URL are the egress-capable paths. Observability is configured in `packages/cli/src/index.ts:150-157` from `OTEL_EXPORTER_OTLP_ENDPOINT`; no endpoint is supplied by default in the probe. | The TUI source/test documentation identifies `Ctrl+C` twice as the exit action (`Press ctrl+c again to exit`). A standalone PTY run rendered the first screen, sent `Ctrl+C` twice, and exited on its own with code 0. Raw capture: [`reports/baseline/tui-pty-clean-quit.txt`](../reports/baseline/tui-pty-clean-quit.txt) and [`reports/baseline/tui-pty-clean-quit.raw`](../reports/baseline/tui-pty-clean-quit.raw). |

Additional static egress surfaces include:

- `packages/core/src/config/plugin/instruction.ts:148-176`: configured `http(s)` instruction URLs are fetched on config reload; this is configuration-directed egress.
- `packages/core/src/mcp/oauth.ts:36,168-169`: MCP OAuth client metadata/branding and callback flow include OpenCode-owned URLs.
- `packages/core/src/tool/plugin/webfetch.ts:18-24`: the web-fetch tool deliberately fetches user/model-requested HTTP(S) URLs.
- `packages/cli/src/services/updater.ts:323-324,542-543`: update/version/install endpoints are OpenCode-hosted or Homebrew-hosted.
- `packages/cli/src/services/remote-tunnel.ts:29`: pairing remote mode constructs OpenTunnel URLs.
- `packages/util/src/observability.ts` and `packages/cli/src/index.ts:150-157`: telemetry/OTLP is environment-configurable. It is not proven absent if a user supplies an OTLP endpoint.
- Provider implementations and `packages/core/src/models-dev/snapshot.txt` contain many provider URLs. These are not contacted merely by displaying help, but they are reachable when the user configures or selects a provider.

### 20-second blackhole probe record

The selected built-binary commands were rerun with temporary HOME/XDG roots, the blackhole proxy configuration, cleared `NO_PROXY`, and `timeout 20s`. Exact results were:

```text
models         exit=124
stats          exit=124
plugin list    exit=124
session list   exit=124
```

The raw record is [`reports/baseline/egress-timeouts-20s.txt`](../reports/baseline/egress-timeouts-20s.txt). The four results are **unknown: command did not finish within 20s; cause not identified.** No conclusion about network isolation or contacted destinations is drawn from these timeouts.

**Phase 0 conclusion:** the audit covers every required command and identifies where egress can occur. It does **not** establish M2 privacy lockdown. Auto-update, hosted OpenCode endpoints, sharing/remote tunnel, telemetry, web fetch, model providers, remote MCP, and remote instruction URLs need explicit policy decisions and M2 tests.

### Remaining unverified items before Part B

- The four 20-second commands remain unknown: each command did not finish within 20s; cause not identified.
- Actual successful or failed model/provider, MCP, OAuth, updater, telemetry, remote-tunnel, web-fetch, and remote-instruction behavior remains policy-dependent and was not exercised beyond the bounded probes and static audit.
- Docker/Podman-backed isolation remains unavailable in this environment.
- The `~root/secret` shell-parser defect remains open and is tracked in [`docs/upstream-sync.md`](upstream-sync.md); it was not fixed in Part A.

## 5. Docker/Podman gate

The checks were run with bounded commands:

```text
--- docker ---
not installed

--- podman ---
command -v podman: exit 1
podman --version: No such file or directory
podman info: No such file or directory
```

No container runtime is installed or usable in this environment. M10 must therefore use the mock runtime interface or a documented weaker isolation fallback until Docker/Podman is available. No runtime was installed as part of Part A.

## 6. Decisions required before M1

1. **Rename policy.** The spec proposes changing all user-visible/dependent surfaces: binary, published package, env vars, config roots/files, UI strings, URLs, and docs, while retaining internal `@opencode/*` workspace scopes to keep upstream synchronization cheap. Please confirm this default or choose a stricter internal-scope rename.
2. **Scope cut.** The current CLI build embeds the PWA and therefore requires the app/session/UI closure listed above. Decide whether to keep those packages as build-only inputs, or approve a build change before removing them. Do not remove them from the tree without changing and re-verifying the build.
3. **Upstream sync strategy.** Confirm a cadence and method (for example, periodic merge/rebase from `upstream/dev` into a clean upstream-sync branch, then replay ZeroCode commits) and require an inventory of any core patches in `docs/upstream-sync.md`.
4. **Baseline acceptance.** Accept the 13 environment-induced failures as sandbox color contamination, with CI/test harness color disabled, or require a source/test fix so they pass even when forced color is injected. Separately decide whether the remaining `~root/secret` shell parser defect blocks M1; the evidence supports treating it as an upstream bug candidate that should be fixed or explicitly tracked.
5. **Compatibility roots.** Confirm `.claude`, `.agents`, and their global counterparts are off by default and only read after an explicit compatibility/migration choice. The current v2.3.0 behavior is permissive and does not meet the ZeroCode requirement.
6. **Egress policy.** Confirm that M2 will disable default updater checks, OpenCode-hosted catalog/auth endpoints, telemetry, sharing/remote tunnel, and other hosted-service calls, while preserving only explicitly selected model-provider, MCP, web-fetch, and remote-server calls.
7. **Container policy.** Confirm that the absence of Docker/Podman is acceptable for the mock runtime path in M10.

## 7. Approved defaults

- Rename: keep internal @opencode/* workspace scopes; rename all user-visible and dependent surfaces.
- Scope cut: keep @opencode/app, @opencode/ui, @opencode/session-ui, and @opencode/gui-plugins as build inputs for M1.
- Baseline: accept the 13 color-contaminated failures with color disabled in the test harness. Track the shell-parser `~root/secret` defect in docs/upstream-sync.md as an open upstream defect that does not block M1.
- Compatibility roots: .claude and .agents are off by default; explicit opt-in only.
- Egress: M2 disables updater checks, hosted catalog/auth endpoints, telemetry, and sharing/remote tunnel by default. Startup egress to models.opencode.ai and GitHub is the first M2 target. Explicit user-selected model-provider, MCP, web-fetch, and remote-server calls are preserved.
- Containers: the mock runtime path is accepted for M10 until Docker or Podman is available.

## 8. Part A boundary

This document is the requested Part A deliverable. No Part B/M1 rename, branding, license, product-source, NOTICE, or migration work was started. No push or force-push was performed. The working tree should be reviewed before any next milestone is approved.
