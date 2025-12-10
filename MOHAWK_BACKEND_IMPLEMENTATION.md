# Mohawk Backend Implementation Guide

> **Decision**: Convex is the primary backend for Mohawk (SaaS, Docker, Desktop).
> This document provides complete implementation patterns for both Convex and Encore.ts.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Project Structure](#2-project-structure)
3. [Schema & Data Modeling](#3-schema--data-modeling)
4. [Authentication](#4-authentication)
5. [Core API Patterns](#5-core-api-patterns)
6. [Real-time Features](#6-real-time-features)
7. [File Storage](#7-file-storage)
8. [Background Jobs & Workflows](#8-background-jobs--workflows)
9. [AI Generation Pipeline](#9-ai-generation-pipeline)
10. [Billing & Subscriptions](#10-billing--subscriptions)
11. [Rate Limiting](#11-rate-limiting)
12. [Self-Hosting Configuration](#12-self-hosting-configuration)
13. [Testing Strategies](#13-testing-strategies)
14. [Migration Patterns](#14-migration-patterns)
15. [Deployment](#15-deployment)
16. [Convex Rules for AI Code Generation](#16-convex-rules-for-ai-code-generation)

---

## 1. Architecture Overview

### 1.1 Convex Architecture (Primary)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mohawk Platform                          │
├─────────────────────────────────────────────────────────────────┤
│  Frontend (React + TanStack Router)                             │
│  ├── useQuery() ──────────────────┐                             │
│  ├── useMutation() ───────────────┤                             │
│  └── useAction() ─────────────────┤                             │
├───────────────────────────────────┼─────────────────────────────┤
│                                   ▼                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Convex Backend                           ││
│  │  ┌──────────────┬──────────────┬──────────────────────────┐ ││
│  │  │   Queries    │  Mutations   │       Actions            │ ││
│  │  │  (reads)     │  (writes)    │  (side effects)          │ ││
│  │  └──────┬───────┴──────┬───────┴──────────┬───────────────┘ ││
│  │         │              │                  │                 ││
│  │         ▼              ▼                  ▼                 ││
│  │  ┌──────────────────────────┐  ┌────────────────────────┐  ││
│  │  │     Reactive Database    │  │    External Services   │  ││
│  │  │  (auto-updates clients)  │  │  - Anthropic API       │  ││
│  │  └──────────────────────────┘  │  - Stripe              │  ││
│  │                                │  - File Storage        │  ││
│  │  ┌──────────────────────────┐  └────────────────────────┘  ││
│  │  │   Scheduled Functions    │                               ││
│  │  │   (crons, intervals)     │                               ││
│  │  └──────────────────────────┘                               ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- **Reactive by default**: All queries auto-update when underlying data changes
- **TypeScript end-to-end**: Schema, functions, and client all typed
- **ACID transactions**: Serializable isolation for mutations
- **No SQL**: Document-based with indexed queries

### 1.2 Encore.ts Architecture (Alternative)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mohawk Platform                          │
├─────────────────────────────────────────────────────────────────┤
│  Frontend (React + TanStack Router)                             │
│  ├── fetch/axios ─────────────────┐                             │
│  └── WebSocket (manual) ──────────┤                             │
├───────────────────────────────────┼─────────────────────────────┤
│                                   ▼                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                   Encore.ts Backend                         ││
│  │  ┌──────────────────────────────────────────────────────┐   ││
│  │  │                  Rust Runtime                        │   ││
│  │  │  (7x throughput, 85% lower latency vs Node.js)       │   ││
│  │  └──────────────────────────────────────────────────────┘   ││
│  │  ┌──────────────┬──────────────┬──────────────────────────┐ ││
│  │  │  Services    │   Pub/Sub    │      Cron Jobs           │ ││
│  │  │  (REST APIs) │   (async)    │      (scheduled)         │ ││
│  │  └──────┬───────┴──────┬───────┴──────────┬───────────────┘ ││
│  │         │              │                  │                 ││
│  │         ▼              ▼                  ▼                 ││
│  │  ┌──────────────────────────┐  ┌────────────────────────┐  ││
│  │  │      PostgreSQL          │  │    External Services   │  ││
│  │  │  (full SQL, migrations)  │  │  - Anthropic API       │  ││
│  │  └──────────────────────────┘  │  - Stripe              │  ││
│  │                                │  - S3/R2               │  ││
│  │  ┌──────────────────────────┐  └────────────────────────┘  ││
│  │  │   Redis Cache            │                               ││
│  │  │   (optional)             │                               ││
│  │  └──────────────────────────┘                               ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- **Microservices-first**: Each service is independently deployable
- **Full SQL**: PostgreSQL with migrations
- **Rust runtime**: High performance with zero NPM dependencies
- **Manual real-time**: WebSocket implementation required

---

## 2. Project Structure

### 2.1 Convex Project Structure

```
mohawk/
├── convex/                          # Backend code
│   ├── _generated/                  # Auto-generated types
│   ├── schema.ts                    # Database schema
│   ├── auth.ts                      # Authentication functions
│   ├── auth.config.ts               # Auth provider config
│   ├── users/                       # User management
│   │   ├── queries.ts
│   │   └── mutations.ts
│   ├── projects/                    # Project management
│   │   ├── queries.ts
│   │   ├── mutations.ts
│   │   └── actions.ts
│   ├── generations/                 # AI generation pipeline
│   │   ├── queries.ts
│   │   ├── mutations.ts
│   │   ├── actions.ts
│   │   └── workflows.ts
│   ├── billing/                     # Stripe integration
│   │   ├── queries.ts
│   │   ├── mutations.ts
│   │   └── webhooks.ts
│   ├── http.ts                      # HTTP endpoints
│   ├── crons.ts                     # Scheduled jobs
│   └── lib/                         # Shared utilities
│       ├── validators.ts
│       └── rateLimiter.ts
├── src/                             # Frontend code
├── convex.json                      # Convex configuration
└── package.json
```

### 2.2 Encore.ts Project Structure

```
mohawk/
├── services/                        # Backend services
│   ├── users/                       # User service
│   │   ├── users.ts
│   │   ├── db.ts
│   │   └── migrations/
│   ├── projects/                    # Project service
│   ├── generations/                 # Generation service
│   │   ├── generations.ts
│   │   ├── worker.ts
│   │   └── pubsub.ts
│   ├── billing/                     # Billing service
│   └── storage/                     # Storage service
├── shared/                          # Shared code
├── src/                             # Frontend code
├── encore.app                       # Encore configuration
└── package.json
```

---

## 3. Schema & Data Modeling

### 3.1 Convex Schema

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // ============================================================
  // USERS & AUTHENTICATION
  // ============================================================
  users: defineTable({
    email: v.string(),
    name: v.optional(v.string()),
    avatarUrl: v.optional(v.string()),
    authProvider: v.union(
      v.literal("clerk"),
      v.literal("convex-auth"),
      v.literal("workos")
    ),
    authProviderId: v.string(),
    subscriptionTier: v.union(
      v.literal("free"),
      v.literal("pro"),
      v.literal("team"),
      v.literal("enterprise")
    ),
    stripeCustomerId: v.optional(v.string()),
    tokensUsedThisMonth: v.number(),
    tokenResetDate: v.number(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_email", ["email"])
    .index("by_auth_provider", ["authProvider", "authProviderId"])
    .index("by_stripe_customer", ["stripeCustomerId"]),

  // ============================================================
  // ORGANIZATIONS (Multi-tenancy)
  // ============================================================
  organizations: defineTable({
    name: v.string(),
    slug: v.string(),
    ownerId: v.id("users"),
    settings: v.object({
      allowedDomains: v.optional(v.array(v.string())),
      defaultProjectVisibility: v.union(
        v.literal("private"),
        v.literal("team"),
        v.literal("public")
      ),
    }),
    stripeCustomerId: v.optional(v.string()),
    subscriptionTier: v.union(v.literal("team"), v.literal("enterprise")),
    seatCount: v.number(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_slug", ["slug"])
    .index("by_owner", ["ownerId"]),

  organizationMembers: defineTable({
    organizationId: v.id("organizations"),
    userId: v.id("users"),
    role: v.union(
      v.literal("owner"),
      v.literal("admin"),
      v.literal("member"),
      v.literal("viewer")
    ),
    invitedBy: v.optional(v.id("users")),
    joinedAt: v.number(),
  })
    .index("by_organization", ["organizationId"])
    .index("by_user", ["userId"])
    .index("by_org_and_user", ["organizationId", "userId"]),

  // ============================================================
  // PROJECTS
  // ============================================================
  projects: defineTable({
    userId: v.id("users"),
    organizationId: v.optional(v.id("organizations")),
    name: v.string(),
    slug: v.string(),
    description: v.optional(v.string()),
    visibility: v.union(
      v.literal("private"),
      v.literal("team"),
      v.literal("public")
    ),
    framework: v.union(
      v.literal("react"),
      v.literal("vue"),
      v.literal("svelte"),
      v.literal("solid")
    ),
    cssFramework: v.union(
      v.literal("tailwind"),
      v.literal("pink"),
      v.literal("vanilla")
    ),
    status: v.union(
      v.literal("draft"),
      v.literal("generating"),
      v.literal("ready"),
      v.literal("deployed"),
      v.literal("archived")
    ),
    currentGenerationId: v.optional(v.id("generations")),
    deploymentUrl: v.optional(v.string()),
    viewCount: v.number(),
    forkCount: v.number(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_organization", ["organizationId"])
    .index("by_slug", ["userId", "slug"])
    .index("by_status", ["status"]),

  // ============================================================
  // GENERATIONS (AI generation jobs)
  // ============================================================
  generations: defineTable({
    projectId: v.id("projects"),
    userId: v.id("users"),
    prompt: v.string(),
    parentGenerationId: v.optional(v.id("generations")),
    model: v.string(),
    temperature: v.number(),
    maxTokens: v.number(),
    status: v.union(
      v.literal("pending"),
      v.literal("processing"),
      v.literal("completed"),
      v.literal("failed"),
      v.literal("cancelled")
    ),
    progress: v.number(),
    currentStep: v.optional(v.string()),
    generatedFiles: v.optional(v.array(v.object({
      path: v.string(),
      content: v.string(),
      language: v.string(),
    }))),
    errorMessage: v.optional(v.string()),
    inputTokens: v.number(),
    outputTokens: v.number(),
    totalCost: v.number(),
    startedAt: v.optional(v.number()),
    completedAt: v.optional(v.number()),
    createdAt: v.number(),
  })
    .index("by_project", ["projectId"])
    .index("by_user", ["userId"])
    .index("by_status", ["status"])
    .index("by_user_and_status", ["userId", "status"]),

  // ============================================================
  // GENERATION MESSAGES (Streaming updates)
  // ============================================================
  generationMessages: defineTable({
    generationId: v.id("generations"),
    role: v.union(
      v.literal("system"),
      v.literal("user"),
      v.literal("assistant"),
      v.literal("tool")
    ),
    content: v.string(),
    toolName: v.optional(v.string()),
    toolInput: v.optional(v.string()),
    toolOutput: v.optional(v.string()),
    tokenCount: v.optional(v.number()),
    createdAt: v.number(),
  })
    .index("by_generation", ["generationId"]),

  // ============================================================
  // FILE STORAGE
  // ============================================================
  projectFiles: defineTable({
    projectId: v.id("projects"),
    generationId: v.optional(v.id("generations")),
    path: v.string(),
    filename: v.string(),
    content: v.optional(v.string()),
    storageId: v.optional(v.id("_storage")),
    mimeType: v.string(),
    size: v.number(),
    hash: v.string(),
    version: v.number(),
    previousVersionId: v.optional(v.id("projectFiles")),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_project", ["projectId"])
    .index("by_project_and_path", ["projectId", "path"])
    .index("by_generation", ["generationId"])
    .index("by_hash", ["hash"]),

  // ============================================================
  // BILLING & USAGE
  // ============================================================
  usageRecords: defineTable({
    userId: v.id("users"),
    organizationId: v.optional(v.id("organizations")),
    type: v.union(
      v.literal("generation"),
      v.literal("storage"),
      v.literal("deployment")
    ),
    generationId: v.optional(v.id("generations")),
    projectId: v.optional(v.id("projects")),
    inputTokens: v.optional(v.number()),
    outputTokens: v.optional(v.number()),
    storageBytes: v.optional(v.number()),
    costCents: v.number(),
    billingPeriodStart: v.number(),
    billingPeriodEnd: v.number(),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_user_and_period", ["userId", "billingPeriodStart"]),

  subscriptions: defineTable({
    userId: v.optional(v.id("users")),
    organizationId: v.optional(v.id("organizations")),
    stripeSubscriptionId: v.string(),
    stripeCustomerId: v.string(),
    stripePriceId: v.string(),
    status: v.union(
      v.literal("active"),
      v.literal("past_due"),
      v.literal("canceled"),
      v.literal("incomplete"),
      v.literal("trialing")
    ),
    tier: v.union(
      v.literal("free"),
      v.literal("pro"),
      v.literal("team"),
      v.literal("enterprise")
    ),
    currentPeriodStart: v.number(),
    currentPeriodEnd: v.number(),
    cancelAtPeriodEnd: v.boolean(),
    monthlyTokenLimit: v.number(),
    monthlyStorageLimit: v.number(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_stripe_subscription", ["stripeSubscriptionId"])
    .index("by_stripe_customer", ["stripeCustomerId"]),
});
```

### 3.2 Encore.ts Schema (PostgreSQL)

```sql
-- services/users/migrations/001_create_users.up.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    avatar_url TEXT,
    auth_provider VARCHAR(50) NOT NULL,
    auth_provider_id VARCHAR(255) NOT NULL,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    stripe_customer_id VARCHAR(255),
    tokens_used_this_month BIGINT NOT NULL DEFAULT 0,
    token_reset_date TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT unique_auth_provider UNIQUE (auth_provider, auth_provider_id)
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe ON users(stripe_customer_id);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

```sql
-- services/projects/migrations/001_create_projects.up.sql
CREATE TYPE project_status AS ENUM ('draft', 'generating', 'ready', 'deployed', 'archived');
CREATE TYPE framework_type AS ENUM ('react', 'vue', 'svelte', 'solid');
CREATE TYPE css_framework_type AS ENUM ('tailwind', 'pink', 'vanilla');

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    organization_id UUID,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    status project_status NOT NULL DEFAULT 'draft',
    framework framework_type NOT NULL DEFAULT 'react',
    css_framework css_framework_type NOT NULL DEFAULT 'tailwind',
    current_generation_id UUID,
    deployment_url TEXT,
    view_count INTEGER NOT NULL DEFAULT 0,
    fork_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    CONSTRAINT unique_user_slug UNIQUE (user_id, slug)
);

CREATE INDEX idx_projects_user ON projects(user_id);
CREATE INDEX idx_projects_status ON projects(status);
```

```sql
-- services/generations/migrations/001_create_generations.up.sql
CREATE TYPE generation_status AS ENUM ('pending', 'processing', 'completed', 'failed', 'cancelled');

CREATE TABLE generations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    prompt TEXT NOT NULL,
    parent_generation_id UUID REFERENCES generations(id),
    model VARCHAR(100) NOT NULL,
    temperature DECIMAL(3,2) NOT NULL DEFAULT 0.7,
    max_tokens INTEGER NOT NULL DEFAULT 4096,
    status generation_status NOT NULL DEFAULT 'pending',
    progress INTEGER NOT NULL DEFAULT 0,
    current_step VARCHAR(255),
    generated_files JSONB,
    error_message TEXT,
    input_tokens BIGINT NOT NULL DEFAULT 0,
    output_tokens BIGINT NOT NULL DEFAULT 0,
    total_cost_cents INTEGER NOT NULL DEFAULT 0,
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_generations_project ON generations(project_id);
CREATE INDEX idx_generations_user ON generations(user_id);
CREATE INDEX idx_generations_status ON generations(status);
```

---

## 4. Authentication

### 4.1 Convex Auth (Built-in)

```typescript
// convex/auth.config.ts
import GitHub from "@auth/core/providers/github";
import Google from "@auth/core/providers/google";
import { convexAuth } from "@convex-dev/auth/server";
import { Password } from "@convex-dev/auth/providers/Password";

export const { auth, signIn, signOut, store } = convexAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID,
      clientSecret: process.env.GITHUB_CLIENT_SECRET,
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
    Password({
      verify: {
        sendVerificationEmail: async ({ email, token }) => {
          // Send via Resend
          await fetch("https://api.resend.com/emails", {
            method: "POST",
            headers: {
              Authorization: `Bearer ${process.env.RESEND_API_KEY}`,
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              from: "Mohawk <auth@mohawk.dev>",
              to: email,
              subject: "Verify your email",
              html: `Click to verify: https://mohawk.dev/verify?token=${token}`,
            }),
          });
        },
      },
    }),
  ],
});
```

```typescript
// convex/auth.ts
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";

export const currentUser = query({
  args: {},
  returns: v.union(
    v.object({
      _id: v.id("users"),
      email: v.string(),
      name: v.optional(v.string()),
      subscriptionTier: v.string(),
      tokensUsedThisMonth: v.number(),
    }),
    v.null()
  ),
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return null;
    
    return await ctx.db
      .query("users")
      .withIndex("by_auth_provider", (q) =>
        q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject)
      )
      .unique();
  },
});

export const upsertUser = mutation({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const existingUser = await ctx.db
      .query("users")
      .withIndex("by_auth_provider", (q) =>
        q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject)
      )
      .unique();
    
    if (existingUser) {
      await ctx.db.patch(existingUser._id, {
        email: identity.email!,
        name: identity.name,
        updatedAt: Date.now(),
      });
      return existingUser._id;
    }
    
    const now = Date.now();
    return await ctx.db.insert("users", {
      email: identity.email!,
      name: identity.name,
      authProvider: identity.issuer as "convex-auth",
      authProviderId: identity.subject,
      subscriptionTier: "free",
      tokensUsedThisMonth: 0,
      tokenResetDate: now,
      createdAt: now,
      updatedAt: now,
    });
  },
});
```

### 4.2 Encore.ts Auth

```typescript
// services/users/users.ts
import { api, APIError } from "encore.dev/api";
import { SQLDatabase } from "encore.dev/storage/sqldb";
import { Secret } from "encore.dev/config";
import * as jwt from "jsonwebtoken";

const db = new SQLDatabase("users", { migrations: "./migrations" });
const jwtSecret = new Secret("JWT_SECRET");

export async function getAuthContext(authorization: string | undefined) {
  if (!authorization?.startsWith("Bearer ")) return null;
  
  try {
    const decoded = jwt.verify(authorization.slice(7), jwtSecret()) as {
      sub: string;
      email: string;
    };
    return { userId: decoded.sub, email: decoded.email };
  } catch {
    return null;
  }
}

export const getCurrentUser = api(
  { expose: true, method: "GET", path: "/users/me" },
  async ({ authorization }: { authorization: string }): Promise<User | null> => {
    const auth = await getAuthContext(authorization);
    if (!auth) return null;
    
    return await db.queryRow`
      SELECT id, email, name, subscription_tier as "subscriptionTier",
             tokens_used_this_month as "tokensUsedThisMonth"
      FROM users WHERE id = ${auth.userId}
    `;
  }
);

export const oauthCallback = api(
  { expose: true, method: "POST", path: "/auth/callback" },
  async (params: { provider: string; code: string; redirectUri: string }) => {
    // Exchange code for tokens, get user info, upsert user, return JWT
    const user = await db.queryRow`
      INSERT INTO users (email, name, auth_provider, auth_provider_id)
      VALUES ($1, $2, $3, $4)
      ON CONFLICT (auth_provider, auth_provider_id) 
      DO UPDATE SET email = EXCLUDED.email, name = EXCLUDED.name
      RETURNING id, email
    `;
    
    const token = jwt.sign({ sub: user.id, email: user.email }, jwtSecret(), { expiresIn: "7d" });
    return { token, user };
  }
);
```

---

## 5. Core API Patterns

### 5.1 Convex Query/Mutation/Action

```typescript
// convex/projects/queries.ts
import { query } from "../_generated/server";
import { v } from "convex/values";
import { paginationOptsValidator } from "convex/server";

export const list = query({
  args: {
    paginationOpts: paginationOptsValidator,
    status: v.optional(v.string()),
  },
  returns: v.object({
    page: v.array(v.object({
      _id: v.id("projects"),
      name: v.string(),
      slug: v.string(),
      status: v.string(),
      framework: v.string(),
      updatedAt: v.number(),
    })),
    isDone: v.boolean(),
    continueCursor: v.string(),
  }),
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const user = await ctx.db
      .query("users")
      .withIndex("by_auth_provider", (q) =>
        q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject)
      )
      .unique();
    
    if (!user) throw new Error("User not found");
    
    let projectsQuery = ctx.db
      .query("projects")
      .withIndex("by_user", (q) => q.eq("userId", user._id));
    
    if (args.status) {
      projectsQuery = projectsQuery.filter((q) => q.eq(q.field("status"), args.status));
    }
    
    return await projectsQuery.order("desc").paginate(args.paginationOpts);
  },
});
```

```typescript
// convex/projects/mutations.ts
import { mutation } from "../_generated/server";
import { v } from "convex/values";

export const create = mutation({
  args: {
    name: v.string(),
    description: v.optional(v.string()),
    framework: v.union(v.literal("react"), v.literal("vue"), v.literal("svelte"), v.literal("solid")),
    cssFramework: v.union(v.literal("tailwind"), v.literal("pink"), v.literal("vanilla")),
  },
  returns: v.id("projects"),
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const user = await ctx.db
      .query("users")
      .withIndex("by_auth_provider", (q) =>
        q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject)
      )
      .unique();
    
    if (!user) throw new Error("User not found");
    
    // Generate unique slug
    const baseSlug = args.name.toLowerCase().replace(/[^a-z0-9]+/g, "-");
    let slug = baseSlug;
    let counter = 1;
    
    while (await ctx.db.query("projects").withIndex("by_slug", (q) => 
      q.eq("userId", user._id).eq("slug", slug)).unique()) {
      slug = `${baseSlug}-${counter++}`;
    }
    
    const now = Date.now();
    return await ctx.db.insert("projects", {
      userId: user._id,
      name: args.name,
      slug,
      description: args.description,
      visibility: "private",
      framework: args.framework,
      cssFramework: args.cssFramework,
      status: "draft",
      viewCount: 0,
      forkCount: 0,
      createdAt: now,
      updatedAt: now,
    });
  },
});
```

```typescript
// convex/projects/actions.ts
import { action } from "../_generated/server";
import { v } from "convex/values";
import { api, internal } from "../_generated/api";

export const fork = action({
  args: { sourceProjectId: v.id("projects"), name: v.optional(v.string()) },
  returns: v.id("projects"),
  handler: async (ctx, args) => {
    const sourceProject = await ctx.runQuery(api.projects.queries.get, {
      projectId: args.sourceProjectId,
    });
    
    if (!sourceProject) throw new Error("Project not found");
    
    const newProjectId = await ctx.runMutation(api.projects.mutations.create, {
      name: args.name ?? `${sourceProject.name} (fork)`,
      framework: sourceProject.framework,
      cssFramework: sourceProject.cssFramework,
    });
    
    await ctx.runMutation(internal.projects.mutations.copyFiles, {
      sourceProjectId: args.sourceProjectId,
      targetProjectId: newProjectId,
    });
    
    return newProjectId;
  },
});
```

### 5.2 Encore.ts API Pattern

```typescript
// services/projects/projects.ts
import { api, APIError } from "encore.dev/api";
import { SQLDatabase } from "encore.dev/storage/sqldb";
import { getAuthContext } from "../users/users";

const db = new SQLDatabase("projects", { migrations: "./migrations" });

export const listProjects = api(
  { expose: true, method: "GET", path: "/projects" },
  async (params: {
    authorization: string;
    page?: number;
    pageSize?: number;
    status?: string;
  }) => {
    const auth = await getAuthContext(params.authorization);
    if (!auth) throw APIError.unauthenticated("Not authenticated");
    
    const page = params.page ?? 1;
    const pageSize = Math.min(params.pageSize ?? 20, 100);
    const offset = (page - 1) * pageSize;
    
    const projects = await db.query`
      SELECT id, name, slug, status, framework, updated_at
      FROM projects
      WHERE user_id = ${auth.userId}
      ${params.status ? db.rawQuery`AND status = ${params.status}` : db.rawQuery``}
      ORDER BY updated_at DESC
      LIMIT ${pageSize} OFFSET ${offset}
    `;
    
    return { projects, page, pageSize };
  }
);

export const createProject = api(
  { expose: true, method: "POST", path: "/projects" },
  async (params: {
    authorization: string;
    name: string;
    framework: string;
    cssFramework: string;
  }) => {
    const auth = await getAuthContext(params.authorization);
    if (!auth) throw APIError.unauthenticated("Not authenticated");
    
    const slug = params.name.toLowerCase().replace(/[^a-z0-9]+/g, "-");
    
    return await db.queryRow`
      INSERT INTO projects (user_id, name, slug, framework, css_framework)
      VALUES (${auth.userId}, ${params.name}, ${slug}, ${params.framework}, ${params.cssFramework})
      RETURNING id, name, slug, status, framework
    `;
  }
);
```

---

## 6. Real-time Features

### 6.1 Convex Real-time (Native - Zero Config)

```typescript
// convex/generations/queries.ts - Auto-updates clients!
export const watchGeneration = query({
  args: { generationId: v.id("generations") },
  returns: v.union(
    v.object({
      _id: v.id("generations"),
      status: v.string(),
      progress: v.number(),
      currentStep: v.union(v.string(), v.null()),
      errorMessage: v.union(v.string(), v.null()),
    }),
    v.null()
  ),
  handler: async (ctx, args) => {
    const generation = await ctx.db.get(args.generationId);
    if (!generation) return null;
    return {
      _id: generation._id,
      status: generation.status,
      progress: generation.progress,
      currentStep: generation.currentStep ?? null,
      errorMessage: generation.errorMessage ?? null,
    };
  },
});

export const watchMessages = query({
  args: { generationId: v.id("generations") },
  returns: v.array(v.object({
    _id: v.id("generationMessages"),
    role: v.string(),
    content: v.string(),
    createdAt: v.number(),
  })),
  handler: async (ctx, args) => {
    return await ctx.db
      .query("generationMessages")
      .withIndex("by_generation", (q) => q.eq("generationId", args.generationId))
      .order("asc")
      .collect();
  },
});
```

```tsx
// Frontend: src/components/GenerationProgress.tsx
import { useQuery } from "convex/react";
import { api } from "../convex/_generated/api";

export function GenerationProgress({ generationId }) {
  // These auto-update when data changes!
  const generation = useQuery(api.generations.queries.watchGeneration, { generationId });
  const messages = useQuery(api.generations.queries.watchMessages, { generationId });
  
  if (!generation) return <div>Loading...</div>;
  
  return (
    <div>
      <div className="w-full bg-gray-200 rounded-full h-2">
        <div className="bg-blue-600 h-2 rounded-full" style={{ width: `${generation.progress}%` }} />
      </div>
      {generation.currentStep && <p>{generation.currentStep}</p>}
      <div>
        {messages?.map((msg) => (
          <div key={msg._id} className={msg.role === "assistant" ? "bg-blue-50" : "bg-white"}>
            <pre>{msg.content}</pre>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 6.2 Encore.ts Real-time (Manual WebSocket)

```typescript
// services/generations/realtime.ts
import { WebSocketHandler } from "encore.dev/api";
import { Subscription } from "encore.dev/pubsub";
import { generationUpdates } from "./pubsub";

const connections = new Map<string, Set<WebSocket>>();

export const generationSocket: WebSocketHandler = {
  path: "/ws/generations/:generationId",
  
  async onConnect(ws, { generationId, authorization }) {
    const auth = await getAuthContext(authorization);
    if (!auth) { ws.close(4001, "Unauthorized"); return; }
    
    if (!connections.has(generationId)) connections.set(generationId, new Set());
    connections.get(generationId)!.add(ws);
    
    // Send current state
    const state = await db.queryRow`SELECT status, progress FROM generations WHERE id = ${generationId}`;
    ws.send(JSON.stringify({ type: "state", data: state }));
  },
  
  onMessage(ws, message) { if (message === "ping") ws.send("pong"); },
  
  onClose(ws, { generationId }) {
    connections.get(generationId)?.delete(ws);
    if (connections.get(generationId)?.size === 0) connections.delete(generationId);
  },
};

// Broadcast updates from Pub/Sub
new Subscription(generationUpdates, "broadcast", {
  handler: async (event) => {
    const conns = connections.get(event.generationId);
    if (!conns) return;
    const message = JSON.stringify({ type: event.type, data: event.data });
    for (const ws of conns) ws.send(message);
  },
});
```

---

## 7. File Storage

### 7.1 Convex File Storage

```typescript
// convex/storage/mutations.ts
export const generateUploadUrl = mutation({
  args: {},
  returns: v.string(),
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    return await ctx.storage.generateUploadUrl();
  },
});

export const saveTextFile = mutation({
  args: {
    projectId: v.id("projects"),
    path: v.string(),
    filename: v.string(),
    content: v.string(),
    mimeType: v.string(),
  },
  returns: v.id("projectFiles"),
  handler: async (ctx, args) => {
    const now = Date.now();
    const size = new TextEncoder().encode(args.content).length;
    
    const existingFile = await ctx.db
      .query("projectFiles")
      .withIndex("by_project_and_path", (q) =>
        q.eq("projectId", args.projectId).eq("path", args.path)
      )
      .unique();
    
    if (existingFile) {
      await ctx.db.patch(existingFile._id, {
        content: args.content,
        size,
        version: existingFile.version + 1,
        updatedAt: now,
      });
      return existingFile._id;
    }
    
    return await ctx.db.insert("projectFiles", {
      projectId: args.projectId,
      path: args.path,
      filename: args.filename,
      content: args.content,
      mimeType: args.mimeType,
      size,
      hash: "placeholder",
      version: 1,
      createdAt: now,
      updatedAt: now,
    });
  },
});
```

### 7.2 Encore.ts File Storage (S3)

```typescript
// services/storage/storage.ts
import { Bucket } from "encore.dev/storage/objects";

const filesBucket = new Bucket("project-files", { versioned: true });

export const getUploadUrl = api(
  { expose: true, method: "POST", path: "/storage/upload-url" },
  async (params: { authorization: string; projectId: string; filename: string; mimeType: string }) => {
    const auth = await getAuthContext(params.authorization);
    if (!auth) throw APIError.unauthenticated("Not authenticated");
    
    const key = `${params.projectId}/${Date.now()}-${params.filename}`;
    const uploadUrl = await filesBucket.signedUploadUrl(key, { expiresIn: 3600, contentType: params.mimeType });
    return { uploadUrl, key };
  }
);

export const getDownloadUrl = api(
  { expose: true, method: "GET", path: "/storage/download/:fileId" },
  async (params: { fileId: string; authorization: string }) => {
    const auth = await getAuthContext(params.authorization);
    if (!auth) throw APIError.unauthenticated("Not authenticated");
    
    const file = await db.queryRow`SELECT storage_key FROM project_files WHERE id = ${params.fileId}`;
    if (!file?.storage_key) throw APIError.notFound("File not found");
    
    return { downloadUrl: await filesBucket.signedDownloadUrl(file.storage_key, { expiresIn: 3600 }) };
  }
);
```

---

## 8. Background Jobs & Workflows

### 8.1 Convex Workflows (Durable)

```typescript
// convex/generations/workflows.ts
import { workflow, workflowMutation } from "@convex-dev/workflow";
import { components } from "../_generated/api";

export const generationWorkflow = workflow(
  components.workflow,
  {
    args: {
      generationId: v.id("generations"),
      projectId: v.id("projects"),
      prompt: v.string(),
      model: v.string(),
    },
  },
  async (step, args) => {
    // Step 1: Prepare context
    await step.runMutation(internal.generations.mutations.updateStatus, {
      generationId: args.generationId,
      progress: 5,
      currentStep: "Preparing context...",
    });
    
    const context = await step.runAction(internal.generations.actions.prepareContext, {
      projectId: args.projectId,
    });
    
    // Step 2: Generate code
    await step.runMutation(internal.generations.mutations.updateStatus, {
      generationId: args.generationId,
      progress: 20,
      currentStep: "Generating code...",
    });
    
    const generated = await step.runAction(internal.generations.actions.generateCode, {
      generationId: args.generationId,
      context,
      model: args.model,
    });
    
    // Step 3: Validate
    await step.runMutation(internal.generations.mutations.updateStatus, {
      generationId: args.generationId,
      progress: 70,
      currentStep: "Validating...",
    });
    
    // Step 4: Save files
    await step.runMutation(internal.generations.mutations.saveGeneratedFiles, {
      generationId: args.generationId,
      projectId: args.projectId,
      files: generated.files,
    });
    
    // Step 5: Complete
    await step.runMutation(internal.generations.mutations.updateStatus, {
      generationId: args.generationId,
      status: "completed",
      progress: 100,
    });
    
    return { success: true, fileCount: generated.files.length };
  }
);

export const startGeneration = workflowMutation(
  components.workflow,
  {
    args: { projectId: v.id("projects"), prompt: v.string(), model: v.optional(v.string()) },
    returns: v.id("generations"),
  },
  async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    // Create generation record
    const generationId = await ctx.db.insert("generations", {
      projectId: args.projectId,
      userId: user._id,
      prompt: args.prompt,
      model: args.model ?? "claude-sonnet-4-20250514",
      status: "pending",
      progress: 0,
      inputTokens: 0,
      outputTokens: 0,
      totalCost: 0,
      createdAt: Date.now(),
    });
    
    // Start workflow
    await ctx.workflow.start(generationWorkflow, {
      generationId,
      projectId: args.projectId,
      prompt: args.prompt,
      model: args.model ?? "claude-sonnet-4-20250514",
    });
    
    return generationId;
  }
);
```

### 8.2 Encore.ts Background Jobs

```typescript
// services/generations/worker.ts
import { Topic, Subscription } from "encore.dev/pubsub";
import { CronJob } from "encore.dev/cron";

export const generationJobs = new Topic<GenerationJob>("generation-jobs", {
  deliveryGuarantee: "at-least-once",
});

new Subscription(generationJobs, "process-generations", {
  handler: async (job) => {
    try {
      await processGeneration(job);
    } catch (error) {
      await db.exec`UPDATE generations SET status = 'failed', error_message = ${error.message} WHERE id = ${job.generationId}`;
      throw error;
    }
  },
  retryPolicy: { maxRetries: 3, initialBackoff: 1000, maxBackoff: 30000 },
});

async function processGeneration(job: GenerationJob) {
  // Update status
  await db.exec`UPDATE generations SET status = 'processing', progress = 5, current_step = 'Preparing...' WHERE id = ${job.generationId}`;
  
  // Call Claude API
  const response = await anthropic.messages.create({
    model: job.model,
    max_tokens: job.maxTokens,
    messages: [{ role: "user", content: job.prompt }],
  });
  
  // Save files and complete
  await db.exec`UPDATE generations SET status = 'completed', progress = 100 WHERE id = ${job.generationId}`;
}

// Monthly token reset
new CronJob("reset-tokens", {
  schedule: "0 0 1 * *",
  endpoint: api({ expose: false }, async () => {
    await db.exec`UPDATE users SET tokens_used_this_month = 0, token_reset_date = NOW()`;
  }),
});
```

---

## 9. AI Generation Pipeline

```typescript
// convex/generations/actions.ts
import { internalAction } from "../_generated/server";
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

export const generateCode = internalAction({
  args: {
    generationId: v.id("generations"),
    context: v.object({ systemPrompt: v.string(), existingFiles: v.array(v.any()) }),
    model: v.string(),
  },
  handler: async (ctx, args) => {
    const generation = await ctx.runQuery(internal.generations.queries.getInternal, {
      generationId: args.generationId,
    });
    
    const response = await anthropic.messages.create({
      model: args.model,
      max_tokens: 8192,
      system: args.context.systemPrompt,
      messages: [{ role: "user", content: buildPrompt(generation.prompt, args.context.existingFiles) }],
    });
    
    const files = parseGeneratedFiles(response.content);
    
    await ctx.runMutation(internal.generations.mutations.addMessage, {
      generationId: args.generationId,
      role: "assistant",
      content: response.content.filter(b => b.type === "text").map(b => b.text).join("\n"),
      tokenCount: response.usage.output_tokens,
    });
    
    return {
      files,
      inputTokens: response.usage.input_tokens,
      outputTokens: response.usage.output_tokens,
    };
  },
});

function parseGeneratedFiles(content) {
  const files = [];
  const text = content.filter(b => b.type === "text").map(b => b.text).join("\n");
  const regex = /```(\w+)\s+([^\n]+)\n([\s\S]*?)```/g;
  let match;
  while ((match = regex.exec(text)) !== null) {
    files.push({ language: match[1], path: match[2].trim(), content: match[3].trim() });
  }
  return files;
}
```

---

## 10. Billing & Subscriptions

### 10.1 Convex Stripe Integration

```typescript
// convex/billing/mutations.ts
export const createCheckoutSession = mutation({
  args: { priceId: v.string(), successUrl: v.string(), cancelUrl: v.string() },
  returns: v.string(),
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const user = await ctx.db.query("users")
      .withIndex("by_auth_provider", q => q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject))
      .unique();
    
    return await ctx.scheduler.runAfter(0, internal.billing.actions.createStripeCheckout, {
      userId: user._id,
      email: user.email,
      stripeCustomerId: user.stripeCustomerId,
      priceId: args.priceId,
      successUrl: args.successUrl,
      cancelUrl: args.cancelUrl,
    });
  },
});

// convex/billing/actions.ts
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export const createStripeCheckout = internalAction({
  args: { userId: v.id("users"), email: v.string(), stripeCustomerId: v.optional(v.string()), priceId: v.string(), successUrl: v.string(), cancelUrl: v.string() },
  returns: v.string(),
  handler: async (ctx, args) => {
    let customerId = args.stripeCustomerId;
    
    if (!customerId) {
      const customer = await stripe.customers.create({ email: args.email, metadata: { convexUserId: args.userId } });
      customerId = customer.id;
      await ctx.runMutation(internal.billing.mutations.updateStripeCustomer, { userId: args.userId, stripeCustomerId: customerId });
    }
    
    const session = await stripe.checkout.sessions.create({
      customer: customerId,
      mode: "subscription",
      line_items: [{ price: args.priceId, quantity: 1 }],
      success_url: args.successUrl,
      cancel_url: args.cancelUrl,
    });
    
    return session.url!;
  },
});

// convex/http.ts - Stripe webhook
http.route({
  path: "/webhooks/stripe",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const signature = request.headers.get("stripe-signature");
    const body = await request.text();
    const event = stripe.webhooks.constructEvent(body, signature!, process.env.STRIPE_WEBHOOK_SECRET!);
    
    await ctx.runMutation(internal.billing.mutations.handleWebhookEvent, { type: event.type, data: event.data });
    return new Response("OK");
  }),
});
```

---

## 11. Rate Limiting

### 11.1 Convex Rate Limiting

```typescript
// convex/lib/rateLimiter.ts
import { RateLimiter, MINUTE, HOUR } from "@convex-dev/ratelimiter";
import { components } from "../_generated/api";

export const rateLimiter = new RateLimiter(components.rateLimiter, {
  generationsPerMinute: { kind: "token bucket", rate: 5, period: MINUTE, capacity: 10 },
  generationsPerHour: { kind: "fixed window", rate: 50, period: HOUR },
  apiRequestsPerMinute: { kind: "token bucket", rate: 100, period: MINUTE, capacity: 200 },
});

// Usage in mutation
export const start = mutation({
  args: { projectId: v.id("projects"), prompt: v.string() },
  handler: async (ctx, args) => {
    const user = await getAuthenticatedUser(ctx);
    
    const minuteLimit = await rateLimiter.limit(ctx, "generationsPerMinute", { key: user._id });
    if (!minuteLimit.ok) throw new Error(`Rate limit. Retry in ${Math.ceil(minuteLimit.retryAfter / 1000)}s`);
    
    const hourLimit = await rateLimiter.limit(ctx, "generationsPerHour", { key: user._id });
    if (!hourLimit.ok) throw new Error(`Hourly limit. Retry in ${Math.ceil(hourLimit.retryAfter / 60000)}m`);
    
    // Continue with generation...
  },
});
```

---

## 12. Self-Hosting Configuration

### 12.1 Convex Self-Hosted (Docker)

```yaml
# docker-compose.yml
version: "3.8"
services:
  convex:
    image: convex/convex-local-backend:latest
    ports:
      - "3210:3210"  # Backend API
      - "3211:3211"  # Dashboard
    environment:
      CONVEX_BACKEND_SQLITE_PATH: /data/convex.db
      # Or PostgreSQL: CONVEX_BACKEND_DATABASE_URL: postgres://user:pass@postgres:5432/convex
      CONVEX_SITE_URL: https://mohawk.example.com
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    volumes:
      - convex-data:/data
      - ./convex:/app/convex:ro
    restart: unless-stopped

  postgres:  # Optional
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: convex
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: convex
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  convex-data:
  postgres-data:
```

### 12.2 Encore.ts Self-Hosted (Docker)

```yaml
# docker-compose.yml
version: "3.8"
services:
  mohawk:
    build: .
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: postgres://mohawk:${POSTGRES_PASSWORD}@postgres:5432/mohawk
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      S3_ENDPOINT: ${S3_ENDPOINT}
      S3_ACCESS_KEY_ID: ${S3_ACCESS_KEY_ID}
      S3_SECRET_ACCESS_KEY: ${S3_SECRET_ACCESS_KEY}
    depends_on: [postgres, redis]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: mohawk
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: mohawk
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

### 12.3 Desktop (Electron + Local Convex)

```typescript
// desktop/main.ts
import { app, BrowserWindow } from "electron";
import { spawn } from "child_process";
import path from "path";

let convexProcess;

async function startLocalConvex() {
  const dataDir = path.join(app.getPath("userData"), "convex-data");
  convexProcess = spawn("convex-local-backend", ["--data-dir", dataDir, "--port", "3210"]);
  await new Promise(resolve => setTimeout(resolve, 2000));
  return "http://localhost:3210";
}

app.whenReady().then(async () => {
  const convexUrl = await startLocalConvex();
  const win = new BrowserWindow({ width: 1400, height: 900 });
  win.webContents.on("did-finish-load", () => win.webContents.send("convex-url", convexUrl));
  win.loadFile("index.html");
});

app.on("before-quit", () => convexProcess?.kill());
```

---

## 13. Testing Strategies

### 13.1 Convex Testing

```typescript
// convex/tests/projects.test.ts
import { convexTest } from "convex-test";
import { expect, test, describe } from "vitest";
import { api } from "../_generated/api";
import schema from "../schema";

describe("Projects", () => {
  const t = convexTest(schema);
  
  test("create project", async () => {
    const asUser = t.withIdentity({ email: "test@example.com", subject: "user-123", issuer: "convex-auth" });
    await asUser.mutation(api.auth.upsertUser, {});
    
    const projectId = await asUser.mutation(api.projects.mutations.create, {
      name: "Test Project",
      framework: "react",
      cssFramework: "tailwind",
    });
    
    expect(projectId).toBeDefined();
    
    const project = await asUser.query(api.projects.queries.get, { projectId });
    expect(project?.name).toBe("Test Project");
  });
  
  test("cannot access other user's project", async () => {
    const asUser1 = t.withIdentity({ subject: "user-1", issuer: "convex-auth", email: "u1@example.com" });
    const asUser2 = t.withIdentity({ subject: "user-2", issuer: "convex-auth", email: "u2@example.com" });
    
    await asUser1.mutation(api.auth.upsertUser, {});
    await asUser2.mutation(api.auth.upsertUser, {});
    
    const projectId = await asUser1.mutation(api.projects.mutations.create, { name: "Private", framework: "react", cssFramework: "tailwind" });
    
    await expect(asUser2.mutation(api.projects.mutations.update, { projectId, name: "Hacked!" })).rejects.toThrow("Not authorized");
  });
});
```

---

## 14. Migration Patterns

### 14.1 Convex Migrations

```typescript
// convex/migrations/001_add_organizations.ts
import { internalMutation } from "../_generated/server";

export const addOrganizationSupport = internalMutation({
  args: {},
  handler: async (ctx) => {
    const users = await ctx.db.query("users").collect();
    
    for (const user of users) {
      // Create personal org
      const orgId = await ctx.db.insert("organizations", {
        name: `${user.name ?? user.email}'s Workspace`,
        slug: `user-${user._id}`,
        ownerId: user._id,
        settings: { defaultProjectVisibility: "private" },
        subscriptionTier: "team",
        seatCount: 1,
        createdAt: Date.now(),
        updatedAt: Date.now(),
      });
      
      // Add as owner
      await ctx.db.insert("organizationMembers", { organizationId: orgId, userId: user._id, role: "owner", joinedAt: Date.now() });
      
      // Update projects
      const projects = await ctx.db.query("projects").withIndex("by_user", q => q.eq("userId", user._id)).collect();
      for (const project of projects) {
        await ctx.db.patch(project._id, { organizationId: orgId });
      }
    }
  },
});
```

---

## 15. Deployment

### 15.1 Convex Cloud Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy to Convex
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run typecheck
      - run: npm test
      - run: npx convex deploy --prod
        env:
          CONVEX_DEPLOY_KEY: ${{ secrets.CONVEX_DEPLOY_KEY }}
```

### 15.2 Encore.ts Deployment

```yaml
# .github/workflows/deploy-encore.yml
name: Deploy to Encore
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm test
      - run: curl -L https://encore.dev/install.sh | bash
      - run: encore deploy --env=prod
        env:
          ENCORE_AUTH_KEY: ${{ secrets.ENCORE_AUTH_KEY }}
```

---

## 16. Convex Rules for AI Code Generation

> **CRITICAL**: Include this section in `convex-mod/knowledge/convex-rules.md` for Ska to generate correct Convex code.

### 16.1 Function Syntax Rules

```typescript
// ✅ CORRECT: New syntax with args/returns/handler
export const myQuery = query({
  args: { id: v.id("users") },
  returns: v.object({ name: v.string(), email: v.string() }),
  handler: async (ctx, args) => {
    return await ctx.db.get(args.id);
  },
});

// ❌ WRONG: Old syntax (deprecated)
export const myQuery = query(async (ctx, args) => {
  return await ctx.db.get(args.id);
});
```

### 16.2 Validator Reference

```typescript
// Primitives
v.string()           // string
v.number()           // number
v.boolean()          // boolean
v.null()             // null
v.any()              // any (avoid if possible)

// Document references
v.id("tableName")    // Id<"tableName">

// Optional fields
v.optional(v.string())  // string | undefined

// Unions (enums)
v.union(v.literal("a"), v.literal("b"), v.literal("c"))

// Arrays
v.array(v.string())  // string[]

// Objects
v.object({
  name: v.string(),
  age: v.number(),
  email: v.optional(v.string()),
})

// Records (dynamic keys)
v.record(v.string(), v.number())  // Record<string, number>
```

### 16.3 Query Best Practices

```typescript
// ✅ CORRECT: Use withIndex for filtered queries
const users = await ctx.db
  .query("users")
  .withIndex("by_email", (q) => q.eq("email", email))
  .unique();

// ❌ WRONG: Using filter without index (slow)
const users = await ctx.db
  .query("users")
  .filter((q) => q.eq(q.field("email"), email))
  .unique();

// Getting single result
.unique()  // Returns T | null, throws if multiple
.first()   // Returns T | null, returns first if multiple

// Getting all results
.collect() // Returns T[]

// Pagination
const results = await ctx.db
  .query("items")
  .paginate(args.paginationOpts);
// Returns { page: T[], isDone: boolean, continueCursor: string }
```

### 16.4 Mutation Best Practices

```typescript
// Insert
const id = await ctx.db.insert("users", { name: "John", email: "john@example.com" });

// Update specific fields (preferred)
await ctx.db.patch(userId, { name: "Jane" });

// Replace entire document
await ctx.db.replace(userId, { name: "Jane", email: "jane@example.com" });

// Delete
await ctx.db.delete(userId);
```

### 16.5 Action Best Practices

```typescript
// Actions can call external APIs
export const callExternalApi = action({
  args: { prompt: v.string() },
  returns: v.string(),
  handler: async (ctx, args) => {
    // ✅ External API calls allowed
    const response = await fetch("https://api.example.com", {
      method: "POST",
      body: JSON.stringify({ prompt: args.prompt }),
    });
    return await response.text();
  },
});

// ❌ WRONG: Actions cannot access ctx.db directly
export const wrongAction = action({
  handler: async (ctx, args) => {
    await ctx.db.get(someId); // ERROR!
  },
});

// ✅ CORRECT: Use runQuery/runMutation in actions
export const correctAction = action({
  handler: async (ctx, args) => {
    const user = await ctx.runQuery(api.users.get, { id: args.userId });
    await ctx.runMutation(api.users.update, { id: args.userId, name: "New" });
  },
});
```

### 16.6 Schema Best Practices

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    // Required fields
    email: v.string(),
    name: v.string(),
    
    // Optional fields
    avatarUrl: v.optional(v.string()),
    
    // Enum fields
    role: v.union(v.literal("admin"), v.literal("user"), v.literal("guest")),
    
    // Timestamps (use numbers, not Date)
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    // Index naming: by_<field> or by_<field1>_and_<field2>
    .index("by_email", ["email"])
    .index("by_role", ["role"]),
    
  // For full-text search
  posts: defineTable({
    title: v.string(),
    content: v.string(),
    authorId: v.id("users"),
  })
    .index("by_author", ["authorId"])
    .searchIndex("search_content", { searchField: "content", filterFields: ["authorId"] }),
});
```

### 16.7 HTTP Endpoints

```typescript
// convex/http.ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { internal } from "./_generated/api";

const http = httpRouter();

http.route({
  path: "/webhooks/stripe",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const body = await request.json();
    await ctx.runMutation(internal.billing.handleWebhook, { data: body });
    return new Response("OK", { status: 200 });
  }),
});

http.route({
  path: "/api/public/:id",
  method: "GET",
  handler: httpAction(async (ctx, request) => {
    const url = new URL(request.url);
    const id = url.pathname.split("/").pop();
    const data = await ctx.runQuery(api.public.get, { id });
    return new Response(JSON.stringify(data), {
      headers: { "Content-Type": "application/json" },
    });
  }),
});

export default http;
```

### 16.8 Scheduled Functions (Crons)

```typescript
// convex/crons.ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

// Run every hour
crons.interval("cleanup-temp-files", { hours: 1 }, internal.storage.cleanupTempFiles);

// Run at specific time (cron syntax)
crons.cron("monthly-billing", "0 0 1 * *", internal.billing.processMonthlyBilling);

// Run daily at midnight UTC
crons.daily("daily-stats", { hourUTC: 0, minuteUTC: 0 }, internal.analytics.computeDailyStats);

export default crons;
```

### 16.9 File Storage

```typescript
// Generate upload URL
export const generateUploadUrl = mutation({
  args: {},
  returns: v.string(),
  handler: async (ctx) => {
    return await ctx.storage.generateUploadUrl();
  },
});

// Get file URL
export const getFileUrl = query({
  args: { storageId: v.id("_storage") },
  returns: v.union(v.string(), v.null()),
  handler: async (ctx, args) => {
    return await ctx.storage.getUrl(args.storageId);
  },
});

// Delete file
export const deleteFile = mutation({
  args: { storageId: v.id("_storage") },
  handler: async (ctx, args) => {
    await ctx.storage.delete(args.storageId);
  },
});

// Query storage directly
const files = await ctx.db.query("_storage").collect();
```

### 16.10 Internal vs Public Functions

```typescript
// convex/users/queries.ts

// Public - exposed to client via api object
export const get = query({ /* ... */ });
// Client: useQuery(api.users.queries.get, { id })

// Internal - only callable from other Convex functions
export const getInternal = internalQuery({ /* ... */ });
// Other function: ctx.runQuery(internal.users.queries.getInternal, { id })

// Mutations: mutation() vs internalMutation()
// Actions: action() vs internalAction()
```

### 16.11 Common Patterns

```typescript
// Authentication check pattern
export const protectedQuery = query({
  args: { /* ... */ },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const user = await ctx.db
      .query("users")
      .withIndex("by_auth_provider", (q) =>
        q.eq("authProvider", identity.issuer).eq("authProviderId", identity.subject)
      )
      .unique();
    
    if (!user) throw new Error("User not found");
    
    // Continue with user context...
  },
});

// Ownership check pattern
const project = await ctx.db.get(args.projectId);
if (!project) throw new Error("Project not found");
if (project.userId !== user._id) throw new Error("Not authorized");

// Unique slug generation
let slug = baseSlug;
let counter = 1;
while (await ctx.db.query("projects").withIndex("by_slug", q => q.eq("userId", userId).eq("slug", slug)).unique()) {
  slug = `${baseSlug}-${counter++}`;
}
```

---

## Appendix: Environment Variables

```bash
# Convex
CONVEX_DEPLOYMENT=dev:your-deployment
AUTH_GITHUB_ID=xxx
AUTH_GITHUB_SECRET=xxx
AUTH_GOOGLE_ID=xxx
AUTH_GOOGLE_SECRET=xxx
ANTHROPIC_API_KEY=sk-ant-xxx
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRICE_PRO=price_xxx
STRIPE_PRICE_TEAM=price_xxx
RESEND_API_KEY=re_xxx

# Encore.ts
DATABASE_URL=postgres://user:pass@localhost:5432/mohawk
JWT_SECRET=xxx
S3_ENDPOINT=https://s3.amazonaws.com
S3_ACCESS_KEY_ID=xxx
S3_SECRET_ACCESS_KEY=xxx
S3_BUCKET=mohawk-files
```

---

## Quick Reference: Convex vs Encore.ts

| Feature | Convex | Encore.ts |
|---------|--------|-----------|
| Real-time | Native (zero config) | Manual WebSocket |
| Database | Document-based | PostgreSQL |
| Queries | TypeScript functions | SQL |
| Transactions | Automatic (mutations) | Manual |
| File Storage | Built-in | S3/R2 |
| Background Jobs | Workflows/Scheduler | Pub/Sub |
| Rate Limiting | @convex-dev/ratelimiter | Custom/Redis |
| Auth | Multiple built-in providers | Custom JWT |
| Self-hosting | Docker + SQLite/Postgres | Docker + Postgres |

---

*Document Version: 1.0.0*
*Last Updated: December 2024*
*Maintainer: Mohawk Team*
