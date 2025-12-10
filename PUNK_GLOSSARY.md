# Punk Framework Glossary

**Version:** 4.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Core Framework

| Term | Definition |
|------|------------|
| **Punk** | The core deterministic rendering engine. Takes JSON schemas, validates with @effect/schema, outputs accessible React components. Tier 1 (free, open source). |
| **Synthpunk** | AI-powered UI generation layer built on Punk. Natural language → validated schemas. Tier 2. |
| **Atompunk** | Full-stack AI generation including backend code via vetted templates. Tier 3. |
| **Punk Pragmatism** | Core philosophy: AI must work within validated rules, not around them. Constrain the output space so bad output becomes structurally impossible. |

---

## AI Engine

| Term | Definition |
|------|------------|
| **Ska** | The AI engine powering Punk. Vision-first, schema-constrained. Three model tiers based on task complexity. |
| **SKA-30B-VL** | Entry tier. Vision-capable, cost-effective (~$0.002/build). Used for Free/Starter tiers. |
| **SKA-106B-VL** | Mid tier. Complex layouts, nuanced interpretation (~$0.016/build). Used for Pro tier. |
| **SKA-235B-VL** | High tier. Flagship capability (~$0.013/build). Used for Pro+ tier. |
| **Vercel AI SDK** | Implementation infrastructure for Ska. Provides native @effect/schema support, streaming, multi-provider abstraction. |

---

## Runtime & Supervision

| Term | Definition |
|------|------------|
| **Nitro** | Universal process supervisor. Manages lifecycle, health monitoring, logging, and graceful shutdown for all Punk processes. Provides consistent behavior across Desktop, Docker, CLI, and self-hosted deployments. Required by Agent Mod. |
| **Trinity Runtime** | Polyglot execution environment inside mods: Lua via wasmoon (orchestration logic), txiki.js (I/O and networking), WAMR (WebAssembly for performance-critical code). |
| **Bun** | JavaScript/TypeScript runtime for full-stack Punk apps. Provides runtime, bundler, and package manager. Used by Mohawk platform and Desktop app. |
| **wasmoon** | Lua 5.4 compiled to WebAssembly. Powers the Lua component of Trinity Runtime. |
| **Effect** | TypeScript library for typed functional programming. Provides schema validation, error handling, dependency injection, and observability. Foundation of Punk's runtime. |

---

## Extension Systems

| Term | Definition |
|------|------------|
| **Mod** | Self-contained capability package distributed as a `.sqlar` archive. Contains code, knowledge docs, templates, schemas, and Trinity Runtime components. |
| **Rig** | External library (Chart.js, TanStack Table, etc.) wrapped for Puck compatibility via `punk wrap` CLI. Exposes safe, validated configuration surfaces. |
| **punk wrap** | CLI tool that generates Rigs from npm packages by extracting TypeScript types and generating Effect schemas, Puck configs, and AI knowledge metadata. |
| **Agent Mod** | A specific mod that enables multi-agent workflow execution. Transforms single-shot generation into deterministic, resumable, auditable multi-step workflows. Requires Nitro. |
| **MCP Gateway** | Mod providing curated access to MCP (Model Context Protocol) servers. Includes knowledge docs for accurate tool selection, tier-based access control, and auth management. |
| **Depot** | Marketplace for discovering and installing Rigs and Mods. Quality-tiered (Bronze/Silver/Gold). |

---

## Rendering Stack

| Term | Definition |
|------|------------|
| **Puck** | Schema renderer. Accepts validated JSON schemas, maps types to React components. Core of deterministic rendering. |
| **Base UI** | Default behavioral foundation for interactive components. 37 unstyled primitives with built-in form system, accessibility, and tree-shaking. From MUI team. |
| **Radix** | Alternative behavioral foundation available via compatibility shim. For legacy projects or preference. |
| **Pink** | Appwrite's CSS design system. Framework-agnostic styling classes. Appearance layer. |
| **TokiForge** | Design token engine with runtime theme switching. Validates WCAG contrast compliance. |
| **@effect/schema** | TypeScript-first schema validation from Effect ecosystem. Creates hard boundary between AI output and rendering. Replaces Zod. |

---

## Storage & Packaging

| Term | Definition |
|------|------------|
| **GlyphCase** | SQLite-based local-first storage. WAL mode, encrypted at rest, sync-ready. Also the recommended database for user apps. |
| **SQLar** | SQLite Archive format. A SQLite database with embedded files. Single-file distribution format for Mods and Rigs. |
| **ULID** | Universally Unique Lexicographically Sortable Identifier. Standard ID format across all Punk systems. 26 characters, time-ordered, URL-safe. |

---

## Delivery Vehicles

| Term | Definition |
|------|------------|
| **Mohawk** | The builder platform. Available as SaaS, Desktop, and Docker. |
| **Mohawk SaaS** | Hosted web builder at app.punk.dev. Freemium tiers: Free, Starter ($15), Pro ($39), Pro+ ($69), Team ($29/seat). |
| **Mohawk Desktop (Studio)** | Electron + Bun application. Full rig/mod library, offline editing. Subscription-based. |
| **Mohawk Docker (Lite)** | Self-hosted Docker deployment. Default rigs/mods, no premium features. Free/open source. |
| **Punk CLI** | Terminal-based TUI built with Go + Charm (Bubble Tea, Lip Gloss). Project scaffolding, mod management, direct schema editing. Free/open source. |

---

## MCP & Integrations

| Term | Definition |
|------|------------|
| **MCP** | Model Context Protocol. Standard for AI-to-service communication. |
| **MCP Market** | Curated catalog of MCP servers available through the MCP Gateway mod. Quality-tiered, documented, vetted for security. |
| **Knowledge Doc** | Per-server documentation teaching Ska when and how to use MCP tools. Transforms 60% raw accuracy to 95%+. |
| **Connector** | Simplified integration for cloud-deployed apps. Managed by platform, metered, rate-limited. Examples: Google Sheets, Airtable, Webhooks. |
| **Active Slot** | An enabled MCP server counting against tier limit. Users can swap servers in/out freely. |

---

## Backend Options (for apps built with Punk)

| Term | Definition |
|------|------------|
| **None (Static)** | No backend. Data baked in at build time or fetched client-side. |
| **Encore.ts** | Infrastructure from code (TypeScript). Type-safe APIs with cloud deployment. |
| **tRPC** | End-to-end TypeScript type safety for API layer. |
| **GlyphCase** | Local SQLite backend. Good for local-first apps, offline support. |
| **Neon** | Serverless PostgreSQL with scale-to-zero. For apps needing cloud database. |

---

## Rig Tier System

| Term | Definition |
|------|------------|
| **Free Rig** | Available to all users. Core primitives (Button, Card, Text, Container). |
| **Starter Rig** | Available to Starter+ subscribers. Enhanced components (Form, Tabs, Modal). |
| **Pro Rig** | Available to Pro+ subscribers. Advanced components (Chart, DataTable, RichText). |
| **Purchasable Rig** | Can be bought à la carte without upgrading tier. Example: DataTable for $19. |
| **Premium Rig** | Must be purchased separately, regardless of tier. Example: Kanban for $39. |

---

## Platform Tier System

| Term | Definition |
|------|------------|
| **Free** | 100 builds/month, static preview, vision-capable (SKA-30B-VL), view-only MCP Market. |
| **Starter** | $15/month. 200 builds, WebContainers preview, 5 MCP active slots (Bronze). |
| **Pro** | $39/month. 350 builds, cloud sandbox preview, 15 MCP active slots (Bronze + Silver). |
| **Pro+** | $69/month. 500 builds, Agent Mod access, cloud sandbox, 30 MCP active slots (all tiers). |
| **Team** | $29/seat/month. 250 builds/seat, collaboration features, shared MCP configs. |
| **Agency** | Custom pricing. Unlimited builds, custom MCP servers, white-label options. |

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Schema-Constrained Generation** | AI generates structured JSON conforming to @effect/schema, not arbitrary code. Simpler task = smaller models = lower cost = higher accuracy. |
| **Deterministic Rendering** | Same schema input produces identical React output every time. No runtime variability. |
| **Vision-First** | Ska processes screenshots and mockups natively. Enables "napkin sketch to app" workflows. |
| **Accessibility by Construction** | WCAG compliance built into component structure, not added via testing. |
| **Knowledge-Enhanced Tool Selection** | MCP tools selected based on documentation ("when to use") not just schema ("what it does"). |
| **Zero Authoring** | Rig creation extracts everything from existing TypeScript types. No manual schema writing. |

---

## Deprecated/Replaced Terms

| Old Term | Status | Replacement |
|----------|--------|-------------|
| Epoch | Deprecated | Vercel AI SDK |
| Agentic Case | Renamed | Agent Mod |
| Nitro Case | Renamed | Nitro |
| .gcasex | Deprecated | .sqlar |
| Punk Cloud | Renamed | Mohawk SaaS |
| Zod | Replaced | @effect/schema |
| Garden.js | Deprecated | punk wrap CLI |
| defineRig() | Deprecated | punk wrap CLI |

---

*Last updated: December 2025*
