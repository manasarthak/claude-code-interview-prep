# Python — DSA, Algorithms & Internals for DE/AI Interviews

**Phase:** 1 (Foundations) → 3 (Differentiator)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Sister tracks:** [AI Engineering](../ai/) · [Data Engineering](../data-engineer/) · [Projects](../projects/)

---

## Mentor's Note — Why This Section Exists and How It's Different

Almost every Python "data structures and algorithms" guide covers the same canon (linked lists, BSTs, sorting, graphs). What gets you hired in 2026 is *not* knowing those structures — it's knowing:

1. **How CPython actually implements them** (so you understand what your code costs).
2. **The complexity of every operation** at the level you can defend it under interview pressure.
3. **The DE/AI-relevant structures** that don't appear in classical CS curricula but show up in real production systems: Bloom filters, HyperLogLog, LSM trees, B+ trees, HNSW, count-min sketches.
4. **Modern advances** the bootcamp generation hasn't heard of yet — like the [Duan et al. 2025 BMSSP](https://arxiv.org/abs/2504.17033) algorithm that beat Dijkstra's 60-year-old O((m+n)log n) bound for directed SSSP.

This folder reorders the classical syllabus into the order things should actually be learned (complexity → internals → simple structures → ADTs → trees/heaps → graphs → algorithms → modern), and at every step it ties theory to where it shows up in [data engineering](../data-engineer/) and [AI/ML/LLM systems](../ai/).

If you can talk fluently about why a `dict` lookup is O(1) amortized but actually probes multiple slots, why Postgres uses B+ trees and not BSTs, why Pinecone uses HNSW and not exact KNN, and why your Redis sorted-set is actually a skip list — you outclass 90% of "I crushed LeetCode" candidates.

---

## The Reordered Topic Index

The original list was a dump of disconnected items. Here's the order they actually depend on, with everything important added in:

### Tier 0 — Mandatory Foundation

1. [**Complexity Analysis & Big-O**](./01-complexity-analysis.md) — Big-O/Θ/Ω, worst vs average vs amortized, space complexity, intractability (P/NP). *Without this, every other file is just memorization.*
2. [**Python Internals — How CPython Implements Things**](./02-python-internals.md) — `list`, `dict`, `set`, `tuple`, `str`, `frozenset`, `collections.deque`, `heapq`, GIL implications, reference counting, slot-based attribute access. *The single highest-leverage file in this section.*

### Tier 1 — Linear Structures

3. [**Arrays, Buffers, Circular & Partially-Filled Arrays**](./03-arrays-and-buffers.md) — dynamic-array amortized analysis, ring buffers, NumPy/Arrow contiguous memory, columnar storage tie-in.
4. [**Linked Lists**](./04-linked-lists.md) — singly, doubly, sentinel nodes, when (and why) you almost never use them in Python. Skip-list teaser → file 17.
5. [**Stacks, Queues, Deques**](./05-stacks-queues-deques.md) — `list` as stack, `collections.deque` (doubly-linked block list), `queue.Queue` (thread-safe), bounded queues, ring buffers in Kafka/Kinesis.

### Tier 2 — Associative & Hierarchical Structures

6. [**Hash Tables — `dict`, `set`, `Counter`**](./06-hash-tables.md) — open addressing in CPython, perturbation, load factor, hash randomization, `__hash__`/`__eq__` contract, ordered dict guarantee since 3.7. Hash partitioning in distributed systems.
7. [**Trees & Heaps**](./07-trees-and-heaps.md) — BSTs, AVL/Red-Black, why Python's stdlib has no balanced tree, `heapq`, priority queues, k-way merge, top-k streaming.

### Tier 3 — Sorting & Algorithm Design

8. [**Sorting**](./08-sorting.md) — selection/insertion/merge/quick/heap/bucket/radix; CPython's Timsort; external merge sort in Spark/databases.
9. [**Algorithm Design Strategies**](./09-algorithm-design.md) — recursion, divide-and-conquer, greedy, dynamic programming, reduction, "leverage data structures." When each wins.

### Tier 4 — Graphs

10. [**Graphs — Representations & Algorithms**](./10-graphs-and-algorithms.md) — adjacency list/matrix, modeling, DFS, BFS, Dijkstra, Bellman-Ford, A\*, Floyd-Warshall, Kruskal, Prim, topological sort, SCCs.

### Tier 5 — Advanced & Industry-Relevant (the differentiators)

11. [**Tries, Strings & Union-Find**](./11-tries-strings-union-find.md) — tries, suffix arrays, KMP, BPE tokenizers in LLMs, DSU with path compression + union-by-rank.
12. [**Probabilistic Data Structures**](./12-probabilistic-structures.md) — Bloom, Cuckoo, HyperLogLog, Count-Min Sketch, Top-K. BigQuery `APPROX_COUNT_DISTINCT`, Cassandra, Redis, streaming.
13. [**Storage-Engine & Vector-Search Structures**](./13-storage-and-vector-structures.md) — B+ trees, LSM trees, skip lists; HNSW, IVF, PQ, DiskANN. The DE/AI bridge.
14. [**Modern Advances You Should Name-Drop**](./14-modern-advances.md) — BMSSP (2025, sub-Dijkstra SSSP), learned indexes, Bw-trees, Swiss tables, GPU graph algos.

---

## Why This Order

- **Complexity first** because every later file uses the vocabulary.
- **Python internals second** because the choice between `list` and `set` only makes sense once you know the underlying algorithms.
- **Linear → associative → hierarchical → graphs** is the classical dependency order: you can't talk about hash maps without arrays, can't do BFS without queues, can't do Dijkstra without heaps.
- **Algorithms after structures** because nearly every algorithm is "the right structure used cleverly."
- **Industry-relevant last** because they require everything before them as prerequisites.

---

## How Every File Is Structured

Every Python file in this folder follows the same template:

1. **Concept first** — what the structure or algorithm *is* and why it exists.
2. **Big-O table** — every operation, every case, with a one-line justification.
3. **CPython internals** — what's actually inside the C implementation.
4. **Pythonic implementation** — minimal code that captures the structure (not LeetCode boilerplate).
5. **Where it shows up in industry** — explicit DE/AI/LLM examples.
6. **Common interview gotchas** — the tricky bits.
7. **Resources & papers** — for going deeper.

---

## Big-O Cheat Sheet (quick reference)

You'll see this table referenced by every file. Memorize the columns.

| Structure | Access | Search | Insert | Delete | Notes |
|---|---|---|---|---|---|
| Array (`list`) | O(1) | O(n) | O(1)* / O(n) | O(n) | *amortized append; insert in middle is O(n) |
| Linked list | O(n) | O(n) | O(1) at head | O(1) at known node | Python doesn't ship one |
| Stack | — | — | O(1) | O(1) | LIFO |
| Queue | — | — | O(1) | O(1) | FIFO; use `deque`, NOT `list.pop(0)` |
| Deque (`collections.deque`) | O(n) | O(n) | O(1) at ends | O(1) at ends | Block-based doubly linked list |
| Hash map (`dict`) | O(1) avg | O(1) avg | O(1) avg | O(1) avg | O(n) worst (collisions) |
| Hash set (`set`) | — | O(1) avg | O(1) avg | O(1) avg | Same as dict |
| BST (balanced) | — | O(log n) | O(log n) | O(log n) | Python stdlib has no balanced tree |
| Heap | O(1) min/max | O(n) | O(log n) | O(log n) | `heapq` is min-heap only |
| Trie | — | O(k) | O(k) | O(k) | k = key length |
| B+ tree | — | O(log n) | O(log n) | O(log n) | Disk-friendly: each node is a page |
| LSM tree | — | O(log n) | O(1) buffered | O(log n) | Optimized for writes |
| Bloom filter | — | O(k) | O(k) | — | k = #hashes; false positives |
| HyperLogLog | — | O(1) | O(1) | — | Cardinality estimate, ~1.6KB for billions |
| HNSW | — | O(log n) | O(log n) | difficult | Vector ANN, used by FAISS/Pinecone |

---

## How to Use This Folder

- **Sequential reading** if you're prepping interviews from scratch — go file 1 → 18.
- **Topic lookup** if you're refreshing — each file is self-contained with prerequisites linked at the top.
- **Mock-interview drill** — read the "Common Interview Gotchas" section in each file before any coding round.
- **System design crossover** — files 15–17 are the bridge to [DE system design](../data-engineer/15-system-design-de.md). Files 13 and 17 are the bridge to [AI/RAG](../ai/rag/01-rag-fundamentals.md).

---

## Sources & Foundational References

- *Introduction to Algorithms* (CLRS) — the bible for complexity proofs.
- *The Algorithm Design Manual* (Skiena) — practical, what-to-use-when.
- [CPython source code on GitHub](https://github.com/python/cpython) — read `Objects/listobject.c`, `Objects/dictobject.c`, `Objects/setobject.c` once in your life.
- [Python Time Complexity Wiki](https://wiki.python.org/moin/TimeComplexity) — official complexity for every built-in.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Kleppmann; the bridge between DSA and real systems.
- [Algorithms by Sedgewick & Wayne](https://algs4.cs.princeton.edu/) — free Princeton course materials.
- [The Morning Paper](https://blog.acolyer.org/) — readable summaries of academic papers (great for file 18).

---

*Start with [01-complexity-analysis.md](./01-complexity-analysis.md). Don't skip it — every other file assumes the vocabulary.*
