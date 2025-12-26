---
name: software-architect
user_invocable: true
description: Architecture review using "A Philosophy of Software Design" principles. Modes: red-flags, depth, complexity, fcis, performance, split, review.
---

# Software Architect

Expert architecture analysis based on John Ousterhout's "A Philosophy of Software Design" and the Functional Core/Imperative Shell pattern.

## Core Principle

**Complexity is the enemy.** It manifests as:
1. **Change amplification** - small changes touch many places
2. **Cognitive load** - too much to know to make a change
3. **Unknown unknowns** - not knowing what you don't know

Root causes: **Dependencies** (can't understand in isolation) and **Obscurity** (important info not obvious).

## The 9 Red Flags

| Flag | Symptom | Fix |
|------|---------|-----|
| **Shallow Module** | Interface ~ implementation complexity | Combine modules or rethink abstraction |
| **Information Leakage** | Same knowledge in multiple modules | Centralize in one module |
| **Temporal Decomposition** | Structure follows execution order | Structure by information hiding |
| **Overexposure** | Must know rare features for common case | Hide behind sensible defaults |
| **Pass-Through Method** | Just delegates to similar method | Merge layers or add value |
| **Repetition** | Same pattern in multiple places | Extract to single location |
| **Special-General Mixture** | Special-case code in general mechanism | Push special code to higher layer |
| **Conjoined Methods** | Can't understand one without the other | Make methods self-contained |
| **Hard to Name** | Difficulty naming = unclear purpose | Rethink responsibilities |

## Deep Module Principle

**Deep:** Simple interface, significant hidden functionality (good).
**Shallow:** Interface complexity ~ implementation complexity (bad).

```
DEEP                SHALLOW
┌────────┐          ┌──────────────────┐
│interface│         │    interface     │
├────────┤          ├──────────────────┤
│        │          │  implementation  │
│  impl  │          └──────────────────┘
│        │
└────────┘
```

## Functional Core / Imperative Shell (FCIS)

| Layer | Contains | Characteristics |
|-------|----------|-----------------|
| **Core** | Business logic, rules, transformations | Pure functions, no I/O, easily tested |
| **Shell** | I/O, DB, network, external services | Thin, orchestrates core, hard to test |

Verify: Core has no imports from shell. Shell calls core, not vice versa.

## Performance Red Flags

| Pattern | Problem | Fix |
|---------|---------|-----|
| Allocation in hot loop | Memory pressure | Reuse buffers, pre-size |
| Lock per operation | Contention | Batch under single lock |
| O(N) where O(1) works | Algorithmic waste | Hash tables, precompute |
| Logging in hot path | Cost even disabled | Remove or sample 1-in-N |
| Creating clients per request | Connection overhead | Reuse/pool clients |
| Sequential I/O | Latency stacking | Parallelize independent calls |

## Quick Reference Numbers

| Operation | Latency |
|-----------|---------|
| L1 cache | 0.5 ns |
| L2 cache | 7 ns |
| Main memory | 100 ns |
| SSD read | 150 us |
| Network (same DC) | 0.5 ms |

## Analysis Modes

| Mode | Command | Focus |
|------|---------|-------|
| `red-flags` | `/software-architect red-flags` | Scan for 9 red flags |
| `depth` | `/software-architect depth` | Module depth analysis |
| `complexity` | `/software-architect complexity` | 3 complexity metrics |
| `fcis` | `/software-architect fcis` | Core/shell separation |
| `performance` | `/software-architect performance` | Performance patterns |
| `split` | `/software-architect split` | Split vs combine advice |
| `review <file>` | `/software-architect review src/foo.py` | Review specific file |
| (none) | `/software-architect` | General review |

## Analysis Process

1. **Explore** - Use `Glob` for structure, `Grep` for patterns, `Read` for details
2. **Identify** - Match against red flags / principles above
3. **Assess** - Rate severity: critical > high > medium > low
4. **Report** - Cite `file:line`, explain fix, prioritize by impact

## Output Templates

### General / Review Mode
```markdown
## Summary
[1-2 sentences]

## Issues
| Issue | Location | Severity | Fix |
|-------|----------|----------|-----|

## Strengths
[What's done well]

## Recommendations
1. [Highest priority first]
```

### Red Flags Mode
| Flag | Location | Severity | Evidence | Fix |
|------|----------|----------|----------|-----|

### Depth Mode
| Module | Interface | Implementation | Rating | Notes |
|--------|-----------|----------------|--------|-------|

### Complexity Mode
| Metric | Score (1-5) | Evidence |
|--------|-------------|----------|
| Change Amplification | | |
| Cognitive Load | | |
| Unknown Unknowns | | |

### FCIS Mode
**Core Layer:**
- [module] - Pure/Impure - Notes

**Shell Layer:**
- [module] - Notes

**Violations:** [Any core importing shell, I/O in core, etc.]

### Performance Mode
| Pattern | Location | Severity | Current | Fix |
|---------|----------|----------|---------|-----|

### Split Mode
```markdown
## Current State
[Description]

## Recommendation: [Split/Keep Combined]
**Rationale:** [Why]

**If splitting:**
- Component A: [responsibility]
- Component B: [responsibility]
```

## Key Heuristics

**Split if:** Truly independent, different rates of change, general vs special-purpose.

**Combine if:** Share information, combining simplifies interface, hard to understand separately.

**Pull complexity down:** Module developer suffers so users don't.

**Define errors out:** Redefine semantics so errors become normal cases.

**Different layer = different abstraction:** Adjacent layers with similar abstractions = problem.
