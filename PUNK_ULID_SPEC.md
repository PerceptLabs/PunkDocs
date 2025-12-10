# Punk ULID Implementation

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Overview

All Punk systems use ULIDs (Universally Unique Lexicographically Sortable Identifiers) as the standard identifier format.

**Key Properties:**
- 26 characters, URL-safe (Crockford's Base32)
- Timestamp-sortable (creation time extractable)
- 128 bits total (48-bit timestamp + 80-bit randomness)
- Monotonic option for strict ordering within millisecond

---

## 1. Core Package

### 1.1 API

```typescript
// @punk/id/index.ts

import { ulid, monotonicFactory, decodeTime } from 'ulid'

/**
 * Generate a standard ULID
 * Use for: most entity IDs, one-off records
 */
export const createId = ulid

/**
 * Create a monotonic ULID generator
 * Use for: revision sequences, ordered batches
 */
export function createMonotonicGenerator() {
  return monotonicFactory()
}

/**
 * Extract timestamp from ULID
 * Returns milliseconds since Unix epoch
 */
export { decodeTime }

/**
 * Extract timestamp as Date
 */
export function getCreatedAt(id: string): Date {
  return new Date(decodeTime(id))
}

/**
 * Validate ULID format
 */
export function isValidUlid(id: string): boolean {
  if (id.length !== 26) return false
  const alphabet = '0123456789ABCDEFGHJKMNPQRSTVWXYZ'
  return id.toUpperCase().split('').every(c => alphabet.includes(c))
}

/**
 * Compare ULIDs (for sorting)
 */
export function compareUlids(a: string, b: string): number {
  return a.localeCompare(b)
}
```

### 1.2 System-Specific Generators

```typescript
// @punk/synthpunk/id.ts
import { createMonotonicGenerator } from '@punk/id'

const revisionIdGenerator = createMonotonicGenerator()

export function createRevisionId(): string {
  return revisionIdGenerator()
}
```

```typescript
// @punk/mohawk/id.ts
import { createId } from '@punk/id'

export const createProjectId = createId
export const createSessionId = createId
export const createAssetId = createId
```

---

## 2. TypeScript Types

### 2.1 Branded Types

```typescript
// @punk/id/types.ts

declare const UlidBrand: unique symbol
export type Ulid = string & { readonly [UlidBrand]: typeof UlidBrand }

export type SchemaId = Ulid & { readonly __entity: 'schema' }
export type RevisionId = Ulid & { readonly __entity: 'revision' }
export type ProjectId = Ulid & { readonly __entity: 'project' }
export type UserId = Ulid & { readonly __entity: 'user' }
export type AssetId = Ulid & { readonly __entity: 'asset' }
export type RigId = Ulid & { readonly __entity: 'rig' }
export type ModId = Ulid & { readonly __entity: 'mod' }

export function isUlid(value: unknown): value is Ulid {
  return typeof value === 'string' && isValidUlid(value)
}

export function asSchemaId(id: string): SchemaId {
  if (!isValidUlid(id)) throw new Error(`Invalid ULID: ${id}`)
  return id as SchemaId
}
```

### 2.2 Effect Schema Integration

```typescript
// @punk/id/schema.ts
import { Schema } from '@effect/schema'
import { isValidUlid } from './index'

/**
 * Effect schema for ULID validation
 */
export const UlidSchema = Schema.String.pipe(
  Schema.filter(isValidUlid, {
    message: () => 'Invalid ULID format',
  })
)

/**
 * Entity-specific branded schemas
 */
export const SchemaIdSchema = UlidSchema.pipe(Schema.brand('SchemaId'))
export const RevisionIdSchema = UlidSchema.pipe(Schema.brand('RevisionId'))
export const ProjectIdSchema = UlidSchema.pipe(Schema.brand('ProjectId'))
export const UserIdSchema = UlidSchema.pipe(Schema.brand('UserId'))
export const AssetIdSchema = UlidSchema.pipe(Schema.brand('AssetId'))
export const RigIdSchema = UlidSchema.pipe(Schema.brand('RigId'))
export const ModIdSchema = UlidSchema.pipe(Schema.brand('ModId'))

/**
 * Type exports
 */
export type Ulid = Schema.Schema.Type<typeof UlidSchema>
export type SchemaId = Schema.Schema.Type<typeof SchemaIdSchema>
export type RevisionId = Schema.Schema.Type<typeof RevisionIdSchema>
export type ProjectId = Schema.Schema.Type<typeof ProjectIdSchema>

/**
 * Usage in Punk schemas
 */
export const PunkComponentSchema: Schema.Schema<PunkComponent> = Schema.suspend(
  () => Schema.Struct({
    id: SchemaIdSchema,
    type: Schema.String,
    props: Schema.Record({ key: Schema.String, value: Schema.Unknown }),
    children: Schema.optional(Schema.Array(PunkComponentSchema)),
  })
)

interface PunkComponent {
  id: SchemaId
  type: string
  props: Record<string, unknown>
  children?: PunkComponent[]
}
```

---

## 3. Usage Patterns

### 3.1 Schema Component IDs

```typescript
const button: Schema = {
  id: createId(),  // 01ARZ3NDEKTSV4RRFFQ69G5FAV
  type: 'Button',
  props: { variant: 'primary', children: 'Click me' }
}
```

### 3.2 Revision Sequences (Monotonic)

```typescript
const revisionId = createMonotonicGenerator()

const r1 = revisionId()  // 01ARZ3NDEKTSV4RRFFQ69G5FA1
const r2 = revisionId()  // 01ARZ3NDEKTSV4RRFFQ69G5FA2
const r3 = revisionId()  // 01ARZ3NDEKTSV4RRFFQ69G5FA3

// Guaranteed: r1 < r2 < r3 (lexicographic)
```

### 3.3 URL Construction

```
https://mohawk.punk.dev/project/01CX6YYLCLBDUAW0FXWHFNNWS0
https://mohawk.punk.dev/project/01CX6YYLCLBDUAW0FXWHFNNWS0/revision/01BX5ZZKBKACTAV9WEVGEMMVR3
```

### 3.4 Debugging with Timestamps

```typescript
import { decodeTime } from '@punk/id'

function debugId(id: string): void {
  const timestamp = decodeTime(id)
  const created = new Date(timestamp)
  console.log(`ID: ${id}`)
  console.log(`Created: ${created.toISOString()}`)
}
```

---

## 4. Database Considerations

### 4.1 SQLite (GlyphCase)

```sql
CREATE TABLE schemas (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  props TEXT NOT NULL,
  created_at GENERATED ALWAYS AS (
    datetime(substr(id, 1, 10), 'unixepoch', 'subsec')
  ) VIRTUAL
);

SELECT * FROM schemas ORDER BY id;
```

### 4.2 PostgreSQL

```sql
CREATE TABLE projects (
  id CHAR(26) PRIMARY KEY,
  name TEXT NOT NULL,
  schema JSONB NOT NULL
);

CREATE INDEX idx_projects_id ON projects(id);
```

### 4.3 Index Performance

ULIDs append sequentially vs UUID's random insertion:
- Sequential writes → cache hits → compact index
- No page splits or fragmentation

### 4.4 Storage

```
UUID (with hyphens):  36 bytes
ULID (string):        26 bytes

Recommendation: Store as 26-char TEXT for readability
```

---

## 5. API Design

### 5.1 REST Endpoints

```
GET  /api/projects/:projectId
GET  /api/projects/:projectId/revisions/:revisionId
POST /api/projects/:projectId/assets
```

### 5.2 GraphQL

```graphql
scalar ULID

type Schema {
  id: ULID!
  type: String!
  props: JSON!
  createdAt: DateTime!
}
```

### 5.3 Pagination

```typescript
interface PaginatedResponse<T> {
  items: T[]
  cursor: string | null  // Last item's ULID
  hasMore: boolean
}

// Efficient query
SELECT * FROM revisions 
WHERE id > :cursor
ORDER BY id
LIMIT 20;
```

---

## 6. Security Considerations

### 6.1 Information Leakage

ULIDs embed timestamps. For sensitive contexts:

```typescript
const publicSlug = hashUlid(userId)  // Non-reversible
```

### 6.2 Predictability

80 bits of randomness—cannot guess or enumerate.

### 6.3 Collision Probability

1.21e+24 unique IDs per millisecond. Negligible collision risk.

---

## 7. Integration Checklist

- [ ] Create `@punk/id` package
- [ ] Add Effect schemas
- [ ] Punk Core: Schema IDs, component instance IDs
- [ ] Mohawk: Project, user, session, asset IDs
- [ ] GlyphCase: Row primary keys
- [ ] Depot: Package, version, publisher IDs

---

*Last updated: December 2025*
