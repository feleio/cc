# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a source snapshot of **Claude Code** — Anthropic's agentic CLI for terminal-based software engineering. The snapshot was exposed on 2026-03-31 via a `.map` file in the npm distribution and is archived here for educational and defensive security research. The original source belongs to Anthropic.

**Tech stack**: TypeScript (strict) · Bun runtime · React + Ink (terminal UI) · Commander.js (CLI parsing) · Zod v4 (schema validation) · ripgrep · Anthropic SDK · MCP SDK · GrowthBook (feature flags)

## Build & Tooling

No `package.json` or `bunfig.toml` are present in this snapshot — only the `src/` tree was captured. The build system uses:

- **Bun** as the runtime and bundler
- **Biome** for linting (evident from `// biome-ignore-all` comments throughout)
- **Bun feature flags** (`bun:bundle`'s `feature()`) for dead-code elimination at build time
- TypeScript strict mode with a custom ESLint rule `custom-rules/no-top-level-side-effects`

Active feature flags (stripped when false): `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`

## Code Layout

The root of the repo *is* `src/` — there's no wrapping directory. Key entry points:

| File | Role |
|---|---|
| `main.tsx` | Commander.js CLI parser + React/Ink renderer init; fires parallel prefetches (`startMdmRawRead()`, `startKeychainPrefetch()`) before heavy imports |
| `entrypoints/cli.tsx` | CLI entrypoint handler |
| `QueryEngine.ts` | Core LLM engine — streaming, tool-call loops, thinking mode, retries, token counting |
| `query.ts` | Query pipeline |
| `Tool.ts` | Base types and interfaces shared by all tools |
| `tools.ts` / `commands.ts` | Tool registry and command registry respectively |
| `context.ts` | System/user context collection for each request |

## Architecture

### Tool System (`tools/`)

Every capability Claude invokes is a self-contained tool module: input Zod schema, permission model, and execution logic. ~42 tools including `BashTool`, `FileReadTool`, `FileWriteTool`, `FileEditTool`, `GlobTool`, `GrepTool`, `WebFetchTool`, `AgentTool`, `SkillTool`, `MCPTool`, `TaskCreateTool`, `EnterPlanModeTool`, `EnterWorktreeTool`, `CronCreateTool`, etc.

### Command System (`commands/`)

~85 user-facing slash commands (`/commit`, `/review`, `/compact`, `/memory`, `/mcp`, etc.) that map to tool invocations and UI workflows.

### Permission System (`hooks/toolPermission/`)

Invoked on every tool call. Resolves to prompt-user, allow, or deny based on the configured mode: `default`, `plan`, `bypassPermissions`, or `auto`.

### Service Layer (`services/`)

| Directory | Purpose |
|---|---|
| `api/` | Anthropic API client, file API |
| `mcp/` | MCP server connection management |
| `oauth/` | OAuth 2.0 auth flow |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | GrowthBook feature flags |
| `compact/` | Conversation context compression |
| `extractMemories/` | Auto memory extraction from conversations |

### Bridge System (`bridge/`)

Bidirectional protocol connecting IDE extensions (VS Code, JetBrains) to the CLI. Uses JWT auth (`jwtUtils.ts`), a message protocol (`bridgeMessaging.ts`), and a session runner (`sessionRunner.ts`).

### State (`state/`)

`AppState.tsx` / `AppStateStore.ts` — React-context-based global state with subscriber notifications for settings/skills changes.

### Key Feature-Gated Subsystems

- `coordinator/` — multi-agent orchestration (PROACTIVE flag)
- `assistant/` — Kairos assistant mode (KAIROS flag)
- `voice/` — voice input (VOICE_MODE flag)
- `server/` — server/daemon mode (DAEMON flag)

### Notable Patterns

**Parallel startup prefetch** — MDM settings, keychain reads, and GrowthBook init are fired as top-level side effects in `main.tsx` before module evaluation, reducing perceived startup time.

**Lazy loading** — OpenTelemetry, gRPC, analytics, and feature-gated subsystems use dynamic `import()` / `require()` to defer cost until needed.

**Agent swarms** — `AgentTool` spawns sub-agents; `coordinator/` orchestrates multi-agent work; `TeamCreateTool` enables parallel team agents with color-coded terminal output.

**Skills** (`skills/`) — reusable user-defined or built-in workflows executed via `SkillTool`.

**Plugins** (`plugins/`) — first-party and third-party plugin loading.
