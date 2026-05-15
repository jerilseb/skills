# `@earendil-works/pi-agent-core`

Stateful agent loop package. In `0.74.0`, the public exports are `Agent`, low-level agent-loop functions, proxy helpers, and core agent types. Session management, built-in coding tools, compaction, extensions, and SDK helpers live in `@earendil-works/pi-coding-agent`, not in `pi-agent-core`.

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

Working rules:

- Throw `Error` on failure; the loop converts failures to error tool-result messages.
- Do not encode failures as normal text content.
- Honor the optional `AbortSignal` in long-running tools.
- `terminate: true` is only an early-stop hint. In a batch, early termination happens only when every finalized tool result in the batch has `terminate: true`.
- `executionMode` can be `sequential` or `parallel`; per-tool overrides interact with the agent-level default.

## Agent Class

`Agent` owns transcript state, streams assistant turns, executes tools, and manages steering/follow-up queues.

Important constructor options:

- `initialState`: partial `AgentState` excluding runtime fields
- `convertToLlm(messages)`: convert/filter custom `AgentMessage`s to provider `Message`s
- `transformContext(messages, signal)`: prune/inject context before LLM conversion
- `streamFn`: custom stream implementation; must return an assistant event stream
- `getApiKey(provider)`: dynamic API-key/OAuth token resolution per LLM call
- `onPayload`, `onResponse`: provider request/response inspection hooks from `pi-ai`
- `beforeToolCall`, `afterToolCall`: tool interception hooks
- `steeringMode`, `followUpMode`: `'all'` or `'one-at-a-time'`
- `sessionId`, `thinkingBudgets`, `transport`, `maxRetryDelayMs`
- `toolExecution`: `'sequential'` or `'parallel'`

```typescript
import { Agent } from '@earendil-works/pi-agent-core';

const isLlmMessage = (message: any) =>
  message?.role === 'user' || message?.role === 'assistant' || message?.role === 'toolResult';

const agent = new Agent({
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
  getApiKey: async (provider) => lookupApiKey(provider),
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

- `convertToLlm` and `transformContext` should not throw. Return a safe fallback instead.
- `afterToolCall` overrides fields shallowly: `content`, `details`, `isError`, and `terminate` replace the original fields. There is no deep merge.
- `AgentOptions` does not expose a `prepareNextTurn` hook in `0.74.0`; change model/thinking/tool state through `agent.state` or a higher-level session wrapper between runs.
- `prompt()` accepts a string with optional images, a single `AgentMessage`, or an array of `AgentMessage`s.
- `continue()` requires the current transcript to end in a user/tool-result message after `convertToLlm`.

## Queues, Abort, And State

Useful methods/properties:

- `steer(message)`: queue a message after the current assistant turn and tool executions finish
- `followUp(message)`: queue a message only after the agent would otherwise stop
- `clearSteeringQueue()`, `clearFollowUpQueue()`, `clearAllQueues()`
- `hasQueuedMessages()`
- `abort()`
- `waitForIdle()`
- `reset()`
- `state`: includes `systemPrompt`, `model`, `thinkingLevel`, tools/messages accessors, `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`

`Agent.subscribe()` listeners are awaited in subscription order. `agent_end` is the last event, but `waitForIdle()` resolves only after awaited `agent_end` listeners settle.

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

- `agentLoop(prompts, context, config, signal?, streamFn?)`
- `agentLoopContinue(context, config, signal?, streamFn?)`
- `runAgentLoop(...)`
- `runAgentLoopContinue(...)`

`AgentLoopConfig` extends `SimpleStreamOptions` and adds model, conversion, context transform, dynamic API-key lookup, queue suppliers, stop hook, tool execution mode, and tool hooks.

```typescript
import { agentLoop, agentLoopContinue } from '@earendil-works/pi-agent-core';

const stream = agentLoop([userMessage], context, config, signal);
for await (const event of stream) {
  // handle AgentEvent
}
const newMessages = await stream.result();
```

Custom `streamFn` contract: it must not throw/reject for request/model/runtime failures. It should return an `AssistantMessageEventStream` whose terminal event encodes failures as an assistant message with `stopReason: 'error'` or `'aborted'` and `errorMessage`.

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
  convertToLlm: (messages: AgentMessage[]) => messages.flatMap((message) => {
    if ((message as any).role === 'notice') return []; // UI-only
    return [message as any];
  }),
});
```

## Example: Gracefully Stop Before Context Gets Too Large

```typescript
import { Agent } from '@earendil-works/pi-agent-core';

const agent = new Agent({
  initialState: { model, systemPrompt, messages: [], tools },
});

// Low-level loops expose shouldStopAfterTurn in AgentLoopConfig. With Agent,
// implement this policy in a wrapper/session layer or stop after observed events.
agent.subscribe((event) => {
  if (event.type === 'turn_end') {
    const approxChars = agent.state.messages.map((m: any) => JSON.stringify(m).length).reduce((a, b) => a + b, 0);
    if (approxChars / 4 > model.contextWindow * 0.8) agent.abort();
  }
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
