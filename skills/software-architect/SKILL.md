---
name: software-architect
user_invocable: true
description: Architecture review using "A Philosophy of Software Design" principles. Modes: red-flags, depth, complexity, fcis, performance, split, comments, naming, consistency, tactical, review.
---

# Software Architect

Expert architecture analysis based on John Ousterhout's "A Philosophy of Software Design" (2nd Edition) and Jeff Dean's Performance Hints.

## Core Principle

**Complexity is the enemy.** It manifests as:
1. **Change amplification** - small changes touch many places
2. **Cognitive load** - too much to know to make a change
3. **Unknown unknowns** - not knowing what you don't know

Root causes: **Dependencies** (can't understand in isolation) and **Obscurity** (important info not obvious).

## Sub-Agent Execution Strategy

When running as a sub-agent, follow this efficient pattern:

```
1. STRUCTURE SCAN (single Glob)
   → Glob **/*.{py,ts,js,go,rs,java} to map codebase

2. ENTRY POINT READ (1-2 files)
   → Read main entry files (main.*, index.*, app.*)

3. TARGETED ANALYSIS (mode-specific)
   → Use grep patterns below for the requested mode
   → Read only flagged files

4. REPORT
   → Use output template for mode
   → Cite file:line for every finding
```

**Token Budget:** Aim for <20 file reads per analysis. Quality over quantity.

## The 9 Red Flags

| Flag | Symptom | Fix | Detection Pattern |
|------|---------|-----|-------------------|
| **Shallow Module** | Interface ~ implementation complexity | Combine modules or rethink abstraction | Classes <50 lines with many public methods |
| **Information Leakage** | Same knowledge in multiple modules | Centralize in one module | Same constant/format in multiple files |
| **Temporal Decomposition** | Structure follows execution order | Structure by information hiding | Files named step1, step2, phase1, etc. |
| **Overexposure** | Must know rare features for common case | Hide behind sensible defaults | Functions with >5 parameters |
| **Pass-Through Method** | Just delegates to similar method | Merge layers or add value | `return self.x.method()` patterns |
| **Repetition** | Same pattern in multiple places | Extract to single location | Duplicate code blocks |
| **Special-General Mixture** | Special-case code in general mechanism | Push special code to higher layer | `if type == "special"` in generic code |
| **Conjoined Methods** | Can't understand one without the other | Make methods self-contained | Methods always called together |
| **Hard to Name** | Difficulty naming = unclear purpose | Rethink responsibilities | Names: Manager, Handler, Processor, Utils |

### Grep Patterns for Red Flag Detection

```bash
# Shallow Module - many public, little implementation
Grep: "^(export |public )" → count per file, flag if ratio > 0.3

# Information Leakage - same magic values
Grep: "(0x[0-9A-F]+|[A-Z_]{4,}.*=.*[0-9]+)" → find duplicates across files

# Pass-Through Methods
Grep: "return (self|this)\.[a-z]+\.[a-z]+\("

# Overexposure - too many parameters
Grep: "def |function |fn |func " → Read and count params, flag if >5

# Hard to Name
Grep: "(Manager|Handler|Processor|Helper|Utils|Misc|Common|Base)(\.|::|\(|$)"
```

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

## Strategic vs Tactical Programming

| Approach | Mindset | Result |
|----------|---------|--------|
| **Tactical** | "Just get it working" | Accumulates complexity, creates debt |
| **Strategic** | "Invest 10-20% in design" | Pays dividends, faster long-term |

### Tactical Red Flags
- Features added without refactoring
- "We'll fix it later" comments
- Copy-paste with minor changes
- No abstractions, just raw implementation
- Growing functions instead of extracting

**The Tactical Tornado:** Prolific coder who ships fast but leaves complexity. Management sees hero; codebase sees villain.

```bash
# Detect tactical patterns
Grep: "TODO|FIXME|HACK|XXX|TEMPORARY|WORKAROUND"
Grep: "# fix later|// fix later|fix this later"
```

## Comments & Documentation (Ch 12-16)

### What Comments Should Capture
- **Intent** - Why, not what
- **Design decisions** - Rationale for choices
- **Non-obvious behavior** - Edge cases, gotchas
- **Cross-module concerns** - How pieces fit together

### Comment Red Flags
| Bad | Why | Better |
|-----|-----|--------|
| `// increment i` | Repeats code | Don't write it |
| `// loop through users` | Obvious | State the purpose |
| `// returns the value` | No information | State what value means |

### Good Comment Patterns
```python
# Low-level: Add precision
timeout = 30  # seconds, chosen to exceed max DB query time

# High-level: Add intuition
# This module handles all authentication flows. External callers
# should only use authenticate() - internal methods handle OAuth,
# SAML, and password flows transparently.

# Interface: Contract documentation
def process(data: bytes) -> Result:
    """Transform raw sensor data into calibrated measurements.

    Args:
        data: Raw bytes from sensor, must be exactly 64 bytes

    Returns:
        Result with .values (list of floats) or .error (string)

    Note:
        Thread-safe. Calibration loaded once at module init.
    """
```

### Detection Patterns
```bash
# Missing interface docs
Grep: "^(def |function |class |interface )" → check for docstring/JSDoc

# Useless comments (repeats code)
Grep: "// (get|set|return|increment|decrement|loop|iterate)"

# Good: Design rationale
Grep: "(because|reason|tradeoff|alternative|chose|decision)"
```

## Naming Conventions (Ch 14)

### Properties of Good Names
1. **Precise** - Says exactly what it is
2. **Consistent** - Same concept = same name everywhere

### Naming Red Flags
| Bad | Problem | Better |
|-----|---------|--------|
| `data`, `info`, `item` | Too vague | `userProfile`, `orderData` |
| `temp`, `tmp`, `x` | Meaningless (unless tiny scope) | Describe the value |
| `processData()` | What kind of processing? | `validateUserInput()` |
| `handleStuff()` | Stuff? | `routeIncomingRequest()` |
| `doWork()` | What work? | `calculateTaxes()` |

### Loop Variable Rule
- `i, j, k` - OK for tiny loops (< 10 lines)
- Larger loops need descriptive names: `userIndex`, `rowNum`

### Consistency Rule
Pick one name per concept and use it everywhere:
- `user` vs `customer` vs `client` - pick ONE
- `fetch` vs `get` vs `retrieve` - pick ONE
- `create` vs `make` vs `build` - pick ONE

```bash
# Detect naming issues
Grep: "\b(data|info|temp|tmp|result|value|item|obj|thing)\b" -i

# Detect inconsistency - look for synonyms
Grep: "(get|fetch|retrieve|load|read)[A-Z]" → should be consistent
Grep: "(user|customer|client|account)" → should be consistent
```

## Consistency (Ch 17)

### Why Consistency Matters
- Reduces cognitive load (learn once, apply everywhere)
- Enables safe assumptions
- Makes code predictable

### What to Keep Consistent
| Area | Example |
|------|---------|
| Naming | Always `userId`, never `user_id` or `uid` |
| Error handling | Always throw, or always return errors |
| Async patterns | Always promises, or always callbacks |
| File structure | `components/Foo/Foo.tsx`, `Foo.test.tsx`, `Foo.styles.ts` |

### The "Better Idea" Trap
> "Having a better idea is NOT a sufficient excuse to introduce inconsistency."

If changing convention, change it everywhere or don't change it.

```bash
# Detect inconsistency
Grep: "(snake_case|camelCase)" → mixing styles
Grep: "Promise|callback|async.*await" → mixing async patterns
Grep: "(throw|return.*error|return.*null)" → mixing error patterns
```

## Design It Twice

Before implementing, design at least two different approaches:
1. Compare tradeoffs explicitly
2. Pick best fit for requirements
3. Even a quick comparison improves design

### When to Apply
- New subsystems
- Complex algorithms
- Public interfaces
- Architectural decisions

## Define Errors Out of Existence (Ch 10)

Instead of adding error handling, redefine the operation so errors can't occur:

| Problem | Bad | Good |
|---------|-----|------|
| Delete non-existent key | Throw KeyNotFound | No-op (success) |
| Substring past end | Throw OutOfBounds | Return partial |
| Divide by zero | Throw DivideByZero | Define as 0 or ∞ |

```bash
# Detect overactive error handling
Grep: "throw|raise|panic" → count, high ratio may indicate opportunity
Grep: "try.*catch|except|rescue" → same analysis
```

## Analysis Modes

| Mode | Command | Focus |
|------|---------|-------|
| `red-flags` | `/software-architect red-flags` | Scan for 9 red flags |
| `depth` | `/software-architect depth` | Module depth analysis |
| `complexity` | `/software-architect complexity` | 3 complexity metrics |
| `fcis` | `/software-architect fcis` | Core/shell separation |
| `performance` | `/software-architect performance` | Performance patterns |
| `split` | `/software-architect split` | Split vs combine advice |
| `comments` | `/software-architect comments` | Comment quality review |
| `naming` | `/software-architect naming` | Naming convention check |
| `consistency` | `/software-architect consistency` | Codebase consistency |
| `tactical` | `/software-architect tactical` | Strategic vs tactical audit |
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

### Comments Mode
```markdown
## Comment Quality Summary
| Category | Count | Assessment |
|----------|-------|------------|
| Interface docs | | |
| Design rationale | | |
| Useless (repeats code) | | |
| Missing (public APIs) | | |

## Issues
| Location | Problem | Suggestion |
|----------|---------|------------|

## Recommendations
1. [Priority-ordered fixes]
```

### Naming Mode
```markdown
## Naming Analysis
| Category | Finding |
|----------|---------|
| Vague names | [list] |
| Inconsistencies | [list] |
| Overly generic | [list] |

## Recommendations
| Current | Suggested | Reason |
|---------|-----------|--------|
```

### Consistency Mode
```markdown
## Consistency Audit
| Area | Convention A | Convention B | Files Affected |
|------|--------------|--------------|----------------|

## Recommendation
Pick [A/B] because [reason]. Update these files: [list]
```

### Tactical Mode
```markdown
## Technical Debt Indicators
| Pattern | Count | Locations |
|---------|-------|-----------|
| TODO/FIXME/HACK | | |
| "Fix later" comments | | |
| Copied code blocks | | |
| Giant functions | | |

## Assessment
Strategic score: [1-10]

## Recommendations
1. [How to reduce tactical debt]
```

## Key Heuristics

**Split if:** Truly independent, different rates of change, general vs special-purpose.

**Combine if:** Share information, combining simplifies interface, hard to understand separately.

**Pull complexity down:** Module developer suffers so users don't.

**Define errors out:** Redefine semantics so errors become normal cases.

**Different layer = different abstraction:** Adjacent layers with similar abstractions = problem.

## Language-Specific Patterns

### Python
```bash
# Entry points
Glob: "**/main.py", "**/__main__.py", "**/app.py", "**/wsgi.py"

# Module structure
Glob: "**/__init__.py" → map package hierarchy

# Red flags
Grep: "from.*import \*" → star imports (information leakage)
Grep: "global " → global state
Grep: "except:" → bare except (overactive error handling)
```

### TypeScript/JavaScript
```bash
# Entry points
Glob: "**/index.{ts,js}", "**/main.{ts,js}", "**/app.{ts,js}"

# Red flags
Grep: "any" → type: any (defeats type safety)
Grep: "as any" → type assertions (same)
Grep: "// @ts-ignore" → suppressed errors
Grep: "export \*" → barrel exports (information leakage)
```

### Go
```bash
# Entry points
Glob: "**/main.go", "**/cmd/**/*.go"

# Red flags
Grep: "interface\{\}" → empty interface (no type safety)
Grep: "panic\(" → panic in library code
Grep: "_ =" → ignored errors
```

### Rust
```bash
# Entry points
Glob: "**/main.rs", "**/lib.rs"

# Red flags
Grep: "unwrap\(\)" → panicking unwrap
Grep: "expect\(" → panicking expect
Grep: "unsafe " → unsafe blocks
```

### Java
```bash
# Entry points
Glob: "**/Main.java", "**/Application.java", "**/*Application.java"

# Red flags
Grep: "catch.*Exception.*\{\s*\}" → empty catch
Grep: "public class.*\{" → check file line count (God class)
Grep: "@SuppressWarnings" → suppressed warnings
```

## Quick Reference Checklists

### Pre-Review Checklist (Run These Greps First)
```bash
1. Glob **/*.{ext} → understand file count and structure
2. Grep "TODO|FIXME|HACK" → find debt markers
3. Grep "Manager|Handler|Utils" → find naming smells
4. Grep "^(class|def|function|interface)" → map abstractions
5. Read entry point files → understand flow
```

### Module Depth Checklist
For each major module, answer:
- [ ] Interface line count vs implementation line count?
- [ ] How many concepts must caller understand?
- [ ] What details are hidden from caller?
- [ ] Rating: Deep / Medium / Shallow

### FCIS Verification Checklist
- [ ] Can I identify "core" (pure logic) modules?
- [ ] Can I identify "shell" (I/O) modules?
- [ ] Does core import from shell? (violation)
- [ ] Is business logic mixed with I/O? (violation)
