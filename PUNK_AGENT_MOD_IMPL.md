# Agent Mod Implementation Specification

**Version:** 3.0  
**Status:** Source of Truth  
**Tier:** Pro+ (not available in Free/Hobbyist)

---

## 1. Overview

Agent Mod adds deterministic multi-agent workflow execution to Mohawk.

| Without Agent Mod | With Agent Mod |
|-------------------|----------------|
| Prompt → Ska → Output | Workflow → Graph execution → Audited output |
| Single turn | Multi-step, resumable |
| No memory | Learns across runs |
| Like v0/Lovable | Like Claude Code workflows |

### Core Philosophy

> "Reproducible, auditable, predictable — like a government process, not a creative brainstorm. Same input → Same output. Every time."

### What It Is NOT

- NOT a replacement for GlyphCase (auth, sync, capsule)
- NOT a replacement for Ska (LLM calls)
- NOT an autonomous agent (deterministic graph, not free-roaming)
- NOT a probabilistic search engine (graph navigation, not vector similarity)

---

## 2. Mod Structure

Agent Mod is a standard Punk Mod (SQLar archive):

```
agent-mod.sqlar
├── manifest.json           # Mod metadata
├── schema.sql              # GlyphCase tables
├── lua/                    # Trinity Lua code
│   ├── graph.lua           # Edge traversal
│   ├── memory.lua          # Memory operations
│   └── context.lua         # Context resolution
├── js/                     # Trinity JS code
│   ├── orchestrator.js     # Main execution loop
│   ├── rag.js              # FTS5 retrieval
│   └── prompt.js           # Template interpolation
├── wasm/                   # Trinity WASM
│   └── validator.wasm      # AJV schema validation
└── docs/                   # Mod documentation
    ├── workflow-authoring.md
    ├── prompt-templates.md
    └── knowledge-authoring.md
```

---

## 3. Manifest

```json
{
  "name": "agent-mod",
  "version": "3.0.0",
  "description": "Deterministic multi-agent workflow execution",
  "tier": "pro",
  "author": "punk",
  
  "capabilities": {
    "requires": [
      "ska:generate",
      "ska:validate"
    ],
    "provides": [
      "workflow:run",
      "workflow:resume",
      "knowledge:index",
      "knowledge:query"
    ]
  },
  
  "runtime": {
    "requires_nitro": true,
    "bridge": []
  },
  
  "schema": "schema.sql",
  "entry": "js/orchestrator.js"
}
```

---

## 4. Database Schema

### File: `schema.sql`

```sql
-- ============================================================
-- AGENT MOD SCHEMA
-- ============================================================

-- ------------------------------------------------------------
-- WORKFLOW DEFINITION
-- ------------------------------------------------------------

CREATE TABLE workflows (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  require_determinism INTEGER DEFAULT 1,  -- 1 = true
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE nodes (
  id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  role TEXT NOT NULL,  -- 'architect', 'designer', 'engineer', etc.
  prompt_path TEXT NOT NULL,  -- Path in user's prompts/ directory
  output_schema TEXT,  -- JSON Schema for validation
  position INTEGER DEFAULT 0,  -- For UI ordering
  
  UNIQUE(workflow_id, name)
);

CREATE TABLE edges (
  id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
  from_node TEXT NOT NULL REFERENCES nodes(id),
  to_node TEXT NOT NULL REFERENCES nodes(id),
  condition TEXT,  -- Lua expression, null = always
  priority INTEGER DEFAULT 0,  -- Higher = evaluated first
  
  UNIQUE(workflow_id, from_node, to_node)
);

CREATE TABLE node_knowledge_subscriptions (
  node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
  knowledge_path TEXT NOT NULL,  -- Explicit path to knowledge doc
  
  PRIMARY KEY (node_id, knowledge_path)
);

CREATE TABLE node_knowledge_tags (
  node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
  tag TEXT NOT NULL,  -- e.g., 'auth', 'api', 'hipaa'
  
  PRIMARY KEY (node_id, tag)
);

-- ------------------------------------------------------------
-- WORKFLOW EXECUTION
-- ------------------------------------------------------------

CREATE TABLE runs (
  id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL REFERENCES workflows(id),
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, running, paused, completed, failed
  global_context TEXT NOT NULL DEFAULT '{}',  -- JSON
  started_at TEXT,
  completed_at TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_runs_workflow ON runs(workflow_id);
CREATE INDEX idx_runs_status ON runs(status);

CREATE TABLE run_steps (
  id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  node_id TEXT NOT NULL REFERENCES nodes(id),
  sequence INTEGER NOT NULL,
  
  -- Input
  input_context TEXT NOT NULL,  -- JSON: context at step start
  resolved_prompt TEXT NOT NULL,  -- Final prompt sent to Ska
  knowledge_sources TEXT NOT NULL DEFAULT '[]',  -- JSON: which docs were injected
  
  -- Output
  output TEXT,  -- Ska's response
  output_valid INTEGER,  -- 1 = passed schema validation
  validation_errors TEXT,  -- JSON array if invalid
  
  -- Audit
  deterministic INTEGER NOT NULL DEFAULT 1,  -- Was this step deterministic?
  rag_mode TEXT NOT NULL DEFAULT 'graph',  -- 'graph', 'fts', 'semantic'
  duration_ms INTEGER,
  
  -- Status
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, running, completed, failed
  error TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  completed_at TEXT
);

CREATE INDEX idx_run_steps_run ON run_steps(run_id);

-- ------------------------------------------------------------
-- MEMORY SYSTEMS
-- ------------------------------------------------------------

-- Per-node memory (persists across runs)
CREATE TABLE node_memory (
  node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
  key TEXT NOT NULL,
  value TEXT NOT NULL,  -- JSON
  updated_at TEXT DEFAULT (datetime('now')),
  
  PRIMARY KEY (node_id, key)
);

-- Shared memory (blackboard, within a run)
CREATE TABLE shared_memory (
  run_id TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  key TEXT NOT NULL,
  value TEXT NOT NULL,  -- JSON
  written_by TEXT NOT NULL,  -- node_id
  updated_at TEXT DEFAULT (datetime('now')),
  
  PRIMARY KEY (run_id, key)
);

-- ------------------------------------------------------------
-- KNOWLEDGE SYSTEM (Deterministic RAG)
-- ------------------------------------------------------------

-- FTS5 virtual table for full-text search
CREATE VIRTUAL TABLE knowledge_docs USING fts5(
  path,
  title,
  content,
  tags,
  tokenize='porter'
);

-- Metadata for knowledge docs
CREATE TABLE knowledge_meta (
  path TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  tags TEXT NOT NULL DEFAULT '[]',  -- JSON array
  checksum TEXT NOT NULL,  -- For change detection
  indexed_at TEXT DEFAULT (datetime('now'))
);

-- Optional: Vector embeddings (non-deterministic, opt-in)
CREATE TABLE knowledge_embeddings (
  path TEXT PRIMARY KEY REFERENCES knowledge_meta(path) ON DELETE CASCADE,
  embedding BLOB NOT NULL,  -- Float32 array
  model TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

-- ------------------------------------------------------------
-- CASE KNOWLEDGE (Cross-Run Learning)
-- ------------------------------------------------------------

CREATE TABLE case_decisions (
  id TEXT PRIMARY KEY,
  domain TEXT NOT NULL,  -- 'architecture', 'ui', 'security', etc.
  decision TEXT NOT NULL,
  rationale TEXT,
  created_by TEXT,  -- node_id or 'user'
  run_id TEXT REFERENCES runs(id),
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_case_decisions_domain ON case_decisions(domain);

CREATE TABLE case_learnings (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,  -- 'pattern', 'anti-pattern', 'optimization'
  content TEXT NOT NULL,
  source TEXT,  -- Where this learning came from
  run_id TEXT REFERENCES runs(id),
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE case_feedback (
  id TEXT PRIMARY KEY,
  run_id TEXT REFERENCES runs(id),
  step_id TEXT REFERENCES run_steps(id),
  sentiment TEXT NOT NULL,  -- 'positive', 'negative', 'neutral'
  message TEXT NOT NULL,
  addressed INTEGER DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now'))
);
```

---

## 5. TypeScript Interfaces

```typescript
// ============================================================
// AGENT MOD TYPE DEFINITIONS
// ============================================================

// WORKFLOW DEFINITION
export interface Workflow {
  id: string
  name: string
  description?: string
  requireDeterminism: boolean
  createdAt: string
  updatedAt: string
}

export interface Node {
  id: string
  workflowId: string
  name: string
  role: string
  promptPath: string
  outputSchema?: object
  position: number
  knowledgeSubscriptions: string[]
  knowledgeTags: string[]
}

export interface Edge {
  id: string
  workflowId: string
  fromNode: string
  toNode: string
  condition?: string
  priority: number
}

// WORKFLOW EXECUTION
export type RunStatus = 'pending' | 'running' | 'paused' | 'completed' | 'failed'
export type StepStatus = 'pending' | 'running' | 'completed' | 'failed'
export type RagMode = 'graph' | 'fts' | 'semantic'

export interface Run {
  id: string
  workflowId: string
  status: RunStatus
  globalContext: GlobalContext
  startedAt?: string
  completedAt?: string
  createdAt: string
}

export interface GlobalContext {
  userPrompt: string
  requirements: Record<string, unknown>
  schema?: object
  decisions: string[]
  [key: string]: unknown
}

export interface RunStep {
  id: string
  runId: string
  nodeId: string
  sequence: number
  inputContext: GlobalContext
  resolvedPrompt: string
  knowledgeSources: string[]
  output?: unknown
  outputValid?: boolean
  validationErrors?: string[]
  deterministic: boolean
  ragMode: RagMode
  durationMs?: number
  status: StepStatus
  error?: string
  createdAt: string
  completedAt?: string
}

// MEMORY
export interface NodeMemory {
  nodeId: string
  key: string
  value: unknown
  updatedAt: string
}

export interface SharedMemory {
  runId: string
  key: string
  value: unknown
  writtenBy: string
  updatedAt: string
}

// KNOWLEDGE
export interface KnowledgeDoc {
  path: string
  title: string
  content: string
  tags: string[]
  checksum: string
  indexedAt: string
}

export interface KnowledgeQuery {
  mode: RagMode
  query?: string
  paths?: string[]
  tags?: string[]
  limit?: number
}

export interface KnowledgeResult {
  path: string
  title: string
  content: string
  score?: number
}

// CASE KNOWLEDGE
export interface CaseDecision {
  id: string
  domain: string
  decision: string
  rationale?: string
  createdBy?: string
  runId?: string
  createdAt: string
}

export interface CaseLearning {
  id: string
  type: 'pattern' | 'anti-pattern' | 'optimization'
  content: string
  source?: string
  runId?: string
  createdAt: string
}

export interface CaseFeedback {
  id: string
  runId?: string
  stepId?: string
  sentiment: 'positive' | 'negative' | 'neutral'
  message: string
  addressed: boolean
  createdAt: string
}

// CAPABILITY API
export interface SkaGenerateParams {
  prompt: string
  knowledge: KnowledgeResult[]
  temperature?: number
  schema?: object
}

export interface SkaGenerateResult {
  content: unknown
  usage: { inputTokens: number; outputTokens: number }
}

export interface ModCapabilities {
  'ska:generate': (params: SkaGenerateParams) => Promise<SkaGenerateResult>
  'ska:validate': (data: unknown, schema: object) => Promise<{ valid: boolean; errors?: string[] }>
}
```

---

## 6. Core Components

### 6.1 Orchestrator (`js/orchestrator.js`)

```javascript
export class Orchestrator {
  constructor(db, capabilities) {
    this.db = db
    this.capabilities = capabilities
    this.graph = new Graph(db)
    this.memory = new Memory(db)
    this.rag = new RAG(db)
    this.prompt = new PromptResolver()
  }

  async run(workflowId, initialContext) {
    const workflow = this.db.getWorkflow(workflowId)
    const run = this.db.createRun(workflowId, initialContext)
    
    try {
      await this.execute(run, workflow)
      this.db.updateRunStatus(run.id, 'completed')
    } catch (error) {
      this.db.updateRunStatus(run.id, 'failed', error.message)
      throw error
    }
    
    return this.db.getRun(run.id)
  }

  async resume(runId) {
    const run = this.db.getRun(runId)
    if (run.status !== 'paused') {
      throw new Error(`Cannot resume run with status: ${run.status}`)
    }
    
    const workflow = this.db.getWorkflow(run.workflowId)
    const lastStep = this.db.getLastStep(runId)
    
    this.db.updateRunStatus(runId, 'running')
    
    try {
      await this.execute(run, workflow, lastStep?.nodeId)
      this.db.updateRunStatus(runId, 'completed')
    } catch (error) {
      this.db.updateRunStatus(runId, 'failed', error.message)
      throw error
    }
    
    return this.db.getRun(runId)
  }

  async execute(run, workflow, resumeFromNode = null) {
    let currentNode = resumeFromNode 
      ? this.db.getNode(resumeFromNode)
      : this.graph.getStartNode(workflow.id)
    
    let sequence = this.db.getStepCount(run.id)
    let context = run.globalContext

    while (currentNode) {
      sequence++
      const result = await this.executeNode(run, currentNode, context, sequence)
      context = this.updateContext(context, currentNode, result)
      this.db.updateRunContext(run.id, context)
      currentNode = await this.graph.getNextNode(workflow.id, currentNode.id, context)
    }
  }

  async executeNode(run, node, context, sequence) {
    const step = this.db.createStep(run.id, node.id, sequence, context)
    const startTime = Date.now()

    try {
      const knowledge = await this.resolveKnowledge(node, context, run.workflowId)
      const caseKnowledge = this.getCaseKnowledge(node)
      
      const prompt = await this.prompt.resolve(node.promptPath, {
        context, knowledge, caseKnowledge,
        nodeMemory: this.memory.getNodeMemory(node.id),
        sharedMemory: this.memory.getSharedMemory(run.id)
      })
      
      const response = await this.capabilities['ska:generate']({
        prompt, knowledge, temperature: 0, schema: node.outputSchema
      })
      
      let valid = true, validationErrors = null
      if (node.outputSchema) {
        const validation = await this.capabilities['ska:validate'](response.content, node.outputSchema)
        valid = validation.valid
        validationErrors = validation.errors
      }
      
      this.db.updateStep(step.id, {
        resolvedPrompt: prompt,
        knowledgeSources: knowledge.map(k => k.path),
        output: response.content,
        outputValid: valid,
        validationErrors,
        deterministic: true,
        ragMode: 'graph',
        durationMs: Date.now() - startTime,
        status: valid ? 'completed' : 'failed'
      })
      
      if (!valid) throw new Error(`Output validation failed: ${JSON.stringify(validationErrors)}`)
      return response.content

    } catch (error) {
      this.db.updateStep(step.id, { status: 'failed', error: error.message, durationMs: Date.now() - startTime })
      throw error
    }
  }

  async resolveKnowledge(node, context, workflowId) {
    const results = []

    // Tier 1: Explicit subscriptions
    for (const path of node.knowledgeSubscriptions) {
      const doc = this.rag.getByPath(path)
      if (doc) results.push(doc)
    }

    // Tier 2: Tag-based (sorted for determinism)
    const tagDocs = this.rag.getByTags(node.knowledgeTags)
    tagDocs.sort((a, b) => a.path.localeCompare(b.path))
    results.push(...tagDocs)

    // Tier 3: FTS (agent-initiated)
    if (context._lookup) {
      results.push(...this.rag.search(context._lookup, { limit: 5 }))
      delete context._lookup
    }

    return results
  }

  getCaseKnowledge(node) {
    return {
      decisions: this.db.getCaseDecisions(node.role),
      learnings: this.db.getCaseLearnings(),
      feedback: this.db.getUnaddressedFeedback()
    }
  }

  updateContext(context, node, output) {
    return {
      ...context,
      [`${node.role}Output`]: output,
      decisions: [...context.decisions, ...(output.decisions || [])]
    }
  }
}
```

### 6.2 Graph Traversal (`lua/graph.lua`)

```lua
local graph = {}

function graph.get_start_node(db, workflow_id)
  local sql = [[
    SELECT n.* FROM nodes n
    WHERE n.workflow_id = ?
    AND n.id NOT IN (SELECT to_node FROM edges WHERE workflow_id = ?)
    ORDER BY n.position ASC LIMIT 1
  ]]
  return db:query_one(sql, workflow_id, workflow_id)
end

function graph.get_next_node(db, workflow_id, current_node_id, context)
  local sql = [[
    SELECT e.*, n.* FROM edges e
    JOIN nodes n ON e.to_node = n.id
    WHERE e.workflow_id = ? AND e.from_node = ?
    ORDER BY e.priority DESC, e.to_node ASC
  ]]
  
  local edges = db:query(sql, workflow_id, current_node_id)
  
  for _, edge in ipairs(edges) do
    if edge.condition == nil then return edge end
    local eval_fn = load("return " .. edge.condition, "condition", "t", context)
    if eval_fn and eval_fn() then return edge end
  end
  
  return nil
end

function graph.validate(db, workflow_id)
  local errors = {}
  local start_nodes = graph.get_start_nodes(db, workflow_id)
  if #start_nodes == 0 then table.insert(errors, "No start node found")
  elseif #start_nodes > 1 then table.insert(errors, "Multiple start nodes found") end
  if graph.has_cycle(db, workflow_id) then table.insert(errors, "Cycle detected") end
  return #errors == 0, errors
end

return graph
```

### 6.3 Memory System (`lua/memory.lua`)

```lua
local memory = {}

-- Node memory (persists across runs)
function memory.get_node(db, node_id, key)
  local row = db:query_one("SELECT value FROM node_memory WHERE node_id = ? AND key = ?", node_id, key)
  return row and json.decode(row.value) or nil
end

function memory.set_node(db, node_id, key, value)
  db:execute([[
    INSERT INTO node_memory (node_id, key, value, updated_at) VALUES (?, ?, ?, datetime('now'))
    ON CONFLICT(node_id, key) DO UPDATE SET value = excluded.value, updated_at = excluded.updated_at
  ]], node_id, key, json.encode(value))
end

-- Shared memory (within a run)
function memory.get_shared(db, run_id, key)
  local row = db:query_one("SELECT value FROM shared_memory WHERE run_id = ? AND key = ?", run_id, key)
  return row and json.decode(row.value) or nil
end

function memory.set_shared(db, run_id, key, value, written_by)
  db:execute([[
    INSERT INTO shared_memory (run_id, key, value, written_by, updated_at) VALUES (?, ?, ?, ?, datetime('now'))
    ON CONFLICT(run_id, key) DO UPDATE SET value = excluded.value, written_by = excluded.written_by, updated_at = excluded.updated_at
  ]], run_id, key, json.encode(value), written_by)
end

return memory
```

### 6.4 RAG System (`js/rag.js`)

```javascript
export class RAG {
  constructor(db) { this.db = db }

  index(path, title, content, tags = []) {
    const checksum = this.checksum(content)
    const existing = this.db.query('SELECT checksum FROM knowledge_meta WHERE path = ?', [path])[0]
    if (existing?.checksum === checksum) return false
    
    this.db.execute('DELETE FROM knowledge_docs WHERE path = ?', [path])
    this.db.execute('INSERT INTO knowledge_docs (path, title, content, tags) VALUES (?, ?, ?, ?)', 
      [path, title, content, tags.join(' ')])
    this.db.execute(`
      INSERT INTO knowledge_meta (path, title, tags, checksum, indexed_at) VALUES (?, ?, ?, ?, datetime('now'))
      ON CONFLICT(path) DO UPDATE SET title = excluded.title, tags = excluded.tags, checksum = excluded.checksum, indexed_at = excluded.indexed_at
    `, [path, title, JSON.stringify(tags), checksum])
    return true
  }

  getByPath(path) {
    return this.db.query('SELECT path, title, content FROM knowledge_docs WHERE path = ?', [path])[0] || null
  }

  getByTags(tags) {
    if (!tags.length) return []
    const placeholders = tags.map(() => '?').join(' OR tags MATCH ')
    return this.db.query(`SELECT path, title, content FROM knowledge_docs WHERE tags MATCH ${placeholders} ORDER BY path ASC`, tags)
  }

  search(query, options = {}) {
    return this.db.query(`
      SELECT path, title, content, bm25(knowledge_docs) as score FROM knowledge_docs
      WHERE knowledge_docs MATCH ? ORDER BY score ASC, path ASC LIMIT ?
    `, [query, options.limit || 10])
  }

  checksum(content) {
    let hash = 0
    for (let i = 0; i < content.length; i++) {
      hash = ((hash << 5) - hash) + content.charCodeAt(i)
      hash = hash & hash
    }
    return hash.toString(16)
  }
}
```

### 6.5 Prompt Resolver (`js/prompt.js`)

```javascript
export class PromptResolver {
  async resolve(templatePath, data) {
    const template = await mod.readFile(`prompts/${templatePath}`)
    return this.interpolate(template, data)
  }

  interpolate(template, data) {
    return template.replace(/\{\{([^}]+)\}\}/g, (match, path) => {
      const value = this.getPath(data, path.trim())
      if (value === undefined) return match
      return typeof value === 'object' ? JSON.stringify(value, null, 2) : String(value)
    })
  }

  getPath(obj, path) {
    return path.split('.').reduce((curr, key) => curr?.[key], obj)
  }
}
```

---

## 7. Context Resolution Hierarchy

```
TIER 1: EXPLICIT SUBSCRIPTIONS (Always Deterministic)
  Node declares: knowledgeSubscriptions: ["patterns/auth.md"]
  Result: Exact document loaded

TIER 2: TAG-BASED (Always Deterministic)
  Node declares: knowledgeTags: ["auth", "security"]
  Result: All docs with tags, sorted by path

TIER 3: FULL-TEXT SEARCH (Deterministic, Agent-Initiated)
  Agent outputs: LOOKUP: authentication best practices
  Result: FTS5 BM25 search, sorted by score then path

TIER 4: SEMANTIC SEARCH (Non-Deterministic, Opt-In Only)
  Only if: workflow.requireDeterminism = false
  Result: Vector similarity search
```

---

## 8. Memory Systems

| System | Scope | Persistence | Use Case |
|--------|-------|-------------|----------|
| **Global Context** | Current run | Per-run | Passed node-to-node, current truth |
| **Shared Memory** | Current run | Per-run | Cross-agent communication (blackboard) |
| **Node Memory** | Per node | Cross-run | Agent-specific learnings |
| **Case Knowledge** | Project-wide | Permanent | Decisions, patterns, feedback |

---

## 9. Nitro Integration

Agent Mod requires Nitro supervision:

| Need | Why |
|------|-----|
| Workflows run 5-30 minutes | Crash recovery |
| User closes browser | Continues execution |
| Must log every step | LOG service |
| Determinism requires complete audit | Graceful shutdown |

---

## 10. User Project Structure

```
/project/
├── mods/agent-mod.sqlar        # From Depot
├── workflows/*.json            # User-defined workflows
├── prompts/*.md                # Agent prompts
├── knowledge/**/*.md           # Domain knowledge
└── state.glyphcase             # Project state
```

---

## 11. Capability API

Agent Mod requests LLM calls from host, never calls directly:

```javascript
// RIGHT
const response = await mod.request('ska:generate', { prompt, knowledge, temperature: 0, schema })

// WRONG
const response = await anthropic.messages.create({ ... })
```

---

## 12. Security

| Concern | Mitigation |
|---------|------------|
| Prompt injection | Ska handles |
| Path traversal | Paths validated against project root |
| RAG injection | Content sanitized |
| API keys | Host manages (capability API) |
| Embedding isolation | Per-project databases |

---

## 13. Audit Trail

Every step records: input context, resolved prompt, knowledge sources, output, validation status, determinism flag, RAG mode, duration, timestamps.

---

## Appendix: Error Codes

| Code | Meaning |
|------|---------|
| `WORKFLOW_NOT_FOUND` | Workflow ID doesn't exist |
| `NODE_NOT_FOUND` | Node referenced in edge doesn't exist |
| `CYCLE_DETECTED` | Workflow graph has a cycle |
| `VALIDATION_FAILED` | Output doesn't match schema |
| `KNOWLEDGE_NOT_FOUND` | Subscribed path doesn't exist |
| `CAPABILITY_DENIED` | Mod doesn't have required capability |

---

*End of Specification*
