# Algorithm Design Strategies — Recursion, D&C, Greedy, DP, Reduction

**Phase:** 3 (Algorithms)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Complexity Analysis](./01-complexity-analysis.md) · [Sorting](./08-sorting.md) · [Graphs & Algorithms](./10-graphs-and-algorithms.md)

---

## Why This File Exists

Most "algorithms" you'll be asked aren't on a list — they're new problems where you have to **pick a strategy**. The strategies are countable: recursion, divide and conquer, greedy, dynamic programming, reduction to a known problem, and "leverage the right data structure." Knowing when each wins is the senior signal.

---

## Recursion — Decompose into Smaller Subproblems

### The Pattern

```python
def f(input):
    if base_case(input):
        return base_value
    return combine(f(smaller), f(smaller2))
```

Three elements:
1. **Base case** — when to stop.
2. **Recursive case** — work on a smaller instance.
3. **Combine** — build the answer from sub-answers.

### Examples

- Factorial, Fibonacci (naive — exponential without memoization).
- Tree traversals — natural recursion.
- Generating subsets / permutations.
- Backtracking (N-queens, Sudoku, word search).
- Parsers (recursive descent).

### Pitfalls in Python

- **Recursion limit ~1000.** Deep recursion crashes. Use `sys.setrecursionlimit` or convert to iteration with an explicit stack.
- **No tail-call optimization.** A recursive call always allocates a new frame.
- **Slicing inside recursion is O(n).** `arr[1:]` copies — use indices instead.

```python
# Bad — O(n²) due to slicing copies
def sum_arr(arr):
    if not arr: return 0
    return arr[0] + sum_arr(arr[1:])

# Good — O(n)
def sum_arr(arr, i=0):
    if i == len(arr): return 0
    return arr[i] + sum_arr(arr, i + 1)
```

### Recurrence Analysis

For T(n) = a·T(n/b) + f(n), apply the **Master Theorem** (covered in [Complexity Analysis](./01-complexity-analysis.md)).

For non-divide recurrences (T(n) = T(n-1) + ...), substitution or recursion-tree analysis.

---

## Divide and Conquer

### The Pattern

1. **Divide** the problem into subproblems.
2. **Conquer** each subproblem recursively.
3. **Combine** sub-solutions.

The defining feature: subproblems are **independent.** (Contrast with DP, where they overlap.)

### Canonical Examples

| Algorithm | Recurrence | Result |
|---|---|---|
| Mergesort | 2·T(n/2) + O(n) | Θ(n log n) |
| Quicksort | 2·T(n/2) + O(n) average | Θ(n log n) avg, Θ(n²) worst |
| Binary search | T(n/2) + O(1) | Θ(log n) |
| Strassen's matrix multiply | 7·T(n/2) + O(n²) | Θ(n^2.807) |
| Closest pair of points | 2·T(n/2) + O(n) | Θ(n log n) |
| FFT | 2·T(n/2) + O(n) | Θ(n log n) |

### When to Reach for D&C

- The problem has a **natural midpoint** (sorted array, median, geometric center).
- Subproblems are **independent** — no shared sub-answers.
- You want **parallelism** — D&C maps directly to fork/join.

### DE Connection: D&C Underlies Distributed Compute

MapReduce, Spark, Flink — all are **fork/join D&C** at scale. "Process each partition independently, then combine." The same insight that gives you mergesort gives you a Spark plan.

---

## Greedy Algorithms

### The Pattern

At each step, take the **locally best** choice. Hope it adds up to a global optimum.

### When Greedy Works

- The problem has the **greedy-choice property**: a globally optimal solution can be reached by local optima.
- The problem has **optimal substructure**: optimal solutions to subproblems compose into an optimal whole.

### Classic Greedy Algorithms (and where they prove correct)

| Algorithm | Greedy choice | Used in |
|---|---|---|
| **Activity selection** | Earliest finish time | Scheduling |
| **Huffman coding** | Merge two least-frequent symbols | gzip, JPEG, MP3 |
| **Dijkstra's shortest path** | Closest unvisited node | Routing, Maps |
| **Prim's MST** | Cheapest edge from tree | Network design |
| **Kruskal's MST** | Cheapest edge that doesn't form a cycle | Same |
| **Fractional knapsack** | Highest value/weight ratio | Resource allocation |
| **Coin change (canonical sets)** | Largest coin first | US/EU coin systems |

### When Greedy *Fails*

- **0/1 knapsack** — local greedy by value/weight ratio doesn't always optimize.
- **Coin change with non-canonical sets** (e.g. coins {1, 3, 4} for amount 6: greedy gives 3 coins, optimal is 2).
- **Most NP-hard problems** — greedy is a heuristic, not optimal.

**The senior signal:** when an interviewer asks "is your greedy approach optimal?" — be ready to either prove it (exchange argument, matroid theory) or admit it's a heuristic and discuss approximation guarantees.

### Real DE/AI Use

- **Beam search** in LLM decoding — greedy variant keeping top-k partial sequences.
- **Job scheduling** in Airflow — greedy ordering of tasks by priority/deadline.
- **Cache eviction** — LRU is greedy on recency.
- **Compression** — gzip uses Huffman + LZ77 (LZ77 is greedy on longest match).

---

## Dynamic Programming

### The Pattern

DP applies when subproblems **overlap** (the same sub-answer is needed many times). Two implementations:

- **Top-down memoization** — recursive + cache results.
- **Bottom-up tabulation** — fill a table iteratively.

### When to Recognize DP

Two boxes must check:
1. **Optimal substructure** — sub-answers compose into the answer.
2. **Overlapping subproblems** — same sub-answer is reused.

If only #1 holds, it's D&C. Both holds → DP.

### The DP Recipe

1. **Define the state** — what does `dp[i]` (or `dp[i][j]`) mean?
2. **Define the transition** — how is `dp[i]` computed from smaller states?
3. **Base case** — `dp[0]` or `dp[''][0]`.
4. **Order of computation** — usually small → large.

### Canonical DP Problems (each is a template)

| Problem | State | Transition |
|---|---|---|
| Fibonacci | dp[i] = nth Fib | dp[i] = dp[i-1] + dp[i-2] |
| Coin change (min coins) | dp[i] = min coins for amount i | dp[i] = 1 + min(dp[i-c] for c in coins) |
| Longest increasing subsequence (LIS) | dp[i] = LIS ending at i | dp[i] = 1 + max(dp[j] for j<i with arr[j]<arr[i]) |
| Edit distance | dp[i][j] = edit cost between A[:i] and B[:j] | three-way recurrence |
| 0/1 knapsack | dp[i][w] = best value with first i items, weight ≤ w | take or skip |
| Longest common subsequence | dp[i][j] | char match → +1; else max of skipping each |
| Matrix chain multiplication | dp[i][j] = min cost for parens of products i..j | try every split point |
| Subset sum | dp[i][s] = can we form s from first i items | take or skip |

### Top-Down with `lru_cache`

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def edit_distance(a, b):
    if not a: return len(b)
    if not b: return len(a)
    if a[0] == b[0]:
        return edit_distance(a[1:], b[1:])
    return 1 + min(
        edit_distance(a[1:], b),       # delete
        edit_distance(a, b[1:]),       # insert
        edit_distance(a[1:], b[1:]),   # substitute
    )
```

`lru_cache` turns exponential into polynomial in one line. **Highest-leverage decorator in interviews.**

### Bottom-Up Tabulation

```python
def edit_distance(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1): dp[i][0] = i
    for j in range(n + 1): dp[0][j] = j
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    return dp[m][n]
```

### Space Optimization

Many DP problems only need the last 1 or 2 rows of the table. Convert `dp[m][n]` to two 1D arrays for O(n) space.

### DP in DE/AI

- **LLM decoding** — viterbi/beam search use DP-style recurrences over partial sequences.
- **Sequence alignment** (BLAST, smith-waterman) — DP on edit-distance variants.
- **Query optimizers** — Selinger DP-style join ordering for small queries.
- **Reinforcement learning** — Bellman equation is DP at its core.
- **Hidden Markov Models** — Viterbi is DP.

---

## Reduction — "This Is Actually Problem X"

### The Pattern

Map your unfamiliar problem to a known one whose algorithm you know. The transformation might be obvious or clever.

### Examples

- "Find pairs that sum to K" → reduce to set lookup. O(n).
- "Detect cycle in dependencies" → DFS on graph. O(V+E).
- "Schedule jobs with constraints" → topological sort. O(V+E).
- "Pick non-overlapping intervals" → activity selection (greedy).
- "Find duplicates in a stream" → reduce to set / Bloom filter.
- "K most frequent items in a stream" → reduce to count + heap.
- "Maximum flow in bipartite graph" → reduce to Hopcroft-Karp matching.

### Recognizing NP-Hard

If your problem reduces to or *contains* an NP-hard problem (3-SAT, vertex cover, knapsack, TSP, set cover), don't keep grinding for polynomial-time. Pivot:

- **DP if dimensions are small.** 0/1 knapsack with W ≤ 10⁶ is solvable.
- **Approximation algorithms.** 2-approximation for vertex cover; PTAS for knapsack.
- **Heuristics.** Simulated annealing, genetic algorithms, beam search.
- **Reduce to ILP / SAT and use a solver.** Gurobi, OR-Tools, Z3.

**The senior move:** "I notice this reduces to vertex cover, which is NP-hard. So I'd switch to [approximation / DP if W small / heuristic]." That single sentence shifts the round from struggle to design.

---

## Leverage Data Structures

### The Pattern

The right data structure converts an O(n²) brute force into O(n) or O(n log n).

| Problem | Brute force | With right structure |
|---|---|---|
| "Is X in this list?" | O(n) | O(1) with `set` |
| "Most common element" | O(n²) (counting per pair) | O(n) with `dict` / `Counter` |
| "Top K of stream" | O(n log n) sort | O(n log k) heap |
| "Range query in time-series" | O(n) scan | O(log n) with B-tree / sortedcontainers |
| "Has substring?" | O(n·m) | O(n + m) with KMP / Z / Aho-Corasick |
| "Approx unique count" | O(n) memory | O(1) memory with HyperLogLog |
| "Approx membership" | O(n) memory | O(n) bits with Bloom filter |
| "Connectivity in evolving graph" | O(V+E) per query | O(α(n)) with Union-Find |
| "kNN over high-dim vectors" | O(n·d) | O(log n · d) with HNSW |

### The Mindset

When you see a brute-force solution, ask: **what's the bottleneck operation?** Then pick the structure whose strength is that operation. This is the universal pattern across [hash tables](./06-hash-tables.md), [trees](./07-trees-and-heaps.md), [tries](./11-tries-strings-union-find.md), [union-find](./11-tries-strings-union-find.md), and [probabilistic structures](./12-probabilistic-structures.md).

---

## Strategy Selection — A Decision Algorithm

When you see a new problem:

1. **Can I see a brute force?** Code it (or describe it). Note the complexity.
2. **What's the bottleneck operation?** Look up the structure that makes it fast → "leverage data structure."
3. **Does the optimal solution build on smaller versions of itself?** → recursion.
4. **Are subproblems independent?** → divide and conquer.
5. **Do subproblems overlap?** → dynamic programming.
6. **Does a local choice always lead to a global optimum?** → greedy (and prove it).
7. **Does this look like a known problem (TSP, knapsack, vertex cover, sort, matching)?** → reduction.
8. **Is it provably NP-hard?** → approximation / heuristic / DP-on-small / SAT solver.

If you walk an interviewer through these seven gates out loud, you sound like a senior even if you don't reach the optimal answer immediately.

---

## Common Interview Gotchas

**Q: When is greedy optimal?**
A: When the problem has the greedy-choice property + optimal substructure. Provable via matroid theory or exchange arguments.

**Q: What's the difference between D&C and DP?**
A: Both decompose into subproblems. D&C subproblems are *independent*; DP subproblems *overlap* and benefit from caching.

**Q: When would memoization not help?**
A: When subproblems don't overlap (D&C). When state space is too large to cache. When recursion already terminates fast (e.g. binary search).

**Q: Why is recursion in Python slow?**
A: Function call overhead is high in CPython, plus you fight the recursion limit. Iterative versions are usually 5–10× faster.

**Q: How would you tackle a problem you've never seen?**
A: Walk the seven-gate decision algorithm above out loud.

**Q: What's the time complexity of a recursive Fibonacci without memoization?**
A: O(2^n) — the recursion tree has ~2^n nodes. With memoization: O(n).

**Q: What problems have polynomial-time DP solutions but exponential brute force?**
A: Edit distance, knapsack (with W bounded), LIS, matrix chain multiplication, all-pairs shortest paths (Floyd-Warshall).

---

## Interview-Ready Cheat Sheet

### Strategy → typical problems

| Strategy | Tells |
|---|---|
| Recursion | "Solve the problem on a smaller version" |
| D&C | "Halves of the problem are independent" |
| Greedy | "Local choice never hurts" — prove it |
| DP | "Subproblems overlap, optimal substructure" |
| Reduction | "Looks like X with relabeled inputs" |
| Data structure | "Brute force is O(n²); right structure is O(n)" |

### Trade-off pairs

- **Recursion vs iteration:** clarity vs Python's stack/perf.
- **Top-down memoization vs bottom-up tabulation:** code clarity vs control over space.
- **Greedy vs DP:** speed vs guaranteed optimality (when greedy fails).
- **Exact (DP) vs approximate (heuristic):** correctness vs scale.

### Top six Q&As

**"Is your solution optimal?"** → State the proof or admit it's a heuristic; give approximation guarantee if known.

**"Why is brute force not good enough?"** → Cite the complexity, do back-of-envelope on input size.

**"Can you do this with less memory?"** → DP roll-back to last 1–2 rows; in-place sort; streaming algorithms.

**"What if input is huge?"** → External sort; streaming + probabilistic structures; distributed.

**"What's the recurrence and what's its solution?"** → Master Theorem.

**"Does greedy work here?"** → Exchange argument or counterexample.

---

## Resources & Links

- *CLRS* ch. 15 (DP), 16 (greedy), 4 (recurrences), 34 (NP-hard).
- *Algorithm Design Manual* (Skiena) — focuses on strategy selection.
- [Erik Demaine's MIT 6.006 lectures (free)](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/)
- [Competitive Programmer's Handbook (Antti Laaksonen)](https://cses.fi/book/book.pdf) — free PDF, dense reference.
- [Algorithms by Jeff Erickson (free)](https://jeffe.cs.illinois.edu/teaching/algorithms/)
- [Exchange argument writeup](https://en.wikipedia.org/wiki/Greedy_algorithm#Cases_of_failure) — for greedy proofs.

### Companion Files
- [Complexity Analysis](./01-complexity-analysis.md) — Master Theorem.
- [Sorting](./08-sorting.md) — D&C in action (mergesort).
- [Graphs & Algorithms](./10-graphs-and-algorithms.md) — DP on DAGs, Dijkstra/Prim greedy.

---

*Next: [Graphs — Representations & Algorithms](./10-graphs-and-algorithms.md).*
