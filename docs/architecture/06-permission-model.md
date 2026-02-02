# Permission Model

## What this component does

The Permission Model controls which tool operations are allowed, denied, or require user confirmation. It uses a glob-based ruleset pattern matching system. Key responsibilities:

- **Rule evaluation**: Matching tool calls against permission rulesets
- **User prompting**: Requesting confirmation for "ask" actions
- **Persistent approvals**: Storing "always allow" decisions per project
- **Event coordination**: Publishing permission requests/replies via event bus

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/permission/next.ts` | Core permission logic - rules, evaluation, ask/reply |
| `packages/opencode/src/config/config.ts` | Permission schema in Config.Info |
| `packages/opencode/src/agent/agent.ts` | Agent-specific permission rulesets |
| `packages/opencode/src/tool/tool.ts` | Tool.Context.ask() interface |

## Core data structures

### PermissionNext.Rule
```typescript
interface Rule {
  permission: string            // Tool/action name: "bash", "read", "write"
  pattern: string               // Glob pattern: "rm *", "~/.ssh/*", "*"
  action: "allow" | "deny" | "ask"
}
```

### PermissionNext.Ruleset
```typescript
type Ruleset = Rule[]

// Example ruleset
[
  { permission: "bash", pattern: "git *", action: "allow" },
  { permission: "bash", pattern: "rm *", action: "ask" },
  { permission: "bash", pattern: "*", action: "ask" },
  { permission: "read", pattern: "~/.ssh/*", action: "deny" },
  { permission: "read", pattern: "*", action: "allow" },
  { permission: "write", pattern: "*", action: "ask" },
]
```

### PermissionNext.Request
```typescript
interface Request {
  id: string                    // Permission request ID
  sessionID: string             // Current session
  permission: string            // Tool name being requested
  patterns: string[]            // Specific patterns being matched
  metadata: Record<string, any> // Tool-specific context
  always: string[]              // Patterns for "always allow" option
  tool?: {
    messageID: string
    callID: string
  }
}
```

### PermissionNext.Reply
```typescript
type Reply = "once" | "always" | "reject"

// "once": Allow this specific invocation
// "always": Add patterns to approved list, allow future matches
// "reject": Deny and throw RejectedError
```

## Control flow (step-by-step)

### 1. Permission Check (Tool Execution)
```
Tool execution in processor:
  → ctx.ask({
      permission: "bash",
      patterns: ["git commit -m 'message'"],
      metadata: { command: "git commit..." },
      always: ["git *"]
    })
  → PermissionNext.ask(input)
```

### 2. Rule Evaluation
```
PermissionNext.ask(input)
  → For each pattern in input.patterns:
      → evaluate(permission, pattern, ruleset, approved)

evaluate(permission, pattern, ruleset, approved):
  → Check approved list first (persistent allowances)
      if match: return { action: "allow" }

  → Find matching rules in ruleset:
      rules.filter(r =>
        Wildcard.match(permission, r.permission) &&
        Wildcard.match(pattern, r.pattern)
      )

  → Return most specific match (last in list)
  → Default: { action: "ask" }
```

### 3. User Prompt Flow
```
If action === "ask":
  → Create Request object with unique ID
  → Store in pending map: pending[id] = { info, resolve, reject }
  → Bus.publish(Event.Asked, request)
  → Return Promise (waits for reply)

UI receives Event.Asked:
  → Display confirmation dialog
  → User selects: "Allow once" | "Always allow" | "Reject"
  → Call PermissionNext.reply({ requestID, reply })
```

### 4. Reply Handling
```
PermissionNext.reply({ requestID, reply })
  → Lookup pending[requestID]
  → Delete from pending

  If reply === "reject":
    → pending.reject(RejectedError)
    → Reject ALL pending permissions for this session
    → Bus.publish(Event.Replied)

  If reply === "once":
    → pending.resolve()
    → Bus.publish(Event.Replied)

  If reply === "always":
    → Add patterns to approved list
    → Storage.write(["permission", projectID], approved)
    → pending.resolve()
    → Bus.publish(Event.Replied)
```

### 5. Ruleset Sources
```
Final ruleset = merge(
  Config.permission,            // From config file
  Agent.permission,             // Agent-specific rules
  Session.permission,           // Session-level overrides
)

// Later rules take precedence
// Approved list checked before ruleset
```

## Invariants / assumptions

1. **Last rule wins**: When multiple rules match, the last one in the array takes precedence

2. **Glob patterns**: Uses `Wildcard.match()` for pattern matching (supports `*`, `?`, `**`)

3. **Path expansion**: `~/` and `$HOME` expanded to actual home directory

4. **Project-scoped approvals**: "Always allow" stored per project ID, persists across sessions

5. **Session isolation**: Rejecting one permission rejects all pending for that session

6. **Synchronous check**: Permission check blocks tool execution until resolved

7. **No implicit allow**: Default action is "ask" if no rule matches

8. **Permission names match tools**: Permission string typically matches tool ID

## Open questions

1. **Rule ordering**: Should rules be explicitly ordered or use specificity-based matching?

2. **Approval expiry**: Do "always allow" approvals ever expire or need refresh?

3. **Audit logging**: Are permission decisions logged for security review?

4. **Batch approvals**: Can multiple pending permissions be approved at once?

5. **Pattern suggestions**: How are "always" patterns derived from specific invocations?

6. **Inheritance**: Do sub-agents inherit parent session's permissions?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add permission type | Tool using ctx.ask(), ruleset patterns |
| Modify evaluation | `next.ts` evaluate(), all permission checks |
| Change storage | `next.ts` state(), Storage calls |
| Add reply option | `next.ts` Reply type, reply(), UI handlers |
| Modify approval persistence | `next.ts` state(), Storage schema |
| Add ruleset source | `next.ts` ask(), merge logic |
| Change glob syntax | `next.ts` evaluate(), Wildcard module |
