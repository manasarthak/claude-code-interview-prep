# Graphs — Representations and Algorithms

**Phase:** 4 (Graphs)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Stacks/Queues/Deques](./05-stacks-queues-deques.md) · [Trees & Heaps](./07-trees-and-heaps.md) · [Algorithm Design](./09-algorithm-design.md) · [Modern Advances](./14-modern-advances.md)

---

## Why This File Exists

Graph problems are everywhere in DE/AI — Airflow DAGs, lineage graphs, knowledge graphs, dependency analysis, social networks, embeddings as a graph (HNSW), citation networks, and most distributed-system problems can be modeled as graphs. Knowing how to **model** a problem as a graph is the senior signal; knowing the algorithms is just table stakes.

This file covers representations, the canonical algorithms, and how each maps to a real DE/AI problem. Modern advances (BMSSP, GPU graph algos) live in [Modern Advances](./14-modern-advances.md).

---

## BEGINNER — Representations and Vocabulary

### Vocabulary

- **Vertex (node)** — a point.
- **Edge** — a connection between two vertices.
- **Directed** — edges have direction (Twitter follows, dependency).
- **Undirected** — bidirectional (Facebook friendship).
- **Weighted** — edges have a numeric value (distance, cost, capacity).
- **DAG** — Directed Acyclic Graph (no cycles). Airflow, dbt, build systems.
- **Tree** — undirected, connected, acyclic. n vertices, n-1 edges.
- **Multigraph** — multiple edges between the same pair allowed.
- **Bipartite** — vertices split into two groups; edges only cross groups (jobs ↔ workers).

### Two Main Representations

#### Adjacency List

```python
graph = {
    'A': ['B', 'C'],
    'B': ['D'],
    'C': ['D'],
    'D': []
}
# Or with weights:
graph = {'A': [('B', 3), ('C', 5)], ...}
```

| | Adjacency list | Adjacency matrix |
|---|---|---|
| Space | O(V + E) | O(V²) |
| Edge lookup `(u, v)?` | O(deg(u)) | O(1) |
| Iterate neighbors of u | O(deg(u)) | O(V) |
| Best for | Sparse graphs (most real graphs) | Dense graphs, fixed V |

For real DE/AI graphs (billions of edges, sparse) — always adjacency list. Matrices show up in linear algebra (graph spectral methods, PageRank's matrix form).

#### Edge List

```python
edges = [('A', 'B', 3), ('A', 'C', 5), ('B', 'D', 1), ...]
```

Compact, easy to load from CSV/Parquet. Used as input format in Spark GraphFrames, NetworkX, when reading from a DB.

### Big-O Cheat Sheet (V vertices, E edges)

| Algorithm | Complexity |
|---|---|
| BFS / DFS | O(V + E) |
| Dijkstra (binary heap) | O((V + E) log V) |
| Dijkstra (Fibonacci heap, theoretical) | O(E + V log V) |
| Bellman-Ford | O(V·E) |
| Floyd-Warshall (all pairs) | O(V³) |
| Kruskal MST | O(E log E) |
| Prim MST (heap) | O((V + E) log V) |
| Topological sort | O(V + E) |
| Tarjan SCC | O(V + E) |
| BMSSP (2025, directed SSSP) | O(m·log^(2/3) n) — beats Dijkstra! |

---

## INTERMEDIATE — The Core Algorithms

### BFS — Breadth-First Search

Use a **queue** (`collections.deque`). Visits nodes layer by layer.

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    q = deque([start])
    while q:
        node = q.popleft()              # ← O(1), critical
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                q.append(neighbor)
    return visited
```

**Use cases:**
- Shortest path in **unweighted** graphs (each layer = one more edge).
- Web crawlers (visit nearby pages first).
- Social-network "friends within K hops."
- **GraphRAG retrieval** — expand neighborhood around an entity.

### DFS — Depth-First Search

Use a **stack** (explicit `list` or recursion).

```python
def dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()              # ← O(1), correct stack op
        if node in visited:
            continue
        visited.add(node)
        for neighbor in graph[node]:
            stack.append(neighbor)
    return visited
```

**Use cases:**
- Cycle detection (back edge during DFS).
- Topological sort.
- Strongly connected components.
- Maze solving / backtracking.
- Memory pretty cheap if graph is deep but narrow.

### Topological Sort — Order a DAG

Two approaches:

**Kahn's algorithm (BFS-style):**

```python
from collections import deque

def topo_sort(graph):
    indeg = {u: 0 for u in graph}
    for u in graph:
        for v in graph[u]:
            indeg[v] = indeg.get(v, 0) + 1
    q = deque([u for u, d in indeg.items() if d == 0])
    result = []
    while q:
        u = q.popleft()
        result.append(u)
        for v in graph[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                q.append(v)
    if len(result) != len(graph):
        raise ValueError("Graph has a cycle")
    return result
```

**DFS-based:** post-order DFS gives reverse topological order.

**Use cases:**
- **Airflow / dbt / Dagster execution order** — every DAG run is a topological traversal. See [Airflow](../data-engineer/05-orchestration-airflow.md).
- Build systems (Make, Bazel).
- Course prerequisites.
- Detecting cycles in dependency graphs.
- Spark RDD lineage execution.

### Dijkstra's Shortest Path

Single-source shortest paths in **non-negative weighted** graphs.

```python
import heapq

def dijkstra(graph, src):
    dist = {src: 0}
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue                          # stale entry
        for v, w in graph[u]:
            nd = d + w
            if nd < dist.get(v, float('inf')):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
```

- Time: O((V+E) log V) with a binary heap.
- **Cannot handle negative weights.**
- Greedy choice: always extract the closest unvisited node.

**Use cases:**
- Maps (Google Maps, Waze) — though they use bidirectional + A\* for speed.
- Network routing protocols (OSPF — link state).
- Game pathfinding (often A\* on top of Dijkstra).

### Bellman-Ford

Like Dijkstra but handles **negative weights** (and detects negative cycles). O(V·E).

Used in:
- Routing protocols where link costs can be negative-ish (RIP, BGP variants).
- Currency arbitrage detection.
- Slack analysis in Critical Path Method.

### A\* Search

Dijkstra with a heuristic `h(node)` that estimates remaining cost. Extract by `f(node) = g(node) + h(node)`.

If `h` is **admissible** (never overestimates), A\* is optimal. Used in:
- Game AI pathfinding.
- Logistics / route optimization.
- LLM agent planning (heuristics over partial solutions).

### Floyd-Warshall — All Pairs Shortest Paths

DP over all triples. O(V³). Practical only for V ≤ 1000.

Used when:
- Small graph + need every-pair distance.
- Transitive closure (reachability).
- Game maps with precomputed paths.

### Minimum Spanning Tree — Kruskal & Prim

**Kruskal:** sort edges by weight, add each if it doesn't form a cycle. Uses [Union-Find](./11-tries-strings-union-find.md). O(E log E).

```python
def kruskal(n, edges):
    edges.sort(key=lambda e: e[2])
    parent = list(range(n))
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]   # path compression
            x = parent[x]
        return x
    def union(x, y):
        px, py = find(x), find(y)
        if px == py: return False
        parent[px] = py
        return True
    return [(u, v, w) for u, v, w in edges if union(u, v)]
```

**Prim:** grow tree from a root, always adding the cheapest crossing edge. Uses a heap. O((V+E) log V).

**Use cases:**
- Cluster-network design (cheapest cabling).
- Image segmentation.
- Approximate TSP (MST + Christofides).
- Phylogenetic trees.

### Strongly Connected Components

A SCC is a maximal subgraph where every node reaches every other. Tarjan's and Kosaraju's algorithms both run in O(V + E).

Used in:
- Compiler dependency analysis.
- Detecting cycles in microservice call graphs.
- Web graph reduction (the "core of the web" is one giant SCC).

---

## ADVANCED — Modeling Real Problems as Graphs

### Lineage Graphs in Data Engineering

Every column-level data lineage system models datasets/columns as nodes and transformations as directed edges. The graph is a DAG (no cycles). See [Governance & Lineage](../data-engineer/14-governance-lineage-observability.md).

Operations:
- **Forward impact analysis** — "what breaks if I change column X?" → DFS/BFS from X.
- **Backward provenance** — "where does column Y come from?" → reverse DFS.
- **Topological replay** — recompute downstream after upstream change.

OpenLineage, DataHub, Marquez all build this graph as the central object.

### Airflow / dbt DAG Execution

The execution engine is fundamentally:
1. Build the DAG.
2. Topological sort.
3. Schedule tasks with no remaining dependencies onto a worker pool.
4. As tasks finish, decrement neighbor in-degrees; enqueue zero-degree tasks.

That's Kahn's algorithm with concurrent execution. See [Airflow](../data-engineer/05-orchestration-airflow.md).

### Knowledge Graphs / GraphRAG

In modern RAG, you index a corpus with entities and relations, building a knowledge graph. At query time:

1. Identify entities in the query.
2. BFS/DFS expand the neighborhood (1–3 hops).
3. Pull the relevant subgraph as additional context.

Microsoft's [GraphRAG](https://www.microsoft.com/en-us/research/project/graphrag/) and Neo4j's LLM Graph Builder both use this pattern. See [RAG](../ai/rag/01-rag-fundamentals.md).

### Vector Indexes as Graphs

HNSW (Hierarchical Navigable Small Worlds) — the index inside Pinecone, Weaviate, FAISS — is a **graph data structure**. Each vector is a node; edges connect "close" vectors. Search is a greedy graph traversal that hops toward the query. See [Storage & Vector Search](./13-storage-and-vector-structures.md).

### PageRank — The Original Web-Scale Graph Algorithm

```
PR(v) = (1-d)/N + d · Σ PR(u)/out_degree(u)   for u → v
```

Implemented as repeated matrix-vector multiplication on the adjacency matrix. Spark's `GraphFrames.pageRank` does this distributed.

Used for:
- Web ranking (the original).
- Twitter "Who To Follow."
- Recommendation systems.
- Neural network attention has flavor of PageRank (the random-walker interpretation).

### Bipartite Matching

Classic problem: given two groups (jobs ↔ workers, ads ↔ slots), find max set of edges with no shared endpoint. Solved with **Hungarian algorithm** (O(V³)) or **Hopcroft-Karp** (O(E·√V)).

Used in:
- Ad auction allocation.
- Driver-rider matching (Uber, Lyft do approximate variants).
- Resource scheduling.
- LLM token-to-token alignment.

### Max Flow / Min Cut

Ford-Fulkerson, Edmonds-Karp (O(VE²)), Dinic's (O(V²E)). The min-cut max-flow theorem makes this dual to many other problems.

Used in:
- Image segmentation.
- Bipartite matching reduces to max flow.
- Project selection.
- Network reliability.

### Graph Algorithms in Distributed Systems (Pregel / GraphX / Giraph)

For graphs too large for one machine, the **Pregel** model runs computations as supersteps:

1. Each vertex receives messages from previous superstep.
2. Each vertex updates its state and sends messages.
3. Synchronize; repeat until convergence.

PageRank, shortest paths, connected components all fit this model. Spark's GraphX, Apache Giraph, and Google's Pregel implement it.

### When Graph DBs (Neo4j, TigerGraph, ArangoDB) Win

A property graph DB optimizes for:
- Multi-hop traversals (1–6 hops typical).
- "Find paths that match a pattern" (Cypher / Gremlin).
- Real-time updates to a connected dataset.

Choose a graph DB over a relational DB when:
- Queries are predominantly **multi-hop** ("friends of friends of friends").
- Data is naturally connected (social, knowledge, fraud).
- Schema is flexible / evolving.

---

## Common Interview Gotchas

**Q: BFS or DFS for finding the shortest path in an unweighted graph?**
A: BFS. DFS doesn't guarantee shortest.

**Q: When does Dijkstra fail?**
A: With negative edge weights — it commits to a "shortest" too early. Use Bellman-Ford.

**Q: What's the complexity of Dijkstra in Python with `heapq`?**
A: O((V+E) log V). Note: `heapq` lacks decrease-key, so we push duplicates and skip stale entries — still asymptotically the same.

**Q: How do you detect a cycle in a directed graph?**
A: DFS with three colors (white/grey/black) — a grey-to-grey edge is a back edge → cycle. Alternatively, Kahn's topo sort fails iff there's a cycle.

**Q: Adjacency list vs matrix?**
A: List for sparse graphs (almost always real-world); matrix for dense graphs or when O(1) edge lookup matters more than O(V²) memory.

**Q: What's the worst-case complexity of BFS/DFS?**
A: Both O(V + E) — every vertex visited once, every edge processed once.

**Q: When would you use union-find?**
A: Kruskal MST, dynamic connectivity ("are these two nodes connected?" with online edge additions). See [Union-Find](./11-tries-strings-union-find.md).

**Q: How do you find connected components?**
A: Run BFS/DFS, count distinct visits → O(V + E). Or repeated unions in Union-Find → near O(V + E).

**Q: How big a graph can you handle on one machine?**
A: Adjacency list of 100M nodes with 1B edges fits in ~50GB RAM. Beyond that → distributed (Spark GraphX, Giraph).

---

## Interview-Ready Cheat Sheet

### Algorithm picker

| Problem | Algorithm |
|---|---|
| Shortest path, unweighted | BFS — O(V+E) |
| Shortest path, weighted, non-neg | Dijkstra — O((V+E) log V) |
| Shortest path, negative allowed | Bellman-Ford — O(V·E) |
| All-pairs shortest paths | Floyd-Warshall — O(V³) |
| Heuristic search | A\* |
| MST | Kruskal (sort+UF) or Prim (heap) |
| Cycle detection (directed) | DFS with colors, or Kahn fails |
| Topological order | Kahn (BFS) or DFS post-order |
| Strongly connected components | Tarjan or Kosaraju — O(V+E) |
| Bipartite matching | Hopcroft-Karp — O(E√V) |
| Max flow | Dinic's — O(V²E) |

### Trade-off pairs

- **BFS vs DFS:** layer-by-layer + queue space vs depth-first + stack/recursion.
- **Adjacency list vs matrix:** sparse vs dense.
- **Dijkstra vs Bellman-Ford:** non-neg + faster vs general but slower.
- **Kruskal vs Prim:** sort-once + UF vs heap-grow.
- **Single-machine vs distributed graph:** memory cap vs Pregel / GraphX overhead.

### Top Q&As

**"How would you design a feature impact analyzer for ML?"** → graph of features → models; reverse BFS/DFS from a feature.

**"How do you order task execution in Airflow?"** → topological sort (Kahn) + worker pool.

**"How do you detect circular imports?"** → cycle detection on the import graph.

**"How does Google Maps compute routes?"** → bidirectional Dijkstra + A\* + contraction hierarchies.

**"Find shortest path in a graph with 1B edges."** → distributed Pregel / GraphX, or external-memory algorithms; classical Dijkstra fails.

---

## Resources & Links

- *CLRS* ch. 22–26 — graphs, MST, shortest paths, max flow.
- [NetworkX docs](https://networkx.org/) — Python graph library; great for prototyping.
- [Spark GraphFrames](https://graphframes.github.io/graphframes/docs/_site/) — distributed graph ops.
- [Stanford CS 224W](https://web.stanford.edu/class/cs224w/) — Machine Learning with Graphs.
- [The Algorithms — graph implementations](https://github.com/TheAlgorithms/Python/tree/master/graphs) — readable Python.
- [BMSSP paper (2025)](https://arxiv.org/abs/2504.17033) — sub-Dijkstra SSSP. Covered in [Modern Advances](./14-modern-advances.md).

### Companion Files
- [Stacks/Queues/Deques](./05-stacks-queues-deques.md) — BFS uses a queue.
- [Trees & Heaps](./07-trees-and-heaps.md) — Dijkstra/Prim use a heap.
- [Union-Find](./11-tries-strings-union-find.md) — Kruskal MST.
- [Modern Advances](./14-modern-advances.md) — BMSSP, learned indexes, GPU graph.
- [DE: Airflow](../data-engineer/05-orchestration-airflow.md) — DAGs in production.

---

*Next: [Tries, Strings & Union-Find](./11-tries-strings-union-find.md).*
