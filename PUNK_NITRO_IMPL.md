# Nitro Runtime Specification

**Version:** 2.0  
**Status:** Source of Truth  
**Role:** Universal Supervision Layer

---

## 1. Overview

**Nitro** is a lightweight, portable process supervisor for the Punk/Mohawk platform. It provides consistent process management, health monitoring, logging, and graceful shutdown across all deployment targets.

Nitro is not required for simple interactions, but provides significant value for:
- Long-running workflows (Agent Mod)
- Multi-process development (preview server + backend + MCP servers)
- Self-hosted deployments
- Consistent behavior across Desktop, Docker, and CLI

**Philosophy:** One supervision model everywhere. Debug once, works everywhere.

---

## 2. Core Value Proposition

| Value | Description |
|-------|-------------|
| **Consistency** | Same supervision behavior in Electron, Docker, CLI, and self-hosted. One mental model. |
| **Self-Healing** | Processes crash. Nitro detects failures and restarts specific processes without taking down siblings. |
| **Graceful Lifecycle** | Proper startup ordering (database before app) and shutdown (finish work before terminating). |
| **Observable** | Unified logging interface. All processes stream to one place with consistent formatting. |
| **Tiny** | ~100KB binary. Negligible overhead for meaningful consistency. |

---

## 3. What Nitro Supervises

| Process | Description |
|---------|-------------|
| **Trinity Runtime** | Lua + txiki.js + WASM execution environment for mods |
| **GlyphCase** | SQLite database (started first, stopped last) |
| **MCP Servers** | Local MCP servers that run as processes |
| **Preview Server** | Vite dev server for live preview |
| **Backend Dev Server** | Encore, tRPC, or other backend in development |
| **Agent Workflows** | Long-running multi-step workflows (via Agent Mod) |
| **Sidecars** | Optional processes (Python workers, FFmpeg, etc.) |

---

## 4. Architecture

### 4.1 Supervisor (Process 0)

Written in Go. Entry point for all supervised execution.

**Responsibilities:**

| Capability | Description |
|------------|-------------|
| Process Lifecycle | Start, stop, restart managed processes |
| Health Monitoring | Ping endpoints, detect hangs, trigger restarts |
| Log Aggregation | Capture stdout/stderr, format as structured JSON |
| Graceful Shutdown | Signal processes to finish work, wait, then terminate |
| Startup Ordering | Ensure dependencies start first (DB before app) |

### 4.2 Process Tree

```
nitro (PID 1)
├── glyphcase (SQLite)
├── orchestrator (Bun/Trinity)
├── preview-server (Vite)
├── mcp-server-github (if enabled)
├── mcp-server-slack (if enabled)
└── sidecars (optional)
```

---

## 5. Configuration

### 5.1 nitro.config.json

Lives in project root. Defines what Nitro supervises.

```json
{
  "app_name": "my-mohawk-app",
  "version": "1.0",
  
  "runtime": {
    "memory_limit": "512mb",
    "health_check_interval": "5s"
  },
  
  "services": [
    {
      "name": "glyphcase",
      "command": "glyphcase serve --db ./state.db",
      "restart_policy": "always",
      "startup_order": 1,
      "shutdown_order": 99
    },
    {
      "name": "orchestrator",
      "command": "bun run orchestrator.js",
      "restart_policy": "on-failure",
      "health_check": "/health",
      "startup_order": 2,
      "depends_on": ["glyphcase"]
    },
    {
      "name": "preview",
      "command": "vite --port 5173",
      "restart_policy": "on-failure",
      "startup_order": 3
    }
  ]
}
```

### 5.2 Minimal Config (Defaults)

For simple projects, Nitro works with minimal config:

```json
{
  "app_name": "my-app",
  "services": [
    {
      "name": "app",
      "command": "bun run start"
    }
  ]
}
```

Nitro applies sensible defaults:
- `restart_policy`: "on-failure"
- `health_check_interval`: "5s"
- `shutdown_timeout`: "30s"

---

## 6. Service Definition

### 6.1 Service Schema

```typescript
interface NitroService {
  name: string                    // Unique identifier
  command: string                 // Shell command to run
  
  // Lifecycle
  restart_policy?: 'always' | 'on-failure' | 'never'
  startup_order?: number          // Lower = starts first
  shutdown_order?: number         // Higher = stops last
  depends_on?: string[]           // Wait for these services
  
  // Health
  health_check?: string           // HTTP path (e.g., "/health")
  health_timeout?: string         // "5s", "30s"
  max_failures?: number           // Restart after N failures
  
  // Resources
  instances?: number              // For scaling (default: 1)
  env?: Record<string, string>    // Environment variables
  working_dir?: string            // Override working directory
}
```

### 6.2 Restart Policies

| Policy | Behavior |
|--------|----------|
| `always` | Restart regardless of exit code |
| `on-failure` | Restart only on non-zero exit |
| `never` | Do not restart |

---

## 7. Health Monitoring

| Check | Frequency | Action on Failure |
|-------|-----------|-------------------|
| HTTP health endpoint | Every 5s (configurable) | Restart after 3 consecutive failures |
| Process alive | Every 1s | Restart immediately if dead |
| Memory usage | Every 30s | Log warning at 80%, restart at limit |
| Startup timeout | Once | Fail service if not healthy within timeout |

### Health Check Flow

```
Service starts
     │
     ▼
Wait for health_timeout (default 30s)
     │
     ├── Health endpoint returns 200 → Service ready
     │
     └── Timeout reached → Mark failed, apply restart_policy
```

---

## 8. Logging

All processes stream to Nitro's log aggregator.

### 8.1 Structured Output

```json
{
  "timestamp": "2025-12-05T14:30:00Z",
  "service": "orchestrator",
  "level": "info",
  "message": "Workflow step completed",
  "metadata": {
    "workflow_id": "wf-123",
    "step": 3,
    "duration_ms": 1240
  }
}
```

### 8.2 Log Destinations

| Environment | Destination |
|-------------|-------------|
| Local dev | Terminal (TUI with Bubble Tea) |
| Docker | stdout (container logging) |
| Production | External service (configurable) |

### 8.3 Log Rotation

Local logs rotate automatically:
- Max file size: 10MB
- Keep last 5 files
- Configurable via `nitro.config.json`

---

## 9. Graceful Shutdown

Nitro ensures clean shutdown:

```
SIGTERM received
     │
     ▼
1. Stop accepting new work
     │
     ▼
2. Signal each service (reverse shutdown_order)
     │
     ▼
3. Wait for services to finish (up to shutdown_timeout)
     │
     ▼
4. SIGKILL any remaining processes
     │
     ▼
5. Stop database last (highest shutdown_order)
```

**Critical for Agent Mod:** Workflows get time to checkpoint current step before termination.

---

## 10. Deployment Targets

### 10.1 Local Development

```bash
punk dev
```

- Starts Nitro with hot reload
- Mounts source code
- Streams logs to terminal TUI
- Restarts on config change

### 10.2 Mohawk Desktop (Electron)

Electron main process spawns Nitro:

```javascript
// main.js
const nitro = spawn('nitro', ['--config', './nitro.config.json'])
nitro.stdout.pipe(logWindow)
```

Benefits:
- Processes survive renderer crashes
- Consistent behavior with other targets
- Clean shutdown on app quit

### 10.3 Docker

Nitro as PID 1:

```dockerfile
FROM punk/nitro:latest
COPY . /app
WORKDIR /app
CMD ["nitro", "--config", "nitro.config.json"]
```

Benefits:
- Proper signal handling (Docker sends SIGTERM to PID 1)
- Multi-process without docker-compose
- Single container, multiple services

### 10.4 Self-Hosted

```bash
# Install
curl -fsSL https://punk.dev/install-nitro.sh | sh

# Run
nitro --config /path/to/nitro.config.json
```

Or with systemd:

```ini
[Unit]
Description=Mohawk App
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/nitro --config /app/nitro.config.json
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 11. When Nitro is NOT Needed

| Scenario | Why No Nitro |
|----------|--------------|
| Simple Ska chat | Stateless request/response |
| Static site preview | Just files, no processes |
| Mohawk SaaS backend | Cloud platforms (K8s/ECS) handle supervision |
| Single-process apps | `bun run start` is sufficient |

Nitro is optional but provides value when you have:
- Multiple processes to coordinate
- Long-running workflows that must survive crashes
- Need for consistent behavior across environments

---

## 12. Agent Mod Integration

Agent Mod declares Nitro as a requirement:

```json
{
  "name": "agent-mod",
  "runtime": {
    "requires_nitro": true
  }
}
```

**Why Agent Mod needs Nitro:**

| Need | Nitro Provides |
|------|----------------|
| Workflows run 5-30 minutes | Crash recovery, resume from checkpoint |
| Must survive browser close | Nitro keeps running in Electron main process |
| Every step must be logged | Unified LOG service |
| Determinism requires audit | Graceful shutdown ensures complete records |

When Agent Mod is enabled, Mohawk ensures Nitro is running.

---

## 13. CLI Commands

| Command | Description |
|---------|-------------|
| `punk dev` | Start Nitro with development config |
| `punk ps` | List running services and status |
| `punk logs` | Stream logs (all services or specific) |
| `punk logs orchestrator` | Stream logs for one service |
| `punk restart <service>` | Restart specific service |
| `punk stop` | Graceful shutdown |

---

## 14. Configuration Schema

```typescript
interface NitroConfig {
  app_name: string
  version?: string
  
  runtime?: {
    memory_limit?: string           // "512mb", "2gb"
    health_check_interval?: string  // "5s", "10s"
    shutdown_timeout?: string       // "30s", "5m"
  }
  
  services: NitroService[]
  
  logging?: {
    level?: 'debug' | 'info' | 'warn' | 'error'
    format?: 'json' | 'pretty'
    max_file_size?: string          // "10mb"
    max_files?: number              // 5
  }
}
```

---

## 15. Security

| Concern | Mitigation |
|---------|------------|
| Process isolation | Each service runs as separate process |
| File access | Services constrained to project directory |
| Network | Internal services not exposed by default |
| Secrets | Environment variables, never in config file |
| Privilege | Nitro runs as unprivileged user |

---

## 16. Error Codes

| Code | Meaning |
|------|---------|
| `SERVICE_NOT_FOUND` | Service name not in config |
| `STARTUP_TIMEOUT` | Service didn't become healthy in time |
| `HEALTH_CHECK_FAILED` | Max consecutive failures reached |
| `DEPENDENCY_FAILED` | Required service (depends_on) not running |
| `SHUTDOWN_TIMEOUT` | Service didn't stop gracefully |
| `CONFIG_INVALID` | nitro.config.json validation failed |

---

## 17. Comparison to Alternatives

### Why not Docker Compose?

| Issue | Explanation |
|-------|-------------|
| Heavyweight | Requires Docker daemon |
| Multi-container model | Agent Mod is monolithic with sidecars |
| No application awareness | Doesn't understand "finish workflow step" |

### Why not PM2?

| Issue | Explanation |
|-------|-------------|
| Node-centric | Punk uses Bun, Lua, WASM |
| Config format | JS/JSON ecosystem assumptions |
| Heavier | More features than needed |

### Why not systemd?

| Issue | Explanation |
|-------|-------------|
| Linux only | No macOS/Windows |
| System-level | Requires root, complex setup |
| Not portable | Can't bundle with app |

### Why not just Bun?

| Issue | Explanation |
|-------|-------------|
| Single process | Can't supervise multiple services |
| No health checks | Process dies, stays dead |
| No logging aggregation | Each process logs separately |

---

*End of Specification*
