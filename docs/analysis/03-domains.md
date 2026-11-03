# Claude Code — Business Domain Map

**Analysis Date:** 2026-04-03  
**Codebase Root:** `src/`  
**Total Source Directories Surveyed:** 35+  

---

## Table of Contents

1. [Overview](#overview)
2. [Domain Catalog](#domain-catalog)
   - [Domain 01: Conversation Engine (Core Agentic Loop)](#domain-01-conversation-engine-core-agentic-loop)
   - [Domain 02: Tool Framework & Ecosystem](#domain-02-tool-framework--ecosystem)
   - [Domain 03: File Operations](#domain-03-file-operations)
   - [Domain 04: Shell & Bash Execution](#domain-04-shell--bash-execution)
   - [Domain 05: Agent Orchestration & Swarms](#domain-05-agent-orchestration--swarms)
   - [Domain 06: MCP (Model Context Protocol)](#domain-06-mcp-model-context-protocol)
   - [Domain 07: Session Management & Storage](#domain-07-session-management--storage)
   - [Domain 08: Authentication & Security](#domain-08-authentication--security)
   - [Domain 09: Permissions & Safety](#domain-09-permissions--safety)
   - [Domain 10: Context Management & Compaction](#domain-10-context-management--compaction)
   - [Domain 11: Memory System](#domain-11-memory-system)
   - [Domain 12: Plugin & Extension System](#domain-12-plugin--extension-system)
   - [Domain 13: UI/UX Terminal (Ink)](#domain-13-uiux-terminal-ink)
   - [Domain 14: Configuration System](#domain-14-configuration-system)
   - [Domain 15: Analytics & Telemetry](#domain-15-analytics--telemetry)
   - [Domain 16: Remote Bridge & Desktop Integration](#domain-16-remote-bridge--desktop-integration)
   - [Domain 17: LSP & Code Intelligence](#domain-17-lsp--code-intelligence)
   - [Domain 18: Version Control (Git)](#domain-18-version-control-git)
   - [Domain 19: Cost & Billing](#domain-19-cost--billing)
   - [Domain 20: Auto-Update System](#domain-20-auto-update-system)
   - [Domain 21: Task Management](#domain-21-task-management)
   - [Domain 22: Hook System](#domain-22-hook-system)
   - [Domain 23: IDE Integration](#domain-23-ide-integration)
   - [Domain 24: Computer Use](#domain-24-computer-use)
   - [Domain 25: Voice Input](#domain-25-voice-input)
   - [Domain 26: GitHub App & CI/CD Integration](#domain-26-github-app--cicd-integration)
   - [Domain 27: Skill System](#domain-27-skill-system)
   - [Domain 28: Web & Search Tools](#domain-28-web--search-tools)
   - [Domain 29: Bootstrap & Global State](#domain-29-bootstrap--global-state)
   - [Domain 30: Buddy / Companion System](#domain-30-buddy--companion-system)
3. [Domain Interaction Map](#domain-interaction-map)
4. [Cross-Cutting Concerns](#cross-cutting-concerns)
5. [Domain Registry](#domain-registry)

---

## Overview

Claude Code is a terminal-based AI coding assistant built on the Anthropic API. Its architecture is highly modular, organized around a central agentic query loop that dispatches to a rich ecosystem of tools, agents, MCP servers, and plugins. The codebase is written in TypeScript (compiled with Bun), uses React/Ink for terminal UI rendering, Zod for schema validation, and @anthropic-ai/sdk for API access.

The application spans approximately 30 distinct business domains, ranging from low-level terminal rendering to high-level multi-agent orchestration. The central abstraction layer is the `Tool` type (`src/Tool.ts`) and `QueryEngine`/`query.ts`, which together implement the agentic conversation loop.

**Key architectural facts:**
- Entry point: `src/entrypoints/cli.tsx`
- Core loop: `src/query.ts` + `src/QueryEngine.ts`
- Tool contract: `src/Tool.ts` (defines the `Tool<Input, Output, Progress>` generic interface)
- App state: `src/state/AppStateStore.ts` (React context + synchronous store)
- Feature flags: GrowthBook via `src/services/analytics/growthbook.ts`
- Dead code elimination: `bun:bundle` `feature()` calls gate ant-only vs external builds

---

## Domain Catalog

---

### Domain 01: Conversation Engine (Core Agentic Loop)

**Purpose and Scope**

The Conversation Engine is the heart of Claude Code. It manages the full lifecycle of a user interaction: accepting a prompt, calling the Anthropic API, dispatching tool uses, handling tool results, compacting context, and producing a response. This domain is the primary orchestrator — all other domains are invoked by it.

Two complementary entry points exist:

1. `query.ts` — the REPL-oriented streaming generator that powers the interactive terminal. It yields `StreamEvent` objects consumed by the UI.
2. `QueryEngine.ts` — a class-based wrapper used by the SDK/headless path. `QueryEngine.submitMessage()` accepts a prompt and yields `SDKMessage` objects over an async generator.

Both paths delegate to the same `query()` function internally. The flow is:

1. User message is normalized and appended to `mutableMessages`.
2. The system prompt is assembled from CLAUDE.md files, plugin instructions, and dynamic context.
3. An API call is made via `src/services/api/claude.ts` using the `@anthropic-ai/sdk`.
4. Streamed content blocks are processed. Text blocks are yielded immediately. Tool-use blocks trigger tool dispatch.
5. Each tool call goes through the permission system (`canUseTool`), then the tool's `call()` method.
6. Tool results are appended as user messages and the loop continues.
7. When the model emits a stop token, the loop ends. If context is too full, auto-compaction fires.

**Key Source Files**

- `src/query.ts` — Core streaming generator, tool dispatch, retry logic
- `src/QueryEngine.ts` — Class wrapper for SDK/headless usage
- `src/Tool.ts` — `Tool` and `ToolUseContext` type definitions (the central contract)
- `src/services/api/claude.ts` — Raw Anthropic API calls, streaming, token accounting
- `src/services/api/client.ts` — Axios HTTP client with auth headers
- `src/utils/queryContext.ts` — System prompt assembly
- `src/utils/queryHelpers.ts` — Message normalization, orphaned permission handling
- `src/utils/processUserInput/processUserInput.ts` — User message preprocessing (attachments, at-mentions)
- `src/utils/messages.ts` — Message creation/manipulation utilities
- `src/constants/prompts.ts` — Default system prompt
- `src/context.ts` — System/user context builders

**Main Types/Interfaces**

- `Tool<Input, Output, Progress>` in `src/Tool.ts` — The generic tool contract
- `ToolUseContext` in `src/Tool.ts` — The runtime context passed to every tool call
- `QueryEngineConfig` in `src/QueryEngine.ts` — Configuration for a conversation session
- `Message`, `AssistantMessage`, `UserMessage` in `src/types/message.ts`
- `StreamEvent` in `src/types/message.ts`

**Interactions with Other Domains**

- Calls into every Tool domain (File Operations, Bash, Agent Orchestration, MCP, etc.)
- Uses Permission & Safety domain to gate tool calls
- Uses Context Management domain to determine when to compact
- Uses Session Management domain to persist transcripts
- Uses Analytics domain to emit turn-level events
- Uses Memory System domain to inject CLAUDE.md content

**Complexity:** Very High

---

### Domain 02: Tool Framework & Ecosystem

**Purpose and Scope**

The Tool Framework defines the universal contract that every capability in Claude Code must implement. It is the lingua franca between the conversation engine and the 36+ built-in tools. The framework provides:

- The `Tool<Input, Output, Progress>` generic interface (650+ lines in `src/Tool.ts`)
- `buildTool()` factory function that fills in safe defaults
- Tool collection type (`Tools = readonly Tool[]`)
- Tool lookup utilities (`findToolByName`, `toolMatchesName`)
- Progress reporting types for streaming UI updates

Every tool follows the same pattern:
1. A Zod input schema for validation and prompt generation
2. A `call()` method returning `ToolResult<Output>`
3. A `checkPermissions()` method returning `PermissionResult`
4. React render methods for terminal UI (`renderToolUseMessage`, `renderToolResultMessage`, etc.)
5. A `prompt()` method generating the tool's system prompt description

The complete set of built-in tools includes:

**File I/O:** FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool, NotebookEditTool  
**Execution:** BashTool, PowerShellTool, REPLTool  
**Agent/Orchestration:** AgentTool, TeamCreateTool, TeamDeleteTool, SendMessageTool  
**Plan Mode:** EnterPlanModeTool, ExitPlanModeV2Tool  
**Worktree:** EnterWorktreeTool, ExitWorktreeTool  
**MCP:** MCPTool, McpAuthTool, ListMcpResourcesTool, ReadMcpResourceTool  
**LSP:** LSPTool  
**Config:** ConfigTool  
**Web:** WebFetchTool, WebSearchTool  
**Memory/Docs:** BriefTool (KAIROS), SyntheticOutputTool  
**Tasks:** TaskCreateTool, TaskGetTool, TaskListTool, TaskStopTool, TaskUpdateTool  
**Scheduling:** CronCreateTool, CronDeleteTool, CronListTool  
**Teams/Swarms:** TeamCreateTool, TeamDeleteTool  
**Misc:** AskUserQuestionTool, SleepTool, ToolSearchTool, TodoWriteTool, SkillTool, RemoteTriggerTool  

**Key Source Files**

- `src/Tool.ts` — The complete Tool interface and `buildTool()` factory
- `src/tools/` — One subdirectory per tool, each containing: `ToolName.ts`, `UI.tsx`, `prompt.ts`, `constants.ts`
- `src/hooks/useMergedTools.ts` — Assembles the active tool set from built-ins + MCP + plugins
- `src/hooks/useCanUseTool.tsx` — Gate function passed to `ToolUseContext`

**Main Types/Interfaces**

- `Tool<Input, Output, P>` — Generic tool interface with 40+ methods/properties
- `ToolDef` — Partial tool definition accepted by `buildTool()`
- `ToolResult<T>` — Return type from `tool.call()`
- `ToolUseContext` — Rich runtime context including app state, abort controller, permission context
- `ToolPermissionContext` — Permission rules and mode passed to all permission checks
- `ValidationResult` — Input validation result

**Interactions with Other Domains**

- Every tool calls into File Operations, Bash Execution, LSP, MCP, or Agent Orchestration domains
- Permission & Safety domain gates every tool call
- Analytics domain receives tool call events
- UI/UX Terminal domain renders tool use/result messages
- Hook System domain fires PreToolUse/PostToolUse hooks around each call

**Complexity:** Very High

---

### Domain 03: File Operations

**Purpose and Scope**

The File Operations domain provides all file system interaction capabilities exposed to the model. It is one of the most security-sensitive domains, as it handles read, write, and edit operations on arbitrary files, with access controls and content validation.

**FileReadTool** reads files with line number formatting, handles images (converts to base64), enforces size limits, and integrates with the FileStateCache for change detection. It supports partial reads via offset/limit parameters.

**FileWriteTool** creates or overwrites entire files. It validates write permissions, tracks the written content in FileStateCache, and records file history snapshots for undo.

**FileEditTool** performs surgical string replacement edits within files. It uses a strict old-string/new-string replacement model, validates that the old string exists uniquely, and handles edge cases like missing files and encoding issues. It records diffs for the UI and integrates with FileHistory for undo.

**GlobTool** wraps the `fast-glob` library with configurable result limits and workspace-aware defaults.

**GrepTool** wraps `ripgrep` for fast regex content search across files.

**NotebookEditTool** edits Jupyter notebook cells (`.ipynb`) with cell-ID-based addressing.

Supporting infrastructure:
- `FileStateCache` (LRU cache tracking file content seen by the model — critical for preventing stale edits)
- `FileHistory` (per-session backup snapshots enabling undo)
- `FileHistoryState` (message-level snapshot for transcript-based undo)

**Key Source Files**

- `src/tools/FileReadTool/FileReadTool.ts`
- `src/tools/FileWriteTool/FileWriteTool.ts`
- `src/tools/FileEditTool/FileEditTool.ts`
- `src/tools/GlobTool/GlobTool.ts`
- `src/tools/GrepTool/GrepTool.ts`
- `src/tools/NotebookEditTool/NotebookEditTool.ts`
- `src/utils/fileStateCache.ts` — LRU cache of file states seen by the model
- `src/utils/fileHistory.ts` — Backup snapshot and undo mechanism
- `src/utils/file.ts` — Low-level file I/O helpers
- `src/tools/FileReadTool/imageProcessor.ts` — Image file handling
- `src/tools/FileReadTool/limits.ts` — Token/size limits for file reading
- `src/tools/FileEditTool/utils.ts` — Old/new string replacement logic
- `src/utils/git/gitignore.ts` — .gitignore-aware filtering

**Main Types/Interfaces**

- `FileState` in `src/utils/fileStateCache.ts` — Cached content + metadata
- `FileStateCache` — LRU cache class
- `FileHistoryBackup`, `FileHistorySnapshot` in `src/utils/fileHistory.ts`
- `FileHistoryState` in `src/utils/fileHistory.ts` — Per-message snapshot for undo

**Interactions with Other Domains**

- Permission & Safety domain validates path access and filesystem permissions
- Session Management domain records file edits in the transcript
- Analytics domain emits file operation events
- Git/VCS domain provides gitignore filtering
- IDE Integration domain notifies VS Code of file changes via MCP

**Complexity:** High

---

### Domain 04: Shell & Bash Execution

**Purpose and Scope**

The Shell/Bash domain handles safe, sandboxed execution of shell commands. It is the highest-risk tool surface and has the most extensive security layering in the codebase.

**BashTool** executes arbitrary shell commands. Its execution path includes:
1. **Input parsing**: `src/utils/bash/bashParser.ts` uses tree-sitter to parse bash ASTs, detecting dangerous patterns, output redirections, and semantic classifications.
2. **Security validation**: `src/tools/BashTool/bashSecurity.ts` blocks dangerous patterns (rm -rf /, etc.), `src/tools/BashTool/pathValidation.ts` validates paths stay within allowed directories, `src/tools/BashTool/sedValidation.ts` validates sed commands.
3. **Permission check**: `src/tools/BashTool/bashPermissions.ts` maps commands to permission rules using the classifier or allow/deny lists.
4. **Sandbox decision**: `src/tools/BashTool/shouldUseSandbox.ts` determines if the command should run in a macOS sandbox.
5. **Execution**: Commands run via `src/utils/Shell.ts` with configurable timeouts, abort signals, and output streaming.
6. **PowerShellTool** provides the Windows equivalent of BashTool with Windows-specific security rules.

The bash parsing infrastructure is sophisticated: `src/utils/bash/ast.ts` provides AST traversal, `src/utils/bash/commands.ts` detects output redirections and pipe semantics, and `src/utils/bash/registry.ts` maintains a registry of known shell commands with their argument specifications (allowing the auto-mode classifier to build per-argument tool descriptions).

**Key Source Files**

- `src/tools/BashTool/BashTool.tsx` — Main BashTool implementation
- `src/tools/BashTool/bashSecurity.ts` — Pattern-based security blocking
- `src/tools/BashTool/bashPermissions.ts` — Permission rule matching for commands
- `src/tools/BashTool/pathValidation.ts` — Path containment checks
- `src/tools/BashTool/shouldUseSandbox.ts` — Sandbox decision logic
- `src/tools/BashTool/commandSemantics.ts` — Command classification (read/write/search)
- `src/tools/BashTool/sedValidation.ts` — sed command safety analysis
- `src/tools/PowerShellTool/` — Windows PowerShell equivalent
- `src/utils/Shell.ts` — Shell execution primitives
- `src/utils/ShellCommand.ts` — Subprocess management
- `src/utils/bash/bashParser.ts` — Tree-sitter AST bash parser
- `src/utils/bash/ast.ts` — AST traversal utilities
- `src/utils/bash/commands.ts` — Command extraction and classification
- `src/utils/bash/registry.ts` — Known command specifications
- `src/utils/bash/treeSitterAnalysis.ts` — Semantic analysis over the AST
- `src/utils/sandbox/sandbox-adapter.ts` — macOS sandbox integration

**Main Types/Interfaces**

- `ParsedCommand` in `src/utils/bash/ParsedCommand.ts`
- `ShellSnapshot` in `src/utils/bash/ShellSnapshot.ts`
- `BashProgress` in `src/types/tools.ts`

**Interactions with Other Domains**

- Permission & Safety domain gates every bash command
- Analytics domain emits bash execution events and classifier results
- Task Management domain creates background shell tasks
- Session Management domain records bash output in transcript

**Complexity:** Very High

---

### Domain 05: Agent Orchestration & Swarms

**Purpose and Scope**

This domain implements Claude Code's multi-agent capabilities — the ability to spawn, coordinate, and manage multiple AI agent instances working in parallel or in series. It is one of the most architecturally complex domains.

**AgentTool** is the primary entry point. It can spawn a subagent in several modes:
- **Fork (cache-sharing)**: Creates a child agent with the same system prompt and a forked message history, sharing the parent's prompt cache for efficiency.
- **Background async**: Spawns a `LocalAgentTask` that runs independently without blocking the parent.
- **Resume**: Restarts a previously backgrounded agent from its saved transcript.

**Built-in agent definitions** (`src/tools/AgentTool/built-in/`):
- `generalPurposeAgent.ts` — Default background agent
- `exploreAgent.ts` — Codebase exploration specialist
- `planAgent.ts` — Plan mode implementation agent
- `verificationAgent.ts` — Automated testing/verification agent
- `claudeCodeGuideAgent.ts` — Documentation assistant

**Coordinator Mode** (`src/coordinator/coordinatorMode.ts`): A special mode where the main model acts as a coordinator, spawning worker agents via `TeamCreateTool`. The coordinator gets a different user context explaining the multi-agent protocol.

**Agent Swarms** (`src/utils/swarm/`): The swarm system manages teams of in-process teammates. Each teammate runs in the same Node.js process using `AsyncLocalStorage` for isolation. Backends include:
- `InProcessBackend.ts` — In-process teammate (same Node.js process)
- `ITermBackend.ts` — iTerm2 pane-based teammate
- `TmuxBackend.ts` — tmux pane-based teammate

**Team files** (`src/utils/swarm/teamHelpers.ts`) persist team definitions to `.claude/teams/<name>.json`, enabling agent coordination through shared task files.

**Agent Memory** (`src/tools/AgentTool/agentMemory.ts`, `agentMemorySnapshot.ts`): Agents can snapshot and restore their context across compaction boundaries.

**Key Source Files**

- `src/tools/AgentTool/AgentTool.tsx` — AgentTool implementation
- `src/tools/AgentTool/runAgent.ts` — Core agent execution loop
- `src/tools/AgentTool/forkSubagent.ts` — Cache-sharing fork logic
- `src/tools/AgentTool/resumeAgent.ts` — Resume backgrounded agent
- `src/tools/AgentTool/loadAgentsDir.ts` — Load agent definitions from disk
- `src/tools/AgentTool/builtInAgents.ts` — Register built-in agent types
- `src/coordinator/coordinatorMode.ts` — Coordinator mode context
- `src/utils/swarm/` — Full swarm backend implementation
- `src/utils/forkedAgent.ts` — Subagent context creation utilities
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx` — Background agent task lifecycle
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` — In-process teammate lifecycle
- `src/tools/TeamCreateTool/TeamCreateTool.ts` — Create agent team
- `src/tools/TeamDeleteTool/` — Delete agent team
- `src/tools/SendMessageTool/` — Send message to teammate

**Main Types/Interfaces**

- `AgentDefinition` in `src/tools/AgentTool/loadAgentsDir.ts`
- `LocalAgentTaskState` in `src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `InProcessTeammateTaskState` in `src/tasks/InProcessTeammateTask/types.ts`
- `TeamFile` in `src/utils/swarm/teamHelpers.ts`
- `BackendType` in `src/utils/swarm/backends/types.ts`

**Interactions with Other Domains**

- Conversation Engine: Each subagent runs its own `query()` loop
- Task Management: Agent tasks are registered in the global task registry
- Session Management: Agent transcripts are persisted as sidechain files
- Hook System: `SubagentStart` / `SubagentStop` hook events
- Analytics domain emits agent invocation events

**Complexity:** Very High

---

### Domain 06: MCP (Model Context Protocol)

**Purpose and Scope**

The MCP domain implements the Model Context Protocol, enabling Claude Code to connect to external tool servers and expose their capabilities as tools. It is a federated tool extension mechanism.

**Transport types** supported: `stdio` (subprocess), `sse` (HTTP Server-Sent Events), `http` (streaming HTTP), `ws` (WebSocket), `sdk` (in-process), `sse-ide`/`ws-ide` (IDE extensions), `claudeai-proxy` (Claude.ai-hosted servers).

**Connection lifecycle**: `MCPConnectionManager.tsx` manages connections to all configured servers, handles reconnection with backoff, surfaces connection status to the UI. Each connection produces a `MCPServerConnection` (Connected/Failed/NeedsAuth/Pending/Disabled).

**Tool wrapping**: `MCPTool` wraps MCP server tools as Claude Code tools, delegating `call()` to the MCP SDK client. Tool names are normalized/prefixed with the server name (`mcp__servername__toolname`).

**OAuth integration**: MCP servers requiring OAuth authentication go through `src/services/mcp/auth.ts` which uses PKCE flows. The `McpAuthTool` provides a first-class tool for interactive auth flows.

**Resource access**: `ListMcpResourcesTool` and `ReadMcpResourceTool` expose MCP Resources (documents, data) alongside tools.

**XAA (Cross-App Access)**: `src/services/mcp/xaa.ts` implements SEP-990 cross-app access, allowing Claude Code to authenticate on behalf of the user to third-party MCP servers using an IdP.

**IDE MCP**: `src/services/mcp/vscodeSdkMcp.ts` provides a special in-process MCP server that bridges IDE extensions (VS Code, JetBrains).

**Official Registry**: `src/services/mcp/officialRegistry.ts` fetches the curated list of Anthropic-approved MCP servers.

**Key Source Files**

- `src/services/mcp/MCPConnectionManager.tsx` — Connection lifecycle management
- `src/services/mcp/client.ts` — MCP client operations (connect, fetch tools, call tool)
- `src/services/mcp/types.ts` — All MCP type definitions and Zod schemas
- `src/services/mcp/config.ts` — MCP server config loading/merging from all scopes
- `src/services/mcp/auth.ts` — OAuth authentication for MCP servers
- `src/services/mcp/normalization.ts` — Tool/server name normalization
- `src/services/mcp/elicitationHandler.ts` — Elicitation protocol (-32042 errors)
- `src/services/mcp/channelPermissions.ts` — Channel-scoped permission callbacks
- `src/services/mcp/xaa.ts` — Cross-App Access (SEP-990)
- `src/services/mcp/vscodeSdkMcp.ts` — VS Code/IDE in-process MCP server
- `src/tools/MCPTool/MCPTool.ts` — MCP tool wrapper
- `src/tools/McpAuthTool/McpAuthTool.ts` — Interactive MCP auth tool
- `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`
- `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`

**Main Types/Interfaces**

- `MCPServerConnection` — Union: Connected/Failed/NeedsAuth/Pending/Disabled
- `McpServerConfig` — Zod-validated config for each transport type
- `ScopedMcpServerConfig` — Config with its source scope
- `ConnectedMCPServer` — Active connection with client + capabilities
- `MCPCliState` — Serialized state for CLI display

**Interactions with Other Domains**

- Plugin System: Plugins can declare MCP servers
- Authentication: MCP servers use OAuth or API keys
- Configuration: MCP servers configured in settings files at multiple scopes
- Tool Framework: MCP tools are wrapped as Tool instances

**Complexity:** High

---

### Domain 07: Session Management & Storage

**Purpose and Scope**

Session Management handles the persistence, retrieval, and lifecycle of conversation sessions. A session represents a single conversation thread that can span multiple turns, be paused and resumed, and be viewed in the history browser.

Sessions are stored as JSONL transcript files under `~/.claude/projects/<project-hash>/<session-id>.jsonl`. Each line is a typed log entry: messages, file history snapshots, attribution snapshots, content replacement entries, and metadata.

**Key capabilities:**
- **Transcript persistence**: Every message, tool call, and tool result is appended to the session JSONL file in real-time.
- **Resume**: Sessions can be resumed from any saved point. The `QueryEngine` accepts `initialMessages` to reconstruct context.
- **Teleport**: Sessions can be "teleported" — migrated to a remote execution environment, continuing the conversation there.
- **Session search**: `src/utils/agenticSessionSearch.ts` indexes session transcripts for `/history` search.
- **Sidechain transcripts**: Subagent transcripts are stored in subdirectories and linked to the parent session.
- **Session naming**: Sessions get human-readable word-slug names via `src/utils/words.ts`.
- **Concurrent sessions**: Multiple sessions can run simultaneously; `src/utils/concurrentSessions.ts` coordinates naming and indexing.
- **File history**: Every file edit produces a backup in `~/.claude/file-history/<session-id>/`.

**Key Source Files**

- `src/utils/sessionStorage.ts` — Core transcript read/write operations
- `src/types/logs.ts` — All JSONL entry types
- `src/utils/fileHistory.ts` — File edit snapshots and undo
- `src/utils/agenticSessionSearch.ts` — Session search indexing
- `src/utils/concurrentSessions.ts` — Multi-session coordination
- `src/hooks/useTeleportResume.tsx` — Teleport session resume hook
- `src/utils/background/remote/remoteSession.ts` — Remote session launch
- `src/assistant/sessionHistory.ts` — Session history for REPL

**Main Types/Interfaces**

- `Entry` in `src/types/logs.ts` — Discriminated union of all JSONL entry types
- `TranscriptMessage` — Serialized form of a Message for disk storage
- `FileHistorySnapshotMessage` — File backup reference
- `PersistedWorktreeSession` — Worktree-linked session metadata
- `SessionId`, `AgentId` in `src/types/ids.ts`

**Interactions with Other Domains**

- Conversation Engine: Records every turn
- File Operations: File history snapshots
- Analytics: Session start/end events, cost aggregation
- Remote Bridge: Remote session creation/resume
- Authentication: Session ingress token for remote sessions

**Complexity:** High

---

### Domain 08: Authentication & Security

**Purpose and Scope**

The Authentication domain manages how Claude Code obtains and validates credentials for the Anthropic API. It supports multiple credential types and storage backends.

**Credential types:**
1. **API Keys** — Direct `sk-ant-*` keys, stored in macOS Keychain, environment variables, or a file descriptor.
2. **OAuth Tokens** — Bearer tokens from the `claude.ai` OAuth flow, including refresh token management.
3. **AWS Bedrock** — STS-based credential validation for Bedrock customers.

**OAuth flow** (`src/services/oauth/`): Implements OAuth 2.0 Authorization Code with PKCE. `OAuthService.startOAuthFlow()` opens a browser, starts a local HTTP server to capture the callback, exchanges the code for tokens, and fetches the user profile. Supports both automatic (browser redirect) and manual (copy-paste) flows.

**Token storage**: Credentials are stored in the macOS Keychain (via a native module), with fallback to `~/.claude/config.json` for non-macOS platforms. `getSecureStorage()` provides a platform-abstracted interface.

**AWS support**: `src/utils/aws.ts` validates AWS STS credentials, `src/utils/awsAuthStatusManager.ts` manages the AWS auth state machine.

**Token refresh**: OAuth tokens are automatically refreshed when close to expiry via `refreshOAuthToken()` in `src/services/oauth/client.ts`.

**Auth file descriptor**: For scripted usage, tokens can be passed via file descriptor (`--api-key-fd`, `--oauth-token-fd`).

**Key Source Files**

- `src/utils/auth.ts` — Primary auth orchestration (getApiKey, getAuthToken, installOAuthTokens)
- `src/services/oauth/index.ts` — `OAuthService` class
- `src/services/oauth/client.ts` — OAuth HTTP operations (build URL, exchange code, refresh)
- `src/services/oauth/crypto.ts` — PKCE helpers (code verifier, challenge, state)
- `src/services/oauth/auth-code-listener.ts` — Local HTTP server for OAuth callback
- `src/utils/authPortable.ts` — Cross-platform keychain normalization
- `src/utils/authFileDescriptor.ts` — FD-based credential injection
- `src/utils/secureStorage/` — macOS Keychain + platform storage abstraction
- `src/utils/aws.ts` — AWS STS credential validation
- `src/utils/awsAuthStatusManager.ts` — AWS auth state manager

**Main Types/Interfaces**

- `OAuthTokens` in `src/services/oauth/types.ts` — Access/refresh tokens + metadata
- `SubscriptionType` — Subscription tier (free/pro/max/api)
- `RateLimitTier` — API rate limit tier

**Interactions with Other Domains**

- API Communication: Every API call uses auth credentials
- Analytics: OAuth events logged
- Configuration: Token stored in config
- Bridge: Session ingress tokens for remote sessions
- Billing: Subscription type determines billing display

**Complexity:** High

---

### Domain 09: Permissions & Safety

**Purpose and Scope**

The Permissions & Safety domain is the security enforcement layer that sits between the Conversation Engine and every tool call. It determines whether a tool call should be allowed, denied, or require user confirmation based on a rich rule system.

**Permission modes:**
- `default` — Normal operation, prompt for sensitive operations
- `plan` — Read-only mode, writes require plan mode exit
- `bypassPermissions` — Auto-allow everything (requires explicit opt-in)
- `auto` (ant-only) — AI classifier determines safety automatically

**Permission rules**: Stored in settings files as `allow`, `deny`, and `ask` arrays using a pattern syntax like `Bash(git *)`, `Read(*.ts)`. Rules can come from user settings, project settings, enterprise MDM, or plugins.

**Classifier** (ant-only, `TRANSCRIPT_CLASSIFIER` feature): `src/utils/permissions/bashClassifier.ts` and `src/utils/permissions/yoloClassifier.ts` implement auto-mode where an AI model classifies the safety of bash commands using the conversation transcript as context.

**Denial tracking**: `src/utils/permissions/denialTracking.ts` counts consecutive denials; after a threshold, the tool falls back to prompting to prevent infinite auto-denial loops.

**Sandbox**: `src/utils/sandbox/sandbox-adapter.ts` wraps macOS's sandbox framework to execute bash commands in a restricted environment.

**Dangerous pattern detection**: `src/utils/permissions/dangerousPatterns.ts` detects commands that would break out of the sandbox, wipe the filesystem, or exfiltrate data.

**Permission update propagation**: `src/utils/permissions/PermissionUpdate.ts` handles saving user decisions to the appropriate settings scope (local file, user file, memory, etc.).

**Key Source Files**

- `src/utils/permissions/permissions.ts` — Core permission decision logic
- `src/utils/permissions/PermissionMode.ts` — Mode enum and display names
- `src/utils/permissions/PermissionResult.ts` — Decision types (allow/deny/ask)
- `src/utils/permissions/PermissionRule.ts` — Rule types and pattern syntax
- `src/utils/permissions/permissionRuleParser.ts` — Rule parsing
- `src/utils/permissions/permissionsLoader.ts` — Load rules from settings
- `src/utils/permissions/denialTracking.ts` — Consecutive denial counter
- `src/utils/permissions/dangerousPatterns.ts` — Blocked command patterns
- `src/utils/permissions/bashClassifier.ts` — Auto-mode bash classifier
- `src/utils/permissions/yoloClassifier.ts` — Auto-mode conversation classifier
- `src/utils/permissions/filesystem.ts` — Filesystem permission checks
- `src/utils/sandbox/sandbox-adapter.ts` — macOS sandbox
- `src/hooks/toolPermission/` — UI permission handlers (interactive/coordinator/swarm)

**Main Types/Interfaces**

- `PermissionMode` — Current permission mode
- `PermissionResult` — `allow | deny | ask` with reason
- `PermissionRule` / `PermissionRuleSource` / `PermissionRuleValue`
- `ToolPermissionContext` — Immutable snapshot of all permission rules
- `DenialTrackingState` — Consecutive denial counter

**Interactions with Other Domains**

- Every tool domain: All tool calls go through `checkPermissions`
- Configuration: Rules loaded from all settings scopes
- Analytics: Permission decisions logged
- Hook System: `PermissionRequest`/`PermissionDenied` hook events

**Complexity:** Very High

---

### Domain 10: Context Management & Compaction

**Purpose and Scope**

Context Management is responsible for tracking, measuring, and managing the LLM's context window — the finite token budget available per conversation. When the context fills up, compaction shrinks it by generating a summary.

**Context analysis**: `src/utils/analyzeContext.ts` and `src/utils/context.ts` count tokens, compute warning thresholds, and categorize where tokens are being used (system prompt, tool definitions, user messages, assistant messages, tool results).

**Auto-compaction**: `src/services/compact/autoCompact.ts` triggers automatically when the context reaches a threshold (default: when remaining tokens fall below a reserve). It calls `compactConversation()`.

**Compaction flow** (`src/services/compact/compact.ts`):
1. Pre-compact hooks fire.
2. A forked subagent generates a summary of the conversation so far.
3. Messages before the compaction boundary are replaced with the summary.
4. Post-compact hooks fire.
5. Session memory is updated with any new learnings.

**Micro-compaction**: `src/services/compact/microCompact.ts` performs lightweight in-place compaction for individual messages without a full LLM call.

**Session memory compaction**: After compaction, `src/services/compact/sessionMemoryCompact.ts` runs an extraction pass to save important facts to the session memory file.

**Context suggestions**: `src/services/compact/` provides `compactWarningHook.ts` that shows a UI warning before context fills up.

**Time-based MC config**: `src/services/compact/timeBasedMCConfig.ts` adjusts compaction parameters based on conversation age.

**Snip compaction** (feature-gated): `src/services/compact/snipCompact.ts` implements a "snip" strategy that removes old turns in chunks rather than summarizing.

**Key Source Files**

- `src/services/compact/compact.ts` — Core compaction algorithm
- `src/services/compact/autoCompact.ts` — Auto-compact trigger and thresholds
- `src/services/compact/microCompact.ts` — Lightweight message compaction
- `src/services/compact/sessionMemoryCompact.ts` — Post-compaction memory update
- `src/services/compact/compactWarningHook.ts` — UI warning before overflow
- `src/utils/analyzeContext.ts` — Token counting and analysis
- `src/utils/context.ts` — Context window constants per model
- `src/services/tokenEstimation.ts` — Token count via API or estimation

**Main Types/Interfaces**

- `CompactionResult` — Output of a compaction run
- `AutoCompactTrackingState` — State tracking auto-compact threshold
- `CompactProgressEvent` — UI progress events during compaction

**Interactions with Other Domains**

- Conversation Engine: Auto-compact fires during the query loop
- Agent Orchestration: Each subagent has its own context budget
- Session Management: Compact boundary messages persisted
- Analytics: Compaction events logged
- Memory System: Post-compaction memory extraction

**Complexity:** High

---

### Domain 11: Memory System

**Purpose and Scope**

The Memory System manages how Claude Code retains information across turns, sessions, and users. It has multiple layers:

**CLAUDE.md files** (`src/utils/claudemd.ts`): Hierarchical instruction files loaded at session start. Discovery order: managed (`/etc/claude-code/CLAUDE.md`), user (`~/.claude/CLAUDE.md`), project (`.claude/CLAUDE.md`, `CLAUDE.md`, `.claude/rules/*.md`), local (`CLAUDE.local.md`). Support `@include` directives for composability.

**Session Memory** (`src/services/SessionMemory/`): An automatically maintained markdown file (`~/.claude/session-memory/<session-id>.md`) that a background forked agent updates after each turn. Tracks key facts, decisions, and in-progress work within the current session.

**Auto Memory / memdir** (`src/memdir/`): A user-editable `MEMORY.md` index file that aggregates cross-session learnings. The `autoDream` service consolidates multiple session transcripts into this persistent memory.

**Auto Dream** (`src/services/autoDream/`): Background memory consolidation that fires when enough sessions have accumulated. Runs the `/dream` prompt as a forked subagent to extract and consolidate learnings from recent session transcripts into `MEMORY.md`.

**Extract Memories** (`src/services/extractMemories/`): A focused service for extracting memorable facts from a conversation and appending them to the user's MEMORY.md.

**Team Memory** (`src/memdir/teamMemPaths.ts`, `teamMemPrompts.ts`, feature-gated `TEAMMEM`): Shared memory files for agent teams.

**Magic Docs** (`src/services/MagicDocs/`): Files marked with `# MAGIC DOC: title` headers are automatically updated by a background agent when relevant information appears in the conversation.

**Memory types** (`src/utils/memory/types.ts`): `User | Project | Local | Managed | AutoMem | TeamMem`

**Key Source Files**

- `src/utils/claudemd.ts` — CLAUDE.md file discovery and loading
- `src/memdir/memdir.ts` — MEMORY.md entrypoint loading
- `src/memdir/paths.ts` — Auto-memory path resolution
- `src/memdir/memoryScan.ts` — Scanning for memory files
- `src/memdir/findRelevantMemories.ts` — Relevance-based memory retrieval
- `src/services/SessionMemory/sessionMemory.ts` — Session memory update loop
- `src/services/SessionMemory/sessionMemoryUtils.ts` — State tracking
- `src/services/autoDream/autoDream.ts` — Cross-session consolidation
- `src/services/extractMemories/extractMemories.ts` — Memory extraction
- `src/services/MagicDocs/magicDocs.ts` — Magic Docs auto-update

**Main Types/Interfaces**

- `MemoryType` — `User | Project | Local | Managed | AutoMem | TeamMem`
- `EntrypointTruncation` — MEMORY.md truncation metadata
- `SessionMemoryConfig` — Session memory feature configuration

**Interactions with Other Domains**

- Conversation Engine: CLAUDE.md injected into system prompt
- Compaction: Session memory updated post-compact
- Analytics: Memory extraction events
- File Operations: Memory files read/written
- Agent Orchestration: Team memory shared across agents

**Complexity:** High

---

### Domain 12: Plugin & Extension System

**Purpose and Scope**

The Plugin System allows Claude Code to be extended with custom slash commands, agents, hooks, MCP servers, LSP configurations, and output styles — without modifying the core CLI. Plugins are distributed as git repositories or npm packages.

**Plugin structure**:
```
my-plugin/
├── plugin.json          # Manifest (name, version, description, author)
├── commands/            # Slash command markdown files
│   └── build.md         # /build command prompt
├── agents/              # Custom agent definitions
│   └── test-runner.md
└── hooks/
    └── hooks.json       # Hook definitions
```

**Plugin sources**: Marketplace-based (via `source@marketplace` identifiers) and session-only (via `--plugin-dir` CLI flag or SDK `plugins` option).

**Marketplace system** (`src/utils/plugins/marketplaceManager.ts`): Marketplaces are git repositories containing a manifest of available plugins. The official Anthropic marketplace is at a known URL. Users can add custom marketplaces. The `reconciler.ts` manages install/update operations.

**Plugin loader** (`src/utils/plugins/pluginLoader.ts`): Discovers, loads, and validates plugins. Handles manifest parsing, hooks configuration, duplicate name detection, enable/disable state.

**Built-in plugins** (`src/plugins/builtinPlugins.ts`): Plugins that ship with the CLI binary, enabling/disabling via user settings. Examples include the computer-use plugin (macOS), LSP plugins, and bundled skill packs.

**Plugin autoupdate** (`src/utils/plugins/pluginAutoupdate.ts`): Background auto-update for marketplace plugins.

**Key Source Files**

- `src/utils/plugins/pluginLoader.ts` — Plugin discovery and loading
- `src/utils/plugins/marketplaceManager.ts` — Marketplace management
- `src/utils/plugins/reconciler.ts` — Install/update reconciliation
- `src/utils/plugins/refresh.ts` — Hot-reload plugins
- `src/services/plugins/PluginInstallationManager.ts` — Background installation
- `src/services/plugins/pluginOperations.ts` — CRUD for plugin state
- `src/types/plugin.ts` — Plugin type definitions
- `src/plugins/builtinPlugins.ts` — Built-in plugin registry
- `src/utils/plugins/schemas.ts` — Zod schemas for manifests
- `src/utils/plugins/pluginAutoupdate.ts` — Auto-update

**Main Types/Interfaces**

- `LoadedPlugin` — A fully loaded and validated plugin
- `PluginManifest` — plugin.json schema
- `BuiltinPluginDefinition` — In-code plugin definition
- `PluginRepository` — Git repository coordinates
- `PluginLoadResult` — Success/error wrapper

**Interactions with Other Domains**

- Tool Framework: Plugins provide additional tools
- MCP: Plugins declare MCP server configs
- Hook System: Plugins register hooks via hooks.json
- Configuration: Plugin enable/disable stored in settings
- Skill System: Plugins provide bundled skills
- Analytics: Plugin telemetry events

**Complexity:** High

---

### Domain 13: UI/UX Terminal (Ink)

**Purpose and Scope**

The Terminal UI domain implements the interactive terminal interface using a heavily customized fork of the `ink` React renderer. This is a complete terminal UI framework, not just a thin wrapper.

**Ink renderer** (`src/ink/`): A custom React reconciler that renders React component trees to terminal output. Key components include:
- Layout engine using Facebook's Yoga flexbox layout (`src/ink/layout/`)
- ANSI color/style output (`src/ink/output.ts`, `src/ink/termio/`)
- Event system for keyboard input, mouse clicks, terminal focus
- Hit testing for click events on rendered boxes
- Search highlight overlays (`src/ink/searchHighlight.ts`)
- Selection tracking (`src/ink/selection.ts`)
- Scroll box management (`src/ink/components/ScrollBox.tsx`)
- Alternate screen support (`src/ink/components/AlternateScreen.tsx`)

**Application components** (`src/components/`): Over 100 React components covering every UI surface:
- Main REPL loop (`src/components/App.tsx`)
- Message rendering (per-tool result views)
- Dialog system (permission prompts, MCP approvals, export dialogs)
- Logo/welcome screen (`src/components/LogoV2/`)
- Status line and footer
- History search (`src/components/HistorySearchDialog.tsx`)
- Context visualization (`src/components/ContextVisualization.tsx`)
- Feedback survey (`src/components/FeedbackSurvey/`)

**Input handling** (`src/hooks/`): React hooks for:
- Vim mode input (`src/hooks/useVimInput.ts`)
- Text input buffer (`src/hooks/useInputBuffer.ts`)
- Arrow key history navigation (`src/hooks/useArrowKeyHistory.tsx`)
- History search (`src/hooks/useHistorySearch.ts`)
- Typeahead completion (`src/hooks/useTypeahead.tsx`)
- Paste handling (`src/hooks/usePasteHandler.ts`)

**Theming**: `src/utils/theme.ts` provides named colors (dark/light terminal theme detection).

**Key Source Files**

- `src/ink/ink.tsx` — Core Ink instance
- `src/ink/reconciler.ts` — React reconciler for terminal
- `src/ink/layout/engine.ts` — Yoga flexbox layout engine
- `src/ink/renderer.ts` — Frame rendering
- `src/ink/render-to-screen.ts` — Terminal output
- `src/ink/termio/` — ANSI/CSI/OSC/SGR terminal control sequences
- `src/components/App.tsx` — Main application component
- `src/components/LogoV2/LogoV2.tsx` — Welcome/idle screen
- `src/hooks/useVimInput.ts` — Vim mode
- `src/hooks/useTextInput.ts` — Text input management
- `src/native-ts/yoga-layout/` — Yoga layout bindings

**Main Types/Interfaces**

- `Theme`, `ThemeName` in `src/utils/theme.ts`
- Ink DOM nodes in `src/ink/node-cache.ts`
- Layout geometry in `src/ink/layout/geometry.ts`

**Interactions with Other Domains**

- Every tool domain: Provides `renderToolUseMessage` / `renderToolResultMessage`
- Configuration: Theme settings
- Permissions: Permission dialog UI
- Analytics: UI interaction events
- Hook System: Notification rendering

**Complexity:** Very High

---

### Domain 14: Configuration System

**Purpose and Scope**

The Configuration System manages Claude Code's settings hierarchy — from global user preferences to enterprise MDM policies. Settings flow from low to high precedence: managed/MDM > enterprise > user > project > local > dynamic.

**Settings file locations**:
- `~/.claude/config.json` — Global user config (auth, preferences)
- `~/.claude/settings.json` — User-level settings (hooks, permissions, etc.)
- `.claude/settings.json` — Project-level settings (shared with team)
- `.claude/settings.local.json` — Local project overrides (gitignored)
- `/etc/claude-code/managed_settings.json` — MDM/enterprise managed settings
- `CLAUDE.md` hierarchy — Natural language instructions

**Settings schema** (`src/utils/settings/types.ts`): Comprehensive Zod schema covering permissions, hooks, MCP servers, environment variables, and feature flags.

**MDM support** (`src/utils/settings/mdm/`): Reads managed settings from platform-appropriate paths, validates them, and enforces them as non-overridable.

**Settings change detection** (`src/utils/settings/changeDetector.ts`): Watches settings files for changes and applies them live without restarting.

**Plugin-only policy** (`src/utils/settings/pluginOnlyPolicy.ts`): Restricts tool access to plugin-defined tools when the `restrictedToPluginOnly` setting is active.

**Global config** (`src/utils/config.ts`): Separate from settings, the global config stores auth tokens, account info, installation metadata, and per-project data.

**Dynamic config**: GrowthBook feature flags provide per-session dynamic configuration overrides, used for A/B testing and gradual rollouts.

**Key Source Files**

- `src/utils/settings/settings.ts` — Primary settings loading/access
- `src/utils/settings/types.ts` — Zod schema for all settings
- `src/utils/settings/constants.ts` — Setting source constants
- `src/utils/settings/mdm/` — MDM settings reading
- `src/utils/settings/applySettingsChange.ts` — Live settings updates
- `src/utils/settings/changeDetector.ts` — File watcher
- `src/utils/config.ts` — Global config (auth, account, install)
- `src/utils/claudemd.ts` — CLAUDE.md config file loading
- `src/tools/ConfigTool/ConfigTool.ts` — Config modification tool

**Main Types/Interfaces**

- `SettingsJson` — Full settings schema type
- `PermissionsSchema` — Permission rules settings
- `HooksSchema` — Hooks settings
- `SettingSource` — Scope identifier
- `AccountInfo` — OAuth account metadata in global config

**Interactions with Other Domains**

- Permissions: Permission rules loaded from settings
- Hook System: Hooks configured in settings
- MCP: MCP server configs in settings
- Authentication: Auth tokens in global config
- Analytics: Settings change events
- Plugin System: Plugin enable/disable in settings

**Complexity:** High

---

### Domain 15: Analytics & Telemetry

**Purpose and Scope**

The Analytics & Telemetry domain provides observability infrastructure for Claude Code. It is designed for zero-overhead startup (events queue before the sink attaches), PII safety (typed metadata prevents accidentally logging code/filepaths), and multi-sink fanout.

**Event logging** (`src/services/analytics/index.ts`): The public API. `logEvent()` / `logEventAsync()` queue events before the sink is ready, then drain synchronously via `queueMicrotask`. PII protection is enforced at the type system level — string values require the `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` marker type.

**Sink** (`src/services/analytics/sink.ts`): Routes events to Datadog and the first-party event logger (1P). Event sampling is configurable via GrowthBook.

**Datadog** (`src/services/analytics/datadog.ts`): Custom metrics via OpenTelemetry counters (session count, LOC, PR count, token count, cost, etc.).

**First-party event logging** (`src/services/analytics/firstPartyEventLogger.ts`, `firstPartyEventLoggingExporter.ts`): Sends events to an internal BigQuery pipeline for product analytics.

**GrowthBook** (`src/services/analytics/growthbook.ts`): Feature flags and A/B experiment assignment. User attributes (platform, subscription type, session ID) are sent to GrowthBook for targeting. The `getDynamicConfig_*` functions return feature values.

**OpenTelemetry tracing** (`src/utils/telemetry/`): Distributed tracing for performance analysis. Session tracing, tool call tracing, and BigQuery export.

**Perfetto tracing** (`src/utils/telemetry/perfettoTracing.ts`): Ant-only instrumentation that records trace events in Perfetto format for deep performance profiling of multi-agent sessions.

**Metadata sanitization** (`src/services/analytics/metadata.ts`): Strips sensitive information from tool names and other metadata before logging.

**Key Source Files**

- `src/services/analytics/index.ts` — Public API: logEvent, logEventAsync
- `src/services/analytics/sink.ts` — Event routing to backends
- `src/services/analytics/datadog.ts` — Datadog metrics via OTLP
- `src/services/analytics/firstPartyEventLogger.ts` — 1P BQ event pipeline
- `src/services/analytics/growthbook.ts` — Feature flags / A/B testing
- `src/services/analytics/metadata.ts` — PII-safe metadata helpers
- `src/services/analytics/sinkKillswitch.ts` — Emergency disable
- `src/utils/telemetry/sessionTracing.ts` — Session-level traces
- `src/utils/telemetry/instrumentation.ts` — OTLP setup
- `src/utils/telemetry/perfettoTracing.ts` — Perfetto trace format
- `src/cost-tracker.ts` — Aggregates token and cost usage

**Main Types/Interfaces**

- `AnalyticsSink` — Interface for pluggable backends
- `GrowthBookUserAttributes` — User targeting attributes
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — PII safety marker type

**Interactions with Other Domains**

- All domains: Every domain emits `logEvent` calls
- Bootstrap: Global state holds OTel providers
- Authentication: User attributes for GrowthBook
- Cost & Billing: Token usage fed to analytics

**Complexity:** High

---

### Domain 16: Remote Bridge & Desktop Integration

**Purpose and Scope**

The Remote Bridge domain enables Claude Code to receive work from external sources — primarily the claude.ai web interface. A "bridge" is a long-lived server process that registers itself with the Anthropic backend as an available environment, polls for work items (sessions), and spawns Claude Code child processes to handle each session.

**Bridge architecture** (`src/bridge/`):
- `bridgeMain.ts` — Main bridge server loop: register environment, poll for work, spawn sessions
- `bridgeApi.ts` — HTTP client for the environments API (register, poll, acknowledge, heartbeat)
- `sessionRunner.ts` — Spawns and manages individual session child processes
- `replBridge.ts` / `initReplBridge.ts` — In-session bridge (the child process side), manages the WebSocket/SSE connection to the session ingress service
- `inboundMessages.ts` — Processes incoming messages from the web client
- `bridgePermissionCallbacks.ts` — Routes permission decisions back to web client
- `trustedDevice.ts` — Trusted device certificate management
- `workSecret.ts` — Decodes the work secret (JWT) containing session config
- `capacityWake.ts` — Wakes sleeping sessions when capacity is available

**Transport system** (`src/cli/transports/`): Multiple transport implementations for the in-session connection:
- `SSETransport.ts` — Server-Sent Events (CCR v1)
- `WebSocketTransport.ts` — WebSocket (CCR v2)
- `HybridTransport.ts` — Automatically picks SSE or WS
- `ccrClient.ts` — CCR v2 worker registration client
- `WorkerStateUploader.ts` — Uploads session state for resume

**Remote IO** (`src/cli/remoteIO.ts`): Handles structured I/O in non-interactive (SDK/headless) mode, formatting messages as NDJSON.

**Desktop handoff**: `src/components/DesktopHandoff.tsx` handles the flow of opening the Claude Desktop app from the CLI.

**Key Source Files**

- `src/bridge/bridgeMain.ts` — Bridge server main loop
- `src/bridge/types.ts` — All bridge type definitions
- `src/bridge/sessionRunner.ts` — Session child process management
- `src/bridge/replBridge.ts` — In-session bridge connection
- `src/bridge/inboundMessages.ts` — Message processing from web
- `src/bridge/bridgePermissionCallbacks.ts` — Permission UI relay
- `src/cli/transports/SSETransport.ts` — SSE connection
- `src/cli/transports/WebSocketTransport.ts` — WS connection
- `src/cli/transports/HybridTransport.ts` — Auto transport selection
- `src/cli/remoteIO.ts` — Structured NDJSON I/O

**Main Types/Interfaces**

- `BridgeConfig` — Bridge server configuration
- `WorkResponse` — Work item from polling API
- `WorkSecret` — Decoded session configuration JWT
- `SessionHandle` — Running session reference
- `BridgeWorkerType` — `claude_code | claude_code_assistant`
- `SpawnMode` — `single-session | worktree | same-dir`

**Interactions with Other Domains**

- Authentication: Session ingress tokens
- Session Management: Session creation and transcript persistence
- Conversation Engine: Drives the query loop in remote sessions
- Permissions: Remote permission callbacks
- Analytics: Session bridge events

**Complexity:** High

---

### Domain 17: LSP & Code Intelligence

**Purpose and Scope**

The LSP domain integrates Language Server Protocol servers to provide real-time code intelligence: hover information, go-to-definition, find-references, diagnostics, and more. This gives the model richer context when working with code.

**LSP client** (`src/services/lsp/LSPClient.ts`): Creates and manages a JSON-RPC connection to an LSP server subprocess via stdio. Uses `vscode-jsonrpc` for message framing. Handles initialization, capability negotiation, crash detection, and restart.

**LSP server manager** (`src/services/lsp/LSPServerManager.ts`): Manages multiple LSP server instances (one per language/configuration). Lazy-initializes servers on first request. Handles workspace folders and file-open events.

**LSPTool** (`src/tools/LSPTool/LSPTool.ts`): The model-facing tool that exposes LSP capabilities. Operations include: `hover`, `definition`, `references`, `document_symbols`, `workspace_symbols`, `completions`, `diagnostics`, `rename`.

**Passive feedback** (`src/services/lsp/passiveFeedback.ts`): Proactively surfaces LSP diagnostics (errors/warnings) after file edits.

**Diagnostic tracking** (`src/services/diagnosticTracking.ts`): Tracks baseline diagnostics before and after edits to detect new errors introduced.

**Symbol context** (`src/tools/LSPTool/symbolContext.ts`): Provides additional context around LSP symbols for better model understanding.

**LSP plugin integration** (`src/utils/plugins/lspPluginIntegration.ts`): Plugins can declare LSP server configurations, automatically starting appropriate language servers.

**LSP recommendation** (`src/utils/plugins/lspRecommendation.ts`): Detects the language of files being edited and recommends installing relevant LSP plugins.

**Key Source Files**

- `src/services/lsp/LSPClient.ts` — JSON-RPC LSP client
- `src/services/lsp/LSPServerManager.ts` — Multi-server manager
- `src/services/lsp/LSPServerInstance.ts` — Single server instance lifecycle
- `src/services/lsp/LSPDiagnosticRegistry.ts` — Diagnostic aggregation
- `src/services/lsp/config.ts` — LSP server configuration
- `src/tools/LSPTool/LSPTool.ts` — Model-facing LSP tool
- `src/tools/LSPTool/formatters.ts` — Result formatting
- `src/tools/LSPTool/symbolContext.ts` — Symbol context enrichment
- `src/services/diagnosticTracking.ts` — Before/after diagnostics
- `src/services/lsp/passiveFeedback.ts` — Proactive diagnostics

**Main Types/Interfaces**

- `LSPClient` — Interface for an LSP client connection
- `LSPServerInstance` — Lifecycle wrapper for one server
- `Diagnostic` in `src/services/diagnosticTracking.ts`

**Interactions with Other Domains**

- File Operations: LSP is notified of file opens/changes
- Plugin System: Plugins register LSP servers
- IDE Integration: IDE LSP server via SSE-IDE transport
- Analytics: LSP usage events

**Complexity:** Medium

---

### Domain 18: Version Control (Git)

**Purpose and Scope**

The Git domain provides version control capabilities, primarily used for commit attribution, worktree management, gitignore filtering, and the GitHub App CI/CD setup workflow.

**Commit attribution** (`src/utils/commitAttribution.ts`): Tracks which files were modified by Claude Code and adds `Co-Authored-By: Claude` trailers to git commits. Maintains a per-session attribution state mapping file paths to their modification history.

**Git utilities** (`src/utils/git.ts`, `src/utils/git/`):
- `findGitRoot()` — Traverses upward to find `.git`
- `gitExe` — Resolved path to git executable
- `getBranch()` — Current branch name
- `getRemoteUrlForDir()` — Remote URL for commit attribution display
- `gitignore.ts` — Parses `.gitignore` patterns for file filtering

**Worktree support**: `EnterWorktreeTool` and `ExitWorktreeTool` create/remove git worktrees for isolated branch work. `src/utils/getWorktreePaths.ts` resolves worktree-relative paths.

**Git config parser** (`src/utils/git/gitConfigParser.ts`): Parses `.git/config` for remote URL extraction.

**GitHub App** (`src/commands/install-github-app/`): Multi-step wizard to install the Claude Code GitHub App into a repository, creating GitHub Actions workflows for automated PR review and code fixes.

**PR workflows** (`src/commands/commit.ts`, `src/commands/commit-push-pr.ts`): Slash commands that drive git commit, push, and PR creation flows.

**Key Source Files**

- `src/utils/commitAttribution.ts` — Attribution tracking and trailer injection
- `src/utils/git.ts` — Core git utilities
- `src/utils/git/gitFilesystem.ts` — Filesystem-level git operations
- `src/utils/git/gitignore.ts` — .gitignore parsing
- `src/utils/git/gitConfigParser.ts` — Git config parsing
- `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` — Create worktree
- `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` — Remove worktree
- `src/commands/install-github-app/` — GitHub App setup wizard
- `src/commands/install-github-app/setupGitHubActions.ts` — Workflow creation

**Interactions with Other Domains**

- File Operations: Gitignore filtering in glob/grep
- Session Management: Branch info in session metadata
- Bridge: Git worktree mode for remote sessions
- Analytics: Commit attribution events
- Agent Orchestration: Worktree isolation for parallel agents

**Complexity:** Medium

---

### Domain 19: Cost & Billing

**Purpose and Scope**

The Cost & Billing domain tracks API usage costs, token consumption, and billing-related access controls. It provides real-time cost visibility and enforces budget limits.

**Cost tracking** (`src/cost-tracker.ts`): Aggregates token counts (input, output, cache read, cache write) and converts them to USD cost using per-model pricing tables. Exposes the running total via `getTotalCost()`. Resets between sessions but accumulates within a session.

**Token counting**: Every API response includes usage in `BetaUsage`. These are accumulated in `src/bootstrap/state.ts` global counters.

**Cost display**: The status bar and `/cost` command show the running cost with breakdown by model.

**Budget enforcement** (`src/services/claudeAiLimits.ts`): For claude.ai subscribers, rate limit information is fetched and displayed. `src/components/LogoV2/` shows upsell messages when limits are reached.

**Billing access** (`src/utils/billing.ts`): `hasConsoleBillingAccess()` determines if a user can see billing information (admin/billing roles only). `hasClaudeAiBillingAccess()` checks for claude.ai subscription.

**Overage credits** (`src/services/api/overageCreditGrant.ts`): Handles auto-grant of overage credits for subscribers.

**Guest passes / Extra usage** (`src/commands/extra-usage/`): UI for managing guest passes and extra usage allocations.

**Key Source Files**

- `src/cost-tracker.ts` — Token/cost aggregation
- `src/utils/billing.ts` — Billing access checks
- `src/utils/modelCost.ts` — USD cost calculation from token counts
- `src/services/claudeAiLimits.ts` — Rate limit status
- `src/services/api/usage.ts` — Usage API integration
- `src/services/api/overageCreditGrant.ts` — Overage credit handling
- `src/commands/cost/` — `/cost` command UI
- `src/commands/extra-usage/` — Extra usage management

**Interactions with Other Domains**

- Authentication: Subscription type determines billing display
- Analytics: Cost events (per-turn, per-session)
- Conversation Engine: Budget limits gate new turns
- Configuration: Cost threshold alerts in settings

**Complexity:** Medium

---

### Domain 20: Auto-Update System

**Purpose and Scope**

The Auto-Update domain keeps Claude Code current by detecting new versions and installing them without user intervention. It supports multiple installation methods and release channels.

**Version detection** (`src/utils/autoUpdater.ts`): Fetches the latest version from a GCS bucket (`claude-code-dist-*.storage.googleapis.com`). Checks against the current running version using semver comparison.

**Installation methods**:
- `npm install -g @anthropic-ai/claude-code` — npm global
- Native binary installer (`src/utils/nativeInstaller/`) — Platform-specific binary distribution
- Local installer (`src/utils/localInstaller.ts`) — Development/local installation

**Update flow** (`src/cli/update.ts`): Checks for updates, validates installation type, installs if newer version found. Auto-update runs on startup if enabled.

**Release channels**: `latest` (stable), `beta`, `alpha`. Configured in user settings.

**Minimum version enforcement**: GrowthBook config can specify a minimum required version. Old clients are told to update.

**AutoUpdater component** (`src/components/AutoUpdater.tsx`): React component that shows update progress/results in the UI.

**Key Source Files**

- `src/utils/autoUpdater.ts` — Version detection and npm installation
- `src/utils/nativeInstaller/` — Platform binary installation
- `src/utils/localInstaller.ts` — Local installation
- `src/cli/update.ts` — Update command implementation
- `src/components/AutoUpdater.tsx` — Update UI component
- `src/hooks/useUpdateNotification.ts` — Update notification hook

**Interactions with Other Domains**

- Analytics: Update events
- Configuration: Release channel setting
- UI/UX Terminal: Update progress display

**Complexity:** Medium

---

### Domain 21: Task Management

**Purpose and Scope**

The Task Management domain provides the infrastructure for tracking and managing background operations. Tasks are long-running operations that execute asynchronously while the main conversation continues.

**Task types**:
- `local_bash` — Background bash commands (`LocalShellTask`)
- `local_agent` — Background agent subagents (`LocalAgentTask`)
- `remote_agent` — Agent running on remote infrastructure
- `in_process_teammate` — In-process swarm member
- `local_workflow` — Multi-step workflow
- `monitor_mcp` — MCP monitoring task
- `dream` — Auto-memory consolidation task

**Task lifecycle**: Tasks go through `pending → running → (completed | failed | killed)`. Each task has a unique ID with a type prefix (e.g., `b` for bash, `a` for agent).

**Task output**: Running tasks write to disk at `~/.claude/task-output/<task-id>`. The UI polls this file for new output. `src/utils/task/diskOutput.ts` manages the on-disk format.

**Task framework** (`src/utils/task/framework.ts`): Shared helpers for updating task state in AppState, managing the standard polling interval, and creating `TaskAttachment` objects for the main message stream.

**Shell task management** (`src/tasks/LocalShellTask/`): Handles spawning, monitoring, and killing bash background tasks with `LocalShellTask`.

**Scheduled tasks / Cron** (`src/tools/ScheduleCronTool/`): The cron system allows scheduling prompts on a cron schedule. Durable tasks persist to `.claude/scheduled_tasks.json`; session-only tasks live in memory.

**Task list** (`src/tools/TaskCreateTool/`, `TaskGetTool/`, etc.): A simpler task list mechanism for tracking work items in a markdown-like format (separate from background tasks).

**Todo list** (`src/tools/TodoWriteTool/`): Per-session to-do tracking that the model maintains and the UI displays.

**Key Source Files**

- `src/Task.ts` — Task type definitions, ID generation, state base
- `src/tasks/types.ts` — Full task state types per task type
- `src/utils/task/framework.ts` — Shared task lifecycle helpers
- `src/utils/task/diskOutput.ts` — On-disk task output management
- `src/tasks/LocalShellTask/LocalShellTask.tsx`
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx`
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- `src/tasks/DreamTask/DreamTask.ts`
- `src/tools/ScheduleCronTool/` — Cron scheduling tools
- `src/utils/cronTasks.ts` — Cron task management
- `src/tools/TodoWriteTool/TodoWriteTool.ts`

**Interactions with Other Domains**

- Agent Orchestration: Agent tasks are a subtype of tasks
- Bash Execution: Shell tasks
- Conversation Engine: Tasks inject output into the message stream
- Session Management: Task output persisted
- UI/UX Terminal: Task status shown in footer and expanded panel

**Complexity:** High

---

### Domain 22: Hook System

**Purpose and Scope**

The Hook System provides lifecycle integration points where external scripts, LLM prompts, or in-process callbacks can intercept, augment, or block Claude Code's actions. Hooks are Claude Code's extensibility mechanism for power users and enterprise policies.

**Hook events** (from `HOOK_EVENTS` in `src/entrypoints/sdk/coreTypes.ts`):
`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, `PermissionRequest`, `PermissionDenied`, `Setup`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `Elicitation`, `ElicitationResult`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, `InstructionsLoaded`, `CwdChanged`, `FileChanged`

**Hook types** (defined in `src/schemas/hooks.ts`):
- `command` — Shell command hook: `{ type: "command", command: "...", if: "Bash(git *)" }`
- `prompt` — LLM prompt hook: `{ type: "prompt", prompt: "..." }`
- `agent` — Agent hook: `{ type: "agent", agent: "..." }`
- `http` — HTTP hook: calls a webhook endpoint

**Hook execution** (`src/utils/hooks/`):
- `execAgentHook.ts` — Runs agent hooks as forked subagents
- `execHttpHook.ts` — HTTP POST to webhook with SSRF protection
- `execPromptHook.ts` — Evaluates LLM prompt hooks inline
- `hookHelpers.ts` — Core hook dispatch logic
- `hooksConfigManager.ts` — Loads and caches hook configurations
- `postSamplingHooks.ts` — Hooks that fire after each model sampling turn
- `sessionHooks.ts` — Session-scoped hooks

**Hook output protocol**: Hooks can return JSON to influence Claude's behavior:
- `continue: false` — Stop execution
- `decision: "approve" | "block"` — Pre-tool permission decision
- `updatedInput: {...}` — Modify the tool's input
- `systemMessage: "..."` — Inject a system message
- `asyncRewake: true` — Run in background; wake model on exit code 2

**Frontmatter hooks** (`src/utils/hooks/registerFrontmatterHooks.ts`): Hooks defined in CLAUDE.md frontmatter are automatically registered.

**Skill hooks** (`src/utils/hooks/registerSkillHooks.ts`): Skills can register their own hooks.

**SSRF protection** (`src/utils/hooks/ssrfGuard.ts`): Validates webhook URLs against SSRF patterns.

**Key Source Files**

- `src/schemas/hooks.ts` — Hook Zod schemas
- `src/types/hooks.ts` — Hook TypeScript types
- `src/utils/hooks/hookHelpers.ts` — Core dispatch
- `src/utils/hooks/hooksConfigManager.ts` — Config management
- `src/utils/hooks/execAgentHook.ts`
- `src/utils/hooks/execHttpHook.ts`
- `src/utils/hooks/execPromptHook.ts`
- `src/utils/hooks/postSamplingHooks.ts`
- `src/utils/hooks/sessionHooks.ts`
- `src/utils/hooks.ts` — Aggregate hook execution

**Interactions with Other Domains**

- Tool Framework: PreToolUse / PostToolUse
- Permissions: Hook output can approve/block tool calls
- Conversation Engine: SessionStart/SessionEnd/Stop hooks
- Plugin System: Plugins register hooks via hooks.json
- Analytics: Hook execution events

**Complexity:** High

---

### Domain 23: IDE Integration

**Purpose and Scope**

The IDE Integration domain enables Claude Code to work alongside IDE extensions for VS Code, JetBrains IDEs, and other editors. The integration provides bidirectional communication: the IDE can send file selections and diagnostics to Claude; Claude can open files and show diffs in the IDE.

**Connection mechanism**: IDE extensions run an MCP SSE server locally. Claude Code connects to it via the `sse-ide` or `ws-ide` transport types. The connection is detected automatically via port scanning and process ancestry analysis.

**IDE utilities** (`src/utils/ide.ts`): Comprehensive IDE detection and interaction:
- `getConnectedIdeClient()` — Returns the active IDE MCP connection
- `callIdeRpc()` — Calls an IDE-specific RPC method
- `openFileInIde()` — Opens a file in the editor
- `showDiffInIde()` — Shows a before/after diff

**IDE-specific features**:
- File selection injection: `src/hooks/useIdeSelection.ts` watches for IDE text selections and injects them as context
- At-mention: `src/hooks/useIdeAtMentioned.ts` handles `@` mentions of files from the IDE
- Status indicators: `src/hooks/notifs/useIDEStatusIndicator.tsx` shows connection status
- Diagnostic tracking: `src/services/diagnosticTracking.ts` uses the IDE MCP connection to read diagnostics

**Auto-connect**: `src/components/IdeAutoConnectDialog.tsx` prompts to connect when an IDE extension is detected.

**VS Code SDK MCP** (`src/services/mcp/vscodeSdkMcp.ts`): An in-process MCP server that the VS Code extension can use directly (no port required).

**Path conversion** (`src/utils/idePathConversion.ts`): Handles WSL ↔ Windows path conversion for WSL-based IDE setups.

**Key Source Files**

- `src/utils/ide.ts` — IDE detection and RPC calls
- `src/hooks/useIdeSelection.ts` — IDE selection injection
- `src/hooks/useIDEIntegration.tsx` — Main IDE integration hook
- `src/services/mcp/vscodeSdkMcp.ts` — VS Code in-process MCP
- `src/services/diagnosticTracking.ts` — IDE diagnostics
- `src/utils/idePathConversion.ts` — WSL path conversion
- `src/utils/jetbrains.ts` — JetBrains-specific detection

**Interactions with Other Domains**

- MCP: IDE connection is a special MCP transport
- LSP: IDE may provide the LSP server
- File Operations: IDE file opening/diff display
- Permissions: IDE-sourced file edits

**Complexity:** Medium

---

### Domain 24: Computer Use

**Purpose and Scope**

The Computer Use domain (macOS only, ant-internal) enables Claude Code to control the computer — taking screenshots, moving the mouse, clicking, and typing — using a macOS-native control layer.

**Architecture**: Computer use is exposed as an MCP server (`computer-use`) via `src/utils/computerUse/mcpServer.ts`. The CLI spawns this MCP server as a subprocess, which wraps a native Swift/Objective-C layer via `src/utils/computerUse/swiftLoader.ts`.

**Capabilities**: Screenshot capture (with app filtering), mouse movement/click, keyboard input, application focus.

**Security**: `src/utils/computerUse/escHotkey.ts` provides an emergency escape hotkey. `src/utils/computerUse/computerUseLock.ts` prevents concurrent computer use operations. `src/utils/computerUse/gates.ts` checks feature flag and system permissions.

**Host adapter**: `src/utils/computerUse/hostAdapter.ts` creates the executor appropriate for the current platform/terminal.

**Plugin**: Computer use is registered as a built-in plugin (`src/plugins/builtinPlugins.ts`), enabling users to disable it.

**Key Source Files**

- `src/utils/computerUse/executor.ts` — macOS click/keyboard execution
- `src/utils/computerUse/mcpServer.ts` — MCP server wrapper
- `src/utils/computerUse/hostAdapter.ts` — Executor factory
- `src/utils/computerUse/gates.ts` — Feature gate checks
- `src/utils/computerUse/escHotkey.ts` — Emergency escape
- `src/utils/computerUse/common.ts` — Shared constants and terminal bundle ID

**Interactions with Other Domains**

- MCP: Exposed as an MCP server
- Plugin System: Registered as a built-in plugin
- Analytics: Computer use events

**Complexity:** Medium

---

### Domain 25: Voice Input

**Purpose and Scope**

The Voice Input domain (ant-internal, `VOICE_MODE` feature gate) enables speech-to-text input for Claude Code. Users can speak their prompts instead of typing.

**State management** (`src/context/voice.tsx`): A React context and store manage voice state: `idle | recording | processing`, audio levels, interim transcript, and errors.

**Voice provider**: The `VoiceProvider` context wraps the application when `VOICE_MODE` is enabled. In external builds, it degrades to a passthrough.

**Audio levels**: `voiceAudioLevels: number[]` provides real-time audio level data for animating a waveform in the UI.

**Key Source Files**

- `src/context/voice.tsx` — Voice state management
- `src/hooks/useVimInput.ts` — Voice mode activation via input handling

**Interactions with Other Domains**

- UI/UX Terminal: Voice recording UI
- Conversation Engine: Transcribed text submitted as user message

**Complexity:** Low

---

### Domain 26: GitHub App & CI/CD Integration

**Purpose and Scope**

The GitHub App domain enables Claude Code to be integrated into GitHub repositories for automated code review and PR assistance via GitHub Actions.

**Installation wizard** (`src/commands/install-github-app/`): A multi-step Ink-based UI wizard that guides the user through:
1. Verifying GitHub CLI is installed and authenticated
2. Selecting the target repository
3. Checking for existing secrets and workflows
4. Installing the Claude Code GitHub App to the repository
5. Creating a GitHub Actions workflow file
6. Generating an API key and storing it as a GitHub Secret

**Workflow generation** (`src/commands/install-github-app/setupGitHubActions.ts`): Creates `.github/workflows/claude.yml` with the appropriate workflow triggers (PR, push, issue comment). Supports both API key and OAuth token authentication.

**Slack App** (`src/commands/install-slack-app/`): Similar wizard for installing Claude Code as a Slack bot.

**Autofix PR** (`src/commands/autofix-pr/`): Tool for automatically fixing PR review comments.

**Key Source Files**

- `src/commands/install-github-app/install-github-app.tsx` — Main wizard component
- `src/commands/install-github-app/setupGitHubActions.ts` — Workflow file creation
- `src/constants/github-app.ts` — Workflow templates
- `src/commands/install-slack-app/` — Slack app installation

**Interactions with Other Domains**

- Authentication: Uses existing OAuth tokens for GitHub API
- Analytics: Installation events
- Configuration: Stores GitHub App installation state

**Complexity:** Low

---

### Domain 27: Skill System

**Purpose and Scope**

The Skill System provides reusable, composable prompt templates that extend Claude Code's capabilities without writing code. Skills are markdown files that define a specialized agent behavior, discoverable via a `SkillTool` search interface.

**Bundled skills** (`src/skills/bundled/`): Pre-packaged skills that ship with the CLI:
- `batch.ts` — Run multiple commands in batch
- `claudeApi.ts` — Call the Anthropic API directly
- `debug.ts` — Debugging helper
- `keybindings.ts` — Custom keybinding setup
- `loop.ts` — Retry/loop until condition met
- `remember.ts` — Save to memory
- `scheduleRemoteAgents.ts` — Schedule background agents
- `simplify.ts` — Simplify a complex prompt
- `skillify.ts` — Convert a pattern into a skill
- `stuck.ts` — Unstick Claude from a loop
- `verify.ts` — Verify code correctness

**Skill loading** (`src/skills/loadSkillsDir.ts`): Scans plugin directories and the user's skill directory for `.md` skill files. Parses YAML frontmatter for metadata.

**SkillTool** (`src/tools/SkillTool/`): A tool that lets the model search and invoke skills by name. The model can discover skills via `ToolSearch` and invoke them as slash commands.

**Skill hooks** (`src/utils/hooks/registerSkillHooks.ts`): Skills can define hooks in their frontmatter that fire automatically.

**Key Source Files**

- `src/skills/bundledSkills.ts` — `BundledSkillDefinition` type and registry
- `src/skills/bundled/` — Built-in skill implementations
- `src/skills/loadSkillsDir.ts` — Disk-based skill loading
- `src/tools/SkillTool/` — Model-facing skill invocation tool
- `src/utils/hooks/registerSkillHooks.ts`

**Main Types/Interfaces**

- `BundledSkillDefinition` — Programmatic skill definition with `getPromptForCommand()`

**Interactions with Other Domains**

- Plugin System: Plugins can provide skills
- Hook System: Skills register hooks
- Tool Framework: Skills invoked via SkillTool
- Conversation Engine: Skills inject prompts as user messages

**Complexity:** Medium

---

### Domain 28: Web & Search Tools

**Purpose and Scope**

The Web & Search domain provides capabilities to interact with the internet: fetching web pages and performing web searches.

**WebFetchTool** (`src/tools/WebFetchTool/`): Fetches a URL and returns its content. Uses Playwright/puppeteer for JavaScript-rendered pages when needed. Strips HTML to markdown for readability. Pre-approved domains bypass permission prompts.

**WebSearchTool** (`src/tools/WebSearchTool/`): Performs web searches using Anthropic's hosted search provider. Returns formatted search results with links. Only available in the US.

**PromptSuggestion** (`src/services/PromptSuggestion/`): Generates speculative prompt suggestions as the user types, using a lightweight model for real-time typeahead.

**AgentSummary** (`src/services/AgentSummary/`): Generates one-line summaries of agent activity for status display.

**Key Source Files**

- `src/tools/WebFetchTool/WebFetchTool.ts`
- `src/tools/WebFetchTool/preapproved.ts` — Pre-approved domain list
- `src/tools/WebSearchTool/` — Web search tool
- `src/services/PromptSuggestion/promptSuggestion.ts`
- `src/services/AgentSummary/agentSummary.ts`

**Interactions with Other Domains**

- Permissions: Web fetch requires permission checks
- Analytics: Web fetch/search events

**Complexity:** Low

---

### Domain 29: Bootstrap & Global State

**Purpose and Scope**

The Bootstrap domain manages process-level initialization and the single source of global singleton state. It is intentionally kept as a leaf module with minimal imports to avoid circular dependencies.

**Global state** (`src/bootstrap/state.ts`): A flat mutable singleton storing:
- Session ID, parent session ID
- Cost/token counters (totalCostUSD, totalAPIDuration, model usage by model)
- Working directory (cwd, originalCwd, projectRoot)
- OpenTelemetry providers (meter, tracerProvider, loggerProvider)
- Feature flags (kairosActive, strictToolResultPairing)
- Session configuration (sessionPersistenceDisabled, sessionBypassPermissionsMode)
- Registered hooks (SDK callbacks + plugin hooks)
- Agent color assignments
- Invoked skills tracking
- Cron tasks
- Session teams (for cleanup)

The state is accessed via typed getter/setter functions (`getSessionId()`, `addToTotalCostState()`, etc.) rather than direct object access.

**Bootstrap flow** (`src/entrypoints/cli.tsx`): The CLI entry point handles:
1. Fast-path flag handling (`--version`, `--dump-system-prompt`)
2. Startup profiler initialization
3. Chrome native host / Claude in Chrome mode
4. Full CLI loading via `commander`

**Init sequence** (`src/entrypoints/init.ts`, `src/commands/init.ts`): Sets up analytics, loads settings, initializes GrowthBook, starts auto-updater, configures LSP manager.

**Key Source Files**

- `src/bootstrap/state.ts` — Global singleton state
- `src/entrypoints/cli.tsx` — CLI entry point
- `src/entrypoints/init.ts` — SDK initialization
- `src/commands/init.ts` — REPL initialization
- `src/entrypoints/agentSdkTypes.ts` — SDK public API types

**Main Types/Interfaces**

- The `State` object in `src/bootstrap/state.ts` (internal, accessed via typed getters)
- `HookEvent` — All valid hook event names

**Interactions with Other Domains**

- All domains: Global state is read-only singleton accessed everywhere
- Analytics: OTel providers stored here
- Authentication: Session ingress token
- Agent Orchestration: Agent color map, invoked skills

**Complexity:** Medium

---

### Domain 30: Buddy / Companion System

**Purpose and Scope**

The Buddy (Companion) domain implements an animated ASCII creature companion that appears in the terminal UI. It is a delightful UX feature that shows a small animated sprite alongside Claude Code's responses.

**Companion types**: Each user gets a deterministically assigned companion based on their session ID. The companion has a species, rarity, hat, eyes, and stats. The seeded PRNG (`mulberry32`) ensures the same user always gets the same companion.

**Sprites** (`src/buddy/sprites.ts`): ASCII art frames for different species and animations.

**Notifications** (`src/buddy/useBuddyNotification.tsx`): The companion reacts to events (new messages, tool calls, errors) with different animations.

**Key Source Files**

- `src/buddy/companion.ts` — Companion generation from seed
- `src/buddy/types.ts` — Companion type definitions
- `src/buddy/sprites.ts` — ASCII art sprites
- `src/buddy/CompanionSprite.tsx` — React rendering component
- `src/buddy/useBuddyNotification.tsx` — Event-driven animations

**Interactions with Other Domains**

- UI/UX Terminal: Rendered via Ink

**Complexity:** Low

---

## Domain Interaction Map

The following describes the key dependency relationships between domains. Arrows indicate "depends on" / "calls into":

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ENTRY POINTS                                     │
│    CLI Entrypoint ──► Bootstrap & Global State                         │
│    SDK/QueryEngine ──► Bootstrap & Global State                        │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   CONVERSATION ENGINE (Core Loop)                       │
│    query.ts / QueryEngine.ts                                            │
│    ┌──────────────────────────────────────────────────────────────┐     │
│    │  System Prompt Assembly ──► Memory System, Config, Plugins   │     │
│    │  API Call ──► Authentication, API Client                     │     │
│    │  Tool Dispatch ──► Tool Framework (all tools)                │     │
│    │  Permission Gate ──► Permissions & Safety                    │     │
│    │  Context Check ──► Context Management & Compaction           │     │
│    │  Transcript ──► Session Management                           │     │
│    │  Events ──► Analytics & Telemetry                            │     │
│    └──────────────────────────────────────────────────────────────┘     │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
              ┌───────────────┼────────────────────┐
              ▼               ▼                    ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│  TOOL FRAMEWORK │ │ AGENT ORCHESTRA-│ │  CONTEXT MANAGEMENT │
│  (36+ tools)    │ │ TION & SWARMS   │ │  & COMPACTION       │
│                 │ │                 │ │                     │
│  File Ops       │ │  AgentTool      │ │  Auto-compact       │
│  Bash/Shell     │ │  TeamCreate     │ │  Micro-compact      │
│  LSP            │ │  Coordinator    │ │  Token counting     │
│  MCP Wrapping   │ │  Swarm backends │ │                     │
│  Web/Search     │ └─────────────────┘ └─────────────────────┘
│  Scheduling     │
└────────┬────────┘
         │
    ┌────┴────────────────────────────────────┐
    │                                         │
    ▼                                         ▼
┌─────────────────┐                 ┌─────────────────────┐
│  PERMISSIONS &  │                 │  MCP PROTOCOL       │
│  SAFETY         │                 │                     │
│                 │                 │  Connections        │
│  Rule matching  │                 │  Tool wrapping      │
│  Classifier     │                 │  OAuth auth         │
│  Sandbox        │                 │  Resources          │
└─────────────────┘                 └─────────────────────┘

SUPPORTING INFRASTRUCTURE (consumed by all domains above):
┌─────────────────────────────────────────────────────────────────────────┐
│  Authentication  │  Configuration  │  Analytics/Telemetry  │  Session   │
│  Memory System   │  Hook System    │  Plugin System        │  Storage   │
│  Version Control │  IDE Integration│  Task Management      │  LSP       │
│  Cost & Billing  │  Auto-Update    │  Bootstrap/State      │  UI/UX     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Cross-Cutting Concerns

### Feature Flags (GrowthBook + `bun:bundle`)
Two types of feature gates:
1. **Build-time DCE**: `feature('FEATURE_NAME')` from `bun:bundle` — evaluated at bundle time. Ant-only code is tree-shaken from external builds. Examples: `VOICE_MODE`, `COORDINATOR_MODE`, `TRANSCRIPT_CLASSIFIER`, `KAIROS`, `TEAMMEM`.
2. **Runtime feature flags**: GrowthBook-based flags fetched at startup and cached. Examples: `tengu_auto_compact`, `tengu_session_memory`, `tengu_tool_search`.

### Error Handling
- `ClaudeError` base class in `src/utils/errors.ts`
- `AbortError` for user interruptions
- `APIError` hierarchy for Anthropic API errors
- Tool-level errors return `ToolResult<Output>` — errors are surfaced as error tool results to the model
- Unhandled errors bubble to the REPL or crash the process with cleanup

### Concurrency & Abort
- Every tool call receives an `AbortController` from `ToolUseContext`
- `createAbortController()` in `src/utils/abortController.ts` creates process-exit-aware controllers
- `combinedAbortSignal.ts` merges multiple abort signals

### Logging
- `logForDebugging()` — development-only verbose logging to stderr
- `logError()` — structured error logging
- `logForDiagnosticsNoPII()` — PII-safe diagnostic logging
- `logEvent()` / `logEventAsync()` — analytics event logging

### Path/Filesystem Abstraction
- `getFsImplementation()` — Returns the appropriate fs module (real vs. test mock)
- `getCwd()` — Current working directory (may differ from `process.cwd()` when worktrees are used)
- Path validation in permission system (`src/utils/permissions/pathValidation.ts`)

---

## Domain Registry

### Domain: Conversation Engine
- **Scope**: Core agentic loop — message processing, API calls, tool dispatch, streaming
- **Key Directories**: `src/query.ts`, `src/QueryEngine.ts`, `src/services/api/`
- **Complexity**: Very High
- **Key Dependencies**: Tool Framework, Authentication, Context Management, Session Management, Permissions, Memory System, Analytics

### Domain: Tool Framework & Ecosystem
- **Scope**: Universal tool contract, buildTool factory, 36+ built-in tool implementations
- **Key Directories**: `src/Tool.ts`, `src/tools/`
- **Complexity**: Very High
- **Key Dependencies**: Permissions, Analytics, UI/UX Terminal, Hook System

### Domain: File Operations
- **Scope**: FileRead, FileWrite, FileEdit, Glob, Grep, Notebook edit with safety and undo
- **Key Directories**: `src/tools/FileReadTool/`, `src/tools/FileWriteTool/`, `src/tools/FileEditTool/`, `src/tools/GlobTool/`, `src/tools/GrepTool/`, `src/utils/fileStateCache.ts`, `src/utils/fileHistory.ts`
- **Complexity**: High
- **Key Dependencies**: Permissions, Session Management, Analytics, Git

### Domain: Shell & Bash Execution
- **Scope**: BashTool, PowerShellTool, bash AST parsing, sandboxing, security validation
- **Key Directories**: `src/tools/BashTool/`, `src/tools/PowerShellTool/`, `src/utils/bash/`, `src/utils/Shell.ts`, `src/utils/sandbox/`
- **Complexity**: Very High
- **Key Dependencies**: Permissions, Analytics, Task Management

### Domain: Agent Orchestration & Swarms
- **Scope**: AgentTool, fork/background/resume, coordinator mode, in-process/pane teammates, team files
- **Key Directories**: `src/tools/AgentTool/`, `src/coordinator/`, `src/utils/swarm/`, `src/tasks/LocalAgentTask/`, `src/tasks/InProcessTeammateTask/`
- **Complexity**: Very High
- **Key Dependencies**: Conversation Engine, Task Management, Session Management, Hook System, Permissions, Analytics

### Domain: MCP (Model Context Protocol)
- **Scope**: MCP client connections (stdio/SSE/HTTP/WS/SDK), tool wrapping, OAuth, resources, XAA
- **Key Directories**: `src/services/mcp/`, `src/tools/MCPTool/`, `src/tools/McpAuthTool/`, `src/tools/ListMcpResourcesTool/`, `src/tools/ReadMcpResourceTool/`
- **Complexity**: High
- **Key Dependencies**: Authentication, Configuration, Tool Framework, Plugin System

### Domain: Session Management & Storage
- **Scope**: JSONL transcript persistence, resume, teleport, session search, file history, sidechain transcripts
- **Key Directories**: `src/utils/sessionStorage.ts`, `src/utils/fileHistory.ts`, `src/types/logs.ts`, `src/utils/agenticSessionSearch.ts`
- **Complexity**: High
- **Key Dependencies**: Authentication, Analytics, Conversation Engine

### Domain: Authentication & Security
- **Scope**: API keys, OAuth 2.0 + PKCE, AWS STS, secure storage (Keychain), token refresh
- **Key Directories**: `src/utils/auth.ts`, `src/services/oauth/`, `src/utils/secureStorage/`, `src/utils/aws.ts`
- **Complexity**: High
- **Key Dependencies**: Configuration, Analytics, Bridge

### Domain: Permissions & Safety
- **Scope**: Allow/deny/ask rules, permission modes, auto-mode classifier, sandbox, denial tracking
- **Key Directories**: `src/utils/permissions/`, `src/utils/sandbox/`, `src/hooks/toolPermission/`
- **Complexity**: Very High
- **Key Dependencies**: Configuration, Analytics, Tool Framework, Hook System

### Domain: Context Management & Compaction
- **Scope**: Token counting, auto-compact trigger, compaction algorithm, snip compaction, token warnings
- **Key Directories**: `src/services/compact/`, `src/utils/analyzeContext.ts`, `src/utils/context.ts`
- **Complexity**: High
- **Key Dependencies**: Conversation Engine, Memory System, Analytics, Session Management

### Domain: Memory System
- **Scope**: CLAUDE.md hierarchy, session memory, MEMORY.md auto-memory, auto-dream consolidation, magic docs
- **Key Directories**: `src/utils/claudemd.ts`, `src/memdir/`, `src/services/SessionMemory/`, `src/services/autoDream/`, `src/services/extractMemories/`, `src/services/MagicDocs/`
- **Complexity**: High
- **Key Dependencies**: Configuration, Conversation Engine, Analytics, File Operations

### Domain: Plugin & Extension System
- **Scope**: Plugin loading, marketplace management, install/update reconciliation, built-in plugins
- **Key Directories**: `src/utils/plugins/`, `src/services/plugins/`, `src/types/plugin.ts`, `src/plugins/`
- **Complexity**: High
- **Key Dependencies**: Configuration, MCP, Hook System, Skill System, Analytics

### Domain: UI/UX Terminal (Ink)
- **Scope**: Custom React terminal renderer, Yoga layout, ANSI output, 100+ components, vim mode, input handling
- **Key Directories**: `src/ink/`, `src/components/`, `src/hooks/`
- **Complexity**: Very High
- **Key Dependencies**: Tool Framework (rendering), Configuration (theming), Permissions (dialog UI)

### Domain: Configuration System
- **Scope**: Settings hierarchy (managed/enterprise/user/project/local), MDM, live reload, CLAUDE.md
- **Key Directories**: `src/utils/settings/`, `src/utils/config.ts`, `src/utils/claudemd.ts`
- **Complexity**: High
- **Key Dependencies**: Authentication, Permissions, Hook System, MCP, Plugin System

### Domain: Analytics & Telemetry
- **Scope**: Event logging (sink queue model), Datadog metrics, 1P BigQuery, GrowthBook feature flags, OTel tracing, Perfetto
- **Key Directories**: `src/services/analytics/`, `src/utils/telemetry/`, `src/cost-tracker.ts`
- **Complexity**: High
- **Key Dependencies**: Bootstrap (OTel providers), Authentication (user attributes)

### Domain: Remote Bridge & Desktop Integration
- **Scope**: Bridge server (poll for work, spawn sessions), CCR v1/v2 transports (SSE/WS), structured I/O, desktop handoff
- **Key Directories**: `src/bridge/`, `src/cli/transports/`, `src/cli/remoteIO.ts`
- **Complexity**: High
- **Key Dependencies**: Authentication, Session Management, Conversation Engine, Permissions, Analytics

### Domain: LSP & Code Intelligence
- **Scope**: Language Server Protocol client, multi-server management, hover/definition/diagnostics, passive feedback
- **Key Directories**: `src/services/lsp/`, `src/tools/LSPTool/`, `src/services/diagnosticTracking.ts`
- **Complexity**: Medium
- **Key Dependencies**: File Operations, Plugin System, IDE Integration

### Domain: Version Control (Git)
- **Scope**: Commit attribution, worktree management, gitignore filtering, GitHub App CI/CD setup
- **Key Directories**: `src/utils/git.ts`, `src/utils/git/`, `src/utils/commitAttribution.ts`, `src/tools/EnterWorktreeTool/`, `src/tools/ExitWorktreeTool/`, `src/commands/install-github-app/`
- **Complexity**: Medium
- **Key Dependencies**: File Operations, Session Management, Analytics, Bridge

### Domain: Cost & Billing
- **Scope**: Token/cost aggregation, USD cost calculation, rate limit display, billing access control
- **Key Directories**: `src/cost-tracker.ts`, `src/utils/billing.ts`, `src/utils/modelCost.ts`, `src/services/claudeAiLimits.ts`
- **Complexity**: Medium
- **Key Dependencies**: Authentication, Analytics, Conversation Engine, Configuration

### Domain: Auto-Update System
- **Scope**: Version detection, npm/native/local installer, release channels, minimum version enforcement
- **Key Directories**: `src/utils/autoUpdater.ts`, `src/utils/nativeInstaller/`, `src/cli/update.ts`
- **Complexity**: Medium
- **Key Dependencies**: Analytics, Configuration, UI/UX Terminal

### Domain: Task Management
- **Scope**: Background task registry (bash/agent/teammate/dream), disk output polling, cron scheduling, todo tracking
- **Key Directories**: `src/Task.ts`, `src/tasks/`, `src/utils/task/`, `src/tools/ScheduleCronTool/`, `src/tools/TaskCreateTool/` etc., `src/tools/TodoWriteTool/`
- **Complexity**: High
- **Key Dependencies**: Agent Orchestration, Bash Execution, Conversation Engine, Session Management

### Domain: Hook System
- **Scope**: 28 hook events, 4 hook types (command/prompt/agent/http), frontmatter hooks, skill hooks, SSRF guard
- **Key Directories**: `src/schemas/hooks.ts`, `src/types/hooks.ts`, `src/utils/hooks/`
- **Complexity**: High
- **Key Dependencies**: Tool Framework, Permissions, Plugin System, Analytics

### Domain: IDE Integration
- **Scope**: VS Code/JetBrains auto-connect, file selection injection, diff display, diagnostic tracking, WSL path conversion
- **Key Directories**: `src/utils/ide.ts`, `src/hooks/useIdeSelection.ts`, `src/services/mcp/vscodeSdkMcp.ts`, `src/services/diagnosticTracking.ts`
- **Complexity**: Medium
- **Key Dependencies**: MCP, LSP, File Operations

### Domain: Computer Use
- **Scope**: macOS-native screen control (screenshot, click, keyboard) via Swift/MCP wrapper
- **Key Directories**: `src/utils/computerUse/`
- **Complexity**: Medium
- **Key Dependencies**: MCP, Plugin System, Analytics

### Domain: Voice Input
- **Scope**: Speech-to-text input (ant-internal), voice state management, audio level visualization
- **Key Directories**: `src/context/voice.tsx`
- **Complexity**: Low
- **Key Dependencies**: UI/UX Terminal, Conversation Engine

### Domain: GitHub App & CI/CD Integration
- **Scope**: GitHub App installation wizard, Actions workflow generation, Slack bot setup
- **Key Directories**: `src/commands/install-github-app/`, `src/commands/install-slack-app/`, `src/constants/github-app.ts`
- **Complexity**: Low
- **Key Dependencies**: Authentication, Analytics, Configuration

### Domain: Skill System
- **Scope**: Reusable prompt templates (bundled + disk-based), SkillTool discovery, skill hooks
- **Key Directories**: `src/skills/`, `src/tools/SkillTool/`
- **Complexity**: Medium
- **Key Dependencies**: Plugin System, Hook System, Tool Framework, Conversation Engine

### Domain: Web & Search Tools
- **Scope**: WebFetchTool (URL fetch + markdown), WebSearchTool (Anthropic-hosted search), prompt suggestions
- **Key Directories**: `src/tools/WebFetchTool/`, `src/tools/WebSearchTool/`, `src/services/PromptSuggestion/`
- **Complexity**: Low
- **Key Dependencies**: Permissions, Analytics

### Domain: Bootstrap & Global State
- **Scope**: Process-level singleton state (cost counters, session ID, OTel providers, registered hooks)
- **Key Directories**: `src/bootstrap/state.ts`, `src/entrypoints/`
- **Complexity**: Medium
- **Key Dependencies**: (leaf module — minimal dependencies)

### Domain: Buddy / Companion System
- **Scope**: ASCII companion creature with seeded-random assignment, animation, and event reactions
- **Key Directories**: `src/buddy/`
- **Complexity**: Low
- **Key Dependencies**: UI/UX Terminal
