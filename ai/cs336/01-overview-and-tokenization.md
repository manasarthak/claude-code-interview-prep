# CS336 · 01 — Foundations & Tokenization

> Concept-only notes. Math is unpacked so a beginner can follow. Wherever a size or token or compute number appears, we ground it in a concrete sentence and compare it to a currently-shipped model (GPT-4o, Claude Sonnet 4.6, Llama 3, DeepSeek V3, Qwen 2.5) so the number stops being abstract.
> **Math rendering policy for this folder:** inline math is Unicode (α β θ ρ ² ⁻¹ Σ Π ∝ ≈ ≤ ≥) so it renders in every viewer — GitHub, VS Code, macOS Preview, Notion, iOS Notes. Display equations use `$$…$$` blocks on their own line with blank lines around, which render in GitHub natively and in any Markdown+Math viewer. Inside math, we never write raw `*` — GitHub's markdown parser eats it as an italic marker before KaTeX sees it and you get *"Missing open brace for superscript"*. Instead we write `\ast` (or brace-wrap: `^{*}`). Same for stray `_` outside a subscript `{}`.
> Companion source: [lecture_01.py](https://github.com/stanford-cs336/spring2025-lectures/blob/main/lecture_01.py) · [Karpathy tokenization video](https://www.youtube.com/watch?v=zduSFxRajkE)

---

## 1. Language modeling as a probability problem

A **language model** is a probability distribution over sequences of tokens **x = (x₁, x₂, …, x_L)** drawn from a fixed vocabulary **V**. Any joint distribution factors by the chain rule of probability:

$$
P_\theta(x) = \prod_{t=1}^{L} P_\theta(x_t \mid x_1, \ldots, x_{t-1})
$$

That factorization is exact — no approximation, no independence assumption. The whole engineering effort of the field is: build a neural network **P_θ** that assigns high probability to real text and low probability to nonsense.

Training minimizes the average negative log-likelihood per token on a corpus **D**. This quantity is called **cross-entropy loss** — the training objective, measured in nats (natural log) or bits (log base 2). Every big number you'll hear about LMs — "loss went from 2.4 to 2.2," "3.8 bits per byte" — is a version of it.

$$
\mathcal{L}(\theta) = -\frac{1}{|\mathcal{D}|} \sum_{x \in \mathcal{D}} \frac{1}{L_x} \sum_{t=1}^{L_x} \log P_\theta(x_t \mid x_1, \ldots, x_{t-1})
$$

### Why cross-entropy is exactly Shannon's entropy in disguise

Shannon (1948) defined the **entropy** of a source distribution **P\*** as the expected code length of an optimal encoding — the irreducible information content:

$$
H(P^{\ast}) = -\mathbb{E}_{x \sim P^{\ast}}[\log P^{\ast}(x)]
$$

Cross-entropy of a model **P_θ** against the true source decomposes into that entropy plus a slack term called the **Kullback-Leibler divergence** — the excess bits you pay by encoding with the wrong distribution:

$$
H(P^{\ast}, P_\theta) = -\mathbb{E}_{x \sim P^{\ast}}[\log P_\theta(x)] = H(P^{\ast}) + \mathrm{KL}(P^{\ast} \,\|\, P_\theta)
$$

So minimizing cross-entropy is the same as minimizing KL divergence from the model to the true text distribution. The floor is H(P\*), the entropy of language itself. This is why better data and bigger models keep helping — you're always above the Shannon floor, never at it.

### Perplexity — cross-entropy for people who prefer positive numbers

**Perplexity** (PPL) is just cross-entropy exponentiated:

$$
\mathrm{PPL} = \exp(\mathcal{L})
$$

Interpretation: perplexity is the effective vocabulary size the model is "hesitating between" at each step. Uniform guessing over |V| tokens has perplexity |V|. A perfect model has perplexity 1.

**Concrete example.** Take the prompt `"I made a cup of ___"`. A well-trained LM assigns most of its mass to a short list: `coffee, tea, hot chocolate, cocoa, water, soup, espresso, herbal tea, decaf, joe`. That's roughly 10 highly plausible next tokens with somewhat unequal probability, and a long tail of rarer ones (`chowder, ambition, kindness`). If perplexity on the held-out corpus is 10, the model is behaving *as if* it's essentially choosing among 10 near-equal candidates at each step on average. In practice, modern models sit lower than you might expect: Claude Sonnet 4.6 and GPT-4-class models on ordinary English land in the single digits (roughly 6–12 depending on the domain); GPT-3 in 2020 sat around 20 on the same style of corpus. Every ~20% improvement in perplexity corresponds to roughly one generation of models.

**A warning that comes up constantly:** perplexity depends on the tokenizer. A model with a bigger vocabulary produces fewer tokens per document and each token carries more information — its perplexity is not directly comparable to a model with a smaller vocab. The tokenizer-independent variant is **bits per byte (BPB)**, which normalizes cross-entropy to source-text bytes:

$$
\mathrm{BPB} = \frac{\mathcal{L}_{\text{nats}}}{\ln 2} \cdot \frac{\text{tokens}}{\text{bytes}}
$$

Always convert to BPB when comparing across tokenizers. A frontier LM on English lands around 0.7–0.9 BPB — meaning it can compress English text to under a bit per byte, well below what gzip achieves (~2 BPB) and getting close to Shannon's estimated entropy of English (~1 BPB).

---

## 2. Compute as a physical quantity: the FLOP

A **FLOP** is one floating-point multiply-add. Every matmul, softmax, and layernorm reduces to some count of FLOPs.

**Fact 1 — a single forward pass of a Transformer costs about 2N FLOPs per token,** where N is the parameter count. A backward pass costs about 4N. So a full training step is roughly **6N FLOPs per token**.

*Derivation sketch:* a linear layer with a **d_in × d_out** weight matrix, applied to a vector of size **d_in**, is **2 · d_in · d_out** FLOPs (multiply + accumulate). Sum over all layers gives ~2N per token forward. Backprop touches each parameter twice more.

**Fact 2 — total training compute is**

$$
C \approx 6 \cdot N \cdot D
$$

where **D** is total training tokens seen. This equation shows up in every scaling-law plot.

**Concrete example — Llama 3 405B.** Meta reports N = 405 × 10⁹ parameters, D = 15.6 × 10¹² tokens. Plug in:

$$
C \approx 6 \times 405{\times}10^9 \times 15.6{\times}10^{12} \approx 3.8 \times 10^{25}\ \text{FLOPs}
$$

That's the "compute budget" number Meta quotes for the run. Divide by an H100's ~1 × 10¹⁵ bf16 FLOPs/sec and ~40% real-world utilization (the rest is memory stalls, comms, checkpointing), and you get ~9.5 × 10¹⁷ seconds of single-H100 time — about 30,000 H100-years. Meta ran it on ~16,000 H100s for ~54 days. Every training-compute headline you see is a version of this arithmetic.

Frontier scale for context: a single H100 does roughly **10¹⁵ FLOPs/sec** of bf16 matmul. Frontier training runs today land in the **10²⁵ to 10²⁶ FLOP** range.

---

## 3. Scale-dependent behavior

Not everything you can measure in a 100 M-parameter model tells you what a 100 B-parameter model will do. Two mathematical reasons.

### 3.1 FLOP allocation shifts with scale

For a Transformer with embedding dimension **d** and sequence length **L**, per-layer FLOPs split roughly:

- **MLP:** proportional to L · d²    (two linear layers, each roughly d × 4d)
- **Attention:** proportional to L² · d    (from the QKᵀ score matrix)

The **crossover** where these are equal is at **L ≈ d**.

**Concrete numbers.**
- **GPT-2 small:** d = 768, typical L = 1024. Since L > d only slightly, attention and MLP costs are comparable, with a mild MLP tilt.
- **Llama 3 70B:** d = 8192, typical training L = 8192. L ≈ d, so per-layer costs are roughly balanced.
- **Llama 3 70B in long-context mode:** L = 128,000, d = 8192. Now L ≫ d and **attention costs dominate by ~16×** — which is why the industry sprints toward flash-attention, grouped-query attention, and sliding-window attention specifically for long-context regimes.

So an "attention optimization" that gave you a 20% training speedup on GPT-2 might give you a 2× speedup on Llama 3 at 128k context — or ~nothing on Llama 3 at 8k. Efficiency work has to know its regime.

### 3.2 Emergence and the power-law lens

Kaplan et al. (2020) fit loss as a power law along each axis independently:

$$
\mathcal{L}(N) \approx A \cdot N^{-\alpha_N}, \quad
\mathcal{L}(D) \approx B \cdot D^{-\alpha_D}, \quad
\mathcal{L}(C) \approx E \cdot C^{-\alpha_C}
$$

with empirical exponents around α_N ≈ 0.076 and α_D ≈ 0.095. Small numbers, but the plots span many orders of magnitude, so the effect is large. On a log-log scale a power law is a straight line — and it's shockingly straight across seven orders of magnitude of compute for pretraining loss. These smooth fits are what people mean by a **scaling law** — a power-law relationship linking a training resource to expected loss.

**Emergence** — a capability appearing sharply past a scale threshold — is when a *downstream metric* looks discontinuous even though the underlying loss is smooth. Suppose task accuracy is roughly a threshold on some latent quantity that follows a power law in N: as soon as the latent crosses the threshold, accuracy jumps from near-zero to near-one. The loss curve is still smooth; the metric isn't.

**Concrete example.** Three-digit arithmetic. At GPT-3 6.7B, accuracy on "132 + 891 = ?" is essentially 0%. At GPT-3 175B, it's ~80%. The pretraining loss curve between those two model sizes shows nothing dramatic — a smooth power-law decrease. What jumped was the downstream metric. Practical consequence: some things you cannot study at 500 M scale — they haven't crossed the threshold yet.

---

## 4. The Bitter Lesson — with the actual arithmetic

Rich Sutton's argument, in one equation:

$$
\text{capability} \;\propto\; \text{efficiency} \times \text{compute}
$$

Half a decade of algorithmic progress can multiply the efficiency coefficient by 10–50× without touching the hardware. Between 2012 and 2019, algorithmic progress alone gave a **44× reduction** in the compute needed to reach fixed accuracy on ImageNet (Hernandez & Brown, 2020).

If you take that seriously, algorithmic efficiency has an *effective doubling time* on the order of a year — comparable to or faster than Moore's Law. The naïve reading "scale is all that matters" is exactly wrong: efficiency scaling *is* one of the compute-yields. What loses is cleverness that doesn't scale — hand-coded rules, small architectural specializations, features engineered by domain experts.

The engineering directive: for a fixed budget C, don't ask "how big can I go?" — ask "what recipe extracts the most capability per FLOP?"

---

## 5. Chinchilla scaling: the compute-optimal ratio

Kaplan's laws tell you loss as a function of N or D alone. But you have to pick both, subject to a compute budget C ≈ 6ND. That's the constrained optimization Chinchilla (Hoffmann et al., 2022) solves.

Model the loss as

$$
\mathcal{L}(N, D) = \underbrace{E}_{\text{irreducible}} + \frac{A}{N^{\alpha}} + \frac{B}{D^{\beta}}
$$

Minimize over (N, D) subject to **6ND ≤ C**. A Lagrangian gives **N\* ∝ Cᵃ**, **D\* ∝ Cᵇ** with a + b = 1. The Chinchilla fits give exponents α ≈ 0.34, β ≈ 0.28, from which

$$
N^{\ast} \propto C^{0.5}, \qquad D^{\ast} \propto C^{0.5}
$$

and empirically the ratio settles at

$$
\boxed{\; D^{\ast} \approx 20 \cdot N^{\ast} \;}
$$

**In words:** for every parameter of model size, use about 20 tokens of training data. A 7 B model *at the compute-optimal point* should see ~140 B tokens.

**But nobody actually trains at the compute-optimal point anymore.** Because *inference* also costs compute — a smaller model over-trained on more tokens is cheaper to serve for the rest of its life. So production models blow past Chinchilla:

| Model | N (params) | D (train tokens) | D/N ratio | vs. Chinchilla |
| --- | --- | --- | --- | --- |
| Chinchilla (2022) | 70 B | 1.4 T | 20 | 1× (the target) |
| Llama 3 8B | 8 B | 15 T | ~1,875 | ~94× |
| Llama 3 70B | 70 B | 15 T | ~214 | ~11× |
| Llama 3 405B | 405 B | 15.6 T | ~38 | ~2× |
| DeepSeek V3 | 671 B (37 B active) | 14.8 T | ~22 | ~1× |

Small models get trained the furthest past Chinchilla because they'll be served the most (Llama 3 8B runs on a laptop). Huge models like DeepSeek V3 stay close to the compute-optimal ratio because at that scale, training FLOPs still dominate the lifetime budget.

---

## 6. Openness as a categorical variable

Three tiers, categorically distinct in terms of scientific reproducibility:

- **Closed** — only API access. Weights, training data, architecture details all proprietary. GPT-4o, Claude, Gemini.
- **Open-weight** — weights are downloadable and inference-ready; architecture is documented; training data and full recipe are not. Reproducing the training is impossible without insider knowledge. Llama 3, DeepSeek V3, Qwen 2.5.
- **Open-source** — weights + training data + training code + logs all published. Reproducing is possible in principle. OLMo 2, Pythia.

The distinction between the last two is the one that gets muddled in press releases, and it matters because *only open-source models let you do the science*. You can't ablate a data-mixture choice on Llama 3; you can on OLMo.

---

## 7. Tokenization — the interface between text and math

Neural networks operate on vectors. Text is bytes. A **tokenizer** is the deterministic bridge, with two functions:

- **encode:** string → list of integer IDs (each ID is a **token**)
- **decode:** list of integer IDs → string

And one property that must hold — **round-trip invariance:** decode(encode(s)) = s for every string s. If your tokenizer ever loses information, you've broken the model's grounding to text.

Two numbers characterize a tokenizer:

- **Vocabulary size |V|** — number of distinct integer IDs. Every ID gets a learned embedding vector; the model's output layer projects to a vector of length |V|, on which we softmax. So |V| directly sizes two big matrices in the model.
- **Compression ratio ρ** — average bytes of source text per token: ρ = bytes / tokens. Higher ρ means fewer tokens per document.

### 7.1 Why the compression ratio isn't a cosmetic choice

Attention cost per layer is

$$
C_{\text{attn}} \;\propto\; L^2 \cdot d
$$

If you double ρ, you halve L for the same text, which quarters the attention cost of processing a fixed corpus. And a fixed context window covers twice as much source text.

**Concrete example.** Take the pangram:

> `The quick brown fox jumps over the lazy dog.`

That's 44 bytes of ASCII (each character = 1 byte). Here's how the four families of tokenizers would encode it:

| Tokenizer | Token count | ρ (bytes/token) | Effective sequence length |
| --- | --- | --- | --- |
| Byte-level (any) | 44 | 1.00 | 44 |
| Character-level | 44 | 1.00 | 44 |
| Word-level (naive split) | 10 | 4.40 | 10 |
| GPT-4o (`o200k_base`) | 10 | 4.40 | 10 |
| Llama 3 | 10 | 4.40 | 10 |
| DeepSeek V3 | ~10 | ~4.40 | ~10 |

Attention cost for this sentence at the byte-level would be (44/10)² = **19.4× more expensive** than at the BPE level — for the exact same text. Now scale that to a document. **A 5% better tokenizer is 5% more effective training compute for the entire pretraining run**, permanently.

### 7.2 Information-theoretic ceiling

Shannon's source-coding theorem: no lossless code can encode a source with expected code length below its entropy. If English has entropy H_english ≈ 1.0 to 1.5 bits per character, then any tokenizer with |V| tokens can at best hit

$$
\rho_{\text{max}} \;\approx\; \frac{\log_2 |V|}{H_{\text{per byte}}}
$$

For |V| = 50,000: log₂(50000) ≈ 15.6 bits per token. If English is around 4 bits per byte, then ρ_max ≈ 4 bytes/token. That's exactly where GPT-2's tokenizer sits on English — not a coincidence, because BPE is (approximately) doing entropy coding.

Push |V| up to 200,000 (GPT-4o's `o200k_base`) and log₂ gains 2 bits per token, pushing ρ_max to ~4.4. In practice o200k_base hits ~4.5 bytes/token on English — right at the ceiling.

### 7.3 Four families of tokenizer, in order of increasing sophistication

**(a) Character-level.** Atoms are Unicode code points — the abstract character identifiers assigned by the Unicode standard, ~150,000 of them. `|V|` ≈ 150,000. ρ ≈ 1 character per byte for English (worse for languages with wide code points). Problems: vocabulary bloat (huge softmax with mostly-untrained rows for rare code points) and a long tail (🌍 shows up so rarely the model never learns it).

**(b) Byte-level.** Atoms are UTF-8 bytes. **UTF-8** is the variable-length encoding of Unicode: ASCII in 1 byte, common Latin accents in 2, most CJK in 3, emoji in 4. |V| = 256, ρ = 1.0 always. Every language on Earth encodes without gaps. No unknown token is ever needed. But ρ = 1.0 means sequences are ~4× longer than BPE on English, and the attention cost quadratics that penalty into ~16×. That's why byte-level, elegant as it is, has not yet won on production LMs.

**Concrete UTF-8 example.**
- `"a"` → 1 byte (`0x61`)
- `"é"` → 2 bytes (`0xC3 0xA9`)
- `"世"` → 3 bytes (`0xE4 0xB8 0x96`)
- `"🌍"` → 4 bytes (`0xF0 0x9F 0x8C 0x8D`)

So the sentence `"你好世界！"` (five characters, "Hello world!") is 5 × 3 = 15 bytes of UTF-8. A byte-level tokenizer produces 15 tokens for it; a Chinese-tuned tokenizer like Qwen 2.5's produces 5 (one per character). At Llama 2's small vocab (32k, primarily English), the same string can balloon to ~10–15 tokens because it falls back to bytes on unfamiliar CJK. This is the mechanism by which "Chinese eats up your context window" — the phrase is not folklore, it's UTF-8.

**(c) Word-level.** Atoms are whitespace-delimited words. Three killers:

- Vocabulary is unbounded. **Zipf's law** — the empirical observation that the k-th most frequent word has frequency proportional to 1/k — means cumulative-frequency curves never really flatten. Cover 90% of running text and you'll still miss the millions of rare words that make up the last 10%.
- Rare words get near-zero gradient signal.
- **Out-of-vocabulary (OOV)** words at inference time — anything the tokenizer wasn't trained on — must map to a special `<UNK>` token, which quietly poisons perplexity: predicting a common `<UNK>` is easy, so loss looks low while the model is useless.

**(d) Subword (BPE).** Atoms are learned byte sequences. Common patterns get their own tokens; rare patterns fall back to shorter tokens or bytes. This is the current standard and deserves its own section.

---

## 8. Byte-Pair Encoding — the algorithm and why it works

**Byte-Pair Encoding (BPE)** was originally a data-compression algorithm (Gage, 1994), then repurposed for NLP tokenization by Sennrich, Haddow & Birch (2016). The core idea: iteratively merge the most frequent adjacent pair, so common patterns become single tokens while rare patterns stay broken up.

### 8.1 Training algorithm

Given a training corpus and a target number of merges k:

1. Represent the corpus as a sequence of byte IDs in [0, 255]. Initial vocab has 256 tokens; each maps to a single byte.
2. Count adjacent pairs. For each pair (a, b) that occurs anywhere in the corpus, compute count(a, b).
3. Pick the most-frequent pair: (a\*, b\*) = argmax over (a, b) of count(a, b).
4. Merge: assign it a new ID 256 + i, add it to the vocab, and rewrite the corpus by replacing every occurrence of (a\*, b\*) with the new ID.
5. Repeat k times.

Final artifacts: a vocabulary of size 256 + k, and an ordered list of merge rules.

**Concrete walk-through.** Train on `"the cat in the hat"` for 3 merges.

- Initial byte sequence (spaces marked as underscore): `t h e _ c a t _ i n _ t h e _ h a t`
- Iteration 1: `(t, h)` occurs twice — tied but pick it. New token 256 = "th". Corpus becomes `256 e _ c a t _ i n _ 256 e _ h a t`.
- Iteration 2: `(256, e)` = "the" occurs twice. New token 257. Corpus becomes `257 _ c a t _ i n _ 257 _ h a t`.
- Iteration 3: `(_, 257)` = " the" occurs twice. New token 258. Corpus becomes `t h e _ c a t _ i n 258 _ h a t`. (The first "the" wasn't preceded by a space in this specific string.)

After 3 merges we have a vocab of 259 tokens, and " the" (space + word) is one token. Notice how this exactly matches the leading-space convention you see in GPT-2 and GPT-4 tokenizers today.

### 8.2 Why this is approximately optimal entropy coding

At each step, the merge that saves the most tokens is the one that eliminates the most instances of an adjacent pair — i.e., the one with the highest count. If pair (a, b) occurs c times, merging it saves exactly c tokens.

You can view BPE as a greedy version of dictionary-based compression (Lempel-Ziv family). Greedily picking the highest-frequency pair each iteration is a good approximation of "assign short codes to frequent patterns" — the Huffman/Shannon strategy for entropy coding. That's why the resulting tokenizer sits near the information-theoretic ceiling from §7.2.

### 8.3 Encoding a new string with a trained BPE

Given the merge rules M = [m₁, m₂, …, m_k] in learned order:

1. Encode s to bytes.
2. Repeatedly apply merges: at each step, find any occurrence of the earliest-learned merge rule whose pair is still present, and apply it. Stop when no rule applies.

A naïve implementation is O(k · L) per encode. Real implementations (OpenAI's `tiktoken`, HuggingFace's `tokenizers`) use a priority queue over pair positions and run in near-linear time.

### 8.4 Pre-tokenization: the invisible design choice

Vanilla BPE will happily learn merges that span across word or punctuation boundaries — you'd end up with a token for `". Th"` because it's common in text. That's noise. GPT-2's fix (which every mainstream BPE has since inherited): run a **regex pre-tokenizer** — a regular expression that splits the raw string into chunks *before* BPE runs, so no merge ever crosses a chunk boundary.

The GPT-2 pre-tokenization regex, un-golfed:

```
'(?:[sdmt]|ll|ve|re)   -- English contractions: 's, 'd, 'm, 't, 'll, 've, 're
 ?\p{L}+               -- optional leading space + a run of letters
 ?\p{N}+               -- optional leading space + a run of digits
 ?[^\s\p{L}\p{N}]+     -- optional leading space + a run of punctuation
\s+(?!\S)              -- trailing whitespace at end of line
\s+                    -- any run of whitespace
```

**Concrete example.** Take the sentence:

> `Don't panic! I've got 42 apples.`

Without pre-tokenization, BPE could learn a merge for `!.I` (punctuation + capital-letter transitions across sentence boundaries) because that's common. With the GPT-2 regex, the sentence splits first into:

`["Don", "'t", " panic", "!", " I", "'ve", " got", " ", "42", " apples", "."]`

BPE then only ever merges *within* those chunks. This is why:

- ` world` and `world` are different tokens (leading-space attached to the letter run).
- The same word at the start of a sentence and mid-sentence tokenizes differently.
- Numbers get chopped into digit runs, which is a big reason vanilla LMs are bad at arithmetic. GPT-4 tokenizes `1234567` as `[123][4567]` — the model literally sees "one-two-three-four-five-six-seven" as two arbitrary chunks rather than a number. Llama 3's tokenizer explicitly forces splits into 1-, 2-, or 3-digit chunks (never 4+), which measurably helps arithmetic accuracy.

### 8.5 Special tokens

Reserved IDs that never come from data:

- `<|endoftext|>` — document boundary.
- `<|im_start|>`, `<|im_end|>` — chat message boundaries in the GPT-3.5/4 chat format.
- `<|fim_prefix|>`, `<|fim_middle|>`, `<|fim_suffix|>` — fill-in-the-middle markers for code models.
- `[PAD]`, `[BOS]`, `[EOS]` — padding, beginning-of-sequence, end-of-sequence.

Special tokens must be detected and preserved *before* BPE runs — you never want a merge rule to eat half of `<|endoftext|>`. Getting this wrong is the source of a class of "chat template" bugs where a fine-tuned model behaves erratically because the wrong bytes ended up on either side of a chat-boundary token.

---

## 9. Variants of subword tokenization

- **Byte-Level BPE (BBPE).** Start from raw bytes (not code points). Guarantees no unknown-token fallback is ever possible — worst case, fall back to individual bytes. Used by GPT-2 onward, Llama 3, DeepSeek.
- **Unigram Language Model (Kudo, 2018).** Trained via Expectation-Maximization: start with a large candidate vocabulary, iteratively remove tokens that contribute least to corpus likelihood, keeping the top-scoring segmentation. Produces probabilistic segmentations. Used by SentencePiece.
- **WordPiece.** BERT's approach — like BPE but merges maximize corpus likelihood instead of raw pair frequency. Concretely, WordPiece picks the pair (a, b) that maximizes

$$
\text{score}(a, b) \;=\; \frac{\mathrm{count}(ab)}{\mathrm{count}(a) \cdot \mathrm{count}(b)}
$$

  which is essentially pointwise mutual information. It prefers merges that are strongly co-occurring rather than merely frequent.

- **SentencePiece.** A library (Google) that implements both BPE and Unigram tokenization, treats whitespace as a normal character (denoted `▁`), and supports **byte-fallback** — if a character isn't in the vocab, encode it as raw UTF-8 bytes rather than emit an unknown token. Llama 2 and Gemma use this.

### Vocabulary sizes in the wild

| Model | Tokenizer | \|V\| |
| --- | --- | --- |
| GPT-2 | Byte-Level BPE | 50,257 |
| GPT-3.5 / GPT-4 | `cl100k_base` | 100,277 |
| GPT-4o | `o200k_base` | ~200,000 |
| Llama 2 / Mistral | SentencePiece BPE + byte-fallback | 32,000 |
| Llama 3 | Tiktoken-based BPE | 128,256 |
| DeepSeek V3 | BPE | ~129,280 |
| Qwen 2.5 | BPE | ~151,646 |
| Gemma | SentencePiece BPE + byte-fallback | 256,000 |

The trend is clear: vocabulary sizes have grown roughly 4× over the last five years. The reason is multilingual and code data — English-only tokenizers grossly underperform on Chinese, Japanese, and code, so throwing more vocabulary slots at those distributions is one of the cheapest quality wins available.

**Concrete comparison across tokenizers.** Take the sentence:

> `Fine-tuning transformers on 1,000,000 examples is expensive.`

That's 60 bytes. Rough token counts by tokenizer:

| Tokenizer | Approx tokens | ρ (bytes/token) |
| --- | --- | --- |
| Byte-level | 60 | 1.0 |
| GPT-2 | ~17 | ~3.5 |
| GPT-4 (`cl100k_base`) | ~13 | ~4.6 |
| GPT-4o (`o200k_base`) | ~12 | ~5.0 |
| Llama 3 | ~13 | ~4.6 |

The digits split notably: GPT-4 encodes `1,000,000` as roughly `[1][,][000][,][000]` — three digit chunks and two commas — whereas Llama 3 forces `[1][,][000][,][000]` similarly but caps chunks at 3 digits.

Now try the same sentence in Chinese: `在一百万个样本上微调变换器很昂贵。` (roughly 17 characters, 51 bytes as UTF-8, since most Chinese chars are 3 bytes each).

| Tokenizer | Approx tokens on that Chinese sentence | ρ |
| --- | --- | --- |
| GPT-2 | ~35–40 | ~1.3 |
| GPT-4o (`o200k_base`) | ~17 | ~3.0 |
| Qwen 2.5 | ~17 | ~3.0 |

So the same string costs ~2× more context in the older English-centric tokenizer than in a Chinese-optimized one. Multiply across a full document and this is exactly why Chinese users noticed prompts "eating tokens" until GPT-4o's `o200k_base` shipped.

---

## 10. Byte-level and tokenizer-free approaches

BPE is a heuristic with known failure modes: arithmetic, morphologically rich languages, adversarial inputs, silent bugs at chat boundaries. So a research line asks: can we skip the tokenizer entirely?

- **ByT5 (2022, Google).** T5 architecture but on raw bytes. Works, but 4–8× slower per unit of text because of §7.
- **MegaByte (2023, Meta).** Hierarchical Transformer: a local model over fixed-length byte patches, then a global model over patches. Reduces the quadratic-attention hit on long sequences.
- **BLT — Byte Latent Transformer (2024, Meta).** Groups bytes into variable-length patches, where boundaries are chosen by a small entropy model — high-entropy regions get finer granularity, low-entropy runs get coarser patches. Best current attempt at closing the compute gap with BPE.
- **T-FREE (2024).** Vocabulary based on character trigrams. Aimed at morphologically rich languages where BPE fragments compound words badly.

None have decisively beaten BPE at frontier scale yet. But if any of them does, expect the "why can't GPT count the r's in strawberry" class of bugs to disappear overnight — that bug is entirely a tokenization artifact (GPT tokenizes `strawberry` as `[str][aw][berry]`, so the model never sees the individual r's).

---

## 11. Consequences worth internalizing

The tokenizer isn't a preprocessing step — it silently determines:

- **Effective context length.** Take Claude Sonnet 4.6's 200k-token window. At ρ ≈ 4.5 bytes/token on English, that's ~900 KB of source text — roughly 150,000 words, or 500 pages of double-spaced English, or the full text of *The Great Gatsby* three times over. But on a poorly-tokenized Chinese document (ρ ≈ 1.5), the same 200k window holds only ~300 KB. And on a codebase of dense Python, ρ ≈ 3.5, so ~700 KB — comfortably a whole small application. The "context window" is only meaningful once you name the domain.
- **Training throughput.** ρ × L × batch_size = tokens/sec. A 20% better tokenizer is ~20% more effective training compute — permanently.
- **Downstream capability shape.** Arithmetic, rare scripts, chemistry SMILES, unusual code idioms — whether these work well is a tokenization question first and a modeling question second.
- **Evaluation comparability.** Perplexity is not comparable across tokenizers; always convert to bits-per-byte before comparing. A model reporting "PPL 12" with a 50k vocab is not obviously better or worse than a model reporting "PPL 15" with a 200k vocab.

Everything downstream — architecture, systems, scaling laws — sits on top of choices made at this layer.
