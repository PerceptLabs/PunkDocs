# Biome Implementation Guide

**Version:** 1.0  
**Status:** Source of Truth  
**Last Updated:** December 2025  
**Audience:** Human developers and AI coding assistants (Claude Code, Cursor, etc.)  
**License:** MIT / Apache 2.0 (permissive, commercial-safe)  
**Replaces:** ESLint, Prettier, Husky (lint-staged), dprint

---

## Executive Summary

Biome is a Rust-based toolchain that replaces the entire JavaScript linting and formatting ecosystem with a single, fast binary. It embodies Punk Pragmatism: **one tool, strict defaults, instant feedback**.

**Why Biome:**

- **97% Prettier compatible** — Same formatting, 35x faster
- **340+ ESLint rules** — TypeScript, React, accessibility built-in
- **Single binary** — No plugin ecosystem to manage
- **Zero config to start** — Sane defaults out of the box
- **Instant feedback** — Formats large files in milliseconds

---

## Philosophy Alignment

| Punk Principle | Biome Implementation |
|----------------|----------------------|
| Safety over Smarts | Strict defaults catch errors early |
| One Tool | Replaces ESLint + Prettier + plugins |
| Zero Config Hell | Single `biome.json`, sensible defaults |
| Rust Speed | Formats/lints in milliseconds |
| Predictable | Same output everywhere, no plugin conflicts |

---

## Installation

### With Bun

```bash
# Install as dev dependency (pinned version)
bun add -d @biomejs/biome --exact

# Initialize configuration
bunx biome init
```

### Verify Installation

```bash
bunx biome --version
# Biome 1.9.x
```

---

## Configuration

### Minimal Config (Recommended Start)

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

### Full Punk Config

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  
  "organizeImports": {
    "enabled": true
  },
  
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error",
        "useExhaustiveDependencies": "warn"
      },
      
      "suspicious": {
        "noExplicitAny": "error",
        "noArrayIndexKey": "warn"
      },
      
      "complexity": {
        "noForEach": "warn",
        "useFlatMap": "error"
      },
      
      "style": {
        "useConst": "error",
        "useTemplate": "error",
        "noNonNullAssertion": "warn"
      },
      
      "a11y": {
        "recommended": true
      }
    }
  },
  
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "trailingCommas": "es5",
      "semicolons": "asNeeded"
    }
  },
  
  "json": {
    "formatter": {
      "trailingCommas": "none"
    }
  },
  
  "files": {
    "ignore": [
      "node_modules",
      "dist",
      "build",
      ".next",
      "coverage",
      "*.min.js",
      "*.d.ts"
    ]
  }
}
```

---

## CLI Commands

### Format

```bash
# Format all files
bunx biome format --write .

# Format specific directory
bunx biome format --write ./src

# Check formatting without writing
bunx biome format ./src
```

### Lint

```bash
# Lint all files
bunx biome lint ./src

# Lint and apply safe fixes
bunx biome lint --write ./src

# Lint with unsafe fixes (review carefully)
bunx biome lint --write --unsafe ./src
```

### Check (Format + Lint + Organize Imports)

```bash
# Check everything
bunx biome check ./src

# Check and fix everything
bunx biome check --write ./src

# CI mode (exits with error on issues)
bunx biome ci ./src
```

---

## Package.json Scripts

```json
{
  "scripts": {
    "lint": "biome lint ./src",
    "lint:fix": "biome lint --write ./src",
    "format": "biome format --write ./src",
    "check": "biome check --write ./src",
    "ci": "biome ci ./src"
  }
}
```

---

## Editor Integration

### VS Code

Install the official extension: `biomejs.biome`

```json
// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[json]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

### Cursor

Same as VS Code — install `biomejs.biome` extension.

### Neovim

```lua
-- Using nvim-lspconfig
require('lspconfig').biome.setup{}
```

### Zed

Biome is built-in to Zed.

---

## CI Integration

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: oven-sh/setup-bun@v1
        with:
          bun-version: latest
      
      - run: bun install
      
      - name: Biome CI
        run: bunx biome ci ./src
```

### Pre-commit Hook (Simple)

```bash
# .husky/pre-commit (if using Husky)
bunx biome check --staged
```

### Pre-commit Hook (Without Husky)

```bash
# Install simple-git-hooks
bun add -d simple-git-hooks

# package.json
{
  "simple-git-hooks": {
    "pre-commit": "bunx biome check --staged"
  }
}

# Activate
bunx simple-git-hooks
```

---

## Migration from ESLint + Prettier

### 1. Remove Old Dependencies

```bash
bun remove eslint prettier eslint-config-prettier eslint-plugin-react \
  eslint-plugin-react-hooks @typescript-eslint/parser \
  @typescript-eslint/eslint-plugin husky lint-staged
```

### 2. Remove Old Config Files

```bash
rm .eslintrc* .prettierrc* .eslintignore .prettierignore
```

### 3. Install Biome

```bash
bun add -d @biomejs/biome --exact
bunx biome init
```

### 4. Run Migration Helper

```bash
# Biome can convert some ESLint rules
bunx biome migrate eslint --write
bunx biome migrate prettier --write
```

### 5. First Run

```bash
# Format everything to new style
bunx biome check --write .

# Commit the formatting changes
git add -A
git commit -m "chore: migrate to biome"
```

---

## Rule Comparison

### ESLint → Biome

| ESLint Rule | Biome Rule |
|-------------|------------|
| `no-unused-vars` | `correctness/noUnusedVariables` |
| `no-console` | `suspicious/noConsoleLog` |
| `eqeqeq` | `suspicious/noDoubleEquals` |
| `prefer-const` | `style/useConst` |
| `@typescript-eslint/no-explicit-any` | `suspicious/noExplicitAny` |
| `react-hooks/exhaustive-deps` | `correctness/useExhaustiveDependencies` |
| `jsx-a11y/*` | `a11y/*` |

### What Biome Doesn't Have (Yet)

- Import sorting by path groups (basic sorting works)
- Some niche ESLint plugins
- Custom rule authoring

For most projects, Biome's 340+ rules cover everything needed.

---

## Ignoring Code

### File-Level Ignore

```javascript
// biome-ignore lint: reason for ignoring entire file
```

### Line-Level Ignore

```javascript
// biome-ignore lint/suspicious/noExplicitAny: legacy code
const data: any = getData()
```

### Block-Level Ignore

```javascript
// biome-ignore format: keep manual formatting
const matrix = [
  [1, 0, 0],
  [0, 1, 0],
  [0, 0, 1],
]
```

---

## Troubleshooting

### "Biome not found" in CI

Ensure `@biomejs/biome` is in `devDependencies`, not a global install.

### Formatting Conflicts with Existing Code

Run the full format once and commit:

```bash
bunx biome format --write .
git add -A
git commit -m "style: apply biome formatting"
```

### Editor Not Picking Up Biome

1. Ensure extension is installed
2. Reload window
3. Check that `biome.json` is in project root
4. Check Output panel for Biome errors

### Performance Issues

Biome is fast, but for very large monorepos:

```json
// biome.json
{
  "files": {
    "maxSize": 1048576,
    "ignore": ["**/generated/**", "**/vendor/**"]
  }
}
```

---

## Effect Integration

Biome works seamlessly with Effect codebases. No special configuration needed.

```typescript
// Biome understands Effect patterns
import { Effect, pipe } from "effect"

// biome-ignore lint/suspicious/noExplicitAny: Effect's internal types
type AnyEffect = Effect.Effect<any, any, any>

const program = pipe(
  Effect.succeed(42),
  Effect.map((n) => n * 2),
  Effect.flatMap((n) => Effect.succeed(n.toString()))
)
```

---

## Recommended Punk Rules

These rules align with Punk philosophy:

```json
{
  "linter": {
    "rules": {
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error"
      },
      "suspicious": {
        "noExplicitAny": "error",
        "noConsoleLog": "warn"
      },
      "style": {
        "useConst": "error",
        "noNonNullAssertion": "error"
      },
      "complexity": {
        "noForEach": "warn"
      }
    }
  }
}
```

**Why these matter:**

| Rule | Punk Reason |
|------|-------------|
| `noUnusedVariables` | Dead code is tech debt |
| `noExplicitAny` | Types are safety |
| `noNonNullAssertion` | Assertions hide bugs |
| `noForEach` | Prefer `.map()`, `.filter()` — functional style |
| `noConsoleLog` | Use proper logging |

---

## Summary

| Aspect | Before | After |
|--------|--------|-------|
| Tools | ESLint + Prettier + plugins | Biome |
| Config files | 3-5 files | 1 file |
| Dependencies | 10-20 packages | 1 package |
| Format time | Seconds | Milliseconds |
| Plugin conflicts | Common | Impossible |
| Learning curve | High | Low |

---

## Related Documents

- **PUNK_DECISIONS.md** — ADR for Biome adoption
- **PUNK_LIGHTNING_CSS_IMPL.md** — CSS tooling companion
- **PUNK_EFFECT_IMPL.md** — Effect patterns (Biome-compatible)

---

*Last updated: December 2025*
