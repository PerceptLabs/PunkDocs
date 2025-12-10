# Punk Rigs Implementation Specification

**Version:** 2.0  
**Status:** Canonical Source of Truth  
**Replaces:** PUNK_GARDENJS_IMPL.md  
**Last Updated:** December 2025

---

## 1. Philosophy

**Punk Pragmatism:** The React ecosystem has thousands of well-typed components. Don't ask authors to rewrite what TypeScript already knows.

**Zero Authoring Goal:** Point at a React component, get a Rig.

```bash
punk wrap react-chartjs-2 --component Chart --tier pro
```

---

## 2. What is a Rig?

A Rig is a React component packaged for Mohawk with:

| Artifact | Purpose | Generated From |
|----------|---------|----------------|
| Puck config | Visual editor fields | TypeScript types |
| Effect schema | AI validation + repair | TypeScript types |
| Knowledge | AI discoverability | LLM analysis |
| Tier metadata | Access control | CLI flags |

---

## 3. Tier Model

### 3.1 Tier Definitions

| Tier | Access | Use Case |
|------|--------|----------|
| `free` | All users | Core primitives (Button, Text, Container) |
| `starter` | Starter+ subscribers | Enhanced components (Form, Tabs, Modal) |
| `pro` | Pro+ subscribers | Advanced components (DataTable, RichText) |
| `purchasable` | À la carte ($X) | Buy individually without upgrading tier |
| `premium` | Separate purchase | High-value components regardless of tier |

### 3.2 Tier Hierarchy

```
free → starter → pro → enterprise
                  ↑
            purchasable (skip the line)
                  
premium = always separate purchase
```

### 3.3 Examples

| Rig | Tier | What It Means |
|-----|------|---------------|
| Button | `free` | Everyone gets it |
| Tabs | `starter` | Need Starter subscription |
| DataTable | `pro` | Need Pro subscription |
| DataTable | `purchasable: $19` | OR buy it without Pro |
| Kanban | `premium: $39` | Must purchase, even if Pro |

### 3.4 Tier Specification

```bash
# Free - all users
punk wrap lucide-react --component Icon --tier free

# Starter tier
punk wrap @base-ui/react-tabs --tier starter

# Pro tier
punk wrap @tanstack/react-table --tier pro

# Purchasable - buy instead of upgrading
punk wrap @tanstack/react-table --tier pro --purchasable 1900

# Premium - always separate purchase
punk wrap react-kanban --tier premium --price 3900
```

---

## 4. The `punk wrap` CLI

### 4.1 Basic Usage

```bash
punk wrap <package> [options]

Arguments:
  package                   npm package to wrap

Options:
  --component <name>        Component to export (default: main export)
  --name <name>             Rig name (default: component name)
  --tier <tier>             free | starter | pro | premium
  --purchasable <cents>     Allow à la carte purchase (price in cents)
  --price <cents>           Premium price in cents (requires --tier premium)
  --category <category>     Component category
  --mapper <path>           Custom prop mapper file
  --output <dir>            Output directory (default: ./rigs/<name>)
  --dry-run                 Show what would be generated
```

### 4.2 Examples

```bash
# Simple - zero config
punk wrap lucide-react --component Icon --tier free

# With category
punk wrap react-chartjs-2 --component Chart --tier pro --category "Data Display"

# Purchasable alternative to tier
punk wrap @tanstack/react-table --component DataTable \
  --tier pro --purchasable 1900

# Premium with price
punk wrap react-kanban --tier premium --price 3900

# Custom prop transformation
punk wrap react-chartjs-2 --component Chart --tier pro \
  --mapper ./mappers/chart.ts
```

---

## 5. Type Extraction

### 5.1 What Gets Extracted

| TypeScript | Punk Prop | Puck Field | Effect Schema |
|------------|-----------|------------|---------------|
| `string` | `string` | `text` | `Schema.String` |
| `number` | `number` | `number` | `Schema.Number` |
| `boolean` | `boolean` | `radio` | `Schema.Boolean` |
| `'a' \| 'b' \| 'c'` | `enum` | `select` | `Schema.Literal` |
| `Array<T>` | `array` | `array` | `Schema.Array` |
| `{ a: T, b: U }` | `object` | `object` | `Schema.Struct` |
| `T \| undefined` | optional | optional | `Schema.optional` |
| `React.ReactNode` | `slot` | `slot` | `Schema.Unknown` |

### 5.2 Extraction Sources

```typescript
// 1. Explicit prop types
interface ChartProps {
  type: 'bar' | 'line' | 'pie';  // → enum
  data: DataPoint[];              // → array
  showLegend?: boolean;           // → optional boolean
}

// 2. Default values
Chart.defaultProps = {
  type: 'bar',      // → default: 'bar'
  showLegend: true, // → default: true
};

// 3. JSDoc comments
interface ChartProps {
  /** The type of chart to render */
  type: 'bar' | 'line' | 'pie';  // → hint: "The type of chart to render"
}

// 4. Generic constraints
type Props<T extends object> = {
  data: T[];  // → array with inferred item shape
};
```

### 5.3 Complex Type Handling

```typescript
// Nested objects
interface CardProps {
  header: {
    title: string;
    subtitle?: string;
  };
}
// → object with nested fields

// Union types (non-literal)
type Variant = 'primary' | 'secondary' | CustomVariant;
// → enum with known literals, extensible

// Function props (callbacks)
interface ButtonProps {
  onClick?: (event: MouseEvent) => void;
}
// → excluded from Puck fields (runtime only)

// Ref props
interface InputProps {
  ref?: React.Ref<HTMLInputElement>;
}
// → excluded from Puck fields
```

---

## 6. LLM Enrichment

### 6.1 When It Runs

```
punk wrap executes
       ↓
  TypeScript analysis complete
       ↓
  LLM enrichment (automatic)
       ↓
  knowledge.json written
```

### 6.2 What LLM Generates

```typescript
interface GeneratedKnowledge {
  description: string;      // What the component does
  useWhen: string[];        // Semantic triggers (3-5)
  doNotUseWhen: string[];   // Anti-patterns (2-3)
  examples: PromptExample[];// Prompt → props mappings (3-5)
  relatedRigs: string[];    // Suggestions for similar needs
}
```

### 6.3 LLM Prompt Template

```typescript
const enrichmentPrompt = `
Analyze this React component and generate AI knowledge metadata.

Component: ${name}
Package: ${packageName}
Category: ${category}
Props:
${JSON.stringify(extractedProps, null, 2)}

Based on the component name, package, and prop structure, generate:

1. description: One sentence explaining what this component does
2. useWhen: 3-5 phrases describing when a user would want this
3. doNotUseWhen: 2-3 anti-patterns where this is wrong choice
4. examples: 3-5 natural language prompts with corresponding prop values
5. relatedRigs: Other component types that serve similar purposes

Return valid JSON matching this schema:
${JSON.stringify(KnowledgeSchema)}
`;
```

### 6.4 Generated Output

```json
{
  "description": "Displays data as bar, line, or pie charts for visual data analysis",
  "useWhen": [
    "user wants to visualize numeric data",
    "user mentions chart, graph, or plot",
    "user needs to show trends over time",
    "user wants to compare values across categories"
  ],
  "doNotUseWhen": [
    "data is not numeric",
    "user wants raw tabular data",
    "single value display"
  ],
  "examples": [
    { "prompt": "show sales by month", "props": { "type": "bar" } },
    { "prompt": "visualize the trend over time", "props": { "type": "line" } },
    { "prompt": "show market share breakdown", "props": { "type": "pie" } },
    { "prompt": "compare Q1 vs Q2 revenue", "props": { "type": "bar" } }
  ],
  "relatedRigs": ["DataTable", "Metric", "Sparkline"]
}
```

### 6.5 Caching & Regeneration

```bash
# Knowledge cached by package@version + component
# Stored in: .punk/knowledge-cache/

# Force regeneration
punk wrap react-chartjs-2 --component Chart --regenerate-knowledge

# Skip LLM (use cached or empty)
punk wrap react-chartjs-2 --component Chart --no-enrich
```

---

## 7. Prop Mappers

### 7.1 When Needed

Most components work directly. Mappers needed when:

1. **Library props are unintuitive** (nested structures, weird names)
2. **You want simpler UX** (flatten complex APIs)
3. **Props need transformation** (format conversion)

### 7.2 Mapper File Structure

```typescript
// mappers/chart.ts
import type { PropMapper } from '@punk/rigs';

export const mapper: PropMapper = {
  // Reshape props for better authoring UX
  // (optional - only if library props are awkward)
  reshapeProps: {
    chartType: {
      type: 'enum',
      options: ['bar', 'line', 'pie'],
      default: 'bar',
    },
    data: {
      type: 'array',
      items: {
        label: { type: 'string' },
        value: { type: 'number' },
      },
    },
    showLegend: {
      type: 'boolean',
      default: true,
    },
  },

  // Transform Punk props → library props
  mapProps: (punk) => ({
    type: punk.chartType,
    data: {
      labels: punk.data.map(d => d.label),
      datasets: [{
        data: punk.data.map(d => d.value),
      }],
    },
    options: {
      plugins: {
        legend: { display: punk.showLegend },
      },
    },
  }),
};
```

### 7.3 When Library Props Are Fine

No mapper needed — types pass through directly:

```bash
# Radix Accordion has clean props, no mapper
punk wrap @radix-ui/react-accordion --tier free

# Lucide icons - just name prop
punk wrap lucide-react --component Icon --tier free
```

---

## 8. Output Structure

### 8.1 Generated Files

```
rigs/chart/
├── index.ts          # Re-exports everything
├── rig.ts            # Generated Rig definition
├── rig.schema.ts     # Effect schema
├── rig.puck.ts       # Puck config
├── knowledge.json    # LLM-generated knowledge
└── manifest.json     # Package metadata
```

### 8.2 Generated Rig Definition

```typescript
// rigs/chart/rig.ts (auto-generated)
import { Chart } from 'react-chartjs-2';
import { schema } from './rig.schema';
import { puckConfig } from './rig.puck';
import knowledge from './knowledge.json';
import { mapper } from './mapper';  // if provided

export const ChartRig = {
  name: 'Chart',
  category: 'Data Display',
  tier: 'pro',
  purchasable: 1900,  // if specified
  
  component: Chart,
  schema,
  puckConfig,
  knowledge,
  
  mapProps: mapper?.mapProps,
};
```

### 8.3 Manifest

```json
{
  "name": "chart",
  "displayName": "Chart",
  "version": "1.0.0",
  "source": "react-chartjs-2@^5.0.0",
  "component": "Chart",
  "category": "Data Display",
  
  "tier": "pro",
  "purchasable": 1900,
  
  "files": {
    "entry": "./index.ts",
    "schema": "./rig.schema.ts",
    "puck": "./rig.puck.ts",
    "knowledge": "./knowledge.json"
  },
  
  "dependencies": {
    "react-chartjs-2": "^5.0.0",
    "chart.js": "^4.0.0"
  },
  
  "generatedAt": "2025-12-08T00:00:00Z",
  "punkVersion": "1.0.0"
}
```

---

## 9. Registry & Runtime

### 9.1 Registration

```typescript
// rigs/index.ts
import { registerRig } from '@punk/rigs';

// Auto-import all rigs in directory
const rigModules = import.meta.glob('./*/rig.ts', { eager: true });

for (const [path, module] of Object.entries(rigModules)) {
  registerRig(module.default);
}
```

### 9.2 Registry API

```typescript
import { rigRegistry } from '@punk/rigs';

// Get rig by name
const chart = rigRegistry.get('Chart');

// List by criteria
const dataRigs = rigRegistry.list({ 
  category: 'Data Display',
  maxTier: 'pro',
});

// Get Puck config for user's access level
const puckComponents = rigRegistry.getPuckConfig({
  userTier: 'starter',
  purchases: ['chart'],  // Bought Chart à la carte
});

// Get knowledge for Ska
const knowledge = rigRegistry.getKnowledge({
  userTier: 'pro',
  purchases: ['kanban'],
});
```

### 9.3 Access Resolution

```typescript
function canAccess(rig: Rig, user: User): boolean {
  // Premium always requires purchase
  if (rig.tier === 'premium') {
    return user.purchases.includes(rig.name);
  }
  
  // Check if purchased à la carte
  if (user.purchases.includes(rig.name)) {
    return true;
  }
  
  // Check tier hierarchy
  const tierRank = { free: 0, starter: 1, pro: 2, enterprise: 3 };
  return tierRank[user.tier] >= tierRank[rig.tier];
}
```

---

## 10. Ska Integration

### 10.1 Knowledge Context

```typescript
function buildSkaContext(user: User): string {
  const available = rigRegistry.getKnowledge({
    userTier: user.tier,
    purchases: user.purchases,
  });
  
  const upgradeable = rigRegistry.getUpgradeOptions(user.tier);
  
  return `
## Available Components

${available.map(rig => `
### ${rig.name}
${rig.knowledge.description}

**Use when:** ${rig.knowledge.useWhen.join(', ')}
**Props:** ${Object.keys(rig.props).join(', ')}
`).join('\n')}

## Upgrade Options

${upgradeable.map(rig => `
- **${rig.name}** (${rig.tier}${rig.purchasable ? ` or $${rig.purchasable/100}` : ''}): ${rig.knowledge.description}
`).join('\n')}
`;
}
```

### 10.2 Upgrade Suggestions

When Ska determines a better Rig exists:

```typescript
interface SkaSuggestion {
  schema: PunkDocument;
  upgradeSuggestions?: Array<{
    current: string;          // What we used
    suggested: string;        // What would be better
    reason: string;           // Why it's better
    access: {
      tier?: Tier;            // Required tier
      purchasable?: number;   // À la carte price
      premium?: number;       // Premium price
    };
  }>;
}

// Example output
{
  schema: { /* uses Text rig */ },
  upgradeSuggestions: [{
    current: 'Text',
    suggested: 'RichText',
    reason: 'Rich text editing with formatting, mentions, and embeds',
    access: {
      tier: 'pro',
      purchasable: 1900,
    },
  }],
}
```

---

## 11. Depot Distribution

### 11.1 Publishing

```bash
# Package and publish to Depot
punk publish ./rigs/chart --version 1.0.0

# Outputs: chart-1.0.0.sqlar
```

### 11.2 SQLar Contents

```
chart-1.0.0.sqlar
├── manifest.json       # Metadata + tier + pricing
├── dist/
│   ├── index.js        # Bundled component
│   ├── rig.schema.js   # Effect schema
│   └── rig.puck.js     # Puck config
├── knowledge.json      # AI knowledge
└── preview.png         # Thumbnail for Depot UI
```

### 11.3 Installation

```bash
# From Depot (checks access)
punk install @depot/chart

# Verifies: user.tier >= rig.tier OR rig in user.purchases
```

---

## 12. Built-in Rigs

### 12.1 Free Tier

| Rig | Package | Description |
|-----|---------|-------------|
| Button | `@punk/primitives` | Clickable button |
| Text | `@punk/primitives` | Typography |
| Image | `@punk/primitives` | Responsive image |
| Container | `@punk/primitives` | Flex/grid container |
| Stack | `@punk/primitives` | Vertical/horizontal stack |
| Card | `@punk/primitives` | Content card |
| Link | `@punk/primitives` | Navigation link |
| Icon | `lucide-react` | Icon library |
| Divider | `@punk/primitives` | Horizontal rule |
| Spacer | `@punk/primitives` | Vertical spacing |

### 12.2 Starter Tier

| Rig | Package | Description |
|-----|---------|-------------|
| Form | `@base-ui/react` | Form with validation |
| Tabs | `@base-ui/react` | Tabbed content |
| Accordion | `@base-ui/react` | Collapsible sections |
| Modal | `@base-ui/react` | Dialog/modal |
| Menu | `@base-ui/react` | Dropdown menu |
| Tooltip | `@base-ui/react` | Hover tooltip |
| Badge | `@punk/primitives` | Status badges |
| Avatar | `@punk/primitives` | User avatars |

### 12.3 Pro Tier

| Rig | Package | Purchasable |
|-----|---------|-------------|
| Chart | `react-chartjs-2` | $19 |
| DataTable | `@tanstack/react-table` | $19 |
| RichText | `@lexical/react` | $29 |
| DatePicker | `@base-ui/react` | $9 |
| Select | `@base-ui/react` | — |
| Combobox | `@base-ui/react` | — |

### 12.4 Premium (Always Separate)

| Rig | Package | Price |
|-----|---------|-------|
| Kanban | `react-kanban` | $39 |
| Calendar | `@fullcalendar/react` | $39 |
| Spreadsheet | `@handsontable/react` | $49 |
| CodeEditor | `@monaco-editor/react` | $29 |
| MapView | `react-map-gl` | $49 |
| FileUpload | `react-dropzone` | $19 |

---

## 13. CLI Reference

### 13.1 Commands

```bash
# Wrap a component
punk wrap <package> [options]

# List registered rigs
punk rigs list [--tier <tier>] [--category <cat>]

# Show rig details
punk rigs info <name>

# Regenerate knowledge
punk rigs enrich <name> [--force]

# Validate rig
punk rigs validate <path>

# Publish to Depot
punk publish <path> --version <ver>

# Install from Depot
punk install <name>
```

### 13.2 Config File

```yaml
# punk.config.yaml
rigs:
  output: ./src/rigs
  enrichment:
    enabled: true
    model: ska-2-128k-instruct  # LLM for enrichment
    cache: .punk/knowledge-cache
  
  defaults:
    tier: free
    category: General

depot:
  registry: https://depot.punk.dev
```

---

## 14. Implementation: Type Walker

### 14.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      punk wrap CLI                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    PackageResolver                          │
│  - Resolves npm package to local node_modules path          │
│  - Finds package.json, types entry point                    │
│  - Handles @types/* fallback                                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     TypeExtractor                           │
│  - Uses TypeScript Compiler API                             │
│  - Walks component props interface                          │
│  - Extracts types, defaults, JSDoc                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    SchemaGenerator                          │
│  - Converts extracted types → Punk prop definitions         │
│  - Generates Effect schema code                             │
│  - Generates Puck config code                               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   KnowledgeEnricher                         │
│  - Sends prop structure to LLM                              │
│  - Receives knowledge.json                                  │
│  - Caches by package@version                                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     RigEmitter                              │
│  - Writes all generated files                               │
│  - Creates manifest.json                                    │
│  - Validates output                                         │
└─────────────────────────────────────────────────────────────┘
```

### 14.2 PackageResolver

```typescript
// src/wrap/package-resolver.ts
import { resolve, join } from 'path';
import { existsSync, readFileSync } from 'fs';

interface ResolvedPackage {
  name: string;
  version: string;
  packagePath: string;
  typesEntry: string | null;
  mainEntry: string | null;
  dependencies: Record<string, string>;
}

export class PackageResolver {
  private nodeModulesPath: string;

  constructor(cwd: string = process.cwd()) {
    this.nodeModulesPath = join(cwd, 'node_modules');
  }

  resolve(packageName: string): ResolvedPackage {
    // Handle scoped packages (@org/package)
    const packagePath = join(this.nodeModulesPath, packageName);
    
    if (!existsSync(packagePath)) {
      throw new PackageNotFoundError(packageName);
    }

    const pkgJson = JSON.parse(
      readFileSync(join(packagePath, 'package.json'), 'utf-8')
    );

    // Find types entry point
    const typesEntry = this.resolveTypes(packagePath, pkgJson, packageName);

    return {
      name: pkgJson.name,
      version: pkgJson.version,
      packagePath,
      typesEntry,
      mainEntry: pkgJson.main || pkgJson.module || null,
      dependencies: pkgJson.dependencies || {},
    };
  }

  private resolveTypes(
    packagePath: string,
    pkgJson: any,
    packageName: string
  ): string | null {
    // 1. Check package.json "types" or "typings" field
    if (pkgJson.types) {
      return join(packagePath, pkgJson.types);
    }
    if (pkgJson.typings) {
      return join(packagePath, pkgJson.typings);
    }

    // 2. Check for index.d.ts
    const indexDts = join(packagePath, 'index.d.ts');
    if (existsSync(indexDts)) {
      return indexDts;
    }

    // 3. Check @types/* package
    const typesPackage = packageName.startsWith('@')
      ? `@types/${packageName.slice(1).replace('/', '__')}`
      : `@types/${packageName}`;
    
    const typesPackagePath = join(this.nodeModulesPath, typesPackage);
    if (existsSync(typesPackagePath)) {
      const typesPkgJson = JSON.parse(
        readFileSync(join(typesPackagePath, 'package.json'), 'utf-8')
      );
      return join(typesPackagePath, typesPkgJson.types || 'index.d.ts');
    }

    return null;
  }
}

export class PackageNotFoundError extends Error {
  constructor(packageName: string) {
    super(`Package "${packageName}" not found in node_modules. Run: npm install ${packageName}`);
    this.name = 'PackageNotFoundError';
  }
}
```

### 14.3 TypeExtractor

```typescript
// src/wrap/type-extractor.ts
import ts from 'typescript';
import { resolve } from 'path';

export interface ExtractedProp {
  name: string;
  type: PunkPropType;
  required: boolean;
  default?: unknown;
  description?: string;
  options?: string[];          // For enums
  items?: ExtractedProp[];     // For arrays
  properties?: ExtractedProp[];// For objects
}

export type PunkPropType = 
  | 'string' 
  | 'number' 
  | 'boolean' 
  | 'enum' 
  | 'array' 
  | 'object' 
  | 'slot' 
  | 'unknown';

export interface ExtractionResult {
  componentName: string;
  props: ExtractedProp[];
  defaultProps: Record<string, unknown>;
  warnings: string[];
}

export class TypeExtractor {
  private program: ts.Program;
  private checker: ts.TypeChecker;
  private warnings: string[] = [];

  constructor(typesEntry: string) {
    this.program = ts.createProgram([typesEntry], {
      target: ts.ScriptTarget.ESNext,
      module: ts.ModuleKind.ESNext,
      moduleResolution: ts.ModuleResolutionKind.NodeNext,
      allowSyntheticDefaultImports: true,
      esModuleInterop: true,
    });
    this.checker = this.program.getTypeChecker();
  }

  extract(componentName: string): ExtractionResult {
    this.warnings = [];
    
    // Find the component export
    const sourceFile = this.program.getSourceFiles()
      .find(sf => !sf.isDeclarationFile || sf.fileName.endsWith('.d.ts'));
    
    if (!sourceFile) {
      throw new ExtractionError('No source file found');
    }

    const componentSymbol = this.findComponentExport(sourceFile, componentName);
    if (!componentSymbol) {
      throw new ExtractionError(`Component "${componentName}" not found in exports`);
    }

    // Get the props type
    const propsType = this.getPropsType(componentSymbol);
    if (!propsType) {
      throw new ExtractionError(`Could not determine props type for "${componentName}"`);
    }

    // Extract prop definitions
    const props = this.extractPropsFromType(propsType);
    
    // Extract default props
    const defaultProps = this.extractDefaultProps(componentSymbol);

    return {
      componentName,
      props,
      defaultProps,
      warnings: this.warnings,
    };
  }

  private findComponentExport(
    sourceFile: ts.SourceFile, 
    componentName: string
  ): ts.Symbol | undefined {
    const exports = this.checker.getExportsOfModule(
      this.checker.getSymbolAtLocation(sourceFile)!
    );
    
    // Try exact match first
    let symbol = exports.find(e => e.name === componentName);
    
    // Try default export
    if (!symbol && componentName === 'default') {
      symbol = exports.find(e => e.name === 'default');
    }
    
    return symbol;
  }

  private getPropsType(componentSymbol: ts.Symbol): ts.Type | null {
    const type = this.checker.getTypeOfSymbolAtLocation(
      componentSymbol,
      componentSymbol.valueDeclaration!
    );

    // Handle React.FC<Props>, React.Component<Props>, (props: Props) => JSX
    const callSignatures = type.getCallSignatures();
    if (callSignatures.length > 0) {
      const firstParam = callSignatures[0].getParameters()[0];
      if (firstParam) {
        return this.checker.getTypeOfSymbolAtLocation(
          firstParam,
          firstParam.valueDeclaration!
        );
      }
    }

    // Handle class components
    const constructSignatures = type.getConstructSignatures();
    if (constructSignatures.length > 0) {
      // Look for props in type arguments
      const typeArgs = (type as ts.TypeReference).typeArguments;
      if (typeArgs && typeArgs.length > 0) {
        return typeArgs[0];
      }
    }

    return null;
  }

  private extractPropsFromType(type: ts.Type): ExtractedProp[] {
    const props: ExtractedProp[] = [];
    
    for (const prop of type.getProperties()) {
      // Skip internal React props
      if (this.isInternalProp(prop.name)) {
        continue;
      }

      const propType = this.checker.getTypeOfSymbolAtLocation(
        prop,
        prop.valueDeclaration!
      );

      const extracted = this.extractSingleProp(prop, propType);
      if (extracted) {
        props.push(extracted);
      }
    }

    return props;
  }

  private extractSingleProp(
    symbol: ts.Symbol, 
    type: ts.Type
  ): ExtractedProp | null {
    const name = symbol.name;
    const required = !this.isOptionalSymbol(symbol);
    const description = this.getJSDocComment(symbol);

    // Unwrap optional type
    const unwrapped = this.unwrapOptional(type);

    // Determine Punk type
    const punkType = this.toPunkType(unwrapped);

    if (punkType === 'unknown') {
      this.warnings.push(`Skipping prop "${name}": unsupported type`);
      return null;
    }

    const prop: ExtractedProp = {
      name,
      type: punkType,
      required,
      description,
    };

    // Handle enum options
    if (punkType === 'enum') {
      prop.options = this.extractEnumOptions(unwrapped);
    }

    // Handle array items
    if (punkType === 'array') {
      const itemType = this.getArrayItemType(unwrapped);
      if (itemType) {
        prop.items = this.extractPropsFromType(itemType);
      }
    }

    // Handle object properties
    if (punkType === 'object') {
      prop.properties = this.extractPropsFromType(unwrapped);
    }

    return prop;
  }

  private toPunkType(type: ts.Type): PunkPropType {
    const typeString = this.checker.typeToString(type);

    // Primitives
    if (type.flags & ts.TypeFlags.String) return 'string';
    if (type.flags & ts.TypeFlags.Number) return 'number';
    if (type.flags & ts.TypeFlags.Boolean) return 'boolean';
    if (type.flags & ts.TypeFlags.BooleanLiteral) return 'boolean';

    // String/number literals (part of union = enum)
    if (type.flags & ts.TypeFlags.StringLiteral) return 'enum';
    if (type.flags & ts.TypeFlags.NumberLiteral) return 'enum';

    // Union of literals = enum
    if (type.isUnion()) {
      const allLiterals = type.types.every(t => 
        t.flags & ts.TypeFlags.StringLiteral ||
        t.flags & ts.TypeFlags.NumberLiteral
      );
      if (allLiterals) return 'enum';
    }

    // Array
    if (this.checker.isArrayType(type)) return 'array';
    if (typeString.endsWith('[]')) return 'array';

    // React.ReactNode = slot
    if (typeString.includes('ReactNode') || typeString.includes('ReactElement')) {
      return 'slot';
    }

    // Function = skip (callbacks)
    if (type.getCallSignatures().length > 0) return 'unknown';

    // Object
    if (type.flags & ts.TypeFlags.Object) {
      // Skip React-specific types
      if (typeString.includes('Ref<')) return 'unknown';
      if (typeString.includes('CSSProperties')) return 'object';
      return 'object';
    }

    return 'unknown';
  }

  private extractEnumOptions(type: ts.Type): string[] {
    if (type.isUnion()) {
      return type.types
        .map(t => {
          if (t.isStringLiteral()) return t.value;
          if (t.isNumberLiteral()) return String(t.value);
          return null;
        })
        .filter((v): v is string => v !== null);
    }
    
    if (type.isStringLiteral()) {
      return [type.value];
    }

    return [];
  }

  private getArrayItemType(type: ts.Type): ts.Type | null {
    const typeArgs = (type as ts.TypeReference).typeArguments;
    if (typeArgs && typeArgs.length > 0) {
      return typeArgs[0];
    }
    return null;
  }

  private isOptionalSymbol(symbol: ts.Symbol): boolean {
    return (symbol.flags & ts.SymbolFlags.Optional) !== 0;
  }

  private unwrapOptional(type: ts.Type): ts.Type {
    if (type.isUnion()) {
      const nonUndefined = type.types.filter(t => 
        !(t.flags & ts.TypeFlags.Undefined) &&
        !(t.flags & ts.TypeFlags.Null)
      );
      if (nonUndefined.length === 1) {
        return nonUndefined[0];
      }
    }
    return type;
  }

  private getJSDocComment(symbol: ts.Symbol): string | undefined {
    const docs = symbol.getDocumentationComment(this.checker);
    if (docs.length > 0) {
      return docs.map(d => d.text).join('\n');
    }
    return undefined;
  }

  private extractDefaultProps(componentSymbol: ts.Symbol): Record<string, unknown> {
    // Look for .defaultProps on the component
    const type = this.checker.getTypeOfSymbolAtLocation(
      componentSymbol,
      componentSymbol.valueDeclaration!
    );

    const defaultPropsMember = type.getProperty('defaultProps');
    if (!defaultPropsMember) return {};

    // Try to extract literal values
    // This is limited - works for simple cases
    const defaults: Record<string, unknown> = {};
    
    try {
      const defaultPropsType = this.checker.getTypeOfSymbolAtLocation(
        defaultPropsMember,
        defaultPropsMember.valueDeclaration!
      );

      for (const prop of defaultPropsType.getProperties()) {
        const propType = this.checker.getTypeOfSymbolAtLocation(
          prop,
          prop.valueDeclaration!
        );
        
        if (propType.isStringLiteral()) {
          defaults[prop.name] = propType.value;
        } else if (propType.isNumberLiteral()) {
          defaults[prop.name] = propType.value;
        } else if (propType.flags & ts.TypeFlags.BooleanLiteral) {
          defaults[prop.name] = (propType as any).intrinsicName === 'true';
        }
      }
    } catch {
      this.warnings.push('Could not extract defaultProps values');
    }

    return defaults;
  }

  private isInternalProp(name: string): boolean {
    const internal = [
      'children',  // Handled as slot separately
      'key',
      'ref',
      'className',
      'style',
      'dangerouslySetInnerHTML',
      'suppressContentEditableWarning',
      'suppressHydrationWarning',
    ];
    return internal.includes(name) || name.startsWith('aria-') || name.startsWith('data-');
  }
}

export class ExtractionError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ExtractionError';
  }
}
```

### 14.4 SchemaGenerator

```typescript
// src/wrap/schema-generator.ts
import { ExtractedProp, PunkPropType } from './type-extractor';

export interface GeneratedSchemas {
  effectSchema: string;   // TypeScript code
  puckConfig: string;     // TypeScript code
  typeDefinition: string; // TypeScript interface
}

export class SchemaGenerator {
  generate(
    componentName: string,
    props: ExtractedProp[],
    defaults: Record<string, unknown>
  ): GeneratedSchemas {
    return {
      effectSchema: this.generateEffectSchema(componentName, props),
      puckConfig: this.generatePuckConfig(componentName, props, defaults),
      typeDefinition: this.generateTypeDefinition(componentName, props),
    };
  }

  private generateEffectSchema(name: string, props: ExtractedProp[]): string {
    const schemaFields = props.map(p => this.propToEffectField(p)).join(',\n    ');

    return `// Auto-generated Effect schema for ${name}
import { Schema } from '@effect/schema';

export const ${name}PropsSchema = Schema.Struct({
    ${schemaFields}
});

export type ${name}Props = Schema.Schema.Type<typeof ${name}PropsSchema>;
`;
  }

  private propToEffectField(prop: ExtractedProp, indent = 4): string {
    const spaces = ' '.repeat(indent);
    let schema: string;

    switch (prop.type) {
      case 'string':
        schema = 'Schema.String';
        break;
      
      case 'number':
        schema = 'Schema.Number';
        break;
      
      case 'boolean':
        schema = 'Schema.Boolean';
        break;
      
      case 'enum':
        const literals = prop.options!.map(o => `'${o}'`).join(', ');
        schema = `Schema.Literal(${literals})`;
        break;
      
      case 'array':
        if (prop.items && prop.items.length > 0) {
          const itemFields = prop.items
            .map(i => this.propToEffectField(i, indent + 2))
            .join(',\n');
          schema = `Schema.Array(Schema.Struct({\n${itemFields}\n${spaces}}))`;
        } else {
          schema = 'Schema.Array(Schema.Unknown)';
        }
        break;
      
      case 'object':
        if (prop.properties && prop.properties.length > 0) {
          const objFields = prop.properties
            .map(p => this.propToEffectField(p, indent + 2))
            .join(',\n');
          schema = `Schema.Struct({\n${objFields}\n${spaces}})`;
        } else {
          schema = 'Schema.Record({ key: Schema.String, value: Schema.Unknown })';
        }
        break;
      
      case 'slot':
        schema = 'Schema.Unknown'; // ReactNode can't be validated
        break;
      
      default:
        schema = 'Schema.Unknown';
    }

    // Wrap optional
    if (!prop.required) {
      schema = `Schema.optional(${schema})`;
    }

    // Add description annotation
    if (prop.description) {
      schema = `${schema}.annotations({ description: ${JSON.stringify(prop.description)} })`;
    }

    return `${spaces}${prop.name}: ${schema}`;
  }

  private generatePuckConfig(
    name: string,
    props: ExtractedProp[],
    defaults: Record<string, unknown>
  ): string {
    const fields = props
      .map(p => this.propToPuckField(p))
      .filter(Boolean)
      .join(',\n    ');

    const defaultProps = Object.entries(defaults)
      .map(([k, v]) => `${k}: ${JSON.stringify(v)}`)
      .join(',\n    ');

    return `// Auto-generated Puck config for ${name}
import type { ComponentConfig } from '@measured/puck';

export const ${name}PuckConfig: ComponentConfig = {
  label: '${name}',
  fields: {
    ${fields}
  },
  defaultProps: {
    ${defaultProps}
  },
  render: ({ ${props.map(p => p.name).join(', ')} }) => {
    // Delegate to actual component with mapProps
    return null; // Replaced at runtime
  },
};
`;
  }

  private propToPuckField(prop: ExtractedProp): string | null {
    let field: string;

    switch (prop.type) {
      case 'string':
        field = `{ type: 'text' }`;
        break;
      
      case 'number':
        field = `{ type: 'number' }`;
        break;
      
      case 'boolean':
        field = `{ type: 'radio', options: [{ label: 'Yes', value: true }, { label: 'No', value: false }] }`;
        break;
      
      case 'enum':
        const options = prop.options!
          .map(o => `{ label: '${this.titleCase(o)}', value: '${o}' }`)
          .join(', ');
        field = `{ type: 'select', options: [${options}] }`;
        break;
      
      case 'array':
        if (prop.items && prop.items.length > 0) {
          const arrayFields = prop.items
            .map(i => {
              const f = this.propToPuckField(i);
              return f ? `${i.name}: ${f}` : null;
            })
            .filter(Boolean)
            .join(', ');
          field = `{ type: 'array', arrayFields: { ${arrayFields} } }`;
        } else {
          field = `{ type: 'array', arrayFields: {} }`;
        }
        break;
      
      case 'object':
        if (prop.properties && prop.properties.length > 0) {
          const objFields = prop.properties
            .map(p => {
              const f = this.propToPuckField(p);
              return f ? `${p.name}: ${f}` : null;
            })
            .filter(Boolean)
            .join(', ');
          field = `{ type: 'object', objectFields: { ${objFields} } }`;
        } else {
          return null; // Skip complex objects
        }
        break;
      
      case 'slot':
        field = `{ type: 'slot' }`;
        break;
      
      default:
        return null;
    }

    // Add label
    const label = prop.description || this.titleCase(prop.name);
    field = field.replace('{', `{ label: '${label}',`);

    return `${prop.name}: ${field}`;
  }

  private generateTypeDefinition(name: string, props: ExtractedProp[]): string {
    const fields = props.map(p => {
      const opt = p.required ? '' : '?';
      const type = this.propToTSType(p);
      const comment = p.description ? `  /** ${p.description} */\n` : '';
      return `${comment}  ${p.name}${opt}: ${type};`;
    }).join('\n');

    return `// Auto-generated types for ${name}
export interface ${name}Props {
${fields}
}
`;
  }

  private propToTSType(prop: ExtractedProp): string {
    switch (prop.type) {
      case 'string': return 'string';
      case 'number': return 'number';
      case 'boolean': return 'boolean';
      case 'enum': return prop.options!.map(o => `'${o}'`).join(' | ');
      case 'array':
        if (prop.items && prop.items.length > 0) {
          const itemType = `{ ${prop.items.map(i => `${i.name}: ${this.propToTSType(i)}`).join('; ')} }`;
          return `${itemType}[]`;
        }
        return 'unknown[]';
      case 'object':
        if (prop.properties && prop.properties.length > 0) {
          return `{ ${prop.properties.map(p => `${p.name}: ${this.propToTSType(p)}`).join('; ')} }`;
        }
        return 'Record<string, unknown>';
      case 'slot': return 'React.ReactNode';
      default: return 'unknown';
    }
  }

  private titleCase(str: string): string {
    return str
      .replace(/([A-Z])/g, ' $1')
      .replace(/^./, s => s.toUpperCase())
      .trim();
  }
}
```

### 14.5 KnowledgeEnricher

```typescript
// src/wrap/knowledge-enricher.ts
import { Schema } from '@effect/schema';
import { ExtractedProp } from './type-extractor';
import { existsSync, readFileSync, writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';
import crypto from 'crypto';

// Knowledge schema for validation
const KnowledgeSchema = Schema.Struct({
  description: Schema.String,
  useWhen: Schema.Array(Schema.String),
  doNotUseWhen: Schema.Array(Schema.String),
  examples: Schema.Array(Schema.Struct({
    prompt: Schema.String,
    props: Schema.Record({ key: Schema.String, value: Schema.Unknown }),
  })),
  relatedRigs: Schema.Array(Schema.String),
});

export type Knowledge = Schema.Schema.Type<typeof KnowledgeSchema>;

export interface EnrichmentOptions {
  model?: string;
  cacheDir?: string;
  skipCache?: boolean;
}

export class KnowledgeEnricher {
  private cacheDir: string;
  private model: string;

  constructor(options: EnrichmentOptions = {}) {
    this.cacheDir = options.cacheDir || '.punk/knowledge-cache';
    this.model = options.model || 'ska-2-128k-instruct';
    
    if (!existsSync(this.cacheDir)) {
      mkdirSync(this.cacheDir, { recursive: true });
    }
  }

  async enrich(
    componentName: string,
    packageName: string,
    packageVersion: string,
    category: string,
    props: ExtractedProp[],
    options: { skipCache?: boolean } = {}
  ): Promise<Knowledge> {
    // Check cache
    const cacheKey = this.getCacheKey(packageName, packageVersion, componentName);
    
    if (!options.skipCache) {
      const cached = this.getFromCache(cacheKey);
      if (cached) {
        return cached;
      }
    }

    // Build prompt
    const prompt = this.buildPrompt(componentName, packageName, category, props);

    // Call LLM
    const response = await this.callLLM(prompt);

    // Parse and validate
    const knowledge = this.parseResponse(response);

    // Cache result
    this.saveToCache(cacheKey, knowledge);

    return knowledge;
  }

  private buildPrompt(
    componentName: string,
    packageName: string,
    category: string,
    props: ExtractedProp[]
  ): string {
    const propsDescription = props.map(p => {
      let desc = `- ${p.name}: ${p.type}`;
      if (p.options) desc += ` (${p.options.join(' | ')})`;
      if (p.description) desc += ` - ${p.description}`;
      if (!p.required) desc += ' [optional]';
      return desc;
    }).join('\n');

    return `Analyze this React component and generate AI knowledge metadata for a visual app builder.

Component: ${componentName}
Package: ${packageName}
Category: ${category}

Props:
${propsDescription}

Generate JSON with:
1. "description": One clear sentence explaining what this component does and when to use it
2. "useWhen": Array of 3-5 natural language phrases describing when a user would want this component. Think about what the user might type or say.
3. "doNotUseWhen": Array of 2-3 anti-patterns where this component is the wrong choice
4. "examples": Array of 3-5 objects with "prompt" (what user might say) and "props" (corresponding prop values to use)
5. "relatedRigs": Array of other component types that serve similar purposes or work well together

Return ONLY valid JSON, no markdown code blocks or explanation.`;
  }

  private async callLLM(prompt: string): Promise<string> {
    // Integration with Ska or other LLM provider
    // This is a placeholder - actual implementation depends on your LLM setup
    
    const response = await fetch('https://api.ska.dev/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.SKA_API_KEY}`,
      },
      body: JSON.stringify({
        model: this.model,
        messages: [
          { role: 'system', content: 'You are a helpful assistant that generates component metadata for AI discoverability.' },
          { role: 'user', content: prompt },
        ],
        temperature: 0.3, // Lower for more consistent output
        response_format: { type: 'json_object' },
      }),
    });

    if (!response.ok) {
      throw new EnrichmentError(`LLM request failed: ${response.statusText}`);
    }

    const data = await response.json();
    return data.choices[0].message.content;
  }

  private parseResponse(response: string): Knowledge {
    try {
      // Clean response (remove markdown if present)
      let cleaned = response.trim();
      if (cleaned.startsWith('```')) {
        cleaned = cleaned.replace(/^```json?\n?/, '').replace(/\n?```$/, '');
      }

      const parsed = JSON.parse(cleaned);
      
      // Validate with Effect schema
      const result = Schema.decodeUnknownSync(KnowledgeSchema)(parsed);
      return result;
    } catch (error) {
      throw new EnrichmentError(`Failed to parse LLM response: ${error}`);
    }
  }

  private getCacheKey(pkg: string, version: string, component: string): string {
    const input = `${pkg}@${version}:${component}`;
    return crypto.createHash('sha256').update(input).digest('hex').slice(0, 16);
  }

  private getFromCache(key: string): Knowledge | null {
    const cachePath = join(this.cacheDir, `${key}.json`);
    if (existsSync(cachePath)) {
      try {
        const data = JSON.parse(readFileSync(cachePath, 'utf-8'));
        return Schema.decodeUnknownSync(KnowledgeSchema)(data);
      } catch {
        return null;
      }
    }
    return null;
  }

  private saveToCache(key: string, knowledge: Knowledge): void {
    const cachePath = join(this.cacheDir, `${key}.json`);
    writeFileSync(cachePath, JSON.stringify(knowledge, null, 2));
  }
}

export class EnrichmentError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'EnrichmentError';
  }
}
```

### 14.6 Main Orchestrator

```typescript
// src/wrap/index.ts
import { PackageResolver } from './package-resolver';
import { TypeExtractor } from './type-extractor';
import { SchemaGenerator } from './schema-generator';
import { KnowledgeEnricher } from './knowledge-enricher';
import { RigEmitter } from './rig-emitter';

export interface WrapOptions {
  package: string;
  component?: string;
  name?: string;
  tier: 'free' | 'starter' | 'pro' | 'premium';
  purchasable?: number;
  price?: number;
  category?: string;
  mapper?: string;
  output?: string;
  dryRun?: boolean;
  noEnrich?: boolean;
  regenerateKnowledge?: boolean;
}

export async function wrapComponent(options: WrapOptions): Promise<void> {
  const resolver = new PackageResolver();
  const generator = new SchemaGenerator();
  const enricher = new KnowledgeEnricher();
  const emitter = new RigEmitter();

  // Step 1: Resolve package
  console.log(`📦 Resolving ${options.package}...`);
  const pkg = resolver.resolve(options.package);

  if (!pkg.typesEntry) {
    throw new Error(`No TypeScript types found for ${options.package}`);
  }

  // Step 2: Extract types
  console.log(`🔍 Extracting types from ${pkg.typesEntry}...`);
  const extractor = new TypeExtractor(pkg.typesEntry);
  const componentName = options.component || 'default';
  const extraction = extractor.extract(componentName);

  if (extraction.warnings.length > 0) {
    console.log('⚠️  Warnings:');
    extraction.warnings.forEach(w => console.log(`   ${w}`));
  }

  console.log(`   Found ${extraction.props.length} props`);

  // Step 3: Generate schemas
  console.log('🔧 Generating schemas...');
  const schemas = generator.generate(
    options.name || extraction.componentName,
    extraction.props,
    extraction.defaultProps
  );

  // Step 4: Enrich with LLM
  let knowledge = null;
  if (!options.noEnrich) {
    console.log('🧠 Enriching with AI knowledge...');
    knowledge = await enricher.enrich(
      options.name || extraction.componentName,
      pkg.name,
      pkg.version,
      options.category || 'General',
      extraction.props,
      { skipCache: options.regenerateKnowledge }
    );
    console.log(`   Generated ${knowledge.examples.length} examples`);
  }

  // Step 5: Emit files
  if (options.dryRun) {
    console.log('\n📄 Dry run - would generate:');
    console.log('   rig.ts');
    console.log('   rig.schema.ts');
    console.log('   rig.puck.ts');
    console.log('   knowledge.json');
    console.log('   manifest.json');
    return;
  }

  console.log('📝 Writing files...');
  const outputDir = options.output || `./rigs/${(options.name || extraction.componentName).toLowerCase()}`;
  
  await emitter.emit({
    outputDir,
    componentName: options.name || extraction.componentName,
    sourcePackage: pkg,
    schemas,
    knowledge,
    tier: options.tier,
    purchasable: options.purchasable,
    price: options.price,
    category: options.category || 'General',
    mapperPath: options.mapper,
  });

  console.log(`\n✅ Rig created at ${outputDir}`);
}
```

---

## 15. Mapper Examples

### 15.1 Chart.js (Complex Transformation)

```typescript
// mappers/chartjs.ts
import type { PropMapper } from '@punk/rigs';

export const mapper: PropMapper = {
  // Reshape Chart.js's complex API to simple flat props
  reshapeProps: {
    chartType: {
      type: 'enum',
      options: ['bar', 'line', 'pie', 'doughnut', 'radar', 'polarArea'],
      default: 'bar',
    },
    title: {
      type: 'string',
      default: '',
    },
    data: {
      type: 'array',
      items: {
        label: { type: 'string' },
        value: { type: 'number' },
        color: { type: 'string' },
      },
    },
    showLegend: {
      type: 'boolean',
      default: true,
    },
    showGrid: {
      type: 'boolean',
      default: true,
    },
    aspectRatio: {
      type: 'number',
      default: 2,
    },
  },

  // Transform simplified props to Chart.js format
  mapProps: (punk) => ({
    type: punk.chartType,
    data: {
      labels: punk.data.map(d => d.label),
      datasets: [{
        label: punk.title,
        data: punk.data.map(d => d.value),
        backgroundColor: punk.data.map(d => d.color || generateColor()),
        borderColor: punk.data.map(d => d.color || generateColor()),
        borderWidth: 1,
      }],
    },
    options: {
      responsive: true,
      maintainAspectRatio: true,
      aspectRatio: punk.aspectRatio,
      plugins: {
        legend: {
          display: punk.showLegend,
        },
        title: {
          display: !!punk.title,
          text: punk.title,
        },
      },
      scales: punk.chartType !== 'pie' && punk.chartType !== 'doughnut' ? {
        x: { grid: { display: punk.showGrid } },
        y: { grid: { display: punk.showGrid } },
      } : undefined,
    },
  }),
};

// Helper for default colors
const colors = ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF', '#FF9F40'];
let colorIndex = 0;
function generateColor(): string {
  return colors[colorIndex++ % colors.length];
}
```

### 15.2 TanStack Table (Heavy Transformation)

```typescript
// mappers/tanstack-table.ts
import type { PropMapper } from '@punk/rigs';

export const mapper: PropMapper = {
  reshapeProps: {
    columns: {
      type: 'array',
      items: {
        key: { type: 'string' },
        header: { type: 'string' },
        sortable: { type: 'boolean' },
        width: { type: 'number' },
      },
    },
    data: {
      type: 'array',
      items: {}, // Dynamic based on columns
    },
    pageSize: {
      type: 'number',
      default: 10,
    },
    showPagination: {
      type: 'boolean',
      default: true,
    },
    showSearch: {
      type: 'boolean',
      default: true,
    },
    striped: {
      type: 'boolean',
      default: true,
    },
  },

  mapProps: (punk) => ({
    columns: punk.columns.map(col => ({
      accessorKey: col.key,
      header: col.header,
      enableSorting: col.sortable ?? true,
      size: col.width,
    })),
    data: punk.data,
    initialState: {
      pagination: {
        pageSize: punk.pageSize,
      },
    },
    // These become wrapper component props
    _punkMeta: {
      showPagination: punk.showPagination,
      showSearch: punk.showSearch,
      striped: punk.striped,
    },
  }),

  // Optional: wrap the component to handle _punkMeta
  wrapComponent: (TanStackTable) => {
    return function PunkDataTable(props) {
      const { _punkMeta, ...tableProps } = props;
      return (
        <div className="punk-datatable">
          {_punkMeta.showSearch && <SearchInput />}
          <TanStackTable {...tableProps} />
          {_punkMeta.showPagination && <Pagination />}
        </div>
      );
    };
  },
};
```

### 15.3 Lexical Rich Text (Composition)

```typescript
// mappers/lexical.ts
import type { PropMapper } from '@punk/rigs';

export const mapper: PropMapper = {
  reshapeProps: {
    initialContent: {
      type: 'string',
      default: '',
    },
    placeholder: {
      type: 'string',
      default: 'Start typing...',
    },
    editable: {
      type: 'boolean',
      default: true,
    },
    toolbar: {
      type: 'enum',
      options: ['none', 'basic', 'full'],
      default: 'basic',
    },
    features: {
      type: 'object',
      properties: {
        bold: { type: 'boolean', default: true },
        italic: { type: 'boolean', default: true },
        underline: { type: 'boolean', default: true },
        strikethrough: { type: 'boolean', default: false },
        links: { type: 'boolean', default: true },
        lists: { type: 'boolean', default: true },
        headings: { type: 'boolean', default: true },
        codeBlocks: { type: 'boolean', default: false },
        images: { type: 'boolean', default: false },
        mentions: { type: 'boolean', default: false },
      },
    },
  },

  mapProps: (punk) => {
    // Build plugins array based on features
    const plugins = [];
    
    if (punk.features.bold || punk.features.italic || punk.features.underline) {
      plugins.push('RichTextPlugin');
    }
    if (punk.features.links) {
      plugins.push('LinkPlugin');
    }
    if (punk.features.lists) {
      plugins.push('ListPlugin');
    }
    if (punk.features.headings) {
      plugins.push('HeadingPlugin');
    }
    if (punk.features.codeBlocks) {
      plugins.push('CodePlugin');
    }
    if (punk.features.images) {
      plugins.push('ImagePlugin');
    }
    if (punk.features.mentions) {
      plugins.push('MentionPlugin');
    }

    return {
      initialConfig: {
        namespace: 'PunkRichText',
        editable: punk.editable,
        onError: console.error,
      },
      _punkPlugins: plugins,
      _punkToolbar: punk.toolbar,
      _punkPlaceholder: punk.placeholder,
      _punkInitialContent: punk.initialContent,
    };
  },

  // Compose multiple Lexical components
  wrapComponent: (LexicalComposer) => {
    return function PunkRichText(props) {
      const { _punkPlugins, _punkToolbar, _punkPlaceholder, _punkInitialContent, ...config } = props;
      
      return (
        <LexicalComposer initialConfig={config.initialConfig}>
          {_punkToolbar !== 'none' && <Toolbar variant={_punkToolbar} />}
          <ContentEditable placeholder={_punkPlaceholder} />
          {_punkPlugins.map(plugin => <Plugin key={plugin} name={plugin} />)}
        </LexicalComposer>
      );
    };
  },
};
```

### 15.4 Simple Passthrough (No Mapper Needed)

These components work without any mapper because their props are already clean:

```bash
# Base UI - clean APIs, no mapper needed
punk wrap @base-ui/react --component Accordion --tier free
punk wrap @base-ui/react --component Tabs --tier starter
punk wrap @base-ui/react --component Dialog --tier starter

# Lucide - simple icon prop
punk wrap lucide-react --component Icon --tier free

# Simple components
punk wrap react-spinners --component ClipLoader --tier free
punk wrap react-syntax-highlighter --component Prism --tier pro
```

---

## 16. Error Handling

### 16.1 Error Hierarchy

```typescript
// src/errors.ts

export class PunkWrapError extends Error {
  constructor(message: string, public code: string) {
    super(message);
    this.name = 'PunkWrapError';
  }
}

export class PackageNotFoundError extends PunkWrapError {
  constructor(packageName: string) {
    super(
      `Package "${packageName}" not found in node_modules.\n` +
      `Run: npm install ${packageName}`,
      'PKG_NOT_FOUND'
    );
  }
}

export class TypesNotFoundError extends PunkWrapError {
  constructor(packageName: string) {
    super(
      `No TypeScript types found for "${packageName}".\n` +
      `Try: npm install @types/${packageName.replace('@', '').replace('/', '__')}`,
      'TYPES_NOT_FOUND'
    );
  }
}

export class ComponentNotFoundError extends PunkWrapError {
  constructor(componentName: string, availableExports: string[]) {
    const suggestion = availableExports.length > 0
      ? `Available exports: ${availableExports.join(', ')}`
      : 'No exports found';
    super(
      `Component "${componentName}" not found in package exports.\n${suggestion}`,
      'COMPONENT_NOT_FOUND'
    );
  }
}

export class ExtractionError extends PunkWrapError {
  constructor(message: string, public prop?: string) {
    super(
      prop ? `Failed to extract prop "${prop}": ${message}` : message,
      'EXTRACTION_FAILED'
    );
  }
}

export class EnrichmentError extends PunkWrapError {
  constructor(message: string) {
    super(`AI enrichment failed: ${message}`, 'ENRICHMENT_FAILED');
  }
}

export class MapperError extends PunkWrapError {
  constructor(mapperPath: string, message: string) {
    super(`Invalid mapper at "${mapperPath}": ${message}`, 'MAPPER_INVALID');
  }
}

export class ValidationError extends PunkWrapError {
  constructor(public errors: Array<{ path: string; message: string }>) {
    const details = errors.map(e => `  - ${e.path}: ${e.message}`).join('\n');
    super(`Rig validation failed:\n${details}`, 'VALIDATION_FAILED');
  }
}
```

### 16.2 Recovery Strategies

```typescript
// src/wrap/recovery.ts

export interface RecoveryResult {
  recovered: boolean;
  message?: string;
  fallback?: unknown;
}

export const recoveryStrategies: Record<string, (error: PunkWrapError) => RecoveryResult> = {
  
  // Types not found - try to continue without full type info
  TYPES_NOT_FOUND: (error) => {
    console.warn(`⚠️  ${error.message}`);
    console.warn('   Continuing with limited type information...');
    return {
      recovered: true,
      message: 'Using basic prop extraction without TypeScript types',
      fallback: { props: [], defaultProps: {} },
    };
  },

  // Extraction failed for a prop - skip it
  EXTRACTION_FAILED: (error) => {
    if (error.prop) {
      console.warn(`⚠️  Skipping prop "${error.prop}": ${error.message}`);
      return { recovered: true };
    }
    return { recovered: false };
  },

  // Enrichment failed - continue without AI knowledge
  ENRICHMENT_FAILED: (error) => {
    console.warn(`⚠️  ${error.message}`);
    console.warn('   Continuing without AI knowledge (use --regenerate-knowledge later)');
    return {
      recovered: true,
      message: 'Generated without AI enrichment',
      fallback: {
        description: 'No description available',
        useWhen: [],
        doNotUseWhen: [],
        examples: [],
        relatedRigs: [],
      },
    };
  },

  // Mapper invalid - continue without mapping
  MAPPER_INVALID: (error) => {
    console.warn(`⚠️  ${error.message}`);
    console.warn('   Continuing without prop mapping');
    return {
      recovered: true,
      message: 'Props passed through without transformation',
    };
  },
};

export function attemptRecovery(error: PunkWrapError): RecoveryResult {
  const strategy = recoveryStrategies[error.code];
  if (strategy) {
    return strategy(error);
  }
  return { recovered: false };
}
```

### 16.3 CLI Error Output

```typescript
// src/cli/error-handler.ts

import chalk from 'chalk';
import { PunkWrapError, attemptRecovery } from '../errors';

export function handleError(error: unknown): never | void {
  if (error instanceof PunkWrapError) {
    // Try recovery
    const recovery = attemptRecovery(error);
    if (recovery.recovered) {
      return; // Continue execution
    }

    // Fatal error
    console.error(chalk.red(`\n❌ ${error.name}: ${error.message}\n`));
    
    // Show help based on error type
    switch (error.code) {
      case 'PKG_NOT_FOUND':
        console.error(chalk.yellow('Hint: Make sure the package is installed locally.'));
        console.error(chalk.dim('  npm install <package-name>'));
        break;
      
      case 'TYPES_NOT_FOUND':
        console.error(chalk.yellow('Hint: Install TypeScript types or use --no-types flag.'));
        console.error(chalk.dim('  npm install @types/<package-name>'));
        console.error(chalk.dim('  punk wrap <package> --no-types'));
        break;
      
      case 'COMPONENT_NOT_FOUND':
        console.error(chalk.yellow('Hint: Check the component name or list exports.'));
        console.error(chalk.dim('  punk inspect <package> --exports'));
        break;
      
      case 'VALIDATION_FAILED':
        console.error(chalk.yellow('Hint: Fix the issues above or use --skip-validation.'));
        break;
    }

    process.exit(1);
  }

  // Unknown error
  console.error(chalk.red('\n❌ Unexpected error:'));
  console.error(error);
  process.exit(1);
}
```

### 16.4 Graceful Degradation Table

| Error | Severity | Recovery | Result |
|-------|----------|----------|--------|
| Package not found | Fatal | None | Exit with install instructions |
| Types not found | Warning | Use JS introspection | Limited prop extraction |
| Component not found | Fatal | Show available exports | Exit with suggestions |
| Prop extraction fails | Warning | Skip prop | Rig missing that prop |
| Complex type unsupported | Warning | Mark as `unknown` | Prop excluded from Puck |
| LLM enrichment fails | Warning | Use empty knowledge | Rig works, no AI hints |
| Mapper syntax error | Warning | Skip mapper | Props pass through directly |
| Validation fails | Error | `--skip-validation` | Rig may have issues |

---

## 17. Testing & Validation

### 17.1 Rig Validation

```typescript
// src/validate/index.ts

import { Schema } from '@effect/schema';
import type { Rig } from '../types';

export interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

export interface ValidationError {
  path: string;
  message: string;
  code: string;
}

export interface ValidationWarning {
  path: string;
  message: string;
  suggestion?: string;
}

export async function validateRig(rigPath: string): Promise<ValidationResult> {
  const errors: ValidationError[] = [];
  const warnings: ValidationWarning[] = [];

  // Load rig
  const rig = await import(rigPath);

  // 1. Structure validation
  validateStructure(rig, errors);

  // 2. Schema validation
  validateSchema(rig, errors, warnings);

  // 3. Puck config validation
  validatePuckConfig(rig, errors, warnings);

  // 4. Knowledge validation
  validateKnowledge(rig, errors, warnings);

  // 5. Runtime validation
  await validateRuntime(rig, errors, warnings);

  return {
    valid: errors.length === 0,
    errors,
    warnings,
  };
}

function validateStructure(rig: any, errors: ValidationError[]): void {
  const required = ['name', 'component', 'tier', 'schema', 'puckConfig'];
  
  for (const field of required) {
    if (!(field in rig)) {
      errors.push({
        path: field,
        message: `Missing required field "${field}"`,
        code: 'MISSING_FIELD',
      });
    }
  }

  // Validate tier
  const validTiers = ['free', 'starter', 'pro', 'premium'];
  if (rig.tier && !validTiers.includes(rig.tier)) {
    errors.push({
      path: 'tier',
      message: `Invalid tier "${rig.tier}". Must be one of: ${validTiers.join(', ')}`,
      code: 'INVALID_TIER',
    });
  }

  // Validate purchasable/price
  if (rig.tier === 'premium' && !rig.price) {
    errors.push({
      path: 'price',
      message: 'Premium tier requires a price',
      code: 'MISSING_PRICE',
    });
  }

  if (rig.purchasable && typeof rig.purchasable !== 'number') {
    errors.push({
      path: 'purchasable',
      message: 'Purchasable must be a number (price in cents)',
      code: 'INVALID_PURCHASABLE',
    });
  }
}

function validateSchema(
  rig: any, 
  errors: ValidationError[], 
  warnings: ValidationWarning[]
): void {
  if (!rig.schema) return;

  try {
    // Verify it's a valid Effect schema
    if (!Schema.isSchema(rig.schema)) {
      errors.push({
        path: 'schema',
        message: 'Schema is not a valid @effect/schema',
        code: 'INVALID_SCHEMA',
      });
      return;
    }

    // Test encoding/decoding with default props
    if (rig.puckConfig?.defaultProps) {
      const result = Schema.decodeUnknownEither(rig.schema)(rig.puckConfig.defaultProps);
      if (result._tag === 'Left') {
        errors.push({
          path: 'schema',
          message: 'Default props do not match schema',
          code: 'SCHEMA_MISMATCH',
        });
      }
    }
  } catch (e) {
    errors.push({
      path: 'schema',
      message: `Schema validation error: ${e}`,
      code: 'SCHEMA_ERROR',
    });
  }
}

function validatePuckConfig(
  rig: any,
  errors: ValidationError[],
  warnings: ValidationWarning[]
): void {
  if (!rig.puckConfig) return;

  const config = rig.puckConfig;

  // Check label
  if (!config.label) {
    warnings.push({
      path: 'puckConfig.label',
      message: 'Missing label, will use component name',
    });
  }

  // Check fields match schema
  if (rig.schema && config.fields) {
    const schemaProps = getSchemaPropertyNames(rig.schema);
    const fieldNames = Object.keys(config.fields);

    for (const prop of schemaProps) {
      if (!fieldNames.includes(prop)) {
        warnings.push({
          path: `puckConfig.fields.${prop}`,
          message: `Schema prop "${prop}" has no Puck field`,
          suggestion: 'Add field or mark as internal',
        });
      }
    }
  }

  // Check render function
  if (typeof config.render !== 'function') {
    errors.push({
      path: 'puckConfig.render',
      message: 'Missing or invalid render function',
      code: 'MISSING_RENDER',
    });
  }
}

function validateKnowledge(
  rig: any,
  errors: ValidationError[],
  warnings: ValidationWarning[]
): void {
  if (!rig.knowledge) {
    warnings.push({
      path: 'knowledge',
      message: 'No AI knowledge - component will have limited discoverability',
      suggestion: 'Run: punk rigs enrich ' + rig.name,
    });
    return;
  }

  const k = rig.knowledge;

  if (!k.description || k.description.length < 10) {
    warnings.push({
      path: 'knowledge.description',
      message: 'Description is missing or too short',
    });
  }

  if (!k.useWhen || k.useWhen.length < 2) {
    warnings.push({
      path: 'knowledge.useWhen',
      message: 'Should have at least 2 useWhen triggers',
    });
  }

  if (!k.examples || k.examples.length < 2) {
    warnings.push({
      path: 'knowledge.examples',
      message: 'Should have at least 2 examples for AI learning',
    });
  }

  // Validate examples have valid props
  if (k.examples && rig.schema) {
    for (let i = 0; i < k.examples.length; i++) {
      const example = k.examples[i];
      const result = Schema.decodeUnknownEither(rig.schema)(example.props);
      if (result._tag === 'Left') {
        warnings.push({
          path: `knowledge.examples[${i}].props`,
          message: 'Example props do not match schema',
        });
      }
    }
  }
}

async function validateRuntime(
  rig: any,
  errors: ValidationError[],
  warnings: ValidationWarning[]
): Promise<void> {
  // Skip if no component
  if (!rig.component) return;

  try {
    // Check if component is a valid React component
    const Component = rig.component;
    
    if (typeof Component !== 'function' && typeof Component?.render !== 'function') {
      errors.push({
        path: 'component',
        message: 'Not a valid React component',
        code: 'INVALID_COMPONENT',
      });
    }

    // Try to render with default props (headless check)
    if (rig.puckConfig?.defaultProps && typeof Component === 'function') {
      try {
        const props = rig.mapProps 
          ? rig.mapProps(rig.puckConfig.defaultProps)
          : rig.puckConfig.defaultProps;
        
        // This would need React test renderer in real implementation
        // Just checking props transformation doesn't throw
        if (rig.mapProps) {
          rig.mapProps(rig.puckConfig.defaultProps);
        }
      } catch (e) {
        errors.push({
          path: 'component',
          message: `Component render failed: ${e}`,
          code: 'RENDER_ERROR',
        });
      }
    }
  } catch (e) {
    errors.push({
      path: 'component',
      message: `Runtime validation failed: ${e}`,
      code: 'RUNTIME_ERROR',
    });
  }
}

function getSchemaPropertyNames(schema: Schema.Schema<any>): string[] {
  // Extract property names from Effect schema
  // Implementation depends on Effect internals
  try {
    const ast = schema.ast;
    if (ast._tag === 'TypeLiteral') {
      return ast.propertySignatures.map((p: any) => p.name);
    }
  } catch {
    return [];
  }
  return [];
}
```

### 17.2 Test Suite Generator

```typescript
// src/validate/test-generator.ts

export function generateTestSuite(rigPath: string, outputPath: string): string {
  return `// Auto-generated test suite for Rig
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { Schema } from '@effect/schema';
import rig from '${rigPath}';

describe('${rigPath} Rig', () => {
  describe('Structure', () => {
    it('has required fields', () => {
      expect(rig.name).toBeDefined();
      expect(rig.component).toBeDefined();
      expect(rig.tier).toBeDefined();
      expect(rig.schema).toBeDefined();
      expect(rig.puckConfig).toBeDefined();
    });

    it('has valid tier', () => {
      expect(['free', 'starter', 'pro', 'premium']).toContain(rig.tier);
    });
  });

  describe('Schema', () => {
    it('validates default props', () => {
      const result = Schema.decodeUnknownEither(rig.schema)(
        rig.puckConfig.defaultProps
      );
      expect(result._tag).toBe('Right');
    });

    it('rejects invalid props', () => {
      const result = Schema.decodeUnknownEither(rig.schema)({
        invalidProp: 'should fail',
      });
      expect(result._tag).toBe('Left');
    });
  });

  describe('Puck Config', () => {
    it('has fields for all schema props', () => {
      const schemaProps = Object.keys(rig.puckConfig.defaultProps);
      const fieldNames = Object.keys(rig.puckConfig.fields);
      
      for (const prop of schemaProps) {
        expect(fieldNames).toContain(prop);
      }
    });

    it('render function exists', () => {
      expect(typeof rig.puckConfig.render).toBe('function');
    });
  });

  describe('Component', () => {
    it('renders without crashing', () => {
      const props = rig.mapProps
        ? rig.mapProps(rig.puckConfig.defaultProps)
        : rig.puckConfig.defaultProps;
      
      expect(() => {
        render(<rig.component {...props} />);
      }).not.toThrow();
    });
  });

  describe('Knowledge', () => {
    it('has description', () => {
      expect(rig.knowledge?.description).toBeDefined();
      expect(rig.knowledge?.description.length).toBeGreaterThan(10);
    });

    it('has useWhen triggers', () => {
      expect(rig.knowledge?.useWhen?.length).toBeGreaterThanOrEqual(2);
    });

    it('has valid examples', () => {
      for (const example of rig.knowledge?.examples || []) {
        expect(example.prompt).toBeDefined();
        expect(example.props).toBeDefined();
        
        // Validate example props against schema
        const result = Schema.decodeUnknownEither(rig.schema)(example.props);
        expect(result._tag).toBe('Right');
      }
    });
  });
});
`;
}
```

### 17.3 CLI Validation Command

```bash
# Validate a single rig
punk rigs validate ./rigs/chart

# Validate all rigs in directory
punk rigs validate ./rigs --all

# Validate with test generation
punk rigs validate ./rigs/chart --generate-tests

# Validate and fix auto-fixable issues
punk rigs validate ./rigs/chart --fix

# CI mode (exit code reflects validation)
punk rigs validate ./rigs --ci
```

### 17.4 Validation Output

```
$ punk rigs validate ./rigs/chart

📋 Validating Chart Rig...

✅ Structure
   ├── name: Chart
   ├── tier: pro
   ├── purchasable: $19
   └── component: ✓

✅ Schema
   ├── Valid @effect/schema
   ├── Default props match schema
   └── 4 props defined

✅ Puck Config
   ├── Label: Chart
   ├── Fields: 4/4 props covered
   └── Render function: ✓

⚠️  Knowledge
   ├── Description: ✓
   ├── useWhen: 4 triggers
   ├── examples: 3 examples
   └── ⚠ Example 2 props don't match schema (missing 'showLegend')

✅ Runtime
   ├── Component loads
   ├── mapProps works
   └── Renders without error

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Result: VALID (1 warning)

Run with --fix to auto-repair warnings
```

---

## 18. Summary

| Goal | Solution |
|------|----------|
| Zero authoring | `punk wrap` extracts from TypeScript |
| AI discoverability | LLM enrichment at wrap time |
| Type safety | Effect schemas from TS types |
| Visual editing | Puck configs from TS types |
| Access control | Tier + purchasable + premium |
| Distribution | Depot + SQLar packages |
| Robustness | Error recovery + validation |
| Quality | Auto-generated test suites |

**Author writes:** Package name + tier flags + optional mapper

**Machine generates:** Everything else

---

## 19. Related Documents

- **PUNK_CORE.md** — Core architecture
- **PUNK_MODS.md** — Depot distribution model
- **PUNK_SKA.md** — AI engine integration
- **PUNK_SCHEMAS.md** — Effect schema details

---

*End of Specification*
