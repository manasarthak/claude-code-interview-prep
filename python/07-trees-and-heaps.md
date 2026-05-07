# Trees & Heaps — BSTs, Balanced Trees, and Priority Queues

**Phase:** 2 (Hierarchical structures)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Complexity Analysis](./01-complexity-analysis.md) · [Hash Tables](./06-hash-tables.md) · [Storage-Engine Structures](./13-storage-and-vector-structures.md)

---

## Why This File Exists

Trees are the structure of choice when you need **sorted access**. Hash tables give O(1) lookup but no ordering. Trees give O(log n) lookup *plus* range queries, predecessor/successor, and ordered iteration.

Heaps are a special tree (almost always implemented on top of an array) that exposes only one operation interestingly fast: O(1) min/max. They power priority queues — schedulers, top-k streaming, Dijkstra's algorithm, A\*, k-way merge in databases.

In Python, both look different than in CS textbooks: `heapq` is a function set on a list, and there is no balanced-BST in the stdlib (you reach for `sortedcontainers`).

---

## BEGINNER — Binary Search Trees

### What a BST Is

A binary tree where every node's left subtree is < node and right subtree is > node:

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

| Operation | Average | Worst (degenerate) |
|---|---|---|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| In-order traversal | O(n) | O(n) |

The worst case happens when you insert sorted data into an unbalanced BST — it degrades into a linked list.

### Tree Traversals (must know)

- **Pre-order** (node, left, right) — used to clone or serialize.
- **In-order** (left, node, right) — yields sorted output for a BST.
- **Post-order** (left, right, node) — used to free / compute aggregates.
- **Level-order / BFS** — level-by-level via a queue.

```python
def inorder(node):
    if not node: return
    yield from inorder(node.left)
    yield node.value
    yield from inorder(node.right)
```

For deep trees (>1000 levels), use an explicit stack to avoid Python's recursion limit.

### Why "Search Tree" Beats a Hash Table Sometimes

Hash tables: O(1) lookup, no ordering.
Search trees: O(log n) lookup + ordered iteration + range queries + min/max in O(log n).

```sql
-- Range query: hash table can't help, BST can.
SELECT * FROM events WHERE ts BETWEEN '2026-04-01' AND '2026-04-30';
```

This is why every relational database **index** is some kind of tree (almost always a B+ tree on disk — see [Storage-Engine Structures](./13-storage-and-vector-structures.md)).

---

## INTERMEDIATE — Balanced BSTs

### The Need for Balance

Vanilla BSTs degenerate to linked lists on sorted input. Balanced BSTs use rotations to keep depth at O(log n) regardless of insertion order.

### AVL Trees

For every node, |height(left) − height(right)| ≤ 1. After insert/delete, rotate to restore. Tighter balance → faster lookups, more rotations on writes. Used when reads vastly outnumber writes.

### Red-Black Trees

Looser balance: longest path is at most 2× shortest. Fewer rotations on writes. Used by:
- Java `TreeMap` / `TreeSet`
- C++ `std::map` / `std::set`
- Linux kernel CFS scheduler
- Linux `epoll` interest list
- Java `ConcurrentSkipListMap` (it's actually a skip list, but the analog)

### B-Trees and B+ Trees (the disk-friendly cousins)

A B-tree node holds many keys (often hundreds, sized to a disk page or SSD block). Depth is much shallower than a binary tree → fewer disk seeks.

**B+ tree** keeps all values at the leaves and links them, enabling O(1) sequential scan after a point lookup. Used by:
- Postgres, MySQL/InnoDB, Oracle, SQL Server (default index)
- SQLite
- File systems (NTFS, HFS+, APFS, ext4 directories)

This is the single most important "industry tree" for DE. Covered in depth in [Storage-Engine Structures](./13-storage-and-vector-structures.md).

### Why Python's Stdlib Has No Balanced BST

CPython only ships:
- `list` (array)
- `dict` / `set` (hash)
- `heapq` (binary heap)
- `collections.deque` (block-list)

For a sorted container, reach for the third-party [`sortedcontainers`](http://www.grantjenks.com/docs/sortedcontainers/) package. It implements `SortedList`, `SortedDict`, `SortedSet` using a clever **list-of-lists** data structure (not a true balanced tree but with similar performance characteristics, often faster in practice in pure Python).

```python
from sortedcontainers import SortedList
sl = SortedList()
sl.add(5); sl.add(2); sl.add(8)
sl[0]               # 2 (sorted access)
sl.bisect_left(4)   # 1 (binary search)
```

Operations are O(log n) average (with surprisingly small constants thanks to cache-friendly chunks).

### Tries

A trie (prefix tree) keys on **strings**, branching on each character. Operations are O(k) where k is the key length, independent of n.

| Operation | Complexity |
|---|---|
| Insert | O(k) |
| Search | O(k) |
| Delete | O(k) |
| Prefix search | O(k + matches) |

Used in:
- Autocomplete (Google search box, IDE intellisense).
- Routing tables (IP address longest-prefix match → radix tree variant).
- BPE / WordPiece tokenizers in LLMs (the tokenizer is essentially a giant trie of byte sequences). See [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md).
- Spell checkers.

Detailed in [Tries, Strings & Union-Find](./11-tries-strings-union-find.md).

---

## ADVANCED — Heaps & Priority Queues

### Binary Heap

A complete binary tree where every node ≤ its children (min-heap) or ≥ (max-heap).

Stored as an **array** (not nodes + pointers) using the indexing trick:

```
Index 0 is root.
Children of i are at 2i+1 and 2i+2.
Parent of i is at (i-1)//2.
```

```
Array: [1, 3, 5, 4, 7, 9, 8]

Tree:           1
              /   \
             3     5
            / \   / \
           4   7 9   8
```

| Operation | Complexity |
|---|---|
| `peek` (root) | O(1) |
| `push` | O(log n) — sift up |
| `pop` | O(log n) — sift down |
| `heapify` (array → heap) | O(n) — Floyd's bottom-up |
| `replace` (pop + push) | O(log n) one operation |
| Find arbitrary value | O(n) |
| Delete arbitrary value | O(n) (find) + O(log n) (sift) |

`heapify` being O(n) is non-obvious — most people guess O(n log n). Floyd's algorithm bottom-up only sifts each subtree once; the geometric series sums to O(n).

### `heapq` — Python's Functional Heap

```python
import heapq

h = [3, 1, 4, 1, 5, 9, 2]
heapq.heapify(h)         # O(n) in place

heapq.heappush(h, 0)     # O(log n)
heapq.heappop(h)         # 0 — the min
heapq.heappushpop(h, x)  # one op, push then pop the smallest
heapq.heapreplace(h, x)  # pop the smallest then push x

heapq.nlargest(5, iterable)   # O(n log 5) = O(n)
heapq.nsmallest(5, iterable)  # same
heapq.merge(*iterables)       # k-way merge, O(N log k) total
```

**Min-heap only.** For max-heap, push `-x` (or wrap in a class with reversed `__lt__`).

For priority queue with stable ordering, push tuples `(priority, counter, item)` so ties break by insertion order (counter is a monotonic int):

```python
import heapq, itertools
counter = itertools.count()
pq = []
heapq.heappush(pq, (priority, next(counter), task))
```

### Top-K Streaming Pattern

Find the k largest elements in a stream of n items, k ≪ n.

```python
import heapq

def top_k(stream, k):
    heap = []
    for x in stream:
        if len(heap) < k:
            heapq.heappush(heap, x)
        elif x > heap[0]:
            heapq.heapreplace(heap, x)
    return sorted(heap, reverse=True)
```

Time: O(n log k). Space: O(k). **Beats sorting (O(n log n) and O(n) space) when k ≪ n.**

This is the canonical pattern for:
- "Top 100 trending hashtags in the last hour"
- "Largest 10 customers by revenue today"
- Top-K log lines by latency
- Spark's `DataFrame.orderBy().limit(k)` planner picks this when k is small.

### k-Way Merge — The External Sort Building Block

Given k sorted streams, merge into one sorted stream:

```python
import heapq
result = list(heapq.merge(stream_a, stream_b, stream_c))
```

`heapq.merge` is a generator — uses O(k) memory regardless of total stream length.

This is the core of:
- **External merge sort** — sort 1TB on a 16GB machine: split into runs, sort each in memory, k-way merge. Used by every database, Spark, Pandas (when out-of-core), Hadoop.
- **LSM tree compaction** — merge sorted SSTables into a larger sorted SSTable. See [Storage-Engine Structures](./13-storage-and-vector-structures.md).
- **Log aggregation across machines.**

### Priority Queues in Algorithms

| Algorithm | Why it needs a PQ |
|---|---|
| Dijkstra's shortest path | Always extract the closest unvisited node |
| A\* search | Extract by `f = g + h` |
| Prim's MST | Extract the cheapest crossing edge |
| Huffman coding | Extract two smallest frequencies |
| Event-driven simulation | Extract the next-earliest event |
| OS schedulers | Pick the highest-priority runnable task |
| Network packet schedulers | Weighted fair queueing |
| Top-K | Maintain a bounded heap |

### Indexed Priority Queue (when you need decrease-key)

Standard `heapq` lacks an O(log n) `decrease-key` operation (the move you need in Dijkstra to update a node's distance). You need to also map item → heap-index. Either implement it yourself or use a library like `pqdict`.

In practice for Python Dijkstra: just push `(new_dist, node)` again and skip stale entries when popping. The heap grows but the algorithm stays correct, and it's simpler:

```python
import heapq

def dijkstra(graph, src):
    dist = {src: 0}
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue   # stale entry
        for v, w in graph[u]:
            nd = d + w
            if nd < dist.get(v, float('inf')):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
```

### Fibonacci Heap, Pairing Heap (theory > practice)

Fibonacci heaps achieve O(1) amortized `decrease-key`, making Dijkstra O(E + V log V). In practice, constant factors are huge — almost no production system uses them. **Pairing heap** is the practical choice when you need fast decrease-key in C/C++.

### Heap Sort

Build a heap (O(n)) then repeatedly pop (O(n log n)). In-place if you sort in-place.

```python
import heapq
def heapsort(arr):
    heapq.heapify(arr)
    return [heapq.heappop(arr) for _ in range(len(arr))]
```

Heap sort is O(n log n) worst case (vs. quicksort's O(n²) worst case) but slower in practice due to cache-unfriendly access patterns. Used as a fallback in **introsort** (quicksort that switches to heap sort when recursion depth gets too high) — what C++ `std::sort` does.

---

## Where Trees and Heaps Live in Real DE/AI Systems

| System | Tree/heap usage |
|---|---|
| Postgres / MySQL indexes | B+ tree |
| Cassandra / RocksDB / LevelDB | LSM tree (skip-list memtables, sorted SSTables) |
| Redis ZSET (sorted set) | Skip list + hash map |
| Linux process scheduler (CFS) | Red-black tree |
| `epoll` | Red-black tree |
| Java `TreeMap`, C++ `std::map` | Red-black tree |
| `heapq` / Python schedulers | Binary heap |
| Spark / Snowflake / BigQuery sort | External merge sort with k-way merge |
| Kafka log compaction | k-way merge of segments |
| FAISS HNSW vector index | Hierarchical navigable graph (graph, not tree, but uses heaps internally) |
| Trino top-N operator | Bounded heap |
| Iceberg / Delta metadata snapshots | Forest of partition stats (range trees in spirit) |

---

## Common Interview Gotchas

**Q: What's the worst-case complexity of BST search?**
A: O(n) for an unbalanced BST (insertions in sorted order). Balanced (AVL, Red-Black, B-tree) is O(log n).

**Q: Why does Python use a binary heap and not a Fibonacci heap?**
A: Constant-factor simplicity. Binary heap fits in a contiguous list, cache-friendly, ~5–10× faster in practice. The asymptotic win of Fibonacci heap rarely shows up in real code.

**Q: How do you implement a max-heap with `heapq`?**
A: Push `(-priority, item)` and negate on pop. Or wrap items in a class that flips `__lt__`.

**Q: What's `heapify`'s complexity?**
A: O(n) — Floyd's bottom-up. Common gotcha — most candidates guess O(n log n).

**Q: How do you find the median of a stream?**
A: Two heaps — a max-heap of the lower half, a min-heap of the upper half. Median is `top of max-heap` (odd) or `(max-heap top + min-heap top) / 2` (even). Each insert is O(log n).

**Q: Why do databases use B+ trees instead of red-black trees?**
A: B+ tree nodes match disk page size — one node fetch reads many keys. Red-black trees do one comparison per cache miss. On disk, this is a 100–1000× difference.

**Q: How would you sort 1TB on a 16GB machine?**
A: External merge sort. Read in 12GB chunks, sort each in memory, write to disk. Then k-way merge using a heap to combine. This is what every database does.

**Q: Why doesn't Python's `heapq` support decrease-key?**
A: Index maintenance would slow every other op. Workaround: push duplicates and lazily skip stale entries on pop.

---

## Interview-Ready Cheat Sheet

### Quick mappings

| Goal | Right structure |
|---|---|
| Sorted iteration / range query | Balanced BST or `sortedcontainers.SortedList` |
| O(1) min or max | Heap (`heapq`) |
| Top-K from stream | Bounded heap |
| Median of stream | Two heaps |
| k-way merge | `heapq.merge` |
| Disk-resident ordered map | B+ tree (built into the database) |
| String prefix search | Trie |
| Concurrent ordered map (Java) | `ConcurrentSkipListMap` |
| Range query on time-series | B+ tree or skip list (in time-series DBs) |

### Trade-off pairs

- **Hash vs tree:** O(1) point lookup vs O(log n) but with ordering.
- **AVL vs Red-Black:** read-heavy vs write-heavy.
- **Binary heap vs Fibonacci heap:** simplicity + cache locality vs theoretical O(1) decrease-key.
- **Heap sort vs Timsort:** worst-case O(n log n) vs better real-world constants.
- **In-memory sort vs external merge sort:** RAM-bounded vs disk-bounded.

---

## Resources & Links

- *CLRS* ch. 6 (heaps), ch. 12–14 (BSTs and red-black trees), ch. 18 (B-trees).
- [Floyd's heapify analysis](https://en.wikipedia.org/wiki/Heap_(data_structure)#Building_a_heap)
- [Python `heapq` docs](https://docs.python.org/3/library/heapq.html)
- [`sortedcontainers` performance](http://www.grantjenks.com/docs/sortedcontainers/performance.html) — surprisingly fast in pure Python.
- [Designing Data-Intensive Applications, ch. 3](https://dataintensive.net/) — B-trees vs LSM in databases.
- [Skip List paper (Pugh 1990)](https://www.cs.umd.edu/~pugh/skiplist.pdf)

### Companion Files
- [Hash Tables](./06-hash-tables.md) — the unordered counterpart.
- [Sorting](./08-sorting.md) — heap sort and Timsort.
- [Graph Algorithms](./10-graphs-and-algorithms.md) — Dijkstra, Prim, A\* all need PQs.
- [Storage-Engine Structures](./13-storage-and-vector-structures.md) — B+, LSM, skip list in databases.

---

*Next: [Sorting](./08-sorting.md) — the canonical playground for complexity analysis.*
