# Enhanced Error Recovery Implementation Guide

**Version:** 2.0  
**Status:** Source of Truth  
**Last Updated:** December 2025  
**Audience:** Human developers and AI coding assistants (Claude Code, Cursor, etc.)  
**Depends On:** PUNK_EFFECT_IMPL.md, PUNK_SCHEMAS.md  
**Dependents:** MCP_GATEWAY, AGENT_MOD, DESKTOP

---

## Executive Summary

Mohawk never shows users hard errors for AI generation failures. Instead, it uses structured error recovery with full audit trails. Users always get a result â€” and they can see exactly how it was produced.

**Core Principles:**

1. **No hard failures** â€” Every generation returns something usable
2. **Typed errors** â€” Know exactly what failed and why
3. **Targeted recovery** â€” Fix the specific problem, not blind retry
4. **Full audit trail** â€” User sees what happened
5. **Confidence signaling** â€” User knows when to review

---

## The Problem We're Solving

### Before: Hard Failures

```typescript
// Old approach
async function generate(prompt: string) {
  try {
    const result = await ai.generate(prompt)
    const validated = Schema.parse(result)  // Throws on invalid
    return validated
  } catch (e) {
    // What is e? AI error? Validation error? Network error?
    console.error(e)
    throw new Error("Generation failed")  // User sees this
  }
}
```

**User experience:** "Generation failed. Try again."

- No information about what went wrong
- No recovery attempted
- User frustration
- Support tickets

### After: Structured Recovery

```typescript
// Effect approach
const generate = (prompt: string): Effect<GenerationResult, never, AIService> =>
  pipe(
    tryGenerate(prompt),
    Effect.catchTag("SchemaValidationError", (e) => 
      tryGenerateWithRepair(prompt, e.issues)
    ),
    Effect.catchTag("SchemaValidationError", (e) =>
      tryGenerateWithLargerModel(prompt)
    ),
    Effect.catchAll(() => 
      Effect.succeed(getSafeTemplate(prompt))
    ),
    Effect.map(buildResultWithAudit)
  )
```

**User experience:** Working UI + "Generated with 2 attempts" + full debug log

---

## Error Type Definitions

### AI Generation Errors

```typescript
// src/errors/generation.ts
import { Data } from "effect"
import { ParseError } from "@effect/schema"

/**
 * Schema validation failed on AI output.
 * Contains the raw output and structured error details.
 */
export class SchemaValidationError extends Data.TaggedError("SchemaValidationError")<{
  readonly raw: unknown              // What the AI actually returned
  readonly issues: ParseError        // Structured validation errors
  readonly prompt: string            // Original prompt (for retry)
  readonly model: string             // Which model produced this
  readonly attemptNumber: number     // Which attempt this was
}> {}

/**
 * AI model returned an error response.
 */
export class AIModelError extends Data.TaggedError("AIModelError")<{
  readonly model: string
  readonly reason: string
  readonly code?: string
  readonly retryable: boolean
}> {}

/**
 * Rate limited by AI provider.
 */
export class AIRateLimitError extends Data.TaggedError("AIRateLimitError")<{
  readonly model: string
  readonly retryAfter: number        // Seconds to wait
  readonly provider: string
}> {}

/**
 * AI request timed out.
 */
export class AITimeoutError extends Data.TaggedError("AITimeoutError")<{
  readonly model: string
  readonly timeoutMs: number
  readonly prompt: string
}> {}

/**
 * Content was filtered/refused by AI.
 */
export class AIContentFilterError extends Data.TaggedError("AIContentFilterError")<{
  readonly model: string
  readonly reason: string
  readonly prompt: string
}> {}

/**
 * Union of all generation errors.
 */
export type GenerationError =
  | SchemaValidationError
  | AIModelError
  | AIRateLimitError
  | AITimeoutError
  | AIContentFilterError
```

### Error Formatting Utilities

```typescript
// src/errors/formatting.ts
import { ParseResult } from "@effect/schema"

/**
 * Format validation errors for repair prompts.
 * Returns human-readable error descriptions.
 */
export function formatValidationError(error: ParseError): string {
  return ParseResult.TreeFormatter.formatErrorSync(error)
}

/**
 * Extract specific field errors for targeted repair.
 */
export function extractFieldErrors(error: ParseError): FieldError[] {
  const errors: FieldError[] = []
  
  function walk(e: ParseError, path: string[] = []) {
    if (e._tag === "Type") {
      errors.push({
        path: path.join("."),
        expected: String(e.expected),
        actual: e.actual,
        message: e.message ?? `Expected ${e.expected}`
      })
    }
    if (e._tag === "Key" || e._tag === "Index") {
      walk(e.error, [...path, String(e.key)])
    }
    if (e._tag === "Union" || e._tag === "Tuple") {
      e.errors.forEach(err => walk(err, path))
    }
  }
  
  walk(error)
  return errors
}

export interface FieldError {
  path: string           // e.g., "props.variant" or "children[2].type"
  expected: string       // e.g., "'primary' | 'secondary' | 'ghost'"
  actual: unknown        // e.g., "blue"
  message: string        // Human readable
}
```

---

## Result Types

### GenerationResult

Every generation returns this enriched type:

```typescript
// src/types/generation.ts

/**
 * The result of a generation request.
 * Always succeeds â€” contains either clean output or recovered output with audit trail.
 */
export interface GenerationResult<T = UIComponent> {
  /** The generated component/output */
  component: T
  
  /** Debug information about how this was generated */
  debug: GenerationDebug
}

export interface GenerationDebug {
  /** All attempts made during generation */
  attempts: AttemptLog[]
  
  /** Total wall-clock time in milliseconds */
  totalTimeMs: number
  
  /** Which model produced the final result */
  finalModel: string
  
  /** Confidence in the result */
  confidence: ConfidenceLevel
  
  /** Whether any recovery was needed */
  recovered: boolean
  
  /** Summary for UI display */
  summary: string
}

export interface AttemptLog {
  /** Attempt number (1-indexed) */
  attempt: number
  
  /** Model used for this attempt */
  model: string
  
  /** Time taken for this attempt in ms */
  durationMs: number
  
  /** What happened */
  outcome: AttemptOutcome
  
  /** Error details if failed */
  error?: {
    type: string           // Error tag name
    message: string        // Human readable
    details?: unknown      // Raw error for debugging
  }
  
  /** What recovery action was taken (if any) */
  recovery?: {
    strategy: RecoveryStrategy
    description: string
  }
}

export type AttemptOutcome =
  | "success"
  | "validation_error"
  | "model_error"
  | "rate_limited"
  | "timeout"
  | "content_filtered"
  | "fallback_used"

export type RecoveryStrategy =
  | "repair_prompt"           // Retry with error-specific prompt
  | "simplified_prompt"       // Retry with simpler request
  | "model_escalation"        // Try larger/better model
  | "model_fallback"          // Try different model
  | "template_fallback"       // Use safe pre-built template
  | "wait_retry"              // Wait and retry (rate limit)

export type ConfidenceLevel = "high" | "medium" | "low"
```

### Pro+ Extended Result

```typescript
/**
 * Extended result for Pro+ users with full trace access.
 */
export interface ProGenerationResult<T = UIComponent> extends GenerationResult<T> {
  trace: GenerationTrace
}

export interface GenerationTrace {
  /** OpenTelemetry span IDs for correlation */
  spanIds: string[]
  
  /** Exact prompts sent to each model */
  prompts: PromptLog[]
  
  /** Raw model responses before validation */
  rawResponses: RawResponseLog[]
  
  /** Token usage per attempt */
  tokenUsage: TokenUsageLog[]
}

export interface PromptLog {
  attempt: number
  model: string
  systemPrompt?: string
  userPrompt: string
  timestamp: Date
}

export interface RawResponseLog {
  attempt: number
  model: string
  raw: unknown
  timestamp: Date
}

export interface TokenUsageLog {
  attempt: number
  model: string
  inputTokens: number
  outputTokens: number
  cost: number           // In USD
}
```

---

## Recovery Strategies

### Strategy 1: Repair Prompt

When validation fails, create a targeted prompt that explains what went wrong:

```typescript
// src/generation/repair.ts
import { FieldError } from "../errors/formatting"

/**
 * Build a repair prompt based on specific validation errors.
 */
export function buildRepairPrompt(
  originalPrompt: string,
  errors: FieldError[]
): string {
  const errorList = errors.map(e => 
    `- ${e.path}: Expected ${e.expected}, but got ${JSON.stringify(e.actual)}`
  ).join("\n")
  
  const fixes = errors.map(e => {
    if (e.path.includes("variant")) {
      return `- Use exactly one of: "primary", "secondary", "ghost" for variant`
    }
    if (e.path.includes("type")) {
      return `- Ensure every component has a valid "type" field`
    }
    if (e.expected.includes("|")) {
      const options = e.expected.replace(/'/g, '"')
      return `- ${e.path} must be one of: ${options}`
    }
    return `- Fix ${e.path}: ${e.message}`
  }).join("\n")
  
  return `
Your previous output had validation errors. Please fix and regenerate.

ORIGINAL REQUEST:
${originalPrompt}

ERRORS FOUND:
${errorList}

REQUIRED FIXES:
${fixes}

Please output valid JSON that passes all validation. Do not include any explanation, only the JSON.
`.trim()
}
```

### Strategy 2: Simplified Prompt

When repair fails, try a simpler request:

```typescript
/**
 * Build a simplified prompt that constrains complexity.
 */
export function buildSimplifiedPrompt(
  originalPrompt: string,
  context: { maxDepth?: number; safeComponents?: string[] } = {}
): string {
  const { maxDepth = 2, safeComponents = ["card", "button", "text", "input"] } = context
  
  return `
Create a simple UI for: ${originalPrompt}

CONSTRAINTS:
- Use only these component types: ${safeComponents.join(", ")}
- Maximum nesting depth: ${maxDepth} levels
- Keep it minimal and focused
- All props must be simple strings, numbers, or booleans

OUTPUT FORMAT:
{
  "type": "card",
  "props": { "title": "string" },
  "children": [...]
}

Output only valid JSON.
`.trim()
}
```

### Strategy 3: Model Escalation

When smaller models can't handle complexity:

```typescript
// src/generation/models.ts

export const MODEL_HIERARCHY = [
  { id: "ska-30b-vl", tier: 1, costPer1k: 0.002 },
  { id: "ska-106b-vl", tier: 2, costPer1k: 0.016 },
  { id: "ska-235b-vl", tier: 3, costPer1k: 0.013 }
] as const

export type ModelId = typeof MODEL_HIERARCHY[number]["id"]

/**
 * Get next model to try when current fails.
 */
export function getEscalationModel(currentModel: ModelId): ModelId | null {
  const currentTier = MODEL_HIERARCHY.find(m => m.id === currentModel)?.tier ?? 0
  const next = MODEL_HIERARCHY.find(m => m.tier > currentTier)
  return next?.id ?? null
}
```

### Strategy 4: Template Fallback

When all else fails, return a safe template:

```typescript
// src/generation/templates.ts

const TEMPLATES: Record<string, UIComponent> = {
  default: {
    type: "card",
    props: { title: "Generated Content", variant: "primary" },
    children: [
      { type: "text", props: { content: "We couldn't fully generate your request." } }
    ]
  },
  
  form: {
    type: "card",
    props: { title: "Form", variant: "primary" },
    children: [
      { type: "input", props: { label: "Name", placeholder: "Enter name" } },
      { type: "input", props: { label: "Email", placeholder: "Enter email" } },
      { type: "button", props: { label: "Submit", variant: "primary" } }
    ]
  },
  
  dashboard: {
    type: "card",
    props: { title: "Dashboard", variant: "primary" },
    children: [
      { type: "text", props: { content: "Dashboard content will appear here." } }
    ]
  },
  
  landing: {
    type: "card",
    props: { title: "Welcome", variant: "primary" },
    children: [
      { type: "text", props: { content: "Welcome to your new page." } },
      { type: "button", props: { label: "Get Started", variant: "primary" } }
    ]
  }
}

/**
 * Get a safe template based on prompt intent.
 */
export function getSafeTemplate(prompt: string): UIComponent {
  const lower = prompt.toLowerCase()
  
  if (lower.includes("form") || lower.includes("input")) {
    return structuredClone(TEMPLATES.form)
  }
  if (lower.includes("dashboard") || lower.includes("chart")) {
    return structuredClone(TEMPLATES.dashboard)
  }
  if (lower.includes("landing") || lower.includes("hero")) {
    return structuredClone(TEMPLATES.landing)
  }
  
  return structuredClone(TEMPLATES.default)
}
```

---

## Core Implementation

### Generation Service Interface

```typescript
// src/services/generation.ts
import { Context, Effect, Layer, Ref, pipe } from "effect"

export interface GenerationService {
  readonly generate: (
    prompt: string,
    options?: GenerationOptions
  ) => Effect<GenerationResult, never, never>
}

export interface GenerationOptions {
  model?: ModelId
  maxAttempts?: number
  timeoutMs?: number
  includeTrace?: boolean
}

export class GenerationService extends Context.Tag("GenerationService")<
  GenerationService,
  GenerationService
>() {}

export const GenerationServiceLive = Layer.effect(
  GenerationService,
  Effect.gen(function* () {
    const ai = yield* AIService
    
    return {
      generate: (prompt, options = {}) =>
        generateWithFullRecovery(ai, prompt, options)
    }
  })
)
```

### Core Generation Logic

```typescript
const generateWithFullRecovery = (
  ai: AIService,
  prompt: string,
  options: GenerationOptions
): Effect<GenerationResult, never, never> =>
  Effect.gen(function* () {
    const { model: startModel = "ska-30b-vl", timeoutMs = 30000 } = options
    
    const startTime = Date.now()
    const attempts = yield* Ref.make<AttemptLog[]>([])
    
    const recordAttempt = (log: AttemptLog) =>
      Ref.update(attempts, arr => [...arr, log])
    
    const component = yield* pipe(
      // Attempt 1: Primary model
      tryGenerateAttempt({ ai, prompt, model: startModel, attemptNumber: 1, timeoutMs, startTime, recordAttempt }),
      
      // Recovery: Validation error â†’ repair prompt
      Effect.catchTag("SchemaValidationError", (error) =>
        pipe(
          recordAttempt({
            attempt: error.attemptNumber,
            model: error.model,
            durationMs: Date.now() - startTime,
            outcome: "validation_error",
            error: { type: "SchemaValidationError", message: formatValidationError(error.issues) },
            recovery: { strategy: "repair_prompt", description: "Retrying with error-specific repair prompt" }
          }),
          Effect.flatMap(() =>
            tryGenerateAttempt({
              ai,
              prompt: buildRepairPrompt(prompt, extractFieldErrors(error.issues)),
              model: error.model,
              attemptNumber: 2,
              timeoutMs, startTime, recordAttempt
            })
          )
        )
      ),
      
      // Recovery: Still failing â†’ simplified prompt
      Effect.catchTag("SchemaValidationError", (error) =>
        pipe(
          recordAttempt({
            attempt: error.attemptNumber,
            model: error.model,
            durationMs: Date.now() - startTime,
            outcome: "validation_error",
            error: { type: "SchemaValidationError", message: formatValidationError(error.issues) },
            recovery: { strategy: "simplified_prompt", description: "Retrying with simplified constraints" }
          }),
          Effect.flatMap(() =>
            tryGenerateAttempt({
              ai,
              prompt: buildSimplifiedPrompt(prompt),
              model: error.model,
              attemptNumber: 3,
              timeoutMs, startTime, recordAttempt
            })
          )
        )
      ),
      
      // Recovery: Simplified failed â†’ escalate model
      Effect.catchTag("SchemaValidationError", (error) => {
        const nextModel = getEscalationModel(error.model as ModelId)
        
        if (!nextModel) {
          return pipe(
            recordAttempt({
              attempt: error.attemptNumber,
              model: error.model,
              durationMs: Date.now() - startTime,
              outcome: "validation_error",
              error: { type: "SchemaValidationError", message: formatValidationError(error.issues) },
              recovery: { strategy: "template_fallback", description: "Using safe template" }
            }),
            Effect.flatMap(() => Effect.succeed(getSafeTemplate(prompt)))
          )
        }
        
        return pipe(
          recordAttempt({
            attempt: error.attemptNumber,
            model: error.model,
            durationMs: Date.now() - startTime,
            outcome: "validation_error",
            error: { type: "SchemaValidationError", message: formatValidationError(error.issues) },
            recovery: { strategy: "model_escalation", description: `Escalating to ${nextModel}` }
          }),
          Effect.flatMap(() =>
            tryGenerateAttempt({ ai, prompt, model: nextModel, attemptNumber: 4, timeoutMs, startTime, recordAttempt })
          )
        )
      }),
      
      // Recovery: Rate limited â†’ wait and retry
      Effect.catchTag("AIRateLimitError", (error) =>
        pipe(
          recordAttempt({
            attempt: 1,
            model: error.model,
            durationMs: Date.now() - startTime,
            outcome: "rate_limited",
            error: { type: "AIRateLimitError", message: `Rate limited, retry after ${error.retryAfter}s` },
            recovery: { strategy: "wait_retry", description: `Waiting ${error.retryAfter}s` }
          }),
          Effect.flatMap(() => Effect.sleep(`${error.retryAfter} seconds`)),
          Effect.flatMap(() =>
            tryGenerateAttempt({ ai, prompt, model: error.model as ModelId, attemptNumber: 2, timeoutMs, startTime, recordAttempt })
          )
        )
      ),
      
      // Recovery: Any other error â†’ template fallback
      Effect.catchAll((error) =>
        pipe(
          recordAttempt({
            attempt: 1,
            model: startModel,
            durationMs: Date.now() - startTime,
            outcome: "model_error",
            error: { type: error._tag ?? "UnknownError", message: String(error) },
            recovery: { strategy: "template_fallback", description: "Using safe template" }
          }),
          Effect.flatMap(() => Effect.succeed(getSafeTemplate(prompt)))
        )
      )
    )
    
    const attemptLog = yield* Ref.get(attempts)
    return { component, debug: buildDebugInfo(attemptLog, startTime) }
  })
```

### Debug Info Builder

```typescript
function buildDebugInfo(attempts: AttemptLog[], startTime: number): GenerationDebug {
  const successAttempt = attempts.find(a => a.outcome === "success")
  const lastAttempt = attempts[attempts.length - 1]
  const usedFallback = lastAttempt?.recovery?.strategy === "template_fallback"
  
  return {
    attempts,
    totalTimeMs: Date.now() - startTime,
    finalModel: successAttempt?.model ?? lastAttempt?.model ?? "unknown",
    confidence: calculateConfidence(attempts, usedFallback),
    recovered: attempts.length > 1,
    summary: buildSummary(attempts, usedFallback)
  }
}

function calculateConfidence(attempts: AttemptLog[], usedFallback: boolean): ConfidenceLevel {
  if (usedFallback) return "low"
  if (attempts.length === 1 && attempts[0].outcome === "success") return "high"
  if (attempts.length <= 2) return "medium"
  return "low"
}

function buildSummary(attempts: AttemptLog[], usedFallback: boolean): string {
  if (usedFallback) return "Generation failed after all retries. Using safe template."
  if (attempts.length === 1) return "Generated successfully on first attempt."
  
  const strategies = attempts.filter(a => a.recovery).map(a => a.recovery!.strategy)
  if (strategies.includes("model_escalation")) {
    return `Generated after ${attempts.length} attempts. Required model escalation.`
  }
  if (strategies.includes("repair_prompt")) {
    return `Generated after ${attempts.length} attempts. Used repair prompt.`
  }
  return `Generated after ${attempts.length} attempts.`
}
```

---

## UI Integration

### React Component

```tsx
// src/components/GenerationStatus.tsx
export function GenerationStatus({ debug }: { debug: GenerationDebug }) {
  const [expanded, setExpanded] = useState(false)
  
  const badgeConfig = {
    high: { color: "green", icon: "âœ“", label: "Generated" },
    medium: { color: "yellow", icon: "âš ï¸", label: "Generated" },
    low: { color: "orange", icon: "âš ï¸", label: "Generated with fallback" }
  }[debug.confidence]
  
  return (
    <div className="generation-status">
      <button className={`badge badge-${badgeConfig.color}`} onClick={() => setExpanded(!expanded)}>
        <span>{badgeConfig.icon}</span>
        <span>{badgeConfig.label}</span>
        <span>{(debug.totalTimeMs / 1000).toFixed(1)}s</span>
        {debug.recovered && <span>({debug.attempts.length} attempts)</span>}
      </button>
      
      {expanded && (
        <div className="generation-details">
          <h4>Generation Debug</h4>
          {debug.attempts.map((attempt, i) => (
            <div key={i} className={`attempt attempt-${attempt.outcome}`}>
              <div>Attempt {attempt.attempt} ({attempt.model}) - {attempt.durationMs}ms</div>
              {attempt.error && <div className="error">{attempt.error.message}</div>}
              {attempt.recovery && <div className="recovery">â†» {attempt.recovery.description}</div>}
            </div>
          ))}
          <div>Confidence: {debug.confidence} | Model: {debug.finalModel}</div>
        </div>
      )}
    </div>
  )
}
```

---

## Analytics

### Event Tracking

```typescript
export function trackGeneration(prompt: string, debug: GenerationDebug, userId: string): void {
  analytics.track({
    event: "generation_complete",
    userId,
    promptCategory: categorizePrompt(prompt),
    totalTimeMs: debug.totalTimeMs,
    attemptCount: debug.attempts.length,
    confidence: debug.confidence,
    recovered: debug.recovered,
    usedRepairPrompt: debug.attempts.some(a => a.recovery?.strategy === "repair_prompt"),
    usedModelEscalation: debug.attempts.some(a => a.recovery?.strategy === "model_escalation"),
    usedTemplateFallback: debug.attempts.some(a => a.recovery?.strategy === "template_fallback"),
  })
}
```

### Sample Queries

```sql
-- Most common validation errors
SELECT jsonb_array_elements_text(error_patterns) as error_path, COUNT(*) as occurrences
FROM generation_events WHERE validation_errors > 0
GROUP BY error_path ORDER BY occurrences DESC LIMIT 20;

-- Recovery strategy effectiveness
SELECT 
  CASE 
    WHEN used_repair_prompt THEN 'repair_prompt'
    WHEN used_model_escalation THEN 'model_escalation'
    WHEN used_template_fallback THEN 'template_fallback'
    ELSE 'first_try'
  END as recovery_path,
  COUNT(*) as count,
  AVG(total_time_ms) as avg_time_ms
FROM generation_events GROUP BY recovery_path;
```

---

## Configuration

```typescript
// src/config/generation.ts
import { Config } from "effect"

export const GenerationConfig = Config.all({
  defaultModel: Config.string("GENERATION_DEFAULT_MODEL").pipe(Config.withDefault("ska-30b-vl")),
  maxAttempts: Config.integer("GENERATION_MAX_ATTEMPTS").pipe(Config.withDefault(4)),
  timeoutMs: Config.integer("GENERATION_TIMEOUT_MS").pipe(Config.withDefault(30000)),
  enableRepairPrompt: Config.boolean("GENERATION_ENABLE_REPAIR").pipe(Config.withDefault(true)),
  enableModelEscalation: Config.boolean("GENERATION_ENABLE_ESCALATION").pipe(Config.withDefault(true)),
  enableTemplateFallback: Config.boolean("GENERATION_ENABLE_FALLBACK").pipe(Config.withDefault(true)),
})
```

---

## Quick Reference

### Recovery Strategy Priority

```
1. Repair Prompt      â†’ Fix specific validation errors
2. Simplified Prompt  â†’ Reduce complexity  
3. Model Escalation   â†’ Try smarter model
4. Wait and Retry     â†’ Handle rate limits
5. Template Fallback  â†’ Safe default (never fails)
```

### Confidence Mapping

| Attempts | Recovery Used | Confidence |
|----------|---------------|------------|
| 1 | None | High |
| 2 | Repair prompt | Medium |
| 3+ | Any | Low |
| Any | Template fallback | Low |

### User-Facing UI States

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  âœ“ Generated                      0.9s   â”‚  â† High confidence
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜

â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  âš ï¸ Generated (2 attempts)        1.6s   â”‚  â† Medium confidence
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜

â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  âš ï¸ Generated with fallback       2.3s   â”‚  â† Low confidence
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

---

## Related Documents

- **PUNK_EFFECT_IMPL.md** â€” Core Effect patterns
- **PUNK_EFFECT_MIGRATION.md** â€” Migration from Zod/try-catch
- **PUNK_SCHEMAS.md** â€” UI component schemas
- **PUNK_DECISIONS.md** â€” ADR-015 (Effect), ADR-018 (@effect/schema)

---

*Last updated: December 2025*
