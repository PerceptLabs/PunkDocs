# Punk Mod Implementation

**Version:** 2.0  
**Status:** Canonical Source of Truth

---

## Overview

This document covers the full implementation details of the Mod system, including manifest schemas, the template/slot system, Trinity Runtime, testing, and publishing workflows.

---

## 1. Mod Types

### 1.1 Backend Mods

Generate code that integrates external services into the user's project.

**Examples:** Supabase, Firebase, Appwrite, PocketBase

**Flow:**
1. Introspection — Fetch schema from external service
2. Constraint Population — Build enums from real data
3. LLM Generation — AI fills slot values (constrained)
4. Template Expansion — Pre-written templates + validated slots → code
5. Validation — Slots validated before expansion
6. Output — Write files to user's project

### 1.2 Runtime Mods

Process data inside Trinity at app runtime.

**Examples:** PDF generation, image processing, data transformation

**Flow:**
1. App calls capability
2. Trinity loads mod
3. Data validation
4. Processing (WASM, Lua, JS)
5. Return result

---

## 2. Manifest Schema

### 2.1 Full Manifest

```json
{
  "$schema": "https://punk.dev/schemas/mod-manifest-v2.json",
  "id": "supabase",
  "name": "Supabase",
  "version": "1.0.0",
  "description": "Supabase PostgreSQL backend integration",
  "author": {
    "name": "Punk Labs",
    "email": "mods@punk.dev",
    "url": "https://punk.dev"
  },
  "license": "MIT",
  "repository": "https://github.com/PerceptLabs/punk-mod-supabase",
  
  "type": "backend",
  
  "capabilities": {
    "requires": [
      "ska:generate",
      "storage:write"
    ],
    "provides": [
      "supabase:query",
      "supabase:auth",
      "supabase:storage",
      "supabase:realtime"
    ]
  },
  
  "config": {
    "projectUrl": {
      "type": "string",
      "description": "Supabase project URL",
      "pattern": "^https://[a-z0-9]+\\.supabase\\.co$",
      "required": true
    },
    "anonKey": {
      "type": "string",
      "secret": true,
      "required": true
    },
    "serviceRoleKey": {
      "type": "string",
      "secret": true,
      "required": false
    }
  },
  
  "network": {
    "allowlist": [
      "*.supabase.co",
      "*.supabase.net"
    ]
  },
  
  "tier": "pro",
  
  "entry": "js/index.js",
  
  "templates": {
    "client": "templates/client/",
    "queries": "templates/queries/",
    "auth": "templates/auth/"
  },
  
  "knowledge": [
    "knowledge/usage.md",
    "knowledge/examples/"
  ]
}
```

### 2.2 Runtime Mod Manifest (PDF Example)

```json
{
  "id": "pdf-generator",
  "name": "PDF Generator",
  "version": "1.0.0",
  "description": "Generate PDFs from templates and data",
  "type": "runtime",
  
  "capabilities": {
    "requires": ["storage:read"],
    "provides": ["pdf:generate", "pdf:merge", "pdf:split"]
  },
  
  "config": {
    "defaultFont": {
      "type": "string",
      "default": "inter"
    },
    "defaultPageSize": {
      "type": "string",
      "enum": ["A4", "Letter", "Legal"],
      "default": "A4"
    }
  },
  
  "entry": "js/index.js",
  
  "wasm": ["wasm/pdf-lib.wasm"]
}
```

---

## 3. Backend Mod Architecture

### 3.1 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND MOD FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. INTROSPECTION                                          │   │
│  │                                                           │   │
│  │    SDK Types (.d.ts)  ─┐                                  │   │
│  │                        ├──►  API Graph                    │   │
│  │    User's Schema      ─┘     (methods, types, chains)     │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2. CONSTRAINT POPULATION                                  │   │
│  │                                                           │   │
│  │    API Graph  ──►  Slot Schemas with Enums               │   │
│  │                                                           │   │
│  │    table: { enum: ["users", "tasks", "projects"] }       │   │
│  │    columns: { enum: ["id", "name", "email"] }            │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 3. LLM GENERATION (Constrained)                          │   │
│  │                                                           │   │
│  │    User Request ──►  LLM  ──►  Slot Values               │   │
│  │                        │                                  │   │
│  │    Constraints:        │                                  │   │
│  │    - Must use enum values only                           │   │
│  │    - Must match schema shape                             │   │
│  │    - Temperature = 0                                     │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 4. TEMPLATE EXPANSION                                     │   │
│  │                                                           │   │
│  │    Template (.hbs)  +  Slot Values  ──►  Code            │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 5. OUTPUT                                                 │   │
│  │                                                           │   │
│  │    Write to user's project:                              │   │
│  │    └── src/lib/supabase/                                 │   │
│  │        ├── client.ts                                     │   │
│  │        ├── queries.ts                                    │   │
│  │        └── types.ts                                      │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Introspector

```javascript
// js/introspect.js
export async function introspect(config) {
  // Fetch schema from Supabase Management API
  const response = await fetch(
    `${config.projectUrl}/rest/v1/`,
    {
      method: 'OPTIONS',
      headers: { 'apikey': config.anonKey }
    }
  )
  
  // Parse OpenAPI spec
  const spec = await response.json()
  
  return {
    tables: extractTables(spec),
    types: extractTypes(spec),
    relationships: extractRelationships(spec)
  }
}
```

### 3.3 Code Generator

```javascript
// js/codegen.js
export async function generate(templateName, slots) {
  // 1. Load slot schema
  const schema = await loadSlotSchema(templateName)
  
  // 2. Validate slots against schema
  const validation = validateSlots(slots, schema)
  if (!validation.valid) {
    throw new SlotValidationError(validation.errors)
  }
  
  // 3. Load template
  const template = await loadTemplate(templateName)
  
  // 4. Expand template (slots are validated, template is pre-tested)
  const code = template(slots)
  
  return code
}
```

---

## 4. Template System

### 4.1 Template Structure

```
templates/
├── client/
│   └── supabase-client.ts.hbs    # SDK client setup
├── queries/
│   ├── select.ts.hbs             # SELECT queries
│   ├── insert.ts.hbs             # INSERT mutations
│   ├── update.ts.hbs             # UPDATE mutations
│   └── delete.ts.hbs             # DELETE mutations
├── auth/
│   ├── sign-in.ts.hbs            # Sign in function
│   ├── sign-up.ts.hbs            # Sign up function
│   └── hooks.ts.hbs              # Auth React hooks
├── storage/
│   ├── upload.ts.hbs             # File upload
│   └── download.ts.hbs           # File download
└── realtime/
    └── subscribe.ts.hbs          # Realtime subscriptions
```

### 4.2 Code Template Example

```handlebars
{{!-- templates/queries/select.ts.hbs --}}
{{!-- 
  Template: Select Query
  Slots: functionName, table, columns, filter?, orderBy?, limit?
--}}
import { supabase } from './supabase-client'
import type { Database } from './database.types'

type {{pascalCase table}}Row = Database['public']['Tables']['{{table}}']['Row']

/**
 * {{description}}
 * @generated by supabase mod
 */
export async function {{functionName}}(
{{#if filter}}
  {{filter.param}}: {{filter.paramType}}
{{/if}}
): Promise<{{pascalCase table}}Row[]> {
  const { data, error } = await supabase
    .from('{{table}}')
    .select('{{join columns ", "}}')
{{#if filter}}
    .{{filter.operator}}('{{filter.column}}', {{filter.param}})
{{/if}}
{{#if orderBy}}
    .order('{{orderBy.column}}', { ascending: {{orderBy.ascending}} })
{{/if}}
{{#if limit}}
    .limit({{limit}})
{{/if}}

  if (error) {
    throw new Error(`Failed to fetch {{table}}: ${error.message}`)
  }

  return data ?? []
}
```

### 4.3 Custom Helpers

```javascript
// js/codegen.js
import Handlebars from 'handlebars'

// String transformations
Handlebars.registerHelper('pascalCase', (str) => 
  str.split(/[-_]/).map(s => s.charAt(0).toUpperCase() + s.slice(1)).join('')
)

Handlebars.registerHelper('camelCase', (str) =>
  str.split(/[-_]/).map((s, i) => 
    i === 0 ? s.toLowerCase() : s.charAt(0).toUpperCase() + s.slice(1)
  ).join('')
)

Handlebars.registerHelper('snakeCase', (str) =>
  str.replace(/([A-Z])/g, '_$1').toLowerCase().replace(/^_/, '')
)

// Array helpers
Handlebars.registerHelper('join', (arr, sep) => arr.join(sep))
Handlebars.registerHelper('first', (arr) => arr[0])
Handlebars.registerHelper('last', (arr) => arr[arr.length - 1])

// Conditional helpers
Handlebars.registerHelper('eq', (a, b) => a === b)
Handlebars.registerHelper('ne', (a, b) => a !== b)

// Type helpers
Handlebars.registerHelper('tsType', (sqlType) => {
  const map = {
    'text': 'string',
    'varchar': 'string',
    'int4': 'number',
    'int8': 'number',
    'float8': 'number',
    'bool': 'boolean',
    'timestamp': 'string',
    'timestamptz': 'string',
    'uuid': 'string',
    'json': 'unknown',
    'jsonb': 'unknown'
  }
  return map[sqlType] || 'unknown'
})
```

### 4.4 Data Templates (Runtime Mods)

```json
{
  "id": "invoice",
  "name": "Invoice Template",
  "version": "1.0.0",
  "pageSize": "A4",
  
  "dataSchema": {
    "type": "object",
    "required": ["invoiceNumber", "customer", "items", "total"],
    "properties": {
      "invoiceNumber": { "type": "string" },
      "customer": { "type": "object" },
      "items": { "type": "array" },
      "total": { "type": "number" }
    }
  },
  
  "layout": {
    "header": {
      "elements": [
        {
          "type": "text",
          "content": "{{company.name}}",
          "x": 50,
          "y": 50,
          "fontSize": 24
        }
      ]
    },
    "body": {
      "elements": [
        {
          "type": "table",
          "data": "{{items}}",
          "columns": [
            { "header": "Description", "field": "description" },
            { "header": "Amount", "field": "amount", "format": "currency" }
          ]
        }
      ]
    }
  }
}
```

---

## 5. Slot Validation

### 5.1 How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    SLOT VALIDATION FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. AI CALLS CAPABILITY                                         │
│     supabase:query({                                            │
│       type: 'select',                                           │
│       table: 'users',        ← Slot value                       │
│       columns: ['id', 'name'] ← Slot value                      │
│     })                                                          │
│                                                                  │
│  2. MOD VALIDATES SLOTS                                         │
│     ├── Is 'users' in enum of tables? ✓                         │
│     ├── Are 'id', 'name' in enum of columns? ✓                  │
│     └── Do values match schema patterns? ✓                      │
│                                                                  │
│  3. MOD EXPANDS TEMPLATE                                        │
│     Pre-written template + validated slots → code               │
│                                                                  │
│  4. OUTPUT                                                      │
│     Generated code is guaranteed valid because:                 │
│     ├── Template was tested by mod author                       │
│     └── Slot values were validated against schema               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Validation Layers

| Layer | What It Validates |
|-------|-------------------|
| **JSON Schema** | Basic types, patterns, required fields |
| **Introspection Enums** | Values exist in actual external service |
| **Template Testing** | Template author's test suite |

### 5.3 Slot Schema Example

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "SelectQuerySlots",
  "type": "object",
  "required": ["functionName", "table", "columns"],
  
  "properties": {
    "functionName": {
      "type": "string",
      "pattern": "^[a-z][a-zA-Z0-9]*$",
      "minLength": 3,
      "maxLength": 50
    },
    
    "table": {
      "type": "string",
      "_enumFrom": "introspection.tables"
    },
    
    "columns": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "uniqueItems": true,
      "_enumFrom": "introspection.tables[table].columns"
    },
    
    "filter": {
      "type": "object",
      "properties": {
        "column": {
          "type": "string",
          "_enumFrom": "introspection.tables[table].columns"
        },
        "operator": {
          "type": "string",
          "enum": ["eq", "neq", "gt", "gte", "lt", "lte", "like", "ilike", "in", "is"]
        },
        "param": {
          "type": "string",
          "pattern": "^[a-z][a-zA-Z0-9]*$"
        },
        "paramType": {
          "type": "string",
          "enum": ["string", "number", "boolean", "string[]", "number[]"]
        }
      },
      "required": ["column", "operator", "param"]
    },
    
    "orderBy": {
      "type": "object",
      "properties": {
        "column": { "type": "string" },
        "ascending": { "type": "boolean", "default": true }
      }
    },
    
    "limit": {
      "type": "integer",
      "minimum": 1,
      "maximum": 1000
    }
  }
}
```

### 5.4 Why No Tree-sitter?

| Concern | How It's Handled |
|---------|------------------|
| Template syntax | Tested by mod author at dev time |
| Slot values | Validated against JSON Schema + enums |
| Invalid combinations | Impossible if schema enums are correct |
| Edge cases | Mod author's test suite catches them |

**Tree-sitter is optional** — useful during mod development, not needed at runtime.

---

## 6. Runtime Mod Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    RUNTIME MOD FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. APP CALLS CAPABILITY                                        │
│     const pdf = await mod.request('pdf:generate', {             │
│       template: 'invoice',                                      │
│       data: invoiceData                                         │
│     })                                                          │
│                                                                  │
│  2. TRINITY LOADS MOD                                           │
│     Opens SQLar archive                                         │
│     Loads WASM modules                                          │
│     Initializes JavaScript environment                          │
│                                                                  │
│  3. DATA VALIDATION                                             │
│     Load template data schema                                   │
│     Validate input data against schema                          │
│     Reject invalid data early                                   │
│                                                                  │
│  4. PROCESSING                                                  │
│     WASM module executes (pdf-lib, image processing)            │
│     Data transformed according to template                      │
│     Output generated in memory                                  │
│                                                                  │
│  5. RETURN RESULT                                               │
│     Return binary blob, JSON, or transformed data               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Trinity Runtime

### 7.1 Components

| Runtime | Role | Use Case |
|---------|------|----------|
| **Lua** | Scripting & orchestration | Configuration, glue code |
| **Txiki.js** | I/O & networking | HTTP, file ops, async |
| **WAMR** | WebAssembly performance | Image processing, encryption |

### 7.2 Desktop Variant

| Standard | Desktop (Electron + Bun) |
|----------|--------------------------|
| Lua | wasmoon (Lua 5.4 via WASM) |
| Txiki.js | Native JS (Bun) |
| WAMR | bun:wasm |

### 7.3 Mod API Surface

```typescript
interface PunkModAPI {
  // Database (scoped to mod's namespace)
  db: {
    query(sql: string, params?: any[]): any[]
    execute(sql: string, params?: any[]): void
    transaction(fn: () => void): void
  }

  // File system (sandboxed to allowed paths)
  fs: {
    read(path: string): Uint8Array
    write(path: string, data: Uint8Array): void
    exists(path: string): boolean
    list(path: string): string[]
  }

  // HTTP (restricted to manifest-declared domains)
  http: {
    get(url: string, options?: RequestOptions): Promise<Response>
    post(url: string, body: any, options?: RequestOptions): Promise<Response>
  }

  // UI integration
  ui: {
    showNotification(message: string, type?: 'info' | 'success' | 'error'): void
    requestInput(prompt: string): Promise<string>
    updateProgress(percent: number, message?: string): void
  }

  // Schema access
  schema: {
    getCurrent(): Schema
    updateComponent(id: string, props: Partial<ComponentProps>): void
    addComponent(parentId: string, component: Schema): string
  }

  // Utilities
  ulid: () => string
  log: (...args: any[]) => void
}
```

---

## 8. Knowledge System

### 8.1 Purpose

Knowledge docs teach Ska how to use the mod's capabilities.

### 8.2 Structure

```markdown
---
tags: [query, select, database]
---

# Querying Data

To query data from Supabase, call `supabase:query` with:
- `type`: Query type ('select', 'insert', 'update', 'delete')
- `table`: Table name
- `columns`: Columns to select

## Example

supabase:query({
  type: 'select',
  table: 'users',
  columns: ['id', 'name', 'email'],
  filter: { column: 'active', operator: 'eq', value: true }
})
```

### 8.3 Best Practices

| Practice | Description |
|----------|-------------|
| **One topic per file** | Better retrieval |
| **Include examples** | Working, tested examples |
| **Use frontmatter tags** | Enable search |
| **Keep updated** | Sync with SDK changes |

---

## 9. Testing

### 9.1 Template Tests

```typescript
// test/templates/select.test.ts
import { generate } from '../js/codegen'
import { describe, it, expect } from 'bun:test'

describe('select template', () => {
  it('generates valid TypeScript', async () => {
    const code = await generate('select', {
      functionName: 'getUsers',
      table: 'users',
      columns: ['id', 'name']
    })
    
    // Should compile
    expect(() => Bun.Transpiler.transform(code, 'ts')).not.toThrow()
  })
  
  it('rejects invalid slot values', async () => {
    await expect(generate('select', {
      functionName: '123invalid',
      table: 'users',
      columns: []
    })).rejects.toThrow('SlotValidationError')
  })
})
```

### 9.2 Integration Tests

```typescript
// test/integration/supabase.test.ts
import { loadMod, executeMod } from '@punk/trinity'

describe('supabase mod', () => {
  it('generates working client', async () => {
    const mod = await loadMod('./supabase.sqlar')
    
    const result = await executeMod(mod, 'supabase:query', {
      type: 'select',
      table: 'users',
      columns: ['id']
    })
    
    expect(result.files).toContain('src/lib/supabase/client.ts')
  })
})
```

---

## 10. Publishing

### 10.1 CLI Workflow

```bash
# Build the .sqlar archive
punk mod build

# Validate manifest and templates
punk mod validate

# Test all templates
punk mod test

# Publish to Depot
punk mod publish
```

### 10.2 Depot Metadata

```json
{
  "id": "supabase",
  "name": "Supabase",
  "version": "1.0.0",
  "downloads": 12500,
  "rating": 4.8,
  "tier": "pro",
  "verified": true,
  "author": {
    "name": "Punk Labs",
    "verified": true
  },
  "tags": ["backend", "database", "auth", "storage"],
  "compatibility": {
    "punk": ">=2.0.0",
    "mohawk": ">=1.5.0"
  }
}
```

---

*Last updated: December 2024*
