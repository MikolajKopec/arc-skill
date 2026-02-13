# Data Storage & Database Adapters — Deep Dive

## Architecture

```
MasterDataStorage
├── Map<storeName, MasterStoreState>    // One store per view/event table
├── DatabaseAdapter (transaction-based)
├── RealTimeCommunicationAdapter (optional)
└── fork() → ForkedDataStorage
                ├── Map<storeName, ForkedStoreState>  // In-memory buffer
                └── merge() → commits to MasterDataStorage
```

## DataStorage (Abstract)

```typescript
abstract class DataStorage {
  abstract getStore<Item extends { _id: string }>(storeName: string): StoreState<Item>
  abstract fork(): ForkedDataStorage
  abstract getReadTransaction(): Promise<ReadTransaction>
  abstract getReadWriteTransaction(): Promise<ReadWriteTransaction>

  // Applies changes to all affected stores
  async commitChanges(changes: DataStorageChanges[]): Promise<void>
}
```

## MasterDataStorage

The main storage instance connected to a database:

```typescript
import { MasterDataStorage } from "@arcote.tech/arc"

const dataStorage = new MasterDataStorage(
  databaseAdapterPromise,  // Promise<DatabaseAdapter> | DatabaseAdapter
  rtcAdapterFactory?,      // optional RTC adapter factory
  context,                 // ArcContext
)

// Get a store for a specific table
const store = dataStorage.getStore<User>("userProfile")

// Fork for transaction isolation
const fork = dataStorage.fork()

// Sync with RTC — callback receives { store, size } object
await dataStorage.sync(({ store, size }) => {
  console.log(`Syncing ${store}: ${size} items`)
})

// Apply changes from external sources
await dataStorage.applyChanges(changes)
await dataStorage.applySerializedChanges(serializedChanges)
```

## ForkedDataStorage

Transaction isolation — buffers all changes in memory until merge:

```typescript
const fork = dataStorage.fork()

// All operations happen in memory
const store = fork.getStore<User>("userProfile")
await store.set({ _id: "usr_1", name: "Jan", email: "jan@example.com" })
await store.modify("usr_1", { name: "Jan Updated" })

// Nothing written to DB yet!

// Commit all changes atomically
await fork.merge()  // → MasterDataStorage → DB + RTC (async, must be awaited)
```

### Fork Behavior

- **Reads:** First checks fork buffer, falls back to master
- **Writes:** Buffered in memory (Map)
- **Merge:** Applies all buffered changes to master in one batch
- **Discard:** Just let the fork be garbage collected

### Nested Forks

Async listeners get their own fork:
```
Command fork
  ├── sync listener 1 (same fork)
  ├── sync listener 2 (same fork)
  └── merge to master
      ├── async listener 1 (new fork → merge)
      ├── async listener 2 (new fork → merge)
      └── async listener 3 (new fork → merge)
```

## StoreState (Abstract)

Interface for CRUD operations on a single table:

```typescript
abstract class StoreState<Item extends { _id: string }> {
  // Create or replace
  set(item: Item): Promise<void>

  // Soft delete
  remove(id: string): Promise<void>

  // Query with conditions
  abstract find(options: FindOptions<Item>): Promise<Item[]>

  // Partial update (returns old and new values, nullable)
  modify(id: string, data: Partial<Item>): Promise<{ from: Item | null, to: Item | null }>

  // Immutable edit via Mutative draft (returns old and new values)
  mutate(id: string, editCallback: (item: Item) => Promise<void>): Promise<{ from: Item | null, to: Item | null }>

  // Apply batch changes
  abstract applyChanges(changes: StoreStateChange<Item>[]): Promise<void>
}
```

## StoreStateChange Types

```typescript
type StoreStateChange<Item> =
  | { type: "set"; data: Item }                     // Create/replace — uses "data", not "item"
  | { type: "delete"; id: string }                  // Soft delete — uses "delete", not "remove"
  | { type: "modify"; id: string; data: Partial<Item> }  // Partial update
  | { type: "mutate"; id: string; patches: Patch[] }     // Mutative patches (via mutative lib)
```

**Important naming:** The change types use `data` (not `item`) for set operations and `delete` (not `remove`) for deletions. The `mutate` type applies [Mutative](https://github.com/nicolo-ribaudo/mutative) patches.

## DataStorageChanges

```typescript
type DataStorageChanges = {
  store: string;                    // Table/store name
  changes: StoreStateChange<any>[];  // Array of changes
}
```

## Database Adapters

### DatabaseAdapter Interface

The adapter uses a **transaction-based API**, not direct `exec()`:

```typescript
interface DatabaseAdapter {
  readWriteTransaction(stores?: string[]): ReadWriteTransaction
  readTransaction(stores?: string[]): ReadTransaction
  executeReinitTables(dataStorage: MasterDataStorage): Promise<void>
}
```

**Transactions** provide the actual `exec()` and `execBatch()` methods.

### PostgreSQL Adapter

```typescript
import { postgreSQLAdapterFactory } from "@arcote.tech/arc-host"

// Uses Bun.SQL for native PostgreSQL
```

**Features:**
- Parameterized queries: `$1, $2, $3`
- Quoted identifiers: `"columnName"` (case-sensitive)
- Soft delete: `WHERE deleted = false`
- JSON/JSONB columns auto-deserialized
- `execBatch()` for atomic multi-statement execution
- IN/NOT IN with array expansion
- Version counter tracking tables (`__arc_version_counters`, `__arc_table_versions`)

**SQL generation examples:**
```sql
-- find({ where: { status: "active", age: { $gte: 18 } }, limit: 10 })
SELECT * FROM "userProfile"
WHERE deleted = false AND "status" = $1 AND "age" >= $2
LIMIT 10

-- set({ _id: "usr_1", name: "Jan", email: "jan@e.com" })
INSERT INTO "userProfile" ("_id", "name", "email")
VALUES ($1, $2, $3)
ON CONFLICT ("_id") DO UPDATE SET "name" = $2, "email" = $3

-- modify("usr_1", { name: "Jan2" })
UPDATE "userProfile" SET "name" = $1, "_v" = "_v" + 1
WHERE "_id" = $2

-- remove("usr_1")
UPDATE "userProfile" SET "deleted" = true WHERE "_id" = $1
```

### SQLite Adapter

```typescript
import { hostLiveModel } from "@arcote.tech/arc-host"
// SQLite adapter created internally via sqliteAdapterFactory
```

**Differences from PostgreSQL:**
- Parameterized queries: `?, ?, ?`
- Uses quoted identifiers in column definitions
- Soft delete: `WHERE deleted = 0` (integer)
- JSON stored and parsed
- Version tracking infrastructure similar to PostgreSQL

## Where Operators

| Operator | SQL (PostgreSQL) | SQL (SQLite) | Description |
|----------|------------------|--------------|-------------|
| `$eq` | `= $1` | `= ?` | Equal |
| `$ne` | `!= $1` | `!= ?` | Not equal |
| `$gt` | `> $1` | `> ?` | Greater than |
| `$gte` | `>= $1` | `>= ?` | Greater or equal |
| `$lt` | `< $1` | `< ?` | Less than |
| `$lte` | `<= $1` | `<= ?` | Less or equal |
| `$in` | `IN ($1, $2, ...)` | `IN (?, ?, ...)` | In array |
| `$nin` | `NOT IN ($1, ...)` | `NOT IN (?, ...)` | Not in array |
| `$exists` | `IS NOT NULL` / `IS NULL` | `IS NOT NULL` / `IS NULL` | Exists check |
| (direct value) | `= $1` | `= ?` | Shorthand for $eq |

## FindOptions

```typescript
interface FindOptions<T> {
  where?: {
    [K in keyof T]?: T[K] | {
      $eq?: T[K]
      $ne?: T[K]
      $gt?: T[K]
      $gte?: T[K]
      $lt?: T[K]
      $lte?: T[K]
      $in?: T[K][]
      $nin?: T[K][]
      $exists?: boolean
    }
  }
  limit?: number
  offset?: number
  orderBy?: { [key: string]: "asc" | "desc" }
}
```

## Transaction Caching

MasterStoreState uses a transaction cache during batch operations:

```typescript
// When applying changes from a fork merge:
// 1. Cache existing items to avoid repeated DB reads
// 2. Apply all changes in batch
// 3. Clear cache
```

## Real-Time Communication

```typescript
interface RealTimeCommunicationAdapter {
  commitChanges(changes: DataStorageChanges[]): void
  sync(progressCallback: ({ store, size }: { store: string; size: number }) => void): Promise<void>
}
```

### WebSocket Messages (RTC Protocol)

**Client → Host:**
```typescript
type MessageClientToHost =
  | { type: "sync"; lastDate: string | null }
  | { type: "changes-executed"; changes: DataStorageChanges[] }
```

**Host → Client:**
```typescript
type MessageHostToClient =
  | { type: "sync-result"; store: string; items: any[] }
  | { type: "state-changes"; changes: DataStorageChanges[] }
  | { type: "sync-done"; date: string }
```

## Deep Merge Utility

```typescript
// packages/core/data-storage/deep-merge.ts
deepMerge(target: object, source: Partial<object>): object
// Recursively merges, handling nested objects
```

## Complete Example: Setting Up Storage

```typescript
import { MasterDataStorage, Model } from "@arcote.tech/arc"

// Server-side: use hostLiveModel which handles adapter creation
import { hostLiveModel } from "@arcote.tech/arc-host"
hostLiveModel(appContext, { db: "app.db", version: 1 })

// Or manually for custom setups:
const dataStorage = new MasterDataStorage(
  dbAdapterPromise,
  rtcAdapterFactory,
  appContext,
)
const model = new Model(appContext, dataStorage)

// Execute commands
const result = await model.commands(authContext).createUser({
  name: "Jan",
  email: "jan@example.com",
})
```
