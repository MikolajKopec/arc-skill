# Views & Queries — Deep Dive

## Views (Materialized Projections)

Views are read-optimized projections built from events. They implement the "read side" of CQRS.

### Definition

```typescript
import { view, id, object, number, string, date } from "@arcote.tech/arc"

const userProfile = view(
  "userProfile",                           // name (also table name)
  id("UserId"),                            // ID type for this view's records
  object({                                 // record schema
    name: string(),
    email: string(),
    postCount: number(),
    lastActive: date(),
  })
)
  .use([userCreatedEvent, postCreatedEvent, userLoggedInEvent])  // events to handle
  .version(1)                              // schema version (for migrations)
  .description("User profile view")        // human-readable description
  .async()                                 // handle events asynchronously
  .auth(authContext => ({                  // row-level security (WhereCondition)
    userId: authContext.userId,              // filters all queries to user's own data
  }))
  .handleEvent(userCreatedEvent, async (ctx, event) => {
    await ctx.set(event.payload.userId, {
      name: event.payload.name,
      email: event.payload.email,
      postCount: 0,
      lastActive: new Date(),
    })
  })
  .handleEvent(postCreatedEvent, async (ctx, event) => {
    const user = await ctx.findOne({ _id: event.payload.userId })
    if (user) {
      await ctx.modify(user._id, { postCount: user.postCount + 1 })
    }
  })
  .handleEvent(userLoggedInEvent, async (ctx, event) => {
    await ctx.modify(event.payload.userId, { lastActive: new Date() })
  })
```

### Builder Methods

| Method | Description |
|--------|-------------|
| `.use(events[])` | Events this view handles |
| `.version(n)` | Schema version for migration tracking |
| `.description(text)` | Human-readable description |
| `.async()` | Handle events asynchronously (default: sync) |
| `.auth(fn)` | Row-level authorization restrictions |
| `.handle(handler)` | Generic handler (alternative to handleEvent) |
| `.handleEvent(event, handler)` | Register event-specific handler |

### Handler Context (ctx)

Inside `.handleEvent()`, the `ctx` provides:

```typescript
{
  // Write operations
  set(id: string, data: Schema): Promise<void>       // Create/replace record
  modify(id: string, changes: Partial<Schema>): Promise<void>  // Partial update
  remove(id: string): Promise<void>                   // Soft delete

  // Read operations
  find(options: FindOptions<Schema>): Promise<Schema[]>
  findOne(where: WhereCondition<Schema>): Promise<Schema | undefined>

  // Auth
  $auth: AuthContext
}
```

### Database Table

Each view gets its own table:

```sql
CREATE TABLE IF NOT EXISTS userProfile (
  _id TEXT PRIMARY KEY,      -- ID from view definition
  name TEXT,
  email TEXT,
  postCount REAL,
  lastActive REAL,           -- dates stored as timestamps
  deleted INTEGER DEFAULT 0,  -- soft delete
  _v INTEGER DEFAULT 0        -- version counter
);
```

### Event Replay

Views can be reinitialized by replaying all events:
1. Drop view table
2. Replay all events from `events` table in chronological order
3. Each event triggers matching `.handleEvent()` handler

### Authorization

View `.auth()` returns a **WhereCondition** — a filter applied to ALL queries as an additional WHERE clause:

```typescript
// User can only see their own data:
.auth(authContext => ({
  userId: authContext.userId,
}))

// Admin sees everything, regular users see own data:
.auth(authContext => {
  if (authContext.role === "admin") return {}  // no filter
  return { userId: authContext.userId }
})
```

**Note:** View auth is different from Event auth. Events use `{ read, write }` permissions (see commands-events.md). Views use WhereCondition for row-level filtering.

## Static View

A view without event handlers — just a typed table for direct CRUD:

```typescript
import { staticView } from "@arcote.tech/arc"

const settings = staticView(
  "settings",
  id("SettingId"),
  object({ key: string(), value: any() })
)
```

## Queries

### Query Builders

Access query builders through context:

```typescript
const qb = context.queryBuilder()

// Find multiple records
const users = await qb.userProfile.find({
  where: { postCount: { $gte: 10 } },
  orderBy: { lastActive: "desc" },
  limit: 20,
  offset: 0,
})

// Find single record
const user = await qb.userProfile.findOne({ _id: someUserId })
```

### Where Conditions

```typescript
type WhereCondition<T> = {
  [K in keyof T]?: T[K] | WhereOperators<T[K]>
}

type WhereOperators<T> = {
  $eq?: T          // equals
  $ne?: T          // not equals
  $gt?: T          // greater than
  $gte?: T         // greater than or equal
  $lt?: T          // less than
  $lte?: T         // less than or equal
  $in?: T[]        // in array
  $nin?: T[]       // not in array
  $exists?: boolean // field exists (not null/undefined)
}
```

### FindOptions

```typescript
type FindOptions<T> = {
  where?: WhereCondition<T>
  limit?: number
  offset?: number
  orderBy?: { [key: string]: "asc" | "desc" }
}
```

### Example Queries

```typescript
// All active users, sorted by name
qb.userProfile.find({
  where: { status: "active" },
  orderBy: { name: "asc" },
})

// Users with 10+ posts, paginated
qb.userProfile.find({
  where: { postCount: { $gte: 10 } },
  limit: 20,
  offset: 40,  // page 3
})

// Find by multiple conditions
qb.userProfile.find({
  where: {
    status: { $in: ["active", "verified"] },
    lastActive: { $gte: new Date("2024-01-01").getTime() },
    postCount: { $gt: 0 },
  },
})

// Find one by ID
qb.userProfile.findOne({ _id: "usr_123" })

// Find one by unique field
qb.userProfile.findOne({ email: "jan@example.com" })
```

## Executing Queries via Model

### Local Model

```typescript
const model = new Model(appContext, dataStorage)

// Single query
const users = await model.query(q => q.userProfile.find({}))

// Subscribe to changes (reactive)
const { result, unsubscribe } = model.subscribe(
  q => q.userProfile.find({ where: { status: "active" } }),
  (data) => {
    console.log("Data updated:", data)
  },
  authContext,
)
```

### Remote Model (Client)

```typescript
const model = new RemoteModelClient(appContext, "http://localhost:5005", "web", onError)

// Query via HTTP POST /query
const users = await model.query(q => q.userProfile.find({}))
```

### React Hooks

```typescript
// useQuery returns [data, loading]
// Args: queryFn, dependencies[] (default []), cacheKey? (optional)
const [users, loading] = useQuery(
  q => q.userProfile.find({ where: { status: "active" } }),
  [],                    // dependencies array
  "active-users",        // optional cache key for revalidation
)

// Revalidate specific cache key
const revalidate = useRevalidate()
await revalidate("active-users")
```

## View Records

Each record in a view has implicit fields:

```typescript
{
  _id: string          // Primary key (from view's ID type)
  // ... user-defined fields ...
  lastUpdate: string   // ISO timestamp, auto-set on set() and modify()
  deleted: boolean     // Soft delete flag (filtered by default)
  _v: number           // Version counter (managed via version tracking tables)
}
```
