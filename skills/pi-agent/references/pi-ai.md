# `@earendil-works/pi-ai`

Stateless LLM/provider abstraction. Public APIs below are based on the installed `0.74.0` package exports.

## Models & Providers

**Exports:** `getProviders`, `getModels`, `getModel`, `modelsAreEqual`, `getSupportedThinkingLevels`, `clampThinkingLevel`, `calculateCost`, plus model/provider types.

```typescript
import {
  getProviders,
  getModels,
  getModel,
  modelsAreEqual,
  getSupportedThinkingLevels,
  clampThinkingLevel,
  calculateCost,
  type Model,
  type Usage,
} from '@earendil-works/pi-ai';

const providers = getProviders();
const anthropicModels = getModels('anthropic');
const model = getModel('anthropic', 'claude-opus-4-5');

const supportsVision = model.input.includes('image');
const levels = getSupportedThinkingLevels(model); // includes 'off' when applicable
const safeLevel = clampThinkingLevel(model, 'xhigh');

const same = modelsAreEqual(model, model);
const cost = calculateCost(model, usage);
```

Custom OpenAI-compatible models can be represented as `Model<'openai-completions'>`; use `compat` to override API quirks such as developer-role support, max-token field, prompt-cache behavior, strict tool schema support, etc.

## API Provider Registry

**Exports:** `registerApiProvider`, `getApiProvider`, `getApiProviders`, `unregisterApiProviders`, `clearApiProviders`.

Use these when adding a custom API implementation. A provider supplies both `stream` and `streamSimple` functions for a custom `api` string.

## Environment API Keys

**Exports:** `findEnvKeys`, `getEnvApiKey`.

`getEnvApiKey(provider)` returns configured API-key environment variables for normal API-key providers. It intentionally does not synthesize ambient credentials such as AWS profiles or Google ADC.

## Tools & Validation

**Exports:** `Type`, `Static`, `TSchema`, `StringEnum`, `validateToolArguments`, `validateToolCall`, `Tool`.

**Important:** use `StringEnum` instead of `Type.Enum` for string enums. `Type.Enum` emits `anyOf`/`const` schema forms that some providers, especially Gemini, reject.

```typescript
import { Type, StringEnum, validateToolCall, type Tool } from '@earendil-works/pi-ai';

const weatherTool: Tool = {
  name: 'get_weather',
  description: 'Get current weather',
  parameters: Type.Object({
    location: Type.String(),
    units: StringEnum(['celsius', 'fahrenheit'], { default: 'celsius' }),
  }),
};

const args = validateToolCall([weatherTool], {
  type: 'toolCall',
  id: 'call_1',
  name: 'get_weather',
  arguments: { location: 'Paris' },
});
```

Validation throws formatted `Error`s. Let callers/tool loops handle those as tool errors.

## Core Generation

**Exports:** `stream`, `complete`, `streamSimple`, `completeSimple`, `getEnvApiKey`.

- `stream` / `complete` accept provider-specific `ProviderStreamOptions`.
- `streamSimple` / `completeSimple` accept unified `SimpleStreamOptions`, including `reasoning?: 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'`.
- Common stream options include `temperature`, `maxTokens`, `signal`, `apiKey`, `transport`, `cacheRetention`, `sessionId`, `onPayload`, `onResponse`, `headers`, `timeoutMs`, `maxRetries`, `maxRetryDelayMs`, and `metadata`.
- `streamSimple` reasoning does not take `'off'`; omit `reasoning` when not requesting reasoning.

```typescript
import { streamSimple } from '@earendil-works/pi-ai';

const s = streamSimple(model, context, {
  reasoning: 'high',
  temperature: 0.7,
  sessionId: 'prompt-cache-session-123',
});

for await (const event of s) {
  if (event.type === 'text_delta') process.stdout.write(event.delta);
}

const finalMessage = await s.result();
```

## Assistant Stream Events

`AssistantMessageEvent` variants:

| Event | Main fields | Notes |
| --- | --- | --- |
| `start` | `partial` | Stream opened. |
| `text_start` | `contentIndex`, `partial` | Text block started. |
| `text_delta` | `contentIndex`, `delta`, `partial` | Text chunk. |
| `text_end` | `contentIndex`, `content`, `partial` | Text block complete. |
| `thinking_start` | `contentIndex`, `partial` | Thinking block started. |
| `thinking_delta` | `contentIndex`, `delta`, `partial` | Thinking chunk. |
| `thinking_end` | `contentIndex`, `content`, `partial` | Thinking block complete. |
| `toolcall_start` | `contentIndex`, `partial` | Tool-call block started. |
| `toolcall_delta` | `contentIndex`, `delta`, `partial` | Raw JSON chunk; partial arguments may be best-effort parsed. |
| `toolcall_end` | `contentIndex`, `toolCall`, `partial` | Tool-call JSON complete. |
| `done` | `reason`, `message` | Successful terminal event; reason is `stop`, `length`, or `toolUse`. |
| `error` | `reason`, `error` | Error terminal event; reason is `error` or `aborted`. |

## Message Types

Main message roles are:

- `UserMessage`: `{ role: 'user', content, timestamp }`
- `AssistantMessage`: includes content blocks, provider/model metadata, `usage`, `stopReason`, optional `errorMessage`, `timestamp`
- `ToolResultMessage`: `{ role: 'toolResult', toolCallId, toolName, content, details?, isError, timestamp }`

Assistant content blocks are `text`, `thinking`, and `toolCall`. User/tool-result content supports `text` and `image` blocks.

## Utilities

**Public root exports include:** `isContextOverflow`, `getOverflowPatterns`, `parseStreamingJson`, `repairJson`, `parseJsonWithRepair`, `EventStream` utilities, diagnostics utilities, and TypeBox helpers.

```typescript
import { isContextOverflow, parseStreamingJson, repairJson } from '@earendil-works/pi-ai';

if (isContextOverflow(assistantMessage, model.contextWindow)) {
  // compact context rather than blindly retrying
}

const partialArgs = parseStreamingJson(rawToolArgumentChunk);
const fixed = repairJson(malformedJson);
```

`sanitizeSurrogates` and `shortHash` exist internally in `dist/utils/*` but are not exported by the package root or package exports in `0.74.0`; do not import them from `@earendil-works/pi-ai`.

## OAuth

Import OAuth helpers from the `@earendil-works/pi-ai/oauth` subpath.

```typescript
import { loginGitHubCopilot, getOAuthApiKey } from '@earendil-works/pi-ai/oauth';

const credentials = await loginGitHubCopilot({
  onAuth: (url, instructions) => console.log(url, instructions),
  onPrompt: async (prompt) => promptUser(prompt.message),
});

const authResult = await getOAuthApiKey('github-copilot', {
  'github-copilot': credentials,
});
```

Use OAuth helpers for OAuth-backed providers such as Anthropic, GitHub Copilot, and OpenAI Codex. Save refreshed credentials when `getOAuthApiKey` returns `newCredentials`. `refreshOAuthToken` exists but is deprecated in favor of `getOAuthProvider(id).refreshToken()`.

## Image Generation

There is no public `generateImages`, `getImageModel`, `getImageModels`, or `getImageProviders` export in `@earendil-works/pi-ai@0.74.0`. Do not document or call a separate image-generation API unless the package version adds it.

## Example: One-Shot Completion With Dynamic API Key

```typescript
import { completeSimple, getEnvApiKey, getModel } from '@earendil-works/pi-ai';

const model = getModel('openai', 'gpt-5.5');
const apiKey = getEnvApiKey(model.provider);

const message = await completeSimple(
  model,
  {
    systemPrompt: 'Answer tersely.',
    messages: [{ role: 'user', content: 'What changed in this diff?', timestamp: Date.now() }],
  },
  { apiKey, reasoning: 'low', maxTokens: 1000 },
);

if (message.stopReason === 'error') throw new Error(message.errorMessage);
console.log(message.content.filter((c) => c.type === 'text').map((c) => c.text).join('\n'));
```

## Example: User Message With Image Input

```typescript
import { completeSimple, getModel, type ImageContent } from '@earendil-works/pi-ai';

const screenshot: ImageContent = {
  type: 'image',
  mimeType: 'image/png',
  data: base64PngWithoutDataUrlPrefix,
};

const result = await completeSimple(getModel('google', 'gemini-3.1-pro-preview'), {
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Describe the visible UI bug.' },
        screenshot,
      ],
      timestamp: Date.now(),
    },
  ],
});
```

## Example: Register A Custom API Provider

```typescript
import {
  createAssistantMessageEventStream,
  registerApiProvider,
  type AssistantMessage,
} from '@earendil-works/pi-ai';

registerApiProvider({
  api: 'my-custom-api',
  stream: (model, context, options) => customStream(model, context, options),
  streamSimple: (model, context, options) => customStream(model, context, options),
}, 'my-extension');

function customStream(model, context, options) {
  const stream = createAssistantMessageEventStream();

  queueMicrotask(() => {
    const message: AssistantMessage = {
      role: 'assistant',
      content: [{ type: 'text', text: 'hello' }],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, totalTokens: 0, cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 } },
      stopReason: 'stop',
      timestamp: Date.now(),
    };
    stream.push({ type: 'start', partial: message });
    stream.push({ type: 'done', reason: 'stop', message });
    stream.end(message);
  });

  return stream;
}
```
