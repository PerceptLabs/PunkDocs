# Punk Ska

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Overview

Ska is the AI engine powering Punk. It is vision-first, agentic-capable, and designed for deterministic schema generation rather than raw code output.

---

## Model Tiers

| Tier | Model | Target Users |
|------|-------|--------------|
| **Entry** | SKA-30B-VL | Hobbyists, students, local builders |
| **Mid** | SKA-106B-VL | Independent devs, small teams |
| **High** | SKA-235B-VL | Studios, startups, power users |

All tiers share the same schema protocol, tools, and agentic interfaces.

---

## Core Capabilities

### Vision-Semantic Interpretation

Ska treats visual input as first-class:
- Parse design references (screenshots, wireframes, sketches)
- Understand layout hierarchies
- Convert images into Punk schema components
- Detect style patterns (glassmorphism, minimalist, retro, etc.)

### Multi-Step Architectural Reasoning

Structured iterative cycles:
1. Interpret request (natural language + visual)
2. Plan architecture
3. Build schema
4. Validate deterministically
5. Generate UI + logic
6. Polish with stylistic alignment

### Rig and Mod Awareness

Ska can:
- Invoke rig packs
- Suggest optimal mods
- Compose multi-rig builds
- Detect missing dependencies
- Suggest upgrades for better components

---

## Implementation

### Vercel AI SDK

Ska is implemented using Vercel AI SDK, which provides:
- Native @effect/schema support for direct PunkSchema validation
- Streaming structured objects for real-time UI generation feedback
- Multi-provider abstraction for easy model swapping
- Vision input support for image-to-UI generation
- End-to-end TypeScript type safety

### Core Integration

```typescript
import { generateObject } from 'ai'
import { createOllama } from 'ollama-ai-provider'
import { PunkDocumentSchema } from '@punk/core/schema'

const ollama = createOllama()

const { object } = await generateObject({
  model: ollama('ska-30b-vl'),
  schema: PunkDocumentSchema,
  prompt: 'Create a task manager with add, delete, complete actions',
})
```

### Provider Configuration

```typescript
// Entry tier (local)
const entryProvider = createOllama()
const entryModel = entryProvider('ska-30b-vl')

// Mid tier (cloud)
const midProvider = createOpenAICompatible({ baseURL: '...' })
const midModel = midProvider('ska-106b-vl')

// High tier (cloud)
const highProvider = createOllama()
const highModel = highProvider('ska-235b-vl')
```

---

## Schema Generation

Ska generates validated JSON schemas, not code:

```
User: "Create a dashboard with sales chart and user table"
         ↓
Ska: {
  "type": "Container",
  "children": [
    {
      "type": "Chart",
      "props": { "chartType": "bar", "title": "Sales" }
    },
    {
      "type": "DataTable",
      "props": { "columns": ["name", "email", "role"] }
    }
  ]
}
         ↓
@effect/schema validates → Puck renders → Accessible UI
```

---

## Upgrade Suggestions

When Ska determines a better Rig exists but user lacks access:

```json
{
  "document": { "/* generated schema */" },
  "upgradeSuggestions": [{
    "current": "TextArea",
    "suggested": "RichText",
    "reason": "Rich text editing with formatting, mentions, embeds",
    "access": { "tier": "pro", "purchasable": 2900 }
  }]
}
```

---

## Context Management

Ska maintains context across iterations:

```
v1 - Initial structure: "Create task manager"
v2 - Refinement: "Add a date picker to each task"  
v3 - Enhancement: "Add dark mode support"
```

Each version is stored, comparable, and restorable.

---

## Mohawk Pricing Tiers

| Tier | Price | AI Model |
|------|-------|----------|
| **Free** | $0 | SKA-30B-VL (100 builds/month) |
| **Starter** | $15/mo | SKA-30B-VL (200 builds) |
| **Pro** | $39/mo | SKA-106B-VL (350 builds) |
| **Pro+** | $69/mo | SKA-235B-VL (500 builds) |
| **Team** | $29/seat/mo | SKA-106B-VL (250 builds/seat) |
| **Agency** | Custom | Dedicated endpoints |

---

*Last updated: December 2025*
