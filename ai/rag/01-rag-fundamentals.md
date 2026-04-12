# RAG — Retrieval-Augmented Generation

**Phase:** 1 (Foundations)  
**Difficulty progression:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — What Is RAG and Why Does It Exist?

### The Problem RAG Solves

LLMs are trained on a fixed snapshot of data. They can't access your company's internal docs, recent events, or private knowledge. When asked about something outside their training data, they either refuse or hallucinate (confidently make things up).

You have two choices: retrain/fine-tune the model on your data (expensive, slow, goes stale) or give the model relevant context at inference time. RAG does the second thing.

### The Core Idea

RAG = **Retrieve** relevant documents + **Augment** the prompt with them + **Generate** a response grounded in that context.

Think of it like an open-book exam. Instead of expecting the LLM to memorize everything, you hand it the right pages before asking the question.

### Simplest RAG Pipeline

```
User Question
    ↓
[1] Embed the question into a vector
    ↓
[2] Search a vector database for similar document chunks
    ↓
[3] Retrieve top-K most relevant chunks
    ↓
[4] Stuff them into the LLM prompt as context
    ↓
[5] LLM generates an answer grounded in those chunks
```

### Key Vocabulary

- **Embedding** — A numerical vector (list of numbers) that represents the meaning of a piece of text. Similar meanings produce similar vectors.
- **Vector database** — A specialized database optimized for storing and searching embeddings by similarity (not exact match).
- **Chunk** — A piece of a larger document, broken up so it fits in the LLM's context window and is granular enough for precise retrieval.
- **Top-K retrieval** — Returning the K most similar results from the vector database.
- **Context window** — The maximum amount of text an LLM can process in one call (e.g., 128K tokens for GPT-4, 200K for Claude).
- **Hallucination** — When the model generates plausible-sounding but incorrect information.

### When to Use RAG

- You need the model to answer questions about specific documents or data
- Your knowledge base changes frequently (news, docs, tickets)
- You need citations / source attribution
- You can't afford to fine-tune for every knowledge update
- You want to reduce hallucinations by grounding responses

### When RAG Is NOT the Answer

- The task doesn't require external knowledge (e.g., summarizing a given text)
- The knowledge fits entirely within the prompt already
- You need the model to learn a new behavior or style (that's fine-tuning)
- Latency requirements are extreme and you can't afford the retrieval step

---

## INTERMEDIATE — Building a Real RAG Pipeline

### Step 1: Document Ingestion & Chunking

Before you can retrieve anything, you need to break your documents into chunks and embed them.

**Chunking strategies:**

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| Fixed-size | Split every N characters/tokens with overlap | Simple, predictable | Cuts mid-sentence, ignores structure |
| Recursive | Split by paragraph → sentence → character, respecting boundaries | Preserves structure | More complex to implement |
| Semantic | Use embeddings to detect topic shifts, split at boundaries | Best coherence | Slowest, needs tuning |
| Document-aware | Use headings, sections, markdown structure | Great for structured docs | Doesn't work on unstructured text |
| Parent-child | Small chunks for retrieval, return the larger parent for context | Precise retrieval + full context | More storage, complex indexing |

**Chunk size trade-offs:**
- Too small (100 tokens): loses context, retrieves fragments
- Too large (2000 tokens): dilutes relevance, wastes context window
- Sweet spot for most use cases: 256–512 tokens with 10–20% overlap

**Overlap matters.** If you chunk without overlap, you can split a critical fact across two chunks and lose it in retrieval. A 50–100 token overlap prevents this.

### Step 2: Embedding Models

The embedding model converts text into vectors. This is the foundation of retrieval quality.

**Popular choices (2026):**

| Model | Dimensions | Strengths | Notes |
|-------|-----------|-----------|-------|
| OpenAI text-embedding-3-large | 3072 | Strong general-purpose, easy API | Proprietary, costs per token |
| Cohere embed-v3 | 1024 | Excellent multilingual, supports search types | Proprietary |
| BGE-large-en-v1.5 | 1024 | Best open-source English | Self-hostable |
| E5-mistral-7b-instruct | 4096 | Instruction-tuned, high quality | Large model, slower |
| Nomic embed-text | 768 | Good quality, open, affordable | Newer entrant |

**Key concept — asymmetric search:** Questions are short, documents are long. Some models (like E5) are trained to handle this asymmetry. Others perform better when you add a prefix like "search_query:" or "search_document:" to differentiate.

**Embedding the query and the documents must use the same model.** You can't embed docs with OpenAI and queries with Cohere — the vector spaces don't align.

### Step 3: Vector Databases

Where you store and search your embeddings.

| Database | Type | Best For | Notes |
|----------|------|----------|-------|
| Pinecone | Managed cloud | Production, low-ops teams | Fully managed, scales well |
| Weaviate | Self-hosted / cloud | Hybrid search (vector + keyword) | GraphQL API, modules |
| Qdrant | Self-hosted / cloud | Performance-sensitive apps | Rust-based, fast |
| Chroma | In-process | Prototyping, small datasets | Easy setup, embeds in Python |
| pgvector | Postgres extension | Teams already on Postgres | No new infra, good enough for many |
| Milvus | Self-hosted | Large-scale, high throughput | Complex to operate |

**Choosing a vector DB** comes down to: managed vs self-hosted, scale requirements, whether you need hybrid search, and what your team already runs.

### Step 4: Retrieval Methods

**Dense retrieval** — Embed query and docs, find nearest neighbors by cosine similarity. Good at semantic matching ("How do I cancel?" matches "subscription termination policy").

**Sparse retrieval (BM25/TF-IDF)** — Traditional keyword matching. Good at exact term matching ("error code ERR_4032" matches documents containing that exact string).

**Hybrid search** — Combine dense + sparse with a weighted score. Best of both worlds. Most production systems use this.

```
final_score = α * dense_score + (1 - α) * sparse_score
```

α is typically 0.5–0.7, tuned on your data.

**Reranking** — After initial retrieval (top 20–50), run a cross-encoder model to re-score and re-order results. Cross-encoders are more accurate than bi-encoders but too slow to run on the entire corpus, so you use them as a second pass.

Popular rerankers: Cohere Rerank, cross-encoder models from sentence-transformers, ColBERT.

**Retrieval pipeline in practice:**
```
Query → BM25 (top 50) + Dense search (top 50)
    ↓
Deduplicate & merge
    ↓
Reranker scores all candidates
    ↓
Top 5-10 → LLM prompt
```

### Step 5: Prompt Construction

Once you have relevant chunks, you build the prompt:

```
System: You are a helpful assistant. Answer the user's question using 
ONLY the provided context. If the context doesn't contain the answer, 
say "I don't have enough information to answer this."

Context:
[chunk 1]
[chunk 2]
[chunk 3]

User: {original question}
```

**Key decisions:**
- How many chunks to include (balance relevance vs. context window cost)
- Whether to include chunk metadata (source, date, section heading)
- Whether to ask the model to cite its sources
- How to handle conflicting information across chunks

### Step 6: Evaluation (Critical and Often Skipped)

You can't improve what you don't measure. RAG evaluation has distinct components:

| Metric | What It Measures | How |
|--------|-----------------|-----|
| **Context Relevance** | Did retrieval find the right chunks? | Score retrieved chunks against the query |
| **Faithfulness** | Does the answer stick to the context? (no hallucination) | Check if every claim in the answer is supported by a chunk |
| **Answer Relevance** | Does the answer actually address the question? | Score answer against original query |
| **Answer Correctness** | Is the answer factually right? | Compare against ground truth (if available) |

**RAGAS** is the standard framework for this — it computes these metrics automatically using an LLM as a judge.

---

## ADVANCED — Production Patterns and Hard Problems

### Agentic RAG

Instead of a fixed retrieve-then-generate pipeline, an agent decides whether and how to retrieve:

```
User question
    ↓
Agent decides: Do I need retrieval?
    ↓ Yes
Agent decides: Which data source? What query?
    ↓
Retrieve → Check relevance → Maybe re-query with refined terms
    ↓
Agent decides: Do I have enough? Or query another source?
    ↓
Generate answer with full gathered context
```

This handles multi-hop questions ("What was the revenue impact of the feature we launched in Q2?") where you need to retrieve from multiple sources and chain the reasoning.

### Multi-Index / Routed RAG

Production systems rarely have one knowledge base. You might have:
- Product documentation (structured, versioned)
- Support tickets (semi-structured, temporal)
- Slack conversations (unstructured, noisy)
- Code repositories (structured, cross-referenced)

A **router** classifies the query and directs it to the right index (or multiple indexes), each with its own chunking strategy and embedding model optimized for that data type.

### Adaptive Chunking & Hierarchical Retrieval

**Problem:** A question like "Summarize our refund policy" needs broad context, while "What's the refund deadline for Enterprise tier?" needs a specific detail.

**Solution — hierarchical indexing:**
- Level 1: Document summaries (for broad questions)
- Level 2: Section summaries (for moderate specificity)
- Level 3: Fine-grained chunks (for specific facts)

The system classifies query specificity and retrieves at the appropriate level.

### Long-Context Models vs. RAG

With models supporting 128K–1M+ tokens, some people ask: "Why not just stuff everything in the context?"

**When long context wins:**
- Small corpus (<100 pages)
- Need for cross-document reasoning
- Simplicity is paramount

**When RAG still wins:**
- Large corpus (thousands of documents)
- Cost matters (long-context = many tokens = expensive)
- You need to update knowledge without re-processing everything
- You need to cite specific sources
- Latency matters (long prompts are slower)

**The real answer:** They complement each other. Use RAG to select the right ~10 pages, then use long context to reason over them thoroughly.

### Query Transformation Techniques

Real user queries are often bad retrieval queries. Transform them before searching:

- **HyDE (Hypothetical Document Embedding)** — Ask the LLM to generate a hypothetical answer, then embed *that* for retrieval. The hypothetical answer is closer to document language than the question.
- **Query decomposition** — Break complex questions into sub-queries, retrieve for each.
- **Step-back prompting** — Rephrase specific questions into more general ones to broaden retrieval, then use the broader context to answer the specific question.

### Semantic Caching

If users ask semantically similar questions, cache the response and skip the LLM call entirely. Hash the embedding of the query, check cache for near-matches, serve cached answer if similarity > threshold. This can cut LLM costs by 20–40% in production.

### Failure Modes to Study

| Failure | Cause | Mitigation |
|---------|-------|------------|
| Retrieved wrong chunks | Poor embedding model or chunking | Better chunking, reranking, hybrid search |
| Right chunks, wrong answer | LLM hallucinating despite context | Faithfulness checks, constrained decoding |
| Missing information | Document not in index, or poor chunking split it | Ingestion audits, overlap, parent-child chunking |
| Stale answers | Index not updated | Incremental indexing, TTL on chunks |
| Slow responses | Large index, complex reranking | Approximate nearest neighbors, caching, filtering |
| Context window overflow | Too many chunks retrieved | Better relevance filtering, compression, summarization |

---

## Interview-Ready Cheat Sheet

**If asked "Design a RAG system," walk through:**
1. Data ingestion pipeline — sources, parsing, chunking strategy with justification
2. Embedding model selection — trade-offs on cost, quality, latency
3. Vector DB choice — why this one for this use case
4. Retrieval strategy — dense, sparse, hybrid, reranking
5. Prompt construction — grounding instructions, citation format
6. Evaluation — metrics (RAGAS), ground truth sets, monitoring
7. Production concerns — caching, scaling, latency, cost, staleness
8. When the LLM is only 20% of the system — orchestration, guardrails, observability

**Quick trade-off pairs to have ready:**
- Chunk size: precision vs. context
- Top-K: recall vs. noise
- Dense vs. sparse: semantic vs. lexical matching
- Reranking: quality vs. latency
- Long context vs. RAG: simplicity vs. cost/scale

---

*Next: Add hands-on notes as you build. Link to papers and blog posts you find valuable below.*

## Resources & Links
_(Add as you go)_
