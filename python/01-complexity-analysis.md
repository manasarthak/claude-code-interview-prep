# Complexity Analysis — Big-O, Worst Case, Space, and Intractability

**Phase:** 0 (Mandatory foundation)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Python Internals](./02-python-internals.md) · [DE: Partitioning & Performance](../data-engineer/13-partitioning-performance.md)

---

## Why This File Exists

Every later file in this folder, every interview answer about a structure, every system-design discussion of "would that scale?" — they all use the same vocabulary: Big-O, Big-Θ, amortized, worst-case, space complexity, intractability. If this vocabulary is fuzzy, you sound junior even when you know the right answer.

This file gives you the language so the rest of the section can use it without re-deriving.

---

## BEGINNER — What Are We Even Measuring?

### The One-Sentence Definition

Complexity analysis describes how the time (or space) required by an algorithm grows as the input size grows — and we describe that growth using **Big-O notation**, which characterizes the *upper bound* of growth, ignoring constants and lower-order terms.

### Why We Don't Just Time It

Wall-clock time depends on:
- The hardware (your laptop vs. an EC2 c7g.16xlarge)
- The language (Python vs. C is ~30–100×)
- Other processes running
- Whether the data is in CPU cache, RAM, SSD, or network

Big-O abstracts all of that away. An O(n²) algorithm is O(n²) on every machine; the constants change, the *shape* doesn't. **Big-O is about scaling, not speed.**

### The Notations You Must Know Cold

| Notation | What it means | Mental model |
|---|---|---|
| **O(f(n))** | Upper bound — "grows *no faster than* f(n)" | The algorithm's time is *at most* f(n) for large n |
| **Ω(f(n))** | Lower bound — "grows *at least as fast as* f(n)" | The algorithm's time is *at least* f(n) for large n |
| **Θ(f(n))** | Tight bound — both O and Ω | The algorithm's time grows *exactly* as f(n) |
| **o(f(n))** (little-o) | Strictly slower than f(n) | Rare in interviews |

In practice, "Big-O" is used loosely to mean "the tightest upper bound I can give." When someone says "binary search is O(log n)," they really mean Θ(log n) — both upper and lower.

### The Common Growth Rates (memorize this ladder)

```
O(1)       constant       — hash lookup, array index
O(log n)   logarithmic    — binary search, balanced tree op
O(n)       linear         — single pass through a list
O(n log n) linearithmic   — Timsort, mergesort, FFT
O(n²)      quadratic      — nested loop, naive matrix multiply
O(n³)      cubic          — Floyd-Warshall, naive 3-loop
O(2ⁿ)      exponential    — brute-force subsets, naive recursion
O(n!)      factorial      — brute-force permutations (TSP)
```

For n = 1,000,000:
- O(1) → 1 op
- O(log n) → ~20 ops
- O(n) → 1,000,000 ops (~1ms in C, ~100ms in Python)
- O(n log n) → ~20,000,000 ops (~1s in Python)
- O(n²) → 10¹² ops (≈11 days in Python — never run this)
- O(2ⁿ) → don't even.

**Practical interview rule:** if n ≤ 20, exponential might be fine. If n ≤ 5000, O(n²) is okay. For n > 10⁶, you need O(n log n) or better.

### Why We Drop Constants and Lower Terms

`f(n) = 3n² + 7n + 50` is O(n²) because:
- The 50 vanishes for large n (constant gets dwarfed)
- The 7n vanishes (n² grows faster)
- The 3 doesn't matter — that's a constant factor, hardware-dependent

So `O(3n²)` is wrong (well, *technically* correct, but you sound green saying it). Just say O(n²).

**The exception that bites you in production:** a 100× constant matters in real life. An O(n) Python loop is often slower than an O(n²) NumPy op because the constants differ by ~100×. *Big-O tells you how it scales, not whether it's fast today.*

---

## INTERMEDIATE — Worst, Average, Amortized

### The Three Cases

For most algorithms, we care about three different scenarios:

| Case | What it asks | Example: hash lookup |
|---|---|---|
| **Best case** | Fastest possible run | O(1) — first hash slot is the right one |
| **Average case** | Expected over random inputs | O(1) — assuming good hash function |
| **Worst case** | Slowest possible run | O(n) — every key collides into one bucket |

**Interviewer test:** if asked "what's the complexity of dict lookup?" the right answer is **"O(1) average, O(n) worst case (when all keys hash to the same slot)"** — not just "O(1)." Demonstrates depth.

### Amortized Analysis — The One Most Candidates Get Wrong

Some operations are *usually* fast but *occasionally* expensive. Amortized analysis averages the cost over a sequence of operations.

**Canonical example: Python `list.append()`.**

CPython's `list` is a dynamic array backed by a C array. When the array fills up, it reallocates a larger one and copies everything over. That single copy is O(n). But it only happens every Θ(n) appends, and the array grows geometrically (CPython grows by ~12.5%, others use 2×).

So over n appends:
- Most appends: O(1)
- Occasional resize copies: O(1) + O(2) + O(4) + ... + O(n) = O(2n)
- Total cost for n appends: O(n)
- **Amortized cost per append: O(1)**

Three flavors of amortized analysis:
- **Aggregate** — total cost / number of operations (the example above).
- **Banker's / accounting method** — each cheap op "saves" credit to pay for future expensive ones.
- **Potential method** — define a potential function Φ; each op's amortized cost = actual cost + ΔΦ.

You don't need to compute these in interviews, but you should *recognize* when "amortized O(1)" applies. Saying "list append is O(1) amortized" is a senior signal.

### Worst-Case Determination — A Mental Algorithm

When asked "what's the worst-case complexity of X?" do this:

1. **Find the dominant loop or recursion.** Outermost first.
2. **Multiply nested loops.** Three nested loops over n → O(n³).
3. **Sum sequential steps; the largest dominates.** O(n) + O(n²) = O(n²).
4. **For recursion: write the recurrence and apply Master Theorem.**
5. **For randomized algorithms: distinguish expected from worst-case.** Quicksort is O(n log n) expected, O(n²) worst.

### The Master Theorem (the only recurrence shortcut you need)

For recurrences of the form `T(n) = a·T(n/b) + f(n)`:

| Case | Condition | Result |
|---|---|---|
| 1 | f(n) = O(n^c) where c < log_b(a) | T(n) = Θ(n^log_b(a)) |
| 2 | f(n) = Θ(n^c) where c = log_b(a) | T(n) = Θ(n^c · log n) |
| 3 | f(n) = Ω(n^c) where c > log_b(a) (with regularity) | T(n) = Θ(f(n)) |

**Examples:**
- Mergesort: T(n) = 2·T(n/2) + O(n) → Case 2 → **Θ(n log n)**
- Binary search: T(n) = T(n/2) + O(1) → Case 2 → **Θ(log n)**
- Strassen's matrix mult: T(n) = 7·T(n/2) + O(n²) → Case 1 → **Θ(n^log₂ 7) ≈ Θ(n^2.807)**

---

## ADVANCED — Space Complexity, Models, Intractability

### Space Complexity

Same notation as time, but counts memory.

- **Auxiliary space** — extra space beyond the input. (What interviewers usually care about.)
- **Total space** — input + auxiliary.

For recursion, the call stack counts. A naive Fibonacci recursion is O(n) space (call depth) even though its time is O(2ⁿ).

**Practical examples:**
- Iterative reverse-a-list: O(1) auxiliary.
- Recursive reverse-a-list: O(n) auxiliary (stack frames).
- Mergesort: O(n) auxiliary (the merge buffer). This is why Quicksort is preferred when memory is tight.
- Quicksort: O(log n) auxiliary (call stack), O(n) worst case.
- DFS: O(V) for the visited set + O(V) for the stack.
- BFS: O(V) for the queue (can be much larger than DFS in wide graphs).

**Real-world: this is why Spark spills to disk.** When a shuffle requires more than executor memory, Spark writes intermediate partitions to disk. The complexity hasn't changed, but the constants have exploded by 100×. See [Spark Fundamentals](../data-engineer/07-spark-fundamentals.md).

### Memory Hierarchy & Cache Effects (the constant Big-O hides)

Big-O treats every memory access as O(1), but in reality:

| Access | Latency |
|---|---|
| L1 cache hit | ~1 ns |
| L2 cache hit | ~5 ns |
| L3 cache hit | ~20 ns |
| RAM | ~100 ns |
| NVMe SSD | ~100 μs |
| Network (same DC) | ~500 μs |
| Disk seek (HDD) | ~10 ms |

That's a 10⁷ range — and it's why a "cache-friendly" O(n²) algorithm sometimes beats a cache-unfriendly O(n log n). It's why columnar storage (Parquet/Arrow) beats row storage for analytics: contiguous reads of one column are cache-friendly, while jumping rows is not. See [Warehouses & Lakehouses](../data-engineer/03-warehouses-lakes-lakehouse.md).

### Computational Models (briefly)

The model affects what counts as O(1):

- **RAM model** — most algorithms textbooks. Memory access is O(1).
- **External memory / I/O model** — used for databases. We count I/Os, not ops. This is why **B+ trees** beat BSTs on disk: each I/O fetches a page that holds many keys.
- **Cache-oblivious model** — algorithms that work well at every level of the memory hierarchy without tuning.
- **Distributed model (BSP, MapReduce)** — counts communication rounds and per-round work. This is why Spark optimizes shuffles.

### Intractability — P, NP, NP-hard, NP-complete

You need to recognize these terms even if you'll never prove anything in an interview.

- **P** — problems solvable in polynomial time. (Sorting, shortest paths.)
- **NP** — problems whose solutions can be *verified* in polynomial time. (Sudoku.)
- **NP-hard** — at least as hard as the hardest NP problem. (TSP optimization version.)
- **NP-complete** — in NP and NP-hard. (SAT, vertex cover, 0/1 knapsack.)
- **P vs NP** — open problem. If P = NP, all of cryptography breaks.

**Why this matters in interviews:** the moment you recognize "this is the traveling salesman problem disguised" or "this reduces to bin packing," you stop trying to find an exact polynomial-time solution and pivot to:

1. **Approximation algorithms** — guaranteed-within-c-of-optimal in polynomial time.
2. **Heuristics** — no guarantee, often great in practice (simulated annealing, beam search).
3. **Solving for small n** — DP on bitmasks works up to n ≈ 20–25.
4. **Reduction to a solver** — model it as ILP, SAT, or constraint programming.

This pivot is the difference between "I'm stuck" and "I see why this is hard; here's how I'd handle it."

### Complexity Classes Beyond P/NP (name-drop tier)

- **PSPACE** — solvable with polynomial space (any time).
- **EXPTIME** — exponential time.
- **BPP** — solvable in polynomial time with bounded error using randomness (where Bloom filters live).
- **#P** — counting solutions to NP problems.

You won't be quizzed on these but a single mention ("this looks like a #P problem so I'd estimate with sampling") signals depth.

---

## Where Complexity Bites You in Real DE/AI Systems

| Situation | Complexity insight |
|---|---|
| Spark job runs forever | Probably an O(n²) join (cross-product or skew-blowup); fix with broadcast or salting. See [Spark](../data-engineer/07-spark-fundamentals.md). |
| Postgres slow query | Sequential scan O(n) where index would be O(log n). `EXPLAIN ANALYZE` first. |
| RAG retrieval slow | Exact KNN is O(n·d) per query; switch to HNSW for O(log n·d). See [Vector Search](./17-vector-search-structures.md). |
| LLM prompt cost ballooning | Quadratic-in-context-length attention; KV cache fixes generation, not prefill. See [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md). |
| Streaming dedup OOM | Dedup with `set` is O(n) memory; use Bloom filter for O(1) per-item with bounded memory. See [Probabilistic Structures](./15-probabilistic-structures.md). |
| SQL `GROUP BY HIGH_CARDINALITY` | Hash aggregate is O(n) avg; with skew it degenerates to O(n²). |
| Pipeline backfill blowup | Joining 5 years × 5 years cross-product is O(n²); partition + window-merge is O(n log n). |

---

## Common Interview Gotchas

**"What's the complexity of `x in lst` for a Python list?"**
O(n) — linear scan. Yes, even though it looks like a single operation. If you need O(1), use a `set`.

**"What's the complexity of `x in s` for a Python set?"**
O(1) average, O(n) worst case (pathological collisions or `__hash__` returning a constant).

**"What's the complexity of `lst.pop(0)`?"**
O(n) — every element shifts left by one. Use `collections.deque.popleft()` for O(1). This trips up *so* many candidates.

**"What's the complexity of slicing `lst[a:b]`?"**
O(b - a) — slicing copies. A common reason naïve recursion explodes in Python is `arr[1:]` inside the recursive call.

**"What's the complexity of `sorted(iterable)`?"**
O(n log n) — Timsort. Best case O(n) when the input is already sorted (Timsort detects runs).

**"What's the complexity of `''.join(parts)` vs repeated `+=`?"**
`join` is O(n total). Repeated `s += other` in a loop is O(n²) because strings are immutable — every concat copies. This is the most common Python performance gotcha in interviews.

**"What's the complexity of `dict.keys()`, `dict.values()`, `dict.items()`?"**
O(1) — they return view objects, not lists. Iterating them is O(n).

**"How can a `dict` be O(n) worst case if it's hashing?"**
Bad hash function or adversarial inputs cause every key to land in the same slot, degrading to a linked list. Python mitigates this with hash randomization (since 3.4). See [Hash Tables](./06-hash-tables.md).

---

## Interview-Ready Cheat Sheet

### When asked "What's the complexity?"

1. State the operation precisely.
2. State the case (best / average / worst / amortized).
3. State the bound (O / Θ).
4. State the assumption (e.g. "assuming a good hash function").
5. (Senior) State the space complexity too.

> *"Inserting into a Python `list` at the end is O(1) amortized — most appends are constant, but occasional reallocations are O(n). Worst case for any single append is O(n). Space is O(n) total."*

### Quick Trade-off Pairs

| Want | Trade |
|---|---|
| O(1) lookup | Spend O(n) space (hash map) |
| O(log n) insert + sorted access | Use a balanced tree, lose O(1) lookup |
| O(1) min/max | Use a heap, lose O(1) middle access |
| O(1) memory | Often forces O(n²) time (in-place sorts, etc.) |
| O(n) on disk | Optimize for I/O blocks (B+ tree), not pointer chasing (BST) |

### Fast Sanity Checks Before Interview

- For each common structure, can you state every op's complexity from memory?
- Can you state the Master Theorem and apply it to mergesort?
- Can you list 5 NP-complete problems? (SAT, 3-SAT, vertex cover, clique, subset-sum, TSP, knapsack, graph coloring.)
- Can you explain why amortized O(1) ≠ O(1) per op but still worth using?
- Can you spot the cache effect: when does an O(n²) algorithm beat O(n log n)?

---

## Resources & Links

### Foundational
- *Introduction to Algorithms* (CLRS) — chapters 1–4 cover this material rigorously.
- *Algorithms* (Sedgewick & Wayne) — [free Princeton course](https://algs4.cs.princeton.edu/).
- [Python's official time-complexity wiki](https://wiki.python.org/moin/TimeComplexity)
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — printable reference.

### Going Deeper
- [Demaine — MIT Advanced Data Structures](https://courses.csail.mit.edu/6.851/) — graduate-level, optional.
- [Erik Demaine's lectures on amortized analysis (YouTube)](https://www.youtube.com/results?search_query=demaine+amortized+analysis)
- [Computational Complexity by Sipser](https://math.mit.edu/~sipser/book.html) — for P/NP foundations.

### Companion Files
- [Python Internals](./02-python-internals.md) — what these complexities look like in CPython.
- [Sorting](./10-sorting.md) — the canonical playground for complexity analysis.
- [Modern Advances](./18-modern-advances.md) — where the ladder above gets refined in 2024–2026 research.

---

*Next: [Python Internals](./02-python-internals.md) — now that you know the language, see what your built-in types are actually doing.*
