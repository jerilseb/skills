# `@earendil-works/pi-agent-core` — Harness Layer

The harness layer of `@earendil-works/pi-agent-core`, `0.84.1`. It adds durable sessions, compaction, skills/prompt templates, filesystem-and-shell tools, and telemetry on top of the `Agent` loop documented in `agent-core.md`.

Import from the package root; Node-specific pieces come from the `/node` subpath:

```typescript
import { AgentHarness, Session, JsonlSessionRepo, compact, loadSkills } from '@earendil-works/pi-agent-core';
import { NodeExecutionEnv } from '@earendil-works/pi-agent-core/node';
```

> [!NOTE]
> `pi-coding-agent` is a **separate, higher-level product** (its own `AgentSession`, extensions, TUI, settings, `~/.pi` layout). Some names appear in both packages with different signatures — `compact`, `Skill`, `PromptTemplate`, `ToolExecutionMode`. Pick one layer per call site and check which package a symbol came from. See `coding-agent-infrastructure.md`.

## `AgentHarness`

`AgentHarness` is a durable, session-backed driver: it owns a `Session`, a `Models` collection, the active model/tools/resources, queueing, compaction, and tree navigation. It implements `AgentLane`, and all of its accessors are **async** (state lives in the session, not in memory).

```typescript
const { harness, suspended } = await AgentHarness.create({
  session,                  // Session (required)
  models,                   // pi-ai Models collection (required)
  model,                    // Model<Api> (required)
  thinkingLevel: 'medium',
  tools: harnessTools,
  toolContext: () => ({ env }),          // static value or per-turn provider
  systemPrompt: () => buildSystemPrompt(),
  resources: { skills, promptTemplates },
  streamOptions: { maxTokens: 8_000 },
  retry: retryPolicy,
  compaction: DEFAULT_COMPACTION_SETTINGS,
  steeringMode: 'all',
  followUpMode: 'all',
  toolExecution: 'parallel',
  drive: 'automatic',                    // or 'manual' — see peekAction/executeAction
  context: telemetryContext,
});
```

`create()` resolves `{ harness, suspended }`; `suspended` lists operations that were interrupted in a previous process and can be resumed.

Run control: `prompt(text | message[], images?)`, `skill(name, additionalInstructions?)`, `promptFromTemplate(name, args?)`, `steer(...)`, `followUp(...)`, `nextRun(...)`, `cancelQueued(entryId)`, `abort()`, `resume()`, `waitForIdle()`, `runWhenIdle(cb)`, `runToCompletion()`. With `drive: 'manual'`, step the loop yourself via `peekAction()` / `executeAction()`.

Session/state: `session` (a `SessionTree`), `getLeafId()`, `navigateTree(targetId, options?)`, `compact({ customInstructions? })`, `recordUsage(usage, options?)`, `watch()` / `watchSession()` (snapshot + listener + `unsubscribe`), `close()`.

Configuration accessors (all async): `getModel`/`setModel`, `getThinkingLevel`/`setThinkingLevel`, `getActiveTools`/`setActiveTools`, `getTools`/`setTools`, `getResources`/`setResources`, `getStreamOptions`/`setStreamOptions`, `getRetryPolicy`/`setRetryPolicy`, `getCompactionSettings`/`setCompactionSettings`, `getSteeringMode`/`setSteeringMode`, `getFollowUpMode`/`setFollowUpMode`. Lanes (parallel branches on one session): `lane(name)`, `createLane(name, at)`, `lanes()`.

`harness.hooks` and `harness.events` expose the interception and observation seams.

## Results And Errors

The harness layer uses a **`Result` type instead of throwing** for expected failures:

```typescript
import { Result, matchError, type Result as ResultType } from '@earendil-works/pi-agent-core';

const outcome = await env.readTextFile('/tmp/x');
if (!outcome.ok) return matchError(outcome.error, { /* per-tag handlers */ });
use(outcome.value);
```

Errors are tagged classes built with `TaggedError(tag)` — `FileError`, `ExecutionError`, `CompactionError`, `BranchSummaryError`, plus harness-operation tags (`LaneBusy`, `NoActiveRun`, `NothingToResume`, `UnknownSkill`, `UnknownTemplate`, `NothingToCompact`, `Closed`, …). `HarnessFault`, `HarnessClosed`, and `HarnessNotImplemented` are thrown `Error`s. Helpers: `ok`, `err`, `getOrThrow`, `getOrUndefined`, `toError`, `matchError`.

## Execution Environment

`ExecutionEnv` = `FileSystem` + `Shell`; every method returns a `Result`, never throws.

- `FileSystem`: `cwd`, `absolutePath`, `joinPath`, `readTextFile`, `readTextLines`, `readBinaryFile`, `writeFile`, `appendFile`, `renameFile`, `fileInfo`, `listDir`, `canonicalPath`, `exists`, directory creation.
- `Shell`: `exec(command, options?)` → `Result<{ stdout, stderr, exitCode }, ExecutionError>` with `ShellExecOptions` (`cwd`, `env`, `inheritEnv`, `timeout` in seconds, `abortSignal`, `onStdout`, `onStderr`), plus `cleanup()`.

`NodeExecutionEnv` (from `@earendil-works/pi-agent-core/node`) is the Node implementation. Supply your own for sandboxes, remote execution, or tests.

## Harness Tools

Built-in harness tools are **`AgentHarnessTool`**, not plain `AgentTool`: their `execute` takes a fifth `context` argument resolved per turn from the harness's `toolContext`. The execution tools require `ExecutionToolContext` = `{ env: ExecutionEnv }`.

```typescript
import { createBashTool, createEditTool, createReadTool, createWriteTool } from '@earendil-works/pi-agent-core';
import { NodeExecutionEnv } from '@earendil-works/pi-agent-core/node';

const env = new NodeExecutionEnv({ cwd: process.cwd() });
const tools = [createReadTool(), createWriteTool(), createEditTool(), createBashTool()];
// harness resolves { env } per turn via toolContext
```

The harness ships four tools — `bash`, `edit`, `read`, `write`; `grep`/`find`/`ls` exist only in `pi-coding-agent`. Output-shaping helpers `truncateHead` / `truncateTail` / `truncateLine` and shell-output utilities are exported alongside them.

## Sessions

A `Session` is a durable, branching transcript over a `SessionStorage`/`SessionRepo`:

- `Session` — `getMetadata`, `view(lane)`, `getLeafId`, `getEntry`, `getStats`, `getName`/`setName`, `getLabel`/`setLabel`, `findEntries`/`findEntry`, `findEntriesOnBranch`/`findEntryOnBranch`, `appendMessage`, `appendCustomEntry`, `appendEntry`, `appendRecord`, `findRecords`, `getLanes`/`createLane`/`moveLane`.
- `JsonlSessionRepo` — JSONL-file-backed repo (`JsonlSessionRepoOptions`, `JsonlSessionCreateOptions`, `JsonlSessionListOptions`, `JsonlSessionMetadata`, `JsonlV4Header`), constructed over an injected `JsonlSessionRepoFileSystem`.
- `InMemorySessionStorage` / `InMemorySessionRepo` — for tests and ephemeral runs.
- Context projection — `buildSessionContext(pathEntries, options?)`, `buildContextEntries`, `sessionEntryToContextMessages`, `defaultContextEntryTransform`, plus `ContextEntryTransform` / `CustomEntryContextMessageProjector` for app-specific entry types.

## Compaction

Canonical compaction lives here (`pi-coding-agent` re-exports a subset and omits `prepareCompaction`/`estimateContextTokens`). The functions take a **`Models` collection** and return a `Result`:

```typescript
import {
  DEFAULT_COMPACTION_SETTINGS,
  estimateContextTokens,
  shouldCompact,
  prepareCompaction,
  compact,
} from '@earendil-works/pi-agent-core';

const usage = estimateContextTokens(messages);
if (shouldCompact(usage.tokens, model.contextWindow, DEFAULT_COMPACTION_SETTINGS)) {
  const prep = prepareCompaction(pathEntries, DEFAULT_COMPACTION_SETTINGS); // Result<CompactionPreparation | undefined, CompactionError>
  if (prep.ok && prep.value) {
    const result = await compact(prep.value, models, model, customInstructions, signal, thinkingLevel, retry, callbacks);
    // result: Result<CompactResult, CompactionError>
  }
}
```

Also exported: `calculateContextTokens`, `estimateTokens`, `getLastAssistantUsage`, `findTurnStartIndex`, `findCutPoint`, `generateSummary`, `generateSummaryWithUsage`, `serializeConversation`, and the branch-summary family (`collectEntriesForBranchSummary`, `prepareBranchEntries`, `generateBranchSummary`). Summary messages are wrapped with the exported `COMPACTION_SUMMARY_PREFIX`/`SUFFIX` and `BRANCH_SUMMARY_PREFIX`/`SUFFIX` markers; `createCompactionSummaryMessage` / `createBranchSummaryMessage` / `createCustomMessage` / `convertToLlm` (from `harness/messages`) build and project them.

## Skills And Prompt Templates

Loaders read through an `ExecutionEnv`, so they work in sandboxes and tests:

```typescript
import { loadSkills, loadPromptTemplates, formatSkillInvocation, formatSkillsForSystemPrompt } from '@earendil-works/pi-agent-core';

const { skills, diagnostics } = await loadSkills(env, ['~/.pi/skills', '.claude/skills']);
const { promptTemplates } = await loadPromptTemplates(env, ['.pi/commands']);
```

- `Skill` = `{ name, description, content, filePath, disableModelInvocation? }`; `PromptTemplate` = `{ name, description?, content }`.
- `loadSourcedSkills` / `loadSourcedPromptTemplates` keep a per-source tag (project vs user) on each loaded item.
- `formatSkillInvocation(skill, additionalInstructions?)` and `formatPromptTemplateInvocation(template, args?)` build the prompt text; `parseCommandArgs` / `substituteArgs` handle `$1`-style argument substitution.
- `formatSkillsForSystemPrompt(skills)` renders the model-visible skill list.
- Loaders return `diagnostics` (`file_info_failed`, `list_failed`, `read_failed`, `parse_failed`, `invalid_metadata`) instead of throwing on a bad file.

Pass both collections to the harness as `resources: { skills, promptTemplates }`; then `harness.skill(name)` and `harness.promptFromTemplate(name, args)` resolve against them.

## Telemetry

`pi-agent-core` re-exports `@earendil-works/pi-telemetry` and defines the agent's span schemas — this is the supported hook for tracing/observability backends.

```typescript
import {
  startAiSpan,
  startHarnessSpan,
  AI_TELEMETRY_SCHEMA,
  HARNESS_TELEMETRY_SCHEMA,
  AGENT_TELEMETRY_SCHEMAS,
  defineTelemetrySchema,
  createTypedSpanStarter,
  InMemoryTelemetryContext,
  NOOP_TELEMETRY_CONTEXT,
} from '@earendil-works/pi-agent-core';
```

A `TelemetryContext` passed as `AgentHarnessOptions.context` parents every span the harness produces; `pi-ai`'s `ProviderRequestOptions.telemetryContext` does the same for a single model request. Span attributes are schema-typed (`SpanAttributes`, `InferStartAttributes`, `TelemetrySchemaSpanName`, …), so custom exporters get compile-time checked attribute names.
