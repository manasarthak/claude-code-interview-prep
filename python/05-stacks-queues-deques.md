# Stacks, Queues, and Deques — The ADTs You Use Every Day

**Phase:** 1 (Linear ADTs)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Arrays & Buffers](./03-arrays-and-buffers.md) · [Linked Lists](./04-linked-lists.md) · [Python Internals](./02-python-internals.md) · [Graph Algorithms](./12-graph-algorithms.md)

---

## Why This File Exists

Stacks, queues, and deques are **abstract data types (ADTs)** — interfaces, not implementations. They show up constantly:

- Stacks: function call stack, undo/redo, parser balanced-paren checks, DFS, expression evaluation.
- Queues: BFS, task schedulers, message brokers, request queues, rate limiters.
- Deques: sliding-window problems, double-ended buffers, work stealing.

What separates seniors from juniors here is knowing **why `list.pop(0)` is the wrong way to implement a queue in Python** and being able to defend the right choice.

---

## BEGINNER — The ADTs

### Stack — LIFO (Last In, First Out)

```
push(A) → push(B) → push(C)        pop() → pop() → pop()
   ┌─┐    ┌─┐    ┌─┐                  C       B       A
   │A│    │B│    │C│
   └─┘    └─┘    └─┘
   │A│    │B│
   └─┘    └─┘
   │A│
   └─┘
```

| Operation | Complexity (any reasonable impl) |
|---|---|
| push | O(1) |
| pop | O(1) |
| peek (top) | O(1) |
| size | O(1) |

**In Python: just use a `list`.** `lst.append(x)` and `lst.pop()` are both O(1) amortized. There is no separate stack class because `list` already is one.

### Queue — FIFO (First In, First Out)

```
enqueue(A) → enqueue(B) → enqueue(C)        dequeue() → dequeue() → dequeue()
            head ──▶ A ─▶ B ─▶ C ◀── tail        A         B         C
```

| Operation | Complexity (right impl) |
|---|---|
| enqueue (push to back) | O(1) |
| dequeue (pop from front) | O(1) |
| peek (front) | O(1) |
| size | O(1) |

**In Python: use `collections.deque`.** `lst.pop(0)` is O(n) — every element shifts left. This is the single most common Python performance mistake in BFS code.

### Deque — Double-Ended Queue

A queue you can push or pop from either end. Operations are O(1) at both ends. Python's `collections.deque` is the canonical implementation.

```
appendleft(A) appendleft(B) appendleft(C)   ┌─C─B─A─┐    append(D)    ┌─C─B─A─D─┐
```

---

## INTERMEDIATE — The Right Implementation in Python

### Stack: Just Use a `list`

```python
stack = []
stack.append(x)   # push
top = stack[-1]   # peek
x = stack.pop()   # pop
```

That's it. The moment you start thinking "do I need a Stack class?" — no, you don't.

### Queue: `collections.deque`, NOT `list.pop(0)`

```python
from collections import deque

q = deque()
q.append(x)        # enqueue (right)
x = q.popleft()    # dequeue (left)  ← O(1)
front = q[0]       # peek (left)
```

#### Why `list.pop(0)` is O(n)

`list` is a contiguous C array. Removing `list[0]` requires shifting every other element one step left to keep the array contiguous. That's O(n) per pop. Doing this in a loop turns BFS into O(n²).

#### How `deque` is implemented

`collections.deque` is a doubly linked list of fixed-size blocks (~64 elements per block). `appendleft` and `popleft` adjust pointers in the leftmost block, allocating a new block if needed. Both ends are O(1).

You can also pass `deque(maxlen=N)` to make it a **bounded ring buffer** — when full, adding to one end discards from the other. This is the cleanest way to implement a sliding window.

### Thread-Safe Queues: `queue.Queue`

For producer-consumer concurrency, the stdlib has `queue.Queue` (FIFO), `queue.LifoQueue` (stack), and `queue.PriorityQueue` (heap). They're slower than `deque` because of locking, but they're thread-safe and support `.get(block=True, timeout=...)`.

| Use case | Use |
|---|---|
| Single-thread BFS / FIFO | `collections.deque` |
| Producer/consumer threads | `queue.Queue` |
| Producer/consumer processes | `multiprocessing.Queue` |
| asyncio coroutines | `asyncio.Queue` |

### Bounded Queues — Backpressure

A bounded queue refuses (or blocks on) `put()` when full. This is the **fundamental backpressure mechanism** in concurrent and distributed systems:

- Kafka producers block when the broker buffer is full.
- AWS SQS supports visibility timeouts and DLQs but doesn't bound producer-side; you bound at the producer.
- Spark Structured Streaming has a `maxOffsetsPerTrigger` knob — implicit bounded queue.
- Airflow `pools` cap concurrency by bounded slot reservations. See [Airflow](../data-engineer/05-orchestration-airflow.md).

**An unbounded queue in production is a bug.** It's a memory leak waiting to happen when consumers fall behind.

---

## ADVANCED — Where These Show Up in Real DE/AI Systems

### Stacks in Practice

| Use case | Why a stack |
|---|---|
| Function call stack | Each call pushes a frame; return pops |
| Recursion → iterative conversion | Explicit stack replaces call stack |
| DFS (graph traversal) | LIFO traversal order |
| Undo/redo (Photoshop, IDEs) | Push state on each action; pop to undo |
| Expression evaluation, parsing | Shunting yard, operator precedence |
| HTML/XML tag balancing | Push open tags, pop on close |
| Browser back button | Push pages on visit |
| Spark RDD lineage / Catalyst optimizer | Plan tree traversal uses stacks |
| Database transaction nesting (savepoints) | Stack of savepoints |

### Queues in Practice

| Use case | Why a queue |
|---|---|
| BFS (graph traversal) | FIFO ensures level-by-level visit |
| Task schedulers, job queues | Fair processing order (FIFO + priority) |
| Print/work queues | First come, first served |
| Producer-consumer pipelines | Decouple rate of production and consumption |
| Message brokers (Kafka, Kinesis, RabbitMQ) | Durable distributed FIFO with replay |
| Web servers' request queues | Bound concurrency; reject when full |
| Rate limiters (token bucket) | Tokens replenish at fixed rate |
| Async event loops | `asyncio` ready-queue of coroutines |

### Deques in Practice

| Use case | Why a deque |
|---|---|
| **Sliding window max/min** (the canonical interview problem) | Monotonic deque keeps candidates ordered |
| Lock-free work-stealing schedulers | Owner pushes/pops one end, thieves the other (Cilk, Java ForkJoinPool) |
| MRU + LRU caches | Move-to-front and evict-from-back |
| Browser history with back+forward | Both ends |
| Sliding window aggregations in [stream processing](../data-engineer/06-batch-vs-streaming.md) | `deque(maxlen=N)` |

### Worked Example — Monotonic Deque (Sliding Window Max)

The "sliding window max" problem: given an array and window size k, return the max in every window. Naive solution is O(n·k); the deque approach is O(n).

```python
from collections import deque

def sliding_max(arr, k):
    out = []
    dq = deque()  # stores indices, values monotonically decreasing
    for i, x in enumerate(arr):
        # drop indices outside window
        while dq and dq[0] <= i - k:
            dq.popleft()
        # drop smaller values from the back — they can never be max again
        while dq and arr[dq[-1]] < x:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            out.append(arr[dq[0]])
    return out
```

Each index is pushed and popped at most once → **amortized O(1) per element, O(n) total.** This pattern recurs for any "min/max in a window" problem and shows up in:
- Streaming analytics (last-N-minute peak load).
- Trading systems (rolling max bid in a window).
- Real-time anomaly detection.

### BFS in Python — The Correct Template

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    q = deque([start])
    while q:
        node = q.popleft()         # ← O(1), critical
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                q.append(neighbor)
```

If you write `q.pop(0)` you've turned an O(V+E) algorithm into O(V·(V+E)). On a million-node graph that's the difference between a second and a day. **This is one of the most common rejections in DE interviews involving graphs.**

### DFS in Python — Two Templates

**Recursive (most natural, but careful with depth):**

```python
def dfs(graph, node, visited):
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```

Python's default recursion limit is 1000. For deeper graphs (like 10⁵-node airflow DAGs), you'll blow the stack — switch to iterative.

**Iterative with explicit stack:**

```python
def dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()         # ← O(1), correct stack op
        if node in visited:
            continue
        visited.add(node)
        for neighbor in graph[node]:
            stack.append(neighbor)
```

### Priority Queues — Heap-Backed

A priority queue is a queue where the highest-priority (or lowest, depending) item dequeues first. It's not implemented as a queue at all — see [Heaps](./08-heaps-and-priority-queues.md).

`heapq` for single-thread, `queue.PriorityQueue` for thread-safe.

### Backpressure & Bounded Concurrency Patterns

In any high-throughput pipeline, **bounded queues + bounded worker pools** is the right pattern. Anti-pattern: launching one worker per item.

```python
import asyncio

async def worker(queue, sem):
    while True:
        item = await queue.get()
        if item is None: break
        async with sem:
            await process(item)
        queue.task_done()

async def main(items, num_workers=10, max_concurrency=20):
    queue = asyncio.Queue(maxsize=100)  # bound to limit memory
    sem = asyncio.Semaphore(max_concurrency)
    workers = [asyncio.create_task(worker(queue, sem)) for _ in range(num_workers)]
    for item in items:
        await queue.put(item)
    await queue.join()
    for _ in workers:
        await queue.put(None)
    await asyncio.gather(*workers)
```

This pattern is the foundation of every async data ingestion service — it bounds memory, gives backpressure, and lets you scale workers independently.

---

## Common Interview Gotchas

**Q: How do you implement a queue with two stacks?**
A: One stack for `enqueue`, one for `dequeue`. When `dequeue` is empty, pop everything from `enqueue` and push to `dequeue` (reversing order). Amortized O(1) per op.

**Q: How do you implement a stack with two queues?**
A: Cute but useless — O(n) push or O(n) pop. Real stacks aren't built like this.

**Q: What's the complexity of `list.pop(0)`?**
A: O(n). Use `collections.deque.popleft()` for O(1).

**Q: Why is `collections.deque` faster than a pure linked list in Python?**
A: It's block-based — each block is a small array, so iteration enjoys cache locality. The linked-list aspect is only between blocks, not between every element.

**Q: What's the difference between `queue.Queue` and `collections.deque`?**
A: `queue.Queue` is thread-safe (acquires a lock). `deque` is faster but not thread-safe for both-end operations. (Single appends/pops happen to be atomic in CPython, but don't rely on it.)

**Q: When does a queue become a priority queue?**
A: When dequeuing is by priority, not insertion order. Implement with a heap, not a list.

**Q: Why use `asyncio.Queue` vs `queue.Queue` in async code?**
A: `queue.Queue` blocks the OS thread; `asyncio.Queue` blocks the coroutine, freeing the event loop. Mixing the two deadlocks.

---

## Interview-Ready Cheat Sheet

### Memorize these mappings

| Need | Use |
|---|---|
| Stack | `list` (`append`, `pop`) |
| Queue (single-thread) | `collections.deque` (`append`, `popleft`) |
| Deque | `collections.deque` |
| Sliding window | `collections.deque(maxlen=N)` |
| Priority queue | `heapq` |
| Thread-safe FIFO | `queue.Queue` |
| Process-safe FIFO | `multiprocessing.Queue` |
| Async FIFO | `asyncio.Queue` |
| Bounded buffer with backpressure | bounded `Queue` (any flavor) |

### Quick Trade-off Pairs

- **`list.pop(0)` vs `deque.popleft()`:** O(n) vs O(1). Always use deque for FIFO.
- **`deque` vs `queue.Queue`:** speed vs thread safety.
- **Recursive vs iterative DFS:** clean vs deep-graph-safe.
- **Bounded vs unbounded queue:** backpressure vs unbounded memory risk.
- **Stack vs queue for traversal:** DFS vs BFS — different semantics, same data structure shape.

### Top 6 Q&As

**"Implement BFS in Python."** → `from collections import deque` + `popleft`. Anyone using `pop(0)` fails the senior bar.

**"Implement DFS without recursion."** → explicit `list` as stack with `append`/`pop`.

**"Implement a sliding window max."** → monotonic deque, O(n).

**"Implement a queue with O(1) push/pop on both ends."** → `collections.deque`.

**"How would you bound the memory of a producer-consumer pipeline?"** → bounded queue with backpressure when producer outpaces consumer.

**"Why would you ever pick a stack over a queue?"** → DFS vs BFS, undo/redo, expression eval, recursion conversion.

---

## Resources & Links

- [Python `collections.deque` docs](https://docs.python.org/3/library/collections.html#collections.deque)
- [CPython `_collectionsmodule.c`](https://github.com/python/cpython/blob/main/Modules/_collectionsmodule.c)
- [`queue` module docs](https://docs.python.org/3/library/queue.html)
- [`asyncio.Queue` docs](https://docs.python.org/3/library/asyncio-queue.html)
- [Cilk's work-stealing deque (Blumofe & Leiserson)](https://supertech.csail.mit.edu/papers/abp.pdf) — classic deque-in-systems paper.

### Companion Files
- [Linked Lists](./04-linked-lists.md) — what `deque` is built on.
- [Arrays & Buffers](./03-arrays-and-buffers.md) — ring buffers as bounded queues.
- [Heaps & Priority Queues](./08-heaps-and-priority-queues.md) — the priority-queue ADT.
- [Graph Algorithms](./12-graph-algorithms.md) — where BFS/DFS live.

---

*Next: [Hash Tables](./06-hash-tables.md) — the most-asked-about Python internal.*
