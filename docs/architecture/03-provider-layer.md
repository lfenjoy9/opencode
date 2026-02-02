# Provider Layer

## What this component does

The Provider Layer abstracts multiple LLM backends (Anthropic, OpenAI, Bedrock, Google, etc.) into a unified interface using Vercel AI SDK. It handles:

- **Provider discovery**: Auto-detecting available providers based on credentials/environment
- **Model resolution**: Mapping user-friendly model names to provider-specific identifiers
- **Cost tracking**: Recording input/output token costs per model
- **Authentication**: Managing API keys, OAuth tokens, and cloud credentials
- **SDK instantiation**: Creating properly configured provider SDK instances

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/provider/provider.ts` | Main provider namespace - registration, resolution, model loading |
| `packages/opencode/src/provider/models.ts` | Model definitions, capabilities, cost data |
| `packages/opencode/src/provider/transform.ts` | Message/tool format transformation between providers |
| `packages/opencode/src/provider/auth.ts` | Provider-specific authentication flows |
| `packages/opencode/src/provider/sdk/copilot.ts` | GitHub Copilot custom SDK adapter |
| `packages/opencode/src/auth/index.ts` | Credential storage and retrieval |

## Core data structures

### Provider.Info
```typescript
interface Info {
  id: string                    // e.g., "anthropic", "openai", "bedrock"
  name: string                  // Human-readable name
  npm: string                   // AI SDK package: "@ai-sdk/anthropic"
  env: string[]                 // Required env vars: ["ANTHROPIC_API_KEY"]
  models: Record<string, Model> // Available models
  options?: Record<string, any> // SDK configuration
}
```

### Provider.Model
```typescript
interface Model {
  name: string                  // Display name
  attachment?: boolean          // Supports file attachments
  reasoning?: boolean           // Has reasoning/thinking capability
  computerUse?: boolean         // Supports computer use tools
  cost: {
    input: number               // Cost per 1M input tokens
    output: number              // Cost per 1M output tokens
  }
}
```

### Bundled Providers
```typescript
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/azure": createAzure,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  // ... 10+ more
}
```

## Control flow (step-by-step)

### 1. Provider Discovery
```
Provider.all()
  → Load ModelsDev.PROVIDERS (static definitions)
  → Merge with config.provider (user overrides)
  → For each provider:
      → Check CUSTOM_LOADERS[id]?.autoload
      → Check env vars / Auth credentials
      → Filter to providers with valid auth
```

### 2. Model Resolution
```
Provider.model(providerID, modelID)
  → Get provider info from Provider.all()
  → Get SDK via Provider.provider(providerID)
  → Get custom loader if exists:
      loader.getModel(sdk, modelID, options)
  → Else: sdk.languageModel(modelID)
  → Return LanguageModelV2 instance
```

### 3. SDK Instantiation
```
Provider.provider(providerID)
  → Lookup npm package in provider.npm
  → Check BUNDLED_PROVIDERS for factory
  → Load custom options from CUSTOM_LOADERS
  → Merge with config.provider[id].options
  → Call factory(options)
  → Return SDK instance
```

### 4. Authentication Flow
```
Auth.get(providerID)
  → Storage.read(["auth", providerID])
  → Return { type: "api" | "oauth" | "wellknown", key/token }

Provider uses auth:
  → Set process.env[envVar] = key
  → Pass to SDK options
```

## Invariants / assumptions

1. **Vercel AI SDK interface**: All providers implement `LanguageModelV2` from `ai` package

2. **Environment precedence**: Env vars > Auth storage > Config file for credentials

3. **Bundled by default**: Common providers are bundled; others installed via `bun add` at runtime

4. **Model ID mapping**: Some providers need ID transformation (e.g., Bedrock region prefixes)

5. **Cost data static**: Token costs defined in ModelsDev, not fetched dynamically

6. **Single SDK per provider**: One SDK instance cached per provider ID per session

7. **Custom loaders optional**: Providers without custom loaders use default `sdk.languageModel()`

## Open questions

1. **Dynamic model lists**: Can model lists be fetched from provider APIs rather than hardcoded?

2. **Cost accuracy**: How are costs kept up-to-date as providers change pricing?

3. **Rate limiting**: How are provider-specific rate limits handled across concurrent sessions?

4. **Fallback providers**: Is there automatic failover if a provider is unavailable?

5. **Model aliases**: How do shorthand names (e.g., "sonnet") resolve to full model IDs?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add new provider | `provider.ts` BUNDLED_PROVIDERS, ModelsDev, possibly custom loader |
| Add model to provider | `models.ts` model definitions |
| Change auth flow | `provider.ts` custom loader, `auth/` handlers, CLI auth command |
| Update costs | `models.ts` cost fields |
| Modify SDK options | `provider.ts` custom loader or config schema |
| Add model capability | `models.ts` Model type, tool filtering in registry |
