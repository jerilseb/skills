# `@earendil-works/pi-ai`

Stateless LLM/provider abstraction. Public APIs below are based on the installed `0.80.2` package exports.

> [!IMPORTANT]
> **0.80 moved the old "global" API out of the package root.** `complete`, `completeSimple`, `stream`, `streamSimple`, `getModel`, `getModels`, `getProviders`, `getEnvApiKey`, `findEnvKeys`, and the `registerApiProvider`/`getApiProvider` registry are **no longer exported from `@earendil-works/pi-ai`**. They now live only in the temporary compat entrypoint **`@earendil-works/pi-ai/compat`** (`export * from "./index.ts"` plus the moved symbols). New code uses a **`Models` collection** built with `createModels()` / `builtinModels()` and calls generation as **methods on that collection** (`models.complete(...)`, `models.streamSimple(...)`). Auth (env API keys, OAuth) is resolved **inside** the collection, so call sites no longer pass `apiKey`. The compat module's own docstring says it "is deleted with the coding-agent ModelManager migration" — treat it as a migration shim, not a destination.

## The `Models` Collection (core)

**Exports:** `createModels`, `createProvider`, `hasApi`, plus `builtinModels` / `builtinProviders` / `getBuiltinModel` / `getBuiltinModels` / `getBuiltinProviders` from the **`@earendil-works/pi-ai/providers/all`** subpath.

A `Models` collection holds providers, resolves request auth, and delegates streaming/completion to the provider that owns each model. Build one per process and reuse it.

```typescript
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

// Every built-in provider registered; API keys resolved per request from the
// default auth context (process.env). No options needed for env-key auth.
const models = builtinModels();

const providers = models.getProviders();             // readonly Provider[]
const provider = models.getProvider('openrouter');   // Provider | undefined
const list = models.getModels('anthropic');          // last-known model list
const model = models.getModel('anthropic', 'claude-opus-4-5'); // Model<Api> | undefined

// Auth resolution (status UIs / pre-flight). undefined ⇒ provider unknown or
// unconfigured. Rejects with ModelsError on OAuth-refresh/credential failure.
const auth = await models.getAuth(model!); // { auth: { apiKey, headers?, baseUrl? }, source? } | undefined
```

`Models` methods:

- `getProviders()` / `getProvider(id)` / `getModels(provider?)` / `getModel(provider, id)` — sync reads of last-known state.
- `getAuth(model)` — async; resolves request auth for a model (or `undefined` when unconfigured).
- `complete(model, context, options?)` / `stream(model, context, options?)` — provider-specific `ApiStreamOptions`.
- `completeSimple(model, context, options?)` / `streamSimple(model, context, options?)` — unified `SimpleStreamOptions`.
- `refresh(provider?)` — re-fetch dynamic providers' model lists (static providers are no-ops).

`MutableModels` (what `createModels`/`builtinModels` return) adds `setProvider(provider)` / `deleteProvider(id)` / `clearProviders()`.

`createModels(options?)` builds an **empty** collection; `options` is `{ credentials?: CredentialStore, authContext?: AuthContext }` (defaults: no stored credentials, `defaultProviderAuthContext()` reading `process.env`). Use `builtinModels(options?)` to get the same thing **with every built-in provider pre-registered**.

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { builtinProviders } from '@earendil-works/pi-ai/providers/all';

const models = createModels();                 // empty
for (const p of builtinProviders()) models.setProvider(p);  // == builtinModels()
```

> [!NOTE]
> The `./providers/*` and `./api/*` subpath exports define **only an `import` (ESM) condition** — no `require`. They resolve under ESM (a `"type": "module"` project, Nitro, Vite) but a CJS `require()` / `tsx -e '...'` context throws `ERR_PACKAGE_PATH_NOT_EXPORTED`. Import them from real ESM modules; for a quick smoke test use a `.ts`/`.mjs` file, not `node -e`/`tsx -e`.

## Catalog Reads

Sync model/provider lookups now come from a `Models` collection (`models.getModel(...)`, `models.getProviders()`), or — without building a collection — from the typed generated catalog on the **`@earendil-works/pi-ai/providers/all`** subpath:

```typescript
import { getBuiltinModel, getBuiltinModels, getBuiltinProviders } from '@earendil-works/pi-ai/providers/all';

const model = getBuiltinModel('anthropic', 'claude-opus-4-5'); // typed Model
const ids = getBuiltinModels('openrouter');
const known = getBuiltinProviders();
```

The root-level `getModel`/`getModels`/`getProviders` are gone (compat-only, deprecated). `hasApi(model, api)` narrows a dynamically looked-up `Model<Api>` to `Model<TApi>` for fully-typed stream options.

## Model & Thinking Utilities

**Exports (root):** `modelsAreEqual`, `getSupportedThinkingLevels`, `clampThinkingLevel`, `calculateCost`, plus model/usage types.

```typescript
import {
  getSupportedThinkingLevels,
  clampThinkingLevel,
  calculateCost,
  modelsAreEqual,
  type Model,
  type Usage,
} from '@earendil-works/pi-ai';

const levels = getSupportedThinkingLevels(model); // ModelThinkingLevel[] (includes 'off' when applicable)
const safe = clampThinkingLevel(model, 'xhigh');
const cost = calculateCost(model, usage);
const same = modelsAreEqual(model, model);
```

## Auth (providers, credentials, context)

**Exports (root):** `envApiKeyAuth`, `defaultProviderAuthContext`, `InMemoryCredentialStore`, `ModelsError`, plus auth types (`ProviderAuth`, `ApiKeyAuth`, `OAuthAuth`, `AuthContext`, `CredentialStore`, `Credential`, `ModelAuth`, `AuthResult`).

Every provider declares `auth: ProviderAuth` (`apiKey?` / `oauth?`). The `Models` collection resolves it per request via:

- **`AuthContext`** — environment access (`defaultProviderAuthContext()` reads `process.env` + `node:fs`; inject a fake for tests/browsers).
- **`CredentialStore`** — optional app-owned stored credentials keyed by provider id (`InMemoryCredentialStore` for a process-local one; OAuth refresh runs serialized inside `store.modify`).

`envApiKeyAuth(name, envVars)` builds a standard api-key `ApiKeyAuth` whose `resolve()` prefers a stored credential, else the first set env var — what built-in api-key providers use. Because `builtinModels()` defaults to `defaultProviderAuthContext()`, **setting a provider's standard key env var is all that's needed** (e.g. `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`); no code change, no per-call `apiKey`.

`getEnvApiKey` / `findEnvKeys` are **no longer at the root** — to read a resolved key directly, use `(await models.getAuth(model))?.auth.apiKey`, or import `getEnvApiKey` from `@earendil-works/pi-ai/compat` if you must.

## Tools & Validation

**Exports (root):** `Type`, `Static`, `TSchema`, `StringEnum`, `validateToolArguments`, `validateToolCall`, `Tool`.

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

## Generation

Generation is now **methods on a `Models` collection** (or the compat free functions). Both signatures match the old globals minus auth, which the collection injects.

- `models.stream` / `models.complete` accept provider-specific `ApiStreamOptions<TApi>` (extends `StreamOptions`).
- `models.streamSimple` / `models.completeSimple` accept unified `SimpleStreamOptions`, including `reasoning?: ThinkingLevel`.
- Common `StreamOptions`: `temperature`, `maxTokens`, `signal`, `apiKey` (optional override — the collection resolves auth otherwise), `transport`, `cacheRetention`, `sessionId`, `onPayload`, `onResponse`, `headers`, `timeoutMs`, `maxRetries`, `maxRetryDelayMs`, `metadata`.
- `streamSimple` reasoning does **not** take `'off'`; omit `reasoning` to disable it.

```typescript
const s = models.streamSimple(model, context, {
  reasoning: 'high',
  temperature: 0.7,
  sessionId: 'prompt-cache-session-123',
});

for await (const event of s) {
  if (event.type === 'text_delta') process.stdout.write(event.delta);
}
const finalMessage = await s.result();
```

`Models.streamSimple` satisfies pi-agent-core's `StreamFn`, so the modern way to give an `Agent` auth is `streamFn: (m, c, o) => models.streamSimple(m, c, o)` — no `getApiKey`.

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

**Public root exports include:** `isContextOverflow`, `getOverflowPatterns`, `parseStreamingJson`, `repairJson`, `parseJsonWithRepair`, `EventStream` / `createAssistantMessageEventStream` / `AssistantMessageEventStream`, diagnostics helpers (`createAssistantMessageDiagnostic`, `appendAssistantMessageDiagnostic`, `extractDiagnosticError`, `formatThrownValue`), session-resource helpers (`registerSessionResourceCleanup`, `cleanupSessionResources`), faux/test providers (`fauxProvider`, `createFauxCore`, `fauxAssistantMessage`, …), and TypeBox helpers.

```typescript
import { isContextOverflow, parseStreamingJson, repairJson } from '@earendil-works/pi-ai';

if (isContextOverflow(assistantMessage, model.contextWindow)) {
  // compact context rather than blindly retrying
}
const partialArgs = parseStreamingJson(rawToolArgumentChunk);
const fixed = repairJson(malformedJson);
```

`sanitizeSurrogates` and `shortHash` exist internally in `dist/utils/*` but are not exported by the package root or package exports; do not import them from `@earendil-works/pi-ai`.

## OAuth

Import OAuth helpers from the `@earendil-works/pi-ai/oauth` subpath.

```typescript
import { loginGitHubCopilot, getOAuthApiKey, getOAuthProvider } from '@earendil-works/pi-ai/oauth';

const credentials = await loginGitHubCopilot({
  notify: (event) => console.log(event),
  prompt: async (p) => promptUser(p.message),
});

const authResult = await getOAuthApiKey('github-copilot', { 'github-copilot': credentials });
```

Available OAuth helpers include `loginAnthropic` / `loginGitHubCopilot` / `loginOpenAICodex` (+ `loginOpenAICodexDeviceCode`), `refreshOAuthToken` (deprecated — prefer `getOAuthProvider(id).refreshToken()`), `getOAuthApiKey`, `getOAuthProvider` / `getOAuthProviders` / `getOAuthProviderInfoList`, the per-provider `anthropicOAuth` / `githubCopilotOAuth` / `openaiCodexOAuth` objects, and `registerOAuthProvider` / `unregisterOAuthProvider` / `resetOAuthProviders`. Save refreshed credentials when a flow returns new ones. In OAuth-backed `Models` collections, pass a `CredentialStore` so `getAuth()` owns the locked refresh.

## Image Generation

0.80 **does** expose an image-generation API (parallel to the chat one): `createImagesModels` / `createImagesProvider` at the root, `builtinImagesModels` / `builtinImagesProviders` from `@earendil-works/pi-ai/providers/all`, plus `ImagesModel` / `ImagesApi` / `MutableImagesModels` types. An `ImagesModels` collection mirrors `Models` (provider registry + auth resolution) for image models.

## Example: One-Shot Completion

```typescript
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const models = builtinModels();                          // reuse per process
const model = models.getModel('openai', 'gpt-5.5');
if (!model) throw new Error('unknown model');

const message = await models.completeSimple(
  model,
  {
    systemPrompt: 'Answer tersely.',
    messages: [{ role: 'user', content: 'What changed in this diff?', timestamp: Date.now() }],
  },
  { reasoning: 'low', maxTokens: 1000 }, // auth resolved by the collection — no apiKey
);

if (message.stopReason === 'error') throw new Error(message.errorMessage);
console.log(message.content.filter((c) => c.type === 'text').map((c) => c.text).join('\n'));
```

## Example: User Message With Image Input

```typescript
import { builtinModels } from '@earendil-works/pi-ai/providers/all';
import type { ImageContent } from '@earendil-works/pi-ai';

const models = builtinModels();
const screenshot: ImageContent = {
  type: 'image',
  mimeType: 'image/png',
  data: base64PngWithoutDataUrlPrefix,
};

const result = await models.completeSimple(models.getModel('google', 'gemini-3.1-pro-preview')!, {
  messages: [
    {
      role: 'user',
      content: [{ type: 'text', text: 'Describe the visible UI bug.' }, screenshot],
      timestamp: Date.now(),
    },
  ],
});
```

## Example: Register A Custom Provider

Custom providers go through `createProvider()` and are registered with `models.setProvider()`. A single `api` implementation streams all the provider's models; an `api` map dispatches on `model.api`.

```typescript
import { createProvider, envApiKeyAuth, createAssistantMessageEventStream, type AssistantMessage } from '@earendil-works/pi-ai';
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const myProvider = createProvider({
  id: 'my-provider',
  name: 'My Provider',
  auth: { apiKey: envApiKeyAuth('My Provider API key', ['MY_PROVIDER_API_KEY']) },
  models: [/* Model<...>[] */],
  api: {
    stream: (model, context, options) => customStream(model, context, options),
    streamSimple: (model, context, options) => customStream(model, context, options),
  },
});

const models = builtinModels();
models.setProvider(myProvider);

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

For a custom `api` string the framework can't emit natively, the compat-only `registerApiProvider()` registry still exists (`@earendil-works/pi-ai/compat`), but prefer `createProvider()` + `setProvider()`.
