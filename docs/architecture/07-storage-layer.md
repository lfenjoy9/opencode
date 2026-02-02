# Storage Layer

## What this component does

The Storage Layer provides file-based JSON persistence for all application state. It handles:

- **Key-value storage**: Hierarchical key paths mapped to JSON files
- **Concurrent access**: File locking for safe read/write operations
- **Migrations**: Schema evolution via numbered migration functions
- **Error handling**: Typed errors for common failure cases (NotFoundError)

## Key files

| File | Purpose |
|------|---------|
| `packages/opencode/src/storage/storage.ts` | Core Storage namespace - CRUD operations, migrations |
| `packages/opencode/src/util/lock.ts` | File locking utilities |
| `packages/opencode/src/global/index.ts` | Global.Path.data - storage root directory |
| `packages/opencode/src/session/index.ts` | Session storage (uses Storage internally) |
| `packages/opencode/src/permission/next.ts` | Permission storage (uses Storage internally) |

## Core data structures

### Storage Directory Layout
```
~/.local/share/opencode/storage/    # Linux (XDG_DATA_HOME)
~/Library/Application Support/opencode/storage/  # macOS
%APPDATA%/opencode/storage/         # Windows

storage/
├── migration                       # Current migration version (integer)
├── project/
│   └── {projectID}.json           # Project metadata
├── session/
│   └── {projectID}/
│       └── {sessionID}.json       # Session info
├── message/
│   └── {sessionID}/
│       └── {messageID}.json       # Message data
├── part/
│   └── {messageID}/
│       └── {partID}.json          # Message parts
├── permission/
│   └── {projectID}.json           # Approved permissions
├── session_diff/
│   └── {sessionID}.json           # Session diffs
└── auth/
    └── {providerID}.json          # Auth credentials
```

### Storage API
```typescript
namespace Storage {
  // Read JSON from key path
  read<T>(key: string[]): Promise<T>
  // Example: Storage.read(["session", projectID, sessionID])
  // → reads: storage/session/{projectID}/{sessionID}.json

  // Write JSON to key path
  write<T>(key: string[], content: T): Promise<void>

  // Update JSON in place
  update<T>(key: string[], fn: (draft: T) => void): Promise<T>

  // Delete file at key path
  remove(key: string[]): Promise<void>

  // List all keys under prefix
  list(prefix: string[]): Promise<string[][]>
  // Example: Storage.list(["session", projectID])
  // → returns array of [projectID, sessionID] keys
}
```

### Migration System
```typescript
type Migration = (dir: string) => Promise<void>

const MIGRATIONS: Migration[] = [
  // Migration 0→1: Restructure project directories
  async (dir) => {
    // Move from project/{hash}/storage/session/...
    // To session/{projectID}/...
  },

  // Migration 1→2: Extract session diffs
  async (dir) => {
    // Move session.summary.diffs to session_diff/
  },
]
```

## Control flow (step-by-step)

### 1. Storage Initialization
```
Storage.state() (lazy singleton)
  → dir = Global.Path.data + "/storage"
  → Read migration version from dir/migration
  → For each pending migration:
      → Run migration function
      → Write new version to dir/migration
  → Return { dir }
```

### 2. Read Operation
```
Storage.read<T>(["session", projectID, sessionID])
  → Get storage dir from state()
  → Build path: dir/session/{projectID}/{sessionID}.json
  → Acquire read lock: Lock.read(path)
  → Read and parse JSON: Bun.file(path).json()
  → Release lock (using statement)
  → Return parsed content as T
  → On ENOENT: throw NotFoundError
```

### 3. Write Operation
```
Storage.write(["session", projectID, sessionID], content)
  → Get storage dir from state()
  → Build path: dir/session/{projectID}/{sessionID}.json
  → Acquire write lock: Lock.write(path)
  → Serialize to JSON: JSON.stringify(content, null, 2)
  → Write file: Bun.write(path, json)
  → Release lock
```

### 4. Update Operation
```
Storage.update(key, fn)
  → Get storage dir from state()
  → Build path from key
  → Acquire write lock
  → Read current content
  → Apply mutation: fn(content)
  → Write updated content
  → Release lock
  → Return updated content
```

### 5. List Operation
```
Storage.list(["session", projectID])
  → Get storage dir from state()
  → Build prefix path: dir/session/{projectID}
  → Glob scan: **/* (all files)
  → Map to key arrays: filename → [...prefix, ...parts]
  → Sort and return
```

## Invariants / assumptions

1. **JSON format**: All stored data is JSON, pretty-printed with 2-space indent

2. **File-per-record**: Each key path maps to exactly one `.json` file

3. **Directory auto-creation**: Parent directories created automatically by Bun.write

4. **Lock scope**: Locks are per-file, not per-directory

5. **Migration order**: Migrations run sequentially, version tracked in `migration` file

6. **Atomic writes**: Bun.write is atomic (write to temp, then rename)

7. **Error translation**: ENOENT becomes NotFoundError, other errors propagate

8. **No transactions**: Operations are independent, no multi-file transactions

## Open questions

1. **Backup strategy**: Are storage files backed up? What's the recovery process?

2. **Corruption handling**: What happens if a JSON file is corrupted?

3. **Size limits**: Are there limits on individual file sizes or total storage?

4. **Cleanup policy**: Are old sessions/messages ever garbage collected?

5. **Encryption**: Is sensitive data (auth tokens) encrypted at rest?

6. **Concurrent migrations**: What if two processes try to migrate simultaneously?

7. **Index files**: Are there indexes for faster queries (e.g., by date)?

## Change impact

| Change | Affected Areas |
|--------|----------------|
| Add data type | Consumer code, possibly new migration |
| Change schema | Migration function, all readers of that data |
| Add migration | `storage.ts` MIGRATIONS array |
| Change storage path | `global/index.ts`, all hardcoded paths |
| Modify lock behavior | `storage.ts`, `util/lock.ts` |
| Add query capability | `storage.ts` new methods, consumers |
| Change serialization | `storage.ts` read/write, all stored data |
