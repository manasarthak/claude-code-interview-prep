# Hash Tables — `dict`, `set`, `Counter`, and the Distributed Versions

**Phase:** 2 (Associative structures)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Python Internals](./02-python-internals.md) · [Complexity Analysis](./01-complexity-analysis.md) · [DE: Partitioning](../data-engineer/13-partitioning-performance.md) · [Probabilistic Structures](./15-probabilistic-structures.md)

---

## Why This File Exists

The hash table is **the** workhorse of computing — `dict` and `set` are the most-used data structures in Python, in databases ("hash join," "hash agg"), in distributed systems ("hash partitioning"), and in caches ("hash + LRU"). Knowing exactly how they work — load factor, perturbation, hash randomization, when O(1) degrades — is one of the highest-leverage interview topics.

This is the deepest CPython-internals file in the section.

---

## BEGINNER — The Core Idea

### What a Hash Table Is

A hash table maps keys to values in expected O(1). The trick:

1. Apply a **hash function** `h(key)` → integer.
2. Map that integer to a **bucket** in a fixed array: `index = h(key) % capacity`.
3. Store `(key, value)` in that bucket.

Lookups, inserts, and deletes hash the key and jump straight to the bucket.

### Why O(1) Average

If `h` distributes keys uniformly across `m` buckets and there are `n` keys, the expected bucket size is `n/m` (the load factor). As long as load factor is bounded (e.g. ≤ 2/3 for CPython), each operation touches O(1) buckets in expectation.

### Why O(n) Worst Case

If every key hashes to the same bucket — adversarial input or a broken hash function — operations degenerate to scanning a list of all keys.

### Collision Resolution: Two Schools

| Strategy | How |
|---|---|
| **Separate chaining** | Each bucket holds a linked list / vector of colliding entries. Java's `HashMap` does this. |
| **Open addressing** | Collisions are resolved by probing alternative slots in the same array. **CPython does this.** |

Open addressing has better cache locality (one contiguous array, no pointer chasing) and avoids per-entry allocations. Chaining is simpler and degrades more gracefully when load factor exceeds 1.

### Big-O Cheat Sheet

| Operation | Average | Worst | CPython `dict` |
|---|---|---|---|
| Lookup `d[k]` | O(1) | O(n) | O(1) avg, O(n) pathological |
| Insert `d[k] = v` | O(1) amortized | O(n) (resize) | Same |
| Delete `del d[k]` | O(1) | O(n) | Same |
| Iteration | O(n) | O(n) | Insertion-ordered since 3.7 |
| `k in d` | O(1) | O(n) | Same |

---

## INTERMEDIATE — How CPython's `dict` Actually Works

### The Compact Layout (since Python 3.6)

CPython uses two arrays:

```
indices[8]  =  [ -1,  2, -1,  0, -1,  1, -1, -1 ]   ← sparse hash table
entries[]   =  [ (h0, k0, v0), (h1, k1, v1), (h2, k2, v2) ]   ← dense, ordered
```

The hash maps to a slot in `indices`. The slot holds an integer index into `entries`, which is the dense, insertion-ordered array of triples. To look up a key:

1. Compute `h = hash(key)` and `i = h & (cap - 1)`.
2. Look at `indices[i]`:
   - `-1` (empty) → key not present.
   - Index `j` → check `entries[j]`. If hash and key match, return `entries[j].value`.
   - Hash mismatch → probe next slot (perturbation sequence).

This split-array design gives:

- **Lower memory** — `indices` is small ints (1, 2, 4, or 8 bytes depending on size); only the dense `entries` holds the full triples.
- **Insertion-ordered iteration** — walk `entries` in order. (3.7 made this a language guarantee.)
- **Better cache locality** for iteration.

### Hashing in Python

`hash(x)` returns a `Py_hash_t` (signed 64-bit on 64-bit systems). Built-in types implement reasonable hashers:

| Type | Hash strategy |
|---|---|
| `int` | Identity (`hash(42) == 42`), modulo a Mersenne prime for very large ints |
| `str`, `bytes` | SipHash24 with a randomized seed (security) |
| `tuple` | Combine element hashes with FNV-style mixing |
| `frozenset` | XOR of element hashes (order-independent) |
| Custom class | `id(self)` by default — i.e. memory address |

#### Hash Randomization (Critical for Web Servers)

`hash("foo")` returns a different value across Python processes (since 3.3). The seed is set at interpreter startup; you can override with `PYTHONHASHSEED=0` for determinism in tests.

**Why:** before randomization, an attacker could craft HTTP request bodies whose JSON keys all hashed to the same dict bucket, turning O(n) parsing into O(n²). Real DoS attacks were demonstrated against PHP, Ruby, and Python frameworks. Hash randomization fixed it.

**Implication:** never rely on `hash(x)` being stable across processes. If you need deterministic hashing (sharding, bloom filters, etc.), use `hashlib.sha256` or `xxhash`.

### Collision Resolution: Open Addressing with Perturbation

```c
// Simplified from CPython's dictobject.c
i = hash & mask
perturb = hash
while (slot is taken and key doesn't match):
    perturb >>= PERTURB_SHIFT  // 5
    i = (5*i + perturb + 1) & mask
```

The perturbation mixes in the high bits of the hash that the initial `& mask` discarded. As `perturb` shifts to zero, the probe sequence becomes purely linear (`5*i + 1`) — guaranteed to visit every slot before repeating, given a power-of-2 capacity.

**Why this matters:** simple linear probing causes "primary clustering" where colliding keys form long runs. Perturbation breaks the cluster early, then degrades to linear probing for the tail. Empirically, this beats both pure linear probing and pure double-hashing on real workloads.

### Load Factor and Resize

CPython resizes when the table is **2/3 full**. New size is the smallest power of 2 such that the new load factor is ≤ 2/3 (typically 2× the old size, or 4× if the dict has shrunk a lot).

Resize cost is O(n) — every entry is rehashed and reinserted. Amortized over n inserts, it averages to O(1).

### The `__hash__` and `__eq__` Contract

```
For any a, b:
    a == b   implies   hash(a) == hash(b)
```

If you define `__eq__` you **must** define a consistent `__hash__`, or the class becomes unhashable. Symmetrically:

- A mutable type can't be hashable in a meaningful way (its hash would change).
- This is why `list`, `dict`, and `set` are unhashable.
- `tuple` is hashable iff all elements are.
- `frozenset` is hashable; `set` is not.

```python
@dataclass(frozen=True)   # auto-generates __hash__ and __eq__
class Address:
    city: str
    zip: str
```

`frozen=True` makes the dataclass immutable and hashable in one go — the standard "lightweight record" pattern.

### `set` and `frozenset`

A `set` is essentially a `dict` without the value column. Same open-addressing internals.

```python
unique = set(stream)              # O(n)
common = set_a & set_b            # O(min(len(a), len(b)))
diff   = set_a - set_b            # O(len(a))
xor    = set_a ^ set_b            # O(len(a) + len(b))
```

Set operations are how SQL `INTERSECT` / `EXCEPT` / `UNION` work conceptually, and `set` membership is the right primitive for things like "is this user in the allowlist?" with O(1) lookup.

### `collections.Counter` — Multiset / Bag

`Counter` is a `dict` subclass mapping element → count.

```python
from collections import Counter
c = Counter("abracadabra")
c.most_common(3)   # [('a', 5), ('b', 2), ('r', 2)]
```

| Operation | Complexity |
|---|---|
| Build | O(n) |
| `c[k] += 1` | O(1) amortized |
| `c.most_common(k)` | O(n) when k is None, O(n log k) for top-k via heap |

`Counter`-style top-k is the textbook MapReduce word-count problem, the foundation of telemetry counting, and the entry point to [Count-Min Sketch](./15-probabilistic-structures.md) when you can't afford O(n) memory.

### `collections.defaultdict`

`defaultdict(factory)` calls `factory()` on missing-key access. Used to flatten the `if key not in d: d[key] = []` pattern.

```python
from collections import defaultdict
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
```

Internally, the missing-key path triggers the factory before raising; complexity is the same as `dict`.

---

## ADVANCED — Hashing Beyond a Single Process

### Hash Functions in Industry

| Algorithm | Use |
|---|---|
| **MurmurHash3** | Fast, non-cryptographic. Used in Cassandra, Hadoop, Lucene. |
| **xxHash / xxh3** | Even faster than Murmur, used in LZ4 / zstd / Iceberg. |
| **CityHash / FarmHash** | Google's fast non-crypto hashes. Used in BigQuery. |
| **SipHash** | Crypto-strength against collision DoS. CPython's `str` / `bytes` hash. |
| **SHA-256 / SHA-1** | Cryptographic. Used for content addressing (Git, IPFS, deduplication). |
| **BLAKE3** | Modern crypto hash, faster than SHA-256. Used in `b3sum`, IPFS. |

For DE: when you need a deterministic, cross-language, fast hash for **partitioning** or **bloom filters**, use Murmur or xxh3 (`mmh3` or `xxhash` Python packages). Don't use built-in `hash()` — randomized.

### Hash Partitioning in Distributed Systems

A central pattern in [partitioning](../data-engineer/13-partitioning-performance.md):

```
shard_id = hash(key) % num_shards
```

Used by:
- **Kafka** — `partition = murmur2(key) % num_partitions`. Keyed records always land in the same partition.
- **Cassandra** — `token = murmur3(partition_key)` mapped to a node in the ring.
- **Spark** — default `HashPartitioner(n)` uses `hash(key) % n`.
- **Postgres / MySQL hash partitioning** — same idea.
- **Redis Cluster** — `CRC16(key) % 16384` slot.

#### The `mod n` Problem and Consistent Hashing

If you add a shard, `hash(k) % n` changes for almost every key, requiring a massive reshuffle. **Consistent hashing** assigns each shard a range on a hash ring; adding a shard only redistributes keys whose hash falls in the new shard's slice — typically O(n / num_shards) keys, not O(n).

Used by Cassandra, DynamoDB, Memcached client libs, Akka Cluster, Riak. **Worth name-dropping in any "scale out a hash-partitioned system" interview question.**

#### Rendezvous Hashing (Highest-Random-Weight)

Alternative to consistent hashing: for each key, compute `h(key, server)` for every server and pick the max. Adding a server only steals keys whose new score wins. Used at scale by some CDNs and by Google's GFS-era systems.

### Hash Joins — Why DBs Love Hash Tables

A "hash join" between two tables R and S:

1. Build a hash table on the smaller table's join key.
2. Probe it with each row of the larger table.

Time: O(|R| + |S|) — beats nested loop's O(|R|·|S|) for non-trivial sizes. Spark's default join, Postgres's hash join, Snowflake / BigQuery / Redshift — all use this when stats say it's cheapest. See [Spark Fundamentals](../data-engineer/07-spark-fundamentals.md).

### Hash Aggregations

`SELECT k, SUM(v) FROM t GROUP BY k` is a hash-aggregate: build a hash table from key to running aggregate, scan once. O(n) time, O(distinct_keys) space.

When `distinct_keys` is small, this is fast and clean. When it's near `n`, you spill to disk — the row → partition → hash-aggregate-per-partition pattern in Spark and the warehouses.

### Hash-Based Set Operations and Bloom Filters

Set membership in distributed contexts is often "is this user in the deny list of 100M users?" Naively that's a 100M-entry hash set per worker — gigabytes of RAM.

A [Bloom filter](./15-probabilistic-structures.md) gives ~1.2 bytes per element with a tunable false-positive rate, often 10-100× smaller. Used by:
- HBase / Cassandra / RocksDB — skip SSTables that don't contain a key.
- BigQuery, ClickHouse — skip data blocks during scan.
- Cache layers — avoid useless DB lookups.

The Bloom filter exists *because* a real hash set would be too big.

### LRU Caches Are Hash Tables Plus a Linked List

Already covered in [Linked Lists](./04-linked-lists.md), but mentioned here because *the cache* is the hash table. Lookup is O(1) hash; the doubly linked list maintains usage order. Pattern is universal — `functools.lru_cache`, Postgres buffer pool (clock-sweep is similar), Memcached, Redis.

### Adversarial Hashing and SipHash

The reason CPython uses SipHash for `str` is that SipHash is keyed (the seed is the secret) and resistant to collision-finding. Without it, attackers could find input strings that all collide.

If you implement custom hashing in industry — for partitioning, sharding, deduplication — you must care whether the hash is **adversary-resistant** (SipHash, BLAKE3) or **just fast** (xxHash, Murmur). DDoS resistance vs throughput.

### Modern Hash Map Implementations

CPython is far from the state of the art. Worth name-dropping:

- **Google Abseil `flat_hash_map` / Swiss Tables** — open addressing with SIMD probe-group inspection. ~2× faster than `std::unordered_map`.
- **Facebook Folly `F14`** — similar SIMD-friendly design.
- **Robin Hood hashing** — open addressing variant that minimizes probe-distance variance.
- **Cuckoo hashing** — two hash functions; bound-worst-case lookup.

These don't change Big-O but cut constants by 2–5×. None of them are in CPython today; if you ever embed a high-throughput hash map in C/C++/Rust, you'll meet them.

---

## Common Interview Gotchas

**Q: Why is `dict` lookup O(1)?**
A: Hash the key, mod by table size, probe (with perturbation) for a match. Average O(1) assuming uniform hash; worst case O(n) on pathological collisions.

**Q: When does `dict` lookup become O(n)?**
A: When all keys hash to the same slot. Causes: bad `__hash__` (returning a constant), pre-3.3 string hashing exposed to adversarial input, custom types with broken hashes.

**Q: Are dicts ordered?**
A: Yes since 3.7 (language guarantee), insertion order. Was an implementation detail in 3.6.

**Q: What's the contract between `__eq__` and `__hash__`?**
A: `a == b` ⇒ `hash(a) == hash(b)`. Implementing only one corrupts hash-based collections.

**Q: Why is `set([1,2,3]) == set([3,2,1])` True but `[1,2,3] == [3,2,1]` False?**
A: Sets are unordered; lists are ordered. Set equality compares membership.

**Q: How would you partition 10B rows across 100 shards?**
A: `shard = hash(key) % 100`. If shards may grow/shrink, use consistent hashing or rendezvous hashing.

**Q: What's the cost of `set(big_list)`?**
A: O(n) average. Same hashing cost as building a dict.

**Q: How do you count distinct elements in a stream you can't fit in memory?**
A: HyperLogLog — O(1) memory regardless of cardinality, ~2% error. See [Probabilistic Structures](./15-probabilistic-structures.md).

**Q: How does `dict.update(other)` cost?**
A: O(len(other)) average — each key is hashed and inserted.

**Q: Why is mutable types unhashable?**
A: Because mutating an element would change its hash, breaking the dict's internal state — the entry would no longer be findable.

---

## Where This Bites You in Real DE/AI Code

| Symptom | Hash-table cause |
|---|---|
| Spark job stalls on one executor | Hot partition: skew on the join key. Salting fixes it. |
| Postgres GROUP BY 100× slower than expected | Hash agg spilling to disk because `work_mem` too small |
| Kafka consumer lag spikes for one partition | Producer keys cluster — bad hash distribution |
| Memory leak in long-running service | `dict` cache without eviction — implement LRU |
| Sets unexpectedly contain duplicates | Custom class without `__hash__`/`__eq__` consistency |
| `hash("foo")` differs across runs | Hash randomization — use `hashlib`/`xxhash` if you need stability |
| Bloom filter false-positive rate too high | Load factor too high; increase bits or use more hash functions |

---

## Interview-Ready Cheat Sheet

### One-liners to memorize

- **`dict` and `set` use open addressing with perturbation; O(1) average, O(n) worst.**
- **Compact layout (3.6+) → indices array + dense entries → ordered iteration since 3.7.**
- **Hash randomization since 3.3 prevents collision DoS.**
- **`__eq__` and `__hash__` must be consistent or your dict breaks.**
- **`list.pop(0)` is O(n); for queues use `deque`.**
- **For partitioning across machines, use Murmur/xxh3, not built-in `hash()`.**
- **Consistent hashing fixes the "add a shard, reshuffle everything" problem.**

### Quick Trade-off Pairs

- **Open addressing vs chaining:** cache locality vs simplicity.
- **`dict` vs `set`:** key→value vs key-only.
- **`Counter` vs manual `dict.get(k, 0) + 1`:** ergonomic vs explicit; same performance.
- **Hash join vs merge join:** O(n+m) extra memory vs O(1) extra memory if both sorted.
- **Real hash set vs Bloom filter:** exact + O(n) memory vs probabilistic + O(1) memory.
- **`hash()` vs `hashlib`/`xxhash`:** fast non-stable vs stable cross-process.

---

## Resources & Links

- [CPython `dictobject.c`](https://github.com/python/cpython/blob/main/Objects/dictobject.c) — read once; the comments are excellent.
- [CPython `setobject.c`](https://github.com/python/cpython/blob/main/Objects/setobject.c)
- [Raymond Hettinger — Modern Python Dictionaries](https://www.youtube.com/watch?v=p33CVV29OG8)
- [SipHash paper (Aumasson & Bernstein)](https://www.aumasson.jp/siphash/siphash.pdf)
- [Consistent Hashing — Karger et al. 1997](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
- [Google Swiss Tables](https://abseil.io/about/design/swisstables)
- [xxHash](https://github.com/Cyan4973/xxHash)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) — modern crypto hash.

### Companion Files
- [Python Internals](./02-python-internals.md) — `dict` and `set` walkthrough.
- [Probabilistic Structures](./15-probabilistic-structures.md) — Bloom, HLL, Count-Min.
- [DE: Partitioning & Performance](../data-engineer/13-partitioning-performance.md) — hash partitioning at scale.
- [DE: Spark Fundamentals](../data-engineer/07-spark-fundamentals.md) — hash joins and aggregations.

---

*Next: [Trees — BST and Balanced BSTs](./07-bst-and-balanced-trees.md).*
