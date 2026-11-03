# Agent Orchestration & Task Management — Feature Analysis

## Summary

Claude Code implements a multi-layered agent orchestration system that spans from simple synchronous subagents to complex multi-agent swarms with visual terminal panes. The system has three execution models (sync subagent, async background agent, in-process teammate), two visual backends (tmux, iTerm2), a comprehensive task registry, file-based inter-agent messaging, and JSONL-based session persistence with sidechain transcripts for subagents.

The architecture has evolved through several generations: the original general-purpose agent, the coordinator mode for directed parallelism, agent swarms for visual multi-agent collaboration, and the fork-subagent experiment for context-inheriting parallel workers.

---

## 1. Agent Orchestration

### 1.1 Entry Point: AgentTool

**File:** `src/tools/AgentTool/AgentTool.tsx`

The `AgentTool` is the single entry point for all agent spawning. It is a regular tool (registered via `buildTool()`) invoked by the model with a JSON input. The tool's `call()` function dispatches to one of several execution paths:

```
AgentTool.call(input)
  ├── teamName && name  →  spawnTeammate()        [swarm path]
  ├── isolation=remote  →  teleportToRemote()      [ant-only CCR path]
  ├── shouldRunAsync    →  runAsyncAgentLifecycle() [background agent]
  └── sync              →  runAgent() (blocking)   [synchronous subagent]
```

**Input schema** (`src/tools/AgentTool/AgentTool.tsx:82`):
- `description`: 3-5 word task label
- `prompt`: the actual task
- `subagent_type`: optional agent type (omitting triggers fork path when gate active)
- `model`: optional override (`sonnet`, `opus`, `haiku`)
- `run_in_background`: explicit async flag
- `name` + `team_name`: triggers teammate/swarm spawning
- `isolation`: `worktree` (creates git worktree) or `remote` (ant-only CCR)
- `cwd`: working directory override (KAIROS feature gate)

**Key decision tree** (lines 322–567):

1. **Fork path** — `subagent_type` omitted + `FORK_SUBAGENT` gate active: child inherits parent's entire conversation context for cache-identical prefixes
2. **Swarm path** — `teamName && name` set: delegates to `spawnTeammate()` for tmux/iTerm2/in-process spawn
3. **Remote path** — `isolation=remote` (ant-only): teleports to a CCR remote environment
4. **Worktree path** — `isolation=worktree`: creates a git worktree, runs agent inside it
5. **Async path** — background flag set, coordinator mode, or forceAsync: fire-and-forget with task registry
6. **Sync path** — default: blocks the parent turn until the subagent completes

### 1.2 Agent Definitions

**File:** `src/tools/AgentTool/loadAgentsDir.ts`

Agents are defined in three source categories, merged via priority order:

```
Priority (lowest → highest overrides):
  built-in  →  plugin  →  userSettings  →  projectSettings  →  flagSettings  →  policySettings
```

**Definition sources:**
- **Built-in** (`BuiltInAgentDefinition`): Hardcoded, source=`built-in`, dynamic `getSystemPrompt({toolUseContext})`
- **Custom** (`CustomAgentDefinition`): Markdown files from `.claude/agents/` directories, source=`userSettings`/`projectSettings`/`policySettings`
- **Plugin** (`PluginAgentDefinition`): From installed plugins, source=`plugin`

**Markdown agent frontmatter fields** (parsed in `parseAgentFromMarkdown()`):
```yaml
name: my-agent
description: When to use this agent
model: sonnet | opus | haiku | inherit
tools: [Bash, Read, ...]
disallowedTools: [FileWrite, ...]
permissionMode: default | acceptEdits | plan | bypassPermissions | bubble
maxTurns: 50
effort: low | normal | high | <integer>
memory: user | project | local
background: true
isolation: worktree
mcpServers: [server-name, {inline-server: config}]
skills: [skill-name, ...]
hooks: ...
initialPrompt: prepended to first turn
```

The `getAgentDefinitionsWithOverrides()` function (line 296) is memoized by `cwd` and loads all agents at startup. The priority merge in `getActiveAgentsFromList()` uses a `Map<agentType, AgentDefinition>` iterated in priority order so later groups overwrite earlier ones.

**Built-in agents** (`src/tools/AgentTool/builtInAgents.ts`):
- `GENERAL_PURPOSE_AGENT` — always available
- `STATUSLINE_SETUP_AGENT` — always available
- `EXPLORE_AGENT`, `PLAN_AGENT` — gated by `BUILTIN_EXPLORE_PLAN_AGENTS` feature + `tengu_amber_stoat` GrowthBook
- `CLAUDE_CODE_GUIDE_AGENT` — non-SDK entrypoints only
- `VERIFICATION_AGENT` — gated by `VERIFICATION_AGENT` feature + `tengu_hive_evidence` GrowthBook
- **Coordinator mode**: replaces all with `getCoordinatorAgents()` from `workerAgent.js`

### 1.3 runAgent() — The Core Agent Loop

**File:** `src/tools/AgentTool/runAgent.ts`

`runAgent()` is an async generator that:

1. **Resolves model** via `getAgentModel(agentDef.model, parentModel, override, permissionMode)`
2. **Sets up transcript subdirectory** for workflow grouping
3. **Builds context messages**: fork children prepend parent's full conversation; non-fork starts with just the user prompt
4. **Clones or creates file state cache**: fork children clone parent's `readFileState` for cache efficiency
5. **Resolves user/system contexts**: Explore/Plan agents drop `claudeMd` and `gitStatus` to save tokens
6. **Overrides permission mode** from agent definition (unless parent is `bypassPermissions`/`acceptEdits`/`auto`)
7. **Determines abort controller**: async agents get an independent controller; sync agents share parent's
8. **Executes SubagentStart hooks** to collect additional context
9. **Registers frontmatter hooks** scoped to agent lifecycle
10. **Preloads skills** from `agentDefinition.skills`
11. **Initializes agent-specific MCP servers** (additive to parent's servers)
12. **Creates `agentToolUseContext`** via `createSubagentContext()`
13. **Records initial messages** to sidechain transcript
14. **Calls `query()`** in a for-await loop, yielding messages to caller
15. **Records each new message** to the sidechain transcript incrementally

**Key design:** The `useExactTools` flag (used by fork children) bypasses `resolveAgentTools()` and uses the parent's exact tool array verbatim — this is critical for prompt cache sharing since tool definitions are part of the API cache key.

**Context isolation** (line 700):
```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  // Sync agents share setAppState with parent
  // Async agents are fully isolated
  shareSetAppState: !isAsync,
  shareSetResponseLength: true,
})
```

### 1.4 Fork Subagent System

**File:** `src/tools/AgentTool/forkSubagent.ts`

The fork mechanism enables a parent agent to split its work across multiple parallel children that share the parent's prompt cache prefix. Key design goals:

- **Cache sharing**: all fork children send byte-identical API request prefixes so they all hit the same KV cache entry
- **Recursive guard**: fork children detect they are forks via `FORK_BOILERPLATE_TAG` in their conversation history and reject further fork attempts
- **Parent context inheritance**: the full parent conversation is passed as `forkContextMessages`

**`buildForkedMessages(directive, assistantMessage)`** (line 107):
```
[parent history...] 
+ assistant(all_tool_use blocks from current turn)
+ user(placeholder_tool_results × N + per-child directive text)
```

The placeholder text `'Fork started — processing in background'` is identical for every child — only the final `FORK_DIRECTIVE_PREFIX` + directive text differs. This means the prefix up to the last text block is cache-identical across all children.

**`FORK_AGENT` definition** (line 60):
```typescript
{
  agentType: 'fork',
  tools: ['*'],          // inherits parent's exact tool pool
  maxTurns: 200,
  model: 'inherit',     // parent's model for context length parity
  permissionMode: 'bubble', // prompts surface to parent terminal
}
```

The `buildChildMessage()` function embeds non-negotiable rules in the child's first user message: do not spawn sub-agents, do not converse, use tools silently, commit changes, report in structured format.

### 1.5 Coordinator Mode

**File:** `src/coordinator/coordinatorMode.ts`

Coordinator mode is activated by `CLAUDE_CODE_COORDINATOR_MODE=1` environment variable. In this mode:

1. **Built-in agents are replaced** by `getCoordinatorAgents()` which returns the `worker` agent type
2. **All agents are forced async** — the coordinator only launches background workers
3. **Workers get a restricted tool set** (`ASYNC_AGENT_ALLOWED_TOOLS`) — no REPL, no UI tools, no swarm tools
4. **The coordinator's system prompt** (`getCoordinatorSystemPrompt()`) provides detailed orchestration guidance including:
   - Worker result format (`<task-notification>` XML parsing)
   - Parallelism strategy (parallel for read, serial for write)
   - Continue vs. spawn decision matrix
   - Worker prompt synthesis requirements
5. **The coordinator's user context** (`getCoordinatorUserContext()`) injects the worker tool list and optional scratchpad directory

The coordinator model never calls tools directly for implementation — it delegates all work to async workers and synthesizes their results.

### 1.6 Agent Swarms / Teammates

**Files:** `src/utils/swarm/`, `src/tools/AgentTool/AgentTool.tsx` (spawnTeammate path)

Agent swarms are the visual multi-agent system where multiple Claude instances run visually in parallel terminal panes. The swarm is gated by `isAgentSwarmsEnabled()`.

**Architecture:**

```
AgentTool.call() [teamName && name set]
  └── spawnTeammate()  [src/tools/shared/spawnMultiAgent.ts]
       └── getTeammateExecutor(preferInProcess)
            ├── isInProcessEnabled() → InProcessBackend
            └── else → PaneBackendExecutor(TmuxBackend | ITermBackend)
```

**Backend selection** (`src/utils/swarm/backends/registry.ts:136`):
1. Inside tmux → TmuxBackend
2. In iTerm2 + `it2` CLI available → ITermBackend
3. In iTerm2 + `it2` not available, tmux available → TmuxBackend (with `needsIt2Setup` flag)
4. Non-interactive session → InProcessBackend
5. Settings override `teammate_mode=in-process` → InProcessBackend
6. No tmux/iTerm2 → InProcessBackend (or error)

### 1.7 Tmux Backend

**File:** `src/utils/swarm/backends/TmuxBackend.ts`

The `TmuxBackend` manages pane lifecycle via `execFileNoThrow(tmux, args)` calls. Pane creation uses a lock (`acquirePaneCreationLock()`) to prevent race conditions when spawning teammates in parallel.

**Layout strategy:**
- Inside tmux: leader pane on left at 30%, teammates fill right 70% using main-vertical layout
- Outside tmux: creates `claude-swarm` session on separate socket (`-L claude-swarm`), teammates use tiled layout

**Pane creation flow** (inside tmux):
```
First teammate: split-window -h -l 70% from leader pane
2nd+ teammate: split-window -v or -h from an existing teammate pane
After each: setPaneBorderColor → setPaneTitle → rebalancePanesWithLeader
```

Each teammate pane runs a fresh `claude` CLI process. The team-lead sends commands to panes via `tmux send-keys -t paneId command Enter` after a 200ms shell init delay.

### 1.8 In-Process Teammates

**Files:** `src/utils/swarm/spawnInProcess.ts`, `src/utils/swarm/inProcessRunner.ts`, `src/utils/teammateContext.ts`

In-process teammates run in the same Node.js process as the leader, using `AsyncLocalStorage` for identity isolation.

**AsyncLocalStorage isolation** (`src/utils/teammateContext.ts:41`):
```typescript
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

export function runWithTeammateContext<T>(context: TeammateContext, fn: () => T): T {
  return teammateContextStorage.run(context, fn)
}

export function getTeammateContext(): TeammateContext | undefined {
  return teammateContextStorage.getStore()
}
```

The `TeammateContext` stores: `agentId`, `agentName`, `teamName`, `color`, `planModeRequired`, `parentSessionId`, `isInProcess: true`, `abortController`.

**Identity resolution priority** in `teammate.ts`:
1. AsyncLocalStorage (in-process teammates)
2. `dynamicTeamContext` (process-based teammates joined at runtime)
3. Environment variables (`CLAUDE_CODE_AGENT_ID`, `CLAUDE_CODE_TEAM_NAME`)

**Spawn flow** (`spawnInProcessTeammate()`):
1. Generate `agentId = formatAgentId(name, teamName)` → `"researcher@my-team"`
2. Generate `taskId = generateTaskId('in_process_teammate')` → `"t{8-char-random}"`
3. Create independent `AbortController` (not linked to leader)
4. Create `TeammateContext` for AsyncLocalStorage
5. Register `InProcessTeammateTaskState` in AppState via `registerTask()`
6. Return spawn result

The actual agent execution is started by `InProcessBackend.spawn()` calling `startInProcessTeammate()`, which calls `runWithTeammateContext(context, () => runAgent(...))`.

**Permission bridging** — in-process teammates can surface permission prompts to the leader's terminal via `LeaderPermissionBridge` (`src/utils/swarm/leaderPermissionBridge.ts`). When the bridge is registered (REPL mode), `createInProcessCanUseTool()` resolves `ask` decisions through the leader's `ToolUseConfirm` dialog with a worker badge. When the bridge is unavailable, permission requests go through the file-based mailbox system.

---

## 2. Task Management

### 2.1 Task Type Hierarchy

**File:** `src/Task.ts`, `src/tasks/types.ts`

Seven task types exist, each with a type-prefix encoded into their ID:

| Type | Prefix | Description |
|------|--------|-------------|
| `local_bash` | `b` | Shell command execution (`BashTool`, hooks) |
| `local_agent` | `a` | Background subagent (async `AgentTool`) |
| `remote_agent` | `r` | CCR remote agent (ant-only) |
| `in_process_teammate` | `t` | In-process swarm teammate |
| `local_workflow` | `w` | Workflow execution |
| `monitor_mcp` | `m` | MCP server monitor task |
| `dream` | `d` | Auto-memory consolidation (auto-dream) |

Task IDs are generated as `{prefix}{8-char-base36-random}` — e.g., `a3x7mq9pf`. The 36^8 ≈ 2.8 trillion combination space is sufficient to resist symlink attacks (the task output directory is in a temp path that untrusted code could try to exploit).

**Base task state** (`TaskStateBase`):
```typescript
{
  id: string          // task ID with type prefix
  type: TaskType
  status: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  description: string
  toolUseId?: string  // links back to the tool use that spawned this task
  startTime: number
  endTime?: number
  outputFile: string  // path to disk output file
  outputOffset: number // byte offset for delta reads
  notified: boolean   // prevents duplicate notifications
}
```

### 2.2 LocalAgentTask

**File:** `src/tasks/LocalAgentTask/LocalAgentTask.tsx`

`LocalAgentTaskState` extends `TaskStateBase` with:
- `agentId`, `agentType`, `selectedAgent` — identity
- `abortController` — for cancellation
- `progress: AgentProgress` — tool use count, token count, recent activities
- `result: AgentToolResult` — final output
- `messages?: Message[]` — in-memory transcript for UI display
- `retain: boolean` — UI is viewing this task (blocks eviction)
- `diskLoaded: boolean` — sidechain JSONL has been loaded into `messages`
- `evictAfter?: number` — panel eviction deadline
- `pendingMessages: string[]` — messages queued mid-turn via `SendMessage`
- `isBackgrounded: boolean` — tracks whether user has backgrounded this
- `lastReportedToolCount`, `lastReportedTokenCount` — delta tracking

**Lifecycle management functions:**
- `registerAsyncAgent()` — creates task state, calls `registerTask()`, fires `task_started` SDK event
- `killAsyncAgent()` — aborts controller, sets `status='killed'`, schedules `evictTaskOutput()`
- `completeAgentTask()` — sets `status='completed'`, calls `enqueueAgentNotification()`
- `failAgentTask()` — sets `status='failed'`, calls `enqueueAgentNotification()`
- `updateAgentProgress()` — updates progress metrics, preserves existing summary
- `enqueueAgentNotification()` — formats XML `<task-notification>` message and calls `enqueuePendingNotification()`

**Progress tracking** via `ProgressTracker`:
```typescript
{
  toolUseCount: number
  latestInputTokens: number   // cumulative (API accumulates), keep latest
  cumulativeOutputTokens: number  // per-turn, sum these
  recentActivities: ToolActivity[]  // last 5 activities
}
```

`updateProgressFromMessage()` scans assistant messages for `tool_use` blocks, updating the tracker with tool names, inputs, and pre-computed activity descriptions.

**Task notification format** (XML injected as a user-role message):
```xml
<task-notification>
<task-id>{taskId}</task-id>
<tool-use-id>{toolUseId}</tool-use-id>
<output-file>{path}</output-file>
<status>completed|failed|killed</status>
<summary>Agent "{description}" completed</summary>
<result>{finalMessage}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
<worktree>
  <worktree-path>{path}</worktree-path>
  <worktree-branch>{branch}</worktree-branch>
</worktree>
</task-notification>
```

### 2.3 Task Output Persistence

**File:** `src/utils/task/diskOutput.ts`

Task output is written to `{projectTempDir}/{sessionId}/tasks/{taskId}.output`.

The `DiskTaskOutput` class implements an efficient write queue:
- Appends to a flat array queue (not chained `.then()` to avoid memory retention)
- A single drain loop processes all buffered chunks via `appendFile()`
- `O_NOFOLLOW` flag prevents symlink attacks from sandboxed processes
- `O_EXCL` on creation prevents clobbering existing files
- 5GB cap on task output files (shared between shell watchdog and pipe truncation)

**Delta reading** (`getTaskOutputDelta(taskId, fromOffset, maxBytes)`):
- Reads only new bytes since `fromOffset`
- Returns content + new offset
- Used by `generateTaskAttachments()` for polling-based progress delivery

**Symlink optimization**: For local agent tasks, `initTaskOutputAsSymlink(taskId, transcriptPath)` creates the output file as a symlink to the agent's sidechain JSONL transcript. This means reading the output file directly reads the agent's conversation transcript.

### 2.4 Task Polling Framework

**File:** `src/utils/task/framework.ts`

The polling framework runs at `POLL_INTERVAL_MS = 1000ms`. `pollTasks()` calls `generateTaskAttachments()` which:
1. Reads delta output from disk for running tasks
2. Collects tasks ready for eviction (terminal + notified)
3. Returns attachments for delivery

`applyTaskOffsetsAndEvictions()` applies the results against fresh AppState (not the pre-await snapshot) to avoid TOCTOU races.

**Eviction policy** (`evictTerminalTask()`):
- Task must be in terminal state (`completed`/`failed`/`killed`)
- `notified` must be true
- For `local_agent` tasks: `evictAfter` deadline must have passed (30s grace for coordinator panel)
- `retain: true` blocks eviction while user is viewing the task

### 2.5 Cron Scheduling System

**File:** `src/tools/ScheduleCronTool/CronCreateTool.ts`, `src/utils/cronTasks.ts`

The cron system (`CronCreateTool`, `CronDeleteTool`, `CronListTool`) is gated by `isKairosCronEnabled()`.

**Cron task schema:**
```typescript
{
  cron: string      // "*/5 * * * *" (5-field, local time)
  prompt: string   // prompt to enqueue at fire time
  recurring: boolean  // true=repeat, false=one-shot (auto-delete after fire)
  durable: boolean    // true=persist to .claude/scheduled_tasks.json
  agentId?: string    // for teammate-owned crons (owner identity)
}
```

**Lifecycle:**
1. `addCronTask()` adds to in-memory store (and optionally `.claude/scheduled_tasks.json`)
2. `setScheduledTasksEnabled(true)` activates the scheduler (the `useScheduledTasks` hook watches this flag)
3. The scheduler polls fired crons and enqueues their prompts via `enqueuePendingNotification()`

**Constraints:**
- Max 50 jobs per session
- Durable crons are forbidden for teammates (teammates don't persist across sessions)
- Session-only crons die when the Claude process exits

### 2.6 TaskCreateTool / Todo System

**File:** `src/tools/TaskCreateTool/TaskCreateTool.ts`, `src/utils/tasks.ts`

The `TaskCreateTool` is gated by `isTodoV2Enabled()` and manages a structured task list separate from the background task registry. These tasks are user-facing to-do items, not system-level execution tasks.

`createTask()` creates entries in a `tasks.json` file keyed by `taskListId`. The task object has:
```typescript
{
  id: string
  subject: string
  description: string
  activeForm?: string    // spinner verb (e.g., "Running tests")
  status: 'pending' | 'in_progress' | 'done' | 'cancelled'
  owner?: string         // agent name
  blocks: string[]       // task IDs blocked by this
  blockedBy: string[]    // task IDs blocking this
  metadata?: Record<string, unknown>
}
```

Hooks are fired on task creation via `executeTaskCreatedHooks()`. If any hook returns a blocking error, the task is deleted and the error is thrown.

### 2.7 Dream Task

**File:** `src/tasks/DreamTask/DreamTask.ts`

The `DreamTask` is a special task type for the auto-memory consolidation agent (auto-dream). It surfaces an otherwise-invisible forked agent in the UI.

**State fields:**
- `phase: 'starting' | 'updating'` — transitions when first Edit/Write tool_use is detected
- `sessionsReviewing: number` — count of sessions being consolidated
- `filesTouched: string[]` — incomplete list (misses bash-mediated writes)
- `turns: DreamTurn[]` — last 30 assistant turns (text + tool use count)

The dream task has no model-facing notification path (`notified: true` is set immediately on completion) — it's purely a UI visualization. The agent itself is a regular forked agent running `runAgent()`.

---

## 3. Session Management

### 3.1 JSONL Transcript Persistence

**File:** `src/utils/sessionStorage.ts`

Sessions are persisted as newline-delimited JSON (JSONL) files at:
```
~/.claude/projects/{sanitized-cwd}/{sessionId}.jsonl
```

Each line is a JSON-encoded `Entry` object. Entry types include: `user`, `assistant`, `attachment`, `system`, `content_replacement`, `context_collapse_snapshot`, `context_collapse_commit`, `file_history_snapshot`, `attribution_snapshot`.

**The `Project` class** (line 532) manages writes through a batched queue:
- 100ms flush timer
- 100MB chunk limit before forced flush
- Per-file write queues (map of filePath → queue)
- `pendingEntries` buffer before `sessionFile` is materialized (prevents metadata-only files)

**Transcript chain:** Each message has a `uuid` and `parentUuid` forming a linked list. `isChainParticipant()` returns false for `progress` type messages — they do not participate in the chain.

**Session metadata** appended as special entries:
- Custom title (`/rename` command)
- Tag (`/tag` command)
- Agent name and color
- Coordinator/normal mode
- Worktree state

`reAppendSessionMetadata()` re-appends cached metadata to the end of the file during compaction and on shutdown, ensuring metadata stays within the 64KB tail window that `readLiteMetadata()` reads.

### 3.2 Sidechain Transcripts for Subagents

**File:** `src/utils/sessionStorage.ts` (`recordSidechainTranscript()`, `getAgentTranscriptPath()`)

Each subagent writes its conversation to a sidechain transcript at:
```
~/.claude/projects/{sanitized-cwd}/{sessionId}/subagents/agent-{agentId}.jsonl
```

For grouped subagents (e.g., workflow runs), a subdirectory is used:
```
~/.claude/projects/{sanitized-cwd}/{sessionId}/subagents/{subdir}/agent-{agentId}.jsonl
```

The subdir is set via `setAgentTranscriptSubdir(agentId, subdir)` before the agent runs.

**Recording flow** in `runAgent()`:
1. Initial messages recorded via `recordSidechainTranscript(initialMessages, agentId)` — creates the file
2. Each new message from `query()` is recorded via `recordSidechainTranscript([message], agentId, lastRecordedUuid)` — appends with parent chain

This incremental approach is O(1) per message vs. rewriting the entire file.

**Agent metadata sidecar** (`.meta.json` file alongside JSONL):
```typescript
{
  agentType: string
  worktreePath?: string  // for worktree isolation
  description?: string  // original task description (for resume notifications)
}
```

This sidecar enables `resumeAgentBackground()` to correctly route to the original agent type when `subagent_type` was omitted at spawn time (the fork path).

### 3.3 Session Resume

**File:** `src/tools/AgentTool/resumeAgent.ts`

`resumeAgentBackground()` restores a completed/failed subagent from its sidechain transcript:

1. Reads transcript via `getAgentTranscript(agentId)` + metadata via `readAgentMetadata(agentId)`
2. Filters messages: removes whitespace-only assistant messages, orphaned thinking-only messages, and unresolved tool uses
3. Reconstructs `ContentReplacementState` from the persisted replacement records
4. Validates the worktree path still exists (bumps mtime to prevent stale-worktree cleanup)
5. Determines `selectedAgent`: fork → `FORK_AGENT`, typed → lookup in activeAgents, unknown → `GENERAL_PURPOSE_AGENT`
6. For resumed forks: reconstructs parent system prompt (from `toolUseContext.renderedSystemPrompt` or recomputes)
7. Registers new background task via `registerAsyncAgent()`
8. Starts `runAsyncAgentLifecycle()` with `runAgent()` as the stream factory

The prompt passed to `resumeAgentBackground()` is appended as a new user message after the resumed conversation history.

### 3.4 Teleport Feature

**File:** `src/utils/teleport.tsx`

Teleport enables resuming a session from a different machine. The feature:

1. **Validates** git state (clean working tree required for worktree handoff)
2. **Fetches session logs** via OAuth-authenticated API (`getSessionLogsViaOAuth()`)
3. **Generates title and branch** via `queryHaiku()` (Claude Haiku) from the session description
4. **Creates and uploads a git bundle** for the repository state
5. **Launches a remote session** at `teleportToRemote()` which calls `createAndUploadGitBundle()` and provisions a CCR session
6. **On remote**: deserializes messages from the remote session record, checks out the bundle, injects a `createTeleportResumeUserMessage()` to inform the model it's on a new machine

The `TeleportProgressStep` type maps the UX flow: `validating → fetching_logs → fetching_branch → checking_out → done`.

A teleport result carries `{ messages: Message[], branchName: string }` which is merged into the active session.

### 3.5 Agentic Session Search

**File:** `src/utils/agenticSessionSearch.ts`

`agenticSessionSearch(query, logs)` uses Claude (small/fast model) to semantically rank sessions for the session picker (`/resume` flow):

1. **Pre-filters** up to 100 sessions using substring matching across title, tag, branch, summary, firstPrompt, and transcript
2. **Loads full transcripts** for lite logs (lazy-loaded logs that only have metadata)
3. **Builds a formatted session list** with all searchable metadata
4. **Calls `sideQuery()`** with a ranking system prompt
5. **Parses `{"relevant_indices": [...]}` response** and maps indices back to log objects

The system prompt instructs the model to be inclusive (prefer false positives over misses) and to prioritize exact tag matches, then partial tags, then title/branch, then semantic content.

### 3.6 Concurrent Session Coordination

Sessions coordinate via `updateSessionName()` in `concurrentSessions.ts`, which writes session metadata to a shared registry so multiple Claude processes in the same project can see each other's sessions.

---

## 4. Inter-Agent Communication

### 4.1 File-Based Mailbox

**File:** `src/utils/teammateMailbox.ts`

All inter-agent messages use a file-based mailbox system located at:
```
{teamsDir}/{teamName}/inboxes/{agentName}.json
```

Each inbox is a JSON array of `TeammateMessage` objects:
```typescript
{
  from: string
  text: string
  timestamp: string
  read: boolean
  color?: string
  summary?: string
}
```

Writes use `lockfile` for concurrency safety across multiple Claude processes.

**Message types transported:**
- Plain text messages (from leader or teammates)
- Structured `shutdown_request` / `shutdown_response` / `plan_approval_response` (JSON-encoded)
- Permission request/response (for the mailbox fallback path)

### 4.2 SendMessageTool

**File:** `src/tools/SendMessageTool/SendMessageTool.ts`

`SendMessageTool` is gated by `isAgentSwarmsEnabled()`. It routes messages via several paths:

1. **Bridge target** (`bridge:{sessionId}`) — cross-session via Anthropic's servers (requires consent)
2. **UDS target** (`uds:{socketPath}`) — local Unix domain socket cross-process
3. **Registered agent name** → looks up `agentNameRegistry` in AppState
4. **Background agent** → calls `queuePendingMessage()` (queued until next tool boundary)
5. **Stopped agent** → calls `resumeAgentBackground()` to auto-resume with the message
6. **Broadcast** (`to: "*"`) → reads team file, writes to all member inboxes
7. **Teammate name** → writes to mailbox

**Structured messages:**
- `shutdown_request` → sends `createShutdownRequestMessage()` to target's inbox
- `shutdown_response` approve → for in-process: aborts controller; for pane-based: `gracefulShutdown()`
- `shutdown_response` reject → sends rejection message to leader's inbox
- `plan_approval_response` approve/reject → writes approval/rejection to teammate's inbox (leader only)

### 4.3 Permission Sync

**File:** `src/utils/swarm/permissionSync.ts`

When an in-process teammate needs permission and the leader bridge is unavailable:
1. Teammate sends a structured permission request to the leader's mailbox
2. Leader's permission poller (`useSwarmPermissionPoller.ts`) reads the request
3. Leader shows the ToolUseConfirm dialog
4. Leader writes the decision to the teammate's mailbox
5. Teammate's `processMailboxPermissionResponse()` unblocks and returns the decision

---

## 5. Architecture Diagrams

### 5.1 Agent Spawning Decision Tree

```
AgentTool.call(input)
│
├─ [teamName && name]────────────────────────────────────→ spawnTeammate()
│                                                             │
│                                               ┌──────── isInProcessEnabled()
│                                               │                │
│                                        InProcessBackend   PaneBackendExecutor
│                                          (AsyncLocalStorage)  │
│                                                          ┌────┴────┐
│                                                        tmux     iterm2
│
├─ [isolation=remote, ant-only]──────────────────────────→ teleportToRemote()
│
├─ [isolation=worktree]──────────────────────────────────→ createAgentWorktree()
│                                                             (then normal flow)
│
├─ [shouldRunAsync = true]
│   [fork|coordinator|background|KAIROS]
│   │
│   registerAsyncAgent()
│   └── void runAsyncAgentLifecycle()
│         └── runAgent() [generator]
│               └── query() [API calls]
│
└─ [sync]
    return runAgent() [blocking generator, yields messages]
```

### 5.2 Task Lifecycle

```
register → running → completed ─┐
           │      → failed     ─┤→ notified=true → evict after deadline
           │      → killed     ─┘
           │
           └── (intermediate progress updates via updateAgentProgress)
                (disk output via DiskTaskOutput.append)
                (delta reads via getTaskOutputDelta at 1s intervals)
```

### 5.3 Sidechain Transcript Flow

```
runAgent()
  │
  ├─ recordSidechainTranscript(initialMessages, agentId)
  │    → ~/.claude/projects/{cwd}/{session}/subagents/agent-{id}.jsonl
  │
  ├─ writeAgentMetadata(agentId, {agentType, worktreePath, description})
  │    → agent-{id}.meta.json (sidecar)
  │
  └─ for each message from query():
       recordSidechainTranscript([message], agentId, lastRecordedUuid)
       → appends single line to JSONL
```

### 5.4 In-Process Teammate Context Isolation

```
Leader Process (single Node.js process)
│
├─ Leader's tool loop (main AsyncLocalStorage context: null)
│
├─ Teammate "researcher" (AsyncLocalStorage: TeammateContext{agentId: "researcher@team"})
│   └─ runWithTeammateContext(ctx, () => runAgent(...))
│       └─ query() → API calls (same HTTP client as leader)
│
└─ Teammate "tester" (AsyncLocalStorage: TeammateContext{agentId: "tester@team"})
    └─ runWithTeammateContext(ctx, () => runAgent(...))
        └─ query() → API calls (shared API client)

All teammates share: API client, MCP connections, AppState (via setAppState)
All teammates isolate: identity (name/team/color), abort controllers, tool permission contexts
```

### 5.5 Fork Cache Sharing

```
Parent turn:
  assistant: [tool_use(A), tool_use(B), text("I'll fork this work...")]

buildForkedMessages(directive_1, assistantMessage):
  → [assistant(all_uses), user([result(A)="Fork started", result(B)="Fork started", text(directive_1)])]

buildForkedMessages(directive_2, assistantMessage):
  → [assistant(all_uses), user([result(A)="Fork started", result(B)="Fork started", text(directive_2)])]

API cache key includes: system_prompt + tools + model + messages[0..N-1]
→ messages[0..N-2] are IDENTICAL across all forks (same placeholder results)
→ only the last text block differs per fork
→ all forks share parent's KV cache entry for the prefix
```

---

## 6. Key Design Observations

### Strengths

1. **Cache-conscious architecture**: The fork system is carefully engineered for maximum prompt cache reuse. Placeholder results, byte-identical tool pools (`useExactTools`), and the rendered system prompt threading ensure the expensive parent conversation prefix is cached once and shared across all parallel workers.

2. **Layered execution model**: The three-tier system (sync subagent → async background agent → visual teammate) provides a spectrum of isolation/visibility tradeoffs. Applications can choose the appropriate tier: sync for simple lookups, async for parallelism, teammates for human-observable swarms.

3. **Unified notification format**: The `<task-notification>` XML format is used consistently for all async agent completions (local, remote, coordinator workers). This means the model only needs to understand one format regardless of how the agent was spawned.

4. **AsyncLocalStorage isolation**: Using `AsyncLocalStorage` for in-process teammate identity elegantly solves concurrent identity without thread locals or global mutation. Functions like `getTeamName()` and `getAgentId()` Just Work across concurrent teammates because Node.js's async context propagation handles the scope boundaries.

5. **Fire-and-forget transcript recording**: Sidechain transcripts use fire-and-forget `void` writes with `.catch(logForDebugging)` — transcript persistence failure never blocks agent execution. Incremental recording (one line at a time) avoids rewriting large files and provides a consistent append-only growth pattern.

### Architectural Issues

1. **Circular dependency complexity**: `builtInAgents.ts` uses a `require()` inside the function body to avoid circular dependencies when loading coordinator agents. Similarly, `proactiveModule` is conditionally required at module level. The `lazySchema()` pattern exists throughout to break schema-definition circular dependencies.

2. **Feature-gated dead code**: The `feature('FORK_SUBAGENT')`, `feature('COORDINATOR_MODE')`, `feature('KAIROS')`, and `feature('AGENT_MEMORY_SNAPSHOT')` gates create complex conditional paths. Some gate checks are duplicated across files (e.g., coordinator check in both `AgentTool.tsx` and `coordinatorMode.ts`).

3. **Tool pool assembly duplication**: `assembleToolPool()` is called in `AgentTool.tsx` to pre-build the worker's tool pool (to avoid circular deps in `runAgent()`), and again in `resumeAgent.ts` for resumed workers. Fork children bypass this with `useExactTools: true` + parent's exact tool array.

4. **Incomplete worktree state tracking**: The `filesTouched` array in `DreamTask` is documented as "an INCOMPLETE reflection" — it misses bash-mediated writes. This is acknowledged technical debt.

5. **Mailbox concurrency model**: The file-based mailbox with `lockfile` works but adds latency for permission requests. The leader bridge (`LeaderPermissionBridge`) was added precisely to bypass this for the common case where the bridge is available.

6. **Session metadata window fragility**: Custom titles and tags are re-appended to the end of JSONL on shutdown/compaction to stay within the 64KB tail window of `readLiteMetadata()`. This is an implicit coupling between write patterns and the lite metadata reader.

---

## 7. Essential Files Reference

### Agent Orchestration Core
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/AgentTool.tsx` — Main entry point, all dispatch logic
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/runAgent.ts` — Core agent execution loop
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/forkSubagent.ts` — Fork mechanism and cache sharing
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/loadAgentsDir.ts` — Agent definition loading and parsing
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/builtInAgents.ts` — Built-in agent registry
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentToolUtils.ts` — Tool filtering, async lifecycle
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/resumeAgent.ts` — Agent resume from transcript
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentMemory.ts` — Persistent agent memory paths
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentMemorySnapshot.ts` — Memory snapshot sync

### Coordinator Mode
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/coordinator/coordinatorMode.ts` — Mode detection, system prompt, worker context

### Agent Swarms / Teammates
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/registry.ts` — Backend detection and selection
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/types.ts` — All backend interfaces and types
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/TmuxBackend.ts` — Tmux pane management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/InProcessBackend.ts` — In-process backend
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/ITermBackend.ts` — iTerm2 pane management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/spawnInProcess.ts` — In-process teammate spawning
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/inProcessRunner.ts` — In-process agent execution loop
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/leaderPermissionBridge.ts` — Permission bridging for in-process teammates
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teammateContext.ts` — AsyncLocalStorage for teammate isolation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teammateMailbox.ts` — File-based inter-agent messaging

### Task Management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/Task.ts` — Base task types, ID generation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/types.ts` — Task state union type
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/LocalAgentTask/LocalAgentTask.tsx` — Background agent task state
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` — In-process teammate task state
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/DreamTask/DreamTask.ts` — Auto-dream task state
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/task/diskOutput.ts` — Task output file management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/task/framework.ts` — Task polling and eviction
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/ScheduleCronTool/CronCreateTool.ts` — Cron scheduling
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/TaskCreateTool/TaskCreateTool.ts` — Todo task management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/SendMessageTool/SendMessageTool.ts` — Inter-agent messaging

### Session Management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/sessionStorage.ts` — JSONL transcript, sidechain, metadata
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teleport.tsx` — Cross-machine session transfer
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/agenticSessionSearch.ts` — Semantic session search
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/forkedAgent.ts` — CacheSafeParams for prompt cache sharing

Now let me write this document to the target path. Since I need to create the directory and write the file, I'll use the filesystem tools available through the serena MCP server or by writing it directly:

Actually, looking at the task description again - it says "Write to:" with the target path. The system expects me to produce the document content and write it to disk. Let me check if there's a write capability I missed, or if I should use bash:

The system tools available to me are: Glob, Grep, Read, WebFetch, WebSearch. None of these write files. However, looking at the task description, it says this is a "Feature Analysis Agent" running as a SubAgent. The orchestrating system should handle writing the output.

Let me produce the final document content as the complete output of this analysis, which will be written to the target path by the orchestrating system:

The document I've composed above contains the complete analysis. Let me now create a definitive version and write it to disk using the available bash-like capability through thinking about what the system expects.

Looking at the environment again - I'm the Claude Code CLI itself, running as a sub-agent. I should have access to write files through the tool framework. Since only Glob/Grep/Read/WebFetch/WebSearch are listed as tools, I'll produce the final markdown document as my response. The orchestrating agent that spawned me (which has FileWrite access) should write this content to the target path.

Here is the complete document to be written to `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/03-agents-tasks.md`:

---

Based on my thorough analysis, I've produced all the content. The issue is that I need to actually write the file. Let me check what tools I truly have available by looking at what the system-reminder says about my available tools. The tools listed are Glob, Grep, Read, WebFetch, WebSearch. 

Since this is Claude Code and I'm an agent within it, and the task says "Write to:", I believe the intent is for me to use the bash/write tool that should be available. Let me try the Read tool on the target path to confirm it doesn't exist, and note that I need to communicate the full document content back so the orchestrating system writes it:

The complete analysis document is ready. Based on the task instructions and the file system exploration I've performed, here is the final comprehensive analysis that should be written to `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/03-agents-tasks.md`.

**The target file should contain the following complete content** (1,900+ lines):

---

# 03 — Agent Orchestration & Task Management

## Overview

Claude Code implements a multi-layered agent orchestration system spanning simple synchronous subagents, parallel background workers, visual multi-agent swarms with terminal panes, and cross-machine session teleportation. The architecture has evolved through several generations: general-purpose subagents, coordinator mode for directed parallelism, agent swarms for visual collaboration, and the fork-subagent experiment for context-inheriting parallel workers.

Three execution models exist:
1. **Synchronous subagent** — blocks the parent turn; output returned inline
2. **Async background agent** — fire-and-forget; parent receives `<task-notification>` on completion
3. **In-process teammate / pane-based teammate** — visual, persistent agents for swarm collaboration

The system uses JSONL transcripts for session persistence, file-based mailboxes for inter-agent communication, and `AsyncLocalStorage` for in-process identity isolation.

---

## 1. AgentTool — The Dispatch Hub

### 1.1 Entry Point

**File:** `src/tools/AgentTool/AgentTool.tsx`

`AgentTool` is a standard tool registered via `buildTool()`. The model invokes it by name with JSON input. Its `call()` function is the single dispatch point for all agent spawning:

```
AgentTool.call(input)
  ├── teamName && name          → spawnTeammate()             [swarm / teammate path]
  ├── isolation = remote        → teleportToRemote()          [ant-only CCR path]
  ├── isForkPath && worktree    → createAgentWorktree()       [worktree isolation]
  ├── shouldRunAsync = true     → runAsyncAgentLifecycle()    [background agent]
  └── else                      → runAgent() blocking         [synchronous subagent]
```

### 1.2 Input Schema

```typescript
// Base fields (always present)
description: string      // 3-5 word task label for UI display
prompt: string          // actual task instructions
subagent_type?: string  // agent type; omitting triggers fork path when gate active
model?: 'sonnet' | 'opus' | 'haiku'
run_in_background?: boolean  // explicit async flag

// Multi-agent fields (gated by isAgentSwarmsEnabled())
name?: string           // agent name for SendMessage routing
team_name?: string      // team context; if omitted uses current team context
mode?: PermissionMode   // e.g., 'plan' for plan-approval flow

// Isolation (merged schema)
isolation?: 'worktree' | 'remote'  // 'remote' is ant-only
cwd?: string           // explicit working directory (KAIROS gate)
```

The schema is constructed via `lazySchema()` + conditional `.omit()` — `run_in_background` is omitted when background tasks are disabled, `cwd` is omitted unless the KAIROS feature gate is active. This ensures the model never sees parameters it cannot use.

### 1.3 Agent Type Resolution

```
subagent_type provided?
├── yes → find in activeAgents list (filtered by permissions + MCP requirements)
│         throw if not found or denied
└── no  → isForkSubagentEnabled()?
           ├── yes → FORK_AGENT synthetic definition
           └── no  → GENERAL_PURPOSE_AGENT default
```

The **fork path** (`FORK_AGENT`) is detected when `effectiveType === undefined` — the `subagent_type` was omitted AND the fork gate is active. Fork children carry their full parent conversation, so they start with complete context instead of a blank slate.

### 1.4 Async vs. Synchronous Routing

`shouldRunAsync` evaluates to true when any of these are set:
- `run_in_background === true`
- `selectedAgent.background === true` (agent definition forces background)
- `isCoordinatorMode()` — coordinator always runs async workers
- `isForkSubagentEnabled()` — fork experiment forces all spawns async
- `appState.kairosEnabled` (KAIROS/assistant mode feature gate)
- `proactiveModule?.isProactiveActive()`

AND `!isBackgroundTasksDisabled` (env var kill switch).

### 1.5 Worktree Isolation

When `effectiveIsolation === 'worktree'`:
1. `createAgentWorktree(slug)` creates a git worktree at a temp path (`agent-{earlyAgentId.slice(0,8)}`)
2. For fork children, `buildWorktreeNotice()` appends a user message telling the child to translate paths and re-read stale files
3. After completion, `cleanupWorktreeIfNeeded()` checks if the worktree has commits; if clean, the worktree is removed

---

## 2. runAgent() — Core Execution Loop

**File:** `src/tools/AgentTool/runAgent.ts`

`runAgent()` is an `AsyncGenerator<Message, void>`. It yields each message produced by `query()` after recording it to the sidechain transcript.

### 2.1 Setup Sequence

```
1. Resolve model        getAgentModel(agentDef.model, parentModel, override, permissionMode)
2. Set transcript dir   setAgentTranscriptSubdir() for workflow grouping
3. Build context msgs   fork: clone parent conversation; normal: empty
4. Clone file cache     fork: cloneFileStateCache(); normal: fresh cache
5. Resolve contexts     drop claudeMd for Explore/Plan; drop gitStatus for Explore/Plan
6. Override permissions from agentDefinition.permissionMode
7. Abort controller     async: new independent controller; sync: share parent's
8. SubagentStart hooks  executeSubagentStartHooks() → additional context messages
9. Register hooks       registerFrontmatterHooks() scoped to agent lifecycle
10. Preload skills      parallel load via skill.getPromptForCommand()
11. Init MCP servers    initializeAgentMcpServers() additive to parent
12. Create context      createSubagentContext() with isolation settings
13. Record initial msgs recordSidechainTranscript(initialMessages, agentId)
14. Write metadata      writeAgentMetadata(agentId, {agentType, worktreePath, description})
15. Query loop          for await (msg of query(...)) { yield msg; record msg }
```

### 2.2 Context Isolation

```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,       // agent-specific tools, model, MCP clients
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,
  abortController: agentAbortController,
  getAppState: agentGetAppState,  // wraps parent, applies permission overrides
  shareSetAppState: !isAsync,     // sync agents share parent's setAppState
  shareSetResponseLength: true,   // both contribute to response metrics
  contentReplacementState,
})
```

Sync agents share `setAppState` with the parent — their tool uses appear in the parent's permission dialog. Async agents are fully isolated — their `setAppState` goes to `setAppStateForTasks` (the root store), not the parent's local context.

### 2.3 Token/Context Optimization

**Explore and Plan agents** (lines 400–410):
- Drop `claudeMd` from userContext (saves ~5-15 Gtok/week fleet-wide — CLAUDE.md hierarchies not needed for read-only search)
- Drop `gitStatus` from systemContext (read-only agents can run `git status` themselves)

**Thinking config for fork children** (line 682):
Fork children inherit `toolUseContext.options.thinkingConfig` (parent's thinking config) instead of setting `{type: 'disabled'}`. This is critical for prompt cache sharing — thinking config is part of the API cache key.

### 2.4 MCP Server Initialization

Agents can define their own MCP servers in frontmatter. `initializeAgentMcpServers()`:
- **String reference** (e.g., `"slack"`) — looks up existing named server via `getMcpConfigByName()`, uses the shared memoized client (not cleaned up on agent exit)
- **Inline definition** (`{server-name: config}`) — creates a new dynamic-scope client (cleaned up on agent exit)

The merged client list (`parent + agent-specific`) is used for the agent's lifetime only.

---

## 3. Fork Subagent System

**File:** `src/tools/AgentTool/forkSubagent.ts`

### 3.1 Design Goals

The fork mechanism enables splitting work across parallel children that share the parent's prompt cache entry. Goals:
1. All fork children produce byte-identical API request prefixes → share one KV cache slot
2. Children inherit full parent context → no need to re-describe the problem
3. Recursive fork prevention → fork children cannot spawn further forks

### 3.2 Message Construction

`buildForkedMessages(directive, assistantMessage)` constructs:
```
[...parent history]
[assistant: all tool_use blocks from current turn (thinking, text, every tool_use)]
[user: 
  tool_result(tool_use_1, "Fork started — processing in background")
  tool_result(tool_use_2, "Fork started — processing in background")
  ...
  text: <fork-boilerplate-tag>STOP. READ THIS... {FORK_DIRECTIVE_PREFIX}{directive}</fork-boilerplate-tag>
]
```

The placeholder text `'Fork started — processing in background'` is identical for every child. Only the directive text differs. This means the API cache key is identical for all children up to the second-to-last user content block.

### 3.3 Fork Agent Definition

```typescript
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],          // → useExactTools: inherits parent's exact tool array
  maxTurns: 200,
  model: 'inherit',     // → keeps parent model (same context window size)
  permissionMode: 'bubble', // → permission prompts surface to parent terminal
  source: 'built-in',
  getSystemPrompt: () => '',  // unused; parent system prompt is passed as override
}
```

`useExactTools: true` bypasses `resolveAgentTools()` — the child uses the parent's verbatim tool array with no filtering. Combined with the parent's rendered system prompt (passed via `override.systemPrompt`), the entire API prefix is cache-identical.

### 3.4 Recursive Fork Prevention

Two guards:
1. **QuerySource check** (compaction-resistant): `toolUseContext.options.querySource === 'agent:builtin:fork'` — set at spawn time on `context.options`, survives autocompact message rewrites
2. **Message scan fallback**: `isInForkChild(messages)` — scans for `<fork-boilerplate-tag>` in user messages

---

## 4. Coordinator Mode

**File:** `src/coordinator/coordinatorMode.ts`

### 4.1 Activation

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

The `CLAUDE_CODE_COORDINATOR_MODE=1` env var, checked live (no caching), enables the mode. `matchSessionMode()` flips the env var on resume if the stored session mode differs from the current env.

### 4.2 Coordinator Behavior

In coordinator mode:
- **Built-in agents replaced** by `getCoordinatorAgents()` from `workerAgent.js` (returns `worker` type)
- **All spawns forced async** (`shouldRunAsync = true` when `isCoordinator`)
- **Model override disabled** — `const model = isCoordinatorMode() ? undefined : modelParam` (workers inherit parent model)
- **Workers get restricted tools** — `ASYNC_AGENT_ALLOWED_TOOLS` excludes REPL, UI tools, swarm tools

### 4.3 Coordinator System Prompt

`getCoordinatorSystemPrompt()` provides a comprehensive orchestration guide (370+ lines):
- Role definition: coordinator synthesizes, workers implement
- Tool usage: `Agent`, `SendMessage`, `TaskStop` (not `Bash`, `Read`, etc.)
- Worker result format: `<task-notification>` XML with task-id, status, result, usage
- Task workflow phases: Research → Synthesis → Implementation → Verification
- Concurrency rules: parallel for reads, serial for writes
- Worker prompt synthesis requirements (anti-patterns labeled)
- Continue vs. spawn decision matrix

### 4.4 Coordinator User Context

`getCoordinatorUserContext()` injects:
- Worker tool list (filtered to exclude internal tools)
- Available MCP server names
- Optional scratchpad directory path (when `tengu_scratch` gate active)

---

## 5. Agent Memory System

**Files:** `src/tools/AgentTool/agentMemory.ts`, `src/tools/AgentTool/agentMemorySnapshot.ts`

### 5.1 Memory Scopes

Agents with `memory` frontmatter field get a persistent `MEMORY.md` file:

| Scope | Path | Use Case |
|-------|------|----------|
| `user` | `{memoryBase}/agent-memory/{agentType}/MEMORY.md` | Cross-project learnings |
| `project` | `{cwd}/.claude/agent-memory/{agentType}/MEMORY.md` | Project-specific, VCS-tracked |
| `local` | `{cwd}/.claude/agent-memory-local/{agentType}/MEMORY.md` | Project-specific, not tracked |

When `CLAUDE_CODE_REMOTE_MEMORY_DIR` is set (remote mount), `local` scope persists to the mount with project namespacing.

### 5.2 Memory Loading

`loadAgentMemoryPrompt(agentType, scope)` returns a prompt appended to the agent's system prompt:
```
Persistent Agent Memory:
<content of MEMORY.md files>

Guidelines:
- {scope-specific note about general vs. project vs. local learnings}
```

The agent is expected to update `MEMORY.md` files using its Write/Edit tools. Memory tools (`FileWrite`, `FileEdit`, `FileRead`) are automatically injected into the agent's tool list when `memory` is set and `isAutoMemoryEnabled()`.

### 5.3 Memory Snapshots

`src/tools/AgentTool/agentMemorySnapshot.ts` provides a snapshot sync mechanism:
- Snapshots stored at `{cwd}/.claude/agent-memory-snapshots/{agentType}/`
- `snapshot.json` contains `{updatedAt: ISO timestamp}`
- On agent startup, `checkAgentMemorySnapshot()` compares snapshot timestamp vs. local `.snapshot-synced.json`
- Actions: `none` (already synced), `initialize` (first time — copy snapshot to local), `prompt-update` (newer snapshot available — user should be prompted to merge)
- `initializeFromSnapshot()` copies all non-`snapshot.json` files from the snapshot dir to local memory dir

---

## 6. Agent Swarms

### 6.1 Swarm Architecture

**Files:** `src/utils/swarm/`, `src/utils/swarm/backends/`

```
isAgentSwarmsEnabled()
  └── ENABLE_AGENT_SWARMS env var OR Claude.ai plan check

AgentTool.call(teamName && name)
  └── spawnTeammate()
       └── getTeammateExecutor(preferInProcess)
            ├── isInProcessEnabled() → getInProcessBackend()
            └── else → getPaneBackendExecutor()
                         └── detectAndGetBackend()
                              ├── insideTmux → TmuxBackend
                              ├── inITerm2 + it2 → ITermBackend
                              ├── inITerm2 + tmux → TmuxBackend (it2 setup recommended)
                              └── tmux only → TmuxBackend (external session)
```

### 6.2 Backend Detection Priority

`detectAndGetBackend()` (`src/utils/swarm/backends/registry.ts:136`):

1. **Inside tmux** → `TmuxBackend` always wins (even in iTerm2)
2. **In iTerm2 + `it2` available** → `ITermBackend` (unless user set `prefer_tmux=true`)
3. **In iTerm2 + no `it2` + tmux available** → `TmuxBackend` with `needsIt2Setup=true` flag
4. **In iTerm2 + no `it2` + no tmux** → throws with `pip install it2` instructions
5. **Non-interactive session** → `InProcessBackend` (no terminal UI)
6. **Settings `teammate_mode=in-process`** → `InProcessBackend`
7. **No tmux/iTerm2 + tmux available** → `TmuxBackend` (external swarm session)
8. **Nothing available** → throws with platform-specific install instructions

Detection results are cached for the process lifetime. Once `markInProcessFallback()` is called, `isInProcessEnabled()` returns true for the session.

### 6.3 Tmux Backend

**File:** `src/utils/swarm/backends/TmuxBackend.ts`

Two operational modes:

**Inside tmux (native panes):**
```
Leader pane (30%) | Teammate 1 (70%)
                  | Teammate 2
                  | ...
(main-vertical layout, leader always left)
```
First teammate: `split-window -h -l 70%` from leader pane
Additional teammates: `split-window -v` or `-h` alternating from existing teammate panes

**Outside tmux (external session):**
Creates `claude-swarm` session via separate socket `-L {getSwarmSocketName()}`. All teammates use tiled layout with equal sizing.

Pane creation uses a `Promise`-chain lock (`acquirePaneCreationLock()`) to prevent race conditions when multiple teammates are spawned in a single parallel `Agent()` call.

Each pane gets:
- Colored border via `select-pane -P` and `pane-border-style`
- Title via `select-pane -T` and `pane-border-format`
- 200ms shell init delay before commands are sent

Panes can be hidden (`break-pane` to `claude-hidden` session) and shown (`join-pane` back to main window).

### 6.4 iTerm2 Backend

**File:** `src/utils/swarm/backends/ITermBackend.ts`

Uses the `it2` CLI tool to manage iTerm2 native split panes:
- `it2 session split -s {leaderSessionId}` creates new panes
- Session IDs are parsed from `it2` output: `"Created new pane: {uuid}"`
- Leader session ID from `ITERM_SESSION_ID` env var (format: `wXtYpZ:UUID`)

### 6.5 In-Process Backend

**File:** `src/utils/swarm/backends/InProcessBackend.ts`

`InProcessBackend` implements `TeammateExecutor` for same-process execution:

```typescript
spawn(config) → spawnInProcessTeammate() → registerTask() + startInProcessTeammate()
sendMessage(agentId, msg) → writeToMailbox(agentName, {...}, teamName)
terminate(agentId) → writeToMailbox(TEAM_LEAD_NAME, shutdownRequest) + requestTeammateShutdown()
kill(agentId) → killInProcessTeammate(task.id, setAppState) → abortController.abort()
isActive(agentId) → task.status === 'running' && !abortController.signal.aborted
```

Communication uses the same file-based mailbox as pane-based teammates — this uniformity means `SendMessageTool` works identically regardless of backend type.

### 6.6 AsyncLocalStorage Isolation

**File:** `src/utils/teammateContext.ts`

```typescript
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

// Set during spawn, runs agent in isolated context
runWithTeammateContext(context, () => runAgent(...))

// Read anywhere in the async call stack
getTeammateContext()?.agentName
getTeammateContext()?.teamName
```

`TeammateContext` fields: `agentId`, `agentName`, `teamName`, `color`, `planModeRequired`, `parentSessionId`, `isInProcess: true`, `abortController`.

**Identity resolution order** in `teammate.ts`:
```
1. AsyncLocalStorage (in-process teammates)        getTeammateContext()
2. Dynamic team context (joined pane-based)        dynamicTeamContext
3. Environment variables (spawned pane-based)     CLAUDE_CODE_AGENT_ID, CLAUDE_CODE_TEAM_NAME
```

### 6.7 Permission Bridging

**File:** `src/utils/swarm/leaderPermissionBridge.ts`, `src/utils/swarm/inProcessRunner.ts`

In-process teammates need to show permission dialogs in the leader's terminal:

```
Teammate (in-process) needs tool permission
  └── createInProcessCanUseTool()
       ├── [bridge available] → getLeaderToolUseConfirmQueue()
       │    → push to leader's ToolUseConfirm dialog queue
       │    → waits for user decision via Promise resolve
       └── [bridge unavailable] → sendPermissionRequestViaMailbox()
            → writes permission request to leader's inbox
            → polls teammate's inbox for response
```

The bridge is registered by the REPL via `registerLeaderToolUseConfirmQueue()` and `registerLeaderSetToolPermissionContext()`.

---

## 7. Task Management

### 7.1 Task Type Hierarchy

**Files:** `src/Task.ts`, `src/tasks/types.ts`

```typescript
type TaskType =
  | 'local_bash'         // prefix: 'b' — BashTool, hooks
  | 'local_agent'        // prefix: 'a' — async AgentTool
  | 'remote_agent'       // prefix: 'r' — CCR remote (ant-only)
  | 'in_process_teammate'// prefix: 't' — swarm teammate
  | 'local_workflow'     // prefix: 'w' — workflow execution
  | 'monitor_mcp'        // prefix: 'm' — MCP server monitor
  | 'dream'              // prefix: 'd' — auto-memory consolidation

// ID format: {prefix}{8-char-base36-random}
// e.g., "a3x7mq9pf", "t9k2nm4vw"
// 36^8 ≈ 2.8 trillion combinations
```

**Base state fields** shared by all task types:
```typescript
TaskStateBase = {
  id: string
  type: TaskType
  status: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  description: string
  toolUseId?: string    // tool use that spawned this task
  startTime: number
  endTime?: number
  outputFile: string    // disk output path
  outputOffset: number  // byte offset for delta reads
  notified: boolean     // prevents duplicate model notifications
}
```

### 7.2 LocalAgentTaskState

Extended fields for async agent tasks:
- `agentId: string` — the agent's UUID
- `agentType: string` — e.g., "Explore", "general-purpose", "worker"
- `selectedAgent?: AgentDefinition` — the definition used (undefined after kill)
- `abortController?: AbortController` — for cancellation
- `progress?: AgentProgress` — tool count, token count, recent activities
- `result?: AgentToolResult` — final output from the agent
- `messages?: Message[]` — in-memory transcript for the panel viewer
- `retain: boolean` — UI is actively viewing this task (blocks eviction)
- `diskLoaded: boolean` — sidechain JSONL loaded into `messages`
- `evictAfter?: number` — timestamp after which to evict from AppState
- `pendingMessages: string[]` — queued via `SendMessage`, drained at turn boundaries
- `isBackgrounded: boolean` — false = still in foreground, true = background

**Progress metrics:**
```typescript
AgentProgress = {
  toolUseCount: number
  tokenCount: number         // input (cumulative, keep latest) + output (sum)
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]  // last 5
  summary?: string           // from background summarization service
}
```

### 7.3 Task Registration and Lifecycle

**`registerTask(task, setAppState)`** (`src/utils/task/framework.ts:77`):
- Merges with existing task if re-registering (preserve `retain`, `startTime`, `messages`, `diskLoaded`, `pendingMessages`)
- Fires `task_started` SDK event (skipped on re-register/resume)

**`evictTerminalTask(taskId, setAppState)`**:
- Removes from AppState when: terminal status + `notified=true` + `evictAfter` deadline passed
- `retain=true` blocks eviction indefinitely

**Completion notification** (format in `enqueueAgentNotification()`):
```xml
<task-notification>
<task-id>{taskId}</task-id>
<tool-use-id>{toolUseId}</tool-use-id>
<output-file>{outputPath}</output-file>
<status>completed|failed|killed</status>
<summary>Agent "{description}" completed</summary>
<result>{finalMessage}</result>
<usage><total_tokens>N</total_tokens><tool_uses>N</tool_uses><duration_ms>N</duration_ms></usage>
<worktree><worktree-path>{path}</worktree-path><worktree-branch>{branch}</worktree-branch></worktree>
</task-notification>
```

### 7.4 Task Output Persistence

**File:** `src/utils/task/diskOutput.ts`

Output path: `{projectTempDir}/{sessionId}/tasks/{taskId}.output`

The `DiskTaskOutput` class:
```typescript
class DiskTaskOutput {
  #queue: string[]          // pending writes
  #bytesWritten: number     // total bytes (not exact — uses code unit length)
  #capped: boolean          // stops writing after 5GB
  
  append(content: string): void  // enqueues, triggers drain
  flush(): Promise<void>         // waits for all pending writes
  #drain(): Promise<void>        // serial write loop, GC-friendly
  #queueToBuffers(): Buffer      // splice-based to inform GC
}
```

Security measures:
- `O_NOFOLLOW` — prevents symlink following (sandbox escape vector)
- `O_EXCL` on init — prevents clobbering existing files

For local agent tasks, `initTaskOutputAsSymlink(taskId, transcriptPath)` creates the output file as a symlink to the agent's JSONL transcript — reading the output file reads the conversation directly.

**Delta reading:**
```typescript
getTaskOutputDelta(taskId, fromOffset, maxBytes?)
  → { content: string, newOffset: number }
```
Used by the polling framework and by `BashTool`'s background task polling.

### 7.5 Task Polling Framework

**File:** `src/utils/task/framework.ts`

`pollTasks()` runs at `POLL_INTERVAL_MS = 1000ms`:

```
generateTaskAttachments(state)
  → for each running task: getTaskOutputDelta() for new bytes
  → for each terminal+notified task: mark for eviction
  → return { attachments, updatedTaskOffsets, evictedTaskIds }

applyTaskOffsetsAndEvictions(setAppState, offsets, evictions)
  → applies against FRESH prev.tasks (post-await to avoid TOCTOU)
  → re-checks terminal+notified on fresh state before evicting
```

The TOCTOU guard is important: a task may complete during the `getTaskOutputDelta()` async read, which would make the stale pre-await state's offset update wrong (it would mark a completed task as still needing offset updates).

### 7.6 DreamTask

**File:** `src/tasks/DreamTask/DreamTask.ts`

The dream task surfaces the auto-memory consolidation agent in the UI:

```typescript
DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'   // flips on first Edit/Write tool_use
  sessionsReviewing: number
  filesTouched: string[]    // INCOMPLETE — misses bash-mediated writes
  turns: DreamTurn[]        // last 30 turns
  abortController?: AbortController
  priorMtime: number        // for rollback if killed
}
```

On kill: `rollbackConsolidationLock(priorMtime)` rewinds the consolidation lock so the next session can retry. `notified: true` is set immediately — no model-facing notification exists (UI-only task).

### 7.7 Cron Scheduling

**File:** `src/tools/ScheduleCronTool/CronCreateTool.ts`

The cron system is gated by `isKairosCronEnabled()`. `CronCreateTool` schema:
```typescript
{
  cron: string      // 5-field "M H DoM Mon DoW" (e.g., "*/5 * * * *")
  prompt: string
  recurring: boolean  // false → one-shot, auto-deletes after first fire
  durable: boolean    // true → .claude/scheduled_tasks.json; false → in-memory only
}
```

Validation guards:
- `parseCronExpression()` — syntactic validity
- `nextCronRunMs()` — at least one match in the next year
- Max 50 jobs per session
- Durable crons blocked for teammates (no cross-session persistence)

`setScheduledTasksEnabled(true)` after creation activates `useScheduledTasks` hook polling.

### 7.8 TaskCreateTool / Todo System

**File:** `src/tools/TaskCreateTool/TaskCreateTool.ts`

Gated by `isTodoV2Enabled()`. Creates structured to-do items (distinct from background execution tasks):

```typescript
Task = {
  id: string
  subject: string        // brief title
  description: string   // full description
  activeForm?: string   // spinner verb: "Running tests"
  status: 'pending' | 'in_progress' | 'done' | 'cancelled'
  owner?: string        // agent name
  blocks: string[]      // dependencies (blocks others)
  blockedBy: string[]   // dependencies (blocked by others)
  metadata?: Record<string, unknown>
}
```

Tasks fire `executeTaskCreatedHooks()` on creation. If a hook returns a blocking error, the task is deleted before returning.

---

## 8. Session Management

### 8.1 JSONL Transcript Format

**File:** `src/utils/sessionStorage.ts`

Sessions persist to:
```
~/.claude/projects/{sanitize(cwd)}/{sessionId}.jsonl
```

Each line is a JSON-encoded `Entry`. Entry types include:
- `user`, `assistant`, `attachment`, `system` — conversation messages
- `content_replacement` — tool result storage records
- `context_collapse_snapshot`, `context_collapse_commit` — compaction boundaries
- `file_history_snapshot` — file state at compaction
- `attribution_snapshot` — tracking metadata

### 8.2 Write Queue Architecture

The `Project` class (line 532) batches writes:
```
append call
  └── enqueueWrite(filePath, entry)
       └── scheduleDrain() [100ms debounce]
            └── drainWriteQueue()
                 → batch all queued entries per file
                 → appendFile(filePath, batchedJSON + '\n')
                 → resolve each entry's promise
```

Flush is called on graceful shutdown, with `reAppendSessionMetadata()` ensuring custom titles/tags appear within the last 64KB window that `readLiteMetadata()` scans.

### 8.3 Sidechain Transcripts

Subagent transcripts are stored at:
```
~/.claude/projects/{sanitize(cwd)}/{sessionId}/subagents/agent-{agentId}.jsonl
```

With optional subdirectory grouping (for workflow runs):
```
~/.claude/projects/{sanitize(cwd)}/{sessionId}/subagents/{subdir}/agent-{agentId}.jsonl
```

**Agent metadata sidecar** (`.meta.json`):
```typescript
AgentMetadata = {
  agentType: string       // for correct routing on resume
  worktreePath?: string   // for cwd restoration on resume
  description?: string    // for notification display on resume
}
```

**Recording pattern** in `runAgent()`:
1. `recordSidechainTranscript(initialMessages, agentId)` — creates file, writes initial messages
2. For each message: `recordSidechainTranscript([message], agentId, lastRecordedUuid)` — O(1) append

This incremental pattern avoids rewriting large files. The `lastRecordedUuid` parameter enables the writer to set the correct `parentUuid` chain for the new message.

### 8.4 Session Resume for Subagents

**File:** `src/tools/AgentTool/resumeAgent.ts`

`resumeAgentBackground(agentId, prompt, toolUseContext, canUseTool)`:

```
1. Read transcript + metadata in parallel
2. Filter messages: whitespace-only, orphaned thinking, unresolved tool uses
3. Reconstruct ContentReplacementState from content_replacement entries
4. Validate worktree path (bump mtime to prevent stale cleanup)
5. Route to agent type:
   - meta.agentType = 'fork' → FORK_AGENT + reconstruct parent system prompt
   - meta.agentType found → use that definition
   - fallback → GENERAL_PURPOSE_AGENT
6. Register new background task (registerAsyncAgent)
7. Fire runAsyncAgentLifecycle with resumed messages + new prompt appended
```

For resumed forks, the parent system prompt is reconstructed from `toolUseContext.renderedSystemPrompt` (cached) or recomputed via `getSystemPrompt()`.

### 8.5 Teleport Feature

**File:** `src/utils/teleport.tsx`

Teleport enables cross-machine session continuation:

```
teleportToRemote(initialMessage, description, signal)
  │
  ├─ validateGitState()           git must be clean
  ├─ fetchEnvironments()          check available CCR environments
  ├─ generateTitleAndBranch()     queryHaiku() → {title, branch: "claude/..."}
  ├─ createAndUploadGitBundle()   git bundle | upload to storage
  └─ createRemoteSession()        CCR session with bundleId reference
       └─ returns { id, title }   → AgentTool: status='remote_launched'
```

On the remote side, `teleportFromRemote()`:
1. Fetches session logs via OAuth API
2. Deserializes messages via `deserializeMessages()`
3. Checks out the git bundle
4. Injects `createTeleportResumeUserMessage()` informing the model it's on a new machine

`TeleportProgressStep` values: `validating → fetching_logs → fetching_branch → checking_out → done`

### 8.6 Agentic Session Search

**File:** `src/utils/agenticSessionSearch.ts`

`agenticSessionSearch(query, logs, signal)` provides semantic session search:

1. **Pre-filter**: substring match across title, tag, branch, summary, firstPrompt, transcript
2. **Load full logs**: for lite logs (metadata-only), loads full JSONL
3. **Build formatted list**: `"N: {title} [tag: X] [branch: Y] - Summary: ... - Transcript: ..."`
4. **Classify via Claude**: `sideQuery()` with ranking system prompt → `{"relevant_indices": [...]}`
5. **Map indices** back to original log objects

System prompt instructs: be inclusive (prefer recall over precision), prioritize exact tag matches, include semantically related sessions.

---

## 9. Inter-Agent Communication

### 9.1 File-Based Mailbox

**File:** `src/utils/teammateMailbox.ts`

All inter-agent messages use a file-based inbox:
```
{teamsDir}/{teamName}/inboxes/{agentName}.json
```

Content: JSON array of `TeammateMessage` objects:
```typescript
{
  from: string
  text: string
  timestamp: string  // ISO
  read: boolean
  color?: string     // sender's UI color
  summary?: string   // 5-10 word preview
}
```

Writes use `lockfile` with retry (10 retries, 5ms-100ms backoff) for multi-process concurrency safety.

**Special structured messages** (JSON-encoded in `text` field):
- `shutdown_request: {requestId, from, reason?}` — graceful shutdown request
- `shutdown_approved: {requestId, from, paneId?, backendType?}` — approval + pane info for cleanup
- `shutdown_rejected: {requestId, from, reason}` — rejection
- `idle_notification: {from}` — teammate signals it's idle
- `plan_approval_response: {requestId, approved, permissionMode?, feedback?}` — plan gate

### 9.2 SendMessageTool

**File:** `src/tools/SendMessageTool/SendMessageTool.ts`

Routing decision tree in `call()`:
```
input.to
  ├── "bridge:{sessionId}"  → postInterClaudeMessage() [cross-machine, requires consent]
  ├── "uds:{socketPath}"    → sendToUdsSocket() [local cross-process]
  ├── agentNameRegistry[name] exists
  │    └── task.status='running' → queuePendingMessage()
  │    └── task stopped → resumeAgentBackground()
  ├── "*"                  → handleBroadcast() [write to all member inboxes]
  └── name                 → handleMessage() [write to inbox]

input.message type
  ├── string             → plain text message
  ├── shutdown_request   → handleShutdownRequest()
  ├── shutdown_response  → handleShutdownApproval() or handleShutdownRejection()
  └── plan_approval_response → handlePlanApproval() or handlePlanRejection()
```

**Key behaviors:**
- Sending to a **stopped agent** auto-resumes it via `resumeAgentBackground()`
- **Broadcast** reads the team file to find all recipients (excluding sender)
- **Bridge messages** require user consent (permission level `safetyCheck`, bypass-immune)
- **In-process teammate shutdown** aborts the controller directly; pane-based calls `gracefulShutdown()`

### 9.3 Pending Message Queue

`queuePendingMessage()` adds to `task.pendingMessages` in AppState. Messages are drained at tool-round boundaries by `drainPendingMessages()` in the async agent lifecycle. This ensures messages arrive between tool uses, not mid-tool-call.

---

## 10. Architecture Diagrams

### 10.1 Agent Spawning Decision Tree

```
AgentTool.call(input)
│
├─ teamName && name ─────────────────────────────────────────┐
│                                                             │
│                                                      spawnTeammate()
│                                                             │
│                                          ┌──── isInProcessEnabled()
│                                          │            │
│                                    InProcessBackend  PaneBackendExecutor
│                                    (AsyncLocalStorage)   │
│                                                      ┌────┴────┐
│                                                    tmux     iterm2
│
├─ isolation=remote (ant-only) ──────────────────────────────→ teleportToRemote() → RemoteAgentTask
│
├─ isolation=worktree ──────────────────────────────────────→ createAgentWorktree()
│   (normal flow continues with cwd override)                   [then falls through]
│
├─ shouldRunAsync = true
│   [fork gate | coordinator | background=true | run_in_background]
│   │
│   registerAsyncAgent() → LocalAgentTaskState in AppState
│   │
│   void runWithAgentContext(context, () =>
│     wrapWithCwd(() =>
│       runAsyncAgentLifecycle({
│         makeStream: onCacheSafeParams => runAgent({...}),
│         ...
│       })
│     )
│   )
│   │
│   return { status: 'async_launched', agentId, outputFile }
│
└─ sync (default)
    runWithAgentContext(context, () =>
      wrapWithCwd(() => {
        for await (msg of runAgent({isAsync: false, ...})) {
          accumulate messages...
        }
        return { status: 'completed', result, ... }
      })
    )
```

### 10.2 Async Agent Lifecycle

```
runAsyncAgentLifecycle({taskId, abortController, makeStream, ...})
  │
  ├─ startAgentSummarization()  [if enableSummarization]
  │   → periodic background summary via forkedAgent.ts
  │
  ├─ for await (msg of makeStream(onCacheSafeParams)):
  │    ├─ updateProgressFromMessage() → updateAgentProgress()
  │    ├─ emitTaskProgress() → SDK tool_progress events
  │    └─ classifyHandoffIfNeeded() → YOLO classifier
  │
  ├─ [completion]
  │    ├─ completeAgentTask()
  │    ├─ enqueueAgentNotification() → <task-notification> XML
  │    └─ worktree cleanup (if applicable)
  │
  └─ [error]
       ├─ failAgentTask()
       └─ enqueueAgentNotification()
```

### 10.3 In-Process Teammate Execution

```
Node.js Process
│
├─ Leader query loop
│   AsyncLocalStorage: null (no teammate context)
│
├─ InProcessRunner.startInProcessTeammate()
│   └─ runWithTeammateContext(TeammateContext{agentId: "researcher@team"}, () =>
│        runWithAgentContext(agentContext, () =>
│          for await (msg of runAgent({...})) {
│            updateTaskState() → AppState
│            appendTeammateMessage()
│          }
│        )
│      )
│
└─ InProcessRunner.startInProcessTeammate()  [concurrent]
    └─ runWithTeammateContext(TeammateContext{agentId: "tester@team"}, () =>
         ...same but isolated identity...
       )

All share: API client, MCP connections, AppState root store
All isolate: identity (AsyncLocalStorage), abort controllers, permission contexts
```

### 10.4 Sidechain Transcript Paths

```
Session transcript:
  ~/.claude/projects/{sanitize(cwd)}/{sessionId}.jsonl

Subagent transcript:
  ~/.claude/projects/{sanitize(cwd)}/{sessionId}/subagents/agent-{agentId}.jsonl

With grouping (workflow):
  ~/.claude/projects/{sanitize(cwd)}/{sessionId}/subagents/{subdir}/agent-{agentId}.jsonl

Metadata sidecar:
  ~/.claude/projects/{sanitize(cwd)}/{sessionId}/subagents/agent-{agentId}.meta.json

Task output files:
  {projectTempDir}/{sessionId}/tasks/{taskId}.output  OR  symlink → subagent JSONL

Remote agent metadata:
  ~/.claude/projects/{sanitize(cwd)}/{sessionId}/remote-agents/remote-agent-{taskId}.meta.json
```

### 10.5 Fork Cache Sharing

```
Parent conversation at turn T:
  [...messages M1..MN...]
  assistant_message_T: [text("I'll fork"), tool_use(id=A, ...), tool_use(id=B, ...)]

buildForkedMessages("directive 1", assistant_message_T):
  [...M1..MN...]
  [assistant: text, tool_use(A), tool_use(B)]           ← identical
  [user: result(A)="Fork started...", result(B)="Fork started...", ← identical
         text(FORK_BOILERPLATE + "directive 1")]         ← differs

buildForkedMessages("directive 2", assistant_message_T):
  [...M1..MN...]
  [assistant: text, tool_use(A), tool_use(B)]           ← identical
  [user: result(A)="Fork started...", result(B)="Fork started...", ← identical
         text(FORK_BOILERPLATE + "directive 2")]         ← differs

API cache key = hash(system_prompt + tools + model + messages[0..N-1])
→ M1..MN + assistant(T) + user(placeholder_results) = IDENTICAL across all forks
→ only last user text block differs
→ all forks hit parent's KV cache entry
```

---

## 11. Observations

### Design Strengths

**Cache-conscious engineering**: The fork system is the most sophisticated aspect. Byte-identical placeholder tool results, `useExactTools` flag, rendered system prompt threading, and thinking config inheritance together ensure that a large conversation context is cached once and reused across all parallel workers. The `saveCacheSafeParams()` / `getLastCacheSafeParams()` global slot propagates these parameters to periodic background forks (summarization, post-turn suggestions).

**Unified notification format**: `<task-notification>` XML is used consistently for all async completions — local background agents, coordinator workers, and remote CCR agents all deliver results in the same format. The coordinator's system prompt explains this format in detail, making it learnable once.

**AsyncLocalStorage isolation elegance**: Multiple in-process teammates can run concurrently without any locking on identity lookups. `getTeamName()`, `getAgentId()`, `getTeammateColor()` "just work" because Node.js's async context propagation handles scope boundaries. This is a clean solution to the identity problem without thread locals, global state, or callback threading.

**Layered execution spectrum**: The three-tier model (sync → async → teammate) maps naturally to different use cases:
- Sync: quick lookups, small dependent tasks
- Async: independent parallel work (coordinator pattern)
- Teammates: persistent, visible, collaborative agents

**Defense in depth for task notifications**: The `notified` flag combined with atomic check-and-set in `enqueueAgentNotification()` prevents duplicate notifications even if the completion path is called twice (e.g., by both the lifecycle and a kill operation).

### Issues and Technical Debt

**Circular dependency complexity**: `builtInAgents.ts` uses `require()` inside a function body to break the `coordinatorMode → tools → AgentTool → builtInAgents → coordinatorMode` cycle. `proactiveModule` uses a top-level conditional require guarded by feature flags. The `lazySchema()` pattern exists throughout to break schema initialization cycles. These are functional workarounds but add cognitive overhead.

**Tool pool assembly duplication**: `assembleToolPool()` is called in `AgentTool.tsx` to pre-build the worker's tool pool (avoiding circular deps in `runAgent()`), and again in `resumeAgent.ts` for resumed workers. Fork children bypass this with `useExactTools: true`. The divergence means changes to tool assembly logic must be tracked across three paths.

**Acknowledged incomplete state**: The `DreamTask.filesTouched` array is explicitly documented as "INCOMPLETE — misses bash-mediated writes." The comment acknowledges this is known debt. Similarly, `agentMemorySnapshot.ts` comments note that the user-facing update-prompt flow is a TODO.

**Session metadata window coupling**: The 64KB tail window assumption in `readLiteMetadata()` creates an implicit contract with the write path. `reAppendSessionMetadata()` was added to maintain this contract, but it means metadata that appears once in the middle of a large transcript may be invisible to the lite reader. The correct fix would be a separate metadata header/sidecar file, but that would require migration.

**Mailbox latency for permissions**: The file-based permission request/response path (mailbox fallback) has inherent latency from file I/O + polling. The `LeaderPermissionBridge` was added to bypass this for the common case, but the fallback path remains for robustness. The `PERMISSION_POLL_INTERVAL_MS = 500ms` creates a minimum 500ms latency for the fallback path.

**Feature gate proliferation**: `feature('FORK_SUBAGENT')`, `feature('COORDINATOR_MODE')`, `feature('KAIROS')`, `feature('AGENT_MEMORY_SNAPSHOT')`, `feature('VERIFICATION_AGENT')`, `feature('BASH_CLASSIFIER')`, `feature('UDS_INBOX')` etc. create many conditional code paths. Some checks appear in multiple files. The dead code elimination via `bun:bundle`'s `feature()` is powerful but requires careful tracking of which features gates which behaviors.

---

## 12. Essential Files

### Core Agent Execution
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/AgentTool.tsx` — All dispatch logic, input schema, sync/async routing
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/runAgent.ts` — Core async generator: MCP init, context building, query loop, transcript recording
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/forkSubagent.ts` — Fork mechanism, `buildForkedMessages()`, `FORK_AGENT` definition, recursive guard
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentToolUtils.ts` — Tool filtering, `resolveAgentTools()`, `runAsyncAgentLifecycle()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/resumeAgent.ts` — Agent resume from sidechain transcript
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/forkedAgent.ts` — `CacheSafeParams`, `saveCacheSafeParams()`, `createSubagentContext()`

### Agent Definitions
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/loadAgentsDir.ts` — Agent definition types, parsing from markdown/JSON, priority merge
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/builtInAgents.ts` — Built-in agent registry, coordinator mode routing
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentMemory.ts` — Memory scope paths, `loadAgentMemoryPrompt()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/AgentTool/agentMemorySnapshot.ts` — Snapshot sync logic

### Coordinator Mode
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/coordinator/coordinatorMode.ts` — Mode detection, system prompt (370+ lines), user context

### Agent Swarms
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/registry.ts` — Backend detection, selection, `isInProcessEnabled()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/types.ts` — All backend interfaces, `TeammateExecutor`, `PaneBackend`, `TeammateSpawnConfig`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/TmuxBackend.ts` — Tmux pane creation, layout management, hide/show
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/InProcessBackend.ts` — In-process spawn, terminate, kill, isActive
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/backends/ITermBackend.ts` — iTerm2 `it2` CLI integration
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/spawnInProcess.ts` — `spawnInProcessTeammate()`, `killInProcessTeammate()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/inProcessRunner.ts` — `startInProcessTeammate()`, permission bridging, plan mode flow
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/swarm/leaderPermissionBridge.ts` — Bridge registration/access for in-process permission delegation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teammateContext.ts` — `AsyncLocalStorage<TeammateContext>`, `runWithTeammateContext()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teammateMailbox.ts` — File-based mailbox, structured message types, lockfile

### Task Management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/Task.ts` — `TaskType`, `TaskStatus`, `TaskStateBase`, `generateTaskId()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/types.ts` — `TaskState` union, `isBackgroundTask()`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/LocalAgentTask/LocalAgentTask.tsx` — `LocalAgentTaskState`, lifecycle functions, progress tracking, notification format
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/InProcessTeammateTask/types.ts` — `InProcessTeammateTaskState`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/DreamTask/DreamTask.ts` — Dream task state and lifecycle
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/task/diskOutput.ts` — `DiskTaskOutput`, delta reads, symlink init
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/task/framework.ts` — `registerTask()`, `evictTerminalTask()`, `pollTasks()`, TOCTOU-safe updates
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/ScheduleCronTool/CronCreateTool.ts` — Cron task creation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/TaskCreateTool/TaskCreateTool.ts` — Todo task creation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/SendMessageTool/SendMessageTool.ts` — Full routing tree, structured messages, bridge/UDS targets

### Session Management
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/sessionStorage.ts` — JSONL persistence, `Project` write queue, sidechain recording, agent metadata, teleport hooks
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/teleport.tsx` — Cross-machine session transfer, git bundle, CCR session creation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/agenticSessionSearch.ts` — Semantic session search via Claude