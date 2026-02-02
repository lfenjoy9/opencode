# TUI App

## What this component does

The TUI (Terminal User Interface) App provides a rich graphical interface for OpenCode using SolidJS and a custom terminal rendering framework (OpenTUI). It handles:

- **Session UI**: Chat interface with message rendering, tool outputs, diffs
- **Real-time updates**: SSE subscription for live streaming responses
- **Permission prompts**: Interactive approval dialogs for tool execution
- **Multi-project**: Directory selection and project switching
- **Theming**: Customizable color schemes and layouts

## Key files

| File | Purpose |
|------|---------|
| `packages/app/src/app.tsx` | Root app component, provider hierarchy |
| `packages/app/src/entry.tsx` | Entry point, platform detection |
| `packages/app/src/pages/session.tsx` | Main session/chat page |
| `packages/app/src/pages/home.tsx` | Project/directory selection |
| `packages/app/src/pages/layout.tsx` | App shell layout |
| `packages/app/src/context/server.ts` | Server connection context |
| `packages/app/src/context/global-sdk.ts` | SDK client provider |
| `packages/app/src/context/permission.ts` | Permission prompt handling |
| `packages/app/src/context/prompt.ts` | Input prompt state |
| `packages/app/src/context/terminal.ts` | Terminal/PTY integration |
| `packages/ui/src/` | Shared UI components library |

## Core data structures

### Provider Hierarchy
```typescript
// Outermost to innermost provider nesting
AppBaseProviders
  → MetaProvider           // Document meta tags
  → Font                   // Font loading
  → ThemeProvider          // Theme state
  → LanguageProvider       // i18n
  → ErrorBoundary          // Error catching
  → DialogProvider         // Modal dialogs
  → MarkedProvider         // Markdown rendering

AppInterface
  → ServerProvider         // Server URL, connection
  → GlobalSDKProvider      // SDK client instance
  → GlobalSyncProvider     // Global state sync
  → Router                 // SolidJS router
    → SettingsProvider     // User settings
    → PermissionProvider   // Permission prompts
    → LayoutProvider       // Layout state
    → NotificationProvider // Toast notifications
    → ModelsProvider       // Available models
    → CommandProvider      // Slash commands
    → HighlightsProvider   // Code highlights
```

### Server Context
```typescript
interface ServerContext {
  url: string                   // Server URL
  setUrl: (url: string) => void
  status: "connected" | "disconnected" | "connecting"
}
```

### SDK Context
```typescript
interface GlobalSDKContext {
  sdk: OpencodeClient           // @opencode-ai/sdk client
  subscribe: () => EventStream  // SSE subscription
}
```

### Permission Context
```typescript
interface PermissionContext {
  pending: PermissionRequest[]  // Pending approval requests
  respond: (id: string, reply: Reply) => void
}
```

## Control flow (step-by-step)

### 1. App Initialization
```
entry.tsx
  → Detect platform (web, desktop, terminal)
  → Load platform-specific adapters
  → Render <App /> with providers

app.tsx
  → AppBaseProviders (theme, fonts, i18n)
  → AppInterface (server, SDK, routing)
  → Router mounts pages based on URL
```

### 2. Server Connection
```
ServerProvider
  → Initialize with default URL (localhost:4096 or env)
  → Create SDK client: createOpencodeClient({ baseUrl })
  → Subscribe to SSE: sdk.event.subscribe()
  → On connection: set status="connected"
  → On error: set status="disconnected", retry
```

### 3. Session Page Load
```
Route: /:dir/session/:id?
  → TerminalProvider (PTY connection)
  → FileProvider (file tree state)
  → PromptProvider (input state)
  → CommentsProvider (inline comments)
  → <Session /> component

Session component:
  → Load session data: sdk.session.get(id)
  → Load messages: sdk.session.messages(id)
  → Subscribe to SSE events for this session
  → Render message list + input prompt
```

### 4. Message Streaming
```
User submits prompt:
  → PromptProvider.submit(text, files)
  → sdk.session.prompt({ sessionID, parts })
  → SSE events arrive:
      "message.created" → Add message to list
      "message.part.updated" → Update part in message
      "session.error" → Show error notification
      "session.idle" → Processing complete
```

### 5. Permission Handling
```
SSE: "permission.asked"
  → PermissionProvider receives event
  → Add to pending list
  → Render permission dialog
  → User clicks Allow/Deny
  → sdk.permission.respond({ permissionID, response })
  → Remove from pending list
```

### 6. Real-time Rendering
```
Message part updates:
  → SolidJS reactive signal updated
  → Component re-renders affected parts
  → Text streams character-by-character
  → Tool results appear on completion
  → Diffs rendered with syntax highlighting
```

## Invariants / assumptions

1. **SolidJS reactivity**: All state managed via SolidJS signals/stores

2. **Provider-based DI**: All shared state accessed via context providers

3. **SSE-driven updates**: UI state updated from SSE events, not polling

4. **SDK abstraction**: All server communication via @opencode-ai/sdk

5. **Route-based sessions**: Session ID in URL path, enables deep linking

6. **Lazy loading**: Pages loaded lazily via `lazy()` for faster startup

7. **Platform adapters**: Platform-specific code injected via context

8. **Shared UI library**: Common components in `packages/ui/`

## Open questions

1. **Offline support**: What happens when server connection is lost mid-session?

2. **State persistence**: Is any UI state persisted locally (drafts, preferences)?

3. **Keyboard navigation**: How comprehensive is keyboard-only navigation?

4. **Accessibility**: What accessibility features are implemented (screen readers)?

5. **Mobile support**: Is the UI responsive for mobile/tablet?

6. **Performance**: How are large message lists virtualized?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add page | `pages/` directory, router in `app.tsx` |
| Add context | `context/` directory, provider hierarchy in `app.tsx` |
| Modify theme | `packages/ui/` theme system, ThemeProvider |
| Add component | `components/` or `packages/ui/src/` |
| Change SDK usage | Context providers using SDK, hooks |
| Add SSE event | SDK types, event handlers in contexts |
| Modify layout | `pages/layout.tsx`, LayoutProvider |
| Add platform | `entry.tsx` detection, platform adapters |
