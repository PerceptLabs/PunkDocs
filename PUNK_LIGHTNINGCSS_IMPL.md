# Lightning CSS Implementation Guide

**Version:** 2.0  
**Status:** Source of Truth  
**Last Updated:** December 2025  
**Audience:** Human developers and AI coding assistants (Claude Code, Cursor, etc.)  
**License:** MPL 2.0 (weak copyleft, commercial-safe for usage)  
**Replaces:** PostCSS, cssnano, Autoprefixer, postcss-preset-env

---

## Executive Summary

Lightning CSS is a Rust-based CSS parser, transformer, and minifier. It's the "final polish" step that takes your CSS (from Pink, TokiForge, or raw) and outputs the smallest, most compatible bundle possible.

**Why Lightning CSS:**

- **Rust speed** â€” 100x faster than PostCSS
- **Spec-compliant** â€” Parses CSS per W3C grammar
- **Modern CSS downleveling** â€” Nesting, color functions, container queries
- **Automatic vendor prefixes** â€” Based on browser targets
- **Aggressive minification** â€” Smaller than cssnano

---

## Philosophy Alignment

| Punk Principle | Lightning CSS Implementation |
|----------------|------------------------------|
| Safety over Smarts | Spec-compliant parsing, no surprises |
| One Tool | Replaces entire PostCSS plugin chain |
| Rust Speed | Processes megabytes in milliseconds |
| Predictable | Deterministic output based on targets |

---

## Role in Punk Stack

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Design System Layer                                    â”‚
â”‚  â”œâ”€ Base UI (behavior)                                â”‚
â”‚  â”œâ”€ Pink CSS (visual classes)                          â”‚
â”‚  â””â”€ TokiForge (runtime tokens)                         â”‚
â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¤
â”‚  Build Layer                                            â”‚
â”‚  â””â”€ Lightning CSS                                       â”‚
â”‚     â”œâ”€ Downlevel modern syntax                         â”‚
â”‚     â”œâ”€ Add vendor prefixes                             â”‚
â”‚     â”œâ”€ Minify aggressively                             â”‚
â”‚     â””â”€ Generate source maps                            â”‚
â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¤
â”‚  Output                                                 â”‚
â”‚  â””â”€ bundle.css (minimal, compatible)                   â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

---

## Installation

### With Bun

```bash
bun add -d lightningcss
```

### Verify Installation

```typescript
import { transform } from "lightningcss"
console.log("Lightning CSS ready")
```

---

## Basic Usage

### Transform CSS

```typescript
import { transform } from "lightningcss"

const css = `
.button {
  color: oklch(70% 0.15 200);
  
  &:hover {
    color: oklch(80% 0.15 200);
  }
}
`

const { code, map } = transform({
  filename: "styles.css",
  code: Buffer.from(css),
  minify: true,
  sourceMap: true,
  targets: {
    chrome: 100 << 16,  // Chrome 100
    safari: 15 << 16,   // Safari 15
    firefox: 100 << 16, // Firefox 100
  },
})

console.log(code.toString())
// Output: minified, prefixed, downleveled CSS
```

### Browser Targets with Browserslist

```typescript
import { transform, browserslistToTargets } from "lightningcss"
import browserslist from "browserslist"

const targets = browserslistToTargets(browserslist(">= 0.25%"))

const { code } = transform({
  filename: "styles.css",
  code: Buffer.from(css),
  minify: true,
  targets,
})
```

---

## Bun Build Integration

### Simple Build Script

```typescript
// scripts/build-css.ts
import { transform, browserslistToTargets } from "lightningcss"
import browserslist from "browserslist"

const inputFile = "./src/styles/main.css"
const outputFile = "./dist/styles.css"

const css = await Bun.file(inputFile).text()

const targets = browserslistToTargets(
  browserslist("defaults and supports css-nesting")
)

const { code, map } = transform({
  filename: inputFile,
  code: Buffer.from(css),
  minify: true,
  sourceMap: true,
  targets,
  drafts: {
    customMedia: true,
  },
})

await Bun.write(outputFile, code)
await Bun.write(`${outputFile}.map`, JSON.stringify(map))

console.log(`âœ“ Built ${outputFile} (${code.byteLength} bytes)`)
```

### Package.json Scripts

```json
{
  "scripts": {
    "build:css": "bun run scripts/build-css.ts",
    "watch:css": "bun run scripts/watch-css.ts"
  }
}
```

---

## Features

### 1. CSS Nesting (Downleveled)

```css
/* Input: Modern CSS */
.card {
  background: white;
  
  & .title {
    font-size: 1.5rem;
  }
  
  &:hover {
    background: #f5f5f5;
  }
}

/* Output: Compatible CSS */
.card {
  background: #fff;
}
.card .title {
  font-size: 1.5rem;
}
.card:hover {
  background: #f5f5f5;
}
```

### 2. Color Functions (Downleveled)

```css
/* Input: Modern color spaces */
.button {
  background: oklch(70% 0.15 200);
  border-color: color-mix(in oklch, var(--primary) 50%, white);
}

/* Output: Fallbacks for older browsers */
.button {
  background: #3b9ccc;
  background: oklch(70% .15 200);
  border-color: #9fcce6;
}
```

### 3. Vendor Prefixes (Automatic)

```css
/* Input */
.flex-container {
  display: flex;
  user-select: none;
}

/* Output (based on targets) */
.flex-container {
  display: -webkit-box;
  display: -webkit-flex;
  display: flex;
  -webkit-user-select: none;
  user-select: none;
}
```

### 4. Minification

Lightning CSS applies:
- Whitespace removal
- Comment stripping
- Color shortening (`#ffffff` â†’ `#fff`)
- Shorthand merging
- Duplicate rule removal
- Selector merging
- `calc()` simplification
- Zero-unit removal (`0px` â†’ `0`)

### 5. CSS Modules (Optional)

```typescript
import { transform } from "lightningcss"

const { code, exports } = transform({
  filename: "component.module.css",
  code: Buffer.from(css),
  cssModules: true,
})

// exports = { className: "className_abc123", ... }
```

---

## Configuration Reference

### Full Options

```typescript
import { transform, Features } from "lightningcss"

const result = transform({
  // Required
  filename: "styles.css",
  code: Buffer.from(css),
  
  // Browser targets (required for prefixing/downleveling)
  targets: {
    chrome: 100 << 16,
    safari: 15 << 16,
    firefox: 100 << 16,
  },
  
  // Minification
  minify: true,
  
  // Source maps
  sourceMap: true,
  inputSourceMap: undefined, // Previous source map
  
  // Features to compile (bitflags)
  include: Features.Nesting | Features.Colors,
  exclude: Features.None,
  
  // Draft specs to enable
  drafts: {
    customMedia: true,
  },
  
  // CSS Modules
  cssModules: false,
  
  // Error recovery
  errorRecovery: false,
  
  // Custom pseudoClasses for frameworks
  pseudoClasses: {
    hover: ":is(:hover, .is-hovered)",
  },
  
  // Unused symbol analysis
  unusedSymbols: ["unused-class"],
})
```

### Feature Flags

```typescript
import { Features } from "lightningcss"

// Include only specific features
const include = 
  Features.Nesting |
  Features.Colors |
  Features.MediaQueries

// Exclude features (let browser handle)
const exclude = Features.VendorPrefixes
```

---

## Integration Patterns

### With Vite

```typescript
// vite.config.ts
import { defineConfig } from "vite"

export default defineConfig({
  css: {
    transformer: "lightningcss",
    lightningcss: {
      targets: {
        chrome: 100 << 16,
        safari: 15 << 16,
      },
      drafts: {
        customMedia: true,
      },
    },
  },
  build: {
    cssMinify: "lightningcss",
  },
})
```

### With Bun Bundler

```typescript
// bun.build.ts
import { transform, browserslistToTargets } from "lightningcss"
import browserslist from "browserslist"

const targets = browserslistToTargets(browserslist("defaults"))

// Post-process CSS after Bun bundles
export async function processCss(cssContent: string): Promise<string> {
  const { code } = transform({
    filename: "bundle.css",
    code: Buffer.from(cssContent),
    minify: true,
    targets,
  })
  return code.toString()
}
```

### With TokiForge

```typescript
// TokiForge generates CSS variables at runtime
// Lightning CSS processes the static base styles

// 1. Build static CSS with Lightning CSS
const staticCss = await buildCss("./src/styles/base.css")

// 2. TokiForge injects runtime tokens
// <style id="tokiforge-tokens">:root { --color-primary: ... }</style>

// 3. Components use both
// .button { background: var(--color-primary); }
```

---

## Punk CSS Workflow

### Directory Structure

```
src/
â”œâ”€â”€ styles/
â”‚   â”œâ”€â”€ base.css        # Reset, typography
â”‚   â”œâ”€â”€ tokens.css      # CSS custom properties
â”‚   â”œâ”€â”€ components/     # Component styles
â”‚   â”‚   â”œâ”€â”€ button.css
â”‚   â”‚   â”œâ”€â”€ card.css
â”‚   â”‚   â””â”€â”€ input.css
â”‚   â””â”€â”€ main.css        # @import aggregator
â””â”€â”€ ...

scripts/
â””â”€â”€ build-css.ts        # Lightning CSS build
```

### Main CSS Entry

```css
/* src/styles/main.css */
@import "./base.css";
@import "./tokens.css";
@import "./components/button.css";
@import "./components/card.css";
@import "./components/input.css";
```

### Build with Bundling

```typescript
// scripts/build-css.ts
import { bundle, browserslistToTargets } from "lightningcss"
import browserslist from "browserslist"

const targets = browserslistToTargets(browserslist("defaults"))

const { code, map } = await bundle({
  filename: "./src/styles/main.css",
  minify: true,
  sourceMap: true,
  targets,
  drafts: {
    customMedia: true,
  },
})

await Bun.write("./dist/styles.css", code)
console.log(`âœ“ Bundled CSS (${code.byteLength} bytes)`)
```

---

## Watch Mode

```typescript
// scripts/watch-css.ts
import { watch } from "fs"
import { bundle, browserslistToTargets } from "lightningcss"
import browserslist from "browserslist"

const targets = browserslistToTargets(browserslist("defaults"))

async function build() {
  const start = Date.now()
  
  try {
    const { code } = await bundle({
      filename: "./src/styles/main.css",
      minify: false, // Don't minify in dev
      targets,
    })
    
    await Bun.write("./dist/styles.css", code)
    console.log(`âœ“ Built in ${Date.now() - start}ms`)
  } catch (error) {
    console.error("âœ— Build failed:", error.message)
  }
}

// Initial build
await build()

// Watch for changes
watch("./src/styles", { recursive: true }, async (event, filename) => {
  if (filename?.endsWith(".css")) {
    console.log(`\n${filename} changed`)
    await build()
  }
})

console.log("Watching for CSS changes...")
```

---

## Size Comparison

| Tool | Bundle Size | Build Time |
|------|-------------|------------|
| cssnano (aggressive) | 42 KB | 1.2s |
| esbuild | 45 KB | 0.1s |
| **Lightning CSS** | **38 KB** | **0.05s** |

Lightning CSS typically produces 5-15% smaller output than alternatives.

---

## Troubleshooting

### "Unknown at-rule" Warnings

Lightning CSS is strict about CSS spec. For custom at-rules:

```typescript
// Add to drafts if using draft specs
drafts: {
  customMedia: true,
}

// Or add custom at-rules handler
// (not yet supported, use errorRecovery)
errorRecovery: true,
```

### Source Maps Not Working

Ensure you're writing the map file:

```typescript
const { code, map } = transform({ sourceMap: true, ... })

await Bun.write("styles.css", code)
await Bun.write("styles.css.map", JSON.stringify(map))
```

And add the sourcemap comment:

```typescript
const codeWithMap = code.toString() + 
  "\n/*# sourceMappingURL=styles.css.map */"
```

### Import Resolution

Use `bundle()` instead of `transform()` for `@import`:

```typescript
// âŒ transform() doesn't resolve imports
const { code } = transform({
  code: Buffer.from('@import "./other.css"'),
  ...
})

// âœ… bundle() resolves imports
const { code } = await bundle({
  filename: "./main.css",  // Entry point
  ...
})
```

---

## Best Practices

### 1. Use Browserslist

```json
// package.json
{
  "browserslist": [
    "defaults",
    "not IE 11",
    "maintained node versions"
  ]
}
```

```typescript
import browserslist from "browserslist"
import { browserslistToTargets } from "lightningcss"

const targets = browserslistToTargets(browserslist())
```

### 2. Production vs Development

```typescript
const isDev = process.env.NODE_ENV === "development"

const { code } = transform({
  code: Buffer.from(css),
  minify: !isDev,           // Only minify in prod
  sourceMap: isDev,         // Source maps in dev
  targets: isDev 
    ? {} // No transforms in dev (faster)
    : browserslistToTargets(browserslist()),
})
```

### 3. Cache Busting

```typescript
import { createHash } from "crypto"

const { code } = transform({ ... })
const hash = createHash("md5")
  .update(code)
  .digest("hex")
  .slice(0, 8)

await Bun.write(`./dist/styles.${hash}.css`, code)
```

---

## Summary

| Aspect | PostCSS | Lightning CSS |
|--------|---------|---------------|
| Speed | Slow | 100x faster |
| Config | Complex plugin chains | Single call |
| Output size | Good | Best |
| Spec compliance | Plugin-dependent | Built-in |
| Vendor prefixes | autoprefixer plugin | Built-in |
| Minification | cssnano plugin | Built-in |
| Modern CSS | postcss-preset-env | Built-in |

---

## Related Documents

- **PUNK_BIOME_IMPL.md** â€” JavaScript/TypeScript tooling companion
- **PUNK_DECISIONS.md** â€” ADR for Lightning CSS adoption
- **PUNK_DESKTOP_IMPL.md** â€” Uses Lightning CSS for app styling

---

*Last updated: December 2025*
