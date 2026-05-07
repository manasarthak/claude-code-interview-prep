# Tries, String Algorithms & Union-Find

**Phase:** 5 (Differentiator)
**Difficulty progression:** Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Hash Tables](./06-hash-tables.md) · [Graphs & Algorithms](./10-graphs-and-algorithms.md) · [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md)

---

## Why This File Exists

These three structures are the "you didn't learn this in CS101" differentiators. Tries power autocomplete and BPE tokenizers. String-matching algorithms (KMP, Aho-Corasick) are the foundation of grep, log scanning, and security tools. Union-Find is the elegant way to track dynamic connectivity in graphs of billions, and it's the engine inside Kruskal's MST.

If you can speak to all three under interview pressure, you outrank 80% of generalist candidates.

---

## TRIES — Prefix Trees

### The Concept

A trie indexes **strings** by walking a path from the root through one node per character.

```
        (root)
       /  |   \
      a   b    c
     /|   |    |
    n p   a    a
    |  |  |    |
    d  l  t    t
       e
       
Words: and, apple, bat, cat
```

Each path from root to a marked node is a stored word.

### Big-O

| Operation | Complexity (k = key length) |
|---|---|
| Insert | O(k) |
| Search exact | O(k) |
| Search prefix | O(k) |
| Delete | O(k) |
| Space | O(total characters) |

**Key insight:** trie ops don't depend on n (the number of stored words). For a billion words averaging 10 chars each, all ops are O(10).

### Minimal Python Implementation

```python
class Trie:
    def __init__(self):
        self.root = {}        # char -> dict (recursive)
        self.END = '$'        # sentinel marking end of word

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.setdefault(ch, {})
        node[self.END] = True

    def __contains__(self, word):
        node = self.root
        for ch in word:
            if ch not in node: return False
            node = node[ch]
        return self.END in node

    def starts_with(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node: return False
            node = node[ch]
        return True
```

For production scale, you'd use compact arrays / bitmaps instead of dicts at each node — a real trie of 1M English words can fit in ~30MB with the right representation.

### Where Tries Show Up

| System | Use |
|---|---|
| **BPE / WordPiece tokenizers in LLMs** | The tokenizer is a trie of byte sequences; longest-match wins. See [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md). |
| **Autocomplete (Google, IDE intellisense)** | Trie walked on each keystroke; suffixes ranked by frequency. |
| **Spell checkers** | Trie + edit-distance walk. |
| **IP routing tables (longest prefix match)** | A radix-tree variant (compressed trie). |
| **`pip install` / package resolvers** | Trie of package name prefixes for fast lookup. |
| **Aho-Corasick** | Trie + failure links → multi-pattern string matching. |
| **DNA / protein motif search** | Suffix tries / suffix trees / suffix arrays. |

### Variants Worth Naming

- **Compressed trie / radix tree** — collapse chains of single children. Used in routing, file systems (ext4 dirs).
- **Suffix tree** — trie of all suffixes of a string; search any substring in O(m). Built in O(n) (Ukkonen). Heavy memory.
- **Suffix array** — sorted array of suffix start indices; same purpose, ~5× less memory. Used in BWT, bzip2, grep.
- **DAWG (Directed Acyclic Word Graph)** — minimized trie sharing common suffixes. Saves memory.

---

## STRING ALGORITHMS — Match Patterns Fast

### Naive Substring Search

```python
def naive_find(text, pat):
    n, m = len(text), len(pat)
    for i in range(n - m + 1):
        if text[i:i+m] == pat:
            return i
    return -1
```

O(n·m) worst case. Python's built-in `text.find(pat)` and `in` operator implement a **Boyer-Moore-Horspool variant** — O(n) average, O(n·m) worst.

### KMP (Knuth-Morris-Pratt)

Precompute a failure function so the pattern never re-matches a known prefix. O(n + m). Used in grep, simple log scanners.

### Boyer-Moore

Skip ahead by **bad-character** and **good-suffix** tables. Sublinear in practice (skips most of the text). Used in `grep`, text editors.

### Aho-Corasick — Multi-Pattern Matching

Build a trie of all patterns + failure links. Scan the text once in O(n + total pattern length + matches). 

Used in:
- `grep -F -f patterns.txt` (fast multi-string search).
- IDS / IPS (Snort) for intrusion patterns.
- Spam filtering on a corpus of patterns.
- Log scanning for many keywords.
- Bloom filter pre-filter + Aho-Corasick verifier in some DBs.

### Z-algorithm and Suffix Automata

Z-algorithm computes "longest substring starting at i that matches a prefix" in O(n). Underrated; powers many competitive-programming string solutions.

Suffix automata generalize to all substrings. Used in compression and pattern-mining.

### Regex Engines

Modern regex engines come in two flavors:
- **Backtracking (PCRE, Python `re`)** — exponential worst case (catastrophic backtracking).
- **NFA/DFA-based (RE2, Hyperscan)** — guaranteed O(n) in text length.

For DE: when scanning logs at scale, use Google's RE2 (`re2` package) or Intel Hyperscan, not Python `re`. A bad regex on 1TB of logs is the canonical "stalled job" story.

### Edit Distance

Levenshtein distance via DP, O(n·m). Used in:
- Spell-checking ("did you mean...?").
- DNA sequence alignment.
- Fuzzy joins / record linkage in data quality. See [DE: Data Quality](../data-engineer/11-data-quality-contracts.md).
- LLM evaluation (BLEU, ROUGE — n-gram-based but related family).

---

## UNION-FIND (Disjoint Set Union)

### The Problem It Solves

Maintain a partition of n elements into disjoint sets. Support:
- `find(x)` — which set is x in?
- `union(x, y)` — merge x's and y's sets.

Naively, O(n) per operation. With **path compression + union by rank**, amortized **α(n)** — the inverse Ackermann function, ~4 for any conceivable n. Effectively O(1).

### Implementation

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path compression
            x = self.parent[x]
        return x

    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py: return False
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        return True
```

### Where Union-Find Lives

| Use case | How |
|---|---|
| **Kruskal's MST** | Sort edges; union iff find differs. See [Graphs](./10-graphs-and-algorithms.md). |
| **Connected components** | Union over edges; count distinct roots. |
| **Online connectivity** | "Are u and v connected?" answered in α(n). |
| **Image segmentation** | Union pixels with similar properties. |
| **Detecting cycles in undirected graph** | If `find(u) == find(v)` already, this edge would form a cycle. |
| **Spark / GraphFrames connected components** | Each iteration unions adjacent vertices in a Pregel-style algorithm. |
| **Identity resolution / record linkage** | Union records whose distance < threshold. See [DE: Project A1](../projects/README.md). |
| **Kruskal-style scheduling** | Job dependencies into job-bundles. |

### Optimizations Quick Reference

- **Path compression alone:** O(log n) amortized.
- **Union by rank/size alone:** O(log n) amortized.
- **Both combined:** O(α(n)) amortized — practically constant.
- **Without either:** O(n) worst case.

### When NOT Union-Find

- When you need to *split* sets (Union-Find can't undo). For that, use **Link-Cut Trees** (heavier, complex).
- When you need set operations beyond union (intersection, etc.).

---

## Worked Example — Identity Resolution Pipeline

Real DE problem: collapse user records that match by email, phone, or device ID into single identities.

```python
import hashlib

def normalize(s):
    return s.strip().lower() if s else None

def hash_id(s):
    return hashlib.sha256(s.encode()).hexdigest()[:16]

def collapse_identities(records):
    """Each record has: email, phone, device_id."""
    uf = UnionFind(len(records))
    by_email = {}; by_phone = {}; by_device = {}

    for i, r in enumerate(records):
        for table, key in [(by_email, normalize(r.email)),
                           (by_phone, normalize(r.phone)),
                           (by_device, r.device_id)]:
            if key is None: continue
            if key in table:
                uf.union(i, table[key])
            else:
                table[key] = i

    # Group records by root
    groups = {}
    for i in range(len(records)):
        groups.setdefault(uf.find(i), []).append(records[i])
    return list(groups.values())
```

This is the core of every identity-resolution / master-data-management pipeline. Splink, Zingg, and most CRM dedup tools build similar union-find structures over scoring functions. See [Project A1](../projects/README.md#a1-multi-source-identity-resolution-service).

---

## Common Interview Gotchas

**Q: Why are tries O(k) not O(log n)?**
A: Each step descends one node per character; n stored strings doesn't change the path length.

**Q: When do you choose a trie over a hash set?**
A: When you need **prefix queries** ("words starting with 'pre'"). Hash sets can't help. Also when many keys share long prefixes — trie shares storage.

**Q: How does a tokenizer use a trie?**
A: For BPE/WordPiece, build a trie of token byte sequences. At encode time, greedy longest-match: walk the trie consuming as many bytes as possible.

**Q: Why is Aho-Corasick faster than running KMP per pattern?**
A: AC scans the text once for *all* patterns in O(n + total pattern length + matches). KMP would scan once per pattern.

**Q: What's the complexity of Union-Find with both optimizations?**
A: O(α(n)) amortized per op, where α is inverse Ackermann. Treat as O(1) for any practical n.

**Q: When does Union-Find fail you?**
A: When you need to undo unions (use Link-Cut Trees) or do set intersection (Union-Find doesn't track members efficiently).

**Q: How would you find connected components in a graph with 1B edges?**
A: Distributed Union-Find (each worker keeps a partial DSU; merge across workers in stages) or Spark GraphFrames `connectedComponents`.

**Q: What does Python's `re` module use under the hood?**
A: A backtracking NFA simulation. Vulnerable to ReDoS on adversarial regex. For safe O(n) matching, use `re2` package.

---

## Interview-Ready Cheat Sheet

### Trie

- **O(k) for everything**, where k = key length.
- Best for prefix queries, autocomplete, BPE tokenizers, IP routing.
- Variants: compressed trie (radix), suffix tree, suffix array, DAWG.

### String matching

- Single pattern: KMP O(n+m), Boyer-Moore sublinear in practice.
- Multi-pattern: Aho-Corasick O(n+m+matches).
- Beware Python's `re` for adversarial input — use `re2`.

### Union-Find

- O(α(n)) ≈ O(1) with path compression + union by rank.
- Used in Kruskal MST, connected components, identity resolution.
- Cannot undo unions.

### Trade-off pairs

- **Trie vs hash set:** prefix support vs O(1) point lookup.
- **Trie vs suffix array:** insertions vs memory/scan speed.
- **KMP vs Boyer-Moore:** worst-case vs average-case.
- **Aho-Corasick vs Bloom prefilter + verify:** exact match vs probabilistic+lookup.
- **Union-Find vs Link-Cut Tree:** static unions vs supporting splits.

---

## Resources & Links

- *CLRS* ch. 21 (Union-Find), 32 (string matching).
- [Aho-Corasick paper, 1975](https://cr.yp.to/bib/1975/aho.pdf) — original.
- [Tries — Stanford lecture notes](https://web.stanford.edu/class/cs166/lectures/12/Slides12.pdf)
- [Hugging Face Tokenizers — BPE in production](https://huggingface.co/docs/tokenizers/index)
- [Splink (record linkage) docs](https://moj-analytical-services.github.io/splink/)
- [RE2 — re2 Python bindings](https://github.com/google/re2)
- [Hyperscan — Intel's regex engine](https://github.com/intel/hyperscan)
- [BMSSP (2025)](https://arxiv.org/abs/2504.17033) — relies on amortized data-structure tricks akin to UF.

### Companion Files
- [Hash Tables](./06-hash-tables.md) — alternative when you don't need prefix queries.
- [Graphs & Algorithms](./10-graphs-and-algorithms.md) — Kruskal uses Union-Find.
- [Modern Advances](./14-modern-advances.md) — modern algorithms that build on these.
- [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md) — BPE tokenizer tries.
- [Project A1 — Identity Resolution](../projects/README.md) — Union-Find in practice.

---

*Next: [Probabilistic Data Structures](./12-probabilistic-structures.md).*
