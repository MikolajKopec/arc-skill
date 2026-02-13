# Host Server (Bun) — Deep Dive

## Package: `@arcote.tech/arc-host`

The host package provides a Bun-native HTTP + WebSocket server that exposes your ARC context as an API.

## Setup

```typescript
import { hostLiveModel } from "@arcote.tech/arc-host"
import { appContext } from "./context"

// SQLite (default, Bun native)
hostLiveModel(appContext, { db: "app.db", version: 1 })
// Creates SQLite adapter internally, starts server on port 5005
```

**Function signature:**
```typescript
function hostLiveModel<C extends ArcContextAny>(
  context: C,
  options: { db: string; version: number },
): void
```

**Internally:**
1. Creates `sqliteAdapterFactory(options.db)` for database
2. Creates `RTCHost` via `rtcHostFactory(context, dbAdapter)`
3. `RTCHost` sets up `MasterDataStorage` + `Model` + Bun HTTP/WebSocket server

### PostgreSQL

```typescript
import { postgreSQLAdapterFactory } from "@arcote.tech/arc-host"
// Use postgreSQLAdapterFactory for PostgreSQL setups
// Requires manual RTCHost configuration
```

## Server Architecture

The server is an `RTCHost` class internally:

```typescript
class RTCHost implements RealTimeCommunicationAdapter {
  private server: Server           // Bun.serve instance
  private dataStore: MasterDataStorage
  private model: Model<ArcContextAny>

  constructor(context, dbAdapter: Promise<DatabaseAdapter>)
}
```

## Endpoints

### POST `/command/:commandName`

Execute a command.

**JSON request:**
```http
POST /command/createUser
Content-Type: application/json
Authorization: Bearer <jwt-token>

{
  "name": "Jan",
  "email": "jan@example.com"
}
```

**FormData request (for blob/file params):**
```http
POST /command/uploadAvatar
Content-Type: multipart/form-data
Authorization: Bearer <jwt-token>

--boundary
Content-Disposition: form-data; name="userId"
usr_123
--boundary
Content-Disposition: form-data; name="avatar"; filename="avatar.png"
Content-Type: image/png
<binary data>
--boundary--
```

**Response (200):**
```json
{ "id": "usr_65a1b2c3xxxxxxxxxxxxxxxx" }
```

**Auto-detection:** Content-Type `multipart/form-data` triggers FormData parsing with nested key support (`user[profile][avatar]`).

### POST `/query`

Execute a query.

```http
POST /query
Content-Type: application/json
Authorization: Bearer <jwt-token>

{
  "query": {
    "element": "userProfile",
    "queryType": "find",
    "params": [{ "where": { "status": "active" }, "limit": 20 }]
  }
}
```

**Request format:**
```typescript
{
  query: {
    element: string      // element/view name
    queryType: string    // "find" or "findOne"
    params: any[]        // arguments to the query method
  }
}
```

**Note:** Token is REQUIRED for queries — returns 401 without it.

### GET `/sync`

Initial data sync for a client. Accepts optional `lastSync` query parameter.

```http
GET /sync?lastSync=2024-01-01T00:00:00.000Z
Authorization: Bearer <jwt-token>
```

**Response:**
```json
{
  "results": [
    { "store": "userProfile", "items": [...] }
  ],
  "syncDate": "2024-06-15T12:00:00.000Z"
}
```

> **Note:** The `/sync` endpoint is partially implemented — the store iteration logic is commented out in the current codebase.

### ANY `/route/*`

Custom route handlers defined with `route()`:

```http
GET /route/api/users/usr_123
Authorization: Bearer <jwt-token>
```

Matched against `.matchesRoutePath()` which returns `{ matches: boolean, params: Record<string, string>, isPublic: boolean }`. Supports path parameters (`:id` → `params.id`).

Methods: GET, POST, PUT, DELETE, PATCH. Returns 404 if no route matches, 405 if method not supported.

### WebSocket `/ws`

Real-time sync channel.

**Upgrade:**
```http
GET /ws
Upgrade: websocket
Authorization: Bearer <jwt-token>  (or ?token=... query param)
```

**Client → Host messages:**
```typescript
// Request changes to be applied
{ type: "changes-executed", changes: DataStorageChanges[] }

// Request sync
{ type: "sync", lastDate: string | null }
```

**Host → Client messages:**
```typescript
// Broadcast state changes to all connected clients
{ type: "state-changes", changes: DataStorageChanges[] }

// Sync results (per store)
{ type: "sync-result", store: string, items: any[] }

// Sync complete
{ type: "sync-done", date: string }
```

**Behavior:**
- Clients subscribe to `"sync"` channel on connection
- `changes-executed` messages: host applies changes to DataStorage, then broadcasts `state-changes` to all connected clients (via `ws.publish("sync", ...)`)
- **Note:** The `"sync"` message type is defined in the protocol types but NOT currently handled by the host's WebSocket `onMessage()` handler — only `"changes-executed"` is processed
- Uses `perMessageDeflate: true` and 16MB backpressure limit

## Authentication

### JWT

Uses `jsonwebtoken` library.

```typescript
// Environment variables
JWT_SECRET=your-secret-key         // REQUIRED in production
JWT_EXPIRES_IN=7d                  // Default: "7d"

// Default (development fallback — NOT safe for production!)
// If JWT_SECRET not set, uses hardcoded fallback secret
```

**Token payload interface:**
```typescript
interface TokenPayload {
  userId: string;    // required
  email: string;     // required
  iat?: number;      // issued at (auto-set by jwt.sign)
  exp?: number;      // expiration (auto-set by jwt.sign)
}
```

**Token flow:**
1. Client sends `Authorization: Bearer <token>` header (or `?token=` query param for WebSocket)
2. Host verifies JWT signature via `verifyToken(token)`
3. Decoded payload mapped to `AuthContext`: `{ userId, ipAddress }`
4. Auth restrictions applied to Views/Events based on `AuthContext`
5. Public endpoints (`.public()` on command/route) skip token verification

### IP Detection

Priority order:
1. `x-forwarded-for` header (first IP in comma-separated list)
2. `x-real-ip` header
3. `cf-connecting-ip` header (Cloudflare)
4. `undefined` (no IP available)

Available in auth context as `authContext.ipAddress`.

### Public Endpoints

The server checks if an endpoint is public by iterating context elements:

```typescript
// For commands: element.matchesCommandPath(pathname) → { matches, isPublic }
// For routes: element.matchesRoutePath(pathname) → { matches, isPublic }
// Public endpoints don't require auth token
```

## Request Processing

### Command Flow

```
1. POST /command/createUser
2. Check if public endpoint (skip auth if yes)
3. Parse Authorization header → verify JWT → authContext
4. Parse body: JSON (default) or FormData (if multipart/form-data)
5. Get commands proxy: model.commands(authContext)
6. Execute: commands[commandName](parsedBody)
   a. Fork DataStorage
   b. Run handler
   c. Run sync listeners
   d. Merge fork
   e. Run async listeners
7. Return JSON result (200) or error (500)
```

### Query Flow

```
1. POST /query
2. Parse Authorization header → verify JWT (REQUIRED)
3. Parse body: { query: { element, queryType, params } }
4. Build query: context.queryBuilder()[element][queryType](...params)
5. Execute: queryObj.run(dataStorage)
6. Return JSON result (200) or error (500)
```

### Route Flow

```
1. ANY /route/api/users/:id
2. Iterate context elements, find matching route via matchesRoutePath()
3. Get handler for HTTP method (GET/POST/PUT/DELETE/PATCH)
4. Verify JWT (skip if route is .public())
5. Execute via model.routes(authContext)[routeName](method, req, params, url)
6. Return Response from handler
```

## Port

Currently hardcoded to **5005** (`host.ts:407`). To change, modify the host source or use a reverse proxy.

## Additional Settings

```typescript
// Bun.serve options set by RTCHost:
{
  port: 5005,
  idleTimeout: 30,                    // seconds
  websocket: {
    perMessageDeflate: true,
    backpressureLimit: 16 * 1024 * 1024,  // 16MB
  }
}
```

## Complete Server Setup

```typescript
// server.ts
import { hostLiveModel } from "@arcote.tech/arc-host"
import { context } from "@arcote.tech/arc"

// Import all domain elements
import { userCreatedEvent } from "./events/user"
import { createUserCommand } from "./commands/user"
import { userProfileView } from "./views/user"
import { onUserCreatedListener } from "./listeners/user"
import { usersRoute } from "./routes/users"

// Compose context
const appContext = context([
  userCreatedEvent,
  createUserCommand,
  userProfileView,
  onUserCreatedListener,
  usersRoute,
])

// Start server with SQLite
hostLiveModel(appContext, { db: "app.db", version: 1 })

console.log("ARC server running on http://localhost:5005")
```
