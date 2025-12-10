# Punk Architectural Decisions

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Overview

This document records the key architectural decisions made in the Punk Framework, including the rationale behind each choice and alternatives considered.

**Amendments:** Some ADRs have been amended to reflect architectural evolution. Amendments are marked with dates.

---

## ADR-001: Schema-First Architecture

### Decision
Punk uses JSON schemas as the intermediate representation between AI output and rendered UI.

### Context
Traditional AI code generators produce raw HTML/CSS/JS, which requires manual auditing for accessibility, security, and consistency.

### Rationale
- **Validation boundary**: Schemas create a hard boundary between AI output and rendering
- **Determinism**: Same schema always produces same UI
- **Auditability**: Schemas are declarative and reviewable
- **Constraints**: Invalid configurations are structurally impossible

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Raw code generation | Unbounded output space, no guarantees |
| AST manipulation | Too complex, still allows invalid states |
| Template interpolation | Limited expressiveness, still allows arbitrary content |

### Consequences
- AI must be trained/prompted to output valid schemas
- All components must have corresponding schema definitions
- Custom components require schema registration

### Amendment (December 2025)
Schema validation migrated from Zod to @effect/schema. See **ADR-015** for rationale.

---

## ADR-002: Puck as Schema Renderer

### Decision
Use [Puck](https://github.com/measuredco/puck) as the core schema-to-React rendering engine.

### Context
Need a battle-tested library that maps JSON configurations to React component trees.

### Rationale
- **Visual editing**: Puck provides drag-and-drop editing out of the box
- **Extensible**: Clean plugin architecture for custom components
- **Production-ready**: Used in production by multiple companies
- **Open source**: MIT licensed, no vendor lock-in

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Builder.io | Proprietary, cloud-dependent |
| GrapesJS | DOM-based, not React-native |
| Custom renderer | Maintenance burden, wheel reinvention |

### Consequences
- Component model follows Puck conventions
- Visual editing comes free
- Schema format must align with Puck expectations

---

## ADR-003: Radix for Accessibility — SUPERSEDED

### Decision
~~Use Radix Primitives as the behavioral foundation for all interactive components.~~

### Amendment (December 2025) — SUPERSEDED by ADR-016

This ADR is **superseded**. **Base UI** is now the default behavioral foundation, with Radix available via compatibility shim for legacy projects. See **ADR-016** for rationale.

---

## ADR-004: Pink CSS Design System

### Decision
Use [Appwrite Pink](https://pink.appwrite.io/) as the CSS design system.

### Context
Need a framework-agnostic styling solution that works with behavioral primitives.

### Rationale
- **Framework-agnostic**: CSS classes, not React components
- **Comprehensive**: Full design system with tokens
- **Well-designed**: Clean, professional aesthetic
- **Accessible**: Designed with a11y in mind

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Tailwind | Utility classes, not semantic |
| Shadcn/ui | Locked to React, copies code |
| Material UI | Heavy, opinionated aesthetic |
| Custom CSS | Maintenance burden |

### Consequences
- Components use Pink class names
- Theming happens through Pink's token system
- Must bridge Pink styles to behavioral primitives

---

## ADR-005: Garden.js for Rig Registry — SUPERSEDED

### Decision
~~Use Garden.js as the component infrastructure layer for Rigs.~~

### Amendment (December 2025) — SUPERSEDED by ADR-017

This ADR is **superseded**. Garden.js adds no value to the Punk/Mohawk architecture:
- Puck already provides component preview
- Multi-framework support is unnecessary (React-only stack)
- Registry is ~30 lines of custom code

Replaced by `punk wrap` CLI. See **ADR-017**.

---

## ADR-006: TokiForge for Theming

### Decision
Build TokiForge as the design token engine with runtime theme switching.

### Context
Need dynamic theming that validates WCAG contrast compliance and works with Pink CSS.

### Rationale
- **Runtime switching**: Themes change without rebuild
- **Validation**: WCAG contrast checking built-in
- **Token-based**: Aligns with modern design system practices
- **Pink integration**: Designed to work with Pink CSS

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| CSS variables only | No validation, no type safety |
| styled-components theming | Framework-locked |
| Tailwind themes | Config-based, not runtime |

### Consequences
- All color/spacing/typography through TokiForge tokens
- Theme validation happens at token level
- Runtime performance considerations for large apps

---

## ADR-007: ULID for Identifiers

### Decision
Use ULIDs (Universally Unique Lexicographically Sortable Identifiers) for all entity IDs.

### Context
Need unique identifiers that work across distributed systems and provide useful properties.

### Rationale
- **Sortable**: Lexicographic sorting = chronological sorting
- **Timestamp embedded**: Creation time extractable without DB lookup
- **URL-safe**: No encoding needed
- **Distributed**: No coordination required between nodes

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| UUID v4 | Not sortable, random distribution hurts B-tree performance |
| Auto-increment | Requires coordination, leaks information |
| Snowflake IDs | More complex, requires node ID management |
| nanoid | Not sortable, no timestamp |

### Consequences
- All IDs are 26 characters
- Timestamps can be extracted from IDs
- Database indexes benefit from sequential nature

---

## ADR-008: SQLar for Mod Packaging

### Decision
Package Mods as SQLite Archive (.sqlar) files.

### Context
Need a single-file distribution format for Mods that contains code, templates, knowledge docs, and metadata.

### Rationale
- **Single file**: One file to download, install, manage
- **Queryable**: SQLite tools can inspect contents
- **Atomic**: Transactional operations, no partial installs
- **Standard**: SQLite is everywhere, well-understood

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| ZIP | Not queryable, no atomic operations |
| npm package | Requires npm infrastructure |
| Custom binary | No tooling, maintenance burden |
| Directory | Multiple files, harder to distribute |

### Consequences
- Mods are SQLite databases with specific schema
- Standard SQLite tools work for inspection
- Trinity runtime opens and queries the archive

---

## ADR-009: Trinity Runtime for Mod Execution

### Decision
Create Trinity, a polyglot runtime combining Lua, JavaScript (Txiki.js), and WebAssembly (WAMR).

### Context
Mods need to execute code in a sandboxed environment with access to I/O and performance-critical operations.

### Rationale
- **Lua**: Lightweight scripting, easy to sandbox
- **Txiki.js**: I/O and networking capabilities
- **WAMR**: Performance-critical code (image processing, PDF generation)
- **Separation**: Each runtime handles what it's best at

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Node.js only | Hard to sandbox properly |
| Deno | Still maturing, heavier |
| WASM only | Poor for I/O and scripting |
| Lua only | No performance path, limited I/O |

### Consequences
- Mods can use any of three languages
- Sandboxing must be implemented for each runtime
- Desktop variant uses wasmoon for Lua, Bun for JS

---

## ADR-010: Vercel AI SDK for AI Integration

### Decision
Use Vercel AI SDK as the implementation infrastructure for Ska (the AI engine).

### Context
Need a unified interface to multiple AI providers with native schema support.

### Rationale
- **Native schema support**: Direct @effect/schema validation integration
- **Streaming**: Real-time generation feedback
- **Multi-provider**: OpenAI, Anthropic, Ollama, etc.
- **TypeScript-first**: End-to-end type safety

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Direct API calls | No abstraction, provider-locked |
| LangChain | Heavier, Python-focused |
| Custom SDK | Maintenance burden |
| Strands Agents SDK | Unnecessary abstraction |

### Consequences
- Provider switching is configuration change
- Streaming structured objects work out of the box
- Schema validation happens at SDK level

### Amendment (December 2025)
Updated to reflect @effect/schema integration (from Zod).

---

## ADR-011: Electron + Bun for Desktop

### Decision
Build Mohawk Desktop on Electron with Bun as the main process runtime.

### Context
Need a cross-platform desktop application with access to local filesystem and native performance.

### Rationale
- **Electron**: Mature, proven for desktop apps
- **Bun**: Fast startup, native SQLite, WASM support
- **Shared code**: Same React UI as web version
- **Native access**: File system, local AI inference

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Tauri | Smaller ecosystem, less mature |
| Qt/QML | Different tech stack, no code sharing |
| Electron + Node | Slower, separate SQLite binding needed |

### Consequences
- Desktop app shares UI code with web
- Bun's SQLite used for GlyphCase
- Cross-platform builds via electron-builder

---

## ADR-012: Slot-Based Template Expansion

### Decision
Backend Mods use slot-based template expansion rather than AI-generated code.

### Context
Need to generate backend code that's guaranteed secure and correct.

### Rationale
- **Templates are tested**: Mod authors test templates at dev time
- **Slots are validated**: JSON Schema ensures valid values
- **No syntax errors**: Impossible if template + slots are valid
- **Security by construction**: Templates include proper protections

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| AI generates code | Same problems as frontend |
| Tree-sitter validation | Runtime overhead, still allows bad code |
| Manual coding | Defeats the purpose |

### Consequences
- AI only fills slot values, never writes code
- Templates must be authored for each operation
- Introspection required to populate enum constraints

---

## ADR-013: SKA Vision Models

### Decision
Canonical vision models are the SKA series (SKA-*-VL variants).

### Context
Need vision-capable models that can interpret design references and UI screenshots.

### Rationale
- **Vision-first**: Native multimodal, not bolted on
- **Open weights**: Can run locally via Ollama
- **Multiple tiers**: Smaller models for local, larger for cloud
- **Strong performance**: Competitive with proprietary models

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| GPT-4V | Proprietary, no local option |
| Claude Vision | Good but secondary to text |
| LLaVA | Lower quality on UI understanding |

### Consequences
- Ollama provider for local inference
- Cloud endpoints for higher tiers
- Vision prompts need specific formatting

---

## ADR-014: Encore.ts for Mohawk Backend

### Decision
Mohawk SaaS uses Encore.ts as its backend framework.

### Context
Need a type-safe backend with automatic infrastructure management.

### Rationale
- **Type-safe APIs**: End-to-end TypeScript types
- **Infrastructure from code**: No separate Terraform/Pulumi
- **Local development**: Full dev environment locally
- **Cloud deployment**: Automatic infrastructure provisioning

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Express/Fastify | No infrastructure abstraction |
| NestJS | Heavier, Angular patterns |
| tRPC alone | No infrastructure management |
| Go backend | Language context switch |

### Consequences
- Mohawk dogfoods its own backend option
- PostgreSQL as database
- Cloud deployment through Encore platform

---

## ADR-015: @effect/schema for Validation (NEW)

### Decision
Migrate from Zod to @effect/schema for all runtime validation.

### Context
As of December 2025, the Punk ecosystem requires a validation library that integrates with Effect for error handling, supports advanced transformations, provides structured error ASTs for repair loops, and tree-shakes effectively.

### Rationale
- **Effect ecosystem**: Native integration with Effect
- **Encoding/Decoding**: Built-in transformations
- **Structured errors**: AST-based errors enable AI repair loops
- **Bundle size**: 8KB vs Zod's 12KB

### Alternatives Considered
| Alternative | Why Rejected |
|-------------|--------------|
| Zod (keep) | No Effect integration |
| io-ts | Older API |
| Valibot | Smaller ecosystem |

### Consequences
- All schemas use `Schema.Struct()`, `Schema.Literal()`, etc.
- Type extraction via `Schema.Schema.Type<typeof X>`
- AI repair loop uses Effect's structured parse errors

---

## ADR-016: Base UI as Default Behavioral Foundation (NEW)

### Decision
Migrate from Radix Primitives to Base UI as the default behavioral foundation. Radix remains available via compatibility shim.

### Context
Base UI reached stable release in late 2024 with significant advantages over Radix.

### Rationale
- **Larger component set**: 37 components vs Radix's 28
- **Form system**: Built-in form validation
- **API patterns**: Cleaner compound component composition
- **Tree-shaking**: Better bundle splitting

### Compatibility
```typescript
// punk.config.ts
export default {
  components: {
    backend: 'base-ui'  // or 'radix' for legacy
  }
}
```

### Consequences
- New projects use Base UI by default
- Existing Radix projects continue via shim
- No runtime performance penalty

---

## ADR-017: `punk wrap` CLI for Rig Generation (NEW)

### Decision
Replace Garden.js with `punk wrap`, a CLI that generates Rigs by walking TypeScript types from npm packages.

### Context
Garden.js adds no value—Puck provides preview, registry is trivial, multi-framework not needed.

### Rationale
- **Zero authoring**: Extract props from TypeScript
- **LLM enrichment**: AI generates knowledge at wrap time
- **Single source of truth**: Types → Puck config → Effect schema

### Usage
```bash
punk wrap react-chartjs-2 --component Chart --tier pro
punk wrap @tanstack/react-table --tier pro --purchasable 1900
punk wrap react-kanban --tier premium --price 3900
```

### Consequences
- Garden.js removed
- Rigs require only package name + tier flags + optional mapper
- TypeScript types become source of truth

---

## ADR-018: Tier-Gated Rig Access (NEW)

### Decision
Implement tiered access for Rigs with subscription tiers and à la carte purchases.

### Tier Model
| Tier | Access | Examples |
|------|--------|----------|
| `free` | All users | Button, Card, Text |
| `starter` | Starter+ | Form, Tabs, Modal |
| `pro` | Pro+ | Chart, DataTable |
| `purchasable` | Buy without upgrading | DataTable $19 |
| `premium` | Always separate | Kanban $39 |

### Consequences
- Registry filters by user tier + purchases
- Ska context includes upgrade options
- Depot enforces access on download

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| 001 | Schema-First Architecture | Active (amended) |
| 002 | Puck as Schema Renderer | Active |
| 003 | Radix for Accessibility | **Superseded** |
| 004 | Pink CSS Design System | Active |
| 005 | Garden.js for Rig Registry | **Superseded** |
| 006 | TokiForge for Theming | Active |
| 007 | ULID for Identifiers | Active |
| 008 | SQLar for Mod Packaging | Active |
| 009 | Trinity Runtime | Active |
| 010 | Vercel AI SDK | Active (amended) |
| 011 | Electron + Bun | Active |
| 012 | Slot-Based Templates | Active |
| 013 | SKA Vision Models | Active |
| 014 | Encore.ts Backend | Active |
| 015 | @effect/schema | **New** |
| 016 | Base UI | **New** |
| 017 | punk wrap CLI | **New** |
| 018 | Tier-Gated Rigs | **New** |

---

*Last updated: December 2025*
