# `@earendil-works/pi-ai`

Stateless LLM/provider abstraction. Public APIs below were verified against the installed **`0.84.1`** package types.

> [!IMPORTANT]
> **The old "global" API is not in the package root.** `complete`, `completeSimple`, `stream`, `streamSimple`, `getModel`, `getModels`, `getProviders`, `getEnvApiKey`, `findEnvKeys`, and the `registerApiProvider`/`getApiProvider` registry live only in the temporary compat entrypoint **`@earendil-works/pi-ai/compat`**. New code uses a **`Models` collection** built with `createModels()` / `builtinModels()` and calls generation as **methods on that collection** (`models.complete(...)`, `models.streamSimple(...)`). Auth (env API keys, OAuth login/refresh) is resolved **inside** the collection, so call sites no longer pass `apiKey`. Compat's own docstring says it "is deleted with the coding-agent ModelManager migration" — treat it as a migration shim, not a destination.

## The `Models` Collection (core)

**Exports:** `createModels`, `createProvider`, `hasApi`, plus `builtinModels` / `builtinProviders` / `getBuiltinModel` / `getBuiltinModels` / `getBuiltinProviders` / `getBuiltinModelDataGeneratedAt` from the **`@earendil-works/pi-ai/providers/all`** subpath.

A `Models` collection holds providers, resolves request auth, owns login/logout, and delegates streaming/completion to the provider that owns each model. Build one per process and reuse it.

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
const auth = await models.getAuth(model!); // { auth: { apiKey, headers?, baseUrl? }, env?, source? } | undefined
```

`Models` methods:

| Method | Notes |
| --- | --- |
| `getProviders()` / `getProvider(id)` | Sync reads of registered providers. |
| `getModels(provider?)` / `getModel(provider, id)` | Sync reads of last-known state. A provider whose `getModels()` throws yields no models. |
| `refresh(options?)` | Re-fetch dynamic providers' model lists. **Takes `ModelsRefreshOptions`, not a provider string** (`{ allowNetwork?, providers?, force?, signal? }`) and resolves `ModelsRefreshResult` `{ aborted, errors: ReadonlyMap<string, Error> }` — provider errors are returned, never thrown. Static/unknown/unconfigured providers are skipped. |
| `checkAuth(providerId, options?)` | Is the provider fully configured, **without** running an OAuth refresh. Resolves `AuthCheck \| undefined`. |
| `getAvailable(providerId?, options?)` | Models whose providers have complete auth configuration (applies each provider's optional `filterModels`). |
| `getAuth(providerIdOrModel, overrides?)` | Async request-auth resolution; accepts a provider id **or** a model (the model form adds static model headers). |
| `login(providerId, type, interaction)` | Run the provider-owned login flow (`type` is `"api_key" \| "oauth"`) and persist the returned credential. |
| `logout(providerId, options?)` | Remove the stored credential. |
| `stream` / `complete` | Provider-specific `ModelsApiStreamOptions<TApi>`. |
| `streamSimple` / `completeSimple` | Unified `ModelsSimpleStreamOptions`. |
| `fetchDeferred(model, handle, options?)` / `cancelDeferred(...)` | Deferred-response polling/cancellation (see below). |

`MutableModels` (what `createModels`/`builtinModels` return) adds `setProvider(provider)` / `deleteProvider(id)` / `clearProviders()`.

`createModels(options?)` builds an **empty** collection. `CreateModelsOptions` is `{ credentials?: CredentialStore, modelsStore?: ModelsStore, authContext?: AuthContext }` (defaults: no stored credentials, no persisted catalog, `defaultProviderAuthContext()` reading `process.env`). `builtinModels(options?)` is the same thing **with every built-in provider pre-registered**.

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { builtinProviders } from '@earendil-works/pi-ai/providers/all';

const models = createModels();                 // empty
for (const p of builtinProviders()) models.setProvider(p);  // == builtinModels()
```

`ModelsStore` persists dynamic providers' fetched catalogs across process restarts (`read`/`write`/`delete` keyed by provider id; entries carry `models`, `lastModified`, `checkedAt`, `etag` for conditional refetch). `InMemoryModelsStore` is the process-local implementation.

> [!NOTE]
> The `./providers/*` and `./api/*` subpath exports define **only an `import` (ESM) condition** — no `require`. They resolve under ESM (a `"type": "module"` project, Nitro, Vite) but a CJS `require()` / `tsx -e '...'` context throws `ERR_PACKAGE_PATH_NOT_EXPORTED`. Import them from real ESM modules; for a quick smoke test use a `.ts`/`.mjs` file, not `node -e`/`tsx -e`.

## Catalog Reads

Sync model/provider lookups come from a `Models` collection (`models.getModel(...)`, `models.getProviders()`), or — without building a collection — from the typed generated catalog on the **`@earendil-works/pi-ai/providers/all`** subpath:

```typescript
import { getBuiltinModel, getBuiltinModels, getBuiltinProviders } from '@earendil-works/pi-ai/providers/all';

const model = getBuiltinModel('anthropic', 'claude-opus-4-5'); // typed Model
const ids = getBuiltinModels('openrouter');
const known = getBuiltinProviders();
```

`getBuiltinModelDataGeneratedAt()` returns when the bundled catalog was generated. The root-level `getModel`/`getModels`/`getProviders` are compat-only and deprecated. `hasApi(model, api)` narrows a dynamically looked-up `Model<Api>` to `Model<TApi>` for fully-typed stream options.

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

`ThinkingLevel` is `minimal | low | medium | high | xhigh | max`; `ModelThinkingLevel` adds `off`. `Usage` carries `input`/`output`/`cacheRead`/`cacheWrite` (+ optional `cacheWrite1h`, `reasoning`), `totalTokens`, and a `cost` breakdown.

## Auth (providers, credentials, context, login)

**Exports (root):** `envApiKeyAuth`, `lazyOAuth`, `defaultProviderAuthContext`, `InMemoryCredentialStore`, `ModelsError`, plus auth types (`ProviderAuth`, `ApiKeyAuth`, `OAuthAuth`, `AuthContext`, `AuthResult`, `AuthCheck`, `AuthType`, `AuthInteraction`, `AuthPrompt`, `AuthEvent`, `CredentialStore`, `Credential`, `ApiKeyCredential`, `OAuthCredential`, `CredentialInfo`, `ModelAuth`).

Every provider declares `auth: ProviderAuth` — at least one of `apiKey?: ApiKeyAuth` / `oauth?: OAuthAuth`, **including keyless/ambient providers** (their `resolve()` just reports whether the provider is configured). The `Models` collection resolves it per request via:

- **`AuthContext`** — environment access (`defaultProviderAuthContext()` reads `process.env` + checks file existence; inject a fake for tests/browsers).
- **`CredentialStore`** — optional app-owned stored credentials keyed by provider id. The interface is `read` / `list` / `modify` / `delete`; **`modify` is the only write path**, a serialized read-modify-write, and `Models.getAuth()` runs OAuth refresh inside it so concurrent requests cannot double-refresh a rotated token. `InMemoryCredentialStore` is the process-local implementation.

`envApiKeyAuth(name, envVars)` builds a standard api-key `ApiKeyAuth` whose `resolve()` prefers a stored credential, else the first set env var — what built-in api-key providers use. Because `builtinModels()` defaults to `defaultProviderAuthContext()`, **setting a provider's standard key env var is all that's needed** (e.g. `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`); no code change, no per-call `apiKey`. `lazyOAuth({ name, load })` wraps a dynamically imported `OAuthAuth` so a provider definition can advertise OAuth without pulling Node-only flow code into a bundle.

`getEnvApiKey` / `findEnvKeys` are **not at the root** — to read a resolved key directly, use `(await models.getAuth(model))?.auth.apiKey`.

## OAuth And Interactive Login

> [!WARNING]
> The **`@earendil-works/pi-ai/oauth` subpath is type-only** in 0.84 — it re-exports `OAuthCredentials`, `OAuthPrompt`, `OAuthLoginCallbacks`, `OAuthAuthInfo`, `OAuthDeviceCodeInfo`, `OAuthSelectOption`/`OAuthSelectPrompt` for coding-agent extension declarations and nothing else. The old value exports (`loginAnthropic`, `loginGitHubCopilot`, `loginOpenAICodex`, `getOAuthApiKey`, `getOAuthProvider(s)`, `refreshOAuthToken`, `registerOAuthProvider`, `anthropicOAuth`, …) **no longer exist**. Do not write imports against them.

Login is now provider-owned and driven through the collection. Pass a `CredentialStore` so the credential is persisted and refreshes stay locked:

```typescript
import { InMemoryCredentialStore, type AuthInteraction } from '@earendil-works/pi-ai';
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const models = builtinModels({ credentials: new InMemoryCredentialStore() });

const interaction: AuthInteraction = {
  prompt: async (p) => askUser(p.message),        // 'text' | 'secret' | 'select' | 'manual_code'
  notify: (event) => renderAuthEvent(event),      // 'info' | 'auth_url' | 'device_code' | 'progress'
};

await models.login('anthropic', 'oauth', interaction);   // or 'api_key'
const check = await models.checkAuth('anthropic');       // { type, source? } | undefined
await models.logout('anthropic');
```

`AuthPrompt` variants carry their own optional `signal` so a flow can cancel one pending prompt (e.g. a `manual_code` prompt racing a callback server) without aborting the whole login. Token refresh happens automatically inside `getAuth()`; a failed refresh rejects with `ModelsError` code `"oauth"` and preserves the stored credential for retry.

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

Validation throws formatted `Error`s. Let callers/tool loops handle those as tool errors. A `Tool` may also declare `constrainedSampling` (`{ type: 'json_schema', strict: 'prefer' | 'require' }` or `{ type: 'grammar', variants }`) for provider-side constrained decoding; set `false` to opt out.

## Generation

Generation is **methods on a `Models` collection** (or the compat free functions). Both signatures match the old globals minus auth, which the collection injects.

- `models.stream` / `models.complete` accept provider-specific `ModelsApiStreamOptions<TApi>` (= `ApiStreamOptions<TApi>` + `ModelsRequestTransforms`).
- `models.streamSimple` / `models.completeSimple` accept `ModelsSimpleStreamOptions` (= `SimpleStreamOptions` + `ModelsRequestTransforms`), including `reasoning?: ThinkingLevel`.
- `ModelsRequestTransforms` adds `transformHeaders(headers)` — a last-chance hook over the fully assembled model/auth/request headers before provider dispatch.
- Request-level options (`ProviderRequestOptions`): `signal`, `apiKey` (optional override), `fetch` (custom fetch implementation), `env` (provider-scoped env overrides taking precedence over `process.env`), `headers` (a `null` value suppresses a default header), `timeoutMs`, `maxRetries`, `maxRetryDelayMs`, `onPayload`, `onResponse`, `telemetryContext`.
- `StreamOptions` adds `temperature`, `maxTokens`, `samplingParams` (arbitrary extra body params for OpenAI-compatible servers — `top_p`, `min_p`, …), `transport`, `cacheRetention`, `sessionId`, `websocketConnectTimeoutMs`, `metadata`.
- `SimpleStreamOptions` adds `reasoning`, `thinkingBudgets`, and `deferred`.
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

## Deferred Responses

Capable providers can accept a request and return a durable handle instead of streaming to completion. Pass `deferred: true` (or `{ window: '15m' | '1h' | '24h' }`) in `SimpleStreamOptions`; the assistant message comes back with `stopReason: 'deferred'` and a `deferred: DeferredHandle` (`{ provider, modelId, api, id, expiresAt?, pollAfterMs?, data? }`). Persist that handle, then resolve it later:

```typescript
const started = await models.completeSimple(model, context, { deferred: { window: '1h' } });
if (started.stopReason === 'deferred' && started.deferred) {
  const finished = await models.fetchDeferred(model, started.deferred, { wait: 30_000 });
  // or: await models.cancelDeferred(model, started.deferred)
}
```

`wait` is the provider long-poll duration in ms (default 0 = one status check).

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
| `done` | `reason`, `message` | Successful terminal event; reason is `stop`, `length`, `toolUse`, or `deferred`. |
| `error` | `reason`, `error` | Error terminal event; reason is `error` or `aborted`. |

## Message Types

Main message roles are:

- `UserMessage`: `{ role: 'user', content, timestamp }`
- `AssistantMessage`: content blocks, `api`/`provider`/`model` (+ optional `responseModel`/`responseId`), `usage`, `stopReason`, optional `deferred`, `diagnostics`, `errorMessage`, `rawStopReason`, `timestamp`
- `ToolResultMessage`: `{ role: 'toolResult', toolCallId, toolName, content, details?, usage?, addedToolNames?, isError, timestamp }`

Assistant content blocks are `text`, `thinking` (with `thinkingSignature`, `redacted`), and `toolCall`. User/tool-result content supports `text` and `image` blocks. `StopReason` is `pending | stop | length | toolUse | error | aborted | deferred`.

## Utilities

**Public root exports include:** `isContextOverflow`, `isRecoverableLength`, `getOverflowPatterns`, `parseStreamingJson`, `repairJson`, `parseJsonWithRepair`, `EventStream` / `createAssistantMessageEventStream` / `AssistantMessageEventStream`, retry helpers (`retryAssistantCall`, `isRetryableAssistantError`, `RetryPolicy`, `RetryCallbacks`), diagnostics helpers (`createAssistantMessageDiagnostic`, `appendAssistantMessageDiagnostic`, `extractDiagnosticError`, `formatThrownValue`), session-resource helpers (`registerSessionResourceCleanup`, `cleanupSessionResources`), `contentText`, `uuidv7`, faux/test providers (`fauxProvider`, `createFauxCore`, `fauxAssistantMessage`, …), and TypeBox helpers.

```typescript
import { isContextOverflow, parseStreamingJson, repairJson } from '@earendil-works/pi-ai';

if (isContextOverflow(assistantMessage, model.contextWindow)) {
  // compact context rather than blindly retrying
}
const partialArgs = parseStreamingJson(rawToolArgumentChunk);
const fixed = repairJson(malformedJson);
```

`sanitizeSurrogates` and `shortHash` exist internally in `dist/utils/*` but are not exported by the package root or package exports; do not import them from `@earendil-works/pi-ai`.

## Image Generation

Image generation mirrors the chat API: `createImagesModels` / `createImagesProvider` at the root, `builtinImagesModels` / `builtinImagesProviders` from `@earendil-works/pi-ai/providers/all`, plus `ImagesModel` / `ImagesApi` / `MutableImagesModels` types. An `ImagesModels` collection mirrors `Models` (provider registry + auth resolution) for image models.

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

Custom providers go through `createProvider()` and are registered with `models.setProvider()`. A single `api` implementation streams all the provider's models; an `api` map dispatches on `model.api`. Dynamic catalogs use `fetchModels(context)` (the factory handles restore/publish transactionally against the `ModelsStore`); `filterModels(models, credential)` narrows availability per credential.

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
