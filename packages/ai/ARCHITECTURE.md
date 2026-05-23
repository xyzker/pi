# packages/ai Architecture

Unified abstraction over many LLM and image-generation providers.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Model** | A typed descriptor (`Model<Api>`) with id, provider, API type, pricing, capabilities, and compatibility overrides. |
| **Context** | A conversation state: system prompt + messages + available tools. |
| **Api** | A string identifier for a specific upstream API shape (e.g. `openai-completions`, `anthropic-messages`). |
| **Provider** | A string like `openai`, `anthropic`, `google` — a company/org that hosts models. |
| **StreamFunction** | A function that takes a `Model`, `Context`, and options, and returns an `AssistantMessageEventStream`. |
| **ApiProvider** | Pairs an `Api` with its `stream` and `streamSimple` implementations. |
| **AssistantMessageEventStream** | An async stream of delta events (text, thinking, tool calls) that resolves to a final `AssistantMessage`. |

---

## Key Files

### `src/types.ts`
The canonical data model for the entire package.

- **Messages**: `UserMessage`, `AssistantMessage`, `ToolResultMessage`
- **Content types**: `TextContent`, `ThinkingContent`, `ImageContent`, `ToolCall`
- **Streaming options**: `StreamOptions`, `SimpleStreamOptions` (adds reasoning/thinking budgets)
- **Model descriptor**: `Model<TApi>` with cost, context window, reasoning support, `thinkingLevelMap`, and compat flags (e.g. `OpenAICompletionsCompat`)
- **Event protocol**: `AssistantMessageEvent` — `start`, `text_delta`, `thinking_delta`, `toolcall_start`, `done`, `error`, etc.
- **Compatibility interfaces**: per-API compat flags for providers that speak the same API but with slightly different behaviors (e.g. `requiresThinkingAsText`, `thinkingFormat`, `cacheControlFormat`)

### `src/models.ts` + `models.generated.ts`
The model registry.

- `models.generated.ts` contains the static database of known models and their metadata.
- `models.ts` builds a runtime `Map<provider, Map<modelId, Model>>` from it.
- Exports: `getModel(provider, id)`, `getProviders()`, `getModels(provider)`, `calculateCost(model, usage)`, `clampThinkingLevel(model, level)`, `getSupportedThinkingLevels(model)`.

### `src/api-registry.ts`
Runtime registry for API implementations.

- `registerApiProvider(apiProvider, sourceId?)` — modules call this to bind an `Api` to its stream function.
- `getApiProvider(api)` — resolves the registered stream function for a given `Api` string.
- Supports extension-scoped registration/unregistration via `sourceId`.

### `src/stream.ts`
The main public API for streaming/completing.

```ts
stream(model, context, options?)       // raw per-API options
streamSimple(model, context, options?) // unified + reasoning level
complete(model, context, options?)     // awaits full message
completeSimple(model, context, options?)
```

These resolve the `Api` from the `Model`, look up the registered `ApiProvider`, and delegate.

### `src/providers/register-builtins.ts`
Bootstraps all built-in providers. Key design patterns:

1. **Lazy loading**: Provider modules are `import()`-ed on first use via `createLazyStream()` so heavy SDKs don't load unless needed.
2. **Registration**: Each provider implements `streamX` / `streamSimpleX` functions, which are then wrapped and registered via `registerApiProvider()` for their respective `Api` string.
3. **Bedrock override**: `setBedrockProviderModule()` allows the Node build to inject a custom Bedrock module (since it uses the AWS SDK, which can't easily be bundled for browser).

### `src/providers/simple-options.ts`
Shared logic for `streamSimple`.

- `buildBaseOptions()` — strips reasoning-specific keys, keeps common options.
- `adjustMaxTokensForThinking()` — reserves a thinking token budget within the model's max token cap.

### `src/utils/event-stream.ts`
The streaming primitive.

- `EventStream<T, R>` — generic async-iterable queue.
- `AssistantMessageEventStream` — specialisation for the `AssistantMessageEvent` protocol.
- Pushing events drives the stream; consumers iterate or call `.result()`.

### `src/images.ts` + `src/images-api-registry.ts`
Counterparts to the text API, but for image generation.

- `ImagesModel<ImagesApi>` — model descriptor for image APIs.
- `registerImagesApiProvider()` / `getImagesApiProvider()` — registry parallel to `api-registry.ts`.

---

## Provider Architecture

Each provider under `src/providers/` typically implements:

1. A `streamX` function that takes `Model<"api-name">`, `Context`, and typed options.
2. Translates the unified `Context` (messages/tools) into provider-specific payload.
3. Consumes the provider's streaming response.
4. Emits `AssistantMessageEvent` deltas (text, thinking, tool calls) via an `AssistantMessageEventStream`.
5. Ends with either `done` (carrying the final `AssistantMessage`) or `error`.

Providers handle their own quirks:

- **OpenAI** variants share code in `openai-responses-shared.ts`, `openai-prompt-cache.ts`, `transform-messages.ts`.
- **Anthropic** uses native SDK streaming.
- **Google** has shared logic in `google-shared.ts` for thinking levels.
- **Bedrock** is Node-only and loaded dynamically.
- **Faux** (`faux.ts`) is a dummy provider for testing.

---

## Data Flow

```
user code
    │
    ▼
streamSimple(model, context, { reasoning: "medium" })
    │
    ▼
model.api ──► api-registry.ts ──► getApiProvider("openai-responses")
    │
    ▼
streamSimpleOpenAIResponses(model, context, options)
    │
    ▼
(translate to OpenAI payload, make request, emit deltas)
    │
    ▼
AssistantMessageEventStream  ──►  user iterates / .result()
    │
    ▼
AssistantMessage (stopReason, usage, content[], etc.)
```

---

## Design Decisions

- **Unified model type**: Every `Model<Api>` has the same shape regardless of provider, with per-API `compat` overrides for edge cases.
- **Event-stream protocol**: Streaming is not raw byte streams; it's semantically typed deltas so consumers (the TUI, tools) can render thinking blocks, tool calls, and text independently.
- **Lazy provider loading**: `register-builtins.ts` wraps every provider in a lazy `import()` so unused providers don't bloat startup.
- **Api vs Provider separation**: One provider can serve models under multiple APIs (e.g. OpenAI has `openai-completions` and `openai-responses`), and one API can be spoken by multiple providers (e.g. OpenRouter speaks `openai-completions`).
