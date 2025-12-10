# Punk Documentation Manifest

**Version:** 2.0  
**Status:** Source of Truth  
**Last Updated:** December 2025  
**Purpose:** Track document dependencies, propagate changes, prevent inconsistencies

---

## Overview

This manifest tracks all Punk/Mohawk documentation, their dependencies, and required updates when foundational documents change.

---

## Document Registry

### Tier 0: Foundational (Changes Ripple Everywhere)

| Document | Status | Description |
|----------|--------|-------------|
| **PUNK_GLOSSARY.md** | ✅ Current | Canonical terminology |
| **PUNK_DECISIONS.md** | ✅ Current (v3.0) | Architectural decisions (18 ADRs) |

### Tier 1: Core Infrastructure

| Document | Status | Description |
|----------|--------|-------------|
| **PUNK_SCHEMAS.md** | ✅ Current (v3.0) | @effect/schema definitions |
| **PUNK_ULID_SPEC.md** | ✅ Current (v3.0) | ID generation with Effect |
| **PUNK_BIOME_IMPL.md** | ✅ Current | Linting & formatting |
| **PUNK_LIGHTNINGCSS_IMPL.md** | ✅ Current | CSS processing |
| **PUNK_NITRO_IMPL.md** | ✅ Current | Process supervisor |
| **PUNK_PORTABLE_EXEC_SPEC.md** | 🅿️ Planned | Cosmopolitan exe export |

### Tier 2: Services & Features

| Document | Status | Description |
|----------|--------|-------------|
| **MCP_GATEWAY_SPECIFICATION_v2.md** | ✅ Current | MCP Gateway service |
| **PUNK_AGENT_MOD_IMPL.md** | ✅ Current | Agent workflows |
| **PUNK_MOD_IMPL.md** | ✅ Current | Mod system |
| **PUNK_ERROR_RECOVERY_IMPL.md** | ✅ Current | AI error recovery |
| **PUNK_RIGS_IMPL.md** | ✅ Current (v2.0) | `punk wrap` CLI, Rig architecture |

### Tier 3: Rendering & UI

| Document | Status | Description |
|----------|--------|-------------|
| **PUNK_SELECTION_SPEC.md** | ✅ Current | Selection UI |

### Tier 4: Applications

| Document | Status | Description |
|----------|--------|-------------|
| **PUNK_DESKTOP_IMPL.md** | ⚠️ Review | Desktop app (check Base UI refs) |
| **PUNK_MOHAWK_ARCHITECTURE.md** | ⚠️ Review | System architecture |

### Deprecated Documents

| Document | Status | Superseded By |
|----------|--------|---------------|
| **PUNK_GARDENJS_IMPL.md** | ❌ Deprecated | PUNK_RIGS_IMPL.md |
| **PUNK_RIGS.md** | ❌ Deprecated | PUNK_RIGS_IMPL.md |

### Overview Documents (Summaries)

| Document | Status | Source Of |
|----------|--------|-----------|
| PUNK_CORE.md | ✅ Current (v3.0) | Summary of SCHEMAS, RIGS |
| PUNK_MODS.md | ⚠️ Review | Summary of MOD_IMPL |
| PUNK_BACKENDS.md | ⚠️ Review | Backend options |
| PUNK_SKA.md | ⚠️ Review | AI engine |
| PUNK_MOHAWK.md | ⚠️ Review | Platform summary |

---

## December 2025 Update Summary

### Documents Updated This Session

| Document | Change | Version |
|----------|--------|---------|
| **PUNK_SCHEMAS.md** | Zod → @effect/schema rewrite | 3.0 |
| **PUNK_DECISIONS.md** | +4 ADRs (Effect, Base UI, punk wrap, Tiers) | 3.0 |
| **PUNK_ULID_SPEC.md** | Zod → Effect schemas | 3.0 |
| **PUNK_RIGS_IMPL.md** | New: punk wrap CLI | 2.0 |
| **PUNK_CORE.md** | Effect + Base UI alignment | 3.0 |
| **PUNK_RIGS.md** | Deprecated → points to RIGS_IMPL | — |
| **PUNK_GARDENJS_IMPL.md** | Deprecated → points to RIGS_IMPL | — |

### Key Architectural Changes

1. **Zod → @effect/schema**: All validation now uses Effect (ADR-015)
2. **Radix → Base UI**: Default component backend (ADR-016)
3. **Garden.js → punk wrap**: Rig generation via CLI (ADR-017)
4. **Tier model**: free/starter/pro/purchasable/premium (ADR-018)

---

## Dependency Graph

```
                       PUNK_GLOSSARY (Tier 0)
                               │
                               ▼
    ┌──────────────────────────┴──────────────────────────┐
    │                                                      │
    ▼                                                      ▼
PUNK_SCHEMAS ◄─────────────────────────────────── PUNK_DECISIONS
    │                                                      
    ├─────────────┬─────────────┬─────────────┐
    │             │             │             │
    ▼             ▼             ▼             ▼
PUNK_NITRO    MCP_GATEWAY    PUNK_MOD    PUNK_RIGS_IMPL
    │             │             │             │
    │             └─────┬───────┘             │
    │                   │                     │
    │                   ▼                     │
    │           PUNK_AGENT_MOD ◄──────────────┘
    │                   │
    └───────────────────┤
                        │
                        ▼
                PUNK_DESKTOP_IMPL
                        │
                        ▼
           PUNK_MOHAWK_ARCHITECTURE
```

---

## Change Impact Matrix

### When PUNK_SCHEMAS.md Changes

| Document | Impact | Action |
|----------|--------|--------|
| PUNK_RIGS_IMPL.md | HIGH | Update generated schema patterns |
| PUNK_AGENT_MOD_IMPL.md | MEDIUM | Update validation patterns |
| PUNK_MOD_IMPL.md | LOW | Update mod schema refs |

### When PUNK_DECISIONS.md Changes

| Document | Impact | Action |
|----------|--------|--------|
| ALL | REVIEW | Check for affected patterns |

### When PUNK_RIGS_IMPL.md Changes

| Document | Impact | Action |
|----------|--------|--------|
| PUNK_DESKTOP_IMPL.md | MEDIUM | Update Rig integration |
| PUNK_CORE.md | LOW | Update overview |

---

## Validation Rules

### Rule 1: Terminology Consistency
All documents must use terminology from PUNK_GLOSSARY.md.

### Rule 2: No Zod References
All schema code must use @effect/schema, not Zod.

### Rule 3: No Garden.js References
Rigs are generated via `punk wrap`, not Garden.js.

### Rule 4: Component Backend
Default is Base UI. Radix only via explicit shim.

### Rule 5: Deprecated Docs
Deprecated documents must be one-liners pointing to replacement.

---

## Status Definitions

| Status | Meaning |
|--------|---------|
| ✅ **Current** | Source of truth, implement from this |
| ⚠️ **Review** | May need minor updates |
| ❌ **Deprecated** | Superseded, do not use |
| 🅿️ **Planned** | Future, not yet written |

---

## Quick Reference: What to Update When

### "I changed Effect schema patterns"
→ Update: RIGS_IMPL, AGENT_MOD, ERROR_RECOVERY

### "I changed Rig tier model"
→ Update: RIGS_IMPL, DECISIONS, MOHAWK

### "I changed component backend (Base UI/Radix)"
→ Update: DECISIONS, CORE, DESKTOP

### "I added a new term"
→ Update: GLOSSARY first, then reference in relevant docs

---

## Appendix: Full Dependency List

### PUNK_SCHEMAS.md
```yaml
depends_on:
  - PUNK_GLOSSARY.md
dependents:
  - PUNK_RIGS_IMPL.md
  - PUNK_MOD_IMPL.md
  - PUNK_AGENT_MOD_IMPL.md
  - PUNK_ERROR_RECOVERY_IMPL.md
```

### PUNK_RIGS_IMPL.md
```yaml
depends_on:
  - PUNK_GLOSSARY.md
  - PUNK_SCHEMAS.md
  - PUNK_DECISIONS.md
dependents:
  - PUNK_DESKTOP_IMPL.md
  - PUNK_CORE.md
```

### PUNK_AGENT_MOD_IMPL.md
```yaml
depends_on:
  - PUNK_GLOSSARY.md
  - PUNK_SCHEMAS.md
  - PUNK_NITRO_IMPL.md
  - MCP_GATEWAY_SPECIFICATION_v2.md
dependents:
  - PUNK_DESKTOP_IMPL.md
```

### PUNK_DESKTOP_IMPL.md
```yaml
depends_on:
  - PUNK_GLOSSARY.md
  - PUNK_SCHEMAS.md
  - PUNK_NITRO_IMPL.md
  - PUNK_AGENT_MOD_IMPL.md
  - PUNK_MOD_IMPL.md
  - PUNK_RIGS_IMPL.md
  - PUNK_SELECTION_SPEC.md
dependents:
  - PUNK_MOHAWK_ARCHITECTURE.md
```

---

*Last updated: December 2025*
