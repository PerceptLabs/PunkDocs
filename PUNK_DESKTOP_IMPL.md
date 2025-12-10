# Punk Mohawk Desktop Implementation

**Version:** 2.0  
**Status:** Canonical Source of Truth

---

## Overview

Mohawk Desktop (Studio) is a premium Electron + Bun application providing full rig/mod library access, offline editing, and native performance.

---

## 1. Architecture

### 1.1 Process Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    RENDERER PROCESS (Chromium)                  │
│                                                                 │
│  React + Puck + Rigs + TokiForge + Pink + Base UI           │
│                                                                 │
└─────────────────────────────────────┬───────────────────────────┘
                                      │ Electron IPC
┌─────────────────────────────────────┴───────────────────────────┐
│                    MAIN PROCESS (Bun)                           │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  GlyphCase  │  │   Claude    │  │  Licensing  │             │
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

### 1.2 Directory Structure

```
mohawk-desktop/
├── src/
│   ├── main/                 # Electron main process
│   │   ├── index.ts          # Entry point
│   │   ├── ipc.ts            # IPC handlers
│   │   ├── glyphcase/        # Local database
│   │   ├── trinity/          # Mod runtime
│   │   ├── license/          # License management
│   │   └── updater.ts        # Auto-update
│   │
│   ├── preload/              # Context bridge
│   │   └── index.ts
│   │
│   └── renderer/             # React app
│       ├── index.tsx
│       └── platform.ts       # Platform API impl
│
├── build/                    # Build resources
│   ├── icon.icns
│   ├── icon.ico
│   └── entitlements.mac.plist
│
├── electron-builder.config.js
└── package.json
```

---

## 2. IPC Layer

### 2.1 Preload Script

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('mohawk', {
  // Database
  db: {
    query: (sql: string, params?: any[]) => 
      ipcRenderer.invoke('db:query', sql, params),
    execute: (sql: string, params?: any[]) => 
      ipcRenderer.invoke('db:execute', sql, params),
  },

  // AI
  ai: {
    generate: (prompt: string, context: Schema[]) =>
      ipcRenderer.invoke('ai:generate', prompt, context),
    streamGenerate: (prompt: string, context: Schema[], callback: (chunk: string) => void) => {
      const channel = `ai:stream:${Date.now()}`
      ipcRenderer.on(channel, (_, chunk) => callback(chunk))
      return ipcRenderer.invoke('ai:streamGenerate', prompt, context, channel)
    }
  },

  // Projects
  projects: {
    list: () => ipcRenderer.invoke('projects:list'),
    get: (id: string) => ipcRenderer.invoke('projects:get', id),
    create: (name: string) => ipcRenderer.invoke('projects:create', name),
    save: (id: string, schema: Schema) => 
      ipcRenderer.invoke('projects:save', id, schema),
    delete: (id: string) => ipcRenderer.invoke('projects:delete', id),
  },

  // Revisions
  revisions: {
    list: (projectId: string) => 
      ipcRenderer.invoke('revisions:list', projectId),
    restore: (revisionId: string) => 
      ipcRenderer.invoke('revisions:restore', revisionId),
  },

  // Mods
  mods: {
    list: () => ipcRenderer.invoke('mods:list'),
    install: (path: string) => ipcRenderer.invoke('mods:install', path),
    uninstall: (id: string) => ipcRenderer.invoke('mods:uninstall', id),
    execute: (modId: string, action: string, params: any) =>
      ipcRenderer.invoke('mods:execute', modId, action, params),
  },

  // License
  license: {
    getStatus: () => ipcRenderer.invoke('license:status'),
    activate: (key: string) => ipcRenderer.invoke('license:activate', key),
    deactivate: () => ipcRenderer.invoke('license:deactivate'),
  },

  // Updates
  updates: {
    check: () => ipcRenderer.invoke('update:check'),
    download: () => ipcRenderer.invoke('update:download'),
    install: () => ipcRenderer.invoke('update:install'),
    onAvailable: (callback: (info: UpdateInfo) => void) =>
      ipcRenderer.on('update:available', (_, info) => callback(info)),
    onProgress: (callback: (progress: number) => void) =>
      ipcRenderer.on('update:progress', (_, p) => callback(p)),
    onReady: (callback: () => void) =>
      ipcRenderer.on('update:ready', callback),
  },

  // Platform info
  platform: 'desktop',
  version: () => ipcRenderer.invoke('app:version'),
})
```

### 2.2 IPC Handlers (Main Process)

```typescript
// src/main/ipc.ts
import { ipcMain } from 'electron'
import { createGlyphCase } from './glyphcase'
import { createTrinity } from './trinity'
import { createLicenseManager } from './license'

export function setupIPC() {
  const db = createGlyphCase()
  const trinity = createTrinity(db)
  const license = createLicenseManager()

  // Database
  ipcMain.handle('db:query', async (_, sql, params) => {
    return db.raw.query(sql).all(params)
  })

  ipcMain.handle('db:execute', async (_, sql, params) => {
    return db.raw.run(sql, params)
  })

  // Projects
  ipcMain.handle('projects:list', async () => {
    return db.projects.list()
  })

  ipcMain.handle('projects:get', async (_, id) => {
    return db.projects.get(id)
  })

  ipcMain.handle('projects:create', async (_, name) => {
    return db.projects.create(name)
  })

  ipcMain.handle('projects:save', async (_, id, schema) => {
    // Save to project
    db.projects.update(id, schema)
    // Create revision
    db.revisions.create(id, schema)
  })

  // Mods
  ipcMain.handle('mods:execute', async (_, modId, action, params) => {
    const status = await license.getStatus()
    const mod = db.mods.get(modId)
    
    // Check tier access
    if (mod.tier === 'pro' && status.tier === 'free') {
      throw new Error('Pro tier required for this mod')
    }
    
    return trinity.execute(mod, action, params)
  })

  // License
  ipcMain.handle('license:status', async () => {
    return license.getStatus()
  })

  ipcMain.handle('license:activate', async (_, key) => {
    return license.activate(key)
  })
}
```

---

## 3. GlyphCase (Local Database)

### 3.1 Schema

```sql
-- Projects
CREATE TABLE IF NOT EXISTS projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  schema TEXT NOT NULL,
  settings TEXT DEFAULT '{}',
  created_at TEXT GENERATED ALWAYS AS (
    datetime(substr(id, 1, 10), 'unixepoch')
  ) VIRTUAL
);

-- Revisions (for version history)
CREATE TABLE IF NOT EXISTS revisions (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  schema TEXT NOT NULL,
  prompt TEXT,
  FOREIGN KEY (project_id) REFERENCES projects(id)
);

-- Installed mods
CREATE TABLE IF NOT EXISTS mods (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  manifest TEXT NOT NULL,
  enabled INTEGER DEFAULT 1,
  installed_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Mod data (namespaced key-value store)
CREATE TABLE IF NOT EXISTS mod_data (
  mod_id TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT,
  PRIMARY KEY (mod_id, key),
  FOREIGN KEY (mod_id) REFERENCES mods(id)
);

-- User preferences
CREATE TABLE IF NOT EXISTS preferences (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- License cache
CREATE TABLE IF NOT EXISTS license_cache (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  expires_at TEXT
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_revisions_project 
  ON revisions(project_id);
CREATE INDEX IF NOT EXISTS idx_mod_data_mod 
  ON mod_data(mod_id);
```

### 3.2 API

```typescript
// src/main/glyphcase/index.ts
import { Database } from 'bun:sqlite'
import { createId } from '@punk/id'

export function createGlyphCase(dbPath?: string) {
  const path = dbPath ?? join(app.getPath('userData'), 'mohawk.db')
  const db = new Database(path)

  // Initialize schema...

  return {
    projects: {
      list(): Project[] {
        return db.query('SELECT * FROM projects ORDER BY id DESC').all()
      },

      get(id: string): Project | null {
        return db.query('SELECT * FROM projects WHERE id = ?').get(id)
      },

      create(name: string, schema: Schema = { type: 'container', children: [] }): Project {
        const id = createId()
        db.query('INSERT INTO projects (id, name, schema) VALUES (?, ?, ?)')
          .run(id, name, JSON.stringify(schema))
        return this.get(id)!
      },

      update(id: string, schema: Schema): void {
        db.query('UPDATE projects SET schema = ? WHERE id = ?')
          .run(JSON.stringify(schema), id)
      },

      delete(id: string): void {
        db.query('DELETE FROM revisions WHERE project_id = ?').run(id)
        db.query('DELETE FROM projects WHERE id = ?').run(id)
      }
    },

    revisions: {
      list(projectId: string): Revision[] {
        return db.query(
          'SELECT * FROM revisions WHERE project_id = ? ORDER BY id DESC'
        ).all(projectId)
      },

      create(projectId: string, schema: Schema, prompt?: string): Revision {
        const id = createId()
        db.query(
          'INSERT INTO revisions (id, project_id, schema, prompt) VALUES (?, ?, ?, ?)'
        ).run(id, projectId, JSON.stringify(schema), prompt ?? null)
        return { id, project_id: projectId, schema, prompt }
      },

      restore(revisionId: string): void {
        const revision = db.query('SELECT * FROM revisions WHERE id = ?').get(revisionId)
        if (revision) {
          db.query('UPDATE projects SET schema = ? WHERE id = ?')
            .run(revision.schema, revision.project_id)
        }
      }
    },

    modData: {
      get(modId: string, key: string): any {
        const row = db.query(
          'SELECT value FROM mod_data WHERE mod_id = ? AND key = ?'
        ).get(modId, key)
        return row ? JSON.parse(row.value) : null
      },

      set(modId: string, key: string, value: any): void {
        db.query(
          'INSERT OR REPLACE INTO mod_data (mod_id, key, value) VALUES (?, ?, ?)'
        ).run(modId, key, JSON.stringify(value))
      }
    },

    raw: db
  }
}
```

---

## 4. Trinity Runtime (Desktop)

### 4.1 Implementation

```typescript
// src/main/trinity/index.ts
import { Database } from 'bun:sqlite'
import { createId } from '@punk/id'
import { LuaFactory } from 'wasmoon'

export async function createTrinity(glyphcase: GlyphCase) {
  const factory = new LuaFactory()
  const lua = await factory.createEngine()

  return {
    async executeLua(code: string, context = {}) {
      for (const [key, value] of Object.entries(context)) {
        lua.global.set(key, value)
      }
      await lua.doString(code)
      return lua.global.get('result')
    },

    async executeJS(code: string, context = {}) {
      const fn = new Function(
        ...Object.keys(context),
        `"use strict"; return (async () => { ${code} })()`
      )
      return fn(...Object.values(context))
    },

    async executeWasm(module: WebAssembly.Module, fn: string, args: any[]) {
      const instance = await WebAssembly.instantiate(module)
      const exportedFn = instance.exports[fn] as Function
      return exportedFn(...args)
    },

    async loadMod(sqlarPath: string) {
      const modDb = new Database(sqlarPath)
      
      const manifest = modDb
        .query('SELECT value FROM _meta WHERE key = ?')
        .get('manifest') as { value: string }
      
      const mod = JSON.parse(manifest.value) as ModManifest
      
      const scripts = modDb
        .query('SELECT name, code, runtime FROM scripts')
        .all() as Script[]
      
      return {
        id: createId(),
        manifest: mod,
        scripts,
        db: modDb
      }
    },

    async execute(mod: LoadedMod, action: string, params: any) {
      const script = mod.scripts.find(s => s.name === action)
      if (!script) throw new Error(`Action not found: ${action}`)

      const context = this.createSandboxContext(mod)

      switch (script.runtime) {
        case 'lua':
          return this.executeLua(script.code, { ...context, params })
        case 'js':
          return this.executeJS(script.code, { ...context, params })
        case 'wasm':
          const wasmModule = await WebAssembly.compile(script.code)
          return this.executeWasm(wasmModule, 'main', [params])
      }
    },

    createSandboxContext(mod: LoadedMod): PunkModAPI {
      const allowedDomains = mod.manifest.network?.allowlist ?? []
      
      return {
        db: {
          query: (sql, params) => {
            // Only access mod's own namespace
            return glyphcase.raw.query(sql).all(params)
          },
          execute: (sql, params) => {
            glyphcase.raw.run(sql, params)
          },
          transaction: (fn) => {
            glyphcase.raw.transaction(fn)()
          }
        },

        fs: {
          read: (path) => Bun.file(this.sandboxPath(mod, path)).bytes(),
          write: (path, data) => Bun.write(this.sandboxPath(mod, path), data),
          exists: (path) => existsSync(this.sandboxPath(mod, path)),
          list: (path) => readdirSync(this.sandboxPath(mod, path))
        },

        http: {
          get: async (url, options) => {
            this.validateUrl(url, allowedDomains)
            return fetch(url, { ...options, method: 'GET' })
          },
          post: async (url, body, options) => {
            this.validateUrl(url, allowedDomains)
            return fetch(url, { ...options, method: 'POST', body: JSON.stringify(body) })
          }
        },

        ui: {
          showNotification: (message, type) => {
            // Send to renderer via IPC
          },
          requestInput: async (prompt) => {
            // Show dialog, return result
          },
          updateProgress: (percent, message) => {
            // Update progress bar
          }
        },

        ulid: createId,
        log: (...args) => console.log(`[${mod.manifest.id}]`, ...args)
      }
    }
  }
}
```

---

## 5. License Management

### 5.1 License Types

```typescript
interface License {
  key: string
  tier: 'free' | 'hobbyist' | 'pro' | 'max' | 'enterprise'
  validUntil: string
  machineId: string
  features: string[]
}

interface LicenseStatus {
  valid: boolean
  tier: string
  license?: License
  warning?: 'refresh_soon' | 'offline_grace'
  reason?: 'no_license' | 'expired' | 'invalid'
}
```

### 5.2 Implementation

```typescript
// src/main/license/index.ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto'
import { app } from 'electron'
import { machineIdSync } from 'node-machine-id'

export function createLicenseManager() {
  const machineId = machineIdSync()
  const cachePath = join(app.getPath('userData'), '.license')
  const apiUrl = 'https://api.punk.dev/v1/licenses'

  async function validateOnline(key: string): Promise<License | null> {
    try {
      const response = await fetch(`${apiUrl}/validate`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ key, machineId })
      })

      if (!response.ok) return null

      const license = await response.json() as License
      await cacheLocally(license)
      return license
    } catch {
      return null
    }
  }

  async function cacheLocally(license: License): Promise<void> {
    const key = machineId.slice(0, 32)
    const iv = randomBytes(16)
    const cipher = createCipheriv('aes-256-cbc', key, iv)
    
    const encrypted = Buffer.concat([
      iv,
      cipher.update(JSON.stringify(license)),
      cipher.final()
    ])

    await Bun.write(cachePath, encrypted)
  }

  async function readCache(): Promise<License | null> {
    try {
      const encrypted = await Bun.file(cachePath).arrayBuffer()
      const buffer = Buffer.from(encrypted)
      
      const iv = buffer.subarray(0, 16)
      const data = buffer.subarray(16)
      
      const key = machineId.slice(0, 32)
      const decipher = createDecipheriv('aes-256-cbc', key, iv)
      
      const decrypted = Buffer.concat([
        decipher.update(data),
        decipher.final()
      ])

      return JSON.parse(decrypted.toString())
    } catch {
      return null
    }
  }

  return {
    async getStatus(): Promise<LicenseStatus> {
      const cached = await readCache()
      
      if (!cached) {
        return { valid: false, tier: 'trial', reason: 'no_license' }
      }

      const validUntil = new Date(cached.validUntil)
      const now = new Date()
      const daysExpired = (now.getTime() - validUntil.getTime()) / (1000 * 60 * 60 * 24)

      if (daysExpired < 0) {
        return { valid: true, tier: cached.tier, license: cached }
      } else if (daysExpired < 7) {
        return { valid: true, tier: cached.tier, license: cached, warning: 'refresh_soon' }
      } else if (daysExpired < 30) {
        return { valid: true, tier: cached.tier, license: cached, warning: 'offline_grace' }
      } else {
        return { valid: false, tier: 'expired', reason: 'expired' }
      }
    },

    async activate(key: string): Promise<boolean> {
      const license = await validateOnline(key)
      return license !== null
    },

    async deactivate(): Promise<void> {
      await Bun.write(cachePath, '')
    },

    async refresh(): Promise<boolean> {
      const cached = await readCache()
      if (!cached) return false
      
      const license = await validateOnline(cached.key)
      return license !== null
    }
  }
}
```

---

## 6. Auto-Update

### 6.1 Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTO-UPDATE FLOW                              │
│                                                                  │
│  App Running                                                     │
│      │                                                           │
│      ▼                                                           │
│  Check for updates (background, every 4 hours)                  │
│  GET https://releases.punk.dev/mohawk/latest.json               │
│      │                                                           │
│      ├── No update ──► Continue silently                        │
│      │                                                           │
│      ▼                                                           │
│  Update available                                                │
│  Show notification: "Mohawk 2.1.0 available"                    │
│      │                                                           │
│      ├── User clicks "Later" ──► Remind next session            │
│      │                                                           │
│      ▼                                                           │
│  User clicks "Update"                                            │
│  Download in background                                          │
│  Show progress bar                                               │
│      │                                                           │
│      ▼                                                           │
│  Download complete                                               │
│  "Update ready — Restart to apply"                              │
│      │                                                           │
│      ▼                                                           │
│  User clicks "Restart"                                           │
│  Install update                                                  │
│  Relaunch app                                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Implementation

```typescript
// src/main/updater.ts
import { autoUpdater } from 'electron-updater'
import { ipcMain, BrowserWindow } from 'electron'
import log from 'electron-log'

export function setupAutoUpdater(mainWindow: BrowserWindow) {
  autoUpdater.logger = log
  autoUpdater.autoDownload = false
  autoUpdater.autoInstallOnAppQuit = true

  // Check every 4 hours
  setInterval(() => {
    autoUpdater.checkForUpdates()
  }, 4 * 60 * 60 * 1000)

  // Initial check after 10 seconds
  setTimeout(() => {
    autoUpdater.checkForUpdates()
  }, 10000)

  autoUpdater.on('update-available', (info) => {
    mainWindow.webContents.send('update:available', {
      version: info.version,
      releaseNotes: info.releaseNotes
    })
  })

  autoUpdater.on('download-progress', (progress) => {
    mainWindow.webContents.send('update:progress', {
      percent: progress.percent,
      transferred: progress.transferred,
      total: progress.total
    })
  })

  autoUpdater.on('update-downloaded', () => {
    mainWindow.webContents.send('update:ready')
  })

  // IPC handlers
  ipcMain.handle('update:download', () => {
    autoUpdater.downloadUpdate()
  })

  ipcMain.handle('update:install', () => {
    autoUpdater.quitAndInstall()
  })
}
```

---

## 7. Security Model

### 7.1 Process Isolation

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY BOUNDARIES                           │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  RENDERER (Sandboxed)                                      │  │
│  │                                                            │  │
│  │  ✗ No Node.js access                                       │  │
│  │  ✗ No file system access                                   │  │
│  │  ✗ No network access (except same-origin)                  │  │
│  │  ✗ No native module loading                                │  │
│  │                                                            │  │
│  │  ✓ Only window.mohawk API (via contextBridge)             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                        contextBridge                             │
│                      (explicit allowlist)                        │
│                              │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  MAIN PROCESS (Full access)                                │  │
│  │                                                            │  │
│  │  ✓ File system                                             │  │
│  │  ✓ Network                                                 │  │
│  │  ✓ Native modules                                          │  │
│  │  ✓ Child processes                                         │  │
│  │                                                            │  │
│  │  All sensitive ops go through validated IPC handlers       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Mod Sandboxing

- File access restricted to mod's directory
- Network access restricted to manifest-declared domains
- Database access namespaced to mod's data
- No access to system APIs beyond exposed surface

---

## 8. Build & Distribution

### 8.1 Electron Builder Config

```javascript
// electron-builder.config.js
module.exports = {
  appId: 'dev.punk.mohawk',
  productName: 'Mohawk',
  
  directories: {
    output: 'dist',
    buildResources: 'build'
  },

  files: [
    'out/**/*',
    'node_modules/**/*'
  ],

  mac: {
    category: 'public.app-category.developer-tools',
    hardenedRuntime: true,
    gatekeeperAssess: false,
    entitlements: 'build/entitlements.mac.plist',
    entitlementsInherit: 'build/entitlements.mac.plist',
    target: [
      { target: 'dmg', arch: ['x64', 'arm64'] },
      { target: 'zip', arch: ['x64', 'arm64'] }
    ]
  },

  win: {
    target: [
      { target: 'nsis', arch: ['x64'] },
      { target: 'portable', arch: ['x64'] }
    ],
    sign: './scripts/sign.js'
  },

  linux: {
    target: [
      { target: 'AppImage', arch: ['x64'] },
      { target: 'deb', arch: ['x64'] },
      { target: 'rpm', arch: ['x64'] }
    ],
    category: 'Development'
  },

  publish: {
    provider: 'github',
    owner: 'PerceptLabs',
    repo: 'mohawk'
  }
}
```

### 8.2 Distribution Channels

| Platform | Formats |
|----------|---------|
| **macOS** | .dmg, Homebrew Cask |
| **Windows** | .exe installer, Portable .zip, WinGet |
| **Linux** | AppImage, .deb, .rpm |

---

## 9. Code Sharing with Web

### 9.1 Monorepo Structure

```
punk/
├── packages/
│   ├── core/           # @punk/core
│   ├── ui/             # @punk/ui (shared React components)
│   └── types/          # @punk/types
│
├── apps/
│   ├── mohawk-web/     # SaaS
│   └── mohawk-desktop/ # Electron app
```

### 9.2 Platform Abstraction

```typescript
// packages/ui/src/platform/types.ts
export interface PlatformAPI {
  db: { query, execute }
  ai: { generate, streamGenerate }
  projects: { list, create, save }
  mods: { list, execute }
  platform: 'web' | 'desktop'
  isOffline: boolean
}

// packages/ui/src/platform/context.tsx
const PlatformContext = createContext<PlatformAPI | null>(null)
export const PlatformProvider = PlatformContext.Provider
export function usePlatform(): PlatformAPI { ... }
```

### 9.3 Shared Entry Point

```tsx
// packages/ui/src/MohawkApp.tsx
export function MohawkApp({ platform }: { platform: PlatformAPI }) {
  return (
    <PlatformProvider value={platform}>
      <ThemeProvider>
        <MohawkShell />
      </ThemeProvider>
    </PlatformProvider>
  )
}

// apps/mohawk-desktop/src/renderer/index.tsx
import { MohawkApp } from '@punk/ui'
import { desktopPlatform } from './platform'

createRoot(document.getElementById('root')!).render(
  <MohawkApp platform={desktopPlatform} />
)
```

---

*Last updated: December 2025*
