# Punk Backends

**Version:** 2.0  
**Status:** Canonical Source of Truth

---

## Overview

Apps built with Punk can use various backend options. Backend choice is independent of Punk's rendering systemâ€”any backend works with the same schema-driven frontend.

---

## Runtime Philosophy: Bun-First

For full-stack applications, Punk is **Bun-first**. Node.js is supported but Bun is preferred.

**Why Bun:**

| Capability | Benefit |
|------------|---------|
| Runtime + bundler + package manager | Single tool, simpler toolchain |
| Native TypeScript | No transpilation step |
| bun:sqlite | Fast SQLite bindings for GlyphCase |
| bun:wasm | Excellent WASM support for Trinity Runtime |
| Speed | Faster startup, faster execution |

**Frontend-only apps** (no backend) build to static HTML/CSS/JS and deploy anywhereâ€”no runtime needed in production.

**Full-stack apps** with Encore.ts, tRPC, or GlyphCase use Bun as the server runtime.

---

## Built-in Backend Options

| Backend | Description | Use Case |
|---------|-------------|----------|
| **None** | Static site, no backend | Landing pages, documentation, portfolios |
| **Encore.ts** | Infrastructure from code (TypeScript) | Type-safe APIs, cloud deployment |
| **Encore** | Infrastructure from code (Go) | High-performance APIs |
| **tRPC** | End-to-end type safety | Full-stack TypeScript applications |
| **GlyphCase** | Local SQLite database | Local-first apps, offline support, data sovereignty |
| **Manifest** | Declarative YAML backend | Simple CRUD without code |
| **Neon** | Serverless PostgreSQL | Scale-to-zero, database branching, autoscaling |

---

## Backend Details

### None (Static)

No backend. Output is static HTML/CSS/JS.

**Good for:** Marketing sites, documentation, simple portfolios.

### Encore.ts

TypeScript-based infrastructure-from-code framework.

**Features:**
- Type-safe APIs
- Automatic cloud infrastructure
- Built-in observability
- Local development environment

**Example:**
```typescript
import { api } from "encore.dev/api"

export const getTasks = api(
  { method: "GET", path: "/tasks" },
  async (): Promise<Task[]> => {
    return db.query("SELECT * FROM tasks")
  }
)
```

### Encore (Go)

Go-based version for high-performance needs.

### tRPC

End-to-end TypeScript type safety without code generation.

**Features:**
- Full type inference from backend to frontend
- No code generation step
- Works with any TypeScript backend

### GlyphCase

Local-first SQLite database.

**Features:**
- Data stays on user's machine
- Works offline
- Single-file database
- Self-hosting friendly

**Good for:** Privacy-focused apps, offline-first applications, enterprise deployments.

### Manifest

Declarative YAML-based backend.

**Features:**
- Define CRUD operations in YAML
- No code required
- Automatic API generation

**Example:**
```yaml
entities:
  Task:
    fields:
      title: string
      completed: boolean
      due_date: date
    operations:
      - create
      - read
      - update
      - delete
```

### Neon

Serverless PostgreSQL with modern features.

**Features:**
- Scale-to-zero (pay only when active)
- Database branching (git-like workflow for data)
- Autoscaling based on load
- Instant provisioning (~300ms)
- Point-in-time recovery

**Good for:** Apps needing PostgreSQL with variable load, development workflows requiring database branches.

---

## External Backends (via Mods)

Install via Depot:

| Backend | Mod |
|---------|-----|
| Supabase | `supabase-mod` |
| Firebase | `firebase-mod` |
| Appwrite | `appwrite-mod` |
| PocketBase | `pocketbase-mod` |

These mods provide:
- Authentication integration
- Database operations
- Storage access
- Real-time subscriptions (where supported)

---

## Backend Generation (Atompunk)

At Tier 3, Ska generates backend code using vetted templates.

**Security model:** AI fills parameters in pre-reviewed templates, not raw code.

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  User: "Add PostgreSQL persistence for tasks with CRUD"         â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                  â†“
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Ska selects template: crud-endpoint.ts                         â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                  â†“
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Ska fills parameters:                                          â”‚
â”‚  â€¢ tableName: "tasks"                                           â”‚
â”‚  â€¢ fields: ["title", "completed", "due_date"]                   â”‚
â”‚  â€¢ operations: ["create", "read", "update", "delete"]           â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                  â†“
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Template includes:                                             â”‚
â”‚  â€¢ Input validation (@effect/schema)                                       â”‚
â”‚  â€¢ CSRF protection                                              â”‚
â”‚  â€¢ Rate limiting                                                â”‚
â”‚  â€¢ SQL injection prevention                                     â”‚
â”‚  â€¢ XSS protection                                               â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

---

## Mohawk Platform Backend

Note: This is separate from app backends.

| Deployment | Backend | Database |
|------------|---------|----------|
| **Mohawk SaaS** | Encore.ts | PostgreSQL |
| **Mohawk Docker** | Encore.ts | PostgreSQL |
| **Mohawk Desktop** | Local | SQLite |

---

*Last updated: December 2025*
