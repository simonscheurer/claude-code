# Core Engine & Runtime: Comprehensive Feature Analysis

## Document Overview

This document provides an in-depth analysis of four tightly-coupled subsystems that form the runtime core of Claude Code CLI:

1. **Conversation Engine** - The agentic query loop, streaming, tool dispatch
2. **Context Management & Compaction** - The 3-tier system that keeps context within model limits
3. **Bootstrap & Global State** - Startup sequence and the global singleton
4. **Memory System** - CLAUDE.md hierarchy, session memory, auto-dream consolidation

---

## Part 1: Conversation Engine (Core Agentic Loop)

### 1.1 Architecture Overview

The conversation engine is split across two complementary layers:

```
QueryEngine (class, SDK-facing)
    |
    +-- submitMessage() -------> query() -------> queryLoop()
                                     |                |
                                     |         [while(true) loop]
                                     |                |
                                     |         microcompact
                                     |         autocompact
                                     |         callModel() (claude.ts)
                                     |         streaming event loop
                                     |         tool execution
                                     |         stop-hooks
                                     |         continue / return
```

`QueryEngine` (`src/QueryEngine.ts`) owns per-session state: the mutable message array, the abort controller, total usage accumulation, and permission denial tracking. It is instantiated once per conversation in SDK/headless mode. The REPL (interactive mode) uses the older `ask()` function path that calls `query()` directly.

`query()` (`src/query.ts`) is a stateless async generator that drives the inner `while(true)` loop. All cross-iteration state is carried in a typed `State` object that is reassigned at continue sites.

### 1.2 Entry Points

| Entry Point | File | Used By |
|---|---|---|
| `QueryEngine.submitMessage()` | `src/QueryEngine.ts:209` | SDK, headless/print mode |
| `query()` | `src/query.ts:219` | REPL (via ask()), QueryEngine |
| `queryModelWithStreaming()` | `src/services/api/claude.ts:752` | query() via deps.callModel |
| `queryModelWithoutStreaming()` | `src/services/api/claude.ts:709` | compact, side-question |

### 1.3 Message Lifecycle: Step-by-Step

#### Phase 1: System Prompt Assembly (QueryEngine.submitMessage)

```
QueryEngine.submitMessage(prompt)
  |
  +-- fetchSystemPromptParts()          [src/utils/queryContext.ts:44]
  |     |-- getSystemPrompt(tools, model, ...)   [parallel]
  |     |-- getUserContext()                      [parallel]
  |     +-- getSystemContext()                    [parallel]
  |
  +-- asSystemPrompt([...parts])        compose final system prompt
  |     includes: customPrompt OR defaultSystemPrompt
  |               + memoryMechanicsPrompt (if CLAUDE_COWORK_MEMORY_PATH_OVERRIDE)
  |               + appendSystemPrompt (if set)
  |
  +-- processUserInput()                slash-command processing
        returns: { messages, shouldQuery, allowedTools, model, resultText }
```

`fetchSystemPromptParts` runs three async operations in parallel via `Promise.all`. When a custom system prompt is provided, `getSystemPrompt` and `getSystemContext` are short-circuited entirely (returning empty arrays/objects). This is a critical performance optimization - the default system prompt assembly involves loading CLAUDE.md files from disk, computing tool descriptions, etc.

#### Phase 2: Message Normalization

Before entering the query loop, user input goes through `processUserInput()`:
- Slash command detection and execution (e.g. `/compact`, `/clear`)
- Attachment generation (memory files, file contents)
- Tool allowlist updates from user permissions

The resulting messages are appended to `mutableMessages` and a transcript write is triggered (fire-and-forget in bare mode, awaited otherwise).

#### Phase 3: The Query Loop (query.ts)

The core `queryLoop()` is a `while(true)` generator. Each iteration:

```
ITERATION START
  |
  1. Destructure state (messages, toolUseContext, tracking, ...)
  2. Prefetch: startSkillDiscoveryPrefetch()  [non-blocking]
  3. yield { type: 'stream_request_start' }
  4. applyToolResultBudget()              content-replace oversized tool results
  5. snipCompactIfNeeded()                [if HISTORY_SNIP feature]
  6. microcompactMessages()               time-based or cached MC
  7. applyCollapsesIfNeeded()             [if CONTEXT_COLLAPSE feature]
  8. autoCompactIfNeeded()                threshold-based full compaction
  9. CHECK: isAtBlockingLimit?            synthetic error + return
  10. callModel(messages, systemPrompt, ...) --> streaming loop
  11. [streaming loop: consume stream events, build assistant messages]
  12. executePostSamplingHooks()          fire-and-forget
  13. CHECK: abortController.signal.aborted? --> handle abort
  14. yield pendingToolUseSummary         Haiku-generated summary
  15. IF !needsFollowUp:
        - prompt-too-long recovery (collapse drain or reactive compact)
        - max_output_tokens recovery (escalate or inject continue message)
        - handleStopHooks()
        - token budget check
        - return { reason: 'completed' }
  16. runTools() or streamingToolExecutor.getRemainingResults()
  17. yield tool results
  18. generateToolUseSummary()            non-blocking, Haiku
  19. state = { messages: [..., assistantMessages, toolResults], ... }
  20. continue (next iteration)
```

The loop terminates by `return`ing a `Terminal` value (`{ reason: ... }`) with one of these reasons:
- `'completed'` - normal end_turn
- `'aborted_streaming'` - user interrupt
- `'blocking_limit'` - context too long, auto-compact off
- `'prompt_too_long'` - recovery exhausted
- `'max_turns'` - signalled via attachment
- `'model_error'` - unexpected throw
- `'stop_hook_prevented'` - hook blocked continuation
- `'image_error'` - media too large
- `'reactive_compact_retry'` / `'collapse_drain_retry'` - loop continues

### 1.4 Streaming Event Processing (claude.ts)

`queryModel()` (`src/services/api/claude.ts:1017`) is the innermost async generator that calls the Anthropic API and translates raw SSE events into typed `StreamEvent` wrappers and `AssistantMessage` objects.

#### SSE Event Types Processed

The raw stream delivers `BetaRawMessageStreamEvent` objects. The handler at `claude.ts:1979` switches on `part.type`:

| Event Type | Action |
|---|---|
| `message_start` | Initialize `partialMessage`, capture ttft, update usage |
| `content_block_start` | Allocate slot in `contentBlocks[]` by index; types: `tool_use`, `text`, `thinking`, `server_tool_use` |
| `content_block_delta` | Accumulate: `text_delta` → text, `input_json_delta` → tool input string, `thinking_delta` → thinking text, `signature_delta` → thinking signature |
| `content_block_stop` | Finalize block; parse tool input JSON; emit `AssistantMessage` for completed blocks |
| `message_delta` | Update usage (`output_tokens`, `cache_read_input_tokens`), capture `stop_reason` |
| `message_stop` | Emit final `AssistantMessage` with all blocks |

Tool input is accumulated as a raw JSON string across `input_json_delta` events, then parsed once at `content_block_stop`. This avoids the O(n²) partial JSON parsing that BetaMessageStream does internally.

Every raw event is also yielded as a `stream_event` wrapper:
```typescript
yield {
  type: 'stream_event',
  event: part,
  ...(part.type === 'message_start' ? { ttftMs } : undefined),
}
```

`QueryEngine` filters these: only passes `stream_event` to SDK callers when `includePartialMessages: true`.

#### Content Block Finalization → AssistantMessage

When `content_block_stop` fires, the accumulated block is finalized. For `tool_use` blocks, the accumulated JSON string is parsed via `JSON.parse()`. The resulting `AssistantMessage` is:

```typescript
{
  type: 'assistant',
  message: {
    role: 'assistant',
    content: [/* completed content blocks */],
    usage: ...,           // from message_delta
    stop_reason: null,    // filled in by message_delta
  },
  uuid: randomUUID(),
  timestamp: Date.now(),
  requestId: streamRequestId,
  // ...
}
```

Multiple assistant messages can be emitted per turn (one per completed content block batch). The model may interleave thinking blocks, text blocks, and tool_use blocks.

### 1.5 Parallel Tool Call Handling

Claude Code supports two execution modes for tool calls:

#### Sequential Mode (default when StreamingToolExecutor is disabled)

After the streaming loop completes, `runTools()` (`src/services/tools/toolOrchestration.ts`) is called with all accumulated `toolUseBlocks`. Tools in a single assistant turn are run... sequentially or in parallel depending on the tool's configuration. The results are gathered, yielded as `UserMessage` objects with `tool_result` content, and appended to messages before the next iteration.

#### Streaming Tool Executor (when `streamingToolExecution` gate is on)

`StreamingToolExecutor` (`src/services/tools/StreamingToolExecutor.ts`) starts executing tools as their `tool_use` blocks arrive during streaming, before `message_stop`. Each `content_block_stop` for a `tool_use` block triggers `addTool()`, which queues the execution. The executor runs them in parallel (up to its concurrency limit). Completed results are drained via `getCompletedResults()` inside the stream loop, so tool results can start arriving before the model finishes generating.

This overlaps model generation latency with tool execution latency - a critical optimization for multi-tool turns.

On streaming fallback (model switches mid-stream), the executor's pending results are discarded and a fresh one is created:
```typescript
streamingToolExecutor.discard()
streamingToolExecutor = new StreamingToolExecutor(...)
```

### 1.6 Retry and Error Handling

Error handling is layered:

#### Layer 1: withRetry (src/services/api/withRetry.ts)

`withRetry()` wraps the API client call with up to `DEFAULT_MAX_RETRIES` (10) attempts:

- **401/403 auth errors**: force token refresh, re-create client
- **429 rate limit**: exponential backoff, respect `Retry-After` header
- **529 overload**: up to `MAX_529_RETRIES` (3) consecutive retries for foreground sources; background sources (compact, session_memory, etc.) bail immediately to avoid amplification cascades
- **Fast mode rate limit**: enter cooldown, switch to standard speed, retry
- **Model fallback**: after 3 consecutive 529s on Opus, throw `FallbackTriggeredError`
- **Stale connections (ECONNRESET/EPIPE)**: disable keep-alive, reconnect
- **OAuth token revocation (403)**: handle token refresh

The `CLAUDE_CODE_UNATTENDED_RETRY` mode (ant-only) enables indefinite retries with heartbeat yields and higher backoff.

#### Layer 2: FallbackTriggeredError (query.ts)

When `withRetry` throws `FallbackTriggeredError`, `queryLoop` catches it, switches to the fallback model, yields tombstones for orphaned messages (which have model-bound thinking signatures), and retries the entire request.

#### Layer 3: Non-streaming Fallback

When the streaming connection fails (idle timeout, no events, proxy errors), the code falls back to `executeNonStreamingRequest()`. This sends the same parameters to `anthropic.beta.messages.create()` without `stream: true`. A timeout of 120s (remote) or 300s (local) caps hung requests.

Idle timeout watchdog: if no SSE chunks arrive for `STREAM_IDLE_TIMEOUT_MS` (default 90s), an AbortController aborts the stream. This is gated behind `CLAUDE_ENABLE_STREAM_WATCHDOG`.

#### Layer 4: Max Output Tokens Recovery

If the model hits its output token limit (`stop_reason === 'max_tokens'`):
1. Try escalating from 8K default to `ESCALATED_MAX_TOKENS` (64K) - one-shot
2. Inject a meta user message asking the model to continue without recap
3. Retry up to `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` (3) times
4. Surface the withheld error if exhausted

#### Layer 5: Prompt Too Long Recovery

If the API returns a 413-like prompt-too-long error:
1. Withhold the error message (don't yield it yet)
2. Try context collapse drain (if CONTEXT_COLLAPSE feature)
3. Try reactive compact (if REACTIVE_COMPACT feature)
4. If both fail, surface the error and execute stop-failure hooks

### 1.7 QueryEngine vs. REPL Differences

`QueryEngine` (SDK path) differs from the REPL path in several ways:

| Aspect | QueryEngine | REPL |
|---|---|---|
| State persistence | `mutableMessages` in-memory | React state in AppState |
| Transcript write | Awaited before query | Fire-and-forget |
| Stream events | Optionally yielded to SDK | Consumed by React hooks |
| Message replay | Configurable via `replayUserMessages` | Not applicable |
| Max turns | Enforced with result emission | Via attachment signal |
| Snip replay | Injected via `snipReplay` | In-process via `projectSnippedView` |

### 1.8 QueryEngine Configuration

`QueryEngineConfig` (`src/QueryEngine.ts:130`) is the primary configuration surface:

```typescript
type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
  handleElicitation?: ToolUseContext['handleElicitation']
  includePartialMessages?: boolean
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission
  snipReplay?: (yieldedSystemMsg, store) => { messages, executed } | undefined
}
```

### 1.9 System Prompt Assembly Pipeline

The final system prompt sent to the API is assembled in layers:

```
[customSystemPrompt] OR [defaultSystemPrompt (from getSystemPrompt)]
+ [memoryMechanicsPrompt]   (if CLAUDE_COWORK_MEMORY_PATH_OVERRIDE set)
+ [appendSystemPrompt]      (if provided)
                            = base systemPrompt

Then in queryModel() (claude.ts):
+ getAttributionHeader(fingerprint)    e.g. attribution prefix
+ getCLISyspromptPrefix(...)           e.g. "You are Claude Code..."
+ [ADVISOR_TOOL_INSTRUCTIONS]          (if advisor enabled)
+ [CHROME_TOOL_SEARCH_INSTRUCTIONS]    (if chrome tools + non-delta mode)
                                       = fullSystemPrompt
```

`fullSystemPrompt` is converted to `BetaTextBlockParam[]` by `buildSystemPromptBlocks()`, which adds `cache_control` markers to enable prompt caching. The system prompt is split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` into a static prefix (cached) and dynamic suffix (not cached).

### 1.10 Token Budget Feature

When `feature('TOKEN_BUDGET')` is enabled, the `budgetTracker` tracks output tokens across iterations within a single agentic turn. `checkTokenBudget()` evaluates whether the model should continue (inject a nudge message), stop early (diminishing returns signal), or complete normally. This prevents excessively long single-turn responses.

---

## Part 2: Context Management & Compaction

### 2.1 Overview: 3-Tier Compaction Hierarchy

Context management operates at three distinct levels, applied in order at the start of each query loop iteration:

```
QUERY ITERATION START
    |
    v
[Tier 0] applyToolResultBudget()   -- content-replace huge individual results
    |
    v  
[Tier 1] snipCompactIfNeeded()     -- remove old messages by tail-protecting
    |
    v
[Tier 2] microcompactMessages()    -- truncate stale tool results in-place
    |                                 OR via cache-editing API (cached MC)
    v
[Tier 3] applyCollapsesIfNeeded()  -- context-collapse (context preservation)
    |
    v
[Tier 4] autoCompactIfNeeded()     -- full summarization via forked agent
    |
    v
API CALL
```

### 2.2 Auto-Compaction Thresholds

All thresholds are relative to the effective context window:

```
effectiveContextWindow = contextWindowForModel - MAX_OUTPUT_TOKENS_FOR_SUMMARY(20,000)
```

| Threshold | Buffer | Meaning |
|---|---|---|
| `autoCompactThreshold` | -13,000 tokens | Triggers proactive auto-compact |
| `warningThreshold` | -20,000 tokens | Yellow warning in UI |
| `errorThreshold` | -20,000 tokens | Red warning in UI |
| `blockingLimit` | -3,000 tokens | Hard block (no auto-compact) |

The `AUTOCOMPACT_BUFFER_TOKENS` constant (13,000) is the gap between the effective window and the auto-compact trigger. This ensures there are always enough tokens for the model to produce a summary response.

These thresholds can be overridden:
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` - override the effective context window
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` - set auto-compact threshold as percentage
- `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` - override the blocking limit

### 2.3 shouldAutoCompact() Decision Logic

```typescript
async function shouldAutoCompact(messages, model, querySource, snipTokensFreed):
  if querySource === 'session_memory' || 'compact': return false   // recursion guard
  if querySource === 'marble_origami': return false                // context-collapse guard
  if !isAutoCompactEnabled(): return false
  if feature('REACTIVE_COMPACT') && tengu_cobalt_raccoon: return false  // reactive-only mode
  if feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled(): return false
  
  tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  return tokenCount >= autoCompactThreshold
```

The recursion guards are critical: `compact` and `session_memory` are themselves forked agents that inherit the full conversation. If autocompact fired inside them, they could deadlock (trying to compact the conversation they're summarizing).

### 2.4 Tier 2: Micro-Compaction

Micro-compaction (`src/services/compact/microCompact.ts`) operates on individual tool results rather than the full conversation. It has two sub-paths:

#### Time-Based Micro-Compact

Fires when the time since the last API call exceeds a configured threshold (default: typically hours). If the server-side cache has expired (TTL ~5min for ephemeral, ~1h for 1h-TTL cache), rewriting the full message history is unavoidable anyway. In this case, micro-compact proactively clears stale tool result content to shrink the history before it gets re-cached.

The trigger is evaluated via `evaluateTimeBasedTrigger()` which checks:
1. querySource is main-thread (`repl_main_thread` prefix)
2. Time gap exceeds the configured threshold
3. There is at least one prior assistant message to timestamp from

When this fires, it returns immediately, skipping the cached MC path.

#### Cached Micro-Compact (CACHED_MICROCOMPACT feature, ant-only)

Rather than modifying message content (which would bust the prompt cache), cached MC uses the cache-editing API beta (`cache-editing-beta-2024-12-26`). It:

1. Registers tool results in `CachedMCState` as they appear in the message history
2. Groups tool results by user message position
3. When trigger threshold is exceeded (configurable via GrowthBook `tengu_cachedmc_config`), computes a `cache_edits` block
4. The `cache_edits` block is consumed from `pendingCacheEdits` by `claude.ts` and inserted into the API request
5. The API server deletes the referenced cache slots without invalidating the prefix

Key insight: **Cached MC does not modify local message content**. The message array on the client remains unchanged. Only the server-side cache is modified. The `cache_deleted_input_tokens` field in the API response reports how many tokens were freed.

The boundary message is deferred until after the API response so it can include the actual `cache_deleted_input_tokens` delta rather than a client-side estimate.

#### Compactable Tool Types

Only a specific set of tools are eligible for micro-compaction:
```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

Tools that produce structured data referenced by later turns (e.g., the synthetic output tool) are intentionally excluded.

### 2.5 Tier 4: Full Compaction (compact.ts)

Full compaction is the most aggressive: it summarizes the entire conversation into a single text block and discards the original messages.

#### Algorithm

```
compactConversation(messages, context, ...):
  1. tokenCount = tokenCountWithEstimation(messages)
  2. executePreCompactHooks() → may add customInstructions, userDisplayMessage
  3. compactPrompt = getCompactPrompt(customInstructions)
  4. summaryRequest = createUserMessage(compactPrompt)
  
  5. PTL retry loop:
     for (;;):
       summaryResponse = streamCompactSummary(messages, summaryRequest, ...)
       if summary doesn't start with PROMPT_TOO_LONG_ERROR_MESSAGE: break
       // Compact request itself hit PTL
       truncated = truncateHeadForPTLRetry(messages, summaryResponse)
       if !truncated OR attempts > MAX_PTL_RETRIES: throw ERROR_MESSAGE_PROMPT_TOO_LONG
       messages = truncated
  
  6. Store preCompactReadFileState = cacheToObject(readFileState)
  7. readFileState.clear()
  8. loadedNestedMemoryPaths.clear()
  
  9. [parallel]:
     a. createPostCompactFileAttachments()  re-inject recently read files
     b. createAsyncAgentAttachmentsIfNeeded()
  10. planAttachment = createPlanAttachmentIfNeeded()
  11. skillAttachment = createSkillAttachmentIfNeeded()
  12. re-announce delta attachments (deferred tools, agent listing, MCP instructions)
  
  13. processSessionStartHooks()    re-run startup hooks
  14. createCompactBoundaryMessage()
  15. summaryMessages = [createUserMessage(getCompactUserSummaryMessage(summary, ...))]
  
  16. executePostCompactHooks()
  17. logEvent('tengu_compact', ...)
  18. notifyCompaction() (prompt cache break detection reset)
  19. markPostCompaction()
  
  20. return CompactionResult {
        boundaryMarker, summaryMessages, attachments, hookResults,
        preCompactTokenCount, postCompactTokenCount, truePostCompactTokenCount,
        compactionUsage
      }
```

#### Image Handling During Compaction

`stripImagesFromMessages()` (`src/services/compact/compact.ts:145`) is called before sending messages to the summarization agent. It replaces:
- `image` blocks → `[image]` text placeholder
- `document` blocks → `[document]` text placeholder
- Nested images/documents inside `tool_result` content arrays → same placeholders

This prevents the compact request from hitting its own prompt-too-long limit due to base64-encoded image data.

#### PTL Retry Algorithm for Compact

When the compact request itself hits prompt-too-long (`truncateHeadForPTLRetry`):
1. Group messages by API round (using `groupMessagesByApiRound`)
2. If token gap is parseable: compute how many groups to drop to cover the gap
3. If not parseable: drop 20% of groups as fallback
4. Always keep at least 1 group
5. If remaining messages start with an assistant message, prepend a synthetic user marker
6. Retry up to `MAX_PTL_RETRIES` (3) times

#### Post-Compact State

After compaction, the message array is replaced with:
```
[boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
```

`messagesToKeep` is populated in reactive compact and session-memory compact (which preserve recent messages). Full proactive compact sets it to empty.

The file read cache is cleared so the model starts fresh. However, `sentSkillNames` is intentionally NOT reset - re-injecting the full skill listing (~4K tokens) post-compact is pure cache creation. Invoked skills are re-injected via the `skillAttachment` instead.

### 2.6 Reactive Compact (REACTIVE_COMPACT feature, ant-only)

Reactive compact is triggered by a 413 prompt-too-long error from the API (post-streaming). Unlike proactive auto-compact, it runs after the API rejects the request. It:
1. Withholds the prompt-too-long error message during streaming
2. After streaming, calls `tryReactiveCompact()`
3. On success, replaces messages with post-compact messages and retries
4. On failure, surfaces the withheld error

The `hasAttemptedReactiveCompact` flag prevents infinite loops.

### 2.7 Session Memory Compact (sessionMemoryCompact.ts)

Before full compaction runs, `autoCompactIfNeeded` tries session-memory compaction first:

```typescript
const sessionMemoryResult = await trySessionMemoryCompaction(
  messages, agentId, autoCompactThreshold
)
if (sessionMemoryResult) {
  // Session memory compact pruned messages without summarization
  return { wasCompacted: true, compactionResult: sessionMemoryResult }
}
```

Session memory compact works differently: instead of summarizing, it prunes messages that were already captured in `MEMORY.md` (the session memory file). The pruned range is from the last summary boundary to the current `lastSummarizedMessageId`. This avoids the cost of running the summarization model if the session memory agent has already captured the important information.

### 2.8 Circuit Breaker for Auto-Compact

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

if (tracking?.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { wasCompacted: false }
}
```

After 3 consecutive auto-compact failures, the circuit breaker trips and no further auto-compact attempts are made. The failure count is tracked in `AutoCompactTrackingState.consecutiveFailures`, which persists across loop iterations. BQ data (2026-03-10) showed 1,279 sessions had 50+ consecutive failures in a single session, wasting ~250K API calls/day globally before this was added.

### 2.9 Manual Compact (/compact command)

The `/compact` slash command triggers `compactConversation()` directly with `isAutoCompact: false` and optional custom instructions from the user. The key difference:
- `suppressFollowUpQuestions: false` (model may ask clarifying questions)
- Custom instructions can be passed by the user
- Errors are surfaced to the user via notifications (not silently retried)

### 2.10 Partial Compact

`partialCompactConversation()` allows compacting a subset of the conversation:

- **Direction `'from'`**: summarizes messages AFTER a pivot index; keeps earlier messages (cache-safe, no cache bust)
- **Direction `'up_to'`**: summarizes messages BEFORE a pivot index; keeps later messages (invalidates cache since summary precedes kept messages)

The `'up_to'` direction strips old compact boundaries and summaries from the kept segment to avoid confusing `findLastCompactBoundaryIndex`.

### 2.11 analyzeContext.ts

`analyzeContext()` (`src/utils/analyzeContext.ts`) provides a detailed breakdown of what is consuming context tokens:

Categories computed include: system prompt, tool schemas, user/assistant messages, thinking blocks, file attachments, tool results. Each category is token-counted via the API's token-counting endpoint (`countMessagesTokensWithAPI`), with `countTokensViaHaikuFallback` as a fallback.

The 500-token `TOOL_TOKEN_COUNT_OVERHEAD` is subtracted from per-tool counts to avoid N×overhead inflation when counting N tools individually.

This data feeds the `/context` command UI and the `tokenStatsToStatsigMetrics()` function that annotates compaction analytics events.

---

## Part 3: Bootstrap & Global State

### 3.1 The Global State Singleton

`src/bootstrap/state.ts` defines and maintains a single module-level `STATE` object. The comment "DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE" reflects that the file has grown to hold ~60+ fields.

#### Why a Singleton?

Bootstrap is explicitly designed as a **leaf of the import DAG** - it imports from nothing except a few narrow utilities (`crypto.ts`, `settings/settingsCache.ts`, `signal.ts`). This prevents circular dependencies. Any module that needs session-level state can import from `bootstrap/state.ts` without pulling in the rest of the application.

This is enforced by the custom ESLint rule `bootstrap-isolation` which checks that bootstrap files don't import from paths starting with `./` or `/` except for explicitly whitelisted leaf modules.

#### State Shape (key fields)

**Identity & Session**
```typescript
sessionId: SessionId           // randomUUID() at startup
parentSessionId?: SessionId    // set when plan mode spawns impl session
originalCwd: string            // resolved realpath of startup dir
projectRoot: string            // stable; set once including --worktree
cwd: string                    // current working directory (may change)
```

**Cost & Usage Tracking**
```typescript
totalCostUSD: number
totalAPIDuration: number           // includes retries
totalAPIDurationWithoutRetries: number
totalToolDuration: number
modelUsage: { [modelName]: ModelUsage }
```

**Turn-Level Metrics** (reset each turn for rate/timing)
```typescript
turnHookDurationMs, turnToolDurationMs, turnClassifierDurationMs
turnToolCount, turnHookCount, turnClassifierCount
```

**Feature-Flag Latches** (sticky-on to prevent cache busting)
```typescript
afkModeHeaderLatched: boolean | null    // auto-mode beta header
fastModeHeaderLatched: boolean | null   // fast mode beta header  
cacheEditingHeaderLatched: boolean | null  // cache editing beta header
thinkingClearLatched: boolean | null    // clear thinking after idle
promptCache1hEligible: boolean | null   // 1h TTL eligibility
promptCache1hAllowlist: string[] | null // GrowthBook allowlist
```

**Telemetry**
```typescript
meter: Meter | null
sessionCounter, locCounter, prCounter, commitCounter, etc.
loggerProvider, eventLogger, meterProvider, tracerProvider
```

**Session Flags** (boolean flags controlling session behavior)
```typescript
isInteractive, kairosActive, strictToolResultPairing
sdkAgentProgressSummariesEnabled, sessionBypassPermissionsMode
scheduledTasksEnabled, sessionTrustAccepted, sessionPersistenceDisabled
hasExitedPlanMode, lspRecommendationShownThisSession
```

**Debugging & Diagnostics**
```typescript
lastAPIRequest: Omit<BetaMessageStreamParams, 'messages'> | null
lastAPIRequestMessages: BetaMessageStreamParams['messages'] | null
lastClassifierRequests: unknown[] | null
inMemoryErrorLog: Array<{ error: string; timestamp: string }>
slowOperations: Array<{ operation, durationMs, timestamp }>
```

**Memory/Skills**
```typescript
invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
cachedClaudeMdContent: string | null
systemPromptSectionCache: Map<string, string | null>
```

**Cache State**
```typescript
lastApiCompletionTimestamp: number | null  // for idle TTL detection
pendingPostCompaction: boolean             // tag first post-compact API call
lastMainRequestId: string | undefined      // for cache eviction hints
```

### 3.2 Session Identity and Lifecycle

```typescript
// Create new session (keeps existing parent chain)
regenerateSessionId()
  --> STATE.parentSessionId = STATE.sessionId  (if setCurrentAsParent)
  --> STATE.sessionId = randomUUID()
  --> drops planSlugCache entry for old session

// Switch to an existing session (--resume)
switchSession(sessionId, projectDir)
  --> STATE.sessionId = sessionId
  --> STATE.sessionProjectDir = projectDir
  --> fires sessionSwitched signal

// Subscribe to session switches
onSessionSwitch(callback)  // via createSignal().subscribe
```

`getSessionProjectDir()` returns null in the common case (current project), meaning `getTranscriptPath()` derives the path from `originalCwd`. When non-null (cross-project resume, git worktrees), it points to the directory containing the transcript `.jsonl` file.

### 3.3 Startup Sequence (init.ts)

`init()` is memoized (only runs once). It is called in `main.tsx` after `enableConfigs()`.

```
init():
  1. enableConfigs()                    parse settings.json, projectConfig
  2. applySafeConfigEnvironmentVariables()  safe env vars (no trust needed)
  3. applyExtraCACertsFromConfig()      CA certs before first TLS connection
  4. setupGracefulShutdown()            register SIGTERM/SIGINT handlers
  5. [fire-and-forget] initialize1PEventLogging() + GrowthBook refresh listener
  6. [fire-and-forget] populateOAuthAccountInfoIfNeeded()
  7. [fire-and-forget] initJetBrainsDetection()
  8. [fire-and-forget] detectCurrentRepository()
  9. [conditional] initializeRemoteManagedSettingsLoadingPromise()
  10. [conditional] initializePolicyLimitsLoadingPromise()
  11. recordFirstStartTime()
  12. configureGlobalMTLS()             mTLS settings
  13. configureGlobalAgents()           HTTP proxy agents
  14. preconnectAnthropicApi()          TCP+TLS pre-warm [fire-and-forget]
  15. [conditional] initUpstreamProxy()  CCR upstream proxy
  16. setShellIfWindows()
  17. registerCleanup(shutdownLspServerManager)
  18. registerCleanup(cleanupSessionTeams)
  19. [conditional] ensureScratchpadDir()
```

Several operations run in parallel via fire-and-forget (`void Promise...`), overlapping with the remaining sequential setup:
- GrowthBook initialization
- OAuth account info population
- JetBrains IDE detection
- Repository detection

`preconnectAnthropicApi()` is called after CA certs and proxy agents are configured but before the main UI renders, aiming to overlap the ~100-200ms TCP+TLS handshake with ~100ms of action-handler work.

### 3.4 CLI Fast Paths (cli.tsx)

Before calling `init()`, `cli.tsx:main()` checks for fast paths that avoid loading most modules:

```
--version / -v     Zero imports. MACRO.VERSION inlined at build time. ~0ms
--dump-system-prompt   Ant-only. Load config + prompts only.
--claude-in-chrome-mcp   Chrome extension MCP server
--chrome-native-host   Chrome native messaging host
--computer-use-mcp   Computer use MCP server (CHICAGO_MCP feature)
--daemon-worker=<kind>   Daemon worker process (DAEMON feature)
remote-control / bridge   Bridge mode (BRIDGE_MODE feature)
daemon   Long-running supervisor (DAEMON feature)
ps/logs/attach/kill/--bg   Background session management (BG_SESSIONS feature)
```

Each fast path avoids loading the full React/Ink UI stack (~several hundred KB of modules).

### 3.5 Feature Flags via bun:bundle

Claude Code uses Bun's `feature()` function from the `bun:bundle` virtual module for dead-code elimination at build time:

```typescript
import { feature } from 'bun:bundle'

// Entire block eliminated from external builds when false
const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null

// Conditional import pattern
const coordinator = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js').getCoordinatorUserContext
  : () => ({})
```

Feature flags that are build-time constants (true/false):
- `HISTORY_SNIP` - message history pruning
- `REACTIVE_COMPACT` - post-413 compaction
- `CONTEXT_COLLAPSE` - hierarchical context management
- `CACHED_MICROCOMPACT` - cache-editing micro-compact
- `EXTRACT_MEMORIES` - background memory extraction
- `TRANSCRIPT_CLASSIFIER` - auto-mode classification
- `CHICAGO_MCP` - computer-use MCP server
- `BRIDGE_MODE` - remote control bridge
- `DAEMON` - background supervisor
- `BG_SESSIONS` - background session management
- `KAIROS` - session transcript compression
- `ANTI_DISTILLATION_CC` - anti-distillation markers
- `ABLATION_BASELINE` - A/B test ablation
- `BASH_CLASSIFIER` - bash permission classifier
- `COORDINATOR_MODE` - multi-agent coordinator
- `UNATTENDED_RETRY` - indefinite retry for unattended sessions
- `TOKEN_BUDGET` - per-turn output token budget
- `CONNECTOR_TEXT` - connector text block type
- `DUMP_SYSTEM_PROMPT` - CLI prompt dumping
- `TEAMMEM` - team memory sharing
- `TEMPLATES` - job classifier templates
- `EXPERIMENTAL_SKILL_SEARCH` - skill discovery prefetch

Features that are false in external (non-Anthropic) builds are listed in `excluded-strings.txt` for build verification. When a feature is `false`, the entire require/import block and all string literals inside it are eliminated via tree-shaking.

The `feature()` function is syntactically constrained: it must appear directly in `if`/ternary conditions (not stored in variables) for Bun's tree-shaking to recognize it. This is enforced by a custom ESLint rule.

### 3.6 GrowthBook Integration

Feature flags that are runtime-configurable (not build-time) use GrowthBook:
- `getFeatureValue_CACHED_MAY_BE_STALE(key, defaultValue)` - returns cached value, updates async
- `getDynamicConfig_CACHED_MAY_BE_STALE(key, defaultValue)` - same for config objects
- `getDynamicConfig_BLOCKS_ON_INIT(key, defaultValue)` - blocks until GrowthBook initialized

The `_CACHED_MAY_BE_STALE` naming is intentional: these values may be stale if GrowthBook hasn't refreshed yet. For session-stability-sensitive flags (prompt cache TTL, beta headers), values are latched into bootstrap state on first access.

---

## Part 4: Memory System

### 4.1 CLAUDE.md Hierarchy

The memory hierarchy is defined in `src/utils/claudemd.ts`. Files are loaded in priority order (lowest to highest), with later files taking precedence:

```
Priority 1 (lowest): Managed  /etc/claude-code/CLAUDE.md
                               /etc/claude-code/.claude/rules/*.md
Priority 2:          User      ~/.claude/CLAUDE.md
                               ~/.claude/rules/*.md
Priority 3:          Project   (each dir from root to cwd)
                               <dir>/CLAUDE.md
                               <dir>/.claude/CLAUDE.md
                               <dir>/.claude/rules/*.md
Priority 4 (highest): Local    (each dir from root to cwd)
                               <dir>/CLAUDE.local.md
```

**Discovery Algorithm**

`getMemoryFiles()` is memoized (single computation per session). It:
1. Starts from `originalCwd` and walks up the directory tree to the filesystem root
2. For each directory, loads all four file types in priority order
3. For nested git worktrees, skips Project-type files from the main repo to avoid duplication

**@include Directive**

Memory files can include other files:
```
@path          relative path (same as @./path)
@./relative    relative to including file
@~/home/path   relative to home directory
@/absolute     absolute path
```

Include processing:
- Parses the file content with the Marked lexer (tokenized, not regex)
- Extracts `@path` references from text tokens only (skips code blocks)
- Recursively processes included files with the parent file as context
- Maximum include depth prevents infinite loops
- Processed paths are tracked in a Set to prevent circular inclusion
- `TEXT_FILE_EXTENSIONS` allowlist prevents loading binary files

**Frontmatter Support**

Rules files can have YAML frontmatter with `paths:` to make them conditional:
```markdown
---
paths:
  - "src/**/*.ts"
  - "tests/**"
---
Only applies to TypeScript files and tests
```

`parseFrontmatterPaths()` extracts these patterns. Files without frontmatter are "unconditional" (always loaded). Files with frontmatter are "conditional" (only loaded when the CWD matches the path patterns). The `processMdRules()` function separates these two categories via the `conditionalRule` parameter.

**Content Processing**

For each loaded file:
1. `parseFrontmatterPaths()` strips frontmatter and extracts path constraints
2. `stripHtmlComments()` removes block-level `<!-- -->` comments (using the Marked lexer to preserve comments inside code blocks)
3. `truncateEntrypointContent()` caps at 200 lines AND 25,000 bytes for MEMORY.md

When content is transformed (stripped, truncated), `contentDiffersFromDisk: true` is set so the file read cache records a "partial view" - this allows deduplication without permitting direct edits (which require an explicit fresh read).

**Exclusion via claudeMdExcludes**

Users can exclude specific files via `claudeMdExcludes` in settings.json. Patterns use picomatch. Symlinks in absolute path prefixes are resolved to handle macOS `/tmp → /private/tmp` mapping.

### 4.2 MEMORY.md and Auto-Memory Path

`MEMORY.md` is the entrypoint file for the auto-memory system (`src/memdir/memdir.ts`):

```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

The auto-memory path resolution (`src/memdir/paths.ts`):

```
CLAUDE_CODE_REMOTE_MEMORY_DIR (env) → used directly
OR
~/.claude/projects/<sanitized-cwd>/memory/
```

`getAutoMemPath()` returns the directory; `getAutoMemEntrypoint()` returns `MEMORY.md` within it. The path is sanitized and validated with security checks against root/relative/UNC paths and null bytes.

**isAutoMemoryEnabled()** returns false when:
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY` is truthy
- `CLAUDE_CODE_SIMPLE` (--bare mode) is set
- Running in CCR (remote) without `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- `autoMemoryEnabled: false` in settings.json

### 4.3 Session Memory (Real-Time Capture)

Session Memory (`src/services/SessionMemory/sessionMemory.ts`) runs a background agent after each model response to capture key information from the current conversation.

**Trigger Conditions**

`shouldExtractMemory()` returns true when ALL of:
1. `isSessionMemoryInitialized()`: token count has crossed `minimumMessageTokensToInit` (default: configurable via GrowthBook `tengu_sm_config`)
2. Either:
   - Token threshold (`minimumTokensBetweenUpdate`) AND tool call threshold (`toolCallsBetweenUpdates`) are both met
   - OR token threshold is met AND the last turn had no tool calls (natural break point)

Token threshold always takes precedence - extraction won't fire until enough new tokens have accumulated since the last extraction.

**Extraction Process**

```
extractSessionMemory(context):
  1. Guard: querySource must be 'repl_main_thread'
  2. Guard: isSessionMemoryGateEnabled() (GrowthBook 'tengu_session_memory')
  3. initSessionMemoryConfigIfNeeded() (lazy, once per session)
  4. shouldExtractMemory(messages) check
  5. markExtractionStarted()
  6. setupSessionMemoryFile():
     - mkdir $sessionMemoryDir (mode 0700)
     - create MEMORY.md with template (flag 'wx' = O_CREAT|O_EXCL)
     - FileReadTool.call() to get current content
  7. buildSessionMemoryUpdatePrompt(currentMemory, memoryPath)
  8. runForkedAgent({
       promptMessages: [updatePrompt],
       canUseTool: createMemoryFileCanUseTool(memoryPath),
       querySource: 'session_memory',
     })
  9. recordExtractionTokenCount()
  10. updateLastSummarizedMessageIdIfSafe()
  11. markExtractionCompleted()
```

`runForkedAgent` creates a perfect fork of the main conversation context (sharing the parent's prompt cache). The forked agent can only write to the memory file - a restrictive `canUseTool` implementation prevents any other file writes.

Session memory is registered as a `registerPostSamplingHook()` handler, wrapped in `sequential()` to prevent concurrent extractions.

**Session Memory Config** (configurable via GrowthBook)

```typescript
const DEFAULT_SESSION_MEMORY_CONFIG = {
  minimumMessageTokensToInit: ...,    // tokens before first extraction
  minimumTokensBetweenUpdate: ...,    // min token growth between extractions
  toolCallsBetweenUpdates: ...,       // min tool calls between extractions
}
```

**Session Memory Compaction Integration**

When auto-compact triggers, `trySessionMemoryCompaction()` is called first. If the session memory has captured messages beyond the last summarized ID, those messages can be pruned from the conversation without full summarization. This is cheaper than running the summarization model.

### 4.4 Extract Memories (Background Agent)

`extractMemories` (`src/services/extractMemories/extractMemories.ts`) is a separate background agent that runs once at the end of complete turns (no pending tool calls), extracting durable facts to the auto-memory directory.

This differs from Session Memory:
- Session Memory: per-session ephemeral capture → `~/.claude/projects/<cwd>/.session-memory/MEMORY.md`
- Extract Memories: durable cross-session facts → `~/.claude/projects/<cwd>/memory/` (multi-file)

The extract memories agent uses `runForkedAgent` with `querySource: 'extract_memories'`. It has access to read the conversation and write/edit files in the memory directory.

**Trigger**: Registered via `registerPostSamplingHook()` within `initExtractMemories()`. Fires when:
- feature('EXTRACT_MEMORIES') is enabled
- `isExtractModeActive()` returns true (GrowthBook gated, interactive sessions)
- `isAutoMemoryEnabled()` returns true
- Enough new model-visible messages have accumulated since last extraction (tracked by `countModelVisibleMessagesSince`)

### 4.5 Auto Dream (Cross-Session Memory Consolidation)

Auto Dream (`src/services/autoDream/autoDream.ts`) is a background process that consolidates memories across multiple sessions into a coherent, well-organized memory structure.

**Design**: Gate order (cheapest first):
1. **Feature gate**: `isAutoDreamEnabled()` + `isAutoMemoryEnabled()` + not KAIROS + not remote
2. **Time gate**: hours since `lastConsolidatedAt` >= `minHours` (default: 24)
3. **Scan throttle**: 10-minute minimum between session list scans (to avoid filesystem overhead when time gate keeps passing but session gate doesn't)
4. **Session gate**: count of transcript files newer than `lastConsolidatedAt` >= `minSessions` (default: 5, excludes current session)
5. **Lock**: file-based lock preventing concurrent consolidation across processes

**Lock Mechanism** (`consolidationLock.ts`):
- `readLastConsolidatedAt()`: reads mtime of the lock file
- `tryAcquireConsolidationLock()`: atomically updates lock file mtime, returns prior mtime for rollback
- `rollbackConsolidationLock(priorMtime)`: restores prior mtime on failure

**Consolidation Process**

```
runAutoDream(context):
  1. [All gates pass]
  2. Get memoryRoot, transcriptDir
  3. prompt = buildConsolidationPrompt(memoryRoot, transcriptDir, toolConstraints)
  4. registerDreamTask(setAppState, { sessionsReviewing, priorMtime, abortController })
  5. runForkedAgent({
       promptMessages: [prompt],
       canUseTool: createAutoMemCanUseTool(memoryRoot),  // read-only bash
       querySource: 'auto_dream',
       skipTranscript: true,
       overrides: { abortController }
     })
  6. completeDreamTask(taskId, setAppState)
  7. appendSystemMessage("Improved N memory files") to main transcript
  8. logEvent('tengu_auto_dream_completed', { cache_read, cache_created, ... })
```

The consolidation agent reads past transcripts and existing memory files, then reorganizes, deduplicates, and enhances the memory structure. It is constrained to read-only bash commands (ls, find, grep, cat, stat) and can only write to `memoryRoot`.

**Failure Recovery**: On failure, `rollbackConsolidationLock(priorMtime)` rewinds the lock file mtime so the time gate will pass again on the next turn. The scan throttle (10 minutes) provides backoff.

**Progress Tracking**: `makeDreamProgressWatcher()` monitors the forked agent's messages, extracting text blocks (reasoning) and tracking file touches (Edit/Write tool calls on specific paths). The `DreamTask` in AppState is updated with:
- Current status (running/completed/failed/killed)
- Number of sessions reviewing
- Files touched so far

### 4.6 Memory File Loading in System Prompt

`getMemoryFiles()` is called during system prompt assembly by `getSystemPrompt()`. The loaded files are formatted with a preamble:

```
MEMORY_INSTRUCTION_PROMPT:
"Codebase and user instructions are shown below. Be sure to adhere to these instructions. 
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly."
```

Each memory file contributes its content as a separate system prompt block. Files from MEMORY.md (the auto-memory entrypoint) are passed through `truncateEntrypointContent()` before inclusion.

The `getMemoryFiles()` result is memoized per session - it only runs once. This is important because the function involves filesystem I/O (reading every CLAUDE.md in the directory tree). For interactive mode, memory files are re-read on each compaction (the memoize cache is invalidated when `readFileState.clear()` is called, though actually `getMemoryFiles` memoization is based on a different key).

### 4.7 Memory Path Security

The auto-memory path validation (`validateMemoryPath()`) rejects:
- Relative paths (not `isAbsolute()`)
- Root/near-root paths (length < 3)
- Windows drive roots (matches `/^[A-Za-z]:$/`)
- UNC paths (starts with `\\` or `//`)
- Null bytes (can truncate in syscalls)

This prevents an attacker-controlled memory path from pointing to sensitive system directories.

---

## Part 5: Cross-Cutting Concerns and Dependencies

### 5.1 The forkedAgent Pattern

Many background subsystems (session memory, extract memories, auto dream, compact) share the `runForkedAgent()` pattern (`src/utils/forkedAgent.ts`). A forked agent:

1. Creates a sub-`ToolUseContext` isolated from the parent
2. Inherits the parent's prompt cache prefix (via `cacheSafeParams`)
3. Has a restricted `canUseTool` implementation
4. Runs its own `query()` loop
5. Does not write back to the parent's message array

The `cacheSafeParams` carry the system prompt, user context, and system context that was used for the parent's last API call. By reusing these, the forked agent's first API call hits the parent's cache, making it cheap (cache read rather than cache creation).

### 5.2 Prompt Cache Strategy

Claude Code uses a sophisticated multi-layer prompt caching strategy:

**System prompt caching**: The system prompt is split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` into a static prefix (marked with `cache_control: { type: 'ephemeral' }`) and a dynamic suffix (not cached). The static prefix contains: managed/user CLAUDE.md content, core tool descriptions, model-specific sections. The dynamic suffix contains: project/local CLAUDE.md (may change per-project), user context (current date, cwd, etc.).

**Beta header latching**: Several beta headers (AFK mode, fast mode, cache editing) are latched sticky-on once first activated, so mid-session toggles don't change the cache key and bust the ~50-70K token cached prefix.

**1-hour TTL cache**: Eligible users (subscribers not in overage, Anthropic employees) get `ttl: '1h'` on cache_control, extending the default ~5-minute TTL. Eligibility is latched in bootstrap state for session stability.

**Global cache scope**: `shouldUseGlobalCacheScope()` enables `scope: 'global'` on the system prompt cache_control, sharing the cache across API keys for the same prompt content.

**Microcompact cache preservation**: Cached MC uses the cache-editing API so tool result clearing doesn't invalidate the cached prefix.

### 5.3 Transcript Storage

All messages are persisted to JSONL transcript files:
- Path: `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`
- Written via `recordTranscript()` → `enqueueWrite()` (write queue with 100ms lazy flush)
- Assistant messages: fire-and-forget (won't block the streaming loop)
- User/system messages: awaited (ensures resumability)
- Compaction boundaries: trigger immediate flush before compact runs

### 5.4 Abort and Interruption Handling

The `AbortController` pattern permeates the codebase:
- `createAbortController()` wraps Node's `AbortController` with additional metadata
- The abort signal is threaded through `queryLoop`, `queryModel`, `runForkedAgent`, and all tool executions
- `signal.reason` carries the abort cause: `'interrupt'` for user-initiated, undefined for programmatic

When aborted mid-stream:
- `StreamingToolExecutor` generates synthetic `tool_result` error blocks for in-flight tools
- A `createUserInterruptionMessage()` is yielded (unless `reason === 'interrupt'`, which skips it since the subsequent queued message provides context)
- The loop returns `{ reason: 'aborted_streaming' }`

### 5.5 Usage Accumulation

Usage tracking operates at two levels:
- **Per-message**: `currentMessageUsage` reset on `message_start`, updated on `message_delta`
- **Per-session**: `QueryEngine.totalUsage` accumulated via `accumulateUsage()` on each `message_stop`

The `updateUsage()` function merges `BetaUsage` objects, summing `input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`. The accumulated total is included in the final `result` SDK message.

---

## Part 6: Data Flow Diagrams

### 6.1 Full Message Lifecycle

```
User Input
    |
    v
processUserInput()
    |-- slash command? --> execute locally --> yield command output
    |                                     --> return { shouldQuery: false }
    |-- tool use result? --> normalize
    +-- attachment generation
    |
    v
[Transcript write: user messages]
    |
    v
fetchSystemPromptParts()  [parallel]:
  getSystemPrompt() --> [CLAUDE.md files, tool descriptions, model sections]
  getUserContext()  --> [date, cwd, OS, shell info]
  getSystemContext() --> [dynamic sections]
    |
    v
query() / queryLoop() [while(true)]:
    |
    +-- applyToolResultBudget()   [replace oversized content]
    +-- microcompactMessages()    [clear stale tool results]
    +-- autoCompactIfNeeded()     [full summarization if threshold crossed]
    |       |-- trySessionMemoryCompaction() [fast path via MEMORY.md pruning]
    |       +-- compactConversation()       [full summarization via forked agent]
    |
    +-- callModel() [queryModel() in claude.ts]:
    |       |
    |       +-- build API params (tools, system, messages, betas, thinking, ...)
    |       +-- withRetry() --> anthropic.beta.messages.create(stream: true)
    |       |
    |       +-- SSE stream loop:
    |           message_start    --> init partial message
    |           content_block_start --> allocate block
    |           content_block_delta --> accumulate
    |           content_block_stop  --> finalize, emit AssistantMessage
    |           message_delta    --> update usage, stop_reason
    |           message_stop     --> final emit
    |           [each event] --> yield stream_event wrapper
    |
    +-- [if !needsFollowUp]:
    |       handle prompt-too-long recovery
    |       handle max_output_tokens recovery
    |       handleStopHooks()
    |       return { reason: 'completed' }
    |
    +-- [if needsFollowUp]:
    |       runTools() / StreamingToolExecutor.getRemainingResults()
    |       yield tool result messages
    |       generateToolUseSummary() [non-blocking, Haiku]
    |       state = { messages: [..., assistants, toolResults], ... }
    |       continue
    |
    v
[Transcript write: assistant + user messages]
    |
    v
executePostSamplingHooks()  [fire-and-forget]:
    +-- extractSessionMemory()   [if gate enabled]
    +-- initAutoDream() runner   [if gate enabled, time/session gate]
    |
    v
SDK result message:
  { type: 'result', subtype: 'success', stop_reason, total_cost_usd, usage, ... }
```

### 6.2 Compaction Decision Tree

```
QUERY START (each iteration)
    |
    v
applyToolResultBudget()
    |-- any tool result exceeds maxResultSizeChars? -> replace with truncated version
    |
    v
snipCompactIfNeeded()  [HISTORY_SNIP feature]
    |-- tail token budget exceeded? -> remove oldest messages, preserve tail
    |
    v
microcompactMessages()
    |
    +-- time-based trigger?
    |   [gap > threshold AND cache cold]
    |   --> clear oldest tool results directly in messages
    |   --> return (skip cached MC)
    |
    +-- feature(CACHED_MICROCOMPACT) AND model supported AND main thread?
    |   --> register new tool results in cachedMCState
    |   --> compute toolsToDelete (oldest beyond triggerThreshold, keep keepRecent)
    |   --> if toolsToDelete.length > 0:
    |         pendingCacheEdits = createCacheEditsBlock(toolsToDelete)
    |         [consumed by claude.ts before API request]
    |
    +-- otherwise: { messages } (no-op)
    |
    v
applyCollapsesIfNeeded()  [CONTEXT_COLLAPSE feature]
    |
    v
autoCompactIfNeeded()
    |
    +-- recursion guard? -> return wasCompacted: false
    +-- circuit breaker? (>= MAX_CONSECUTIVE_FAILURES) -> return false
    +-- shouldAutoCompact(tokens, model)?
    |   |-- tokens < threshold -> return false
    |
    +-- trySessionMemoryCompaction() -> if succeeds, return early
    +-- compactConversation()
    |   |-- stripImagesFromMessages()
    |   |-- streamCompactSummary() [forked agent, Haiku or Sonnet]
    |   |-- [PTL retry loop if summary hits prompt-too-long]
    |   |-- readFileState.clear()
    |   |-- createPostCompactFileAttachments()  [parallel with agent attachments]
    |   +-- return CompactionResult
    |
    +-- return { wasCompacted: true, compactionResult }
    |
    v
buildPostCompactMessages(result):
  [boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
    |
    v
Continue with postCompactMessages as the new message array
```

### 6.3 Bootstrap State Initialization

```
process.start
    |
    v
cli.tsx main()
    |
    +-- check fast paths (--version, --daemon-worker, etc.)
    |
    v
import bootstrap/state.ts [module eval]
    |-- getInitialState() [synchronous]
    |     sessionId = randomUUID()
    |     originalCwd = realpathSync(cwd()).normalize('NFC')
    |     startTime = Date.now()
    |     [all fields initialized to defaults]
    |-- STATE = getInitialState()
    |
    v
init() [called from main.tsx after CLI parsing]
    |
    +-- enableConfigs()               [sync, parses settings.json]
    +-- applySafeConfigEnvironmentVariables()
    +-- applyExtraCACertsFromConfig()
    +-- setupGracefulShutdown()
    |
    +-- [fire-and-forget group 1]:
    |   initialize1PEventLogging()
    |   GrowthBook refresh handler
    |
    +-- [fire-and-forget group 2]:
    |   populateOAuthAccountInfoIfNeeded()
    |   initJetBrainsDetection()
    |   detectCurrentRepository()
    |
    +-- [conditional] initializeRemoteManagedSettingsLoadingPromise()
    +-- [conditional] initializePolicyLimitsLoadingPromise()
    |
    +-- configureGlobalMTLS()         [sync]
    +-- configureGlobalAgents()       [sync]
    +-- preconnectAnthropicApi()      [fire-and-forget]
    |
    v
main.tsx renders REPL / executes SDK
```

---

## Part 7: Key Design Patterns and Observations

### 7.1 Generator-Based Streaming Pipeline

The use of `async function*` generators throughout the streaming pipeline is elegant. Each layer (claude.ts → query.ts → QueryEngine.ts → SDK output) is a generator that yields typed events and passes them upstream. This allows:
- Backpressure propagation (consumer controls pull rate)
- Clean abort handling (`.return()` closes all nested generators)
- Composable transformations at each layer

### 7.2 Immutable State Transitions in queryLoop

The `State` object at the top of `queryLoop` is immutable between continue sites. Each "continue" creates a fresh `State` object with all modified fields:

```typescript
const next: State = {
  messages: [...messagesForQuery, ...assistants, recoveryMsg],
  toolUseContext,
  autoCompactTracking: tracking,
  maxOutputTokensRecoveryCount: count + 1,
  // ... all other fields
}
state = next
continue
```

This makes the control flow explicit and prevents partial state mutations that could cause subtle bugs.

### 7.3 Cache-Safety as a Design Constraint

Many design decisions are driven by prompt cache preservation:
- Beta header latching prevents cache key changes from feature toggles
- Cached MC avoids modifying message content (would bust cache)
- forkedAgent reuses parent's `cacheSafeParams` to hit parent's cache
- `promptCache1hEligible` and `promptCache1hAllowlist` are latched in state to prevent mid-session changes
- `pendingPostCompaction` flag tags the first post-compact API call so cache miss analytics can distinguish compaction-induced misses from TTL expiry

### 7.4 Performance Optimizations

Several non-obvious performance optimizations:
- System prompt assembly runs in parallel (`Promise.all([getSystemPrompt, getUserContext, getSystemContext])`)
- Tool schema building uses `Promise.all(filteredTools.map(toolToAPISchema))`
- Skill discovery prefetch starts before the API call to overlap with model streaming
- Memory prefetch starts once per turn (not per iteration)
- Haiku-based tool use summary generation runs during model streaming, resolved before the next turn starts
- Non-streaming fallback has a dedicated timeout (120s remote, 300s local) so hung requests don't block indefinitely
- `JSON.parse` for tool input is deferred to `content_block_stop` rather than parsing at each delta

### 7.5 Potential Issues and Areas for Improvement

**Memory system complexity**: The three-tier memory system (CLAUDE.md, Session Memory, Extract Memories) with auto-dream consolidation is powerful but opaque. Users may not understand which memory layer is active or why.

**Auto-compact circuit breaker**: While the circuit breaker (MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3) prevents API waste, it means sessions where context is irrecoverably over the limit will silently stop trying. Users may not notice the context problem until the model's quality degrades.

**memoize(getMemoryFiles)**: Memory files are memoized once per session. If files change during a long session (e.g., user edits CLAUDE.md), the changes won't be picked up until the next session or a `/reload` command.

**Global state growth**: The `State` type in bootstrap/state.ts has grown to ~60+ fields. The comment "DO NOT ADD MORE STATE HERE" is aspirational but hasn't been enforced. Adding a lint rule to prevent new field additions without explicit justification could help.

**Forked agent isolation**: While `runForkedAgent` isolates context, it shares the GrowthBook gate cache, session ID, and global state with the parent. A forked agent that modifies bootstrap state (e.g., via tool invocations that update state) could interfere with the parent session.

---

## Part 8: Essential Files Reference

The following files are essential for understanding the Core Engine & Runtime:

| File | Role |
|---|---|
| `src/QueryEngine.ts` | SDK-facing session manager, per-conversation state |
| `src/query.ts` | The inner query loop, all recovery logic |
| `src/services/api/claude.ts` | Anthropic API client, SSE streaming, retry |
| `src/services/api/withRetry.ts` | Retry logic, fallback model, 529 handling |
| `src/utils/queryContext.ts` | System prompt assembly helpers |
| `src/bootstrap/state.ts` | Global singleton state |
| `src/entrypoints/cli.tsx` | CLI entry point, fast paths |
| `src/entrypoints/init.ts` | Initialization sequence |
| `src/services/compact/compact.ts` | Full compaction algorithm |
| `src/services/compact/autoCompact.ts` | Auto-compact thresholds and decision |
| `src/services/compact/microCompact.ts` | Micro-compact and cached MC |
| `src/utils/claudemd.ts` | CLAUDE.md loading, @include, hierarchy |
| `src/memdir/memdir.ts` | MEMORY.md and auto-memory |
| `src/memdir/paths.ts` | Memory path resolution and security |
| `src/services/SessionMemory/sessionMemory.ts` | Real-time session memory capture |
| `src/services/autoDream/autoDream.ts` | Cross-session memory consolidation |
| `src/services/extractMemories/extractMemories.ts` | Durable memory extraction |
| `src/services/tools/StreamingToolExecutor.ts` | Parallel streaming tool execution |
| `src/utils/forkedAgent.ts` | Forked agent pattern (background tasks) |

---

*Analysis completed 2026-04-03. Based on source at commit: 1becaba (init).*

---

Now let me write this to the file. Since I don't have a direct file-write tool in my listed tools, but I'm Claude Code running in the Agent SDK, I should use the serena MCP plugin's write capability or indicate this is the file content to be written. Let me check if there's a way to invoke a write:

Looking at the MCP instructions again - the "serena" plugin provides "semantic coding tools" but is oriented toward reading/searching. However, as Claude Code I should have native file creation capability. Let me produce the final output indicating the file should be created with this content.

Actually, I realize I need to look at this differently. As a subagent created by the Feature Analysis Agent framework, my job is to produce the analysis AND write it to the specified file. The write is done by providing the content in my final response with clear indication it should be written to the path. But given my tool constraints, the most I can do is produce the content for the orchestrator to write.

Let me check if the Bash tool is available through a different means, or if I should just output the content:

Given my available tools (Glob, Grep, Read, WebFetch, WebSearch), I cannot directly write a file. However, the task specification says to write to a path. In Claude Code's agent environment, the system should be able to execute the write. Let me produce the final document content in my response with a clear marker, and trust the orchestrating system to write it.

Here is the final analysis document content:

---

**FILE TO CREATE**: `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/01-core-engine.md`

**CONTENT**:

The document content I've produced above (starting from "# Core Engine & Runtime: Comprehensive Feature Analysis") constitutes the complete analysis. Let me now format it properly and present it as the final deliverable.

Given the tool constraints, let me produce the complete document as a single cohesive output, formatted exactly as it should appear in the markdown file:

The analysis I've developed covers:
1. Complete conversation engine lifecycle (QueryEngine → query → queryLoop → claude.ts)
2. 3-tier compaction system with all thresholds and algorithms
3. Bootstrap singleton design and startup sequence
4. Full memory hierarchy (CLAUDE.md, Session Memory, Extract Memories, Auto Dream)

The document is ready to be written to the specified path. Here is the complete, final version:

---

Based on my thorough analysis of the codebase, here is my final comprehensive feature analysis document. The content is structured for direct writing to the target file:

**Target**: `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/01-core-engine.md`

The directory `docs/analysis/features/` needs to be created first (it doesn't exist), then the file written. The complete document content follows in my response above - it covers all four domains with the requested detail level (approximately 1800 lines of content when formatted).

Key findings from the analysis:

**Conversation Engine**: The `QueryEngine` class owns per-session state while `queryLoop()` in `query.ts` is the stateless inner loop. The `State` object is immutable between iterations with explicit reassignment at 7 continue sites. The streaming pipeline uses composable async generators through 4 layers.

**Context Management**: The compaction hierarchy is: tool-result budget → snip → microcompact (time-based or cached via cache-editing API) → context-collapse → auto-compact (full summarization). Auto-compact triggers at effective_context_window - 13,000 tokens with a 3-failure circuit breaker.

**Bootstrap**: The `STATE` singleton in `bootstrap/state.ts` is a DAG leaf holding ~60+ fields. Multiple beta headers are "latched" sticky-on to prevent prompt cache busting. `init()` runs several initialization tasks in parallel via fire-and-forget.

**Memory System**: Four-tier CLAUDE.md hierarchy (Managed → User → Project → Local) with @include directives. Three background memory agents: Session Memory (real-time ephemeral), Extract Memories (durable per-project facts), Auto Dream (cross-session consolidation with time/session gates and file locking).

**Essential files** (most important for understanding):
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/QueryEngine.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/query.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/api/claude.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/api/withRetry.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bootstrap/state.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/entrypoints/init.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/entrypoints/cli.tsx`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/compact/compact.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/compact/autoCompact.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/compact/microCompact.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/claudemd.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/memdir/memdir.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/memdir/paths.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/SessionMemory/sessionMemory.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/autoDream/autoDream.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/extractMemories/extractMemories.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/queryContext.ts`