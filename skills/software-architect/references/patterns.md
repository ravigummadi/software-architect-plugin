# Patterns and Anti-Patterns Reference

Common patterns that reduce complexity and anti-patterns to avoid.

## Table of Contents
1. [Module Design Patterns](#module-design-patterns)
2. [Error Handling Patterns](#error-handling-patterns)
3. [Decomposition Patterns](#decomposition-patterns)
4. [Code Organization Patterns](#code-organization-patterns)
5. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Module Design Patterns

### Deep Module Pattern
Create modules with simple interfaces that hide significant complexity.

```python
# Example: Simple interface, complex implementation
class ImageProcessor:
    def resize(self, image: Image, width: int, height: int) -> Image:
        """Resize image to specified dimensions.
        
        Handles: aspect ratio, quality settings, format conversion,
        memory optimization, parallel processing for large images.
        """
        ...
```

### Defaults Pattern
Provide sensible defaults so common cases require minimal configuration.

```python
# BAD: Caller must specify everything
cache = Cache(
    size=1000,
    eviction="lru", 
    ttl=3600,
    serializer=JsonSerializer(),
    metrics=NullMetrics()
)

# GOOD: Sensible defaults
cache = Cache()  # Works out of the box

# Can customize if needed
cache = Cache(size=5000, ttl=7200)
```

### Information Encapsulation Pattern
Group related data and operations together, hiding format details.

```python
# Instead of passing raw dicts around:
user_data = {"name": "Alice", "email": "alice@example.com"}

# Encapsulate in a class:
class User:
    def __init__(self, name: str, email: str):
        self._name = name
        self._email = email
    
    @property
    def display_name(self) -> str:
        return self._name.title()
    
    def send_email(self, message: str) -> None:
        # Encapsulates email format knowledge
        ...
```

### Builder Pattern for Complex Construction
When object creation requires many parameters, use a builder to provide a fluent interface.

```python
# Instead of constructor with 10 parameters:
query = (QueryBuilder()
    .select("name", "email")
    .from_table("users")
    .where("active", True)
    .order_by("created_at", desc=True)
    .limit(10)
    .build())
```

---

## Error Handling Patterns

### Define Errors Out of Existence
Redefine semantics so the "error" case is a normal case.

```python
# BEFORE: Must handle KeyError
try:
    value = data[key]
except KeyError:
    value = default

# AFTER: No error to handle
value = data.get(key, default)
```

### Exception Aggregation
Handle many errors in a single place rather than scattered handlers.

```python
# BAD: Handler at each call site
try:
    param1 = get_param("param1")
except MissingParam:
    return error_response("Missing param1")

try:
    param2 = get_param("param2")  
except MissingParam:
    return error_response("Missing param2")

# GOOD: Single handler at top level
def handle_request(request):
    try:
        return process_request(request)
    except MissingParam as e:
        return error_response(f"Missing {e.param_name}")
    except ValidationError as e:
        return error_response(str(e))
```

### Exception Masking
Handle exceptions at low level so higher layers don't need to know.

```python
class NetworkClient:
    def send(self, data: bytes) -> bytes:
        """Send data and return response.
        
        Automatically retries on transient failures.
        Caller doesn't need to handle NetworkError.
        """
        for attempt in range(3):
            try:
                return self._socket.send(data)
            except NetworkError:
                if attempt == 2:
                    raise
                time.sleep(1)
```

### Result Type Pattern
For expected failures, return a result object instead of throwing.

```python
from dataclasses import dataclass
from typing import Generic, TypeVar, Union

T = TypeVar('T')

@dataclass
class Success(Generic[T]):
    value: T

@dataclass
class Failure:
    error: str

Result = Union[Success[T], Failure]

def parse_int(s: str) -> Result[int]:
    try:
        return Success(int(s))
    except ValueError:
        return Failure(f"Cannot parse '{s}' as integer")

# Caller handles explicitly
match parse_int(user_input):
    case Success(value):
        process(value)
    case Failure(error):
        show_error(error)
```

---

## Decomposition Patterns

### Extract Subtask Pattern
Pull out a coherent subtask into its own method/function.

```python
# BEFORE
def process_order(order):
    # 50 lines of validation logic
    if not order.items:
        raise ValidationError("No items")
    for item in order.items:
        if item.quantity <= 0:
            raise ValidationError("Invalid quantity")
        # ... more validation
    
    # 30 lines of pricing logic
    total = 0
    for item in order.items:
        price = get_price(item.sku)
        total += price * item.quantity
        # ... more pricing
    
    # 20 lines of persistence
    ...

# AFTER
def process_order(order):
    validate_order(order)
    total = calculate_total(order)
    save_order(order, total)

def validate_order(order):
    """Independent, reusable validation logic."""
    ...

def calculate_total(order) -> Decimal:
    """Pure function for pricing."""
    ...
```

### Layered Abstraction Pattern
Each layer provides a different abstraction level.

```
┌─────────────────────────┐
│   Application Layer     │  High-level operations
│   (use cases)           │  create_user(), process_payment()
├─────────────────────────┤
│   Domain Layer          │  Business entities and rules
│   (business logic)      │  User, Order, validate_order()
├─────────────────────────┤
│   Infrastructure Layer  │  Technical concerns
│   (persistence, I/O)    │  database, http, file system
└─────────────────────────┘
```

### Plugin/Strategy Pattern
When behavior varies, define an interface and provide implementations.

```python
from abc import ABC, abstractmethod

class StorageBackend(ABC):
    @abstractmethod
    def save(self, key: str, data: bytes) -> None: ...
    
    @abstractmethod
    def load(self, key: str) -> bytes: ...

class FileStorage(StorageBackend):
    def save(self, key, data):
        with open(key, 'wb') as f:
            f.write(data)

class S3Storage(StorageBackend):
    def save(self, key, data):
        self.client.put_object(key, data)

# Consumer doesn't know which implementation
class DataProcessor:
    def __init__(self, storage: StorageBackend):
        self.storage = storage
```

---

## Code Organization Patterns

### Write Comments First
Use comments as a design tool—write interface comments before implementation.

```python
def calculate_shipping(order: Order, destination: Address) -> ShippingQuote:
    """Calculate shipping cost and delivery estimate.
    
    Args:
        order: The order containing items to ship
        destination: Delivery address
        
    Returns:
        ShippingQuote containing cost and estimated delivery date
        
    The calculation considers:
        - Total weight and dimensions of items
        - Distance from nearest warehouse
        - Available shipping methods
        - Current carrier rates
    """
    # If this comment is hard to write, the design may be wrong
    ...
```

### Proximity Pattern
Keep related code close together.

```python
# BAD: Related code scattered
class User:
    def __init__(self, name):
        self.name = name
    
    # ... 100 lines of other code ...
    
    def validate_name(self):  # Related to __init__ but far away
        ...

# GOOD: Related code together
class User:
    def __init__(self, name):
        self._validate_name(name)
        self.name = name
    
    def _validate_name(self, name):
        """Called by __init__."""
        ...
```

### Consistency Pattern
Do similar things in similar ways throughout the codebase.

```python
# Pick a style and stick with it everywhere:

# Naming: always snake_case for functions
def get_user(): ...
def create_user(): ...
def delete_user(): ...

# Error handling: always raise exceptions (or always return Result)
# File structure: always one class per file (or always group related classes)
# Imports: always absolute (or always relative)
```

---

## Anti-Patterns to Avoid

### God Class/Module
**Problem:** One class does everything, knows everything.

```python
# BAD: Does too much
class ApplicationManager:
    def handle_request(self, request): ...
    def query_database(self, sql): ...
    def send_email(self, to, body): ...
    def generate_report(self, data): ...
    def validate_user(self, user): ...
    def cache_result(self, key, value): ...
```

**Fix:** Split into focused classes, each with single responsibility.

### Premature Generalization
**Problem:** Building flexibility for imagined future needs.

```python
# BAD: Over-engineered for "flexibility"
class DataProcessor:
    def __init__(self, 
                 reader_factory,
                 parser_factory, 
                 transformer_factory,
                 writer_factory,
                 plugin_manager):
        ...

# GOOD: Simple solution for current needs
class CsvToJsonConverter:
    def convert(self, csv_path: str, json_path: str): ...
```

**Rule:** Build for current needs. Refactor when actual new requirements emerge.

### Configuration Overload
**Problem:** Exposing every decision as a configuration parameter.

```python
# BAD: Every decision is a parameter
app = Application(
    thread_pool_size=10,
    max_connections=100,
    retry_count=3,
    retry_delay=1.0,
    timeout=30,
    buffer_size=8192,
    log_level="INFO",
    cache_ttl=3600,
    # ... 20 more parameters
)
```

**Fix:** Compute reasonable defaults, expose only what users genuinely need to vary.

### Anemic Domain Model
**Problem:** Data classes with no behavior, logic scattered elsewhere.

```python
# BAD: Data class with no behavior
@dataclass
class Order:
    items: list[Item]
    customer_id: str
    created_at: datetime

# Logic scattered in services
class OrderService:
    def calculate_total(self, order): ...
    def validate(self, order): ...
    def apply_discount(self, order, code): ...
```

**Fix:** Put behavior with data:
```python
class Order:
    def __init__(self, items, customer_id):
        self._validate_items(items)
        self.items = items
        self.customer_id = customer_id
    
    @property
    def total(self) -> Decimal:
        return sum(item.price for item in self.items)
    
    def apply_discount(self, code: str) -> None:
        ...
```

### Primitive Obsession
**Problem:** Using primitives where a domain concept exists.

```python
# BAD: Primitives everywhere
def create_user(email: str, phone: str, postal_code: str): ...

# Easy to pass wrong string to wrong parameter
create_user("555-1234", "alice@test.com", "user@domain.com")  # Compiles but wrong!

# GOOD: Domain types
def create_user(email: Email, phone: Phone, postal_code: PostalCode): ...
```

### Feature Envy
**Problem:** Method uses more features of another class than its own.

```python
# BAD: This method envies UserData
class OrderProcessor:
    def format_receipt(self, user_data):
        return f"""
        Name: {user_data.first_name} {user_data.last_name}
        Email: {user_data.email}
        Address: {user_data.street}, {user_data.city}, {user_data.zip}
        """

# GOOD: Move method to where data lives
class UserData:
    def format_for_receipt(self) -> str:
        return f"""
        Name: {self.first_name} {self.last_name}
        ...
        """
```
