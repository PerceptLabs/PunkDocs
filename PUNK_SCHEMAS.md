# Punk Schemas

**Version:** 3.0  
**Status:** Canonical Source of Truth  
**Last Updated:** December 2025

---

## Overview

This document contains all **@effect/schema** definitions, TypeScript interfaces, and SQL schemas used across the Punk Framework.

As of December 2025, Punk uses **@effect/schema** (not Zod) for all runtime validation, type inference, and AI schema generation.

---

## 1. Effect Schema Primer

### 1.1 Why Effect Schema

| Feature | Zod | @effect/schema |
|---------|-----|----------------|
| Bundle size | 12KB | 8KB |
| Encoding/Decoding | Manual | Built-in |
| Transformations | Limited | First-class |
| Error messages | Basic | Structured AST |
| Effect integration | None | Native |
| Tree-shaking | Partial | Full |

### 1.2 Import Pattern

```typescript
// Standard import
import { Schema } from '@effect/schema'

// With helpers
import { Schema, AST, ParseResult } from '@effect/schema'

// Type extraction
type MyType = Schema.Schema.Type<typeof MySchema>
```

### 1.3 Common Conversions from Zod

| Zod | Effect Schema |
|-----|---------------|
| `z.object({})` | `Schema.Struct({})` |
| `z.string()` | `Schema.String` |
| `z.number()` | `Schema.Number` |
| `z.boolean()` | `Schema.Boolean` |
| `z.array(x)` | `Schema.Array(x)` |
| `z.enum(['a','b'])` | `Schema.Literal('a', 'b')` |
| `z.optional(x)` | `Schema.optional(x)` |
| `z.record(k, v)` | `Schema.Record({ key: k, value: v })` |
| `z.lazy(() => x)` | `Schema.suspend(() => x)` |
| `z.infer<typeof X>` | `Schema.Schema.Type<typeof X>` |
| `x.refine(fn)` | `x.pipe(Schema.filter(fn))` |
| `x.brand<'T'>()` | `x.pipe(Schema.brand('T'))` |
| `x.default(v)` | `Schema.withDefault(x, () => v)` |

---

## 2. Core Schema Types

### 2.1 Punk Component

```typescript
// @punk/core/schema/punk.ts
import { Schema } from '@effect/schema'

/**
 * Base props shared by all components
 */
export const BasePropsSchema = Schema.Struct({
  id: Schema.optional(Schema.String),
  className: Schema.optional(Schema.String),
  style: Schema.optional(Schema.Record({ 
    key: Schema.String, 
    value: Schema.String 
  })),
  'data-testid': Schema.optional(Schema.String),
})

/**
 * Recursive component schema using suspend for self-reference
 */
export interface PunkComponent {
  type: string
  props: Record<string, unknown>
  children: PunkComponent[]
}

export const PunkComponentSchema: Schema.Schema<PunkComponent> = Schema.suspend(
  () => Schema.Struct({
    type: Schema.String.pipe(Schema.minLength(1)),
    props: Schema.Record({ 
      key: Schema.String, 
      value: Schema.Unknown 
    }).pipe(Schema.withDefault(() => ({}))),
    children: Schema.Array(PunkComponentSchema).pipe(
      Schema.withDefault(() => [])
    ),
  })
)

/**
 * Document metadata
 */
export const PunkMetadataSchema = Schema.Struct({
  generatedAt: Schema.optional(Schema.String.pipe(Schema.pattern(/^\d{4}-\d{2}-\d{2}T/))),
  model: Schema.optional(Schema.String),
  prompt: Schema.optional(Schema.String),
  duration: Schema.optional(Schema.Number),
  tokens: Schema.optional(Schema.Struct({
    input: Schema.Number,
    output: Schema.Number,
  })),
})

/**
 * Root schema for complete Punk documents
 */
export const PunkDocumentSchema = Schema.Struct({
  version: Schema.Literal('1.0', '2.0').pipe(
    Schema.withDefault(() => '2.0' as const)
  ),
  root: PunkComponentSchema,
  metadata: Schema.optional(PunkMetadataSchema),
})

/**
 * Inferred types
 */
export type PunkMetadata = Schema.Schema.Type<typeof PunkMetadataSchema>
export type PunkDocument = Schema.Schema.Type<typeof PunkDocumentSchema>
```

### 2.2 Component Type Enum

```typescript
// @punk/core/schema/components.ts
import { Schema } from '@effect/schema'

/**
 * All built-in component types
 * Uses Schema.Literal for exhaustive union
 */
export const ComponentTypeSchema = Schema.Literal(
  // Layout
  'Box',
  'Flex',
  'Grid',
  'Container',
  'Section',
  'Card',
  'Stack',
  'Columns',
  'Spacer',
  'Divider',
  
  // Typography
  'Text',
  'Heading',
  'Code',
  'Quote',
  'Link',
  
  // Form
  'Button',
  'TextField',
  'TextArea',
  'Select',
  'Checkbox',
  'RadioGroup',
  'Switch',
  'Slider',
  'Form',
  
  // Feedback
  'Dialog',
  'AlertDialog',
  'Toast',
  'Tooltip',
  'Popover',
  'Alert',
  
  // Navigation
  'Tabs',
  'TabsList',
  'TabsTrigger',
  'TabsContent',
  'Menu',
  'Accordion',
  
  // Data Display
  'Table',
  'Avatar',
  'Badge',
  'Progress',
  'Separator',
  
  // Media
  'Image',
  'Icon',
)

export type ComponentType = Schema.Schema.Type<typeof ComponentTypeSchema>

/**
 * Extended component types (Rigs)
 */
export const ExtendedComponentTypeSchema = Schema.Literal(
  'Chart',
  'DataTable',
  'RichText',
  'DatePicker',
  'Calendar',
  'Kanban',
  'CodeEditor',
  'MapView',
  'FileUpload',
)

export type ExtendedComponentType = Schema.Schema.Type<typeof ExtendedComponentTypeSchema>

/**
 * All component types (base + extended)
 */
export const AllComponentTypesSchema = Schema.Union(
  ComponentTypeSchema,
  ExtendedComponentTypeSchema,
)
```

---

## 3. Component Props Schemas

### 3.1 Layout Components

```typescript
// @punk/core/schema/props/layout.ts
import { Schema } from '@effect/schema'

export const FlexDirectionSchema = Schema.Literal(
  'row', 'column', 'row-reverse', 'column-reverse'
)

export const AlignItemsSchema = Schema.Literal(
  'start', 'center', 'end', 'stretch', 'baseline'
)

export const JustifyContentSchema = Schema.Literal(
  'start', 'center', 'end', 'between', 'around', 'evenly'
)

export const FlexPropsSchema = Schema.Struct({
  direction: Schema.optional(FlexDirectionSchema),
  align: Schema.optional(AlignItemsSchema),
  justify: Schema.optional(JustifyContentSchema),
  gap: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  wrap: Schema.optional(Schema.Literal('nowrap', 'wrap', 'wrap-reverse')),
})

export const GridPropsSchema = Schema.Struct({
  columns: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  rows: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  gap: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  areas: Schema.optional(Schema.String),
})

export const ContainerPropsSchema = Schema.Struct({
  size: Schema.optional(Schema.Literal('sm', 'md', 'lg', 'xl', 'full')),
  padding: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  centered: Schema.optional(Schema.Boolean),
})

export const StackPropsSchema = Schema.Struct({
  direction: Schema.optional(Schema.Literal('vertical', 'horizontal')),
  gap: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  align: Schema.optional(AlignItemsSchema),
})

export const CardPropsSchema = Schema.Struct({
  variant: Schema.optional(Schema.Literal('elevated', 'outlined', 'filled')),
  padding: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  interactive: Schema.optional(Schema.Boolean),
})

// Type exports
export type FlexProps = Schema.Schema.Type<typeof FlexPropsSchema>
export type GridProps = Schema.Schema.Type<typeof GridPropsSchema>
export type ContainerProps = Schema.Schema.Type<typeof ContainerPropsSchema>
export type StackProps = Schema.Schema.Type<typeof StackPropsSchema>
export type CardProps = Schema.Schema.Type<typeof CardPropsSchema>
```

### 3.2 Typography Components

```typescript
// @punk/core/schema/props/typography.ts
import { Schema } from '@effect/schema'

export const TextPropsSchema = Schema.Struct({
  content: Schema.String,
  size: Schema.optional(Schema.Literal('xs', 'sm', 'md', 'lg', 'xl', '2xl')),
  weight: Schema.optional(Schema.Literal('normal', 'medium', 'semibold', 'bold')),
  color: Schema.optional(Schema.String),
  align: Schema.optional(Schema.Literal('left', 'center', 'right', 'justify')),
})

export const HeadingPropsSchema = Schema.Struct({
  content: Schema.String,
  level: Schema.optional(Schema.Literal(1, 2, 3, 4, 5, 6)).pipe(
    Schema.withDefault(() => 2 as const)
  ),
  size: Schema.optional(Schema.Literal('xs', 'sm', 'md', 'lg', 'xl', '2xl', '3xl')),
})

export const LinkPropsSchema = Schema.Struct({
  href: Schema.String,
  content: Schema.String,
  target: Schema.optional(Schema.Literal('_self', '_blank', '_parent', '_top')),
  variant: Schema.optional(Schema.Literal('default', 'subtle', 'underline')),
})

export type TextProps = Schema.Schema.Type<typeof TextPropsSchema>
export type HeadingProps = Schema.Schema.Type<typeof HeadingPropsSchema>
export type LinkProps = Schema.Schema.Type<typeof LinkPropsSchema>
```

### 3.3 Form Components

```typescript
// @punk/core/schema/props/form.ts
import { Schema } from '@effect/schema'

export const ButtonVariantSchema = Schema.Literal(
  'default', 'primary', 'secondary', 'destructive', 
  'outline', 'ghost', 'link'
)

export const SizeSchema = Schema.Literal('sm', 'md', 'lg')

export const ButtonPropsSchema = Schema.Struct({
  label: Schema.String,
  variant: Schema.optional(ButtonVariantSchema).pipe(
    Schema.withDefault(() => 'default' as const)
  ),
  size: Schema.optional(SizeSchema).pipe(
    Schema.withDefault(() => 'md' as const)
  ),
  disabled: Schema.optional(Schema.Boolean),
  loading: Schema.optional(Schema.Boolean),
  icon: Schema.optional(Schema.String),
  iconPosition: Schema.optional(Schema.Literal('left', 'right')),
})

export const TextFieldPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  placeholder: Schema.optional(Schema.String),
  defaultValue: Schema.optional(Schema.String),
  type: Schema.optional(Schema.Literal(
    'text', 'email', 'password', 'number', 'tel', 'url', 'search'
  )).pipe(Schema.withDefault(() => 'text' as const)),
  required: Schema.optional(Schema.Boolean),
  disabled: Schema.optional(Schema.Boolean),
  error: Schema.optional(Schema.String),
})

export const TextAreaPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  placeholder: Schema.optional(Schema.String),
  defaultValue: Schema.optional(Schema.String),
  rows: Schema.optional(Schema.Number.pipe(
    Schema.greaterThan(0),
    Schema.lessThanOrEqualTo(20)
  )).pipe(Schema.withDefault(() => 3)),
  required: Schema.optional(Schema.Boolean),
  disabled: Schema.optional(Schema.Boolean),
  resize: Schema.optional(Schema.Literal('none', 'vertical', 'horizontal', 'both')),
})

export const SelectOptionSchema = Schema.Struct({
  value: Schema.String,
  label: Schema.String,
  disabled: Schema.optional(Schema.Boolean),
})

export const SelectPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  placeholder: Schema.optional(Schema.String),
  defaultValue: Schema.optional(Schema.String),
  options: Schema.Array(SelectOptionSchema),
  required: Schema.optional(Schema.Boolean),
  disabled: Schema.optional(Schema.Boolean),
})

export const CheckboxPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.String,
  checked: Schema.optional(Schema.Boolean),
  defaultChecked: Schema.optional(Schema.Boolean),
  disabled: Schema.optional(Schema.Boolean),
  indeterminate: Schema.optional(Schema.Boolean),
})

export const RadioOptionSchema = Schema.Struct({
  value: Schema.String,
  label: Schema.String,
  disabled: Schema.optional(Schema.Boolean),
})

export const RadioGroupPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  options: Schema.Array(RadioOptionSchema),
  defaultValue: Schema.optional(Schema.String),
  orientation: Schema.optional(Schema.Literal('horizontal', 'vertical')),
  disabled: Schema.optional(Schema.Boolean),
})

export const SwitchPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  checked: Schema.optional(Schema.Boolean),
  defaultChecked: Schema.optional(Schema.Boolean),
  disabled: Schema.optional(Schema.Boolean),
})

export const SliderPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  min: Schema.optional(Schema.Number).pipe(Schema.withDefault(() => 0)),
  max: Schema.optional(Schema.Number).pipe(Schema.withDefault(() => 100)),
  step: Schema.optional(Schema.Number).pipe(Schema.withDefault(() => 1)),
  defaultValue: Schema.optional(Schema.Number),
  disabled: Schema.optional(Schema.Boolean),
  showValue: Schema.optional(Schema.Boolean),
})

// Type exports
export type ButtonProps = Schema.Schema.Type<typeof ButtonPropsSchema>
export type TextFieldProps = Schema.Schema.Type<typeof TextFieldPropsSchema>
export type TextAreaProps = Schema.Schema.Type<typeof TextAreaPropsSchema>
export type SelectOption = Schema.Schema.Type<typeof SelectOptionSchema>
export type SelectProps = Schema.Schema.Type<typeof SelectPropsSchema>
export type CheckboxProps = Schema.Schema.Type<typeof CheckboxPropsSchema>
export type RadioGroupProps = Schema.Schema.Type<typeof RadioGroupPropsSchema>
export type SwitchProps = Schema.Schema.Type<typeof SwitchPropsSchema>
export type SliderProps = Schema.Schema.Type<typeof SliderPropsSchema>
```

### 3.4 Feedback Components

```typescript
// @punk/core/schema/props/feedback.ts
import { Schema } from '@effect/schema'

export const DialogPropsSchema = Schema.Struct({
  open: Schema.optional(Schema.Boolean),
  modal: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  title: Schema.optional(Schema.String),
  description: Schema.optional(Schema.String),
})

export const AlertDialogPropsSchema = Schema.Struct({
  open: Schema.optional(Schema.Boolean),
  title: Schema.String,
  description: Schema.String,
  confirmLabel: Schema.optional(Schema.String).pipe(
    Schema.withDefault(() => 'Confirm')
  ),
  cancelLabel: Schema.optional(Schema.String).pipe(
    Schema.withDefault(() => 'Cancel')
  ),
  variant: Schema.optional(Schema.Literal('default', 'destructive')),
})

export const ToastVariantSchema = Schema.Literal(
  'default', 'success', 'error', 'warning', 'info'
)

export const ToastPropsSchema = Schema.Struct({
  title: Schema.String,
  description: Schema.optional(Schema.String),
  variant: Schema.optional(ToastVariantSchema).pipe(
    Schema.withDefault(() => 'default' as const)
  ),
  duration: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 5000)
  ),
  action: Schema.optional(Schema.Struct({
    label: Schema.String,
    onClick: Schema.optional(Schema.String),
  })),
})

export const TooltipPropsSchema = Schema.Struct({
  content: Schema.String,
  side: Schema.optional(Schema.Literal('top', 'right', 'bottom', 'left')).pipe(
    Schema.withDefault(() => 'top' as const)
  ),
  align: Schema.optional(Schema.Literal('start', 'center', 'end')).pipe(
    Schema.withDefault(() => 'center' as const)
  ),
  delayDuration: Schema.optional(Schema.Number),
})

export const PopoverPropsSchema = Schema.Struct({
  open: Schema.optional(Schema.Boolean),
  side: Schema.optional(Schema.Literal('top', 'right', 'bottom', 'left')),
  align: Schema.optional(Schema.Literal('start', 'center', 'end')),
})

export const AlertPropsSchema = Schema.Struct({
  title: Schema.optional(Schema.String),
  description: Schema.String,
  variant: Schema.optional(Schema.Literal(
    'default', 'success', 'error', 'warning', 'info'
  )),
  icon: Schema.optional(Schema.String),
  dismissible: Schema.optional(Schema.Boolean),
})

// Type exports
export type DialogProps = Schema.Schema.Type<typeof DialogPropsSchema>
export type AlertDialogProps = Schema.Schema.Type<typeof AlertDialogPropsSchema>
export type ToastProps = Schema.Schema.Type<typeof ToastPropsSchema>
export type TooltipProps = Schema.Schema.Type<typeof TooltipPropsSchema>
export type PopoverProps = Schema.Schema.Type<typeof PopoverPropsSchema>
export type AlertProps = Schema.Schema.Type<typeof AlertPropsSchema>
```

### 3.5 Navigation Components

```typescript
// @punk/core/schema/props/navigation.ts
import { Schema } from '@effect/schema'

export const TabSchema = Schema.Struct({
  id: Schema.String,
  label: Schema.String,
  disabled: Schema.optional(Schema.Boolean),
  icon: Schema.optional(Schema.String),
})

export const TabsPropsSchema = Schema.Struct({
  tabs: Schema.Array(TabSchema),
  defaultValue: Schema.optional(Schema.String),
  orientation: Schema.optional(Schema.Literal('horizontal', 'vertical')),
})

export const MenuItemSchema = Schema.Struct({
  id: Schema.String,
  label: Schema.String,
  icon: Schema.optional(Schema.String),
  shortcut: Schema.optional(Schema.String),
  disabled: Schema.optional(Schema.Boolean),
  destructive: Schema.optional(Schema.Boolean),
})

export const MenuPropsSchema = Schema.Struct({
  items: Schema.Array(MenuItemSchema),
  label: Schema.optional(Schema.String),
})

export const AccordionItemSchema = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  content: Schema.String,
  disabled: Schema.optional(Schema.Boolean),
})

export const AccordionPropsSchema = Schema.Struct({
  items: Schema.Array(AccordionItemSchema),
  type: Schema.optional(Schema.Literal('single', 'multiple')).pipe(
    Schema.withDefault(() => 'single' as const)
  ),
  collapsible: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  defaultValue: Schema.optional(Schema.Union(
    Schema.String,
    Schema.Array(Schema.String)
  )),
})

// Type exports
export type Tab = Schema.Schema.Type<typeof TabSchema>
export type TabsProps = Schema.Schema.Type<typeof TabsPropsSchema>
export type MenuItem = Schema.Schema.Type<typeof MenuItemSchema>
export type MenuProps = Schema.Schema.Type<typeof MenuPropsSchema>
export type AccordionItem = Schema.Schema.Type<typeof AccordionItemSchema>
export type AccordionProps = Schema.Schema.Type<typeof AccordionPropsSchema>
```

### 3.6 Data Display Components

```typescript
// @punk/core/schema/props/data-display.ts
import { Schema } from '@effect/schema'

export const TableColumnSchema = Schema.Struct({
  key: Schema.String,
  header: Schema.String,
  sortable: Schema.optional(Schema.Boolean),
  width: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  align: Schema.optional(Schema.Literal('left', 'center', 'right')),
})

export const TablePropsSchema = Schema.Struct({
  columns: Schema.Array(TableColumnSchema),
  data: Schema.Array(Schema.Record({
    key: Schema.String,
    value: Schema.Unknown,
  })),
  pagination: Schema.optional(Schema.Boolean),
  pageSize: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 10)
  ),
  sortable: Schema.optional(Schema.Boolean),
  striped: Schema.optional(Schema.Boolean),
})

export const AvatarPropsSchema = Schema.Struct({
  src: Schema.optional(Schema.String),
  alt: Schema.optional(Schema.String),
  fallback: Schema.optional(Schema.String),
  size: Schema.optional(Schema.Literal('xs', 'sm', 'md', 'lg', 'xl')),
})

export const BadgePropsSchema = Schema.Struct({
  label: Schema.String,
  variant: Schema.optional(Schema.Literal(
    'default', 'secondary', 'success', 'warning', 'error', 'outline'
  )),
  size: Schema.optional(Schema.Literal('sm', 'md', 'lg')),
})

export const ProgressPropsSchema = Schema.Struct({
  value: Schema.Number.pipe(
    Schema.greaterThanOrEqualTo(0),
    Schema.lessThanOrEqualTo(100)
  ),
  max: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 100)
  ),
  label: Schema.optional(Schema.String),
  showValue: Schema.optional(Schema.Boolean),
  variant: Schema.optional(Schema.Literal('default', 'success', 'warning', 'error')),
})

// Type exports
export type TableColumn = Schema.Schema.Type<typeof TableColumnSchema>
export type TableProps = Schema.Schema.Type<typeof TablePropsSchema>
export type AvatarProps = Schema.Schema.Type<typeof AvatarPropsSchema>
export type BadgeProps = Schema.Schema.Type<typeof BadgePropsSchema>
export type ProgressProps = Schema.Schema.Type<typeof ProgressPropsSchema>
```

### 3.7 Media Components

```typescript
// @punk/core/schema/props/media.ts
import { Schema } from '@effect/schema'

export const ImagePropsSchema = Schema.Struct({
  src: Schema.String,
  alt: Schema.String,
  width: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  height: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
  objectFit: Schema.optional(Schema.Literal('contain', 'cover', 'fill', 'none', 'scale-down')),
  loading: Schema.optional(Schema.Literal('eager', 'lazy')),
  fallback: Schema.optional(Schema.String),
})

export const IconPropsSchema = Schema.Struct({
  name: Schema.String,
  size: Schema.optional(Schema.Union(Schema.String, Schema.Number)).pipe(
    Schema.withDefault(() => 24)
  ),
  color: Schema.optional(Schema.String),
  strokeWidth: Schema.optional(Schema.Number),
})

// Type exports
export type ImageProps = Schema.Schema.Type<typeof ImagePropsSchema>
export type IconProps = Schema.Schema.Type<typeof IconPropsSchema>
```

---

## 4. Extended Rig Schemas

### 4.1 Chart Rig

```typescript
// @punk/rigs/chart/schema.ts
import { Schema } from '@effect/schema'

export const ChartTypeSchema = Schema.Literal(
  'bar', 'line', 'pie', 'doughnut', 'radar', 'area', 'scatter'
)

export const ChartDataPointSchema = Schema.Struct({
  label: Schema.String,
  value: Schema.Number,
  color: Schema.optional(Schema.String),
})

export const ChartPropsSchema = Schema.Struct({
  chartType: ChartTypeSchema,
  title: Schema.optional(Schema.String),
  data: Schema.Array(ChartDataPointSchema).pipe(
    Schema.minItems(1)
  ),
  showLegend: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  legendPosition: Schema.optional(Schema.Literal('top', 'bottom', 'left', 'right')),
  showGrid: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  aspectRatio: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 2)
  ),
})

export type ChartType = Schema.Schema.Type<typeof ChartTypeSchema>
export type ChartDataPoint = Schema.Schema.Type<typeof ChartDataPointSchema>
export type ChartProps = Schema.Schema.Type<typeof ChartPropsSchema>
```

### 4.2 DataTable Rig

```typescript
// @punk/rigs/data-table/schema.ts
import { Schema } from '@effect/schema'

export const DataTableColumnSchema = Schema.Struct({
  key: Schema.String,
  header: Schema.String,
  sortable: Schema.optional(Schema.Boolean),
  filterable: Schema.optional(Schema.Boolean),
  width: Schema.optional(Schema.Number),
  align: Schema.optional(Schema.Literal('left', 'center', 'right')),
  format: Schema.optional(Schema.Literal('text', 'number', 'date', 'currency', 'percent')),
})

export const DataTablePropsSchema = Schema.Struct({
  columns: Schema.Array(DataTableColumnSchema).pipe(
    Schema.minItems(1)
  ),
  data: Schema.Array(Schema.Record({
    key: Schema.String,
    value: Schema.Unknown,
  })),
  pagination: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  pageSize: Schema.optional(Schema.Literal(10, 25, 50, 100)).pipe(
    Schema.withDefault(() => 10 as const)
  ),
  sortable: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  filterable: Schema.optional(Schema.Boolean),
  selectable: Schema.optional(Schema.Boolean),
  striped: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
})

export type DataTableColumn = Schema.Schema.Type<typeof DataTableColumnSchema>
export type DataTableProps = Schema.Schema.Type<typeof DataTablePropsSchema>
```

### 4.3 RichText Rig

```typescript
// @punk/rigs/rich-text/schema.ts
import { Schema } from '@effect/schema'

export const RichTextFeaturesSchema = Schema.Struct({
  bold: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  italic: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  underline: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  strikethrough: Schema.optional(Schema.Boolean),
  code: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  link: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  lists: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  headings: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  quote: Schema.optional(Schema.Boolean).pipe(Schema.withDefault(() => true)),
  codeBlock: Schema.optional(Schema.Boolean),
  image: Schema.optional(Schema.Boolean),
  table: Schema.optional(Schema.Boolean),
  mention: Schema.optional(Schema.Boolean),
})

export const RichTextPropsSchema = Schema.Struct({
  initialContent: Schema.optional(Schema.String),
  placeholder: Schema.optional(Schema.String).pipe(
    Schema.withDefault(() => 'Start typing...')
  ),
  editable: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  features: Schema.optional(RichTextFeaturesSchema),
  minHeight: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 200)
  ),
  maxHeight: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 500)
  ),
})

export type RichTextFeatures = Schema.Schema.Type<typeof RichTextFeaturesSchema>
export type RichTextProps = Schema.Schema.Type<typeof RichTextPropsSchema>
```

### 4.4 DatePicker Rig

```typescript
// @punk/rigs/date-picker/schema.ts
import { Schema } from '@effect/schema'

// ISO date string pattern
const ISODatePattern = /^\d{4}-\d{2}-\d{2}$/

export const DatePickerPropsSchema = Schema.Struct({
  name: Schema.String,
  label: Schema.optional(Schema.String),
  value: Schema.optional(Schema.String.pipe(Schema.pattern(ISODatePattern))),
  defaultValue: Schema.optional(Schema.String.pipe(Schema.pattern(ISODatePattern))),
  minDate: Schema.optional(Schema.String.pipe(Schema.pattern(ISODatePattern))),
  maxDate: Schema.optional(Schema.String.pipe(Schema.pattern(ISODatePattern))),
  disabled: Schema.optional(Schema.Boolean),
  required: Schema.optional(Schema.Boolean),
  placeholder: Schema.optional(Schema.String),
  format: Schema.optional(Schema.Literal('iso', 'us', 'eu')).pipe(
    Schema.withDefault(() => 'iso' as const)
  ),
})

export type DatePickerProps = Schema.Schema.Type<typeof DatePickerPropsSchema>
```

### 4.5 Calendar Rig

```typescript
// @punk/rigs/calendar/schema.ts
import { Schema } from '@effect/schema'

export const CalendarEventSchema = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  start: Schema.String, // ISO datetime
  end: Schema.optional(Schema.String),
  allDay: Schema.optional(Schema.Boolean),
  color: Schema.optional(Schema.String),
  description: Schema.optional(Schema.String),
})

export const CalendarPropsSchema = Schema.Struct({
  events: Schema.Array(CalendarEventSchema),
  view: Schema.optional(Schema.Literal(
    'dayGridMonth', 'timeGridWeek', 'timeGridDay', 'listWeek'
  )).pipe(Schema.withDefault(() => 'dayGridMonth' as const)),
  editable: Schema.optional(Schema.Boolean),
  selectable: Schema.optional(Schema.Boolean),
  weekends: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  firstDay: Schema.optional(Schema.Literal(0, 1)).pipe(
    Schema.withDefault(() => 0 as const) // Sunday
  ),
  height: Schema.optional(Schema.Union(
    Schema.Literal('auto'),
    Schema.Number
  )),
})

export type CalendarEvent = Schema.Schema.Type<typeof CalendarEventSchema>
export type CalendarProps = Schema.Schema.Type<typeof CalendarPropsSchema>
```

### 4.6 Code Editor Rig

```typescript
// @punk/rigs/code-editor/schema.ts
import { Schema } from '@effect/schema'

export const CodeEditorPropsSchema = Schema.Struct({
  code: Schema.String,
  language: Schema.optional(Schema.String).pipe(
    Schema.withDefault(() => 'typescript')
  ),
  theme: Schema.optional(Schema.Literal('vs-dark', 'vs-light', 'hc-black')).pipe(
    Schema.withDefault(() => 'vs-dark' as const)
  ),
  readOnly: Schema.optional(Schema.Boolean),
  lineNumbers: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => true)
  ),
  minimap: Schema.optional(Schema.Boolean),
  wordWrap: Schema.optional(Schema.Literal('on', 'off', 'wordWrapColumn')),
  fontSize: Schema.optional(Schema.Number).pipe(
    Schema.withDefault(() => 14)
  ),
  height: Schema.optional(Schema.Union(Schema.String, Schema.Number)),
})

export type CodeEditorProps = Schema.Schema.Type<typeof CodeEditorPropsSchema>
```

---

## 5. ID Schemas

### 5.1 ULID Validation

```typescript
// @punk/id/schema.ts
import { Schema } from '@effect/schema'

/**
 * ULID alphabet (Crockford's Base32)
 */
const ULID_ALPHABET = '0123456789ABCDEFGHJKMNPQRSTVWXYZ'
const ULID_REGEX = /^[0-9A-HJKMNP-TV-Z]{26}$/i

/**
 * Validates a string is a valid ULID
 */
const isValidUlid = (id: string): boolean => {
  if (id.length !== 26) return false
  return ULID_REGEX.test(id)
}

/**
 * Base ULID schema with validation
 */
export const UlidSchema = Schema.String.pipe(
  Schema.filter(isValidUlid, {
    message: () => 'Invalid ULID format',
  })
)

/**
 * Branded ULID types for different entity types
 */
export const SchemaIdSchema = UlidSchema.pipe(Schema.brand('SchemaId'))
export const RevisionIdSchema = UlidSchema.pipe(Schema.brand('RevisionId'))
export const ProjectIdSchema = UlidSchema.pipe(Schema.brand('ProjectId'))
export const UserIdSchema = UlidSchema.pipe(Schema.brand('UserId'))
export const ModIdSchema = UlidSchema.pipe(Schema.brand('ModId'))
export const RigIdSchema = UlidSchema.pipe(Schema.brand('RigId'))
export const AssetIdSchema = UlidSchema.pipe(Schema.brand('AssetId'))

// Type exports
export type Ulid = Schema.Schema.Type<typeof UlidSchema>
export type SchemaId = Schema.Schema.Type<typeof SchemaIdSchema>
export type RevisionId = Schema.Schema.Type<typeof RevisionIdSchema>
export type ProjectId = Schema.Schema.Type<typeof ProjectIdSchema>
export type UserId = Schema.Schema.Type<typeof UserIdSchema>
export type ModId = Schema.Schema.Type<typeof ModIdSchema>
export type RigId = Schema.Schema.Type<typeof RigIdSchema>
export type AssetId = Schema.Schema.Type<typeof AssetIdSchema>
```

---

## 6. Mod Schemas

### 6.1 Manifest Schema

```typescript
// @punk/mods/schema.ts
import { Schema } from '@effect/schema'

export const ModAuthorSchema = Schema.Struct({
  name: Schema.String,
  email: Schema.optional(Schema.String.pipe(Schema.pattern(/@/))),
  url: Schema.optional(Schema.String),
})

export const ModCapabilitiesSchema = Schema.Struct({
  requires: Schema.optional(Schema.Array(Schema.String)),
  provides: Schema.Array(Schema.String),
})

export const ModConfigFieldTypeSchema = Schema.Literal(
  'string', 'number', 'boolean', 'array', 'object', 'secret'
)

export const ModConfigFieldSchema = Schema.Struct({
  type: ModConfigFieldTypeSchema,
  description: Schema.optional(Schema.String),
  default: Schema.optional(Schema.Unknown),
  required: Schema.optional(Schema.Boolean),
  secret: Schema.optional(Schema.Boolean),
  enum: Schema.optional(Schema.Array(Schema.String)),
  pattern: Schema.optional(Schema.String),
})

export const ModNetworkSchema = Schema.Struct({
  allowlist: Schema.Array(Schema.String),
})

export const ModTypeSchema = Schema.Literal('backend', 'runtime', 'agent')

export const ModTierSchema = Schema.Literal(
  'free', 'starter', 'pro', 'enterprise'
)

export const ModManifestSchema = Schema.Struct({
  $schema: Schema.optional(Schema.String),
  id: Schema.String.pipe(Schema.pattern(/^[a-z0-9-]+$/)),
  name: Schema.String,
  version: Schema.String.pipe(Schema.pattern(/^\d+\.\d+\.\d+$/)),
  description: Schema.String,
  author: ModAuthorSchema,
  license: Schema.optional(Schema.String),
  repository: Schema.optional(Schema.String),
  
  type: ModTypeSchema,
  
  capabilities: ModCapabilitiesSchema,
  config: Schema.optional(Schema.Record({
    key: Schema.String,
    value: ModConfigFieldSchema,
  })),
  network: Schema.optional(ModNetworkSchema),
  
  tier: Schema.optional(ModTierSchema),
  
  entry: Schema.String,
  templates: Schema.optional(Schema.Record({
    key: Schema.String,
    value: Schema.String,
  })),
  knowledge: Schema.optional(Schema.Array(Schema.String)),
  wasm: Schema.optional(Schema.Array(Schema.String)),
})

export type ModAuthor = Schema.Schema.Type<typeof ModAuthorSchema>
export type ModCapabilities = Schema.Schema.Type<typeof ModCapabilitiesSchema>
export type ModConfigField = Schema.Schema.Type<typeof ModConfigFieldSchema>
export type ModNetwork = Schema.Schema.Type<typeof ModNetworkSchema>
export type ModType = Schema.Schema.Type<typeof ModTypeSchema>
export type ModTier = Schema.Schema.Type<typeof ModTierSchema>
export type ModManifest = Schema.Schema.Type<typeof ModManifestSchema>
```

### 6.2 Rig Manifest Schema

```typescript
// @punk/rigs/schema.ts
import { Schema } from '@effect/schema'

export const RigTierSchema = Schema.Literal('free', 'starter', 'pro', 'premium')

export const RigManifestSchema = Schema.Struct({
  name: Schema.String,
  displayName: Schema.String,
  version: Schema.String.pipe(Schema.pattern(/^\d+\.\d+\.\d+$/)),
  source: Schema.String, // npm package@version
  component: Schema.String,
  category: Schema.String,
  
  tier: RigTierSchema,
  purchasable: Schema.optional(Schema.Number), // cents
  price: Schema.optional(Schema.Number), // cents, for premium
  
  files: Schema.Struct({
    entry: Schema.String,
    schema: Schema.String,
    puck: Schema.String,
    knowledge: Schema.String,
  }),
  
  dependencies: Schema.optional(Schema.Record({
    key: Schema.String,
    value: Schema.String,
  })),
  
  generatedAt: Schema.String,
  punkVersion: Schema.String,
})

export type RigTier = Schema.Schema.Type<typeof RigTierSchema>
export type RigManifest = Schema.Schema.Type<typeof RigManifestSchema>
```

---

## 7. Project Schemas

### 7.1 Project

```typescript
// @punk/core/schema/project.ts
import { Schema } from '@effect/schema'
import { ProjectIdSchema } from '@punk/id/schema'
import { PunkDocumentSchema } from './punk'

export const BackendTypeSchema = Schema.Literal(
  'none', 'glyphcase', 'neon', 'turso', 'planetscale', 'supabase'
)

export const ProjectSettingsSchema = Schema.Struct({
  theme: Schema.optional(Schema.String),
  backend: Schema.optional(BackendTypeSchema),
  mods: Schema.optional(Schema.Array(Schema.String)),
  customDomain: Schema.optional(Schema.String),
  analytics: Schema.optional(Schema.Boolean),
})

export const ProjectSchema = Schema.Struct({
  id: ProjectIdSchema,
  name: Schema.String.pipe(
    Schema.minLength(1),
    Schema.maxLength(100)
  ),
  schema: PunkDocumentSchema,
  settings: ProjectSettingsSchema.pipe(
    Schema.withDefault(() => ({}))
  ),
  public: Schema.optional(Schema.Boolean).pipe(
    Schema.withDefault(() => false)
  ),
  createdAt: Schema.optional(Schema.String),
  updatedAt: Schema.optional(Schema.String),
})

export type BackendType = Schema.Schema.Type<typeof BackendTypeSchema>
export type ProjectSettings = Schema.Schema.Type<typeof ProjectSettingsSchema>
export type Project = Schema.Schema.Type<typeof ProjectSchema>
```

### 7.2 Revision

```typescript
// @punk/core/schema/revision.ts
import { Schema } from '@effect/schema'
import { RevisionIdSchema, ProjectIdSchema } from '@punk/id/schema'
import { PunkDocumentSchema } from './punk'

export const RevisionSchema = Schema.Struct({
  id: RevisionIdSchema,
  projectId: ProjectIdSchema,
  schema: PunkDocumentSchema,
  prompt: Schema.optional(Schema.String),
  parentId: Schema.optional(RevisionIdSchema),
  createdAt: Schema.optional(Schema.String),
})

export type Revision = Schema.Schema.Type<typeof RevisionSchema>
```

---

## 8. AI Context Schemas

### 8.1 Selection Context

```typescript
// @punk/core/schema/ai-context.ts
import { Schema } from '@effect/schema'

export const PropTypeSchema = Schema.Literal(
  'string', 'number', 'boolean', 'enum', 'array', 'object', 'slot'
)

export const PropCapabilitySchema = Schema.Struct({
  name: Schema.String,
  type: PropTypeSchema,
  current: Schema.Unknown,
  options: Schema.optional(Schema.Array(Schema.Unknown)),
  required: Schema.optional(Schema.Boolean),
  description: Schema.optional(Schema.String),
})

export const SelectedComponentSchema = Schema.Struct({
  id: Schema.String,
  type: Schema.String,
  props: Schema.Record({
    key: Schema.String,
    value: Schema.Unknown,
  }),
  path: Schema.optional(Schema.Array(Schema.Number)),
})

export const AIContextSchema = Schema.Struct({
  selected: SelectedComponentSchema,
  capabilities: Schema.Struct({
    props: Schema.Array(PropCapabilitySchema),
    canHaveChildren: Schema.Boolean,
    canBeDeleted: Schema.Boolean,
    canBeMoved: Schema.Boolean,
  }),
  siblings: Schema.optional(Schema.Array(Schema.Struct({
    id: Schema.String,
    type: Schema.String,
  }))),
  parent: Schema.optional(Schema.Struct({
    id: Schema.String,
    type: Schema.String,
  })),
})

export type PropType = Schema.Schema.Type<typeof PropTypeSchema>
export type PropCapability = Schema.Schema.Type<typeof PropCapabilitySchema>
export type SelectedComponent = Schema.Schema.Type<typeof SelectedComponentSchema>
export type AIContext = Schema.Schema.Type<typeof AIContextSchema>
```

### 8.2 Ska Response Schema

```typescript
// @punk/ska/schema.ts
import { Schema } from '@effect/schema'
import { PunkDocumentSchema } from '@punk/core/schema/punk'

export const SkaUpgradeSuggestionSchema = Schema.Struct({
  current: Schema.String,
  suggested: Schema.String,
  reason: Schema.String,
  access: Schema.Struct({
    tier: Schema.optional(Schema.String),
    purchasable: Schema.optional(Schema.Number),
    premium: Schema.optional(Schema.Number),
  }),
})

export const SkaResponseSchema = Schema.Struct({
  document: PunkDocumentSchema,
  usage: Schema.optional(Schema.Struct({
    promptTokens: Schema.Number,
    completionTokens: Schema.Number,
    totalTokens: Schema.Number,
  })),
  model: Schema.String,
  timestamp: Schema.String,
  upgradeSuggestions: Schema.optional(Schema.Array(SkaUpgradeSuggestionSchema)),
})

export type SkaUpgradeSuggestion = Schema.Schema.Type<typeof SkaUpgradeSuggestionSchema>
export type SkaResponse = Schema.Schema.Type<typeof SkaResponseSchema>
```

---

## 9. API Response Schemas

### 9.1 Paginated Response

```typescript
// @punk/api/schema/responses.ts
import { Schema } from '@effect/schema'

/**
 * Generic paginated response factory
 */
export const PaginatedResponseSchema = <A, I, R>(itemSchema: Schema.Schema<A, I, R>) =>
  Schema.Struct({
    items: Schema.Array(itemSchema),
    cursor: Schema.NullOr(Schema.String),
    hasMore: Schema.Boolean,
    total: Schema.optional(Schema.Number),
  })
```

### 9.2 Error Response

```typescript
// @punk/api/schema/errors.ts
import { Schema } from '@effect/schema'

export const ErrorCodeSchema = Schema.Literal(
  'BAD_REQUEST',
  'UNAUTHORIZED',
  'FORBIDDEN',
  'NOT_FOUND',
  'CONFLICT',
  'RATE_LIMITED',
  'INTERNAL_ERROR',
  'VALIDATION_ERROR',
  'SCHEMA_ERROR',
)

export const ErrorResponseSchema = Schema.Struct({
  error: Schema.Struct({
    code: ErrorCodeSchema,
    message: Schema.String,
    details: Schema.optional(Schema.Record({
      key: Schema.String,
      value: Schema.Unknown,
    })),
    path: Schema.optional(Schema.String),
    timestamp: Schema.optional(Schema.String),
  }),
})

export type ErrorCode = Schema.Schema.Type<typeof ErrorCodeSchema>
export type ErrorResponse = Schema.Schema.Type<typeof ErrorResponseSchema>
```

### 9.3 Success Response

```typescript
// @punk/api/schema/success.ts
import { Schema } from '@effect/schema'

/**
 * Generic success response factory
 */
export const SuccessResponseSchema = <A, I, R>(dataSchema: Schema.Schema<A, I, R>) =>
  Schema.Struct({
    success: Schema.Literal(true),
    data: dataSchema,
  })
```

---

## 10. SQL Schemas

### 10.1 GlyphCase (SQLite)

```sql
-- Projects
CREATE TABLE IF NOT EXISTS projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  schema TEXT NOT NULL,
  settings TEXT DEFAULT '{}',
  public INTEGER DEFAULT 0,
  created_at TEXT GENERATED ALWAYS AS (
    datetime(substr(id, 1, 10), 'unixepoch')
  ) VIRTUAL
);

-- Revisions
CREATE TABLE IF NOT EXISTS revisions (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  schema TEXT NOT NULL,
  prompt TEXT,
  parent_id TEXT,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE,
  FOREIGN KEY (parent_id) REFERENCES revisions(id)
);

-- Installed mods
CREATE TABLE IF NOT EXISTS mods (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  manifest TEXT NOT NULL,
  enabled INTEGER DEFAULT 1,
  config TEXT DEFAULT '{}',
  installed_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Mod data
CREATE TABLE IF NOT EXISTS mod_data (
  mod_id TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (mod_id, key),
  FOREIGN KEY (mod_id) REFERENCES mods(id) ON DELETE CASCADE
);

-- Installed rigs
CREATE TABLE IF NOT EXISTS rigs (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  version TEXT NOT NULL,
  manifest TEXT NOT NULL,
  tier TEXT NOT NULL,
  purchased INTEGER DEFAULT 0,
  installed_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- Preferences
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

-- Assets
CREATE TABLE IF NOT EXISTS assets (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  name TEXT NOT NULL,
  mime_type TEXT NOT NULL,
  size INTEGER NOT NULL,
  data BLOB,
  path TEXT,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_revisions_project ON revisions(project_id);
CREATE INDEX IF NOT EXISTS idx_revisions_parent ON revisions(parent_id);
CREATE INDEX IF NOT EXISTS idx_mod_data_mod ON mod_data(mod_id);
CREATE INDEX IF NOT EXISTS idx_assets_project ON assets(project_id);
CREATE INDEX IF NOT EXISTS idx_rigs_tier ON rigs(tier);
```

### 10.2 Mohawk SaaS (PostgreSQL)

```sql
-- Users
CREATE TABLE users (
  id CHAR(26) PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  avatar_url TEXT,
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'starter', 'pro', 'enterprise')),
  stripe_customer_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- User purchases
CREATE TABLE user_purchases (
  id CHAR(26) PRIMARY KEY,
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  item_type TEXT NOT NULL CHECK (item_type IN ('rig', 'mod', 'template')),
  item_id TEXT NOT NULL,
  price_cents INTEGER NOT NULL,
  purchased_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (user_id, item_type, item_id)
);

-- Projects
CREATE TABLE projects (
  id CHAR(26) PRIMARY KEY,
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  schema JSONB NOT NULL,
  settings JSONB DEFAULT '{}',
  public BOOLEAN DEFAULT FALSE,
  slug TEXT UNIQUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Revisions
CREATE TABLE revisions (
  id CHAR(26) PRIMARY KEY,
  project_id CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  schema JSONB NOT NULL,
  prompt TEXT,
  parent_id CHAR(26) REFERENCES revisions(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Team members
CREATE TABLE team_members (
  project_id CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role TEXT DEFAULT 'viewer' CHECK (role IN ('viewer', 'editor', 'admin')),
  invited_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (project_id, user_id)
);

-- Assets
CREATE TABLE assets (
  id CHAR(26) PRIMARY KEY,
  project_id CHAR(26) NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id CHAR(26) NOT NULL REFERENCES users(id),
  name TEXT NOT NULL,
  mime_type TEXT NOT NULL,
  size INTEGER NOT NULL,
  url TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- API keys
CREATE TABLE api_keys (
  id CHAR(26) PRIMARY KEY,
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  key_hash TEXT NOT NULL,
  last_used_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usage tracking
CREATE TABLE usage (
  id CHAR(26) PRIMARY KEY,
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  action TEXT NOT NULL,
  tokens_used INTEGER DEFAULT 0,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_projects_user ON projects(user_id);
CREATE INDEX idx_projects_slug ON projects(slug) WHERE slug IS NOT NULL;
CREATE INDEX idx_revisions_project ON revisions(project_id);
CREATE INDEX idx_revisions_parent ON revisions(parent_id);
CREATE INDEX idx_assets_project ON assets(project_id);
CREATE INDEX idx_usage_user ON usage(user_id);
CREATE INDEX idx_usage_created ON usage(created_at);
CREATE INDEX idx_user_purchases_user ON user_purchases(user_id);
```

---

## 11. Schema Registry

```typescript
// @punk/core/schema/registry.ts
import { Schema } from '@effect/schema'

// Import all prop schemas
import { FlexPropsSchema, GridPropsSchema, ContainerPropsSchema } from './props/layout'
import { ButtonPropsSchema, TextFieldPropsSchema, SelectPropsSchema } from './props/form'
import { DialogPropsSchema, ToastPropsSchema, TooltipPropsSchema } from './props/feedback'
import { ChartPropsSchema } from '@punk/rigs/chart/schema'
import { DataTablePropsSchema } from '@punk/rigs/data-table/schema'
import { RichTextPropsSchema } from '@punk/rigs/rich-text/schema'

/**
 * Registry of all component prop schemas
 */
export const ComponentPropsSchemas: Record<string, Schema.Schema<unknown>> = {
  // Layout
  Flex: FlexPropsSchema,
  Grid: GridPropsSchema,
  Container: ContainerPropsSchema,
  
  // Form
  Button: ButtonPropsSchema,
  TextField: TextFieldPropsSchema,
  Select: SelectPropsSchema,
  
  // Feedback
  Dialog: DialogPropsSchema,
  Toast: ToastPropsSchema,
  Tooltip: TooltipPropsSchema,
  
  // Extended Rigs
  Chart: ChartPropsSchema,
  DataTable: DataTablePropsSchema,
  RichText: RichTextPropsSchema,
}

/**
 * Get props schema for a component type
 */
export function getPropsSchema(componentType: string): Schema.Schema<unknown> | null {
  return ComponentPropsSchemas[componentType] ?? null
}

/**
 * Validate component props
 */
export function validateComponentProps(
  componentType: string,
  props: unknown
): Schema.ParseResult<unknown> {
  const schema = getPropsSchema(componentType)
  if (!schema) {
    return Schema.decodeUnknownEither(Schema.Unknown)(props)
  }
  return Schema.decodeUnknownEither(schema)(props)
}

/**
 * Validate entire Punk document
 */
export function validateDocument(document: unknown): Schema.ParseResult<PunkDocument> {
  return Schema.decodeUnknownEither(PunkDocumentSchema)(document)
}
```

---

## 12. Validation Utilities

```typescript
// @punk/core/schema/utils.ts
import { Schema, ParseResult } from '@effect/schema'
import { Either, pipe } from 'effect'

/**
 * Parse with detailed error formatting
 */
export function safeParse<A>(
  schema: Schema.Schema<A>,
  input: unknown
): { success: true; data: A } | { success: false; errors: string[] } {
  const result = Schema.decodeUnknownEither(schema)(input)
  
  return pipe(
    result,
    Either.match({
      onLeft: (error) => ({
        success: false as const,
        errors: formatParseErrors(error),
      }),
      onRight: (data) => ({
        success: true as const,
        data,
      }),
    })
  )
}

/**
 * Format parse errors for display
 */
export function formatParseErrors(error: ParseResult.ParseError): string[] {
  return ParseResult.ArrayFormatter.formatErrorSync(error).errors.map(
    (e) => `${e.path.join('.')}: ${e.message}`
  )
}
```

---

## 13. Migration from Zod

### 13.1 Automated Migration

```bash
# Use codemod for bulk migration
npx @effect/codemod zod-to-effect ./src
```

### 13.2 Common Patterns

```typescript
// Before (Zod)
const UserSchema = z.object({
  name: z.string().min(1),
  age: z.number().optional(),
  role: z.enum(['admin', 'user']),
}).refine(data => data.age === undefined || data.age >= 0)

// After (Effect)
const UserSchema = Schema.Struct({
  name: Schema.String.pipe(Schema.minLength(1)),
  age: Schema.optional(Schema.Number),
  role: Schema.Literal('admin', 'user'),
}).pipe(
  Schema.filter((data) => data.age === undefined || data.age >= 0, {
    message: () => 'Age must be non-negative',
  })
)
```

---

*Last updated: December 2025*
