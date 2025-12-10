# Punk / Mohawk Architecture Diagram

**Version:** 2.0 | December 2025

---

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    USER INTERFACES                                                  │
│                                                                                                     │
│   ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐    │
│   │   MOHAWK SAAS     │   │  MOHAWK DESKTOP   │   │  MOHAWK DOCKER    │   │     PUNK CLI      │    │
│   │   (app.punk.dev)  │   │     (Studio)      │   │      (Lite)       │   │                   │    │
│   │                   │   │                   │   │                   │   │                   │    │
│   │ • Web GUI         │   │ • Electron + Bun  │   │ • Self-hosted     │   │ • Go + Charm      │    │
│   │ • Chat with Ska   │   │ • Offline editing │   │ • Docker Compose  │   │ • TUI interface   │    │
│   │ • Team collab     │   │ • Full Rig/Mod    │   │ • Default Rigs    │   │ • Scaffolding     │    │
│   │ • Cloud preview   │   │   library         │   │ • OSS/Free        │   │ • Mod management  │    │
│   │                   │   │                   │   │                   │   │                   │    │
│   │ Tiers:            │   │ Subscription      │   │ Free              │   │ Free/OSS          │    │
│   │ Free/$15/$39/$69  │   │                   │   │                   │   │                   │    │
│   └─────────┬─────────┘   └─────────┬─────────┘   └─────────┬─────────┘   └─────────┬─────────┘    │
│             │                       │                       │                       │              │
│             └───────────────────────┴───────────────────────┴───────────────────────┘              │
│                                                 │                                                   │
│                                                 ▼                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                              PUNK FRAMEWORK  ("The Engine")                                         │
│                      "Don't make AI smarter—make the output space safer"                            │
│                                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              ATOMPUNK  (Tier 3 - Pro+)                                      │   │
│   │                            Full-Stack AI Generation                                         │   │
│   │                                                                                             │   │
│   │    • Backend code via vetted templates      • Database schema generation                    │   │
│   │    • Auth scaffolding (pre-audited)         • Deployment configuration                      │   │
│   │    • Security by construction               • Slot-based parameter filling                  │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                 │                                                   │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              SYNTHPUNK  (Tier 2 - Starter+)                                 │   │
│   │                            AI-Powered UI Generation                                         │   │
│   │                                                                                             │   │
│   │    • Ska AI engine                          • Natural language → schemas                    │   │
│   │    • Vision-semantic interpretation         • Revision history & branching                  │   │
│   │    • Multi-step reasoning                   • Context management                            │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                 │                                                   │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                PUNK  (Tier 1 - Free)                                        │   │
│   │                            Deterministic Rendering                                          │   │
│   │                                                                                             │   │
│   │    • Schema → React renderer                • @effect/schema validation (hard boundary)                │   │
│   │    • Type safety guaranteed                 • WCAG 2.1 AA by construction                   │   │
│   │    • Deterministic output                   • Same input = Same output                      │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                  │
                          ┌───────────────────────┼───────────────────────┐
                          │                       │                       │
                          ▼                       ▼                       ▼
┌─────────────────────────────────────┐ ┌─────────────────────────────────────┐ ┌─────────────────────┐
│                                     │ │                                     │ │                     │
│        SKA  (AI Engine)             │ │     RENDERING STACK                 │ │   EXTENSION SYSTEM  │
│                                     │ │                                     │ │                     │
│  ┌───────────────────────────────┐  │ │  ┌───────────────────────────────┐  │ │  ┌───────────────┐  │
│  │  SKA-30B-VL (Free/Starter)    │  │ │  │       EFFECT                  │  │ │  │     MODS      │  │
│  │  Qwen3-30B-A3B-VL             │  │ │  │  @effect/schema validation    │  │ │  │  (.sqlar)     │  │
│  │  ~$0.002/build                │  │ │  │  Type safety boundary         │  │ │  │               │  │
│  └───────────────────────────────┘  │ │  └───────────────┬───────────────┘  │ │  │ • Agent Mod   │  │
│  ┌───────────────────────────────┐  │ │                  │                  │ │  │ • MCP Gateway │  │
│  │  SKA-106B-VL (Pro)            │  │ │                  ▼                  │ │  │ • Custom      │  │
│  │  GLM-4.5V                     │  │ │  ┌───────────────────────────────┐  │ │  └───────┬───────┘  │
│  │  ~$0.016/build                │  │ │  │         PUCK                  │  │ │          │         │
│  └───────────────────────────────┘  │ │  │  Schema → React components    │  │ │  ┌───────────────┐  │
│  ┌───────────────────────────────┐  │ │  └───────────────┬───────────────┘  │ │  │     RIGS      │  │
│  │  SKA-235B-VL (Pro+)           │  │ │                  │                  │ │  │  (Components) │  │
│  │  Qwen3-VL-235B                │  │ │                  ▼                  │ │  │               │  │
│  │  ~$0.013/build                │  │ │  ┌───────────────────────────────┐  │ │  │ • Chart.js    │  │
│  └───────────────────────────────┘  │ │  │      PUNK WRAP CLI            │  │ │  │ • TanStack    │  │
│                                     │ │  │  Rig generation from types    │──┼─┼──│ • Mapbox      │  │
│  ┌───────────────────────────────┐  │ │  └───────────────┬───────────────┘  │ │  │ • etc.        │  │
│  │  VERCEL AI SDK                │  │ │                  │                  │ │  └───────────────┘  │
│  │  • Native @effect/schema support         │  │ │      ┌──────────┼──────────┐       │ │                     │
│  │  • Streaming                  │  │ │      │          │          │       │ │  ┌───────────────┐  │
│  │  • Multi-provider             │  │ │      ▼          ▼          ▼       │ │  │     DEPOT     │  │
│  └───────────────────────────────┘  │ │  ┌───────┐ ┌─────────┐ ┌────────┐  │ │  │  Marketplace  │  │
│                                     │ │  │BASE UI│ │  PINK   │ │TOKIFORG│  │ │  │               │  │
│  VISION-FIRST:                      │ │  │       │ │         │ │   E    │  │ │  │ • Discovery   │  │
│  Screenshots & sketches → apps      │ │  │Behav- │ │ CSS     │ │        │  │ │  │ • Install     │  │
│                                     │ │  │ior &  │ │ Design  │ │ Design │  │ │  │ • Ratings     │  │
│                                     │ │  │A11y   │ │ System  │ │ Tokens │  │ │  │ • Tier-gated  │  │
│                                     │ │  └───────┘ └─────────┘ └────────┘  │ │  └───────────────┘  │
└─────────────────────────────────────┘ └─────────────────────────────────────┘ └─────────────────────┘
                          │                                                               │
                          └───────────────────────────┬───────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    RUNTIME LAYER                                                    │
│                                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                                             │   │
│   │                              NITRO  (Universal Supervisor)                                  │   │
│   │                                                                                             │   │
│   │    Process supervision │ Health monitoring │ Log aggregation │ Graceful shutdown            │   │
│   │                                                                                             │   │
│   │    Supervises:                                                                              │   │
│   │    • Trinity Runtime        • MCP Servers (local)       • Preview Server                    │   │
│   │    • GlyphCase DB           • Backend Dev Server        • Sidecars                          │   │
│   │                                                                                             │   │
│   │    Where it runs:                                                                           │   │
│   │    • Desktop: Spawned by Electron main process                                              │   │
│   │    • Docker: PID 1 in container                                                             │   │
│   │    • CLI: `punk dev` starts Nitro                                                           │   │
│   │    • SaaS: Cloud platforms handle supervision (K8s/ECS)                                     │   │
│   │                                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                     │
│   ┌─────────────────────────────┐   ┌─────────────────────────────┐   ┌─────────────────────────┐   │
│   │                             │   │                             │   │                         │   │
│   │     TRINITY RUNTIME         │   │        GLYPHCASE            │   │      MCP GATEWAY        │   │
│   │                             │   │                             │   │                         │   │
│   │  ┌─────────────────────┐    │   │  ┌─────────────────────┐    │   │  ┌─────────────────┐    │   │
│   │  │   LUA (wasmoon)     │    │   │  │  SQLite (WAL mode)  │    │   │  │   MCP Market    │    │   │
│   │  │   Brain / Logic     │    │   │  │  Local-first        │    │   │  │   50-100 curated│    │   │
│   │  └─────────────────────┘    │   │  │  Encrypted at rest  │    │   │  │   servers       │    │   │
│   │  ┌─────────────────────┐    │   │  │  Sync-ready         │    │   │  └─────────────────┘    │   │
│   │  │   TXIKI.JS          │    │   │  └─────────────────────┘    │   │  ┌─────────────────┐    │   │
│   │  │   Hands / I/O       │    │   │                             │   │  │  Knowledge Docs │    │   │
│   │  └─────────────────────┘    │   │  Also: Mod packaging        │   │  │  95%+ accuracy  │    │   │
│   │  ┌─────────────────────┐    │   │  format (.sqlar)            │   │  │  vs 60% raw     │    │   │
│   │  │   WAMR (WASM)       │    │   │                             │   │  └─────────────────┘    │   │
│   │  │   Muscle / Perf     │    │   │                             │   │  ┌─────────────────┐    │   │
│   │  └─────────────────────┘    │   │                             │   │  │  Tier Access    │    │   │
│   │                             │   │                             │   │  │  5/15/30/∞ slots│    │   │
│   └─────────────────────────────┘   └─────────────────────────────┘   │  └─────────────────┘    │   │
│                                                                       │                         │   │
│                                                                       │  Auth: OAuth, API keys  │   │
│                                                                       │  encrypted in GlyphCase │   │
│                                                                       └─────────────────────────┘   │
│                                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                                             │   │
│   │                              AGENT MOD  (Pro+ Only)                                         │   │
│   │                                                                                             │   │
│   │    Deterministic multi-agent workflow execution                                             │   │
│   │                                                                                             │   │
│   │    • Graph-based workflows          • Resumable on crash         • Full audit trail         │   │
│   │    • Knowledge docs (RAG)           • Case learning              • Memory systems           │   │
│   │    • Requires Nitro supervision     • 5-30 minute workflows      • FTS5 retrieval           │   │
│   │                                                                                             │   │
│   │    Without Agent Mod: Prompt → Ska → Output (single turn, like v0/Lovable)                  │   │
│   │    With Agent Mod:    Workflow → Graph execution → Audited output (like Claude Code)        │   │
│   │                                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                          BACKEND OPTIONS  (For Generated Apps)                                      │
│                                                                                                     │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌─────────────┐   │
│   │               │   │               │   │               │   │               │   │             │   │
│   │     NONE      │   │   ENCORE.TS   │   │     TRPC      │   │   GLYPHCASE   │   │    NEON     │   │
│   │   (Static)    │   │               │   │               │   │    (Local)    │   │             │   │
│   │               │   │ Infra-as-code │   │ Type-safe     │   │               │   │ Serverless  │   │
│   │ Build-time    │   │ TypeScript    │   │ API layer     │   │ SQLite        │   │ PostgreSQL  │   │
│   │ data only     │   │ Cloud deploy  │   │               │   │ Offline-first │   │ Scale-to-0  │   │
│   │               │   │               │   │               │   │               │   │             │   │
│   └───────────────┘   └───────────────┘   └───────────────┘   └───────────────┘   └─────────────┘   │
│                                                                                                     │
│   These are NOT parallel choices. You can combine: Encore.ts + Neon + GlyphCase (caching)           │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                          DEPLOYMENT TARGETS  (For Published Apps)                                   │
│                                                                                                     │
│   ┌───────────────────────────────┐   ┌───────────────────────────────┐   ┌───────────────────────┐ │
│   │                               │   │                               │   │                       │ │
│   │       STATIC EXPORT           │   │       LOCAL DEPLOY            │   │     CLOUD DEPLOY      │ │
│   │                               │   │                               │   │                       │ │
│   │   • HTML/JS/CSS bundle        │   │   • User's machine            │   │   • Managed hosting   │ │
│   │   • Any static host           │   │   • Full MCP access           │   │   • Connectors only   │ │
│   │   • No backend                │   │   • User's credentials        │   │   • Rate limited      │ │
│   │   • Build-time data           │   │   • Nitro optional            │   │   • Metered usage     │ │
│   │                               │   │                               │   │                       │ │
│   └───────────────────────────────┘   └───────────────────────────────┘   └───────────────────────┘ │
│                                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                                             │   │
│   │                              CONNECTORS  (Cloud Deploy Only)                                │   │
│   │                                                                                             │   │
│   │    Simplified integrations for cloud-hosted apps (not raw MCP)                              │   │
│   │                                                                                             │   │
│   │    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │   │
│   │    │ Data Refresh │ │  Webhook In  │ │ Webhook Out  │ │Google Sheets │ │   Airtable   │     │   │
│   │    │  (scheduled) │ │  (receive)   │ │   (send)     │ │              │ │              │     │   │
│   │    └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘     │   │
│   │                                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Relationships

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    DATA FLOW                                                        │
│                                                                                                     │
│                                                                                                     │
│    User Intent ──────► Ska (AI) ──────► JSON Schema ──────► @effect/schema ──────► Puck ──────► React UI      │
│         │                  │                                                             │          │
│         │                  │                                                             │          │
│         ▼                  ▼                                                             ▼          │
│    Screenshot         Vision Model                                              Accessible,         │
│    or Sketch          Processing                                                Deterministic       │
│                                                                                 Output              │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    SUPERVISION HIERARCHY                                            │
│                                                                                                     │
│                                                                                                     │
│    ┌─────────────────────────────────────────────────────────────────┐                              │
│    │                         NITRO                                   │                              │
│    │                   (Process Supervisor)                          │                              │
│    └──────────────────────────┬──────────────────────────────────────┘                              │
│                               │                                                                     │
│           ┌───────────────────┼───────────────────────────┐                                         │
│           │                   │                           │                                         │
│           ▼                   ▼                           ▼                                         │
│    ┌─────────────┐     ┌─────────────┐            ┌─────────────┐                                   │
│    │ GlyphCase   │     │  Trinity    │            │   Other     │                                   │
│    │ (starts 1st │     │  Runtime    │            │  Services   │                                   │
│    │  stops last)│     │             │            │             │                                   │
│    └─────────────┘     └──────┬──────┘            └─────────────┘                                   │
│                               │                   • MCP Servers                                     │
│                    ┌──────────┼──────────┐        • Preview Server                                  │
│                    │          │          │        • Backend Server                                  │
│                    ▼          ▼          ▼        • Sidecars                                        │
│               ┌────────┐ ┌────────┐ ┌────────┐                                                      │
│               │  Lua   │ │txiki.js│ │  WAMR  │                                                      │
│               │(logic) │ │ (I/O)  │ │ (perf) │                                                      │
│               └────────┘ └────────┘ └────────┘                                                      │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    MCP FLOW                                                         │
│                                                                                                     │
│                                                                                                     │
│    User Request ───► Ska reads ───► Ska selects ───► MCP Gateway ───► MCP Server ───► Result       │
│         │            Knowledge       correct tool        │                │               │         │
│         │            Docs            (95%+ accuracy)     │                │               │         │
│         │                                                │                │               │         │
│         │                                                ▼                ▼               │         │
│         │                                           Validates       Injects auth          │         │
│         │                                           arguments       (from GlyphCase)      │         │
│         │                                                                                 │         │
│         └─────────────────────────────────────────────────────────────────────────────────┘         │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Tier Summary

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                                     │
│                                    MOHAWK TIERS                                                     │
│                                                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                                             │   │
│   │  FREE           STARTER ($15)      PRO ($39)        PRO+ ($69)       TEAM ($29/seat)       │   │
│   │                                                                                             │   │
│   │  100 builds     200 builds         350 builds       500 builds       250/seat              │   │
│   │  Static prev    WebContainers      Cloud sandbox    Cloud sandbox    Collaboration         │   │
│   │  SKA-30B-VL     SKA-30B-VL         SKA-106B-VL      SKA-235B-VL      Shared configs        │   │
│   │  MCP: view      5 slots (Bronze)   15 slots (B+S)   30 slots (all)   30 slots (all)        │   │
│   │  —              —                  —                AGENT MOD        —                      │   │
│   │                                                                                             │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                     │
│   Agency (custom): Unlimited builds, custom MCP servers, white-label                                │
│                                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Legend

| Symbol | Meaning |
|--------|---------|
| `───►` | Data flow |
| `───┐` | Hierarchical relationship |
| `[ ]` | Component boundary |
| `•` | List item / feature |

---

*Last updated: December 2025*
