# Software Architect Plugin

Expert software architecture and performance guidance based on John Ousterhout's **"A Philosophy of Software Design"** and **Jeff Dean's Performance Hints** from Abseil.

## Overview

This plugin helps you write better code and design better systems by applying battle-tested architectural and performance principles. Use it for code reviews, module design, decomposition decisions, complexity analysis, and identifying inefficient patterns.

## Installation

### From GitHub

```bash
# Add the marketplace
claude plugins add-marketplace https://github.com/ravigummadi/claude-code-plugins

# Install the plugin
claude plugins install software-architect@ravigummadi
```

### Local Installation

```bash
# Clone the repository
git clone https://github.com/ravigummadi/software-architect-plugin.git ~/.claude/plugins/software-architect

# Enable in Claude Code
claude plugins enable software-architect@local
```

## Usage

### Skill: `/software-architect`

Invoke the skill for architecture guidance:

```bash
# General architecture review
/software-architect

# Specific analysis modes
/software-architect red-flags      # Scan for 9 architectural red flags
/software-architect depth          # Evaluate module depth
/software-architect complexity     # Analyze complexity manifestations
/software-architect fcis           # Verify Functional Core / Imperative Shell
/software-architect split          # Advise on splitting or combining code
/software-architect performance    # Scan for inefficient patterns
/software-architect review <file>  # Review specific file
```

### Agent: `software-architect`

The agent can be spawned for autonomous codebase analysis:

```
"Launch the software-architect agent to review this codebase for FCIS violations"
```

## Core Principles

### The Enemy is Complexity

Complexity manifests as:
1. **Change amplification** - Small changes require touching many places
2. **Cognitive load** - How much you must know to make a change
3. **Unknown unknowns** - Not knowing what you don't know (the worst)

### Deep Modules

```
        Interface (cost)
    +-----------------------+
    |                       |
    |    Implementation     |  <- Depth = benefit
    |     (functionality)   |
    |                       |
    +-----------------------+
```

**Deep modules**: Simple interface, significant functionality hidden. GOOD.
**Shallow modules**: Interface complexity approaches implementation. BAD.

### Functional Core / Imperative Shell

```python
# Functional Core (pure, testable)
def get_expired_users(users: list[User], cutoff: date) -> list[User]:
    return [u for u in users if u.subscription_end <= cutoff]

# Imperative Shell (side effects)
email.send(get_expired_users(db.get_users(), date.today()))
```

- **Core**: Pure functions, no side effects, easy to test
- **Shell**: I/O, databases, network - keep thin

### The 9 Red Flags

| Red Flag | Symptom |
|----------|---------|
| Shallow Module | Interface nearly as complex as implementation |
| Information Leakage | Same knowledge in multiple modules |
| Temporal Decomposition | Structure follows execution order |
| Pass-Through Method | Just passes args to another similar method |
| Repetition | Same code pattern appears multiple times |
| Special-General Mixture | Special-purpose mixed with general mechanism |
| Conjoined Methods | Can't understand one without another |
| Hard to Name | Difficulty naming suggests unclear purpose |
| Nonobvious Code | Behavior not clear from quick reading |

### Performance Red Flags (Jeff Dean's Hints)

| Pattern | Problem | Fix |
|---------|---------|-----|
| Allocation in hot loop | Memory pressure | Reuse buffers, pre-size |
| Lock per operation | Contention overhead | Batch, sharding |
| Protobuf in hot path | 20X slower than structs | Use plain structs |
| Logging in hot path | Cost even when disabled | Remove or sample |
| O(N) where O(1) possible | Algorithmic waste | Hash tables |
| Stats on every operation | Overhead accumulates | Sample (1 in 32) |

**Key Numbers:**
| Operation | Latency |
|-----------|---------|
| L1 cache | 0.5 ns |
| Main memory | 50-100 ns |
| SSD read | 150 μs |
| Network (DC) | 0.5 ms |

## Output Examples

### Architecture Review
```
## Summary
Well-structured codebase following FCIS pattern.

## Architecture Score: 8.5/10

## Issues Found

### Information Leakage - Medium
- **Location**: src/api/handler.py:45, src/cli/commands.py:89
- **Problem**: Config parsing duplicated in both modules
- **Fix**: Extract to shared config module

## Strengths
- Clean separation of pure logic and I/O
- Deep modules with simple interfaces

## Recommendations
1. Extract shared config parsing
2. Add validation to shell layer
```

### Red Flags Table
| Flag | Location | Severity | Fix |
|------|----------|----------|-----|
| Shallow Module | utils.py | Low | Merge into caller |
| Repetition | validators/*.py | Medium | Extract base class |

### FCIS Verification
| Layer | Module | Status | Notes |
|-------|--------|--------|-------|
| Core | rules.py | PASS | Pure functions |
| Core | formatter.py | PASS | No side effects |
| Shell | api_client.py | PASS | HTTP only |
| Shell | db.py | VIOLATION | Contains business logic |

## Plugin Structure

```
software-architect/
├── .claude-plugin/
│   └── plugin.json            # Plugin metadata
├── agents/
│   └── software-architect.md  # Agent definition
├── skills/
│   └── software-architect/
│       ├── SKILL.md           # Core principles
│       └── references/
│           ├── principles.md
│           ├── red-flags.md
│           ├── patterns.md
│           └── performance-hints.md  # Jeff Dean's hints
└── README.md
```

## When to Use

**Use for:**
- Code reviews and refactoring decisions
- Designing new modules or classes
- Evaluating complexity and technical debt
- Verifying architectural patterns

**Don't use for:**
- Trivial bug fixes
- Style/formatting issues
- Language-specific syntax questions

## References

- [A Philosophy of Software Design](https://www.amazon.com/Philosophy-Software-Design-John-Ousterhout/dp/1732102201) by John Ousterhout
- [Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) by Gary Bernhardt
- [Google Testing Blog: FCIS](https://testing.googleblog.com/2025/10/simplify-your-code-functional-core.html)
- [Abseil Performance Hints](https://abseil.io/fast/hints.html) by Jeff Dean & Google

## Author

Ravi Gummadi

## License

MIT

## Version

1.1.0
