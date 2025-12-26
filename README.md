# Software Architect Plugin

Expert architecture analysis based on John Ousterhout's **"A Philosophy of Software Design"** and **Jeff Dean's Performance Hints**.

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
/software-architect              # General architecture review
/software-architect red-flags    # Scan for 9 red flags
/software-architect depth        # Module depth analysis
/software-architect complexity   # Complexity metrics
/software-architect fcis         # Core/shell separation check
/software-architect performance  # Performance anti-patterns
/software-architect split        # Split vs combine advice
/software-architect review <file> # Review specific file
```

## Core Concepts

### Complexity (The Enemy)
1. **Change amplification** - small changes touch many places
2. **Cognitive load** - too much to know
3. **Unknown unknowns** - not knowing what you don't know

### Deep Modules
Simple interface, significant hidden functionality = good.

### FCIS Pattern
- **Core**: Pure functions, no I/O, easy to test
- **Shell**: I/O only, thin, orchestrates core

### 9 Red Flags
| Flag | Symptom |
|------|---------|
| Shallow Module | Interface ~ implementation |
| Information Leakage | Same knowledge in multiple modules |
| Temporal Decomposition | Structure follows execution order |
| Pass-Through Method | Just delegates |
| Repetition | Same pattern repeated |
| Special-General Mixture | Mixed concerns |
| Conjoined Methods | Can't understand separately |
| Hard to Name | Unclear purpose |
| Nonobvious Code | Behavior unclear |

### Performance Red Flags
| Pattern | Fix |
|---------|-----|
| Allocation in hot loop | Reuse buffers |
| Lock per operation | Batch |
| O(N) where O(1) works | Hash tables |
| Logging in hot path | Sample |

## Structure

```
software-architect/
├── .claude-plugin/plugin.json
├── agents/software-architect.md
├── skills/software-architect/SKILL.md
└── README.md
```

## References

- [A Philosophy of Software Design](https://www.amazon.com/Philosophy-Software-Design-John-Ousterhout/dp/1732102201)
- [Abseil Performance Hints](https://abseil.io/fast/hints.html)

## License

MIT | Ravi Gummadi | v2.0.0
