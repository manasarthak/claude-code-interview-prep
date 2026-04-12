# LLMs — Large Language Models Internals

**Phase:** 1 (Foundations)  
**Difficulty progression:** Beginner → Intermediate → Advanced  
**Last updated:** April 12, 2026

---

## BEGINNER — What Are LLMs and How Do They Work?

### The One-Sentence Version

An LLM is a neural network trained to predict the next word (token) in a sequence, and it turns out that getting extremely good at this task produces something that can reason, write code, summarize, translate, and more.

### How Training Works (Simplified)

1. **Gather massive text data** — books, websites, code, papers (trillions of tokens)
2. **Pre-training** — The model learns to predict the next token. Given "The cat sat on the ___", it learns "mat" is more likely than "quantum." This stage is where the model absorbs world knowledge.
3. **Fine-tuning / Instruction tuning** — Train the model on question-answer pairs and instructions so it follows directions instead of just completing text.
4. **Alignment** — RLHF (Reinforcement Learning from Human Feedback) or DPO (Direct Preference Optimization) to make the model helpful, harmless, and honest.

### Tokens, Not Words

LLMs don't see words — they see **tokens**. Tokenization breaks text into subword pieces.

- "unhappiness" → ["un", "happiness"] or ["un", "hap", "pi", "ness"]
- The average English word ≈ 1.3 tokens
- Code, non-English languages, and rare words use more tokens
- Why this matters: context window limits, API pricing, and generation speed are all measured in tokens

### Key Parameters

- **Parameters** — The learned weights in the neural network. GPT-4 is rumored at ~1.8T, Llama 3 comes in 8B/70B/405B variants.
- **Context window** — Max tokens the model can see at once (input + output). Claude: 200K, GPT-4: 128K, Gemini: up to 1M+.
- **Temperature** — Controls randomness. 0 = deterministic (always pick most likely token), 1 = creative, >1 = chaotic.
- **Top-p (nucleus sampling)** — Only consider tokens whose cumulative probability reaches p. Top-p 0.9 means ignore the bottom 10% of unlikely tokens.

### The Model Landscape (2026)

**Closed/proprietary:**
- OpenAI: GPT-4o, GPT-4 Turbo, o1/o3 (reasoning models)
- Anthropic: Claude Opus 4.6, Sonnet 4.6, Haiku 4.5
- Google: Gemini 2.x series (Ultra, Pro, Flash)

**Open-weight:**
- Meta: Llama 3.x (8B, 70B, 405B)
- Mistral: Mistral, Mixtral (MoE), Mistral Large
- DeepSeek: DeepSeek-V2, DeepSeek-Coder
- Qwen (Alibaba): Qwen 2.x series
- Cohere: Command R+

**Why this matters:** Open vs. closed affects cost, customizability, data privacy, and deployment options. You need to know when to recommend each.

---

## INTERMEDIATE — The Transformer Architecture

### Why Transformers Dominate

Before transformers (2017), NLP used RNNs and LSTMs that processed tokens sequentially — slow and struggled with long-range dependencies. The transformer processes all tokens in parallel using **attention**, making it faster and better at capturing relationships across long text.

### Architecture Overview

A transformer has two main components (though modern LLMs use only the decoder):

```
Input tokens
    ↓
[Token Embedding + Positional Encoding]
    ↓
[Attention Layer]  ← This is the key innovation
    ↓
[Feed-Forward Network]
    ↓
(Repeat N times — these are the "layers")
    ↓
[Output projection → vocabulary probabilities]
    ↓
Next token prediction
```

### Self-Attention — The Core Mechanism

Attention answers: "When processing this token, how much should I look at each other token?"

For each token, the model computes three vectors:
- **Query (Q)** — "What am I looking for?"
- **Key (K)** — "What do I contain?"
- **Value (V)** — "What information do I carry?"

The attention score between tokens is:

```
Attention(Q, K, V) = softmax(QK^T / √d_k) × V
```

- `QK^T` — Dot product between query and all keys (how relevant is each token?)
- `√d_k` — Scaling factor to prevent huge values that make softmax extreme
- `softmax` — Normalize scores to sum to 1 (attention weights)
- `× V` — Weighted sum of value vectors

**Example intuition:** In "The cat sat on the mat because it was tired," when processing "it," the attention mechanism learns to put high weight on "cat" (understanding what "it" refers to).

### Multi-Head Attention

Instead of one set of Q, K, V — use multiple "heads" (e.g., 32 or 64), each learning different relationships. One head might track syntax, another coreference, another semantic similarity. Their outputs are concatenated and projected.

### Positional Encoding

Since attention processes all tokens in parallel, the model has no inherent sense of order. Positional encodings add position information.

**Original approach:** Sinusoidal functions  
**Modern approach:** RoPE (Rotary Position Embeddings) — encodes relative positions, better for long sequences, used in Llama, Mistral, etc.

### Layer Normalization & Residual Connections

Each sub-layer (attention, feed-forward) is wrapped with:
```
output = LayerNorm(x + SubLayer(x))
```

- **Residual connection** (`x + SubLayer(x)`) — Prevents gradient vanishing, lets information flow directly through the network
- **LayerNorm** — Stabilizes training by normalizing activations

Modern LLMs use **pre-norm** (normalize before the sub-layer) rather than post-norm, which is more stable for deep networks.

### Feed-Forward Network (FFN)

After attention, each token passes through a 2-layer neural network independently:

```
FFN(x) = activation(xW₁ + b₁)W₂ + b₂
```

This is where the model stores "knowledge." Research suggests attention layers handle routing/reasoning while FFN layers act as key-value memory for facts.

**Modern variants:**
- **SwiGLU** — Used in Llama, more expressive activation function
- **MoE (Mixture of Experts)** — Only activate a subset of FFN parameters per token (e.g., Mixtral activates 2 of 8 experts), giving massive capacity with less compute

### The Full GPT-Style Architecture

Modern LLMs are **decoder-only transformers**:
- No separate encoder (unlike the original paper)
- **Causal masking** — Each token can only attend to tokens before it (not future tokens), which is what makes it autoregressive
- Stack 32–96+ transformer blocks deep
- Vocabulary typically 32K–128K tokens

---

## ADVANCED — Inference, Optimization, and Scaling

### KV Cache — Why Inference is Memory-Bound

During generation, each new token needs to attend to all previous tokens. Recomputing Q, K, V for the entire sequence every step would be absurdly slow.

**Solution: KV Cache** — Store the K and V vectors for all previous tokens. For each new token, only compute its Q, K, V, then attend against the cached K, V values.

**Trade-off:** KV cache uses a lot of memory. For a 70B model with 128K context, the KV cache alone can use 40+ GB of GPU memory. This is often the bottleneck in production, not compute.

**Optimizations:**
- **Multi-Query Attention (MQA)** — All attention heads share one K and V (huge memory savings, small quality cost)
- **Grouped-Query Attention (GQA)** — Groups of heads share K, V (middle ground). Used in Llama 3, Mistral.
- **Sliding window attention** — Only attend to the last W tokens (Mistral uses 4096). Reduces cache size linearly.

### Quantization

Full-precision models use float16 (2 bytes per parameter). A 70B model = 140 GB — doesn't fit on consumer hardware.

**Quantization** reduces precision:
- **INT8** — 1 byte per param → 70 GB for 70B model
- **INT4 (GPTQ, AWQ)** — 0.5 bytes per param → 35 GB for 70B model
- **GGUF (llama.cpp)** — Various quantization levels (Q4_K_M, Q5_K_M), optimized for CPU+GPU inference

**Quality impact:** INT8 is nearly lossless. INT4 introduces some degradation, especially on reasoning tasks. The sweet spot depends on your use case.

### Speculative Decoding

LLM generation is slow because it's sequential — one token at a time. Speculative decoding uses a small "draft" model to generate N candidate tokens cheaply, then the large model verifies them all in one forward pass (parallel).

If the draft model guessed correctly (common for routine tokens), you get N tokens for the cost of ~1 large model forward pass. Typical speedup: 2–3x.

### Scaling Laws

**Chinchilla scaling laws** (2022) showed that for a fixed compute budget, you should scale data and parameters roughly equally. Previously, people built huge models with too little data.

Key implications:
- A 10B model trained on 200B tokens can outperform a 100B model trained on 20B tokens
- Data quality and quantity are as important as model size
- This is why Llama 3 8B outperforms much larger older models

### Inference Serving in Production

| Technique | What It Does |
|-----------|-------------|
| **Continuous batching** | Don't wait for all requests to finish — slot new requests into freed positions. Dramatically improves throughput. |
| **PagedAttention (vLLM)** | Manage KV cache like virtual memory pages — reduces waste from pre-allocation. Foundation of vLLM. |
| **Tensor parallelism** | Split model layers across GPUs for models too large for one GPU. |
| **Pipeline parallelism** | Different layers on different GPUs, pass activations between them. |
| **Flash Attention** | Fused CUDA kernel that computes attention with O(N) memory instead of O(N²). Now standard. |

**Key tools:** vLLM, TensorRT-LLM (NVIDIA), TGI (HuggingFace), Triton Inference Server

### Reasoning Models — A New Paradigm

Models like OpenAI's o1/o3 and DeepSeek-R1 represent a shift: instead of answering immediately, they "think" with chain-of-thought internally, using more compute at inference time for harder problems.

This introduces a new scaling axis: **inference-time compute**. Instead of only scaling training, you can throw more compute at each individual query. This matters for math, coding, and complex reasoning.

### The GPU Landscape (Brief)

| GPU | VRAM | Good For | Notes |
|-----|------|----------|-------|
| A100 | 40/80 GB | Training and inference | Still workhorse of most cloud providers |
| H100 | 80 GB | Large-scale training | 2–3x faster than A100 for transformers |
| L40S | 48 GB | Inference | Cost-effective for serving |
| Consumer (4090) | 24 GB | Small models, experimentation | Great for quantized 7–13B models |

---

## Interview-Ready Cheat Sheet

**"Explain how a transformer works"** — Token embeddings + positional encoding → multi-head self-attention (Q, K, V with causal masking) → feed-forward network → repeat N layers → project to vocabulary → sample next token. Residual connections and layer norm throughout.

**"What is KV cache and why does it matter?"** — Stores key/value vectors from previous tokens during generation to avoid recomputation. It's the main memory bottleneck in inference. GQA and MQA reduce its size.

**"How would you serve a 70B model?"** — Quantize to INT4/INT8 to reduce memory, use vLLM with PagedAttention for efficient batching, tensor parallelism across multiple GPUs, Flash Attention for the attention kernel. Add semantic caching for cost reduction.

**"Open vs. closed models — when would you choose which?"** — Closed (GPT-4, Claude) for fastest time-to-market, highest quality, and when data doesn't leave the org's API contract. Open (Llama, Mistral) when you need fine-tuning control, on-prem deployment, cost optimization at scale, or data sovereignty.

**Quick trade-off pairs:**
- Parameters vs. data: Chinchilla laws — balance both
- Precision vs. speed: INT4 is 2x faster, slight quality cost
- Context length vs. cost: Longer context = more tokens billed
- Quality vs. latency: Bigger models are better but slower
- Batching vs. latency: Bigger batches improve throughput but individual latency rises

---

*Next: Add notes on specific models as you study them, papers you read, and experiments you run.*

## Resources & Links
_(Add as you go)_
