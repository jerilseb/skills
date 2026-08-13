# `@earendil-works/pi-agent-core` — Agent Loop

Stateful agent loop package, verified against the installed **`0.84.1`** package types. This file covers the **low-level layer**: `AgentTool`, the `Agent` class, agent events, queues, and the raw loop functions.

> [!IMPORTANT]
> As of the 0.81–0.84 line, `pi-agent-core` **also ships a harness layer** — sessions, compaction, prompt templates, skills, harness tools, an execution environment, and telemetry. That surface is in `references/agent-harness.md`. The older rule "sessions/compaction/tools live only in `pi-coding-agent`" is **no longer true**.

> [!NOTE]
> `pi-agent-core` still imports some `pi-ai` types/helpers from `@earendil-works/pi-ai/compat`, so the compat shim is a transitive dependency you'll see even if your own code is on the modern `createModels()` API.

## Agent Tools

`AgentTool` extends a `pi-ai` `Tool` with a UI label, optional argument preparation, an `execute` function, and an optional per-tool execution mode.

```typescript
import { Type, type Static } from '@earendil-works/pi-ai';
import type { AgentTool } from '@earendil-works/pi-agent-core';

const Params = Type.Object({ path: Type.String() });

const readFileTool: AgentTool<typeof Params, { path: string }> = {
  name: 'read_file',
  label: 'Read File',
  description: 'Reads a local text file',
  parameters: Params,
  executionMode: 'parallel',

  prepareArguments: (raw) => raw as Static<typeof Params>,

  execute: async (toolCallId, params, signal, onUpdate) => {
    onUpdate?.({
      content: [{ type: 'text', text: 'Reading...' }],
      details: { path: params.path },
    });

    if (signal?.aborted) throw new Error('Aborted');

    const text = await fs.promises.readFile(params.path, 'utf8');
    return {
      content: [{ type: 'text', text }],
      details: { path: params.path },
    };
  },
};
```

`AgentToolResult<T>` is `{ content, details, usage?, addedToolNames?, terminate? }` — `usage` reports the tool's own token spend (not counted toward main LLM context accounting) and `addedToolNames` names tools that become available from this transcript point onward.

Working rules:

- Throw `Error` on failure; the loop converts failures to error tool-result messages.
- Do not encode failures as normal text content.
- Honor the optional `AbortSignal` in long-running tools.
- `terminate: true` is only an early-stop hint. In a batch, early termination happens only when every finalized tool result in the batch has `terminate: true`.
- `executionMode` can be `sequential` or `parallel`; per-tool overrides interact with the agent-level default.

## Agent Class

`Agent` owns transcript state, streams assistant turns, executes tools, and manages steering/follow-up queues.

`AgentOptions`:

- **`streamFn` (REQUIRED)**: the stream implementation. A `pi-ai` `Models` collection's `streamSimple` satisfies the shape — `streamFn: (m, c, o) => models.streamSimple(m, c, o)` — and the collection resolves auth internally, so `getApiKey` is unnecessary. A host that installs a process-wide default via `setDefaultStreamFn()` can pass `getDefaultStreamFn()` here.
- `initialState`: partial `AgentState` excluding runtime fields (`pendingToolCalls`, `isStreaming`, `streamingMessage`, `errorMessage`)
- `convertToLlm(messages)`: convert/filter custom `AgentMessage`s to provider `Message`s
- `transformContext(messages, signal)`: prune/inject context before LLM conversion
- `getApiKey(provider)`: dynamic API-key/OAuth token resolution per LLM call (not needed with a `Models`-backed `streamFn`)
- `onPayload`, `onResponse`: provider request/response inspection hooks from `pi-ai`
- `beforeToolCall`, `afterToolCall`: tool interception hooks
- `shouldStopAfterTurn(context, signal)`: graceful stop after the current turn — see below
- `prepareNextTurn(signal)` / `prepareNextTurnWithContext(context, signal)`: return replacement context/model/thinking state for the next turn, or `undefined` to keep the current config
- `steeringMode`, `followUpMode`: `'all'` or `'one-at-a-time'`
- `sessionId`, `thinkingBudgets`, `transport`, `maxRetryDelayMs`
- `toolExecution`: `'sequential'` or `'parallel'`

```typescript
import { Agent } from '@earendil-works/pi-agent-core';
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const models = builtinModels();
const isLlmMessage = (message: any) =>
  message?.role === 'user' || message?.role === 'assistant' || message?.role === 'toolResult';

const agent = new Agent({
  streamFn: (model, context, options) => models.streamSimple(model, context, options),
  initialState: {
    model,
    systemPrompt: 'You are helpful.',
    thinkingLevel: 'medium',
    tools: [readFileTool],
    messages: [],
  },
  toolExecution: 'parallel',
  convertToLlm: (messages) => messages.filter(isLlmMessage),
  transformContext: async (messages) => messages,
  beforeToolCall: async ({ toolCall }) => {
    if (toolCall.name === 'dangerous_tool') {
      return { block: true, reason: 'Forbidden' };
    }
  },
  afterToolCall: async ({ result, isError }) => {
    if (!isError) return { details: { audited: true, original: result.details } };
  },
});

await agent.prompt('Read package.json');
await agent.waitForIdle();
```

Notes:

- `convertToLlm`, `transformContext`, and `shouldStopAfterTurn` should not throw. Return a safe fallback instead — throwing interrupts the loop without a normal event sequence.
- `afterToolCall` overrides fields shallowly: `content`, `details`, `isError`, `usage`, and `terminate` replace the original fields. There is no deep merge.
- `beforeToolCall` returning `{ block: true }` emits an error tool result instead of executing; it may also set `terminate: true` to join the batch early-termination rule.
- `prompt()` accepts a string with optional images, a single `AgentMessage`, or an array of `AgentMessage`s.
- `continue()` requires the current transcript to end in a user/tool-result message after `convertToLlm`.

## Stopping Gracefully After A Turn

`shouldStopAfterTurn` is available on **both** `AgentOptions` and `AgentLoopConfig`. It runs after `turn_end` is emitted; returning `true` makes the loop emit `agent_end` and exit **before** polling the steering/follow-up queues, without starting another LLM call. The in-flight assistant response and its tool executions finish normally — this is a clean stop, not `abort()`.

```typescript
const agent = new Agent({
  streamFn: (m, c, o) => models.streamSimple(m, c, o),
  initialState: { model, systemPrompt, messages: [], tools },
  shouldStopAfterTurn: async ({ context, message, toolResults, newMessages }) => {
    const approxTokens = context.messages
      .map((m) => JSON.stringify(m).length)
      .reduce((a, b) => a + b, 0) / 4;
    return approxTokens > model.contextWindow * 0.8;
  },
});
```

The hook context is `{ message, toolResults, context, newMessages }`. Use `prepareNextTurn`/`prepareNextTurnWithContext` instead when you want to *continue* with a compacted context or a different model rather than stop.

## Queues, Abort, And State

Useful methods/properties:

- `steer(message)`: queue a message after the current assistant turn and tool executions finish
- `followUp(message)`: queue a message only after the agent would otherwise stop
- `clearSteeringQueue()`, `clearFollowUpQueue()`, `clearAllQueues()`
- `hasQueuedMessages()`
- `abort()`, `signal`
- `waitForIdle()`
- `reset()`
- `state`: includes `systemPrompt`, `model`, `thinkingLevel`, tools/messages accessors, `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`

`Agent.subscribe()` listeners are awaited in subscription order and receive the active abort signal. `agent_end` is the last event, but `waitForIdle()` resolves only after awaited `agent_end` listeners settle.

## Agent Events

| Event | Fields |
| --- | --- |
| `agent_start` | — |
| `turn_start` | — |
| `message_start` | `message` |
| `message_update` | `message`, `assistantMessageEvent` |
| `message_end` | `message` |
| `tool_execution_start` | `toolCallId`, `toolName`, `args` |
| `tool_execution_update` | `toolCallId`, `toolName`, `args`, `partialResult` |
| `tool_execution_end` | `toolCallId`, `toolName`, `result`, `isError` |
| `turn_end` | `message`, `toolResults` |
| `agent_end` | `messages` |

```typescript
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === 'tool_execution_end') {
    console.log(event.toolName, event.isError);
  }
});
```

## Low-Level Loops

Exports:

- `agentLoop(prompts, context, config, signal, streamFn)`
- `agentLoopContinue(context, config, signal, streamFn)`
- `runAgentLoop(prompts, context, config, emit, signal, streamFn)`
- `runAgentLoopContinue(context, config, emit, signal, streamFn)`

`streamFn` is a **required positional argument** on all four (`signal` may be `undefined`). `AgentLoopConfig` extends `SimpleStreamOptions` and adds `model`, `convertToLlm` (required), `transformContext`, `getApiKey`, `shouldStopAfterTurn`, `prepareNextTurn`, `getSteeringMessages`/`getFollowUpMessages` queue suppliers, `toolExecution`, and the `beforeToolCall`/`afterToolCall` hooks.

```typescript
import { agentLoop, agentLoopContinue } from '@earendil-works/pi-agent-core';

const stream = agentLoop([userMessage], context, config, signal, streamFn);
for await (const event of stream) {
  // handle AgentEvent
}
const newMessages = await stream.result();
```

Custom `streamFn` contract: it must not throw/reject for request/model/runtime failures. It returns an `AssistantMessageEventStream` (or a promise of one) whose terminal event encodes failures as an assistant message with `stopReason: 'error'` or `'aborted'` and `errorMessage`.

`setDefaultStreamFn(streamFn)` / `getDefaultStreamFn()` install and read a process-wide fallback, so a host can wire a model runtime once without making `pi-agent-core` depend on a provider catalog.

## Example: Custom AgentMessage Type

Use declaration merging when your app needs UI-only or app-specific messages, then filter or convert them in `convertToLlm`.

```typescript
import { Agent, type AgentMessage } from '@earendil-works/pi-agent-core';

type NoticeMessage = {
  role: 'notice';
  level: 'info' | 'warning';
  content: string;
  timestamp: number;
};

declare module '@earendil-works/pi-agent-core' {
  interface CustomAgentMessages {
    notice: NoticeMessage;
  }
}

const agent = new Agent({
  streamFn,
  convertToLlm: (messages: AgentMessage[]) => messages.flatMap((message) => {
    if ((message as any).role === 'notice') return []; // UI-only
    return [message as any];
  }),
});
```

## Example: Sequential Tool For Shared Mutable State

```typescript
import { Type } from '@earendil-works/pi-ai';
import type { AgentTool } from '@earendil-works/pi-agent-core';

const Params = Type.Object({ path: Type.String() });

const updateIndexTool: AgentTool<typeof Params> = {
  name: 'update_index',
  label: 'Update Index',
  description: 'Updates a shared on-disk index.',
  parameters: Params,
  executionMode: 'sequential',
  async execute(id, params) {
    await updateSharedIndex(params.path);
    return { content: [{ type: 'text', text: 'Index updated.' }], details: {} };
  },
};
```
