# Punk Core

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Philosophy

**"Don't make AI smarter—make the output space safer."**

Traditional AI code generators produce probabilistic "best guesses" requiring manual auditing. Punk inverts the paradigm: AI generates validated JSON schemas that deterministic renderers transform into production-ready applications.

The output is correct by construction.

---

## Core Tenets

1. **Schemas over code** — AI generates configuration, not implementation
2. **Validation over hope** — Every schema passes @effect/schema before rendering
3. **Determinism over probability** — Same input = same output, every time
4. **Constraints as features** — Limitations on output are guarantees for users
5. **Accessibility by construction** — A11y is structural, not behavioral

---

## The Three Tiers

```
┌─────────────────────────────────────────────────────────────────┐
│                    ATOMPUNK (Tier 3)                            │
│              Full-Stack AI Generation                           │
│   • Backend code generation    • Database schemas               │
│   • Auth scaffolding           • Deployment config              │
├─────────────────────────────────────────────────────────────────┤
│                    SYNTHPUNK (Tier 2)                           │
│              AI-Powered UI Generation                           │
│   • Ska engine                 • Revision history               │
│   • Natural language → schemas • Context management             │
├─────────────────────────────────────────────────────────────────┤
│                      PUNK (Tier 1)                              │
│              Deterministic Rendering                            │
│   • Schema → React renderer    • @effect/schema validation      │
│   • Type safety                • Accessibility                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tier 1: Punk (Free/Open Source)

The core rendering engine. Takes JSON schemas, validates them, outputs accessible React components.

**Provides:**
- `@punk/core` — Schema renderer
- @effect/schema validation with typed error recovery
- TypeScript type safety
- WCAG 2.1 Level AA compliance
- Deterministic output

**Example:**
```typescript
import { PunkRenderer } from '@punk/core'

const schema = {
  type: 'button',
  props: {
    variant: 'primary',
    children: 'Submit Form',
    'aria-label': 'Submit the registration form'
  }
}

<PunkRenderer schema={schema} />
```

### Tier 2: Synthpunk

AI-powered schema generation layer.

**Provides:**
- Ska AI engine
- Natural language → validated schemas
- Revision history
- Context management across iterations

### Tier 3: Atompunk

Full-stack generation including backend.

**Provides:**
- Backend code generation via vetted templates
- Database schema generation
- Authentication scaffolding
- Deployment configuration

**Security model:** AI fills parameters in pre-vetted templates, not raw code. Templates include input validation, CSRF protection, rate limiting, SQL injection prevention, XSS protection.

---

## Rendering Stack

```
Ska (AI)
 ↓
@effect/schema (Validation + Repair Loop)
 ↓
PunkDocument
 ↓
Puck (Schema Renderer)
 ↓
Rigs (via punk wrap)
 ↓
Base UI (Behavior) + Pink (Appearance)
 ↓
TokiForge (Theming)
 ↓
Accessible, Secure, Deterministic UI
```

### Layer Responsibilities

| Layer | Responsibility | Does NOT Do |
|-------|---------------|-------------|
| **Ska** | Convert intent to schema | Render UI, validate, style |
| **@effect/schema** | Validate schema structure, typed errors, repair loop | Generate schemas, render |
| **Puck** | Schema → React component tree | Accessibility logic, styling decisions |
| **Rigs** | Extended component library (Chart, DataTable, etc.) | Core rendering |
| **Base UI** | Accessible behavior primitives | Visual styling |
| **Pink** | CSS design system classes | Component logic, theming values |
| **TokiForge** | Token values + runtime theming | Component structure, behavior |

**Key principle:** No layer reaches into another's domain.

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Intent                             │
│    "Create a task manager with add, delete, complete actions"   │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│                         Ska (AI Engine)                         │
│              SKA-30B-VL / SKA-106B-VL / SKA-235B-VL             │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓ JSON Schema
┌─────────────────────────────────────────────────────────────────┐
│                    @effect/schema Validation                    │
│         Invalid schemas trigger structured repair loop          │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓ Validated PunkDocument
┌─────────────────────────────────────────────────────────────────┐
│                         Puck Renderer                           │
│                    Schema → React Components                    │
└─────────────────────────────────┬───────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Production Application                     │
│              Accessible, Secure, Deterministic                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Validation & Recovery

Punk uses @effect/schema as its exclusive validation layer. When AI generation produces invalid output, the system enters a structured repair loop:

1. **Generate** — Ska produces a JSON candidate
2. **Validate** — @effect/schema checks structure with typed errors
3. **Repair** — If invalid, specific errors feed back to Ska via repair prompt
4. **Retry** — Up to 3 attempts with increasing context
5. **Fallback** — Safe skeleton template (never fails)

This ensures users never see "Generation Failed" — they always get a usable result.

---

## Behavioral Primitives

Base UI is the default behavioral primitive layer for all new components. It provides:

- Unstyled, accessible components
- Built-in form validation system
- Keyboard navigation and focus management
- ARIA attributes handled automatically

Legacy components may use Radix Primitives via the `@punk/ui-primitives` shim. The shim provides a unified API with build-time backend selection:

```typescript
// punk.config.ts
export default defineConfig({
  ui: {
    primitives: 'base-ui',  // 'base-ui' (default) | 'radix' (legacy)
  },
})
```

---

## Packages

| Component | Package |
|-----------|---------|
| Core renderer | `@punk/core` |
| Synthpunk (AI) | `@punk/synthpunk` |
| Extended Rigs | `@punk/extended` |
| Punk CLI | `@punk/cli` |
| UI Primitives (shim) | `@punk/ui-primitives` |
| TokiForge Core | `@tokiforge/core` |
| TokiForge React | `@tokiforge/react` |
| Pink CSS | `@appwrite.io/pink` |
| Pink Icons | `@appwrite.io/pink-icons` |
| Base UI | `@base-ui-components/react` |
| Effect | `effect` |
| Effect Schema | `@effect/schema` |

---

*Last updated: December 2025*
