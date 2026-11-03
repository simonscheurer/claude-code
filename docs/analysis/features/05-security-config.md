# 05 — Security, Authentication & Configuration

*Feature analysis of the security, auth, and configuration domains in Claude Code CLI*

---

## Table of Contents

1. [Permissions & Safety](#1-permissions--safety)
   - 1.1 Permission Decision Pipeline
   - 1.2 Permission Modes
   - 1.3 Auto-Mode AI Classifier
   - 1.4 Denial Tracking
   - 1.5 Dangerous Patterns
2. [Authentication & Security](#2-authentication--security)
   - 2.1 OAuth 2.0 + PKCE Flow
   - 2.2 API Key Resolution Chain
   - 2.3 macOS Keychain Integration
   - 2.4 AWS Bedrock / STS Authentication
   - 2.5 Token Refresh
3. [Configuration System](#3-configuration-system)
   - 3.1 Settings Source Hierarchy
   - 3.2 Settings Loading & Merging
   - 3.3 Live Settings Reload
   - 3.4 Enterprise MDM Controls
4. [Analytics & Telemetry](#4-analytics--telemetry)
   - 4.1 Zero-Startup-Cost Event Queue
   - 4.2 PII Safety Mechanism
   - 4.3 GrowthBook Feature Flag Integration
   - 4.4 OpenTelemetry / Perfetto Tracing
5. [Cost & Billing](#5-cost--billing)
   - 5.1 Real-Time Cost Tracking
   - 5.2 Token Aggregation
   - 5.3 Rate Limits & Billing Tiers
6. [Essential Files Reference](#6-essential-files-reference)

---

## 1. Permissions & Safety

### 1.1 Permission Decision Pipeline

The core permission gate is `hasPermissionsToUseTool` in `src/utils/permissions/permissions.ts:473`. It calls `hasPermissionsToUseToolInner` (line 1158) which runs a deterministic multi-step decision tree, then applies mode-specific transformations on the result.

**Step-by-step flow inside `hasPermissionsToUseToolInner`:**

```
Tool use requested
       |
       v
[1a] getDenyRuleForTool() -- entire tool in alwaysDenyRules?
       | YES -> return { behavior: 'deny' }
       | NO
       v
[1b] getAskRuleForTool() -- entire tool in alwaysAskRules?
       | YES (and not sandbox-auto-allowable) -> return { behavior: 'ask' }
       | NO / falls through for sandbox
       v
[1c] tool.checkPermissions(parsedInput, context)
       | (content-level rules, path safety, command matching)
       v
[1d] toolPermissionResult.behavior === 'deny'?
       | YES -> return deny
       v
[1e] tool.requiresUserInteraction() && result === 'ask'?
       | YES -> return ask (cannot be auto-approved)
       v
[1f] result === 'ask' && decisionReason.type === 'rule' && ruleBehavior === 'ask'?
       | YES -> return ask (user-configured ask rule trumps bypass mode)
       v
[1g] result === 'ask' && decisionReason.type === 'safetyCheck'?
       | YES -> return ask (safety checks immune to bypassPermissions)
       v
[2a] mode === 'bypassPermissions' OR (mode === 'plan' && isBypassPermissionsModeAvailable)?
       | YES -> return { behavior: 'allow' }
       v
[2b] toolAlwaysAllowedRule() -- entire tool in alwaysAllowRules?
       | YES -> return { behavior: 'allow' }
       v
[3] Convert 'passthrough' -> 'ask'
       |
       v
    return result (ask or allow from tool.checkPermissions)
```

**Post-inner transformations** (in `hasPermissionsToUseTool`, line 473):

- On `allow`: if auto mode and consecutiveDenials > 0, call `recordSuccess()` to reset the streak.
- On `ask` + mode `dontAsk`: transform to `deny` with `DONT_ASK_REJECT_MESSAGE`.
- On `ask` + mode `auto` (or plan with autoMode active):
  - Safety check with `classifierApprovable=false` → block classifier, forward 'ask' or deny if headless.
  - `tool.requiresUserInteraction()` → skip classifier, return 'ask'.
  - Check denial tracking limit (`shouldFallbackToPrompting`) → if limit hit, fall through to prompting.
  - Run `acceptEdits` fast-path: re-run `tool.checkPermissions` in a synthetic acceptEdits context; if 'allow', skip classifier and allow.
  - Check `isAutoModeAllowlistedTool(tool.name)` allowlist (read-only tools, task tools, etc.); if match, allow.
  - Run `classifyYoloAction()` — the AI classifier.

**Key data structures:**

The `ToolPermissionContext` (defined in `src/types/permissions.ts:427`) held in React `AppState` carries all rule sets:

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // { [source]: string[] }
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}
```

Rule strings have the format `ToolName` or `ToolName(content)` with escaped parentheses. Parsing is handled by `permissionRuleValueFromString()` in `src/utils/permissions/permissionRuleParser.ts:93`, which finds the first unescaped `(` and last unescaped `)`, extracts tool name and content, and resolves legacy tool name aliases (`Task` → `Agent`, `KillShell` → `TaskStop`, etc.).

**MCP tool matching** (line 248): Rules with format `mcp__serverName` match all tools from that MCP server (`mcp__serverName__toolName`), providing server-level permission grants.

**Shell rule matching** (`src/utils/permissions/shellRuleMatching.ts`): Rules for Bash/PowerShell content support three pattern types:
- `exact`: literal command match
- `prefix` (legacy `command:*` syntax): matches commands starting with prefix
- `wildcard`: patterns using `*` (not escaped), compiled to RegExp with null-byte placeholders to avoid escaping conflicts

### 1.2 Permission Modes

Modes are defined in `src/types/permissions.ts:16` as two sets:

**External modes** (user-facing, storable in settings):
- `default` — standard interactive prompting
- `plan` — read-only planning mode; Claude can propose but not execute
- `acceptEdits` — auto-accept file edits in working directory, prompt for shell commands
- `bypassPermissions` — skip all permission prompts (dangerousmode, requires explicit trust dialog)
- `dontAsk` — transform all 'ask' decisions into 'deny' (headless/CI)

**Internal-only modes** (ant/Anthropic builds only):
- `auto` — AI classifier makes approve/deny decisions instead of human prompts
- `bubble` — internal swarm coordination mode (not user-addressable)

Mode configuration in `src/utils/permissions/PermissionMode.ts:42`:

```
mode          | symbol | color      | description
------------- | ------ | ---------- | --------------------------------
default       |        | text       | Normal prompting
plan          | ⏸      | planMode   | Read-only planning
acceptEdits   | ⏵⏵     | autoAccept | Auto-allow file writes in CWD
bypassPermissions | ⏵⏵ | error      | Skip all prompts
dontAsk       | ⏵⏵     | error      | Deny when would have prompted
auto          | ⏵⏵     | warning    | AI classifier (ant-only)
```

**acceptEdits behavior**: Each tool's `checkPermissions` is passed a modified context with mode='acceptEdits'. File write/edit tools allow when the target path is within the working directory. Shell tools (Bash) always return 'ask' from `checkPermissions` regardless of mode — they fall through to the classifier fast-path.

**bypassPermissions**: The trust dialog check (`hasSkipDangerousModePermissionPrompt()`) only consults `userSettings`, `localSettings`, `flagSettings`, and `policySettings` — **never projectSettings**. A malicious project file cannot auto-grant bypass. The mode can also be globally disabled via a Statsig feature gate check in `checkAndDisableBypassPermissionsIfNeeded()` (`src/utils/permissions/bypassPermissionsKillswitch.ts:19`).

**plan mode + bypassPermissions**: When a user was in bypassPermissions and enters plan mode (shift-tab), `isBypassPermissionsModeAvailable` is preserved so that if the user exits plan mode, bypass returns. During plan mode itself, step 2a still applies (`mode === 'plan' && isBypassPermissionsModeAvailable`), meaning even plan mode respects bypass for whichever user started in that mode.

### 1.3 Auto-Mode AI Classifier

The auto-mode classifier (ant/TRANSCRIPT_CLASSIFIER builds only) is implemented across several files:

**Entry point**: `classifyYoloAction()` in `src/utils/permissions/yoloClassifier.ts`. Called from `hasPermissionsToUseTool` after all fast-paths are exhausted.

**Two fast-paths before classifier invocation** (lines 596-686):
1. **acceptEdits fast-path**: Re-runs `tool.checkPermissions` with mode='acceptEdits'. If it would have been allowed (file write in CWD), skip the classifier API call.
2. **Allowlist fast-path**: Checks `isAutoModeAllowlistedTool(tool.name)` against a static set of read-only tools (FileRead, Glob, Grep, LSP, TODO tools, task management, etc.) and allows immediately.

**Classifier operation** (`classifyYoloAction`): Sends the current message history (truncated) plus the proposed tool action as a side-query (`sideQuery()`) to a separate AI model. The classifier uses a system prompt loaded from `.txt` files at build time — `auto_mode_system_prompt.txt` as base, then either `permissions_external.txt` or `permissions_anthropic.txt` depending on `USER_TYPE`. The prompt includes configurable sections for `allow`, `soft_deny`, and `environment` rules that users can customize via `settings.autoMode`.

**Two-stage classifier** (ant builds): A fast XML stage 1 runs first; if uncertain, a thinking-enabled stage 2 is invoked. The `YoloClassifierResult` type (`src/types/permissions.ts:346`) carries both stage usage, model IDs, request/message IDs, and durations for analytics.

**Classifier state machine**:
- `shouldBlock=false` → allow, call `recordSuccess()` on denial state
- `shouldBlock=true` → record denial, check `shouldFallbackToPrompting()`, then either deny or fall back to interactive prompt
- `unavailable` + iron gate closed → deny with retry message
- `unavailable` + iron gate open → fail open, return original 'ask' result
- `transcriptTooLong` → fall back to interactive prompt (deterministic, no retry)

**`autoModeState.ts`** (`src/utils/permissions/autoModeState.ts`): Module-level flags for `autoModeActive`, `autoModeFlagCli`, and `autoModeCircuitBroken`. Circuit breaker is set when a GrowthBook gate check returns `enabled=false`, blocking all future auto-mode activation for the session.

**Auto mode configuration** (`getAutoModeConfig()` in settings.ts): Merges `autoMode.allow`, `autoMode.soft_deny`, and `autoMode.environment` arrays from userSettings, localSettings, flagSettings, policySettings (excludes projectSettings for security). These are injected into the classifier prompt to customize its behavior.

**Dangerous rule stripping** at auto-mode entry (`permissionSetup.ts`): When transitioning to auto mode, `isDangerousBashPermission()` scans existing allow rules. Rules like `Bash(python:*)`, `Bash(node:*)`, `Bash(eval:*)` — patterns from `DANGEROUS_BASH_PATTERNS` — are stripped and stored in `strippedDangerousRules`. This prevents pre-authorized code execution from bypassing the classifier.

### 1.4 Denial Tracking

`src/utils/permissions/denialTracking.ts` implements loop prevention:

```typescript
DENIAL_LIMITS = {
  maxConsecutive: 3,   // 3 consecutive denials → fall back to prompt
  maxTotal: 20,        // 20 total denials in session → fall back to prompt
}
```

State is immutable:
- `createDenialTrackingState()` → `{ consecutiveDenials: 0, totalDenials: 0 }`
- `recordDenial(state)` → increments both counters
- `recordSuccess(state)` → resets `consecutiveDenials` to 0 (total never decreases)
- `shouldFallbackToPrompting(state)` → `consecutiveDenials >= 3 || totalDenials >= 20`

For sync agents (`context.localDenialTracking`), denial state is local to the agent call. For interactive sessions, it lives in `AppState.denialTracking` and is persisted via `persistDenialState()` which calls `context.setAppState()`.

When `shouldFallbackToPrompting()` is true and the classifier has just blocked something, `handleDenialLimitExceeded()` escalates to a user prompt with the classifier's reason included in the message.

### 1.5 Dangerous Patterns

`src/utils/permissions/dangerousPatterns.ts` defines two exported lists:

**`CROSS_PLATFORM_CODE_EXEC`** — interpreters and package runners on both Unix and Windows:
- Interpreters: python, python3, python2, node, deno, tsx, ruby, perl, php, lua
- Package runners: npx, bunx, npm run, yarn run, pnpm run, bun run
- Shells: bash, sh
- Remote execution: ssh

**`DANGEROUS_BASH_PATTERNS`** — `CROSS_PLATFORM_CODE_EXEC` plus Unix-only:
- Additional shells: zsh, fish
- Execution builtins: eval, exec, env, xargs
- Privilege escalation: sudo
- Ant-only additions (guarded by `process.env.USER_TYPE === 'ant'`): fa run, coo, gh, gh api, curl, wget, git, kubectl, aws, gcloud, gsutil

These patterns are used by `isDangerousBashPermission()` and `isDangerousPowerShellPermission()` in `permissionSetup.ts` to strip overly-broad allow rules at auto-mode entry. The matching handles four shapes: exact match, `:*` prefix, trailing `*` wildcard, and ` *`/` -...*` variants.

There is a separate `allowManagedPermissionRulesOnly` policy setting (`src/utils/permissions/permissionsLoader.ts:30`) that, when true in policySettings, causes `shouldShowAlwaysAllowOptions()` to return false — hiding the "always allow" UI option and restricting which rules can be added.

---

## 2. Authentication & Security

### 2.1 OAuth 2.0 + PKCE Flow

The OAuth implementation lives in `src/services/oauth/` and uses standard Authorization Code + PKCE (RFC 7636).

**PKCE cryptography** (`src/services/oauth/crypto.ts`):
- `generateCodeVerifier()`: 32 random bytes → base64URL-encoded (no padding, `+`→`-`, `/`→`_`)
- `generateCodeChallenge(verifier)`: SHA-256 hash of verifier → base64URL-encoded
- `generateState()`: 32 random bytes → base64URL-encoded (CSRF protection)

**Authorization URL construction** (`buildAuthUrl()` in `src/services/oauth/client.ts:46`):
- Base URL: `CONSOLE_AUTHORIZE_URL` or `CLAUDE_AI_AUTHORIZE_URL` depending on `loginWithClaudeAi` flag
- Parameters: `client_id`, `response_type=code`, `redirect_uri`, `scope`, `code_challenge`, `code_challenge_method=S256`, `state`
- Optional: `orgUUID`, `login_hint` (pre-fills email), `login_method` (sso/magic_link/google)
- Scope variants: `inferenceOnly` → only `CLAUDE_AI_INFERENCE_SCOPE`; otherwise `ALL_OAUTH_SCOPES`

**Redirect capture** (`src/services/oauth/auth-code-listener.ts`):
The `AuthCodeListener` class creates a temporary local HTTP server on a random OS-assigned port (`listen(0)`). The server validates:
1. Path matches configured callback path (`/callback` default)
2. `state` parameter matches stored `expectedState` (CSRF protection)
3. Authorization code is present

The server stores the `ServerResponse` object for a deferred redirect after token exchange succeeds.

**Token exchange** (`exchangeCodeForTokens()`, client.ts:107):
- POST to `TOKEN_URL` with `grant_type=authorization_code`, code, verifier, client_id, redirect_uri, state
- 15-second timeout
- Response: `{ access_token, refresh_token, expires_in, scope }`
- Fires `tengu_oauth_token_exchange_success` analytics event

**Complete login sequence**:

```
User runs /login
    │
    ├─ generateCodeVerifier() + generateCodeChallenge()
    ├─ generateState()
    ├─ AuthCodeListener.start() → binds to localhost:PORT
    ├─ buildAuthUrl() → opens browser to authorize URL
    │
Browser:
    ├─ User authenticates at claude.ai/console
    └─ Redirect to http://localhost:PORT/callback?code=AUTH_CODE&state=STATE
    │
    ├─ AuthCodeListener validates state
    ├─ exchangeCodeForTokens(code, state, verifier, port)
    ├─ fetchProfileInfo(accessToken) → subscription type, billing type
    ├─ saveOAuthTokensIfNeeded() → stores in secure storage
    ├─ optionally createAndStoreApiKey() → creates API key via API
    ├─ storeOAuthAccountInfo() → saves to global config (config.json)
    └─ AuthCodeListener.handleSuccessRedirect() → redirects browser to success page
```

**Scope detection**: `shouldUseClaudeAIAuth(scopes)` (client.ts:38) checks for `CLAUDE_AI_INFERENCE_SCOPE` in the token's scopes. This gate controls whether the bearer `Authorization` header uses OAuth format vs. API key format in HTTP requests.

### 2.2 API Key Resolution Chain

`getAnthropicApiKeyWithSource()` in `src/utils/auth.ts:226` implements a strict priority chain. The function is called both synchronously (for cached values) and with `skipRetrievingKeyFromApiKeyHelper` to avoid blocking.

**Priority order for non-bare-mode** (most to least privileged):

```
1. isBareMode() branch:
   - ANTHROPIC_API_KEY env var
   - apiKeyHelper from --settings flag only
   
2. Normal mode:
   a. preferThirdPartyAuthentication() && ANTHROPIC_API_KEY env → return (CI/--print mode)
   b. CI/test env:
      - getApiKeyFromFileDescriptor() (FD from CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR)
      - ANTHROPIC_API_KEY env var (throws if neither present)
   c. ANTHROPIC_API_KEY env + approved in globalConfig.customApiKeyResponses.approved
   d. getApiKeyFromFileDescriptor() (file descriptor path)
   e. apiKeyHelper (configured helper command, sync cache)
   f. getApiKeyFromConfigOrMacOSKeychain() (keychain or config file)
   → { key: null, source: 'none' }
```

**apiKeyHelper caching** (auth.ts:456-536): Stale-while-revalidate pattern with epoch invalidation:
- Fresh cache (within TTL) → return cached value
- Stale cache → return stale value immediately, fire background refresh (SWR)
- Cold cache → deduplicate concurrent calls via `_apiKeyHelperInflight` promise
- On failure: stale-while-error returns existing good value; cold failure caches `' '` sentinel to prevent OAuth fallback

**File descriptor path** (`src/utils/authFileDescriptor.ts`): Two-step fallback:
1. Read from `/dev/fd/N` (macOS/BSD) or `/proc/self/fd/N` (Linux) using the FD number from env var `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`
2. If FD read fails (subprocess can't inherit pipe), read from well-known CCR file `/home/claude/.claude/remote/.api_key`
On successful FD read, the token is persisted to the well-known file for subprocess access.

**Auth token source** (`getAuthTokenSource()`, auth.ts:152): Determines the auth bearer token source (distinct from API key):

```
--bare mode → apiKeyHelper or none
ANTHROPIC_AUTH_TOKEN env (not managed) → 'ANTHROPIC_AUTH_TOKEN'
CLAUDE_CODE_OAUTH_TOKEN env → 'CLAUDE_CODE_OAUTH_TOKEN'
CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR → 'CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR'
CCR disk fallback → 'CCR_OAUTH_TOKEN_FILE'
apiKeyHelper configured → 'apiKeyHelper'
claude.ai OAuth tokens → 'claude.ai'
→ 'none'
```

### 2.3 macOS Keychain Integration

**Storage backend selection** (`src/utils/secureStorage/index.ts:9`):
- `darwin`: `createFallbackStorage(macOsKeychainStorage, plainTextStorage)` — tries keychain first, falls back to plaintext
- Other platforms: `plainTextStorage` only (TODO: add libsecret for Linux)

**macOS Keychain implementation** (`src/utils/secureStorage/macOsKeychainStorage.ts`):

Uses the macOS `security` CLI to interact with the system keychain:
- Read: `security find-generic-password -a USERNAME -w -s SERVICE_NAME`
- Write: `security add-generic-password -U -a USERNAME -s SERVICE_NAME -X HEX_VALUE`
- Delete: `security delete-generic-password -a USERNAME -s SERVICE_NAME`

Data is JSON serialized and hex-encoded before writing to defeat naive plaintext-grep rules in security monitoring tools (e.g., CrowdStrike). The hex encoding also avoids shell escaping issues.

**Write strategy** (line 115-126): Uses `security -i` (stdin mode) when the hex payload fits within `SECURITY_STDIN_LINE_LIMIT = 4096 - 64` bytes, to prevent credential payload from appearing in process monitor argument lists (CrowdStrike INC-3028). Falls back to argv when payload exceeds the limit.

**Caching** (`src/utils/secureStorage/macOsKeychainHelpers.ts`): Module-level `keychainCacheState` with TTL. Stale-while-error: if a `security` subprocess fails but we had a valid prior value, serve the stale value and reset the TTL — prevents transient spawn failures from causing "Not logged in" errors.

**Async read** (`readAsync()`): Single in-flight deduplication via `keychainCacheState.readInFlight`. Uses generation counter to discard results from superseded reads.

**Keychain lock detection** (`isMacOsKeychainLocked()`): Calls `security show-keychain-info` and checks for exit code 36. Cached for process lifetime since lock state doesn't change during a session. Used to show an unlock hint in SSH sessions where the keychain isn't automatically unlocked.

**Plaintext fallback** (`src/utils/secureStorage/plainTextStorage.ts`): Writes to `~/.claude/.credentials.json` with `chmod 600`. The file has no encryption — it's a fallback for platforms without system keychain and for CCR environments.

**`SecureStorageData`**: The object stored contains `{ oauthTokens?: OAuthTokens }`. `OAuthTokens` holds `{ accessToken, refreshToken, expiresAt, scopes, subscriptionType, rateLimitTier }`.

### 2.4 AWS Bedrock / STS Authentication

AWS authentication flows through `src/utils/aws.ts` and `src/utils/auth.ts`.

**STS verification** (`checkStsCallerIdentity()`, aws.ts:49): Dynamically imports `@aws-sdk/client-sts` and calls `GetCallerIdentityCommand`. Used during setup to verify AWS credentials are valid before saving them.

**Credential cache clearing** (`clearAwsIniCache()`, aws.ts:60): Forces a refresh of the `@aws-sdk/credential-providers` `fromIni` provider cache, picking up any changes to `~/.aws/credentials` without restarting.

**Settings-based auth helpers**:
- `awsAuthRefresh` setting: a command run periodically to refresh STS credentials
- `awsCredentialExport` setting: a command that exports AWS credentials as environment variables

Project/local settings can configure these commands, but callers check `isAwsAuthRefreshFromProjectSettings()` and `isAwsCredentialExportFromProjectSettings()` to display appropriate trust warnings.

**`AwsAuthStatusManager`** (`src/utils/awsAuthStatusManager.ts`): Tracks the current AWS auth status across the session, managing the periodic STS refresh cycle.

### 2.5 Token Refresh

**Expiry check** (`isOAuthTokenExpired()`, client.ts:344): Adds a 5-minute buffer — a token is considered expired when `now + 5min >= expiresAt`. This proactive refresh window prevents auth failures from race conditions at exact expiry.

**Refresh flow** (`refreshOAuthToken()`, client.ts:146):
1. POST to TOKEN_URL with `grant_type=refresh_token`, `refresh_token`, `client_id`, and requested scopes
2. Scopes default to `CLAUDE_AI_OAUTH_SCOPES` — the backend allows scope expansion beyond the original grant
3. Optimization check: if `globalConfig.oauthAccount` already has `billingType`, `accountCreatedAt`, and `subscriptionCreatedAt`, AND secure storage has `subscriptionType` and `rateLimitTier`, skip the `/api/oauth/profile` round-trip (saves ~7M req/day fleet-wide)
4. Otherwise, call `fetchProfileInfo()` → `getOauthProfileFromOauthToken()` to get updated subscription/billing data
5. Merge new profile data into `globalConfig.oauthAccount` if changed
6. Returns new `OAuthTokens` with updated `accessToken`, `expiresAt`, `scopes`, profile data

**Automatic refresh** (`checkAndRefreshOAuthTokenIfNeeded()` in auth.ts): Called before any operation that needs a valid token. Uses a memoized async wrapper to deduplicate concurrent refresh calls.

**`CLAUDE_CODE_OAUTH_REFRESH_TOKEN` path**: When this env var is set, `installOAuthTokens()` performs a login using just the refresh token — calls `refreshOAuthToken()`, stores the result, then clears any existing stored tokens with `performLogout()` before setting the new tokens. This prevents the stale subscription type bug where a re-login path would wipe secure storage after the refresh had already used stale data.

---

## 3. Configuration System

### 3.1 Settings Source Hierarchy

Settings are loaded from five sources in priority order (lowest to highest):

```
Source            File/Location                          Priority
----------------- -------------------------------------- --------
userSettings      ~/.claude/settings.json                1 (low)
projectSettings   $CWD/.claude/settings.json             2
localSettings     $CWD/.claude/settings.local.json       3
flagSettings      --settings <path> or SDK inline        4
policySettings    [remote | MDM | file | HKCU]           5 (high)
```

**Cowork mode exception**: When `CLAUDE_CODE_USE_COWORK_PLUGINS` is set or `--cowork` flag is passed, `userSettings` reads from `cowork_settings.json` instead of `settings.json`.

**policySettings internal priority** (first-source-wins, highest to lowest):
```
remote managed settings (API-delivered)
  └─ HKLM\SOFTWARE\Policies\ClaudeCode (Windows admin)
       └─ /Library/Managed Preferences/com.anthropic.claudecode.plist (macOS MDM)
            └─ managed-settings.json + managed-settings.d/*.json (file-based)
                 └─ HKCU\SOFTWARE\Policies\ClaudeCode (Windows user-writable)
```

The `getEnabledSettingSources()` function in `src/utils/settings/constants.ts:159` respects the `--setting-sources` CLI flag which can restrict which sources are loaded (used in SDK/bare mode scenarios). `policySettings` and `flagSettings` are always included regardless of flag.

**Merge semantics** (`settingsMergeCustomizer()`): Arrays are concatenated and deduplicated (`uniq`). Non-array values use lodash `mergeWith` default (deep merge for objects, right-side-wins for primitives). This means a later settings source can add permission rules without removing earlier ones — a user can layer rules from project settings on top of user settings.

**Security restriction**: `projectSettings` is deliberately excluded from security-sensitive lookups:
- `hasSkipDangerousModePermissionPrompt()` — only checks user/local/flag/policy
- `hasAutoModeOptIn()` — only checks user/local/flag/policy
- `getAutoModeConfig()` — only checks user/local/flag/policy
- `getUseAutoModeDuringPlan()` — only checks user/local/flag/policy

A malicious `.claude/settings.json` in a cloned repository cannot grant bypass permissions or enable auto mode.

### 3.2 Settings Loading & Merging

**`loadSettingsFromDisk()`** in `src/utils/settings/settings.ts:645`:

1. Start with `pluginSettings` as lowest-priority base (allowlisted keys only: `agent`, etc.)
2. For each source in `getEnabledSettingSources()` order:
   - For `policySettings`: use first-source-wins (remote → MDM → file → HKCU)
   - For others: `parseSettingsFile(filePath)` with deduplication by resolved path (prevents symlink double-loading)
   - For `flagSettings`: also merge `getFlagSettingsInline()` (SDK-provided inline settings)
3. Each `parseSettingsFile()` call: reads file → `safeParseJSON` → `filterInvalidPermissionRules()` (strips invalid rules without failing) → `SettingsSchema().safeParse()` (Zod validation)
4. Merge with `mergeWith(existing, new, settingsMergeCustomizer)`

Recursion guard: `isLoadingSettings` flag prevents `loadSettingsFromDisk → logEvent → getGlobalConfig → loadSettingsFromDisk` cycles.

**Session-level cache** (`getSessionSettingsCache()` / `setSessionSettingsCache()`): Settings are cached for the entire session in module state. Cache is only invalidated by `resetSettingsCache()` which is called by `updateSettingsForSource()` after writes and by the change detector's `fanOut()` function.

**`getSettingsForSource(source)`**: Per-source cache on top of the full-merge cache. The per-source cache is also reset by `resetSettingsCache()`.

**`updateSettingsForSource(source, settings)`** (line 416): Writes back to the settings file. Uses `getSettingsForSourceUncached()` to bypass cache and get the real-disk state. Falls back to raw JSON parse if Zod validation fails (to preserve existing settings). Merges new settings with `undefined`-as-deletion semantics (set key to undefined → delete from file). Marks the write as internal with `markInternalWrite()` before touching the file to prevent the change detector from firing an external-change notification.

### 3.3 Live Settings Reload

`src/utils/settings/changeDetector.ts` implements file watching using **chokidar** and **MDM polling**:

**Chokidar setup**:
- Watches parent directories of all settings files (not the files themselves, to detect creation)
- `depth: 0` — immediate children only
- `awaitWriteFinish: { stabilityThreshold: 1000ms, pollInterval: 500ms }` — waits for writes to stabilize
- Ignores non-file entries (sockets, FIFOs, devices → `EOPNOTSUPP` prevention), `.git/` directories
- Also watches `managed-settings.d/` drop-in directory for policy fragments

**Event handling**:
- `change`: check if internal write (`consumeInternalWrite()`); if not external, fire `ConfigChange` hook, then call `fanOut(source)` if not blocked
- `add`: cancels pending deletion, treats as change (handles delete-and-recreate pattern)
- `delete`: deferred by `DELETION_GRACE_MS = 1700ms` — if file reappears within grace period, cancels deletion. Handles atomic writes that temporarily remove the file.

**MDM polling**: Every 30 minutes, fires `refreshMdmSettings()` (plutil/reg query subprocess), JSON-stringifies the result as a snapshot, and compares to the last snapshot. On change, updates MDM cache and calls `fanOut('policySettings')`.

**`fanOut(source)`** (line 437): The single point of cache invalidation — calls `resetSettingsCache()` once before notifying subscribers. Previously listeners reset defensively, causing N-way thrashing (N listeners × N disk reloads per notification). Centralizing the reset means one notification = one disk reload.

**Subscribers**: React components use `useSettingsChange` hook which subscribes to `settingsChanged` signal. The signal pattern uses `createSignal()` from `src/utils/signal.ts` — a publish-subscribe system.

### 3.4 Enterprise MDM Controls

**macOS MDM** (`src/utils/settings/mdm/`): Uses `plutil -convert json -o - -- <plist>` to read the `com.anthropic.claudecode` preference domain. Checks three paths in priority order:
1. `/Library/Managed Preferences/<username>/com.anthropic.claudecode.plist` (per-user managed)
2. `/Library/Managed Preferences/com.anthropic.claudecode.plist` (device-level managed)
3. `~/Library/Preferences/com.anthropic.claudecode.plist` (ant-only, for testing)

Paths are checked with `existsSync()` before spawning `plutil` (saves ~5ms per non-MDM machine).

**Windows MDM**: Two registry keys via `reg query`:
- `HKLM\SOFTWARE\Policies\ClaudeCode` (admin-only, under `SOFTWARE\Policies` which is on the WOW64 shared key list)
- `HKCU\SOFTWARE\Policies\ClaudeCode` (user-writable, lowest MDM priority)
JSON settings stored in `Settings` REG_SZ/REG_EXPAND_SZ value.

**File-based MDM** (Linux + cross-platform): Two locations:
- `managed-settings.json` — base file
- `managed-settings.d/*.json` — drop-in fragments, sorted alphabetically (later files win, like systemd drop-ins)
Multiple teams can maintain independent policy files without coordinating edits.

**Startup optimization** (`src/utils/settings/mdm/rawRead.ts`): `startMdmRawRead()` fires at `main.tsx` module evaluation — before the event loop even starts processing. The plutil/reg subprocesses run in parallel with JS module loading. By the time settings are first needed, the MDM data is already in memory. `fireRawRead()` checks `existsSync()` before spawning plutil to skip the ~5ms spawn overhead on machines with no MDM plist.

**`allowManagedPermissionRulesOnly`**: When set to `true` in policySettings, only permission rules from policySettings are honored — user/project/local rules are effectively ignored for allow decisions. The UI hides the "Always allow" options.

**`disableBypassPermissionsMode: 'disable'`**: Schema field that prevents users from entering bypassPermissions mode.

**`disableAutoMode: 'disable'`**: Schema field (TRANSCRIPT_CLASSIFIER builds only) that prevents auto mode.

**Remote managed settings**: Separate service (`src/services/remoteManagedSettings/`) delivers managed settings over the network with signed JSON, taking priority over all local MDM sources.

---

## 4. Analytics & Telemetry

### 4.1 Zero-Startup-Cost Event Queue

`src/services/analytics/index.ts` implements a zero-dependency pre-sink queue:

**Design principle**: The module has NO dependencies (not even logger) to avoid import cycles. Events logged during module loading (before the app has fully initialized) are queued in memory.

```typescript
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

export function logEvent(eventName, metadata) {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

**Sink attachment** (`attachAnalyticsSink()`, line 95):
- Idempotent — second call is no-op (safe to call from both `preAction` hook and `setup()`)
- Drains queue via `queueMicrotask()` to avoid blocking the startup path
- For ant builds, logs `analytics_sink_attached` with the queue size for debugging analytics timing

**Sink initialization** (`src/services/analytics/sink.ts:109`): Called from `setupBackend()` in `main.tsx`. Reads the Datadog gate (`tengu_log_datadog_events`) from GrowthBook cached value, then calls `attachAnalyticsSink()`.

**Event routing** (`logEventImpl()`):
1. Check `shouldSampleEvent(eventName)` — if sampled, get rate; if not selected, drop event
2. If Datadog gate enabled (and sink not killed): `trackDatadogEvent()` with `_PROTO_*` keys stripped
3. Always: `logEventTo1P()` with full payload (1P exporter handles `_PROTO_*` routing)

**Sampling** (`shouldSampleEvent()` in firstPartyEventLogger.ts): Reads `tengu_event_sampling_config` dynamic config from GrowthBook (a JSON object mapping event names to `{ sample_rate: 0-1 }`). Events not in the config pass at 100%. Returns `null` (log, no annotation) or the sample rate (log with `sample_rate` in metadata) or `0` (drop).

**Per-sink killswitch** (`src/services/analytics/sinkKillswitch.ts`): GrowthBook JSON config `tengu_frond_boric` (mangled name) maps `{ datadog?: boolean, firstParty?: boolean }`. Each sink checks `isSinkKilled(name)` before dispatching.

### 4.2 PII Safety Mechanism

The `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type (index.ts:19) is defined as `never`:

```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```

**How it works**: The `logEvent()` metadata parameter only accepts `{ [key: string]: boolean | number | undefined }` — **no strings**. To include a string value, callers must use a type assertion:

```typescript
logEvent('tengu_auto_mode_decision', {
  decision: 'allowed' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  toolName: sanitizeToolNameForAnalytics(tool.name),
})
```

TypeScript allows `never` casts but the cast is a visible signal in code review that the developer has verified the value doesn't contain file paths or code snippets. This is enforced at compile time — non-cast strings won't type-check.

**PII-tagged values** (`AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never`): For values intended for privileged BigQuery columns. Keys prefixed `_PROTO_*` are stripped by `stripProtoFields()` before reaching Datadog. The 1P exporter hoists `_PROTO_*` keys to top-level proto fields. A defensive `stripProtoFields()` call in the exporter also catches any unrecognized `_PROTO_` keys.

**`sanitizeToolNameForAnalytics()`** (`src/services/analytics/metadata.ts`): MCP tool names include arbitrary server names — this function strips server-specific parts to prevent accidental information leakage through tool names.

### 4.3 GrowthBook Feature Flag Integration

`src/services/analytics/growthbook.ts` integrates the GrowthBook SDK for A/B testing and feature flags.

**Client creation** (`createGrowthBookClient()`): Creates a `GrowthBook` instance with:
- User attributes: `id` (device UUID), `sessionId`, `platform`, `organizationUUID`, `accountUUID`, `userType`, `subscriptionType`, `rateLimitTier`, `appVersion`, `github` metadata
- Remote eval mode — features evaluated server-side for security
- `trackingCallback`: logs exposure events to 1P analytics

**Initialization** (`initGrowthBook()`): Called during startup. Fetches features from the GrowthBook CDN via `loadFeatures()` with auth headers. Stores results in `remoteEvalFeatureValues` map (workaround for SDK not reliably respecting remote eval response).

**Caching strategy**: Two-level caching:
1. Module-level `remoteEvalFeatureValues` Map — in-process for current session
2. `globalConfig.statsigCache` (stored in `config.json`) — persists across sessions for startup before network

On startup, stale cached values from the previous session are served immediately (avoid data loss for early events). Fresh values arrive when `loadFeatures()` completes.

**Refresh listeners** (`onGrowthBookRefresh()`): Subscribe to feature value refreshes. Used by `firstPartyEventLogger` which builds a `LoggerProvider` once and needs to rebuild when batch config changes. Includes a catch-up mechanism: if `remoteEvalFeatureValues` is already populated when a subscriber registers (fast network + slow REPL mount race), the listener fires on the next microtask.

**Auth-change recreation**: When the user logs in/out, `resetGrowthBook()` clears the client and `initGrowthBook()` is called again with the new auth headers. A `reinitializingPromise` prevents stale gate values from being served during re-initialization.

**`getFeatureValue_CACHED_WITH_REFRESH(feature, default, ttl)`**: For security-sensitive gates (like `tengu_iron_gate_closed` for classifier fail-closed), uses a TTL-based refresh that checks GrowthBook at most once per 30 minutes. This ensures the gate value doesn't become permanently stale if the client fails to reconnect.

**Security gate checks** (`checkSecurityRestrictionGate()`): Used for `disableBypassPermissions` and `disableAutoMode` checks. Awaits `reinitializingPromise` before returning to ensure fresh values post-login.

### 4.4 OpenTelemetry / Perfetto Tracing

**Perfetto tracing** (`src/utils/telemetry/perfettoTracing.ts`): Ant-only feature (eliminated from external builds by `feature('TRANSCRIPT_CLASSIFIER')` guards). Activated by `CLAUDE_CODE_PERFETTO_TRACE=1` or `=<path>`. Generates Chrome Trace Event Format files (`~/.claude/traces/trace-<session-id>.json`).

Trace events cover:
- Agent hierarchy (parent-child via `parentSessionId`)
- API requests (TTFT, TTLT, prompt lengths, cache stats, message IDs, speculative flag)
- Tool executions (name, duration, token usage)
- User input waiting time

Process IDs map to swarm agents; thread IDs are `djb2Hash(agentName)` to keep agent lanes visually distinct in Perfetto UI.

**Beta session tracing** (`src/utils/telemetry/betaSessionTracing.ts`): OpenTelemetry-based tracing via `feature('ENHANCED_TELEMETRY_BETA')`. Creates spans for interactions, LLM requests, tool calls, hooks. Activated by `OTEL_TRACES_EXPORTER` env var. Content is truncated before being included as span attributes.

**Session tracing** (`src/utils/telemetry/sessionTracing.ts`): Unified coordinator that delegates to both Perfetto and beta OTEL tracing. Uses `AsyncLocalStorage` for context propagation through async chains without explicit parameter threading.

**BigQuery exporter** (`src/utils/telemetry/bigqueryExporter.ts`): Internal Anthropic analytics pipeline for detailed request/response logging. Separate from the GrowthBook/Datadog analytics path.

**Plugin telemetry** (`src/utils/telemetry/pluginTelemetry.ts`): Wraps OTEL API for plugin-provided telemetry, isolating plugin spans from core spans.

---

## 5. Cost & Billing

### 5.1 Real-Time Cost Tracking

`src/cost-tracker.ts` provides the public API for cost tracking. State lives in `src/bootstrap/state.ts` as module-level counters (not React state) to avoid rendering overhead.

**`addToTotalSessionCost(cost, usage, model)`** (cost-tracker.ts:278):
1. Calls `addToTotalModelUsage(cost, usage, model)` → builds/updates per-model usage map
2. Calls `addToTotalCostState(cost, modelUsage, model)` → updates global running totals in `state.ts`
3. Reports to OpenTelemetry cost/token counters (`getCostCounter()`, `getTokenCounter()`)
4. Handles fast-mode variants (separate token type attribution for `speed: 'fast'` usage)
5. Recursively processes advisor model usage (inner model calls within extended thinking)

**State persistence** (`saveCurrentSessionCosts()`): Called before session switches. Saves to `projectConfig` (per-working-directory config):
- `lastCost`, `lastAPIDuration`, `lastToolDuration`, `lastDuration`
- `lastLinesAdded`, `lastLinesRemoved`
- `lastModelUsage` (per-model token breakdown)
- `lastSessionId` (for session-identity validation on restore)
- FPS metrics: `lastFpsAverage`, `lastFpsLow1Pct`

**State restoration** (`restoreCostStateForSession()`): Called when resuming a session. Only restores if `projectConfig.lastSessionId === sessionId`. Reconstructs `ModelUsage` with context window info added back.

### 5.2 Token Aggregation

`src/utils/modelCost.ts` defines pricing tiers as constants:

```typescript
COST_TIER_3_15   = { inputTokens: $3, outputTokens: $15 }      // Sonnet standard
COST_TIER_15_75  = { inputTokens: $15, outputTokens: $75 }     // Opus 4/4.1
COST_TIER_5_25   = { inputTokens: $5, outputTokens: $25 }      // Opus 4.5
COST_TIER_30_150 = { inputTokens: $30, outputTokens: $150 }    // Opus 4.6 fast mode
COST_HAIKU_35    = { inputTokens: $0.80, outputTokens: $4 }
COST_HAIKU_45    = { inputTokens: $1, outputTokens: $5 }
```

All prices are per million tokens. Prompt cache write/read rates and web search request prices are included in each tier.

**`calculateUSDCost(model, usage)`**: Looks up the `ModelCosts` configuration for the model canonical name, then:
```
cost = (inputTokens × inputRate + outputTokens × outputRate
      + cacheWriteTokens × writeRate + cacheReadTokens × readRate
      + webSearchRequests × searchRate) / 1_000_000
```

Unknown models log a `tengu_unknown_model_cost` analytics event and use `DEFAULT_UNKNOWN_MODEL_COST = COST_TIER_5_25`. The `hasUnknownModelCost()` flag causes a warning suffix in `formatTotalCost()`.

**`calculateCostFromTokens(model, usage)`**: Used specifically for classifier overhead telemetry — calculates the cost of the auto-mode classifier's API calls for overhead percentage analysis.

### 5.3 Rate Limits and Billing Tiers

`src/services/claudeAiLimits.ts` handles rate limit tracking for Claude.ai subscribers:

**Rate limit types**: `five_hour` (session limit), `seven_day` (weekly limit), `seven_day_opus`, `seven_day_sonnet`, `overage`.

**Early warning system**: Checks utilization thresholds before the server sends a hard rejection:
- 5-hour: warn at 90% utilization when ≤72% of window has elapsed
- 7-day: warn at 75%/50%/25% utilization at 60%/35%/15% elapsed respectively

**Header-based detection**: Parses `x-claude-ratelimit-*` headers from API responses to get current utilization and reset timestamps. Falls back to the time-progress heuristic when server headers are absent.

**Billing access gating** (`src/utils/billing.ts`):

`hasConsoleBillingAccess()` — for API Console users:
- Always false for Claude.ai subscribers (no per-token billing)
- Requires `organizationRole ∈ {admin, billing}` OR `workspaceRole ∈ {workspace_admin, workspace_billing}`
- False if no authentication or `DISABLE_COST_WARNINGS` env is set

`hasClaudeAiBillingAccess()` — for Claude.ai subscribers:
- True for `max` and `pro` plans (consumer plans, individual billing)
- For `team`/`enterprise`: requires `organizationRole ∈ {admin, billing, owner, primary_owner}`

This controls whether the cost display appears in the status bar and whether usage warnings are shown.

---

## 6. Essential Files Reference

The following files are the absolute core of the security, auth, and configuration domains:

### Permissions & Safety

- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/permissions.ts` — Main permission gate (`hasPermissionsToUseTool`, `hasPermissionsToUseToolInner`). Decision tree lines 1158-1319.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/types/permissions.ts` — All permission type definitions: `PermissionMode`, `ToolPermissionContext`, `PermissionDecision`, `YoloClassifierResult`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/PermissionMode.ts` — Mode metadata (title, symbol, color, external mapping).
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/permissionRuleParser.ts` — `permissionRuleValueFromString()` / `permissionRuleValueToString()` — rule string parsing with escaped parentheses.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/denialTracking.ts` — Anti-loop limits: 3 consecutive / 20 total.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/dangerousPatterns.ts` — `DANGEROUS_BASH_PATTERNS`, `CROSS_PLATFORM_CODE_EXEC` lists.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/yoloClassifier.ts` — AI classifier implementation, two-stage architecture, prompt loading.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/classifierDecision.ts` — `SAFE_YOLO_ALLOWLISTED_TOOLS` set, `isAutoModeAllowlistedTool()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/autoModeState.ts` — Module-level auto mode flags and circuit breaker.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/permissionSetup.ts` — Dangerous rule detection, auto-mode entry logic, `isDangerousBashPermission()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/bypassPermissionsKillswitch.ts` — `checkAndDisableBypassPermissionsIfNeeded()`, GrowthBook gate integration.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/shellRuleMatching.ts` — `ShellPermissionRule` type, exact/prefix/wildcard matching for Bash/PowerShell.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/permissions/permissionsLoader.ts` — `shouldAllowManagedPermissionRulesOnly()`, rule persistence helpers.

### Authentication & Security

- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/auth.ts` — `getAnthropicApiKeyWithSource()` resolution chain (lines 226-348), `getAuthTokenSource()`, `isAnthropicAuthEnabled()`, apiKeyHelper SWR cache.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/oauth/client.ts` — `buildAuthUrl()`, `exchangeCodeForTokens()`, `refreshOAuthToken()`, `isOAuthTokenExpired()`, `fetchProfileInfo()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/oauth/crypto.ts` — PKCE: `generateCodeVerifier()`, `generateCodeChallenge()`, `generateState()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/oauth/auth-code-listener.ts` — `AuthCodeListener` class — temporary localhost server for OAuth redirect capture with CSRF state validation.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/secureStorage/index.ts` — Platform dispatch: darwin→keychain+fallback, others→plaintext.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/secureStorage/macOsKeychainStorage.ts` — `security` CLI interaction, hex encoding, stdin/argv overflow handling, stale-while-error caching.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/secureStorage/plainTextStorage.ts` — `~/.claude/.credentials.json` with `chmod 600`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/authFileDescriptor.ts` — File descriptor credential reading, CCR well-known file fallback.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/aws.ts` — `checkStsCallerIdentity()`, `clearAwsIniCache()`, STS credential types.

### Configuration System

- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/settings.ts` — `loadSettingsFromDisk()`, `getSettingsForSource()`, `updateSettingsForSource()`, `getSettingsWithSources()`, security-restricted lookups.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/types.ts` — `SettingsJson` Zod schema (all configurable fields), `PermissionsSchema`, `EnvironmentVariablesSchema`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/constants.ts` — `SETTING_SOURCES`, `SettingSource` type, `getEnabledSettingSources()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/changeDetector.ts` — chokidar file watching, MDM polling, delete grace period, internal write suppression, `fanOut()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/mdm/settings.ts` — MDM parsing, first-source-wins policy, startup async load.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/mdm/rawRead.ts` — `fireRawRead()` — minimal-import subprocess spawner for plutil/reg.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/settings/mdm/constants.ts` — macOS plist paths, Windows registry key paths, `plutil` invocation args.

### Analytics & Telemetry

- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/analytics/index.ts` — `logEvent()`, `attachAnalyticsSink()`, event queue, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type, `stripProtoFields()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/analytics/sink.ts` — `initializeAnalyticsSink()`, Datadog gate, sampling, `_PROTO_*` stripping, dual-sink routing.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/analytics/growthbook.ts` — GrowthBook client, `onGrowthBookRefresh()`, `getFeatureValue_CACHED_WITH_REFRESH()`, remote eval, auth-change reset.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/analytics/firstPartyEventLogger.ts` — OpenTelemetry `LoggerProvider`, event sampling config, `shouldSampleEvent()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/analytics/sinkKillswitch.ts` — Per-sink killswitch via GrowthBook JSON config.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/telemetry/perfettoTracing.ts` — Chrome Trace Event format, agent hierarchy, span types (ant-only).
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/telemetry/sessionTracing.ts` — Unified tracing coordinator, `AsyncLocalStorage` context propagation.

### Cost & Billing

- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/cost-tracker.ts` — `addToTotalSessionCost()`, `saveCurrentSessionCosts()`, `restoreCostStateForSession()`, `formatTotalCost()`.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/modelCost.ts` — Pricing tiers, `calculateUSDCost()`, `calculateCostFromTokens()`, unknown model fallback.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/utils/billing.ts` — `hasConsoleBillingAccess()`, `hasClaudeAiBillingAccess()`, role-based gating.
- `/Users/simonscheurer/Development/agentic-coding/claude-code/src/services/claudeAiLimits.ts` — Rate limit types, early warning thresholds, header parsing, utilization calculation.

---
