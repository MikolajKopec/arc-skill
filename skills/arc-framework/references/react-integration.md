# React Integration — Deep Dive

## Package: `@arcote.tech/arc-react`

Dependencies: React 18+, `@arcote.tech/arc` (core)

## reactModel Factory

```typescript
import { reactModel } from "@arcote.tech/arc-react"

// Takes context instance + options (NOT generic-only!)
// Returns an ARRAY (use array destructuring, NOT object destructuring)
const [
  LiveModelProvider,     // [0] Main provider component
  LocalModelProvider,    // [1] Local model provider
  useQuery,              // [2] Query hook
  useRevalidate,         // [3] Cache revalidation hook
  useCommands,           // [4] Commands hook
  query,                 // [5] Imperative query function
  useLocalModel,         // [6] Access local model
  useModel,              // [7] Access parent model
] = reactModel(appContext, { remoteUrl: "http://localhost:5005" })

// Alternative: local SQLite mode
const [...hooks] = reactModel(appContext, { sqliteAdapter: sqliteDb })
```

**Options (union type):**
- `{ remoteUrl: string }` — connects to ARC host server via HTTP
- `{ sqliteAdapter: SQLiteDatabase }` — local SQLite via WASM adapter

## Providers

### LiveModelProvider

```tsx
<LiveModelProvider
  client="web"
  token={jwtToken}
  catchErrorCallback={(error) => console.error(error)}
  syncView={SyncProgress}  // optional loading component
>
  <App />
</LiveModelProvider>
```

**Props:**

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `children` | `ReactNode` | Yes | Child components |
| `client` | `string` | Yes | Client identifier |
| `token` | `string` | Yes | JWT auth token (NOT `authToken`) |
| `catchErrorCallback` | `(error: any) => void` | Yes | Error callback (NOT `onError`) |
| `syncView` | `React.ComponentType<{ progress }>` | No | Component shown during sync |

**Note:** `context` and `apiBaseUrl` are NOT props — they're captured in the closure from `reactModel()`.

**Sync progress:** When `syncView` is provided, it receives `progress: { store: string; size: number }[]` showing sync state per store. The provider renders `null` until sync completes if no `syncView` is given.

**Internally creates:**
- `RemoteModelClient` (when using `remoteUrl`) or `Model` with `MasterDataStorage` (when using `sqliteAdapter`)
- Singleton model — same instance reused across re-renders

### LocalModelProvider

Wraps the parent LiveModelProvider's model for local access:

```tsx
<LiveModelProvider ...>
  <LocalModelProvider>
    <LocalFeature />
  </LocalModelProvider>
</LiveModelProvider>
```

Must be nested inside `LiveModelProvider`. Throws if used outside or if parent model is not a local `Model` instance.

## Hooks

### useQuery

```typescript
function useQuery<Q extends IArcQueryBuilder>(
  queryBuilderFn: (qb: QueryBuilder) => Q,
  dependencies?: any[],     // default: []
  cacheKey?: string,         // optional, for revalidation
): [data, loading]
// Returns: [T, false] when loaded, [undefined, true] when loading
```

**Example:**
```typescript
// Basic query
const [users, loading] = useQuery(q => q.userProfile.find({}))

// With dependencies (re-runs when deps change)
const [user, loading] = useQuery(
  q => q.userProfile.findOne({ _id: userId }),
  [userId],
)

// With cache key (for manual revalidation)
const [users, loading] = useQuery(
  q => q.userProfile.find({ where: { status: "active" } }),
  [],
  "active-users",
)
```

**Behavior:**
1. Subscribes to model on mount via `model.subscribe()`
2. Callback updates state when data changes
3. Re-subscribes when `dependencies` or `revalidationTrigger` changes
4. Unsubscribes on unmount
5. Uses default auth context: `{ userId: "react-user" }`

**Important:** The second argument is `dependencies: any[]` (array), NOT an options object. The third argument is `cacheKey?: string` (positional), NOT inside options.

### useRevalidate

```typescript
const revalidate = useRevalidate()

// Revalidate all useQuery hooks with matching cacheKey
await revalidate("active-users")  // returns Promise<void>
```

**Mechanism:**
- Each `useQuery` with a `cacheKey` registers a revalidation function in a `__cacheRegistry` Map on the model
- `revalidate(key)` triggers all registered revalidators for that key
- Returns a Promise that resolves when all queries have completed re-fetching
- Components re-render with fresh data

### useCommands

```typescript
const commands = useCommands()

// Execute a command (type-safe based on context)
const result = await commands.createUser({
  name: "Jan",
  email: "jan@example.com",
})
```

**Note:** Uses default auth context `{ userId: "react-user" }`. For proper auth, the token passed to `LiveModelProvider` handles server-side auth via JWT.

### useModel

```typescript
const model = useModel(token)
```

**Note:** Accepts a `token: string` parameter in its signature, but the token is currently unused — the hook simply returns the parent model from context. Throws if used outside provider.

### useLocalModel

```typescript
const localModel = useLocalModel()
// Returns Model instance or null if not in LocalModelProvider
```

### query (imperative)

```typescript
// Execute query outside of React components
const users = await query(
  q => q.userProfile.find({}),
  optionalModelOverride,  // optional Model instance
)
```

Uses the singleton model created by `reactModel()`. Throws if model hasn't been initialized yet.

## Query Cache System

### How It Works

```
LiveModelProvider
└── Model instance (singleton)
    └── __cacheRegistry: Map<cacheKey, Set<revalidateFn>>
        ├── useQuery("active-users") registers → Set<revalidateFn>
        ├── useQuery("user-detail")  registers → Set<revalidateFn>
        └── useRevalidate()("active-users") → calls all fns in Set
```

### Deduplication

```
Component A: useQuery(q => q.users.find({}), [], "users")
Component B: useQuery(q => q.users.find({}), [], "users")

→ Both register in same cacheKey Set
→ revalidate("users") updates BOTH components
→ Each still makes its own subscription (no request deduplication at this level)
```

## Form System

### Form Component

Schema-driven forms with ARC validation:

```tsx
import { Form, FormField, FormMessage, FormPart, useForm, useFormField, useFormPart } from "@arcote.tech/arc-react"

const schema = object({
  name: string().minLength(1).maxLength(100),
  email: string().email(),
  age: number().min(0).optional(),
})

// Form uses a render prop that receives Fields (auto-generated from schema)
<Form
  schema={schema}
  onSubmit={async (data) => {
    // data is validated and typed
    await commands.createUser(data)
  }}
  onUnvalidatedSubmit={(values, errors) => {
    // optional: called when validation fails
    console.log("Validation failed:", errors)
  }}
  defaults={{ name: "Default Name" }}  // optional initial values
  render={(Fields, values) => (
    <>
      <Fields.name
        translations={{ minLength: "Name is too short" }}
        render={({ value, onChange, name, setFieldValue }) => (
          <div>
            <input value={value} onChange={e => onChange(e.target.value)} />
            <FormMessage />
          </div>
        )}
      />

      <Fields.email
        translations={{ email: "Invalid email" }}
        render={({ value, onChange }) => (
          <div>
            <input type="email" value={value} onChange={e => onChange(e.target.value)} />
            <FormMessage />
          </div>
        )}
      />

      <button type="submit">Submit</button>
    </>
  )}
/>
```

**Form Props:**

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `schema` | `ArcObject` | Yes | Validation schema |
| `render` | `(Fields, values) => ReactNode` | Yes | Render function receiving auto-generated field components |
| `onSubmit` | `(data) => void \| Promise<void>` | Yes | Called with validated data |
| `onUnvalidatedSubmit` | `(values, errors) => void` | No | Called when validation fails |
| `defaults` | `Partial<Schema> \| null` | No | Initial field values |

### FormField Factory

```typescript
// FormField is a FUNCTION FACTORY (auto-created by Form's render prop)
// Can also be used standalone:
const Field = FormField<ArcElement>(
  name: string,           // field name (matches schema key)
  defaultValue?: any,     // optional default
  subFields?: any,        // for nested objects
)

// Returns a component that accepts:
interface FormFieldProps {
  translations: Record<string, string>  // validator name → user message
  render: (field: FormFieldData) => ReactNode
}

// FormFieldData passed to render callback:
interface FormFieldData {
  value: any                                   // current field value
  onChange: (value: any) => void               // update field value
  name: string                                // field name
  defaultValue?: any                          // default value
  subFields?: any                             // nested field components (for object fields)
  setFieldValue: (field: string, value: any) => void  // set any field's value
}

// NOTE: errors and messages are NOT in FormFieldData.
// Access them via useFormField() hook or FormMessage component.
```

### FormPart (Partial Validation)

Wraps a subset of fields for partial validation. Fields auto-register with their parent FormPart.

```tsx
<FormPart>
  <Fields.name ... />
  <Fields.email ... />
</FormPart>

// Access FormPart context via hook:
function StepButton() {
  const { validatePart } = useFormPart()
  return <button onClick={() => validatePart()}>Next Step</button>
}
```

**FormPart props:** Only `children: ReactNode`. No `fields` prop — fields register automatically when rendered inside FormPart.

**useFormPart() hook** returns `{ registerField, unregisterField, validatePart }`.

### useForm Hook

Access form context from within Form:

```tsx
const form = useForm()
// form.values, form.errors, form.dirty, form.isSubmitted
// form.setFieldValue(name, value), form.setFieldDirty(name)
// form.validatePartial(fieldNames[])
```

### useFormField Hook

Access field-level errors from within a FormField's render tree:

```tsx
const field = useFormField()
// field.errors — validation error object (or false if valid)
// field.messages — translated error message strings
// Must be used within a FormField component
```

### FormMessage

Displays the first validation error for the current field:

```tsx
<FormMessage />
// Renders the first translated error message
// Must be used within FormField
```

## Auth Token Management

```tsx
// Token passed to LiveModelProvider
<LiveModelProvider token={token} ...>

// Internally:
// - RemoteModelClient: token sent as Bearer header in all HTTP requests
// - model.setAuthToken(token) called when token prop changes
// - For local mode: token passed to RTC client factory
```

## Complete Example

```tsx
import { reactModel } from "@arcote.tech/arc-react"
import { appContext } from "./context"

const [
  LiveModelProvider,
  LocalModelProvider,
  useQuery,
  useRevalidate,
  useCommands,
  query,
  useLocalModel,
  useModel,
] = reactModel(appContext, { remoteUrl: "http://localhost:5005" })

function App() {
  return (
    <LiveModelProvider
      client="web"
      token="your-jwt-token"
      catchErrorCallback={console.error}
    >
      <UserList />
      <CreateUserForm />
    </LiveModelProvider>
  )
}

function UserList() {
  const [users, loading] = useQuery(
    q => q.userProfile.find({ orderBy: { name: "asc" } }),
    [],
    "users",
  )

  if (loading) return <div>Loading...</div>
  return (
    <ul>
      {users?.map(user => (
        <li key={user._id}>{user.name} — {user.email}</li>
      ))}
    </ul>
  )
}

function CreateUserForm() {
  const commands = useCommands()
  const revalidate = useRevalidate()

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault()
    const form = new FormData(e.target as HTMLFormElement)
    await commands.createUser({
      name: form.get("name") as string,
      email: form.get("email") as string,
    })
    await revalidate("users")
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Create User</button>
    </form>
  )
}
```
