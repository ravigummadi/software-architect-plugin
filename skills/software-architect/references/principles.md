# Design Principles Reference

Extended explanations and examples for core software design principles.

## Table of Contents
1. [Complexity Is Incremental](#complexity-is-incremental)
2. [Deep Modules](#deep-modules)
3. [Information Hiding](#information-hiding)
4. [Pull Complexity Downward](#pull-complexity-downward)
5. [Define Errors Out of Existence](#define-errors-out-of-existence)
6. [General-Purpose Modules](#general-purpose-modules)
7. [Different Layer, Different Abstraction](#different-layer-different-abstraction)
8. [Functional Core, Imperative Shell](#functional-core-imperative-shell)

---

## Complexity Is Incremental

Complexity isn't caused by a single catastrophic error—it accumulates in small chunks. A single dependency or obscurity by itself won't significantly affect maintainability. But hundreds of small issues compound.

**The zero-tolerance philosophy:** Since complexity accumulates incrementally, you must fight it incrementally. Each small compromise compounds. Each small improvement compounds too.

**Investment mindset:** Spend 10-20% of development time on design improvements. This pays for itself within months as future development speeds up.

```python
# Each "small" compromise compounds
class UserService:
    # "Just this once" - exposing internal state
    def get_db_connection(self): return self._conn  # Leak #1
    
    # "It's faster this way" - skipping validation  
    def quick_save(self, data): self._conn.execute(data)  # Leak #2
    
    # "We'll refactor later" - mixing concerns
    def save_and_email(self, user): ...  # Leak #3
    
# After 50 such decisions, the class is unmaintainable
```

---

## Deep Modules

**Core principle:** The best modules provide powerful functionality behind simple interfaces.

```
    DEEP MODULE                    SHALLOW MODULE
    ┌──────────┐                   ┌────────────────────┐
    │ interface│                   │     interface      │
    ├──────────┤                   ├────────────────────┤
    │          │                   │   implementation   │
    │  impl    │                   └────────────────────┘
    │          │
    │          │
    └──────────┘
```

**Unix file I/O as canonical example:**
```c
int fd = open("/path/file", O_RDONLY);  // Simple!
read(fd, buffer, size);
write(fd, buffer, size);
lseek(fd, offset, SEEK_SET);
close(fd);
```

These 5 calls hide:
- Disk block allocation strategies
- Directory traversal and path resolution
- Permission checking and enforcement
- Buffer caching and write-back policies
- File system format details
- Multiple storage device types

**Shallow module anti-example:**
```python
class LinkedList:
    def insert_after(self, node, value): ...
    def insert_before(self, node, value): ...
    def delete(self, node): ...
    def find(self, value): ...
```
The interface reveals almost everything about the implementation. Callers must understand linked list internals to use it effectively.

**Classitis:** The mistaken belief that "classes are good, so more classes are better." Results in many shallow classes with accumulated interface complexity.

---

## Information Hiding

Each module should encapsulate design decisions, exposing only what's necessary in its interface.

**What to hide:**
- Data structures and their formats
- Algorithms and their implementation details
- Lower-level details (sizes, offsets, encoding)
- Mechanisms for achieving functionality

**Information leakage example:**
```python
# BAD: Both classes know about the file format
class FileReader:
    def read(self, path):
        with open(path) as f:
            header = f.read(4)  # Knows format
            if header != "MAGIC":
                raise ValueError()
            ...

class FileWriter:
    def write(self, path, data):
        with open(path, 'w') as f:
            f.write("MAGIC")  # Also knows format
            ...

# GOOD: Encapsulate format knowledge
class FileFormat:
    MAGIC = "MAGIC"
    
    @classmethod
    def validate_header(cls, data): ...
    
    @classmethod  
    def write_header(cls, file): ...
```

**Temporal decomposition trap:**
```python
# BAD: Structured by execution order
class RequestReader:    # Step 1: Read
    def read(self): ...

class RequestParser:    # Step 2: Parse  
    def parse(self): ...

class RequestValidator: # Step 3: Validate
    def validate(self): ...

# All three need to know request format = leakage!

# GOOD: Structured by information
class HttpRequest:
    def __init__(self, raw_data):
        self._parse_and_validate(raw_data)  # All format knowledge here
```

---

## Pull Complexity Downward

When complexity is unavoidable, handle it within the module rather than pushing it to callers. Module developers should suffer so users don't have to.

**Configuration parameters as complexity escape:**
```python
# BAD: Punting decisions to callers
class Cache:
    def __init__(self, size, eviction_policy, ttl, 
                 max_memory, shard_count, ...):
        ...

# GOOD: Smart defaults, measure and adapt
class Cache:
    def __init__(self, hint_size=None):
        # Automatically tune based on memory/usage
        self._size = hint_size or self._compute_optimal_size()
```

**Exception handling as complexity escape:**
```python
# BAD: Every caller must handle this
def get_user(user_id):
    if not exists(user_id):
        raise UserNotFoundError()
    ...

# GOOD: Handle it internally when possible
def get_user(user_id, default=None):
    if not exists(user_id):
        return default
    ...
```

---

## Define Errors Out of Existence

The best way to handle exceptions is to define your API so they don't occur.

**Example: Tcl's unset command**
```tcl
# Original (error-prone): Throws if variable doesn't exist
unset myvar  ;# Error if myvar not defined

# Better definition: "Ensure variable doesn't exist"
# Now it's always successful—either it deleted the var or it was already gone
```

**Example: File deletion**
```
Windows: Can't delete file if it's open → error
Unix: Mark file for deletion, let processes finish → no error

Unix defines the error out of existence by changing semantics.
```

**Example: Substring**
```python
# Java: Throws IndexOutOfBoundsException
s.substring(100, 200)  # Fails if s is shorter

# Python: Returns what exists
s[100:200]  # Returns "" if out of range—no error
```

**When NOT to define errors away:** When the caller genuinely needs to know about the condition. Don't hide important information.

---

## General-Purpose Modules

Somewhat general-purpose interfaces are deeper because they:
- Cover more use cases with the same interface
- Hide more implementation details
- Are more likely to be reused

**Questions to ask:**
1. What is the simplest interface covering all current needs?
2. In how many situations will this be used?
3. Is this easy to use for my current needs?

```python
# BAD: Special-purpose text editor methods
class TextBuffer:
    def backspace(self): ...           # UI-specific
    def delete_selection(self): ...    # UI-specific
    def insert_at_cursor(self, ch): ...# UI-specific

# GOOD: General-purpose operations
class TextBuffer:
    def insert(self, position, text): ...
    def delete(self, start, end): ...
    def get_text(self, start, end): ...
    
# UI layer builds backspace from: delete(cursor-1, cursor)
```

**Push specialization upward:** General-purpose code at lower layers, special-purpose code at higher layers where specific context is known.

---

## Different Layer, Different Abstraction

Each layer should provide a different abstraction. If adjacent layers have similar abstractions, something is wrong.

**Pass-through method smell:**
```python
# BAD: Method just delegates
class UserController:
    def get_user(self, id):
        return self.user_service.get_user(id)  # Just passes through!

# Either:
# 1. The controller is unnecessary (remove it)
# 2. The abstractions are wrong (redesign)
# 3. The controller should add value (validation, transformation, etc.)
```

**Pass-through variable smell:**
```python
# BAD: Variable threaded through many layers
def handle_request(request, config):  # config passed but not used
    return process(request, config)

def process(request, config):  # config passed but not used
    return validate(request, config)
    
def validate(request, config):  # finally used here
    if config.strict_mode: ...
```

**Solutions:**
- Store in shared context/object
- Use global/singleton for truly global config
- Restructure so caller and user are closer

---

## Functional Core, Imperative Shell

Separate pure business logic from side effects for testability and clarity.

**Structure:**
```
┌─────────────────────────────────────┐
│         Imperative Shell            │
│  (I/O, databases, network, state)   │
│                                     │
│    ┌─────────────────────────────┐  │
│    │     Functional Core         │  │
│    │  (pure business logic)      │  │
│    │  - no side effects          │  │
│    │  - only operates on inputs  │  │
│    │  - returns new data         │  │
│    └─────────────────────────────┘  │
│                                     │
└─────────────────────────────────────┘
```

**Extended example:**
```python
# ===== FUNCTIONAL CORE =====
# Pure functions, easily tested, no mocking needed

from dataclasses import dataclass
from datetime import date
from typing import List, Tuple

@dataclass(frozen=True)
class User:
    id: str
    email: str
    subscription_end: date
    is_trial: bool

@dataclass(frozen=True)  
class EmailMessage:
    to: str
    subject: str
    body: str

def find_expired_users(users: List[User], today: date) -> List[User]:
    """Pure function: filters users whose subscription has expired."""
    return [u for u in users 
            if u.subscription_end <= today and not u.is_trial]

def create_expiry_notification(user: User) -> EmailMessage:
    """Pure function: creates email content for one user."""
    return EmailMessage(
        to=user.email,
        subject="Subscription Expired",
        body=f"Dear {user.id}, your subscription has expired."
    )

def generate_all_notifications(users: List[User], today: date) -> List[EmailMessage]:
    """Pure function: composes the above."""
    expired = find_expired_users(users, today)
    return [create_expiry_notification(u) for u in expired]


# ===== IMPERATIVE SHELL =====
# Side effects here, minimal logic

class NotificationService:
    def __init__(self, db, email_client):
        self.db = db
        self.email_client = email_client
    
    def send_expiry_notifications(self):
        # Shell: get data (side effect)
        users = self.db.get_all_users()
        today = date.today()
        
        # Core: pure logic
        emails = generate_all_notifications(users, today)
        
        # Shell: send emails (side effect)
        for email in emails:
            self.email_client.send(email)


# ===== TESTING =====
# Core is trivial to test—no mocks needed!

def test_find_expired_users():
    users = [
        User("1", "a@test.com", date(2024, 1, 1), False),  # expired
        User("2", "b@test.com", date(2024, 12, 31), False), # not expired
        User("3", "c@test.com", date(2024, 1, 1), True),   # trial
    ]
    result = find_expired_users(users, date(2024, 6, 1))
    assert len(result) == 1
    assert result[0].id == "1"
```

**Benefits:**
- Core logic is trivially testable (just call functions with data)
- Side effects are isolated and obvious
- Business rules are explicit and documented by pure functions
- Easy to reason about behavior
- Shell can be swapped (different DB, different email provider)
