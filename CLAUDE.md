# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the Claude Code CLI source — a large TypeScript/React terminal application (~1,884 `.ts`/`.tsx` files) built on **Bun** and **Ink** (terminal UI framework). It is the source for the `claude` CLI that users interact with.

## Build & Runtime

- **Runtime**: Bun (not Node.js). Build features use `bun:bundle` for feature-gated tree-shaking.
- **Feature gates**: `import { feature } from 'bun:bundle'` gates code for `KAIROS`, `VOICE_MODE`, `PROACTIVE`, `BRIDGE_MODE`, `COORDINATOR_MODE`. These compile away unused code.
- **Internal-only features**: `process.env.USER_TYPE === 'ant'` gates Anthropic-internal functionality.

## Code Architecture

### Entry & Startup (`main.tsx` → `setup.ts`)

`main.tsx` must perform three side-effects **before** other imports for startup perf:
1. `profileCheckpoint('main_tsx_entry')` — startup profiler
2. `startMdmRawRead()` — fires MDM reads in parallel (~parallel subprocesses)
3. `startKeychainPrefetch()` — fires macOS keychain reads in parallel

### Core Query Loop

`query.ts` → `QueryEngine.ts`: processes user input, builds messages with system/user context, calls the Claude API via `@anthropic-ai/sdk`, dispatches tool results, handles compaction.

### Commands (`commands.ts`, `commands/`)

101+ slash commands. Each command exports a descriptor with type `local-jsx` (interactive React/Ink UI) or similar. Commands are lazy-loaded via dynamic `import()`.

### Tools (`Tool.ts`, `tools.ts`, `tools/`)

43+ tools (BashTool, FileEditTool, FileReadTool, GlobTool, GrepTool, WebSearchTool, AgentTool, TaskCreateTool, etc.). Each extends the `Tool` base with `canUseToolFn`, input validation, and execution. Permission checks prevent operation outside allowed directories.

### State Management (`state/AppState.tsx`)

Custom store via `createStore()` with React 19's `useSyncExternalStore`. Access via `useAppState(selector)` — selector prevents unnecessary re-renders. Providers: `AppStateProvider`, `StatsProvider`, `FpsMetricsProvider`, `MailboxProvider`.

### Services (`services/`)

Key services: `api/` (Claude API, files API, retry), `analytics/` (GrowthBook, telemetry), `mcp/` (Model Context Protocol servers), `compact/` (message history compaction), `policyLimits/`, `oauth/`, `plugins/`, `lsp/`.

### Bootstrap Global State (`bootstrap/`)

Session metadata and ~150+ global variables. Access via memoized functions, not direct imports — this pattern breaks circular dependencies.

### Ink (TUI) (`ink/`)

Heavily customized fork of the Ink terminal UI library. Includes custom layout engine (yoga-layout), ANSI parser (`termio/`), and renderer.

## Key Patterns

- **Circular dependency breaking**: Use lazy `require()` or dynamic `import()` instead of top-level imports when cycles are detected.
- **ESLint custom rules**: `custom-rules/no-top-level-side-effects`, `custom-rules/bootstrap-isolation`, `custom-rules/no-process-exit`, `custom-rules/no-process-env-top-level`. Respect these — they enforce startup correctness and module isolation.
- **Biome formatter**: `biome-ignore-all assist/source/organizeImports` is used where import order must be preserved for side-effects.
- **Test mode**: `NODE_ENV === 'test'` skips git operations and enables `resetStateForTests()`. Memoized functions expose `.cache.clear?.()` for test isolation.
- **Settings**: User config lives in `~/.claude/settings.json`. Remote managed settings can override local ones.
- **ESM imports**: Always use `.js` extensions in import paths even for `.ts` source files.
