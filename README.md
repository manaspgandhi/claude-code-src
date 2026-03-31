# Claude Code Source: Deep-Dive Findings

A technical investigation of the Claude Code CLI source — interesting, surprising, and revealing things found by reading the code.

---

## Tier 1: Most Surprising / Revelatory

Things that would genuinely surprise most Claude users.

### Hidden Modes & Gated Features

**1. KAIROS — Background Daemon Mode**
Feature-gated with `bun:bundle` flag `KAIROS`. Enables a background assistant/daemon mode for claude.ai orchestration. Supports proactive background task handling separate from the interactive REPL. (`main.tsx:80`, `assistant/`)

**2. COORDINATOR_MODE — Multi-Agent Orchestration**
A full coordinator/worker architecture where Claude Code spawns and manages subordinate worker agents. Workers can't communicate with each other — only through the coordinator. Workers get different tool sets depending on "simple" vs "full" mode. (`coordinator/coordinatorMode.ts:36-41`)

**3. Auto-Dream — Background Memory Consolidation Without User Awareness**
`autoDream` automatically runs `/dream` as a forked subagent when two gates pass: time gate (24 hours since last run) + session gate (5+ sessions completed). It fires entirely in the background without surfacing anything to the user. Uses a file-based distributed lock for multi-process safety. (`services/autoDream/autoDream.ts:1-66`)

**4. TRANSCRIPT_CLASSIFIER — Auto-Approve Bash Commands**
A feature-gated permission mode (`auto`) that runs async classifiers — with optional 2-stage thinking — to automatically approve bash commands *before the user can respond*. The permission system has a `PendingClassifierCheck` type that holds a decision as "ask" while a background classifier evaluates it. (`types/permissions.ts:304-307, 346-397`)

**5. Indefinite Unattended Retry Mode**
`CLAUDE_CODE_UNATTENDED_RETRY` env var (ant-only) enables indefinite retry with 5-minute backoff for unattended sessions. Keeps sessions alive with periodic keep-alive yields. (`services/api/withRetry.ts:91-104`)

**6. Bridge Architecture — Remote Control**
A multi-session persistent server that polls for work via an environments API. Supports three spawn modes: single-session, worktree (isolated git branch per session), and same-dir (shared). Sessions auto-heartbeat to extend leases with max backoff of 5 minutes and reset cap of 6 hours. (`bridge/types.ts:69, 81-115`)

**7. Work Secrets — Encoded Session Bootstrap Payloads**
Session secrets are base64url-encoded JSON blobs containing the ingress token, API base URL, auth config, MCP config, env vars, and even git repo/ref/token for source-based deployments. This is how remote bridge workers get fully bootstrapped. (`bridge/types.ts`)

**8. Permission Responses Sent Back via Events API**
When the bridge is active, human permission decisions (allow/deny) are sent back to the running session as special `control_response` events via the session events API — enabling remote permission UIs decoupled from the terminal. (`bridge/types.ts:124-131`)

### Telemetry & Analytics

**9. Hardcoded Public Datadog Client Token**
Telemetry is sent to `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` with a hardcoded public client token (`pubbbf48e6d78dae54bceaa4acf463299bf`). The token is explicitly "public" (safe to expose), but the destination is confirmed Datadog. An event allowlist controls exactly which 50+ event types get shipped. (`services/analytics/datadog.ts:12-14`)

**10. User Identity Bucketing for Privacy**
Rather than tracking user IDs in telemetry, user IDs are hashed into 30 buckets. Only the bucket number appears in alerts. This deliberately destroys identity while preserving ability to detect per-user anomalies. (`services/analytics/datadog.ts:281-299`)

**11. Model Names Normalized to "other" for External Users**
To reduce Datadog cardinality, external users' model names are generalized to `"other"`. Internal dev version strings like `2.0.53-dev.20251124.t173302.sha526cc6a` are truncated to just `2.0.53-dev.20251124`. (`services/analytics/datadog.ts:204-217`)

**12. MCP Tools All Bucketed as "mcp" in Telemetry**
All `mcp__*` tool calls are normalized to just `"mcp"` in Datadog to prevent cardinality explosion from dynamically-named MCP tools. (`services/analytics/datadog.ts:196-202`)

**13. Internal Feature Config Override via Env Var**
`CLAUDE_INTERNAL_FC_OVERRIDES` (ant-only) lets developers bypass all remote evaluation and disk caches with inline JSON feature values. This is the top layer of a three-tier feature override system. (`services/analytics/growthbook.ts:159-192`)

---

## Tier 2: Technically Fascinating

Clever engineering, unusual patterns, and architectural curiosities.

### Startup Optimization

**14. Three Side-Effects That Must Run Before All Other Imports**
`main.tsx` documents that three specific side-effects must execute before any other imports fire: startup profiler checkpoint, MDM subprocess launch, and keychain prefetch. This saves ~65ms+ by parallelizing macOS system calls that would otherwise run sequentially. (`main.tsx:1-20`)

**15. Bootstrap Isolation Wall**
`bootstrap/state.ts` is explicitly constrained to only leaf re-exports. Circular dependencies are broken by ensuring nothing in the bootstrap layer imports from the main application graph. This is enforced by a custom ESLint rule: `custom-rules/bootstrap-isolation`. (`bootstrap/state.ts:14-18`)

**16. Startup Profiler Checkpoints**
`profileCheckpoint()` marks named milestones (e.g., `main_tsx_entry`) during startup. These are used to measure module evaluation time and identify startup regressions. Comments note ~135ms of import overhead. (`utils/startupProfiler.ts`)

### Security Hardening

**17. xargs Shell-Quote Attack Prevention**
The bash read-only validator explicitly blocks lowercase `-i`/`-e` flags to xargs because of GNU getopt optional-argument semantics — `-i X` gets parsed as `-i` followed by `X` as a command argument, not `-i X` as a single option. This prevents an attacker-controlled string from being treated as a shell command. (`tools/BashTool/readOnlyValidation.ts:129-150`)

**18. fd `--list-details` Blocked to Prevent PATH Hijacking**
The `-l`/`--list-details` flag for `fd` is blocked because internally `fd` spawns `ls` as a subprocess, creating a PATH hijacking vector if a malicious `ls` is on the user's PATH. (`tools/BashTool/readOnlyValidation.ts:77-78`)

**19. Classifier-Approvable vs. Always-Prompt Safety Checks**
The permission system distinguishes between safety checks where the transcript classifier is allowed to auto-approve (e.g., sensitive file paths) vs. checks that always force a human prompt (e.g., Windows path bypass attempts). (`types/permissions.ts:318-320`)

**20. Token Redaction in Debug Logs**
MCP XAA token exchange logs are scrubbed with a regex that matches all quoted token-bearing keys regardless of JSON nesting depth: `access_token`, `refresh_token`, `id_token`, `assertion`, `subject_token`, `client_secret`. (`services/mcp/xaa.ts:91-96`)

### Retry & Resilience

**21. 529 Counter Seeded Across Streaming Boundaries**
When a streaming request fails with 529, the consecutive 529 counter is pre-seeded before retrying as a non-streaming request. This prevents the retry from appearing "fresh" — the count carries over so fallback triggers correctly. (`services/api/withRetry.ts:52-88`)

**22. Source-Based 529 Retry Discrimination**
Only retry 529s for user-blocking sources (`repl_main_thread`, `agent`, `verification_agent`, `auto_mode`). Background sources like summaries and classifiers bail immediately to avoid amplifying load during gateway cascades. (`services/api/withRetry.ts:57-82`)

**23. Three-Layer Feature Override Hierarchy**
Feature flag resolution: env var (`CLAUDE_INTERNAL_FC_OVERRIDES`) → local config (`growthBookOverrides` in `.claude/globalSettings.json`) → remote GrowthBook. Each layer can override the next. (`services/analytics/growthbook.ts:170-236`)

**24. Feature Config Race Condition Handling**
`onGrowthBookRefresh()` handles the race where GrowthBook init (100ms) completes before the REPL mounts (600ms). Listeners queued before init will fire immediately on registration if GrowthBook is already ready. (`services/analytics/growthbook.ts:139-157`)

### MCP & OAuth

**25. Enterprise Managed OAuth Without Browser Consent**
The XAA (Enterprise Managed Authorization) flow implements a four-layer token exchange without ever opening a browser: RFC 8693 Token Exchange → RFC 7523 JWT Bearer, with PRM (Protected Resource Metadata) discovery. (`services/mcp/xaa.ts:1-22`)

**26. CCR v2 Dual-Transport Architecture**
Bridge sessions can use two transport modes: legacy WebSocket (bidirectional) or CCR v2 (SSE for reads + CCRClient for writes/heartbeat). Version tracked per session via `workerEpoch`. (`bridge/types.ts:192-207`)

### Fast Mode

**27. Fast Mode Cooldown Separate from Toggle State**
Fast mode has two independent states: the user's toggle setting and a cooldown flag. When cooldown is active, fast mode is "effectively disabled" even if the setting is `true`. Overage rejection from the API triggers the cooldown automatically. (`utils/fastMode.ts:38-46`, `services/api/withRetry.ts:28-32`)

**28. `tengu_penguins_off` Feature Flag Can Disable Fast Mode**
GrowthBook feature `tengu_penguins_off` can disable fast mode with per-subscription-level checks (free users require paid tier, etc). This is the mechanism for gradual rollout gating. (`utils/fastMode.ts:72-94`)

### Beta Headers

**29. 15+ Active Beta Feature Headers**
The codebase has 15+ beta headers in active use:
- `interleaved-thinking-2025-05-14` — extended thinking
- `context-1m-2025-08-07` — 1M context window
- `tool-search-tool-2025-10-19` — tool search
- `prompt-caching-scope-2026-01-05` — cache control
- `fast-mode-2026-02-01` — fast mode
- `afk-mode-2026-01-31` — AFK/assistant mode
- `advisor-tool-2026-03-01` — advisor tool
(`constants/betas.ts:3-31`)

**30. Provider-Specific Beta Headers**
The tool search beta uses different headers depending on provider: `advanced-tool-use-2025-11-20` for Claude API/Foundry, `tool-search-tool-2025-10-19` for Vertex AI and Bedrock. (`constants/betas.ts:10-14`)

**31. Bedrock Can't Use HTTP Headers for Betas**
Some betas only work via `extraBodyParams` on Bedrock, not HTTP headers — `INTERLEAVED_THINKING`, `CONTEXT_1M`, `TOOL_SEARCH_3P`. This reflects a Bedrock API limitation. (`constants/betas.ts:38-42`)

**32. Vertex countTokens API Restricted to 3 Betas**
Only three specific betas are allowed on the Vertex AI `countTokens` API call. Others would cause the call to fail. (`constants/betas.ts:48-52`)

### Coordinator Architecture

**33. Task Notifications as XML in User-Role Messages**
Worker results from the coordinator arrive as `<task-notification>` XML blocks embedded inside user-role messages — not as system messages or tool results. This is how coordinators get worker output without a new message type. (`coordinator/coordinatorMode.ts:142-160`)

**34. Scratchpad Directory Injected into Worker System Prompts**
The coordinator injects a scratchpad directory path and available tool context into each worker's system prompt so workers know their working context and constraints. (`coordinator/coordinatorMode.ts:80-109`)

---

## Tier 3: Good to Know

Useful context, well-executed patterns, and product decisions visible in code.

### Model Management

**35. Sonnet 4.6 Migration via Code**
Model defaults are advanced through explicit named migrations: `migrateSonnet45ToSonnet46()`. This ensures existing users get upgraded without overwriting custom settings. (`main.tsx:182`)

**36. Model Cost Tracking Persisted Per Session**
`lastCost`, `lastAPIDuration`, `lastToolDuration`, etc. are stored in the project config per session and restored when a session resumes. (`cost-tracker.ts:86-123`)

**37. Model Name Normalization Before API Calls**
User-specified model strings go through `normalizeModelStringForAPI()` before being sent, allowing aliases and shorthand. 

### Session Management

**38. CLAUDE.md Snapshot Comparison**
On session resume, if the stored CLAUDE.md content doesn't match current content, a dialog prompts the user — preventing stale instructions from silently influencing the resumed session. (`screens/`)

**39. Session Persistence Disabling**
Sessions can explicitly opt out of persistence via `setSessionPersistenceDisabled()`, useful for temporary/test runs that shouldn't pollute session history. 

**40. Session Resume with History Picker**
`resumeChooser` displays a list of recent sessions with summaries for recovery from crashes or restarts.

### Hooks System

**41. Hook Source Attribution**
Every hook execution is tagged with its origin: `userSettings`, `projectSettings`, `plugin`, or `SDK`. This helps debug which hook is firing when there are conflicts. 

**42. Pre-Turn vs. Post-Sampling Hook Timing**
Hooks fire at different lifecycle points: setup/init hooks, `SessionStart`, `postSampling`, before tool execution, and after tool execution — each with different execution windows and context available. 

### Configuration Layers

**43. Six-Layer Config Merging**
User > Project > Local > Flag (CLI `--settings`) > Policy > Remote Managed. Each layer has explicit precedence. Fields can have different sources, and the source is tracked for debugging. 

**44. MDM Integration on macOS**
macOS MDM (Mobile Device Management) policies are loaded at startup via `plutil` subprocesses. Organizations can enforce settings like allowed models, permission modes, and network policies via MDM profiles.

**45. Runtime Config Override via `--settings` Flag**
`--settings` accepts inline JSON on the CLI, allowing a single-session override of any config value without touching user config files. Used for testing and CI.

### Plugins & Skills

**46. Skill Directory Auto-Discovery from CLAUDE.md**
Skills are discovered by scanning `skill-dir` entries in CLAUDE.md files. This means skill availability is scoped to the project, not just the user.

**47. Marketplace Clones from GitHub**
`marketplaceManager` downloads skills by cloning from a GitHub marketplace repo, with version management. Internal and external builds use different source repos.

**48. Background Orphaned Plugin Cleanup**
`cleanupOrphanedPluginVersionsInBackground()` runs async cleanup of old plugin versions without blocking startup. 

**49. Bundled First-Party Skills**
`initBundledSkills()` loads built-in skills (`update-config`, `schedule`, `debug`, etc.) as part of startup. These are separate from user-installed marketplace skills.

### Voice Mode

**50. Voice Mode Availability Multi-Step Check**
Six sequential checks before voice mode is enabled: auth enabled → recording available → API key available → SoX recording dependency installed → mic permission granted → language configured. Any failure short-circuits. (`commands/voice/voice.ts:14-145`)

**51. Microphone Permission Probed Early**
`requestMicrophonePermission()` fires at startup to trigger the OS permission dialog before the user first tries to use hold-to-talk, avoiding a mid-recording permission interruption.

**52. Language Fallback Hints Shown Max Twice**
Voice STT shows language configuration hints at most 2 times across sessions to avoid annoyance. After two hints, the user is assumed to have seen and ignored the suggestion.

### Bare Mode

**53. `--bare` Flag Skips Everything Non-Essential**
`--bare` disables: hooks, LSP integration, plugin sync, attribution, auto-memory, background prefetches, and keychain reads. Used for fast scripting and CI environments where startup time matters more than features.

### Permission System Details

**54. Permission Rules Tracked with Source**
Every permission rule carries its source: `userSettings`, `projectSettings`, `localSettings`, `flagSettings`, `policySettings`, `cliArg`, `command`, or `session`. This enables rule provenance debugging.

**55. Suggestions in Permission Responses**
Permission decisions can include a `suggestions` array with recommended rule changes, allowing the UI to offer "add this to your allow list" options inline.

**56. Content Blocks in Permission Prompts**
Permission prompts support image/attachment content blocks in responses — enabling users to attach screenshots or context when making a permission decision via remote UI.

### Ant-Only / Internal Features

**57. `/ant-trace` Command Stubbed in External Builds**
The internal `/ant-trace` command has `isEnabled: () => false, isHidden: true` in external builds. The stub exists so the command system doesn't need conditional loading.

**58. Internal GrowthBook Override Env Var**
`CLAUDE_INTERNAL_FC_OVERRIDES` lets internal developers completely bypass feature flag evaluation with hardcoded JSON values.

**59. `USER_TYPE=ant` Feature Branching Throughout**
`process.env.USER_TYPE === 'ant'` is checked throughout the codebase to gate internal-only features that aren't behind compile-time `bun:bundle` flags.

### Error Handling & Observability

**60. Graceful Shutdown Registry**
`registerCleanup()` allows subsystems to register cleanup functions that run before `process.exit()`. Each has a timeout so a hung cleanup doesn't stall shutdown.

**61. In-Memory Error Ring Buffer**
`getInMemoryErrors()` maintains a rolling buffer of recent errors that can be included in transcripts or diagnostic reports without requiring file I/O.

**62. Stale Connection vs. General Connection Error Discrimination**
`ECONNRESET`/`EPIPE` are treated as stale connections (retryable) vs. other `APIConnectionError` subtypes (may fail-fast). (`services/api/withRetry.ts:112-118`)

**63. FPS Metrics for Terminal Performance**
A terminal FPS tracker monitors rendering frame rate for performance debugging. Exposed in diagnostics. (`utils/fpsTracker.ts`)

### Context & Memory

**64. Context Window Dynamically Calculated with Beta Awareness**
Context window size is calculated at runtime, accounting for beta headers that may extend limits (e.g., `context-1m-2025-08-07` enables 1M token context for supported models).

**65. Max Output Tokens Tracked Separately from Context Window**
Max output tokens per model is tracked independently from the context window size — they're different limits and neither implies the other.

**66. Prompt Caching Scope Beta**
`prompt-caching-scope-2026-01-05` beta enables fine-grained control over which message blocks participate in prompt caching. This is not yet publicly documented.

**67. Session Activity Ring Buffer in Bridge**
Bridge sessions maintain a ring buffer of the last ~10 session activities (timestamp + summary) for debugging reconnections and session state inspection.

**68. Stderr Ring Buffer for Child Processes**
Bridge worker processes maintain a ring buffer of recent stderr output for diagnosing child process failures after the fact.

### Workspace

**69. Project Root Fixed at Startup**
Project root is set once at startup (including via `--worktree` flag) and never changes, even if `EnterWorktreeTool` is called mid-session. Worktree mode creates isolated git branches per session.

**70. Additional Working Directories**
Users can grant file system access to directories outside the project root. Each granted directory is tracked with its source (how the grant was made) for auditability.

### Product Commands

**71. `/insights` Command**
Collects comprehensive diagnostic data: env state, auth status, config validation, MCP server status, tool availability. Useful for support and debugging.

**72. `/memory` and `/dream` Commands**
`/memory` provides a UI for viewing and editing memory files. `/dream` triggers manual memory consolidation. Both interact with the same memory system that `autoDream` manages automatically.

**73. `/remote-setup` and `/bridge` Commands**
First-class commands for configuring remote code session handling, exposing the bridge architecture to users.

**74. `/backfill-sessions` Command**
A background command to rebuild missing session metadata, used when session data is corrupted or missing from the session store.

**75. `/cost` Shows Detailed Session Cost Breakdown**
Detailed cost reporting per session including API cost, duration, and token counts. `/fast` toggles fast mode with an explanation of what it does and current cooldown state.

### Infrastructure & Operations

**76. SSE + CCRClient Dual Transport**
CCR v2 uses SSE (server-sent events) for read-only streaming and a separate CCRClient for writes and heartbeats. This splits the read and write paths for better reliability.

**77. JWT-Based Lease Extension**
Session lease extension via `heartbeatWork()` uses JWT token validation instead of a database lookup. This makes heartbeats cheap and stateless. (`bridge/types.ts:171-175`)

**78. Plugin System with Marketplace, Bundled, and External Sources**
Three distinct plugin loading paths: bundled first-party skills, marketplace-sourced plugins (downloaded from GitHub), and externally configured plugins. Each has different trust and loading semantics.

**79. Speculation System for Faster Response**
Optional prompt speculation (`shouldEnablePromptSuggestion()`) can pre-compute likely next responses for reduced perceived latency.

**80. Exposure Deduplication in Hot Paths**
GrowthBook experiment exposures are tracked in-session to prevent duplicate telemetry events from render loops and other hot paths. (`services/analytics/growthbook.ts:87-89`)

---

## Summary Statistics

| Category | Count |
|---|---|
| Hidden/gated features | 8 |
| Telemetry/analytics findings | 5 |
| Security hardening | 6 |
| Performance optimizations | 5 |
| Retry/resilience patterns | 4 |
| Beta/API features | 6 |
| Coordinator/multi-agent | 4 |
| Model management | 5 |
| Session management | 4 |
| Configuration system | 5 |
| Plugins & skills | 6 |
| Voice mode | 3 |
| Permission system | 5 |
| Ant-only/internal | 3 |
| Error handling | 4 |
| Infrastructure | 7 |
| **Total** | **80+** |

---

*Generated by reading the Claude Code source at `/Users/manasgandhi/Projects/claude-code-src`.*
