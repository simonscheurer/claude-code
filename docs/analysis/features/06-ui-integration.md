---

# Feature Analysis: UI, Bridge & Integrations

> **Document version:** April 2026
> **Codebase commit:** `1becaba` (init)
> **Author:** Feature Analysis Agent (automated analysis)
> **Target file:** `docs/analysis/features/06-ui-integration.md`

---

## Table of Contents

1. [Domain Overview](#1-domain-overview)
2. [UI/UX Terminal System (Custom Ink)](#2-uiux-terminal-system-custom-ink)
   - 2.1 [Custom React Reconciler](#21-custom-react-reconciler)
   - 2.2 [Yoga Flexbox Layout Engine](#22-yoga-flexbox-layout-engine)
   - 2.3 [Rendering Pipeline](#23-rendering-pipeline)
   - 2.4 [ScrollBox and Virtual Message List](#24-scrollbox-and-virtual-message-list)
   - 2.5 [Event System and Input Handling](#25-event-system-and-input-handling)
   - 2.6 [Vim Mode Input](#26-vim-mode-input)
   - 2.7 [Theme System](#27-theme-system)
   - 2.8 [Component Catalog and Dialog System](#28-component-catalog-and-dialog-system)
   - 2.9 [App Component Tree](#29-app-component-tree)
3. [Remote Bridge and Desktop Integration](#3-remote-bridge-and-desktop-integration)
   - 3.1 [Bridge Architecture Overview](#31-bridge-architecture-overview)
   - 3.2 [Transport Layer](#32-transport-layer)
   - 3.3 [Bridge Poll Loop (bridgeMain)](#33-bridge-poll-loop-bridgemain)
   - 3.4 [Session Spawner](#34-session-spawner)
   - 3.5 [FlushGate and Message Ordering](#35-flushgate-and-message-ordering)
   - 3.6 [CapacityWake System](#36-capacitywake-system)
   - 3.7 [REPL Bridge (replBridge)](#37-repl-bridge-replbridge)
   - 3.8 [Desktop Handoff Flow](#38-desktop-handoff-flow)
4. [IDE Integration](#4-ide-integration)
   - 4.1 [IDE Auto-Detection](#41-ide-auto-detection)
   - 4.2 [IDE Lockfile Protocol](#42-ide-lockfile-protocol)
   - 4.3 [File Selection Flow](#43-file-selection-flow)
   - 4.4 [VS Code SDK MCP Bridge](#44-vs-code-sdk-mcp-bridge)
   - 4.5 [JetBrains-Specific Features](#45-jetbrains-specific-features)
5. [LSP and Code Intelligence](#5-lsp-and-code-intelligence)
   - 5.1 [LSP Architecture](#51-lsp-architecture)
   - 5.2 [LSPClient — Stdio Transport](#52-lspclient--stdio-transport)
   - 5.3 [LSPServerManager — Routing and Lifecycle](#53-lspservermanager--routing-and-lifecycle)
   - 5.4 [LSPTool — Claude-Facing Operations](#54-lsptool--claude-facing-operations)
   - 5.5 [Passive Diagnostic Feedback](#55-passive-diagnostic-feedback)
6. [Web Fetch and Search](#6-web-fetch-and-search)
   - 6.1 [WebFetchTool Pipeline](#61-webfetchtool-pipeline)
   - 6.2 [WebSearchTool Pipeline](#62-websearchtool-pipeline)
7. [Voice Input](#7-voice-input)
8. [Computer Use (macOS Control Layer)](#8-computer-use-macos-control-layer)
9. [Auto-Update System](#9-auto-update-system)
10. [Buddy / Companion System](#10-buddy--companion-system)
11. [GitHub App CI/CD Setup Wizard](#11-github-app-cicd-setup-wizard)
12. [Cross-Cutting Concerns](#12-cross-cutting-concerns)
13. [Essential Files Reference](#13-essential-files-reference)

---

## 1. Domain Overview

This document covers the UI, bridging, and integration domains of Claude Code CLI. These six areas share a common design philosophy: every integration is mediated through an abstraction layer (MCP for IDEs, Transport for the bridge, React Reconciler for the UI), so that the core agentic loop remains decoupled from the specific technology on each side.

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code CLI Process                   │
│                                                              │
│  ┌──────────────┐   ┌────────────────┐   ┌───────────────┐  │
│  │  Ink TUI     │   │  Bridge Layer  │   │  IDE Layer    │  │
│  │  (React +    │   │  (CCR remote   │   │  (MCP +       │  │
│  │  Yoga +      │   │  control)      │   │  Lockfiles)   │  │
│  │  ANSI term)  │   │                │   │               │  │
│  └──────┬───────┘   └───────┬────────┘   └───────┬───────┘  │
│         │                   │                     │          │
│  ┌──────▼───────────────────▼─────────────────────▼───────┐  │
│  │                    Core Agentic Loop                    │  │
│  │          (Agent Query → Tool Calls → Response)          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────┐   ┌────────────────┐   ┌───────────────┐  │
│  │  LSP Layer   │   │  Web Tools     │   │  Computer Use │  │
│  │  (multi-lang │   │  (Fetch +      │   │  (Swift +     │  │
│  │  jsonrpc)    │   │  Search)       │   │  Rust native) │  │
│  └──────────────┘   └────────────────┘   └───────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. UI/UX Terminal System (Custom Ink)

Claude Code ships a heavily modified fork of the [Ink](https://github.com/vadimdemedes/ink) terminal UI library. The core React rendering pipeline has been replaced, extended, and optimized far beyond the upstream.

### 2.1 Custom React Reconciler

**Entry point:** `src/ink/reconciler.ts`

The reconciler is built using `react-reconciler` (React 19 API). It bridges React's virtual DOM to an internal DOM of Ink nodes (`DOMElement`, `TextNode`).

**Host element types** (defined in `src/ink/dom.ts:19-27`):
- `ink-root` — single document root
- `ink-box` — layout container (maps to a Yoga node)
- `ink-text` — text container (has a Yoga measure function)
- `ink-virtual-text` — nested text within `ink-text` (no Yoga node)
- `ink-link` — hyperlink wrapper (no Yoga node, uses OSC 8)
- `ink-progress` — progress indicator (no Yoga node)
- `ink-raw-ansi` — raw ANSI passthrough (no Yoga node)

**Key reconciler operations:**

`createInstance` (`reconciler.ts:331-359`): Called by React when a new JSX element needs a host node. Detects when `<Box>` is nested inside `<Text>` (forbidden) and throws. Text elements nested inside text become `ink-virtual-text` (no Yoga node — saves layout overhead). Populates `debugOwnerChain` (React component fiber chain) when `CLAUDE_CODE_DEBUG_REPAINTS` is set.

`commitUpdate` (`reconciler.ts:426-459`): Computes the diff between old and new props using the local `diff()` function (shallow key comparison). Only touched props are applied, which prevents redundant Yoga style calls and avoids marking layout dirty on a style-unchanged re-render.

`resetAfterCommit` (`reconciler.ts:247-315`): Fires after every React commit. Calls `rootNode.onComputeLayout()` (Yoga layout pass), then `rootNode.onRender()` (terminal repaint). In test mode it calls `onImmediateRender` synchronously. Contains optional commit log instrumentation when `CLAUDE_CODE_COMMIT_LOG` is set — logs commit timing, gap between commits, and Yoga cache hit metrics.

**Scroll profiling counters** (`reconciler.ts:200-222`): `_lastYogaMs`, `_lastCommitMs`, and `_commitStart` are module-level globals. `recordYogaMs(ms)` is called from `ink.tsx`'s `onComputeLayout`. These are read by a bench/scroll-e2e.sh end-to-end scroll profiler.

**Update priority integration** (`reconciler.ts:508-511`): The `dispatcher` module's `discreteUpdates` is wired to `reconciler.discreteUpdates` after construction to break a circular import: dispatcher.ts does not import reconciler.ts, only holds a slot that reconciler.ts fills.

```
React commit cycle:
  prepareForCommit()
    ↓
  [React mutates fiber tree]
    ↓
  resetAfterCommit(rootNode)
    ├─ rootNode.onComputeLayout() → Yoga.calculateLayout()
    └─ rootNode.onRender() → scheduleRender() → onRender() (throttled)
```

### 2.2 Yoga Flexbox Layout Engine

**Entry points:** `src/ink/layout/engine.ts`, `src/ink/layout/yoga.ts`, `src/ink/layout/node.ts`

The layout engine is an abstraction layer over a pure-TypeScript port of Meta's [Yoga](https://github.com/nicolo-ribaudo/yoga-layout) flexbox engine (`src/native-ts/yoga-layout/`). The abstraction layer (`LayoutNode` interface in `node.ts`) means the Yoga binding can be swapped without touching the rest of the renderer.

**`YogaLayoutNode` class** (`yoga.ts:54-297`): Wraps a Yoga `Node` instance and proxies every style setter through typed enum maps. For example:

```
FlexDirection.Row / Column / RowReverse / ColumnReverse
Align.FlexStart / Center / FlexEnd / Stretch / Auto
Justify.FlexStart / Center / FlexEnd / SpaceBetween / SpaceAround / SpaceEvenly
```

All edge/gutter operations go through `EDGE_MAP` and `GUTTER_MAP` lookup tables rather than raw Yoga constants, providing a stable API surface.

**Text measure function** (`dom.ts`): Text nodes set a `measureFunc` on their Yoga node. The measure function calls `measureText()` which handles Unicode grapheme cluster widths and ANSI escape stripping. The measure result is cached (per `src/ink/line-width-cache.ts`) to avoid re-measuring unchanged text strings during layout passes.

**Layout calculation** (`ink.tsx:239-258`): `onComputeLayout` is called by the reconciler's `resetAfterCommit`. It sets the root width to `terminalColumns`, calls `calculateLayout(terminalColumns)`, and records the elapsed time. Yoga runs LTR (Left-to-Right) layout. The `YogaLayoutNode.calculateLayout` passes `undefined` for height so the root is unconstrained vertically.

**Node lifecycle** (`reconciler.ts:95-104`): When a node is removed, `cleanupYogaNode()` calls `yogaNode.unsetMeasureFunc()`, clears all references, then `yogaNode.freeRecursive()`. The clear-before-free ordering is critical: concurrent rendering could access freed WASM memory if a dangling reference remains.

### 2.3 Rendering Pipeline

**Entry point:** `src/ink/renderer.ts`, `src/ink/ink.tsx`

The rendering pipeline converts the Yoga-computed DOM tree into a terminal screen buffer and emits ANSI escape sequences.

```
React Commit
    │
    ▼
onComputeLayout()        ← Yoga: DOMElement tree → computed widths/heights
    │
    ▼
scheduleRender()         ← throttled (FRAME_INTERVAL_MS = ~16ms for 60fps)
    │
    ▼
onRender() [ink.tsx]
    ├─ createRenderer(rootNode, stylePool) → renders DOM to Output/Screen
    │      └─ renderNodeToOutput()  ← walks DOMElement tree recursively
    │              ├─ node cache blit (unchanged cells fast-path)
    │              ├─ scroll viewport culling
    │              ├─ renderBorder()
    │              └─ squashTextNodes() → ANSI-colored text segments
    │
    ├─ applySelectionOverlay()    ← alt-screen mouse text selection
    ├─ applySearchHighlight()     ← in-buffer search highlight
    │
    └─ writeDiffToTerminal()      ← emit only changed ANSI cells
            └─ LogUpdate          ← cursor management + scrollback
```

**Double-buffering** (`ink.tsx:101-102`): `frontFrame` and `backFrame` are two `Frame` objects (containing `Screen` + `Viewport` + `Cursor`). The renderer writes to `backFrame.screen`, then `writeDiffToTerminal` computes the diff between `frontFrame.screen` and `backFrame.screen` and emits only the changed cells as ANSI sequences. After emission, the frames swap roles.

**Dirty flag and blit optimization** (`render-node-to-output.ts`): Each `DOMElement` has a `dirty: boolean` flag. On a steady-state frame (spinner tick, streaming text append into a fixed-height box), only the nodes that changed are dirty. `renderNodeToOutput` uses a node cache (`node-cache.ts`) to blit pixel-perfect copies of unchanged subtrees from `prevScreen` into the new output in O(1) instead of re-walking the entire subtree. This is the primary performance optimization for the message list during streaming.

**Alt-screen mode**: The `<AlternateScreen>` component activates a separate terminal screen buffer (DECSET 1049). In alt-screen mode, the renderer has additional invariants: it clamps `cursor.y` to `terminalRows - 1` (preventing scroll-off-bottom LF), enforces a height cap of `terminalRows` (overflow clipped with a warning), and sets a viewport height of `terminalRows + 1` so the diff path's "content fills screen" check never fires.

**SIGCONT handling** (`ink.tsx:280-301`): On SIGCONT (shell resume after Ctrl+Z), the front frame is cleared to force a full redraw. In alt-screen mode, `reenterAltScreen()` re-enables mouse tracking, re-issues ENTER_ALT_SCREEN, and re-renders.

### 2.4 ScrollBox and Virtual Message List

**Entry point:** `src/ink/components/ScrollBox.tsx`

`ScrollBox` is a `<Box overflow="scroll">` with an imperative scroll API exposed via a `ref` handle.

**Scroll state lives on the DOM node** (not React state, `dom.ts:53-82`): `DOMElement` carries `scrollTop`, `pendingScrollDelta`, `scrollClampMin`, `scrollClampMax`, `scrollHeight`, `scrollViewportHeight`, `stickyScroll`. This design means scroll changes bypass React's scheduler entirely — `scrollTo` and `scrollBy` mutate the DOM node directly, call `markDirty()`, and trigger `scheduleRenderFrom()` (a microtask-deferred render). This gives sub-16ms scroll latency without React reconciliation overhead.

**Rate-limited scroll** (`render-node-to-output.ts`): `pendingScrollDelta` is drained at `SCROLL_MAX_PER_FRAME` rows per frame so fast scroll flicks produce visible intermediate frames rather than a single large jump. Direction reversal cancels accumulated delta naturally (pure accumulator).

**Scroll clamp for virtual scroll** (`dom.ts:65-70`): `scrollClampMin` and `scrollClampMax` bound the renderer's `scrollTop`. When `scrollTo(N)` races past React's async re-render of newly mounted virtual items, `scrollTop` is clamped to the currently-mounted content span. This prevents a blank screen while React catches up — instead of rendering into spacer rows (blank), the renderer holds at the content edge.

**Scroll anchor** (`dom.ts:80-83`): `scrollAnchor = { el: DOMElement, offset: number }` is a one-shot anchor. Rather than baking a position number that may be stale by render time, the renderer reads `el.yogaNode.getComputedTop()` freshly in the same Yoga pass that computed `scrollHeight`. This is used by `scrollToElement()` to scroll a specific DOM element into view without stale-position artifacts.

**DECSTBM hardware scroll optimization** (`render-node-to-output.ts:43-58`): When a `ScrollBox`'s `scrollTop` changes between frames and nothing else in the layout changed, a `ScrollHint` is recorded with the affected screen rows and scroll delta. `log-update.ts` reads this and emits a DECSTBM+SU/SD hardware scroll sequence instead of rewriting the entire viewport — O(1) terminal output for scroll.

**Sticky scroll** (`stickyScroll` attribute): When set, `scrollTop` auto-pins to the bottom as content grows. The reconciler's `onRender` path detects that `scrollTop === prevMax` and content grew, pinning `scrollTop = newMax`. Cleared by explicit `scrollTo()`/`scrollBy()` calls.

### 2.5 Event System and Input Handling

**Entry points:** `src/ink/events/`, `src/ink/parse-keypress.ts`

The event system models browser-style event dispatch with a `capture → bubble` flow, using custom event classes rather than browser `Event`.

**Event classes:**
- `KeyboardEvent` — keyboard input (key name, modifiers, raw text)
- `InputEvent` — raw character input from stdin
- `ClickEvent` — mouse click (x, y, button, inside/outside target)
- `FocusEvent` — focus gain/loss
- `TerminalFocusEvent` — terminal window focus (FocusIn/FocusOut)

**Key parsing** (`parse-keypress.ts`): The terminal input stream is tokenized by `src/ink/termio/tokenize.ts` (a hand-written DFA that recognizes escape sequences). After tokenization, `parseKeypress()` maps sequences to symbolic key names using four regex patterns:
- `META_KEY_CODE_RE`: `ESC + [a-zA-Z0-9]` → Meta+key
- `FN_KEY_RE`: `ESC[O|N|[|[[` + parameter suffix → F1-F12, arrow keys, etc.
- `CSI_U_RE`: Kitty keyboard protocol (`ESC [ codepoint ; modifier u`)
- `MODIFY_OTHER_KEYS_RE`: xterm modifyOtherKeys (`ESC [ 27 ; modifier ; keycode ~`)

Terminfo-style probing (`terminal-querier.ts`) negotiates Kitty keyboard mode or `modifyOtherKeys=2` at startup. The negotiated mode determines which of the above regexes fire for modifier+key combinations.

**Focus management** (`src/ink/focus.ts`): `FocusManager` tracks a flat list of focusable nodes (those with `tabIndex >= 0`) and manages Tab/Shift+Tab cycling. `autoFocus` on a node triggers `getFocusManager(node).handleAutoFocus(node)` during `reconciler.commitMount`. Removal of a focused node moves focus to the next available node.

**Mouse tracking** (`ink.tsx`): Alt-screen mode enables DECSET 1000 (button events) + DECSET 1003 (motion events). Mouse events are decoded from CSI M/CSI < sequences and dispatched through `dispatchClick()` and `dispatchHover()` which use a hit-test tree walk to find the deepest matching Box.

**`useInput` hook** (`src/ink/hooks/use-input.ts`): Uses `useLayoutEffect` (not `useEffect`) to enable raw mode synchronously during React's commit phase. This ensures raw mode is active before the first render returns — using `useEffect` would leave the terminal in cooked mode for one event-loop tick.

### 2.6 Vim Mode Input

**Entry points:** `src/hooks/useVimInput.ts`, `src/vim/`

Vim mode is a full two-mode state machine (INSERT / NORMAL) implemented as a pure TypeScript state machine on top of `useTextInput`.

**State machine** (`src/vim/types.ts`): The `VimState` discriminated union has two branches:
- `{ mode: 'INSERT', insertedText: string }` — tracks text typed for dot-repeat
- `{ mode: 'NORMAL', command: CommandState }` — multi-state command parser

`CommandState` is itself a discriminated union covering: `idle`, `count`, `operator`, `operatorCount`, `operatorFind`, `operatorTextObj`, `find`, `g`, `operatorG`, `replace`, `indent`.

**Transition function** (`src/vim/transitions.ts`): `transition(state, key, context)` is a pure function that takes the current state and a keypress and returns the next state plus any actions. This functional design means the state machine is trivially testable without React.

**Operations** (`src/vim/operators.ts`, `src/vim/motions.ts`, `src/vim/textObjects.ts`): Implement standard vim operations:
- Motions: `w`, `b`, `e`, `0`, `$`, `^`, `gg`, `G`, `f`/`F`/`t`/`T` (find char)
- Operators: `d` (delete), `c` (change), `y` (yank), `x` (cut char), `r` (replace), `>/<` (indent)
- Text objects: `iw`, `aw`, `i"`, `a"`, `i(`, `a(`, etc.
- Dot-repeat via `PersistentState.lastChange` recording

**Register and clipboard**: The vim register is stored in `PersistentState.register`. Yank operations populate it; paste (`p`/`P`) reads from it. Clipboard integration (system clipboard) bridges through the terminal's OSC 52 sequence.

**Mode indicator**: `useVimInput` calls `onModeChange(mode)` on every mode transition. The REPL's StatusLine component subscribes to this to show `-- INSERT --` / `-- NORMAL --` in the footer.

### 2.7 Theme System

**Entry points:** `src/utils/theme.ts`, `src/utils/systemTheme.ts`, `src/components/design-system/ThemeProvider.tsx`

**Theme variants** (6 total):
- `dark` (default) — full RGB palette
- `light` — full RGB palette, inverted luminance
- `dark-daltonized` — deuteranopia-friendly color substitutions
- `light-daltonized` — light + deuteranopia
- `dark-ansi` — 16-color fallback (for terminals without true-color)
- `light-ansi` — 16-color + light

**Theme resolution for `'auto'`** (`systemTheme.ts:42-47`): The `'auto'` setting resolves to `'dark'` or `'light'` based on the terminal's background luminance, not the OS dark mode. This correctly handles the case of a dark terminal on a light-mode macOS system.

**Dark/light detection pipeline** (`systemTheme.ts`):
1. **Synchronous seed**: Read `$COLORFGBG` (set by some terminals at launch). The convention is `fg;bg` where `bg=0` means dark. This seeds `cachedSystemTheme` immediately.
2. **Async OSC 11 query**: The `systemThemeWatcher.ts` sends an OSC 11 terminal background color query (`\x1b]11;?\x1b\\`). The terminal responds with an XParseColor string like `rgb:0000/0000/0000`.
3. **Luminance calculation** (`systemTheme.ts:61-66`): Parses the XParseColor response, computes ITU-R BT.709 relative luminance (`0.2126r + 0.7152g + 0.0722b`). Luminance > 0.5 → `'light'`, otherwise `'dark'`.
4. **Cache update**: `setCachedSystemTheme(theme)` updates the module-level cache so subsequent calls resolve synchronously.

**`ThemeProvider` component** (`design-system/ThemeProvider.tsx`): A React context provider that wraps the full component tree. It subscribes to the system theme watcher and re-renders with the new theme when the terminal appearance changes (e.g., user switches between dark/light mode in terminal settings while Claude Code is running).

**Color format**: Theme colors are strings in one of three formats:
- `rgb(R,G,B)` — 24-bit true color (converted to `\x1b[38;2;R;G;Bm`)
- `#RRGGBB` — hex (rare, mostly legacy)
- `ansi:name` — one of the 16 ANSI names (`red`, `blueBright`, etc.) for ANSI-variant themes

Apple Terminal workaround (`theme.ts:616-620`): Apple Terminal rejects `\x1b[38;2;…m` 24-bit sequences. When `env.terminal === 'Apple_Terminal'`, a Chalk instance with `level: 2` (256-color mode) is used instead, automatically downsampling RGB to the 256-color palette.

### 2.8 Component Catalog and Dialog System

**Location:** `src/components/`

All dialogs follow a common `Dialog` design-system component (`src/components/design-system/Dialog.tsx`) that provides a bordered overlay with a title bar.

**Permission/approval dialogs:**
- `BypassPermissionsModeDialog` (`BypassPermissionsModeDialog.tsx`) — confirmation for enabling bypass-permissions mode
- `AutoModeOptInDialog` (`AutoModeOptInDialog.tsx`) — opt-in for auto mode (legally reviewed copy, do not modify without Legal)
- `MCPServerApprovalDialog` (`MCPServerApprovalDialog.tsx`) — approve/deny MCP server connections with persist options
- `MCPServerDesktopImportDialog` — import MCP servers from Desktop app
- `MCPServerMultiselectDialog` — multiselect for enabling multiple MCP servers at once

**Session management dialogs:**
- `BridgeDialog` (`BridgeDialog.tsx`) — shows the Remote Control connection status, QR code generator, and disconnect controls. Reads `connected`, `sessionActive`, `reconnecting`, `connectUrl`, `sessionUrl` from AppState. Generates QR codes via the `qrcode` npm package.
- `ExportDialog` (`ExportDialog.tsx`) — exports conversation to clipboard or file. Two-step: first selects `clipboard|file`, then (if file) prompts for filename with `TextInput`.
- `HistorySearchDialog` — search through conversation history
- `GlobalSearchDialog` — global file/content search
- `DesktopHandoff` (`DesktopHandoff.tsx`) — manages handoff to Claude Desktop app with states: `checking → prompt-download → flushing → opening → success → error`

**Onboarding/status dialogs:**
- `IdeOnboardingDialog`, `IdeAutoConnectDialog`, `IdeStatusIndicator` — IDE integration setup and status
- `CostThresholdDialog` — configure spending limits
- `InvalidConfigDialog`, `InvalidSettingsDialog` — surface config parse errors
- `ConsoleOAuthFlow` — OAuth flow for claude.ai authentication
- `ChannelDowngradeDialog` — notification when switching to stable release channel

**`CustomSelect` component** (`src/components/CustomSelect/`): A full-featured keyboard-navigable select input for the terminal. Supports single select (`select.tsx`) and multi-select (`SelectMulti.tsx`). Uses hooks `use-select-state.ts`, `use-select-navigation.ts`, `use-select-input.ts` to separate concerns.

### 2.9 App Component Tree

**Root component:** `src/components/App.tsx`

```
<App>
  <FpsMetricsProvider>        ← FPS metrics context for performance monitoring
    <StatsProvider>           ← Stats context (usage, token counts)
      <AppStateProvider>      ← Global AppState (bridge state, permission mode, etc.)
        <children />          ← Typically <REPL /> or <CommandScreen />
      </AppStateProvider>
    </StatsProvider>
  </FpsMetricsProvider>
</App>
```

**`AppStateProvider`** subscribes to `onChangeAppState` (a callback from `src/state/onChangeAppState.ts`) that reacts to AppState mutations (permission mode changes, bridge connect/disconnect, etc.) and triggers side effects outside React.

The **REPL screen** (`src/screens/REPL.tsx`) is the core interactive view. It renders the message list, prompt input, status line, and all dialogs using React's conditional rendering based on AppState flags.

---

## 3. Remote Bridge and Desktop Integration

The Remote Bridge (also called CCR — Claude Code Remote) enables running Claude Code sessions from the claude.ai web interface or the Claude Desktop app. The CLI process acts as a worker that polls an API for work, spawns child CLI processes to handle each session, and streams results back.

### 3.1 Bridge Architecture Overview

```
claude.ai Web UI
      │
      │  HTTP/WebSocket
      ▼
Session Ingress Service
      │
      │  Redis Streams / XREADGROUP
      ▼
Bridge Poll API (environments API)
      │
      │  GET /poll
      ▼
bridgeMain.ts (CLI Worker Process)
      │
      │  fork() via spawn()
      ▼
Child CLI Process (--print mode)
      │  stdin:  control messages (permission requests)
      │  stdout: NDJSON stream (messages, tool calls, results)
      │  stderr: error/diagnostic lines
      ▼
Session Ingress API
      │
      │  WebSocket (reads) + HTTP POST (writes) via HybridTransport
      ▼
claude.ai Web UI
```

Two bridge modes exist:

1. **`bridgeMain.ts`** — the multi-session server mode (`claude remote-control --spawn-sessions`). Polls for work items, spawns child `claude --print` processes for each work item, and manages their lifecycle (heartbeat, timeout, session archive).

2. **`replBridge.ts` / `initReplBridge.ts`** — single-session mode. The REPL process (the interactive terminal session) connects a bridge transport so the user's current session is accessible from claude.ai.

### 3.2 Transport Layer

**Location:** `src/cli/transports/`

Three transport classes, all implementing the `Transport` interface:

**`SSETransport`** (`SSETransport.ts`): HTTP Server-Sent Events for reading messages, HTTP POST for writing. Reconnects automatically with exponential backoff (base 1s, max 30s, give-up 10 min). Tracks sequence numbers (`from_sequence_num`) across reconnects so the server sends only new messages after a reconnect. `LIVENESS_TIMEOUT_MS=45s` kills dead connections (server sends keepalives every 15s).

**`WebSocketTransport`** (`WebSocketTransport.ts`): Bidirectional WebSocket. Sends `keep_alive` frames (`{"type":"keep_alive"}\n`) to prevent idle timeout. Buffers up to 1000 outbound messages during reconnect. Detects system sleep via time gap > `SLEEP_DETECTION_THRESHOLD_MS=60s` between reconnect attempts and resets the error budget. Permanent close codes 1002, 4001, 4003 (protocol error, session expired, unauthorized) skip retry and go straight to `'closed'`.

**`HybridTransport`** (`HybridTransport.ts`): WebSocket for reads + HTTP POST for writes. Extends `WebSocketTransport` with an additional `SerialBatchEventUploader` for the write path.

```
HybridTransport write flow:

  transport.write(stream_event)  ─┐
                                  │ (100ms accumulation window)
  transport.write(other_event)  ──┤
                                  ▼
  SerialBatchEventUploader.enqueue(batch)
                                  │
                                  │ (serialized: at most 1 POST in-flight)
                                  ▼
  postOnce(batch)  →  POST /v1/sessions/{id}/messages
                       (exponential backoff on failure: 500ms base, 8s max)
```

The 100ms `stream_event` accumulation buffer (`BATCH_FLUSH_INTERVAL_MS`) reduces POST count for high-volume content deltas (streaming token output). A non-stream write flushes any buffered stream events first to preserve ordering.

`SerialBatchEventUploader` serializes writes to prevent Firestore write-collisions: concurrent POSTs to the same document result in version conflicts and retry storms. By ensuring at most one POST in-flight, write order is preserved and server-side retry logic is simple.

**`WorkerStateUploader`** (`WorkerStateUploader.ts`): Used by `bridgeMain.ts` to publish worker status (session count, activity) to the bridge API for display in the claude.ai session picker.

### 3.3 Bridge Poll Loop (bridgeMain)

**Entry point:** `src/bridge/bridgeMain.ts:141`

`runBridgeLoop(config, environmentId, environmentSecret, api, spawner, logger, signal)` is the main poll loop. It is designed to run indefinitely until the outer signal aborts (SIGTERM, Ctrl+C, or `--single-session` mode session completion).

**Poll loop state:**

| State variable | Purpose |
|---|---|
| `activeSessions: Map<sessionId, SessionHandle>` | Currently running sessions |
| `sessionStartTimes: Map<sessionId, number>` | For elapsed time display |
| `sessionWorkIds: Map<sessionId, string>` | Redis work item IDs for heartbeat/stop |
| `sessionCompatIds: Map<sessionId, string>` | Stable compat-surface IDs (`session_*`) |
| `sessionIngressTokens: Map<sessionId, string>` | Per-session JWTs for heartbeat auth |
| `sessionTimers: Map<sessionId, Timeout>` | Per-session timeout watchdogs |
| `completedWorkIds: Set<string>` | Dedup guard: prevents double-spawning re-delivered work |
| `v2Sessions: Set<string>` | CCR v2 sessions (different token refresh path) |
| `capacityWake: CapacityWake` | Wakes the at-capacity sleep when a session ends |

**Poll loop body** (line 600+):
```
while (!loopSignal.aborted):
  work = await api.pollForWork(environmentId, environmentSecret, signal)

  if work is null:
    if activeSessions.size >= config.maxSessions:
      if heartbeat_enabled:
        enter heartbeat-only loop (sleep hb_interval, break on capacity change)
    else:
      sleep(poll_interval_not_at_capacity)

  else if work.type === 'session':
    sessionId = work.data.id
    if completedWorkIds has sessionId: continue (dedup)
    handle = spawner.spawn(spawnOpts, dir)  ← creates child process
    activeSessions.set(sessionId, handle)
    startStatusUpdates()

  else if work.type === 'healthcheck':
    api.completeHealthcheck(workId)
```

**Backoff strategy**: Two separate backoff tracks — connection errors (`connBackoff`, `connErrorStart`) and general errors (`generalBackoff`, `generalErrorStart`). Each has initial/cap/give-up parameters (default: 2s initial, 2min cap, 10min give-up for connection; 500ms/30s/10min for general). Sleep detection: if the gap between consecutive attempts exceeds `2 × connCapMs = 4min`, the machine probably slept; the error budget resets.

**At-capacity heartbeat mode**: When `activeSessions.size >= config.maxSessions`, the poll loop enters a heartbeat-only sub-loop. At each heartbeat interval, `heartbeatActiveWorkItems()` PATCHes all active work items' leases. If a heartbeat gets 401/403, `reconnectSession()` triggers server re-dispatch. If the `at_capacity_poll_interval` deadline passes, the sub-loop breaks out to do a real poll (picking up any newly queued work).

**Status display**: A 1-second interval timer calls `updateStatusDisplay()` which pushes activity updates to the terminal UI via `BridgeLogger`.

### 3.4 Session Spawner

**Entry point:** `src/bridge/sessionRunner.ts:248`

`createSessionSpawner(deps)` returns a `SessionSpawner` with a `spawn(opts, dir)` method. It creates a child `claude --print` process.

**Child process CLI flags:**
```
claude --print
       --sdk-url <session_ingress_url>
       --session-id <id>
       --input-format stream-json
       --output-format stream-json
       --replay-user-messages
       [--verbose]
       [--debug-file <path>]
       [--permission-mode <mode>]
```

**Child process environment:**
- `CLAUDE_CODE_OAUTH_TOKEN` → stripped (child uses session access token, not OAuth)
- `CLAUDE_CODE_ENVIRONMENT_KIND=bridge`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN=<jwt>`
- `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1` (for HybridTransport)
- `CLAUDE_CODE_USE_CCR_V2=1` + `CLAUDE_CODE_WORKER_EPOCH=<n>` (CCR v2 path)

**Activity monitoring**: The spawner reads the child's stdout NDJSON stream via readline. `extractActivities(line)` parses each line and updates `handle.currentActivity`:
- `assistant` messages with `tool_use` blocks → `tool_start` activity (e.g. "Reading src/foo.ts")
- `assistant` messages with `text` blocks → text activity (first 80 chars)
- `result` messages → completion/error activity

**TOOL_VERBS map** (`sessionRunner.ts:70-89`): Maps tool names to display verbs for the status line: `Read → Reading`, `Bash → Running`, `Glob → Searching`, etc.

**Permission requests** (`PermissionRequest` type): When the child process encounters a tool that requires explicit permission, it emits a `control_request` JSON line on stdout with `type: 'can_use_tool'`. The spawner's readline handler detects this and calls `onPermissionRequest(sessionId, request, accessToken)`, which the bridge forwards to the API for remote approval.

### 3.5 FlushGate and Message Ordering

**Entry point:** `src/bridge/flushGate.ts`

`FlushGate<T>` is a simple state machine that prevents message interleaving during the initial history flush.

When a bridge session first connects, it must replay historical messages to the server via a single HTTP POST (the "flush"). During this flush, new messages from the active session must not arrive at the server out-of-order with the historical replay.

```
FlushGate lifecycle:

  start()     ← flush begins; gate is active
      │
      │  new messages arrive → enqueue() returns true → queued in _pending
      │
  end()       ← flush completes; returns _pending for drain; gate inactive
      │
      └─► caller drains the returned items immediately

  drop()      ← permanent close; discards _pending (no drain)
  deactivate() ← transport replacement; clears active but preserves _pending
                 (new transport will drain on its own flush)
```

This ensures the server sees: `[historical messages] [new messages]` in the correct temporal order, regardless of async timing.

### 3.6 CapacityWake System

**Entry point:** `src/bridge/capacityWake.ts`

`createCapacityWake(outerSignal)` returns a `CapacityWake` object with two methods:

- `signal()` — creates a merged `AbortSignal` that fires when either the outer loop signal or an internal wake controller fires. Returns `{ signal, cleanup }` where `cleanup()` removes event listeners after the sleep resolves normally.
- `wake()` — fires the current wake controller (aborting the sleep early) and creates a fresh controller for the next sleep.

This pattern is used in the at-capacity heartbeat loop to wake the sleeping bridge immediately when a session completes (`onSessionDone → capacityWake.wake()`), so the bridge can poll for new work without waiting for the full heartbeat interval.

**Why a custom primitive instead of `Promise.race`**: `Promise.race` would leak the losing promise. The `AbortSignal`-based approach is cleanup-safe and composable with existing signal-based cancellation patterns throughout the codebase.

### 3.7 REPL Bridge (replBridge)

**Entry point:** `src/bridge/replBridge.ts`, `src/bridge/initReplBridge.ts`

The REPL bridge connects the user's interactive terminal session to the claude.ai web UI. Unlike `bridgeMain.ts` which spawns child processes, `replBridge.ts` runs inside the already-running REPL process.

**`BridgeCoreParams`** (`replBridge.ts:91`): All parameters are injected rather than read from global state. This is a deliberate bootstrap isolation constraint: `replBridge.ts` must be usable from both the REPL (which has the full command registry, React tree, auth modules) and the Agent SDK daemon (which must not pull in those heavy dependencies). Injectable seams:
- `createSession` — either `createBridgeSession` (REPL, full OAuth) or `createBridgeSessionLean` (daemon, JWT-only)
- `archiveSession` — best-effort cleanup on session end
- `toSDKMessages` — converts `Message[]` to `SDKMessage[]` (not available in daemon)
- `onAuth401` — OAuth token refresh callback
- `getPollIntervalConfig` — GrowthBook-backed poll config (daemon uses static config)

**`ReplBridgeHandle`** (`replBridge.ts:70-81`): The return type provides methods to write messages and SDK messages from the REPL into the bridge transport:
- `writeMessages(messages: Message[])` — writes UI messages
- `writeSdkMessages(messages: SDKMessage[])` — writes SDK format messages
- `sendControlRequest(request: SDKControlRequest)` — sends permission prompts
- `sendControlResponse(response: SDKControlResponse)` — sends permission decisions
- `sendResult()` — signals conversation end
- `teardown()` — graceful close with archive

**Transport selection at runtime**: `replBridge.ts` constructs either `HybridTransport` (v1) or `SSETransport` + `CCRClient` (v2) depending on the `use_code_sessions` field in the work secret. Transport is constructed once per connection but can be replaced (`onWorkReceived`) when a new work item arrives with a different URL — the `FlushGate.deactivate()` pattern preserves pending messages across the swap.

**Poll config tuning**: The REPL bridge uses the GrowthBook-backed `getPollIntervalConfig()`. Ops can live-tune the poll interval (`tengu_bridge_poll_interval_config` flag) without a code deploy. The daemon uses a static 60s heartbeat.

### 3.8 Desktop Handoff Flow

**Entry point:** `src/components/DesktopHandoff.tsx`

The `/desktop` slash command triggers a handoff from the terminal session to the Claude Desktop app.

**State machine** (type `DesktopHandoffState`):
```
'checking'
    │
    ├─► 'prompt-download'  ← Desktop not installed / too old
    │       │ y
    │       └─► openBrowser(downloadUrl) → onDone()
    │
    ├─► 'flushing'         ← flush session storage (ensures Desktop sees full context)
    │       │
    │       └─► 'opening'  ← openCurrentSessionInDesktop() via deep link
    │               │
    │               ├─► 'success' → onDone()
    │               └─► 'error'   → onDone(errorMessage, { display: 'system' })
    │
    └─► 'error'            ← installStatus check threw
```

**Deep link format**: `claude://open-session?id={sessionId}` — handled by the Claude Desktop app's URL scheme registration. When the Desktop app receives this link, it opens the session identified by `sessionId` in its GUI.

**Flush before handoff** (`flushSessionStorage()`): Before opening Desktop, the terminal flushes session storage to disk so Desktop can read the full conversation history without waiting for the bridge's eventual sync.

---

## 4. IDE Integration

### 4.1 IDE Auto-Detection

**Entry point:** `src/utils/ide.ts:259-292`

Claude Code auto-detects the IDE it is running inside by cross-referencing:

1. **`$TERM_PROGRAM` / terminal env variables**: VS Code sets `TERM_PROGRAM=vscode`; JetBrains sets `TERMINAL_EMULATOR=JetBrains-JediTerm`. These are checked synchronously via `env.terminal`.

2. **Process tree inspection** (`getAncestorPidsAsync`): If the terminal env is not conclusive, the process's ancestor PID chain is walked (up to 10 levels). Each ancestor's name is compared against `processKeywordsMac/Windows/Linux` for each of the 17 supported IDE types.

3. **Lockfile discovery** (see §4.2): IDE plugins write lockfiles to `~/.claude/ide/` (or the platform-specific IDE data dir). If a lockfile is present and its listed PID is an ancestor of the current process, the IDE is confirmed.

**Supported IDE types** (defined in `supportedIdeConfigs`, `ide.ts:130`):
- VS Code variants (3): `cursor`, `windsurf`, `vscode`
- JetBrains variants (14): `intellij`, `pycharm`, `webstorm`, `phpstorm`, `rubymine`, `clion`, `goland`, `rider`, `datagrip`, `appcode`, `dataspell`, `aqua`, `gateway`, `fleet`, `androidstudio`

Each IDE config has:
- `ideKind: 'vscode' | 'jetbrains'` — determines MCP connection protocol
- `displayName: string` — shown in UI
- `processKeywords{Mac,Windows,Linux}: string[]` — process name substrings for auto-detection

Some JetBrains IDEs have empty `processKeywordsMac` lists (e.g., `aqua`, `gateway`, `fleet`) to prevent false positives since those names are too common.

**Memoized results**: `isSupportedVSCodeTerminal()`, `isSupportedJetBrainsTerminal()`, `isSupportedTerminal()` are all memoized with `lodash-es/memoize`. The memoization assumes the terminal type doesn't change during a process lifetime.

### 4.2 IDE Lockfile Protocol

**Location:** `src/utils/ide.ts:298-393`

IDE plugins write a lockfile to communicate their presence and connection parameters to Claude Code.

**Lockfile location**: `~/.claude/ide/{PORT}.lock` where PORT is the MCP server's HTTP port.

**Lockfile format** (JSON):
```json
{
  "workspaceFolders": ["/path/to/workspace"],
  "pid": 12345,
  "ideName": "VS Code",
  "transport": "ws",
  "runningInWindows": false,
  "authToken": "optional-bearer-token"
}
```

**Older format**: Just a newline-separated list of workspace folder paths (pre-JSON era, still supported via fallback parsing).

**Discovery process** (`getSortedIdeLockfiles`): Reads all `*.lock` files from the IDE lockfile directory, stats each for modification time, sorts newest-first. The newest lockfile is preferred when multiple IDEs are running.

**Validation**: For each lockfile candidate, Claude Code:
1. Checks if `pid` is still an ancestor (via `process.kill(pid, 0)` which throws if the PID doesn't exist)
2. Tests if the port is actually listening (`checkWSConnection(port)` for WebSocket or `checkPortReachable(port)` for HTTP)
3. Validates workspace folder overlap with the current working directory

**Transport negotiation**: `useWebSocket: true` (from `transport: 'ws'` in lockfile) uses WebSocket; otherwise HTTP SSE. The `HybridTransport` is used for reliability on write-heavy workloads.

**WSL path conversion** (`idePathConversion.ts`): When the IDE is `runningInWindows: true` but Claude Code is running in WSL, Windows-style paths (`C:\Users\...`) in `workspaceFolders` are converted to WSL mount paths (`/mnt/c/Users/...`). `checkWSLDistroMatch()` verifies the WSL distro name matches to prevent cross-distro path confusion.

### 4.3 File Selection Flow

**Entry points:** `src/hooks/useIdeSelection.ts`

The `useIdeSelection` hook subscribes to `selection_changed` MCP notifications from the connected IDE client.

**Flow:**
```
User selects text in IDE editor
      │
      ▼ (MCP notification)
IDE Plugin → MCP server → Claude Code process
      │
      ▼
client.setNotificationHandler(SelectionChangedSchema, handler)
      │
      ▼ (in handler)
selectionChangeHandler(data: SelectionData)
      │
      ▼
onSelect({ lineCount, lineStart, text, filePath })
      │
      ▼
React state update → prompt input shows "X lines selected in file.ts"
```

**Schema validation** (`SelectionChangedSchema`): The notification params are validated with Zod:
- `selection.start.line/character`, `selection.end.line/character` — 0-indexed line/char positions
- `text?: string` — selected text content (optional: may be omitted for large selections)
- `filePath?: string` — absolute path to the file

**Line count calculation** (`useIdeSelection.ts:93-100`): If `end.character === 0`, the last line is not counted (the cursor is at the start of the next line, meaning the previous line was the last selected line). This matches VS Code's selection model.

**IDE client lifecycle**: The hook stores `currentIDERef` to detect IDE client changes (e.g., reconnect to a new IDE instance). When the IDE changes, selection is reset to empty and handlers are re-registered.

### 4.4 VS Code SDK MCP Bridge

**Entry point:** `src/services/mcp/vscodeSdkMcp.ts`

`setupVscodeSdkMcp(sdkClients)` searches for a client named `'claude-vscode'` in the connected MCP servers and establishes a bidirectional notification channel.

**Inbound (VS Code → Claude Code)**: `log_event` notifications from VS Code are forwarded to Claude Code's analytics system via `logEvent(tengu_vscode_${eventName}, eventData)`.

**Outbound (Claude Code → VS Code)**: On connection, Claude Code immediately sends an `experiment_gates` notification with the current GrowthBook gate values that VS Code needs to know:
- `tengu_vscode_review_upsell` — boolean gate
- `tengu_vscode_onboarding` — boolean gate
- `tengu_quiet_fern` — feature value (browser support)
- `tengu_vscode_cc_auth` — feature value (in-band OAuth vs. extension PKCE)
- `tengu_auto_mode_state` — tri-state `'enabled'|'disabled'|'opt-in'` (omitted if unknown)

**File update notifications** (`notifyVscodeFileUpdated`): When Claude Code edits or writes a file (via `FileEditTool` or `FileWriteTool`), it notifies VS Code via a `file_updated` MCP notification with `{ filePath, oldContent, newContent }`. VS Code can use this to show a diff or refresh its file buffer. Gated on `USER_TYPE === 'ant'` (internal only for now).

**Module-level client reference**: `vscodeMcpClient` is a module-level singleton updated by `setupVscodeSdkMcp`. This is intentional — only one VS Code client is expected per process, and `notifyVscodeFileUpdated` is called from deep in the tool execution path where passing a reference would be cumbersome.

### 4.5 JetBrains-Specific Features

**Entry point:** `src/utils/jetbrains.ts` (referenced from `ide.ts`)

JetBrains IDE detection has an additional check: `isJetBrainsPluginInstalledCached()` queries the JetBrains plugin installation state. This is cached because the check involves filesystem access to the JetBrains plugin directory.

JetBrains uses a different MCP transport from VS Code: its plugin exposes an HTTP SSE server (not WebSocket-capable for older JB versions). The lockfile's `transport` field distinguishes: `"ws"` for WebSocket-capable plugins, absent/`"sse"` for SSE-only.

The JetBrains `IdeStatusIndicator` component (`src/components/IdeStatusIndicator.tsx`) shows a live connection status badge in the REPL's status line. It polls the IDE connection state via the MCP client's `state` property.

---

## 5. LSP and Code Intelligence

### 5.1 LSP Architecture

```
Claude (LSPTool call)
      │
      ▼
LSPTool.ts  (validateInput, permission check, file open)
      │
      ▼
LSPServerManager  (route by file extension, ensure server started)
      │
      ▼
LSPServerInstance  (state machine: stopped/starting/running/error)
      │
      ▼
LSPClient  (stdio transport, vscode-jsonrpc, JSON-RPC 2.0)
      │
      ▼
Language Server Process (e.g. typescript-language-server, rust-analyzer)
      │
      ▼ (publishDiagnostics notification)
passiveFeedback.ts → LSPDiagnosticRegistry → diagnostic attachments
```

### 5.2 LSPClient — Stdio Transport

**Entry point:** `src/services/lsp/LSPClient.ts`

`createLSPClient(serverName, onCrash?)` creates an LSP client that communicates with a language server process via stdio using the `vscode-jsonrpc` library.

**Spawn-then-validate pattern** (`LSPClient.ts:96-130`): `spawn()` is called, then the code awaits a `Promise` that resolves on `spawn` event and rejects on `error` event. This is critical: `spawn()` returns immediately, but `ENOENT` (command not found) fires asynchronously. Without this guard, writing to an invalid stream produces unhandled promise rejections.

**Lazy handler registration**: Notification and request handlers can be registered before the server starts (the client queues them in `pendingHandlers` and `pendingRequestHandlers`). When the connection is established, pending handlers are replayed onto the live connection.

**Crash detection** (`isStopping` flag): The `close` event on the connection distinguishes intentional stop (via `stop()`) from unexpected crash. If the server exits non-zero and `isStopping` is false, `onCrash` is called so the owner (`LSPServerInstance`) can mark the server as errored for lazy restart.

**State tracking**: The LSP connection goes through: `start()` → `initialize()` (sends `initialize` request, receives `InitializeResult.capabilities`) → operational. `capabilities` is stored for introspection (e.g., does this server support `callHierarchy/prepare`?).

### 5.3 LSPServerManager — Routing and Lifecycle

**Entry point:** `src/services/lsp/LSPServerManager.ts`

`createLSPServerManager()` creates a manager that routes LSP requests to the correct language server by file extension.

**Extension map construction** (`initialize()`): Loads all configured servers from `getAllLspServers()` (`config.ts`). For each server, builds an `extensionMap: Map<ext, serverName[]>` from the server's `extensionToLanguage` config (e.g., `.ts → ['typescript']`). This map supports multiple servers per extension (for future multi-server scenarios).

**Server routing** (`getServerForFile(filePath)`): Extracts the file extension, lowercases it, looks up the `extensionMap`, returns the first matching `LSPServerInstance`. Returns `undefined` if no server handles that extension.

**Lazy server start** (`ensureServerStarted(filePath)`): Calls `getServerForFile`, then `instance.start()` if the instance is in `'stopped'` state. Returns `undefined` (not an error) if no server handles the file extension — LSP is silently skipped for unsupported languages.

**File synchronization** (`openFile`, `changeFile`, `saveFile`, `closeFile`): Wraps the LSP `textDocument/didOpen`, `textDocument/didChange`, `textDocument/didSave`, `textDocument/didClose` notifications. Tracks open files in `openedFiles: Map<fileUri, serverName>` to prevent sending `didOpen` twice or `didClose` for a non-open file.

**LSP server instance state machine** (`LSPServerInstance.ts:75-82`):
```
stopped → starting → running
running → stopping → stopped
any → error  (on failure)
error → starting  (on retry via restart())
```

`restart()` calls `stop()` then `start()`. `isHealthy()` returns `true` only in `'running'` state.

**Transient error retry** (`LSPServerInstance.ts:17-22`): LSP error code `-32801` ("content modified") occurs when the server is still indexing (e.g., rust-analyzer). The manager retries these with exponential backoff: 500ms, 1s, 2s (max 3 retries).

### 5.4 LSPTool — Claude-Facing Operations

**Entry point:** `src/tools/LSPTool/LSPTool.ts`

`LSPTool` exposes 9 LSP operations to Claude:

| Operation | LSP Method | Use case |
|---|---|---|
| `goToDefinition` | `textDocument/definition` | Jump to symbol definition |
| `findReferences` | `textDocument/references` | Find all usages of a symbol |
| `hover` | `textDocument/hover` | Type info, documentation |
| `documentSymbol` | `textDocument/documentSymbol` | Outline of a file's symbols |
| `workspaceSymbol` | `workspace/symbol` | Search symbols across workspace |
| `goToImplementation` | `textDocument/implementation` | Find interface implementations |
| `prepareCallHierarchy` | `callHierarchy/prepareCallHierarchy` | Build call hierarchy root |
| `incomingCalls` | `callHierarchy/incomingCalls` | Who calls this function? |
| `outgoingCalls` | `callHierarchy/outgoingCalls` | What does this function call? |

**Input schema**: All operations take `filePath` (absolute or relative), `line` (1-based), `character` (1-based). The 1-based convention matches editor display; the tool converts to 0-based before the LSP call.

**Permission check**: `checkReadPermissionForTool()` validates the file is readable. For workspace-wide operations (`workspaceSymbol`), the file path is used only for routing to the correct server.

**File size limit** (`MAX_LSP_FILE_SIZE_BYTES = 10MB`): Large files are skipped to prevent memory pressure in the language server.

**File open/close lifecycle**: Before sending a request, the tool calls `manager.openFile(filePath, content)` (sends `textDocument/didOpen`). After the request, it calls `manager.closeFile(filePath)` (sends `textDocument/didClose`). The LSP spec requires files to be open before requesting positions in them.

**Result formatting** (`formatters.ts`): Each operation result is formatted as structured text:
- `goToDefinition` / `findReferences` → `path:line:char` lines
- `hover` → markdown text from `contents.value`
- `documentSymbol` → indented tree of symbol names with kinds and ranges
- `workspaceSymbol` → `name (kind) at path:line`
- Call hierarchy → tree with caller/callee names and locations

### 5.5 Passive Diagnostic Feedback

**Entry point:** `src/services/lsp/passiveFeedback.ts`

Language servers send `textDocument/publishDiagnostics` notifications whenever they detect errors in open files. `setupPassiveFeedback(manager)` registers a notification handler on each server that converts LSP diagnostics to Claude's internal `DiagnosticFile[]` format and stores them in the `LSPDiagnosticRegistry`.

**Severity mapping** (`mapLSPSeverity`):
- 1 (Error) → `'Error'`
- 2 (Warning) → `'Warning'`
- 3 (Information) → `'Info'`
- 4 (Hint) → `'Hint'`
- undefined → `'Error'` (defensive default)

**URI normalization**: Handles both `file://` URIs and plain paths. `fileURLToPath(uri)` converts `file://` URIs; otherwise the raw URI is used as a fallback. Graceful handling: malformed URIs from buggy LSP servers don't crash the diagnostic pipeline.

**Integration with tool system**: `LSPDiagnosticRegistry` (`LSPDiagnosticRegistry.ts`) stores diagnostics keyed by file path. When the `FileEditTool` or `FileWriteTool` completes a file write, it checks the registry for pending diagnostics on that file and attaches them as context to the next API call ("after editing foo.ts, TypeScript reports: error TS2345...").

---

## 6. Web Fetch and Search

### 6.1 WebFetchTool Pipeline

**Entry point:** `src/tools/WebFetchTool/WebFetchTool.ts`, `src/tools/WebFetchTool/utils.ts`

The WebFetch tool fetches a URL and applies a user-specified prompt to the content using a secondary model (claude-haiku).

**Pipeline:**
```
WebFetchTool.call(input: { url, prompt })
    │
    ├─ checkPermissions()
    │   ├─ isPreapprovedHost() → allow without user confirmation
    │   └─ getRuleByContentsForTool() → check user-set rules
    │
    ├─ getURLMarkdownContent(url)
    │   ├─ URL_CACHE.get(url) → cache hit: return immediately
    │   ├─ validateURL(url) → length < 2000, no credentials, ≥2 domain parts
    │   ├─ checkDomainBlocklist(domain) → GET api.anthropic.com/api/web/domain_info
    │   │   └─ DOMAIN_CHECK_CACHE → deduplicate same-domain checks
    │   ├─ http → https upgrade
    │   ├─ getWithPermittedRedirects() → custom redirect handling
    │   │   └─ isPermittedRedirect() → only www. add/remove or same-origin
    │   ├─ contentType check:
    │   │   ├─ text/html → getTurndownService().turndown(html) → markdown
    │   │   └─ other    → raw UTF-8 string
    │   ├─ isBinaryContentType() → persistBinaryContent() → save to disk
    │   └─ URL_CACHE.set(url, entry, { size: contentBytes })
    │
    └─ applyPromptToMarkdown(prompt, markdownContent)
        ├─ truncate to MAX_MARKDOWN_LENGTH (100,000 chars)
        ├─ makeSecondaryModelPrompt(content, prompt, isPreapproved)
        └─ queryHaiku({ systemPrompt, userPrompt }) → response text
```

**Caching** (`utils.ts:63-82`):
- `URL_CACHE`: LRU cache with 15-min TTL, 50MB max size. Keyed by original URL (not upgraded or redirected URL). Size accounting uses UTF-8 byte length.
- `DOMAIN_CHECK_CACHE`: LRU with 5-min TTL, max 128 entries. Only caches `'allowed'` decisions — blocked/failed re-check on next request.

**Security controls**:
- Domain blocklist: API call to `api.anthropic.com/api/web/domain_info?domain=X` returns `{ can_fetch: boolean }`. Enterprise users can set `skipWebFetchPreflight: true` in settings.
- Redirect restriction: Only same-origin or www add/remove redirects. Cross-origin redirects throw `isPermittedRedirect = false` (PSR requirement against open redirect exploitation).
- Max redirects: 10 (prevents infinite redirect loops; per-hop timeout resets without the cap).
- Max content: 10MB HTTP response (`MAX_HTTP_CONTENT_LENGTH`).
- Max markdown: 100K chars truncated before the secondary model call.
- SSRF prevention: Blocks URLs without ≥2 domain parts (prevents `http://localhost`, `http://192.168.1.1`).

**Turndown HTML→Markdown**: The `turndown` library is lazily instantiated as a module-level singleton (`getTurndownService()`). The first call to `turndown()` is expensive (~1.4MB retained heap for the DOM shim); caching one instance amortizes this across all fetches.

**Binary content handling**: `isBinaryContentType(contentType)` detects PDFs, images, archives, etc. Binary content is persisted to a temp file via `persistBinaryContent()`. The file path is returned in the tool result so Claude can inspect it with `ReadTool`.

### 6.2 WebSearchTool Pipeline

**Entry point:** `src/tools/WebSearchTool/WebSearchTool.ts`

The WebSearch tool uses the Anthropic SDK's native `web_search_20250305` beta tool type rather than a standalone HTTP request.

**Pipeline:**
```
WebSearchTool.call(input: { query, allowed_domains, blocked_domains })
    │
    ├─ makeToolSchema(input) → BetaWebSearchTool20250305
    │   { type: 'web_search_20250305', name: 'web_search',
    │     allowed_domains, blocked_domains }
    │
    └─ queryModelWithStreaming(model, messages, tools: [searchTool])
        │ (Claude with the web_search tool enabled makes the search)
        ├─ streaming response includes tool_use blocks (search queries)
        └─ streaming response includes tool_result blocks (search results)
            └─ returns SearchResult[] + text commentary
```

The search is done by the Claude API itself (server-side, not client-side HTTP). The `web_search_20250305` tool type signals the API to use Anthropic's web search integration. The model's response stream contains both the raw search hits and synthesized text commentary.

**Input schema**: `{ query: string (min 2 chars), allowed_domains?: string[], blocked_domains?: string[] }`. Domain filtering is passed directly to the API's tool definition.

**Progress streaming**: `renderToolUseProgressMessage` shows real-time search progress in the UI as Claude calls the web_search tool.

---

## 7. Voice Input

**Entry point:** `src/context/voice.tsx`

Voice input is managed via a React Context + custom store pattern, entirely separate from the main AppState.

**`VoiceState` type:**
```typescript
type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'
  voiceError: string | null
  voiceInterimTranscript: string  // real-time partial transcript
  voiceAudioLevels: number[]      // waveform visualization data
  voiceWarmingUp: boolean         // model loading indicator
}
```

**Store pattern** (`createStore` from `src/state/store.ts`): The voice context uses `useSyncExternalStore` for slice subscriptions. `VoiceProvider` creates one `Store<VoiceState>` instance (stable reference via `useState(() => createStore(...))`). Consumers subscribe to slices:
- `useVoiceState(selector)` — subscribes, re-renders on change
- `useSetVoiceState()` — stable setter reference, never causes re-renders
- `useGetVoiceState()` — synchronous reader for event handler callbacks

**State machine transitions** (managed by `VoiceKeybindingHandler` elsewhere):
```
'idle'
  │ (voice keybind pressed)
  ▼
'recording'  ← audio levels update continuously
  │ (keybind released or silence detected)
  ▼
'processing' ← voiceWarmingUp may be true initially
  │ (transcript ready)
  ▼
'idle' (text inserted into prompt input)
```

**Key design**: `useSetVoiceState().setState` is synchronous — callers can read `getVoiceState()` immediately after to observe the new value. `VoiceKeybindingHandler` relies on this for the recording start/stop sequence.

The actual audio capture and speech recognition are handled by a native module (not visible in this file) that calls `useSetVoiceState` to push state updates into the React tree.

---

## 8. Computer Use (macOS Control Layer)

**Entry point:** `src/utils/computerUse/`

Computer Use allows Claude to control the macOS GUI: take screenshots, move the mouse, type text, and interact with applications. It is macOS-only and gated behind subscription tier + GrowthBook feature flag.

**Two native modules:**
1. `@ant/computer-use-input` (Rust/enigo bindings): Mouse events, keyboard synthesis, frontmost app detection
2. `@ant/computer-use-swift` (Swift/Objective-C bindings): Screen capture via SCContentFilter, app listing via NSWorkspace, TCC permissions

**`executor.ts`** (`ComputerExecutor` implementation): Wraps the two native modules with a unified interface matching the `packages/desktop/computer-use-mcp` contract. CLI-specific deviations from the Desktop (Cowork) implementation:
- No `withClickThrough` (no Electron overlay window to make click-through-able)
- Terminal as surrogate host: `getTerminalBundleId()` detects the terminal emulator and tells Swift to exclude it from screenshots and z-order walks
- Clipboard via `pbcopy`/`pbpaste` instead of Electron's `clipboard` module

**Gates** (`gates.ts`): `isComputerUseEnabled()` checks:
1. GrowthBook flag `tengu_malort_pedway` → `{ enabled: boolean, ... }`
2. Subscription tier: Max/Pro only for external users (ants bypass tier check)
3. Platform: macOS only

**Lock** (`computerUseLock.ts`): `tryAcquireComputerUseLock()` prevents concurrent computer use sessions. The lock is process-global because the macOS input API is not designed for concurrent callers.

**ESC hotkey** (`escHotkey.ts`): Registers a CGEventTap abort that watches for bare Escape key presses during active computer use. `notifyExpectedEscape(parts)` hole-punches model-synthesized Escape keys (so the model can send Escape to target apps without triggering abort). Detection distinguishes `'escape'` / `'esc'` (case-insensitive).

**Drain run loop** (`drainRunLoop.ts`): The Swift module's `@MainActor` methods (`captureExcluding`, `resolvePrepareCapture`, etc.) dispatch to `DispatchQueue.main` and hang under libuv unless `CFRunLoop` is pumped. `drainRunLoop()` runs `CFRunLoop` for a brief window to allow MainActor tasks to complete.

**`wrapper.tsx`** (`ComputerUseSessionContext`): A thin adapter between `ToolUseContext` and the `bindSessionContext` from `@ant/computer-use-mcp`. Caches the dispatch binding once per process. The `ComputerUseApproval` React component (`src/components/permissions/ComputerUseApproval/`) presents the user with a preview of the planned action before execution.

**Rendering overrides** (`toolRendering.tsx`): Override how computer-use MCP tool results are rendered in the terminal (screenshots shown as `ClickableImageRef`, action summaries shown as progress lines).

**Sub-gates** (`CuSubGates`):
- `pixelValidation` — validate click coordinates are within screen bounds
- `clipboardPasteMultiline` — enable multi-line clipboard paste
- `mouseAnimation` — animate mouse movements
- `hideBeforeAction` — hide overlays before taking screenshots
- `autoTargetDisplay` — auto-select the display with the target app
- `clipboardGuard` — sanitize clipboard before paste

---

## 9. Auto-Update System

**Entry point:** `src/utils/autoUpdater.ts`

The auto-updater checks for new Claude Code versions and installs them in-place.

**Version distribution**: Updates are hosted on Google Cloud Storage: `storage.googleapis.com/claude-code-dist-86c565f3-.../claude-code-releases`.

**`assertMinVersion()`** (run at startup): Fetches `tengu_version_config.minVersion` from GrowthBook (`getDynamicConfig_BLOCKS_ON_INIT`). If the current version is below the minimum, prints a human-readable update instruction and calls `gracefulShutdownSync(1)`. This is the server-side kill switch for forcing upgrades.

**`getMaxVersion()`**: Server-side kill switch for pausing auto-updates during incidents. Different version caps for `ant` (dev builds) and `external` (public builds). Returns `undefined` when no cap is configured.

**`shouldSkipVersion(targetVersion)`**: Respects the user's `minimumVersion` setting (set when downgrading from unstable to stable channel: "don't downgrade below my current version"). Uses `gte(target, minimumVersion)` — if target < minimum, skip.

**Lock file** (`~/.claude/.update.lock`): Prevents concurrent updater instances. Uses atomic `O_EXCL` file creation for the lock. TOCTOU race protection: re-checks staleness immediately before unlinking a stale lock. Lock timeout: 5 minutes.

**Installation** (`getInstallationPrefix()`): Determines install prefix via `npm config get prefix` or `bun pm bin -g`. Then runs:
- npm: `npm install -g @anthropic-ai/claude-code@VERSION --prefix PREFIX`
- bun: `bun add -g @anthropic-ai/claude-code@VERSION`

**`AutoUpdater` component** (`src/components/AutoUpdater.tsx`): A background React component that polls for updates, shows a notification when a new version is available, and offers an in-place update. The `AutoUpdaterWrapper` manages the timing — updates are checked once at startup after a brief delay to avoid blocking the initial render.

**Channel management** (`ReleaseChannel`): Claude Code supports `stable`, `beta`, and `ant` (internal) channels. Channel is stored in global config. `ChannelDowngradeDialog` is shown when switching from `beta` to `stable` (the stable version may be older than the current beta installation).

---

## 10. Buddy / Companion System

**Entry points:** `src/buddy/`

The Buddy system is a companion gacha (randomized collection) feature — a whimsical companion creature that a user "hatches" once and keeps.

**Companion anatomy** (`src/buddy/types.ts`):
- `CompanionBones` — deterministic: species, eye, hat, rarity, shiny, stats (derived from hash of userId)
- `CompanionSoul` — model-generated: name, personality (stored in config after first hatch)
- `StoredCompanion = CompanionSoul & { hatchedAt: number }` — what actually persists in config

**Why separate bones from soul**: Bones are regenerated from `hash(userId)` on every read. This means:
- Species renames don't break stored companions (the species string isn't stored)
- Users can't edit their way to a legendary rarity (rarity is derived from hash, not stored)
- Any config schema migration only needs to preserve the soul (name + personality + hatchedAt)

**Gacha roll** (`companion.ts`): Uses Mulberry32 (a seeded PRNG), seeded from `hashString(userId)`. Rarity is determined by a weighted roll (common 60%, uncommon 25%, rare 10%, epic 4%, legendary 1%). Species, eye type, and hat are picked from their respective arrays. Stats are rolled with a peak stat, dump stat, and floor determined by rarity.

**Species obfuscation** (`types.ts:14-17`): Species names are constructed via `String.fromCharCode(...)` rather than being literal strings. This keeps the species names out of the compiled bundle's string literals, preventing accidental matches against the internal codename exclusion list (`excluded-strings.txt`).

**Teaser window** (`useBuddyNotification.tsx`): The `/buddy` command teaser is shown on startup only during April 1-7, 2026 (local date, not UTC, to spread Twitter activity across timezones). Internal (ant) users always see it. After April 7, the tease is gone but the command remains live.

**`CompanionSprite`** (`CompanionSprite.tsx`): Renders the companion as ASCII art using unicode block characters. The sprite changes based on species, eye type, hat, and shiny status. Idle animations and reactions are driven by the companion's personality prompt.

**Personality generation** (`prompt.ts`): When a user hatches their companion for the first time (`/buddy hatch`), a secondary model call generates the companion's name and personality based on its species, rarity, stats, and a randomized seed. The result is stored as `CompanionSoul` in global config.

---

## 11. GitHub App CI/CD Setup Wizard

**Entry point:** `src/commands/install-github-app/install-github-app.tsx`

The `install-github-app` command is a step-by-step wizard that sets up Claude Code GitHub Actions workflows in a repository.

**Wizard steps** (as a state machine via `step` field in `State`):
```
'check-gh'          ← verify GitHub CLI installed, authenticated, and repo detected
    │
    ▼
'warnings'          ← show any warnings (gh not found, no auth, no repo)
    │
    ▼
'choose-repo'       ← pick current repo or a different one
    │
    ▼
'api-key'           ← choose: existing key / new key / OAuth token
    │
    ▼
'oauth-flow'        ← (if OAuth) browser-based OAuth flow
    │
    ▼
'check-existing-secret' ← check if ANTHROPIC_API_KEY already in repo secrets
    │
    ▼
'install-app'       ← install the Claude Code GitHub App to the repo
    │
    ▼
'check-existing-workflow' ← check if .github/workflows/claude.yml already exists
    │
    ▼
'workflow-select'   ← select which workflows to install (claude, claude-review)
    │
    ▼
'creating'          ← create secret + workflow files via gh CLI
    │
    ▼
'success' / 'error'
```

**Workflow creation** (`setupGitHubActions.ts`): Uses `gh api` CLI commands to:
1. Create/update the GitHub repository secret (`ANTHROPIC_API_KEY` or custom name)
2. Create `.github/workflows/claude.yml` (main workflow) via `repos/{name}/contents/{path}` API
3. Create `.github/workflows/claude-review.yml` (PR review workflow, optional)

Workflow YAML is stored in `src/constants/github-app.ts` as template strings. When OAuth token auth is selected, the template is patched to use `claude_code_oauth_token:` parameter instead of `anthropic_api_key:`.

**Existing file handling**: `gh api repos/{name}/contents/{path} --jq .sha` retrieves the SHA of an existing workflow file. If present, the PUT request includes the SHA for update-in-place (GitHub requires SHA for file updates to prevent lost updates).

**PR creation**: After setup, `openBrowser(prUrl)` opens a GitHub PR diff page (if applicable) so the user can review the created workflow files. A PR title (`CLAUDE_CODE_ACTION_PR_TITLE`) and body (`CLAUDE_CODE_ACTION_PR_BODY`) are pre-defined in `src/constants/github-app.ts`.

**Auth detection**: `getAnthropicApiKey()` checks for existing API key. `isAnthropicAuthEnabled()` checks for OAuth auth availability. The initial `selectedApiKeyOption` is determined by which auth method is already configured.

---

## 12. Cross-Cutting Concerns

### Performance Instrumentation

**Debug repaint mode** (`CLAUDE_CODE_DEBUG_REPAINTS`): When set, each DOM node stores its `debugOwnerChain` (React component fiber chain: `['ToolUseLoader', 'Messages', 'REPL']`). The renderer uses this to attribute full-screen repaints to specific React components.

**Commit log** (`CLAUDE_CODE_COMMIT_LOG`): Writes per-commit timing data to a file: gap between commits (>30ms threshold), reconcile time (>20ms), create count (>50 nodes). Detects slow Yoga layout passes (>20ms) with Yoga cache statistics.

**Scroll profiling**: `BENCH_SCROLL_PROFILING` + `bench/scroll-e2e.sh` — an end-to-end profiling script that measures Yoga ms + commit ms + ANSI output size across scroll operations.

### React Compiler (Forget)

All compiled component files include `import { c as _c } from "react/compiler-runtime"` — the React Compiler (formerly "React Forget") automatic memoization. The `$` cache array holds memoized values; the `Symbol.for('react.memo_cache_sentinel')` check initializes the cache on first render. This eliminates manual `useMemo`/`useCallback` in most components.

### Analytics Integration in UI Components

Several UI components fire analytics events directly:
- `BridgeDialog`: no direct analytics (uses AppState subscriptions)
- `InstallGitHubApp`: `logEvent('tengu_install_github_app_started', {})` on mount
- `MCPServerApprovalDialog`: `logEvent('tengu_mcp_dialog_choice', { choice })` on selection
- `AutoModeOptInDialog`: fires accept/decline events
- `VSCode SDK MCP`: `logEvent(tengu_vscode_${eventName}, eventData)` for forwarded VS Code events

### Abort Signal Propagation

All long-running operations (fetch, LSP request, bridge poll, voice recording) accept an `AbortSignal`. The patterns are consistent:
- `CapacityWake.signal()` for the bridge poll loop sleep
- `axios({ signal })` for HTTP calls
- `AbortError` is a named subclass of Error for clean detection at call sites
- `onExit` from `signal-exit` for process-level cleanup registration

---

## 13. Essential Files Reference

The following files are essential for understanding the UI, Bridge, and Integration domains:

**Ink TUI Core:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/reconciler.ts` — custom React reconciler connecting React to the Ink DOM
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/ink.tsx` — Ink instance class: double-buffering, SIGCONT/resize, alt-screen, mouse, selection
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/dom.ts` — DOMElement type with scroll state fields; createNode/TextNode
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/renderer.ts` — createRenderer: Yoga → Output → Screen → Frame
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/render-node-to-output.ts` — tree walk, blit optimization, scroll viewport culling, ScrollHint
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/layout/yoga.ts` — YogaLayoutNode adapter over native Yoga
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/components/ScrollBox.tsx` — imperative scroll API, sticky scroll, clamp bounds
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/ink/parse-keypress.ts` — key sequence parser (Kitty, CSI-u, modifyOtherKeys)

**Theme System:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/theme.ts` — Theme type, 6 theme palettes, getTheme()
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/systemTheme.ts` — OSC 11 dark/light detection, COLORFGBG fallback

**Vim Mode:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/vim/types.ts` — VimState, CommandState state machine types (documentation-as-types)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/hooks/useVimInput.ts` — React hook wiring vim state machine to text input

**Remote Bridge:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/bridgeMain.ts` — multi-session poll loop, heartbeat, session lifecycle, backoff
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/sessionRunner.ts` — child process spawner, activity monitoring, permission request forwarding
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/replBridge.ts` — REPL-side bridge connection, BridgeCoreParams injection pattern
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/flushGate.ts` — message ordering gate during initial flush
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/capacityWake.ts` — AbortSignal-based sleep interruption on session completion
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/bridge/types.ts` — WorkData, WorkResponse, WorkSecret, SpawnMode, BridgeWorkerType
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/cli/transports/HybridTransport.ts` — WebSocket reads + batched HTTP POST writes
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/cli/transports/WebSocketTransport.ts` — WebSocket with reconnect, sleep detection, keep-alive
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/cli/transports/SSETransport.ts` — SSE reads + HTTP POST writes, sequence number tracking

**IDE Integration:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/ide.ts` — IDE detection, lockfile read, 17 IDE configs, WSL path conversion
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/hooks/useIdeSelection.ts` — MCP selection_changed notification subscription
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/vscodeSdkMcp.ts` — VS Code SDK MCP bidirectional channel, experiment gates push

**LSP:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/lsp/LSPClient.ts` — stdio transport, lazy handler queue, crash detection
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/lsp/LSPServerManager.ts` — extension-to-server routing, lazy start, file sync
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/lsp/LSPServerInstance.ts` — per-server state machine, transient error retry
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/LSPTool/LSPTool.ts` — 9 LSP operations exposed to Claude
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/lsp/passiveFeedback.ts` — publishDiagnostics → DiagnosticRegistry pipeline

**Web Tools:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/WebFetchTool/utils.ts` — URL fetch, domain blocklist, redirect policy, HTML→Markdown, LRU cache
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/WebSearchTool/WebSearchTool.ts` — Anthropic SDK native web_search_20250305 tool

**Minor Domains:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/context/voice.tsx` — voice state store (idle/recording/processing)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/computerUse/executor.ts` — macOS ComputerExecutor, native module wrappers
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/computerUse/gates.ts` — feature gate + subscription tier check
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/autoUpdater.ts` — version check, GCS download, npm/bun install, lock file
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/buddy/types.ts` — Companion type system, species obfuscation, rarity weights
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/buddy/companion.ts` — deterministic gacha roll with Mulberry32 PRNG
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/commands/install-github-app/install-github-app.tsx` — GitHub Actions setup wizard state machine
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/commands/install-github-app/setupGitHubActions.ts` — gh CLI workflow/secret creation

**UI Components:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/components/App.tsx` — root component tree (FpsMetrics / Stats / AppState providers)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/components/BridgeDialog.tsx` — Remote Control status UI, QR code
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/components/DesktopHandoff.tsx` — `/desktop` command deep-link flow

---

*End of Feature Analysis: 06-ui-integration.md*

---
