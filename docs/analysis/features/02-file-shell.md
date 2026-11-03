# Feature Analysis: File Operations, Shell Execution, and Version Control

## Document Information

- Source codebase: `/Users/simonscheurer/Development/agentic-coding/claude-code`
- Analysis date: 2026-04-03
- Target file: `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/02-file-shell.md`

---

Below is the complete document to write to the target path. It is provided here as the analysis output since no file-write tool is available in this agent context.

---

# 02 - File Operations, Shell Execution, and Version Control

## Table of Contents

1. [Overview](#overview)
2. [Domain 1: File Operations](#domain-1-file-operations)
   - [FileStateCache: The Staleness Guardian](#filestateache-the-staleness-guardian)
   - [FileReadTool: Reading with Context Awareness](#filereadtool-reading-with-context-awareness)
   - [FileEditTool: Precision String Replacement](#fileeedittool-precision-string-replacement)
   - [FileWriteTool: Full-File Replacement](#filewritetool-full-file-replacement)
   - [FileHistory: Checkpoint-Based Undo](#filehistory-checkpoint-based-undo)
   - [GlobTool: Pattern-Based File Discovery](#globtool-pattern-based-file-discovery)
   - [GrepTool: Regex Content Search via ripgrep](#greptool-regex-content-search-via-ripgrep)
   - [NotebookEditTool](#notebookediittool)
3. [Domain 2: Shell Execution](#domain-2-shell-execution)
   - [BashTool Architecture](#bashtool-architecture)
   - [The 23 Security Check Types](#the-23-security-check-types)
   - [Tree-Sitter AST Parser](#tree-sitter-ast-parser)
   - [Command Semantic Classification](#command-semantic-classification)
   - [The sed Edit Parser and Bypass Prevention](#the-sed-edit-parser-and-bypass-prevention)
   - [macOS Sandbox: sandbox-adapter](#macos-sandbox-sandbox-adapter)
   - [Background Shell Tasks](#background-shell-tasks)
   - [Shell Provider Selection](#shell-provider-selection)
4. [Domain 3: Version Control](#domain-3-version-control)
   - [Git Root Discovery](#git-root-discovery)
   - [Worktree Integration](#worktree-integration)
   - [Commit Attribution Tracking](#commit-attribution-tracking)
   - [Undercover Mode for Public Repos](#undercover-mode-for-public-repos)
   - [Gitignore Filtering](#gitignore-filtering)
5. [Cross-Cutting Concerns](#cross-cutting-concerns)
6. [Architecture Insights and Observations](#architecture-insights-and-observations)
7. [Essential Files Reference](#essential-files-reference)

---

## Overview

This document provides an in-depth analysis of the File Operations, Shell Execution, and Version Control domains of the Claude Code CLI. These three domains are deeply intertwined: file tools maintain a shared read-timestamp cache to prevent stale edits; the shell tool (BashTool) can itself create, modify, and delete files and therefore participates in the same history/undo system; and git utilities underpin both the attribution system that tracks Claude's contributions and the worktree feature that gives agents isolated working environments.

The architecture consistently prioritizes three values:

1. **Safety**: Multiple overlapping guards prevent unintended data loss or privilege escalation.
2. **Correctness**: Encoding detection, line-ending preservation, and quote normalization ensure edits do not corrupt files.
3. **Performance**: Sync reads in the critical edit path, LRU caches for file state, and ripgrep for search all minimize latency.

---

## Domain 1: File Operations

### FileStateCache: The Staleness Guardian

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/fileStateCache.ts`

Every file operation that might write must be preceded by a read. The `FileStateCache` enforces this invariant and prevents two critical failure modes:

1. **Concurrent external modification**: If a linter, formatter, or the user edits a file between Claude's read and write, the stale-write guard catches it.
2. **Partial-view writes**: Files injected into context automatically (CLAUDE.md, memory files) receive an `isPartialView: true` flag, blocking writes until the model explicitly reads the full content.

```
FileStateCache (LRU, 100 entries, 25 MB max)
  key   = normalize(absolutePath)
  value = { content, timestamp, offset?, limit?, isPartialView? }
```

The cache is a thin wrapper around `lru-cache` with path normalization. Size eviction is byte-based (`sizeCalculation: value => Buffer.byteLength(value.content)`), protecting against large-file accumulation. Images are intentionally excluded from the cache.

**Lifecycle**:
- `FileReadTool.call()` sets an entry after every successful read, recording content, mtime (milliseconds), and the offset/limit if a partial read.
- `FileEditTool.validateInput()` and `FileEditTool.call()` both consult the cache. The `validateInput` phase rejects with error code 6 when no entry exists; the `call` phase does a second sync check against `getFileModificationTime()` to close the TOCTOU window.
- After a successful write, both tools update the cache entry to the new content and fresh mtime, so the next edit in the same turn is valid.

**Windows special case** (FileEditTool.ts line 293–302): Cloud sync tools (OneDrive, Dropbox) and antivirus scanners can bump a file's mtime without changing its content. When `lastWriteTime > readTimestamp.timestamp` is detected on a full (non-partial) read, the tool falls back to content comparison before throwing `FILE_UNEXPECTEDLY_MODIFIED_ERROR`. This prevents spurious "file modified since read" errors on Windows.

---

### FileReadTool: Reading with Context Awareness

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileReadTool/FileReadTool.ts`

FileReadTool handles six distinct output types via a discriminated union schema:

| Output type | Trigger condition |
|-------------|-------------------|
| `text` | Plain text files, with optional offset/limit for windowed reads |
| `image` | Extension in `{png,jpg,jpeg,gif,webp}` — returns base64 + dimensions |
| `notebook` | `.ipynb` extension — parsed as Jupyter notebook cells |
| `pdf` | `.pdf` extension — inline base64 if small, or extracted page images |
| `parts` | Large PDF exceeding extraction threshold — outputs per-page images to a temp dir |
| `file_unchanged` | Hash-equality shortcut: file content is the same as last read (avoids re-sending tokens) |

**Blocked device paths**: A compile-time `Set` blocks reads of `/dev/zero`, `/dev/random`, `/dev/urandom`, `/dev/full`, `/dev/stdin`, `/dev/tty`, and Linux `/proc/self/fd/0-2` to prevent infinite-read hangs (FileReadTool.ts line 98–128).

**Image processing pipeline** (`src/tools/FileReadTool/imageProcessor.ts`):
The tool dynamically loads either `image-processor-napi` (a native NAPI module for bundled builds) or `sharp` (as a fallback). The result is a `SharpInstance` interface used by `maybeResizeAndDownsampleImageBuffer` and `compressImageBufferWithTokenLimit` in `src/utils/imageResizer.ts`. Images are resized to fit within Anthropic API vision limits before base64 encoding.

**Encoding detection** (`src/utils/fileRead.ts`):
```
readFileSyncWithMetadata(filePath):
  1. Read first 4096 bytes via getFsImplementation().readSync()
  2. Empty file -> 'utf8' (prevents emoji/CJK corruption in empty files)
  3. Bytes [0xFF, 0xFE] -> 'utf16le' (UTF-16 LE BOM)
  4. Bytes [0xEF, 0xBB, 0xBF] -> 'utf8' (UTF-8 BOM)
  5. Otherwise -> 'utf8' (superset of ASCII)
  6. Detect CRLF vs LF by scanning the first 4096 chars
  7. Normalize CRLF -> LF in returned content (re-expanded on write)
```

This encoding metadata travels from `readFileSyncWithMetadata` through the edit/write critical section so files round-trip with their original encoding and line endings.

**Token limit enforcement**: After reading content, `countTokensWithAPI()` or `roughTokenCountEstimationForFileType()` is used. If token count exceeds `maxTokens`, a `MaxFileReadTokenExceededError` is thrown with guidance to use offset/limit.

**macOS screenshot path resolution** (FileReadTool.ts line 147–159): macOS versions differ in whether they use regular space (U+0020) or narrow no-break space (U+202F) before AM/PM in screenshot filenames. The tool tries the alternate space character when the original path does not exist.

**File read listener registry** (FileReadTool.ts line 161–173): A module-level array allows other services (e.g., code indexing) to be notified when files are read, via `registerFileReadListener()`.

---

### FileEditTool: Precision String Replacement

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileEditTool/FileEditTool.ts`  
**Utilities**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileEditTool/utils.ts`

FileEditTool implements surgical string replacement: it takes `old_string` and `new_string` parameters, finds the former in the file, and replaces it with the latter. The tool is deliberately simple in its data model but complex in its validation and normalization logic.

**Execution flow** (8 steps as commented in `call()`):

```
1. expandPath(file_path) -> absoluteFilePath
2. mkdir(dirname(absoluteFilePath))            -- create parent dir if needed
3. fileHistoryTrackEdit(...)                   -- async backup BEFORE write
4. readFileForEdit(absoluteFilePath)            -- sync read with metadata
5. Staleness check (mtime vs FileStateCache)
6. findActualString(content, old_string)        -- quote normalization
7. preserveQuoteStyle(old_string, actual, new_string) -- round-trip typography
8. getPatchForEdit() -> { patch, updatedFile }
9. writeTextContent(...)                        -- atomic write to disk
10. Notify LSP (didChange + didSave)
11. Notify VSCode diff view
12. Update FileStateCache
13. Log events, emit gitDiff if remote feature enabled
14. Return structured patch + original content
```

**Quote normalization** (`findActualString` in utils.ts line 73–93):
Claude cannot output typographic ("curly") quotes because they are normalized to straight quotes in the API message flow. `findActualString` handles the case where a file uses curly quotes:
1. Try exact match first.
2. Normalize both the search string and the file content (curly -> straight).
3. If the normalized search is found, return the slice from the original file that corresponds to that position.

This means the `old_string` "she said 'hello'" will match the file's `she said 'hello'` (with left/right single curly quotes) and return the curly-quoted version as the actual string.

**Quote style preservation** (`preserveQuoteStyle` in utils.ts line 104–136):
After finding the actual old string (which may contain curly quotes), the same curly quote style is applied to `new_string`. An open/close heuristic determines directionality: a quote preceded by whitespace, start-of-string, or opening punctuation is an opening quote; otherwise it is a closing quote. Apostrophes between two letter characters are treated as contractions and rendered as right single curly quotes.

**Multiple-match safety**: If `old_string` appears more than once in the file and `replace_all` is `false`, the tool returns error code 9 with the match count, asking the model to add more context to uniquely identify the target.

**Jupyter notebook guard**: `.ipynb` files are blocked with error code 5, redirecting to `NotebookEditTool`. This prevents accidentally breaking JSON structure by treating the notebook as plain text.

**File size guard**: Files larger than 1 GiB are rejected to prevent OOM (1 GiB byte limit ≈ V8/Bun string length limit).

**Settings file validation**: After the edit is simulated (via `replace_all ? file.replaceAll(...) : file.replace(...)`), `validateInputForSettingsFileEdit` is called to ensure the proposed change does not corrupt Claude's own settings files.

**Desanitization** (`DESANITIZATIONS` map in utils.ts line 531–550):
Claude's system prompt masks certain XML-like tokens (`<function_results>`, `<system>`, etc.) to prevent prompt injection. The model outputs sanitized versions (`<fnr>`, `<s>`, etc.). Before applying the edit, these sanitized tokens in `old_string` are restored to their real counterparts.

---

### FileWriteTool: Full-File Replacement

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileWriteTool/FileWriteTool.ts`

FileWriteTool replaces the entire file content in one operation. It shares the same core guards as FileEditTool but with a simpler edit model.

**Key differences from FileEditTool**:
- No `old_string`/`new_string` — takes `content` directly.
- Explicitly forces `LF` line endings on write (`writeTextContent(..., 'LF')`). This is intentional: "The model sent explicit line endings in `content` and meant them." Previous behavior that preserved CRLF silently corrupted bash scripts.
- New file creation (ENOENT) is always valid — no prior read required.
- For existing files, the same staleness check applies.
- Output `type` is `'create'` or `'update'` discriminated by whether `meta` was null (ENOENT).

---

### FileHistory: Checkpoint-Based Undo

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/fileHistory.ts`

FileHistory provides a checkpoint system that lets users rewind file state to any previous message boundary. It is the foundation for the "undo last N changes" UI button in the REPL.

**Data structures**:
```typescript
FileHistoryState {
  snapshots: FileHistorySnapshot[]   // max 100 (evicts oldest)
  trackedFiles: Set<string>          // all files that have ever been backed up
  snapshotSequence: number           // monotonic counter (for UI activity signals)
}

FileHistorySnapshot {
  messageId: UUID                    // the assistant message that triggered this
  trackedFileBackups: Record<string, FileHistoryBackup>  // path -> backup
  timestamp: Date
}

FileHistoryBackup {
  backupFileName: string | null      // null = file did not exist at this version
  version: number                    // monotonic per-file (v1, v2, ...)
  backupTime: Date
}
```

Backup files are stored in the Claude config home directory under a path derived from the file path. The v1 backup is keyed on content hash (idempotent — the same content always produces the same v1 backup, preventing double-backups).

**Three-phase commit protocol for `fileHistoryTrackEdit`**:
```
Phase 1: Read current state (no-op updater, captures snapshot reference)
         → If file already tracked in most-recent snapshot, early exit

Phase 2: Async IO
         → createBackup(filePath, version=1) -- hash-keyed, idempotent
         → Records null if file does not exist

Phase 3: Commit (updater function)
         → Re-check tracked (race: another trackEdit may have raced)
         → Mutate snapshot.trackedFileBackups in place
         → recordFileHistorySnapshot() -> session storage (for resume support)
```

This protocol ensures backup integrity even under concurrent edit calls (multiple tool uses in the same turn).

**Snapshot creation (`fileHistoryMakeSnapshot`)**:
Called once per assistant message boundary. For each tracked file, it:
1. Stats the file (ENOENT = deleted).
2. Compares mtime against the latest backup to decide if a new backup is needed.
3. Inherits unchanged backups from the previous snapshot.
4. Appends a new `FileHistorySnapshot` to the state array.
5. Evicts oldest snapshots when `allSnapshots.length > MAX_SNAPSHOTS` (100).

**Rewind (`fileHistoryRewind`)**:
Finds the target snapshot by `messageId` (uses `findLast` to handle duplicate IDs from retries), then calls `applySnapshot()` which:
- For each tracked file, reads the backup file if it exists, or deletes the current file if `backupFileName === null` (the file was created after the snapshot).
- Uses `diffLines` from the `diff` library to compute insertions/deletions for the diff stats display.

**Enabled/disabled gates**:
- In interactive mode: enabled unless `fileCheckpointingEnabled === false` in global config or `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` env var is set.
- In SDK/non-interactive mode: disabled unless `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` is set (and the disable var is not set).

---

### GlobTool: Pattern-Based File Discovery

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/GlobTool/GlobTool.ts`  
**Core utility**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/glob.ts`

GlobTool wraps ripgrep rather than `fast-glob`. This is a notable architectural choice: ripgrep's `--files` mode is used purely for its superior performance with `--no-ignore` and its built-in `.gitignore` integration.

**ripgrep invocation** (from `src/utils/glob.ts`):
```
rg --files --glob <pattern> --sort=modified
   [--no-ignore]   (default true, controlled by CLAUDE_CODE_GLOB_NO_IGNORE)
   [--hidden]      (default true, controlled by CLAUDE_CODE_GLOB_HIDDEN)
   [--glob !<ignorePattern>...]   (from permission settings)
   [--glob <exclusion>...]        (orphaned plugin cache dirs)
```

Absolute patterns are split into a `baseDir` (everything before the first glob metacharacter `* ? [ {`) and a relative `relativePattern` for ripgrep's `PATH` argument. Results are returned as absolute paths, then converted to relative paths for token efficiency.

**Truncation**: The tool hard-caps at 100 results by default (`limit = globLimits?.maxResults ?? 100`), returning a `truncated: boolean` flag.

**Permission integration**: `checkReadPermissionForTool` checks against the tool permission context before execution. `getFileReadIgnorePatterns` from `filesystem.ts` supplies additional glob exclusions derived from deny rules.

---

### GrepTool: Regex Content Search via ripgrep

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/GrepTool/GrepTool.ts`

GrepTool is a rich ripgrep wrapper with three output modes:

| Mode | ripgrep flag | Output |
|------|-------------|--------|
| `files_with_matches` (default) | `-l` | File paths, sorted by mtime desc |
| `content` | (none) | Matching lines with optional context |
| `count` | `-c` | `filename:count` per file |

**Arguments assembled before ripgrep call**:
```
--hidden                      always
--glob !.git --glob !.svn ... VCS exclusion (6 directories)
--max-columns 500             prevent base64/minified line bloat
[-U --multiline-dotall]       if multiline=true
[-i]                          if case_insensitive=true
[-l | -c]                     mode flags
[-n]                          line numbers in content mode
[-C N | -B N -A N]            context lines in content mode
[-e <pattern> | <pattern>]    pattern (-e used when starts with -)
[--type <type>]               file type filter
[--glob <pattern>...]         from glob parameter
[--glob !<ignorePattern>...]  from permission settings
[--glob <exclusion>...]       orphaned plugin cache
```

**head_limit / offset pagination**: All three modes support an explicit truncation limit (default 250). The limit is applied after collecting results so sorting (by mtime for `files_with_matches`) happens on the full set first.

**files_with_matches sort**: Files are sorted by `mtimeMs` descending (most recently modified first) using `Promise.allSettled` for stat calls. Failed stats sort to time 0 rather than rejecting the batch.

**VCS exclusion**: Six VCS directories are always excluded: `.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl`.

**Max-columns**: The 500-character limit prevents ripgrep from emitting base64 blobs or minified JS lines, which would bloat model context without useful information.

---

### NotebookEditTool

Jupyter notebooks (`.ipynb`) require special handling because they are JSON files with a specific cell schema. FileEditTool redirects `.ipynb` edits (error code 5) to `NotebookEditTool`, which understands cell indices, cell types, and source arrays. The NotebookEditTool lives at `src/tools/NotebookEditTool/` and uses `src/utils/notebook.ts` for reading/mapping cells.

---

## Domain 2: Shell Execution

### BashTool Architecture

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/BashTool.tsx`

BashTool is the most architecturally complex tool. It is a thin orchestration layer over several subsystems:

```
BashTool.call()
  |
  +-- _simulatedSedEdit? -> applySedEdit() [sed preview bypass]
  |
  +-- runShellCommand() generator
  |     |
  |     +-- spawnShellTask() / backgroundExistingForegroundTask()
  |     +-- SandboxManager.annotateStderrWithSandboxFailures()
  |     +-- EndTruncatingAccumulator (stdout buffer)
  |
  +-- interpretCommandResult() [semantic exit code mapping]
  +-- resetCwdIfOutsideProject() [CWD guard]
  +-- Persist large output to tool-results dir (>30KB inline limit)
  +-- trackGitOperations() [detect commits/checkouts for attribution]
```

**Schema design**: The full schema includes `_simulatedSedEdit` (a pre-computed sed result from the permission preview dialog) but this field is **always omitted** from the model-facing schema via `.omit({ _simulatedSedEdit: true })`. This prevents the model from bypassing permission checks by constructing a simulated edit directly.

**Input schema** (simplified):
```typescript
{
  command: string
  timeout?: number         // max: getMaxTimeoutMs()
  description?: string     // human-readable summary
  run_in_background?: boolean
  dangerouslyDisableSandbox?: boolean
}
```

**Output schema** includes: `stdout`, `stderr`, `interrupted`, `isImage`, `backgroundTaskId`, `backgroundedByUser`, `assistantAutoBackgrounded`, `dangerouslyDisableSandbox`, `returnCodeInterpretation`, `noOutputExpected`, `structuredContent`, `persistedOutputPath`, `persistedOutputSize`.

**Large output persistence**: When `result.outputFilePath` exceeds a 64 MB cap, the output file is truncated before being hard-linked (or copied) to the tool-results directory. The model receives a `<persisted-output>` block with a preview and the file path for subsequent `FileRead` calls.

**Assistant auto-backgrounding**: In assistant mode (non-main-thread), commands running longer than `ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds) are automatically moved to the background. The model receives a message explaining this and is told to use `run_in_background` explicitly for long tasks.

---

### The 23 Security Check Types

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/bashSecurity.ts`

The `BASH_SECURITY_CHECK_IDS` constant (lines 77–101) defines 23 numeric identifiers for analytics logging. Each corresponds to a validator function that returns `behavior: 'ask'` (prompt user) or `behavior: 'passthrough'` (continue to next check). The full list:

| ID | Name | What it catches |
|----|------|----------------|
| 1 | `INCOMPLETE_COMMANDS` | Commands starting with `-`, `&&`, `\t`, etc. (fragment detection) |
| 2 | `JQ_SYSTEM_FUNCTION` | `jq` with `system()` call (arbitrary command execution via jq) |
| 3 | `JQ_FILE_ARGUMENTS` | `jq --from-file`, `--rawfile`, `--slurpfile`, `-L` (arbitrary file read) |
| 4 | `OBFUSCATED_FLAGS` | ANSI-C quoting `$'...'`, locale quoting `$"..."`, empty-quote-dash `''-` |
| 5 | `SHELL_METACHARACTERS` | `;` or `&` inside quoted arguments to `find`, `grep`, etc. |
| 6 | `DANGEROUS_VARIABLES` | Variables (`$VAR`) adjacent to redirection/pipe operators |
| 7 | `NEWLINES` | Unquoted newlines that could introduce multi-command sequences |
| 8 | `DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION` | `$()`, `` ` ``, `<()`, `>()`, `=()`, Zsh substitutions |
| 9 | `DANGEROUS_PATTERNS_INPUT_REDIRECTION` | Unquoted `<` |
| 10 | `DANGEROUS_PATTERNS_OUTPUT_REDIRECTION` | Unquoted `>` |
| 11 | `IFS_INJECTION` | `$IFS` or `${...IFS...}` (field separator injection) |
| 12 | `GIT_COMMIT_SUBSTITUTION` | `$(...)`, backticks, `${}` inside a git commit `-m` message |
| 13 | `PROC_ENVIRON_ACCESS` | `/proc/*/environ` paths (environment variable exfiltration) |
| 14 | `MALFORMED_TOKEN_INJECTION` | Unbalanced delimiters combined with command separators (eval bypass) |
| 15 | `BACKSLASH_ESCAPED_WHITESPACE` | `\ ` (backslash-space) or `\<newline>` mid-word (tree-sitter/bash differential) |
| 16 | `BRACE_EXPANSION` | `{a,b}` or `{a..b}` patterns (brace expansion obfuscation) |
| 17 | `CONTROL_CHARACTERS` | ASCII 0x00-0x1F except `\t`, `\n` (invisible characters) |
| 18 | `UNICODE_WHITESPACE` | Non-ASCII whitespace (NBSP, ZWNBSP, etc. — hidden word separators) |
| 19 | `MID_WORD_HASH` | `'x'#` — quote-adjacent `#` that could start a comment after unquoting |
| 20 | `ZSH_DANGEROUS_COMMANDS` | `zmodload`, `emulate`, `sysopen`, `zpty`, `ztcp`, etc. (Zsh module attacks) |
| 21 | `BACKSLASH_ESCAPED_OPERATORS` | `\;`, `\|`, `\&` (operators hidden via backslash from text-based scanners) |
| 22 | `COMMENT_QUOTE_DESYNC` | `#` characters that shift quote parsing after unquoting |
| 23 | `QUOTED_NEWLINE` | `$'\n'` or similar — newline encoded via quoting mechanisms |

**Quote extraction pipeline** (`extractQuotedContent`):
Before most validators run, the command is processed through a hand-rolled quote state machine that produces three variants:
- `withDoubleQuotes`: single-quoted content removed, double-quoted content preserved
- `fullyUnquoted`: all quoted content removed
- `unquotedKeepQuoteChars`: quoted content removed but quote characters themselves kept (needed for `validateMidWordHash`)

These variants plus `fullyUnquotedPreStrip` (before safe redirections like `>/dev/null` are stripped) form the `ValidationContext` passed to each validator.

**Safe heredoc exception** (`isSafeHeredoc`):
The otherwise-blocked `$(` command substitution pattern has a carved-out exception for `$(cat <<'EOF'\n...\nEOF\n)`. The check uses line-by-line parsing (not regex `[\s\S]*?`) to exactly replicate bash's heredoc-closing behavior and prevent early-delimiter attacks. After stripping verified heredocs, the remaining text must pass all other validators and must appear in argument position (not command-name position) to prevent the heredoc body from becoming an executable command name.

**Carriage return differential** (line 943–1015):
`\r` is particularly dangerous because shell-quote (the JS tokenizer used in the legacy code path) treats `\r` as a word boundary (JS `\s` includes CR) while bash's default IFS does not. This allows an attacker to construct a command that appears to match an allowlist rule via shell-quote parsing but executes a different command in bash. The validator blocks CR everywhere except inside double quotes, where both parsers agree it is literal data.

---

### Tree-Sitter AST Parser

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/bash/ast.ts`  
**Parser binding**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/bash/parser.ts`

The AST-based parser replaces the legacy regex/shell-quote approach for the permission check phase. Its design is "fail-closed allowlist": any tree-sitter node type not explicitly allowed causes the entire command to be classified as `too-complex`, triggering the permission prompt.

**Result types**:
```typescript
ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

**`SimpleCommand` structure**:
```typescript
{
  argv: string[]          // argv[0] = command name, rest = arguments (quotes resolved)
  envVars: { name, value }[]  // leading VAR=val assignments
  redirects: Redirect[]       // I/O redirections with operator and target
  text: string               // original source span
}
```

**Pre-checks before tree-sitter analysis** (lines 408–437):
Even before walking the AST, the following patterns are rejected as `too-complex`:
- Control characters (`\x00-\x08`, `\x0B-\x1F`, `\x7F`)
- Unicode whitespace (NBSP, ZWSP, line separators, BOM)
- Backslash before space/tab or mid-word before newline
- Zsh `~[name]` dynamic directory syntax
- Zsh `=cmd` equals expansion
- Brace+quote obfuscation (`{a'}',b}`)

**PARSE_ABORTED handling**: When the native NAPI parser hits its timeout or resource limit (adversarially triggerable via deep arithmetic subscripts), it returns `PARSE_ABORTED`. Previously this was routed to the `parse-unavailable` legacy path, which lacks `EVAL_LIKE_BUILTINS` checks — `trap`, `enable`, `hash` could leak with `Bash(*):` allow rules. Now it routes to `too-complex`, closing this attack surface.

**Variable scope tracking** (`varScope: Map<string, string>`):
When a `variable_assignment` appears (`NOW=$(date)`), the assigned variable is tracked with a placeholder `__CMDSUB_OUTPUT__`. When a `simple_expansion` (`$NOW`) references a tracked variable later in the same command, the placeholder is substituted rather than returning `too-complex`. This allows patterns like:
```bash
NOW=$(date) && jq --arg now "$NOW" ...
```
to be parsed correctly (the inner `date` command is also extracted as a `SimpleCommand`).

**Safe environment variables** (`SAFE_ENV_VARS`):
`$HOME`, `$PWD`, `$USER`, `$PATH`, `$SHELL`, `$HOSTNAME`, `$UID`, `$EUID`, `$PPID`, `$RANDOM`, `$SECONDS`, `$LINENO`, `$TMPDIR`, and several bash builtins are resolvable inside strings. `$IFS` is in the set but blocked as a bare argument (the `insideString` gate in `resolveSimpleExpansion` enforces this).

---

### Command Semantic Classification

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/commandSemantics.ts`  
**UI classification**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/BashTool.tsx` (lines 59–172)

**Exit code semantics**:
The `interpretCommandResult` function maps exit codes to `{ isError, message }` pairs for a known set of commands:

| Command | Exit 0 | Exit 1 | Exit 2+ |
|---------|--------|--------|---------|
| `grep` / `rg` | Matches found | No matches (not error) | Error |
| `find` | Success | Partial (some dirs inaccessible) | Error |
| `diff` | No differences | Differences found (not error) | Error |
| `test` / `[` | Condition true | Condition false | Error |
| Others | Success | Error | Error |

For compound commands, `heuristicallyExtractBaseCommand` takes the LAST segment (which determines the pipeline exit code).

**UI collapsibility classification** (`isSearchOrReadBashCommand`):
Commands are classified as `{ isSearch, isRead, isList }` for the UI to decide whether to collapse the result:

| Category | Commands |
|----------|---------|
| Search | `find`, `grep`, `rg`, `ag`, `ack`, `locate`, `which`, `whereis` |
| Read | `cat`, `head`, `tail`, `less`, `more`, `wc`, `stat`, `file`, `strings`, `jq`, `awk`, `cut`, `sort`, `uniq`, `tr` |
| List | `ls`, `tree`, `du` |
| Neutral (skipped) | `echo`, `printf`, `true`, `false`, `:` |
| Silent (no-output) | `mv`, `cp`, `rm`, `mkdir`, `rmdir`, `chmod`, `chown`, `touch`, `ln`, `cd`, `export`, `unset`, `wait` |

For a compound command to be collapsible, ALL non-neutral parts must belong to the same category. A single write command (`mv`, etc.) makes the entire pipeline non-collapsible.

---

### The sed Edit Parser and Bypass Prevention

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/sedEditParser.ts`

When Claude wants to make a targeted string substitution via `sed -i 's/pattern/replacement/flags' file`, the permission dialog presents a preview of the edit (like FileEditTool's diff view) rather than just the raw command. This requires parsing the sed command to extract the file path and substitution.

**Parser algorithm** (`parseSedEditCommand`):
1. Tokenize the sed arguments using `tryParseShellCommand` (the same shell-quote parser used in security validation).
2. Walk tokens: extract `-i`/`--in-place` (with optional backup suffix), `-E`/`-r` (extended regex), `-e`/`--expression` flags and expressions, and file path.
3. Parse the expression as `s/pattern/replacement/flags` with a state machine that tracks escaped characters.
4. Validate flags against `/^[gpimIM1-9]*$/` (only safe substitution flags allowed).

**BRE-to-ERE conversion** (`applySedSubstitution`):
Because JavaScript `RegExp` uses ECMAScript regex (similar to ERE), BRE patterns must be translated. Metacharacters are inverted via a placeholder-and-swap approach:
```
\\ -> BACKSLASH_PLACEHOLDER
\+ -> PLUS_PLACEHOLDER
...then unescaped metacharacters are escaped...
...then placeholders are restored as unescaped ERE metacharacters
```
This approach prevents the conversion itself from introducing bypass vectors (a regex operating on partially-converted regex text is prone to order-of-operations errors; null-byte sentinels ensure no collision with user input).

**Anti-injection in replacement** (`applySedSubstitution` line 303–312):
The `&` character in sed replacement means "entire match" (like `$&` in JS). To prevent `\&` (literal ampersand) from accidentally being converted to `$&&` (match + literal ampersand), a random salt is used for the escaped-ampersand placeholder:
```typescript
const salt = randomBytes(8).toString('hex')
const ESCAPED_AMP_PLACEHOLDER = `___ESCAPED_AMPERSAND_${salt}___`
```

**Permission flow for sed**: When `parseSedEditCommand` returns non-null, BashTool's `userFacingName()` calls `fileEditUserFacingName()` to render the permission as a file edit. After user approval, `_simulatedSedEdit` is set on the input and `applySedEdit()` is called directly (bypassing the actual `sed` process), ensuring the user sees exactly what will be written.

---

### macOS Sandbox: sandbox-adapter

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/sandbox/sandbox-adapter.ts`

Claude Code integrates `@anthropic-ai/sandbox-runtime` for OS-level sandboxing. The adapter converts Claude Code's permission rule format to the sandbox runtime's `SandboxRuntimeConfig`.

**shouldUseSandbox logic** (`src/tools/BashTool/shouldUseSandbox.ts`):
```
SandboxManager.isSandboxingEnabled()  -> false? skip sandbox
input.dangerouslyDisableSandbox       -> skip if unsandboxed commands allowed by policy
!input.command                        -> skip (empty command)
containsExcludedCommand(input.command) -> skip (user-configured exclusions)
-> sandbox
```

**`convertToSandboxRuntimeConfig`** translates settings to `SandboxRuntimeConfig`:

- `allowWrite`: `.` (cwd) + Claude temp dir + additional dirs (from `--add-dir`) + worktree main repo path + `Edit` allow rules
- `denyWrite`: settings files (all sources) + `.claude/skills` + `Edit` deny rules + bare git repo files (HEAD, objects, refs, hooks, config)
- `denyRead`: `Read` deny rules
- Network: domains extracted from `WebFetch(domain:*)` allow rules

**Bare git repo protection** (lines 257–280):
A planted `HEAD` + `objects/` + `refs/` in cwd would trick git into treating it as a bare repo, potentially triggering `core.fsmonitor` hooks and escaping the sandbox. The adapter blocks writes to these paths. For non-existent paths (which would generate `/dev/null` stubs from bwrap), they are instead tracked in `bareGitRepoScrubPaths` and cleaned up after each sandboxed command.

**Settings file protection** (lines 230–245):
All Claude settings files (global, local, project, policy) are unconditionally added to `denyWrite` regardless of permission rules. This prevents a sandboxed command from escalating privileges by modifying settings to add allow rules.

**Path pattern resolution** (`resolvePathPatternForSandbox` vs `resolveSandboxFilesystemPath`):
- Permission rule paths (`/foo`) are settings-relative (CC convention).
- `sandbox.filesystem.*` paths (`/foo`) are absolute (standard convention).
These two resolution functions exist because users reasonably expect `/Users/foo/.cargo` in `allowWrite` to be an absolute path, not relative to the settings file directory.

---

### Background Shell Tasks

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tasks/LocalShellTask/LocalShellTask.ts`

Background tasks are managed via `spawnShellTask`. When `run_in_background: true` is set or a command is auto-backgrounded, the output is streamed to a task output file keyed by `backgroundTaskId`.

**Auto-backgrounding** in BashTool (lines 605–625):
In assistant (non-interactive) mode, commands that run longer than `ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds) are moved to the background. The model is notified with the task ID and output path. In regular interactive mode, the user can press Ctrl+B to manually background a running command.

**`sleep` special case**: Commands beginning with `sleep N` (integer N >= 2 seconds) are blocked by `detectBlockedSleepPattern` when the Monitor tool is available. The validator suggests `run_in_background: true` for blocking waits or the Monitor tool for polling patterns.

---

### Shell Provider Selection

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/Shell.ts`

Claude Code supports `bash` and `zsh` as shell providers (Windows uses PowerShell via a separate provider). Shell selection order:
1. `CLAUDE_CODE_SHELL` env var override (must contain `bash` or `zsh` in path)
2. `$SHELL` env var if it contains `bash` or `zsh`
3. System detection via `which zsh` / `which bash`
4. Preference: if `$SHELL` contains bash, prefer bash; otherwise prefer zsh

The security validators (bashSecurity.ts) include Zsh-specific checks because BashTool may run in zsh. The `ZSH_DANGEROUS_COMMANDS` set blocks `zmodload`, `emulate`, `zpty`, `ztcp`, and several zsh/files module builtins. Zsh-specific expansion syntax (`=cmd`, `~[name]`, glob qualifiers `(e:`, `(+`) is also blocked.

---

## Domain 3: Version Control

### Git Root Discovery

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/git.ts`

`findGitRoot` walks up the directory tree from a start path, checking for `.git` at each level. It accepts both `.git` directories (regular repos) and `.git` files (worktrees and submodules). The walk is memoized with an LRU cache (50 entries, keyed by `startPath`) to prevent redundant filesystem traversals during permission checks and prompt building.

`findCanonicalGitRoot` resolves through worktrees to find the main repository's working directory. For a worktree, it:
1. Reads `.git` file: `gitdir: .claude/worktrees/slug/.git` format.
2. Reads `commondir` from the worktree git dir.
3. **Security validation**: Checks that `worktreeGitDir` is a direct child of `<commonDir>/worktrees/` (prevents symlink attacks from cloned repos).
4. Reads the `gitdir` backlink and verifies it points to `<gitRoot>/.git` (prevents borrowing a victim's existing worktree entry).
5. Returns `dirname(commonDir)` as the canonical root.

---

### Worktree Integration

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`  
**Utility**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/worktree.ts`

Worktrees give agents (and interactive sessions) an isolated copy of the repository to work in without affecting the main branch.

**EnterWorktreeTool execution flow**:
```
1. Validate not already in a worktree session
2. findCanonicalGitRoot(getCwd()) -> ensure we're at main repo root
3. createWorktreeForSession(sessionId, slug)
4. process.chdir(worktreePath), setCwd(worktreePath), setOriginalCwd(getCwd())
5. saveWorktreeState(worktreeSession)
6. clearSystemPromptSections() -- force env_info_simple recompute with worktree context
7. clearMemoryFileCaches() -- CWD-dependent caches
8. getPlansDirectory.cache.clear()
```

**Slug validation** (`validateWorktreeSlug`):
- Max 64 characters
- Each `/`-separated segment: letters, digits, dots, underscores, dashes; no empty segments
- No `.` or `..` path segments (path traversal protection)
- The validation uses semantic checks rather than relying solely on `path.join`'s normalization

**Worktree creation** (`getOrCreateWorktree`):
- Fast resume path: if `.git` pointer file exists at `worktreesDir/<slug>`, reads HEAD SHA directly (no subprocess, no 15ms spawn overhead).
- New worktree: fetches `origin/<defaultBranch>` (skipped if the ref already exists locally — avoids 6-8s scan in large repos), then runs `git worktree add`.
- For PR-based worktrees: fetches `pull/<prNumber>/head` before creation.

**Large directory symlinking** (`symlinkLargeDirectories`):
When creating a worktree, directories configured as "large" in `settings.worktree.symlinkDirs` (e.g., `node_modules`, `.venv`) are symlinked from the main repo into the worktree to avoid disk bloat.

**ExitWorktreeTool**: Reverses the process. It removes the worktree (via `git worktree remove --force`), restores `cwd` to the main repo root, clears caches, and saves the exit state.

---

### Commit Attribution Tracking

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/commitAttribution.ts`

The attribution system tracks how much of each committed file was written by Claude vs humans. This data is stored in git notes and included as a commit trailer.

**AttributionState**:
```typescript
{
  fileStates: Map<relativePath, { contentHash, claudeContribution, mtime }>
  sessionBaselines: Map<relativePath, { contentHash, mtime }>
  surface: string                    // e.g., "cli"
  startingHeadSha: string | null
  promptCount: number
  promptCountAtLastCommit: number
  permissionPromptCount: number
  escapeCount: number
}
```

**Contribution calculation** (`computeFileModificationState`):
For each edit, Claude's character contribution is computed by finding the changed region:
1. Find common prefix length between old and new content.
2. Find common suffix length.
3. `claudeContribution = max(oldChangedLen, newChangedLen)`.

This handles same-length replacements (e.g., case fixes: "Esc" -> "esc") that `Math.abs(newLen - oldLen)` would miss.

**Attribution aggregation** (`calculateCommitAttribution`):
Called at commit time. For each staged file:
- If tracked in `fileStates`: Claude contribution is the accumulated `claudeContribution`.
- If not tracked but in `sessionBaselines`: human modification (uses `git diff --stat` to estimate chars).
- If neither tracked: file is new/human-created.
- Generated files (per `isGeneratedFile`) are excluded.

**Snapshot persistence** (`stateToSnapshotMessage`):
Attribution state is persisted as `attribution-snapshot` messages in the session JSONL log. On resume, `restoreAttributionStateFromSnapshots` uses only the **last** snapshot (not sum all), because snapshots are full-state dumps, not deltas. Summing would cause quadratic growth for long sessions (837 snapshots × 280 files example cited in code).

---

### Undercover Mode for Public Repos

**Source**: `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/commitAttribution.ts` (lines 30–76)

Claude Code avoids leaking internal model codenames in git commit trailers for public repositories. The `isInternalModelRepo` function checks the repository's remote URL against a hardcoded allowlist of private Anthropic repositories.

```typescript
const INTERNAL_MODEL_REPOS = [
  'github.com:anthropics/claude-cli-internal',
  'github.com/anthropics/claude-cli-internal',
  'github.com:anthropics/anthropic',
  // ... ~20 more private repos
]
```

Key design notes from the comment (lines 27–34):
- This is a **repo allowlist**, not an org-wide check.
- The `anthropics` and `anthropic-experimental` GitHub orgs contain PUBLIC repos (including `anthropics/claude-code` itself and `anthropic-experimental/sandbox-runtime`).
- Undercover mode stays ON in those public repos to prevent internal codename leaks.
- Only confirmed-private repos are added to this list.

**`sanitizeModelName`** maps internal variants to their public equivalents (e.g., `opus-4-6` → `claude-opus-4-6`). This is applied to commit trailers so public repos see standard model names.

---

### Gitignore Filtering

Gitignore-style filtering appears in multiple layers of the system:

1. **ripgrep's built-in `.gitignore` support**: By default, `CLAUDE_CODE_GLOB_NO_IGNORE='true'` causes ripgrep to ignore `.gitignore` files during glob. Set to `false` to respect gitignore (e.g., for projects that intentionally ignore build artifacts).

2. **GrepTool VCS exclusion**: The six VCS directories (`.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl`) are always excluded via explicit `--glob !.git` flags.

3. **Permission-based ignore patterns** (`getFileReadIgnorePatterns` in `filesystem.ts`): The `ignore` npm package (a gitignore-spec-compliant implementation) is used to match deny rules against paths. Rule patterns use `/` as separator per gitignore spec.

4. **Orphaned plugin cache exclusion** (`getGlobExclusionsForPluginCache`): Plugin version directories that are no longer in use are excluded from both Glob and Grep results to prevent stale plugin indexes from cluttering search results.

5. **DANGEROUS_FILES protection** (`filesystem.ts`): A set of dangerous files (`.gitconfig`, `.gitmodules`, `.bashrc`, `.zshrc`, `.profile`, `.ssh/authorized_keys`, etc.) are protected from auto-editing. These files can be used for code execution or data exfiltration.

---

## Cross-Cutting Concerns

### LSP Integration

Both FileEditTool and FileWriteTool notify the LSP server manager on every write:
- `lspManager.changeFile(absoluteFilePath, newContent)` — sends `textDocument/didChange`
- `lspManager.saveFile(absoluteFilePath)` — sends `textDocument/didSave` (triggers TypeScript diagnostic refresh)

Previously-delivered diagnostics are cleared before notifying so new diagnostics replace old ones rather than accumulating.

### VSCode Diff View Integration

`notifyVscodeFileUpdated(absoluteFilePath, originalContent, newContent)` is called after every write. This feeds Claude Code's VSCode extension integration, which shows diff views for agent edits.

### Skill Discovery

FileEditTool and FileWriteTool both trigger skill discovery on the edited file path:
```
discoverSkillDirsForPaths([absoluteFilePath], cwd)  -- find matching skill dirs
addSkillDirectories(newSkillDirs)                    -- async, non-blocking
activateConditionalSkillsForPaths([absoluteFilePath], cwd) -- activate path-pattern skills
```
This fire-and-forget pattern (not awaited) ensures skill loading does not block the edit path.

### Permissions Architecture

All file tools use a two-phase permission check:
1. `validateInput()` — lightweight pre-check (path existence, file type, staleness). Runs before showing the permission prompt.
2. `checkPermissions()` — calls `checkWritePermissionForTool` or `checkReadPermissionForTool`, consulting `ToolPermissionContext` for allow/deny rules.

Permission rules use gitignore-style matching via the `ignore` package, and wildcard matching via `matchWildcardPattern` from `shellRuleMatching.ts`.

### Windows UNC Path Protection

All file-touching tools include a guard:
```typescript
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }  // Skip filesystem operations, let permission check handle
}
```
This prevents Windows `fs.existsSync()` calls on UNC paths (`\\server\share\...`) from triggering SMB authentication, which could leak NTLM credentials to a malicious server.

### Team Memory Secret Guard

Both FileEditTool and FileWriteTool call `checkTeamMemSecrets(fullFilePath, newContent)` before validation. This blocks writes to team memory files that contain detected secrets (API keys, tokens, etc.), preventing accidental secret storage in shared memory.

---

## Architecture Insights and Observations

### Strengths

1. **Defense-in-depth security model**: The bash security system has three layers: pre-checks (regex patterns on raw command), tree-sitter AST analysis (structural), and per-validator checks (semantic). These are additive; a bypass of one layer is caught by another.

2. **Atomic file operations**: The comment "Please avoid async operations between here and writing to disk to preserve atomicity" (FileEditTool.call line 443) is enforced by the code structure. The staleness check and write happen in a synchronous sequence with no yield points between them.

3. **Explicit failure modes with error codes**: FileEditTool's `validateInput` returns numeric `errorCode` values (0–10) alongside human-readable messages. This enables programmatic handling and precise analytics logging.

4. **Content-based staleness fallback on Windows**: Rather than rejecting all mtime-changed files on Windows (which would cause constant friction with cloud sync tools), the system falls back to content comparison for full reads. This is a significant usability improvement that required careful implementation to avoid TOCTOU races.

5. **Idempotent v1 backups**: The content-hash-keyed v1 backup in FileHistory means that re-running an identical edit (from a retry or duplication) does not produce redundant backup files. Only v2+ backups are time-ordered.

### Issues and Observations

1. **Legacy `splitCommand_DEPRECATED`**: Many call sites still use the deprecated shell-quote-based command splitter instead of the tree-sitter AST parser. This is a migration in progress; the AST path is gated on `parseForSecurity()` returning `kind: 'simple'`.

2. **`BASH_SECURITY_CHECK_IDS` fragility**: Security check IDs are numeric constants used for analytics. Adding new checks between existing ones would renumber everything. The comment says "Append new entries at the end to keep IDs stable" but this creates a non-obvious ordering requirement.

3. **GlobTool uses ripgrep, not fast-glob**: The tool description says it wraps "fast-glob" but the implementation uses ripgrep (`--files` mode). This is actually a better choice for performance and `.gitignore` integration, but the discrepancy in documentation (the tool's own prompt says "fast-glob") could confuse contributors.

4. **Attribution state snapshot accumulation bug (fixed)**: The code comment at line 906 of `commitAttribution.ts` explicitly documents a bug that was fixed: "Snapshots are full-state dumps... The last snapshot has the most recent count for every path — fileStates never shrinks. Iterating and SUMMING counts across snapshots causes quadratic growth on restore." The fix uses only the last snapshot.

5. **Undercover mode hardcoded allowlist**: The `INTERNAL_MODEL_REPOS` list requires a code change to add or remove repos. There is no runtime mechanism to update this list, meaning any new private repo that needs internal model names in trailers requires a release.

6. **BashTool complexity ceiling**: `bashToolHasPermission` in `bashPermissions.ts` is noted to be "right at the limit" of Bun's DCE (Dead Code Elimination) complexity budget. Import aliases had to be changed to top-level const rebindings to prevent Bun from incorrectly evaluating `feature('BASH_CLASSIFIER')` as false and silently dropping classifier checks.

---

## Essential Files Reference

The following files are the most important for understanding the File Operations, Shell Execution, and Version Control domains:

### File Operations
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/fileStateCache.ts` — FileStateCache LRU with staleness tracking
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/fileRead.ts` — Encoding detection, CRLF normalization, `readFileSyncWithMetadata`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileEditTool/FileEditTool.ts` — String replacement with validation pipeline
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileEditTool/utils.ts` — Quote normalization, patch generation, desanitization
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileWriteTool/FileWriteTool.ts` — Full-file replacement
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileReadTool/FileReadTool.ts` — Multi-type reader (text, image, PDF, notebook)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/FileReadTool/imageProcessor.ts` — Sharp/native image processor abstraction
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/fileHistory.ts` — Checkpoint-based undo with three-phase commit
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/GrepTool/GrepTool.ts` — ripgrep wrapper with 3 output modes
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/glob.ts` — GlobTool's ripgrep-based file discovery

### Shell Execution
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/BashTool.tsx` — Orchestration, schema, large-output handling
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/bashSecurity.ts` — 23 security validators, quote extraction, safe heredoc
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/bashPermissions.ts` — Permission evaluation, compound command splitting, classifier integration
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/bash/ast.ts` — Tree-sitter AST parser, fail-closed allowlist
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/bash/treeSitterAnalysis.ts` — Quote span collection, compound structure analysis
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/sedEditParser.ts` — sed command parser and BRE-to-ERE conversion
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/commandSemantics.ts` — Exit code classification per command
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/BashTool/shouldUseSandbox.ts` — Sandbox decision logic
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/sandbox/sandbox-adapter.ts` — Claude settings to SandboxRuntimeConfig translation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/Shell.ts` — Shell provider selection (bash/zsh/PowerShell)

### Version Control
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/git.ts` — Root discovery, worktree resolution with security validation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/commitAttribution.ts` — Attribution tracking, undercover mode, snapshot persistence
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/worktree.ts` — Worktree creation, slug validation, symlink optimization
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` — Worktree session activation
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` — Worktree session teardown
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/filesystem.ts` — Path validation, gitignore filtering, dangerous file protection