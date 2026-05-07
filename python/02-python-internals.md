# Python Internals — How CPython Actually Implements Things

**Phase:** 0 (Foundation)
**Difficulty progression:** Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Complexity Analysis](./01-complexity-analysis.md) · [Hash Tables](./06-hash-tables.md) · [Arrays & Buffers](./03-arrays-and-buffers.md)

---

## Why This File Exists

Most candidates can name the complexity of a `dict` operation. Far fewer can explain *how* a `dict` lookup actually works inside CPython — the open-addressing strategy, the perturbation, the hash randomization, the load factor, the resize policy. That gap is the difference between memorizing and *understanding*, and interviewers can smell it.

This file isn't an exhaustive C tour; it's the parts that come up in interviews and that change how you write Python.

> Reference: [CPython source](https://github.com/python/cpython) — particularly `Objects/listobject.c`, `Objects/dictobject.c`, `Objects/setobject.c`, `Objects/tupleobject.c`. Read each at least once.

---

## BEGINNER — The Mental Model You Need First

### Everything Is an Object

In CPython, every value — even `42` and `True` — is heap-allocated and accessed through a `PyObject*`. Every object has at minimum:

```c
typedef struct {
    Py_ssize_t ob_refcnt;     // reference count
    PyTypeObject *ob_type;    // pointer to type
} PyObject;
```

This is why **a million Python integers consume ~28MB of RAM, while a million NumPy int32s consume 4MB.** Boxing every value is expensive. It's also why `int` is "free" of overflow but slower than C ints, and why **NumPy/PyArrow are critical for DE workloads** — they store contiguous unboxed C arrays. See [Arrays & Buffers](./03-arrays-and-buffers.md) for how Arrow exploits this for analytics.

### Reference Counting + Cycle Collector

CPython manages memory primarily by reference counting. Each assignment increments a counter; each `del` or scope exit decrements it. When count hits zero, the object is freed immediately (deterministic destruction).

Reference cycles (a list containing itself, a graph node referencing its parent) defeat refcounting, so CPython runs a periodic **generational garbage collector** (`gc` module) to detect and break cycles.

**Implication for interviews:** if you say *"Python uses garbage collection like Java"* you're half-right at best. Refcounting first, GC for cycles only.

### The GIL

The Global Interpreter Lock allows only one thread to execute Python bytecode at a time. Threads still help for **I/O-bound** work (the GIL is released during I/O syscalls and during NumPy's C-extension calls), but not for **CPU-bound** pure-Python work. For CPU parallelism, use:

- `multiprocessing` — separate processes, separate GILs.
- C extensions that release the GIL (`numpy`, `pandas`, `scipy`, custom Cython).
- `concurrent.futures.ProcessPoolExecutor`.
- Python 3.13+ free-threaded build (PEP 703) — opt-in, GIL removed, slowly stabilizing.

In DE: **PySpark sidesteps the GIL** by sending data to JVM executors. **pandas releases the GIL** for many vectorized ops. Pure-Python loops do not — and that's why `df.apply(lambda x: ...)` is a known anti-pattern compared to vectorized ops.

---

## INTERMEDIATE — Built-in Types and What's Inside Them

### `list` — A Resizable C Array of `PyObject*`

CPython's `list` is **not a linked list.** It's a contiguous C array of pointers (`PyObject **ob_item`) plus a length and an allocated capacity.

```
list:
  ob_size:      4
  allocated:    8
  ob_item ──▶  [ ptr, ptr, ptr, ptr, _, _, _, _ ]
                  │     │     │     │
                  ▼     ▼     ▼     ▼
                "a"   "b"   "c"   "d"   ← actual PyObjects on the heap
```

| Operation | Complexity | Why |
|---|---|---|
| `lst[i]` | O(1) | Pointer arithmetic on the contiguous array |
| `lst.append(x)` | O(1) amortized | Geometric growth ~12.5%; occasional realloc |
| `lst.insert(0, x)` | O(n) | All existing pointers shift right |
| `lst.pop()` (end) | O(1) | Decrement size |
| `lst.pop(0)` | O(n) | All pointers shift left — **DON'T USE FOR QUEUES** |
| `lst[a:b]` | O(b-a) | Copies pointers into a new list |
| `x in lst` | O(n) | Linear scan |
| `lst.sort()` | O(n log n) | Timsort (see [Sorting](./10-sorting.md)) |

**Growth strategy:** when `ob_size` reaches `allocated`, CPython resizes to `new = (size + (size >> 3) + 6) & ~3` — roughly 12.5% growth. This is more conservative than 2× (which Java's ArrayList uses) — saves memory at the cost of more frequent resizes.

**Why amortized O(1) append:** the geometric series of resize costs sums to O(n) for n appends → O(1) per append on average. (See [Complexity Analysis — Amortized](./01-complexity-analysis.md).)

### `tuple` — Like `list` But Immutable and Cheaper

A `tuple` is a fixed-size array of `PyObject*`. No `allocated` slack, no resize. Slightly faster to create, slightly smaller, and **hashable** if all its elements are.

CPython caches small tuples (`()` and tuples up to length 20 of recent shapes) in a free list, so `(1, 2)` literally reuses memory across calls. This is why immutable data is faster to ship around in Python.

**DE example:** dictionaries keyed on `(date, country)` rely on tuple hashing. NamedTuples and dataclasses with `frozen=True` are the standard "lightweight row" pattern.

### `dict` — Open-Addressed Hash Table with Compact Layout

This is the most-asked-about internal in interviews. CPython's `dict` (since 3.6) uses a **compact layout** with two arrays:

```
indices:   [ -1, 2, -1, 0, -1, 1, -1, -1 ]   ← sparse, mostly empty
entries:   [ (h0, k0, v0), (h1, k1, v1), (h2, k2, v2) ]   ← dense, insertion order
```

Lookup walks the `indices` array using the hash; the index found points into `entries` to fetch the actual `(hash, key, value)` triple. This layout:

- Preserves **insertion order** (since 3.7 it's a language guarantee).
- Saves memory (the sparse array holds tiny indices, not full triples).
- Makes iteration fast (just walk the dense `entries` array).

#### Open Addressing with Perturbation

When two keys hash to the same slot, CPython doesn't chain — it probes another slot using a *perturbation* sequence:

```c
i = hash & mask
while (slot is taken and key doesn't match):
    perturb >>= 5
    i = (5*i + perturb + 1) & mask
```

The `perturb` term mixes in the high bits of the hash so probing isn't just linear. This avoids the clustering problem of pure linear probing while staying cache-friendly.

#### Hash Randomization (Security)

Since Python 3.3, `hash(str)` and `hash(bytes)` are seeded with a per-process random value (`PYTHONHASHSEED`). Without this, attackers could craft inputs that all hash to the same bucket, degrading every dict to O(n) — a real DoS vector for web servers parsing user JSON. **Run a Python script twice and `hash("foo")` returns different values across processes; this is intentional.**

#### Resize Policy

CPython resizes when the table is 2/3 full. New table is the next power of 2 large enough to be at most 2/3 full. Resize cost is O(n); amortized over inserts it's O(1).

| Operation | Complexity | Worst case |
|---|---|---|
| `d[k]` | O(1) | O(n) (pathological collisions) |
| `d[k] = v` | O(1) amortized | O(n) on resize |
| `del d[k]` | O(1) | O(n) |
| `k in d` | O(1) | O(n) |
| Iteration | O(n) | O(n) |

**Key invariants you can repeat in interviews:**
- O(1) **assumes** a uniform hash. If `__hash__` returns a constant, `dict` becomes O(n).
- `__hash__` and `__eq__` must be consistent: `a == b` → `hash(a) == hash(b)`. Violating this corrupts the dict.
- Mutable types are unhashable. `dict[list]` is a TypeError because mutating the list would invalidate its hash.

### `set` and `frozenset` — Same Structure, No Values

`set` is essentially a `dict` without the value column. Same open-addressing, same perturbation. Same O(1) average for `add`, `remove`, `in`.

`frozenset` is the immutable, hashable variant — useful as a dict key when you need a "set of things" as an identifier.

**Key DE/AI applications:**
- Dedup with `seen = set()` and `seen.add(x)` — the canonical streaming dedup at small scale. Switch to a [Bloom filter](./15-probabilistic-structures.md) when memory matters.
- Set operations (`|`, `&`, `-`, `^`) are O(min(len(a), len(b))) and are how SQL `EXCEPT`, `INTERSECT`, etc. are conceptually implemented in single-machine engines.

### `str` — Immutable, with Multiple Internal Representations (PEP 393)

Since Python 3.3, strings store their characters in the narrowest representation that fits:
- Pure ASCII → 1 byte/char (Latin-1 internally).
- BMP → 2 bytes/char.
- Full Unicode → 4 bytes/char.

This makes `len()` O(1) (it's stored), indexing O(1), and most operations memory-efficient.

**The interview trap:** because strings are *immutable*, `s += x` in a loop is O(n²) — every concat copies the full string. Use `''.join(parts)` for O(n) total.

```python
# BAD — O(n²)
s = ""
for word in words:
    s += word

# GOOD — O(n)
s = "".join(words)
```

CPython has a small optimization that *sometimes* mutates strings in-place if their refcount is exactly 1, but you can't rely on it.

### `collections.deque` — Doubly Linked List of Fixed-Size Blocks

When you need O(1) `appendleft` / `popleft`, `list` won't do it (those are O(n)). `collections.deque` is implemented as a **doubly linked list of arrays** (each block ~64 elements), giving:

| Operation | Complexity |
|---|---|
| `append`, `appendleft` | O(1) |
| `pop`, `popleft` | O(1) |
| `d[i]` (random access) | O(n) — has to walk blocks |
| `len(d)` | O(1) |

**This is the right Python queue.** Use `deque` for BFS, sliding-window problems, and any FIFO/LIFO with both-end mutation.

### `heapq` — Binary Heap on Top of a `list`

`heapq` doesn't expose a heap class — it's a set of functions that operate on a regular `list`. The list is interpreted as a binary heap (`list[i]`'s children are at `2i+1` and `2i+2`).

| Operation | Complexity |
|---|---|
| `heapq.heappush(h, x)` | O(log n) |
| `heapq.heappop(h)` | O(log n) |
| `heapq.heapify(lst)` | O(n) — Floyd's algorithm |
| `h[0]` (min) | O(1) |
| `heapq.nlargest(k, lst)` / `nsmallest` | O(n log k) |

CPython's heapq is **min-heap only.** For a max-heap, push negated values. For priority queues with task-priority + insertion-order tie-break, push tuples `(priority, counter, task)`.

### `collections.OrderedDict`, `defaultdict`, `Counter`

- **`OrderedDict`** — was the way to get ordered dicts pre-3.7. Now mostly redundant *except* for `move_to_end` (O(1)) which the regular `dict` doesn't have. Used to implement an LRU cache.
- **`defaultdict`** — `defaultdict(list)`, `defaultdict(int)` — saves you `if key not in d: d[key] = []`.
- **`Counter`** — a `dict` subclass for counting. `Counter.most_common(k)` is O(n + k log n) using a heap internally.

### `functools.lru_cache` — Memoization with an Ordered Dict

`lru_cache` caches function results in an `OrderedDict`. On each call, it looks up `(args, kwargs)` keys; if hit, return cached; if miss, compute, store, evict oldest if at capacity.

This is one of the highest-leverage decorators in Python — a single line can turn an exponential recursion into linear (memoized DP). See [Algorithm Design](./11-algorithm-design.md).

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n < 2: return n
    return fib(n-1) + fib(n-2)
```

---

## ADVANCED — The Less-Obvious Internals That Show Up in System Design

### Slots — Why `__slots__` Saves Memory

Every object normally has a `__dict__` for attributes. `__slots__` replaces it with a fixed C-array of pointers — saving ~50 bytes per instance.

```python
class Point:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x, self.y = x, y
```

For 10M Points, this saves ~500MB. **In DE/ML pipelines that materialize many small objects (parsed log lines, AST nodes), `__slots__` is the difference between a 4GB and 8GB process.**

But you sacrifice flexibility — no dynamic attributes. dataclasses since 3.10 support `@dataclass(slots=True)` to combine the two.

### Interning — Why `'foo' is 'foo'` Sometimes

CPython interns small ints (-5 to 256) and short identifier-like strings. This means:

```python
a = 257
b = 257
a is b  # False (might be True in some implementations)

a = "hello_world"
b = "hello_world"
a is b  # Usually True — short strings get interned
```

**Don't ever use `is` for value comparison** — it's identity, not equality. The behavior depends on internals you can't control.

### The Buffer Protocol (`memoryview`)

CPython has a low-level interface for sharing raw memory between C extensions without copying. NumPy, PyArrow, struct, mmap, and bytes all expose this. **This is why pandas can pass a column to NumPy without serializing — they share the underlying buffer.**

```python
data = bytearray(b"hello")
view = memoryview(data)
view[0] = ord('H')  # mutates underlying bytearray
```

For DE, this is critical for zero-copy interop between Arrow, NumPy, and PySpark — the difference between O(n) and O(0) memory traffic.

### `__hash__` and `__eq__` Contract (the bug that ruins your dict)

```python
class Bad:
    def __init__(self, x): self.x = x
    def __eq__(self, other): return self.x == other.x
    # forgot __hash__!

s = {Bad(1)}
Bad(1) in s  # False (or undefined behavior)
```

Defining `__eq__` without `__hash__` makes the class unhashable. Defining one inconsistently with the other corrupts every dict and set you put it in. The two must satisfy: `a == b` ⇒ `hash(a) == hash(b)`.

For dataclasses, set `frozen=True, eq=True` to get a correct hashable record automatically.

### Generators, Iterators, and Lazy Evaluation

Generators (`yield`) are coroutines that resume from where they paused. They're O(1) memory regardless of stream length, which is why streaming DE pipelines often use them:

```python
def parse_lines(file):
    for line in file:
        yield json.loads(line)

# Process 100GB file in constant memory
for record in parse_lines(open("logs.ndjson")):
    process(record)
```

`itertools` (groupby, chain, islice, tee, accumulate) gives you O(1)-memory composable transforms. **This is the "Spark on a single machine" toolkit.**

### Asyncio — Cooperative Concurrency

`asyncio` runs many I/O-bound coroutines on a single thread by yielding at every `await`. It's not faster per-operation — it's O(1) thread overhead even with 10K connections. Critical for:
- API gateways
- Web crawlers
- Database connection pools
- Streaming consumers (aiokafka, aiobotocore)

`asyncio` doesn't help CPU-bound code (still bound by the GIL).

### CPython 3.11+ Speedups (Faster CPython initiative)

Worth name-dropping if asked about modern Python:
- **Specialization** (PEP 659) — the interpreter rewrites bytecodes based on observed types, dramatically speeding up hot loops.
- **Inline caches** — skip method-resolution lookups when the type is stable.
- **Zero-cost exceptions** — `try` blocks now have ~zero overhead in the no-exception path.
- **Per-interpreter GIL** (PEP 684, 3.12) — allows independent subinterpreters with separate GILs.
- **Free-threaded mode** (PEP 703, 3.13 experimental) — GIL-free Python.

These don't change Big-O but cut constants by 25–50%, which matters in high-throughput pipelines.

---

## Where This Bites You in DE/AI Code

| Symptom | Internals reason | Fix |
|---|---|---|
| `pd.DataFrame.apply(lambda)` is glacial | Pure-Python loop boxing/unboxing every value | Use vectorized ops or `numba`/`numpy` |
| Pipeline OOMs on a "small" dataset | Each Python object has 28+ bytes of overhead | Use NumPy/Arrow/pandas, or `__slots__` for dataclasses |
| Multithreaded job uses 100% of one core | GIL serializes Python bytecode | Use `multiprocessing` or release GIL in C ext |
| Same script gives different dict iteration order | Pre-3.7 Python or hash randomization | Don't rely on dict ordering before 3.7; use `collections.OrderedDict` |
| Set membership inconsistent | Custom class with `__eq__` but no `__hash__` | Implement both consistently |
| String concat in loop slow | Strings immutable; each concat O(n) | `"".join(parts)` |
| `lst.pop(0)` in BFS is slow | List leftpop is O(n) | `collections.deque.popleft()` |
| 50GB JSON file crashes the process | `json.load` reads everything | Stream with `ijson` or use `polars`/`pyarrow.json` |
| Spark UDF 100× slower than SQL | Python<->JVM serialization per row | Use built-in functions or pandas UDFs (Arrow batch) |

---

## Common Interview Gotchas

**Q: How does CPython store a `list`?**
A: As a contiguous C array of `PyObject*` pointers, not a linked list. Indexing is O(1).

**Q: How is `dict` lookup O(1)?**
A: Hash the key, mod by table size to get a slot, probe with perturbation if collision. Average O(1) assuming uniform hash; worst case O(n).

**Q: What changed about dicts in 3.6 / 3.7?**
A: 3.6 introduced compact layout (insertion-ordered as implementation detail). 3.7 made insertion order a language guarantee.

**Q: Why does `('a',) + ('b',)` work but `[1,2,3].extend(4)` fails?**
A: Tuples are immutable; `+` creates a new tuple. `extend` requires an iterable; an int isn't.

**Q: What's the cost of `a == b` for two large lists?**
A: O(n) — element-wise comparison. Same for sets and dicts.

**Q: How does `is` differ from `==`?**
A: `is` compares object identity (same address); `==` compares value via `__eq__`. Use `is` only with `None`, `True`, `False`.

**Q: Why is mutable default argument a footgun?**
A: `def f(x=[])` — the list is created once at function definition time and shared across calls. Default arguments are evaluated once.

**Q: How does Python implement closures?**
A: Each closure carries a tuple of `cell` objects pointing to the enclosing scope's variables. Read `f.__closure__`.

---

## Interview-Ready Cheat Sheet

### One-liners to remember

- **`list` is a dynamic C array, not a linked list. `pop(0)` is O(n).**
- **`dict` and `set` use open addressing with perturbation. O(1) average, O(n) worst.**
- **Strings are immutable. Concat in a loop is O(n²); use `join`.**
- **`collections.deque` is the right queue. Block-based doubly linked list, O(1) at both ends.**
- **`heapq` is a function set on a regular list. Min-heap only.**
- **The GIL means threading is for I/O. CPU-bound work needs `multiprocessing` or C extensions.**
- **Generators give O(1) memory streaming. Use them for big data on a single machine.**

### When to reach for which structure

| Goal | Structure |
|---|---|
| Constant-time membership test | `set` |
| Constant-time key→value | `dict` |
| Both ends O(1) push/pop | `collections.deque` |
| Smallest item O(1) | `heapq` |
| Sorted insertion + range query | `sortedcontainers.SortedList` (3rd party) |
| Streaming aggregation | generators + `itertools` |
| Numeric arrays | `numpy.ndarray` (release the GIL) |
| Tabular data | `pandas` / `polars` / `pyarrow` |
| Tiny immutable record | `tuple` or `dataclass(frozen=True, slots=True)` |

---

## Resources & Links

### CPython Source Tour (read each at least once)
- [`Objects/listobject.c`](https://github.com/python/cpython/blob/main/Objects/listobject.c)
- [`Objects/dictobject.c`](https://github.com/python/cpython/blob/main/Objects/dictobject.c)
- [`Objects/setobject.c`](https://github.com/python/cpython/blob/main/Objects/setobject.c)
- [`Objects/unicodeobject.c`](https://github.com/python/cpython/blob/main/Objects/unicodeobject.c)
- [`Modules/_collectionsmodule.c`](https://github.com/python/cpython/blob/main/Modules/_collectionsmodule.c) — deque
- [`Modules/_heapqmodule.c`](https://github.com/python/cpython/blob/main/Modules/_heapqmodule.c)

### Articles & Talks
- [Raymond Hettinger — Modern Python Dictionaries (PyCon talk)](https://www.youtube.com/watch?v=p33CVV29OG8)
- [Brandon Rhodes — Dictionary Even Mightier](https://www.youtube.com/watch?v=66P5FMkWoVU)
- [Łukasz Langa — How to write deterministic Python](https://www.youtube.com/results?search_query=lukasz+langa+deterministic)
- [The CPython Internals book](https://realpython.com/products/cpython-internals-book/) — Anthony Shaw.
- [PEP 8201 — A new C API for extensions](https://peps.python.org/pep-0820/)
- [PEP 659 — Specializing Adaptive Interpreter](https://peps.python.org/pep-0659/) — 3.11+ speedups.
- [PEP 703 — No-GIL Python](https://peps.python.org/pep-0703/)

### Companion Files
- [Complexity Analysis](./01-complexity-analysis.md) — the language for talking about all this.
- [Hash Tables](./06-hash-tables.md) — dict/set deep dive.
- [Arrays & Buffers](./03-arrays-and-buffers.md) — list internals + NumPy/Arrow.

---

*Next: [Arrays & Buffers](./03-arrays-and-buffers.md) — now we apply the internals knowledge to the most common structure.*
