# Tool System

## What this component does

The Tool System provides the interface between LLM tool calls and actual execution. It handles:

- **Tool registration**: Built-in tools, MCP tools, and custom plugin tools
- **Schema validation**: Zod-based parameter validation with helpful error messages
- **Execution context**: Providing session, message, and permission context to tools
- **Output truncation**: Automatically truncating large outputs to fit context windows
- **Permission integration**: Requesting user approval before sensitive operations

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/tool/tool.ts` | Base Tool interface, `Tool.define()` helper |
| `packages/opencode/src/tool/registry.ts` | Tool registration, filtering by model/agent |
| `packages/opencode/src/tool/truncation.ts` | Output truncation logic |
| `packages/opencode/src/tool/bash.ts` | Bash command execution tool |
| `packages/opencode/src/tool/edit.ts` | File editing tool with diff support |
| `packages/opencode/src/tool/read.ts` | File reading with line limits |
| `packages/opencode/src/tool/grep.ts` | Content search via ripgrep |
| `packages/opencode/src/tool/glob.ts` | File pattern matching |
| `packages/opencode/src/tool/task.ts` | Sub-agent spawning tool |
| `packages/opencode/src/tool/write.ts` | File creation tool |
| `packages/opencode/src/tool/webfetch.ts` | URL fetching with HTML→markdown |
| `packages/opencode/src/tool/websearch.ts` | Web search integration |

## Core data structures

### Tool.Info
```typescript
interface Info<Parameters extends z.ZodType, Metadata> {
  id: string                    // Unique tool identifier
  init: (ctx?: InitContext) => Promise<{
    description: string         // LLM-facing description
    parameters: Parameters      // Zod schema for args
    execute: (args, ctx) => Promise<Result>
    formatValidationError?: (error: ZodError) => string
  }>
}
```

### Tool.Context
```typescript
interface Context {
  sessionID: string             // Current session
  messageID: string             // Current message
  agent: string                 // Active agent ID
  abort: AbortSignal            // Cancellation signal
  callID?: string               // Tool call identifier
  messages: MessageV2.WithParts[] // Conversation history
  metadata(input): void         // Update tool call metadata
  ask(input): Promise<void>     // Request permission
}
```

### Tool.Result
```typescript
interface Result {
  title: string                 // Display title for UI
  output: string                // Tool output text
  metadata: {
    truncated?: boolean         // Was output truncated?
    outputPath?: string         // Path to full output if truncated
    [key: string]: any          // Tool-specific metadata
  }
  attachments?: FilePart[]      // File attachments
}
```

### Built-in Tools
```typescript
// Registered in ToolRegistry.all()
const BUILTIN_TOOLS = [
  QuestionTool,     // Ask user questions
  BashTool,         // Execute shell commands
  ReadTool,         // Read files
  GlobTool,         // Find files by pattern
  GrepTool,         // Search file contents
  EditTool,         // Edit files with diffs
  WriteTool,        // Create new files
  TaskTool,         // Spawn sub-agents
  WebFetchTool,     // Fetch URLs
  TodoWriteTool,    // Manage task lists
  TodoReadTool,     // Read task lists
  WebSearchTool,    // Search the web
  CodeSearchTool,   // Semantic code search
  SkillTool,        // Execute skills
  ApplyPatchTool,   // Apply unified diffs
]
```

## Control flow (step-by-step)

### 1. Tool Registration
```
ToolRegistry.tools(model, agent)
  → Load built-in tools array
  → Load custom tools from Config.directories():
      glob "{tool,tools}/*.{js,ts}"
      import and wrap with fromPlugin()
  → Load plugin tools from Plugin.list()
  → Filter by model capabilities:
      - websearch/codesearch: require opencode provider or flag
      - apply_patch: only for gpt-* models
      - edit/write: excluded for patch models
  → Initialize each tool: tool.init({ agent })
  → Return { id, description, parameters, execute }[]
```

### 2. Tool Execution (in Processor)
```
processor receives tool-call event
  → Lookup tool by name in registry
  → Create context: { sessionID, messageID, agent, abort, ... }
  → Check permission: ctx.ask({ permission, patterns })
  → Execute: tool.execute(args, ctx)
  → Truncate output if needed
  → Return result to LLM
```

### 3. Permission Flow
```
tool.execute() calls ctx.ask()
  → PermissionNext.ask({
      permission: "bash",
      patterns: ["rm *"],
      ruleset: agent.permission
    })
  → Check ruleset for matching rule
  → If "allow": continue
  → If "deny": throw DeniedError
  → If "ask":
      → Bus.publish(Event.Asked)
      → Wait for user reply
      → If "reject": throw RejectedError
      → If "once"/"always": continue
```

### 4. Output Truncation
```
Truncate.output(content, options, agent)
  → Check content.length vs threshold
  → If under threshold: return as-is
  → Calculate available tokens from agent.maxTokens
  → Write full output to temp file
  → Truncate to fit context
  → Return { content: truncated, truncated: true, outputPath }
```

## Invariants / assumptions

1. **Zod validation first**: Parameters are validated before execute() is called

2. **Context always provided**: Every tool execution receives full Context object

3. **Truncation is automatic**: Tools don't need to handle truncation themselves (unless metadata.truncated is set)

4. **Permission before execution**: ctx.ask() must resolve before any side effects

5. **Single execution model**: Tools are stateless; each call is independent

6. **String output**: Tool results are always string (or converted to string)

7. **Abort signal honored**: Long-running tools should check ctx.abort.aborted

8. **Description files**: Many tools have companion `.txt` files with extended descriptions

## Open questions

1. **Tool versioning**: How are breaking changes to tool schemas handled?

2. **Concurrent execution**: Can multiple tools execute in parallel within one turn?

3. **Tool dependencies**: Can tools call other tools directly, or only via sub-agents?

4. **Streaming output**: Can tools stream partial results during execution?

5. **Timeout handling**: What happens when a tool exceeds execution time limits?

6. **MCP tool priority**: When MCP tool names conflict with built-ins, which wins?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add new tool | `tool/<name>.ts`, `registry.ts` imports, agent permissions |
| Modify tool schema | Tool file, possibly session replay compatibility |
| Change truncation | `truncation.ts`, may affect all tool outputs |
| Add tool capability | Tool file, model filtering in registry |
| Modify permission flow | `tool.ts` Context.ask, `permission/next.ts` |
| Add MCP tool | MCP server config, no code changes |
| Add plugin tool | Plugin package, `registry.ts` plugin loading |
