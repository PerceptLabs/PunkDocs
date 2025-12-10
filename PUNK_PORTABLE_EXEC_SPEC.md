# Portable Executable Specification

**Version:** 1.0  
**Status:** Planned (Future Feature)  
**Last Updated:** December 2025  
**Audience:** Human developers and AI coding assistants  
**Depends On:** PUNK_DESKTOP_IMPL.md  
**Target:** Mohawk Desktop App (Pro+ tier)

---

## Executive Summary

Mohawk Desktop can export user applications as single-file portable executables using Cosmopolitan Libc / Redbean. Users get a single binary that runs on Windows, macOS, and Linux with no dependencies.

**Architecture:** Pre-built "golden base" binary + user assets appended via zip.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Golden Redbean Base (Mohawk-controlled, tested, signed)│
│  └─ redbean-mohawk-v1.x.x.com (~2-3 MB)                │
│     ├─ HTTP server                                      │
│     ├─ SQLite engine                                    │
│     ├─ Lua scripting runtime                           │
│     ├─ Gzip/Brotli compression                         │
│     ├─ TLS support                                      │
│     └─ Self-test diagnostics                           │
├─────────────────────────────────────────────────────────┤
│  User's App (appended via zip)                          │
│  └─ .mohawk/                                           │
│     ├─ config.lua        ← App configuration           │
│     ├─ manifest.json     ← Asset checksums             │
│     └─ flags.txt         ← Runtime flags               │
│  └─ www/                                               │
│     ├─ index.html                                      │
│     ├─ bundle.js                                       │
│     ├─ styles.css                                      │
│     └─ assets/                                         │
│  └─ data/ (optional)                                   │
│     └─ app.sqlite        ← Pre-seeded database         │
└─────────────────────────────────────────────────────────┘
                         ↓
              myapp.com (single file, ~3-10 MB)
```

### Key Principle

**We own the hard parts. User just brings content.**

| Component | Owner | Testing |
|-----------|-------|---------|
| HTTP server | Mohawk | Exhaustive CI matrix |
| SQLite engine | Mohawk | Exhaustive CI matrix |
| TLS/compression | Mohawk | Exhaustive CI matrix |
| Self-diagnostics | Mohawk | Exhaustive CI matrix |
| User's HTML/JS/CSS | User | Validated at bundle |
| User's config | Generated | Schema validated |
| User's data.sqlite | User | Schema validated |

---

## Golden Base Management

### Versioning

```
Golden Bases (hosted, cached locally):
├─ redbean-mohawk-v1.0.0.com  (deprecated)
├─ redbean-mohawk-v1.1.0.com  (stable)
├─ redbean-mohawk-v1.2.0.com  (latest)
└─ redbean-mohawk-v1.3.0-beta.com (testing)
```

### Base Contents

The golden base includes:

- **Redbean server** — Cosmopolitan-compiled HTTP server
- **SQLite** — Embedded database engine
- **Lua runtime** — For config and simple server-side logic
- **Compression** — Gzip/Brotli for assets
- **TLS** — HTTPS support with auto-certs
- **Self-test** — Diagnostic checks on startup

### CI Test Matrix

```yaml
# Golden base verification
test-golden-base:
  strategy:
    matrix:
      os: 
        - ubuntu-22.04
        - ubuntu-24.04
        - macos-13
        - macos-14
        - windows-2022
  steps:
    - run: ./redbean-mohawk.com --self-test
    - run: ./redbean-mohawk.com &
    - run: curl http://localhost:8080/
    - run: curl http://localhost:8080/__health
    - run: pkill -f redbean-mohawk
```

### Code Signing

| Platform | Approach | Purpose |
|----------|----------|---------|
| macOS | Developer ID + Notarization | Avoid Gatekeeper warnings |
| Windows | Authenticode signing | Avoid SmartScreen warnings |
| Linux | Unsigned (acceptable) | No platform gating |

---

## User App Layer

### Config File

```lua
-- .mohawk/config.lua (generated per app)

-- Server settings
PORT = 8080
HOST = "0.0.0.0"

-- SPA mode: all routes serve index.html
SPA_MODE = true

-- Compression
GZIP = true
BROTLI = true

-- Custom routes (optional)
ROUTES = {
  ["/api/health"] = function()
    return 200, "ok"
  end,
  ["/api/version"] = function()
    return 200, APP_VERSION
  end
}

-- CORS configuration (optional)
CORS = {
  enabled = true,
  origins = {"*"},
  methods = {"GET", "POST", "OPTIONS"},
  headers = {"Content-Type", "Authorization"}
}

-- SQLite database (optional)
DATABASE = {
  enabled = false,
  path = "data/app.sqlite",
  wal_mode = true
}
```

### Manifest File

```json
{
  "app": {
    "name": "My Landing Page",
    "version": "1.0.0",
    "created": "2025-12-05T12:00:00Z"
  },
  "base": {
    "version": "redbean-mohawk-v1.2.0",
    "checksum": "sha256:abc123..."
  },
  "files": {
    "www/index.html": "sha256:def456...",
    "www/bundle.js": "sha256:ghi789...",
    "www/styles.css": "sha256:jkl012..."
  },
  "checksum": "sha256:mno345..."
}
```

### Runtime Flags

```
# .mohawk/flags.txt (default runtime behavior)
--port=8080
--daemonize=false
--log-level=info
--max-connections=100
--timeout=30
```

User can override at runtime:
```bash
./myapp.com                    # Uses defaults
./myapp.com --port=3000        # Override port
./myapp.com --daemonize        # Run in background
./myapp.com --log-level=debug  # Verbose logging
```

---

## Bundle Process

### Build Flow

```typescript
const buildPortableApp = (userBuild: BuildOutput): Effect<PortableApp, BuildError, GoldenBaseService> =>
  Effect.gen(function* () {
    const goldenBase = yield* GoldenBaseService
    
    // 1. Get golden base (cached locally or fetch)
    const base = yield* goldenBase.get("latest")
    
    // 2. Validate user assets
    const validatedAssets = yield* validateAssets(userBuild.files)
    
    // 3. Generate config from user preferences
    const config = generateLuaConfig({
      port: userBuild.preferredPort ?? 8080,
      spa: userBuild.isSPA,
      routes: userBuild.customRoutes,
      cors: userBuild.corsConfig,
      database: userBuild.hasDatabase
    })
    
    // 4. Create manifest with checksums
    const manifest = yield* createManifest({
      app: userBuild.metadata,
      base: base.metadata,
      files: validatedAssets
    })
    
    // 5. Prepare zip contents
    const zipContents = [
      { path: ".mohawk/config.lua", content: config },
      { path: ".mohawk/manifest.json", content: JSON.stringify(manifest) },
      { path: ".mohawk/flags.txt", content: generateFlags(userBuild) },
      ...validatedAssets.map(f => ({ 
        path: `www/${f.relativePath}`, 
        content: f.content 
      }))
    ]
    
    // 6. Add optional database
    if (userBuild.database) {
      zipContents.push({
        path: "data/app.sqlite",
        content: userBuild.database
      })
    }
    
    // 7. Append to base binary
    const bundled = yield* appendZipToExecutable(base.binary, zipContents)
    
    // 8. Verify bundle integrity
    yield* verifyBundle(bundled, manifest)
    
    // 9. Return with metadata
    return {
      binary: bundled,
      manifest,
      size: bundled.byteLength,
      filename: `${userBuild.metadata.name}.com`
    }
  })
```

### Verification

```typescript
const verifyBundle = (binary: Uint8Array, manifest: Manifest): Effect<void, VerifyError, never> =>
  Effect.all([
    // Verify zip structure is valid
    verifyZipIntegrity(binary),
    
    // Verify all manifest files present
    verifyFilesPresent(binary, manifest.files),
    
    // Verify checksums match
    verifyChecksums(binary, manifest.files),
    
    // Run self-test in sandbox (if available)
    runSelfTestInSandbox(binary).pipe(
      Effect.catchAll(() => Effect.void)  // Optional, don't fail if no sandbox
    )
  ])
```

---

## Self-Test Diagnostics

Golden base includes self-diagnostics that run on startup:

```lua
-- Built into golden base
function SelfTest()
  local results = {}
  
  -- 1. Verify zip integrity
  results.zip = VerifyZipIntegrity()
  
  -- 2. Check port availability
  results.port = CheckPortAvailable(PORT)
  
  -- 3. Verify temp directory writable
  results.temp = CheckTempWritable()
  
  -- 4. Test SQLite if enabled
  if DATABASE.enabled then
    results.sqlite = TestSqliteConnection()
  end
  
  -- 5. Verify all www files accessible
  results.assets = VerifyAssetsAccessible()
  
  -- Report
  if AllPassed(results) then
    return true
  else
    ShowDiagnosticReport(results)
    return false
  end
end
```

User can run manually:
```bash
./myapp.com --self-test
# Output:
# ✓ Zip integrity: OK
# ✓ Port 8080: Available
# ✓ Temp directory: Writable
# ✓ SQLite: Connected
# ✓ Assets: 12/12 accessible
# All checks passed.
```

---

## Size Estimates

| Component | Typical Size |
|-----------|--------------|
| Golden base | ~2.5 MB |
| Simple landing page | ~200 KB |
| SPA with assets | ~1-3 MB |
| SQLite database | ~100 KB - 5 MB |
| **Total typical** | **3-8 MB** |

Small enough to:
- Email to a client
- Host on any server
- Run from USB drive

---

## Failure Modes & Mitigations

| Risk | Mitigation |
|------|------------|
| Build variability | Pre-built golden base, no per-user compilation |
| Asset corruption | SHA256 manifest, verify after bundle |
| Platform quirks | CI test matrix (Win/Mac/Linux) |
| Gatekeeper/SmartScreen | Code signing |
| Port conflicts | Auto-find open port, clear error message |
| Large binaries | Asset optimization pass, tree-shake |
| SQLite locking | WAL mode, platform-specific testing |
| Antivirus false positives | Sign + submit to AV vendors |

---

## Confidence Levels

| Scenario | Confidence |
|----------|------------|
| Static site serving | 🟢 99%+ |
| SQLite read/write | 🟢 95%+ |
| Works on Linux (unsigned) | 🟢 99% |
| Works on macOS (signed) | 🟢 95%+ |
| Works on Windows (signed) | 🟡 90%+ |
| Works on Windows (unsigned) | 🟠 70% |

**Windows unsigned is the weak spot.** Code signing is required for reliable Windows distribution.

---

## Tier Gating

| Tier | Portable Executable |
|------|---------------------|
| Free | ❌ |
| Starter | ❌ |
| Pro | ❌ |
| Pro+ | ✅ |

Rationale: High-value differentiator for top tier. Requires infrastructure (golden base hosting, signing certs).

---

## Future Considerations

### Potential Enhancements

1. **Auto-update mechanism** — Binary phones home for updates
2. **Plugin system** — User can add Lua plugins
3. **Server-side rendering** — Lua-based SSR for SEO
4. **WebSocket support** — Real-time features
5. **Background jobs** — Scheduled tasks via Lua

### Alternative: Docker Export

For users who can't use Cosmopolitan:

```dockerfile
# Generated Dockerfile
FROM nginx:alpine
COPY www/ /usr/share/nginx/html/
EXPOSE 80
```

```bash
docker build -t myapp .
docker run -p 8080:80 myapp
```

Fallback when portable exe isn't suitable.

---

## Implementation Status

**Current:** Specification only (this document)

**Next Steps (when prioritized):**
1. Build and test golden base
2. Set up signing infrastructure
3. Implement bundle process in Electron app
4. CI verification pipeline
5. Beta testing with Pro+ users

---

## Related Documents

- **PUNK_DESKTOP_IMPL.md** — Electron app architecture
- **PUNK_DECISIONS.md** — ADR for Cosmopolitan adoption (TBD)

---

*Last updated: December 2025*
