# CLI Entrypoint

## What this component does

The CLI entrypoint handles command-line parsing, routing to subcommands, and bootstrapping the application context. It transforms raw `process.argv` into typed argument objects, initializes project-scoped singletons, and dispatches to command handlers.

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/index.ts` | Main entry - yargs setup, command registration, error handling |
| `packages/opencode/src/cli/cmd/cmd.ts` | Type helper for defining commands with `--` passthrough |
| `packages/opencode/src/cli/cmd/run.ts` | Default command - prompt execution, event streaming |
| `packages/opencode/src/cli/cmd/auth.ts` | Provider authentication flows |
| `packages/opencode/src/cli/cmd/mcp.ts` | MCP server management |
| `packages/opencode/src/cli/cmd/agent.ts` | Agent CRUD operations |
| `packages/opencode/src/cli/cmd/serve.ts` | Standalone HTTP server mode |
| `packages/opencode/src/cli/bootstrap.ts` | `bootstrap()` wrapper for Instance initialization |
| `packages/opencode/src/cli/ui.ts` | Terminal output helpers, markdown rendering, styles |
| `packages/opencode/src/cli/error.ts` | `FormatError()` - user-friendly error formatting |

## Core data structures

### Yargs Command Module
```typescript
// Standard yargs interface, wrapped by cmd()
interface CommandModule<T, U> {
  command: string              // "run [message..]"
  describe: string             // Help text
  builder: (yargs: Argv<T>) => Argv<U>  // Option definitions
  handler: (args: U) => Promise<void>   // Execution logic
}
```

### Parsed Args Object
```typescript
// Output of yargs parsing for RunCommand
interface RunArgs {
  message: string[]           // Positional: words after "run"
  model?: string              // --model, -m
  continue?: boolean          // --continue, -c
  session?: string            // --session, -s
  file?: string[]             // --file, -f (array)
  format: "default" | "json"  // --format
  agent?: string              // --agent
  attach?: string             // --attach (server URL)
  port?: number               // --port
  variant?: string            // --variant (reasoning effort)
  title?: string              // --title
  share?: boolean             // --share
  "--": string[]              // Passthrough args after --
}
```

### Bootstrap Context
```typescript
// Instance.provide() creates scoped context
interface InstanceContext {
  directory: string           // Project root
  worktree: string            // Git worktree or directory
  // Lazy-initialized via Instance.state():
  config: Config.Info
  storage: Storage
  providers: Provider[]
  mcpServers: MCP.Server[]
}
```

## Control flow (step-by-step)

### 1. Process Start
```
bin/opencode.ts
  → import("../src/index.ts")
  → Top-level await: cli.parse()
```

### 2. Yargs Initialization
```
index.ts:
  yargs(hideBin(process.argv))
    .scriptName("opencode")
    .parserConfiguration({ "populate--": true })  // Capture -- args
    .option("print-logs", { global: true })
    .option("log-level", { global: true })
```

### 3. Middleware Execution
```
.middleware(async (opts) => {
  await Log.init({
    level: opts.logLevel,
    print: opts.printLogs
  })
  process.env.OPENCODE = "1"
})
```

### 4. Command Matching
```
.command(RunCommand)     // "run [message..]" - default
.command(AuthCommand)    // "auth"
.command(McpCommand)     // "mcp"
... 16 more commands

yargs matches argv[0] to command string
  → Invokes matched command's builder()
  → Validates/coerces options
  → Calls handler(args)
```

### 5. Handler Bootstrap
```
RunCommand.handler(args):
  // Attach mode - connect to existing server
  if (args.attach) {
    sdk = createOpencodeClient({ baseUrl: args.attach })
    → execute(sdk, sessionID)
    return
  }

  // Normal mode - bootstrap local instance
  await bootstrap(process.cwd(), async () => {
    // Creates in-process SDK client
    sdk = createOpencodeClient({
      baseUrl: "http://opencode.internal",
      fetch: Server.App().fetch  // Direct function call
    })
    → execute(sdk, sessionID)
  })
```

### 6. Bootstrap Internals
```
bootstrap(directory, fn):
  await Instance.provide({
    directory,
    init: InstanceBootstrap  // From project/bootstrap.ts
  }, async () => {
    try {
      await fn()
    } finally {
      await Instance.dispose()  // Cleanup MCP, etc.
    }
  })

InstanceBootstrap:
  → Config.get()        // Load layered config
  → Storage.init()      // Initialize file storage
  → Provider.init()     // Setup LLM providers
  → MCP.init()          // Start MCP servers
  → App.init()          // Initialize Hono server
```

## Invariants / assumptions

1. **Single command per invocation**: Yargs selects exactly one command handler per CLI call

2. **Global options before command**: `--print-logs` and `--log-level` apply to all commands via middleware

3. **Bootstrap required for most commands**: Commands accessing Instance/Session/Provider must call `bootstrap()`

4. **SDK abstraction**: Handlers never call Session/Provider directly; always through SDK client

5. **Passthrough args preserved**: Args after `--` are captured in `args["--"]` for forwarding

6. **Error boundary at top**: `cli.fail()` catches all errors, formats them, sets exit code

7. **TTY detection**: Commands check `process.stdin.isTTY` and `process.stdout.isTTY` for interactive vs piped mode

8. **Attach mode bypasses bootstrap**: `--attach` connects to existing server, skips local Instance init

## Open questions

1. **Command discovery**: Are commands statically imported or could they be dynamically loaded?

2. **Middleware ordering**: What happens if middleware throws? Does it prevent command execution?

3. **Completion generation**: How does shell completion work with dynamic options (e.g., model names)?

4. **Subcommand nesting**: Some commands (mcp, auth) have subcommands - how deep can nesting go?

5. **Config precedence in CLI**: Do CLI flags override config file values? (Appears yes for `--model`)

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add new command | `index.ts` registration, new file in `cli/cmd/`, update help |
| Add global option | `index.ts` yargs setup, middleware if needed, all handlers |
| Modify bootstrap | `cli/bootstrap.ts`, `project/bootstrap.ts`, all commands using it |
| Change arg parsing | Command's `builder()`, handler type signature, tests |
| Add command alias | Command's `command` string or yargs `.alias()` |
| Modify error format | `cli/error.ts`, affects all error display |
| Change SDK client | `@opencode-ai/sdk`, server routes, all handlers using SDK |
