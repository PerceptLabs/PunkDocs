# Punk VM Implementation Guide

**Version:** 2.0  
**Status:** Source of Truth  
**Audience:** Human Developers & Claude Code

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Punk VM Lite (Open Source)](#3-punk-vm-lite-open-source)
4. [Punk VM Full (Mohawk Edition)](#4-punk-vm-full-mohawk-edition)
5. [Core Primitives](#5-core-primitives)
6. [Backends](#6-backends)
7. [Rigs (Component Wrappers)](#7-rigs-component-wrappers)
8. [Starter Templates](#8-starter-templates)
9. [VM Foundation (libkrun)](#9-vm-foundation-libkrun)
10. [MCP Server Interface](#10-mcp-server-interface)
11. [Platform Support](#11-platform-support)
12. [Package Structure](#12-package-structure)
13. [API Reference](#13-api-reference)
14. [CLI Reference](#14-cli-reference)
15. [Implementation Phases](#15-implementation-phases)
16. [Testing Strategy](#16-testing-strategy)
17. [Distribution](#17-distribution)
18. [Upgrade Path: Lite → Full](#18-upgrade-path-lite--full)
19. [Appendix: Code Examples](#19-appendix-code-examples)

---

## 1. Executive Summary

### What Is Punk VM?

Punk VM is a schema-first development environment that enables AI assistants (via MCP) to build validated, type-safe applications locally. It combines hardware-isolated code execution (via libkrun microVMs) with Punk's core primitives for schema validation, component rendering, and styling.

### Two Editions

| Edition | License | Purpose |
|---------|---------|---------|
| **Punk VM Lite** | Apache 2.0 | Open source, local development, BYOM (bring your own model) |
| **Punk VM Full** | Commercial | Mohawk platform integration, advanced features, cloud support |

### Key Terminology

| Term | Definition |
|------|------------|
| **Rig** | An external library wrapped for Puck compatibility (e.g., Chart.js → Chart rig) |
| **Starter Template** | Project scaffold for a specific app type (e.g., react-spa, api-glyphcase) |
| **Mod** | Capability extension loaded at runtime (Full only) |
| **Depot** | Mohawk's marketplace for rigs and mods |

### Core Principle

**Lite is full Punk, locally.** The difference is ecosystem (mods, Depot rigs) and scale (cloud, multi-tenant), not capability.

---

## 2. Architecture Overview

### 2.1 Lite Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     PUNK VM LITE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    MCP SERVER                              │ │
│  │  Exposes Punk primitives to any MCP-compatible client      │ │
│  │  (Claude Desktop, Cursor, Continue, custom clients)        │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  PUNK CORE PRIMITIVES                      │ │
│  │                                                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │ │
│  │  │  Effect  │ │   Puck   │ │ Garden.js│ │   Pink   │     │ │
│  │  │  Schema  │ │ Renderer │ │Components│ │   CSS    │     │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘     │ │
│  │                                                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │ │
│  │  │TokiForge │ │GlyphCase │ │   ULID   │                   │ │
│  │  │  Tokens  │ │  SQLite  │ │   IDs    │                   │ │
│  │  └──────────┘ └──────────┘ └──────────┘                   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              CORE RIGS (Component Wrappers)                │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │ │
│  │  │ Table  │ │ Chart  │ │  Date  │ │  Code  │ │Markdown│  │ │
│  │  │TanStack│ │Chart.js│ │ Picker │ │ Mirror │ │  (new) │  │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    BACKENDS                                │ │
│  │  ┌────────────────────────┐ ┌────────────────────────┐    │ │
│  │  │   None (Static)        │ │   GlyphCase (SQLite)   │    │ │
│  │  │   No server needed     │ │   Local persistence    │    │ │
│  │  └────────────────────────┘ └────────────────────────┘    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              STARTER TEMPLATES (Project Scaffolds)         │ │
│  │  ┌────────────┐ ┌────────────┐ ┌─────────────┐ ┌────────┐│ │
│  │  │Static Site │ │ React SPA  │ │API GlyphCase│ │CLI Tool││ │
│  │  └────────────┘ └────────────┘ └─────────────┘ └────────┘│ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    LIBKRUN VM                              │ │
│  │  Hardware-isolated execution (KVM/HVF)                     │ │
│  │  OCI image support, resource limits, networking            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Full Architecture (Mohawk SaaS VM)

```
┌─────────────────────────────────────────────────────────────────┐
│                     PUNK VM FULL (MOHAWK)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Everything in Lite, PLUS:                                      │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  DEPOT INTEGRATION                         │ │
│  │                                                            │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │              DEPOT (Rig Marketplace)                 │  │ │
│  │  │                                                      │  │ │
│  │  │  90+ rigs available, pulled on demand:               │  │ │
│  │  │  • RichText (Lexical)    • Mermaid                   │  │ │
│  │  │  • FileDrop              • Command (cmdk)            │  │ │
│  │  │  • ColorPicker           • Slider                    │  │ │
│  │  │  • Carousel              • Timeline                  │  │ │
│  │  │  • TreeView              • Kanban                    │  │ │
│  │  │  • (community rigs...)                               │  │ │
│  │  │                                                      │  │ │
│  │  │  App uses 19 rigs? VM pulls exactly those 19.        │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │              CLOUD BACKENDS                          │  │ │
│  │  │                                                      │  │ │
│  │  │  Built-in:                                           │  │ │
│  │  │  • Encore.ts (infrastructure from code)              │  │ │
│  │  │  • tRPC (end-to-end type safety)                     │  │ │
│  │  │  • Neon (serverless PostgreSQL)                      │  │ │
│  │  │  • Manifest (YAML → CRUD)                            │  │ │
│  │  │                                                      │  │ │
│  │  │  Via Mods:                                           │  │ │
│  │  │  • Supabase, Firebase, PocketBase, Appwrite          │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  MOHAWK INTEGRATION                        │ │
│  │                                                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │ │
│  │  │  Trinity │ │   Nitro  │ │    Ska   │ │   Mods   │     │ │
│  │  │ Runtime  │ │Supervisor│ │Orchestr. │ │Ecosystem │     │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘     │ │
│  │                                                            │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │ │
│  │  │ Snapshot │ │  Display │ │  Multi-  │                   │ │
│  │  │ /Restore │ │ Capture  │ │  tenant  │                   │ │
│  │  └──────────┘ └──────────┘ └──────────┘                   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Rig Distribution Model

```
LITE VM                              FULL VM (MOHAWK SAAS)
                                     
┌─────────────────────┐             ┌─────────────────────┐
│  5 BUNDLED RIGS     │             │   DEPOT REGISTRY    │
│  (shipped with CLI) │             │   (90+ rigs)        │
│                     │             │         │           │
│  • Table            │             │         ▼           │
│  • Chart            │             │  ┌──────────────┐   │
│  • DatePicker       │             │  │ App manifest │   │
│  • Code             │             │  │ uses: 19 rigs│   │
│  • Markdown         │             │  └──────┬───────┘   │
│                     │             │         │           │
│  Self-contained     │             │         ▼           │
│  Works offline      │             │  VM pulls only      │
│                     │             │  what's needed      │
└─────────────────────┘             └─────────────────────┘
```

---

## 3. Punk VM Lite (Open Source)

### 3.1 Design Philosophy

1. **Full Punk primitives** — Schema validation, rendering, styling all work
2. **Local-first** — No cloud dependency, no account required
3. **BYOM** — Connect any LLM via MCP (Claude, GPT, Llama, etc.)
4. **Deterministic** — Same input → same output
5. **Self-contained** — Core rigs bundled, works offline

### 3.2 Component Inventory

| Component | Package | Purpose |
|-----------|---------|---------|
| Effect Schema | `@effect/schema` | Validation, type safety |
| Puck | `@punk/puck` | Schema → React components |
| Garden.js | `@punk/garden` | Component infrastructure, rig definitions |
| Pink | `@punk/pink` | CSS design system |
| TokiForge | `@punk/tokiforge` | Design tokens |
| GlyphCase | `@punk/glyphcase` | Local SQLite logging/storage |
| ULID | `@punk/ulid` | ID generation |
| VM Core | `@punk/vm-core` | libkrun bindings |
| MCP Server | `@punk/vm-mcp` | MCP protocol implementation |
| CLI | `@punk/vm-cli` | Command-line interface |

### 3.3 What's Included

| Category | Lite | Notes |
|----------|------|-------|
| **Primitives** | All 7 | Full Punk core |
| **Backends** | 2 | None (static), GlyphCase (SQLite) |
| **Rigs** | 5 core | Table, Chart, DatePicker, Code, Markdown |
| **Templates** | 4 starters | static-site, react-spa, api-glyphcase, cli-tool |
| **VM** | Full | libkrun with all features |
| **MCP** | Full | Complete tool/resource interface |

### 3.4 What's NOT Included

| Component | Why Excluded |
|-----------|--------------|
| Trinity Runtime | Executes mod code; no mods in Lite |
| Mods | Ecosystem feature, requires Depot |
| Depot Rigs | Marketplace integration (90+ additional rigs) |
| Ska Orchestration | Cloud AI service |
| Nitro (cloud) | Multi-process supervision for complex workflows |
| Snapshot/Restore | Advanced VM state management |
| Display Capture | Screenshot/video requires complex pipeline |

---

## 4. Punk VM Full (Mohawk Edition)

### 4.1 Rig Inheritance from Depot

The key difference: **Full VM pulls rigs dynamically from Depot**.

```typescript
// App manifest declares what it needs
{
  "name": "my-dashboard",
  "rigs": [
    "Table",       // Core (always available)
    "Chart",       // Core
    "RichText",    // From Depot
    "Kanban",      // From Depot  
    "Timeline"     // From Depot
  ]
}

// Mohawk VM resolves and loads:
// - Core rigs: bundled
// - Depot rigs: pulled on first use, cached
```

### 4.2 Additional Features

| Feature | Purpose |
|---------|---------|
| Trinity Runtime | Polyglot execution (Lua, WASM, JS) for mods |
| Nitro Supervision | Process management, health checks, restart |
| Ska Integration | AI orchestration with verification loops |
| Mods | Extensible capabilities (PDF, Supabase, etc.) |
| Depot Access | Full rig marketplace (90+ components) |
| Snapshot/Restore | VM state persistence, workflow resume |
| Display Capture | Screenshots, video recording |
| Multi-tenant | User isolation in SaaS |

### 4.3 Licensing

| Use Case | License |
|----------|---------|
| Development/Testing | Free |
| Self-hosted (small) | Free |
| Self-hosted (>10 users) | Commercial license |
| Offer as service | Commercial license |

---

## 5. Core Primitives

### 5.1 Effect Schema

**Purpose:** Runtime validation with TypeScript type inference.

```typescript
// @punk/schema/component.ts
import { Schema as S } from "@effect/schema"

export const PunkComponent: S.Schema<PunkComponent> = S.suspend(() =>
  S.Struct({
    type: S.String.pipe(S.minLength(1)),
    props: S.Record(S.String, S.Unknown).pipe(S.withDefault(() => ({}))),
    children: S.Array(PunkComponent).pipe(S.withDefault(() => [])),
  })
)

export const PunkDocument = S.Struct({
  version: S.Literal("1.0").pipe(S.withDefault(() => "1.0" as const)),
  root: PunkComponent,
  metadata: S.optional(S.Struct({
    generatedAt: S.optional(S.DateFromString),
    model: S.optional(S.String),
    prompt: S.optional(S.String),
  })),
})

export type PunkComponent = {
  type: string
  props: Record<string, unknown>
  children: PunkComponent[]
}

export type PunkDocument = S.Schema.Type<typeof PunkDocument>
```

### 5.2 Puck (Schema Renderer)

**Purpose:** Transform validated schemas into React component trees.

```typescript
// @punk/puck/renderer.tsx
import { useGardenRegistry } from "@punk/garden"
import type { PunkComponent } from "@punk/schema"

export function PuckRenderer({ schema }: { schema: PunkComponent }) {
  const registry = useGardenRegistry()
  
  const renderNode = (node: PunkComponent): React.ReactNode => {
    const Component = registry.get(node.type)
    
    if (!Component) {
      console.warn(`Unknown component type: ${node.type}`)
      return null
    }
    
    const children = node.children.map((child, i) => (
      <React.Fragment key={child.props?.id ?? i}>
        {renderNode(child)}
      </React.Fragment>
    ))
    
    return <Component {...node.props}>{children}</Component>
  }
  
  return <>{renderNode(schema)}</>
}
```

### 5.3 Garden.js (Component Infrastructure)

**Purpose:** Declarative rig definitions, component registry, preview engine.

```typescript
// @punk/garden/define-rig.ts
export interface RigDefinition<P = unknown> {
  name: string
  component: React.ComponentType<P>
  library: string
  category: string
  
  props: Record<string, PropDefinition>
  
  mapProps?: (props: Record<string, unknown>) => P
}

export function defineRig<P>(definition: RigDefinition<P>) {
  // Auto-generate Effect schema from prop definitions
  const schema = generateSchema(definition.props)
  
  // Auto-generate Puck configuration
  const puckConfig = generatePuckConfig(definition)
  
  return { ...definition, schema, puckConfig }
}
```

### 5.4 Pink (CSS System)

**Purpose:** Utility-first CSS design system with semantic tokens.

```typescript
// @punk/pink/classes.ts
export const flex = "pink-flex"
export const grid = "pink-grid"
export const container = "pink-container"

export const p = (size: 1 | 2 | 3 | 4 | 5 | 6) => `pink-p-${size}`
export const m = (size: 1 | 2 | 3 | 4 | 5 | 6) => `pink-m-${size}`

export const button = {
  primary: "pink-button-primary",
  secondary: "pink-button-secondary",
  ghost: "pink-button-ghost",
}
```

### 5.5 TokiForge (Design Tokens)

**Purpose:** Design token management with runtime theming.

```typescript
// @punk/tokiforge/tokens.ts
export const defaultTokens = {
  colors: {
    primary: { value: "#6366f1" },
    secondary: { value: "#8b5cf6" },
    background: { value: "#ffffff" },
    foreground: { value: "#1f2937" },
  },
  spacing: {
    1: { value: "0.25rem" },
    2: { value: "0.5rem" },
    3: { value: "0.75rem" },
    4: { value: "1rem" },
  },
  // ...
}
```

### 5.6 GlyphCase (SQLite Storage)

**Purpose:** Local-first SQLite storage for projects, audit logs, preferences.

```typescript
// @punk/glyphcase/client.ts
import Database from "bun:sqlite"

export class GlyphCase {
  private db: Database
  
  constructor(path: string = "./punk.db") {
    this.db = new Database(path)
    this.migrate()
  }
  
  createProject(name: string, schema: object): string { /* ... */ }
  getProject(id: string) { /* ... */ }
  log(eventType: string, payload: object) { /* ... */ }
  getPref(key: string): string | null { /* ... */ }
  setPref(key: string, value: string) { /* ... */ }
}
```

### 5.7 ULID (ID Generation)

**Purpose:** Universally Unique Lexicographically Sortable Identifiers.

```typescript
// @punk/ulid/index.ts
export function ulid(seedTime?: number): string { /* ... */ }
export function isValidUlid(id: string): boolean { /* ... */ }
export function timestampFromUlid(id: string): number { /* ... */ }

// Effect schema integration
export const UlidSchema = S.String.pipe(
  S.filter(isValidUlid, { message: () => "Invalid ULID format" })
)
```

---

## 6. Backends

### 6.1 Lite Backends

Punk VM Lite includes two backends that work offline with no external dependencies:

| Backend | Description | Use Case |
|---------|-------------|----------|
| **None (Static)** | No server, static HTML/CSS/JS | Landing pages, docs, portfolios, tools |
| **GlyphCase (SQLite)** | Local SQLite database | Persistent apps, APIs, CRUD operations |

### 6.2 None (Static)

No backend required. Output is pure HTML/CSS/JS that runs in any browser.

**Good for:**
- Landing pages
- Documentation sites
- Portfolios
- Stateless tools (calculators, formatters, viewers)

**Storage options (client-side):**
- React state (memory, lost on refresh)
- localStorage (persists, ~5MB limit)
- IndexedDB (persists, larger capacity)

### 6.3 GlyphCase (SQLite)

Local-first SQLite database via Bun's native bindings.

**Features:**
- Single-file database
- Works offline
- Data stays on user's machine
- Full SQL support
- Audit logging built in

**Good for:**
- Personal data apps (expense trackers, notes)
- REST APIs with persistence
- Admin panels
- Local-first applications

```typescript
import { GlyphCase } from "@punk/glyphcase"

const db = new GlyphCase("./data.db")

// Create table
db.exec(`CREATE TABLE IF NOT EXISTS tasks (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  completed INTEGER DEFAULT 0
)`)

// CRUD operations
db.query("INSERT INTO tasks (id, title) VALUES (?, ?)").run(id, title)
const tasks = db.query("SELECT * FROM tasks").all()

// Audit logging
db.log("task.created", { id, title })
```

### 6.4 Full Backends (Mohawk Edition)

Punk VM Full adds cloud-capable backends:

| Backend | Description | Requires |
|---------|-------------|----------|
| **Encore.ts** | Infrastructure from code | Encore account |
| **tRPC** | End-to-end type safety | Database backend |
| **Neon** | Serverless PostgreSQL | Neon account |
| **Manifest** | YAML → CRUD | May target cloud DB |

**Via Mods (Depot):**

| Backend | Mod |
|---------|-----|
| Supabase | `supabase-mod` |
| Firebase | `firebase-mod` |
| PocketBase | `pocketbase-mod` |
| Appwrite | `appwrite-mod` |

### 6.5 Backend Selection Guide

```
Need persistence?
├── No → Static (None)
└── Yes
    └── Single user, local data?
        ├── Yes → GlyphCase (Lite)
        └── No → Cloud backend (Full)
            ├── Type-safe cloud APIs → Encore.ts
            ├── Serverless PostgreSQL → Neon
            ├── Full-stack TypeScript → tRPC
            └── BaaS (auth, realtime) → Supabase/Firebase (mod)
```

---

## 7. Rigs (Component Wrappers)

### 7.1 What Is a Rig?

A **Rig** is an external library wrapped for Puck compatibility. Rigs expose safe, validated configuration surfaces, allowing complex components to work within the schema system.

```
RAW LIBRARY (e.g., Chart.js)
│
│  • Full API surface
│  • Requires React knowledge  
│  • Not schema-compatible
│
▼
GARDEN.JS RIG DEFINITION
│
│  • Restricted configuration surface
│  • Auto-generated Effect Schema
│  • Puck-compatible interface
│
▼
PUCK COMPONENT REGISTRY
│
│  • Chart available as schema type
│  • Same guarantees as native components
```

### 7.2 Core Rigs (Bundled with Lite)

| Rig | Library | Purpose | Why Essential |
|-----|---------|---------|---------------|
| **Table** | TanStack Table | Sortable, paginated data tables | Can't build CRUD, admin, data apps without it |
| **Chart** | Chart.js | Bar, line, pie, doughnut, radar charts | Can't build dashboards without it |
| **DatePicker** | react-day-picker | Date selection with calendar | Forms almost always need dates |
| **Code** | CodeMirror | Syntax-highlighted code display/editing | Dev tools, docs with code snippets |
| **Markdown** | marked + custom | Render markdown to styled HTML | Docs sites, README viewers, content apps |

### 7.3 Table Rig Implementation

```typescript
// @punk/rigs/table/index.ts
import { defineRig } from "@punk/garden"
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  flexRender,
} from "@tanstack/react-table"

export default defineRig({
  name: "Table",
  library: "@tanstack/react-table",
  category: "Data",
  
  props: {
    columns: {
      type: "array",
      required: true,
      items: {
        type: "object",
        properties: {
          key: { type: "string", required: true },
          header: { type: "string", required: true },
          sortable: { type: "boolean", default: true },
        },
      },
    },
    data: {
      type: "array",
      required: true,
      items: { type: "object" },
    },
    pageSize: {
      type: "number",
      default: 10,
    },
    sortable: {
      type: "boolean",
      default: true,
    },
  },
  
  component: function Table({ columns, data, pageSize, sortable }) {
    const table = useReactTable({
      columns: columns.map(col => ({
        accessorKey: col.key,
        header: col.header,
        enableSorting: col.sortable ?? sortable,
      })),
      data,
      getCoreRowModel: getCoreRowModel(),
      getSortedRowModel: sortable ? getSortedRowModel() : undefined,
      getPaginationRowModel: getPaginationRowModel(),
      initialState: { pagination: { pageSize } },
    })
    
    return (
      <div className="pink-table-container">
        <table className="pink-table">
          <thead>
            {table.getHeaderGroups().map(headerGroup => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map(header => (
                  <th key={header.id} onClick={header.column.getToggleSortingHandler()}>
                    {flexRender(header.column.columnDef.header, header.getContext())}
                    {header.column.getIsSorted() === "asc" && " ↑"}
                    {header.column.getIsSorted() === "desc" && " ↓"}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody>
            {table.getRowModel().rows.map(row => (
              <tr key={row.id}>
                {row.getVisibleCells().map(cell => (
                  <td key={cell.id}>
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
        <div className="pink-pagination">
          <button onClick={() => table.previousPage()} disabled={!table.getCanPreviousPage()}>
            Previous
          </button>
          <span>Page {table.getState().pagination.pageIndex + 1} of {table.getPageCount()}</span>
          <button onClick={() => table.nextPage()} disabled={!table.getCanNextPage()}>
            Next
          </button>
        </div>
      </div>
    )
  },
})
```

### 7.4 Chart Rig Implementation

```typescript
// @punk/rigs/chart/index.ts
import { defineRig } from "@punk/garden"
import { Chart as ChartJS, registerables } from "chart.js"
import { Chart } from "react-chartjs-2"

ChartJS.register(...registerables)

export default defineRig({
  name: "Chart",
  library: "chart.js",
  category: "Data",
  
  props: {
    chartType: {
      type: "enum",
      options: ["bar", "line", "pie", "doughnut", "radar", "area"],
      default: "bar",
    },
    title: {
      type: "string",
      default: "",
    },
    showLegend: {
      type: "boolean",
      default: true,
    },
    legendPosition: {
      type: "enum",
      options: ["top", "bottom", "left", "right"],
      default: "top",
    },
    data: {
      type: "array",
      required: true,
      items: {
        type: "object",
        properties: {
          label: { type: "string" },
          value: { type: "number" },
        },
      },
    },
    colors: {
      type: "array",
      items: { type: "string" },
    },
  },
  
  component: function ChartRig({ chartType, title, showLegend, legendPosition, data, colors }) {
    const defaultColors = [
      "#6366f1", "#8b5cf6", "#ec4899", "#f59e0b", "#10b981",
      "#3b82f6", "#ef4444", "#84cc16", "#06b6d4", "#f97316",
    ]
    
    const chartData = {
      labels: data.map(d => d.label),
      datasets: [{
        data: data.map(d => d.value),
        backgroundColor: colors ?? defaultColors.slice(0, data.length),
      }],
    }
    
    const options = {
      responsive: true,
      plugins: {
        title: { display: !!title, text: title },
        legend: { display: showLegend, position: legendPosition },
      },
    }
    
    return <Chart type={chartType} data={chartData} options={options} />
  },
})
```

### 7.5 DatePicker Rig Implementation

```typescript
// @punk/rigs/datepicker/index.ts
import { defineRig } from "@punk/garden"
import { DayPicker } from "react-day-picker"
import "react-day-picker/dist/style.css"

export default defineRig({
  name: "DatePicker",
  library: "react-day-picker",
  category: "Form",
  
  props: {
    value: {
      type: "string",
      description: "ISO date string",
    },
    placeholder: {
      type: "string",
      default: "Select date",
    },
    minDate: {
      type: "string",
    },
    maxDate: {
      type: "string",
    },
    disabled: {
      type: "boolean",
      default: false,
    },
    onChange: {
      type: "string",
      description: "Handler function name",
    },
  },
  
  component: function DatePickerRig({ value, placeholder, minDate, maxDate, disabled, onChange }) {
    const [selected, setSelected] = React.useState(value ? new Date(value) : undefined)
    const [isOpen, setIsOpen] = React.useState(false)
    
    return (
      <div className="pink-datepicker">
        <button 
          className="pink-input" 
          onClick={() => setIsOpen(!isOpen)}
          disabled={disabled}
        >
          {selected ? selected.toLocaleDateString() : placeholder}
        </button>
        {isOpen && (
          <div className="pink-datepicker-popover">
            <DayPicker
              mode="single"
              selected={selected}
              onSelect={(date) => {
                setSelected(date)
                setIsOpen(false)
              }}
              disabled={disabled}
              fromDate={minDate ? new Date(minDate) : undefined}
              toDate={maxDate ? new Date(maxDate) : undefined}
            />
          </div>
        )}
      </div>
    )
  },
})
```

### 7.6 Code Rig Implementation

```typescript
// @punk/rigs/code/index.ts
import { defineRig } from "@punk/garden"
import CodeMirror from "@uiw/react-codemirror"
import { javascript } from "@codemirror/lang-javascript"
import { python } from "@codemirror/lang-python"
import { html } from "@codemirror/lang-html"
import { css } from "@codemirror/lang-css"
import { json } from "@codemirror/lang-json"

const languages = { javascript, python, html, css, json }

export default defineRig({
  name: "Code",
  library: "@uiw/react-codemirror",
  category: "Display",
  
  props: {
    code: {
      type: "string",
      required: true,
    },
    language: {
      type: "enum",
      options: ["javascript", "typescript", "python", "html", "css", "json"],
      default: "javascript",
    },
    editable: {
      type: "boolean",
      default: false,
    },
    lineNumbers: {
      type: "boolean",
      default: true,
    },
    theme: {
      type: "enum",
      options: ["light", "dark"],
      default: "light",
    },
  },
  
  component: function CodeRig({ code, language, editable, lineNumbers, theme }) {
    const langExtension = languages[language]?.() ?? javascript()
    
    return (
      <CodeMirror
        value={code}
        extensions={[langExtension]}
        editable={editable}
        basicSetup={{ lineNumbers }}
        theme={theme}
        className="pink-code-block"
      />
    )
  },
})
```

### 7.7 Markdown Rig Implementation

```typescript
// @punk/rigs/markdown/index.ts
import { defineRig } from "@punk/garden"
import { marked } from "marked"
import DOMPurify from "dompurify"

export default defineRig({
  name: "Markdown",
  library: "marked",
  category: "Display",
  
  props: {
    content: {
      type: "string",
      required: true,
    },
    allowHtml: {
      type: "boolean",
      default: false,
    },
  },
  
  component: function MarkdownRig({ content, allowHtml }) {
    const html = marked(content)
    const sanitized = allowHtml ? html : DOMPurify.sanitize(html)
    
    return (
      <div 
        className="pink-markdown"
        dangerouslySetInnerHTML={{ __html: sanitized }}
      />
    )
  },
})
```

### 7.8 Using Rigs in Schemas

Once wrapped, rigs work like any other component in Punk schemas:

```json
{
  "type": "Chart",
  "props": {
    "chartType": "bar",
    "title": "Sales by Region",
    "showLegend": true,
    "data": [
      { "label": "Q1", "value": 120 },
      { "label": "Q2", "value": 150 },
      { "label": "Q3", "value": 180 }
    ]
  }
}
```

Claude (or any LLM) can use Chart.js without learning Chart.js—it just knows valid schema props.

### 7.9 Depot Rigs (Full Only)

In Mohawk Full, additional rigs are available from Depot:

| Rig | Library | Category |
|-----|---------|----------|
| RichText | Lexical | Form |
| Mermaid | Mermaid | Display |
| FileDrop | react-dropzone | Form |
| Command | cmdk | Navigation |
| ColorPicker | react-colorful | Form |
| Slider | @radix-ui/react-slider | Form |
| Carousel | embla-carousel | Display |
| Timeline | custom | Display |
| TreeView | custom | Navigation |
| Kanban | custom | Layout |

---

## 8. Starter Templates

### 8.1 What Is a Starter Template?

A **Starter Template** is a project scaffold for a specific app type. Templates provide the file structure, dependencies, and boilerplate code to get started quickly.

**Rigs ≠ Templates:**
- Rig = Component wrapper (Chart, Table)
- Template = Project scaffold (react-spa, api-glyphcase)

### 8.2 Available Templates

| Template | Backend | Use Case |
|----------|---------|----------|
| **static-site** | None | Landing pages, portfolios, docs |
| **react-spa** | None (localStorage) | Interactive apps, tools |
| **api-glyphcase** | GlyphCase (SQLite) | REST APIs with persistence |
| **cli-tool** | None | Automation, scripts |

### 8.3 static-site Template

```typescript
// @punk/templates/static-site/index.ts
export default {
  id: "static-site",
  name: "Static Site",
  description: "HTML/CSS/JS static website",
  
  files: {
    "index.html": `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{name}}</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <main id="app"></main>
  <script type="module" src="main.js"></script>
</body>
</html>`,
    
    "styles.css": `@import "@punk/pink/css";

:root {
  --color-primary: #6366f1;
}

body {
  font-family: var(--font-sans);
  margin: 0;
  padding: 0;
}`,
    
    "main.js": `console.log("Hello from {{name}}!")`,
    
    "punk.config.json": `{
  "template": "static-site",
  "name": "{{name}}"
}`,
  },
  
  build: {
    command: ["bun", "build", "--outdir=dist", "main.js"],
    output: "dist",
  },
  
  preview: {
    command: ["bun", "--hot", "serve", "."],
    port: 3000,
  },
}
```

### 8.4 react-spa Template

```typescript
// @punk/templates/react-spa/index.ts
export default {
  id: "react-spa",
  name: "React SPA",
  description: "React single-page application with Vite",
  
  files: {
    "index.html": `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{name}}</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>`,
    
    "src/main.tsx": `import React from "react"
import { createRoot } from "react-dom/client"
import { ThemeProvider } from "@punk/tokiforge/react"
import { App } from "./App"
import "@punk/pink/css"

createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <ThemeProvider>
      <App />
    </ThemeProvider>
  </React.StrictMode>
)`,
    
    "src/App.tsx": `export function App() {
  return (
    <div className="pink-container pink-p-4">
      <h1 className="pink-text-2xl pink-font-bold">{{name}}</h1>
      <p>Edit src/App.tsx to get started.</p>
    </div>
  )
}`,
    
    "vite.config.ts": `import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [react()],
})`,
    
    "tsconfig.json": `{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true
  },
  "include": ["src"]
}`,
    
    "package.json": `{
  "name": "{{name}}",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "@punk/pink": "^1.0.0",
    "@punk/tokiforge": "^1.0.0",
    "@effect/schema": "^0.75.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.6.0",
    "vite": "^5.4.0"
  }
}`,
  },
  
  build: {
    command: ["bun", "run", "build"],
    output: "dist",
  },
  
  preview: {
    command: ["bun", "run", "dev"],
    port: 5173,
  },
}
```

### 8.5 api-glyphcase Template

```typescript
// @punk/templates/api-glyphcase/index.ts
export default {
  id: "api-glyphcase",
  name: "API with GlyphCase",
  description: "REST API with SQLite persistence via GlyphCase",
  
  files: {
    "src/index.ts": `import { Schema as S } from "@effect/schema"
import { GlyphCase } from "@punk/glyphcase"

const db = new GlyphCase("./data.db")

// Initialize tables
db.exec(\`
  CREATE TABLE IF NOT EXISTS items (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    value REAL NOT NULL,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
  )
\`)

// Request schemas
const CreateItemRequest = S.Struct({
  name: S.String.pipe(S.minLength(1)),
  value: S.Number,
})

const server = Bun.serve({
  port: 3000,
  
  async fetch(req) {
    const url = new URL(req.url)
    
    // Health check
    if (url.pathname === "/health") {
      return Response.json({ status: "ok" })
    }
    
    // List items
    if (req.method === "GET" && url.pathname === "/items") {
      const items = db.query("SELECT * FROM items ORDER BY created_at DESC").all()
      return Response.json(items)
    }
    
    // Get single item
    if (req.method === "GET" && url.pathname.startsWith("/items/")) {
      const id = url.pathname.split("/")[2]
      const item = db.query("SELECT * FROM items WHERE id = ?").get(id)
      if (!item) return Response.json({ error: "Not found" }, { status: 404 })
      return Response.json(item)
    }
    
    // Create item
    if (req.method === "POST" && url.pathname === "/items") {
      try {
        const body = await req.json()
        const validated = S.decodeSync(CreateItemRequest)(body)
        const id = crypto.randomUUID()
        
        db.query("INSERT INTO items (id, name, value) VALUES (?, ?, ?)")
          .run(id, validated.name, validated.value)
        
        db.log("item.created", { id, ...validated })
        
        return Response.json({ id, ...validated }, { status: 201 })
      } catch (error) {
        return Response.json({ error: "Invalid request" }, { status: 400 })
      }
    }
    
    // Delete item
    if (req.method === "DELETE" && url.pathname.startsWith("/items/")) {
      const id = url.pathname.split("/")[2]
      db.query("DELETE FROM items WHERE id = ?").run(id)
      db.log("item.deleted", { id })
      return Response.json({ deleted: true })
    }
    
    return Response.json({ error: "Not found" }, { status: 404 })
  },
})

console.log(\`API running at http://localhost:\${server.port}\`)`,
    
    "package.json": `{
  "name": "{{name}}",
  "type": "module",
  "scripts": {
    "dev": "bun --hot src/index.ts",
    "start": "bun src/index.ts"
  },
  "dependencies": {
    "@effect/schema": "^0.75.0",
    "@punk/glyphcase": "^1.0.0"
  }
}`,
  },
  
  build: {
    command: ["bun", "build", "--compile", "src/index.ts", "--outfile=server"],
    output: "server",
  },
  
  preview: {
    command: ["bun", "run", "dev"],
    port: 3000,
  },
}
```

### 8.6 cli-tool Template

```typescript
// @punk/templates/cli-tool/index.ts
export default {
  id: "cli-tool",
  name: "CLI Tool",
  description: "Command-line tool with argument parsing",
  
  files: {
    "src/index.ts": `#!/usr/bin/env bun
import { Schema as S } from "@effect/schema"

const Args = S.Struct({
  input: S.String,
  output: S.optional(S.String),
  verbose: S.optional(S.Boolean),
})

function parseArgs(args: string[]) {
  const result: Record<string, unknown> = {}
  
  for (let i = 0; i < args.length; i++) {
    const arg = args[i]
    if (arg === "--input" || arg === "-i") result.input = args[++i]
    else if (arg === "--output" || arg === "-o") result.output = args[++i]
    else if (arg === "--verbose" || arg === "-v") result.verbose = true
    else if (!arg.startsWith("-") && !result.input) result.input = arg
  }
  
  return S.decodeSync(Args)(result)
}

async function main() {
  const args = parseArgs(Bun.argv.slice(2))
  
  if (args.verbose) console.log("Arguments:", args)
  
  const input = await Bun.file(args.input).text()
  const output = input.toUpperCase() // Process here
  
  if (args.output) {
    await Bun.write(args.output, output)
    console.log(\`Written to \${args.output}\`)
  } else {
    console.log(output)
  }
}

main().catch(console.error)`,
    
    "package.json": `{
  "name": "{{name}}",
  "type": "module",
  "bin": { "{{name}}": "./src/index.ts" },
  "scripts": {
    "start": "bun src/index.ts",
    "build": "bun build --compile src/index.ts --outfile={{name}}"
  },
  "dependencies": {
    "@effect/schema": "^0.75.0"
  }
}`,
  },
  
  build: {
    command: ["bun", "build", "--compile", "src/index.ts", "--outfile={{name}}"],
    output: "{{name}}",
  },
}
```

---

## 9. VM Foundation (libkrun)

### 9.1 What Is libkrun?

libkrun is an open-source (Apache 2.0) library for lightweight virtual machines:

- **Library, not daemon** — Link directly, no separate process
- **KVM on Linux** — Uses kernel's hypervisor
- **HVF on macOS** — Uses Apple's Hypervisor.framework
- **~125ms boot** — Fast enough for interactive use
- **<5MB overhead** — Minimal resource consumption

### 9.2 High-Level VM API

```typescript
// @punk/vm-core/vm.ts
import { Schema as S } from "@effect/schema"

export const VMConfig = S.Struct({
  image: S.String,
  resources: S.Struct({
    memory_mb: S.Number.pipe(S.between(128, 8192)),
    cpus: S.Number.pipe(S.between(1, 8)),
    timeout_ms: S.Number.pipe(S.between(1000, 3600000)),
  }),
  network: S.Union(
    S.Literal("none"),
    S.Literal("outbound-only"),
    S.Literal("full")
  ),
  mounts: S.optional(S.Array(S.Struct({
    host: S.String,
    guest: S.String,
    readonly: S.optional(S.Boolean),
  }))),
  env: S.optional(S.Record(S.String, S.String)),
})

export interface VMResult {
  id: string
  exitCode: number
  stdout: string
  stderr: string
  durationMs: number
}

export async function run(config: VMConfig, command: string[]): Promise<VMResult>
```

### 9.3 Pre-built Base Images

| Image | Contents | Size |
|-------|----------|------|
| `punk-base-node` | Alpine + Node 22 LTS + npm | ~50MB |
| `punk-base-bun` | Alpine + Bun 1.x | ~40MB |
| `punk-base-python` | Alpine + Python 3.12 + pip | ~80MB |
| `punk-base-full` | Debian + Node + Python + Bun + build tools | ~200MB |

---

## 10. MCP Server Interface

### 10.1 Tools

| Tool | Description |
|------|-------------|
| `punk.schema.create` | Create and register Effect schema |
| `punk.schema.validate` | Validate data against schema |
| `punk.component.create` | Create component from spec |
| `punk.rig.list` | List available rigs |
| `punk.template.list` | List available starter templates |
| `punk.template.scaffold` | Scaffold project from template |
| `punk.build` | Build current project |
| `punk.preview` | Start dev server |
| `punk.vm.run` | Execute in isolated VM |
| `punk.file.write` | Write file |
| `punk.file.read` | Read file |

### 10.2 MCP Tool Implementations

```typescript
// @punk/vm-mcp/tools.ts

export const tools = {
  "punk.rig.list": {
    description: "List available rigs (component wrappers)",
    inputSchema: S.Struct({}),
    handler: async () => ({
      rigs: [
        { id: "Table", library: "@tanstack/react-table", category: "Data" },
        { id: "Chart", library: "chart.js", category: "Data" },
        { id: "DatePicker", library: "react-day-picker", category: "Form" },
        { id: "Code", library: "@uiw/react-codemirror", category: "Display" },
        { id: "Markdown", library: "marked", category: "Display" },
      ],
    }),
  },
  
  "punk.template.list": {
    description: "List available starter templates",
    inputSchema: S.Struct({}),
    handler: async () => ({
      templates: [
        { id: "static-site", description: "Static HTML/CSS/JS site" },
        { id: "react-spa", description: "React single-page application" },
        { id: "api-glyphcase", description: "REST API with SQLite persistence" },
        { id: "cli-tool", description: "Command-line tool" },
      ],
    }),
  },
  
  "punk.template.scaffold": {
    description: "Scaffold a new project from a starter template",
    inputSchema: S.Struct({
      template: S.String,
      name: S.String.pipe(S.pattern(/^[a-z0-9-]+$/)),
    }),
    handler: async ({ template, name }) => {
      // Create project from template
      return { projectPath: `/projects/${name}`, files: [] }
    },
  },
  
  // ... other tools
}
```

### 10.3 Resources

| URI | Description |
|-----|-------------|
| `punk://schemas` | Registered schemas |
| `punk://components` | Project components |
| `punk://rigs` | Available rigs |
| `punk://templates` | Available templates |
| `punk://project` | Project config |

---

## 11. Platform Support

### 11.1 Platform Matrix

| Platform | VM Backend | Status |
|----------|------------|--------|
| Linux x86_64 | KVM (native) | ✅ Full support |
| Linux ARM64 | KVM (native) | ✅ Full support |
| macOS ARM64 | HVF (native) | ✅ Full support |
| macOS x86_64 | HVF (native) | ⚠️ Works, less tested |
| Windows | WSL2 (nested) | ✅ Full support via shim |

### 11.2 Windows Implementation

Windows uses WSL2, identical to Docker's architecture:

```
Windows User runs: punk-vm run punk-base-node -- node app.js

What actually happens:

Windows Host
    │
    └──► punk-vm.exe (thin shim)
              │
              └──► WSL2 distro (punk-wsl)
                        │
                        └──► libkrun (KVM)
                                  │
                                  └──► Punk VM
```

User types one command. It just works.

---

## 12. Package Structure

```
punk-vm/
├── packages/
│   ├── vm-core/           # libkrun bindings, VM management
│   ├── vm-mcp/            # MCP server implementation
│   ├── vm-cli/            # Command-line interface
│   ├── schema/            # Effect Schema definitions
│   ├── puck/              # Schema renderer
│   ├── garden/            # Component infrastructure
│   ├── pink/              # CSS design system
│   ├── tokiforge/         # Design tokens
│   ├── glyphcase/         # SQLite storage
│   ├── ulid/              # ID generation
│   ├── rigs/              # Core rig implementations
│   │   ├── table/
│   │   ├── chart/
│   │   ├── datepicker/
│   │   ├── code/
│   │   └── markdown/
│   └── templates/         # Starter templates
│       ├── static-site/
│       ├── react-spa/
│       ├── api-glyphcase/
│       └── cli-tool/
├── images/                # OCI base images
└── apps/
    └── punk-vm/           # Main CLI entry point
```

---

## 13. API Reference

### 13.1 VM Core

```typescript
// Run code in isolated VM
function run(config: VMConfig, command: string[]): Promise<VMResult>

// Create long-running VM
function create(config: VMConfig): Promise<VM>

class VM {
  readonly id: string
  exec(command: string[]): Promise<VMResult>
  destroy(): Promise<void>
  readonly stdout: ReadableStream<Uint8Array>
  readonly stderr: ReadableStream<Uint8Array>
}
```

### 13.2 Garden Registry

```typescript
import { useGardenRegistry } from "@punk/garden"

const registry = useGardenRegistry()
registry.get("Chart")              // Get rig component
registry.list()                    // List all rigs
registry.listByCategory("Data")    // Filter by category
```

---

## 14. CLI Reference

```bash
# Run code in VM
punk-vm run <image> -- <command>
punk-vm run punk-base-node -- node -e "console.log('hello')"

# Scaffold from template
punk-vm scaffold <template> <name>
punk-vm scaffold react-spa my-app

# Build project
punk-vm build [path]

# Start preview server
punk-vm preview [path]

# Start MCP server
punk-vm mcp start

# List rigs
punk-vm rigs

# List templates
punk-vm templates
```

---

## 15. Implementation Phases

### Phase 1: Foundation (Weeks 1-4)

- libkrun FFI bindings
- Effect Schema integration
- Basic VM API
- OCI image handling

### Phase 2: Core Primitives (Weeks 5-8)

- Puck renderer
- Garden.js integration
- Pink CSS
- TokiForge, GlyphCase, ULID

### Phase 3: Rigs (Weeks 9-10)

- Table rig
- Chart rig
- DatePicker rig
- Code rig
- Markdown rig

### Phase 4: Templates & MCP (Weeks 11-13)

- 4 starter templates
- MCP server implementation
- Tool handlers

### Phase 5: CLI & Distribution (Weeks 14-16)

- CLI commands
- Platform packaging
- Documentation

---

## 16. Testing Strategy

### Unit Tests

```typescript
import { describe, test, expect } from "bun:test"
import { run } from "@punk/vm-core"

describe("VM execution", () => {
  test("runs simple command", async () => {
    const result = await run(
      { image: "punk-base-node", resources: { memory_mb: 256, cpus: 1, timeout_ms: 10000 }, network: "none" },
      ["node", "-e", "console.log('hello')"]
    )
    expect(result.stdout.trim()).toBe("hello")
  })
})
```

---

## 17. Distribution

### Package Managers

```bash
# npm/bun
npm install -g @punk/vm-lite

# Homebrew (macOS)
brew install punk-vm-lite

# apt (Debian/Ubuntu)
sudo apt install punk-vm-lite
```

### Claude Desktop Config

```json
{
  "mcpServers": {
    "punk": {
      "command": "punk-vm",
      "args": ["mcp", "start"]
    }
  }
}
```

---

## 18. Upgrade Path: Lite → Full

### What Changes

| Aspect | Lite | Full |
|--------|------|------|
| Rigs | 5 bundled | All from Depot |
| Mods | None | Full ecosystem |
| AI | BYOM via MCP | Ska orchestration |
| Storage | Local SQLite | Cloud + Local |
| Templates | 4 starters | Extended catalog |

### Migration

```bash
# Install Mohawk
npm install -g @mohawk/cli

# Import project
mohawk import ./my-punk-vm-project

# Now has access to Depot rigs, mods, Ska
```

---

## 19. Appendix: Code Examples

### Claude Building a Dashboard

```
User: "Build me a sales dashboard"

Claude calls: punk.template.scaffold
{ "template": "react-spa", "name": "sales-dashboard" }

Claude calls: punk.file.write (App.tsx with Chart and Table rigs)

Claude calls: punk.preview
Returns: { "url": "http://localhost:5173" }

Result: Dashboard with Chart and Table components, schema-validated
```

### Schema-Validated Form

```json
{
  "type": "Flex",
  "props": { "direction": "column", "gap": "4" },
  "children": [
    {
      "type": "DatePicker",
      "props": { "placeholder": "Select date", "minDate": "2024-01-01" }
    },
    {
      "type": "Table",
      "props": {
        "columns": [
          { "key": "date", "header": "Date" },
          { "key": "amount", "header": "Amount" }
        ],
        "data": [],
        "sortable": true
      }
    }
  ]
}
```

---

## Summary

### Punk VM Lite

- **7 core primitives**: Effect Schema, Puck, Garden.js, Pink, TokiForge, GlyphCase, ULID
- **2 backends**: None (static), GlyphCase (SQLite)
- **5 core rigs**: Table, Chart, DatePicker, Code, Markdown
- **4 starter templates**: static-site, react-spa, api-glyphcase, cli-tool
- **Hardware isolation**: libkrun microVMs
- **MCP interface**: Connect any LLM
- **License**: Apache 2.0

### Punk VM Full (Mohawk)

- Everything in Lite, plus:
- **Cloud backends**: Encore.ts, tRPC, Neon, Manifest
- **Backend mods**: Supabase, Firebase, PocketBase, Appwrite
- **Depot integration**: 90+ additional rigs on demand
- **Trinity Runtime**: Mod execution
- **Nitro**: Process supervision
- **Ska**: AI orchestration
- **Mods**: Extensible ecosystem
- **Cloud features**: Multi-tenant, collaboration

### Key Principle

**"Lite is Punk. Full is Punk at scale."**

---

*End of Implementation Guide*
