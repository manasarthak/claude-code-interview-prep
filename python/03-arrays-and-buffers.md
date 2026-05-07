# Arrays, Buffers, Circular & Partially-Filled Arrays

**Phase:** 1 (Linear structures)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Complexity Analysis](./01-complexity-analysis.md) · [Python Internals](./02-python-internals.md) · [Stacks/Queues/Deques](./05-stacks-queues-deques.md) · [DE: Partitioning & Performance](../data-engineer/13-partitioning-performance.md)

---

## Why This File Exists

Arrays are the most fundamental data structure — every higher structure (lists, hashes, heaps, even trees on disk) is ultimately built on a contiguous block of memory. The variants below (partially-filled, circular, dynamic) capture *how* arrays are reused to build other things.

In Python, `list` is the default array, but a real DE/ML pipeline pushes through `numpy.ndarray`, `array.array`, `bytearray`, and Apache Arrow buffers. The handover between these — when you switch and why — is a real interview signal.

---

## BEGINNER — The Array Concept

### What an Array Is (and Isn't)

An array is a **contiguous block of memory** holding fixed-size elements. Index `i` is found at `base_address + i * element_size`. That's why:
- **Random access is O(1).** Pure pointer arithmetic.
- **Insertion in the middle is O(n).** Everything to the right shifts.
- **Cache locality is excellent.** Sequential reads stream from RAM into L1/L2 cache.

A Python `list` is *almost* an array — except it stores pointers (`PyObject*`), not raw values. That's the price you pay for `[1, "hello", [2,3]]` (heterogeneous types). For homogeneous numeric data, `numpy.ndarray` stores raw C values inline.

```
Python list:   [ ptr, ptr, ptr, ptr ] → each ptr → heap-allocated PyObject
NumPy array:   [  1 ,  2 ,  3 ,  4  ]   raw int32, contiguous, no boxing
```

This single difference makes a million-element NumPy array ~10× more memory-efficient and ~100× faster to traverse than a Python list. **It's why every serious DE/ML library is NumPy-or-Arrow under the hood.**

### Static vs. Dynamic Arrays

| | Static array | Dynamic array (Python `list`, C++ `vector`) |
|---|---|---|
| Size | Fixed at allocation | Grows on demand |
| Insert at end | O(1) (or fail if full) | O(1) amortized |
| Memory | Exactly n slots | Often 1.25× – 2× n |
| Use case | Embedded, predictable, max-fixed | General purpose |

### Big-O Cheat Sheet

| Operation | Python `list` | `numpy.ndarray` |
|---|---|---|
| Random access `arr[i]` | O(1) | O(1) |
| Append | O(1) amortized | O(n) (no in-place grow) |
| Insert at index k | O(n - k) | O(n) (creates new array) |
| Delete at index k | O(n - k) | O(n) |
| Search (unsorted) | O(n) | O(n), but vectorized in C |
| Slice `arr[a:b]` | O(b-a), copies pointers | O(1), creates a view (zero-copy!) |

The "slice = view" property of NumPy is huge: `arr[1000:2000]` doesn't copy, it returns a struct pointing to the same memory. This is why pandas/Polars/PyArrow can transform multi-GB columns without copying — they're slicing views.

---

## INTERMEDIATE — Partially-Filled Arrays, Dynamic Growth, Circular Buffers

### Partially-Filled Arrays — The Hidden Mechanism in Every `list`

A partially-filled array allocates more capacity than current size. You track:
- `size` — number of valid elements
- `capacity` — total allocated slots

```
size = 4, capacity = 8
[ A, B, C, D, _, _, _, _ ]
```

This lets you append in O(1) without reallocating every time. When `size == capacity`, you allocate a larger block and copy.

**CPython's exact growth formula** (from `listobject.c`):

```c
new_allocated = (size_t)newsize + (newsize >> 3) + 6
new_allocated &= ~(size_t)3;  // round up to multiple of 4
```

So roughly **12.5% growth + 6**, rounded to a multiple of 4. More conservative than 2× (Java/C++) — saves memory but causes more frequent resizes.

**Why geometric growth (any factor > 1)?** Linear growth (capacity += k) would give O(n²) total cost for n appends. Doubling gives O(n) total → O(1) amortized per append. (See [Complexity Analysis — Amortized](./01-complexity-analysis.md).)

#### Implementation Sketch

```python
class DynamicArray:
    def __init__(self):
        self._size = 0
        self._capacity = 1
        self._data = [None] * self._capacity

    def append(self, x):
        if self._size == self._capacity:
            self._resize(self._capacity * 2)
        self._data[self._size] = x
        self._size += 1

    def _resize(self, new_capacity):
        new_data = [None] * new_capacity
        for i in range(self._size):
            new_data[i] = self._data[i]
        self._data = new_data
        self._capacity = new_capacity

    def __getitem__(self, i):
        if i < 0 or i >= self._size:
            raise IndexError
        return self._data[i]

    def __len__(self):
        return self._size
```

You won't be asked to write this every interview, but you should be able to.

### Circular Arrays (Ring Buffers)

A circular array uses a fixed-size array with two pointers — `head` (read) and `tail` (write) — that wrap around modulo capacity:

```
capacity = 8
       head            tail
        ↓               ↓
[ _, _, A, B, C, D, _, _ ]    head=2, tail=6
        size=4
```

When `tail` reaches the end, it wraps to 0. When `head` catches up to `tail`, the buffer is empty; when `tail + 1 == head`, it's full.

#### Big-O

| Operation | Complexity |
|---|---|
| Push at tail | O(1) |
| Pop at head | O(1) |
| Random access from head | O(1) (just `(head + i) % cap`) |
| Resize (when full) | O(n), if you allow growth |

#### Where Ring Buffers Show Up in DE/ML

- **Kafka log segments** — each partition is a circular log file; old data is reclaimed by deleting the head segment.
- **Kinesis shards** — bounded retention behaves like a giant ring buffer.
- **Audio/video streaming** — fixed-latency buffer.
- **Sliding window streaming aggregations** — last N events.
- **Profiler ring buffer** — last N stack samples (Linux `perf`, py-spy).
- **Linux dmesg, journald** — ring-buffered kernel logs.
- **WAL / write-ahead logs** — Postgres, RocksDB.

This is the most underappreciated structure in [streaming systems](../data-engineer/06-batch-vs-streaming.md). When an interviewer asks "design a sliding window over a stream," ring buffer is the answer if the window size is fixed.

#### Implementation Sketch

```python
class RingBuffer:
    def __init__(self, capacity):
        self._buf = [None] * capacity
        self._cap = capacity
        self._head = 0
        self._size = 0

    def push(self, x):
        if self._size == self._cap:
            raise OverflowError("buffer full")
        tail = (self._head + self._size) % self._cap
        self._buf[tail] = x
        self._size += 1

    def pop(self):
        if self._size == 0:
            raise IndexError("buffer empty")
        x = self._buf[self._head]
        self._head = (self._head + 1) % self._cap
        self._size -= 1
        return x

    def __len__(self):
        return self._size
```

### `array.array` — When You Want Typed Numeric Arrays Without NumPy

Python's standard library has `array.array(typecode, ...)` for unboxed numeric arrays. It's lighter than NumPy (no dependency) but lacks vectorized ops. Used in embedded code, networking, and stdlib-only environments.

```python
import array
a = array.array('i', [1, 2, 3, 4])  # int32 array
a.append(5)
```

For DE/ML, almost always reach for `numpy` instead. `array.array` is mostly a relic for "pure stdlib" purists.

### `bytearray` and `bytes` — Mutable / Immutable Byte Arrays

- `bytes` — immutable, hashable. Network payloads, file reads.
- `bytearray` — mutable. In-place edits, encoders, parsers.

Used heavily in:
- Reading binary files (`open(path, "rb").read()` returns bytes)
- Network protocols (struct.pack/unpack)
- Custom serializers (Avro/Protobuf in C extensions)

`bytearray` supports the buffer protocol, so you can `memoryview` it for zero-copy slicing.

---

## ADVANCED — NumPy, Arrow, and Columnar Memory in DE/ML

### NumPy Internals (the array under every ML library)

A `numpy.ndarray` has:

```
shape:    (rows, cols, ...)        — Python tuple
strides:  (bytes per row, bytes per col, ...)
dtype:    int32, float64, bool, etc.
data:     pointer to raw C array (contiguous, unboxed)
flags:    C_CONTIGUOUS, F_CONTIGUOUS, OWNDATA, ...
```

**Strides** let one underlying buffer support many "views." `arr.T` (transpose) doesn't copy — it just swaps strides. `arr[::2]` doesn't copy — it creates a view with stride 2. This is why analytic code using NumPy is so memory-efficient: most operations are zero-copy until forced.

#### Vectorization

Element-wise ops on NumPy arrays run in compiled C, releasing the GIL:

```python
arr = np.arange(10_000_000)
arr * 2   # ~5ms — runs in C SIMD
[x * 2 for x in arr]  # ~1500ms — pure Python loop with boxing
```

The 300× difference is *the* reason serious numerics live in NumPy. **In Spark, the equivalent is "use built-in functions, not Python UDFs."** Pandas UDFs (Arrow-batched) recover most of the perf by passing whole columns to Python.

### Apache Arrow — The Columnar Memory Format That Powers DE in 2026

Arrow is an **in-memory columnar format** designed for zero-copy interop between languages. A column is a contiguous buffer of values + a null bitmap:

```
Column "age":  values:    [25, 30, 17, 99, ...]
               nulls:     [ 1,  1,  0,  1, ...]   (bitmap: 1 = present)
```

**Why this matters in DE:**
- Arrow is the on-the-wire format for [pandas ↔ Spark](../data-engineer/07-spark-fundamentals.md) (Pandas UDFs send Arrow batches).
- Arrow is the in-memory format of [Polars](https://pola.rs/), [DuckDB](https://duckdb.org/), [Dremio](https://www.dremio.com/), [Velox](https://velox-lib.io/), Snowflake's vectorized engine.
- **Parquet** (the on-disk format) was designed to round-trip cheaply with Arrow. See [Warehouses & Lakehouses](../data-engineer/03-warehouses-lakes-lakehouse.md).
- ADBC, Flight, Flight SQL — all are Arrow-native database protocols replacing ODBC/JDBC for column-heavy workloads.

**Arrow's memory layout maps directly onto SIMD instructions** — each batch is contiguous, predictable, and operations like "sum a column" become a tight CPU loop. This is the foundation of every "vectorized engine" you've heard of since 2020.

### Memory-Mapped Files — Treating Disk as an Array

`mmap` lets you treat a file as a `memoryview`-able byte array. The OS handles paging — you read at "memory speeds" but only for hot pages.

```python
import mmap
with open("big.bin", "rb") as f:
    mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    chunk = mm[1024:2048]  # zero-copy slice
```

Used by:
- LMDB, RocksDB (indirectly), DuckDB
- Loading model weights (HuggingFace `safetensors` is mmap-friendly).
- Reading Parquet/Arrow files in PyArrow.
- **Fast LLM weight loading** — `llama.cpp` mmaps the GGUF model file so the OS only loads pages on demand, allowing multiple processes to share weights.

### SIMD, Vectorized Ops, and Why Loops Are Slow

Modern CPUs have SIMD instructions (SSE, AVX-2, AVX-512, NEON) that operate on 4–16 numbers per instruction. NumPy/Arrow/Polars exploit this; Python `for` loops do not.

**Rule of thumb:** if your inner loop is in Python, you're paying ~100× overhead vs. compiled vectorized code. If you can't vectorize, push the loop into C/Rust via:
- `numpy` operations
- `numba` JIT
- `cython` compiled
- `pyo3` / `maturin` (Rust)
- `polars` / `duckdb` (already vectorized)

### GPU Arrays (CuPy, RAPIDS, JAX, PyTorch tensors)

GPU arrays are arrays in GPU memory. Same Big-O, vastly different constants — and a transfer cost across PCIe.

- **CuPy / cuDF (RAPIDS)** — NumPy/pandas API but on GPU.
- **PyTorch / JAX tensors** — same idea, ML-tuned.
- **DLPack** — interchange format so a tensor can move between PyTorch, JAX, CuPy without copying.

For DE: GPU databases (Brytlyt, BlazingSQL, HEAVY.AI) operate on GPU arrays. Niche but trending up. For ML/LLM: training and inference live almost entirely on GPU tensors. See [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md).

---

## Where This Bites You in Real Code

| Issue | Cause | Fix |
|---|---|---|
| `list.insert(0, x)` in a loop is O(n²) | Each insert shifts all elements | `collections.deque.appendleft` |
| `numpy.append(arr, x)` is O(n) per call | NumPy can't grow in place | Pre-allocate, or build a Python list and convert at the end |
| Sliding window over stream OOMs | Naïve list of all events | Ring buffer with fixed capacity |
| Pandas `df.iterrows()` glacial | Pure Python loop, boxing each row | Vectorized ops or `df.apply` (still slow) → ideally `numpy`/`polars` |
| Arrow → pandas conversion creates copies | Older PyArrow defaulted to copy | `df.to_pandas(zero_copy_only=True)` |
| 8GB Parquet file blows up RAM | Reading all columns | Project (`columns=[...]`) and use predicate pushdown |
| `bytearray.append(b)` slow in tight loop | Per-call overhead | Build into pre-sized buffer with `struct.pack_into` |

---

## Common Interview Gotchas

**Q: Why is `list.append` O(1) amortized but `numpy.append` O(n)?**
A: Python `list` is a partially-filled dynamic array — geometric growth gives amortized O(1). NumPy arrays are fixed-size; "append" creates a new array and copies. **Don't grow NumPy arrays incrementally — pre-allocate.**

**Q: What's the time complexity of `[i*2 for i in range(n)]`?**
A: O(n). It also pre-sizes the result list since `range` has a known length, so no reallocation.

**Q: What's the difference between `arr[1:5]` for a Python list vs. NumPy array?**
A: List: O(k) copy of pointers. NumPy: O(1), creates a view sharing memory.

**Q: How is a partially-filled array different from a circular array?**
A: Partially-filled grows dynamically; circular has fixed capacity and wraps. They solve different problems.

**Q: Why is `''.join(parts)` O(n) but `+=` in a loop O(n²)?**
A: `join` pre-computes total size and allocates once. `+=` reallocates and copies on every iteration.

**Q: Why does `numpy.zeros((10000, 10000))` use ~800MB?**
A: 10⁸ float64 cells × 8 bytes = 800MB. NumPy stores raw values inline.

**Q: What is "C-contiguous" vs "F-contiguous"?**
A: Row-major (C) vs column-major (Fortran) memory layout. Affects which axis is fast to iterate. Matters when interfacing with BLAS/LAPACK.

**Q: When would you use `array.array` over `numpy.ndarray`?**
A: Only when you can't depend on NumPy (stdlib-only environments). Otherwise NumPy wins on every dimension.

---

## Interview-Ready Cheat Sheet

### Quick mappings to internalize

| Real-world need | Right tool |
|---|---|
| General-purpose Python list | `list` |
| Numeric vector, fast math | `numpy.ndarray` |
| Tabular data | `pandas`, `polars`, `pyarrow.Table` |
| Fixed-capacity FIFO, both ends | Ring buffer / `collections.deque(maxlen=N)` |
| Streaming sliding window | `deque(maxlen=N)` |
| Zero-copy bytes manipulation | `bytearray` + `memoryview` |
| Memory-mapped large file | `mmap.mmap` |
| GPU array | PyTorch / CuPy / JAX |
| Cross-language column data | Apache Arrow |

### Quick Trade-off Pairs

- **`list` vs `numpy`:** flexibility vs. performance + memory.
- **`list.append(x)` vs `numpy.append`:** O(1) amortized vs O(n) — never grow NumPy in a loop.
- **Slicing `list[a:b]` vs `arr[a:b]`:** copy vs view.
- **Arrow vs Pandas:** Arrow is canonical wire format; Pandas adds DataFrame ergonomics on top (sometimes copies).
- **Circular vs dynamic:** fixed memory vs unbounded; predictable latency vs flexibility.

---

## Resources & Links

- [CPython listobject.c](https://github.com/python/cpython/blob/main/Objects/listobject.c)
- [NumPy Internals docs](https://numpy.org/doc/stable/dev/internals.html)
- [Apache Arrow Format](https://arrow.apache.org/docs/format/Columnar.html) — short, dense, essential.
- [Arrow Flight](https://arrow.apache.org/docs/format/Flight.html) — wire protocol for big columnar data.
- [Polars: Why DataFrames Need a New Engine](https://pola.rs/posts/polars_birds_eye_view/)
- [DuckDB internals blog](https://duckdb.org/docs/internals/overview)
- *Designing Data-Intensive Applications* ch. 3 — Storage and Retrieval covers row-vs-column memory.

### Companion Files
- [Python Internals](./02-python-internals.md) — CPython list/tuple internals.
- [Stacks/Queues/Deques](./05-stacks-queues-deques.md) — built on arrays and ring buffers.
- [DE: Partitioning & Performance](../data-engineer/13-partitioning-performance.md) — columnar memory in big-data systems.
- [DE: Warehouses & Lakehouses](../data-engineer/03-warehouses-lakes-lakehouse.md) — Parquet (columnar on-disk) tie-in.

---

*Next: [Linked Lists](./04-linked-lists.md) — the structure you almost never use in Python, but always need to discuss in interviews.*
