# Agent Runtime

## What this component does

The Agent Runtime is the core execution engine that orchestrates the agentic loop: receiving user input, streaming LLM responses, executing tools, and persisting session state. It handles:

- **Prompt assembly**: Combining system prompts, instructions, MCP tools, and conversation history
- **LLM streaming**: Managing streaming responses from multiple LLM providers via Vercel AI SDK
- **Tool execution**: Dispatching tool calls with permission checks and result handling
- **Session management**: Message persistence, compaction, summarization, and usage tracking
- **Error recovery**: Retries, rate limiting, and graceful degradation

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/session/processor.ts` | Core agentic loop - streams LLM, executes tools, handles state transitions |
| `packages/opencode/src/session/prompt.ts` | Assembles chat messages, system prompts, and tool definitions |
| `packages/opencode/src/session/llm.ts` | LLM streaming wrapper with retry logic and provider abstraction |
| `packages/opencode/src/session/index.ts` | Session CRUD, message storage, usage calculation |
| `packages/opencode/src/session/message.ts` | MessageV2 structure and part management |
| `packages/opencode/src/agent/agent.ts` | Agent definitions with permission rulesets and tool filtering |
| `packages/opencode/src/tool/registry.ts` | Tool registration, filtering by model/agent, plugin loading |
| `packages/opencode/src/tool/tool.ts` | Base Tool interface and execution context |
| `packages/opencode/src/permission/next.ts` | Permission checking with glob-based rulesets |
| `packages/opencode/src/provider/provider.ts` | Multi-provider LLM abstraction (Anthropic, OpenAI, Bedrock, etc.) |

## Core data structures

### Session
```typescript
// packages/opencode/src/session/index.ts
Session.Info = {
  id: string                    // Unique session ID
  parentID?: string             // For sub-agent sessions
  version: 2                    // Schema version
  title?: string                // Auto-generated or user-set
  createdAt: number             // Unix timestamp
  modelID: string               // e.g., "claude-sonnet-4-20250514"
  providerID: string            // e.g., "anthropic"
}
```

### MessageV2
```typescript
// packages/opencode/src/session/message.ts
MessageV2.Info = {
  id: string
  sessionID: string
  role: "user" | "assistant"
  parts: MessageV2.Part[]       // Text, tool calls, tool results
  createdAt: number
  metadata: {
    assistant?: { model, provider, summary? }
  }
}

MessageV2.Part =
  | { type: "text", text: string }
  | { type: "tool-invocation", toolInvocation: { id, name, args, state, result? } }
  | { type: "file", ... }
  | { type: "summary", ... }
```

### Agent
```typescript
// packages/opencode/src/agent/agent.ts
Agent.Info = {
  id: string                    // e.g., "default", "build", "plan"
  system?: string[]             // System prompt sections
  permission: Permission.Ruleset // Tool permission rules
  tools?: string[]              // Allowed tool IDs (whitelist)
  maxTokens?: number
  model?: string
}
```

### Tool
```typescript
// packages/opencode/src/tool/tool.ts
Tool.Info = {
  id: string                    // e.g., "bash", "edit", "read"
  init: (ctx) => Promise<{
    parameters: ZodSchema       // Input validation
    description: string         // For LLM
    execute: (args, ctx) => Promise<Tool.Result>
  }>
}

Tool.Result = {
  title: string                 // Display title
  output: string                // Tool output text
  metadata?: { ... }            // Truncation info, etc.
}
```

## Control flow (step-by-step)

### 1. User Input → Session
```
CLI: RunCommand.handler()
  → Instance.init(directory)
  → Session.create() or Session.get()
  → Session.addMessage({ role: "user", parts: [...] })
  → Bus.publish(MessageV2.Event.Created)
```

### 2. Prompt Assembly
```
SessionPrompt.chat(sessionID, modelID, agent)
  → InstructionPrompt.build()           // User instructions from config
  → SystemPrompt.generate()             // Agent-specific system prompt
  → MCP.tools()                         // External MCP server tools
  → ToolRegistry.tools(model, agent)    // Filtered tool definitions
  → Session.messages()                  // Conversation history
  → Return { system, tools, messages }
```

### 3. Agentic Loop (Processor)
```
SessionProcessor.process(sessionID, options)
  LOOP:
    → SessionPrompt.chat()              // Build prompt
    → LLM.stream(prompt)                // Call provider

    FOR EACH stream event:
      "text-delta":
        → Session.updatePart(textPart)  // Stream text to UI

      "tool-call":
        → PermissionNext.check(ruleset, toolName, args)
        → IF denied: throw RejectedError
        → IF ask: Bus.publish(Confirmation.Event.Pending)
        → Tool.execute(args, context)
        → Session.updatePart(toolResult)

      "tool-result":
        → Session.updatePart({ state: "completed" })

      "finish-step":
        → Update usage stats
        → Check compaction threshold
        → IF has tool results: CONTINUE LOOP
        → IF no tool results: BREAK

      "error":
        → Retry with backoff OR throw
```

### 4. Result Persistence
```
Session.updateMessage(messageID, parts)
  → Storage.write(path, message)
  → Bus.publish(MessageV2.Event.Updated)
  → Server SSE broadcasts to connected clients
```

## Invariants / assumptions

1. **Single active loop per session**: Only one `SessionProcessor.process()` runs per session at a time (enforced by `processing` lock)

2. **Tool results continue loop**: If the LLM returns tool calls, the loop continues after execution until a text-only response

3. **Permission before execution**: All tool executions pass through `PermissionNext.check()` before running

4. **Message append-only**: Messages are never deleted mid-session (only compaction removes old messages)

5. **Storage is file-based**: All session data persists to `~/.opencode/storage/` as JSON files

6. **Provider abstraction**: All LLM calls go through `Provider.provider()` → Vercel AI SDK, never direct API calls

7. **Streaming is required**: The runtime assumes streaming responses; non-streaming models need wrapper adapters

8. **Tool output truncation**: Large outputs are automatically truncated via `Truncate.output()` to stay within context limits

## Open questions

1. **Compaction trigger**: What exact token threshold triggers message compaction? (Appears to be model-specific maxTokens)

2. **Sub-agent isolation**: How does `parentID` affect session state isolation? Can sub-agents access parent session data?

3. **MCP tool priority**: When MCP tools conflict with built-in tool names, which takes precedence?

4. **Retry semantics**: What errors trigger LLM retries vs immediate failure? (Rate limits vs auth errors)

5. **Snapshot timing**: When exactly are snapshots created during the agentic loop? (Appears to be on tool execution)

6. **Permission inheritance**: Do sub-agents inherit parent agent permissions or start fresh?

7. **Streaming backpressure**: How does the system handle slow consumers of the SSE stream?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add new tool | `tool/registry.ts`, `tool/<name>.ts`, permission rules in agents |
| New LLM provider | `provider/provider.ts`, `provider/sdk/`, possibly `provider/transform.ts` |
| Modify prompt structure | `session/prompt.ts`, `agent/prompt/`, may affect all agent behaviors |
| Change permission model | `permission/next.ts`, all agent definitions, CLI confirmation flow |
| Add message part type | `session/message.ts`, `session/processor.ts`, TUI rendering in `packages/app/` |
| Modify storage schema | `storage/storage.ts`, migration script needed, all session reads |
| Change streaming format | `session/llm.ts`, `server/routes/`, TUI `packages/app/src/hooks/` |
| Add agent type | `agent/agent.ts`, `agent/prompt/`, `cli/cmd/run.ts` agent selection |
