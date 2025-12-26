# Performance Hints

Based on [Abseil Performance Hints](https://abseil.io/fast/hints.html) from Jeff Dean and the Google performance engineering team.

## Core Philosophy

> Focus on the "critical 3%" where performance matters most. Avoid premature optimization while maintaining awareness of efficiency in hot paths.

## Quick Reference: Performance Red Flags

| Pattern | Problem | Fix |
|---------|---------|-----|
| Allocation in hot loop | Memory pressure, fragmentation | Reuse buffers, pre-size containers |
| Lock per operation | Contention, overhead | Batch operations, sharding |
| Protobuf in hot path | 20X slower than structs | Use plain structs/arrays |
| Logging in hot path | Cost even when disabled | Remove or sample |
| Copying large objects | Memory bandwidth | Move semantics, views |
| O(N) where O(1) possible | Algorithmic waste | Hash tables, precomputation |
| Virtual calls in loop | Indirection cost | Devirtualize, templates |
| Stats on every operation | Overhead accumulates | Sample (1 in 32) |

---

## 1. Estimation & Measurement

### Know Your Numbers
| Operation | Latency |
|-----------|---------|
| L1 cache hit | 0.5 ns |
| L2 cache hit | 7 ns |
| Main memory | 50-100 ns |
| SSD random read | 150 μs |
| Disk seek | 5-10 ms |
| Network round-trip (same DC) | 0.5 ms |

### Profiling Strategy
- Use profilers (pprof, perf) to identify bottlenecks
- Look for functions with high cache miss rates
- Many small 1-3% improvements often beat one large optimization

---

## 2. API Design for Performance

### Bulk Operations
```python
# Bad: Individual operations
for item in items:
    store.delete(item)  # Lock acquired each time

# Good: Batch operation
store.delete_batch(items)  # Single lock acquisition
```

### Views Over Copies
```python
# Bad: Forces copy
def process(data: str) -> None: ...

# Good: Accepts view (no copy if caller has compatible type)
def process(data: str | bytes | memoryview) -> None: ...
```

### Thread-Compatible vs Thread-Safe
- Default to thread-compatible (caller manages sync)
- Only internalize locking when typical usage requires it
- Avoids overhead for single-threaded use cases

---

## 3. Algorithmic Improvements

### Data Structure Selection
| Need | Bad Choice | Good Choice | Improvement |
|------|------------|-------------|-------------|
| Key lookup | Binary tree O(lg N) | Hash table O(1) | 4X+ |
| Membership test | `set` | Bit vector | 26-31% |
| Small collection | `std::vector` | Inlined vector | Fewer allocs |
| Nested maps | `map<A, map<B,C>>` | `map<(A,B), C>` | Better cache |

### Precomputation
```python
# Bad: Repeated classification
for item in items:
    if expensive_classify(item) == SPECIAL:
        handle_special(item)

# Good: Precompute and store
class Item:
    is_special: bool  # Set once at creation
```

---

## 4. Memory Layout & Representation

### Compact Data Structures
```python
# Bad: Default enum (word-sized)
class OpType(Enum):
    READ = 1
    WRITE = 2

# Good: Explicit small size
class OpType(IntEnum):
    READ = 1
    WRITE = 2
    # Use uint8 in languages that support it
```

### Strategic Field Ordering
```python
# Bad: Mixed hot/cold fields
@dataclass
class Request:
    id: int           # Hot
    metadata: dict    # Cold
    status: int       # Hot
    audit_log: list   # Cold

# Good: Hot fields together
@dataclass
class Request:
    id: int           # Hot
    status: int       # Hot
    metadata: dict    # Cold (or behind indirection)
    audit_log: list   # Cold
```

### Indices Instead of Pointers
- 32-bit index vs 64-bit pointer saves 4 bytes per reference
- Better cache utilization in pointer-heavy structures

---

## 5. Allocation Optimization

### Pre-size Containers
```python
# Bad: Repeated reallocation
result = []
for i in range(1000):
    result.append(compute(i))

# Good: Pre-allocate
result = [None] * 1000
for i in range(1000):
    result[i] = compute(i)

# Or use list comprehension (Python optimizes this)
result = [compute(i) for i in range(1000)]
```

### Reuse Temporary Objects
```python
# Bad: Allocation per iteration
for data in dataset:
    buffer = BytesIO()  # New allocation each time
    process(data, buffer)

# Good: Reuse buffer
buffer = BytesIO()
for data in dataset:
    buffer.seek(0)
    buffer.truncate()
    process(data, buffer)
```

### Avoid Unnecessary Copies
```python
# Bad: Copy on sort
def get_sorted(items):
    return sorted(items)  # Creates new list

# Good: Sort in place if caller doesn't need original
def sort_in_place(items):
    items.sort()
    return items
```

---

## 6. Avoiding Unnecessary Work

### Fast Paths for Common Cases
```python
# Bad: Full validation always
def parse_int(s: str) -> int:
    return full_parser(s)  # Handles all edge cases

# Good: Fast path for common case
def parse_int(s: str) -> int:
    if len(s) == 1 and '0' <= s <= '9':
        return ord(s) - ord('0')  # Fast path
    return full_parser(s)  # Slow path
```

### Move Work Outside Loops
```python
# Bad: Repeated computation
for item in items:
    config = load_config()  # Same result each time!
    process(item, config)

# Good: Hoist invariant
config = load_config()
for item in items:
    process(item, config)
```

### Defer Expensive Computation
```python
# Bad: Eager stats update
class Allocator:
    def allocate(self, size):
        self.total_allocated += size
        self.allocation_count += 1
        self.update_histogram(size)  # Expensive!
        return self._allocate(size)

# Good: Lazy computation
class Allocator:
    def allocate(self, size):
        self._allocations.append(size)
        return self._allocate(size)

    def stats(self):  # Compute only when needed
        return Stats(self._allocations)
```

### Caching & Memoization
```python
from functools import lru_cache

@lru_cache(maxsize=1024)
def expensive_computation(key):
    return do_heavy_work(key)
```

---

## 7. Reduce Logging Overhead

### Remove from Hot Paths
```python
# Bad: Logging in hot loop (cost even when disabled)
for item in items:
    logger.debug(f"Processing {item}")  # String formatting happens!
    process(item)

# Good: Check once outside loop
if logger.isEnabledFor(logging.DEBUG):
    for item in items:
        logger.debug(f"Processing {item}")
        process(item)
else:
    for item in items:
        process(item)
```

---

## 8. Parallelization & Synchronization

### Amortize Lock Acquisition
```python
# Bad: Lock per item
def delete_items(items):
    for item in items:
        with self.lock:
            self._delete(item)

# Good: Single lock for batch
def delete_items(items):
    with self.lock:
        for item in items:
            self._delete(item)
```

### Reduce Contention Through Sharding
```python
# Bad: Single lock for all
class Cache:
    def __init__(self):
        self.lock = Lock()
        self.data = {}

# Good: Sharded locks (16 shards = 2X throughput)
class ShardedCache:
    def __init__(self, num_shards=16):
        self.shards = [{'lock': Lock(), 'data': {}} for _ in range(num_shards)]

    def _get_shard(self, key):
        return self.shards[hash(key) % len(self.shards)]
```

### Avoid False Sharing
- Place frequently-mutated fields on different cache lines
- Cache line is typically 64 bytes
- Use padding or alignment to separate hot mutable data

---

## 9. Sampling for Stats

```python
# Bad: Stats on every operation
def process(item):
    start = time.time()
    result = do_work(item)
    self.histogram.record(time.time() - start)  # Always
    return result

# Good: Sample 1 in 32
import random
def process(item):
    sample = random.randint(0, 31) == 0
    if sample:
        start = time.time()
    result = do_work(item)
    if sample:
        self.histogram.record(time.time() - start)
    return result
```

---

## 10. Protocol Buffer / Serialization Advice

| Tip | Impact |
|-----|--------|
| Use plain structs in hot paths | 20X faster than protobuf |
| Flatten message hierarchies | Fewer allocations |
| Fields 1-15 use 1 byte tag | Put frequent fields low |
| Use `bytes` over `string` | Avoids UTF-8 validation |
| Use arenas for many messages | Amortize allocation |
| Store serialized form for long-lived data | 5X smaller than parsed |

---

## 11. Language-Specific Tips

### Python
- Use `__slots__` to reduce memory
- Prefer list comprehensions (optimized by interpreter)
- Use `array.array` for numeric data
- Consider NumPy for numeric hot paths
- Use `collections.deque` for queue operations

### General
- Prefer move semantics over copy
- Use string views / spans for read-only access
- Inline small, hot functions
- Avoid virtual calls in tight loops

---

## Real-World Improvements

| Optimization | Improvement |
|--------------|-------------|
| Hash table vs binary tree | 4X |
| Lock sharding (64 shards) | 69% wall-clock reduction |
| 16-way cache sharding | 2X throughput |
| Remove protobuf from hot path | 20X |
| Sampling stats (1 in 32) | 62% |
| CRC loop unrolling | Variable |
| Deferred computation | 43s → 2s |

---

## Performance Analysis Checklist

When reviewing code for performance:

1. **Hot Paths Identified?** Know which code runs frequently
2. **Allocations Minimized?** Check for allocation in loops
3. **Locks Amortized?** Batch operations under single lock
4. **Data Locality?** Hot data together, cold data separate
5. **Appropriate Data Structures?** O(1) where possible
6. **Unnecessary Work Avoided?** Fast paths, caching, lazy eval
7. **Contention Reduced?** Sharding, lock-free where appropriate
8. **Logging/Stats Sampled?** Not on every operation

---

## References

- [Abseil Performance Hints](https://abseil.io/fast/hints.html)
- [Numbers Every Programmer Should Know](http://brenocon.com/dean_perf.html)
- [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
