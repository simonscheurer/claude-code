# Feature Analysis: Extension & Integration Systems

**Domain Cluster 04 — Plugin System, Skill System, Hook System, MCP Integration**

---

## Table of Contents

1. [Plugin System](#1-plugin-system)
   1.1 [Loading Pipeline Overview](#11-loading-pipeline-overview)
   1.2 [Plugin Directory Structure and Manifest Schema](#12-plugin-directory-structure-and-manifest-schema)
   1.3 [Plugin Source Types and Installation Methods](#13-plugin-source-types-and-installation-methods)
   1.4 [Plugin Components: What a Plugin Can Provide](#14-plugin-components-what-a-plugin-can-provide)
   1.5 [Marketplace System](#15-marketplace-system)
   1.6 [Built-in Plugin Registration and Toggling](#16-built-in-plugin-registration-and-toggling)
   1.7 [Enterprise Policy Controls](#17-enterprise-policy-controls)
   1.8 [Cache Architecture and ZIP Cache](#18-cache-architecture-and-zip-cache)
   1.9 [Dependency Resolution](#19-dependency-resolution)
   1.10 [User Configuration (userConfig and Secure Storage)](#110-user-configuration-userconfig-and-secure-storage)

2. [Skill System](#2-skill-system)
   2.1 [Discovery Sources and Priority](#21-discovery-sources-and-priority)
   2.2 [Frontmatter Fields and Their Effects](#22-frontmatter-fields-and-their-effects)
   2.3 [Bundled Skills](#23-bundled-skills)
   2.4 [SkillTool Execution: Fork vs Inline](#24-skilltool-execution-fork-vs-inline)
   2.5 [Skill-as-Command Wrapping](#25-skill-as-command-wrapping)

3. [Hook System](#3-hook-system)
   3.1 [All 27 Hook Events](#31-all-27-hook-events)
   3.2 [Four Hook Types](#32-four-hook-types)
   3.3 [Hook Output Protocol (JSON stdout)](#33-hook-output-protocol-json-stdout)
   3.4 [Async and Background Hooks](#34-async-and-background-hooks)
   3.5 [Environment Variables Available to Hooks](#35-environment-variables-available-to-hooks)
   3.6 [Hook Execution Flow Diagram](#36-hook-execution-flow-diagram)
   3.7 [Trust and Policy Guards](#37-trust-and-policy-guards)

4. [MCP Integration](#4-mcp-integration)
   4.1 [Transport Types and Connection Strategy](#41-transport-types-and-connection-strategy)
   4.2 [Connection Lifecycle and Memoization](#42-connection-lifecycle-and-memoization)
   4.3 [MCP Tools as Claude Code Tools (MCPTool Wrapper)](#43-mcp-tools-as-claude-code-tools-mcptool-wrapper)
   4.4 [OAuth Flow for MCP Servers](#44-oauth-flow-for-mcp-servers)
   4.5 [Elicitation Protocol](#45-elicitation-protocol)
   4.6 [Agent-Scoped MCP Servers](#46-agent-scoped-mcp-servers)
   4.7 [Config Assembly and Deduplication](#47-config-assembly-and-deduplication)
   4.8 [In-Process MCP Servers](#48-in-process-mcp-servers)

5. [Cross-Cutting Concerns](#5-cross-cutting-concerns)
6. [Architecture Diagrams](#6-architecture-diagrams)
7. [Key Files Reference](#7-key-files-reference)
8. [Observations and Improvement Opportunities](#8-observations-and-improvement-opportunities)

---

## 1. Plugin System

### 1.1 Loading Pipeline Overview

The plugin loading pipeline has two modes that share the same discovery and policy logic but differ in whether they hit the network:

- `loadAllPlugins` — full mode, performs git clone / npm install / network fetch if the plugin is not cached
- `loadAllPluginsCacheOnly` — fast startup mode, reads only from the on-disk cache recorded in `installed_plugins.json`

Both are memoized with lodash `memoize`. When `loadAllPlugins` completes, it warms the `loadAllPluginsCacheOnly` cache so that downstream consumers (MCP config build, command listing) never see a cache miss after an explicit refresh.

The shared `assemblePluginLoadResult` function orchestrates the pipeline:

**Entry point:** `src/utils/plugins/pluginLoader.ts:3155` — `assemblePluginLoadResult()`

```
assemblePluginLoadResult(marketplaceLoader)
  ├─ Parallel load:
  │   ├─ marketplaceLoader()         → loadPluginsFromMarketplaces({ cacheOnly })
  │   └─ loadSessionOnlyPlugins()    → --plugin-dir / SDK inline plugins
  ├─ getBuiltinPlugins()             → built-in plugins from BUILTIN_PLUGINS map
  ├─ mergePluginSources()            → session overrides marketplace; managed locks override all
  ├─ verifyAndDemote()               → disable plugins with unsatisfied deps
  └─ cachePluginSettings()           → push plugin settings into settings cascade
```

**`loadPluginsFromMarketplaces`** (`src/utils/plugins/pluginLoader.ts:1888`) runs per-plugin logic:

```
loadPluginsFromMarketplaces({ cacheOnly })
  ├─ Read settings.enabledPlugins (merged with --add-dir sources)
  ├─ Load known_marketplaces.json → loadKnownMarketplacesConfigSafe()
  ├─ Enterprise policy gate:
  │   ├─ strictKnownMarketplaces (allowlist) check
  │   └─ blockedMarketplaces (blocklist) check
  ├─ Pre-load marketplace catalogs (one read per unique marketplace)
  ├─ Promise.allSettled(per-plugin):
  │   ├─ Resolve entry from pre-loaded catalog
  │   └─ cacheOnly ? loadPluginFromMarketplaceEntryCacheOnly()
  │                : loadPluginFromMarketplaceEntry()
  └─ Collect errors (policy block, not-found, rejected promises)
```

### 1.2 Plugin Directory Structure and Manifest Schema

A plugin is a directory with a conventional layout. The manifest is read from `.claude-plugin/plugin.json` (preferred) or legacy `plugin.json` at the root.

**Standard layout:**

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Manifest (preferred location)
├── commands/                # Slash commands (.md files or SKILL.md directories)
├── agents/                  # Agent definitions (.md files)
├── skills/                  # Skills (directories with SKILL.md)
├── hooks/
│   └── hooks.json           # Hook definitions (PluginHooksSchema)
├── output-styles/           # Output style definitions
├── .mcp.json                # MCP server configs (lowest priority)
└── settings.json            # Plugin-provided settings (only 'agent' key allowed)
```

**Manifest schema** (`src/utils/plugins/schemas.ts:884` — `PluginManifestSchema`):

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `string` (required, no spaces) | Plugin identifier for namespacing |
| `version` | `string` | Semver version |
| `description` | `string` | Human-readable description |
| `author` | `{ name, email?, url? }` | Plugin author info |
| `keywords` | `string[]` | Discovery tags |
| `dependencies` | `DependencyRef[]` | Required plugins (bare names resolve against same marketplace) |
| `commands` | path / path[] / record | Extra command sources beyond `commands/` |
| `agents` | path / path[] | Extra agent sources beyond `agents/` |
| `skills` | path / path[] | Extra skill directories beyond `skills/` |
| `outputStyles` | path / path[] | Extra output-style sources beyond `output-styles/` |
| `hooks` | path / HooksObject / array | Additional hooks (merged with `hooks/hooks.json`) |
| `mcpServers` | JSON-path / MCPB-path / inline-record / array | MCP server configurations |
| `lspServers` | JSON-path / inline-record / array | LSP server configurations |
| `userConfig` | record of config option schemas | User-configurable values prompted at install |
| `channels` | channel definition array | MCP channels for messaging (Telegram, Slack, etc.) |
| `settings` | record | Settings to merge into session; only `agent` key is kept |

Unknown top-level fields are stripped silently (Zod strip behavior). Nested config objects (`userConfig` options, `channels`) use strict validation.

**`createPluginFromPath`** (`src/utils/plugins/pluginLoader.ts:1348`) is the central assembly function:

1. Load manifest from `.claude-plugin/plugin.json` → `loadPluginManifest()`
2. Build base `LoadedPlugin` object with path, source, enabled state
3. Auto-detect `commands/`, `agents/`, `skills/`, `output-styles/` directories in parallel
4. Process manifest `commands` field (path, array, or object-mapping format)
5. Process manifest `agents`, `skills`, `outputStyles` paths
6. Load hooks from `hooks/hooks.json` then merge manifest `hooks` declarations
7. Load plugin settings via `loadPluginSettings()` → allowlist filter to `agent` key only
8. Return `{ plugin, errors }`

### 1.3 Plugin Source Types and Installation Methods

Plugin sources are defined by `PluginSource` / `MarketplaceSourceSchema` (`src/utils/plugins/schemas.ts:906`):

| Source Type | Schema `source` field | Installation Method |
|-------------|----------------------|---------------------|
| GitHub repo | `'github'` | `installFromGitHub()` → SSH or HTTPS git clone |
| Git URL | `'git'` | `installFromGit()` → `gitClone()` |
| Git subdirectory | `'git-subdir'` | `installFromGitSubdir()` → sparse-checkout |
| NPM package | `'npm'` | `installFromNpm()` → npm install to global cache |
| Local path | string (no `source` key) | `installFromLocal()` → `copyDir()` |
| URL (marketplace) | `'url'` | HTTP fetch via axios |
| File | `'file'` | Local JSON file read |
| Directory | `'directory'` | Local directory read |

**`cachePlugin`** (`src/utils/plugins/pluginLoader.ts:911`) is the main installation entry point:

1. Create temp directory `~/.claude/plugins/cache/temp_{prefix}_{ts}_{rand}`
2. Dispatch to appropriate installer based on source type
3. Read manifest from `.claude-plugin/plugin.json` or legacy `plugin.json`
4. Rename temp dir to `{cachePath}/{manifest.name}`
5. Return `{ path, manifest, gitCommitSha? }`

**`copyPluginToVersionedCache`** (`src/utils/plugins/pluginLoader.ts:365`) handles versioned storage:

- Versioned path format: `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`
- Sanitizes marketplace and plugin names to `[a-zA-Z0-9\-_]` only (path traversal protection)
- Checks ZIP cache first when `isPluginZipCacheEnabled()`
- Falls back through: versioned path → seed directories → fresh copy
- Removes `.git` directory from cached copy

**`installFromGitSubdir`** (`src/utils/plugins/pluginLoader.ts:718`) uses git sparse-checkout for monorepos:

1. Clone with `--filter=tree:0 --no-checkout` (lazy blobs)
2. `sparse-checkout set --cone -- <subdirPath>` (cone mode, git >= 2.25 required)
3. Checkout HEAD (materializes only the subdirectory files)
4. Capture resolved SHA via `rev-parse HEAD`
5. `rename(cloneDir/subdirPath, targetPath)` — atomic extraction
6. Remove the clone directory (ephemeral)

### 1.4 Plugin Components: What a Plugin Can Provide

The `LoadedPlugin` type (`src/types/plugin.ts:48`) captures all components discovered from a plugin:

| Field | Component Type | Description |
|-------|---------------|-------------|
| `commandsPath` | Commands | Path to `commands/` directory |
| `commandsPaths` | Commands | Additional command paths from manifest |
| `commandsMetadata` | Commands | Object-mapping format: `{ "name": { source, description, ... } }` |
| `agentsPath` | Agents | Path to `agents/` directory |
| `agentsPaths` | Agents | Additional agent paths from manifest |
| `skillsPath` | Skills | Path to `skills/` directory |
| `skillsPaths` | Skills | Additional skill paths from manifest |
| `outputStylesPath` | Output Styles | Path to `output-styles/` directory |
| `outputStylesPaths` | Output Styles | Additional output-style paths |
| `hooksConfig` | Hooks | Merged `HooksSettings` (from `hooks/hooks.json` + manifest) |
| `mcpServers` | MCP | `Record<string, McpServerConfig>` loaded at plugin load time |
| `lspServers` | LSP | `Record<string, LspServerConfig>` |
| `settings` | Settings | Allowlisted settings (currently only `agent` key) |
| `isBuiltin` | Meta | `true` for CLI-shipped plugins managed in the /plugin UI |

**Command namespace format:** `{pluginName}:{commandName}` — e.g., a file `commands/deploy.md` in plugin `devops` becomes `/devops:deploy`. Nested directories add extra colon-separated segments: `commands/docker/build.md` → `/devops:docker:build`.

**Object-mapping commands** (`commandsMetadata`) allow inline markdown content without a file:
```json
{
  "commands": {
    "about": { "content": "## About this plugin\n...", "description": "Plugin info" },
    "build": { "source": "./commands/build.md", "model": "claude-opus-4-5" }
  }
}
```

**MCPB support** (`src/utils/plugins/mcpbHandler.ts`): Plugins can declare `mcpServers` as a path to a `.mcpb` or `.dxt` file (a DXT bundle). The MCPB handler downloads, extracts, converts the DXT manifest to an MCP config, and handles user configuration prompting.

**LSP servers** (`src/utils/plugins/lspPluginIntegration.ts`): Plugins can declare Language Server Protocol servers via `lspServers` in the manifest or a `.lsp.json` file. Each server declares `command`, `args`, `extensionToLanguage` mapping, transport (stdio or socket), and optional settings.

### 1.5 Marketplace System

The marketplace is a two-layer system: **intent** (settings) and **state** (filesystem cache).

**File:** `src/utils/plugins/marketplaceManager.ts`

**Intent layer — `extraKnownMarketplaces` in settings.json:**
```json
{
  "extraKnownMarketplaces": {
    "my-company": {
      "source": { "source": "github", "repo": "mycompany/claude-plugins" },
      "autoUpdate": true
    }
  }
}
```

**State layer — `~/.claude/plugins/known_marketplaces.json`:**
```json
{
  "my-company": {
    "source": { "source": "github", "repo": "mycompany/claude-plugins" },
    "installLocation": "/Users/me/.claude/plugins/marketplaces/my-company",
    "lastUpdated": "2024-01-15T10:30:00.000Z"
  }
}
```

**Marketplace source types** (all valid `source` field values in `MarketplaceSourceSchema`):

| Type | Value | Description |
|------|-------|-------------|
| URL | `'url'` | Direct HTTPS URL to `marketplace.json` |
| GitHub | `'github'` | GitHub repo with optional `ref`, `path`, `sparsePaths` |
| Git | `'git'` | Arbitrary git URL |
| NPM | `'npm'` | NPM package containing `marketplace.json` |
| File | `'file'` | Local file path to `marketplace.json` |
| Directory | `'directory'` | Local directory with `.claude-plugin/marketplace.json` |
| Settings | `'settings'` | Inline marketplace in settings.json (synthetic cache) |
| Host pattern | `'hostPattern'` | Policy allowlist: regex matching source hostname |
| Path pattern | `'pathPattern'` | Policy allowlist: regex matching local path |

**Marketplace loading flow** (`loadAndCacheMarketplace`):

1. Resolve declared marketplace intent from merged settings
2. Check state layer for existing installation and version
3. If up-to-date → return cached path
4. Otherwise → fetch/clone, write to `marketplaces/{name}/` or `marketplaces/{name}.json`
5. Update `known_marketplaces.json` with new `lastUpdated`

**Official marketplace** (`src/utils/plugins/officialMarketplace.ts`): The `claude-code-marketplace` (or `claude-plugins-official`) marketplace is implicitly declared when any enabled plugin references it. It can also be fetched from GCS for faster cold-start (`officialMarketplaceGcs.ts`).

**Name protection** (`src/utils/plugins/schemas.ts:19-101`):
- `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`: set of reserved names (e.g., `claude-code-marketplace`, `anthropic-plugins`)
- `BLOCKED_OFFICIAL_NAME_PATTERN`: regex blocking impersonation attempts
- `OFFICIAL_GITHUB_ORG = 'anthropics'`: reserved names must come from this org
- Non-ASCII characters are blocked (homograph attack prevention)

**Auto-update logic** (`isMarketplaceAutoUpdate`): official marketplaces auto-update by default; third-party marketplaces require explicit `autoUpdate: true`. The `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES` set excludes `knowledge-work-plugins` from the auto-update default.

### 1.6 Built-in Plugin Registration and Toggling

**File:** `src/plugins/builtinPlugins.ts`

Built-in plugins are registered programmatically at startup into a module-level `BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition>`.

```typescript
// At startup (initBuiltinPlugins() equivalent):
registerBuiltinPlugin({
  name: 'my-builtin',
  description: 'Does something useful',
  defaultEnabled: true,
  skills: [{ name: 'my-skill', ... }],
  hooks: { PreToolUse: [{ matcher: 'Bash', hooks: [{ type: 'command', command: '...' }] }] },
  mcpServers: { 'my-server': { command: '...', args: [] } },
})
```

Plugin IDs use format `{name}@builtin` — `'my-builtin@builtin'`.

**Enable/disable state** (`getBuiltinPlugins()`):
1. Read `settings.enabledPlugins["{name}@builtin"]`
2. If set to `true`/`false`, use that
3. Otherwise use `definition.defaultEnabled ?? true`
4. Check `definition.isAvailable?.()` — unavailable plugins are hidden entirely
5. Return `{ enabled: LoadedPlugin[], disabled: LoadedPlugin[] }`

The `LoadedPlugin` for built-in plugins uses `path: 'builtin'` as a sentinel (no filesystem path). Skills from built-in plugins use `source: 'bundled'` and `loadedFrom: 'bundled'` to integrate with the standard skill listing and analytics.

### 1.7 Enterprise Policy Controls

**File:** `src/utils/plugins/pluginPolicy.ts` and `src/utils/plugins/marketplaceHelpers.ts`

**Policy settings** come from `managed-settings.json` (MDM-deployed) at `policySettings` source in the settings cascade.

**Plugin-level policy:**

| Policy Setting | Effect |
|---------------|--------|
| `enabledPlugins["myplugin@mymarket"] = false` | Forces plugin disabled; user cannot enable |
| `strictKnownMarketplaces: [...]` | Allowlist; only these marketplace sources are permitted |
| `blockedMarketplaces: [...]` | Blocklist; these marketplace sources are denied |
| `allowManagedHooksOnly: true` | Only hooks from managed settings are executed |
| `disableAllHooks: true` | No hooks execute at all |
| `allowedHttpHookUrls: [...]` | Allowlist for HTTP hook URLs |

**`isPluginBlockedByPolicy(pluginId)`** (`src/utils/plugins/pluginPolicy.ts:17`): checks `policySettings.enabledPlugins[pluginId] === false`. This is the single gate used at install chokepoint, enable operation, and UI filters.

**`isSourceAllowedByPolicy(source)`** (`src/utils/plugins/marketplaceHelpers.ts`): validates a marketplace source against the enterprise allowlist/blocklist. Used during `loadPluginsFromMarketplaces`. Returns false if:
- `strictKnownMarketplaces` is set and source does not match any allowed entry
- `blockedMarketplaces` is set and source matches a blocked entry

**Fail-closed behavior:** if enterprise policy is active (`hasEnterprisePolicy`) and a plugin's marketplace cannot be resolved from `known_marketplaces.json`, the plugin is blocked rather than silently skipped (`src/utils/plugins/pluginLoader.ts:1982`).

**`shouldAllowManagedHooksOnly()`** and **`shouldDisableAllHooksIncludingManaged()`** (`src/utils/hooks/hooksConfigSnapshot.ts`) — runtime flags checked before every hook execution.

### 1.8 Cache Architecture and ZIP Cache

**Versioned cache path format:** `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`

All path components are sanitized to `[a-zA-Z0-9\-_]` before use in filesystem paths to prevent path traversal.

**Seed directories** (`getPluginSeedDirs()`, `src/utils/plugins/pluginDirectories.ts`): Organizations can pre-populate plugin caches via seed directories (e.g., in enterprise Docker images). `probeSeedCache()` checks seed directories before network fetch. `probeSeedCacheAnyVersion()` handles the first-boot case where the version is unknown.

**ZIP cache** (`src/utils/plugins/zipCache.ts`): When `CLAUDE_CODE_PLUGIN_ZIP_CACHE=1` is set, plugins are stored as `.zip` files instead of directories. This reduces inode count and improves startup time on systems with slow filesystem metadata operations. `convertDirectoryToZipInPlace()` and `extractZipToDirectory()` handle the format conversion.

**`loadAllPlugins.cache` and `loadAllPluginsCacheOnly.cache`** are both cleared by `clearPluginCache(reason?)` (`src/utils/plugins/pluginLoader.ts:3225`). Clearing also resets the settings cascade cache when plugins had contributed settings.

### 1.9 Dependency Resolution

**File:** `src/utils/plugins/dependencyResolver.ts`

Plugins declare dependencies via the `dependencies` field in `plugin.json`:
```json
{ "dependencies": [{ "name": "base-plugin", "marketplace": "my-marketplace" }] }
```

Bare names (no `@marketplace`) are resolved against the declaring plugin's own marketplace.

`verifyAndDemote(allPlugins)` runs after all plugins are loaded:
- Checks that each enabled plugin's declared dependencies exist in the enabled set
- Adds demoted plugin IDs to a `Set<string>`
- Returns `{ demoted, errors }` — demoted plugins have `enabled` set to `false`
- Does NOT write settings (demotion is session-local; user sees the issue in `/doctor`)

### 1.10 User Configuration (userConfig and Secure Storage)

Plugins declare configurable values in `manifest.userConfig`:

```json
{
  "userConfig": {
    "API_KEY": {
      "type": "string",
      "title": "API Key",
      "description": "Your service API key",
      "required": true,
      "sensitive": true
    },
    "BASE_URL": {
      "type": "string",
      "title": "Base URL",
      "default": "https://api.example.com"
    }
  }
}
```

**Key behaviors:**
- Non-sensitive values → `settings.json` under `pluginConfigs[pluginId].options`
- Sensitive values → macOS Keychain or `.credentials.json` (via `getSecureStorage()`)
- All values available in hooks as `${user_config.KEY}` substitutions
- All values available as env vars: `CLAUDE_PLUGIN_OPTION_{KEY}` (sanitized to `[A-Z0-9_]`)
- Non-sensitive values substituted into skill/agent markdown content
- The config schema is structurally compatible with `McpbUserConfigurationOption` from `@anthropic-ai/mcpb`

**Sensitive storage warning** (`schemas.ts:650`): sensitive values share a keychain entry with OAuth tokens. The total must stay under ~2KB (stdin-safe limit) to avoid INC-3028 class bugs.

---

## 2. Skill System

### 2.1 Discovery Sources and Priority

Skills are discovered from multiple sources, with later sources taking lower priority (deduplication by canonical real path):

**File:** `src/skills/loadSkillsDir.ts`

| Source | Directory | `source` field | `loadedFrom` field |
|--------|-----------|---------------|-------------------|
| Policy (MDM) | `{managedPath}/.claude/skills/` | `'policySettings'` | `'skills'` |
| User global | `~/.claude/skills/` | `'userSettings'` | `'skills'` |
| Project | `.claude/skills/` | `'projectSettings'` | `'skills'` |
| Plugin | `{pluginPath}/skills/` | plugin source | `'plugin'` |
| Bundled | Compiled into binary | `'bundled'` | `'bundled'` |
| MCP server | Via MCP prompts list | `'mcp'` | `'mcp'` |

The older `commands/` directory is also supported for backward compatibility (`loadedFrom: 'commands_DEPRECATED'`).

**Deduplication** is handled by `getFileIdentity()` which resolves symlinks to canonical real paths via `realpath()`. This prevents the same file from appearing twice if accessed through different symlink chains. Inode-based dedup was abandoned due to unreliable inode values on some filesystems (ExFAT, some NFS/container mounts report inode 0).

**`getProjectDirsUpToHome()`** walks from the current directory to home, loading skills from each `.claude/` directory found. This creates a directory-scoped inheritance hierarchy.

### 2.2 Frontmatter Fields and Their Effects

Skills are Markdown files with YAML frontmatter. Parsed by `parseFrontmatter()` in `src/utils/frontmatterParser.ts`, then processed by `parseSkillFrontmatterFields()` (`src/skills/loadSkillsDir.ts:185`):

| Frontmatter Key | Type | Effect |
|----------------|------|--------|
| `description` | `string` | Override auto-extracted description (first H1/H2 or first line) |
| `name` | `string` | Override display name (does not change the command key) |
| `argument-hint` | `string` | Hint shown in completions, e.g. `"[file]"` |
| `arguments` | `string` or `string[]` | Named argument slots for `$ARGUMENT_1`, `$ARGUMENT_2`, etc. |
| `when_to_use` | `string` | Guidance text for the model on when to invoke this skill |
| `allowed-tools` | `string[]` | Restrict which tools the skill can use |
| `model` | `string` or `'inherit'` | Override model; `'inherit'` uses session model |
| `disable-model-invocation` | `boolean` | Run as pure template expansion, no LLM call |
| `user-invocable` | `boolean` | If `false`, hidden from user but available to model |
| `context` | `'fork'` | Execute in forked subprocess (see section 2.4) |
| `agent` | `string` | Specify the agent type to use for execution |
| `effort` | `string` or `int` | Effort level for thinking tokens (see effort system) |
| `shell` | shell-type | Shell to use for any inline shell commands in the prompt |
| `version` | `string` | Version string (informational) |
| `paths` | `string` or `string[]` | File path patterns; skill only activates when CWD matches |
| `hooks` | `HooksSettings` | Hooks that fire during this skill's execution |

**`paths` frontmatter** restricts a skill to specific directory contexts using ignore-library glob patterns. A trailing `/**` is stripped (ignore handles subtree matching automatically). Skills with `paths: ["**"]` or empty patterns are treated as having no path restriction.

### 2.3 Bundled Skills

**File:** `src/skills/bundled/index.ts` — `initBundledSkills()`

Bundled skills are compiled into the CLI binary and registered via `registerBundledSkill()` in `src/skills/bundledSkills.ts`. They are always available regardless of user configuration.

**Currently registered bundled skills:**

| Skill | Module | Gating |
|-------|--------|--------|
| `update-config` | `updateConfig.ts` | Always on |
| `keybindings` | `keybindings.ts` | Always on |
| `verify` | `verify.ts` | Always on |
| `debug` | `debug.ts` | Always on |
| `lorem-ipsum` | `loremIpsum.ts` | Always on |
| `skillify` | `skillify.ts` | Always on |
| `remember` | `remember.ts` | Always on |
| `simplify` | `simplify.ts` | Always on |
| `batch` | `batch.ts` | Always on |
| `stuck` | `stuck.ts` | Always on |
| `dream` | `dream.ts` | `feature('KAIROS') \|\| feature('KAIROS_DREAM')` |
| `hunter` | `hunter.ts` | `feature('REVIEW_ARTIFACT')` |
| `loop` | `loop.ts` | `feature('AGENT_TRIGGERS')` |
| `schedule-remote-agents` | `scheduleRemoteAgents.ts` | `feature('AGENT_TRIGGERS_REMOTE')` |
| `claude-api` | `claudeApi.ts` | `feature('BUILDING_CLAUDE_APPS')` |
| `claude-in-chrome` | `claudeInChrome.ts` | `shouldAutoEnableClaudeInChrome()` |
| `run-skill-generator` | `runSkillGenerator.ts` | `feature('RUN_SKILL_GENERATOR')` |

**`BundledSkillDefinition`** (`src/skills/bundledSkills.ts:15`) provides:
- `name`, `description`, `aliases?`, `whenToUse?`, `argumentHint?`
- `allowedTools?`, `model?`, `disableModelInvocation?`
- `userInvocable?`, `isEnabled?: () => boolean`
- `hooks?: HooksSettings`
- `context?: 'inline' | 'fork'`
- `agent?: string`
- `files?: Record<string, string>` — reference files extracted to disk on first invocation
- `getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>`

When `files` is set, `registerBundledSkill` extracts them to `~/.claude/bundled-skills/{name}/` on first call and prepends `Base directory for this skill: {dir}\n\n` to the prompt (same contract as disk-based skills with `skillRoot`).

### 2.4 SkillTool Execution: Fork vs Inline

The `context` frontmatter field controls execution mode:

**Inline (default):** The skill's prompt is injected into the current conversation as a user message. The response uses the active session's tool permissions and model. No subprocess is spawned.

**Fork (`context: fork`):** The skill executes in a separate process with its own conversation context. The fork has:
- Independent message history
- Potentially different tool restrictions (`allowedTools`)
- Isolation from the parent session's state
- The result is returned as a tool-use response

**`getPromptForCommand`** in `createSkillCommand` (`src/skills/loadSkillsDir.ts:344`):
1. Prepend `Base directory for this skill: {baseDir}\n\n` if `baseDir` is set
2. Apply `substituteArguments()` for `$ARGUMENT_N` / named argument slots
3. Optionally call `executeShellCommandsInPrompt()` for inline shell expansions
4. Return as `ContentBlockParam[]`

### 2.5 Skill-as-Command Wrapping

Skill directories use a `SKILL.md` file convention. When `loadPluginMarkdown` encounters a directory containing `SKILL.md`, it treats the directory name as the command name rather than the filename. This allows skills to have associated reference files in the same directory that the model can `Read` during execution.

**`isSkillFile(filePath)`** — matches `/^skill\.md$/i` (case-insensitive) (`src/utils/plugins/loadPluginCommands.ts:53`).

**Name construction** for skill directories: `{pluginName}:{namespace}:{directoryName}` where namespace comes from parent directories relative to the skills root.

MCP skill builders (`src/skills/mcpSkillBuilders.ts`) create skill-like commands from MCP server prompts, bridging the MCP prompts capability into the skill system.

---

## 3. Hook System

### 3.1 All 27 Hook Events

**Source:** `src/entrypoints/sdk/coreTypes.ts:25` — `HOOK_EVENTS` constant array

| Event | When It Fires | Key Input Fields |
|-------|--------------|-----------------|
| `PreToolUse` | Before any tool is called | `tool_name`, `tool_input` |
| `PostToolUse` | After a tool succeeds | `tool_name`, `tool_input`, `tool_response` |
| `PostToolUseFailure` | After a tool throws an error | `tool_name`, `tool_input`, `error` |
| `Notification` | Model sends a notification | `message`, `notificationType` |
| `UserPromptSubmit` | User submits a message | `prompt` |
| `SessionStart` | Session begins | `session_id`, `cwd` |
| `SessionEnd` | Session ends (shutdown/clear) | `session_id`, `exit_reason` |
| `Stop` | Model stops responding | `stop_reason` |
| `StopFailure` | Model stop fails | `error` |
| `SubagentStart` | A sub-agent task begins | `agent_id`, `agent_type`, `prompt` |
| `SubagentStop` | A sub-agent task ends | `agent_id`, `result` |
| `PreCompact` | Before conversation compaction | `conversation_length` |
| `PostCompact` | After conversation compaction | `new_length` |
| `PermissionRequest` | Permission check initiated | `tool_name`, `tool_input` |
| `PermissionDenied` | Permission denied | `tool_name`, `tool_input`, `reason` |
| `Setup` | First hook before any prompts | `session_id`, `cwd` |
| `TeammateIdle` | Teammate agent goes idle | `teammate_id` |
| `TaskCreated` | A task is created | `task_id`, `description` |
| `TaskCompleted` | A task completes | `task_id`, `result` |
| `Elicitation` | MCP server requests user input | `server_name`, `params` |
| `ElicitationResult` | User responds to elicitation | `server_name`, `result` |
| `ConfigChange` | Settings change detected | `key`, `old_value`, `new_value` |
| `WorktreeCreate` | Git worktree created | `path`, `branch` |
| `WorktreeRemove` | Git worktree removed | `path` |
| `InstructionsLoaded` | CLAUDE.md files loaded | `paths`, `combined_length` |
| `CwdChanged` | Working directory changed | `old_cwd`, `new_cwd` |
| `FileChanged` | A watched file changes | `path`, `change_type` |

All events receive a base input from `createBaseHookInput()` (`src/utils/hooks.ts:301`):

```json
{
  "session_id": "<uuid>",
  "transcript_path": "<path>",
  "cwd": "<working-directory>",
  "permission_mode": "<mode>",
  "agent_id": "<id-if-subagent>",
  "agent_type": "<type-if-set>"
}
```

### 3.2 Four Hook Types

**Defined in `src/schemas/hooks.ts` via `buildHookSchemas()`:**

**1. Command hook (`type: 'command'`)**

Executes a shell command. The hook input JSON is passed via `stdin`.

```json
{
  "type": "command",
  "command": "python3 /path/to/hook.py",
  "if": "Bash(git *)",
  "shell": "bash",
  "timeout": 30,
  "async": false,
  "asyncRewake": false,
  "once": false,
  "statusMessage": "Running pre-commit check..."
}
```

**2. Prompt hook (`type: 'prompt'`)**

Sends the hook input to an LLM. Uses `$ARGUMENTS` placeholder in the prompt.

```json
{
  "type": "prompt",
  "prompt": "Review this tool call for security issues: $ARGUMENTS",
  "model": "claude-haiku-4-5",
  "timeout": 30,
  "once": false,
  "statusMessage": "Reviewing..."
}
```

Executed via `execPromptHook()` (`src/utils/hooks/execPromptHook.ts`). Uses the small fast model by default; `model` field overrides.

**3. Agent hook (`type: 'agent'`)**

Launches a multi-turn LLM agent to verify/inspect. Returns a structured JSON response via a synthetic output tool.

```json
{
  "type": "agent",
  "prompt": "Verify that the tests ran and passed. Use the provided tools to check. $ARGUMENTS",
  "model": "claude-haiku-4-5",
  "timeout": 60,
  "once": false
}
```

Executed via `execAgentHook()` (`src/utils/hooks/execAgentHook.ts`). Creates a full `query()` call with:
- A `StructuredOutputTool` enforcing the hook response schema
- Filtered tool set (disallows agent-spawning tools to prevent infinite recursion)
- Independent abort controller with per-hook timeout
- Separate transcript path from parent session

**4. HTTP hook (`type: 'http'`)**

POSTs hook input JSON to a URL. Response must be JSON.

```json
{
  "type": "http",
  "url": "https://internal.company.com/claude-hooks/pre-tool",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 30,
  "once": false
}
```

Executed via `execHttpHook()` (`src/utils/hooks/execHttpHook.ts`). Enforces:
- URL allowlist via `allowedHttpHookUrls` policy setting
- SSRF protection via `ssrfGuardedLookup()`
- Header env-var interpolation restricted to `allowedEnvVars` list
- CRLF injection prevention via `sanitizeHeaderValue()`
- Routes through sandbox proxy when sandboxing is enabled

**Shared fields for all hook types:**
- `if`: permission rule syntax filter (e.g., `"Bash(git *)"`) — short-circuits hook spawn
- `timeout`: seconds (positive number)
- `once`: if `true`, hook self-removes after first execution
- `statusMessage`: custom spinner text while hook runs

### 3.3 Hook Output Protocol (JSON stdout)

Command hooks output JSON to stdout to control behavior. If stdout doesn't start with `{`, it's treated as plain text and displayed as a system message.

**Full JSON output schema** (`hookJSONOutputSchema` in `src/types/hooks.ts`):

```json
{
  "continue": false,
  "suppressOutput": true,
  "stopReason": "Validation failed",
  "decision": "approve",
  "reason": "All checks passed",
  "systemMessage": "Additional context for the model",
  "permissionDecision": "allow",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Dangerous file path",
    "updatedInput": { "command": "echo safe" }
  }
}
```

**Sync vs Async output** (`isSyncHookJSONOutput`, `isAsyncHookJSONOutput`):
- Sync: processed immediately, can block/modify the tool call
- Async: hook spawned in background, response processed when it exits

**`processHookJSONOutput`** (`src/utils/hooks.ts:489`) processes sync responses:

| JSON Field | Effect |
|-----------|--------|
| `continue: false` | Sets `preventContinuation = true`; stops the agent turn |
| `stopReason` | Message shown when continuation is prevented |
| `decision: 'approve'` | Sets `permissionBehavior = 'allow'` |
| `decision: 'block'` | Sets `permissionBehavior = 'deny'`; creates blocking error |
| `systemMessage` | Injected as system message into next turn |
| `hookSpecificOutput.hookEventName` | Validates event name matches expected |

**Per-event `hookSpecificOutput` fields:**

| Event | Specific Fields |
|-------|----------------|
| `PreToolUse` | `permissionDecision` (allow/deny/ask), `permissionDecisionReason`, `updatedInput` |
| `UserPromptSubmit` | `additionalContext` (string injected before user message) |
| `SessionStart` | `additionalContext`, `initialUserMessage`, `watchPaths` |
| `Setup` | `additionalContext` |
| `SubagentStart` | `additionalContext` |
| `PostToolUse` | `additionalContext`, `updatedMCPToolOutput` |
| `PostToolUseFailure` | `additionalContext` |
| `PermissionDenied` | `retry` (boolean, retry the permission request) |
| `PermissionRequest` | `decision: { behavior, updatedInput? }` |
| `Elicitation` | `action` (accept/decline/cancel), `content` |
| `ElicitationResult` | `action`, `content` |

HTTP hooks use `parseHttpHookOutput()` — empty body is treated as `{}` (success, no-op); non-JSON body is an error.

### 3.4 Async and Background Hooks

**Async hooks** (`async: true` in hook config):

When a hook returns `{ "type": "async" }` in its JSON output (or is configured with `async: true`), it is registered in `AsyncHookRegistry` (`src/utils/hooks/AsyncHookRegistry.ts`) and executed in the background. The registry tracks pending hooks. When they complete, results are processed.

**`asyncRewake: true`** hooks:

A special variant that bypasses the registry. When the hook process exits:
- Exit code 0: success, no action
- Exit code 2: "blocking error" — enqueues a `task-notification` via `enqueuePendingNotification()`. This wakes the model mid-conversation (while idle) or gets injected via `queued_command` attachments (while busy)

The `asyncRewake` path deliberately does NOT call `shellCommand.background()` — it keeps in-memory stdout/stderr capture instead of spilling to disk, so `getStderr()` returns the actual error content for the notification message.

**`executeInBackground`** (`src/utils/hooks.ts:184`):

```
executeInBackground({ asyncRewake: true })
  → shellCommand.result.then(result => {
      if (result.code === 2) {
        enqueuePendingNotification({
          value: wrapInSystemReminder("Stop hook blocking error from..."),
          mode: 'task-notification'
        })
      }
    })

executeInBackground({ asyncRewake: false })
  → shellCommand.background(processId)   // spills stdout/stderr to disk
  → registerPendingAsyncHook(...)        // registry tracking
```

**SessionEnd hook timeout:** `SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500ms`. Overridable via `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`. SessionEnd hooks run in parallel with this as both the per-hook and overall cap.

**`once: true` hooks** self-remove from the hook registration after the first execution. Implemented by tracking in the hook runner and calling `clearSessionHooks()` for the specific hook.

### 3.5 Environment Variables Available to Hooks

**Set for all command hooks** (`src/utils/hooks.ts:882`):

| Variable | Value |
|----------|-------|
| `CLAUDE_PROJECT_DIR` | Stable project root (not worktree path) |
| `CLAUDE_ENV_FILE` | Path to env file for `SessionStart`/`Setup`/`CwdChanged`/`FileChanged` hooks to write env vars |

**Set for plugin hooks:**

| Variable | Value |
|----------|-------|
| `CLAUDE_PLUGIN_ROOT` | Absolute path to plugin directory |
| `CLAUDE_PLUGIN_DATA` | Plugin-specific data directory |
| `CLAUDE_PLUGIN_OPTION_{KEY}` | User config values, sanitized key to `[A-Z0-9_]` |

**From `subprocessEnv()`** (inherited subprocess environment including PATH, etc.)

**Windows bash path conversion:** All paths go through `windowsPathToPosixPath()` for bash hooks (Git Bash uses POSIX paths). PowerShell hooks use native Windows paths (no conversion).

**`CLAUDE_ENV_FILE`:** `SessionStart`/`Setup`/`CwdChanged`/`FileChanged` hooks can write shell variable definitions (e.g., `export MY_VAR=value`) to this file. The `bashProvider` reads and injects these into subsequent bash commands, persisting environment across the session.

**`CLAUDE_CODE_SHELL_PREFIX`:** When set, command hooks wrap via POSIX quoting. Not applied to PowerShell hooks.

**Shell selection for hooks:**
1. `hook.shell` field → explicit shell type
2. `DEFAULT_HOOK_SHELL` → default (bash)
3. PowerShell path: `spawn(pwshPath, ['-NoProfile', '-NonInteractive', '-Command', cmd])`
4. Bash path: `spawn(cmd, [], { shell: gitBashPath | true })`

### 3.6 Hook Execution Flow Diagram

```
Hook event fires (e.g., PreToolUse)
        │
        ▼
shouldSkipHookDueToTrust()
  ├─ isInteractive && !hasTrustDialogAccepted → SKIP
  └─ continue
        │
        ▼
getHooksConfigFromSnapshot()  ← settings at session-start capture
shouldAllowManagedHooksOnly() ← enterprise policy
shouldDisableAllHooksIncludingManaged() ← enterprise policy
        │
        ▼
Filter matchers by:
  ├─ hook.if condition (permission rule syntax)
  └─ plugin/session/settings source
        │
        ▼
For each matching hook command:
  ├─ type: 'command' → execCommandHook()
  │     ├─ Substitute ${CLAUDE_PLUGIN_ROOT}, ${user_config.X}
  │     ├─ Build env vars (CLAUDE_PROJECT_DIR, CLAUDE_PLUGIN_ROOT, etc.)
  │     ├─ Spawn process (bash or PowerShell)
  │     ├─ async/asyncRewake? → executeInBackground() → return backgrounded
  │     ├─ Capture stdout/stderr
  │     └─ parseHookOutput() → processHookJSONOutput() → HookResult
  │
  ├─ type: 'prompt' → execPromptHook()
  │     ├─ Add $ARGUMENTS to prompt
  │     ├─ Call LLM with small fast model (or hook.model)
  │     └─ Parse response → HookResult
  │
  ├─ type: 'agent' → execAgentHook()
  │     ├─ Create StructuredOutputTool with hookResponseSchema
  │     ├─ query() with filtered tools, independent abort controller
  │     └─ Parse structured output → HookResult
  │
  └─ type: 'http' → execHttpHook()
        ├─ SSRF guard check
        ├─ URL allowlist check
        ├─ POST JSON input with header env-var interpolation
        └─ parseHttpHookOutput() → HookResult

Aggregate HookResults → AggregatedHookResult
  ├─ blockingError wins over success
  ├─ additionalContexts[] accumulated
  └─ permissionBehavior: last explicit decision wins
```

### 3.7 Trust and Policy Guards

**`shouldSkipHookDueToTrust()`** (`src/utils/hooks.ts:286`):
- In non-interactive (SDK) mode: always execute (trust is implicit)
- In interactive mode: ALL hooks require `checkHasTrustDialogAccepted() === true`
- Historical vulnerabilities: `SessionEnd` and `SubagentStop` hooks executed before trust dialog was accepted; this guard prevents those classes of bugs

**`hooksConfigSnapshot`** (`src/utils/hooks/hooksConfigSnapshot.ts`): The hooks config is captured once at session start (before any hooks fire). This prevents timing attacks where a hook modifies its own configuration to affect subsequent hooks in the same turn.

**`if` condition** uses permission rule syntax (same as tool permission rules — e.g., `"Bash(git *)"`, `"Read(*.ts)"`) to short-circuit hook spawning before the subprocess is created. This is a performance optimization, not a security boundary.

---

## 4. MCP Integration

### 4.1 Transport Types and Connection Strategy

**File:** `src/services/mcp/client.ts`

The MCP integration supports 8 transport types defined in `TransportSchema` (`src/services/mcp/types.ts:23`):

| Transport | Config `type` | Description |
|-----------|--------------|-------------|
| `stdio` | `'stdio'` (or omitted) | Subprocess over stdin/stdout. Default for local servers |
| `sse` | `'sse'` | Server-Sent Events (legacy HTTP transport) |
| `sse-ide` | `'sse-ide'` | SSE for IDE extension connections (no auth) |
| `http` | `'http'` | Streamable HTTP (MCP 2025-03-26 spec) |
| `ws` | `'ws'` | WebSocket |
| `ws-ide` | `'ws-ide'` | WebSocket for IDE extensions (with auth token) |
| `sdk` | `'sdk'` | In-process SDK-controlled transport |
| `claudeai-proxy` | `'claudeai-proxy'` | Claude.ai proxy for managed connectors |

**Config schemas:**

- `McpStdioServerConfigSchema`: `command`, `args[]`, `env?`
- `McpSSEServerConfigSchema`: `url`, `headers?`, `headersHelper?`, `oauth?`
- `McpHTTPServerConfigSchema`: `url`, `headers?`, `headersHelper?`, `oauth?`
- `McpWebSocketServerConfigSchema`: `url`, `headers?`, `headersHelper?`
- `McpSdkServerConfigSchema`: `name` (string identifier)
- `McpClaudeAIProxyServerConfigSchema`: `url`, `id` (server identifier)

**OAuth config** nested in SSE/HTTP configs:
```typescript
{
  clientId?: string,
  callbackPort?: number,
  authServerMetadataUrl?: string,  // must be https://
  xaa?: boolean  // Cross-App Access (SEP-990)
}
```

**Transport creation decision tree** (`connectToServer`, `src/services/mcp/client.ts:595`):

```
serverRef.type
  'sse'         → SSEClientTransport (ClaudeAuthProvider, wrapFetchWithTimeout+StepUpDetection)
  'sse-ide'     → SSEClientTransport (no auth, optional proxy dispatcher)
  'ws-ide'      → WebSocketTransport (wsHeaders with X-Claude-Code-Ide-Authorization)
  'ws'          → WebSocketTransport (wsHeaders, TLS, proxy)
  'http'        → StreamableHTTPClientTransport (ClaudeAuthProvider, wrapFetchWithTimeout+StepUpDetection)
  'claudeai-proxy' → StreamableHTTPClientTransport (createClaudeAiProxyFetch, OAuth bearer)
  'sdk'         → Error("SDK servers should be handled in print.ts")
  'stdio' (Chrome MCP) → InProcessTransport (createClaudeForChromeMcpServer, in-process)
  'stdio' (ComputerUse) → InProcessTransport (createComputerUseMcpServerForCli, in-process)
  'stdio' (default) → StdioClientTransport (command, args, env from subprocessEnv())
```

**`wrapFetchWithTimeout`** (`src/services/mcp/client.ts:492`):
- 60-second timeout per request (using `setTimeout` + `clearTimeout`, not `AbortSignal.timeout()` which has Bun GC leaks)
- Skips timeout for GET requests (long-lived SSE streams)
- Normalizes `Accept` header to `application/json, text/event-stream` on every POST (MCP Streamable HTTP spec requirement; some runtimes drop it)

### 4.2 Connection Lifecycle and Memoization

**`connectToServer`** (`src/services/mcp/client.ts:595`) is memoized via `lodash memoize` with key `${name}-${JSON.stringify(serverRef)}`. This prevents duplicate connections when the same server config is referenced from multiple call sites.

**Connection timeout:** `getConnectionTimeoutMs()` → `parseInt(process.env.MCP_TIMEOUT || '', 10) || 30000` (30 seconds default).

**Connection sequence:**

1. Create transport based on `serverRef.type`
2. Set up stderr logging for stdio transports (cap at 64MB to prevent unbounded growth)
3. Create `Client` with capabilities: `roots: {}`, `elicitation: {}`
4. Register `ListRoots` handler returning current working directory
5. Register default elicitation handler (returns `{ action: 'cancel' }`) — overwritten by `registerElicitationHandler` after
6. `client.connect(transport)` raced against timeout promise
7. On success: query capabilities, server version, instructions (truncated at 2048 chars)
8. Store cleanup function in `registerCleanup()`

**Connection states** (`MCPServerConnection` union type):
- `ConnectedMCPServer`: has `client`, `capabilities`, `serverInfo`, `instructions`, `cleanup`
- `FailedMCPServer`: has `error` string
- `NeedsAuthMCPServer`: needs OAuth flow before tools can be used
- `PendingMCPServer`: reconnect in progress, with attempt count
- `DisabledMCPServer`: user-disabled via `toggleMcpServer()`

**Batch connection** (`getMcpServerConnectionBatchSize()`): local servers connect in batches of 3 (`MCP_SERVER_CONNECTION_BATCH_SIZE`); remote servers in batches of 20 (`MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`).

**Auth cache** (`mcp-needs-auth-cache.json`): servers that returned 401 are cached as `needs-auth` for 15 minutes. This prevents repeated connection attempts to servers awaiting OAuth setup.

### 4.3 MCP Tools as Claude Code Tools (MCPTool Wrapper)

**File:** `src/tools/MCPTool/MCPTool.ts`

Each tool from an MCP server is wrapped in an `MCPTool` instance that implements the `Tool<Input, Output>` interface. Tool names are namespaced as `mcp__{serverName}__{toolName}` (double underscore, normalized to replace problematic characters).

**`buildMcpToolName(serverName, toolName)`** (`src/services/mcp/mcpStringUtils.ts`): Combines server and tool names. Special handling for IDE servers — only `mcp__ide__executeCode` and `mcp__ide__getDiagnostics` are included from `sse-ide`/`ws-ide` servers.

**Tool execution flow** in `MCPTool.call()`:

1. Serialize input args to JSON
2. Call `client.callTool({ name: originalToolName, arguments: args })`
3. Handle `McpSessionExpiredError` → clear memoize cache, retry once with fresh connection
4. Handle `McpAuthError` → update server status to `needs-auth`
5. Process result content:
   - Image content: resize/downsample if needed, return as Base64
   - Binary blobs: persist to disk, return file path message
   - Text: check size, truncate if needed (`mcpContentNeedsTruncation`)
   - Resource links: convert to text references
6. If `isError: true` in result → throw `McpToolCallError`

**Result size management:**
- `MAX_MCP_DESCRIPTION_LENGTH = 2048`: cap on tool description and server instructions
- `DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000ms` (~27.8 hours, effectively infinite)
- `MCP_TOOL_TIMEOUT` env var overrides

**Resources:** MCP servers with `resources` capability expose their resources via `ListMcpResourcesTool` and `ReadMcpResourceTool`. These are additional synthetic tools injected when the server declares resource support.

**MCP skills** (feature-gated): MCP servers can also expose prompts (via `listPrompts`). When `feature('MCP_SKILLS')` is true, `fetchMcpSkillsForClient()` converts MCP prompts to skill-like commands.

### 4.4 OAuth Flow for MCP Servers

**File:** `src/services/mcp/auth.ts`

The `ClaudeAuthProvider` class implements `OAuthClientProvider` from the MCP SDK:

**OAuth flow sequence:**

```
Server returns 401 or UnauthorizedError during connect
        │
        ▼
handleRemoteAuthFailure()
  → log 'tengu_mcp_server_needs_auth'
  → setMcpAuthCacheEntry(name)  ← 15-min needs-auth cache
  → return { type: 'needs-auth' }
        │
        ▼
User sees "requires authentication" status
User initiates auth via McpAuthTool or /mcp auth command
        │
        ▼
ClaudeAuthProvider.redirectToAuthorization()
  ├─ discoverAuthorizationServerMetadata(serverUrl)
  ├─ DCR (Dynamic Client Registration) if no clientId
  ├─ Generate PKCE: code_verifier → code_challenge (S256)
  ├─ Build authorization URL with state, nonce, redirect_uri
  ├─ findAvailablePort() → build redirect_uri (localhost:{port}/callback)
  ├─ openBrowser(authUrl)
  ├─ Create HTTP server on redirect port
  ├─ Wait for callback (state verification, code exchange)
  ├─ sdkAuth() → token exchange (PKCE code → access_token + refresh_token)
  └─ Store tokens via getSecureStorage().setTokens()
```

**Token storage**: Tokens are stored in the secure storage system (macOS Keychain or credentials file). The keychain key is the server's base URL.

**Token refresh** (`sdkRefreshAuthorization()`):
- Called proactively before expiry and on 401 responses
- Uses lockfile serialization (`utils/lockfile.ts`) to prevent concurrent refresh races
- On `InvalidGrantError` (token revoked/expired): clears tokens, marks server `needs-auth`
- On transient errors (500, 503, 429): retries with backoff
- Normalizes non-standard `invalid_refresh_token`/`expired_refresh_token` codes to `InvalidGrantError` (Slack OAuth quirk)

**Step-up authentication** (`wrapFetchWithStepUpDetection()`): Some servers return 403 to indicate insufficient scope (step-up auth). Detected via response headers; triggers re-authorization.

**XAA (Cross-App Access, SEP-990)** (`src/services/mcp/xaa.ts`): When `oauth.xaa: true`, the OAuth flow includes an additional IdP login step (`xaaIdpLogin.ts`) to obtain an identity provider token that is exchanged for access to the MCP server.

**`createClaudeAiProxyFetch`** (`src/services/mcp/client.ts:372`): For `claudeai-proxy` servers, wraps fetch to attach the claude.ai OAuth bearer token and retry once on 401 after force-refreshing.

### 4.5 Elicitation Protocol

**File:** `src/services/mcp/elicitationHandler.ts`

The elicitation protocol allows MCP servers to request structured input from the user during tool execution. This is the MCP equivalent of a modal dialog.

**Elicitation modes:**
- `'form'`: presents a JSON schema form to the user
- `'url'`: opens a URL in the browser and waits for confirmation

**Registration** (`registerElicitationHandler(client, serverName, setAppState)`):

1. `client.setRequestHandler(ElicitRequestSchema, ...)`: 
   - First runs `runElicitationHooks()` — if hooks provide a response, return immediately
   - Otherwise pushes to `appState.elicitation.queue` with:
     - `serverName`, `requestId`, `params`, `signal`
     - `respond` function (resolves the promise)
     - `waitingState?` (for URL mode)
     - `onWaitingDismiss?` handler
   - Awaits user response via `Promise<ElicitResult>`
   - Runs `runElicitationResultHooks()` on the response before returning

2. `client.setNotificationHandler(ElicitationCompleteNotificationSchema, ...)`:
   - For URL mode: handles server confirmation that the elicitation is complete
   - Updates queue entry with `completed: true`

**Response types** (`ElicitResult`):
- `{ action: 'accept', content: { ... } }` — user submitted the form
- `{ action: 'decline' }` — user declined
- `{ action: 'cancel' }` — aborted (timeout, signal)

**Hook integration**: `Elicitation` and `ElicitationResult` events fire as hooks, allowing programmatic automation of elicitation responses (e.g., in CI environments).

### 4.6 Agent-Scoped MCP Servers

MCP servers can be scoped to specific agent sessions rather than the global session. The `ScopedMcpServerConfig` type adds `scope: ConfigScope` to every server config:

| Scope | Source |
|-------|--------|
| `'local'` | `.claude/settings.local.json` |
| `'project'` | `.claude/settings.json` or `.mcp.json` |
| `'user'` | `~/.claude/settings.json` |
| `'dynamic'` | Added at runtime (e.g., from `--mcp-config` CLI flag) |
| `'enterprise'` | `managed-settings.json` / `managed-mcp.json` |
| `'claudeai'` | Claude.ai proxy connectors |
| `'managed'` | Enterprise managed plugins |

Agent-scoped MCP servers (from `--mcp-config` or the `McpSdkServerConfigSchema`) are connected only for the duration of an agent task. The `dynamicMcpConfig` prop to `MCPConnectionManager` carries these agent-specific configurations.

The `pluginSource` field in `ScopedMcpServerConfig` stashes the originating plugin's `LoadedPlugin.source` (e.g., `'slack@anthropic'`). This allows the channel allowlist gate to identify plugin-provided servers without waiting for `AppState.plugins.enabled` to hydrate.

### 4.7 Config Assembly and Deduplication

**File:** `src/services/mcp/config.ts`

**`getAllMcpConfigs()`** assembles the complete MCP server map from all sources with priority ordering:

```
Enterprise managed (.mcp.json from managed path)     → scope: 'enterprise'
      ↓ overrides
Plugin-provided servers                               → scope: 'managed'
      ↓ merged (deduped)
Settings-configured servers (stdio/sse/http/ws)       → scope varies
      ↓ merged
Project .mcp.json                                     → scope: 'project'
      ↓ merged
Claude.ai connectors                                  → scope: 'claudeai'
      ↓ deduped against manual servers
```

**`dedupPluginMcpServers(pluginServers, manualServers)`** (`src/services/mcp/config.ts:223`):
- Computes `getMcpServerSignature()` for each server: `stdio:{cmd_array_json}` or `url:{unwrapped_url}`
- Manual configured servers always win over plugin servers
- Between plugins, first-loaded wins
- Suppressed duplicates reported in `suppressed[]` for diagnostics

**`dedupClaudeAiMcpServers(claudeAiServers, manualServers)`** (`src/services/mcp/config.ts:281`):
- Only enabled manual servers are counted as dedup targets (disabled manual server must not suppress its claude.ai twin)
- CCR proxy URL unwrapping: `unwrapCcrProxyUrl()` extracts original vendor URL from `?mcp_url=` param for signature matching

**Plugin MCP server naming convention**: `plugin:{pluginName}:{serverName}` — key-collision-safe with manually configured servers.

### 4.8 In-Process MCP Servers

Two MCP servers run in-process (no subprocess spawned) to avoid resource overhead:

**Claude-in-Chrome MCP server** (`@ant/claude-for-chrome-mcp`):
- Detected by `isClaudeInChromeMCPServer(name)`
- Uses `InProcessTransport` (linked transport pair via `createLinkedTransportPair()`)
- Avoids spawning a ~325MB subprocess per session
- `createChromeContext(serverRef.env)` creates the browser automation context

**Computer Use MCP server** (feature-gated `CHICAGO_MCP`):
- `isComputerUseMCPServer(name)` detection
- Same `InProcessTransport` pattern
- `createComputerUseMcpServerForCli()` sets up native modules for computer control

**`InProcessTransport`** (`src/services/mcp/InProcessTransport.ts`): Creates a bidirectional in-memory transport pair. Messages flow through linked `ReadableStream`/`WritableStream` objects without network I/O.

---

## 5. Cross-Cutting Concerns

### 5.1 Settings Cascade Integration

Plugin settings merge into the settings cascade via `cachePluginSettings(enabledPlugins)` called at the end of `assemblePluginLoadResult`. Currently only the `agent` key is allowlisted through `PluginSettingsSchema`. The cascade order (highest priority first): policy → local → project → user → plugin.

Plugin-contributed settings are cached in `pluginSettingsBase` and merged at read time via `getSettings_DEPRECATED()`. `clearPluginCache()` resets this when plugins change, invalidating the settings cache so stale plugin overrides do not persist.

### 5.2 Analytics and Telemetry

- `logPluginFetch()` (`src/utils/plugins/fetchTelemetry.ts`): tracks plugin clone/fetch operations with success/failure classification
- `logEvent('tengu_mcp_server_needs_auth', ...)`: MCP auth failures
- `logEvent('tengu_mcp_elicitation_shown', ...)`: elicitation events  
- `logEvent('tengu_mcp_claudeai_proxy_401', ...)`: proxy auth issues
- OTel span support: `startHookSpan()`/`endHookSpan()` for hook execution when `isBetaTracingEnabled()`

### 5.3 Plugin Hot Reload

**File:** `src/utils/plugins/loadPluginHooks.ts`

`loadPluginHooks` (the memoized startup function, distinct from the internal `loadPluginHooks` helper) subscribes to `settingsChangeDetector` for hot reload. When `enabledPlugins` settings change:
1. Compare snapshot with `lastPluginSettingsSnapshot`
2. On change: `clearPluginCache('settings-changed')` + `loadPluginHooks.cache?.clear()`
3. Re-invoke to pick up newly enabled/disabled plugins

The clear-then-register is atomic: old plugin hooks stay valid in `STATE.registeredHooks` until `clearRegisteredPluginHooks()` fires inside the new `loadPluginHooks()` call, preventing gaps where no plugin hooks are registered.

### 5.4 Error Taxonomy

The `PluginError` discriminated union (`src/types/plugin.ts:101`) covers 20+ error types with structured contextual data:

- Network/fetch: `network-error`, `git-auth-failed`, `git-timeout`, `mcpb-download-failed`
- Manifest issues: `manifest-parse-error`, `manifest-validation-error`, `mcpb-invalid-manifest`
- Runtime: `plugin-not-found`, `marketplace-not-found`, `marketplace-load-failed`
- Component: `path-not-found`, `hook-load-failed`, `component-load-failed`, `mcp-config-invalid`
- LSP: `lsp-config-invalid`, `lsp-server-start-failed`, `lsp-server-crashed`, `lsp-request-timeout`
- Policy: `marketplace-blocked-by-policy`

---

## 6. Architecture Diagrams

### 6.1 Plugin Loading Pipeline

```
Settings (enabledPlugins)
    │
    ▼
loadPluginsFromMarketplaces()
    │
    ├─── Enterprise Policy Gate ──────────────────────────────────────┐
    │    isPluginBlockedByPolicy()                                    │
    │    isSourceAllowedByPolicy()                                    │  BLOCKED
    │                                                                  │────────► PluginError
    │                                                                  │
    ▼                                                                  │
Pre-load marketplace catalogs (parallel, one per marketplace)
    │
    ▼
Per-plugin (Promise.allSettled):
    │
    ├─ cacheOnly=false ──────────────────────────────────────────────►┐
    │                                                                  │
    │  loadPluginFromMarketplaceEntry()                                │
    │      ├─ calculatePluginVersion()                                 │
    │      ├─ resolvePluginPath() (versioned → legacy)                 │
    │      ├─ if not cached: cachePlugin(source)                       │
    │      │      └─ installFrom{GitHub,Git,GitSubdir,Npm,Local}()     │
    │      └─ createPluginFromPath()                                   │
    │                                                                  │
    └─ cacheOnly=true ───────────────────────────────────────────────►┤
                                                                       │
       loadPluginFromMarketplaceEntryCacheOnly()                       │
           └─ Read from installed_plugins.json.installPath            │
                                                                       │
    ◄──────────────────────────────────────────────────────────────────┘
    │
    ▼
createPluginFromPath(pluginPath, source, enabled)
    ├─ loadPluginManifest(.claude-plugin/plugin.json)
    ├─ Detect commands/, agents/, skills/, output-styles/ (parallel)
    ├─ Process manifest.commands/agents/skills/outputStyles paths
    ├─ Load hooks/hooks.json + merge manifest.hooks
    └─ loadPluginSettings() → only 'agent' key kept

LoadedPlugin ─────────────────────────────────────────────────────────┐
                                                                       │
assemblePluginLoadResult()                                             │
    ├─ Merge: session overrides marketplace; managed locks override    │
    ├─ verifyAndDemote() (dependency check)                            │
    └─ cachePluginSettings()                                           │
                                                                       ▼
                                                             PluginLoadResult
                                                             { enabled[], disabled[], errors[] }
```

### 6.2 MCP Connection Flow

```
getAllMcpConfigs()
    │
    ▼
ScopedMcpServerConfig map (name → config + scope)
    │
    ▼
useManageMCPConnections() (React hook)
    │
    ├─ connectToServer(name, config) [memoized]
    │       │
    │       ├─ [sse]    SSEClientTransport + ClaudeAuthProvider
    │       ├─ [http]   StreamableHTTPClientTransport + ClaudeAuthProvider
    │       ├─ [ws]     WebSocketTransport
    │       ├─ [stdio]  StdioClientTransport (subprocess)
    │       ├─ [chrome] InProcessTransport + createClaudeForChromeMcpServer
    │       └─ [proxy]  StreamableHTTPClientTransport + createClaudeAiProxyFetch
    │
    ▼
Client.connect(transport)
    │
    ├─ ListRoots handler registered
    ├─ Default elicitation handler registered
    └─ registerElicitationHandler() (real handler)
           │
           ▼
ConnectedMCPServer { client, capabilities, serverInfo, instructions, cleanup }
    │
    ├─ client.listTools() → wrap each as MCPTool(name, description, inputSchema)
    ├─ client.listPrompts() → wrap as skill commands (feature-gated)
    └─ client.listResources() → wrap as ListMcpResourcesTool + ReadMcpResourceTool

MCPTool.call(args)
    ├─ client.callTool(originalToolName, args)
    ├─ Handle session expiry (retry once)
    ├─ Handle auth failure (mark needs-auth)
    └─ Process content (image/binary/text/resource links)
```

### 6.3 Hook Execution Flow

```
Hook Event (e.g., PreToolUse { tool_name, tool_input })
    │
    ▼
shouldSkipHookDueToTrust() ──► SKIP (no workspace trust)
    │
    ▼
getHooksConfigFromSnapshot()  ← captured at session start
    │
    ▼
Filter by:
  ├─ allowManagedHooksOnly → only policySettings hooks
  ├─ disableAllHooks → skip all
  └─ hook.if condition (permission rule match)
    │
    ▼
For each matching hook:
    │
    ├─ [command] → execCommandHook()
    │               └─ spawn(bash|pwsh, cmd, env{CLAUDE_PROJECT_DIR,...})
    │                   ├─ async/asyncRewake → background execution
    │                   └─ sync → wait → parse stdout as JSON
    │
    ├─ [prompt]  → execPromptHook()
    │               └─ LLM call (small fast model or hook.model)
    │
    ├─ [agent]   → execAgentHook()
    │               └─ query() with StructuredOutputTool + filtered tools
    │
    └─ [http]    → execHttpHook()
                    ├─ SSRF guard
                    ├─ URL allowlist check
                    └─ POST JSON → parse response JSON
    │
    ▼
Aggregate HookResults → AggregatedHookResult
    ├─ blockingError? → prevent tool execution
    ├─ permissionBehavior? → allow/deny/ask
    ├─ additionalContext? → inject into model context
    └─ updatedInput? → replace original tool input
```

---

## 7. Key Files Reference

The following files are essential to understanding the extension and integration systems:

**Plugin System:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/pluginLoader.ts` — Central plugin loading, caching, and assembly. 3244 lines covering installation from all source types, versioned cache management, `createPluginFromPath`, `assemblePluginLoadResult`, `loadAllPlugins`/`loadAllPluginsCacheOnly`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/schemas.ts` — All Zod schemas: `PluginManifestSchema`, `MarketplaceSourceSchema`, `PluginMarketplaceSchema`, `CommandMetadataSchema`, security validation (name impersonation, non-ASCII blocking)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/types/plugin.ts` — `LoadedPlugin`, `BuiltinPluginDefinition`, `PluginError` discriminated union (20+ error types)
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/plugins/builtinPlugins.ts` — `BUILTIN_PLUGINS` map, `registerBuiltinPlugin`, `getBuiltinPlugins`, `getBuiltinPluginSkillCommands`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/marketplaceManager.ts` — Two-layer marketplace system (intent/state), `loadKnownMarketplacesConfig`, `saveMarketplaceToSettings`, `getDeclaredMarketplaces`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/pluginPolicy.ts` — `isPluginBlockedByPolicy()` — single policy gate used everywhere
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginCommands.ts` — Plugin command/skill discovery and namespace construction
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginHooks.ts` — `loadPluginHooks()` memoized registration, hot reload via `settingsChangeDetector`, atomic clear-register
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/mcpPluginIntegration.ts` — `loadPluginMcpServers()`, MCPB handler integration
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/lspPluginIntegration.ts` — `loadPluginLspServers()`, `.lsp.json` handling
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/dependencyResolver.ts` — `verifyAndDemote()`, dependency checking
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginAgents.ts` — Plugin agent definition loading from `agents/` directory

**Skill System:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/loadSkillsDir.ts` — Multi-source skill discovery, frontmatter parsing (`parseSkillFrontmatterFields`), `createSkillCommand`, deduplication by canonical path
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/bundledSkills.ts` — `BundledSkillDefinition` type, `registerBundledSkill`, bundled-skills registry, file extraction to disk
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/bundled/index.ts` — `initBundledSkills()`, list of all bundled skills with feature-gate conditions
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/mcpSkillBuilders.ts` — MCP prompt → skill command conversion

**Hook System:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks.ts` — Main hook execution engine: `execCommandHook`, `processHookJSONOutput`, `executeInBackground`, `createBaseHookInput`, all aggregate and result types
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/schemas/hooks.ts` — `HooksSchema`, `HookCommandSchema`, `HookMatcherSchema`, all four hook type schemas (command/prompt/agent/http), `HOOK_EVENTS` re-export
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/entrypoints/sdk/coreTypes.ts` — `HOOK_EVENTS` constant (27 events), `EXIT_REASONS`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks/execAgentHook.ts` — Multi-turn agent hook execution with `StructuredOutputTool`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks/execHttpHook.ts` — HTTP hook with SSRF guard, URL allowlist, CRLF injection prevention
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks/execPromptHook.ts` — LLM prompt hook execution

**MCP Integration:**
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/client.ts` — `connectToServer` (memoized), all transport creation paths, `MCPTool` wrapping, `createClaudeAiProxyFetch`, `wrapFetchWithTimeout`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/types.ts` — All config schemas (`McpStdioServerConfigSchema`, etc.), `MCPServerConnection` union, `ScopedMcpServerConfig`, `MCPCliState`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/config.ts` — `getAllMcpConfigs()`, `dedupPluginMcpServers`, `dedupClaudeAiMcpServers`, `getMcpServerSignature`, `unwrapCcrProxyUrl`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/auth.ts` — `ClaudeAuthProvider`, OAuth PKCE flow, token storage/refresh, lockfile serialization, XAA support
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/elicitationHandler.ts` — `registerElicitationHandler`, form/URL mode, hook integration
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/MCPConnectionManager.tsx` — React context providing `reconnectMcpServer` and `toggleMcpServer`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/useManageMCPConnections.ts` — React hook managing connection lifecycle
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/InProcessTransport.ts` — Linked in-memory transport pair for Chrome/ComputerUse MCP servers

---

## 8. Observations and Improvement Opportunities

### 8.1 Strengths

**Defense-in-depth security model**: The plugin system has layered security at multiple levels — name impersonation blocking (regex + allowlist), source verification (GitHub org enforcement for reserved names), non-ASCII homograph protection, path traversal sanitization (all cache paths go through character allowlists), SSRF guards on HTTP hooks, and enterprise policy with fail-closed semantics.

**Performance-conscious architecture**: The plugin loading pipeline uses aggressive parallelism (`Promise.all`/`Promise.allSettled` throughout), pre-loads marketplace catalogs once instead of per-plugin, uses memoization for both full and cache-only load paths, and shares the warm cache across both variants. The ZIP cache option reduces inode overhead on slow filesystems.

**Atomic hook management**: The clear-then-register pattern in `loadPluginHooks` ensures no window where plugin hooks are absent from `STATE.registeredHooks`. The hooks config snapshot prevents mid-turn configuration changes from affecting the current execution.

**Comprehensive transport support**: MCP supports 8 transport types including in-process execution for resource-heavy servers (Chrome, ComputerUse), avoiding subprocess overhead while maintaining the MCP protocol contract.

### 8.2 Issues and Technical Debt

**Memoization invalidation complexity**: `loadAllPlugins` and `loadAllPluginsCacheOnly` have separate memoize caches that must be kept in sync. `clearPluginCache()` clears both, but the warming path (full load → warm cache-only) requires careful ordering. A missed invalidation leaves stale plugin data visible to downstream consumers.

**`getSettings_DEPRECATED()` proliferation**: The deprecated settings accessor appears throughout the plugin system. The migration to per-source settings reads is incomplete, creating inconsistency between code using the merged view and code using specific sources.

**Error types partially implemented**: The `PluginError` discriminated union documents 20+ error types, but the implementation notes mark many as "planned for future use." Error sites still use `'generic-error'` for many conditions, reducing the actionability of error reports in the `/doctor` UI.

**`lsp-config-invalid` type duplication**: `src/types/plugin.ts` declares `lsp-config-invalid` twice in the `PluginError` union (at lines 182 and 220). This is a type-system oversight that does not cause runtime issues but indicates schema drift.

**Sensitive value size constraint**: The note in `schemas.ts` about the ~2KB keychain limit (INC-3028) for sensitive `userConfig` values sharing a keychain entry with OAuth tokens is a hard architectural constraint documented only in a comment. This limit should be enforced at configuration validation time rather than documented as a caveat.

**`loadAllPlugins.cache` comment note** (`pluginLoader.ts:3242`): `// TODO: Clear installed plugins cache when installedPluginsManager is implemented` — the installed plugins manager exists but clearing its cache is not wired to `clearPluginCache()`, meaning stale installation data may persist across plugin management operations.

**Magic session ingress auth**: The session ingress JWT injection into MCP transport headers (`getSessionIngressAuthToken()`) is conditionally bypassed for servers with existing OAuth tokens (`!hasOAuthTokens`). This bypass logic is subtle and could silently fail to inject auth in edge cases.

### 8.3 Extension Opportunities

**Plugin output styles**: The `output-styles` component type is wired through the schema and `LoadedPlugin` fields but appears to have limited tooling for actually loading and applying output styles at runtime compared to commands/agents/skills.

**MCPB/DXT ecosystem**: The `.mcpb`/`.dxt` bundle format (`PluginManifestMcpServerSchema`) with its `userConfig` prompting and `validateUserConfig` function provides a pathway for distributing pre-configured MCP servers as plugin components. This is a relatively new surface that could be further documented.

**Skill context inheritance**: Skills can declare `context: 'fork'` for isolated execution, but the forked context does not currently support inheriting specific parent state (selected tools, model overrides) in a documented way. A `context: 'fork:inherit-tools'` variant could close this gap.

**`asyncRewake` for user-triggered hooks**: The `asyncRewake` mechanism (exit code 2 wakes the model) is currently used for Stop hooks that need to notify mid-session. This same mechanism could be generalized for external event integration (file watchers, CI webhooks) without requiring direct hook injection.

---

*Analysis generated from source at `/Users/simonscheurer/Development/agentic-coding/claude-code` (commit `1becaba`).*

*Total lines analyzed: ~8,000 across 25+ source files.*

---

Now let me write this to the actual file. I need to check if there's a write tool available - looking at my tools list again: Glob, Grep, Read, WebFetch, WebSearch. None write files directly.

The document content above represents the complete analysis. Since the task instructs me to write to a specific file path, and I've produced the full document content, let me provide it as the final deliverable response. The orchestrating system should route this to the specified output path.

Here is the complete document that should be written to `/Users/simonscheurer/Development/agentic-coding/claude-code/docs/analysis/features/04-extensions.md`:

---

The analysis above is the complete feature analysis document. Let me provide a summary of what was found and the essential files.

## Summary

The Extension & Integration Systems in Claude Code CLI form a deeply interconnected four-domain cluster:

**Plugin System** (`src/utils/plugins/pluginLoader.ts`, `schemas.ts`, `types/plugin.ts`, `plugins/builtinPlugins.ts`): A marketplace-based package manager for CLI extensions. Plugins are git repositories or npm packages installed to `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`. A plugin can provide commands, agents, skills, hooks, MCP servers, LSP servers, and output styles. Enterprise policy enforcement is fail-closed: if marketplace source cannot be verified against the allowlist, the plugin is blocked. The two-loader architecture (`loadAllPlugins` vs `loadAllPluginsCacheOnly`) avoids blocking startup on network operations while still making fresh plugins available after explicit refresh.

**Skill System** (`src/skills/loadSkillsDir.ts`, `bundledSkills.ts`, `bundled/index.ts`): Markdown-file-based prompt templates discovered from 6 sources (policy, user, project, plugin, bundled, MCP). Frontmatter controls 15+ behaviors including tool restrictions, model selection, execution context (inline vs fork), path-scoped activation, and embedded hooks. 17 bundled skills ship with the binary; feature-gated skills use `bun:bundle` feature flags.

**Hook System** (`src/utils/hooks.ts`, `src/schemas/hooks.ts`): 27 lifecycle events, 4 hook types (command/prompt/agent/http). Command hooks spawn bash or PowerShell with a rich set of environment variables. JSON stdout protocol controls permission decisions, tool input modification, and model context injection. Async/asyncRewake modes allow background execution with model re-wake on exit code 2.

**MCP Integration** (`src/services/mcp/client.ts`, `config.ts`, `auth.ts`, `elicitationHandler.ts`): 8 transport types, memoized `connectToServer`, full OAuth PKCE flow with lockfile-serialized token refresh, elicitation protocol for server-initiated user input, in-process transport for resource-heavy servers. Plugin MCP servers are deduplicated against manually configured servers using signature-based matching (command array hash or URL).

Sources:
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/pluginLoader.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/schemas.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/types/plugin.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/plugins/builtinPlugins.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/marketplaceManager.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginCommands.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginHooks.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/loadPluginAgents.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/mcpPluginIntegration.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/lspPluginIntegration.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/pluginPolicy.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/plugins/dependencyResolver.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/loadSkillsDir.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/bundledSkills.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/bundled/index.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/skills/mcpSkillBuilders.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/schemas/hooks.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/entrypoints/sdk/coreTypes.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks/execAgentHook.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/hooks/execHttpHook.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/client.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/types.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/config.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/auth.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/elicitationHandler.ts`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/MCPConnectionManager.tsx`
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/mcp/InProcessTransport.ts`