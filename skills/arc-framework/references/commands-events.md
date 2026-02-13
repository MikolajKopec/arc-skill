# Commands, Events, Listeners & Translator — Deep Dive

## Commands

### Definition

```typescript
import { command, object, string, id } from "@arcote.tech/arc"

const createUser = command("createUser")
  .use([userCreatedEvent, userView])     // elements available in handler
  .withParams({                           // raw shape or ArcObject
    name: string().minLength(1),
    email: string().email(),
  })
  .withResult({ id: id("UserId") })      // spread args for union results
  .description("Creates a new user")     // for documentation/API
  .public()                               // exposed to clients
  .handle(async (ctx, params) => {
    const newId = userId.generate()
    await ctx.get(userCreatedEvent).emit({
      userId: newId,
      name: params.name,
      email: params.email,
    })
    return { id: newId }
  })
```

### Builder Methods

| Method | Description |
|--------|-------------|
| `.use(elements[])` | Context elements available in handler |
| `.withParams(rawShape \| ArcObject)` | Input validation schema |
| `.withResult(...rawShapes)` | Output schema(s) — spread args, each wrapped in `object()` |
| `.description(text)` | Human-readable description |
| `.public()` | Mark as publicly accessible from client |
| `.handle(fn \| false)` | Implementation function, or `false` for abstract/placeholder |

### Handler Context

The `ctx` parameter in `.handle()` is a Proxy providing:

```typescript
// Access any element registered via .use()
ctx.get(elementInstance)  // returns element's commandContext

// Auth context
ctx.$auth  // { userId?: string, ipAddress?: string, [key: string]: any }
```

Each element type exposes different methods in commandContext:
- **Event elements:** `.emit(payload)` — emit an event
- **View elements:** `.find()`, `.findOne()`, `.set()`, `.modify()`, `.remove()`

### Execution Flow (Model.commands())

```
1. client calls model.commands(authContext).createUser(params)
2. params validated against .withParams() schema
3. DataStorage forked (transaction isolation)
4. handler(ctx, params) executed on fork
5. sync listeners run (same fork — part of transaction)
6. fork merged to master → DB updated
7. async listeners run (separate forks, parallel)
8. result returned to client
```

### matchesCommandPath

```typescript
createUser.matchesCommandPath("/command/createUser")
// Returns: { matches: true, isPublic: true }
// NOT a boolean — returns an object with matches and isPublic fields
```

### JSON Schema

```typescript
const schema = createUser.toJsonSchema()
// {
//   type: "function",
//   name: "createUser",
//   description: "Creates a new user",
//   parameters: { type: "object", properties: { ... } },
//   strict: true,
// }
```

## Events

### Definition

```typescript
import { event, object, string, id } from "@arcote.tech/arc"

// Can pass ArcObject or raw shape
const userCreated = event("userCreated", object({
  userId: id("UserId"),
  name: string(),
  email: string(),
}))

// Or with raw shape (auto-wrapped in object())
const userCreated = event("userCreated", {
  userId: id("UserId"),
  name: string(),
  email: string(),
})
```

### Authorization

```typescript
const userCreated = event("userCreated", userPayloadSchema)
  .auth(authContext => ({
    read: true,                              // anyone can read
    write: authContext.role === "admin",       // only admins can emit
  }))
```

**Default auth:** `{ read: true, write: true, modify: false, delete: false }`
Events are immutable — `.auth()` can only control `read` and `write`; `modify` and `delete` are **always forced to `false`** regardless of custom restrictions.

### Storage

All events stored in a **single shared table** with this schema:

```typescript
// Primary key is "id", NOT "_id"
const eventSchema = object({
  id: string().primaryKey(),  // NOT _id — events use "id"
  type: string(),             // event name (e.g. "userCreated")
  payload: any(),             // JSON payload
  createdAt: date(),          // Date timestamp
})

// Table options
{ softDelete: false, versioning: true }
```

**Important:** Events use `id` as primary key, not `_id` like views. The table has `softDelete: false` — events are never soft-deleted.

### Event Instance Type

```typescript
type ArcEventInstance<E> = {
  type: E["name"]                    // "userCreated"
  payload: $type<E["payload"]>       // { userId: UserId, name: string, ... }
  createdAt: Date
}
```

### Emitting Events

Events are emitted inside command handlers:

```typescript
.handle(async (ctx, params) => {
  // ctx.get(event) returns { emit: async (payload) => void }
  await ctx.get(userCreatedEvent).emit({
    userId: newId,
    name: params.name,
    email: params.email,
  })
})
```

The `.emit()` method:
1. Generates an ID via `id("event").generate()`
2. Stores `{ id, type, payload, createdAt }` in the events table (on forked DataStorage)
3. Calls `publishEvent()` → triggers listeners

## EventPublisher

### Architecture

```typescript
class EventPublisher<C extends ArcContextAny> {
  private asyncEvents: Array<{
    event: any
    listeners: Function[]
    authContext: AuthContext
  }> = []

  async publishEvent(event, commandDataStorage): Promise<void>
  async runAsyncListeners(): Promise<void>
}
```

### Flow

```
publishEvent(event):
  1. Get sync listeners for event type from context
  2. Run sync listeners sequentially (await each)
  3. If any sync listener throws → command fails (rollback)
  4. Collect async listeners for later

runAsyncListeners():
  1. For each queued async event:
     a. Fork DataStorage (separate from command fork)
     b. Create child EventPublisher (for nested events)
     c. Run listener handler
     d. Merge fork to master
     e. Recursively run child publisher's async listeners
  2. All async listeners run in parallel
  3. Errors logged but don't break the original command
```

**Key:** Sync listeners participate in the command's transaction — if they fail, the entire command rolls back. Async listeners run independently after the command succeeds.

## Listeners

### Definition

```typescript
import { listener } from "@arcote.tech/arc"

const onUserCreated = listener("onUserCreated")
  .use([sendEmailCommand])           // context elements
  .listenTo([userCreatedEvent])       // events to react to
  .async()                            // run after commit (default: sync)
  .description("Send welcome email")
  .handle(async (ctx, event) => {
    // event = { type: "userCreated", payload: {...}, createdAt: Date }
    // Commands are callable functions, NOT objects with .execute()
    await ctx.get(sendEmailCommand)({
      to: event.payload.email,
      template: "welcome",
    })
  })
```

### Builder Methods

| Method | Description |
|--------|-------------|
| `.use(elements[])` | Context elements available in handler |
| `.listenTo(events[])` | Events this listener reacts to |
| `.async()` | Run asynchronously (after commit) |
| `.description(text)` | Human-readable description |
| `.handle(fn)` | Implementation function |

### Sync vs Async

| Aspect | Sync Listener | Async Listener |
|--------|--------------|----------------|
| Runs when | During command execution | After command succeeds |
| Transaction | Same fork as command | Separate fork |
| Failure impact | Rolls back entire command | Logged, no rollback |
| Use case | Validation, derived data | Notifications, side effects |

### Handler Signature

```typescript
(ctx: ArcCommandContext, event: ArcEventInstance) => Promise<void> | void
```

The `ctx` provides same proxy as commands — access to elements via `.get()`.

## Translator

Syntactic sugar for event-to-event mapping:

```typescript
import { translate } from "@arcote.tech/arc"

const userToWelcome = translate(
  userCreatedEvent,      // source event
  welcomeEmailEvent,     // target event
  (event) => ({          // transformation
    email: event.payload.email,
    template: "welcome",
    name: event.payload.name,
  })
)
```

Creates a listener named `${from.name}To${to.name}Translator`.

> **Known Issue:** The current implementation (`translator.ts:24`) calls `ctx.get(from).emit(translation(event))` — emitting on the SOURCE event instead of the TARGET event. This appears to be a bug. When using translators, verify the behavior and consider implementing the listener manually if needed.

**Under the hood:**
```typescript
listener(`userCreatedTowelcomeEmailTranslator`)
  .listenTo([userCreatedEvent])
  .use([welcomeEmailEvent])
  .handle((ctx, event) => {
    // BUG: Uses ctx.get(from) instead of ctx.get(to)
    ctx.get(userCreatedEvent).emit(translation(event))
  })
```

## Complete Example

```typescript
// 1. Define event
const orderPlaced = event("orderPlaced", {
  orderId: id("OrderId"),
  userId: id("UserId"),
  total: number(),
  items: array(object({ productId: id("ProductId"), qty: number() })),
})

// 2. Define command
const placeOrder = command("placeOrder")
  .use([orderPlaced])
  .withParams({
    items: array(object({ productId: id("ProductId"), qty: number() })),
  })
  .withResult({ orderId: id("OrderId") })
  .public()
  .handle(async (ctx, params) => {
    const orderId = id("OrderId").generate()
    const total = await calculateTotal(params.items)
    await ctx.get(orderPlaced).emit({
      orderId, userId: ctx.$auth.userId, total, items: params.items,
    })
    return { orderId }
  })

// 3. Sync listener — validate stock (rolls back on failure)
const validateStock = listener("validateStock")
  .use([inventoryView])
  .listenTo([orderPlaced])
  // sync by default
  .handle(async (ctx, event) => {
    for (const item of event.payload.items) {
      const stock = await ctx.get(inventoryView).findOne({ productId: item.productId })
      if (!stock || stock.quantity < item.qty) {
        throw new Error(`Insufficient stock for ${item.productId}`)
      }
    }
  })

// 4. Async listener — send confirmation (doesn't rollback)
const sendConfirmation = listener("sendOrderConfirmation")
  .use([emailService])
  .listenTo([orderPlaced])
  .async()
  .handle(async (ctx, event) => {
    await ctx.get(emailService).send({
      userId: event.payload.userId,
      subject: "Order confirmed",
      orderId: event.payload.orderId,
    })
  })

// 5. Register all in context
const orderContext = context([
  orderPlaced,
  placeOrder,
  validateStock,
  sendConfirmation,
])
```
