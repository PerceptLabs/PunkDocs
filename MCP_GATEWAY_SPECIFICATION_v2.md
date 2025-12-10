# MCP Gateway Mod & MCP Market Specification

**Version 2.0 | December 2025**

**Punk Framework / Mohawk Platform**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Tool Search Accuracy Problem](#2-the-tool-search-accuracy-problem)
3. [MCP Market](#3-mcp-market)
4. [MCP Gateway Mod Architecture](#4-mcp-gateway-mod-architecture)
5. [Knowledge Documentation System](#5-knowledge-documentation-system)
6. [Documentation Generation Pipeline](#6-documentation-generation-pipeline)
7. [Authentication Architecture](#7-authentication-architecture)
8. [MCP Access Tier System](#8-mcp-access-tier-system)
9. [Runtime Flow](#9-runtime-flow)
10. [Error Handling & Recovery](#10-error-handling--recovery)
11. [MCP in Published Apps](#11-mcp-in-published-apps)
12. [Connectors (Cloud Alternative)](#12-connectors-cloud-alternative)
13. [Security Model](#13-security-model)
14. [Observability](#14-observability)
15. [Knowledge Doc Lifecycle](#15-knowledge-doc-lifecycle)
16. [Custom Server Onboarding](#16-custom-server-onboarding)
17. [Implementation Roadmap](#17-implementation-roadmap)
18. [Appendix](#18-appendix)

---

## 1. Executive Summary

The MCP Gateway Mod is Punk/Mohawk's solution for structured, reliable AI-to-service integration. Rather than exposing raw MCP (Model Context Protocol) servers with 4000+ tools and ~60% retrieval accuracy, we provide a curated, knowledge-enhanced gateway that achieves 95%+ accuracy through documentation-driven tool selection.

### The Problem

Research shows AI tool search achieves only 60% accuracy with large tool registries. When an LLM has access to 4000 tools, it frequently selects the wrong one based on name/description alone. This leads to failed automations, frustrated users, and unreliable applications.

### Our Solution

The MCP Gateway Mod wraps the official MCP registry with a knowledge layer. Each curated server gets documentation that teaches Ska (our AI) exactly when and how to use each tool. Combined with slot validation and tier-based access control, this transforms unreliable tool search into dependable integration.

### Key Components

| Component | Purpose |
|-----------|---------|
| **MCP Market** | Curated catalog of vetted MCP servers with quality tiers |
| **Knowledge Docs** | Per-server documentation teaching AI when/how to use tools |
| **Gateway Mod** | Single mod that proxies calls to any enabled MCP server |
| **Tier System** | Access control based on user plan (Hobbyist → Agency) |
| **Auth Layer** | Secure credential storage and OAuth proxy |
| **Published App Integration** | Scoped MCP access for deployed applications |

---

## 2. The Tool Search Accuracy Problem

Understanding why raw MCP access fails is crucial to appreciating our solution.

### 2.1 Research Findings

Arcade.dev tested Anthropic's tool search with 4000 MCP tools:

| Search Method | Success Rate | Tasks Completed |
|---------------|--------------|-----------------|
| Regex-based | 56% | 14/25 |
| BM25-based | 64% | 16/25 |

Common failures: Gmail_SendEmail, Slack_SendMessage, Zendesk_CreateTicket—tools that should be obvious.

### 2.2 Why Tool Search Fails

| Raw MCP (The Problem) | Punk MCP Mod (Our Solution) |
|-----------------------|-----------------------------|
| 4000+ tools with just name/description | Curated servers with knowledge docs |
| LLM guesses based on string matching | Ska reads docs, knows exact tool to call |
| No context, examples, or guidance | Examples, patterns, gotchas included |
| ~60% accuracy = unreliable | 95%+ accuracy = reliable |

### 2.3 The Knowledge Doc Difference

**The fundamental insight:** Tool schemas tell you WHAT a tool does, but knowledge docs tell you WHEN to use it.

This transforms tool selection from probabilistic string matching to deterministic pattern matching.

```
WITHOUT KNOWLEDGE DOCS:
User: "Tell the team the build is ready"
LLM sees: 4000 tools, picks "team_notify" (wrong—that's for alerts)
Result: Failed automation

WITH KNOWLEDGE DOCS:
User: "Tell the team the build is ready"  
Ska reads slack.md: "Use when user wants to send a message to a Slack channel"
Ska sees: "Mentions 'tell the team', 'post to Slack', or 'message'"
Result: slack_post_message(channel="#engineering", text="Build is ready!")
```

---

## 3. MCP Market

MCP Market is Punk/Mohawk's curated catalog of MCP servers. It's not a raw registry—it's a quality-controlled marketplace where every server has been vetted, documented, and tiered.

### 3.1 Registry Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  OFFICIAL MCP REGISTRY                       │
│              registry.modelcontextprotocol.io                │
│              (Source of truth, 1000+ servers)                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ Nightly sync
┌─────────────────────────────────────────────────────────────┐
│                      PUNK MCP MARKET                         │
│  • Syncs metadata from official registry (daily, 2am UTC)    │
│  • Curates to ~50-100 quality servers                        │
│  • Adds knowledge docs for each server                       │
│  • Assigns quality tiers (Bronze/Silver/Gold)                │
│  • Validates server availability and schema stability        │
│  • Exposes via Depot for user installation                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  USER'S MOHAWK WORKSPACE                     │
│  • Sees MCP Market in Depot                                  │
│  • Enables servers within tier limits                        │
│  • Configures authentication per server                      │
│  • Ska reads knowledge docs, calls tools accurately          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Why Not Direct Registry Access?

| Concern | Raw Registry | MCP Market |
|---------|--------------|------------|
| **Quality** | Variable, some broken | All tested and working |
| **Security** | Unknown, potentially malicious | Vetted for safety |
| **Documentation** | Schemas only, no guidance | Full knowledge docs |
| **Support** | Community varies | We support what we ship |
| **Accuracy** | 60% with 4000 tools | 95%+ with 50-100 curated |

### 3.3 Server Quality Tiers

| Tier | Label | Criteria |
|------|-------|----------|
| **Bronze** | Community Maintained | Basic docs, may have rough edges, community support only. Schema stable for 30+ days. |
| **Silver** | Verified & Tested | Full knowledge docs, tested integration, responsive maintainer. 99% uptime over 90 days. |
| **Gold** | Official/Enterprise | First-party or partner maintained, SLA-backed, premium support. Dedicated support channel. |

### 3.4 Sync Configuration

```json
{
  "sync": {
    "registry_url": "https://registry.modelcontextprotocol.io/v0",
    "schedule": "0 2 * * *",
    "timeout_ms": 30000,
    "retry_attempts": 3,
    "on_failure": "keep_existing"
  },
  "validation": {
    "schema_check": true,
    "availability_check": true,
    "min_uptime_percent": 95
  }
}
```

---

## 4. MCP Gateway Mod Architecture

The MCP Gateway Mod is a single mod that provides access to all MCP Market servers. Users don't install individual MCP mods—they install the gateway and enable servers within it.

### 4.1 Mod Structure

```
mcp-gateway.sqlar/
├── manifest.json              # Mod metadata, capabilities
├── lua/
│   ├── orchestrator.lua       # Brain: routing, tier enforcement
│   ├── auth.lua               # Credential management
│   └── errors.lua             # Error handling patterns
├── js/
│   ├── registry-sync.js       # Sync from official registry (I/O)
│   ├── doc-fetcher.js         # Fetch READMEs from repos (I/O)
│   ├── doc-generator.js       # Generate knowledge docs
│   └── proxy.js               # Proxy calls to MCP servers (I/O)
├── knowledge/
│   ├── overview.md            # How to use MCP through Punk
│   ├── tiers.md               # What's available at each tier
│   ├── _template.md           # Template for generating docs
│   └── servers/
│       ├── slack.md           # Knowledge doc for Slack
│       ├── github.md          # Knowledge doc for GitHub
│       ├── notion.md          # Knowledge doc for Notion
│       └── ...                # One per curated server
├── config/
│   ├── tiers.json             # Tier definitions
│   ├── curated.json           # Curated server list
│   ├── errors.json            # Error message templates
│   └── cache/                 # Cached registry data
└── schemas/
    ├── gateway-call.json      # Slot schema for gateway calls
    └── server-config.json     # Per-server config schema
```

### 4.2 Why Lua + JS Split

The split follows Trinity runtime philosophy:

| Runtime | Role | Used For |
|---------|------|----------|
| **Lua** | Brain | Routing decisions, tier enforcement, orchestration logic |
| **JS (txiki.js)** | Hands | Network I/O, HTTP calls, file operations |

This separation ensures:
- Fast decision-making in Lua (no async complexity)
- Reliable I/O in JS (mature networking stack)
- Clear responsibility boundaries

### 4.3 Manifest

```json
{
  "name": "mcp-gateway",
  "version": "2.0.0",
  "display_name": "MCP Gateway",
  "description": "Unified gateway to MCP Market servers with knowledge-enhanced tool selection",
  "type": "integration",
  "runtime": {
    "lua": "lua/orchestrator.lua",
    "js": "js/proxy.js"
  },
  "capabilities": {
    "mcp:list-servers": {
      "description": "List available MCP servers for your tier",
      "slots": {
        "tier_filter": { "type": "string", "required": false, "enum": ["bronze", "silver", "gold"] },
        "tag_filter": { "type": "array", "required": false }
      }
    },
    "mcp:server-info": {
      "description": "Get detailed info about an MCP server including knowledge doc",
      "slots": {
        "server": { "type": "string", "required": true }
      }
    },
    "mcp:call": {
      "description": "Call a tool on an MCP server",
      "slots": {
        "server": { "type": "string", "required": true },
        "tool": { "type": "string", "required": true },
        "arguments": { "type": "object", "required": false }
      },
      "validation": "strict"
    },
    "mcp:enable-server": {
      "description": "Enable an MCP server for your workspace",
      "slots": {
        "server": { "type": "string", "required": true }
      }
    },
    "mcp:disable-server": {
      "description": "Disable an MCP server, freeing up an active slot",
      "slots": {
        "server": { "type": "string", "required": true }
      }
    },
    "mcp:configure-auth": {
      "description": "Configure authentication for an MCP server",
      "slots": {
        "server": { "type": "string", "required": true },
        "auth_type": { "type": "string", "required": true, "enum": ["api_key", "oauth", "bearer"] },
        "credentials": { "type": "object", "required": true, "sensitive": true }
      }
    }
  }
}
```

### 4.4 Input Validation

Before proxying any `mcp:call`, the gateway validates arguments against the tool's inputSchema:

```lua
-- lua/orchestrator.lua
function validate_call(server_id, tool_name, arguments)
  local schema = get_tool_schema(server_id, tool_name)
  if not schema then
    return error("TOOL_NOT_FOUND", "Tool does not exist on server")
  end
  
  -- Validate required fields
  for field, spec in pairs(schema.required or {}) do
    if arguments[field] == nil then
      return error("MISSING_REQUIRED", "Missing required field: " .. field)
    end
  end
  
  -- Validate types
  for field, value in pairs(arguments) do
    local expected_type = schema.properties[field]?.type
    if expected_type and type(value) ~= lua_type(expected_type) then
      return error("TYPE_MISMATCH", "Field " .. field .. " expected " .. expected_type)
    end
  end
  
  return true
end
```

---

## 5. Knowledge Documentation System

The knowledge layer is what transforms 60% accuracy into 95%+. Each MCP server gets comprehensive documentation that teaches Ska exactly when and how to use it.

### 5.1 Three Documentation Layers

| Layer | Source | Content | LLM Value |
|-------|--------|---------|-----------|
| **Schema** | MCP server `list_tools()` | Tool name, inputSchema | LLM guesses based on name |
| **README** | Server repository | Installation, setup | Developer-focused, not AI-focused |
| **Knowledge Doc** | Punk-generated | "When to Use", examples, patterns | **THIS is what Ska needs** |

### 5.2 Knowledge Doc Template

```markdown
---
server: {{server_id}}
tier: {{tier}}
tags: [{{tags}}]
last_validated: {{date}}
schema_version: {{version}}
---

# {{display_name}}

{{description}}

## When to Use

Use this server when the user:
{{#each use_cases}}
- {{this}}
{{/each}}

## Do NOT Use When

Avoid this server if:
{{#each anti_patterns}}
- {{this}}
{{/each}}

## Available Tools

{{#each tools}}
### {{name}}

{{description}}

**Parameters:**
{{#each inputSchema.properties}}
- `{{@key}}` ({{type}}{{#if required}}, required{{/if}}): {{description}}
{{/each}}

**Example:**
> User: "{{example_prompt}}"
> Ska: `{{name}}({{example_args}})`

**Common Errors:**
{{#each common_errors}}
- {{this}}
{{/each}}

{{/each}}

## Authentication

{{auth_instructions}}

**Required Scopes:** {{scopes}}

## Rate Limits

{{rate_limit_info}}

## Common Patterns

{{#each patterns}}
- {{this}}
{{/each}}

## Gotchas

{{#each gotchas}}
- {{this}}
{{/each}}
```

### 5.3 Example Knowledge Doc: Slack

```markdown
---
server: io.github.modelcontextprotocol/slack
tier: silver
tags: [communication, messaging, team, notifications]
last_validated: 2025-12-01
schema_version: 1.2.0
---

# Slack MCP Server

Send and receive messages in Slack workspaces.

## When to Use

Use this server when the user:
- Wants to send a message to a Slack channel
- Asks to check Slack messages or history
- Wants to list channels or users
- Mentions "Slack", "post to Slack", or "message the team"
- Wants to notify a channel about something

## Do NOT Use When

Avoid this server if:
- User wants email notifications (use email server)
- User mentions "Teams" or "Discord" (different platforms)
- User wants to schedule messages (not supported, suggest workaround)

## Available Tools

### slack_post_message

Post a message to a Slack channel.

**Parameters:**
- `channel` (string, required): Channel name with # prefix or channel ID
- `text` (string, required): Message content (supports Slack markdown)
- `thread_ts` (string, optional): Thread timestamp for replies

**Example:**
> User: "Tell the team in #engineering that the build is ready"
> Ska: `slack_post_message(channel="#engineering", text="The build is ready! :rocket:")`

**Common Errors:**
- "channel_not_found": Channel doesn't exist or bot not invited
- "not_in_channel": Bot needs to be added to private channels

### slack_list_channels

List all channels in the workspace.

**Parameters:**
- `types` (string, optional): "public_channel", "private_channel", or both
- `limit` (number, optional): Max results (default 100)

**Example:**
> User: "What Slack channels do we have?"
> Ska: `slack_list_channels()`

### slack_get_history

Get message history from a channel.

**Parameters:**
- `channel` (string, required): Channel name or ID
- `limit` (number, optional): Max messages to retrieve (default 10)

**Example:**
> User: "What were the last 5 messages in #general?"
> Ska: `slack_get_history(channel="#general", limit=5)`

## Authentication

**Type:** OAuth 2.0 or Bot Token

**Setup Steps:**
1. Create app at api.slack.com/apps
2. Add Bot Token Scopes: `chat:write`, `channels:read`, `channels:history`
3. Install to workspace
4. Copy Bot User OAuth Token (starts with `xoxb-`)

**Required Scopes:** 
- `chat:write` - Post messages
- `channels:read` - List channels
- `channels:history` - Read message history
- `users:read` - List users (optional)

## Rate Limits

- Tier 2: 20 requests per minute for most methods
- Tier 3: 50 requests per minute for posting
- Tier 4: 100 requests per minute for reads

If rate limited, wait 60 seconds before retry.

## Common Patterns

- Always use channel names with # prefix for clarity
- For DMs, use user ID instead of channel name
- Check channel exists with list_channels if user is unsure of name
- Use thread_ts to reply in threads, keeping channels clean

## Gotchas

- Bot must be invited to private channels before posting
- Channel names are case-sensitive
- Emoji shortcodes like :rocket: work in messages
- Links auto-unfurl unless wrapped in angle brackets: <https://example.com>
```

---

## 6. Documentation Generation Pipeline

Knowledge docs are generated at **BUILD TIME** (when we update the mod), not at runtime. This keeps the gateway fast and reliable.

### 6.1 Pipeline Steps

```
┌──────────────────────────────────────────────────────────────┐
│ 1. SYNC                                                      │
│    GET registry.modelcontextprotocol.io/v0/servers           │
│    → Raw server list with metadata                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. FILTER                                                    │
│    Apply curation rules from curated.json                    │
│    → ~50-100 quality servers                                 │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. FETCH SCHEMAS                                             │
│    Connect to each MCP server, call list_tools()             │
│    → Tool schemas with inputSchema                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. FETCH README                                              │
│    GET README.md from server's repository                    │
│    → Installation docs, sometimes usage hints                │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. GENERATE DRAFT                                            │
│    Expand _template.md with schema + README data             │
│    → Draft knowledge doc with placeholders                   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 6. HUMAN CURATION                                            │
│    - Write "When to Use" patterns                            │
│    - Add 3+ examples per tool                                │
│    - Document gotchas and common errors                      │
│    - Assign quality tier                                     │
│    → Production-ready knowledge doc                          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 7. VALIDATE                                                  │
│    - All required sections present                           │
│    - Examples are syntactically correct                      │
│    - Schema matches current server                           │
│    → Validated knowledge doc                                 │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 8. BUNDLE                                                    │
│    Include all knowledge docs in mod archive                 │
│    → mcp-gateway.sqlar ready for distribution                │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Auto-Generated vs Human-Written

| Component | Auto-Generated | Human-Written | Notes |
|-----------|----------------|---------------|-------|
| Tool list | ✓ From list_tools() | Curate to useful subset | Remove deprecated/internal tools |
| Parameter schemas | ✓ From inputSchema | Add constraints, enums | Improve type safety |
| Doc structure | ✓ From template | — | Consistent format |
| **"When to Use"** | Placeholder only | **✓ REQUIRED** | Most important section |
| **"Do NOT Use"** | — | **✓ REQUIRED** | Prevents wrong tool selection |
| **Examples** | — | **✓ REQUIRED (3+ each)** | Must cover common cases |
| Auth instructions | From README | ✓ Verify/expand | Often incomplete in READMEs |
| Rate limits | From API docs | ✓ Verify | Critical for reliability |
| **Gotchas** | — | **✓ REQUIRED** | Tribal knowledge |
| Common errors | From testing | ✓ Expand | Real-world failures |

### 6.3 CLI Commands

```bash
# Sync registry and generate draft docs for all curated servers
punk mcp sync --registry=official --output=knowledge/servers/

# Generate doc for specific server
punk mcp generate-doc io.github.modelcontextprotocol/slack \
  --template=knowledge/_template.md \
  --output=knowledge/servers/slack.md

# Validate all knowledge docs
punk mcp validate-docs knowledge/servers/
# Checks: required sections, example syntax, schema match

# Check for schema drift (server updated, doc stale)
punk mcp check-drift knowledge/servers/
# Flags docs where server schema changed since last_validated

# Update docs only for servers that changed
punk mcp sync --registry=official --update-only

# Test knowledge doc accuracy (requires test cases)
punk mcp test-accuracy knowledge/servers/slack.md \
  --test-cases=tests/slack-cases.json
```

---

## 7. Authentication Architecture

Secure credential management is critical for MCP integration.

### 7.1 Credential Storage

```
┌─────────────────────────────────────────────────────────────┐
│                    CREDENTIAL FLOW                           │
└─────────────────────────────────────────────────────────────┘

User provides credentials
         │
         ▼
┌─────────────────────┐
│  Input Validation   │  Verify format, required fields
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Encryption         │  AES-256-GCM with workspace key
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  GlyphCase Storage  │  Encrypted blob in mcp_credentials table
└─────────────────────┘
         │
         ▼ At call time
┌─────────────────────┐
│  Decrypt in Memory  │  Never written to disk decrypted
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Inject into Call   │  Added to MCP request headers
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Clear from Memory  │  Zeroed after use
└─────────────────────┘
```

### 7.2 Credential Schema

```json
{
  "mcp_credentials": {
    "workspace_id": "TEXT NOT NULL",
    "server_id": "TEXT NOT NULL",
    "auth_type": "TEXT NOT NULL",
    "encrypted_data": "BLOB NOT NULL",
    "created_at": "INTEGER NOT NULL",
    "updated_at": "INTEGER NOT NULL",
    "expires_at": "INTEGER",
    "PRIMARY KEY": "(workspace_id, server_id)"
  }
}
```

### 7.3 Supported Auth Types

| Type | Storage | Usage |
|------|---------|-------|
| **api_key** | Encrypted string | Added to header or query param |
| **bearer** | Encrypted token | `Authorization: Bearer <token>` |
| **oauth** | Encrypted refresh token | Auto-refresh access tokens |
| **basic** | Encrypted user:pass | `Authorization: Basic <base64>` |

### 7.4 OAuth Flow

```
┌──────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────┐
│ User │     │ MCP Gateway │     │ OAuth Server │     │ Service │
└──┬───┘     └──────┬──────┘     └──────┬───────┘     └────┬────┘
   │                │                   │                  │
   │ Enable server  │                   │                  │
   │───────────────>│                   │                  │
   │                │                   │                  │
   │                │ Auth URL          │                  │
   │<───────────────│                   │                  │
   │                │                   │                  │
   │ Authorize ─────────────────────────>                  │
   │                │                   │                  │
   │                │    Auth code      │                  │
   │<───────────────────────────────────│                  │
   │                │                   │                  │
   │ Callback       │                   │                  │
   │───────────────>│                   │                  │
   │                │                   │                  │
   │                │ Exchange code     │                  │
   │                │──────────────────>│                  │
   │                │                   │                  │
   │                │ Access + Refresh  │                  │
   │                │<──────────────────│                  │
   │                │                   │                  │
   │                │ Store encrypted   │                  │
   │                │ refresh token     │                  │
   │                │                   │                  │
   │ Server enabled │                   │                  │
   │<───────────────│                   │                  │
```

### 7.5 Token Refresh

```lua
-- lua/auth.lua
function get_access_token(workspace_id, server_id)
  local creds = load_credentials(workspace_id, server_id)
  
  if creds.auth_type ~= 'oauth' then
    return creds.token
  end
  
  -- Check if access token still valid
  if creds.access_token_expires > now() + 300 then  -- 5 min buffer
    return creds.access_token
  end
  
  -- Refresh needed
  local new_tokens = js_call('refresh_oauth_token', {
    server_id = server_id,
    refresh_token = decrypt(creds.refresh_token)
  })
  
  -- Store new tokens
  update_credentials(workspace_id, server_id, {
    access_token = encrypt(new_tokens.access_token),
    access_token_expires = now() + new_tokens.expires_in,
    refresh_token = encrypt(new_tokens.refresh_token or creds.refresh_token)
  })
  
  return new_tokens.access_token
end
```

### 7.6 Security Principles

1. **Never log credentials** - Not in errors, not in debug output
2. **Never send to Ska** - Credentials injected at proxy layer, invisible to AI
3. **Workspace isolation** - Credentials scoped to workspace, not user
4. **Encryption at rest** - AES-256-GCM, key derived from workspace secret
5. **Memory clearing** - Decrypted credentials zeroed after use
6. **Expiration support** - Auto-expire credentials, require re-auth

---

## 8. MCP Access Tier System

MCP access is gated by tier. The framing is "active slots" not "limits"—users have access to the full catalog but can only enable a certain number at once.

### 8.1 Tier Definitions

| Tier | Active Slots | Catalog Access | Notes |
|------|--------------|----------------|-------|
| **Free** | 0 | View only | Can browse MCP Market, can't enable |
| **Hobbyist** | 5 | Bronze only | Basics: filesystem, fetch, simple APIs |
| **Pro** | 15 | Bronze + Silver | Most curated servers available |
| **Team** | 30 | All tiers | Shared configs across team |
| **Agency** | Unlimited | All + Custom | Can add custom MCP servers |

### 8.2 Why "Active Slots" Framing

| Framing | Psychology |
|---------|------------|
| "You have access to 50+ servers, 15 can be active" | Abundant, flexible |
| "You're limited to 15 servers" | Stingy, restrictive |

Same mechanics, different perception. Additionally:
- Users can swap servers in/out anytime—flexibility, not restriction
- Fewer active servers = better AI accuracy (this is a feature!)
- Encourages thoughtful selection over hoarding

### 8.3 Tier Enforcement

```lua
-- lua/orchestrator.lua

local TIER_CONFIG = {
  free     = { max_slots = 0,  catalog = {} },
  hobbyist = { max_slots = 5,  catalog = {'bronze'} },
  pro      = { max_slots = 15, catalog = {'bronze', 'silver'} },
  team     = { max_slots = 30, catalog = {'bronze', 'silver', 'gold'} },
  agency   = { max_slots = -1, catalog = {'bronze', 'silver', 'gold', 'custom'} }
}

function enforce_tier(workspace_id, server_id, action)
  local user_tier = get_workspace_tier(workspace_id)
  local config = TIER_CONFIG[user_tier]
  local server_tier = get_server_tier(server_id)
  
  -- Check catalog access
  if not table_contains(config.catalog, server_tier) then
    return error('TIER_REQUIRED', {
      message = 'This server requires ' .. server_tier .. ' tier access',
      upgrade_url = '/settings/billing'
    })
  end
  
  -- Check active slots (for enable action)
  if action == 'enable' then
    local active = get_active_servers(workspace_id)
    
    if table_contains(active, server_id) then
      return true  -- Already enabled
    end
    
    if config.max_slots >= 0 and #active >= config.max_slots then
      return error('SLOT_LIMIT', {
        message = 'Active slot limit reached. Disable a server first.',
        active_count = #active,
        max_slots = config.max_slots,
        active_servers = active
      })
    end
  end
  
  return true
end
```

### 8.4 Slot Management UI

```
┌─────────────────────────────────────────────────────────────┐
│ MCP Servers                              Active: 12/15 slots │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ● Slack            [Silver]    ✓ Enabled    [Disable]      │
│ ● GitHub           [Silver]    ✓ Enabled    [Disable]      │
│ ● Notion           [Silver]    ✓ Enabled    [Disable]      │
│ ○ Airtable         [Silver]    ○ Available  [Enable]       │
│ ○ Linear           [Gold]      🔒 Upgrade   [View]         │
│                                                             │
│ [Browse MCP Market →]                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Runtime Flow

How MCP calls flow through the system at runtime.

### 9.1 Call Sequence

1. User asks Ska something involving external service
2. Ska reads knowledge docs in `mcp-gateway/knowledge/servers/`
3. Ska identifies correct server + tool based on "When to Use" patterns
4. Ska calls `mcp:call` with server, tool, arguments
5. Gateway validates arguments against tool schema
6. Gateway's Lua orchestrator checks tier permissions
7. Gateway retrieves and decrypts credentials
8. Gateway's JS proxy connects to actual MCP server
9. Real MCP server executes tool, returns result
10. Gateway validates response, handles errors
11. Gateway returns result to Ska
12. Ska presents result to user

### 9.2 Sequence Diagram

```
┌──────┐     ┌─────┐     ┌─────────────┐     ┌────────────┐
│ User │     │ Ska │     │ MCP Gateway │     │ MCP Server │
└──┬───┘     └──┬──┘     └──────┬──────┘     └─────┬──────┘
   │            │               │                  │
   │ "Post to   │               │                  │
   │ #general"  │               │                  │
   │───────────>│               │                  │
   │            │               │                  │
   │            │ read          │                  │
   │            │ knowledge/    │                  │
   │            │ slack.md      │                  │
   │            │──────────────>│                  │
   │            │               │                  │
   │            │ mcp:call      │                  │
   │            │ server=slack  │                  │
   │            │ tool=post_msg │                  │
   │            │──────────────>│                  │
   │            │               │                  │
   │            │               │ validate args    │
   │            │               │ ──────────>      │
   │            │               │                  │
   │            │               │ check tier       │
   │            │               │ ──────────>      │
   │            │               │                  │
   │            │               │ get credentials  │
   │            │               │ ──────────>      │
   │            │               │                  │
   │            │               │ proxy call       │
   │            │               │─────────────────>│
   │            │               │                  │
   │            │               │      result      │
   │            │               │<─────────────────│
   │            │               │                  │
   │            │    result     │                  │
   │            │<──────────────│                  │
   │            │               │                  │
   │ "Posted!"  │               │                  │
   │<───────────│               │                  │
```

---

## 10. Error Handling & Recovery

Robust error handling is essential for reliable MCP integration.

### 10.1 Error Categories

| Category | Examples | Handling |
|----------|----------|----------|
| **Validation** | Missing field, wrong type | Return immediately with clear message |
| **Permission** | Tier insufficient, server disabled | Clear upgrade path or enable instruction |
| **Auth** | Token expired, invalid credentials | Prompt re-auth, auto-refresh if possible |
| **Rate Limit** | Too many requests | Backoff, retry with delay, inform user |
| **Server Error** | MCP server down, timeout | Retry with backoff, fallback if available |
| **Response Error** | Unexpected schema, malformed response | Log for debugging, return sanitized error |

### 10.2 Error Response Schema

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Slack API rate limit exceeded",
    "category": "rate_limit",
    "server": "slack",
    "tool": "post_message",
    "retry_after": 60,
    "details": {
      "limit": "20 requests per minute",
      "reset_at": "2025-12-05T14:30:00Z"
    },
    "suggestions": [
      "Wait 60 seconds before retrying",
      "Consider batching multiple messages"
    ]
  }
}
```

### 10.3 Retry Logic

```lua
-- lua/errors.lua

local RETRY_CONFIG = {
  max_attempts = 3,
  base_delay_ms = 1000,
  max_delay_ms = 30000,
  retryable_codes = {
    'TIMEOUT', 'SERVER_UNAVAILABLE', 'RATE_LIMITED', 'CONNECTION_RESET'
  }
}

function with_retry(fn, context)
  local attempts = 0
  local last_error = nil
  
  while attempts < RETRY_CONFIG.max_attempts do
    attempts = attempts + 1
    
    local success, result = pcall(fn)
    
    if success then
      return result
    end
    
    last_error = result
    
    -- Check if retryable
    if not table_contains(RETRY_CONFIG.retryable_codes, result.code) then
      break
    end
    
    -- Calculate delay (exponential backoff with jitter)
    local delay = math.min(
      RETRY_CONFIG.base_delay_ms * (2 ^ (attempts - 1)),
      RETRY_CONFIG.max_delay_ms
    )
    delay = delay * (0.5 + math.random() * 0.5)  -- Add jitter
    
    -- Use retry_after if provided
    if result.retry_after then
      delay = result.retry_after * 1000
    end
    
    log_retry(context, attempts, delay, result)
    sleep(delay)
  end
  
  return error('MAX_RETRIES_EXCEEDED', {
    original_error = last_error,
    attempts = attempts
  })
end
```

### 10.4 Fallback Strategies

```lua
-- lua/orchestrator.lua

local FALLBACKS = {
  ['slack:post_message'] = {
    strategy = 'queue',
    action = function(args)
      -- Queue message for later delivery
      queue_for_retry('slack', 'post_message', args, {
        max_age = 3600,  -- 1 hour
        notify_on_failure = true
      })
      return { queued = true, message = 'Message queued for delivery' }
    end
  },
  ['github:create_issue'] = {
    strategy = 'draft',
    action = function(args)
      -- Save as local draft
      save_draft('github_issue', args)
      return { drafted = true, message = 'Saved as draft, will create when GitHub is available' }
    end
  }
}

function handle_server_failure(server, tool, args, error)
  local key = server .. ':' .. tool
  local fallback = FALLBACKS[key]
  
  if fallback then
    log_fallback(server, tool, fallback.strategy)
    return fallback.action(args)
  end
  
  -- No fallback, propagate error
  return error
end
```

### 10.5 User-Facing Error Messages

| Internal Code | User Message |
|---------------|--------------|
| `AUTH_EXPIRED` | "Your Slack connection expired. Please reconnect in Settings → Integrations." |
| `RATE_LIMITED` | "Slack is temporarily limiting requests. Your message will be sent in about 1 minute." |
| `SERVER_DOWN` | "GitHub is currently unavailable. We've saved your request and will retry automatically." |
| `PERMISSION_DENIED` | "The GitHub token doesn't have permission to create issues. Please reconnect with the 'repo' scope." |
| `TIER_REQUIRED` | "Linear integration requires Pro tier. Upgrade to access 15+ MCP servers." |

---

## 11. MCP in Published Apps

When users build and publish apps with Mohawk, how does MCP access work? The answer depends on deployment target.

### 11.1 Key Principle: Scoped, Not Full

**Published apps get SCOPED MCP access—only the specific servers and tools declared at build time.** They do NOT get full MCP Market access.

This provides:
- **Security**: Apps can't access servers the builder didn't intend
- **Predictability**: App behavior is deterministic
- **Cost control**: Metered by declared capabilities, not open-ended

### 11.2 Build Time vs Runtime

| MCP for Building | MCP in App |
|------------------|------------|
| Ska uses MCP to help you build | App itself calls MCP at runtime |
| *Example:* "Ska, look at my GitHub issues and build a dashboard" | *Example:* "Build me an app that shows live GitHub issues" |
| Ska calls GitHub MCP, gets data | App has GitHub integration built-in |
| Generates app with that data | App calls GitHub at RUNTIME |
| Data fetched at BUILD TIME | Requires backend, auth, connection |
| Resulting app may be static | Data is always fresh |

### 11.3 MCP by Deployment Target

| Target | MCP Access | Details |
|--------|------------|---------|
| **Nitro Local** | ✓ Full | User's machine, user's tokens, user controls access. Full MCP for declared servers. |
| **Nitro Cloud** | ⚠ Connectors | Hosted by us. Use Connectors (simplified integrations) instead of raw MCP. Rate limited, metered. |
| **Static Export** | ✗ None | No backend. Data must be baked in at build time or fetched client-side. |
| **Share Link** | ✗ None | Security risk. Can't expose user's credentials to viewers. |

### 11.4 App Manifest for MCP

Published apps declare exactly which MCP servers and tools they need:

```json
{
  "name": "Team Dashboard",
  "version": "1.0.0",
  "capabilities": {
    "mcp": {
      "required": true,
      "servers": [
        {
          "id": "github",
          "tools": ["list_issues", "get_issue", "create_issue"],
          "auth": "user",
          "reason": "Display and manage GitHub issues"
        },
        {
          "id": "slack",
          "tools": ["post_message", "list_channels"],
          "auth": "user",
          "reason": "Send notifications to team channels"
        }
      ]
    }
  },
  "permissions": {
    "mcp:github:list_issues": "Read GitHub issues for display",
    "mcp:github:create_issue": "Create issues from dashboard",
    "mcp:slack:post_message": "Send notifications when issues change"
  }
}
```

### 11.5 What Gets Bundled

When an app declares MCP dependencies:

| Bundled | Not Bundled |
|---------|-------------|
| Knowledge docs for declared servers | Full MCP gateway mod |
| Tool schemas for declared tools only | Access to undeclared servers |
| Auth flow configuration | Builder's credentials |
| Permission descriptions | Other workspace data |

### 11.6 Published App MCP Tiers

| Tier | Local Deploy | Cloud Deploy | Connectors |
|------|--------------|--------------|------------|
| **Hobbyist** | 3 servers | — | — |
| **Pro** | Same as workspace | — | 3 per app |
| **Team** | Same as workspace | 5 servers | 10 per app |
| **Agency** | Unlimited | 15 servers | Unlimited |

### 11.7 Runtime Permission Check

```lua
-- In published app runtime
function check_app_mcp_permission(app_manifest, server, tool)
  local mcp_config = app_manifest.capabilities.mcp
  
  if not mcp_config then
    return error('MCP_NOT_DECLARED', 'This app does not use MCP')
  end
  
  local server_config = find_server(mcp_config.servers, server)
  
  if not server_config then
    return error('SERVER_NOT_DECLARED', {
      message = 'App did not declare access to ' .. server,
      declared_servers = map(mcp_config.servers, 'id')
    })
  end
  
  if not table_contains(server_config.tools, tool) then
    return error('TOOL_NOT_DECLARED', {
      message = 'App did not declare access to ' .. server .. ':' .. tool,
      declared_tools = server_config.tools
    })
  end
  
  return true
end
```

---

## 12. Connectors (Cloud Alternative)

For cloud-deployed apps, we offer Connectors—simplified, curated integrations that are easier to secure, rate limit, and support than raw MCP.

### 12.1 Why Connectors Instead of Raw MCP?

When apps run on OUR infrastructure and call MCP:

| Concern | Problem with Raw MCP | Connector Solution |
|---------|---------------------|-------------------|
| **Cost** | We pay for compute to proxy MCP calls | Predictable, metered usage |
| **Security** | User's API keys stored/used on our servers | Managed OAuth, encrypted storage |
| **Reliability** | We're responsible for MCP server uptime | We control the integration |
| **Rate Limits** | Multiple users sharing external API limits | Per-app rate limiting |
| **Support** | Debugging raw MCP issues is complex | Simplified, documented integrations |

### 12.2 Available Connectors

| Connector | Capabilities | Rate Limit |
|-----------|--------------|------------|
| **Data Refresh** | Fetch URL on schedule, update app data | 1/min per URL |
| **Webhook In** | Receive webhooks from external services | 100/min |
| **Webhook Out** | Send data to external URL on events | 50/min |
| **Google Sheets** | Read/write to user's Google Sheets | 60/min |
| **Airtable** | Read/write to user's Airtable bases | 5/sec |
| **Supabase** | Read/write to user's Supabase database | 100/min |
| **Notion** | Read/write to user's Notion workspace | 3/sec |

### 12.3 Connector vs Raw MCP

```
RAW MCP (Local Deploy):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Your App   │────>│ MCP Gateway │────>│ Slack API   │
└─────────────┘     └─────────────┘     └─────────────┘
                    (on your machine)

CONNECTOR (Cloud Deploy):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Your App   │────>│  Connector  │────>│ Slack API   │
└─────────────┘     └─────────────┘     └─────────────┘
                    (managed by us)
                    - Rate limited
                    - Metered
                    - Monitored
```

### 12.4 Connector Configuration

```json
{
  "connectors": {
    "google_sheets": {
      "enabled": true,
      "auth": "oauth",
      "scopes": ["spreadsheets.readonly"],
      "rate_limit": {
        "requests_per_minute": 60,
        "on_exceed": "queue"
      }
    },
    "webhook_out": {
      "enabled": true,
      "allowed_domains": ["api.example.com", "hooks.slack.com"],
      "rate_limit": {
        "requests_per_minute": 50,
        "on_exceed": "drop"
      }
    }
  }
}
```

### 12.5 MCP Under the Hood

Connectors may use MCP internally, or direct API calls—the user doesn't need to know. Benefits:
- Simpler mental model for users
- We control what's exposed
- Easier to secure and rate limit
- "My app connects to Sheets" is clearer than "My app uses MCP"

---

## 13. Security Model

### 13.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| Credential theft | Encrypted at rest, never logged, memory cleared |
| Malicious MCP server | Curation layer, only vetted servers |
| Cross-workspace access | Workspace isolation, scoped credentials |
| Privilege escalation | Tier enforcement, declared capabilities |
| API key exposure in apps | Keys never bundled, auth at runtime |

### 13.2 Server Vetting Process

Before a server enters MCP Market:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SOURCE REVIEW                                            │
│    - Repository public and maintained?                      │
│    - License compatible?                                    │
│    - Known/reputable maintainer?                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. CODE AUDIT                                               │
│    - No credential logging                                  │
│    - No data exfiltration                                   │
│    - No unexpected network calls                            │
│    - Dependencies reviewed                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. BEHAVIOR TEST                                            │
│    - Run in sandbox environment                             │
│    - Monitor network traffic                                │
│    - Verify tool outputs match schemas                      │
│    - Test error handling                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. ONGOING MONITORING                                       │
│    - Version updates reviewed                               │
│    - Uptime tracked                                         │
│    - User reports monitored                                 │
│    - Quarterly re-review                                    │
└─────────────────────────────────────────────────────────────┘
```

### 13.3 Credential Security

| Principle | Implementation |
|-----------|----------------|
| **Encryption at rest** | AES-256-GCM with workspace-derived key |
| **Never logged** | Credentials stripped from all log output |
| **Never sent to AI** | Injected at proxy layer, invisible to Ska |
| **Workspace isolation** | Credentials scoped to workspace, not shared |
| **Memory clearing** | Decrypted credentials zeroed after use |
| **Expiration** | Optional TTL, automatic re-auth prompts |

### 13.4 Principle of Least Privilege

For published apps:
- Apps declare exact servers and tools needed
- Runtime enforces declared scope
- No tool discovery at runtime
- User sees permission list before installing

```
┌─────────────────────────────────────────────────────────────┐
│ "Team Dashboard" wants to access:                           │
│                                                             │
│ ✓ GitHub                                                    │
│   • Read issues (list_issues, get_issue)                    │
│   • Create issues (create_issue)                            │
│                                                             │
│ ✓ Slack                                                     │
│   • Send messages (post_message)                            │
│   • List channels (list_channels)                           │
│                                                             │
│ [Allow]  [Deny]  [Review Permissions]                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 14. Observability

### 14.1 Logging

All MCP calls logged with:

```json
{
  "timestamp": "2025-12-05T14:30:00Z",
  "level": "info",
  "event": "mcp_call",
  "workspace_id": "ws_123",
  "server": "slack",
  "tool": "post_message",
  "latency_ms": 234,
  "status": "success",
  "input_size_bytes": 156,
  "output_size_bytes": 89,
  "knowledge_doc_hit": true,
  "retry_count": 0
}
```

**Never logged:**
- Credential values
- Full request/response bodies (PII risk)
- User content

### 14.2 Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `mcp.calls.total` | Total MCP calls | — |
| `mcp.calls.success_rate` | Successful calls / total | < 95% |
| `mcp.calls.latency_p50` | Median latency | > 500ms |
| `mcp.calls.latency_p99` | 99th percentile latency | > 5000ms |
| `mcp.accuracy.first_try` | Correct tool on first attempt | < 90% |
| `mcp.auth.failures` | Auth failures by server | Spike detection |
| `mcp.rate_limits.hit` | Rate limit events | > 10/min |

### 14.3 Knowledge Doc Effectiveness

Track which knowledge docs lead to successful calls:

```sql
SELECT 
  server,
  COUNT(*) as total_calls,
  SUM(CASE WHEN first_try_success THEN 1 ELSE 0 END) as first_try_successes,
  ROUND(100.0 * SUM(CASE WHEN first_try_success THEN 1 ELSE 0 END) / COUNT(*), 2) as accuracy
FROM mcp_calls
WHERE timestamp > datetime('now', '-7 days')
GROUP BY server
ORDER BY accuracy ASC;
```

If a server has < 90% first-try accuracy, its knowledge doc needs improvement.

### 14.4 Dashboards

**Operations Dashboard:**
- Real-time call volume by server
- Error rate trends
- Latency distribution
- Rate limit proximity

**Quality Dashboard:**
- First-try accuracy by server
- Knowledge doc coverage
- Schema drift detection
- User success rate

### 14.5 Alerts

| Alert | Condition | Action |
|-------|-----------|--------|
| Server Down | Error rate > 50% for 5 min | Page on-call, notify users |
| Accuracy Drop | First-try < 80% for 1 hour | Review knowledge doc |
| Auth Spike | Auth failures > 10x baseline | Check OAuth config |
| Rate Limit | > 80% of limit | Notify affected users |

---

## 15. Knowledge Doc Lifecycle

### 15.1 Creation

```
New server added to curated.json
         │
         ▼
Auto-generate draft from template
         │
         ▼
Human writes "When to Use", examples, gotchas
         │
         ▼
Validation (required sections, syntax)
         │
         ▼
Review by second team member
         │
         ▼
Merge to mcp-gateway mod
         │
         ▼
Released in next mod version
```

### 15.2 Update Triggers

| Trigger | Detection | Action |
|---------|-----------|--------|
| **Schema change** | Daily sync detects tool changes | Flag for review |
| **New tools added** | Daily sync detects new tools | Generate draft additions |
| **Accuracy drop** | Metrics show < 90% first-try | Prioritize doc improvement |
| **User feedback** | Support tickets mentioning server | Review and improve |
| **Server update** | Major version bump | Full doc review |

### 15.3 Schema Drift Detection

```bash
# Run daily after registry sync
punk mcp check-drift knowledge/servers/

# Output:
# slack.md: DRIFT DETECTED
#   - New tool: slack_update_message (not documented)
#   - Changed: slack_post_message.blocks now supports 'section' type
#   - Removed: slack_delete_message (deprecated)
#
# github.md: OK (no drift)
# notion.md: OK (no drift)
```

### 15.4 Deprecation Flow

```
Server deprecated in official registry
         │
         ▼
Mark as deprecated in curated.json (deprecated_at, reason)
         │
         ▼
Add deprecation notice to knowledge doc
         │
         ▼
Warn users when they call deprecated server
         │
         ▼
After 90 days: disable for new enables
         │
         ▼
After 180 days: disable completely, suggest alternatives
         │
         ▼
Remove from MCP Market (knowledge doc archived)
```

### 15.5 Quality Scoring

Each knowledge doc gets a quality score:

```json
{
  "server": "slack",
  "quality_score": 92,
  "breakdown": {
    "required_sections": 100,
    "examples_per_tool": 95,
    "first_try_accuracy": 94,
    "user_feedback_score": 88,
    "freshness": 85
  },
  "last_updated": "2025-12-01",
  "next_review": "2025-03-01"
}
```

Score components:
- **Required sections**: All template sections present (0-100)
- **Examples per tool**: At least 3 examples per tool (0-100)
- **First-try accuracy**: From production metrics (0-100)
- **User feedback**: From ratings/support (0-100)
- **Freshness**: Days since last update, penalized after 90 (0-100)

---

## 16. Custom Server Onboarding

Agency tier users can add custom MCP servers. This requires a structured onboarding process.

### 16.1 Requirements

To add a custom MCP server:

1. **Server accessible** - Must be reachable from user's environment
2. **Valid MCP implementation** - Responds to standard MCP methods
3. **Documentation** - User must provide or generate knowledge doc
4. **Security acknowledgment** - User accepts responsibility for custom server

### 16.2 Onboarding Flow

```
User requests custom server
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. VALIDATION                                               │
│    - Connect to server                                      │
│    - Call list_tools()                                      │
│    - Verify schema validity                                 │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. KNOWLEDGE DOC                                            │
│    Option A: User uploads custom doc                        │
│    Option B: Auto-generate from schema + README             │
│    Option C: User fills interactive template                │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. CONFIGURATION                                            │
│    - Auth method (api_key, oauth, bearer)                   │
│    - Rate limits (user-defined)                             │
│    - Retry policy                                           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. TESTING                                                  │
│    - User runs test calls                                   │
│    - Verify knowledge doc accuracy                          │
│    - Confirm auth works                                     │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. ACTIVATION                                               │
│    - Server added to workspace                              │
│    - Knowledge doc loaded                                   │
│    - Available for Ska to use                               │
└─────────────────────────────────────────────────────────────┘
```

### 16.3 Custom Server Configuration

```json
{
  "custom_servers": {
    "my-internal-api": {
      "url": "https://api.internal.company.com/mcp",
      "transport": "http",
      "auth": {
        "type": "bearer",
        "token_env": "INTERNAL_API_TOKEN"
      },
      "rate_limit": {
        "requests_per_minute": 100
      },
      "retry": {
        "max_attempts": 3,
        "base_delay_ms": 1000
      },
      "knowledge_doc": "custom/my-internal-api.md",
      "added_at": "2025-12-01T10:00:00Z",
      "added_by": "user@company.com"
    }
  }
}
```

### 16.4 Limitations

| Aspect | Curated Servers | Custom Servers |
|--------|-----------------|----------------|
| **Support** | Full support | Self-supported |
| **Knowledge doc** | Maintained by Punk | Maintained by user |
| **Security vetting** | Full audit | User responsibility |
| **Published apps** | Can be included | Local deploy only |
| **Uptime monitoring** | Included | User responsibility |

### 16.5 Sharing Custom Servers

Agency users can share custom server configs within their organization:

```bash
# Export custom server config
punk mcp export-custom my-internal-api --output=my-api-config.json

# Import in another workspace (same org)
punk mcp import-custom my-api-config.json
```

---

## 17. Implementation Roadmap

### 17.1 Phase 1: Foundation (Week 1-2)

**Goal:** Registry sync and doc generation pipeline

- [ ] Set up registry sync from official MCP registry
- [ ] Build doc generation pipeline (template → knowledge doc)
- [ ] Create initial curated list (10-15 high-value servers)
- [ ] Hand-write knowledge docs for top 5 servers
- [ ] Implement validation CLI commands

**Deliverable:** Can generate and validate knowledge docs

### 17.2 Phase 2: Gateway Mod Core (Week 3-4)

**Goal:** Working gateway with tier enforcement

- [ ] Build mcp-gateway mod structure
- [ ] Implement Lua orchestrator with tier enforcement
- [ ] Implement JS proxy for MCP server connections
- [ ] Build credential storage and encryption
- [ ] Implement input validation
- [ ] Test with curated servers

**Deliverable:** Can make MCP calls through gateway

### 17.3 Phase 3: Auth & Error Handling (Week 5-6)

**Goal:** Production-ready auth and reliability

- [ ] Implement OAuth flow for supported servers
- [ ] Build token refresh mechanism
- [ ] Implement retry logic with backoff
- [ ] Add error categorization and user-friendly messages
- [ ] Build fallback strategies for common failures

**Deliverable:** Reliable auth and graceful error handling

### 17.4 Phase 4: MCP Market UI (Week 7-8)

**Goal:** User-facing discovery and management

- [ ] Build MCP Market section in Depot
- [ ] Server discovery and search
- [ ] Enable/disable server flow
- [ ] Auth configuration UI
- [ ] Slot management UI

**Deliverable:** Users can browse and enable MCP servers

### 17.5 Phase 5: Published App Integration (Week 9-10)

**Goal:** MCP in deployed apps

- [ ] App manifest MCP declaration
- [ ] Knowledge doc bundling at publish
- [ ] Runtime permission enforcement
- [ ] Connector implementation for cloud deploy
- [ ] Tier enforcement for published apps

**Deliverable:** Published apps can use declared MCP servers

### 17.6 Phase 6: Observability & Quality (Week 11-12)

**Goal:** Monitoring and continuous improvement

- [ ] Implement logging infrastructure
- [ ] Build metrics collection
- [ ] Create dashboards
- [ ] Set up alerting
- [ ] Implement accuracy tracking
- [ ] Build schema drift detection

**Deliverable:** Full observability into MCP usage

### 17.7 Phase 7: Scale (Ongoing)

**Goal:** Expand coverage and capabilities

- [ ] Expand curated server list (50+ servers)
- [ ] Quality tier assignments based on data
- [ ] Community contribution process for knowledge docs
- [ ] Custom MCP server support for Agency tier
- [ ] Additional connectors based on demand

---

## 18. Appendix

### 18.1 Glossary

| Term | Definition |
|------|------------|
| **MCP** | Model Context Protocol - standard for AI ↔ service communication |
| **MCP Market** | Punk/Mohawk's curated catalog of MCP servers |
| **MCP Gateway** | Single mod providing access to all MCP Market servers |
| **Knowledge Doc** | Documentation teaching AI when/how to use a server |
| **Active Slot** | An enabled MCP server counting against tier limit |
| **Connector** | Simplified integration for cloud-deployed apps |
| **Ska** | Punk/Mohawk's AI assistant |
| **Trinity** | Punk's runtime: Lua (brain) + txiki.js (hands) + WAMR (muscle) |
| **Schema Drift** | When MCP server schema changes but knowledge doc is stale |
| **First-Try Accuracy** | Percentage of calls where correct tool was selected first |

### 18.2 Related Documents

- `PUNK_MOD_SPECIFICATION.md` - Complete mod system specification
- `NITRO_ARCHITECTURE.md` - Deployment targets and supervision
- `MOHAWK_PRODUCT_STRATEGY.md` - Product positioning and tiers
- `GLYPHCASE_SPECIFICATION.md` - Local-first database for credential storage
- `TRINITY_RUNTIME.md` - Lua + txiki.js + WAMR execution environment

### 18.3 External References

- Official MCP Registry: `registry.modelcontextprotocol.io`
- MCP Specification: `modelcontextprotocol.io`
- Arcade Tool Search Research: `blog.arcade.dev/anthropic-tool-search-4000-tools-test`

### 18.4 Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-12-01 | Initial specification |
| 2.0 | 2025-12-05 | Added auth architecture, error handling, security model, observability, lifecycle management, custom server onboarding |

---

*— END OF DOCUMENT —*
