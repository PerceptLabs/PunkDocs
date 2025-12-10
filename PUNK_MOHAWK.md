# Punk Mohawk

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Overview

Mohawk is the builder interface for Punk. Three delivery vehicles serve different user needs.

---

## Delivery Vehicles

| Product | Interface | Target User | Pricing |
|---------|-----------|-------------|---------|
| **Mohawk SaaS** | Web GUI | Casual users, teams | Freemium |
| **Mohawk Desktop (Studio)** | Native Electron | Power users, pros | Subscription |
| **Mohawk Docker (Lite)** | Self-hosted web GUI | Self-hosters, privacy-focused | Free/OSS |

---

## Mohawk SaaS

**URL:** app.punk.dev

Web-based builder interface—"Bolt.new but safer."

### Features
- Chat with Ska to build apps
- Live preview as you iterate
- Direct deployment
- No terminal required
- Team collaboration

### Pricing

| Tier | Price | Projects | Builds/mo | Rigs & Mods | Seats |
|------|-------|----------|-----------|-------------|-------|
| **Free** | $0 | 1 | 100 | Free tier | 1 |
| **Starter** | $15/mo | 3 | 200 | Starter tier | 1 |
| **Pro** | $39/mo | Unlimited | 350 | Pro tier | 3 |
| **Pro+** | $69/mo | Unlimited | 500 | All + Agent Mod | 5 |
| **Team** | $29/seat/mo | Unlimited | 250/seat | Pro tier | Unlimited |
| **Agency** | Custom | Unlimited | Unlimited | Custom/Private | Unlimited |

### Tech Stack
- **Backend:** Encore.ts
- **Database:** PostgreSQL

---

## Mohawk Desktop (Studio)

Premium native application for power users.

### Features
- Full rig and mod library
- Offline editing (AI requires connection)
- Native performance
- All premium components
- BYO API key or subscription

### Tech Stack
- **Shell:** Electron
- **Runtime:** Bun
- **Database:** SQLite (local)
- **UI:** React + Puck + Base UI + TokiForge + Pink

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    RENDERER PROCESS (Chromium)                  │
│                                                                 │
│  React + Puck + Base UI + TokiForge + Pink                     │
│                                                                 │
└─────────────────────────────────┬───────────────────────────────┘
                                  │ Electron IPC
┌─────────────────────────────────┴───────────────────────────────┐
│                    MAIN PROCESS (Bun)                           │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  GlyphCase  │  │    Ska      │  │  Licensing  │             │
│  │ (bun:sqlite)│  │    API      │  │   Manager   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────────────────────────────────────────┐           │
│  │              TRINITY RUNTIME                     │           │
│  │  Lua (wasmoon) + JS (native) + WASM (bun:wasm)  │           │
│  └─────────────────────────────────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Distribution
- macOS: `.dmg`
- Windows: `.exe` installer
- Linux: `.AppImage`

### Pricing
- Subscription-based
- Same as SaaS tiers (synced account)

---

## Mohawk Docker (Lite)

Self-hosted, privacy-first option.

### Features
- Full Docker Compose deployment
- Local inference via Docker Model Runner
- Default rig and mod set (not full library)
- No cloud dependency
- Complete data ownership

### Deployment

```bash
# Clone and start
git clone https://github.com/PerceptLabs/mohawk-lite
cd mohawk-lite
docker compose up
```

### Components

```yaml
services:
  mohawk:
    image: perceptlabs/mohawk-lite
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - model-runner

  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data

  model-runner:
    image: docker/model-runner
    # Runs SKA-30B-VL locally
```

### Limitations vs SaaS/Desktop
- Default rigs/mods only (no premium library)
- No Depot access
- No cloud sync
- Self-managed updates

### Pricing
- **Free** (open source)

---

## Punk CLI

Companion terminal interface for developers.

### Features
- Beautiful TUI (Bubble Tea + Lip Gloss)
- Project scaffolding
- Mod management
- Rig wrapping via `punk wrap`
- Live reload
- Direct schema editing

### Tech Stack
- **Language:** Go
- **TUI:** Charm (Bubble Tea, Lip Gloss)

### Commands

```bash
punk create             # Interactive project creation
punk dev                # Start development server
punk build              # Build for production
punk deploy             # Deploy application
punk add mod <name>     # Add a mod
punk wrap <package>     # Wrap npm package as Rig
```

### Themes
- **Punk** — Hot pink, electric blue
- **Synthpunk** — Cyan, magenta, purple
- **Atompunk** — Orange, teal, cream
- **Solarpunk** — Green, gold, earth tones

### Pricing
- **Free** (open source)

---

## Feature Comparison

| Feature | CLI | Docker (Lite) | SaaS | Desktop (Studio) |
|---------|-----|---------------|------|------------------|
| Visual Builder | ❌ | ✅ | ✅ | ✅ |
| AI Generation | BYO Key | Local (Docker MR) | Credits/Sub | BYO Key or Sub |
| Rigs | Manual Import | Default Set | Full Library | Full Library |
| Mods | Manual Import | Default Set | Full Library | Full Library |
| Offline | ✅ | ✅ | ❌ | ✅ (editing) |
| Collaboration | ❌ | ❌ | ✅ | ❌ |
| Price | Free | Free | Freemium | Subscription |

---

*Last updated: December 2025*
