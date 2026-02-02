# MCP Integration

## What this component does

The MCP (Model Context Protocol) Integration enables OpenCode to connect to external tool servers using the standardized MCP protocol. It handles:

- **Server discovery**: Loading MCP server configs from configuration
- **Transport management**: Supporting stdio, SSE, and HTTP transports
- **Tool bridging**: Converting MCP tools to AI SDK tool format
- **Authentication**: OAuth and API key flows for remote MCP servers
- **Lifecycle management**: Starting, stopping, and reconnecting servers

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/mcp/index.ts` | Main MCP namespace - client management, tool conversion |
| `packages/opencode/src/mcp/auth.ts` | MCP authentication credential storage |
| `packages/opencode/src/mcp/oauth-provider.ts` | OAuth provider implementation for MCP |
| `packages/opencode/src/mcp/oauth-callback.ts` | OAuth callback handling |
| `packages/opencode/src/server/routes/mcp.ts` | HTTP routes for MCP management |
| `packages/opencode/src/cli/cmd/mcp.ts` | CLI commands for MCP server management |

## Core data structures

### MCP.Config (from Config.Info)
```typescript
interface McpConfig {
  [serverName: string]: {
    // Stdio transport (local process)
    command?: string            // e.g., "npx"
    args?: string[]             // e.g., ["-y", "@modelcontextprotocol/server-filesystem"]
    env?: Record<string, string>

    // HTTP/SSE transport (remote server)
    url?: string                // e.g., "https://mcp.example.com/sse"

    // Common options
    enabled?: boolean           // Default: true
    timeout?: number            // Request timeout in ms
  }
}
```

### MCP.Status
```typescript
type Status =
  | { status: "connected" }
  | { status: "disabled" }
  | { status: "failed"; error: string }
  | { status: "needs_auth" }
  | { status: "needs_client_registration"; error: string }
```

### MCP.Resource
```typescript
interface Resource {
  name: string                  // Resource name
  uri: string                   // Resource URI
  description?: string
  mimeType?: string
  client: string                // MCP server name
}
```

### Internal State
```typescript
// Per-server state in Instance.state()
interface ServerState {
  name: string
  client: MCPClient             // @modelcontextprotocol/sdk Client
  transport: Transport          // Stdio | SSE | HTTP
  tools: Tool[]                 // Converted AI SDK tools
  resources: Resource[]         // Available resources
  status: Status
}
```

## Control flow (step-by-step)

### 1. MCP Initialization
```
MCP.state() (Instance-scoped lazy singleton)
  → Load MCP config from Config.get()
  → For each server in config:
      → If disabled: skip
      → Create transport based on config:
          - command+args → StdioClientTransport
          - url (SSE) → SSEClientTransport
          - url (HTTP) → StreamableHTTPClientTransport
      → Create Client from @modelcontextprotocol/sdk
      → Connect with timeout
      → On success: fetch tools, set status=connected
      → On auth error: set status=needs_auth
      → On failure: set status=failed, log error
```

### 2. Tool Discovery
```
MCP.tools()
  → Get all connected servers from state
  → For each server:
      → client.listTools()
      → Convert each MCPTool to AI SDK Tool:
          convertMcpTool(mcpTool, client, timeout)
      → Prefix tool name with server name: "{server}_{tool}"
  → Return combined tool array
```

### 3. Tool Conversion
```
convertMcpTool(mcpTool, client, timeout)
  → Extract inputSchema from MCPTool
  → Create AI SDK dynamicTool:
      description: mcpTool.description
      inputSchema: jsonSchema(schema)
      execute: async (args) => {
        client.callTool({
          name: mcpTool.name,
          arguments: args
        })
      }
```

### 4. Tool Execution
```
MCP tool called via AI SDK:
  → dynamicTool.execute(args)
  → client.callTool({ name, arguments })
  → MCP server processes request
  → Return CallToolResult
  → Convert to string output
```

### 5. OAuth Authentication
```
Server returns UnauthorizedError:
  → Set status=needs_auth
  → User initiates auth via CLI/TUI
  → McpOAuthProvider.authorize():
      → Get authorization URL
      → Open browser for user consent
      → McpOAuthCallback handles redirect
      → Store tokens via McpAuth
  → Reconnect server with tokens
```

### 6. Tool Change Notification
```
MCP server sends ToolListChangedNotification:
  → Client notification handler fires
  → Bus.publish(MCP.ToolsChanged, { server })
  → Listeners refresh tool list
  → SessionPrompt picks up new tools
```

## Invariants / assumptions

1. **AI SDK compatibility**: All MCP tools converted to Vercel AI SDK `Tool` interface

2. **Namespaced tools**: MCP tools prefixed with server name to avoid collisions

3. **Lazy connection**: Servers connect on first access, not at startup

4. **Timeout protection**: All MCP calls have configurable timeout (default 30s)

5. **OAuth state persistence**: OAuth tokens stored in McpAuth for reconnection

6. **Transport auto-detection**: URL ending in `/sse` uses SSE, otherwise HTTP

7. **Process lifecycle**: Stdio servers are child processes, killed on shutdown

8. **Tool schema passthrough**: MCP tool schemas passed directly to LLM

## Open questions

1. **Resource usage**: How are MCP resources (not tools) exposed to the LLM?

2. **Server health**: Is there periodic health checking for connected servers?

3. **Reconnection strategy**: What triggers reconnection after transient failures?

4. **Tool caching**: Are tool definitions cached, or fetched on every prompt?

5. **Concurrent calls**: Can multiple tool calls go to the same MCP server in parallel?

6. **Streaming results**: Does MCP support streaming tool results?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add transport type | `mcp/index.ts` transport creation logic |
| Modify auth flow | `mcp/oauth-*.ts`, `mcp/auth.ts`, server routes |
| Add MCP feature | `mcp/index.ts`, possibly session prompt |
| Change tool naming | `mcp/index.ts` tool conversion, all tool references |
| Add server config | `config/config.ts` schema, config loading |
| Modify timeout | `mcp/index.ts` DEFAULT_TIMEOUT, per-server config |
