# Red Flags Reference

Detailed explanations of warning signs that indicate design problems.

## Table of Contents
1. [Shallow Module](#shallow-module)
2. [Information Leakage](#information-leakage)
3. [Temporal Decomposition](#temporal-decomposition)
4. [Overexposure](#overexposure)
5. [Pass-Through Method](#pass-through-method)
6. [Repetition](#repetition)
7. [Special-General Mixture](#special-general-mixture)
8. [Conjoined Methods](#conjoined-methods)
9. [Hard to Name](#hard-to-name)
10. [Nonobvious Code](#nonobvious-code)

---

## Shallow Module

**Symptom:** Interface complexity approaches implementation complexity. The module doesn't hide much.

**Why it's bad:** You pay the cost (learning the interface) without getting the benefit (hidden complexity).

**Example:**
```python
# SHALLOW: Interface reveals everything
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def get_x(self): return self.x
    def set_x(self, x): self.x = x
    def get_y(self): return self.y
    def set_y(self, y): self.y = y

# Just use the data directly—class adds no value
```

**Deeper alternative:**
```python
# DEEP: Simple interface, rich behavior
class Shape:
    def contains_point(self, x, y) -> bool: ...
    def intersects(self, other: 'Shape') -> bool: ...
    def bounding_box(self) -> tuple: ...
    # Implementation handles complex geometry internally
```

**Fix:** Combine shallow modules, or ask if the module is necessary at all.

---

## Information Leakage

**Symptom:** Same design decision (data format, algorithm, protocol) appears in multiple modules.

**Why it's bad:** Changes to that decision require changes in multiple places. Creates hidden coupling.

**Example:**
```python
# LEAKAGE: JSON format knowledge in multiple places
class UserAPI:
    def create_user(self, request):
        data = json.loads(request.body)
        name = data['user']['name']  # Knows JSON structure
        ...

class UserValidator:
    def validate(self, request):
        data = json.loads(request.body)
        if 'user' not in data:  # Also knows JSON structure
            raise ValidationError()
```

**Fix:** Centralize the knowledge:
```python
class UserRequest:
    """Single place that knows the request format."""
    def __init__(self, raw_body):
        self._data = json.loads(raw_body)
    
    @property
    def name(self):
        return self._data.get('user', {}).get('name')
    
    def is_valid(self):
        return 'user' in self._data and 'name' in self._data['user']
```

---

## Temporal Decomposition

**Symptom:** Code structure mirrors execution order rather than information organization.

**Why it's bad:** Operations at different times often share information, causing leakage.

**Example:**
```python
# TEMPORAL: Structured by "first read, then parse, then process"
class FileReader:
    def read(self, path) -> bytes: ...

class FileParser:
    def parse(self, data: bytes) -> dict: ...

class FileProcessor:
    def process(self, parsed: dict) -> Result: ...

# Problem: All three need to know file format details!
```

**Fix:** Structure by information, not time:
```python
class ConfigFile:
    """Encapsulates all knowledge of config file format."""
    def __init__(self, path):
        self._load_and_parse(path)
    
    def get_setting(self, key): ...
    def save(self): ...
```

---

## Overexposure

**Symptom:** API forces callers to be aware of rarely-used features to use common features.

**Why it's bad:** Increases cognitive load for the common case.

**Example:**
```java
// OVEREXPOSED: Must know about buffering to read a file
FileInputStream fileStream = new FileInputStream(fileName);
BufferedInputStream bufferedStream = new BufferedInputStream(fileStream);
ObjectInputStream objectStream = new ObjectInputStream(bufferedStream);
```

**Fix:** Make common case simple, hide rare options:
```python
# Good: Buffering is default, rare cases can override
def open_file(path, buffered=True, buffer_size=None):
    ...

# Most callers just do:
f = open_file("data.txt")
```

---

## Pass-Through Method

**Symptom:** Method does little except call another method with similar signature.

**Why it's bad:** Adds interface complexity without adding functionality. Sign of wrong abstraction boundaries.

**Example:**
```python
# PASS-THROUGH
class Controller:
    def get_user(self, user_id):
        return self.service.get_user(user_id)
    
    def save_user(self, user):
        return self.service.save_user(user)
```

**Questions to ask:**
1. Should the layers be combined?
2. Should responsibilities be redistributed?
3. Should the interface be different at each layer?

**Valid pass-through (adds value):**
```python
class Controller:
    def get_user(self, user_id):
        # Adds logging, metrics, caching
        self.metrics.record("get_user")
        cached = self.cache.get(user_id)
        if cached:
            return cached
        user = self.service.get_user(user_id)
        self.cache.set(user_id, user)
        return user
```

---

## Repetition

**Symptom:** Same code pattern (or nearly the same) appears multiple times.

**Why it's bad:** 
- Bug fixes must be applied in multiple places
- Easy to miss one instance
- Suggests missing abstraction

**Example:**
```python
# REPETITION
def process_user_csv(path):
    with open(path) as f:
        reader = csv.reader(f)
        next(reader)  # skip header
        for row in reader:
            ...

def process_order_csv(path):
    with open(path) as f:
        reader = csv.reader(f)
        next(reader)  # skip header
        for row in reader:
            ...
```

**Fix options:**

1. **Extract helper function:**
```python
def read_csv_rows(path):
    with open(path) as f:
        reader = csv.reader(f)
        next(reader)  # skip header
        yield from reader
```

2. **Use goto to single location (in languages that support it):**
```c
// Multiple error cases jump to single cleanup
if (error1) goto cleanup;
if (error2) goto cleanup;
...
cleanup:
    free(resources);
    return error_code;
```

---

## Special-General Mixture

**Symptom:** General-purpose mechanism contains code for specific use case.

**Why it's bad:** 
- Pollutes the general mechanism with special concerns
- Creates coupling between mechanism and use case
- Makes both harder to understand and modify

**Example:**
```python
# MIXED: General text class has UI-specific method
class TextBuffer:
    def insert(self, pos, text): ...
    def delete(self, start, end): ...
    def handle_backspace_key(self):  # UI-specific!
        if self.cursor > 0:
            self.delete(self.cursor - 1, self.cursor)
            self.cursor -= 1
```

**Fix:** Keep general mechanism pure, put special code where context is known:
```python
# General mechanism
class TextBuffer:
    def insert(self, pos, text): ...
    def delete(self, start, end): ...

# UI layer has context about keys and cursors
class TextEditor:
    def handle_backspace(self):
        if self.cursor > 0:
            self.buffer.delete(self.cursor - 1, self.cursor)
            self.cursor -= 1
```

---

## Conjoined Methods

**Symptom:** Can't understand one method without understanding another.

**Why it's bad:** Increases cognitive load. The apparent interface (one method) is smaller than the actual interface (multiple coupled methods).

**Example:**
```python
# CONJOINED: Must understand both to use either
class Parser:
    def begin_parse(self, data):
        self._buffer = data
        self._pos = 0
    
    def next_token(self):
        # Assumes begin_parse was called
        # Modifies _pos, must be called in loop
        ...
```

**Fix:** Make methods self-contained:
```python
class Parser:
    def parse(self, data) -> list[Token]:
        """Complete operation, no hidden state."""
        tokens = []
        pos = 0
        while pos < len(data):
            token, pos = self._extract_token(data, pos)
            tokens.append(token)
        return tokens
```

---

## Hard to Name

**Symptom:** Struggling to find a precise, intuitive name for a class, method, or variable.

**Why it matters:** Difficulty naming often indicates:
- The entity has unclear purpose
- It's doing too many things
- Abstraction boundaries are wrong

**Examples of vague names:**
```python
data = ...        # What data?
result = ...      # Result of what?
process(item)     # Process how?
handle(event)     # Handle how?
manager = ...     # Manages what? How?
utils.py          # Grab bag of unrelated things
```

**Better names:**
```python
user_preferences = ...
validation_errors = ...
compress_image(image)
route_to_handler(request)
connection_pool = ...
```

**If naming is hard:** Reconsider the design. Maybe split the entity, or combine it with something else, or rethink its responsibilities.

---

## Nonobvious Code

**Symptom:** Code's behavior or meaning isn't clear from a quick reading.

**Why it's bad:** 
- Readers waste time understanding
- Misunderstandings cause bugs
- Future maintainers will struggle

**Types of non-obviousness:**

1. **Hidden dependencies:**
```python
# Non-obvious: must call init before process
def init():
    global _state
    _state = load_config()

def process(x):
    return _state.transform(x)  # Fails if init not called
```

2. **Magic values:**
```python
if status == 3:  # What is 3?
    ...
```

3. **Unexpected side effects:**
```python
def get_user(id):  # "get" implies read-only
    user = db.fetch(id)
    user.last_accessed = now()  # Surprise! Modifies data
    db.save(user)
    return user
```

4. **Complex control flow:**
```python
for item in items:
    if condition1:
        if condition2:
            if not condition3:
                for sub in item.subs:
                    if sub.valid:
                        ...
```

**Fixes:**
- Use clear, precise names
- Add comments for non-obvious behavior
- Simplify control flow (extract methods, use early returns)
- Follow conventions readers expect
- Make side effects obvious in names (save_user, update_timestamp)
