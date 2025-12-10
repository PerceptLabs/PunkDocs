# PUNK THEME SYSTEM
## Source of Truth v1.0

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Theme Anatomy](#theme-anatomy)
4. [TokiForge Token Specification](#tokiforge-token-specification)
5. [Pink CSS Layer](#pink-css-layer)
6. [Theme Distribution](#theme-distribution)
7. [Implementation Guide: Human Developers](#implementation-guide-human-developers)
8. [Implementation Guide: Claude Code](#implementation-guide-claude-code)
9. [Theme Marketplace Integration](#theme-marketplace-integration)
10. [Examples](#examples)

---

## Overview

### Philosophy

The Punk Theme System separates **structure** from **style**:

- **Base UI** provides unstyled, accessible primitives
- **Themes** provide visual identity through tokens and CSS
- **Rigs** compose Base UI components with theme awareness
- **Apps** consume themes without coupling to implementation

This separation enables:
- AI-safe UI generation (structure is deterministic, style is swappable)
- Marketplace monetization (free vs premium themes)
- Brand consistency across Rigs and Mods
- Runtime theme switching without component changes

### Core Principle

```
Base UI (structure) × Theme (style) = Rendered UI
```

A Button is always a Button. A Theme makes it look brutalist, glassmorphic, or 8-bit.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            USER APPLICATION                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ consumes
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              THEME BUNDLE                                │
│  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │     TokiForge Tokens        │  │         Pink CSS                 │  │
│  │  (Design Token JSON)        │  │   (Utility Classes + Overrides)  │  │
│  └─────────────────────────────┘  └─────────────────────────────────┘  │
│                                    │                                     │
│  ┌─────────────────────────────┐  │                                     │
│  │     Theme Metadata          │  │                                     │
│  │  (name, author, tier, etc)  │  │                                     │
│  └─────────────────────────────┘  │                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ styles
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              BASE UI                                     │
│  Surface │ Box │ Stack │ Row │ Column │ Grid │ Text │ Button │ Input   │
│                                                                          │
│  • Pure, unstyled primitives                                            │
│  • Accessibility built-in (ARIA, keyboard nav)                          │
│  • Theme-aware via CSS custom properties                                │
│  • Deterministic rendering                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ renders via
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            GARDEN.JS                                     │
│  • Adapter layer between Schema → Base UI                               │
│  • Provides Pride context (theme provider)                              │
│  • Ensures cross-host consistency                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Theme Anatomy

A theme is a distributable package containing:

```
theme-name/
├── theme.json           # Metadata and configuration
├── tokens/
│   ├── base.json        # Core tokens (always required)
│   ├── semantic.json    # Semantic aliases
│   ├── components.json  # Component-specific tokens (optional)
│   └── dark.json        # Dark mode overrides (optional)
├── pink/
│   ├── base.css         # Core Pink overrides
│   ├── components.css   # Component-specific styles
│   └── utilities.css    # Additional utility classes
├── assets/              # Optional: fonts, images, SVGs
│   ├── fonts/
│   └── images/
├── preview/             # For Garden.js / Depot display
│   ├── thumbnail.png
│   └── showcase.html
└── README.md
```

### theme.json

```json
{
  "$schema": "https://depot.punk.dev/schema/theme.json",
  "name": "retro-8bit",
  "displayName": "Retro 8-Bit",
  "version": "1.0.0",
  "author": {
    "name": "PerceptLabs",
    "url": "https://perceptlabs.dev"
  },
  "description": "Pixel-perfect nostalgia with CRT glow effects",
  "license": "MIT",
  "tier": "free",
  "tags": ["retro", "pixel", "gaming", "8-bit"],
  "compatibility": {
    "punk": ">=1.0.0",
    "tokiforge": ">=1.0.0",
    "pink": ">=1.0.0"
  },
  "tokens": {
    "base": "tokens/base.json",
    "semantic": "tokens/semantic.json",
    "components": "tokens/components.json",
    "modes": {
      "dark": "tokens/dark.json"
    }
  },
  "css": {
    "base": "pink/base.css",
    "components": "pink/components.css",
    "utilities": "pink/utilities.css"
  },
  "assets": {
    "fonts": [
      {
        "family": "Press Start 2P",
        "src": "assets/fonts/PressStart2P.woff2",
        "weight": "400",
        "style": "normal"
      }
    ]
  },
  "preview": {
    "thumbnail": "preview/thumbnail.png",
    "showcase": "preview/showcase.html"
  }
}
```

---

## TokiForge Token Specification

TokiForge tokens follow the [Design Tokens Format](https://design-tokens.github.io/community-group/format/) with Punk-specific extensions.

### Token Categories

#### 1. Color Tokens

```json
{
  "color": {
    "primitive": {
      "gray": {
        "50": { "$value": "#fafafa", "$type": "color" },
        "100": { "$value": "#f4f4f5", "$type": "color" },
        "900": { "$value": "#18181b", "$type": "color" }
      },
      "brand": {
        "primary": { "$value": "#7c3aed", "$type": "color" },
        "secondary": { "$value": "#06b6d4", "$type": "color" }
      }
    },
    "semantic": {
      "background": {
        "default": { "$value": "{color.primitive.gray.50}", "$type": "color" },
        "subtle": { "$value": "{color.primitive.gray.100}", "$type": "color" },
        "inverse": { "$value": "{color.primitive.gray.900}", "$type": "color" }
      },
      "foreground": {
        "default": { "$value": "{color.primitive.gray.900}", "$type": "color" },
        "muted": { "$value": "{color.primitive.gray.500}", "$type": "color" },
        "inverse": { "$value": "{color.primitive.gray.50}", "$type": "color" }
      },
      "border": {
        "default": { "$value": "{color.primitive.gray.200}", "$type": "color" },
        "strong": { "$value": "{color.primitive.gray.400}", "$type": "color" }
      },
      "interactive": {
        "default": { "$value": "{color.primitive.brand.primary}", "$type": "color" },
        "hover": { "$value": "{color.primitive.brand.primary}", "$type": "color", "$alpha": 0.9 },
        "active": { "$value": "{color.primitive.brand.primary}", "$type": "color", "$alpha": 0.8 }
      }
    }
  }
}
```

#### 2. Typography Tokens

```json
{
  "typography": {
    "fontFamily": {
      "heading": { "$value": "'Press Start 2P', monospace", "$type": "fontFamily" },
      "body": { "$value": "'VT323', monospace", "$type": "fontFamily" },
      "mono": { "$value": "'Fira Code', monospace", "$type": "fontFamily" }
    },
    "fontSize": {
      "xs": { "$value": "0.75rem", "$type": "dimension" },
      "sm": { "$value": "0.875rem", "$type": "dimension" },
      "base": { "$value": "1rem", "$type": "dimension" },
      "lg": { "$value": "1.125rem", "$type": "dimension" },
      "xl": { "$value": "1.25rem", "$type": "dimension" },
      "2xl": { "$value": "1.5rem", "$type": "dimension" },
      "3xl": { "$value": "1.875rem", "$type": "dimension" }
    },
    "fontWeight": {
      "normal": { "$value": "400", "$type": "fontWeight" },
      "medium": { "$value": "500", "$type": "fontWeight" },
      "bold": { "$value": "700", "$type": "fontWeight" }
    },
    "lineHeight": {
      "tight": { "$value": "1.25", "$type": "number" },
      "normal": { "$value": "1.5", "$type": "number" },
      "relaxed": { "$value": "1.75", "$type": "number" }
    },
    "letterSpacing": {
      "tight": { "$value": "-0.025em", "$type": "dimension" },
      "normal": { "$value": "0", "$type": "dimension" },
      "wide": { "$value": "0.05em", "$type": "dimension" }
    }
  }
}
```

#### 3. Spacing Tokens

```json
{
  "spacing": {
    "0": { "$value": "0", "$type": "dimension" },
    "1": { "$value": "0.25rem", "$type": "dimension" },
    "2": { "$value": "0.5rem", "$type": "dimension" },
    "3": { "$value": "0.75rem", "$type": "dimension" },
    "4": { "$value": "1rem", "$type": "dimension" },
    "6": { "$value": "1.5rem", "$type": "dimension" },
    "8": { "$value": "2rem", "$type": "dimension" },
    "12": { "$value": "3rem", "$type": "dimension" },
    "16": { "$value": "4rem", "$type": "dimension" }
  }
}
```

#### 4. Border & Radius Tokens

```json
{
  "border": {
    "width": {
      "none": { "$value": "0", "$type": "dimension" },
      "thin": { "$value": "1px", "$type": "dimension" },
      "medium": { "$value": "2px", "$type": "dimension" },
      "thick": { "$value": "4px", "$type": "dimension" }
    },
    "radius": {
      "none": { "$value": "0", "$type": "dimension" },
      "sm": { "$value": "0.125rem", "$type": "dimension" },
      "md": { "$value": "0.375rem", "$type": "dimension" },
      "lg": { "$value": "0.5rem", "$type": "dimension" },
      "full": { "$value": "9999px", "$type": "dimension" }
    }
  }
}
```

#### 5. Shadow Tokens

```json
{
  "shadow": {
    "none": { "$value": "none", "$type": "shadow" },
    "sm": {
      "$value": {
        "offsetX": "0",
        "offsetY": "1px",
        "blur": "2px",
        "spread": "0",
        "color": "rgba(0, 0, 0, 0.05)"
      },
      "$type": "shadow"
    },
    "md": {
      "$value": {
        "offsetX": "0",
        "offsetY": "4px",
        "blur": "6px",
        "spread": "-1px",
        "color": "rgba(0, 0, 0, 0.1)"
      },
      "$type": "shadow"
    },
    "lg": {
      "$value": {
        "offsetX": "0",
        "offsetY": "10px",
        "blur": "15px",
        "spread": "-3px",
        "color": "rgba(0, 0, 0, 0.1)"
      },
      "$type": "shadow"
    },
    "pixel": {
      "$value": {
        "offsetX": "4px",
        "offsetY": "4px",
        "blur": "0",
        "spread": "0",
        "color": "#000000"
      },
      "$type": "shadow",
      "$description": "Retro pixel shadow"
    }
  }
}
```

#### 6. Animation Tokens

```json
{
  "animation": {
    "duration": {
      "instant": { "$value": "0ms", "$type": "duration" },
      "fast": { "$value": "100ms", "$type": "duration" },
      "normal": { "$value": "200ms", "$type": "duration" },
      "slow": { "$value": "300ms", "$type": "duration" }
    },
    "easing": {
      "linear": { "$value": "linear", "$type": "cubicBezier" },
      "ease": { "$value": "ease", "$type": "cubicBezier" },
      "easeIn": { "$value": "cubic-bezier(0.4, 0, 1, 1)", "$type": "cubicBezier" },
      "easeOut": { "$value": "cubic-bezier(0, 0, 0.2, 1)", "$type": "cubicBezier" },
      "easeInOut": { "$value": "cubic-bezier(0.4, 0, 0.2, 1)", "$type": "cubicBezier" },
      "step": { "$value": "steps(4, end)", "$type": "cubicBezier", "$description": "Retro step animation" }
    }
  }
}
```

#### 7. Component Tokens

```json
{
  "component": {
    "button": {
      "paddingX": { "$value": "{spacing.4}", "$type": "dimension" },
      "paddingY": { "$value": "{spacing.2}", "$type": "dimension" },
      "borderRadius": { "$value": "{border.radius.md}", "$type": "dimension" },
      "fontSize": { "$value": "{typography.fontSize.sm}", "$type": "dimension" },
      "fontWeight": { "$value": "{typography.fontWeight.medium}", "$type": "fontWeight" },
      "primary": {
        "background": { "$value": "{color.semantic.interactive.default}", "$type": "color" },
        "foreground": { "$value": "{color.semantic.foreground.inverse}", "$type": "color" },
        "border": { "$value": "transparent", "$type": "color" }
      },
      "secondary": {
        "background": { "$value": "transparent", "$type": "color" },
        "foreground": { "$value": "{color.semantic.foreground.default}", "$type": "color" },
        "border": { "$value": "{color.semantic.border.default}", "$type": "color" }
      }
    },
    "input": {
      "paddingX": { "$value": "{spacing.3}", "$type": "dimension" },
      "paddingY": { "$value": "{spacing.2}", "$type": "dimension" },
      "borderRadius": { "$value": "{border.radius.md}", "$type": "dimension" },
      "borderWidth": { "$value": "{border.width.thin}", "$type": "dimension" },
      "borderColor": { "$value": "{color.semantic.border.default}", "$type": "color" },
      "background": { "$value": "{color.semantic.background.default}", "$type": "color" },
      "foreground": { "$value": "{color.semantic.foreground.default}", "$type": "color" },
      "placeholder": { "$value": "{color.semantic.foreground.muted}", "$type": "color" },
      "focus": {
        "borderColor": { "$value": "{color.semantic.interactive.default}", "$type": "color" },
        "ring": { "$value": "{color.semantic.interactive.default}", "$type": "color", "$alpha": 0.2 }
      }
    },
    "card": {
      "padding": { "$value": "{spacing.6}", "$type": "dimension" },
      "borderRadius": { "$value": "{border.radius.lg}", "$type": "dimension" },
      "background": { "$value": "{color.semantic.background.default}", "$type": "color" },
      "border": { "$value": "{color.semantic.border.default}", "$type": "color" },
      "shadow": { "$value": "{shadow.md}", "$type": "shadow" }
    }
  }
}
```

---

## Pink CSS Layer

Pink provides CSS utilities and component styles that consume TokiForge tokens.

### Base CSS Structure

```css
/* pink/base.css */

/* ═══════════════════════════════════════════════════════════════════════════
   PINK BASE - Theme Foundation
   ═══════════════════════════════════════════════════════════════════════════ */

/* Token-to-CSS-Variable mapping (generated by TokiForge) */
:root {
  /* Colors */
  --pink-color-bg-default: var(--tokiforge-color-semantic-background-default);
  --pink-color-bg-subtle: var(--tokiforge-color-semantic-background-subtle);
  --pink-color-fg-default: var(--tokiforge-color-semantic-foreground-default);
  --pink-color-fg-muted: var(--tokiforge-color-semantic-foreground-muted);
  --pink-color-border: var(--tokiforge-color-semantic-border-default);
  --pink-color-interactive: var(--tokiforge-color-semantic-interactive-default);
  
  /* Typography */
  --pink-font-heading: var(--tokiforge-typography-fontFamily-heading);
  --pink-font-body: var(--tokiforge-typography-fontFamily-body);
  --pink-font-mono: var(--tokiforge-typography-fontFamily-mono);
  
  /* Spacing */
  --pink-space-1: var(--tokiforge-spacing-1);
  --pink-space-2: var(--tokiforge-spacing-2);
  --pink-space-4: var(--tokiforge-spacing-4);
  --pink-space-6: var(--tokiforge-spacing-6);
  --pink-space-8: var(--tokiforge-spacing-8);
  
  /* Borders */
  --pink-radius-sm: var(--tokiforge-border-radius-sm);
  --pink-radius-md: var(--tokiforge-border-radius-md);
  --pink-radius-lg: var(--tokiforge-border-radius-lg);
  
  /* Shadows */
  --pink-shadow-sm: var(--tokiforge-shadow-sm);
  --pink-shadow-md: var(--tokiforge-shadow-md);
  --pink-shadow-lg: var(--tokiforge-shadow-lg);
  
  /* Animation */
  --pink-duration-fast: var(--tokiforge-animation-duration-fast);
  --pink-duration-normal: var(--tokiforge-animation-duration-normal);
  --pink-easing-default: var(--tokiforge-animation-easing-easeOut);
}

/* Dark mode */
[data-theme="dark"] {
  --pink-color-bg-default: var(--tokiforge-color-semantic-background-default-dark);
  --pink-color-bg-subtle: var(--tokiforge-color-semantic-background-subtle-dark);
  --pink-color-fg-default: var(--tokiforge-color-semantic-foreground-default-dark);
  --pink-color-fg-muted: var(--tokiforge-color-semantic-foreground-muted-dark);
  --pink-color-border: var(--tokiforge-color-semantic-border-default-dark);
}

/* Base reset */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-family: var(--pink-font-body);
  font-size: 16px;
  line-height: 1.5;
  color: var(--pink-color-fg-default);
  background-color: var(--pink-color-bg-default);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

### Component Styles

```css
/* pink/components.css */

/* ═══════════════════════════════════════════════════════════════════════════
   BUTTON
   ═══════════════════════════════════════════════════════════════════════════ */

.pink-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--pink-space-2);
  
  padding: var(--tokiforge-component-button-paddingY) var(--tokiforge-component-button-paddingX);
  border-radius: var(--tokiforge-component-button-borderRadius);
  
  font-family: var(--pink-font-body);
  font-size: var(--tokiforge-component-button-fontSize);
  font-weight: var(--tokiforge-component-button-fontWeight);
  line-height: 1;
  text-decoration: none;
  
  cursor: pointer;
  transition: all var(--pink-duration-fast) var(--pink-easing-default);
  
  /* Remove default button styles */
  border: none;
  outline: none;
  background: none;
}

.pink-button:focus-visible {
  outline: 2px solid var(--pink-color-interactive);
  outline-offset: 2px;
}

.pink-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Variants */
.pink-button--primary {
  background-color: var(--tokiforge-component-button-primary-background);
  color: var(--tokiforge-component-button-primary-foreground);
  border: 1px solid var(--tokiforge-component-button-primary-border);
}

.pink-button--primary:hover:not(:disabled) {
  filter: brightness(0.95);
}

.pink-button--primary:active:not(:disabled) {
  filter: brightness(0.9);
}

.pink-button--secondary {
  background-color: var(--tokiforge-component-button-secondary-background);
  color: var(--tokiforge-component-button-secondary-foreground);
  border: 1px solid var(--tokiforge-component-button-secondary-border);
}

.pink-button--secondary:hover:not(:disabled) {
  background-color: var(--pink-color-bg-subtle);
}

/* Sizes */
.pink-button--sm {
  padding: var(--pink-space-1) var(--pink-space-2);
  font-size: 0.75rem;
}

.pink-button--lg {
  padding: var(--pink-space-3) var(--pink-space-6);
  font-size: 1rem;
}

/* ═══════════════════════════════════════════════════════════════════════════
   INPUT
   ═══════════════════════════════════════════════════════════════════════════ */

.pink-input {
  display: block;
  width: 100%;
  
  padding: var(--tokiforge-component-input-paddingY) var(--tokiforge-component-input-paddingX);
  border-radius: var(--tokiforge-component-input-borderRadius);
  border: var(--tokiforge-component-input-borderWidth) solid var(--tokiforge-component-input-borderColor);
  
  font-family: var(--pink-font-body);
  font-size: 1rem;
  line-height: 1.5;
  
  color: var(--tokiforge-component-input-foreground);
  background-color: var(--tokiforge-component-input-background);
  
  transition: border-color var(--pink-duration-fast) var(--pink-easing-default),
              box-shadow var(--pink-duration-fast) var(--pink-easing-default);
}

.pink-input::placeholder {
  color: var(--tokiforge-component-input-placeholder);
}

.pink-input:focus {
  outline: none;
  border-color: var(--tokiforge-component-input-focus-borderColor);
  box-shadow: 0 0 0 3px var(--tokiforge-component-input-focus-ring);
}

.pink-input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* ═══════════════════════════════════════════════════════════════════════════
   CARD
   ═══════════════════════════════════════════════════════════════════════════ */

.pink-card {
  padding: var(--tokiforge-component-card-padding);
  border-radius: var(--tokiforge-component-card-borderRadius);
  border: 1px solid var(--tokiforge-component-card-border);
  background-color: var(--tokiforge-component-card-background);
  box-shadow: var(--tokiforge-component-card-shadow);
}
```

### Theme-Specific Overrides (Example: 8-Bit Retro)

```css
/* themes/retro-8bit/pink/base.css */

/* ═══════════════════════════════════════════════════════════════════════════
   RETRO 8-BIT THEME - Pink Overrides
   ═══════════════════════════════════════════════════════════════════════════ */

:root {
  /* Override border radius to 0 for pixel-perfect look */
  --tokiforge-border-radius-sm: 0;
  --tokiforge-border-radius-md: 0;
  --tokiforge-border-radius-lg: 0;
  
  /* Override shadows to pixel shadows */
  --tokiforge-shadow-sm: 2px 2px 0 0 #000;
  --tokiforge-shadow-md: 4px 4px 0 0 #000;
  --tokiforge-shadow-lg: 6px 6px 0 0 #000;
  
  /* Override animations to step-based */
  --tokiforge-animation-easing-easeOut: steps(4, end);
  --tokiforge-animation-easing-easeIn: steps(4, end);
  --tokiforge-animation-easing-easeInOut: steps(4, end);
  
  /* CRT scanline effect */
  --retro-scanline-opacity: 0.05;
  --retro-glow-color: rgba(0, 255, 0, 0.1);
}

/* Global CRT effect overlay */
html::after {
  content: "";
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  background: repeating-linear-gradient(
    0deg,
    rgba(0, 0, 0, var(--retro-scanline-opacity)),
    rgba(0, 0, 0, var(--retro-scanline-opacity)) 1px,
    transparent 1px,
    transparent 2px
  );
  z-index: 9999;
}

/* Pixel font rendering */
* {
  -webkit-font-smoothing: none;
  -moz-osx-font-smoothing: unset;
  image-rendering: pixelated;
}
```

```css
/* themes/retro-8bit/pink/components.css */

/* Button retro overrides */
.pink-button {
  text-transform: uppercase;
  letter-spacing: 0.1em;
  border: 4px solid;
  border-color: #fff #000 #000 #fff;
}

.pink-button:active:not(:disabled) {
  border-color: #000 #fff #fff #000;
  transform: translate(2px, 2px);
  box-shadow: none;
}

/* Input retro overrides */
.pink-input {
  border: 4px solid;
  border-color: #000 #fff #fff #000;
}

.pink-input:focus {
  box-shadow: 0 0 0 2px var(--retro-glow-color);
}

/* Card retro overrides */
.pink-card {
  border: 4px solid;
  border-color: #fff #000 #000 #fff;
}
```

---

## Theme Distribution

### Package Format

Themes are distributed as SQLar archives (`.sqlar`) via Depot:

```
retro-8bit.sqlar
├── theme.json
├── tokens/
├── pink/
├── assets/
└── preview/
```

### CLI Commands

```bash
# List available themes
punk themes list

# Search themes
punk themes search "retro"

# Preview theme in browser
punk themes preview @depot/retro-8bit

# Install theme
punk add @depot/theme-retro-8bit

# Set active theme
punk theme set retro-8bit

# Remove theme
punk remove @depot/theme-retro-8bit
```

### Depot Registry Entry

```json
{
  "$schema": "https://depot.punk.dev/schema/registry-item.json",
  "name": "theme-retro-8bit",
  "type": "registry:theme",
  "title": "Retro 8-Bit",
  "author": "PerceptLabs",
  "description": "Pixel-perfect nostalgia with CRT glow effects",
  "tier": "free",
  "version": "1.0.0",
  "tags": ["retro", "pixel", "gaming"],
  "downloads": 1542,
  "rating": 4.8,
  "files": [
    { "path": "themes/retro-8bit/theme.json", "type": "registry:theme-config" },
    { "path": "themes/retro-8bit/tokens/base.json", "type": "registry:tokens" },
    { "path": "themes/retro-8bit/tokens/semantic.json", "type": "registry:tokens" },
    { "path": "themes/retro-8bit/pink/base.css", "type": "registry:css" },
    { "path": "themes/retro-8bit/pink/components.css", "type": "registry:css" },
    { "path": "themes/retro-8bit/assets/fonts/PressStart2P.woff2", "type": "registry:asset" }
  ],
  "preview": {
    "thumbnail": "https://depot.punk.dev/previews/retro-8bit/thumb.png",
    "showcase": "https://depot.punk.dev/previews/retro-8bit/showcase.html"
  }
}
```

---

## Implementation Guide: Human Developers

### Creating a New Theme

#### Step 1: Scaffold

```bash
punk theme create my-theme
```

This creates:

```
themes/my-theme/
├── theme.json
├── tokens/
│   ├── base.json
│   └── semantic.json
├── pink/
│   ├── base.css
│   └── components.css
└── preview/
    └── showcase.html
```

#### Step 2: Define Tokens

Edit `tokens/base.json` with your primitive values:

```json
{
  "color": {
    "primitive": {
      "brand": {
        "primary": { "$value": "#your-color", "$type": "color" }
      }
    }
  }
}
```

#### Step 3: Create Semantic Mappings

Edit `tokens/semantic.json` to map primitives to semantic roles:

```json
{
  "color": {
    "semantic": {
      "interactive": {
        "default": { "$value": "{color.primitive.brand.primary}", "$type": "color" }
      }
    }
  }
}
```

#### Step 4: Add CSS Overrides

Edit `pink/components.css` for component-specific styling.

#### Step 5: Build & Preview

```bash
# Build tokens → CSS variables
punk theme build my-theme

# Preview in browser
punk theme preview my-theme
```

#### Step 6: Publish

```bash
# Package as .sqlar
punk theme package my-theme

# Publish to Depot
punk theme publish my-theme --tier free
```

### Theme Development Best Practices

1. **Start with tokens, not CSS** — Let TokiForge generate base CSS variables
2. **Use semantic tokens** — Don't reference primitives directly in components
3. **Test all components** — Use the component showcase to verify every Base UI element
4. **Support dark mode** — Provide `tokens/dark.json` with overrides
5. **Include preview assets** — Good thumbnails increase adoption
6. **Document customization points** — List which tokens users might want to tweak

---

## Implementation Guide: Claude Code

### Task: Create a New Theme

When asked to create a theme, follow this process:

#### 1. Understand Requirements

```
Extract from user request:
- Visual style (retro, modern, minimal, etc.)
- Color palette (specific colors or mood)
- Typography preferences
- Special effects (shadows, animations, etc.)
- Target use case (dashboard, marketing, app, etc.)
```

#### 2. Generate Token Structure

```typescript
// Generate base tokens
const baseTokens = {
  color: {
    primitive: generateColorPrimitives(palette),
  },
  typography: {
    fontFamily: selectFonts(style),
    fontSize: generateTypeScale(baseSize),
    // ...
  },
  spacing: generateSpacingScale(baseUnit),
  border: generateBorderTokens(style),
  shadow: generateShadowTokens(style),
  animation: generateAnimationTokens(style),
};

// Generate semantic tokens
const semanticTokens = {
  color: {
    semantic: mapColorsToSemantics(baseTokens.color.primitive),
  },
};

// Generate component tokens
const componentTokens = {
  component: generateComponentTokens(baseTokens, semanticTokens),
};
```

#### 3. Generate CSS

```typescript
// Generate Pink base CSS
const baseCss = generatePinkBase(semanticTokens);

// Generate component CSS
const componentCss = generateComponentStyles(componentTokens, style);

// Generate theme-specific effects
const effectsCss = generateEffects(style); // e.g., CRT lines, glassmorphism
```

#### 4. File Structure Output

Create files in this order:

```
1. theme.json (metadata)
2. tokens/base.json
3. tokens/semantic.json
4. tokens/components.json
5. tokens/dark.json (if dark mode requested)
6. pink/base.css
7. pink/components.css
8. pink/utilities.css (if needed)
```

#### 5. Validation Checklist

Before completing:

- [ ] All Base UI components have styles
- [ ] Color contrast meets WCAG AA (4.5:1 for text)
- [ ] Focus states are visible
- [ ] Dark mode tokens provided (or explicitly omitted)
- [ ] Fonts are available (Google Fonts, Fontsource, or included)
- [ ] theme.json is valid against schema
- [ ] Preview thumbnail described or generated

### Task: Modify Existing Theme

```typescript
// 1. Load existing theme
const theme = await loadTheme(themeName);

// 2. Parse modification request
const modifications = parseModificationRequest(userRequest);

// 3. Apply modifications
for (const mod of modifications) {
  switch (mod.type) {
    case 'color':
      theme.tokens.color = mergeDeep(theme.tokens.color, mod.value);
      break;
    case 'typography':
      theme.tokens.typography = mergeDeep(theme.tokens.typography, mod.value);
      break;
    case 'component':
      theme.tokens.component[mod.component] = mergeDeep(
        theme.tokens.component[mod.component],
        mod.value
      );
      break;
    case 'css':
      theme.css[mod.file] = applyPatch(theme.css[mod.file], mod.patch);
      break;
  }
}

// 4. Regenerate derived values
theme.css.base = regenerateBaseCss(theme.tokens);

// 5. Validate
await validateTheme(theme);

// 6. Output
await writeTheme(theme);
```

### Task: Apply Theme to Project

```typescript
// 1. Install theme files
async function applyTheme(themeName: string, projectRoot: string) {
  const theme = await depot.fetchTheme(themeName);
  
  // 2. Write token files
  await writeFile(
    join(projectRoot, 'tokens/theme.json'),
    JSON.stringify(theme.tokens, null, 2)
  );
  
  // 3. Write CSS files
  for (const [name, content] of Object.entries(theme.css)) {
    await writeFile(
      join(projectRoot, `styles/${name}`),
      content
    );
  }
  
  // 4. Copy assets
  for (const asset of theme.assets) {
    await copyFile(asset.src, join(projectRoot, asset.dest));
  }
  
  // 5. Update project config
  const config = await loadConfig(projectRoot);
  config.theme = themeName;
  config.tokiforge.input = 'tokens/theme.json';
  await writeConfig(projectRoot, config);
  
  // 6. Run TokiForge build
  await exec('punk build:tokens', { cwd: projectRoot });
}
```

### Code Generation Templates

#### Token File Template

```typescript
function generateTokenFile(category: string, values: Record<string, any>): string {
  return JSON.stringify({
    [category]: values
  }, null, 2);
}
```

#### CSS Template

```typescript
function generateCssFile(
  name: string,
  variables: Record<string, string>,
  rules: CssRule[]
): string {
  const header = `/* ${name} - Generated by Punk Theme System */\n\n`;
  
  const vars = Object.entries(variables)
    .map(([key, value]) => `  ${key}: ${value};`)
    .join('\n');
  
  const rootBlock = `:root {\n${vars}\n}\n\n`;
  
  const ruleBlocks = rules
    .map(rule => `${rule.selector} {\n${formatDeclarations(rule.declarations)}\n}`)
    .join('\n\n');
  
  return header + rootBlock + ruleBlocks;
}
```

---

## Theme Marketplace Integration

### Tier System

| Tier | Access | Features |
|------|--------|----------|
| **Free** | All users | Community themes, basic customization |
| **Pro** | Paid users | Premium themes, dark mode variants, priority support |
| **Enterprise** | Enterprise | Custom themes, white-labeling, source access |

### Submission Process

1. **Validate** — `punk theme validate my-theme`
2. **Package** — `punk theme package my-theme`
3. **Submit** — `punk theme submit my-theme --tier free`
4. **Review** — Depot team reviews for quality/security
5. **Publish** — Theme appears in marketplace

### Revenue Sharing

- Free themes: Attribution to author
- Pro themes: 70% to author, 30% to platform
- Enterprise themes: Custom agreements

---

## Examples

### Example 1: Glassmorphic Theme

```json
// tokens/base.json
{
  "color": {
    "primitive": {
      "glass": {
        "white": { "$value": "rgba(255, 255, 255, 0.2)", "$type": "color" },
        "blur": { "$value": "rgba(255, 255, 255, 0.1)", "$type": "color" }
      }
    }
  },
  "effect": {
    "blur": {
      "sm": { "$value": "4px", "$type": "dimension" },
      "md": { "$value": "8px", "$type": "dimension" },
      "lg": { "$value": "16px", "$type": "dimension" }
    }
  }
}
```

```css
/* pink/components.css */
.pink-card {
  background: var(--tokiforge-color-primitive-glass-white);
  backdrop-filter: blur(var(--tokiforge-effect-blur-md));
  border: 1px solid rgba(255, 255, 255, 0.3);
}
```

### Example 2: Brutalist Theme

```json
// tokens/base.json
{
  "border": {
    "width": {
      "brutal": { "$value": "4px", "$type": "dimension" }
    },
    "radius": {
      "none": { "$value": "0", "$type": "dimension" },
      "sm": { "$value": "0", "$type": "dimension" },
      "md": { "$value": "0", "$type": "dimension" },
      "lg": { "$value": "0", "$type": "dimension" }
    }
  },
  "shadow": {
    "brutal": {
      "$value": {
        "offsetX": "8px",
        "offsetY": "8px",
        "blur": "0",
        "spread": "0",
        "color": "#000000"
      },
      "$type": "shadow"
    }
  }
}
```

```css
/* pink/components.css */
.pink-button {
  border: var(--tokiforge-border-width-brutal) solid #000;
  box-shadow: var(--tokiforge-shadow-brutal);
  text-transform: uppercase;
}

.pink-button:hover {
  transform: translate(-2px, -2px);
  box-shadow: 10px 10px 0 0 #000;
}

.pink-button:active {
  transform: translate(4px, 4px);
  box-shadow: none;
}
```

### Example 3: Corporate Theme

```json
// tokens/base.json
{
  "color": {
    "primitive": {
      "brand": {
        "primary": { "$value": "#0066CC", "$type": "color" },
        "secondary": { "$value": "#004C99", "$type": "color" }
      },
      "neutral": {
        "50": { "$value": "#F8FAFC", "$type": "color" },
        "900": { "$value": "#0F172A", "$type": "color" }
      }
    }
  },
  "typography": {
    "fontFamily": {
      "heading": { "$value": "'Inter', -apple-system, sans-serif", "$type": "fontFamily" },
      "body": { "$value": "'Inter', -apple-system, sans-serif", "$type": "fontFamily" }
    }
  }
}
```

---

## Appendix A: Token Naming Conventions

| Category | Pattern | Example |
|----------|---------|---------|
| Primitive colors | `color.primitive.{palette}.{shade}` | `color.primitive.gray.500` |
| Semantic colors | `color.semantic.{role}.{variant}` | `color.semantic.foreground.muted` |
| Typography | `typography.{property}.{variant}` | `typography.fontSize.lg` |
| Spacing | `spacing.{scale}` | `spacing.4` |
| Border | `border.{property}.{variant}` | `border.radius.md` |
| Shadow | `shadow.{size}` | `shadow.lg` |
| Animation | `animation.{property}.{variant}` | `animation.duration.fast` |
| Component | `component.{name}.{property}` | `component.button.paddingX` |

## Appendix B: CSS Class Naming

Pink uses BEM-like naming:

```
.pink-{component}
.pink-{component}--{variant}
.pink-{component}--{size}
.pink-{component}__{element}
.pink-{component}__{element}--{modifier}
```

Examples:
- `.pink-button`
- `.pink-button--primary`
- `.pink-button--lg`
- `.pink-card__header`
- `.pink-card__header--sticky`

## Appendix C: File Hashes

For integrity verification, themes should include SHA-256 hashes:

```json
{
  "integrity": {
    "tokens/base.json": "sha256-abc123...",
    "pink/base.css": "sha256-def456...",
    "assets/fonts/PressStart2P.woff2": "sha256-ghi789..."
  }
}
```

---

## Changelog

- **v1.0.0** — Initial specification

---

*Document maintained by PerceptLabs. Last updated: 2024.*
