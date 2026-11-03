# Claude Code Extensibility: Complete Analysis

---

## 1. Overview

Claude Code exposes a deeply layered extensibility model. At the outermost ring, end-users can drop markdown files into `.claude/` directories. At the inner rings, plugin authors can ship entire feature bundles — including MCP servers, hooks, and agent definitions. At the system level, enterprise administrators can lock every surface down via managed settings. The layers are:

```
Enterprise Policy (managed-settings.json / MDM)
  └─ User Settings (~/.claude/settings.json)
       └─ Project Settings (.claude/settings.json)
            └─ Local Settings (.claude/settings.local.json)
                 └─ Plugins (marketplace-sourced, git-cloned bundles)
                      └─ Skills / Commands (markdown files in .claude/commands/ or .claude/skills/)
                           └─ Agents (markdown files in .claude/agents/)
                                └─ Hooks (shell commands / HTTP / LLM prompts wired to lifecycle events)
                                     └─ MCP Servers (external process tool servers)
```

Every extension point feeds into the same tool-permission-execution pipeline. Understanding that pipeline is the foundation for understanding all extensibility.

---

## 2. Tool System

### 2.1 The Tool Interface

**File:** `src/Tool.ts`

The `Tool<Input, Output, P>` generic interface is the fundamental extensibility contract. Every capability that the model can invoke must implement it.

**Required methods:**

| Method | Purpose |
|--------|---------|
| `call(args, context, canUseTool, parentMessage, onProgress?)` | Execute the tool, return `ToolResult<Output>` |
| `description(input, options)` | Human-readable description for permission dialogs |
| `prompt(options)` | System-prompt text teaching the model how to use the tool |
| `checkPermissions(input, context)` | Tool-specific permission logic (before the general permission system) |
| `mapToolResultToToolResultBlockParam(content, toolUseID)` | Serialize output for the Anthropic API |
| `renderToolUseMessage(input, options)` | React component for the tool-call display |
| `userFacingName(input)` | Short name for UI display |
| `toAutoClassifierInput(input)` | Compact representation for the auto-mode security classifier |
| `isConcurrencySafe(input)` | Whether parallel invocations are safe |
| `isReadOnly(input)` | Hint for permission system |
| `maxResultSizeChars` | Threshold before result is persisted to disk |

**Optional but important methods:**

| Method | Purpose |
|--------|---------|
| `validateInput(input, context)` | Fail before permission check (returns `ValidationResult`) |
| `isEnabled()` | Feature-flag gate |
| `isDestructive(input)` | Triggers extra confirmation |
| `interruptBehavior()` | `'cancel'` or `'block'` on new user message |
| `preparePermissionMatcher(input)` | Compile pattern matchers for hook `if` conditions |
| `backfillObservableInput(input)` | Mutate input before hooks/permissions see it |
| `shouldDefer` | Flag: tool is hidden until `ToolSearch` loads it |
| `alwaysLoad` | Flag: always include in context even when `shouldDefer` is default |
| `searchHint` | Short keyword phrase for `ToolSearch` scoring |
| `renderToolResultMessage(...)` | React component for the result display |
| `renderToolUseProgressMessage(...)` | React component for in-flight progress |
| `renderGroupedToolUse(...)` | Render multiple parallel calls as a group |
| `getToolUseSummary(input)` | Short summary for compact views |
| `getActivityDescription(input)` | Present-tense activity for spinner |
| `isSearchOrReadCommand(input)` | Whether UI should collapse the result |

**The `buildTool` factory** (`src/Tool.ts`) fills in safe defaults so authors only override what they need:

- `isEnabled` → `() => true`
- `isConcurrencySafe` → `() => false` (fail-closed)
- `isReadOnly` → `() => false` (fail-closed: assume writes)
- `isDestructive` → `() => false`
- `checkPermissions` → `{ behavior: 'allow', updatedInput }` (defer to general system)
- `toAutoClassifierInput` → `() => ''` (skip classifier)
- `userFacingName` → `() => def.name`

Usage pattern for every built-in tool:

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  maxResultSizeChars: 100_000,
  get inputSchema() { return myInputSchema() },
  async call(input, context) { /* ... */ },
  async description() { return '...' },
  async prompt() { return '...' },
  // ... other required methods
} satisfies ToolDef<MyInputSchema, MyOutput>)
```

### 2.2 Tool Registration

**File:** `src/tools.ts`

All built-in tools are assembled at startup by `getAllBaseTools()`. This function is the single source of truth. Tools are conditionally included based on:

- `process.env.USER_TYPE === 'ant'` — internal Anthropic tools
- `feature('FEATURE_NAME')` — compile-time feature flags (dead code elimination)
- `process.env.NODE_ENV === 'test'` — test-only tools
- Runtime helpers like `isWorktreeModeEnabled()`, `isAgentSwarmsEnabled()`
- `isTodoV2Enabled()` — dynamic task-management tools

The public entry point is `getTools(permissionContext)` which:
1. Starts with `getAllBaseTools()`
2. Removes tools blanket-denied by permission rules (`filterToolsByDenyRules`)
3. Removes REPL-primitive tools when REPL mode is active
4. Filters by each tool's `isEnabled()` method

The final merge with MCP tools happens in `assembleToolPool(permissionContext, mcpTools)` which sorts each partition for prompt-cache stability (built-ins first, then MCP tools, both sorted by name).

### 2.3 Tool Lifecycle: Validation → Permission → Execution → Result

The complete lifecycle when the model calls a tool:

```
Model outputs tool_use block
        │
        ▼
  backfillObservableInput()     ← mutate input for observers (idempotent)
        │
        ▼
  validateInput()               ← tool-specific validation (optional)
        │ failure → return error to model without permission check
        ▼
  canUseTool()                  ← the permission gate
        │
        ├─ hasPermissionsToUseTool()
        │     ├─ check alwaysAllow rules
        │     ├─ check alwaysDeny rules
        │     ├─ check mode (bypassPermissions, dontAsk, plan, acceptEdits)
        │     ├─ tool.checkPermissions()  ← tool-specific logic
        │     └─ run PreToolUse hooks
        │
        ├─ allow → proceed
        ├─ deny  → return rejection result to model
        └─ ask   → show UI dialog, await user decision
              │
              └─ user allows / denies / always-allows
                       │
                       ▼
                 tool.call(args, context, canUseTool, parentMessage, onProgress)
                       │
                       ▼
                 run PostToolUse hooks
                       │
                       ▼
                 mapToolResultToToolResultBlockParam()
                       │
                       ▼
                 result injected back to model
```

**Key files:**
- `src/hooks/useCanUseTool.tsx` — the React hook wiring the permission gate
- `src/utils/permissions/permissions.ts` — `hasPermissionsToUseTool()`
- `src/utils/hooks.ts` — `executePreToolUseHooks()`, `executePostToolUseHooks()`

### 2.4 Tool Search and Deferred Loading

**File:** `src/tools/ToolSearchTool/ToolSearchTool.ts`

When many tools are present (especially from MCP), the system can defer loading tool schemas to save prompt-cache tokens. Tools with `shouldDefer: true` are sent to the model with `defer_loading: true` in the API call. The model must call `ToolSearch` before using them.

`ToolSearchTool` supports:
- `select:<tool_name>` — direct selection by name
- `select:A,B,C` — multi-select
- keyword queries scored against tool name parts, `searchHint`, and full prompt text

Scoring weights:
- Exact MCP part match: +12
- Exact non-MCP part match: +10
- Partial MCP part: +6
- `searchHint` word-boundary match: +4
- Description word-boundary match: +2

### 2.5 MCP Tools vs. Built-In Tools

MCP tools are wrapped as `Tool` objects by `src/services/mcp/client.ts`. Each MCP tool:
- Gets a name of the form `mcp__serverName__toolName`
- Has `isMcp: true` on the tool object
- Has `mcpInfo: { serverName, toolName }` for tracing
- Has `checkPermissions` returning `{ behavior: 'passthrough' }` — defers to the general permission system
- The `call()` method makes the actual MCP JSON-RPC call via the MCP SDK client

The `MCPTool` in `src/tools/MCPTool/MCPTool.ts` is a template that `client.ts` clones and overrides for each discovered MCP tool.

---

## 3. Plugin System

### 3.1 Plugin Concepts

A **plugin** is a versioned bundle — typically a git repository — that can provide any combination of:
- Slash commands / skills (markdown files in `commands/` or `skills/`)
- Agent definitions (markdown files in `agents/`)
- Hooks (in `hooks/hooks.json` or `plugin.json`)
- MCP servers (in `plugin.json` `mcpServers` field)
- LSP servers (in `plugin.json` `lspServers` field)
- Output styles (in `output-styles/`)

Plugins are identified by `{name}@{marketplace}` format (e.g., `slack@anthropic`).

### 3.2 Plugin Loading Pipeline

**Files:**
- `src/utils/plugins/pluginLoader.ts` — main loader
- `src/types/plugin.ts` — `LoadedPlugin`, `PluginError`, `PluginLoadResult`
- `src/utils/plugins/schemas.ts` — `PluginManifestSchema`, `PluginHooksSchema`

The loading pipeline:

```
loadAllPlugins()
    │
    ├─ loadBuiltinPlugins()         ← registered via registerBuiltinPlugin()
    │
    ├─ loadInlinePlugins()          ← from --plugin-dir CLI flag / SDK option
    │
    └─ loadMarketplacePlugins()     ← from enabledPlugins in settings
           │
           ├─ resolve marketplace source (github, git, url, npm, settings)
           ├─ check policy gates (strictKnownMarketplaces, blockedMarketplaces)
           ├─ clone or update git repo to cache dir
           ├─ read plugin.json manifest
           ├─ validate with PluginManifestSchema (Zod)
           ├─ collect enabled/disabled state from user settings
           └─ build LoadedPlugin object with all paths
```

A `LoadedPlugin` contains:
```typescript
{
  name: string
  manifest: PluginManifest       // from plugin.json
  path: string                   // root dir on filesystem
  source: string                 // "{name}@{marketplace}"
  repository: string             // same as source
  enabled?: boolean
  isBuiltin?: boolean
  commandsPath?: string          // path to commands/ dir
  commandsPaths?: string[]       // multiple command dirs
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  hooksConfig?: HooksSettings    // from hooks/hooks.json
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>  // user-configured plugin options
}
```

### 3.3 Plugin Directory Structure

```
my-plugin/
├── plugin.json           # Manifest (optional but recommended)
├── commands/             # Slash commands
│   ├── build.md          # /plugin-name:build
│   └── deploy.md         # /plugin-name:deploy
├── skills/               # Skills (invokable by model via SkillTool)
│   └── my-workflow/
│       └── SKILL.md      # /plugin-name:my-workflow
├── agents/               # Custom agent definitions
│   └── test-runner.md    # subagent_type: "plugin-name:test-runner"
├── hooks/
│   └── hooks.json        # Hook configuration
└── output-styles/        # Custom output style definitions
```

### 3.4 Plugin Manifest (plugin.json)

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "My plugin description",
  "author": { "name": "My Name", "url": "https://example.com" },
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"]
    }
  },
  "hooks": "./hooks/extra-hooks.json",
  "dependencies": ["other-plugin@anthropic"],
  "userConfig": {
    "apiKey": {
      "type": "string",
      "description": "API key for the service",
      "sensitive": true
    }
  }
}
```

User-configurable values are accessed in hooks via `${user_config.apiKey}` or `CLAUDE_PLUGIN_OPTION_APIKEY` env var.

### 3.5 Built-in Plugins

**Files:**
- `src/plugins/builtinPlugins.ts` — registry and utilities
- `src/plugins/bundled/index.ts` — `initBuiltinPlugins()` called at startup

Built-in plugins use the sentinel ID `{name}@builtin`. They appear in `/plugin` UI with toggleable enable/disable. To add a new built-in plugin:

```typescript
// In src/plugins/bundled/index.ts
import { registerBuiltinPlugin } from '../builtinPlugins.js'

export function initBuiltinPlugins(): void {
  registerBuiltinPlugin({
    name: 'my-feature',
    description: 'Describes what my feature provides',
    defaultEnabled: true,
    skills: [/* BundledSkillDefinition[] */],
    hooks: { PreToolUse: [/* matchers */] },
    mcpServers: { 'my-server': { /* McpServerConfig */ } },
  })
}
```

### 3.6 Plugin Marketplace

Marketplaces are registries of available plugins. A marketplace is either:
- **Official** (names in `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`) — auto-updates by default
- **Third-party** — discovered via `extraKnownMarketplaces` in settings

Marketplace sources: `github:org/repo`, `git:url`, `url:https://...`, `npm:package`, `settings:name` (inline in settings.json).

Enterprise policy fields:
- `strictKnownMarketplaces` — allowlist of sources; blocks all others
- `blockedMarketplaces` — denylist of sources
- `strictPluginOnlyCustomization` — forces all customizations (skills, agents, hooks, mcp) through plugins only

---

## 4. Skill System

### 4.1 What Skills Are

Skills are **markdown prompt templates** invocable by the model via the `SkillTool`. They differ from slash commands in that they are model-invoked (not user-invoked) and run in a forked sub-agent context.

A skill is a `Command` object with `type: 'prompt'` and a `getPromptForCommand(args, context)` method.

### 4.2 Skill Discovery

**File:** `src/skills/loadSkillsDir.ts`

Skills are discovered from these sources (in priority order):

| Source | Path | `loadedFrom` value |
|--------|------|-------------------|
| Managed (enterprise) | `/etc/claude-code/.claude/skills/` | `'managed'` |
| User global | `~/.claude/skills/` | `'skills'` |
| Project shared | `.claude/skills/` (walked up to home) | `'skills'` |
| Plugin skills | `{plugin-root}/skills/` | `'plugin'` |
| Bundled (in binary) | registered via `registerBundledSkill()` | `'bundled'` |
| MCP prompts | from connected MCP servers | `'mcp'` |

`getSkillDirCommands(cwd)` walks all these directories, parsing each `.md` file's frontmatter. Results are memoized per session.

### 4.3 Skill Frontmatter

A skill markdown file begins with a YAML frontmatter block:

```markdown
---
description: "Runs unit tests and reports failures"
argument-hint: "[test-pattern]"
when_to_use: "Use this when the user wants to run tests"
allowed-tools: ["Bash", "Read"]
model: claude-haiku-4-5
effort: low
context: fork
agent: general-purpose
user-invocable: true
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo 'About to run bash'"
paths:
  - "src/**"
  - "test/**"
---

Run the tests matching the pattern $ARGUMENTS using `bun test`.
```

All frontmatter fields:

| Field | Type | Purpose |
|-------|------|---------|
| `description` | string | Shown in skill listings and prompt |
| `argument-hint` | string | Hint shown in autocomplete |
| `when_to_use` | string | Guidance for model selection |
| `allowed-tools` | string[] | Restrict which tools are available |
| `model` | string | Override model for this skill |
| `effort` | string/int | `low`, `medium`, `high`, `max` or integer |
| `context` | `'fork'` | Execute in isolated fork agent |
| `agent` | string | Agent type to use for execution |
| `user-invocable` | bool | Whether user can invoke via slash command |
| `disable-model-invocation` | bool | Run without LLM (pure command) |
| `hooks` | HooksSettings | Skill-scoped hooks |
| `paths` | string[] | Glob patterns to gate when skill is active |
| `arguments` | string/string[] | Named argument substitution |
| `version` | string | Version string for tracking |
| `name` | string | Display name override |
| `shell` | object | Shell configuration for shell-mode execution |

### 4.4 Bundled Skills

**Files:** `src/skills/bundledSkills.ts`, `src/skills/bundled/`

Bundled skills ship inside the CLI binary. They are registered at startup via `registerBundledSkill(definition: BundledSkillDefinition)`.

The `BundledSkillDefinition` type:

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // Reference files extracted to disk on first use
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

### 4.5 SkillTool Execution

**File:** `src/tools/SkillTool/SkillTool.ts`

The `SkillTool` (`name: 'Skill'`) accepts `{ skill: string, args?: string }` and:

1. Looks up the command via `getAllCommands(context)` (includes local, bundled, and MCP skills)
2. Validates the skill exists and is invocable
3. Checks permissions (same path as other tools)
4. Calls `executeForkedSkill()` which:
   - Creates a fork agent context via `prepareForkedCommandContext()`
   - Runs `runAgent()` with the skill's prompt as the initial message
   - Reports progress via `onProgress` callbacks
   - Returns the agent's final response text

Execution context options:
- `context: 'fork'` (default) — isolated sub-agent with its own token budget
- `context: 'inline'` — executed inline in the current conversation (legacy, not recommended)

---

## 5. Hook System

### 5.1 Hook Events

**File:** `src/entrypoints/sdk/coreTypes.ts`

The complete set of lifecycle events that hooks can attach to:

| Event | Trigger |
|-------|---------|
| `PreToolUse` | Before any tool executes |
| `PostToolUse` | After a tool completes successfully |
| `PostToolUseFailure` | After a tool fails |
| `UserPromptSubmit` | When the user submits a message |
| `SessionStart` | At the beginning of a new session |
| `SessionEnd` | When a session ends (clear, logout, exit) |
| `Stop` | When the model stops generating (turn end) |
| `StopFailure` | When a stop fails |
| `SubagentStart` | When a sub-agent spawns |
| `SubagentStop` | When a sub-agent finishes |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction |
| `PermissionRequest` | When the model requests a permission |
| `PermissionDenied` | When a permission is denied |
| `Notification` | When a notification fires |
| `Setup` | One-time per process (before SessionStart) |
| `TeammateIdle` | When a teammate agent becomes idle |
| `TaskCreated` | When a background task is created |
| `TaskCompleted` | When a background task completes |
| `Elicitation` | When an MCP server requests elicitation |
| `ElicitationResult` | After elicitation resolves |
| `ConfigChange` | When settings change |
| `WorktreeCreate` | When a git worktree is created |
| `WorktreeRemove` | When a git worktree is removed |
| `InstructionsLoaded` | When CLAUDE.md files are loaded |
| `CwdChanged` | When the working directory changes |
| `FileChanged` | When a watched file changes |

### 5.2 Hook Types

**File:** `src/schemas/hooks.ts`

Hooks are configured in the `hooks` section of any settings file. Four hook types exist:

**Command hook** — runs a shell command:
```json
{
  "type": "command",
  "command": "python3 /path/to/validator.py",
  "if": "Bash(git *)",
  "shell": "bash",
  "timeout": 30,
  "async": false,
  "asyncRewake": false,
  "once": false,
  "statusMessage": "Validating..."
}
```

**Prompt hook** — evaluates an LLM prompt:
```json
{
  "type": "prompt",
  "prompt": "Review the tool call: $ARGUMENTS. Return {\"continue\": false} if it's unsafe.",
  "if": "Bash(*)",
  "model": "claude-haiku-4-5",
  "timeout": 60
}
```

**Agent hook** — runs a mini-agent for verification:
```json
{
  "type": "agent",
  "prompt": "Verify that unit tests ran and passed.",
  "if": "Bash(npm test*)",
  "model": "claude-haiku-4-5",
  "timeout": 120,
  "statusMessage": "Running verification..."
}
```

**HTTP hook** — POSTs the hook input to a URL:
```json
{
  "type": "http",
  "url": "https://hooks.example.com/pre-tool",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 10
}
```

### 5.3 Hook Matcher Configuration

Hooks are grouped by event with optional matchers:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "echo 'Running bash'" }
        ]
      },
      {
        "matcher": "Write(*.py)",
        "hooks": [
          { "type": "command", "command": "python3 -m py_compile $CLAUDE_FILE_PATHS" }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          { "type": "command", "command": "cleanup.sh" }
        ]
      }
    ]
  }
}
```

The `if` field (on individual hook commands) and `matcher` (on matchers) both use permission-rule syntax: `ToolName(pattern)`. Examples: `Bash(git *)`, `Write(*.ts)`, `Read`.

### 5.4 Hook JSON Output Protocol

**File:** `src/types/hooks.ts`

Hooks communicate results back via their stdout as JSON. The schema is:

```typescript
// Sync response (default):
{
  continue?: boolean           // false = stop execution
  suppressOutput?: boolean     // true = hide stdout from transcript
  stopReason?: string          // shown when continue is false
  decision?: 'approve' | 'block'
  reason?: string
  systemMessage?: string       // warning shown to user
  hookSpecificOutput?: {
    // For PreToolUse:
    hookEventName: 'PreToolUse'
    permissionDecision?: 'allow' | 'deny' | 'ask'
    permissionDecisionReason?: string
    updatedInput?: Record<string, unknown>  // modify the tool's input
    additionalContext?: string

    // For PostToolUse:
    hookEventName: 'PostToolUse'
    additionalContext?: string
    updatedMCPToolOutput?: unknown  // overwrite MCP tool result

    // For SessionStart:
    hookEventName: 'SessionStart'
    additionalContext?: string
    initialUserMessage?: string   // inject initial message
    watchPaths?: string[]         // register FileChanged watchers

    // For UserPromptSubmit:
    hookEventName: 'UserPromptSubmit'
    additionalContext?: string    // appended as context to the prompt

    // For PermissionRequest:
    hookEventName: 'PermissionRequest'
    decision: { behavior: 'allow' | 'deny', updatedInput?, updatedPermissions? }
  }
}
```

### 5.5 Hook Execution Environment

Shell command hooks receive these environment variables:

| Variable | Value |
|----------|-------|
| `CLAUDE_SESSION_ID` | Current session UUID |
| `CLAUDE_PROJECT_DIR` | Stable project root path |
| `CLAUDE_TRANSCRIPT_PATH` | Path to session transcript JSONL |
| `CLAUDE_CWD` | Current working directory |
| `CLAUDE_PERMISSION_MODE` | Current permission mode |
| `CLAUDE_AGENT_ID` | Set for subagent hooks |
| `CLAUDE_AGENT_TYPE` | Agent type name |
| `CLAUDE_PLUGIN_ROOT` | Plugin root dir (for plugin hooks) |
| `CLAUDE_PLUGIN_DATA` | Plugin data dir (for plugin hooks) |
| `CLAUDE_PLUGIN_OPTION_*` | User-configured plugin options |
| `CLAUDE_ENV_FILE` | Path to env file for SessionStart/Setup/CwdChanged/FileChanged |

Hook input (as JSON piped to stdin or in `$ARGUMENTS`) contains the full tool call details, session info, and permission mode.

### 5.6 Async and Background Hooks

Hooks with `"async": true` detach immediately and run in the background. The `asyncRewake: true` flag additionally wakes the model if the hook exits with code 2 (blocking error), delivering its stderr as a system notification.

### 5.7 Programmatic Hooks (SDK)

The SDK allows registering hooks as JavaScript callbacks:

```typescript
// Registered via registerHookCallbacks() in bootstrap/state.ts
type HookCallback = {
  type: 'callback'
  callback: (input: HookInput, toolUseID: string | null, abort: AbortSignal) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean
}
```

Session-scoped function hooks can be registered at runtime for features like LSP integration, IDE connectivity, and background monitoring.

---

## 6. Command System

### 6.1 Slash Commands

**File:** `src/commands.ts`

Slash commands are user-facing operations invoked as `/command-name [args]`. They differ from tools in that they are user-invoked (not model-invoked) and can have rich interactive UI.

The `Command` type is a discriminated union:
- `type: 'prompt'` — sends a prompt to the model
- `type: 'local'` — runs JavaScript logic locally (returns UI)
- `type: 'local-jsx'` — renders a React component

### 6.2 Built-in Commands

Commands are registered in `COMMANDS()` in `src/commands.ts`. All ~60+ commands ship as static imports. The list is `memoize()`d since command loading requires reading config at runtime.

### 6.3 User-Defined Commands

Users can create custom commands as markdown files in:
- `~/.claude/commands/` — global user commands
- `.claude/commands/` — project-specific commands (searched up to home dir)
- Managed: `/etc/claude-code/.claude/commands/`

**File loading:** `getSkillDirCommands(cwd)` in `src/skills/loadSkillsDir.ts` scans these directories. Commands are loaded from `commands/` subdirectories and distinguished from skills (in `skills/` subdirectory).

Command frontmatter supports the same fields as skills, plus the command becomes invocable as a slash command.

### 6.4 Plugin Commands

Plugins provide commands via their `commands/` directory. Command names are namespaced: `{plugin-name}:{command-name}`. Nested directories produce `{plugin-name}:{subdir}:{name}`.

Plugin commands loaded by `getPluginCommands()` and `getPluginSkills()` in `src/utils/plugins/loadPluginCommands.ts`.

### 6.5 `getCommands(cwd)` Assembly

The full command list assembled in `getCommands(projectRoot)`:

```
COMMANDS()                      ← built-in slash commands
+ getSkillDirCommands(cwd)      ← user/project/managed skill dirs
+ getBundledSkills()            ← compiled-in skills
+ getBuiltinPluginSkillCommands() ← skills from built-in plugins
+ getPluginCommands()           ← commands from enabled plugins
+ getPluginSkills()             ← skills from enabled plugins
+ MCP prompts                   ← from AppState.mcp.commands
```

All deduplicated by name via `uniqBy`.

---

## 7. Agent / Subagent System

### 7.1 Architecture

The `AgentTool` (`name: 'Agent'`, legacy `'Task'`) is the mechanism for spawning sub-agents. It creates an isolated execution context with its own conversation, token budget, tool set, and permission context.

**Files:**
- `src/tools/AgentTool/AgentTool.tsx` — main tool implementation
- `src/tools/AgentTool/loadAgentsDir.ts` — agent definition loading
- `src/tools/AgentTool/runAgent.ts` — sub-agent execution engine
- `src/tools/AgentTool/builtInAgents.ts` — built-in agent registry

### 7.2 Agent Definition Types

```typescript
// All agents share these base fields:
type BaseAgentDefinition = {
  agentType: string              // unique identifier
  whenToUse: string              // guidance for model selection
  tools?: string[]               // tool allowlist (* = all)
  disallowedTools?: string[]     // tool denylist
  skills?: string[]              // skill names to preload
  mcpServers?: AgentMcpServerSpec[]  // agent-specific MCP servers
  hooks?: HooksSettings          // session-scoped hooks
  color?: AgentColorName         // UI color
  model?: string                 // model override
  effort?: EffortValue           // effort level
  permissionMode?: PermissionMode
  maxTurns?: number              // max agentic turns
  background?: boolean           // always run in background
  initialPrompt?: string         // prepended to first user turn
  memory?: 'user' | 'project' | 'local'  // persistent memory
  isolation?: 'worktree' | 'remote'
  omitClaudeMd?: boolean         // skip CLAUDE.md hierarchy
}
```

### 7.3 Built-in Agents

| Agent Type | Purpose |
|------------|---------|
| `general-purpose` | Default agent for complex research/coding tasks |
| `statusline-setup` | One-time setup agent for custom status lines |
| `Explore` | Read-only codebase exploration (when feature enabled) |
| `Plan` | Planning mode agent (when feature enabled) |
| `code-guide` | Claude Code documentation guide (non-SDK only) |
| `verification` | Plan verification (when GrowthBook-gated) |

### 7.4 User-Defined Agents

Agents can be defined as markdown files in `.claude/agents/` directories:

```markdown
---
name: test-runner
description: "Specialized agent for running and fixing test failures"
tools: ["Bash", "Read", "Edit"]
model: claude-haiku-4-5
effort: low
permissionMode: acceptEdits
maxTurns: 20
mcpServers:
  - jest-runner
hooks:
  SessionStart:
    - hooks:
        - type: command
          command: "echo 'Test runner agent starting'"
---

You are a specialized test runner agent. Your sole purpose is to run
tests and fix failures. Always run the full test suite first...
```

Required frontmatter: `name` (string), `description` (string).

The markdown body becomes the agent's system prompt. Agents are loaded from:
- `~/.claude/agents/` — user-global agents
- `.claude/agents/` — project agents (walked up to home)
- Managed: enterprise policy agents
- Plugin: `{plugin-root}/agents/`

Priority (later overrides earlier): built-in → plugin → user → project → flag → managed.

### 7.5 Agent Isolation Modes

Agents can run in isolation:
- **Default** — shares the project working directory
- **Worktree isolation** (`isolation: 'worktree'`) — creates a temporary git worktree, gives agent a clean copy of the repo
- **Remote isolation** (`isolation: 'remote'`, Anthropic-internal only) — runs in CCR environment

### 7.6 Agent Communication (Teams)

When agent swarms are enabled (`isAgentSwarmsEnabled()`):

**TeamCreate** / **TeamDelete** — create/destroy named agent teams.

**SendMessage** — send a message to a named agent. Enables coordinator-worker patterns where a coordinator delegates tasks and communicates results.

**ListPeers** — list running agents in the current team context.

### 7.7 Coordinator Mode

**File:** `src/coordinator/coordinatorMode.ts`

When `CLAUDE_CODE_COORDINATOR_MODE=true`, the instance becomes a coordinator that spawns worker agents. Coordinators get a restricted tool set focused on delegation (AgentTool, TaskStop, SendMessage). Workers get the regular tool set filtered by `COORDINATOR_MODE_ALLOWED_TOOLS`.

### 7.8 Agent Invocation

```typescript
// AgentTool input schema:
{
  description: string          // short (3-5 word) task description
  prompt: string               // full task for the agent
  subagent_type?: string       // agent type name (matches agentType field)
  model?: 'sonnet' | 'opus' | 'haiku'
  run_in_background?: boolean  // detach to background
  // Multi-agent fields:
  name?: string                // addressable name for SendMessage
  team_name?: string
  mode?: PermissionMode
  isolation?: 'worktree' | 'remote'
  cwd?: string                 // working directory override
}
```

---

## 8. MCP Integration

### 8.1 MCP Server Configuration

**Files:** `src/services/mcp/types.ts`, `src/services/mcp/config.ts`

MCP servers are configured at multiple scopes:

```json
// .mcp.json (project-level, shared with team)
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["server.js"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

Transport types:
- `stdio` — subprocess with stdin/stdout JSON-RPC
- `sse` — Server-Sent Events over HTTP
- `http` — Streamable HTTP (MCP 2025-03 spec)
- `ws` — WebSocket
- `sdk` — in-process (SDK only)
- `sse-ide` / `ws-ide` — IDE extension connections

Configuration is merged from all scopes:
- Managed (`/etc/managed-mcp.json`) — scope `enterprise`
- User (`~/.claude/settings.json` `mcpServers` field) — scope `user`
- Project (`.mcp.json`) — scope `project`
- Dynamic (from MCP client startup) — scope `dynamic`
- Plugin-provided — scope varies
- Claude.ai proxy — scope `claudeai`

### 8.2 MCP Connection Lifecycle

**File:** `src/services/mcp/client.ts`

```
getAllMcpConfigs()
    │
    ▼
connectToServer(name, config)
    │
    ├─ create transport (StdioClientTransport, SSEClientTransport, etc.)
    ├─ create MCP Client
    ├─ client.connect()
    ├─ fetchToolsForClient() → ListToolsResult
    │     └─ for each tool:
    │           ├─ buildMcpToolName() → "mcp__server__tool"
    │           ├─ clone MCPTool
    │           └─ override: name, description, call(), checkPermissions(), etc.
    ├─ fetchPromptsForClient() → ListPromptsResult
    │     └─ convert to Command objects (skills)
    └─ listResources() → resources exposed via ListMcpResourcesTool / ReadMcpResourceTool
```

### 8.3 MCP Tool Wrapping

Each MCP tool is a clone of the `MCPTool` template with these overrides:
- `name` = `mcp__serverName__toolName` (or unprefixed in SDK mode)
- `isMcp = true`
- `mcpInfo = { serverName, toolName }`
- `description()` returns the MCP tool's description
- `call()` invokes `client.callTool()` via JSON-RPC and handles:
  - Binary content (persisted to disk with reference)
  - Large output truncation
  - OAuth 401 refresh
  - Elicitation requests
- `checkPermissions()` defers to the general permission system with `{ behavior: 'passthrough' }`
- `alwaysLoad` is set if `_meta['anthropic/alwaysLoad']` is true in the MCP tool definition

### 8.4 MCP Prompts as Skills

MCP servers can expose `prompts` (via `prompts/list`). These become skills with `loadedFrom: 'mcp'` and are accessible via `SkillTool`. The model can call `Skill({ skill: 'mcp__server__prompt-name' })`.

### 8.5 MCP Resources

MCP servers can expose `resources`. These are accessible via:
- `ListMcpResourcesTool` — list available resources per server
- `ReadMcpResourceTool` — read a specific resource URI

### 8.6 Agent-Scoped MCP Servers

Agent definitions can specify MCP servers that are only connected when that agent runs:

```yaml
mcpServers:
  - jest-runner           # reference to existing server by name
  - my-inline-server:     # inline server definition
      type: stdio
      command: node
      args: ["server.js"]
```

### 8.7 OAuth and Authentication

MCP servers can require OAuth authentication. The flow:
1. Server responds with 401
2. `NeedsAuthMCPServer` state set
3. User shown `/mcp auth` option
4. `McpAuthTool` handles the OAuth flow
5. Token stored in keychain, attached to future requests

### 8.8 Elicitation Protocol

MCP servers can request information from users via the elicitation protocol (MCP 2025-03). Claude Code:
1. Receives `ElicitRequest` from server mid-call
2. Runs `Elicitation` hooks (which can auto-respond)
3. Shows UI dialog if not auto-handled
4. Returns user's choice to the server

---

## 9. Configuration System

### 9.1 Settings Hierarchy

**File:** `src/utils/settings/constants.ts`

Settings are loaded from these sources in order (later overrides earlier for most fields):

| Source | `SettingSource` | Path | Notes |
|--------|----------------|------|-------|
| User | `userSettings` | `~/.claude/settings.json` | Global user preferences |
| Project | `projectSettings` | `.claude/settings.json` | Shared with team via git |
| Local | `localSettings` | `.claude/settings.local.json` | Gitignored, personal |
| Flag | `flagSettings` | `--settings` CLI flag | CLI override |
| Policy | `policySettings` | `/etc/claude-code/managed-settings.json` | Enterprise MDM |

Policy and flag settings are always included. User/project/local can be disabled via `--setting-sources user,project,local` CLI flag.

### 9.2 Key Settings Fields

```typescript
// Abbreviated SettingsSchema:
{
  // Auth
  apiKeyHelper?: string          // command to get API key
  awsCredentialExport?: string   // AWS credential helper
  gcpAuthRefresh?: string        // GCP auth refresh command

  // Behavior
  model?: string                 // default model override
  permissions?: PermissionsConfig
  env?: Record<string, string>   // env vars for all sessions
  cleanupPeriodDays?: number     // transcript retention

  // MCP
  mcpServers?: Record<string, McpServerConfig>
  enabledMcpjsonServers?: string[]
  allowedMcpServers?: AllowedMcpServerEntry[]
  deniedMcpServers?: DeniedMcpServerEntry[]
  allowManagedMcpServersOnly?: boolean

  // Hooks
  hooks?: HooksSettings
  disableAllHooks?: boolean
  allowManagedHooksOnly?: boolean
  allowedHttpHookUrls?: string[]

  // Plugins
  enabledPlugins?: Record<string, boolean>
  extraKnownMarketplaces?: Record<string, MarketplaceConfig>
  strictKnownMarketplaces?: MarketplaceSource[]
  blockedMarketplaces?: MarketplaceSource[]
  strictPluginOnlyCustomization?: boolean | string[]

  // Permissions
  permissions?: {
    allow?: PermissionRule[]
    deny?: PermissionRule[]
    ask?: PermissionRule[]
    defaultMode?: PermissionMode
    disableBypassPermissionsMode?: 'disable'
    additionalDirectories?: string[]
  }

  // UI/UX
  statusLine?: { type: 'command', command: string }
  fileSuggestion?: { type: 'command', command: string }
  outputStyle?: string
  language?: string
  defaultShell?: 'bash' | 'powershell'

  // Attribution
  attribution?: { commit?: string, pr?: string }

  // Enterprise
  availableModels?: string[]
  allowManagedPermissionRulesOnly?: boolean
  forceLoginMethod?: 'claudeai' | 'console'
}
```

### 9.3 CLAUDE.md Files

**File:** `src/utils/claudemd.ts`

CLAUDE.md files provide persistent context injected into the model's system prompt. They are discovered by traversing from the current directory up to the home directory.

Loading order (lowest to highest priority):
1. Managed memory (`/etc/claude-code/CLAUDE.md`)
2. User memory (`~/.claude/CLAUDE.md`)
3. Project memory (CLAUDE.md and `.claude/CLAUDE.md` and `.claude/rules/*.md` files, walking upward)
4. Local memory (`CLAUDE.local.md` files, walking upward)

Files closer to the current directory have higher priority. The `@include` directive (`@./path/to/file.md`) allows including other files.

CLAUDE.md files support frontmatter:
```markdown
---
paths:
  - "src/**"
  - "test/**"
---

Project-specific instructions that apply only to src/ and test/ files.
```

### 9.4 Permission Modes

| Mode | `PermissionMode` | Behavior |
|------|-----------------|---------|
| Default | `default` | Ask for all write/execute operations |
| Plan | `plan` | Model must propose a plan before acting (read-only first) |
| Accept Edits | `acceptEdits` | Auto-accept file edits, ask for shell commands |
| Bypass Permissions | `bypassPermissions` | Skip all permission checks (dangerous) |
| Don't Ask | `dontAsk` | Auto-accept everything |
| Auto | `auto` | AI classifier decides (Anthropic-internal) |

### 9.5 Permission Rules

Permission rules follow the syntax `ToolName(pattern)`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm test*)",
      "Write(src/**)",
      "Read"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Write(/etc/**)"
    ],
    "ask": [
      "Bash(curl *)"
    ]
  }
}
```

Rules are evaluated in priority order: `deny` > `allow` > `ask` > mode default.

---

## 10. Component Diagrams

### 10.1 Full Extension Point Map

```
CLAUDE CODE EXTENSIBILITY TREE
═══════════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│                        CONFIGURATION LAYER                          │
│                                                                     │
│  Policy (MDM)  ←  manages   →  settings.json (user/project/local)  │
│       │                               │                             │
│       └──── CLAUDE.md hierarchy ──────┘                             │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          PLUGIN LAYER                               │
│                                                                     │
│  Marketplace Plugins ──── Built-in Plugins ──── --plugin-dir        │
│  (git repos)              (ship in binary)       (session-only)     │
│       │                        │                                    │
│       └──── provides: commands/skills/agents/hooks/mcp/lsp          │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TOOL LAYER                                   │
│                                                                     │
│  Built-in Tools ──── MCP Tools ──── Plugin Tools                    │
│  (compiled in)    (from servers)    (via MCP)                       │
│       │                                                             │
│       └──── ToolSearch (deferred loading for large tool sets)       │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        HOOK LAYER                                   │
│                                                                     │
│  PreToolUse → PostToolUse → Stop → SessionStart → SessionEnd → ... │
│       │           │           │                                     │
│       │      [shell command / HTTP / LLM prompt / agent]            │
│       │                                                             │
│       └──── can: block / modify input / inject context              │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     AGENT LAYER                                     │
│                                                                     │
│  Built-in Agents ──── User/Project Agents ──── Plugin Agents        │
│  (general-purpose,    (.claude/agents/*.md)    ({plugin}/agents/)   │
│   explore, plan...)                                                 │
│       │                                                             │
│       └──── isolation: worktree / remote                            │
│       └──── communication: SendMessage / Teams                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.2 Tool Lifecycle Flow

```
Model → tool_use block
            │
            ▼
   backfillObservableInput(input)
            │
            ▼
   validateInput(input, context)  ────── FAIL → return error to model
            │ OK
            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                    PERMISSION GATE                           │
   │                                                              │
   │  1. check alwaysAllow rules (fast path)                      │
   │  2. check alwaysDeny rules (fast path)                       │
   │  3. check mode (bypassPermissions, dontAsk, acceptEdits...)  │
   │  4. tool.checkPermissions() — tool-specific logic            │
   │  5. run PreToolUse hooks → can approve/block/modify input    │
   │  6. if still 'ask': show UI dialog → user decides            │
   │                                                              │
   └──────────────────────────────────────────────────────────────┘
            │ allow
            ▼
   tool.call(args, context, canUseTool, parentMessage, onProgress)
            │
            ├─── onProgress callbacks ──── UI progress display
            │
            ▼
   run PostToolUse hooks → can inject additionalContext
            │
            ▼
   mapToolResultToToolResultBlockParam(result)
            │
            ▼
   inject result into conversation → model processes it
```

### 10.3 Plugin Loading Flow

```
startup
    │
    ▼
initBuiltinPlugins()           ← register built-in plugins
    │
    ▼
loadAllPlugins() [async]
    │
    ├─ getBuiltinPlugins()      ← from in-memory registry
    │
    ├─ getInlinePlugins()       ← from --plugin-dir / SDK option
    │
    └─ for each enabledPlugins entry:
            │
            ├─ parsePluginIdentifier("{name}@{marketplace}")
            ├─ resolve marketplace (github/git/url/npm/settings)
            ├─ check policy gates (strictKnownMarketplaces, blockedMarketplaces)
            ├─ clone/pull git repo to ~/.claude/plugins/{marketplace}/{name}/
            ├─ read and validate plugin.json
            └─ build LoadedPlugin {
                  commandsPath, agentsPath, skillsPath,
                  hooksConfig, mcpServers, lspServers
               }
    │
    ▼
for each enabled plugin:
    ├─ loadPluginCommands()     → Commands with source='plugin'
    ├─ loadPluginAgents()       → AgentDefinitions with source='plugin'
    ├─ loadPluginHooks()        → register in bootstrap state
    └─ getPluginMcpServers()    → add to MCP config
```

### 10.4 Hook Execution Flow in Tool Pipeline

```
tool call arrives
       │
       ▼
executePreToolUseHooks(tool, input, context)
       │
       ├─ get hooks from settings + plugins + skills + session
       ├─ filter by event = PreToolUse
       ├─ for each matching hook:
       │     ├─ evaluate `if` condition (tool name / pattern match)
       │     ├─ serialize input as JSON → pipe to command stdin
       │     ├─ exec shell command / HTTP POST / LLM prompt / agent
       │     ├─ parse stdout JSON → HookJSONOutput
       │     └─ aggregate results
       │
       ├─ if any hook: permissionBehavior='deny' → block tool use
       ├─ if any hook: updatedInput → pass to tool.call()
       ├─ if any hook: additionalContext → inject as system message
       └─ if any hook: continue=false → stop generation
       │
       ▼
tool.call(possibly-modified-input, context)
       │
       ▼
executePostToolUseHooks(tool, input, result, context)
       │
       ├─ (same structure as pre)
       ├─ updatedMCPToolOutput → overwrite MCP tool result
       └─ additionalContext → inject as follow-up context
```

### 10.5 Agent Spawning Flow

```
model calls Agent({ prompt, subagent_type, ... })
       │
       ▼
AgentTool.call()
       │
       ├─ resolve AgentDefinition from subagent_type
       │     └─ built-in → user/project → plugin → managed
       │
       ├─ check agent deny rules (getDenyRuleForAgent)
       │
       ├─ if isolation='worktree':
       │     └─ createAgentWorktree() → git worktree create
       │
       ├─ if run_in_background or background agent:
       │     └─ registerAsyncAgent() → returns immediately
       │
       ├─ assembleToolPool(permissionContext, mcpTools)
       │     └─ filter by agent.tools allowlist / disallowedTools denylist
       │
       ├─ connect agent-scoped MCP servers
       │
       ├─ createSubagentContext()
       │     └─ isolated AppState, message list, abort controller
       │
       ├─ registerFrontmatterHooks(agent.hooks)
       │     └─ session-scoped hooks for this agent's lifetime
       │
       ├─ executeSubagentStartHooks()
       │
       ├─ for await (message of runAgent(...)):
       │     ├─ query() → Anthropic API
       │     ├─ process tool calls (recursive tool execution)
       │     └─ emit progress events → parent sees AgentToolProgress
       │
       ├─ executeSubagentStopHooks()
       │
       └─ if isolation='worktree':
             └─ removeAgentWorktree() → git worktree remove
```

---

## 11. Extension Points Summary

### Which Extension Point to Use

| Goal | Recommended Approach |
|------|---------------------|
| Add a new tool capability to the model | MCP server (external) or built-in tool (internal) |
| Automate a workflow the model invokes | Skill (`.claude/skills/`) or bundled skill |
| Create a user-facing slash command | Command (`.claude/commands/`) |
| Define a specialized sub-agent | Agent definition (`.claude/agents/`) |
| Run code before/after tool calls | Hook (settings.json `hooks.PreToolUse/PostToolUse`) |
| Intercept and modify tool inputs | `PreToolUse` hook with `updatedInput` response |
| Block dangerous operations | `PreToolUse` hook with `decision: 'block'` |
| Inject context into conversations | `UserPromptSubmit` hook with `additionalContext` |
| Run setup code at session start | `SessionStart` hook |
| Bundle multiple features together | Plugin (git repo with `plugin.json`) |
| Distribute features to a team | Marketplace plugin + `extraKnownMarketplaces` |
| Lock down enterprise environments | Policy settings (`managed-settings.json`) |
| Add project-wide AI instructions | `CLAUDE.md` in project root |
| Add personal AI instructions | `~/.claude/CLAUDE.md` |

### Extension Points Summary Table

| Extension Point | Location | Configuration | How Loaded |
|-----------------|----------|---------------|-----------|
| Built-in tools | `src/tools/*/` | Compiled in | `getAllBaseTools()` |
| MCP tools | External process | `.mcp.json`, `settings.json mcpServers` | `client.ts connectToServer()` |
| User skills | `~/.claude/skills/*.md` | Frontmatter | `getSkillDirCommands()` |
| Project skills | `.claude/skills/*.md` | Frontmatter | `getSkillDirCommands()` |
| Bundled skills | `src/skills/bundled/` | `registerBundledSkill()` | `initBundledSkills()` |
| Plugin skills | `{plugin}/skills/` | Frontmatter | `loadPluginCommands()` |
| MCP skills | MCP server prompts | MCP protocol | `fetchPromptsForClient()` |
| User commands | `~/.claude/commands/*.md` | Frontmatter | `getSkillDirCommands()` |
| Project commands | `.claude/commands/*.md` | Frontmatter | `getSkillDirCommands()` |
| Plugin commands | `{plugin}/commands/` | Frontmatter | `loadPluginCommands()` |
| User agents | `~/.claude/agents/*.md` | Frontmatter | `getAgentDefinitionsWithOverrides()` |
| Project agents | `.claude/agents/*.md` | Frontmatter | `getAgentDefinitionsWithOverrides()` |
| Plugin agents | `{plugin}/agents/` | Frontmatter | `loadPluginAgents()` |
| Built-in agents | `src/tools/AgentTool/built-in/` | Code | `getBuiltInAgents()` |
| User hooks | `settings.json hooks` | JSON schema | `getSessionHooks()` |
| Project hooks | `.claude/settings.json hooks` | JSON schema | `getSessionHooks()` |
| Plugin hooks | `{plugin}/hooks/hooks.json` | JSON schema | `loadPluginHooks()` |
| Skill hooks | Skill frontmatter | YAML | `registerSkillHooks()` |
| CLAUDE.md (user) | `~/.claude/CLAUDE.md` | Markdown | `getMemoryFiles()` |
| CLAUDE.md (project) | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/` | Markdown | `getMemoryFiles()` |
| CLAUDE.md (managed) | `/etc/claude-code/CLAUDE.md` | Markdown | `getMemoryFiles()` |
| Built-in plugins | `src/plugins/bundled/` | `registerBuiltinPlugin()` | `initBuiltinPlugins()` |
| Marketplace plugins | `~/.claude/plugins/{marketplace}/{name}/` | `enabledPlugins` in settings | `loadAllPlugins()` |
| Inline plugins | `--plugin-dir` / SDK option | CLI / SDK | `getInlinePlugins()` |
| Permission rules | `settings.json permissions` | JSON | `getSettingsForSource()` |
| MCP server config | `.mcp.json`, `settings.json mcpServers` | JSON | `getAllMcpConfigs()` |

---

This document provides a complete map of every extension point in Claude Code. The system is deeply layered: every customization ultimately flows through the tool permission pipeline, and every layer (plugins, skills, agents, hooks) is composable with every other.
