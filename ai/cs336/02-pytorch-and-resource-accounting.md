# CS336 · 02 — PyTorch & Resource Accounting

> Concept-only notes. We assume you'll pick up PyTorch API mechanics elsewhere (learn-pytorch.io is the canonical friendly source). What this file covers is the specific slice of PyTorch that CS336 leans on — the pieces that let you *reason quantitatively* about how many bytes, how many FLOPs, and how much wall-clock a piece of code costs. Percy's mantra: efficiency is the name of the game, and you can't optimize what you can't count.
> Rendering: Unicode inline math (α β Σ ρ ² ⁻¹) + `$$…$$` display blocks with blank lines around, KaTeX-safe subset, never raw `*` or `_` outside a subscript brace (use `\ast`, `\_`).
> Companion source: [lecture_02.py](https://github.com/stanford-cs336/spring2025-lectures/blob/main/lecture_02.py) · YouTube: [Lecture 2](https://www.youtube.com/watch?v=msHyYioAyNE)

---

## 1. The napkin-math frame

Percy opens the lecture with two questions. Both are answered by dimensional-analysis arithmetic and both matter because at frontier scale, a wrong estimate translates directly into dollars.

**Question 1 — Time to train.** How long does it take to train a 70 B-parameter dense Transformer on 15 T tokens using 1,024 H100s?

The recipe:

$$
C_\text{needed} = 6 \cdot N \cdot D = 6 \times 70{\times}10^9 \times 15{\times}10^{12} \approx 6.3{\times}10^{24}\ \text{FLOPs}
$$

An H100 offers ~1,979 TFLOP/s advertised in bf16, but **Model FLOPs Utilization** (MFU) — the fraction of promised FLOPs your model actually consumes — is realistically ~0.5 (we'll define MFU properly in §6). So per GPU per second you land ~1 × 10¹⁵ useful FLOPs. Multiply by 1,024 GPUs, by 86,400 s/day: ~8.8 × 10¹⁹ FLOPs per day. Divide:

$$
\frac{6.3{\times}10^{24}}{8.8{\times}10^{19}} \approx 72\ \text{days}
$$

Percy quotes 144 days at MFU = 0.5 with more conservative numbers; either way the point is: this class of arithmetic takes 90 seconds on a napkin and it is the *only* thing standing between you and a factually-wrong budget in a planning meeting.

**Question 2 — Largest model on 8 H100s "naively" with AdamW.** Each H100 has **80 GB HBM** (high-bandwidth memory), so 8 × 80 GB = **640 GB total**. Without any cleverness (no sharding, no quantization, no activation checkpointing), AdamW requires **16 bytes per parameter** — a number we'll derive in §9. So:

$$
N_\text{max} \approx \frac{640{\times}10^9\ \text{bytes}}{16\ \text{bytes/param}} = 40{\times}10^9 = 40\text{ B parameters}
$$

That's before you've allocated a single byte for activations, so the real answer is smaller. In practice, teams get to 70 B on 8 H100s only with ZeRO-style sharding and mixed-precision tricks we'll meet in later lectures.

Every section below is one of the ingredients that make these back-of-envelope numbers reliable.

---

## 2. Tensors: the atom, and its memory footprint

Every quantity in deep learning — parameters, gradients, optimizer state, activations, the input batch itself — lives in a **tensor**. The memory footprint of a tensor is trivially:

$$
\text{bytes}(T) = (\text{numel}) \times (\text{bytes per element})
$$

And "bytes per element" is set by the tensor's data type. This is the number that dominates every memory-budget calculation you'll do.

| dtype | bits | bytes | Sign | Exponent | Fraction | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **fp32** (float32, "single") | 32 | 4 | 1 | 8 | 23 | The scientific-computing default. Deep-learning "full precision." |
| **fp16** (float16, "half") | 16 | 2 | 1 | 5 | 10 | Small dynamic range — underflows badly on small numbers. |
| **bf16** (Brain Float 16) | 16 | 2 | 1 | 8 | 7 | Same range as fp32, coarser resolution. **The modern default for the forward/backward pass.** |
| **fp8** (E4M3 / E5M2) | 8 | 1 | 1 | 4 or 5 | 3 or 2 | Introduced 2022; supported only on H100+. Two variants trading range vs precision. |
| **int8 / int4** | 8 or 4 | 1 or 0.5 | — | — | — | Post-training quantization for inference; extremely aggressive. |

**Why fp16 hurts and bf16 fixes it.** Deep learning trains fine with rough resolution but needs a wide dynamic range — gradients span many orders of magnitude in a single training step. fp16 has only 5 exponent bits, so it underflows at ~6 × 10⁻⁵. Try to store 10⁻⁸ (a plausible small gradient) in fp16 and you get literal zero. bf16 keeps fp32's 8 exponent bits, sacrificing fraction bits instead — so it can represent 10⁻⁸ without underflow, at the cost of coarser resolution. Since gradient magnitudes matter far more than the 3rd decimal place, this is a win for deep learning.

**Concrete example.** The MLP `up_proj` matrix of GPT-3 175B is 12,288 × 49,152 (d_model × 4·d_model). That's ~604 million parameters in a single matrix. In fp32 that's **2.42 GB for one weight matrix**. In bf16 it's 1.21 GB. In fp8 it's 604 MB. On an 80 GB H100, that single matrix is 3% of your entire memory in fp32 — and there are ~1,600 such matrices in the model. The choice of dtype is not decorative.

### Mixed-precision recipe (industry standard, 2025)

- **Master weights and optimizer state → fp32** (accumulating small updates in bf16 loses signal).
- **Forward and backward pass → bf16** (the compute-heavy matmuls are here; the H100 tensor cores are 2× faster in bf16 vs fp32).
- **Attention softmax and layer-norm → fp32** in production (softmax is numerically touchy).
- **Loss scaling** was needed for fp16 to prevent gradient underflow. bf16 makes this obsolete.

Llama 3, DeepSeek V3, Qwen 2.5 all follow this recipe. DeepSeek V3 additionally uses fp8 for major matmuls in the forward pass, which is one of the reasons its reported training cost ($5.6M) is dramatically lower than what the same model in bf16 would run.

---

## 3. Tensor storage: the memory layout you actually care about

A tensor in PyTorch is *not* a matrix. It's a **pointer into a flat array of memory plus metadata** (a stride for each dimension) that tells PyTorch how to interpret that flat array as a multi-dimensional array.

Concrete example. A 2 × 3 tensor:

```
tensor:  [[1, 2, 3],
          [4, 5, 6]]

storage: [1, 2, 3, 4, 5, 6]   ← flat array of 6 floats
strides: (3, 1)               ← "to step one row, skip 3; to step one column, skip 1"
```

Element (i, j) is at storage index `i · stride[0] + j · stride[1]`. Element (1, 2) is at index 1·3 + 2·1 = 5. Look up storage[5] and you get 6. Correct.

### Views: free, and dangerous

Most reshape/index/transpose operations don't touch the underlying storage — they just create a new tensor with different strides pointing at the *same* memory. So:

- `x[0]` (first row), `x[:, 1]` (second column), `x.transpose(0, 1)`, `x.view(3, 2)` — all zero-copy.
- Mutations to the view mutate the original, silently. `y = x[0]; y[0] = 99` will set `x[0][0] = 99`.

This is the fastest way to write subtle bugs. Percy's advice: know when a call is a view vs. a copy, and force a copy with `.clone()` when you actually need one.

### Contiguity

A tensor is **contiguous** if walking through its elements in row-major order matches walking sequentially through storage. A transpose breaks contiguity (strides are no longer sorted). Some ops (like `.view()`) require contiguity — call `.contiguous()` first if a transpose is in the way. `.reshape()` is the polite version: it's `.view()` if possible and `.contiguous().view()` otherwise, materializing a copy silently.

**Why contiguity matters beyond bugs.** Cache locality. Sequential memory access is 5–20× faster than random access on GPU global memory. A non-contiguous access pattern silently kills your bandwidth. This is one of the reasons kernel writers (Lecture 6) obsess about memory layout.

---

## 4. Devices and data movement

By default, tensors live in **CPU RAM**. GPU compute is orders of magnitude faster, so tensors must be *moved* to GPU memory to be useful — and that transfer over PCIe is slow (~64 GB/s vs the H100's ~3 TB/s HBM bandwidth). So the rule: minimize crossings of the CPU/GPU boundary.

Two practical consequences:

- Create tensors directly on the device: `torch.zeros(1024, 1024, device="cuda")` beats `torch.zeros(1024, 1024).cuda()`, because the first never touches CPU RAM.
- Don't call `.item()` or index into a tensor for a scalar in a hot loop — every such call is a device sync, blocking until the GPU catches up. Print `loss.item()` once per step, not once per operation.

Percy's operational habit worth stealing: assert device explicitly at key boundaries. `assert x.device.type == "cuda"` in your training loop is a 10-second insurance policy against subtly-CPU-bound training.

---

## 5. Matmul is the whole game

Percy's claim, restated: for any Transformer at any scale that matters, **matrix multiplications dominate the compute**. Element-wise ops, softmax, layernorm — collectively they are the sub-percent noise floor. Every FLOP-count you'll do reduces to counting matmul FLOPs.

### The matmul FLOP rule

For matrices **A ∈ ℝ^(n × m)** and **B ∈ ℝ^(m × k)**, computing A · B costs

$$
\text{FLOPs} = 2 \cdot n \cdot m \cdot k
$$

The 2 is because each output element is a sum of m multiply-adds, and a multiply-add counts as 2 floating-point operations. Convention treats + and × as equally expensive; hardware treats fused-multiply-add (FMA) as one instruction, but it does two arithmetic operations, so the FLOP count includes both.

**Concrete example.** A single Llama 3 70B decoder block, forward pass, processing a batch of 1 sequence of 8,192 tokens: the four projections in the MLP alone (`gate_proj`, `up_proj`, `down_proj` — Llama uses SwiGLU) hit ~5 × 10¹¹ FLOPs. Multiply by 80 layers → ~4 × 10¹³ FLOPs per token. Multiply by ~1 M tokens/sec on an H100 → hitting a meaningful fraction of the H100's 1 × 10¹⁵ FLOP/s peak.

### The forward-pass shortcut

For a Transformer, sum the matmul FLOPs across all layers and you land at a beautifully clean approximation:

$$
\text{FLOPs}_\text{forward} \approx 2 \cdot N \cdot D_\text{tokens}
$$

where N is the parameter count and D_tokens is the number of tokens processed. The 2 comes from the matmul rule. The intuition: every parameter is a weight in some matrix, and every token has to be multiplied by every weight it interacts with, then summed. So on average each parameter participates in ~2 FLOPs per token.

There's an asterisk: this misses the attention QKᵀ and softmax·V multiplications, which don't touch parameters but do scale as L² · d. When sequence length is short relative to d_model, the asterisk is negligible. When it's long, attention contributes meaningfully (see Lecture 1 §3.1).

### Backward pass: 2× forward

The backward pass through a matmul requires computing two gradient tensors: the gradient wrt the input, and the gradient wrt the weight. Each of these is *itself* a matmul, and each has the same FLOP count as the forward matmul. So:

$$
\text{FLOPs}_\text{backward} \approx 4 \cdot N \cdot D_\text{tokens}
$$

Combining forward + backward:

$$
\boxed{\; \text{FLOPs}_\text{total} \approx 6 \cdot N \cdot D_\text{tokens} \;}
$$

This is where the 6 in `C ≈ 6ND` comes from. It's the entire justification for the training-compute estimate you use to plan every run. Memorize the split: **2 forward, 4 backward, 6 total.**

*Derivation on a two-layer linear model.* Forward: `H = X · W₁` (2·B·D·D FLOPs) then `Y = H · W₂` (2·B·D·K). Backward for W₂: needs `∂L/∂W₂ = Hᵀ · ∂L/∂Y` (2·B·D·K) and `∂L/∂H = ∂L/∂Y · W₂ᵀ` (2·B·D·K) — that's 4·B·D·K, exactly double the forward. Same for W₁. Total = 2× forward = 4·B·(D·D + D·K), and grand total forward + backward = 6·B·(D·D + D·K) = 6 · B · #params.

---

## 6. FLOP/s, MFU, and the difference between marketing and reality

Two units, one letter of difference, endless confusion:

- **FLOPs** (lowercase s) — number of floating-point operations. A count. What your model *needs*.
- **FLOP/s** (per second) — throughput. What your hardware *provides*.

Percy insists on the `/s` notation for the latter to prevent the muddle. Adopt the habit.

**Hardware throughput cheat sheet (bf16 tensor-core peak):**

| GPU | Peak throughput (bf16) | HBM | HBM bandwidth |
| --- | --- | --- | --- |
| A100 80GB | ~312 TFLOP/s | 80 GB | ~2 TB/s |
| H100 80GB SXM | ~990 TFLOP/s (dense) | 80 GB | ~3.35 TB/s |
| H200 141GB | ~990 TFLOP/s | 141 GB | ~4.8 TB/s |
| B200 (Blackwell) | ~2,250 TFLOP/s | 192 GB | ~8 TB/s |

The "1,979 TFLOP/s" you see on Nvidia's H100 slide is *with structured sparsity* — a specific 2:4 pattern where 2 of every 4 elements are zero. Real dense matmuls hit ~half that: ~990 TFLOP/s. As Percy's TA remarks, "no one uses" the sparsity mode. Use the dense number.

### MFU — how well are you using the hardware?

**Model FLOPs Utilization (MFU)** measures the fraction of the peak advertised throughput your training run actually converts into useful model FLOPs:

$$
\mathrm{MFU} = \frac{\text{model FLOPs actually performed per second}}{\text{hardware peak FLOP/s}}
$$

You measure model FLOPs (not wall-clock arithmetic) so someone can't cheat by inserting redundant computation. This is why the metric is called *model* FLOPs utilization — the numerator is a property of the model, not of what the runtime happened to execute.

Rough calibration you should have in your head:

- **MFU > 0.5** — good. Frontier pretraining runs (Llama 3, DeepSeek V3) report 0.35–0.55.
- **MFU < 0.05** — really bad. Almost always means data movement is the bottleneck, or you're recomputing something, or your batch size is way too small.
- **MFU ~0.9** — probably impossible in practice; the gap comes from collectives, memory-bandwidth-bound ops (layernorm, softmax), and kernel launch overhead.

There's a related metric — **Hardware FLOPs Utilization (HFU)** — that counts *actual* FLOPs the hardware executed, including redundant ones like recomputed activations. HFU > MFU when you're using activation checkpointing. MFU is the one you compare across models; HFU tells you what the silicon is actually doing.

---

## 7. Einsum and named dimensions

By ~day two of writing Transformer code, you'll hit a bug that traces to writing `-2` when you meant `-1`. The industry-standard defense is **named-dimension notation**, either through `torch.einsum` or via the `einops` library.

The idea is Einstein summation, borrowed from physics. Every dimension in every tensor gets a *name*. The equation specifies exactly which dimensions match up and which get summed.

Example — a batched query-key inner product:

```python
# Old style, bug-prone
scores = torch.matmul(Q, K.transpose(-2, -1))  # what's -2 again?

# einsum style, self-documenting
scores = einsum("batch heads seq_q d, batch heads seq_k d -> batch heads seq_q seq_k", Q, K)
```

Rules to internalize:

- Every dimension that appears in *both* input strings and *not* in the output → summed over.
- Every dimension that appears in the output → iterated over.
- `...` is a broadcast placeholder for any number of leading dimensions.

Companion libraries the CS336 assignments use:

- **`jaxtyping`** — annotates function signatures with tensor shape info as types: `def attention(Q: Float[Tensor, "batch heads seq d_head"], ...)`. Purely documentation by default; enable a runtime checker for enforcement.
- **`einops.rearrange`** — reshape with named dimensions; makes multi-head split/merge readable. E.g., `rearrange(x, "batch seq (heads d_head) -> batch heads seq d_head", heads=8)`.
- **`einops.reduce`** — aggregate over named dimensions.

Once your code names its dimensions, you'll stop writing dimension bugs. This is not glamorous but it's the single highest-leverage habit in the course.

---

## 8. Parameter initialization

An `nn.Parameter` is just a tensor that PyTorch tracks as a learnable weight (autograd knows to accumulate gradients into it and the optimizer knows to update it). The interesting question is: *what values do you initialize it with?*

Naively initializing a d-in × d-out weight matrix with unit Gaussian entries and multiplying by an input of unit variance gives outputs with variance ~d-in — which explodes as models get wide. For d = 8,192 (Llama 3 70B width), unit-Gaussian outputs would have std ~90, causing the next layer to output tens of thousands, then softmax to saturate, then training to diverge.

**Xavier / Glorot initialization** fixes this: scale entries by 1/√d_in so the output variance stays ~1 regardless of width.

$$
W_{ij} \sim \mathcal{N}\!\left(0,\ \frac{1}{d_\text{in}}\right)
$$

Variants you'll see in modern LM code:

- **Truncated normal** — draw from N(0, σ²) but reject anything with |z| > 3σ. Prevents rare huge initial weights that could destabilize the first few steps.
- **Kaiming init** — a factor of √2 adjustment for ReLU / GELU / SwiGLU nonlinearities (which zero out half the input on average).
- **Small init** for residual streams — some models initialize the last linear in each residual branch to zero, so the model starts as an identity function.

Llama 3's exact init: truncated normal with σ = 0.02 for embeddings and 0.02 / √(2 · n_layers) for output projections in each block. The `1/√n_layers` factor prevents variance blowup across the residual stream in deep models.

---

## 9. Optimizers and their memory tax

The optimizer is the piece of code that turns a computed gradient into a parameter update. The reason it matters for resource accounting is that most modern optimizers keep **per-parameter state** — extra tensors as big as the parameter tensor itself.

### The optimizer family, briefly

- **SGD.** `θ ← θ − η · g`. No state. 0 extra bytes per parameter.
- **SGD + momentum.** `v ← μ·v + g; θ ← θ − η·v`. One state tensor (v). 4 bytes/param in fp32.
- **AdaGrad.** `G² ← G² + g²; θ ← θ − η · g / √G²`. One state tensor. Scales down parameters with historically large gradients. Never used at scale (denominator monotonically grows → learning rate → 0).
- **RMSProp.** Same shape as AdaGrad but uses an exponential moving average of g². Fixes the "denominator grows forever" issue.
- **Adam (Kingma & Ba, 2014).** RMSProp + momentum. Two state tensors — first moment m (running mean of g) and second moment v (running mean of g²).

$$
m_t = \beta_1 m_{t-1} + (1-\beta_1) g_t, \qquad v_t = \beta_2 v_{t-1} + (1-\beta_2) g_t^2
$$

$$
\theta_t = \theta_{t-1} - \eta \cdot \frac{\hat m_t}{\sqrt{\hat v_t} + \epsilon}
$$

where m̂, v̂ are bias-corrected versions. Typical hyperparameters: β₁ = 0.9, β₂ = 0.95 or 0.999, ε = 10⁻⁸.

- **AdamW.** Adam with decoupled weight decay: the L2 regularization term is applied to the parameter directly (`θ ← θ · (1 − η·λ)`) rather than folded into the gradient. Empirically much better generalization than Adam-with-L2. **This is the modern LM default** (Llama, Mistral, DeepSeek, OLMo all use AdamW).
- **Muon (2024), SOAP (2024).** Recent research optimizers that are showing wins in some regimes. Muon has been used in production runs (Moonshot's Kimi) and might displace AdamW; too early to tell.

### The 16-bytes-per-parameter accounting

This is the number that answered Question 2 in §1. For AdamW at "vanilla" precision:

| Item | Precision | Bytes / param |
| --- | --- | --- |
| Master weights (θ) | fp32 | 4 |
| Gradient (g) | fp32 (or bf16 → 2) | 4 |
| Adam first moment (m) | fp32 | 4 |
| Adam second moment (v) | fp32 | 4 |
| **Total** | | **16** |

Note this excludes activations (which depend on batch size and sequence length) and any auxiliary buffers.

**Where the numbers go when you get clever:**

- Mixed precision: weights + optimizer state in fp32 stays at 12 B/param, but gradients can be bf16 (2 B) → 14 B/param.
- **ZeRO-1** shards the optimizer state across N GPUs → optimizer bytes divided by N per GPU.
- **ZeRO-2** additionally shards gradients.
- **ZeRO-3** additionally shards the parameters themselves. Each GPU only holds 1/N of everything.
- **8-bit Adam** (bitsandbytes): stores m and v in fp8 with block-wise scaling → optimizer state drops to 2 B/param instead of 8. Wins by ~30% of total memory on the biggest models.

### Concrete plug-in

Question: can I train a 7 B model (Llama 3 8B-scale) on a single 80 GB H100 with vanilla AdamW?

- Parameters + gradients + optimizer state: 7 × 10⁹ × 16 = 112 GB. **No — exceeds 80 GB before activations.**
- With bf16 gradients: 7e9 × 14 = 98 GB. Still no.
- With 8-bit Adam: 7e9 × 10 = 70 GB. Yes, barely, before activations. In practice you also need LoRA (train only a few million adapter parameters, freeze the rest) to fit a 7 B fine-tune on a single GPU.

This is why LoRA / QLoRA / adapter tuning became universal for single-GPU fine-tuning workflows.

---

## 10. Activations: the memory nobody warns you about

For every layer, the forward pass produces **activation tensors** that must be kept around for the backward pass to compute gradients. Their footprint scales as **B · L · d · n_layers · bytes**:

- B = batch size
- L = sequence length
- d = model width
- n_layers = layer count

For Llama 3 8B (d = 4,096, n_layers = 32) training at batch 1, seq 8,192, bf16 (2 B/element):

$$
\text{activations} \approx 1 \times 8{,}192 \times 4{,}096 \times 32 \times 2 = 2.15\ \text{GB}
$$

That's for a batch size of *1*. Scale batch to 4 and you're at 8.6 GB just for activations. Scale seq to 128k for long-context training and you're at 33 GB — activations become the memory bottleneck long before parameters do.

Two techniques to attack this, both introduced later in the course:

- **Activation checkpointing (gradient checkpointing).** Don't store activations at every layer; recompute them during the backward pass. Trades ~30% extra compute for ~10× less activation memory. Every serious training stack uses this.
- **Sequence / context parallelism.** Shard the sequence dimension across GPUs so each GPU stores activations for only 1/N of the tokens. Essential for long-context training (128k+).

For assignment 1's small models this won't bite you. For the project (§next file) it will.

---

## 11. The total memory equation

Everything in one line:

$$
\text{memory} = (\text{params} + \text{grads} + \text{optimizer state} + \text{activations}) \times \text{bytes per element}
$$

For AdamW at vanilla precision, no checkpointing:

$$
\text{memory} \approx 16 N + \text{bytes}(\text{activations})
$$

For AdamW with bf16 forward + fp32 optimizer state + activation checkpointing (the modern default):

$$
\text{memory} \approx 14 N + \frac{\text{bytes}(\text{activations})}{\sim 10}
$$

Every number that gets quoted about training-run feasibility ultimately reduces to this equation and the throughput arithmetic in §6.

---

## 12. Data loading and the memmap trick

Training corpora don't fit in RAM. Llama 3 saw 15 T tokens, which at ~2 bytes/token (uint16 vocab IDs, since 128k < 2¹⁶) is ~30 TB of pre-tokenized data. You cannot `torch.load` this.

The trick is **memory-mapping** — the OS lets you address a file as though it were a giant tensor, loading pages on demand:

```python
data = np.memmap("tokens.bin", dtype=np.uint16, mode="r")
# data has shape (num_tokens,) but nothing is loaded yet
batch = data[offset : offset + context_length]
# now the kernel pages in exactly the bytes you touched
```

Zero-copy in the sense that you never materialize the file in Python memory. Every real LM training loop is built on some variant of this pattern.

---

## 13. Randomness and reproducibility

Sources of randomness in a training run:

- Parameter initialization
- Dropout mask
- Data shuffle order
- Data augmentation (if any)
- CUDA nondeterministic kernels (matmul rounding differences across runs)

Percy's practical advice: **use different seeds for different sources.** Then you can hold data ordering fixed while varying initialization (to check for lucky/unlucky init), or fix init while varying data (to check the model, not the sample). One global seed → you can only ablate the union of everything.

For full determinism (rare but useful when debugging a specific failure): `torch.use_deterministic_algorithms(True)`, `torch.backends.cudnn.deterministic = True`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`. Accept a ~10% throughput penalty.

---

## 14. Checkpointing (the "save the model" kind)

Training runs crash. Preemptions happen. Nodes fail. **Checkpoint** everything you'd need to resume: model state, optimizer state, learning-rate scheduler state, step count, data-loader position, RNG state. Skip any of these and your resumed run is subtly different from the one that would have completed uninterrupted.

At frontier scale this is nontrivial — a 405 B checkpoint is ~800 GB in fp32, ~400 GB in bf16, and you're taking one every hour or two. Serious training stacks parallelize checkpoint writes across nodes and stream to distributed storage.

---

## 15. The mixed-precision endgame

Percy closes on where the field is pushing. The trend has been steady: fp32 → mixed fp32/bf16 (~2020) → mixed fp32/bf16/fp8 (DeepSeek V3, 2024) → possibly INT4 mainstream (2026?).

Two asymmetries make this work:

- **Training is precision-sensitive; inference is not.** A trained model can be quantized to 4-bit for serving with minimal quality loss (see `gguf`, `AWQ`, `GPTQ`). But *training* at 4-bit remains an open research problem.
- **Different parts of the network tolerate different precisions.** The forward pass matmuls are robust to bf16 or fp8. The optimizer's exponential moving averages are not — they accumulate over thousands of steps and small errors compound. So you keep master weights in fp32 and only run the compute-heavy matmuls in low precision.

This co-design between the numerical stack, the kernel, and the model architecture is what Percy calls "the systems and the model architecture design are synergistic" — and it's the through-line for the systems half of the course (Lectures 5–8).

---

*You now have every unit you need to do the compute + memory arithmetic Percy performed in §1 — for any model, at any scale, at any precision. Every question the rest of the course asks is some flavor of "how do I get more capability per FLOP or per byte?" The framework here is the ruler.*
