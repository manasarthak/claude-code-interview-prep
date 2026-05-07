# Storage-Engine & Vector-Search Structures — The DE/AI Bridge

**Phase:** 5 (Differentiator)
**Difficulty progression:** Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Trees & Heaps](./07-trees-and-heaps.md) · [Probabilistic Structures](./12-probabilistic-structures.md) · [DE: Warehouses](../data-engineer/03-warehouses-lakes-lakehouse.md) · [AI: RAG](../ai/rag/01-rag-fundamentals.md)

---

## Why This File Exists

The structures here aren't on any classical CS syllabus, but they power **every system you'll be asked to design** in DE or AI:

- **B+ trees** are the index structure inside Postgres, MySQL, SQL Server, SQLite.
- **LSM trees** are the storage engine of Cassandra, ScyllaDB, RocksDB, LevelDB, HBase, BigTable.
- **Skip lists** power Redis sorted sets, LevelDB MemTable, Lucene posting lists.
- **HNSW graphs** are the index inside FAISS, Pinecone, Weaviate, Milvus, pgvector.
- **IVF + Product Quantization** are how billion-vector indexes fit on one machine.

Naming and explaining these correctly in an interview is what separates "knows DSA" from "would design a system that scales."

---

## STORAGE-ENGINE STRUCTURES

### B-Tree and B+ Tree — The Index Inside Every Relational DB

A B-tree is a balanced tree where each node holds many keys (often 100s) — sized to a disk page or SSD block (4 KB – 16 KB).

**B+ tree** specializations:
- **All values live at the leaves.**
- **Leaves are linked** in sorted order — enables fast range scans.
- Internal nodes only hold separator keys for routing.

```
                    [ 30 | 60 ]
                  /     |     \
            [10|20]  [40|50]  [70|80]
              ↓       ↓        ↓
           leaves linked: ──▶──▶──▶──▶
```

| Operation | Complexity (n keys, branching factor B) |
|---|---|
| Search | O(log_B n) |
| Insert | O(log_B n) |
| Delete | O(log_B n) |
| Range scan k keys | O(log_B n + k / B) — leaf list-walk |
| Disk I/Os per op | O(log_B n) — one I/O per level |

For B = 200 and n = 10⁹: log_B(n) ≈ 4. **Four disk reads to find any of a billion keys.** That's why B+ trees dominate OLTP indexes.

#### Where B+ Trees Live

| System | Use |
|---|---|
| Postgres, MySQL/InnoDB, Oracle, SQL Server | Default index structure (`CREATE INDEX`) |
| SQLite | Both tables and indexes are B-trees |
| MongoDB (WiredTiger) | B-tree storage engine option |
| File systems (NTFS, HFS+, APFS, ext4 dirs) | B-trees for directory entries |
| Iceberg / Delta data files (sort-keyed) | B-tree-like leaf layout via sorted Parquet |

#### The Read/Write Trade-off

B+ trees optimize **reads** (point and range queries). Inserts into a sorted index require I/O for each new key. For write-heavy workloads — telemetry, IoT, log streams — B+ trees become a bottleneck. Hence: LSM trees.

---

### LSM Tree — Log-Structured Merge Tree (the write-optimized cousin)

The LSM idea:
1. **Buffer all writes in memory** (a sorted structure — usually a skip list).
2. **Flush** periodically to an immutable sorted file on disk (an SSTable).
3. **Merge** older SSTables into bigger, more sorted SSTables (compaction).

This converts random-write I/O into sequential-write I/O — vastly faster on both HDDs and SSDs.

```
Memory (MemTable, skip list):  [ recent writes, sorted ]
                                          │ flush
                                          ▼
Level 0:  [ SSTable_1 ]  [ SSTable_2 ]  [ SSTable_3 ]
                                          │ compact
                                          ▼
Level 1:  [          big SSTable          ]
                                          │ compact
                                          ▼
Level 2:  [             bigger SSTable             ]
```

#### Read Path

A read may need to check: MemTable → Level 0 → Level 1 → ... Bloom filters per SSTable skip lookups for keys not present (see [Probabilistic Structures](./12-probabilistic-structures.md)).

| Operation | Complexity |
|---|---|
| Write | O(1) buffered + O(log n) eventual cost |
| Read (with Bloom) | ~O(log n) average; O(L · log n) worst with L levels |
| Range scan | O(log n + k); typically merges results from several SSTables |
| Compaction (background) | O(n) over many writes |

#### Where LSMs Live

| System | Use |
|---|---|
| Cassandra, ScyllaDB | LSM is the core storage engine |
| RocksDB, LevelDB | Embedded LSM KV stores; underlie Kafka Streams state, Flink, MyRocks |
| HBase | LSM with HFiles |
| BigTable, DynamoDB | LSM internally |
| InfluxDB, TimescaleDB | LSM-like for time-series |
| ClickHouse | MergeTree (LSM-inspired but specialized) |

#### Read vs Write Trade-off Summary

| | B+ tree | LSM tree |
|---|---|---|
| Write throughput | OK | Excellent |
| Read latency | Excellent | OK (better with Bloom filters) |
| Storage amplification | Low | Higher (compaction temp) |
| Range scan | Excellent | OK (multiple SSTables to merge) |
| Use case | OLTP index | Write-heavy KV / time-series / event store |

When asked "Postgres or Cassandra for this workload?" — name the read/write ratio and the structure underlying each.

---

### Skip List — Probabilistic Sorted Linked List

Already covered in [Linked Lists](./04-linked-lists.md), so just the highlights:
- O(log n) expected for search/insert/delete.
- Simpler concurrent implementations than balanced trees.
- Used in:
  - **Redis ZSET** (sorted set) — score → member with O(log n) range queries.
  - **LevelDB / RocksDB MemTable** — newest writes go here.
  - **HBase MemStore.**
  - **Lucene posting lists** — `nextDoc(target)` skip pointers.
  - **Apache Druid segment indexes.**

---

### Bw-Tree (Worth Name-Dropping for Senior Roles)

A **lock-free B-tree variant** that maps logical pages to physical addresses through an indirection table. Updates append delta records to the page chain, avoiding in-place mutation. Used by:
- Microsoft Hekaton (in-memory SQL Server)
- CockroachDB experiments
- Several modern HTAP systems

You won't be asked to implement it, but recognizing it shows depth on modern indexing.

---

### Learned Indexes (Modern Research)

Kraska et al. 2018 paper showed that an ML model can replace a B+ tree's structure: train a model to predict `position = f(key)`. Faster lookups, smaller memory.

Now in production-adjacent systems (Google's Bigtable Plasma, ALEX, RadixSpline). See [Modern Advances](./14-modern-advances.md).

---

## VECTOR-SEARCH STRUCTURES

In RAG and any embedding-based retrieval, you store millions–billions of high-dimensional vectors and need to retrieve the **k nearest neighbors** of a query vector. Exact KNN is O(n·d) per query — unusable above ~100K vectors. Hence specialized structures.

### Flat (Brute-Force) Index

Just a NumPy matrix; cosine or L2 distance to every vector. O(n·d) per query.

- **Best when n < 10⁵.**
- Always exact.
- Used as a "baseline" or final reranking layer in HNSW + flat hybrid indexes.

### IVF (Inverted File) Index

1. **Train phase:** k-means cluster the corpus into K centroids.
2. **Index:** assign each vector to its nearest centroid; store the inverted lists.
3. **Query:** find the closest `nprobe` centroids, search only their members.

| | Complexity |
|---|---|
| Build | O(n · K · iters) |
| Query | O(K + nprobe · n/K · d) — much smaller than O(n·d) |
| Recall | tunable via `nprobe` |

Used in: FAISS `IndexIVFFlat`, FAISS `IndexIVFPQ`, ScaNN, Vespa.

### Product Quantization (PQ)

Compress each d-dim vector to a tiny code by:
1. Split into m sub-vectors.
2. Quantize each sub-vector against a codebook of 2^c codes.
3. Store the m × c bits per vector.

A 768-d float32 vector (3 KB) compresses to ~16 bytes. Trade: ~5–10% recall loss.

Combined with IVF: **IVF-PQ** indexes 100M vectors in ~2 GB. The standard FAISS large-scale config.

### HNSW — Hierarchical Navigable Small World

The dominant ANN structure today. A multi-layer graph where:
- Lower layers contain all vectors with many edges to neighbors.
- Higher layers are progressively sparser, like an "express lane" for long jumps.

Search starts at a top-layer entry node, greedily hops toward the query, descends to lower layers, refines.

| | Complexity |
|---|---|
| Build | O(n · log n · M) with branching M |
| Query | ~O(log n · d) expected |
| Recall | very high (95%+ typical) |
| Memory | ~vector size + M·8 bytes per vector |

#### Where HNSW Lives

| System | Use |
|---|---|
| **Pinecone, Weaviate, Qdrant, Milvus, Vespa** | Default index. |
| **FAISS** | `IndexHNSW`. |
| **pgvector (Postgres)** | HNSW since v0.5.0; the default for production. |
| **Elasticsearch / OpenSearch dense vectors** | HNSW (via Lucene). |
| **Chroma** | HNSW under the hood. |

### DiskANN

Microsoft's algorithm for **billion-scale** ANN that lives on SSD. Combines compressed vectors in RAM (PQ codes) with a graph index on disk. Can serve a 1B-vector index from a single VM with 64GB RAM. Used by Bing, OpenAI's RAG infra, several enterprise vector DBs.

### Picking a Vector Index

| Scale | Index | Why |
|---|---|---|
| <100K vectors | Flat (exact) | Brute force fine |
| 100K – 1M | HNSW | Fast, high recall |
| 1M – 100M | IVF-PQ or HNSW | Memory matters |
| 100M – 1B | IVF-PQ or DiskANN | Compression + disk |
| >1B | DiskANN, sharded HNSW | Distributed |

This decision shows up in [RAG system design](../ai/rag/01-rag-fundamentals.md). Naming HNSW vs IVF-PQ correctly is the senior signal.

### Distance Metrics

- **L2 (Euclidean)** — generic.
- **Cosine** — dominant for text embeddings (normalize, then it's equivalent to dot product).
- **Inner product (IP)** — when embeddings are unnormalized but optimized for IP similarity.
- **Hamming** — for binary/quantized embeddings.

Most LLM embedding models (OpenAI ada-002/3, Cohere, Sentence-BERT) are tuned for cosine.

---

## Common Interview Gotchas

**Q: Why do databases use B+ trees instead of red-black trees?**
A: Each B+ tree node fits a disk page; one I/O fetches many keys. RB-trees would do one I/O per compare → 100× slower on disk.

**Q: When does an LSM beat a B+ tree?**
A: Write-heavy workloads, especially append-mostly (logs, time-series, telemetry). LSMs sequentialize writes; B+ trees don't.

**Q: Why does Cassandra need Bloom filters?**
A: Reads must check multiple SSTables. Bloom skips SSTables that definitely don't have the key, eliminating most disk reads.

**Q: What's "write amplification" in LSMs?**
A: Each value is rewritten by every compaction it goes through. RocksDB's WA is typically 10–30×. Tunable via compaction strategy.

**Q: Why is HNSW the default vector index in 2026?**
A: High recall (95%+) at logarithmic query time, in-place updates, and works well in pure RAM. Trade-off is memory and slower builds.

**Q: When would you choose IVF-PQ over HNSW?**
A: At billion-vector scale where compression matters more than recall, or when the index needs to live on disk.

**Q: Why is exact KNN bad for 10M vectors?**
A: O(n·d) per query — at d = 768, n = 10M, that's ~7.7B float ops per query — multi-second latency.

**Q: What's the role of Bloom filters in Iceberg/Parquet?**
A: Skip data files / row groups that don't contain a queried value, saving I/O on selective filters.

---

## Interview-Ready Cheat Sheet

### Pick the structure

| Need | Right structure |
|---|---|
| OLTP point/range query | B+ tree |
| Write-heavy KV / log / time-series | LSM tree |
| Concurrent ordered index | Skip list |
| Approximate membership | Bloom / Cuckoo filter |
| Sorted set with rank | Skip list (Redis ZSET) |
| ANN over <1M vectors | HNSW |
| ANN over 1B vectors | IVF-PQ or DiskANN |
| Exact KNN for tiny indexes | Flat |

### One-liners

- **B+ trees fit a node per page → log_B reads → 4–5 I/Os for a billion keys.**
- **LSMs trade compaction overhead for sequential write speed.**
- **Bloom filters in Cassandra/RocksDB skip 90%+ of unnecessary disk reads.**
- **HNSW is graph-based ANN: expected O(log n · d) queries, very high recall.**
- **IVF-PQ + HNSW are why a billion-vector index fits on one machine.**

### Trade-off pairs

- **B+ vs LSM:** read-optimized vs write-optimized.
- **Skip list vs balanced tree:** simpler concurrency vs deterministic balance.
- **Bloom vs Cuckoo:** simple vs supports delete + better FPR.
- **Flat vs HNSW vs IVF-PQ:** exact + slow vs approximate + fast vs compressed + cheap.
- **HNSW vs DiskANN:** RAM-resident, fast vs disk-resident, billion-scale.

---

## Resources & Links

- *Designing Data-Intensive Applications* (Kleppmann), ch. 3 — B-trees and LSMs in detail.
- [LSM Tree paper (O'Neil et al., 1996)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
- [RocksDB tuning guide](https://github.com/facebook/rocksdb/wiki) — production knobs.
- [Postgres B-tree internals](https://www.postgresql.org/docs/current/btree.html)
- [HNSW paper (Malkov & Yashunin, 2016)](https://arxiv.org/abs/1603.09320)
- [DiskANN paper (Subramanya et al., NeurIPS 2019)](https://proceedings.neurips.cc/paper_files/paper/2019/file/09853c7fb1d3f8ee67a61b6bf4a7f8e6-Paper.pdf)
- [FAISS Wiki — Index types](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes)
- [pgvector docs](https://github.com/pgvector/pgvector)
- [Pinecone — algorithms guide](https://www.pinecone.io/learn/series/faiss/)
- [Learned Indexes paper (Kraska et al., 2018)](https://arxiv.org/abs/1712.01208)

### Companion Files
- [Trees & Heaps](./07-trees-and-heaps.md) — B-tree generalization context.
- [Linked Lists](./04-linked-lists.md) — skip lists.
- [Probabilistic Structures](./12-probabilistic-structures.md) — Bloom in LSMs.
- [DE: Warehouses](../data-engineer/03-warehouses-lakes-lakehouse.md) — where these live.
- [DE: Partitioning & Performance](../data-engineer/13-partitioning-performance.md)
- [AI: RAG Fundamentals](../ai/rag/01-rag-fundamentals.md) — vector indexes in context.

---

*Next: [Modern Advances](./14-modern-advances.md) — what's new in 2024–2026 that you can name-drop.*
