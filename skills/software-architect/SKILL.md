---
name: software-architect
user_invocable: true
description: Expert software architecture guidance for day-to-day code writing and system design. Use when reviewing code, designing modules/classes, making decomposition decisions, evaluating complexity, or evolving a codebase over time. Provides strategic thinking about architecture from global system view down to implementation details, emphasizing complexity reduction, deep modules, information hiding, and clean abstractions.
---

# Software Architect

A seasoned software architecture advisor that helps you write better code and design better systems. This skill applies battle-tested principles from John Ousterhout's "A Philosophy of Software Design" combined with the Functional Core/Imperative Shell pattern.

## Core Philosophy

**The enemy is complexity.** Every design decision should be evaluated by whether it increases or decreases the complexity of the system. Complexity accumulates incrementally through hundreds of small decisions.

**Complexity defined:** Anything that makes software hard to understand or modify. It manifests as:
1. **Change amplification** — small changes require touching many places
2. **Cognitive load** — how much you must know to make a change
3. **Unknown unknowns** — not knowing what you don't know (the worst)

**Root causes of complexity:**
- **Dependencies** — code cannot be understood/modified in isolation
- **Obscurity** — important information is not obvious

## Strategic vs Tactical Mindset

**Tactical programming** (avoid): Focus on getting features working quickly. Creates technical debt that compounds over time.

**Strategic programming** (embrace): Invest 10-20% of time in design improvements. Working code isn't enough—your primary goal is a great design that happens to work.

> "The most important thing is the long-term structure of the system."

## The Deep Module Principle

The most important structural principle: **Modules should be deep**.

```
        Interface (cost)
    ┌─────────────────────┐
    │                     │
    │   Implementation    │  ← Depth = benefit
    │    (functionality)  │
    │                     │
    └─────────────────────┘
```

**Deep module:** Simple interface, significant functionality hidden behind it. Good abstraction.

**Shallow module:** Interface complexity approaches implementation complexity. Poor leverage.

**Example of deep:** Unix file I/O — 5 system calls (open, read, write, lseek, close) hide hundreds of thousands of lines managing disk blocks, caching, permissions, etc.

**Example of shallow:** A class with many getters/setters that just expose internal state.

## Functional Core, Imperative Shell

Separate pure logic from side effects:

**Functional Core:**
- Pure functions with no side effects
- Operates only on data passed to it
- Contains all business logic
- Easy to test in isolation

**Imperative Shell:**
- Handles I/O, databases, network, external state
- Thin wrapper that orchestrates the functional core
- Difficult to test but minimal logic

```python
# Functional Core (pure, testable)
def get_expired_users(users: list[User], cutoff: date) -> list[User]:
    return [u for u in users if u.subscription_end <= cutoff and not u.is_trial]

def generate_expiry_emails(users: list[User]) -> list[tuple[str, str]]:
    return [(u.email, f"Your account has expired {u.name}.") for u in users]

# Imperative Shell (side effects)
email.bulk_send(generate_expiry_emails(get_expired_users(db.get_users(), date.today())))
```

## Information Hiding

**Hide design decisions** within modules. Each module should encapsulate knowledge that doesn't need to be exposed in its interface.

**Information leakage** (red flag): Same design decision reflected in multiple modules. Creates hidden dependencies.

**Temporal decomposition** (red flag): Structuring code around the order operations execute rather than around information hiding.

**Fix leakage by:**
- Merging classes that share information
- Creating a new class that encapsulates the shared knowledge
- Ensuring one module owns each piece of knowledge

## Key Design Principles

### 1. Pull Complexity Downward
If complexity is unavoidable, handle it internally rather than exposing it to callers. The module developer should suffer, not the users.

```python
# Bad: Caller must handle complexity
def get_user(id): 
    if not valid_id(id): raise InvalidIdError()
    ...

# Good: Module handles complexity internally  
def get_user(id):
    if not valid_id(id): return None  # or default user
    ...
```

### 2. Define Errors Out of Existence
Redefine semantics so exceptional conditions become normal cases.

```python
# Error-prone: substring throws if indices out of range
s[start:end]  # IndexError if bad indices

# Better: Return what exists within the range (Python already does this)
s[100:200]  # Returns "" if s is shorter, no error
```

### 3. Design It Twice
For any significant design decision, consider at least two radically different approaches. Compare pros/cons. Often leads to a better third option.

### 4. Different Layer, Different Abstraction
Each layer in a system should provide a different abstraction than layers above/below. If two adjacent layers have similar abstractions, there's probably an opportunity to combine them.

**Pass-through methods** (red flag): Methods that do little except call another method with similar signature. Indicates shallow decomposition.

### 5. General-Purpose Modules Are Deeper
When designing a module, make it somewhat more general than immediately needed:
- What is the simplest interface that covers all current needs?
- In how many situations will this method be used?
- Is this API easy to use for my current needs?

### 6. Separate General-Purpose and Special-Purpose Code
General mechanisms should not contain special-case code. Special-purpose code should live in modules associated with that particular purpose.

## Red Flags Checklist

When reviewing code, watch for these warning signs:

| Red Flag | Symptom |
|----------|---------|
| **Shallow Module** | Interface nearly as complex as implementation |
| **Information Leakage** | Same knowledge in multiple modules |
| **Temporal Decomposition** | Structure follows execution order, not information |
| **Pass-Through Method** | Just passes args to another similar method |
| **Repetition** | Same code pattern appears multiple times |
| **Special-General Mixture** | Special-purpose code mixed with general mechanism |
| **Conjoined Methods** | Can't understand one without understanding another |
| **Hard to Name** | Difficulty naming suggests unclear purpose |
| **Nonobvious Code** | Behavior not clear from quick reading |

## Architecture Review Questions

When evaluating a design or code change, ask:

1. **Complexity:** Does this increase or decrease overall complexity?
2. **Depth:** Is this module deep (simple interface, rich functionality)?
3. **Information hiding:** What knowledge is encapsulated? What leaks?
4. **Abstraction quality:** Does the interface reflect what matters to users?
5. **Dependencies:** What new dependencies does this create?
6. **Evolution:** How will this design handle future changes?
7. **Separation:** Is pure logic separated from side effects?

## When to Split vs Combine

**Bring together if:**
- Components share information
- Combining simplifies the interface
- Eliminates duplication
- Hard to understand one without the other

**Keep separate if:**
- Truly independent (no shared knowledge)
- General-purpose vs special-purpose
- Different rates of change
- Combining would create a "god class"

**Split methods only if:**
- Results in cleaner abstractions overall
- Child method is general-purpose and independent
- Don't split just because of length—depth matters more

## Comments Philosophy

Write comments that describe things **not obvious from the code**:
- **Interface comments:** Abstraction, what not how
- **Implementation comments:** What and why, not how
- Lower-level comments add precision
- Higher-level comments enhance intuition

**Write comments first:** Use comments as a design tool. If hard to describe, the design may need work.

## Code Obviousness

Make code obvious so readers' first guesses are correct:
- Choose precise, meaningful names
- Be consistent in similar situations
- Use whitespace to reveal structure
- Document non-obvious behavior

> "Obvious is in the mind of the reader."

## Workflow Integration

### Before Writing Code
1. Identify the abstraction you're creating
2. Design the interface first (write the comments/docstrings)
3. Consider at least two different approaches
4. Ask: What information can be hidden?

### During Code Review
1. Check for red flags (see checklist above)
2. Evaluate module depth
3. Look for information leakage
4. Assess separation of pure logic from side effects
5. Consider: Is this strategic or tactical?

### When Modifying Existing Code
1. Stay strategic—don't just patch
2. Look for opportunities to improve design
3. Keep comments near the code they describe
4. Maintain consistency with existing patterns

## References

For detailed examples and extended discussion, see:
- [references/principles.md](references/principles.md) — Full design principles with examples
- [references/red-flags.md](references/red-flags.md) — Detailed red flag explanations
- [references/patterns.md](references/patterns.md) — Common patterns and anti-patterns
