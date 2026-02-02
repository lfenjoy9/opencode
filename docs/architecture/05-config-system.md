# Config System

## What this component does

The Config System provides layered configuration with multiple precedence levels. It handles:

- **Multi-layer merging**: Managed → Remote → Global → Project config precedence
- **File formats**: JSON and JSONC (JSON with comments) support
- **Schema validation**: Zod-based validation with helpful error messages
- **Dynamic loading**: Agents, commands, modes, and skills from config directories
- **Environment integration**: Flag overrides and env var expansion

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/config/config.ts` | Main config namespace - loading, merging, schema |
| `packages/opencode/src/config/markdown.ts` | Markdown frontmatter parsing for agents/commands |
| `packages/opencode/src/global/index.ts` | Global paths (data, config, home directories) |
| `packages/opencode/src/flag/flag.ts` | Environment variable flags |
| `packages/opencode/src/env/index.ts` | Environment variable access |

## Core data structures

### Config.Info (Schema)
```typescript
interface Info {
  $schema?: string              // JSON schema URL

  // Model settings
  model?: string                // Default model: "provider/model"

  // Provider overrides
  provider?: Record<string, {
    disabled?: boolean
    env?: Record<string, string>
    models?: Record<string, Model>
    options?: Record<string, any>
  }>

  // Permissions
  permission?: Permission       // Tool permission rules

  // Custom definitions
  agent?: Record<string, Agent> // Custom agents
  mode?: Record<string, Agent>  // Mode definitions
  command?: Record<string, Command> // Slash commands

  // Plugins
  plugin?: string[]             // Plugin package names
  instructions?: string[]       // Additional instruction files

  // UI/UX
  theme?: string                // UI theme name
  keybinds?: Keybinds          // Custom key bindings

  // Features
  share?: "auto" | "manual" | false
  mcp?: McpConfig              // MCP server configs
  experimental?: {
    batch_tool?: boolean
  }
  compaction?: {
    auto?: boolean
    prune?: boolean
  }
}
```

### Config.Permission
```typescript
// Simple form
type Permission = Record<string, "allow" | "deny" | "ask">

// Pattern form
type Permission = Record<string, Record<string, "allow" | "deny" | "ask">>

// Example
{
  "bash": "ask",                    // All bash commands need approval
  "read": {
    "~/.ssh/*": "deny",             // Deny reading SSH keys
    "*": "allow"                    // Allow reading everything else
  }
}
```

### Config Directories
```typescript
// Scanned in order (later overrides earlier)
const directories = [
  "/etc/opencode",              // Managed (enterprise)
  "~/.config/opencode",         // Global user config
  "~/.opencode/",               // User home directory
  ".opencode/",                 // Project directory (walked up)
]

// Config files checked
const configFiles = [
  "opencode.jsonc",
  "opencode.json"
]
```

## Control flow (step-by-step)

### 1. Config Loading
```
Config.state()
  → Initialize empty result: {}

  // Layer 1: Remote/Well-known (lowest precedence)
  → For each wellknown auth:
      fetch("{url}/.well-known/opencode")
      mergeConfigConcatArrays(result, remoteConfig)

  // Layer 2: Global user config
  → result = merge(result, await global())

  // Layer 3: Custom config path (OPENCODE_CONFIG env)
  → if Flag.OPENCODE_CONFIG:
      result = merge(result, loadFile(path))

  // Layer 4: Project config (highest precedence)
  → if !Flag.OPENCODE_DISABLE_PROJECT_CONFIG:
      for file in findUp(["opencode.jsonc", "opencode.json"]):
        result = merge(result, loadFile(file))

  // Layer 5: Inline content (OPENCODE_CONFIG_CONTENT)
  → if Flag.OPENCODE_CONFIG_CONTENT:
      result = merge(result, JSON.parse(content))

  → Apply defaults and flag overrides
  → Return { config, directories }
```

### 2. Dynamic Content Loading
```
Config.state() also scans directories for:

  // Agents: {agent,agents}/**/*.md
  → Parse frontmatter + markdown body
  → Validate against Agent schema
  → Add to config.agent[name]

  // Commands: {command,commands}/**/*.md
  → Parse frontmatter + template body
  → Validate against Command schema
  → Add to config.command[name]

  // Modes: {mode,modes}/*.md
  → Parse like agents
  → Add to config.mode[name]

  // Skills: {skill,skills}/**/*.md
  → Load skill definitions
```

### 3. Config Merge Strategy
```
mergeConfigConcatArrays(target, source):
  → Deep merge objects (source overrides target)
  → Special handling for arrays:
      - plugin: concatenate and deduplicate
      - instructions: concatenate and deduplicate
  → Return merged config
```

### 4. Config Access
```
Config.get()
  → Return cached config from state()
  → Singleton per Instance

Config.directories()
  → Return list of scanned directories
  → Used by ToolRegistry for custom tools
```

## Invariants / assumptions

1. **Precedence order**: Managed > Inline > Project > Custom > Global > Remote

2. **JSONC support**: Comments allowed in config files via jsonc-parser

3. **Array concatenation**: `plugin` and `instructions` arrays merge, not replace

4. **Schema validation**: Invalid config throws `Config.InvalidError`

5. **Directory scanning**: `.opencode/` directories walked up from project root

6. **Lazy loading**: Config loaded once per Instance, cached thereafter

7. **Environment expansion**: Paths like `~/` expanded to home directory

8. **Managed immutable**: `/etc/opencode` (or platform equivalent) for admin control

## Open questions

1. **Hot reload**: Can config changes be picked up without restart?

2. **Schema versioning**: How are config schema changes handled across versions?

3. **Validation timing**: When exactly does validation occur - load time or access time?

4. **Circular refs**: Can agents/commands reference each other?

5. **Plugin config**: How do plugins contribute their own config schemas?

6. **Encryption**: Are sensitive values (API keys in config) encrypted at rest?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add config field | `config.ts` Info schema, docs, possibly UI |
| Change precedence | `config.ts` state() load order |
| Add config layer | `config.ts` state(), possibly auth integration |
| Modify agent schema | `config.ts` Agent schema, agent loading |
| Add command field | `config.ts` Command schema, command execution |
| Change merge behavior | `config.ts` mergeConfigConcatArrays |
| Add directory source | `config.ts` directories array |
