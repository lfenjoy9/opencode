# Server API

## What this component does

The Server API provides an HTTP interface to OpenCode's core functionality using the Hono framework. It enables:

- **REST endpoints**: CRUD operations for sessions, messages, providers, config
- **SSE streaming**: Real-time event broadcasting for UI updates
- **OpenAPI spec**: Auto-generated API documentation
- **Authentication**: Optional basic auth for remote access
- **CORS handling**: Safe cross-origin access for web/desktop clients

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/server/server.ts` | Main Hono app, middleware, route mounting |
| `packages/opencode/src/server/routes/session.ts` | Session CRUD, prompt, command execution |
| `packages/opencode/src/server/routes/provider.ts` | Provider listing, model info |
| `packages/opencode/src/server/routes/config.ts` | Configuration endpoints |
| `packages/opencode/src/server/routes/mcp.ts` | MCP server management |
| `packages/opencode/src/server/routes/file.ts` | File system operations |
| `packages/opencode/src/server/routes/permission.ts` | Permission responses |
| `packages/opencode/src/server/routes/tui.ts` | TUI-specific endpoints |
| `packages/opencode/src/server/routes/global.ts` | Global state endpoints |
| `packages/opencode/src/server/error.ts` | Error response helpers |

## Core data structures

### Hono App Structure
```typescript
const app = new Hono()
  .onError(errorHandler)        // Global error handling
  .use(authMiddleware)          // Optional basic auth
  .use(loggingMiddleware)       // Request logging
  .use(cors())                  // CORS configuration
  .route("/global", GlobalRoutes())
  .route("/session", SessionRoutes())
  .route("/provider", ProviderRoutes())
  .route("/config", ConfigRoutes())
  .route("/mcp", McpRoutes())
  .route("/file", FileRoutes())
  .route("/permission", PermissionRoutes())
  // ... more routes
```

### API Response Format
```typescript
// Success response
interface SuccessResponse<T> {
  data: T
}

// Error response (NamedError)
interface ErrorResponse {
  name: string                  // Error type: "NotFoundError"
  data: Record<string, any>     // Error details
}
```

### SSE Event Format
```typescript
interface SSEEvent {
  type: string                  // Event type: "message.part.updated"
  properties: Record<string, any>
}

// Common event types
type EventTypes =
  | "session.created"
  | "session.updated"
  | "session.idle"
  | "session.error"
  | "message.created"
  | "message.updated"
  | "message.part.updated"
  | "permission.asked"
  | "mcp.tools.changed"
```

## Control flow (step-by-step)

### 1. Server Startup
```
Server.serve(options)
  → App() returns configured Hono instance
  → Bun.serve({
      port: options.port ?? 4096,
      fetch: app.fetch,
      websocket: websocketHandler
    })
  → Optional: MDNS.broadcast() for discovery
  → Return server instance
```

### 2. Request Flow
```
HTTP Request arrives
  → CORS middleware (check origin)
  → Auth middleware (if password set)
  → Logging middleware (log method, path)
  → Route matching (Hono router)
  → Validator middleware (param, json, query)
  → Route handler execution
  → Response serialization (JSON)
  → Error boundary (catch and format)
```

### 3. SSE Event Streaming
```
GET /event
  → streamSSE(async (stream) => {
      → Subscribe to Bus events
      → For each event:
          stream.writeSSE({
            event: event.type,
            data: JSON.stringify(event.properties)
          })
      → On client disconnect: unsubscribe
    })
```

### 4. Session Prompt Flow
```
POST /session/:id/prompt
  → Validate session exists
  → Parse request body (parts, model, agent)
  → Session.addMessage({ role: "user", parts })
  → SessionProcessor.process(sessionID, options)
  → Return { data: { messageID } }
  → Processor publishes events → SSE stream
```

### 5. Permission Response Flow
```
POST /permission/respond
  → Validate request (sessionID, permissionID, response)
  → PermissionNext.reply({
      requestID: permissionID,
      reply: response  // "once" | "always" | "reject"
    })
  → Return success
  → Processor continues or aborts
```

## API Endpoints Reference

### Session Routes (`/session`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List all sessions |
| POST | `/` | Create new session |
| GET | `/:id` | Get session by ID |
| DELETE | `/:id` | Delete session |
| POST | `/:id/prompt` | Send prompt to session |
| POST | `/:id/command` | Execute slash command |
| POST | `/:id/abort` | Abort processing |
| POST | `/:id/share` | Create share link |
| GET | `/:id/messages` | Get session messages |

### Provider Routes (`/provider`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List available providers |
| GET | `/:id/models` | List models for provider |

### Config Routes (`/config`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Get current config |
| GET | `/agents` | List available agents |
| GET | `/commands` | List slash commands |

### MCP Routes (`/mcp`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List MCP servers |
| POST | `/:name/enable` | Enable MCP server |
| POST | `/:name/disable` | Disable MCP server |
| POST | `/:name/restart` | Restart MCP server |

### Event Route
| Method | Path | Description |
|--------|------|-------------|
| GET | `/event` | SSE event stream |

## Invariants / assumptions

1. **JSON-only**: All request/response bodies are JSON (except SSE)

2. **NamedError responses**: All errors wrapped in NamedError format with appropriate status codes

3. **OpenAPI compliance**: Routes decorated with `describeRoute()` for spec generation

4. **Zod validation**: All inputs validated via `validator()` middleware

5. **Event-driven updates**: State changes broadcast via Bus → SSE, not polling

6. **CORS whitelist**: localhost, tauri://, and *.opencode.ai allowed by default

7. **Optional auth**: Basic auth only enabled if OPENCODE_SERVER_PASSWORD set

8. **Stateless handlers**: All state in Session/Storage, not server memory

## Open questions

1. **Rate limiting**: Is there rate limiting for API endpoints?

2. **Request size limits**: What's the max request body size?

3. **WebSocket usage**: When is WebSocket used vs SSE?

4. **API versioning**: How are breaking API changes handled?

5. **Batch operations**: Can multiple operations be batched in one request?

6. **Pagination**: How are large lists (sessions, messages) paginated?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add endpoint | Route file, OpenAPI spec, SDK client |
| Modify response | Route handler, SDK types, UI consumers |
| Add middleware | `server.ts`, affects all routes |
| Change auth | `server.ts` auth middleware, client auth |
| Add SSE event | Bus event definition, SSE handler, SDK types |
| Modify validation | Route validators, error messages |
| Add route group | New route file, `server.ts` mounting |
