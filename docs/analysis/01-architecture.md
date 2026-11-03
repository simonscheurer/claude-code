# Claude Code CLI — Comprehensive Architecture Analysis

> **Document version:** April 2026  
> **Codebase commit:** `1becaba` (init)  
> **Author:** Architecture Agent (automated analysis)  
> **Target file:** `docs/analysis/01-architecture.md`

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Layout](#2-repository-layout)
3. [High-Level System Architecture](#3-high-level-system-architecture)
4. [Startup and Initialization Flow](#4-startup-and-initialization-flow)
5. [Layer Architecture](#5-layer-architecture)
6. [Core Runtime Model: The Agentic Query Loop](#6-core-runtime-model-the-agentic-query-loop)
7. [Tool System](#7-tool-system)
8. [State Management](#8-state-management)
9. [API Communication Layer](#9-api-communication-layer)
10. [Session Persistence](#10-session-persistence)
11. [Permission System](#11-permission-system)
12. [Bridge System (CCR — Claude Code Remote)](#12-bridge-system-ccr--claude-code-remote)
13. [MCP Integration](#13-mcp-integration)
14. [Subagent and Task Architecture](#14-subagent-and-task-architecture)
15. [Plugin and Skills System](#15-plugin-and-skills-system)
16. [Terminal UI (React + Ink)](#16-terminal-ui-react--ink)
17. [Analytics, Observability, and Feature Gates](#17-analytics-observability-and-feature-gates)
18. [Build System and Feature Flags](#18-build-system-and-feature-flags)
19. [Key Design Decisions](#19-key-design-decisions)
20. [Component Interaction Diagrams](#20-component-interaction-diagrams)
21. [Essential Files Reference](#21-essential-files-reference)

---

## 1. Project Overview

Claude Code is Anthropic's CLI tool for AI-assisted software engineering. It runs as a terminal application that presents a REPL interface to the user, communicates with the Anthropic API (or compatible providers) to drive an LLM, and grants that LLM the ability to read and modify files, execute shell commands, search the web, and spawn subagents.

Key characteristics:

- **Runtime:** Bun (not Node.js), enabling the `bun:bundle` dead-code elimination system used for build-time feature flagging.
- **UI framework:** React 19 + Ink 5, rendering a React component tree to a TTY terminal.
- **LLM provider:** Anthropic Claude models (direct API, AWS Bedrock, GCP Vertex, Azure).
- **Extensibility:** Model Context Protocol (MCP) servers, Plugins, Skills (slash commands), Hooks (shell commands at lifecycle points).
- **Agentic loop:** Streaming response → extract tool_use blocks → execute tools concurrently/serially → loop until no tool calls or stop condition.
- **Dual mode:** Interactive REPL with full React UI, and headless/SDK mode with JSON output to stdout.

The codebase is approximately 785KB in the main module (`src/main.tsx`) alone, with hundreds of supporting modules organized in clearly-delineated directories.

---

## 2. Repository Layout

```
claude-code/
├── src/
│   ├── entrypoints/          # Binary entry points
│   │   ├── cli.tsx           # True executable entry, fast-paths for --version
│   │   ├── init.ts           # Memoized initialization function
│   │   └── agentSdkTypes.ts  # TypeScript types for SDK/headless protocol
│   │
│   ├── main.tsx              # Primary CLI module (~785KB): Commander setup,
│   │                         # all command definitions, startup orchestration
│   │
│   ├── screens/
│   │   └── REPL.tsx          # Main interactive screen component
│   │
│   ├── components/
│   │   ├── App.tsx           # Top-level React context provider tree
│   │   └── ...               # ~50 UI components
│   │
│   ├── query.ts              # Core agentic query loop (~1730 lines)
│   ├── QueryEngine.ts        # SDK/headless conversation lifecycle manager
│   ├── Tool.ts               # Tool type definitions and ToolUseContext
│   ├── tools.ts              # Tool registry (getAllBaseTools, getTools)
│   ├── commands.ts           # Slash command registry
│   │
│   ├── tools/                # ~35+ built-in tool implementations
│   │   ├── BashTool/
│   │   ├── FileReadTool/
│   │   ├── FileEditTool/
│   │   ├── FileWriteTool/
│   │   ├── GlobTool/
│   │   ├── GrepTool/
│   │   ├── WebFetchTool/
│   │   ├── WebSearchTool/
│   │   ├── AgentTool/        # Subagent spawning
│   │   ├── MCPTool/          # MCP server tool wrapper
│   │   └── ...
│   │
│   ├── state/
│   │   ├── store.ts          # Generic pub/sub store primitive
│   │   ├── AppStateStore.ts  # AppState type definition
│   │   ├── AppState.tsx      # React provider + hooks
│   │   └── onChangeAppState.ts # Side-effect handler on state changes
│   │
│   ├── bootstrap/
│   │   └── state.ts          # Global process-level singleton state
│   │
│   ├── services/
│   │   ├── api/              # Anthropic API communication
│   │   │   ├── claude.ts     # queryModel() - the API call function
│   │   │   └── client.ts     # getAnthropicClient() - SDK factory
│   │   ├── mcp/              # MCP client management
│   │   ├── compact/          # Context compression strategies
│   │   ├── analytics/        # Telemetry, GrowthBook, OpenTelemetry
│   │   ├── oauth/            # OAuth flow for claude.ai auth
│   │   ├── plugins/          # Plugin installation and management
│   │   ├── lsp/              # Language Server Protocol integration
│   │   └── tools/
│   │       ├── toolOrchestration.ts  # runTools() - concurrent/serial batching
│   │       └── toolExecution.ts      # runToolUse() - single tool execution
│   │
│   ├── bridge/               # CCR (Claude Code Remote) bridge system
│   │   ├── bridgeMain.ts     # runBridgeLoop() - CCR polling loop
│   │   ├── replBridge.ts     # ReplBridgeHandle - WebSocket to claude.ai
│   │   └── types.ts          # Bridge types
│   │
│   ├── hooks/
│   │   └── useCanUseTool.tsx # Permission decision hook
│   │
│   ├── utils/                # ~60+ utility modules
│   │   ├── sessionStorage.ts # JSONL transcript persistence
│   │   ├── config.ts         # Global/project config read/write
│   │   ├── permissions/      # Permission mode utilities
│   │   └── ...
│   │
│   ├── types/                # Shared TypeScript type definitions
│   │   ├── message.ts        # Message union type
│   │   ├── permissions.ts    # PermissionMode, PermissionResult
│   │   └── ...
│   │
│   ├── cli/                  # Non-interactive CLI output
│   │   ├── print.ts          # Headless text output
│   │   └── structuredIO.ts   # JSON/NDJSON streaming
│   │
│   ├── tasks/                # Background task management
│   ├── skills/               # Bundled skill definitions
│   ├── plugins/              # Built-in plugin registry
│   ├── commands/             # Individual slash command implementations
│   ├── context/              # Context data (notifications, stats)
│   ├── memdir/               # Memory directory management (CLAUDE.md)
│   ├── jobs/                 # Background classification jobs
│   ├── buddy/                # Companion sprite system
│   └── ink.ts                # Ink re-export with ThemeProvider wrapper
│
└── docs/
    └── analysis/             # (created by this analysis)
```

---

## 3. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PROCESS BOUNDARY                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    cli.tsx (entry point)                     │   │
│  │   fast-paths: --version, --dump-system-prompt               │   │
│  │   env setup: COREPACK, NODE_OPTIONS, ablation baseline      │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │ dynamic import                        │
│  ┌───────────────────────────▼──────────────────────────────────┐   │
│  │                      main.tsx (CLI core)                      │   │
│  │  Commander.js argument parsing                                │   │
│  │  preAction: init() + runMigrations()                         │   │
│  │  action: kairos checks → showSetupScreens → launchRepl()     │   │
│  └──────┬────────────────────────────────────────────────────────┘   │
│         │                                                             │
│  ┌──────▼──────────────────────────────────────────────────────┐    │
│  │               React/Ink Render Tree                          │    │
│  │  <App>          ← FpsMetrics, Stats, AppState contexts       │    │
│  │    <REPL>       ← main interactive screen (~200 imports)     │    │
│  │    </REPL>                                                    │    │
│  │  </App>                                                       │    │
│  └──────┬──────────────────────────────────────────────────────┘    │
│         │ user submits message                                        │
│  ┌──────▼──────────────────────────────────────────────────────┐    │
│  │               query() AsyncGenerator                         │    │
│  │   queryLoop():                                               │    │
│  │     while(true):                                             │    │
│  │       compaction pipeline (snip, micro, collapse, auto)      │    │
│  │       callModel() → streaming API                            │    │
│  │       collect tool_use blocks                                │    │
│  │       runTools() → execute tools                             │    │
│  │       collect results → loop or terminate                    │    │
│  └──────┬──────────────────────────────────────────────────────┘    │
│         │                                                             │
│  ┌──────▼──────────────────────────────────────────────────────┐    │
│  │               Anthropic API (streaming)                      │    │
│  │   - Direct API / Bedrock / Vertex / Azure                    │    │
│  │   - Beta features: extended thinking, tool use, cache        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              External Tool Execution                          │   │
│  │   BashTool → child process                                   │   │
│  │   FileReadTool → fs.readFile                                 │   │
│  │   FileEditTool / FileWriteTool → atomic write                │   │
│  │   AgentTool → in-process subagent OR worktree OR remote      │   │
│  │   MCPTool → MCP server via stdio/SSE/HTTP/WS                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Session Persistence                              │   │
│  │   ~/.claude/projects/<hash>/<uuid>.jsonl (JSONL transcript)  │   │
│  │   ~/.claude/.claude.json (global config)                     │   │
│  │   ~/.claude/projects/<hash>/.claude.json (project config)    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              Bridge System (CCR)                              │   │
│  │   WebSocket → claude.ai ingress                              │   │
│  │   Enables: remote control, claude.ai UI, background tasks    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Startup and Initialization Flow

### 4.1 Entry Point: `src/entrypoints/cli.tsx`

The binary entry point is `src/entrypoints/cli.tsx` (lines 1-90+). It performs the following in order:

1. **Synchronous env setup** (top of module, before `main()`):
   - Sets `COREPACK_ENABLE_AUTO_PIN=0` to prevent corepack interference.
   - If `CLAUDE_CODE_REMOTE=true`, increases Node.js heap to 8GB via `NODE_OPTIONS`.
   - If `ABLATION_BASELINE` feature flag and `CLAUDE_CODE_ABLATION_BASELINE` env var are set, enables a set of env vars that disable features (thinking, compact, auto-memory, background tasks) to create a controlled science baseline.

2. **Fast-path: `--version`** (`cli.tsx:37-42`)  
   If the only argument is `--version`, `-v`, or `-V`, inlines `MACRO.VERSION` (a build-time macro replaced by Bun at bundle time) and exits. Zero module loading required.

3. **Load startup profiler** (`cli.tsx:47-48`)  
   For all other paths, dynamically imports `startupProfiler.js` and calls `profileCheckpoint('cli_entry')` to begin timing.

4. **Fast-path: `--dump-system-prompt`** (`cli.tsx:53-80+`)  
   ANT-only (dead-code eliminated in external builds via `feature('DUMP_SYSTEM_PROMPT')`). Loads just enough to render the system prompt and exit.

5. **Main path: load `main.tsx`** (`cli.tsx:~80+`)  
   Dynamically imports `src/main.tsx` and calls its exported `main()` function. All heavy initialization is deferred to this point.

### 4.2 Commander Setup: `src/main.tsx`

`main.tsx` contains the full CLI definition using `@commander-js/extra-typings`. The `main()` function:

1. **`eagerLoadSettings()`** — Loads config files from disk immediately to avoid stalls during startup.

2. **Creates Commander program** with all CLI options: `--model`, `--print`/`-p`, `--output-format`, `--max-turns`, `--system-prompt`, `--permission-mode`, `--dangerously-skip-permissions`, `--mcp-config`, `--resume`, `--continue`, `--session-id`, `--cwd`, `--worktree`, `--remote`, `--agent`, `--verbose`, etc.

3. **`preAction` hook** — Runs before any action. Calls:
   - `init()` from `src/entrypoints/init.ts` (memoized)
   - `runMigrations()` for schema migrations

4. **Main action handler** — After Commander parses, the main action:
   - Checks kairos/assistant mode conditions
   - Computes initial `AppState` including resolved model, permission mode, MCP config
   - Calls `showSetupScreens()` for first-time onboarding/trust dialogs (if interactive)
   - Calls `launchRepl()` for interactive mode OR processes headless/print mode

### 4.3 Initialization: `src/entrypoints/init.ts`

The `init()` function is memoized (called once per process). It performs:

1. `enableConfigs()` — Activates config loading subsystem
2. `applySafeEnvVars()` — Applies whitelisted environment variables
3. `setupGracefulShutdown()` — Registers SIGTERM/SIGINT handlers
4. OAuth initialization
5. Policy limits initialization (enterprise controls)
6. mTLS configuration
7. HTTP proxy setup
8. LSP server manager initialization
9. JetBrains plugin detection

### 4.4 REPL Launch: `src/replLauncher.tsx`

`launchRepl()` performs the critical render bootstrap:

```
launchRepl(root, appProps, replProps, renderAndRun)
  → dynamic import App from './components/App.js'
  → dynamic import REPL from './screens/REPL.js'
  → renderAndRun(root, <App {...appProps}><REPL {...replProps}/></App>)
```

The dynamic imports here mean the React component tree is NOT loaded until `launchRepl()` is called. This keeps the startup profiler overhead-free and enables the `--version` fast-path.

`renderAndRun()` from `src/interactiveHelpers.tsx`:
1. `root.render()` — Starts the Ink/React rendering loop
2. `startDeferredPrefetches()` — Background work after first render (memory loading, analytics, etc.)
3. `root.waitUntilExit()` — Blocks until the Ink app exits
4. `gracefulShutdown(0)` — Clean process termination

### 4.5 Startup Sequence Diagram

```
cli.tsx::main()
│
├── (sync) env setup: COREPACK, NODE_OPTIONS, ablation baseline
│
├── args[0] === '--version' ? → print version, exit
│
├── profileCheckpoint('cli_entry')
│
├── args[0] === '--dump-system-prompt' ? → [ANT-ONLY] render+exit
│
└── dynamic import main.tsx → main()
    │
    ├── eagerLoadSettings()
    │
    ├── Commander.parse(process.argv)
    │   │
    │   ├── preAction:
    │   │   ├── profileCheckpoint('pre_action')
    │   │   ├── init()                     ← one-time initialization
    │   │   └── runMigrations()
    │   │
    │   └── action handler:
    │       ├── resolve model, permission mode
    │       ├── build initial AppState
    │       ├── (interactive only) showSetupScreens()
    │       │   ├── trust dialog (if new project)
    │       │   └── onboarding wizard (if first run)
    │       │
    │       ├── (interactive) launchRepl()
    │       │   ├── import App, REPL
    │       │   ├── root.render(<App><REPL/></App>)
    │       │   ├── startDeferredPrefetches()
    │       │   └── root.waitUntilExit()
    │       │
    │       └── (headless/print) processUserInput() → query()
    │           └── JSON/text streaming to stdout
```

---

## 5. Layer Architecture

The codebase has clear architectural layers with well-defined responsibilities:

```
┌─────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER (React/Ink)                                 │
│  src/screens/REPL.tsx      — Main interactive screen           │
│  src/components/           — ~50 reusable UI components        │
│  src/ink.ts                — Ink primitives with ThemeProvider  │
├─────────────────────────────────────────────────────────────────┤
│  APPLICATION / COMMAND LAYER                                    │
│  src/main.tsx              — CLI argument handling             │
│  src/commands.ts           — Slash command registry (~80 cmds) │
│  src/commands/*/           — Individual slash command impls    │
│  src/hooks/                — React hooks bridging UI ↔ logic   │
├─────────────────────────────────────────────────────────────────┤
│  QUERY / AGENTIC LOOP LAYER                                     │
│  src/query.ts              — Core query loop                   │
│  src/QueryEngine.ts        — SDK/headless conversation mgmt    │
│  src/query/                — Query config, deps, transitions   │
│  src/services/tools/       — Tool orchestration & execution    │
├─────────────────────────────────────────────────────────────────┤
│  TOOL LAYER                                                     │
│  src/Tool.ts               — Tool<I,O,P> type + ToolUseContext │
│  src/tools.ts              — getAllBaseTools(), getTools()      │
│  src/tools/                — ~35+ tool implementations         │
├─────────────────────────────────────────────────────────────────┤
│  SERVICE LAYER                                                  │
│  src/services/api/         — Anthropic API (queryModel)        │
│  src/services/mcp/         — MCP client management            │
│  src/services/compact/     — Context compaction strategies     │
│  src/services/analytics/   — Telemetry + feature gates         │
│  src/services/oauth/       — Authentication                    │
│  src/services/lsp/         — Language server integration       │
│  src/bridge/               — CCR remote bridge                 │
├─────────────────────────────────────────────────────────────────┤
│  STATE LAYER                                                    │
│  src/state/store.ts        — Generic pub/sub store             │
│  src/state/AppStateStore.ts — AppState type (DeepImmutable)    │
│  src/state/AppState.tsx    — React provider + useSyncExtStore  │
│  src/bootstrap/state.ts    — Process-global singleton state    │
├─────────────────────────────────────────────────────────────────┤
│  PERSISTENCE LAYER                                              │
│  src/utils/sessionStorage.ts — JSONL transcript files          │
│  src/utils/config.ts         — Global/project config files     │
│  src/memdir/                 — CLAUDE.md memory files          │
├─────────────────────────────────────────────────────────────────┤
│  UTILITY / INFRASTRUCTURE LAYER                                 │
│  src/utils/                — ~60+ utility modules             │
│  src/types/                — Shared TypeScript types           │
│  src/constants/            — Application constants             │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Between Layers

```
User Input (keyboard/CLI)
         │
         ▼
Presentation Layer (REPL.tsx)
  - keyboard handling (Ink useInput)
  - message queue management
  - renders conversation history
         │
         ▼  processUserInput() → enriches with context
Application Layer (main.tsx / commands.ts)
  - slash command dispatch
  - permission mode checks
  - system prompt assembly
         │
         ▼  query(params) AsyncGenerator
Query Layer (query.ts)
  - compaction pipeline
  - API call orchestration
  - tool dispatch loop
  - message accumulation
         │
         ▼  yield messages (StreamEvent | Message)
Tool Layer (tools/*.ts)
  - individual tool execution
  - file system operations
  - subprocess management
  - MCP protocol calls
         │
         ▼  API request (normalized messages + tools schema)
Service Layer (services/api/claude.ts)
  - model selection
  - beta headers
  - prompt caching
  - streaming response parsing
         │
         ▼  HTTP/2 stream → anthropic.beta.messages.stream()
Anthropic API
  - Claude model inference
  - tool calling protocol
  - thinking blocks
```

---

## 6. Core Runtime Model: The Agentic Query Loop

### 6.1 Overview

The query loop is the beating heart of Claude Code. It is implemented as an AsyncGenerator function in `src/query.ts` and runs indefinitely until Claude produces a response with no tool calls, a stop condition is triggered, or an abort signal fires.

### 6.2 `QueryParams` — What Drives the Loop

```typescript
// src/query.ts:181-199
export type QueryParams = {
  messages: Message[]          // conversation history
  systemPrompt: SystemPrompt   // base system prompt
  userContext: { [k: string]: string }   // CLAUDE.md content, date, etc.
  systemContext: { [k: string]: string } // git status, branch, etc.
  canUseTool: CanUseToolFn     // permission check function
  toolUseContext: ToolUseContext // "god object": tools, options, state refs
  fallbackModel?: string        // model to retry on 529
  querySource: QuerySource      // 'repl_main_thread' | 'agent:...' | etc.
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps              // injectable deps for testing
}
```

### 6.3 `ToolUseContext` — The God Object

`ToolUseContext` (defined in `src/Tool.ts`) is passed to every tool call. It is the central "god object" for a query session:

- `options.tools: Tools` — the assembled tool pool (built-in + MCP)
- `options.mainLoopModel` — model identifier
- `options.thinkingConfig` — extended thinking settings
- `options.agentDefinitions` — loaded agent specs
- `options.appendSystemPrompt` — per-session system prompt additions
- `options.isNonInteractiveSession` — headless mode flag
- `abortController` — the AbortController for the current query
- `agentId?: AgentId` — present for subagents, undefined for main thread
- `getAppState()` — function to read current AppState
- `setAppState()` — function to update AppState
- `addNotification()` — surface notifications to UI
- `messages: Message[]` — snapshot of messages at loop start (updated each iteration)
- `queryTracking` — chain ID + depth for analytics
- `contentReplacementState` — tracks tool result truncations
- `fileStateCache` — per-session file read cache
- `denialTracking` — tracks which tools were denied

### 6.4 The Loop in Detail

```
query(params)                    [src/query.ts:219]
  │
  ├── consumedCommandUuids: string[] = []
  │
  └── yield* queryLoop(params, consumedCommandUuids)
                │
                ├── ONCE AT ENTRY:
                │   ├── buildQueryConfig()       — snapshot gates/flags
                │   ├── startRelevantMemoryPrefetch()  — background memory load
                │   └── startSkillDiscoveryPrefetch()  — background skill lookup
                │
                └── while (true):                [main loop]
                    │
                    ├── STEP 1: Apply tool result budget
                    │   applyToolResultBudget()   — truncate oversized tool results
                    │
                    ├── STEP 2: History snip (feature-gated)
                    │   snipModule.snipCompactIfNeeded()
                    │
                    ├── STEP 3: Microcompaction
                    │   deps.microcompact()        — inline compression of repetitive patterns
                    │
                    ├── STEP 4: Context collapse (feature-gated)
                    │   contextCollapse.applyCollapsesIfNeeded()
                    │
                    ├── STEP 5: Autocompaction
                    │   deps.autocompact()
                    │   if triggered:
                    │     yield post-compact boundary messages
                    │     messagesForQuery = postCompactMessages
                    │
                    ├── STEP 6: Blocking limit check
                    │   if tokenCount > blockingLimit AND auto-compact disabled:
                    │     yield error message
                    │     return { reason: 'blocking_limit' }
                    │
                    ├── STEP 7: Stream from model
                    │   for await (message of deps.callModel({...})):
                    │     ├── handle tombstones on streaming fallback
                    │     ├── backfill tool_use observable inputs
                    │     ├── withhold recoverable errors (max_output_tokens, PTL)
                    │     ├── if tool_use block: push to toolUseBlocks[]
                    │     └── yield message to upstream consumer
                    │
                    ├── STEP 8: Post-stream recovery checks
                    │   ├── max_output_tokens recovery (up to 3 retries)
                    │   ├── prompt-too-long reactive compact
                    │   └── media error recovery
                    │
                    ├── STEP 9: Tool execution
                    │   if needsFollowUp (tool_use blocks present):
                    │     for await (update of runTools(toolUseBlocks, ...)):
                    │       ├── yield result messages
                    │       └── collect context modifiers
                    │     state.messages = [...messages, ...assistantMessages, ...toolResults]
                    │     CONTINUE (next loop iteration)
                    │
                    ├── STEP 10: Post-sampling hooks
                    │   executePostSamplingHooks()
                    │
                    ├── STEP 11: Check stop hooks
                    │   handleStopHooks()
                    │   if stop hook requests re-query: CONTINUE
                    │
                    ├── STEP 12: Token budget check (feature-gated)
                    │   checkTokenBudget()
                    │   if budget exceeded: CONTINUE with budget message
                    │
                    └── STEP 13: Terminate
                        return { reason: 'done' | 'max_turns' | ... }
```

### 6.5 Mutable Loop State

The loop carries a `State` object between iterations:

```typescript
// src/query.ts:204-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number    // max 3 retries
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // why previous iter continued
}
```

The single `let state: State = {...}` at loop start is reassigned at each `continue` site — a clean pattern that avoids 9 separate mutable variable assignments and makes continue points auditable.

### 6.6 Terminal Return Value

`query()` returns a `Terminal` type that describes WHY the loop ended:

```typescript
type Terminal =
  | { reason: 'done' }
  | { reason: 'max_turns' }
  | { reason: 'blocking_limit' }
  | { reason: 'abort' }
  | { reason: 'stop_hook' }
  | { reason: 'token_budget' }
  | { reason: 'task_budget' }
  // ... etc
```

### 6.7 QueryEngine: The Headless Wrapper

`src/QueryEngine.ts` provides `QueryEngine` — a class for SDK/headless use (not the interactive REPL). It wraps `query()` with:

- Manages `mutableMessages: Message[]` across multiple `submitMessage()` calls
- Tracks `canUseTool` denials per turn
- Converts `query()` yields to `SDKMessage` objects for JSON streaming
- Supports `--resume` and `--continue` flows
- Exposes `processInput()` for one-shot headless use

A single `QueryEngine` instance represents one conversation lifecycle.

---

## 7. Tool System

### 7.1 Tool Type Definition

```typescript
// src/Tool.ts (simplified)
export type Tool<Input, Output, P extends PermissionContext> = {
  name: string
  aliases?: string[]
  searchHint?: string
  call(input: Input, ctx: ToolUseContext, ...): AsyncGenerator<ToolProgress, Output>
  description(): string
  inputSchema: z.ZodType<Input>
  inputJSONSchema: ToolInputJSONSchema
  isEnabled(ctx: ToolPermissionContext, state: AppState): boolean
  isConcurrencySafe(input: Input): boolean
  userFacingName(input: Input): string
  renderResult(output: Output, ctx: ToolUseContext): React.ReactNode
  maxResultSizeChars?: number      // truncation budget
  backfillObservableInput?: (input) => void  // for transcript backfill
}
```

Key design points:
- `call()` returns an `AsyncGenerator` yielding `ToolProgress` events before returning the final `Output`. This enables streaming progress (e.g., bash command line-by-line output) without blocking.
- `isConcurrencySafe()` determines whether the tool can run in parallel with others. Read-only tools return `true`; write tools return `false`.
- `isEnabled()` takes both `ToolPermissionContext` (mode/rules) and `AppState` to decide whether to offer the tool to Claude at all.

### 7.2 Built-in Tool Registry

`src/tools.ts` exports `getAllBaseTools()` which assembles all tools:

```typescript
// src/tools.ts (core tools — always present)
BashTool, FileEditTool, FileReadTool, FileWriteTool,
GlobTool, GrepTool, WebFetchTool, WebSearchTool,
AgentTool, SkillTool, TaskStopTool, BriefTool,
NotebookEditTool, TaskOutputTool, TodoWriteTool,
ExitPlanModeV2Tool, TungstenTool, GrepSearchTool,
ListMcpResourcesTool, ReadMcpResourceTool, ...

// Feature-gated (runtime require())
REPLTool         — USER_TYPE === 'ant'
SuggestBackgroundPRTool — USER_TYPE === 'ant'
SleepTool        — feature('PROACTIVE') || feature('KAIROS')
CronCreateTool   — feature('AGENT_TRIGGERS')
CronDeleteTool   — feature('AGENT_TRIGGERS')
CronListTool     — feature('AGENT_TRIGGERS')
RemoteTriggerTool — feature('AGENT_TRIGGERS_REMOTE')
MonitorTool      — feature('MONITOR_TOOL')
SendUserFileTool  — feature('KAIROS')
PushNotificationTool — feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')
SubscribePRTool  — feature('KAIROS_GITHUB_WEBHOOKS')
```

`getTools()` filters by permission context and mode. `assembleToolPool()` merges built-in tools with MCP tools.

### 7.3 Tool Execution Flow

```
query.ts: toolUseBlocks collected from streaming response
    │
    ▼
runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
  [src/services/tools/toolOrchestration.ts]
    │
    ├── partitionToolCalls() — split into isConcurrencySafe and !isConcurrencySafe groups
    │
    ├── for each group (ordered):
    │   ├── if isConcurrencySafe:
    │   │     runToolsConcurrently()     — Promise.all up to MAX_TOOL_USE_CONCURRENCY (10)
    │   │     collect contextModifiers
    │   │     apply contextModifiers in original order
    │   │
    │   └── if !isConcurrencySafe:
    │         runToolsSerially()         — sequential for-await loop
    │         apply contextModifiers immediately
    │
    └── yield MessageUpdate { message?, newContext }

For each tool:
runToolUse(block, assistantMessages, canUseTool, toolUseContext)
  [src/services/tools/toolExecution.ts]
    │
    ├── findToolByName(tools, block.name)
    ├── canUseTool(tool, input, ctx) → PermissionResult
    │   └── [if denied] → yield error tool_result, return
    │
    ├── tool.call(parsedInput, ctx)
    │   → AsyncGenerator yields ToolProgress events
    │   → collect into progressMessages[]
    │
    ├── logEvent('tengu_tool_use') — analytics
    │
    └── yield tool_result UserMessage
```

### 7.4 Concurrency Model

The maximum concurrent tool executions is `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default: 10). However, the actual concurrency within a batch is limited by the `isConcurrencySafe()` predicate:

- **Read-only tools** (FileReadTool, GlobTool, GrepTool, WebFetchTool) return `true` — run concurrently
- **Write tools** (FileEditTool, FileWriteTool, BashTool, AgentTool) return `false` — run serially
- **Batching:** consecutive read-only tool calls are grouped into one concurrent batch; write tools break the batch boundary

This means Claude can batch `[Read A, Read B, Read C, Write D, Read E]` as:
```
Batch 1 (concurrent): [Read A, Read B, Read C]
Batch 2 (serial):     [Write D]
Batch 3 (concurrent): [Read E]
```

---

## 8. State Management

Claude Code uses two parallel state systems: a React context-based pub/sub store for UI-visible application state, and a module-level singleton for process-level global counters.

### 8.1 The Store Primitive

`src/state/store.ts` implements a minimal pub/sub store:

```typescript
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return   // reference equality guard
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    }
  }
}
```

This is a 34-line implementation of the Redux-like pattern. The `Object.is` check prevents unnecessary re-renders.

### 8.2 AppState — The Application State Object

`AppState` (defined in `src/state/AppStateStore.ts`) is a `DeepImmutable<{...}>` with 60+ fields:

**UI state:**
- `verbose`, `expandedView`, `isBriefOnly`, `selectedIPAgentIndex`
- `statusLineText`, `spinnerTip`, `footerSelection`
- `expandedView: 'none' | 'tasks' | 'teammates'`

**Model/session state:**
- `settings: SettingsJson` — user preferences
- `mainLoopModel: ModelSetting`, `mainLoopModelForSession: ModelSetting`
- `toolPermissionContext: ToolPermissionContext`
- `kairosEnabled: boolean` — assistant/kairos mode flag
- `agent: string | undefined` — agent name from CLI

**Bridge/remote state:**
- `replBridgeEnabled`, `replBridgeConnected`, `replBridgeSessionActive`
- `replBridgeConnectUrl`, `replBridgeSessionUrl`
- `replBridgeEnvironmentId`, `replBridgeSessionId`
- `remoteSessionUrl`, `remoteConnectionStatus`
- `remoteBackgroundTaskCount`

**Plugin/MCP state:**
- `mcp: { clients, tools, commands, resources, pluginReconnectKey }`
- `plugins: { enabled, disabled, commands, errors, installationStatus, needsRefresh }`
- `agentDefinitions: AgentDefinitionsResult`

**Agent/task state (NOT DeepImmutable):**
- `tasks: { [taskId: string]: TaskState }` — excluded from DeepImmutable because TaskState contains function types
- `agentNameRegistry: Map<string, AgentId>`
- `foregroundedTaskId?: string`

**Speculation state (for prompt suggestions):**
- `SpeculationState` — idle or active with abort, messagesRef, writtenPathsRef

Notable: the `tasks` and `agentNameRegistry` fields are explicitly excluded from `DeepImmutable<...>` because they contain function types that cannot be structurally deep-frozen. The type signature is:
```typescript
export type AppState = DeepImmutable<{...without tasks...}> & {
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  ...
}
```

### 8.3 React Integration: `src/state/AppState.tsx`

The `AppStateProvider` component:
1. Creates the store with `createStore<AppState>(initialState, onChangeAppState)`
2. Exposes it via `AppStoreContext`
3. Provides two hooks:
   - `useAppState<T>(selector: (s: AppState) => T): T` — uses `useSyncExternalStore` for safe concurrent reads
   - `useSetAppState()` — returns the store's `setState` function

The `useSyncExternalStore` pattern is the React 18/19 canonical way to subscribe to external stores. It ensures consistent snapshots during rendering even in concurrent mode.

### 8.4 Side Effects on State Change: `src/state/onChangeAppState.ts`

`onChangeAppState` is called on every AppState change. It handles cross-cutting concerns:
- Syncs `permissionMode` to CCR external metadata when it changes
- Handles `mainLoopModel` changes (clears prompt cache break detection state)
- Clears auth caches when OAuth state changes
- Triggers bridge reconnection on relevant state changes

### 8.5 Process-Level Global State: `src/bootstrap/state.ts`

A module-level singleton with strict "DO NOT ADD MORE STATE HERE" warning at line 31. Contains:
- `originalCwd: string`, `projectRoot: string`, `cwd: string`
- `totalCostUSD`, `totalAPIDuration`, `totalToolDuration`
- `modelUsage: { [modelName: string]: ModelUsage }`
- `sessionId: SessionId` — unique per session
- `startTime: number`
- Various counters: `totalLinesAdded`, `totalLinesRemoved`, `turnHookDurationMs`, etc.
- OpenTelemetry providers: `tracerProvider`, `meterProvider`, `loggerProvider`
- `apiRequestCapture` — optional request capture for debugging
- `channelAllowlist` — MCP channel permission entries

This state is intentionally kept minimal. Token counts, model costs, and API durations live here because they are needed across all execution contexts (REPL, headless, subagents) without React.

---

## 9. API Communication Layer

### 9.1 `queryModel()` — The API Call Function

`src/services/api/claude.ts` contains `queryModel()`, an async generator that makes a single API call:

```
queryModel({
  messages, systemPrompt, thinkingConfig,
  tools, signal, options: {
    model, toolChoice, fallbackModel,
    isNonInteractiveSession, querySource,
    agents, allowedAgentTypes,
    maxOutputTokensOverride, mcpTools,
    effortValue, advisorModel,
    skipCacheWrite, taskBudget, ...
  }
})
```

Steps performed by `queryModel()`:

1. **Resolve model** — `getMainLoopModel()` with permission mode override (plan mode may switch to different model for 200k+ context)
2. **Compute beta headers** — collects all required beta strings: extended-thinking, tool-use, prompt-caching, interleaved-thinking, etc. Latches betas that must persist across turns (e.g., `tool-results-image-synthesis-beta-1`)
3. **Handle deferred tools** — if `ToolSearchTool` is in the tool set, may defer tool loading
4. **Build system prompt blocks** — splits system prompt into prefix/user portions, applies cache control markers for prompt caching (ephemeral, 5min, 1h TTL types)
5. **Normalize messages** — `normalizeMessagesForAPI()` strips internal fields, applies cache control to messages, handles thinking block rules
6. **Build tool schemas** — `toolToAPISchema()` for each tool
7. **Configure task budget** — if `taskBudget` present, adds `output_config.task_budget` to request (beta: task-budgets-2026-03-13)
8. **Make API call** — `client.beta.messages.stream(params)` 
9. **Parse streaming events** — converts raw Anthropic SDK stream events to internal `StreamEvent | AssistantMessage` types
10. **529 retry** — on 529 Overloaded, if `fallbackModel` is set, retries with fallback

### 9.2 `getAnthropicClient()` — Multi-Provider Factory

`src/services/api/client.ts` supports four API providers:

- **Direct Anthropic API**: `ANTHROPIC_API_KEY` → `new Anthropic({ apiKey, baseURL })` with optional OAuth token refresh
- **AWS Bedrock**: `AWS_REGION` + Bedrock credentials → `AnthropicBedrock` client
- **GCP Vertex AI**: `ANTHROPIC_VERTEX_PROJECT_ID` → `AnthropicVertex` client  
- **Azure AI Foundry**: Custom base URL with Azure credentials

All providers share the same interface. The factory also handles:
- mTLS certificate configuration
- HTTP proxy settings (CLAUDE_CODE_PROXY_URL)
- API key header naming (custom header support)
- OAuth token refresh (30-second cache with background refresh)

### 9.3 Prompt Caching Strategy

The API layer implements three caching scopes:

- **Ephemeral (default)**: `cache_control: { type: 'ephemeral' }` — 5-minute server-side TTL, minimal cost
- **Standard**: no cache control — not cached
- **1-hour**: `cache_control: { type: 'ephemeral', ttl: 3600 }` — 1-hour TTL, higher cache creation cost but cheaper on long-running sessions

Cache breakpoints are inserted at strategic positions:
1. End of the system prompt prefix (CLI identity block)
2. End of the user context (CLAUDE.md content, date, etc.)
3. Tail of the most recent few assistant messages

The `promptCacheBreakDetection` service monitors when cache breaks occur (detected by non-zero `cache_creation_input_tokens` in response usage) and uses this signal to detect context changes.

### 9.4 Message Normalization

Before sending to the API, messages go through `normalizeMessagesForAPI()` which:
- Converts internal `Message[]` types to `BetaMessageParam[]`
- Applies thinking block rules (thinking blocks must be preserved in trajectories)
- Handles `redacted_thinking` blocks
- Strips internal metadata fields (uuid, timestamps, etc.)
- Merges consecutive same-role messages if needed

---

## 10. Session Persistence

### 10.1 JSONL Transcript Files

Sessions are persisted as JSONL (newline-delimited JSON) files at:
```
~/.claude/projects/<normalized-cwd-hash>/<session-uuid>.jsonl
```

For subagents:
```
~/.claude/projects/<hash>/<session-uuid>/<agent-id>.jsonl
```

Each line is a JSON-encoded `Entry` object with a discriminated type field:

```typescript
type Entry =
  | { type: 'transcript'; message: SerializedMessage; uuid: string; ... }
  | { type: 'content_replacement'; ... }
  | { type: 'context_collapse_snapshot'; ... }
  | { type: 'context_collapse_commit'; ... }
  | { type: 'attribution_snapshot'; ... }
  | { type: 'file_history_snapshot'; ... }
```

### 10.2 Key Functions

`src/utils/sessionStorage.ts` exports:

- `recordTranscript(message, agentId?)` — appends a message to the JSONL file
- `loadTranscriptFromFile(filePath)` — reads and parses the JSONL file on resume
- `recordSidechainTranscript(message, agentId)` — for subagent transcripts
- `flushSessionStorage()` — ensures all pending writes complete before exit
- `writeAgentMetadata(agentId, metadata)` — writes per-agent metadata alongside transcript
- `clearAgentTranscriptSubdir(agentId)` — removes a completed agent's transcript dir

Session writes use `fs.appendFile` for atomicity — each message is a complete JSON line, so partial writes only corrupt the last line, which can be skipped on resume.

### 10.3 Config Persistence

`src/utils/config.ts` manages two config files:

- **Global config** (`~/.claude/.claude.json`): user preferences shared across all projects
- **Project config** (`~/.claude/projects/<hash>/.claude.json`): per-project settings, stored under a key derived from the normalized project CWD path

Config writes use a lock file (`<config>.lock`) to prevent concurrent write corruption when multiple claude instances run against the same project.

### 10.4 Session Resume

The `--resume <session-id>` flag triggers:
1. `loadTranscriptFromFile(sessionPath)` — load all messages
2. Reconstruct `Message[]` from transcript entries
3. Pass reconstructed messages as `params.messages` to `query()`
4. The conversation picks up exactly where it left off

The `--continue` flag auto-resumes the most recent session for the current project.

---

## 11. Permission System

### 11.1 Permission Modes

Defined in `src/types/permissions.ts` and `src/utils/permissions/PermissionMode.ts`:

```typescript
const PERMISSION_MODES = [
  'default',           // ask for write/exec permission
  'plan',              // read-only, no tool execution (except ExitPlanMode)
  'acceptEdits',       // auto-accept file edits, ask for bash
  'bypassPermissions', // never ask permission
  'dontAsk',           // never ask, but still track denials
  'auto',              // ANT-ONLY, feature-gated via TRANSCRIPT_CLASSIFIER
  'bubble',            // internal (not in external modes list)
] as const
```

External modes (exposed to SDK/API callers) are a subset:
```typescript
const EXTERNAL_PERMISSION_MODES = [
  'default', 'plan', 'acceptEdits', 'bypassPermissions', 'dontAsk'
]
```

The `auto` mode is internal ANT-only and uses a transcript classifier to automatically decide permission levels.

### 11.2 `CanUseToolFn` and `useCanUseTool`

`src/hooks/useCanUseTool.tsx` provides the `useCanUseTool` React hook, which returns a `CanUseToolFn`:

```typescript
type CanUseToolFn = (
  tool: Tool<any, any, any>,
  input: any,
  context: ToolUseContext,
) => Promise<PermissionResult>
```

The decision flow inside `CanUseToolFn`:

```
1. hasPermissionsToUseTool(tool, input, context, mode)
   → checks rules in order:
     a. Plan mode check — no tools except ExitPlanMode
     b. bypassPermissions/dontAsk mode — always allow
     c. Tool-specific permission rules (BashTool: command allowlist, etc.)
     d. Default mode: check global+project allowlists

2. If result === 'allow':
   → return { behavior: 'allow' }

3. If result === 'deny':
   → return { behavior: 'deny', reason: '...' }

4. If result === 'ask':
   → check coordinator/swarm permission delegation
   → check bash classifier (speculative auto-approval)
   → if interactive: show permission dialog to user
   → if non-interactive: auto-deny
```

### 11.3 Permission Rules Storage

Permission rules are stored in project config under `permissions.allow[]` and `permissions.deny[]`. Each rule is a glob pattern with a tool name prefix:
```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Read(*)", "Edit(src/**/*.ts)"],
    "deny": ["Bash(rm -rf *)"]
  }
}
```

### 11.4 Denial Tracking

`src/utils/permissions/denialTracking.ts` tracks which tools were denied in the current session. This feeds the `QueryEngine`'s `SDKPermissionDenial` events, allowing SDK callers to surface "Claude wanted to use X but was denied" information.

---

## 12. Bridge System (CCR — Claude Code Remote)

### 12.1 Purpose

The Bridge system connects a running Claude Code CLI to claude.ai, enabling:
- **Remote control**: Sending prompts from claude.ai to a local Claude Code instance
- **Background execution**: Running Claude Code agents in cloud containers
- **Live streaming**: Viewing agent output in real-time on claude.ai

### 12.2 Bridge Architecture

There are two bridge modes:

**Mode 1: Always-on bridge (REPL mode)**
The REPL has a persistent `ReplBridgeHandle` (via `replBridge.ts`) that maintains a WebSocket connection to the claude.ai ingress service. This is the "always-on" bridge enabled by `/remote-control` or the bridge toggle in the REPL footer.

**Mode 2: CCR worker mode (background tasks)**
`runBridgeLoop()` in `bridgeMain.ts` runs a polling loop that:
1. Registers the environment with the CCR API
2. Polls for work items
3. For each work item, spawns a new claude process
4. Manages the session lifecycle (spawn, monitor, stop)
5. Reports results back via the Bridge API

### 12.3 CCR Polling Loop (`src/bridge/bridgeMain.ts`)

```
runBridgeLoop(config, logger)
  │
  ├── registerWorker() — register environment with CCR
  ├── createBridgeApiClient() — HTTP client for CCR API
  ├── createCapacityWake() — capacity-aware sleep
  │
  └── while (!aborted):
      │
      ├── GET /v1/workers/poll  — poll for work item
      │
      ├── if work item received:
      │   ├── decodeWorkSecret(workData.secret)
      │   ├── validateBridgeId()
      │   ├── createSessionSpawner()
      │   ├── spawner.spawn(workData)
      │   │   └── launch claude process with:
      │   │       CLAUDE_CODE_REMOTE=true
      │   │       session-specific env vars
      │   │       worktree path (if SpawnMode=worktree)
      │   │
      │   ├── monitor session until done
      │   └── POST /v1/workers/stop-work (report result)
      │
      ├── exponential backoff on errors
      │   (initial: 2s, cap: 120s, give-up: 600s)
      │
      └── JWT token refresh scheduler (background)
```

### 12.4 `ReplBridgeHandle` — WebSocket Bridge

`src/bridge/replBridge.ts` provides `ReplBridgeHandle`:
- Maintains WebSocket connection to claude.ai ingress
- Handles inbound control messages (prompts, permission decisions)
- Sends SDK messages (assistant responses, tool results) back upstream
- Implements reconnection logic with exponential backoff
- Exposes bridge state to AppState (`replBridgeConnected`, `replBridgeSessionActive`, etc.)

### 12.5 Bridge Types (`src/bridge/types.ts`)

Key types:
- `BridgeConfig` — CCR connection settings (URLs, credentials)
- `WorkData` / `WorkResponse` — work item protocol
- `WorkSecret` — encrypted session credentials (JWT)
- `SessionHandle` — represents a running claude session
- `SpawnMode` — `'local' | 'worktree' | 'remote'`
- `BridgeApiClient` — interface for CCR HTTP API

---

## 13. MCP Integration

### 13.1 Overview

The Model Context Protocol (MCP) allows Claude to use tools provided by external processes. Claude Code supports four transport types for MCP servers: stdio (subprocess), SSE (server-sent events), HTTP (streamable HTTP), and WebSocket.

### 13.2 Client Management

`src/services/mcp/client.ts` provides:

- `connectToServer(config)` — creates an MCP client for a server configuration. Transport type is auto-detected from config.
- `getMcpToolsCommandsAndResources(client)` — queries a connected server for its tools, slash commands, and resources.
- `fetchToolsForClient(client)` — wraps each MCP tool as an `MCPTool` instance (a built-in `Tool` wrapper).

### 13.3 MCPTool Wrapper

`src/tools/MCPTool/MCPTool.ts` wraps an MCP server tool as a native `Tool<Input, Output, P>`:
- `call()` sends a `tools/call` MCP request and streams back results as `MCPProgress` events
- `inputSchema` is derived from the MCP tool's JSON schema
- `isConcurrencySafe()` checks the MCP tool's `readOnlyHint` annotation

### 13.4 Transport Configuration

MCP servers are configured in `.claude.json` or passed via `--mcp-config`:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["./my-mcp-server.js"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

Supported transport types:
- `stdio` — spawns subprocess, communicates via stdin/stdout
- `sse` — connects to SSE endpoint
- `http` — uses streamable HTTP transport (MCP 2025+)
- `ws` — WebSocket transport
- `sdk` — in-process transport for bundled servers

### 13.5 MCP Tools in AppState

Connected MCP clients and their tools are stored in `AppState.mcp`:
```typescript
mcp: {
  clients: MCPServerConnection[]   // connection status per server
  tools: Tool[]                    // assembled MCPTool instances
  commands: Command[]              // MCP server slash commands
  resources: Record<string, ServerResource[]>
  pluginReconnectKey: number       // incremented to force reconnect
}
```

`assembleToolPool()` in `src/tools.ts` merges `getAllBaseTools()` with `appState.mcp.tools` to produce the final tool list sent to the API.

### 13.6 MCP Elicitation

MCP 2025 introduced the "elicitation" feature — servers can request additional information from the user mid-tool-call. `src/services/mcp/elicitationHandler.ts` handles these requests by:
1. Pausing tool execution
2. Surfacing an elicitation dialog to the user via `AppState.elicitationRequest`
3. Waiting for user response
4. Continuing tool execution with the provided data

---

## 14. Subagent and Task Architecture

### 14.1 Task Types

```typescript
// src/Task.ts
type TaskType =
  | 'local_bash'         // shell command running in background
  | 'local_agent'        // in-process subagent
  | 'remote_agent'       // CCR remote subagent
  | 'in_process_teammate' // parallel agent for swarm coordination
  | 'local_workflow'     // workflow execution
  | 'monitor_mcp'        // MCP server monitoring
  | 'dream'              // autoDream speculative execution
```

### 14.2 AgentTool

`src/tools/AgentTool/AgentTool.tsx` is the primary mechanism for spawning subagents. Input parameters:

- `prompt: string` — the task for the subagent
- `run_in_background?: boolean` — fire-and-forget
- `isolation?: 'worktree' | 'remote'` — isolate in git worktree or CCR
- `cwd?: string` — working directory
- `model?: string` — override model for this subagent
- `name?: string` — register in agentNameRegistry for SendMessage routing
- `subagent_type?: string` — hint for agent type selection

### 14.3 `runAgent()` — Subagent Execution

`src/tools/AgentTool/runAgent.ts` handles the actual subagent execution:

```
runAgent(agentInput, toolUseContext)
  │
  ├── generateTaskId() — unique task ID
  ├── setAgentTranscriptSubdir() — isolate agent transcript
  │
  ├── fetchSystemPromptParts() — load agent system prompt
  ├── getUserContext() + getSystemContext() — context blocks
  │
  ├── initializeMCPServers() — agent-specific MCP servers
  │
  ├── createSubagentContext(params)
  │   └── builds ToolUseContext for the subagent:
  │       - inherits parent's tools (filtered)
  │       - new agentId
  │       - new abortController (linked to parent's)
  │       - cloned fileStateCache
  │       - clearSessionHooks() (subagent starts fresh)
  │
  ├── registerFrontmatterHooks() — parse CLAUDE.md hooks for this agent
  ├── executeSubagentStartHooks() — run session-start hooks
  │
  ├── query(subagentParams)         ← THE RECURSIVE CALL
  │   yields messages to parent's runAgent() generator
  │
  └── collect SDKMessages from yield stream
      return { output: collectedMessages }
```

Key design: `runAgent()` calls `query()` recursively. The subagent shares the same `queryLoop` code path as the main thread, but with its own `ToolUseContext` (different `agentId`, `abortController`, `fileStateCache`, etc.).

### 14.4 Subagent Isolation Modes

**In-process (default):** The subagent runs in the same Node.js process as the parent. This is efficient but means any uncaught error or infinite loop in the subagent could affect the parent.

**Worktree isolation (`isolation: 'worktree'`):** Creates a new git worktree at a temp path, spawns a new `claude` subprocess in that directory. The subprocess communicates back via the CCR bridge protocol. This provides true process isolation and prevents filesystem conflicts between parallel agents.

**Remote isolation (`isolation: 'remote'`):** Dispatches the subagent to a CCR cloud worker. Used for long-running background tasks that should outlive the CLI session.

### 14.5 Task State in AppState

```typescript
tasks: {
  [taskId: string]: TaskState  // live tasks, excluded from DeepImmutable
}
```

`TaskState` includes status tracking, output buffering, abort signal, and progress information. The `foregroundedTaskId` field controls which task's transcript is shown in the main view (the user can switch to viewing a subagent's progress).

---

## 15. Plugin and Skills System

### 15.1 Skills (Slash Commands)

Skills are slash commands (`/skill-name`) that can be:
- **Bundled TypeScript**: registered in `src/skills/bundled/index.ts` (verify, debug, remember, simplify, batch, stuck, etc.)
- **Markdown files**: any `.md` file in `~/.claude/skills/` or project-local `skills/` directory
- **Plugin-provided**: skills registered by installed plugins

Skills are surfaced to Claude as additional context or as executable commands via `SkillTool`.

### 15.2 Plugins

Plugins are marketplace-based extensions (`~/.claude/plugins/<plugin-name>/`). A plugin can provide:
- Additional MCP servers
- Custom slash commands
- Bundled skills
- Hook configurations
- Additional tools

`src/plugins/builtinPlugins.ts` registers built-in plugin definitions that are user-toggleable (appear in `/plugin` UI).

`src/services/plugins/PluginInstallationManager.ts` handles background installation from marketplace.

### 15.3 Hooks

Hooks are user-defined shell commands that run at specific lifecycle points. Defined in `.claude.json`:

```json
{
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "hooks": [{ "type": "command", "command": "echo pre-bash" }] }],
    "PostToolUse": [...],
    "PreSampling": [...],
    "PostSampling": [...],
    "SessionStart": [...],
    "Stop": [...]
  }
}
```

Hook implementations:
- `src/utils/hooks.ts` — `executeStopFailureHooks()`, `executePermissionDeniedHooks()`
- `src/utils/hooks/postSamplingHooks.ts` — `executePostSamplingHooks()`
- `src/query/stopHooks.ts` — `handleStopHooks()` called after each query turn

Hook execution is sandboxed — hooks run as shell subprocesses with the session's environment.

---

## 16. Terminal UI (React + Ink)

### 16.1 React + Ink Architecture

Claude Code renders its terminal UI using [Ink](https://github.com/vadimdemedes/ink) — a React renderer that outputs to a TTY. The component tree:

```
<App>                          [src/components/App.tsx]
  <FpsMetricsProvider>         — frame rate metrics
    <StatsProvider>            — session statistics
      <AppStateProvider>       — global app state context
        <REPL>                 [src/screens/REPL.tsx]
          <PromptInput>        — text input component
          <ConversationHistory> — message list
          <PermissionDialog>   — tool permission prompts
          <StatusBar>          — model, cost, mode indicators
          <TaskPanel>          — background task view
          ...
        </REPL>
      </AppStateProvider>
    </StatsProvider>
  </FpsMetricsProvider>
</App>
```

### 16.2 React Compiler

The compiled output uses React Compiler optimizations (visible as `_c` arrays from `react/compiler-runtime`). The compiler automatically memoizes components and hooks to avoid unnecessary re-renders. This is especially important in Claude Code because the conversation history can have hundreds of messages, and re-rendering the entire tree on every state update would be prohibitively slow.

### 16.3 REPL Component

`src/screens/REPL.tsx` is the most complex component (~200+ imports). It manages:

- **Query processing**: hooks up to the `query()` AsyncGenerator, drives streaming display
- **Tool permissions**: integrates `useCanUseTool()` for interactive permission dialogs
- **MCP connections**: `useManageMCPConnections()` hook manages client lifecycle
- **Task management**: tracks background tasks, foregrounds/backgrounds them
- **Bridge integration**: `initReplBridge()`, bridge state sync
- **Keyboard bindings**: escape for abort, Ctrl+C for exit, etc.
- **Session persistence**: calls `recordTranscript()` on every message
- **Agent spawning**: receives `AgentTool` outputs, registers task IDs

### 16.4 Ink Customizations

`src/ink.ts` wraps Ink's exports with:
- `ThemeProvider` wrapping for all renders
- Custom `Box` and `Text` components that apply theme colors
- Re-exports of Ink hooks: `useInput`, `useStdin`, `useFocus`, etc.

The theme system supports color configuration for different permission modes (plan mode shows blue indicators, bypass shows red, etc.).

### 16.5 Headless/Print Mode

In headless mode (`--print` or `-p`), the React/Ink rendering is bypassed entirely. Instead:
- `processUserInput()` handles the input
- `query()` is called directly without REPL
- Output is streamed via `src/cli/print.ts` (human-readable) or `src/cli/structuredIO.ts` (JSON/NDJSON)
- The `QueryEngine` class manages conversation state for SDK consumers

---

## 17. Analytics, Observability, and Feature Gates

### 17.1 GrowthBook Feature Gates

`src/services/analytics/growthbook.ts` provides runtime feature flags:

- `checkGate_CACHED_OR_BLOCKING(gateName)` — fast cached gate check
- `getFeatureValue_CACHED_MAY_BE_STALE(key, defaultValue)` — feature value with staleness caveat

GrowthBook gates are used for A/B testing, gradual rollouts, and ANT-internal features. Gates are fetched at startup and cached; the `_CACHED_MAY_BE_STALE` suffix is a code smell warning that the value may not be fresh.

### 17.2 OpenTelemetry

`src/bootstrap/state.ts` stores the OTEL providers:
- `tracerProvider: BasicTracerProvider` — distributed tracing
- `meterProvider: MeterProvider` — metrics
- `loggerProvider: LoggerProvider` — structured logging

These are initialized during `init()` and used throughout the codebase for performance profiling and error tracking.

### 17.3 Analytics Events

`src/services/analytics/index.ts` exports `logEvent(eventName, metadata)`. Key events:
- `tengu_tool_use` — every tool execution
- `tengu_auto_compact_succeeded` — context compaction
- `tengu_orphaned_messages_tombstoned` — streaming fallback
- `tengu_query_*` — query loop lifecycle events

The `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type suffix is a lint annotation preventing accidental logging of code or file paths (PII concerns).

### 17.4 Datadog Integration

`src/services/analytics/datadog.ts` provides Datadog APM integration for internal monitoring. Shutdown is coordinated via `shutdownDatadog()` called from the bridge loop and graceful shutdown handlers.

### 17.5 Startup Profiling

`src/utils/startupProfiler.ts` provides `profileCheckpoint(name)` for measuring startup performance. Checkpoints are emitted via analytics at session start.

---

## 18. Build System and Feature Flags

### 18.1 Bun Runtime and `bun:bundle`

Claude Code runs on Bun, not Node.js. The most important Bun-specific feature is the dead-code elimination system via `feature()` from `bun:bundle`:

```typescript
import { feature } from 'bun:bundle'

// At BUILD TIME, feature('MY_FEATURE') is replaced with true/false
// based on build configuration. The dead branch is eliminated.
if (feature('ANT_ONLY_FEATURE')) {
  // This entire block is removed from external builds
  const secretTool = require('./tools/secret/SecretTool.js').SecretTool
}
```

This enables Anthropic to maintain a single source tree for both internal (ANT) and external (Claude.ai Pro, API) builds. Features like `TRANSCRIPT_CLASSIFIER`, `ABLATION_BASELINE`, `DUMP_SYSTEM_PROMPT`, `REACTIVE_COMPACT`, `CONTEXT_COLLAPSE`, `HISTORY_SNIP`, `TOKEN_BUDGET`, `BG_SESSIONS`, etc. are conditionally compiled.

**Key feature flags (internal ANT-only):**
- `TRANSCRIPT_CLASSIFIER` — enables `auto` permission mode
- `ABLATION_BASELINE` — science experiment harness
- `DUMP_SYSTEM_PROMPT` — system prompt extraction tool
- `REACTIVE_COMPACT` — reactive context compaction
- `CONTEXT_COLLAPSE` — context collapse strategy
- `HISTORY_SNIP` — history snipping
- `TOKEN_BUDGET` — token budget tracking (+500k auto-continue)
- `BG_SESSIONS` — background session management
- `PROACTIVE` / `KAIROS` — proactive/assistant mode tools
- `AGENT_TRIGGERS` — cron scheduling tools
- `MONITOR_TOOL` — monitoring tool

**Key feature flags (external/gradual rollout):**
- `STREAMING_TOOL_EXECUTION` — execute tools while streaming
- `EXPERIMENTAL_SKILL_SEARCH` — ML-based skill discovery
- `TEMPLATES` — job template system
- `CACHED_MICROCOMPACT` — cache-aware microcompaction

### 18.2 `USER_TYPE` Conditional

Some ANT-only tools and code paths use `process.env.USER_TYPE === 'ant'` rather than Bun feature flags. This is a runtime check rather than a compile-time elimination:

```typescript
// src/tools.ts
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null
```

### 18.3 `MACRO.VERSION`

Bun inlines `MACRO.VERSION` at build time — this is the version string shown by `--version`. It avoids loading `package.json` at runtime for the fast-path.

### 18.4 Circular Dependency Management

The codebase has a notable comment pattern:
```typescript
// Types extracted to src/types/permissions.ts to break import cycles
import { type PermissionMode } from '../../types/permissions.js'
```

Key circular dependency breaks:
- `src/types/permissions.ts` — breaks the cycle between `PermissionMode.ts` and callers that use both permission types and tool types
- `src/types/tools.ts` — tool progress types extracted to avoid circular deps between `Tool.ts` and tool implementations
- `src/types/message.ts` — message types centralized to prevent cycles between `query.ts`, `Tool.ts`, and tool implementations

### 18.5 `lazySchema`

`src/utils/lazySchema.ts` provides a `lazySchema()` wrapper that defers Zod schema construction until first use. This avoids Zod initialization overhead during module import, important for startup time.

---

## 19. Key Design Decisions

### 19.1 AsyncGenerator Pattern Throughout

The entire query pipeline is built on AsyncGenerators. `query()`, `queryModel()`, `tool.call()`, and `runTools()` are all `AsyncGenerator` functions. This design:

- **Enables streaming**: callers receive messages as they become available without buffering
- **Enables backpressure**: the generator doesn't advance until the consumer reads
- **Simplifies abort**: an `AbortController` signal flows through all layers; when fired, the generators clean up and return
- **Enables middleware**: the REPL, QueryEngine, and print mode all consume the same `query()` generator with different consumers

The tradeoff is complexity: error handling across generator boundaries requires careful `yield*` and `try/finally` patterns.

### 19.2 Immutable State + External Pub/Sub

React's `useSyncExternalStore` is used to subscribe to the `Store<AppState>` rather than using `useState` or `useReducer`. This choice:
- **Allows state to be read outside React** (in tool implementations, hooks, services)
- **Prevents tearing** in concurrent React mode
- **Supports the onChange callback** for side effects (`onChangeAppState`)

The DeepImmutable constraint on AppState prevents accidental mutation and makes state transitions explicit (always `setState(prev => ({ ...prev, field: newValue }))`).

### 19.3 Dual Execution Paths

The codebase maintains two parallel execution paths for the same underlying `query()` logic:

- **Interactive (REPL)**: React/Ink UI, streaming renders, interactive permission dialogs
- **Headless/SDK**: JSON/NDJSON to stdout, non-interactive (auto-deny permissions), `QueryEngine` wrapper

The convergence point is `query()` — both paths call the same function with the same `QueryParams`. The difference is purely in how the yielded messages are consumed and rendered.

### 19.4 Compaction as a Recovery Strategy

Context compaction (reducing conversation history to fit the context window) is treated as a first-class recovery mechanism, not just a maintenance task:

- **Proactive**: `autoCompact()` fires when context exceeds a threshold (e.g., 90% of window)
- **Reactive**: `reactiveCompact()` fires on `prompt_too_long` API errors (feature-gated)
- **Snip**: `snipCompact()` removes middle-section messages (less aggressive)
- **Microcompact**: inline compression of repetitive tool result patterns
- **Context collapse**: segment-level collapse with summary messages

The key insight is that compaction runs a **forked subagent** to summarize the conversation — so compaction itself uses the LLM. The `forkedAgent.ts` pattern ensures the summarizer shares the parent's prompt cache to minimize cost.

### 19.5 The `ToolUseContext` God Object

Every tool call receives the full `ToolUseContext`. While this is a "god object" anti-pattern, it's deliberate: tools often need access to diverse context (current messages, permission mode, MCP connections, subagent spawning capability, notification system). The alternative — passing individual parameters — would result in tools with 15+ parameters, which is worse.

The pattern is documented and accepted with discipline: `ToolUseContext` is defined centrally in `src/Tool.ts` and new fields require justification.

### 19.6 Session Storage as Append-Only Log

JSONL files are append-only. This design:
- **Never corrupts existing data** — a crash during write only affects the last line
- **Enables efficient resume** — read the file once at startup
- **Enables cheap streaming writes** — no seek or re-read needed

The tradeoff is that space is never reclaimed within a session file. Old sessions accumulate in `~/.claude/projects/`.

### 19.7 Feature Flags as First-Class Architecture

The `feature()` gate system is deeply integrated: ~30 feature flags throughout the codebase, controlling everything from entire subsystems (reactive compact, context collapse) to individual optimizations (cached microcompact). This enables:
- Internal experiment code to coexist with external stable code
- Gradual rollout without separate branches
- Scientific ablation baselines (controlled experiment arm)

The tradeoff is complexity: understanding the full system requires knowing which gates are on, and `_CACHED_MAY_BE_STALE` gate checks mean the system can make decisions based on stale data.

---

## 20. Component Interaction Diagrams

### 20.1 Message Processing Pipeline

```
User input (keyboard / CLI arg)
         │
         ▼
src/utils/processUserInput/processUserInput.ts
  enriches with:
  ├── slash command resolution
  ├── image attachment processing
  ├── @mention file attachments
  └── queued command dispatch
         │
         ▼
src/query.ts:query(params)
  ├── yield RequestStartEvent        → UI: show spinner
  │
  ├── [compaction pipeline]
  │   ├── applyToolResultBudget()   → truncate large tool results
  │   ├── snipCompact()             → remove middle messages
  │   ├── microcompact()            → inline compression
  │   ├── contextCollapse()         → segment summarization
  │   └── autoCompact()             → full conversation summary
  │
  ├── yield StreamEvent(text_delta) → UI: stream text to screen
  ├── yield StreamEvent(thinking)   → UI: show thinking block
  ├── yield AssistantMessage        → UI: complete assistant turn
  │                                 → sessionStorage.recordTranscript()
  │
  ├── [tool execution]
  │   ├── runTools(toolUseBlocks)
  │   │   ├── yield ProgressMessage → UI: show tool progress
  │   │   └── yield UserMessage     → contains tool_result
  │   └──                           → sessionStorage.recordTranscript()
  │
  └── return Terminal               → REPL: update state, re-enable input
```

### 20.2 Tool Execution Flow

```
tool_use block from API response
         │
         ▼
src/services/tools/toolOrchestration.ts:runTools()
  │
  ├── partitionToolCalls()
  │   → [{isConcurrencySafe: true, blocks: [A,B,C]},
  │      {isConcurrencySafe: false, blocks: [D]},
  │      {isConcurrencySafe: true, blocks: [E]}]
  │
  ├── runToolsConcurrently([A,B,C])     — parallel
  │   └── for each: runToolUse(block)
  │
  ├── runToolsSerially([D])             — sequential
  │   └── runToolUse(block)
  │
  └── runToolsConcurrently([E])         — parallel

src/services/tools/toolExecution.ts:runToolUse(block, ...)
  │
  ├── canUseTool(tool, input, ctx)
  │   ├── hasPermissionsToUseTool()    — check rules
  │   ├── [if ask] show dialog         — interactive only
  │   └── return PermissionResult
  │
  ├── [if denied] yield error tool_result, return
  │
  ├── tool.call(input, ctx)            — actual tool execution
  │   yields ToolProgress events:
  │   ├── BashProgress: { stdout lines }
  │   ├── AgentToolProgress: { subagent messages }
  │   └── MCPProgress: { mcp events }
  │
  ├── logEvent('tengu_tool_use')       — analytics
  │
  └── yield UserMessage (tool_result)
```

### 20.3 Startup Flow with Timing

```
Binary invocation
    0ms ──── cli.tsx loaded
             ├── env setup (COREPACK, NODE_OPTIONS)
             ├── --version fast-path check
    1ms ──── profileCheckpoint('cli_entry')
             ├── dynamic import main.tsx
    ?ms ──── profileCheckpoint('main_loaded')
             ├── Commander arg parsing
    ?ms ──── profileCheckpoint('pre_action')
             ├── init() [memoized first-call init]
             │   ├── enableConfigs()
             │   ├── OAuth init
             │   ├── LSP server manager init
             │   └── ...
    ?ms ──── profileCheckpoint('init_done')
             ├── buildInitialAppState()
             ├── showSetupScreens() [if needed]
    ?ms ──── profileCheckpoint('pre_render')
             ├── launchRepl()
             │   ├── import App, REPL (deferred)
             │   └── root.render(<App><REPL/></App>)
    ?ms ──── [first React render]
             ├── startDeferredPrefetches()
             │   ├── loadMemoryPrompt()   — CLAUDE.md
             │   ├── analytics flush
             │   └── MCP connection init
             └── root.waitUntilExit()    — blocks until exit
```

### 20.4 Bridge Communication Flow

```
User on claude.ai                    Claude Code CLI (local machine)
     │                                         │
     │ ──── WebSocket connect ────────────────▶ │
     │                            replBridge.ts │
     │                            WebSocket established
     │                                         │
     │ ──── inbound prompt message ───────────▶ │
     │                            inboundMessages.ts
     │                            → queue message in REPL
     │                                         │
     │                            query() runs
     │                            ├── API calls
     │                            └── yields messages
     │                                         │
     │ ◀──── SDK messages streaming ──────────── │
     │                            replBridgeMessaging.ts
     │ display in claude.ai UI                  │
     │                                         │
     │ ──── permission decision ──────────────▶ │
     │                            bridgePermissionCallbacks.ts
     │                            → resolves permission dialog
     │                                         │
```

### 20.5 Plugin/Extensibility Architecture

```
Extension Points:

1. MCP Servers (most powerful)
   └── Config in .claude.json mcpServers
   └── Provides: custom tools, slash commands, resources
   └── Transport: stdio, SSE, HTTP, WebSocket

2. Plugins (marketplace-based)
   └── Installed to ~/.claude/plugins/<name>/
   └── Provides: MCP servers, skills, hooks, tools
   └── Managed by PluginInstallationManager

3. Skills (markdown or TypeScript)
   └── Source: ~/.claude/skills/, project skills/, bundled
   └── Exposed as /skill-name slash commands
   └── Surfaced to Claude via SkillTool or attachment

4. Hooks (shell commands)
   └── Config in .claude.json hooks
   └── Lifecycle points: PreToolUse, PostToolUse,
       PreSampling, PostSampling, SessionStart, Stop
   └── Runs as shell subprocess

5. Agent Definitions
   └── YAML/JSON files in agent definitions directory
   └── Pre-configured AgentTool invocations
   └── Loaded as AgentDefinition[]

Permission flow for each extension point:
  Plugin MCP tools → MCP channel allowlist → canUseTool()
  Custom hooks → hook matcher → shell execution
  Skills → SkillTool.isEnabled() → direct execution
```

### 20.6 Layer Dependency Diagram

```
Presentation   ┌──────────────────┐
(React/Ink)    │  REPL.tsx        │──reads──▶ AppState
               │  Components/     │──calls──▶ query()
               └──────┬───────────┘
                       │ submits messages
                       ▼
Query          ┌──────────────────────────────────────┐
               │  query.ts / QueryEngine.ts           │
               │  ├── src/query/config.ts             │
               │  ├── src/query/deps.ts               │
               │  └── src/query/stopHooks.ts          │
               └──┬───────────────────────────────────┘
                  │ calls                │ dispatches tools
                  ▼                      ▼
API Service   ┌──────────────┐   Tool   ┌──────────────────────────┐
              │  claude.ts   │   Layer  │  toolOrchestration.ts    │
              │  client.ts   │          │  toolExecution.ts        │
              └──┬───────────┘          │  tools/*.ts              │
                 │ HTTP/stream          └──┬───────────────────────┘
                 ▼                         │ spawns
              Anthropic API     Subagents ┌▼──────────────────────┐
              (direct/          & MCP     │  AgentTool/runAgent.ts │
               Bedrock/                  │  services/mcp/client.ts│
               Vertex)                   └───────────────────────┘

State         ┌──────────────────┐   ┌──────────────────┐
              │  AppState.tsx    │   │  bootstrap/state │
              │  (React context) │   │  (module global) │
              └──────────────────┘   └──────────────────┘

Persistence   ┌──────────────────┐   ┌──────────────────┐
              │  sessionStorage  │   │  config.ts       │
              │  (JSONL files)   │   │  (JSON files)    │
              └──────────────────┘   └──────────────────┘
```

---

## 21. Essential Files Reference

The following files are the minimum required reading to understand Claude Code's architecture:

### Core Execution

| File | Purpose | Lines |
|------|---------|-------|
| `src/entrypoints/cli.tsx` | Binary entry point, fast-paths | ~90 |
| `src/query.ts` | Core agentic query loop | ~1730 |
| `src/QueryEngine.ts` | SDK/headless conversation wrapper | ~400 |
| `src/Tool.ts` | Tool type definitions, ToolUseContext | ~200 |

### Startup and Initialization

| File | Purpose | Lines |
|------|---------|-------|
| `src/entrypoints/init.ts` | One-time process initialization | ~150 |
| `src/replLauncher.tsx` | React/Ink render bootstrap | ~25 |
| `src/interactiveHelpers.tsx` | renderAndRun(), showSetupScreens() | ~100 |
| `src/components/App.tsx` | Top-level React context tree | ~100 |

### State Management

| File | Purpose | Lines |
|------|---------|-------|
| `src/state/store.ts` | Generic pub/sub store primitive | ~35 |
| `src/state/AppStateStore.ts` | AppState type definition | ~250 |
| `src/state/AppState.tsx` | React provider + useSyncExternalStore | ~100 |
| `src/bootstrap/state.ts` | Process-level global singleton | ~200 |
| `src/state/onChangeAppState.ts` | State change side-effects | ~100 |

### API Layer

| File | Purpose | Lines |
|------|---------|-------|
| `src/services/api/claude.ts` | queryModel() API call function | ~1500 |
| `src/services/api/client.ts` | Multi-provider Anthropic client factory | ~300 |
| `src/services/api/errors.ts` | API error classification | ~100 |

### Tool System

| File | Purpose | Lines |
|------|---------|-------|
| `src/tools.ts` | Built-in tool registry | ~150 |
| `src/services/tools/toolOrchestration.ts` | Concurrent/serial tool batching | ~150 |
| `src/services/tools/toolExecution.ts` | Single tool execution | ~300 |
| `src/tools/AgentTool/runAgent.ts` | Subagent execution | ~400 |

### Session and Config

| File | Purpose | Lines |
|------|---------|-------|
| `src/utils/sessionStorage.ts` | JSONL transcript persistence | ~500 |
| `src/utils/config.ts` | Global/project config read/write | ~300 |

### Permission System

| File | Purpose | Lines |
|------|---------|-------|
| `src/types/permissions.ts` | PermissionMode type definitions | ~50 |
| `src/utils/permissions/PermissionMode.ts` | Mode config and utilities | ~142 |
| `src/hooks/useCanUseTool.tsx` | Permission decision hook | ~300 |
| `src/utils/permissions/permissions.ts` | Core permission rule evaluation | ~400 |

### Bridge and MCP

| File | Purpose | Lines |
|------|---------|-------|
| `src/bridge/bridgeMain.ts` | CCR polling loop | ~500 |
| `src/bridge/replBridge.ts` | WebSocket bridge to claude.ai | ~300 |
| `src/bridge/types.ts` | Bridge type definitions | ~100 |
| `src/services/mcp/client.ts` | MCP client management | ~600 |
| `src/services/mcp/types.ts` | MCP type definitions | ~100 |

### Context and Compaction

| File | Purpose | Lines |
|------|---------|-------|
| `src/services/compact/autoCompact.ts` | Auto-compaction trigger | ~300 |
| `src/services/compact/compact.ts` | Compaction algorithm | ~400 |
| `src/utils/forkedAgent.ts` | CacheSafeParams + forked subagent helper | ~100 |
| `src/context.ts` | getSystemContext(), getUserContext() | ~200 |

### Query Infrastructure

| File | Purpose | Lines |
|------|---------|-------|
| `src/query/config.ts` | buildQueryConfig() snapshot | ~100 |
| `src/query/deps.ts` | Injectable query dependencies | ~100 |
| `src/query/stopHooks.ts` | Post-sampling stop hook handling | ~150 |
| `src/query/tokenBudget.ts` | Token budget tracking | ~100 |

---

*Document generated by Architecture Agent analysis of commit `1becaba`.*  
*File path: `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/01-architecture.md`*

---
