# Modern Advances — What's New in 2024–2026 You Should Name-Drop

**Phase:** 5 (Differentiator)
**Difficulty progression:** Advanced
**Last updated:** April 24, 2026
**Related:** [Graphs & Algorithms](./10-graphs-and-algorithms.md) · [Storage & Vector Search](./13-storage-and-vector-structures.md) · [Hash Tables](./06-hash-tables.md)

---

## Why This File Exists

Most CS curricula stop at CLRS (1990s) and the Sedgewick textbook (2010). The field has moved on. A senior interview round becomes memorable when you say things like *"there's a 2025 Duan et al. paper that beats Dijkstra's 60-year-old bound,"* or *"we're moving from B+ trees to learned indexes for some workloads."*

This file is your **name-drop arsenal** — short, accurate, and dated. None of these will be a "do you know this?" question; mentioning them when relevant is pure upside.

---

## BMSSP — The First Sub-Dijkstra SSSP Algorithm in 60 Years (Duan et al. 2025)

### What

Single-Source Shortest Paths in directed graphs with non-negative real weights, in **O(m · log^(2/3) n)** time on the comparison-addition model.

### Why It Matters

Dijkstra's classical bound is O(m + n log n). For dense graphs (m = Θ(n²)), Dijkstra is O(n² + n log n) = Θ(n²). For sparse graphs Dijkstra is O((m+n) log n).

The BMSSP bound `O(m · log^(2/3) n)` beats Dijkstra **for sparse graphs** when `m = O(n · log^(1/3) n)` — which covers most real graphs. It also breaks the long-believed "sorting barrier" (the implicit Ω(m + n log n) lower bound that came from sort-by-distance reasoning).

### How (Very Briefly)

The algorithm cleverly avoids the global priority queue Dijkstra relies on. Instead it:
1. Recursively partitions the graph into pieces with bounded "boundary."
2. Uses a relaxed-priority structure that doesn't enforce strict ordering globally.
3. Combines per-piece distance arrays.

Implementation is non-trivial — currently theoretical, but research follow-ups are working toward practical variants.

### When To Mention

Any system-design question involving routing, shortest paths, or network analysis. **"Modern advance worth flagging: Duan et al. 2025 beats Dijkstra asymptotically for sparse directed graphs — though Dijkstra is still the practical choice in 2026 because the constants haven't been worked out yet."**

[Paper: arXiv:2504.17033](https://arxiv.org/abs/2504.17033) (April 2025).

---

## Learned Indexes (Kraska et al. 2018+, mainstream by 2024–2025)

### What

Replace a B+ tree with an ML model that predicts the position of a key in a sorted array: `position ≈ f(key)`. After prediction, do a small local search to find the exact key.

### Why It Matters

A linear interpolation is O(1). A neural net might be O(d) for d parameters but with cache-friendly weights. Compared to B+ tree's log_B(n) cache misses, learned indexes can be 1.5–4× faster for read-heavy workloads, and use less memory.

### Where

- **Google Bigtable / Plasma** — variants in production for some workloads.
- **ALEX** (Microsoft) — adaptive learned B+ tree replacement that supports inserts.
- **RadixSpline, PGM-Index** — research with strong worst-case bounds.
- **DuckDB** — has experimented with learned scan-skip.
- A running thread of papers since SIGMOD 2018.

### When To Mention

Any DE system-design that touches indexes — particularly time-series or log databases where keys are roughly sorted. **"Learned indexes are an emerging alternative for read-heavy workloads — if our key distribution is predictable, an interpolation-search learned index can beat B+ trees by 2-3×."**

[Original paper: Kraska et al., 2018](https://arxiv.org/abs/1712.01208).

---

## Modern Hash Maps — Swiss Tables and F14

### What

Open-addressed hash maps that use SIMD instructions to inspect 16 hash slots at once during probing. Originated at Google (Abseil's `flat_hash_map`) and Facebook (Folly's `F14`).

### Why It Matters

CPython's `dict` and Java's `HashMap` walk slots one at a time. SIMD-aware tables are typically 1.5–4× faster on real workloads with the same Big-O guarantee — a constant-factor improvement that matters at scale.

### Where

- Google Abseil (used in Bigtable, Spanner internals).
- Facebook Folly F14.
- Rust's `hashbrown` (which is `std::HashMap` since 2018).
- ClickHouse's hash join uses Swiss-style tables.
- Not in CPython yet — but the per-process improvement is huge in the C/C++/Rust DBs you query.

### When To Mention

System design where hash maps are bottlenecks (joins, distinct counts, in-memory caches). **"At ClickHouse's scale they use SIMD-friendly Swiss tables — 2-3× faster than std::unordered_map; same Big-O, much better constants."**

---

## Bw-Tree — Lock-Free B+ Tree

### What

A B+ tree where updates append delta records to a logical page chain instead of mutating in place. A separate mapping table redirects logical → physical addresses, enabling lock-free concurrency.

### Where

- Microsoft Hekaton (in-memory SQL Server).
- CockroachDB experiments.
- Some HTAP systems combining row + column stores.

### When To Mention

When designing a high-concurrency database engine. **"For many-core OLTP, Bw-trees scale better than locked B+ trees — used by Hekaton."**

---

## RUM Conjecture (Athanassoulis et al., 2016)

### What

Index designs face a fundamental three-way trade-off between **R**ead amplification, **U**pdate amplification, and **M**emory amplification. You can be excellent on at most two.

| Structure | Read | Update | Memory |
|---|---|---|---|
| B+ tree | Excellent | OK | OK |
| LSM | OK | Excellent | OK |
| Hash | Excellent | Excellent | Bad |
| Sorted file | Excellent | Bad | Excellent |

### When To Mention

Any "read-heavy vs write-heavy" indexing decision. **"This is the RUM conjecture in action — we can't get all three; we're trading update for read."**

---

## Roaring Bitmaps

### What

A compressed bitmap format that picks the best representation per chunk: array, run-length, or raw bitmap.

### Where

- Apache Druid, Pinot — segment indexes.
- ClickHouse — bitmap aggregates.
- Lucene — term-id bitmaps.
- Pilosa, FeatureBase — full bitmap databases.
- Redis bitmaps (`BITCOUNT`, `BITOP`).

Bitmap operations (AND, OR, XOR, COUNT) on roaring bitmaps run at ~1 billion bits/sec on a single core.

### When To Mention

Any "user segments / cohort analysis" question. **"For high-cardinality membership joins, roaring bitmaps run AND/OR on a billion-element set in <1 second."**

---

## SIMD JSON / Serialization Acceleration

### What

`simdjson` parses JSON at ~3 GB/s on commodity hardware using SIMD instructions to find structural characters in parallel. Rewrites the assumption that JSON parsing is slow.

### Where

- Used by Snowflake's variant column.
- ClickHouse's JSON ingest.
- Drop-in faster Python via `pysimdjson`.

### When To Mention

Any "we have lots of JSON" pipeline. **"For event ingestion, simdjson moves parsing from CPU-bound to I/O-bound."**

---

## Apache Arrow + DataFusion + Velox

### What

The "vectorized engine" pattern is taking over: query operators process **column batches** in tight CPU loops, exploiting SIMD and cache.

- **Apache Arrow** — in-memory columnar format.
- **Apache DataFusion** — Rust SQL engine on Arrow.
- **Velox (Meta)** — C++ vectorized engine, used by Presto and Spark Gluten.
- **DuckDB** — vectorized engine with sub-second analytics on a laptop.

### When To Mention

When designing analytics infra. **"The 2024+ pattern is vectorized engines on Arrow batches — DuckDB / Velox / DataFusion all share this design and beat row-at-a-time engines by 5-10×."**

---

## GPU Graph Algorithms — RAPIDS cuGraph

### What

Run BFS, PageRank, connected components, shortest paths on a GPU using NVIDIA's cuGraph (part of RAPIDS).

### Why

A 4090 GPU can run BFS on a 1B-edge graph in seconds — orders of magnitude faster than single-CPU NetworkX or even multi-machine GraphX for graphs that fit in GPU memory.

### When To Mention

Large-scale graph analytics or fraud detection. **"For graphs that fit in GPU memory (up to ~10B edges on H100), cuGraph beats Spark GraphX by 10-100×."**

---

## Mojo, Cython 3, JAX (Python Performance Frontiers)

- **Mojo** (Modular AI, public 2023) — superset-of-Python language compiling to MLIR, designed for ML perf.
- **Cython 3** — better Python interop, C++ generics.
- **JAX** — auto-differentiable NumPy with XLA compilation; foundation of many ML frameworks.
- **PyPy** — long-running JIT-compiled Python; 5-10× speedups on pure-Python workloads.

### When To Mention

Performance-bound Python pipelines that can't trivially escape to NumPy. **"For pure-Python hot paths, the modern fix is Mojo or Cython 3 rather than 'rewrite in Rust' — better dev velocity, 10-50× faster than CPython."**

---

## Free-Threaded Python (PEP 703, 3.13+)

### What

CPython 3.13 introduced an opt-in **GIL-free build** (`python3.13t`). Removes the Global Interpreter Lock entirely.

### Why

Python finally supports true parallelism in pure-Python code. Single-threaded perf takes ~10-15% hit; multi-core scales near-linearly.

### When To Mention

CPU-bound Python workloads: feature engineering, JSON parsing, image preprocessing. **"With Python 3.13+ free-threaded, you no longer need multiprocessing for parallel Python — and process-overhead disappears."**

---

## Vector Databases — Beyond HNSW

- **DiskANN** — Microsoft's billion-scale ANN (covered in [13-storage-and-vector-structures.md](./13-storage-and-vector-structures.md)).
- **ScaNN (Google)** — uses anisotropic vector quantization; powers Vertex AI.
- **SOAR** — improves HNSW recall with secondary lists.
- **Multi-vector retrieval** (ColBERT-style) — store one vector per token instead of per document; rerank by late interaction. Boosts retrieval accuracy 5–10% in [RAG](../ai/rag/01-rag-fundamentals.md).
- **Hybrid sparse-dense** (SPLADE, BM25 + dense) — best practice in production RAG today.

### When To Mention

Any RAG system design. **"For maximal retrieval quality, hybrid SPLADE + dense + ColBERT-rerank now beats single-vector dense retrieval by 5-10% on benchmarks."**

---

## LSM Variants and Modern Storage Engines

- **Tiered vs leveled compaction** — RocksDB tunables that swap write amp for read amp.
- **Universal compaction** (Cassandra) — keeps few sorted runs; good for time-series.
- **Differential dataflow** (Materialize) — incremental view maintenance using a totally different formalism.
- **Bw-tree** (above).
- **Spanner / TigerBeetle** — bespoke LSMs with novel features (LWW, durable log replicas, bounded staleness).

---

## Other Name-Drops Worth Knowing

| Term | One-liner |
|---|---|
| **t-digest** (Dunning, 2014) | Approximate quantiles in O(log n) memory. Used in Apache Druid, Snowflake, Redis. |
| **Hyperloglog++ (HLL+)** | Google's bias-corrected HLL — what Spark and BigQuery use. |
| **Theta sketches** | Sketches that support intersect/diff (HLL can't). Apache DataSketches. |
| **HDR Histogram** | Lossless latency histograms for SLO tracking. |
| **Robin Hood hashing** | Open addressing variant minimizing probe-distance variance. |
| **Cuckoo hashing** | Two-hash variant with O(1) worst-case lookup. |
| **Skip graphs** | Distributed skip lists for P2P. |
| **Cache-oblivious algorithms** (Frigo et al., 1999) | Work well at every cache level without tuning. |
| **Functional / persistent data structures** | Used by Clojure, Datomic, Git's object store. |
| **Bloom + Trie hybrids** | Some pre-2010 IDS / packet filters. |
| **xor filters** (Graf & Lemire, 2020) | Smaller than Bloom for the same FPR. Used in Cassandra 4.x. |
| **Ribbon filters** (RocksDB, 2021) | Smaller than xor filters for very low FPRs. |

---

## Common Interview Gotchas

**Q: Has anyone improved Dijkstra in modern times?**
A: Yes — Duan et al. 2025 broke the sorting barrier with O(m · log^(2/3) n) for directed SSSP with non-negative weights. Theoretical so far; Dijkstra still wins in practice.

**Q: Can ML replace classical data structures?**
A: For some read-heavy workloads — see Kraska et al. 2018 on learned indexes. Not a wholesale replacement; a complementary tool.

**Q: Why hasn't Python adopted Swiss tables?**
A: CPython's `dict` is already extremely optimized in C and balances flexibility (`PyObject*` keys/values) with speed. Swiss tables shine for trivially-comparable types (ints, strings) — Python's flexibility tax makes the win smaller.

**Q: Will the GIL ever go away?**
A: PEP 703 (3.13+) introduced an opt-in GIL-free build. Wide adoption in libraries is happening through 2026; the default may flip in 3.14 or later.

**Q: What's the modern equivalent of 'use a Bloom filter'?**
A: Often **xor filters** or **ribbon filters** — same idea, smaller, slightly different API.

---

## How to Use Modern Advances in Interviews

The trick is to **mention them in passing** rather than as the centerpiece — sprinkle them into trade-off discussions:

> *"For SSSP I'd implement Dijkstra in production today, even though Duan et al. 2025 has an asymptotically better algorithm — its constants aren't worked out yet."*

> *"For a write-heavy KV store, LSM. Modern variants like RocksDB use Bloom filters and increasingly xor filters to keep read amp low."*

> *"Vector DB choice: I'd start with pgvector + HNSW. At billion scale we'd need DiskANN or sharded HNSW."*

> *"For high-cardinality COUNT(DISTINCT), HLL++ — what Spark and BigQuery use under the hood — gives ~2% error in 12KB."*

A few of these per round signals you read papers, not just textbooks.

---

## Reading List

### Recent (2023+)
- [BMSSP — Duan, Mao, Shu, Yin, 2025](https://arxiv.org/abs/2504.17033)
- [Survey: Learned Indexes (Ferragina & Vinciguerra, 2023)](https://www.vldb.org/pvldb/vol13/p1162-ferragina.pdf)
- [Velox: Meta's Unified Execution Engine](https://research.facebook.com/publications/velox-metas-unified-execution-engine/)
- [DataFusion — Apache Arrow Rust SQL Engine](https://arrow.apache.org/datafusion/)

### 2018–2023
- [Kraska et al. — The Case for Learned Index Structures (2018)](https://arxiv.org/abs/1712.01208)
- [DiskANN (NeurIPS 2019)](https://proceedings.neurips.cc/paper_files/paper/2019/file/09853c7fb1d3f8ee67a61b6bf4a7f8e6-Paper.pdf)
- [HNSW (Malkov & Yashunin, 2016)](https://arxiv.org/abs/1603.09320)
- [xor filters (Graf & Lemire, 2020)](https://arxiv.org/abs/1912.08258)
- [PEP 703 — GIL-free Python](https://peps.python.org/pep-0703/)

### Classic but Underused
- [RUM Conjecture (Athanassoulis et al., 2016)](https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf)
- [Cache-oblivious algorithms (Frigo et al., 1999)](https://erikdemaine.org/papers/FOCS99/)

### Companion Files
- [Storage & Vector Search](./13-storage-and-vector-structures.md) — where many of these advances live.
- [Probabilistic Structures](./12-probabilistic-structures.md) — xor filters, t-digest, theta.
- [Graphs & Algorithms](./10-graphs-and-algorithms.md) — Dijkstra context for BMSSP.
- [Hash Tables](./06-hash-tables.md) — Swiss/F14 context.

---

*This is the last file in the python/ section. Use it as a reference and as a talking-point library; the structures here are what separates senior from staff in a 2026 interview.*
