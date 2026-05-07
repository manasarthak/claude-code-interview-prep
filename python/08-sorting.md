# Sorting — From Selection Sort to Timsort to External Merge Sort

**Phase:** 3 (Algorithms)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Complexity Analysis](./01-complexity-analysis.md) · [Trees & Heaps](./07-trees-and-heaps.md) · [DE: Spark](../data-engineer/07-spark-fundamentals.md)

---

## Why This File Exists

Sorting is the canonical playground for complexity analysis: every algorithm has a clear best/average/worst case, and you can demonstrate stability, in-place, comparison-based, and non-comparison-based properties. Python's built-in `sorted` / `list.sort` is **Timsort** — a beautifully engineered hybrid that almost no candidate can describe. In real DE work, you mostly *don't* implement sorting; you understand why your data warehouse / Spark job is doing **external merge sort** under the hood.

---

## BEGINNER — The Classical Six

### The Big-O Table to Memorize

| Algorithm | Best | Average | Worst | Space | Stable | In-place | Comparison? |
|---|---|---|---|---|---|---|---|
| **Selection** | Θ(n²) | Θ(n²) | Θ(n²) | O(1) | No | Yes | Yes |
| **Insertion** | Θ(n) | Θ(n²) | Θ(n²) | O(1) | Yes | Yes | Yes |
| **Bubble** | Θ(n) | Θ(n²) | Θ(n²) | O(1) | Yes | Yes | Yes |
| **Merge sort** | Θ(n log n) | Θ(n log n) | Θ(n log n) | O(n) | Yes | No | Yes |
| **Quick sort** | Θ(n log n) | Θ(n log n) | Θ(n²) | O(log n) | No | Yes | Yes |
| **Heap sort** | Θ(n log n) | Θ(n log n) | Θ(n log n) | O(1) | No | Yes | Yes |
| **Bucket sort** | Θ(n + k) | Θ(n + k) | Θ(n²) | O(n + k) | Yes | No | No (needs known range) |
| **Radix sort** | Θ(d·(n + b)) | Θ(d·(n + b)) | Θ(d·(n + b)) | O(n + b) | Yes | No | No |
| **Counting sort** | Θ(n + k) | Θ(n + k) | Θ(n + k) | O(n + k) | Yes | No | No |
| **Timsort** (Python) | Θ(n) | Θ(n log n) | Θ(n log n) | O(n) | Yes | No | Yes |

Memorize *worst case*, *space*, *stability* for every row. These three columns capture 95% of interview questions.

### Selection Sort

Find the smallest, swap to position 0. Find the next smallest, swap to position 1. Repeat.

```python
def selection_sort(arr):
    for i in range(len(arr)):
        min_idx = i
        for j in range(i + 1, len(arr)):
            if arr[j] < arr[min_idx]:
                min_idx = j
        arr[i], arr[min_idx] = arr[min_idx], arr[i]
```

Always Θ(n²). Useful only as a teaching device. **No real system uses it.**

### Insertion Sort

Walk left to right, sliding each new element into its place in the sorted prefix.

```python
def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]; j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]; j -= 1
        arr[j + 1] = key
```

**Best case Θ(n)** when input is already sorted — only n-1 comparisons. This is why Timsort uses insertion sort for small runs and nearly-sorted data: in practice, real-world data is often partially sorted.

### Merge Sort

Divide-and-conquer: split in half, recursively sort, merge.

```python
def merge_sort(arr):
    if len(arr) <= 1: return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return _merge(left, right)

def _merge(a, b):
    out, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]: out.append(a[i]); i += 1
        else:            out.append(b[j]); j += 1
    out.extend(a[i:]); out.extend(b[j:])
    return out
```

- Always O(n log n). Stable. Not in place (O(n) extra memory).
- Recurrence: T(n) = 2·T(n/2) + Θ(n) → Master Theorem case 2 → Θ(n log n).
- Needs O(n) extra memory for the merge buffer — this is its only real downside vs. quicksort.

### Quick Sort

Pick a pivot. Partition into "less than pivot" and "greater than pivot." Recursively sort each.

```python
def quick_sort(arr, lo=0, hi=None):
    if hi is None: hi = len(arr) - 1
    if lo >= hi: return
    p = _partition(arr, lo, hi)
    quick_sort(arr, lo, p - 1)
    quick_sort(arr, p + 1, hi)

def _partition(arr, lo, hi):
    pivot = arr[hi]; i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    return i + 1
```

- Average Θ(n log n), worst Θ(n²) (already-sorted with a bad pivot). In-place, not stable.
- **Pivot choice is everything.** Random pivot, median-of-three, or median-of-medians (deterministic O(n log n) but slow constants) all reduce worst-case probability.
- C `qsort`, Java `Arrays.sort` for primitives — quicksort variants (often introsort).

### Heap Sort

Build a heap, repeatedly extract max. See [Trees & Heaps](./07-trees-and-heaps.md).

- Always Θ(n log n). In place. Not stable.
- Slower than quicksort in practice because heap operations are cache-unfriendly.
- Used as the fallback in **introsort** (quicksort that switches to heap sort when recursion goes too deep) — the algorithm under C++ `std::sort` and Rust's `sort_unstable`.

### Bucket Sort

Distribute elements into buckets by some property (e.g. value range), sort each bucket, concatenate.

- Best when data is uniformly distributed across the range.
- Time: Θ(n + k) average, Θ(n²) if all elements pile into one bucket.
- Used in: hash partitioning before per-partition sort (Spark / Hadoop), counting sort (special case where buckets = single values).

### Radix Sort

Sort by least-significant digit, then next, ... using a stable sort within each pass.

- Time: Θ(d·(n + b)) where d = number of digits, b = base.
- Beats Θ(n log n) for fixed-width keys (e.g. 32-bit ints, fixed-length strings).
- Used in: GPU sorting (CUDA Thrust), GPU databases, HPC, BigQuery's internal sort for fixed-width keys.

### Counting Sort

Count occurrences of each value, prefix-sum, place. Linear time when value range is small.

- Θ(n + k). Stable. Non-comparison.
- Used as the building block of radix sort.

---

## INTERMEDIATE — Stability, Comparators, In-Place

### Stability — Why It Matters

A sort is **stable** if equal elements keep their original relative order.

```
Input:   [(Alice, 90), (Bob, 85), (Carol, 90), (Dave, 80)]
Stable sort by score desc:   [(Alice, 90), (Carol, 90), (Bob, 85), (Dave, 80)]
Unstable could give:          [(Carol, 90), (Alice, 90), (Bob, 85), (Dave, 80)]
```

**Real-world necessity:** sorting by multiple keys via stable single-key sort. To sort by (department, name): first sort by name, then *stably* sort by department. The within-department name order is preserved. Pandas `sort_values(..., kind='mergesort')` is the explicit stable option.

In Python, `sorted` and `list.sort` are guaranteed stable (since 2.3) because Timsort is stable. **Never assume non-Python sorts are stable** — `numpy.sort` defaults to quicksort (unstable); use `kind='stable'`.

### Comparators and Keys

```python
sorted(items, key=lambda x: (-x.priority, x.timestamp))
```

The `key` function is called once per element (not n log n times) — efficient. Always prefer `key=` over a `cmp` function (Python 3 doesn't even have cmp without `functools.cmp_to_key`).

For complex sorts, return a tuple — Python compares lexicographically:

```python
# Sort by department asc, then salary desc, then name asc
sorted(emps, key=lambda e: (e.dept, -e.salary, e.name))
```

For non-numeric reverse, use `functools.cmp_to_key`.

### In-Place vs Out-of-Place

| | In-place | Out-of-place |
|---|---|---|
| Memory | O(1) extra (or O(log n) for stack) | O(n) extra |
| Examples | Selection, insertion, quick, heap | Merge, Tim, bucket, radix |
| Trade | Slow (random access) or unstable | Fast and stable, but O(n) memory |

**Tradeoff exists everywhere in DE:** Spark's sort can spill to disk (out-of-place external) when memory is bounded. Pandas can't — it's all in-RAM.

---

## ADVANCED — Timsort and External Sort

### Timsort — What Python Actually Uses

Timsort (Tim Peters, 2002) is a hybrid merge sort + insertion sort that exploits **runs** (already-sorted subsequences) in real-world data.

Algorithm sketch:
1. Scan the array left to right, identifying runs (ascending or strictly descending).
2. If a run is shorter than `minrun` (~32–64), extend it with insertion sort up to `minrun`.
3. Push runs onto a stack. Merge runs with **invariants** that maintain balanced merge depths (the famous galloping mode).
4. Final merge to combine the remaining runs.

**Why it wins:**
- O(n) on already-sorted or reverse-sorted data (one run, no merges).
- Stable.
- Adaptive — reuses pre-existing order.
- Cache-aware merge (galloping skips long runs of one source).

Adopted by Java (`Arrays.sort` for objects), Android, V8 (`Array.prototype.sort`), Rust's `slice::sort`, Swift, and others. **It's arguably the most-deployed algorithm of the last 25 years.**

A 2015 paper found a stack-invariant bug in Timsort's worst case. Python's implementation was patched. Worth name-dropping: ["Refining Timsort's Merge Invariant"](https://drops.dagstuhl.de/opus/volltexte/2018/9591/).

### Patdefaultpat — Pattern-Defeating Quicksort (the modern quick variant)

`pdqsort` (Orson Peters) — used by Rust's `sort_unstable`, C++ `std::sort` libstdc++ since GCC 14, Boost. Hybrid quicksort with insertion sort fallback for small arrays + heap sort fallback when recursion gets too deep + special handling for already-sorted, reversed, or repeated patterns.

Better worst-case constants than Timsort for *unstable* sort. Now the default in many systems languages.

### External Merge Sort — How DBs and Spark Sort

If your data doesn't fit in memory, you can't use any in-memory sort directly. You do:

1. **Run-generation phase:** read into memory in chunks of size M (e.g. 1GB); sort each chunk in memory; write each sorted run to disk. After this you have ~⌈n/M⌉ sorted files.
2. **Merge phase:** k-way merge the runs using a heap (see [Trees & Heaps](./07-trees-and-heaps.md)) and stream the output. If runs > k, do multiple merge passes (each pass reduces run count by factor k).

Total I/O: 2·⌈log_k(n/M)⌉ passes × n bytes. For typical n = 1TB, M = 16GB, k = 64: ~2 passes — 2TB read and 2TB written.

This is what:
- Spark does for `sortBy`, `orderBy`, sort-merge join.
- Hadoop / MapReduce shuffles do.
- Postgres does for `ORDER BY` larger than `work_mem`.
- Snowflake / BigQuery / Redshift do for big sorts.
- The classic Unix `sort` command does (it spills to `$TMPDIR`).

The "sort runs" analogy is also why Spark's shuffle write is so I/O heavy.

### Distributed Sort (TeraSort)

Even external sort doesn't work when one machine can't hold all data. The **terasort** approach (won the Sort Benchmark in 2008):

1. **Sample** the input to estimate the value distribution.
2. **Partition** by sampled splits so each machine gets roughly equal contiguous ranges.
3. **Sort** each partition locally (external if needed).
4. **Output** in partition order — global ordering is implicit.

Spark uses this for `sortByKey`. TeraSort holds records for sorting 1TB in 60 seconds (2008) and 100PB benchmarks in modern runs.

### Comparison-Based Lower Bound

You can prove that any **comparison-based** sort needs Ω(n log n) comparisons in the worst case. The proof: there are n! permutations; each comparison gives 1 bit of info; ⌈log₂(n!)⌉ = Θ(n log n).

**Non-comparison sorts** (counting, radix, bucket) escape this bound by using more info than just compare — they need a known finite range. That's why GPU/HPC sorts use radix.

### Specialized DE Sorts

- **Pandas** — Timsort (mergesort) by default; quicksort or heapsort selectable via `kind=`.
- **NumPy** — quicksort by default for `numpy.sort`. Use `kind='stable'` for stable sort.
- **Polars** — multithreaded radix sort for fixed-width types, partition-based sort for big ones.
- **DuckDB** — vectorized sort with cache-aware merge; spills to disk.
- **Arrow** — multithreaded radix-quicksort hybrid in C++.

---

## Common Interview Gotchas

**Q: What sort does Python use?**
A: Timsort (since 2.3). Hybrid mergesort + insertion sort. Stable. O(n) best case on sorted/nearly-sorted data; O(n log n) worst.

**Q: What's the lower bound for comparison sort?**
A: Ω(n log n) by the decision-tree argument — there are n! permutations, each compare gives 1 bit.

**Q: How can radix sort be O(n)?**
A: It's O(d·n) where d is digit count. For fixed-width keys, d is constant, giving O(n). Doesn't violate the comparison lower bound because radix isn't comparison-based.

**Q: Is quicksort always faster than mergesort?**
A: No. Quicksort has worst-case O(n²); mergesort is always O(n log n). But quicksort has better cache behavior on average (in-place) and tighter constants. For stability or guaranteed worst-case, prefer mergesort.

**Q: How do you sort 1TB of data on a 16GB machine?**
A: External merge sort. Sort 12GB chunks in memory, write to disk; k-way merge.

**Q: How do you sort by multiple keys?**
A: Pass a tuple to `key=`, or use stable sort multiple times in reverse priority order.

**Q: Why is `sorted` faster than calling `list.sort`?**
A: They're roughly equivalent. `sorted` returns a new list; `list.sort` mutates in place (slightly less memory).

**Q: When is bucket sort the right choice?**
A: When values are uniformly distributed in a known range. Used internally by radix sort and Spark partitioning.

**Q: Difference between `sort()` and `sorted()` in Python?**
A: `lst.sort()` mutates and returns None; `sorted(iterable)` returns a new list and accepts any iterable.

---

## Where Sorting Lives in Real Systems

| System | Sorting choice | Why |
|---|---|---|
| Python `sorted` / `list.sort` | Timsort | Adaptive, stable |
| NumPy default | Quicksort | Speed; not stable |
| Pandas default | Mergesort/Timsort | Stable for DataFrame ops |
| Polars | Multithreaded radix | Fixed-width, parallel |
| Spark `orderBy` | Sample → partition → external sort | Distributed terasort variant |
| Postgres `ORDER BY` | Quicksort if fits memory; external merge if not | Standard DB sort |
| Redshift / Snowflake | Distributed external sort | Built into the engine |
| Java `Arrays.sort` (objects) | Timsort | Stable |
| Java `Arrays.sort` (primitives) | Dual-pivot quicksort | No need for stability |
| Rust `sort` / `sort_unstable` | Timsort / pdqsort | Stable / fast |
| Linux `sort` command | External merge sort | Spills to `$TMPDIR` |

---

## Interview-Ready Cheat Sheet

### One-liners to memorize

- **Comparison-based lower bound: Ω(n log n).**
- **Python uses Timsort. Best O(n), average/worst O(n log n), stable.**
- **Quicksort is not stable, has O(n²) worst case, but great cache locality.**
- **Mergesort is always O(n log n) and stable but needs O(n) extra memory.**
- **Heap sort is O(n log n) worst case, in place, not stable, slow constants.**
- **Radix sort beats O(n log n) for fixed-width keys (it's not comparison-based).**
- **For data that doesn't fit in memory: external merge sort with k-way merge.**
- **For data across machines: sample → partition → local sort → output (TeraSort).**

### Quick Trade-off Pairs

- **Quick vs merge:** in-place vs stable; cache-friendly vs guaranteed O(n log n).
- **Comparison vs non-comparison:** general vs faster but requires known range / fixed width.
- **In-place vs out-of-place:** memory vs speed/stability.
- **Single-machine vs external vs distributed:** which size of data; same algorithmic family at each level.

---

## Resources & Links

- *CLRS* ch. 6, 7, 8 — sorting fundamentals.
- [Tim Peters — Timsort description](https://github.com/python/cpython/blob/main/Objects/listsort.txt) — read this once.
- [Refining Timsort's Merge Invariant](https://drops.dagstuhl.de/opus/volltexte/2018/9591/)
- [pdqsort paper / blog](https://github.com/orlp/pdqsort)
- [TeraSort paper](https://sortbenchmark.org/) — sort benchmark site.
- [Designing Data-Intensive Applications, ch. 10](https://dataintensive.net/) — batch sorting in Hadoop.

### Companion Files
- [Trees & Heaps](./07-trees-and-heaps.md) — k-way merge with `heapq.merge`.
- [Algorithm Design](./09-algorithm-design.md) — divide & conquer mental model.
- [DE: Spark Fundamentals](../data-engineer/07-spark-fundamentals.md) — sort-merge join.

---

*Next: [Algorithm Design Strategies](./09-algorithm-design.md).*
