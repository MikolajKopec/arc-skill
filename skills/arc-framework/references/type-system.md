# ARC Type System — Complete Reference

ARC has its own schema/validation system in `packages/core/elements/`. Every type extends `ArcAbstract` which implements the `ArcElement` interface.

## Base Interface

```typescript
interface ArcElement {
  serialize(value: any): any       // Convert to storage format
  deserialize(value: any): any     // Convert from storage format
  parse(value: any): any           // Validate + deserialize
  validate(value: any): any        // Returns validation error object or false
  toJsonSchema(): object           // JSON Schema Draft-07
}
```

**Note:** `validate()` returns `false` if valid, or an error object (not string array) describing the validation failure per validator.

### Common Methods on All Elements

Every element inherits from `ArcAbstract` and has these methods:

```typescript
element.optional()                   // Wraps in ArcOptional (allows null/undefined)
element.branded("BrandName")         // Wraps in ArcBranded (nominal typing)
element.default(valueOrCallback)     // Wraps in ArcDefault (provides fallback value)
element.description("text")          // Set description for docs/JSON Schema

// Examples:
const nickname = string().optional()                    // string | null | undefined
const email = string().email().branded("Email")         // branded type
const count = number().default(0)                       // default value
const createdAt = date().default(() => new Date())      // default via callback
```

## Primitive Types

### ArcString

```typescript
import { string } from "@arcote.tech/arc"

const name = string()
  .minLength(1)         // min character length
  .maxLength(255)       // max character length
  .length(10)           // exact length
  .includes("@")        // must contain substring
  .startsWith("usr_")   // prefix check
  .endsWith(".com")     // suffix check
  .regex(/^[a-z]+$/)    // regex pattern
  .email()              // email format
  .url()                // URL format
  .ip()                 // IP address format
```

**DB column type:** `TEXT`

### ArcNumber

```typescript
import { number } from "@arcote.tech/arc"

const age = number()
  .min(0)     // minimum value (inclusive)
  .max(150)   // maximum value (inclusive)
```

**DB column type:** `REAL` / `NUMERIC`

### ArcBoolean

```typescript
import { boolean } from "@arcote.tech/arc"

const active = boolean()
const mustAgree = boolean().hasToBeTrue()  // must be truthy
```

**DB column type:** `INTEGER` (SQLite 0/1) / `BOOLEAN` (PostgreSQL)

### ArcDate

```typescript
import { date } from "@arcote.tech/arc"

const created = date()
  .after(new Date("2020-01-01"))    // must be after date
  .before(new Date("2030-01-01"))   // must be before date
```

**Serialization:** `Date` → timestamp number (`.getTime()`). **Deserialization:** `new Date(value)` — accepts timestamp numbers or ISO date strings.
**DB column type:** `REAL` (timestamp)

## ID Types

### ArcId

```typescript
import { id } from "@arcote.tech/arc"

const userId = id("UserId")

// Generate new ID
const newId = userId.generate()
// Format: variable-length hex timestamp (typically 8+ chars) + 16 random hex chars
// Example: "65a1b2c3xxxxxxxxxxxxxxxx" (NO underscore separator)

// Custom generator function
const customUserId = id("UserId", () => crypto.randomUUID())
```

**Branded type:** `string & { __brand: "UserId" }` — prevents mixing IDs from different domains.
**DB column type:** `TEXT`

### ArcCustomId

```typescript
import { customId } from "@arcote.tech/arc"

const shortId = customId("ShortId", (prefix: string) => prefix + "_" + crypto.randomUUID().slice(0, 8))

// Generate using the custom function
const newId = shortId.get("usr")  // "usr_a1b2c3d4"
// .get() passes arguments through to the createFn
```

## Complex Types

### ArcObject

```typescript
import { object, string, number, id } from "@arcote.tech/arc"

const userSchema = object({
  id: id("UserId"),
  name: string().minLength(1),
  email: string().email(),
  age: number().min(0),
})

// Type extraction
type User = $type<typeof userSchema>
// { id: UserId; name: string; email: string; age: number }
```

#### Object Methods

```typescript
// Merge with another raw shape (NOT another ArcObject)
const extendedUser = userSchema.merge({ role: string(), dept: string() })

// Partial (all fields wrapped in ArcOptional)
const userPatch = userSchema.partial()

// Pick specific fields
const nameEmail = userSchema.pick("name", "email")

// Omit specific fields
const withoutId = userSchema.omit("id")

// Get keys (typed array)
const keys = userSchema.keys()  // ("id" | "name" | "email" | "age")[]

// Get entries (key-element pairs)
const entries = userSchema.entries()  // [string, ArcElement][]

// Partial operations (validate/parse/serialize/deserialize only provided fields)
const validated = userSchema.validatePartial({ name: "" })  // validates only name
const parsed = userSchema.parsePartial({ name: "Jan" })
const serialized = userSchema.serializePartial({ age: 25 })
const deserialized = userSchema.deserializePartial({ age: 25 })
```

**DB:** Each field becomes a column. Nested objects stored as JSON.

### ArcArray

```typescript
import { array, string, number } from "@arcote.tech/arc"

const tags = array(string())
  .minLength(1)     // at least 1 element
  .maxLength(10)    // at most 10 elements
  .length(5)        // exactly 5 elements
  .nonEmpty()       // shorthand for minLength(1)

const scores = array(number().min(0).max(100))
```

**DB column type:** `TEXT` (JSON serialized) / `JSONB` (PostgreSQL)

### ArcOptional

```typescript
// optional() is a METHOD on any ArcElement, NOT a standalone function
const nickname = string().maxLength(50).optional()
// Allows: string | null | undefined
```

**DB:** Marks column as `NULLABLE`.

### ArcOr (Union)

```typescript
import { or, string, number } from "@arcote.tech/arc"

// IMPORTANT: Uses spread args, NOT an array!
const idOrName = or(id("UserId"), string())
// Validates against each type in order, first match wins
```

### ArcStringEnum

```typescript
import { stringEnum } from "@arcote.tech/arc"

// IMPORTANT: Uses spread args, NOT an array!
const status = stringEnum("active", "inactive", "banned")
// Type: "active" | "inactive" | "banned"

// Get enum values
status.getEnumerators()  // ["active", "inactive", "banned"]
```

**DB column type:** `TEXT` (with stringEnum column type hint)

### ArcRecord

```typescript
import { record, string, number } from "@arcote.tech/arc"

const scores = record(string(), number())
// Type: Record<string, number>
// Key type must be ArcString, ArcNumber, or ArcId
```

**DB column type:** `TEXT` (JSON serialized) / `JSONB` (PostgreSQL)

### ArcBlob

```typescript
import { blob } from "@arcote.tech/arc"

const avatar = blob()
  .type("image/png")                    // exact MIME type
  .types(["image/png", "image/jpeg"])   // one of MIME types
  .minSize(1024)                        // min size in bytes
  .maxSize(5 * 1024 * 1024)             // max size in bytes
  .size(1024)                           // exact size in bytes
```

**Special handling:** When used in command params, RemoteModelClient automatically switches to FormData upload.
**DB column type:** `BLOB` / `BYTEA`

### ArcFile

```typescript
import { file } from "@arcote.tech/arc"

const document = file()
  // All blob validators:
  .type("application/pdf")
  .types(["application/pdf", "image/png"])
  .minSize(1024)
  .maxSize(10 * 1024 * 1024)
  .size(1024)
  // File-specific validators:
  .name("report.pdf")                   // exact filename
  .namePattern(/^report_\d+\.pdf$/)     // filename regex
  .extension("pdf")                     // file extension (with or without dot)
  .extensions(["pdf", "docx", "xlsx"])  // one of extensions
  .lastModifiedAfter(new Date("2024-01-01"))
  .lastModifiedBefore(new Date("2025-01-01"))
```

**DB column type:** `BLOB` / `BYTEA`

### ArcBranded

```typescript
// branded() is a METHOD on any ArcElement, NOT a standalone function
const Email = string().email().branded("Email")
// Creates a type-safe wrapper: string & { __brand: "Email" }
```

### ArcAny

```typescript
import { any } from "@arcote.tech/arc"

const metadata = any()
// Pass-through, no validation
```

**DB column type:** `TEXT` (JSON) / `JSONB`

## Type Extraction Utilities

```typescript
import { $type, $partial } from "@arcote.tech/arc"

// Extract TypeScript type from ArcElement
type User = $type<typeof userSchema>

// Extract partial (all optional) type
type UserPatch = $partial<typeof userSchema>
```

## JSON Schema Generation

Every element can produce JSON Schema Draft-07:

```typescript
const schema = userSchema.toJsonSchema()
// {
//   type: "object",
//   properties: {
//     id: { type: "string" },
//     name: { type: "string", minLength: 1 },
//     email: { type: "string", format: "email" },
//     age: { type: "number", minimum: 0 }
//   },
//   required: ["id", "name", "email", "age"]
// }
```

Optional fields are excluded from `required` array. Their schema comes from the wrapped element directly (no `anyOf` wrapper in object context).

## Database Column Mapping

| ArcElement | SQLite Type | PostgreSQL Type |
|------------|-------------|-----------------|
| string     | TEXT        | TEXT            |
| number     | REAL        | NUMERIC         |
| boolean    | INTEGER     | BOOLEAN         |
| date       | REAL        | NUMERIC (timestamp) |
| id         | TEXT        | TEXT            |
| array      | TEXT (JSON) | JSONB           |
| object     | TEXT (JSON) | JSONB           |
| record     | TEXT (JSON) | JSONB           |
| stringEnum | TEXT        | TEXT            |
| any        | TEXT (JSON) | JSONB           |
| blob       | BLOB        | BYTEA           |
| file       | BLOB        | BYTEA           |
| optional   | NULLABLE    | NULLABLE        |

## Validation

```typescript
const errors = userSchema.validate({
  id: "not-a-valid-id",
  name: "",
  email: "invalid",
  age: -5,
})
// Returns false if valid
// Returns error object if invalid: { schema: { name: { minLength: {...} }, ... } }

// Validate only some fields
const partialErrors = userSchema.validatePartial({ email: "bad" })

// Parse = validate + deserialize (throws on failure)
const user = userSchema.parse(rawData)
```
