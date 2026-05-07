# Probabilistic Data Structures — Bloom, HyperLogLog, Count-Min Sketch

**Phase:** 5 (Differentiator)
**Difficulty progression:** Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Hash Tables](./06-hash-tables.md) · [DE: Streaming](../data-engineer/06-batch-vs-streaming.md) · [DE: Partitioning](../data-engineer/13-partitioning-performance.md)

---

## Why This File Exists

Real DE doesn't deal in "give me an exact set." It deals in:
- "Is this user *probably* in the deny list of 100M users?" (Bloom filter)
- "Roughly how many distinct values has this stream seen?" (HyperLogLog)
- "Roughly how often has each value appeared?" (Count-Min Sketch)
- "Top-k frequent items in a stream?" (Heavy Hitters)

These are **probabilistic** structures: they trade tiny, controlled error for **massive** memory savings. They are the secret sauce inside BigQuery's `APPROX_*` functions, Cassandra's read path, Redis's HLL commands, ClickHouse's `uniq()`, and every streaming analytics system.

Almost no candidate brings them up unprompted. Bringing one up correctly in a system design round is a senior signal.

---

## BLOOM FILTER

### What It Does

Membership test with **no false negatives, controlled false positives**. "Is X in the set?" Yes (always correct) / Maybe (with bounded error).

### How It Works

- Bit array of m bits, all zero initially.
- k independent hash functions h₁ ... h_k.
- Insert(x): set bits at positions h₁(x), h₂(x), ..., h_k(x).
- Query(x): if all k bits are set, "maybe present"; if any is zero, "definitely absent."

### Implementation Sketch

```python
import mmh3   # MurmurHash3
from bitarray import bitarray

class BloomFilter:
    def __init__(self, m_bits, k_hashes):
        self.bits = bitarray(m_bits); self.bits.setall(0)
        self.m = m_bits; self.k = k_hashes

    def add(self, item):
        for i in range(self.k):
            self.bits[mmh3.hash(item, i) % self.m] = 1

    def __contains__(self, item):
        return all(self.bits[mmh3.hash(item, i) % self.m]
                   for i in range(self.k))
```

### Sizing

For false-positive rate `p` and `n` items:
```
m = -n · ln(p) / (ln 2)²        ≈ 1.44 · n · log₂(1/p) bits
k = (m / n) · ln 2               ≈ 0.7 · m / n
```

Practical examples (n = 1B):
- p = 1% → m ≈ 9.6 Gb (1.2 GB)
- p = 0.1% → m ≈ 14.4 Gb (1.8 GB)
- p = 0.01% → m ≈ 19.2 Gb (2.4 GB)

A real `set` of 1B 64-bit ids would take 8 GB just for values plus dict overhead — **Bloom is 5–10× smaller**.

### Where Bloom Filters Live in Industry

| System | Use |
|---|---|
| **Cassandra / HBase / RocksDB / LevelDB** | Skip SSTables that don't contain a key. Saves 90%+ of unnecessary disk reads. |
| **BigQuery, Snowflake, Iceberg, Parquet** | Skip data blocks during scan; metadata Bloom filters (sometimes called "min-max + bloom" predicates). |
| **Chrome / Safari** | Malicious URL pre-check before contacting Google's database. |
| **CDN / cache layers** | "Is this URL in our cache?" before doing the expensive lookup. |
| **Distributed dedup** | Streaming pipelines that need to filter duplicates without keeping all keys. |
| **Bitcoin / Ethereum (SPV clients)** | Filter relevant transactions without downloading every block. |

### Variants

- **Counting Bloom** — slots are counters, supports `delete`. ~4× more space.
- **Cuckoo filter** — uses cuckoo hashing; supports delete; better space-error trade-off than Bloom for low FPR. Used in modern Iceberg manifests, Apache Doris.
- **Quotient filter** — cache-friendly variant.
- **Stable Bloom** — designed for streams where you can tolerate forgetting old items.

### Big-O

| Op | Complexity |
|---|---|
| Insert | O(k) |
| Query | O(k) |
| Space | ~1.44·n·log₂(1/p) bits |
| Delete | Not supported (use Counting Bloom or Cuckoo) |

---

## HYPERLOGLOG

### What It Does

Estimate **cardinality** (distinct count) of a multiset using **O(1) memory**. Not "approximately O(n)" — actually constant.

### How It Works (intuition)

If you flip a fair coin until you get heads, the expected number of trailing zeros in a random hash is correlated with how many distinct values you've seen. HyperLogLog:

1. Hash each value.
2. Track the **maximum number of leading zeros** seen in a register, across many registers (split by hash prefix).
3. Combine via harmonic mean to estimate cardinality.

### Memory and Accuracy

A standard HLL uses 12 KB and gives **~2% standard error for any cardinality up to ~10¹⁸**. To put that in perspective:

> Counting distinct values among 1 billion users with HLL costs ~12KB.
> A real `set` would cost ~50GB.

### API in Practice

```python
# Using datasketches-python
from datasketches import hll_sketch
sketch = hll_sketch(12)            # 2^12 = 4096 buckets
for x in stream:
    sketch.update(x)
print(sketch.get_estimate())       # ~ exact distinct count, ±2%
```

### Where HLL Lives

| System | API |
|---|---|
| **BigQuery** | `APPROX_COUNT_DISTINCT(col)` |
| **Redshift** | `APPROXIMATE COUNT(DISTINCT col)` |
| **Snowflake** | `APPROX_COUNT_DISTINCT(col)` |
| **Athena / Presto / Trino** | `approx_distinct(col)` |
| **ClickHouse** | `uniqHLL12(col)` |
| **Spark** | `approxCountDistinct(col)` (Spark uses HLL+) |
| **Redis** | `PFADD`, `PFCOUNT`, `PFMERGE` commands |
| **Druid, Pinot** | Native HLL columns |

**Mergeability** is HLL's killer feature: HLL(A ∪ B) = `merge(HLL(A), HLL(B))`. So you can compute per-shard, per-day, per-segment HLLs and combine — distributed cardinality estimation is trivial.

### Variants

- **HLL++** — Google's improvement; corrects HLL's bias for low cardinalities. Used by Spark, BigQuery.
- **Theta sketches** — generalize HLL to support arbitrary set operations (intersect, difference) with bounded error. Used by Apache Druid.
- **KMV (k-Minimum Values)** — simpler alternative for small cardinalities.

---

## COUNT-MIN SKETCH

### What It Does

Estimate the **frequency** of each item in a stream using O(1) memory per item — answers "how many times has X appeared?" with bounded over-estimate error.

### How It Works

- 2D array of counters with d rows × w columns.
- d independent hash functions, one per row.
- Increment(x): for each row i, increment cell `[i, h_i(x) mod w]`.
- Query(x): return `min(cell[i, h_i(x) mod w] for i in 0..d)`.

The minimum across rows is robust against collisions; that's the "min" in the name.

### Where Count-Min Lives

- **Top-K / Heavy Hitters** — combine Count-Min with a heap of top-k.
- **Network anomaly detection** — count IPs in a packet stream.
- **Recommender systems** — approximate frequency signals.
- **DDoS detection** — count requests per source.
- **Streaming analytics** — Apache Flink, Spark Structured Streaming.

### Variants

- **Conservative update CMS** — only increment the minimum cell. Tighter over-estimate bound.
- **Count-Mean-Min Sketch** — bias-corrected.
- **Heavy Hitters / Top-K via Count-Min + heap** — Apache Druid's "topN" uses this family.

---

## SPACE-SAVING / TOP-K (HEAVY HITTERS)

### What It Does

Maintain a fixed set of k counters; if you see a new key and counters are full, replace the smallest counter's key with the new one and increment.

### Guarantees

For k counters, finds all items with frequency > N/k with overcount bounded by N/k (where N is stream size).

### Used For

- "Top 10 trending hashtags right now."
- "Most common error code in last hour."
- DDoS source tracking.
- Apache Druid's `groupBy + topN` (combined with Theta/HLL).

---

## HYPERLOGLOG vs BLOOM vs COUNT-MIN — Picking the Right Tool

| Question | Tool |
|---|---|
| "Is X in the set?" | **Bloom** (or Cuckoo) |
| "How many distinct values?" | **HyperLogLog** |
| "How often did X appear?" | **Count-Min Sketch** |
| "What are the top-k items?" | **Space-Saving / Count-Min + heap** |
| "Approximate set intersection?" | **Theta sketch** |

### When NOT to use a probabilistic structure

- When the dataset already fits in memory and exact count is fast.
- When you can't tolerate any false positives (e.g. authoritative deny lists).
- When you need exact deletes (use Counting Bloom or just a hash set).
- When the input is small (<10⁵) — overhead doesn't pay off.

---

## Worked Example — Streaming Dedup at Scale

Naive: keep `seen = set()`. Memory grows linearly with distinct events. At 1B events, that's ~50GB and the process OOMs.

Bloom-filter version:

```python
import mmh3
from bitarray import bitarray

class StreamDedup:
    def __init__(self, expected_n=1_000_000_000, fpr=0.001):
        import math
        self.m = int(-expected_n * math.log(fpr) / (math.log(2) ** 2))
        self.k = int(self.m / expected_n * math.log(2))
        self.bits = bitarray(self.m); self.bits.setall(0)

    def is_new(self, key):
        idxs = [mmh3.hash(key, seed=i) % self.m for i in range(self.k)]
        if all(self.bits[i] for i in idxs):
            return False                # probably duplicate
        for i in idxs: self.bits[i] = 1
        return True
```

For 1B unique events at 0.1% FPR, this uses ~1.8GB instead of ~50GB. **The 1 in 1000 false positives is acceptable for most "dedup" use cases** (e.g. counting "unique sessions" — slightly under-counted is fine). For exact dedup, you need a real set or a database.

---

## Common Interview Gotchas

**Q: What's the false-negative rate of a Bloom filter?**
A: Zero by construction. False positives only.

**Q: Can you delete from a Bloom filter?**
A: No. Use a Counting Bloom (slots are counters, supports decrement) or a Cuckoo filter.

**Q: Why is HyperLogLog only ~2% error regardless of cardinality?**
A: Statistical magic — the maximum-leading-zero estimator's standard error is 1.04/√m where m is the bucket count. Doubling m halves error.

**Q: How does Spark's `approxCountDistinct` work?**
A: HLL++ under the hood. Mergeable across partitions, so it parallelizes trivially.

**Q: When would you choose a Bloom filter over a hash set?**
A: When the set is too large to fit in memory and false positives are tolerable. Classic example: Cassandra's per-SSTable Bloom filter.

**Q: What's a Cuckoo filter?**
A: A cuckoo-hashing-based alternative to Bloom that supports deletion and has better space-error trade-off for low false-positive rates.

**Q: How do you find the top 100 trending hashtags in a stream of 100B tweets?**
A: Space-Saving / Count-Min Sketch + a heap. ~MB of memory, 99% accurate.

**Q: Are these structures "machine learning"?**
A: No — they're randomized, but their guarantees are deterministic in expectation/probability. Distinct from ML which learns from data.

---

## Interview-Ready Cheat Sheet

### Memory comparison (1B distinct items)

| Structure | Memory | Operation |
|---|---|---|
| Hash set | ~50 GB | Exact membership |
| Bloom filter (1% FPR) | ~1.2 GB | Approximate membership |
| HyperLogLog (12KB total!) | 12 KB | Approximate cardinality |
| Count-Min Sketch (size depends on width × depth) | ~MB-GB tuneable | Approximate frequency |

### Trade-off pairs

- **Hash set vs Bloom:** exact + memory hungry vs probabilistic + tiny.
- **Bloom vs Cuckoo:** simple + add-only vs delete-supporting + better FPR.
- **Count vs HLL:** exact frequency vs constant-memory cardinality.
- **Sample-based vs sketch-based:** simpler vs better statistical guarantees.

### Top Q&As

**"How would you check if an email is in our 100M-entry blocklist with low memory?"** → Bloom filter; warn that confirmed-positives need a real DB lookup.

**"How many unique users hit our service today?"** → HyperLogLog; mergeable across servers/days.

**"Top 10 most-viewed pages last hour from a 1B-event stream?"** → Space-Saving or Count-Min + heap.

**"Why does Cassandra read-amplify less than expected?"** → Bloom filters skip SSTables that definitely don't have the key.

**"How do you join two HLL sketches from different shards?"** → Bitwise merge of registers; the math is built into HLL.

---

## Resources & Links

### Foundational Papers
- [Burton Bloom — Space/time trade-offs in hash coding (1970)](https://dl.acm.org/doi/10.1145/362686.362692) — original Bloom paper.
- [Flajolet et al. — HyperLogLog (2007)](https://stefanheule.com/papers/edbt2013-hyperloglog.pdf)
- [Heule et al. — HyperLogLog++ (2013)](https://research.google/pubs/pub40671/) — Google's improvement.
- [Cormode & Muthukrishnan — Count-Min Sketch (2005)](https://www.cs.rutgers.edu/~muthu/countmin.pdf)
- [Cuckoo Filter (Fan et al. 2014)](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)

### Production Libraries
- [`bloom-filter2`](https://github.com/hiway/bloom-filter2) — Python.
- [`pybloom-live`](https://github.com/joseph-fox/python-bloomfilter)
- [Apache DataSketches (Python bindings)](https://datasketches.apache.org/) — production-grade HLL, Theta, CMS.
- [`mmh3`](https://github.com/hajimes/mmh3) — Murmur3 for Python.
- [Redis Probabilistic module](https://redis.io/docs/data-types/probabilistic/) — Bloom, Cuckoo, CMS, Top-K, t-digest.

### Companion Files
- [Hash Tables](./06-hash-tables.md) — exact alternatives.
- [Storage & Vector Search](./13-storage-and-vector-structures.md) — Bloom in Cassandra/RocksDB, HNSW for ANN.
- [DE: Streaming](../data-engineer/06-batch-vs-streaming.md) — sketches in stream processing.

---

*Next: [Storage & Vector Search Structures](./13-storage-and-vector-structures.md).*
