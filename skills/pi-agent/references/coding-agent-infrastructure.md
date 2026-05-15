# `@earendil-works/pi-coding-agent` SDK & Infrastructure

The higher-level coding-agent infrastructure is exported from `@earendil-works/pi-coding-agent`, not from `@earendil-works/pi-agent-core` in `0.74.0`.

Use this reference for sessions, built-in coding tools, compaction, resource loading, extensions, settings, prompt templates, and interactive/RPC/print mode integration.

## Main Public Entry Points

Selected root exports:

- `createAgentSession`, `createAgentSessionServices`, `createAgentSessionRuntime`, `createAgentSessionFromServices`
- `AgentSession`, `AgentSessionRuntime`
- `SessionManager`, `SettingsManager`, `ModelRegistry`, `AuthStorage`
- Built-in tool factories: `createReadTool`, `createBashTool`, `createEditTool`, `createWriteTool`, `createGrepTool`, `createFindTool`, `createLsTool`, `createCodingTools`, `createReadOnlyTools`
- Tool-definition factories: `createReadToolDefinition`, `createBashToolDefinition`, etc.
- Compaction utilities: `estimateContextTokens`, `shouldCompact`, `prepareCompaction`, `compact`, `generateSummary`, `generateBranchSummary`, etc.
- Skills: `loadSkills`, `loadSkillsFromDir`, `formatSkillsForPrompt`
- Extensions: `defineTool`, `createExtensionRuntime`, `discoverAndLoadExtensions`, `ExtensionRunner`, `wrapRegisteredTool(s)`

There is no public `AgentHarness`, `NodeExecutionEnv`, `JsonlSessionRepo`, or `executeShellWithCapture` export in `0.74.0`.

## Creating A Session

`createAgentSession()` is the high-level SDK entry point for embedding the coding agent.

```typescript
import { createAgentSession } from '@earendil-works/pi-coding-agent';
import { getModel } from '@earendil-works/pi-ai';

const { session, extensionsResult, modelFallbackMessage } = await createAgentSession({
  cwd: process.cwd(),
  model: getModel('anthropic', 'claude-opus-4-5'),
  thinkingLevel: 'high',
  tools: ['read', 'bash', 'edit', 'write'], // optional allowlist
});

session.subscribe((event) => {
  if (event.type === 'message_end') console.log(event.message);
});

await session.prompt('Inspect this project and summarize it.');
```

Useful `CreateAgentSessionOptions`:

- `cwd`, `agentDir`
- `authStorage`, `modelRegistry`, `sessionManager`, `settingsManager`, `resourceLoader`
- `model`, `thinkingLevel`, `scopedModels`
- `noTools: 'all' | 'builtin'`
- `tools`: allowlist of tool names
- `customTools`: custom `ToolDefinition[]`
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
- overflow recovery and retry handling
- bash execution recording
- tree navigation and branch summarization
- export to HTML/JSONL

Important methods/properties:

- `prompt(text, options?)`
- `steer(text, images?)`
- `followUp(text, images?)`
- `sendUserMessage(content, options?)`
- `sendCustomMessage(message, options?)`
- `clearQueue()`, `pendingMessageCount`, `getSteeringMessages()`, `getFollowUpMessages()`
- `abort()`, `abortCompaction()`, `abortBranchSummary()`, `abortRetry()`, `abortBash()`
- `setModel(model)`, `cycleModel(direction?)`
- `setThinkingLevel(level)`, `cycleThinkingLevel()`, `supportsThinking()`
- `setActiveToolsByName(names)`, `getActiveToolNames()`, `getAllTools()`
- `compact(customInstructions?)`
- `navigateTree(targetId, options?)`
- `executeBash(command, onChunk?, options?)`
- `reload()`, `dispose()`
- `getSessionStats()`, `getContextUsage()`
- `exportToHtml(path?)`, `exportToJsonl(path?)`

`AgentSession.prompt()` expands prompt templates by default, accepts image attachments, and requires `streamingBehavior: 'steer' | 'followUp'` when called while the agent is already streaming.

## AgentSession Events

`AgentSessionEvent` includes all core `AgentEvent`s plus:

| Event | Fields |
| --- | --- |
| `queue_update` | `steering`, `followUp` |
| `compaction_start` | `reason: 'manual' | 'threshold' | 'overflow'` |
| `compaction_end` | `reason`, `result`, `aborted`, `willRetry`, `errorMessage?` |
| `auto_retry_start` | `attempt`, `maxAttempts`, `delayMs`, `errorMessage` |
| `auto_retry_end` | `success`, `attempt`, `finalError?` |
| `session_info_changed` | `name` |
| `thinking_level_changed` | `level` |

## Built-In Coding Tools

Built-in tool names:

- `read`
- `bash`
- `edit`
- `write`
- `grep`
- `find`
- `ls`

Factories:

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

Use `withFileMutationQueue` when composing tools that mutate files to serialize unsafe file writes/edits.

## Compaction & Summarization

Compaction exports are from `@earendil-works/pi-coding-agent`.

```typescript
import {
  DEFAULT_COMPACTION_SETTINGS,
  estimateContextTokens,
  shouldCompact,
  prepareCompaction,
  compact,
} from '@earendil-works/pi-coding-agent';

const usage = estimateContextTokens(session.messages);
if (usage.tokens && shouldCompact(usage.tokens, model.contextWindow, DEFAULT_COMPACTION_SETTINGS)) {
  const pathEntries = session.sessionManager.getBranch();
  const prep = prepareCompaction(pathEntries, DEFAULT_COMPACTION_SETTINGS);
  if (prep) {
    const result = await compact(prep, model, apiKey, headers, customInstructions, signal, thinkingLevel);
    session.sessionManager.appendCompaction(
      result.summary,
      result.firstKeptEntryId,
      result.tokensBefore,
      result.details,
    );
  }
}
```

Public compaction utilities include:

- `calculateContextTokens(usage)`
- `getLastAssistantUsage(entries)`
- `estimateContextTokens(messages)`
- `shouldCompact(contextTokens, contextWindow, settings)`
- `estimateTokens(message)`
- `findTurnStartIndex(entries, entryIndex, startIndex)`
- `findCutPoint(entries, startIndex, endIndex, keepRecentTokens)`
- `prepareCompaction(pathEntries, settings)`
- `generateSummary(...)`
- `compact(...)`
- `collectEntriesForBranchSummary(...)`
- `prepareBranchEntries(entries, tokenBudget?)`
- `generateBranchSummary(entries, options)`
- `serializeConversation(messages)`

Compaction preparation preserves safe cut points, tracks file operations, supports split-turn summaries, and carries previous summaries forward.

## Session Management

`SessionManager` and related types are exported from `@earendil-works/pi-coding-agent`.

Relevant exported types include `SessionEntry`, `SessionHeader`, `SessionInfo`, `SessionContext`, `SessionMessageEntry`, `CompactionEntry`, `BranchSummaryEntry`, `FileEntry`, `ModelChangeEntry`, and `ThinkingLevelChangeEntry`.

Use `AgentSession` where possible instead of directly mutating session entries; it coordinates persistence, active branch position, compaction, extension events, and agent state.

## Skills & Resources

Skill utilities:

- `loadSkills`
- `loadSkillsFromDir`
- `formatSkillsForPrompt`
- `Skill`, `SkillFrontmatter`

`AgentSession` expands `/skill:name args` prompts using loaded resources. Resource loading is handled through `ResourceLoader` / `DefaultResourceLoader`.

## Extensions & Hooks

The extension system replaces the older “harness hooks” mental model.

Extensions can:

- subscribe to lifecycle events
- register LLM-callable tools
- register slash commands, keyboard shortcuts, and CLI flags
- intercept/modify context, provider payloads, tool calls/results, compaction, session tree navigation, and bash execution
- interact with the UI through `ExtensionUIContext`

Important exports:

- `defineTool`
- `createExtensionRuntime`
- `discoverAndLoadExtensions`
- `ExtensionRunner`
- `wrapRegisteredTool`, `wrapRegisteredTools`
- `isReadToolResult`, `isBashToolResult`, `isEditToolResult`, etc.

Extension event types include agent lifecycle events, provider request/payload/response events, tool-call/result events, session compaction/tree/fork/switch events, input events, user bash events, and session shutdown/start events. Consult `@earendil-works/pi-coding-agent` extension types/docs before implementing new hooks.

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
