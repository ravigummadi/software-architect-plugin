# Software Architect Plugin

Expert architecture analysis based on John Ousterhout's **"A Philosophy of Software Design"** (2nd Edition) and **Jeff Dean's Performance Hints**.

Optimized for use as a **Claude Code sub-agent** with efficient tool usage patterns and structured output templates.

## Installation

```bash
# From marketplace
claude plugins add-marketplace https://github.com/ravigummadi/claude-code-plugins
claude plugins install software-architect@ravigummadi

# Or local
git clone https://github.com/ravigummadi/software-architect-plugin.git ~/.claude/plugins/software-architect
```

## Usage

```bash
# Core analysis modes
/software-architect              # General architecture review
/software-architect red-flags    # Scan for 9 red flags
/software-architect depth        # Module depth analysis (deep vs shallow)
/software-architect complexity   # 3 complexity metrics scoring
/software-architect fcis         # Functional Core/Imperative Shell check
/software-architect performance  # Performance anti-patterns
/software-architect split        # Split vs combine advice

# NEW: Additional modes from "A Philosophy of Software Design"
/software-architect comments     # Comment quality & documentation review
/software-architect naming       # Naming convention analysis
/software-architect consistency  # Codebase consistency audit
/software-architect tactical     # Strategic vs tactical programming assessment

# File-specific review
/software-architect review <file> # Review specific file
```

## Core Concepts

### Complexity (The Enemy)
Complexity manifests as:
1. **Change amplification** - small changes touch many places
2. **Cognitive load** - too much to know to make a change
3. **Unknown unknowns** - not knowing what you don't know

Root causes: **Dependencies** and **Obscurity**

### Deep Modules
```
DEEP (good)         SHALLOW (bad)
┌────────┐          ┌──────────────────┐
│interface│         │    interface     │
├────────┤          ├──────────────────┤
│        │          │  implementation  │
│  impl  │          └──────────────────┘
│        │
└────────┘
```

### FCIS Pattern (Functional Core / Imperative Shell)
- **Core**: Pure functions, no I/O, business logic, easily tested
- **Shell**: I/O only, thin orchestrator, calls core

### The 9 Red Flags
| Flag | Symptom | Detection |
|------|---------|-----------|
| Shallow Module | Interface ≈ implementation | Small classes with many public methods |
| Information Leakage | Same knowledge in multiple modules | Duplicate constants/formats |
| Temporal Decomposition | Structure follows execution order | step1, step2, phase1 naming |
| Overexposure | Must know rare features for common case | Functions with >5 parameters |
| Pass-Through Method | Just delegates | `return self.x.method()` |
| Repetition | Same pattern repeated | Duplicate code blocks |
| Special-General Mixture | Mixed concerns | `if type == "special"` in generic code |
| Conjoined Methods | Can't understand separately | Methods always called together |
| Hard to Name | Unclear purpose | Manager, Handler, Utils, Processor |

### Strategic vs Tactical Programming (NEW)
- **Tactical**: "Just get it working" → accumulates complexity
- **Strategic**: Invest 10-20% in design → faster long-term

### Comments & Documentation (NEW)
Good comments capture:
- **Intent** - Why, not what
- **Design decisions** - Rationale for choices
- **Non-obvious behavior** - Edge cases, gotchas

### Naming Conventions (NEW)
Good names are:
- **Precise** - Says exactly what it is
- **Consistent** - Same concept = same name everywhere

### Performance Red Flags
| Pattern | Fix |
|---------|-----|
| Allocation in hot loop | Reuse buffers, pre-size |
| Lock per operation | Batch under single lock |
| O(N) where O(1) works | Hash tables, precompute |
| Logging in hot path | Remove or sample 1-in-N |
| Creating clients per request | Reuse/pool clients |
| Sequential I/O | Parallelize independent calls |

## Sub-Agent Optimization

This plugin is optimized for use as a Claude Code sub-agent:

1. **Efficient tool usage** - Predefined grep patterns for each analysis mode
2. **Token budget awareness** - Aims for <20 file reads per analysis
3. **Structured outputs** - Consistent table formats for each mode
4. **File:line citations** - Every finding references exact location
5. **Language-specific patterns** - Optimized detection for Python, TypeScript, Go, Rust, Java

## Structure

```
software-architect/
├── .claude-plugin/
│   ├── plugin.json          # Plugin metadata
│   └── marketplace.json     # Marketplace listing
├── agents/
│   └── software-architect.md # Agent definition with execution protocol
├── skills/
│   └── software-architect/
│       └── SKILL.md         # Complete analysis framework
└── README.md
```

## References

- [A Philosophy of Software Design](https://www.amazon.com/Philosophy-Software-Design-John-Ousterhout/dp/1732102201) - John Ousterhout
- [Abseil Performance Hints](https://abseil.io/fast/hints.html) - Jeff Dean / Google
- [Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) - Gary Bernhardt

## License

MIT | Ravi Gummadi | v3.0.0
