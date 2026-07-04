# Distributed Systems — Concept Cheatsheet

**Phase:** 2 (System Design)
**Difficulty:** Beginner → Intermediate
**Last updated:** July 3, 2026
**Related:** [System Design for DE](./15-system-design-de.md) · [Batch vs Streaming](./06-batch-vs-streaming.md) · [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md)

---

## Why This File Exists

System-design questions test recognition of terms and one-line justifications: CAP, ACID vs BASE, sharding, replication, caching, load balancing, Kubernetes, APIs. This file is the compact reference. Deep worked examples are in [System Design for DE](./15-system-design-de.md).

---

## CAP THEOREM

> In the presence of a **network Partition**, a distributed system can guarantee **Consistency** OR **Availability**, not both.

- **C (Consistency)** — every read gets the most recent write (or an error). Linearizability in the strict definition.
- **A (Availability)** — every request gets a response (may be stale).
- **P (Partition tolerance)** — the system keeps working even when the network splits nodes into disconnected groups.

**P is not optional** in real distributed systems — network partitions happen. So in practice CAP is a choice between C and A **during a partition**.

### Examples

| System | Choice during partition | Why |
|---|---|---|
| Postgres, MySQL (single primary) | CP | Fails writes on partition rather than serving stale |
| Cassandra, DynamoDB (default) | AP | Accept writes anywhere; eventual consistency |
| Zookeeper, etcd | CP | Consensus (Paxos/Raft); refuse work when quorum lost |
| MongoDB | CP by default (majority write concern) | Configurable via write concern |
| Kafka | CP | Refuses producers on partition without ISR |

### PACELC — the CAP extension you should know

> **P**artition → **A**vailability vs **C**onsistency. **E**lse (no partition) → **L**atency vs **C**onsistency.

Even without a partition, there's a trade-off between speed (lower latency = weaker consistency) and correctness. Cassandra is AP + EL (chooses latency). Spanner is CP + EC (chooses consistency).

### Common traps

- **"CAP means you pick 2 of 3."** — technically true, but P isn't optional; the real trade-off is C vs A.
- **"MongoDB is CP."** — depends on write concern.
- **"CAP applies to single-machine databases."** — no, it's specifically about distributed systems with partitions.

---

## ACID vs BASE

### ACID (traditional OLTP databases)

- **A**tomicity — transaction all-or-nothing.
- **C**onsistency — DB moves from one valid state to another; constraints preserved.
- **I**solation — concurrent txns behave as if serial (levels: read-uncommitted / committed / repeatable-read / serializable).
- **D**urability — committed data survives crashes.

Examples: Postgres, MySQL, Oracle, SQL Server, Spanner.

### BASE (many NoSQL / distributed systems)

- **B**asically **A**vailable — the system stays available even during partitions.
- **S**oft state — state may change over time even without input (eventual consistency background).
- **E**ventual consistency — reads converge to latest write eventually.

Examples: Cassandra, DynamoDB (some modes), Riak, Couchbase.

### Isolation Levels (interview-common)

| Level | Prevents | Allows |
|---|---|---|
| Read uncommitted | — | Dirty reads |
| Read committed (Postgres default) | Dirty reads | Non-repeatable reads, phantoms |
| Repeatable read (MySQL InnoDB default) | Non-repeatable reads | Phantoms |
| Serializable | Everything (behaves as if serial) | — |

Anomalies to know:
- **Dirty read** — reading uncommitted data from another txn.
- **Non-repeatable read** — same query returns different rows within a txn.
- **Phantom read** — new rows appear that match a WHERE clause within a txn.
- **Lost update** — two txns update the same row, one wins silently.

---

## CONSISTENCY MODELS

Ranked strongest → weakest:

| Model | Guarantee |
|---|---|
| **Linearizability** (strong) | All operations appear to execute atomically in some real-time order |
| **Sequential consistency** | Same order for all clients, but not necessarily real-time |
| **Causal consistency** | Preserves happens-before order; concurrent writes may be seen in different orders |
| **Read-your-writes** | You see your own writes |
| **Monotonic reads** | Never see an older value after a newer one |
| **Eventual consistency** | Reads converge to latest write, given enough time and no new writes |

Note: "linearizable" = "strong consistency" = "one copy semantics." Not the same as serializability (that's for transactions, not single-object reads).

---

## SHARDING (partitioning data across nodes)

- **Range sharding:** rows with keys in [a, b) go to shard X. Good for range scans (time-series). Risk: hot shards on skewed keys.
- **Hash sharding:** `shard = hash(key) mod N`. Uniform distribution but breaks range scans.
- **Consistent hashing:** map both keys and shards to a ring; each key goes to the next shard clockwise. **Adding a shard only relocates ~1/N of keys** — huge win over vanilla hash-mod. Used by Cassandra, DynamoDB, Memcached client libs. See [Hash Tables](../python/06-hash-tables.md#consistent-hashing).
- **Rendezvous hashing (HRW):** for each key, compute `h(key, shard)` for every shard and pick the max. Alternative to consistent hashing; simpler for some use cases.
- **Directory-based sharding:** a separate service maps key → shard. Flexible but a single point of failure.

### The Hot Shard Problem

Even with hash sharding, one key can dominate traffic (e.g., celebrity user). Solutions:

- **Sub-sharding hot keys** — append random suffix to spread across shards.
- **Salting** — for join keys with skew, salt one side, expand the other. See [Spark](./07-spark-fundamentals.md).
- **Read replicas** for the hot shard.

---

## REPLICATION (copying data across nodes)

- **Leader–follower (a.k.a. primary–secondary, master–slave):** writes go to leader, replicated to followers. Followers serve reads.
  - **Synchronous replication:** slow but strongly consistent.
  - **Asynchronous:** fast, but followers can be stale; risk of data loss on leader crash.
  - **Semi-sync:** wait for at least one follower to acknowledge.
- **Multi-leader:** writes accepted anywhere; conflicts resolved by policy (last-write-wins, custom merger, CRDTs). Higher availability, harder consistency.
- **Leaderless (Dynamo-style):** clients write to N replicas, read from R; if W + R > N, quorum guarantees single-value read. Cassandra, DynamoDB, Riak.

### Replication Lag

Async replication means followers lag behind. Common consequences:
- **Read-your-writes issue** — write hits leader, immediate read hits follower, user "loses" their own write.
- **Monotonic-reads issue** — user sees a value, refreshes, sees older value (different follower).
- **Cross-user causality** — B replies to A's post before A's post is visible to some readers.

Fixes: read-from-leader for own writes, session affinity, causal tokens.

---

## SQL vs NoSQL

| Family | Examples | Best for |
|---|---|---|
| **Relational (SQL)** | Postgres, MySQL, Oracle, SQL Server, Spanner | Structured data, complex joins, transactions, ACID |
| **Key-Value** | Redis, DynamoDB, Riak, Memcached | Simple lookups, caches, session stores |
| **Wide-column** | Cassandra, HBase, ScyllaDB, BigTable | Time-series, IoT, write-heavy, huge datasets |
| **Document** | MongoDB, Couchbase, Firestore | Semi-structured JSON, flexible schema, one-collection reads |
| **Graph** | Neo4j, TigerGraph, JanusGraph, ArangoDB | Many-hop relationships, social/knowledge graphs |
| **Time-series** | InfluxDB, TimescaleDB, Prometheus | Metrics, sensor data with heavy write + time-range queries |
| **Search** | Elasticsearch, OpenSearch, Solr | Full-text search, log analytics |
| **Vector** | Pinecone, Weaviate, Qdrant, Milvus, pgvector | Embeddings, ANN retrieval for RAG |

Rule of thumb: relational when you need JOINs and transactions; key-value when you need blazing-fast point reads; wide-column for hyperscale writes; document for flexible schema; graph for multi-hop relations.

---

## CACHING

### Cache Placement

- **Client-side** (browser, mobile app cache).
- **CDN** (CloudFront, Fastly, Akamai) — edge cache for static assets.
- **Reverse proxy** (nginx, Varnish).
- **Application-level** (in-process, Redis, Memcached).
- **Database-level** (query cache).

### Cache Strategies

- **Cache-aside (lazy loading):** app reads cache; on miss, loads from DB and populates cache. Most common. Stale on write unless invalidated.
- **Read-through:** app reads cache, cache loads from DB on miss. Similar to cache-aside but abstracted.
- **Write-through:** writes go to cache and DB synchronously. Consistent but slow writes.
- **Write-back (write-behind):** writes go to cache, async flushed to DB. Fast writes but risk of loss on cache crash.
- **Write-around:** writes go directly to DB, cache is populated on first read.

### Eviction Policies

- **LRU** (Least Recently Used) — most common.
- **LFU** (Least Frequently Used) — better for hot-set workloads.
- **FIFO** (First In First Out).
- **Random** — surprisingly OK for uniform-access workloads.
- **TTL-based** — evict after expiry.

Redis supports `allkeys-lru`, `volatile-lru`, `allkeys-lfu`, and combinations.

### Cache Problems (interview favorites)

- **Cache stampede / thundering herd** — a hot key expires; many concurrent requests all miss and hit DB. Solutions: request coalescing, probabilistic early expiration, "hot key" replication.
- **Cache penetration** — requests for keys that don't exist repeatedly hit DB. Solutions: cache negative results with short TTL, Bloom filter.
- **Cache avalanche** — many keys expire simultaneously. Solutions: jittered TTLs.

---

## LOAD BALANCING

### Layer 4 vs Layer 7

- **L4 (Transport, TCP/UDP):** load balancer routes by IP:port; opaque to payload. Fast. Examples: AWS NLB, HAProxy in TCP mode.
- **L7 (Application, HTTP/gRPC):** routes based on URL, headers, cookies. Feature-rich (path routing, JWT auth). Examples: nginx, AWS ALB, Envoy, Traefik.

### Algorithms

- **Round-robin** — rotate through backends.
- **Weighted round-robin** — larger backends get more requests.
- **Least connections** — send to backend with fewest active connections.
- **Least response time** — pick fastest backend.
- **IP hash / consistent hash** — pin same client / same key to same backend. Useful for session affinity and cache locality.
- **Random** — surprisingly OK for uniform loads.

### Health Checks

Active (LB pings backends periodically) and passive (LB observes failures on real traffic). Failed backends are removed from rotation until they recover.

---

## MESSAGE QUEUES / STREAMING

Compare quickly (deeper in [Batch vs Streaming](./06-batch-vs-streaming.md)):

| System | Model | Ordering | Delivery | Retention |
|---|---|---|---|---|
| **Kafka** | Distributed log, pub/sub | Per-partition | At-least-once (exactly-once possible) | Configurable, days–weeks |
| **Kinesis** | Kafka-alike (AWS) | Per-shard | At-least-once | 24h–365d |
| **RabbitMQ** | Broker with queues, pub/sub, routing | Per-queue | At-most / at-least / exactly | Until consumed |
| **SQS** | Standard queue (AWS) | Unordered (standard) or FIFO | At-least-once (standard), exactly (FIFO) | Up to 14 days |
| **NATS / NATS JetStream** | Pub/sub + streaming | Per-subject | At-most or at-least | Configurable |
| **Redis Streams** | Log-like | Per-stream | At-least-once via ACKs | Configurable |

**Delivery guarantees:**
- **At-most-once:** message may be lost, never duplicated.
- **At-least-once:** message may be duplicated, never lost. Requires **idempotent consumers**.
- **Exactly-once:** hardest; usually at-least-once + idempotent sink is the practical answer.

**Kafka specifics (very high-frequency in interviews):**
- **Topic** — logical stream. **Partition** — parallelism unit; ordering guaranteed *within a partition only*.
- **Producer** — publishes to a topic; picks partition by key hash (default) or round-robin.
- **Consumer group** — partitions distributed among consumers; one partition = one consumer within a group.
- **Offset** — consumer's position in a partition; committed after processing.
- **ISR (In-Sync Replicas)** — followers caught up with the leader.
- **Replication factor** — how many copies of each partition. Typical: 3.
- **min.insync.replicas + acks=all** = strong durability.

---

## CONTAINERS AND ORCHESTRATION

### Docker Basics

- **Image** — read-only template (OS + app + deps).
- **Container** — runtime instance of an image.
- **Dockerfile** — build recipe.
- **Layers** — each Dockerfile instruction adds a layer; layers are cached and reused.
- **Registry** — stores images (Docker Hub, ECR, GHCR).
- **Container vs VM** — containers share the host kernel (lighter); VMs have their own kernel (heavier, stronger isolation).

### Kubernetes Basics

- **Pod** — smallest deployable unit; usually one container per pod.
- **Deployment** — manages a set of pod replicas; supports rolling updates.
- **Service** — stable network endpoint for a set of pods.
- **Ingress** — HTTP(S) routing into the cluster.
- **ConfigMap / Secret** — externalized config / secrets.
- **StatefulSet** — pods with stable identity + storage (for databases).
- **DaemonSet** — one pod per node (log collectors, monitoring).
- **CronJob** — scheduled jobs.
- **Namespace** — logical isolation within a cluster.
- **HPA (Horizontal Pod Autoscaler)** — scales pods based on CPU/mem/custom metrics.
- **Node** — a worker machine in the cluster.
- **Control plane** — API server, scheduler, controller manager, etcd.

Note: "container that shares the host OS kernel" = container (not VM). "Smallest K8s unit" = pod.

---

## APIs

### REST

- **Verbs map to CRUD:** GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE.
- **Statelessness** — every request carries all context; server holds no session state.
- **Idempotency** — GET, PUT, DELETE are idempotent. POST is not.
- **HTTP status codes (interview-common):**
  - 2xx success (200 OK, 201 Created, 204 No Content).
  - 3xx redirect (301 Moved, 304 Not Modified).
  - 4xx client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 429 Too Many Requests).
  - 5xx server error (500 Internal, 502 Bad Gateway, 503 Unavailable, 504 Gateway Timeout).

### GraphQL

- Single endpoint; client specifies exact fields to return.
- Avoids over/under-fetching common in REST.
- N+1 problem if resolvers naively fetch relations; solved with DataLoader batching.
- Type system + introspection.

### gRPC

- Binary protocol over HTTP/2. Fast, low overhead.
- Strongly typed via Protocol Buffers (.proto files).
- Supports streaming (bidirectional).
- Great for microservice-to-microservice; harder to debug than JSON.

### Comparison

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Wire format | JSON | JSON | Protobuf (binary) |
| Schema | OpenAPI (optional) | Required | Required (.proto) |
| Streaming | Limited (SSE, WebSockets) | Subscriptions | Bidi native |
| Human-readable | Yes | Yes | No (needs proto) |
| Browser-friendly | Yes | Yes | Needs grpc-web bridge |
| Speed | Moderate | Moderate | Fast |

---

## API RATE LIMITING

- **Fixed window** — N requests per minute; edge issues at window boundaries (bursts of 2N).
- **Sliding window log** — track timestamps of each request. Accurate but memory-heavy.
- **Sliding window counter** — approximation of sliding log; standard.
- **Token bucket** — refill tokens at rate r, allow burst up to capacity. Standard.
- **Leaky bucket** — process at fixed rate regardless of arrivals; smooths bursts.

429 Too Many Requests is the status code to return when limits are exceeded.

---

## SECURITY / AUTH BASICS

- **Authentication (AuthN)** — who are you.
- **Authorization (AuthZ)** — what can you do.
- **JWT (JSON Web Token)** — signed token containing claims; stateless auth.
- **OAuth 2.0** — delegated authorization framework (grants + tokens).
- **OpenID Connect (OIDC)** — identity layer on top of OAuth 2.0.
- **API keys** — simplest; identify calling app.
- **mTLS** — mutual TLS; both client and server present certs.
- **RBAC vs ABAC** — role-based (user has role X) vs attribute-based (rules over user + resource + context attributes).
- **Encryption at rest** — data encrypted on disk (KMS-managed keys).
- **Encryption in transit** — TLS.

---

## MICROSERVICES vs MONOLITH

| | Monolith | Microservices |
|---|---|---|
| Deployment | One artifact | Many, independent |
| Scaling | Whole app | Per-service |
| Team boundary | Feature branch | Own the whole service |
| Failure isolation | Weak (one bug = whole down) | Better (blast radius contained) |
| Latency | Local calls | Network hops |
| Complexity | Low ops | High ops (service discovery, tracing, retries) |
| Data | One DB | DB per service (usually) |
| Best for | Small teams, early stage | Large orgs, mature scale |

---

## RESILIENCE PATTERNS

- **Timeout** — always set one for network calls.
- **Retry with exponential backoff + jitter** — retry on transient failures; jitter prevents thundering herd.
- **Circuit breaker** — stop calling a failing service after N errors; half-open probe periodically.
- **Bulkhead** — isolate resources (thread pools, connections) so one bad tenant can't starve others.
- **Idempotency key** — client supplies a unique key; server dedupes.
- **Rate limiting** (client-side and server-side).
- **Graceful degradation** — serve stale data / defaults when downstream is down.
- **Dead letter queue** — messages that fail all retries go here for manual inspection.

---

## THE 25 ONE-LINERS TO MEMORIZE

1. **CAP theorem** — during a partition, pick C or A; P is not optional.
2. **PACELC** — Partition → A/C; Else → L/C.
3. **ACID vs BASE** — strong txns vs eventually-consistent + high availability.
4. **Serializable** = strongest txn isolation; **Read Committed** = Postgres default; **Repeatable Read** = MySQL InnoDB default.
5. **Consistent hashing** — adding a shard only relocates ~1/N of keys.
6. **Sync vs async replication** — safety vs speed.
7. **Cache-aside** — most common cache pattern; app manages cache.
8. **LRU** — most common eviction policy.
9. **Cache stampede** — hot key expires + concurrent load. Fix: request coalescing / probabilistic early expiration.
10. **L4 vs L7 LB** — transport (fast, opaque) vs application (feature-rich).
11. **Kafka partition** — ordering guaranteed within, not across.
12. **Consumer group in Kafka** — partitions split among consumers; one partition = one consumer per group.
13. **At-least-once + idempotent sink** = practical exactly-once.
14. **Docker container vs VM** — shares host kernel vs runs own kernel.
15. **K8s Pod** — smallest deployable unit.
16. **REST** — stateless; GET/POST/PUT/DELETE/PATCH; verbs + URIs.
17. **Idempotent HTTP verbs** — GET, PUT, DELETE. **Not** POST.
18. **gRPC** — HTTP/2 + Protobuf; fast, typed, bidi streaming.
19. **429** — Too Many Requests (rate limit).
20. **JWT** — stateless auth token, signed, claims-based.
21. **RBAC** — roles decide access. **ABAC** — attributes decide access.
22. **Circuit breaker** — stop calling failing service; half-open probe.
23. **Exponential backoff + jitter** — retry strategy for transient failures.
24. **DLQ (Dead Letter Queue)** — messages that fail all retries end here.
25. **Sharding** = horizontal partitioning across nodes. **Replication** = copies of same data.

---

## Companion Files

- [System Design for DE](./15-system-design-de.md) — 4 full worked examples + framework.
- [Batch vs Streaming](./06-batch-vs-streaming.md) — Kafka/Kinesis/Flink deeper.
- [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) — OLTP vs OLAP.
- [ETL vs ELT](./04-etl-vs-elt.md) — idempotency, backfills.
- [CDC](./09-cdc.md).
- [Partitioning & Performance](./13-partitioning-performance.md).
- [Python: Hash Tables — Consistent Hashing](../python/06-hash-tables.md#hash-partitioning-in-distributed-systems).
- [Python: Storage Structures](../python/13-storage-and-vector-structures.md) — B+/LSM/skip list.

## External refs

- *Designing Data-Intensive Applications* (Kleppmann) — the bible; ch. 5–9 map to this file.
- [Kafka docs](https://kafka.apache.org/documentation/).
- [Kubernetes concepts](https://kubernetes.io/docs/concepts/).
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — free online.
- [High Scalability](http://highscalability.com/) — case studies.
- [System Design Primer (GitHub)](https://github.com/donnemartin/system-design-primer) — most-starred repo for this topic.

---

*Skim the 25 one-liners before any system-design interview. Term-recognition + one-line-justification wins the quick-fire questions.*
