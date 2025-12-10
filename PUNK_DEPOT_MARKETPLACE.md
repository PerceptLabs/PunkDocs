# Punk Depot Marketplace

**Version:** 1.0  
**Status:** Source of Truth  
**Audience:** Human Developers & Claude Code

---

## Overview

Depot is Mohawk's marketplace for extending Punk applications. It provides four modalities, each available individually or as curated bundles.

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEPOT                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    RIGS      │  │    MODS      │  │   THEMES     │          │
│  │   Rigsets    │  │   Modpacks   │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  ┌──────────────┐                                              │
│  │  TEMPLATES   │                                              │
│  │              │                                              │
│  └──────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Modality Summary

| Modality | Individual | Bundle | Purpose |
|----------|------------|--------|---------|
| **Rigs** | Single component wrapper | **Rigset** | UI components |
| **Mods** | Single capability | **Modpack** | Backend/services |
| **Themes** | Single token set | — | Visual styling |
| **Templates** | Single project scaffold | — | Starting points |

---

## 1. Rigs & Rigsets

### What Is a Rig?

A **Rig** is an external library wrapped for Puck compatibility. Rigs expose validated, schema-driven configuration surfaces for complex UI components.

```
External Library → Garden.js Wrapper → Puck-Compatible Component
```

### Individual Rigs

| Category | Rig | Library | Description |
|----------|-----|---------|-------------|
| **Data** | Table | TanStack Table | Sortable, paginated tables |
| **Data** | Chart | Chart.js | Bar, line, pie, doughnut charts |
| **Data** | DataGrid | AG Grid | Enterprise data grid |
| **Data** | Pivot | PivotTable.js | Pivot tables |
| **Form** | DatePicker | react-day-picker | Calendar date selection |
| **Form** | DateRange | react-date-range | Date range selection |
| **Form** | TimePicker | react-time-picker | Time selection |
| **Form** | Select | React Select | Searchable dropdowns |
| **Form** | MultiSelect | React Select | Multi-value selection |
| **Form** | Combobox | Downshift | Autocomplete input |
| **Form** | ColorPicker | react-colorful | Color selection |
| **Form** | Slider | Radix Slider | Range input |
| **Form** | Switch | Radix Switch | Toggle switch |
| **Form** | FileDrop | react-dropzone | File upload drag/drop |
| **Form** | Signature | react-signature-canvas | Signature capture |
| **Display** | Code | CodeMirror | Syntax-highlighted code |
| **Display** | Markdown | marked | Markdown rendering |
| **Display** | RichText | Lexical | WYSIWYG editor |
| **Display** | Mermaid | Mermaid | Diagrams from text |
| **Display** | PDF | react-pdf | PDF viewer |
| **Display** | Image | react-image-gallery | Image gallery |
| **Display** | Video | react-player | Video player |
| **Display** | Map | react-map-gl | Interactive maps |
| **Display** | Timeline | custom | Event timeline |
| **Display** | Diff | react-diff-viewer | Code/text diff |
| **Navigation** | Command | cmdk | Command palette |
| **Navigation** | TreeView | react-arborist | Tree navigation |
| **Navigation** | Tabs | Radix Tabs | Tabbed content |
| **Navigation** | Accordion | Radix Accordion | Collapsible sections |
| **Navigation** | Breadcrumb | custom | Breadcrumb trail |
| **Navigation** | Stepper | custom | Multi-step wizard |
| **Layout** | Kanban | @hello-pangea/dnd | Kanban board |
| **Layout** | Carousel | embla-carousel | Image/content carousel |
| **Layout** | Masonry | react-masonry-css | Masonry grid |
| **Layout** | Split | react-split | Resizable panels |
| **Layout** | VirtualList | react-window | Virtualized lists |
| **Feedback** | Toast | sonner | Toast notifications |
| **Feedback** | Dialog | Radix Dialog | Modal dialogs |
| **Feedback** | Drawer | Radix Dialog | Slide-out panels |
| **Feedback** | Tooltip | Radix Tooltip | Hover tooltips |
| **Feedback** | Progress | Radix Progress | Progress indicators |
| **Feedback** | Skeleton | custom | Loading skeletons |

### What Is a Rigset?

A **Rigset** is a curated bundle of rigs that work well together for a specific use case.

| Rigset | Included Rigs | Use Case |
|--------|---------------|----------|
| **Data Analytics** | Table, Chart, DataGrid, Pivot, DateRange | Dashboards, BI tools |
| **Form Builder** | DatePicker, Select, MultiSelect, ColorPicker, Slider, FileDrop, Signature | Complex forms |
| **Content Editor** | RichText, Markdown, Code, Image, FileDrop | CMS, documentation |
| **Developer Tools** | Code, Diff, Mermaid, Terminal, JSON Viewer | Dev utilities |
| **Media Gallery** | Image, Video, Carousel, Masonry, FileDrop | Media management |
| **Project Management** | Kanban, Timeline, Table, DatePicker, Stepper | PM tools |
| **Navigation Suite** | Command, TreeView, Tabs, Accordion, Breadcrumb | Complex navigation |

### Rig Pricing Model

| Tier | Access |
|------|--------|
| **Lite (Open Source)** | 5 core rigs bundled |
| **Hobby** | All individual rigs |
| **Pro** | All rigs + rigsets |
| **Enterprise** | Custom rigsets, priority support |

---

## 2. Mods & Modpacks

### What Is a Mod?

A **Mod** is a self-contained capability extension packaged as a .sqlar archive. Mods add backend services, integrations, and features that run in Trinity Runtime.

```
.sqlar archive contains:
├── Code (Lua/WASM/JS)
├── Schemas (validation)
├── Knowledge docs
├── Templates
└── Trinity Runtime (bundled)
```

### Individual Mods

| Category | Mod | Description |
|----------|-----|-------------|
| **Database** | supabase-mod | Supabase integration |
| **Database** | firebase-mod | Firebase/Firestore |
| **Database** | pocketbase-mod | PocketBase backend |
| **Database** | appwrite-mod | Appwrite backend |
| **Database** | neon-mod | Neon PostgreSQL |
| **Database** | planetscale-mod | PlanetScale MySQL |
| **Database** | turso-mod | Turso/libSQL |
| **Auth** | clerk-mod | Clerk authentication |
| **Auth** | auth0-mod | Auth0 authentication |
| **Auth** | lucia-mod | Lucia auth |
| **Auth** | nextauth-mod | NextAuth.js |
| **Payments** | stripe-mod | Stripe payments |
| **Payments** | lemonsqueezy-mod | Lemon Squeezy |
| **Payments** | paddle-mod | Paddle payments |
| **Email** | resend-mod | Resend email |
| **Email** | sendgrid-mod | SendGrid email |
| **Email** | postmark-mod | Postmark email |
| **Storage** | s3-mod | AWS S3 storage |
| **Storage** | r2-mod | Cloudflare R2 |
| **Storage** | uploadthing-mod | UploadThing |
| **AI** | openai-mod | OpenAI API |
| **AI** | anthropic-mod | Anthropic API |
| **AI** | replicate-mod | Replicate models |
| **AI** | huggingface-mod | HuggingFace inference |
| **Search** | algolia-mod | Algolia search |
| **Search** | meilisearch-mod | Meilisearch |
| **Search** | typesense-mod | Typesense search |
| **Analytics** | posthog-mod | PostHog analytics |
| **Analytics** | plausible-mod | Plausible analytics |
| **Analytics** | mixpanel-mod | Mixpanel analytics |
| **CMS** | sanity-mod | Sanity CMS |
| **CMS** | contentful-mod | Contentful CMS |
| **CMS** | strapi-mod | Strapi CMS |
| **Communication** | twilio-mod | SMS/voice |
| **Communication** | slack-mod | Slack integration |
| **Communication** | discord-mod | Discord integration |
| **Maps** | mapbox-mod | Mapbox maps |
| **Maps** | google-maps-mod | Google Maps |
| **Scheduling** | cal-mod | Cal.com scheduling |
| **Scheduling** | calendly-mod | Calendly integration |
| **PDF** | pdf-mod | PDF generation/parsing |
| **Automation** | zapier-mod | Zapier webhooks |
| **Automation** | n8n-mod | n8n workflows |

### What Is a Modpack?

A **Modpack** is a curated bundle of mods that work together for a complete solution.

| Modpack | Included Mods | Use Case |
|---------|---------------|----------|
| **SaaS Starter** | clerk-mod, stripe-mod, neon-mod, resend-mod, posthog-mod | SaaS applications |
| **E-commerce** | stripe-mod, supabase-mod, s3-mod, resend-mod, algolia-mod | Online stores |
| **Content Platform** | sanity-mod, clerk-mod, s3-mod, algolia-mod | CMS-driven sites |
| **AI App** | openai-mod, anthropic-mod, supabase-mod, clerk-mod | AI applications |
| **Mobile Backend** | supabase-mod, clerk-mod, s3-mod, twilio-mod | Mobile app backends |
| **Analytics Suite** | posthog-mod, mixpanel-mod, plausible-mod | Full analytics |
| **Communication Hub** | twilio-mod, slack-mod, discord-mod, resend-mod | Messaging apps |

### Mod Pricing Model

| Tier | Access |
|------|--------|
| **Lite (Open Source)** | No mods (no Trinity Runtime) |
| **Hobby** | 3 mods included, add more à la carte |
| **Pro** | All individual mods |
| **Enterprise** | All mods + modpacks, custom mods |

---

## 3. Themes

### What Is a Theme?

A **Theme** is a TokiForge token set that defines colors, typography, spacing, and other design variables. Themes apply globally to Pink CSS and all components.

```typescript
// Theme structure
{
  name: "corporate",
  tokens: {
    colors: {
      primary: { value: "#1e40af" },
      secondary: { value: "#3b82f6" },
      background: { value: "#ffffff" },
      foreground: { value: "#111827" },
      muted: { value: "#f3f4f6" },
      accent: { value: "#dbeafe" },
      destructive: { value: "#dc2626" },
    },
    typography: {
      fontFamily: { value: "'Inter', sans-serif" },
      fontFamilyMono: { value: "'JetBrains Mono', monospace" },
    },
    radii: {
      sm: { value: "0.25rem" },
      md: { value: "0.375rem" },
      lg: { value: "0.5rem" },
    },
    shadows: { /* ... */ },
    spacing: { /* ... */ },
  },
  modes: {
    light: { /* overrides */ },
    dark: { /* overrides */ },
  }
}
```

### Available Themes

| Category | Theme | Description |
|----------|-------|-------------|
| **Base** | Default | Punk's default theme (bundled with Lite) |
| **Base** | Dark | Dark mode variant |
| **Professional** | Corporate | Blue, clean, business-appropriate |
| **Professional** | Enterprise | Gray, muted, serious |
| **Professional** | Legal | Conservative, high contrast |
| **Professional** | Healthcare | Calm blues and greens |
| **Professional** | Finance | Dark blue, trust-inducing |
| **Creative** | Playful | Bright, colorful, rounded |
| **Creative** | Retro | Nostalgic, warm colors |
| **Creative** | Neon | Cyberpunk, high contrast |
| **Creative** | Pastel | Soft, muted pastels |
| **Creative** | Brutalist | Raw, stark, minimal |
| **Minimal** | Minimal Light | Near-white, subtle |
| **Minimal** | Minimal Dark | Near-black, subtle |
| **Minimal** | Monochrome | Black and white only |
| **Seasonal** | Spring | Fresh greens, light |
| **Seasonal** | Summer | Warm yellows, bright |
| **Seasonal** | Autumn | Orange, brown, warm |
| **Seasonal** | Winter | Cool blues, icy |
| **Branded** | GitHub | GitHub's color scheme |
| **Branded** | Stripe | Stripe's color scheme |
| **Branded** | Linear | Linear's color scheme |
| **Branded** | Vercel | Vercel's color scheme |
| **Accessibility** | High Contrast | WCAG AAA compliant |
| **Accessibility** | Dyslexia Friendly | OpenDyslexic font, high readability |

### Theme Features

| Feature | Description |
|---------|-------------|
| **Light/Dark modes** | Each theme includes both modes |
| **CSS variables** | Tokens exported as CSS custom properties |
| **Tailwind config** | Auto-generated Tailwind theme |
| **Component tokens** | Semantic tokens for buttons, inputs, etc. |
| **Live preview** | Preview theme before applying |

### Theme Pricing Model

| Tier | Access |
|------|--------|
| **Lite (Open Source)** | Default theme only |
| **Hobby** | All base + minimal themes |
| **Pro** | All themes |
| **Enterprise** | Custom theme creation |

---

## 4. Templates

### What Is a Template?

A **Template** is a complete project scaffold with pre-configured structure, dependencies, and boilerplate code. Templates combine primitives, rigs, mods, and themes into ready-to-customize starting points.

```
Template includes:
├── File structure
├── Package dependencies
├── Pre-configured rigs
├── Optional mod integrations
├── Theme selection
├── Build configuration
└── Deployment setup
```

### Available Templates

| Category | Template | Backend | Rigs | Mods | Description |
|----------|----------|---------|------|------|-------------|
| **Static** | Landing Page | None | — | — | Marketing landing page |
| **Static** | Portfolio | None | Markdown, Image | — | Personal portfolio |
| **Static** | Documentation | None | Markdown, Code | — | Docs site with search |
| **Static** | Blog | GlyphCase | Markdown, Code | — | Static blog with SQLite |
| **App** | React SPA | None | Table, Chart | — | Single-page application |
| **App** | Dashboard | GlyphCase | Table, Chart, DatePicker | — | Analytics dashboard |
| **App** | Admin Panel | GlyphCase | Table, Form components | — | CRUD admin interface |
| **App** | Form Builder | GlyphCase | Form Rigset | — | Dynamic form creation |
| **SaaS** | SaaS Starter | Neon | Table, Chart | SaaS Modpack | Full SaaS boilerplate |
| **SaaS** | Multi-tenant | Neon | Table | clerk-mod, stripe-mod | Multi-tenant architecture |
| **Commerce** | E-commerce | Supabase | Table, Image | E-commerce Modpack | Online store |
| **Commerce** | Marketplace | Neon | Table, Image | stripe-mod, clerk-mod | Two-sided marketplace |
| **Commerce** | Booking | GlyphCase | DatePicker, Table | cal-mod, stripe-mod | Appointment booking |
| **Content** | CMS Blog | Sanity | RichText, Image | sanity-mod | CMS-powered blog |
| **Content** | Knowledge Base | GlyphCase | Markdown, TreeView | algolia-mod | Searchable docs |
| **Content** | Wiki | GlyphCase | RichText, TreeView | — | Collaborative wiki |
| **AI** | Chat Interface | GlyphCase | Markdown, Code | openai-mod | AI chat application |
| **AI** | AI Dashboard | Neon | Table, Chart | AI Modpack | AI model management |
| **Mobile** | Mobile App | Supabase | Native components | Mobile Modpack | Capacitor mobile app |
| **Mobile** | PWA | GlyphCase | — | — | Progressive web app |
| **Tool** | CLI Tool | None | — | — | Command-line utility |
| **Tool** | API Server | GlyphCase | — | — | REST API backend |
| **Tool** | Webhook Handler | GlyphCase | — | — | Webhook processing |
| **Internal** | Internal Tool | GlyphCase | Table, Form components | — | Company internal app |
| **Internal** | Ops Dashboard | GlyphCase | Table, Chart, Timeline | — | Operations monitoring |

### Template Structure Example

```
saas-starter/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── signup/
│   │   ├── (dashboard)/
│   │   │   ├── overview/
│   │   │   ├── settings/
│   │   │   └── billing/
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/           # Pink components
│   │   └── features/     # Feature components
│   ├── lib/
│   │   ├── db.ts         # Neon connection
│   │   ├── auth.ts       # Clerk setup
│   │   └── stripe.ts     # Stripe setup
│   └── schemas/          # Effect schemas
├── public/
├── punk.config.json
├── package.json
└── README.md
```

### Template Pricing Model

| Tier | Access |
|------|--------|
| **Lite (Open Source)** | 4 basic templates (static-site, react-spa, api-glyphcase, cli-tool) |
| **Hobby** | All static + app templates |
| **Pro** | All templates |
| **Enterprise** | Custom templates, white-label |

---

## Depot Architecture

### How Depot Works

```
┌─────────────────────────────────────────────────────────────────┐
│                     DEPOT REGISTRY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    CATALOG                               │   │
│  │                                                          │   │
│  │  Rigs: 50+    Mods: 40+    Themes: 25+    Templates: 30+ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 CDN / DISTRIBUTION                       │   │
│  │                                                          │   │
│  │  • Versioned packages                                    │   │
│  │  • Integrity hashes                                      │   │
│  │  • Regional edge caching                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MOHAWK VM                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  App manifest requests:                                         │
│  • 5 rigs                                                       │
│  • 3 mods                                                       │
│  • 1 theme                                                      │
│                                                                 │
│  Depot resolves → downloads → caches → loads                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### App Manifest

```json
{
  "name": "my-saas-app",
  "template": "saas-starter",
  "depot": {
    "rigs": [
      "Table",
      "Chart",
      "DatePicker",
      "RichText",
      "Command"
    ],
    "rigsets": [],
    "mods": [
      "clerk-mod",
      "stripe-mod",
      "neon-mod"
    ],
    "modpacks": [],
    "theme": "corporate"
  }
}
```

### Resolution & Caching

| Step | Action |
|------|--------|
| 1 | Parse app manifest |
| 2 | Resolve dependencies (rigsets → individual rigs, modpacks → individual mods) |
| 3 | Check local cache |
| 4 | Download missing packages from CDN |
| 5 | Verify integrity hashes |
| 6 | Load into VM |

---

## Lite vs Full Access

### What's Bundled with Lite

| Modality | Bundled | Available via Depot |
|----------|---------|---------------------|
| **Rigs** | Table, Chart, DatePicker, Code, Markdown | 50+ additional |
| **Rigsets** | None | All |
| **Mods** | None | 40+ |
| **Modpacks** | None | All |
| **Themes** | Default | 25+ |
| **Templates** | static-site, react-spa, api-glyphcase, cli-tool | 30+ additional |

### Feature Matrix

| Feature | Lite | Hobby | Pro | Enterprise |
|---------|------|-------|-----|------------|
| Core Rigs (5) | ✅ | ✅ | ✅ | ✅ |
| All Rigs | ❌ | ✅ | ✅ | ✅ |
| Rigsets | ❌ | ❌ | ✅ | ✅ |
| Mods (3) | ❌ | ✅ | ✅ | ✅ |
| All Mods | ❌ | ❌ | ✅ | ✅ |
| Modpacks | ❌ | ❌ | ✅ | ✅ |
| Basic Themes | ✅ | ✅ | ✅ | ✅ |
| All Themes | ❌ | ✅ | ✅ | ✅ |
| Basic Templates (4) | ✅ | ✅ | ✅ | ✅ |
| All Templates | ❌ | ✅ | ✅ | ✅ |
| Custom Rigs | ❌ | ❌ | ❌ | ✅ |
| Custom Mods | ❌ | ❌ | ❌ | ✅ |
| Custom Themes | ❌ | ❌ | ❌ | ✅ |
| Custom Templates | ❌ | ❌ | ❌ | ✅ |

---

## Summary

### Four Modalities

| Modality | Individual | Bundle | Purpose |
|----------|------------|--------|---------|
| **Rigs** | Component | Rigset | UI components |
| **Mods** | Capability | Modpack | Backend/services |
| **Themes** | Token set | — | Visual styling |
| **Templates** | Scaffold | — | Starting points |

### Depot Numbers (Approximate)

| Modality | Count |
|----------|-------|
| Rigs | 50+ |
| Rigsets | 7 |
| Mods | 40+ |
| Modpacks | 7 |
| Themes | 25+ |
| Templates | 30+ |

### Key Principle

**Lite gives you the tools to build. Depot gives you shortcuts and scale.**

---

*End of Depot Marketplace Document*
