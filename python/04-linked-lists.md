# Linked Lists — When You Need Pointers (And Why You Mostly Don't in Python)

**Phase:** 1 (Linear structures)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Arrays & Buffers](./03-arrays-and-buffers.md) · [Stacks/Queues/Deques](./05-stacks-queues-deques.md) · [Python Internals](./02-python-internals.md)

---

## Why This File Exists

Linked lists are the "Hello World" of data structures and the most common low-level interview warm-up — reverse a list, detect a cycle, merge two sorted lists, find the kth from end. You'll be asked at least one of these.

The honest truth in 2026 Python: **you almost never use a linked list in production.** Cache locality kills it for any real workload, and `list` / `deque` cover the use cases. But the interview tradition is real, and several DE-relevant structures (skip lists, LRU caches, deque internals, jemalloc free lists) *are* linked lists in disguise.

---

## BEGINNER — Concept and Variants

### What a Linked List Is

A linked list is a sequence of nodes, each holding a value and one or two pointers to other nodes:

```
Singly:    [A]──▶[B]──▶[C]──▶[D]──▶ None

Doubly:    None ◀──[A]◀──▶[B]◀──▶[C]◀──▶[D]──▶ None

Circular:  ┌──▶[A]──▶[B]──▶[C]──▶[D]──┐
           └────────────────────────────┘
```

**Compared to an array:**

| | Array | Linked list |
|---|---|---|
| Memory layout | Contiguous | Scattered (one alloc per node) |
| Random access `[i]` | O(1) | O(n) |
| Insert at known position | O(n) | O(1) |
| Insert at head | O(n) | O(1) |
| Memory overhead per element | ~8 bytes (pointer) | ~32 bytes (node + pointers + alloc header) |
| Cache locality | Excellent | Terrible |

**The real-world verdict:** the cache locality of arrays beats linked lists by 5–50× in practice for any workload that touches many elements. That's why CPython's `list` is an array and `deque` is *block-based* (hybrid: linked list of small arrays).

### Big-O Cheat Sheet

| Operation | Singly | Doubly |
|---|---|---|
| Access by index | O(n) | O(n) |
| Search by value | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail (with tail pointer) | O(1) | O(1) |
| Insert at middle (given node) | O(1) | O(1) |
| Delete at head | O(1) | O(1) |
| Delete given node | O(n) (need prev) or O(1) if doubly | O(1) |

### A Minimal Python Singly Linked List

```python
class Node:
    __slots__ = ('value', 'next')   # save memory; see Python Internals
    def __init__(self, value, nxt=None):
        self.value = value
        self.next = nxt

class LinkedList:
    def __init__(self):
        self.head = None

    def push_front(self, x):
        self.head = Node(x, self.head)

    def __iter__(self):
        cur = self.head
        while cur is not None:
            yield cur.value
            cur = cur.next
```

### The Sentinel / Dummy Node Trick

Many linked-list bugs come from special-casing the head. A sentinel node (a permanent dummy at the start) eliminates the special case:

```python
class LinkedList:
    def __init__(self):
        self.sentinel = Node(None)   # dummy head, never holds data
        self.tail = self.sentinel

    def append(self, x):
        node = Node(x)
        self.tail.next = node
        self.tail = node
```

Now `head` is always `sentinel.next`, never None, and inserts/deletes don't need a "is this the head?" branch. **Always reach for sentinels in interview problems involving deletion** — it's the single biggest source of off-by-one bugs.

---

## INTERMEDIATE — Classic Interview Patterns

### 1. Reverse a Linked List (Iterative)

```python
def reverse(head):
    prev = None
    cur = head
    while cur:
        nxt = cur.next       # save next
        cur.next = prev      # reverse pointer
        prev = cur           # advance prev
        cur = nxt            # advance cur
    return prev              # new head
```

O(n) time, O(1) space. Recursive version is also O(n) but uses O(n) stack.

### 2. Detect a Cycle (Floyd's Tortoise and Hare)

Two pointers, one at 1× speed, one at 2×. If there's a cycle, they meet.

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

O(n) time, O(1) space — vs. O(n) space if you used a `set` of seen nodes. **The senior signal here is preferring O(1) space.**

To find the *start* of the cycle: after collision, reset one pointer to head and advance both at 1×. They meet at the cycle start.

### 3. Find the Middle Node

Same two-pointer trick: slow advances by 1, fast by 2. When fast hits the end, slow is at the middle. O(n) time, O(1) space.

### 4. Merge Two Sorted Lists

Standard merge step from mergesort, applied to lists. O(n + m) time, O(1) extra space.

```python
def merge_sorted(a, b):
    dummy = Node(None)
    tail = dummy
    while a and b:
        if a.value <= b.value:
            tail.next, a = a, a.next
        else:
            tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next
```

### 5. Remove the Nth From End

Two pointers, one starts n nodes ahead. When the leader hits None, the follower is at the node to remove. Single pass; O(n) time, O(1) space.

### 6. Reverse in K-Groups, Rotate, Partition…

All variants of the above. The pattern is always: **two or three pointers, careful next-saves before re-pointing.**

---

## ADVANCED — Where Linked Lists Actually Live in Real Systems

### Where Linked Lists *Are* Used (despite cache woes)

- **`collections.deque`** — block-based doubly linked list (each block is a small array of ~64 elements, blocks are linked). Best of both worlds.
- **LRU caches** — a doubly linked list of usage order + a hash map for O(1) lookup. `functools.lru_cache` does exactly this in C.
- **Memory allocators** — `malloc` free lists, jemalloc / tcmalloc bin lists.
- **Skip lists** — Redis sorted sets (ZSET), LevelDB MemTable. See [Storage Engine Structures](./16-storage-engine-structures.md).
- **Filesystem inode chains** — old-school but still common.
- **Compiler/interpreter linked structures** — AST node sibling lists.
- **Postgres internal lists (`List` type)** — actually a "list of lists" in C, used pervasively in the planner.

The pattern: **wherever you need O(1) splice / removal of a known node, linked lists win.** Where you need iteration speed or random access, arrays win.

### LRU Cache — The Canonical Application

A classic interview problem and a real production primitive:

```python
class LRUCache:
    """O(1) get and put for a fixed-capacity cache."""

    class _Node:
        __slots__ = ('key', 'value', 'prev', 'next')
        def __init__(self, key, value):
            self.key, self.value = key, value
            self.prev = self.next = None

    def __init__(self, capacity):
        self.cap = capacity
        self.map = {}                  # key -> node
        self.head = self._Node(0, 0)   # sentinel
        self.tail = self._Node(0, 0)   # sentinel
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key):
        if key not in self.map:
            return None
        node = self.map[key]
        self._remove(node); self._add_to_front(node)
        return node.value

    def put(self, key, value):
        if key in self.map:
            self._remove(self.map[key])
        node = self._Node(key, value)
        self._add_to_front(node)
        self.map[key] = node
        if len(self.map) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.map[lru.key]
```

This pattern (hash + doubly linked list) is the foundation of:
- `functools.lru_cache`
- Database buffer-pool replacement (Postgres's `clock-sweep` is similar in spirit)
- Memcached, Redis (configurable LRU/LFU policies)
- CDN edge caches

### Skip Lists — Probabilistic Sorted Linked Lists

A skip list is a stack of linked lists where each upper level skips over more nodes. Lookup walks from top-left, descending when needed:

```
L3:  ──▶ 1 ───────────────▶ 9 ──▶ ∞
L2:  ──▶ 1 ──▶ 4 ─────────▶ 9 ──▶ ∞
L1:  ──▶ 1 ──▶ 4 ──▶ 6 ──▶ 9 ──▶ ∞
L0:  ──▶ 1 ──▶ 4 ──▶ 6 ──▶ 7 ──▶ 9 ──▶ ∞
```

Each insertion picks a "level" by flipping coins (P(level k) = (1/p)^k for some p, often 2). Operations:

| Operation | Expected complexity |
|---|---|
| Search | O(log n) |
| Insert | O(log n) |
| Delete | O(log n) |

**Why skip lists matter in DE:**
- **Redis ZSET (sorted set)** — implemented as skip list + hash map.
- **LevelDB / RocksDB MemTable** — newest writes go into a skip list before being flushed to SSTables. See [LSM Trees](./16-storage-engine-structures.md).
- **Apache HBase MemStore** — same idea.
- **Lucene** — uses skip lists in posting list iteration for fast `nextDoc(target)`.

Skip lists rival balanced BSTs but with simpler concurrent implementations (lock-free skip lists are well-known), which is why they win in databases.

### Why You Don't Use Linked Lists for Iteration in Python

Pure-Python linked-list iteration is **slow**: every `cur = cur.next` is a Python bytecode op + attribute lookup + heap-allocated `Node` indirection. A `list` of the same data iterates in pure C with cache-friendly access.

```
1M-element traversal:
  list:           ~10ms
  deque:          ~25ms
  custom linked list: ~250ms
  numpy array (sum): ~2ms
```

So in interviews, **never propose a linked list as a Python production data structure unless the problem explicitly requires its O(1) splice/insert properties.** Use `list` or `deque`.

### Doubly Linked Lists in `collections.deque`

CPython's `deque` is a doubly linked list of "blocks" — small fixed-size arrays of ~64 PyObject pointers. This gives:

- O(1) push/pop at both ends (just like a linked list)
- Way better cache locality than a node-per-element list
- Less memory overhead

Use `deque` for any FIFO/LIFO/sliding window in production. Roll your own linked list only when interviewing or implementing a specialized algorithm.

---

## Common Interview Gotchas

**Q: How do you reverse a singly linked list in O(1) extra space?**
A: Iterative three-pointer (prev / cur / next).

**Q: How do you detect a cycle without using extra space?**
A: Floyd's tortoise-and-hare. (A `set` of seen nodes is O(n) space — works but not the senior answer.)

**Q: Why doesn't Python's stdlib have a linked list class?**
A: Because `list` is an array (better cache locality) and `deque` is a block-list (better than a pure linked list for almost every use case). Pure linked lists rarely win in Python. *(Note: the `llist` package on PyPI exists if you really need one.)*

**Q: When *would* you use a linked list?**
A: When you need O(1) splice/insertion at known positions in the middle of a sequence — e.g. an LRU cache, an order book, an event scheduler. Or as a teaching device.

**Q: What's the time/space complexity of recursive linked-list reversal?**
A: O(n) time, O(n) stack space. Iterative is preferable in Python because the recursion limit defaults to ~1000.

**Q: What's the difference between a linked list and a deque?**
A: Both support O(1) ends. Deque adds O(1) length, better cache locality (block layout), and is implemented in C — making it ~10× faster in practice.

**Q: How does `random.choice(linked_list)` work?**
A: It can't be O(1) — you'd need O(n) to pick an index. Hence: don't use linked lists for random sampling.

---

## Interview-Ready Cheat Sheet

### Two-pointer patterns to memorize

| Problem | Pattern |
|---|---|
| Cycle detection | slow + 2× fast |
| Find middle | slow + 2× fast (slow lands at middle) |
| Find Kth from end | leader starts K ahead |
| Find cycle start | after collision, restart one at head, advance both 1× |
| Merge sorted lists | two pointers + dummy head |
| Reverse | three pointers (prev, cur, next) |
| Detect intersection | walk both, then swap heads when one hits null |

### Quick Trade-off Pairs

- **Array vs linked list:** cache locality vs O(1) splicing.
- **Singly vs doubly:** memory vs ability to walk backward / O(1) deletion of given node.
- **Linked list vs `deque`:** flexibility vs production performance.
- **Skip list vs balanced BST:** simpler concurrent ops vs deterministic balance.

### Don't Get Tripped Up By

- Edge cases: empty list, single node, head is the answer.
- Always pre-save `cur.next` before re-pointing in reversal.
- Use sentinels to eliminate "is this the head?" branches.
- The Python recursion limit is ~1000 — recursive linked-list code dies on long inputs.

---

## Resources & Links

### Foundational
- *CLRS* ch. 10 — pointer-based structures.
- *Cracking the Coding Interview* — best collection of linked-list practice problems.
- [Floyd's Cycle Detection (Wikipedia)](https://en.wikipedia.org/wiki/Cycle_detection)
- [Skip Lists: A Probabilistic Alternative to Balanced Trees (Pugh, 1990)](https://www.cs.umd.edu/~pugh/skiplist.pdf) — the original paper.

### CPython / Real Implementations
- [`collections.deque` source](https://github.com/python/cpython/blob/main/Modules/_collectionsmodule.c) — block-based doubly linked list.
- [`functools.lru_cache` source](https://github.com/python/cpython/blob/main/Modules/_functoolsmodule.c) — hash map + doubly linked list.
- [Redis ZSET implementation](https://github.com/redis/redis/blob/unstable/src/t_zset.c) — skip list.
- [LevelDB MemTable](https://github.com/google/leveldb/blob/main/db/skiplist.h) — skip list.

### Companion Files
- [Stacks/Queues/Deques](./05-stacks-queues-deques.md) — `deque` is the practical successor.
- [Storage Engine Structures](./16-storage-engine-structures.md) — skip lists in databases.
- [Python Internals](./02-python-internals.md) — why linked lists underperform in Python.

---

*Next: [Stacks, Queues & Deques](./05-stacks-queues-deques.md) — the ADTs you'll actually use.*
