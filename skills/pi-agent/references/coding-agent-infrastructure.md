# `@earendil-works/pi-coding-agent` SDK & Infrastructure

The coding-agent product SDK: sessions, built-in coding tools, extensions, settings, resource loading, prompt templates, and the interactive/RPC/print modes. Export lists below were verified against the published **`0.84.1`** package types.

> [!IMPORTANT]
> **This is no longer the only home for sessions/compaction/tools.** `pi-agent-core@0.84` ships its own harness layer (durable `Session`, `AgentHarness`, compaction, `ExecutionEnv`, read/write/edit/bash tools) — see `agent-harness.md`. The two layers overlap by name but not by signature. `pi-coding-agent` is the batteries-included product layer (`~/.pi` layout, extensions, TUI); `pi-agent-core`'s harness is the embeddable primitive. Choose one per call site and keep imports unambiguous.

> [!NOTE]
> `@earendil-works/pi-coding-agent` is **not installed in this repo**; the surface below was verified by inspecting the published package, not by compiling against it.

## Main Public Entry Points

Selected root exports:

- `createAgentSession`, `createAgentSessionServices`, `createAgentSessionRuntime`, `createAgentSessionFromServices`
- `AgentSession`, `AgentSessionRuntime`
- `SessionManager`, `SettingsManager`, `ModelRegistry`, `ModelRuntime`, `readStoredCredential`
- `DefaultResourceLoader` / `loadProjectContextFiles`, `DefaultPackageManager`, `ProjectTrustStore`
- Built-in tool factories: `createReadTool`, `createBashTool`, `createEditTool`, `createWriteTool`, `createGrepTool`, `createFindTool`, `createLsTool`, `createCodingTools`, `createReadOnlyTools`
- Tool-definition factories: `createReadToolDefinition`, `createBashToolDefinition`, etc.
- Compaction utilities: `shouldCompact`, `compact`, `generateSummary`, `generateBranchSummary`, `calculateContextTokens`, … (see the caveat below)
- Skills: `loadSkills`, `loadSkillsFromDir`, `formatSkillsForPrompt`
- Extensions: `defineTool`, `createExtensionRuntime`, `discoverAndLoadExtensions`, `ExtensionRunner`, `wrapRegisteredTool(s)`
- Modes/UI: `main`, `InteractiveMode`, `runPrintMode`, `runRpcMode`, `RpcClient`, plus the interactive components and `Theme`

**Auth-related rename:** there is no `AuthStorage` export. Credentials are owned by **`ModelRuntime`** (`CreateModelRuntimeOptions`, `ModelRuntimeAuthOverrides`, `CredentialSynchronizationError`), with `readStoredCredential` for direct reads.

## Creating A Session

`createAgentSession()` is the high-level SDK entry point for embedding the coding agent.

```typescript
import { createAgentSession } from '@earendil-works/pi-coding-agent';
import { getBuiltinModel } from '@earendil-works/pi-ai/providers/all';

const { session, extensionsResult, modelFallbackMessage } = await createAgentSession({
  cwd: process.cwd(),
  model: getBuiltinModel('anthropic', 'claude-opus-4-5'),
  thinkingLevel: 'high',
  tools: ['read', 'bash', 'edit', 'write'], // optional allowlist
});

session.subscribe((event) => {
  if (event.type === 'message_end') console.log(event.message);
});

await session.prompt('Inspect this project and summarize it.');
```

`CreateAgentSessionOptions`:

- `cwd` (default `process.cwd()`), `agentDir` (default `~/.pi/agent`)
- `modelRuntime` — canonical model/auth runtime; defaults to one backed by `agentDir/auth.json` + `models.json`
- `model`, `thinkingLevel`, `scopedModels`
- `noTools: 'all' | 'builtin'`
- `tools` — allowlist of tool names; `excludeTools` — denylist applied after `tools`
- `customTools` — custom `ToolDefinition[]`
- `resourceLoader`, `sessionManager`, `settingsManager`
- `sessionStartEvent`

## AgentSession Responsibilities

`AgentSession` wraps `pi-agent-core`'s `Agent` and adds coding-agent behavior:

- session persistence
- model and thinking-level management
- prompt template and skill expansion
- queue status tracking
- extension command handling
- built-in tool registry and active-tool selection
- manual and automatic compaction
- overflow recovery and auto-retry handling
- bash execution recording
- tree navigation and branch summarization
- export to HTML/JSONL

Important methods/properties:

- `prompt(text, options?)`, `sendUserMessage(content, options?)`, `sendCustomMessage(message, options?)`
- `steer(text, images?)`, `followUp(text, images?)`, `clearQueue()`, `pendingMessageCount`, `getSteeringMessages()`, `getFollowUpMessages()`
- `abort()`, `waitForIdle()`, `abortCompaction()`, `abortBranchSummary()`, `abortRetry()`, `abortBash()`
- `setModel(model)`, `cycleModel(direction?)`, `setScopedModels(...)`, `modelRuntime`
- `setThinkingLevel(level)`, `cycleThinkingLevel()`, `getAvailableThinkingLevels()`, `supportsThinking()`
- `getAllTools()`, `getActiveToolNames()`, `setActiveToolsByName(names)`, `getToolDefinition(name)`
- `compact(customInstructions?)`, `setAutoCompactionEnabled(enabled)`, `autoCompactionEnabled`, `isCompacting`
- `setAutoRetryEnabled(enabled)`, `autoRetryEnabled`, `isRetrying`, `retryAttempt`
- `executeBash(command, onChunk?, options?)`, `recordBashResult(...)`, `isBashRunning`, `hasPendingBashMessages`
- `navigateTree(targetId, options?)`, `getUserMessagesForForking()`
- `bindExtensions(bindings)`, `reload(options?)`, `dispose()`
- `setSessionName(name)`, `sessionId`, `sessionName`, `sessionFile`
- `getSessionStats()`, `getContextUsage()`, `exportToHtml(path?)`, `exportToJsonl(path?)`
- State getters: `state`, `messages`, `model`, `thinkingLevel`, `systemPrompt`, `isStreaming`, `isIdle`, `steeringMode`/`followUpMode`, `agent`, `sessionManager`, `settingsManager`, `resourceLoader`

`AgentSession.prompt()` expands prompt templates by default, accepts image attachments, and requires `streamingBehavior: 'steer' | 'followUp'` when called while the agent is already streaming.

## AgentSession Events

`AgentSessionEvent` includes all core `AgentEvent`s (with `agent_end` extended by `willRetry`) plus:

| Event | Fields |
| --- | --- |
| `agent_settled` | — |
| `entry_appended` | `entry` |
| `queue_update` | `steering`, `followUp` |
| `compaction_start` | `reason: 'manual' \| 'threshold' \| 'overflow'` |
| `compaction_end` | `reason`, `result`, `aborted`, `willRetry`, `errorMessage?` |
| `auto_retry_start` | `attempt`, `maxAttempts`, `delayMs`, `errorMessage` |
| `auto_retry_end` | `success`, `attempt`, `finalError?` |
| `session_info_changed` | `name` |
| `thinking_level_changed` | `level` |

## Built-In Coding Tools

Built-in tool names: `read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`. (Only read/write/edit/bash have `pi-agent-core` harness equivalents; grep/find/ls are exclusive to this package.)

```typescript
import {
  createCodingTools,
  createReadOnlyTools,
  createReadTool,
  createBashTool,
  createEditTool,
  truncateHead,
  truncateTail,
  truncateLine,
  DEFAULT_MAX_BYTES,
  DEFAULT_MAX_LINES,
} from '@earendil-works/pi-coding-agent';

const tools = createCodingTools(process.cwd());
const readOnly = createReadOnlyTools(process.cwd());
```

Each tool also has an injectable operations interface (`ReadOperations`, `BashOperations`, `EditOperations`, `WriteOperations`, `GrepOperations`, `FindOperations`, `LsOperations`) plus `createLocalBashOperations` and bash spawn hooks. Use `withFileMutationQueue` when composing tools that mutate files to serialize unsafe writes/edits.

## Compaction & Summarization

> [!WARNING]
> This package keeps its **own** compaction implementation with the pre-`Result` signatures: `compact(preparation, model, apiKey, headers?, customInstructions?, signal?, thinkingLevel?, streamFn?, env?, retry?, callbacks?)` resolving `CompactionResult` (it **throws** on failure). `pi-agent-core`'s version takes a `Models` collection and returns `Result<CompactResult, CompactionError>`. Don't mix them.
>
> Also: `prepareCompaction` and `estimateContextTokens` exist internally but are **not re-exported from the package root** — importing them from `@earendil-works/pi-coding-agent` fails. Use `AgentSession.compact()` / `getContextUsage()`, or the `pi-agent-core` equivalents.

Root-exported compaction utilities:

- `calculateContextTokens(usage)`
- `getLastAssistantUsage(entries)`
- `shouldCompact(contextTokens, contextWindow, settings)`
- `estimateTokens(message)`
- `findTurnStartIndex(entries, entryIndex, startIndex)`
- `findCutPoint(entries, startIndex, endIndex, keepRecentTokens)`
- `generateSummary(...)`, `generateSummaryWithUsage(...)`
- `compact(...)`, `DEFAULT_COMPACTION_SETTINGS`
- `collectEntriesForBranchSummary(...)`, `prepareBranchEntries(entries, tokenBudget?)`, `generateBranchSummary(entries, options)`
- `serializeConversation(messages)`

Compaction preparation preserves safe cut points, tracks file operations, supports split-turn summaries, and carries previous summaries forward. In practice prefer `session.compact(customInstructions?)`, which coordinates persistence, events, and auto-compaction settings.

## Session Management

`SessionManager` and its entry types are exported from this package: `SessionEntry`, `SessionHeader`, `SessionInfo`, `SessionContext`, `SessionMessageEntry`, `CompactionEntry`, `BranchSummaryEntry`, `FileEntry`, `ModelChangeEntry`, `ThinkingLevelChangeEntry`, `SessionInfoEntry`, `CustomEntry`, `SessionTreeNode`, plus `CURRENT_SESSION_VERSION`, `parseSessionEntries`, `migrateSessionEntries`, `buildSessionContext`, `buildContextEntries`, `getLatestCompactionEntry`, `sessionEntryToContextMessages`.

Use `AgentSession` where possible instead of directly mutating session entries; it coordinates persistence, active branch position, compaction, extension events, and agent state.

## Skills & Resources

Skill utilities: `loadSkills`, `loadSkillsFromDir` (`LoadSkillsFromDirOptions`, `LoadSkillsResult`), `formatSkillsForPrompt`, and the `Skill` / `SkillFrontmatter` types. Note these are **path-based**, unlike `pi-agent-core`'s `ExecutionEnv`-based loaders.

`AgentSession` expands `/skill:name args` prompts using loaded resources. Resource loading goes through `ResourceLoader` / `DefaultResourceLoader` (with `ResourceCollision` / `ResourceDiagnostic` reporting), package installation through `DefaultPackageManager`, and project-context files through `loadProjectContextFiles`. Untrusted-project gating is `ProjectTrustStore` / `hasTrustRequiringProjectResources`.

## Extensions & Hooks

The extension system replaces the older "harness hooks" mental model.

Extensions can:

- subscribe to lifecycle events
- register LLM-callable tools
- register slash commands, keyboard shortcuts, and CLI flags
- intercept/modify context, provider payloads, tool calls/results, compaction, session tree navigation, and bash execution
- interact with the UI through `ExtensionUIContext`

Important exports: `defineTool`, `createExtensionRuntime`, `discoverAndLoadExtensions`, `ExtensionRunner`, `wrapRegisteredTool` / `wrapRegisteredTools`, and the tool-result guards `isReadToolResult`, `isBashToolResult`, `isEditToolResult`, `isWriteToolResult`, `isGrepToolResult`, `isFindToolResult`, `isLsToolResult`, `isToolCallEventType`.

Extension event types include agent lifecycle events (`BeforeAgentStartEvent`, `AgentStartEvent`, `AgentEndEvent`, `AgentSettledEvent`), provider events (`BeforeProviderRequestEvent`, `BeforeProviderHeadersEvent`), tool-call/result events, session events (`SessionBeforeCompactEvent`, `SessionCompactEvent`, `SessionBeforeForkEvent`, `SessionBeforeSwitchEvent`, `SessionBeforeTreeEvent`, `SessionTreeEvent`, `SessionInfoChangedEvent`, `SessionStartEvent`, `SessionShutdownEvent`), `InputEvent`, `UserBashEvent`, and `ProjectTrustEvent`. Consult the package's extension types before implementing new hooks.

## Example: Custom Tool With `defineTool`

Use `defineTool` when assigning a tool to a variable or passing it through `customTools`; it preserves parameter inference.

```typescript
import { Type } from '@earendil-works/pi-ai';
import { defineTool, createAgentSession } from '@earendil-works/pi-coding-agent';

const echoTool = defineTool({
  name: 'echo',
  label: 'Echo',
  description: 'Echoes text back to the model.',
  promptSnippet: '- `echo`: echo short text for debugging.',
  parameters: Type.Object({ text: Type.String() }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    ctx.ui?.notify?.(`Echo called: ${params.text}`, 'info');
    return {
      content: [{ type: 'text', text: params.text }],
      details: { length: params.text.length },
    };
  },
});

const { session } = await createAgentSession({
  customTools: [echoTool],
});
```

## Example: Extension Registers Command And Tool

```typescript
import { Type } from '@earendil-works/pi-ai';
import { defineTool, type ExtensionFactory } from '@earendil-works/pi-coding-agent';

const extension: ExtensionFactory = (pi) => {
  pi.registerCommand('hello', {
    description: 'Insert a hello prompt',
    handler: async (args, ctx) => {
      ctx.ui.setEditorText('Say hello and explain the current project.');
    },
  });

  pi.registerTool(defineTool({
    name: 'project_name',
    label: 'Project Name',
    description: 'Returns the current project directory name.',
    parameters: Type.Object({}),
    async execute(id, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: 'text', text: ctx.cwd.split('/').pop() || ctx.cwd }],
        details: { cwd: ctx.cwd },
      };
    },
  }));
};

export default extension;
```

## Example: Save Extension State In The Session

```typescript
pi.registerCommand('remember', {
  description: 'Persist a note in the session file.',
  handler: async (args, ctx) => {
    const note = args;
    pi.appendEntry('demo.note', { note, createdAt: new Date().toISOString() });
    ctx.ui.notify('Saved note.', 'info');
  },
});
```

## Example: Toggle Active Tools From A Command

```typescript
pi.registerCommand('readonly', {
  description: 'Switch to read-only built-in tools.',
  handler: async (args, ctx) => {
    pi.setActiveTools(['read', 'grep', 'find', 'ls']);
    ctx.ui.notify('Write/edit/bash disabled for this session.', 'warning');
  },
});
```
