# 🧠 Week 13 — Deep Conceptual Questions

> **Depth level:** frontier-lab research bar. Every answer includes derivation from first principles, paper citations by author + year, actual parameter counts and cost, and named failure modes.
>
> **How to use:** if you can defend these answers without slides, you're at the Anthropic/OpenAI/Meta AI Applied Research bar.

---

# 🌟 The 4 Questions Worth Mastering

> These four span the fine-tuning + multimodal surface. If you can answer them with the rigor shown, you clear ~85% of training-layer interviews.

---

## 🏆 Question 1: Derive LoRA From First Principles. Why Does Low-Rank Adaptation Work? When Does It Fail? ⭐⭐⭐⭐

### A strong answer covers:

- **The intrinsic rank hypothesis** (Aghajanyan et al. 2020, Hu et al. 2021)
- **The parameterization**: `W_updated = W_base + BA` where `B ∈ R^{d×r}, A ∈ R^{r×d}, r << d`
- **Parameter count reduction math**
- **Target module selection** (Q, K, V, O, MLP gate/up/down)
- **Rank selection framework**
- **When LoRA underperforms** (large distribution shifts, capability injection)
- **Gradient flow** through low-rank factorization
- **Comparison to full fine-tuning** with actual numbers

### Full Answer:

"LoRA (Low-Rank Adaptation) originates from the observation in **Aghajanyan et al. 2020 ('Intrinsic Dimensionality Explains the Effectiveness of Language Model Fine-tuning')** that fine-tuning updates to pretrained models lie in a low-dimensional subspace. **Hu et al. 2021 ('LoRA: Low-Rank Adaptation of Large Language Models')** operationalized this into a training method.

**The parameterization:**

For a pretrained weight matrix `W₀ ∈ R^{d×k}`, standard fine-tuning learns `ΔW ∈ R^{d×k}` such that the new forward pass is:

```
h = (W₀ + ΔW) x
```

LoRA reparameterizes `ΔW` as a product of two low-rank matrices:

```
ΔW = BA where B ∈ R^{d×r}, A ∈ R^{r×k}, r << min(d, k)
```

The forward pass becomes:

```
h = W₀ x + BA x
```

`W₀` remains frozen. Only `A` and `B` are trained. Typically initialized as `A ~ N(0, σ²)` and `B = 0`, so the initial update is zero (identity to pretraining).

**Parameter count:**

- Full fine-tune of one layer: `d × k` parameters trainable
- LoRA of one layer: `d × r + r × k = r × (d + k)` parameters trainable

For Llama-70B attention layer with `d = k = 8192`, `r = 16`:
- Full FT: 67M parameters per layer
- LoRA: 262K parameters per layer
- **Reduction: 256×**

For the whole model (80 layers, 4 attention projections + 3 MLP projections):
- Full FT trainable: 70B params
- LoRA (Q,V only, r=16): ~16M params
- **Trainable ratio: 0.023%**

**Why it works — the intrinsic rank hypothesis:**

Empirically, task-specific fine-tuning updates have effective rank much smaller than the full weight matrix rank. Hu et al. 2021 showed on GLUE that `r=1` or `r=2` can match full fine-tuning on many tasks.

The intuition: pretraining encodes broad knowledge in high-dimensional weights. Task adaptation modifies these along a low-dimensional direction — the model doesn't need new capabilities, just refined behavior along a task-specific manifold.

**Target module selection:**

The Q, K, V, O projections in attention are where task-specific information routing happens. Attention captures 'what to attend to' — highly task-dependent. Empirically:
- Q + V only (r=16): often sufficient, cheap
- Q + K + V + O (r=8): balanced
- Q + K + V + O + gate_proj + up_proj + down_proj (r=8): best quality, more params

MLP layers (gate/up/down_proj) contain more general capability; adapting them risks capability drift. **QLoRA paper (Dettmers 2023) recommends adapting ALL linear layers** because their 4-bit base weights don't degrade the pretrained knowledge as much.

**Rank selection framework:**

- `r=4`: capacity-limited, works for very narrow style adaptation
- `r=8-16`: production sweet spot for most tasks
- `r=32-64`: complex domain shifts, larger training sets (100K+ examples)
- `r=128+`: approaching full-rank adaptation, diminishing returns vs full FT

**The alpha parameter:**

`ΔW = (α/r) BA` scales the update. Alpha is typically set to `r` or `2r`. Higher alpha = stronger LoRA influence. It's a Lora-specific learning rate multiplier that lets you keep the base optimizer LR constant.

**Gradient flow:**

Backward pass:
```
∂L/∂A = B^T (∂L/∂ΔW)
∂L/∂B = (∂L/∂ΔW) A^T
```

The full weight `W₀ + BA` is used in forward pass. Gradients flow through the frozen `W₀` (no update) and update only `A` and `B`. Memory savings come from not storing optimizer state (momentum, variance in Adam) for `W₀`.

**When LoRA underperforms:**

1. **Large distribution shift.** If your target domain differs significantly from pretraining (e.g., pretraining on English, target is Turkish medical text), the low-rank update can't span the required capability shift. **Continued pretraining** is the right tool.

2. **Capability injection.** Teaching the model a fundamentally new skill (arithmetic, code generation from scratch, formal reasoning) exceeds low-rank capacity. Requires full fine-tune or new pretraining.

3. **Very small datasets (< 500 examples).** LoRA can overfit as easily as full FT with insufficient data. Data quality matters more than method.

4. **Long-context adaptation.** Extending a 4K-context model to 128K requires modifying RoPE-scaled attention patterns; LoRA on Q/V doesn't capture this well. Techniques like YaRN + continued pretraining work better.

**Comparison to full fine-tune (real numbers, Llama-70B):**

| Method | Trainable Params | VRAM | Time (10K samples) | Cost | Quality (MT-Bench) |
|---|---|---|---|---|---|
| Full FT | 70B | 1400 GB (8×H100 with FSDP + ZeRO-3) | 8h | ~$3000 | 8.4 |
| LoRA r=16 | 16M | 180 GB (4×A100) | 3h | ~$150 | 8.3 |
| QLoRA r=16 | 16M | 40 GB (1×A100) | 8h | ~$16 | 8.2 |

96% of full-FT quality at 0.5% of the cost. This is why LoRA is the production default.

**Frontier-lab context:** Anthropic and OpenAI likely use full fine-tuning for their models (they have the compute) but LoRA is the industry-standard for enterprise fine-tuning. Meta's LlaMA fine-tuning recipes ship LoRA."

---

## 🏆 Question 2: Derive DPO. Why Does It Eliminate the Reward Model? Under What Conditions Does It Fail? ⭐⭐⭐⭐

### A strong answer covers:

- **The RLHF objective** and its reward maximization + KL constraint
- **The closed-form solution** to the KL-constrained optimization
- **Rearrangement** to eliminate the reward function
- **The Bradley-Terry preference model**
- **The DPO loss** derivation
- **When DPO fails** (distribution shift, noisy preferences, verbosity bias)
- **Successors:** IPO, KTO, ORPO, SimPO

### Full Answer:

"DPO (Direct Preference Optimization) from **Rafailov et al. 2023 ('Direct Preference Optimization: Your Language Model is Secretly a Reward Model')** is one of the most elegant results in modern LLM training. Let me derive it.

**Setup — the RLHF objective:**

Standard RLHF optimizes:

```
max_π E_{x~D, y~π(y|x)} [r(x,y)] - β · D_KL(π(y|x) || π_ref(y|x))
```

Where:
- `π(y|x)` is the policy (language model) being trained
- `π_ref` is the reference (typically SFT-trained) model
- `r(x,y)` is a learned reward model
- `β` controls the KL penalty (how far π can drift from π_ref)

**Step 1 — Closed-form optimal policy:**

The KL-constrained reward maximization has a known closed-form solution (analogous to entropy-regularized RL):

```
π*(y|x) = (1/Z(x)) · π_ref(y|x) · exp(r(x,y)/β)
```

Where `Z(x) = Σ_y π_ref(y|x) exp(r(x,y)/β)` is the partition function.

**Step 2 — Solve for r(x,y):**

Rearranging:

```
r(x,y) = β log(π*(y|x)/π_ref(y|x)) + β log Z(x)
```

**The key insight:** we can express the reward function in terms of the optimal policy and reference model.

**Step 3 — Substitute into a preference model:**

Human preference data comes as pairs `(x, y_w, y_l)` where `y_w` (winner) is preferred over `y_l` (loser). The **Bradley-Terry model (1952)** models preference probability as:

```
P(y_w > y_l | x) = σ(r(x, y_w) - r(x, y_l))
```

Where `σ` is the logistic function.

Substituting our expression for `r`:

```
r(x, y_w) - r(x, y_l) = β log(π*(y_w|x)/π_ref(y_w|x)) - β log(π*(y_l|x)/π_ref(y_l|x))
```

**Critical: the `log Z(x)` terms cancel** (they don't depend on y).

**Step 4 — The DPO loss:**

Maximum likelihood on the preference data:

```
L_DPO(π_θ; π_ref) = -E_{(x, y_w, y_l)~D} [log σ(β log(π_θ(y_w|x)/π_ref(y_w|x)) - β log(π_θ(y_l|x)/π_ref(y_l|x)))]
```

**This is a simple classification loss on paired preferences.** No reward model. No RL. No PPO. Just a differentiable loss on the policy directly.

**Why this is beautiful:**

- **No reward model** to train (saves compute and a source of noise)
- **No sampling from π** during training (RLHF requires this, and it's expensive + unstable)
- **Simple training loop** — just supervised learning
- **Stable** — no policy collapse, no reward hacking

**When DPO fails:**

1. **Off-policy problem.** DPO assumes the preference data was collected FROM your policy (or something close to it). If preferences were collected from GPT-4 outputs and you're training LlaMA, the distribution mismatch hurts. **IPO (Azar 2023)** addresses this with a different loss.

2. **Verbosity bias.** DPO consistently generates longer responses than reference. This is because longer responses have more surface area to match preferences. **SimPO (Meng 2024)** removes the reference model dependence and normalizes by length.

3. **Reward hacking through length.** Related to verbosity. Human preference labelers often prefer longer, more detailed responses even when quality is equivalent.

4. **Preference noise.** DPO is sensitive to label noise. Bradley-Terry assumes clean preferences. In practice, ~15% of human labels disagree with re-labels. This creates a training-time signal ceiling.

5. **Distribution shift after DPO.** DPO doesn't provide the same policy regularization as PPO's explicit KL constraint at every step. Post-DPO models can drift substantially from SFT baseline. Monitoring MT-Bench + MMLU + IFEval is critical.

**DPO successors — the 2024 evolution:**

| Method | Year | Key change | When to use |
|---|---|---|---|
| DPO | 2023 | Original | Default preference optimization |
| IPO | 2023 (Azar) | Different loss, more robust to overfitting | Small preference datasets |
| KTO | 2024 (Ethayarajh) | Uses binary desirable/undesirable signal, not pairs | When you have ratings not pairs |
| ORPO | 2024 (Hong) | Combines SFT + preference in one loss | Simplify pipeline, single training run |
| SimPO | 2024 (Meng) | No reference model, length-normalized | Verbosity issues, memory constraints |

**Empirical comparison (from various papers):**

On MT-Bench, RLHF-PPO ≈ DPO ≈ IPO ≈ SimPO in aggregate. Differences show in edge cases: DPO better at complex reasoning, KTO better on unbalanced datasets, SimPO more concise.

**When to still use PPO/RLHF:**

Frontier labs (Anthropic, OpenAI) still use RL-based methods because:
- **Online exploration** — RLHF can query the reward model on new samples during training
- **Constitutional AI** (Anthropic) uses RL with AI feedback, harder to do with DPO
- **PPO's exploration** finds better solutions than off-policy DPO on complex objectives

For enterprise fine-tuning: **DPO first, always.** It's simpler, more stable, and matches RLHF quality within noise on typical tasks. RLHF is a research investment, not a production default."

---

## 🏆 Question 3: Explain FSDP vs DeepSpeed ZeRO-1/2/3. What's the Memory Math? When Do You Need Each? ⭐⭐⭐⭐

### A strong answer covers:

- **The memory footprint of training** (weights, gradients, optimizer state, activations)
- **The specific savings** at each stage
- **Communication cost** trade-off
- **When each stage is right**

### Full Answer:

"Training a large model requires a lot more VRAM than the model itself. Understanding this decomposition is fundamental to distributed training.

**Memory footprint of training a model with N parameters (in FP16 mixed precision, Adam optimizer):**

- **Model weights (FP16):** 2N bytes
- **Model weights master copy (FP32):** 4N bytes
- **Gradients (FP16):** 2N bytes
- **Adam optimizer state (FP32):** 8N bytes (momentum + variance)
- **Total non-activation:** 16N bytes per parameter

For Llama-70B: 70B × 16 = **1120 GB just for weights + gradients + optimizer state**. Plus activations, which scale with batch size and sequence length.

That's why 70B doesn't fit on any single GPU (max ~140 GB on H200).

**Enter distributed training:**

- **DDP (Data Parallel):** each GPU holds a FULL copy of the model + optimizer state. Different data per GPU. All-reduce gradients at each step. **Memory:** no savings. **Communication:** proportional to model size per step.

- **ZeRO (Rajbhandari et al. 2020 'ZeRO: Memory Optimizations Toward Training Trillion Parameter Models'):** shards the memory across GPUs. Three stages:

**ZeRO Stage 1 — Optimizer state sharding**

Each GPU stores 1/N of the optimizer state (where N = number of GPUs).
Weights and gradients still fully replicated.

Memory per GPU (for N GPUs):
- Weights (FP16): 2N_params bytes
- Gradients (FP16): 2N_params bytes  
- Optimizer state (FP32): 12N_params / N bytes  (sharded)

Savings on Llama-70B with 8 GPUs: optimizer state 840 GB → 105 GB per GPU. Total per GPU: ~245 GB (still doesn't fit).

**ZeRO Stage 2 — Optimizer state + Gradient sharding**

Gradients also sharded. During backward pass, each GPU computes ALL gradients but only stores the shard it owns. Requires **reduce-scatter** at end of backward.

Memory per GPU:
- Weights (FP16): 2N_params bytes
- Gradients: 2N_params / N bytes  (sharded)
- Optimizer state: 12N_params / N bytes  (sharded)

Llama-70B with 8 GPUs: 140 GB weights + 17.5 GB grads + 105 GB opt state = ~262 GB per GPU. Better, still doesn't fit.

**ZeRO Stage 3 — Full sharding (weights + gradients + optimizer)**

Weights ALSO sharded. Each GPU stores only its shard of every layer's weights. During forward pass, **all-gather** is called to reconstruct the full layer weights just for that layer's computation, then released.

Memory per GPU:
- Weights: 2N_params / N bytes  (sharded)
- Gradients: 2N_params / N bytes  (sharded)
- Optimizer state: 12N_params / N bytes  (sharded)

Llama-70B with 8 GPUs: 17.5 GB weights + 17.5 GB grads + 105 GB opt = 140 GB per GPU. **Fits on H100/H200.**

**Communication cost — the hidden trade-off:**

- ZeRO-1: 2× baseline (reduce-scatter + all-gather for optimizer sync)
- ZeRO-2: 2× baseline (adds gradient reduce-scatter)  
- ZeRO-3: 1.5-2× baseline PER LAYER (weights all-gathered at each forward layer)

ZeRO-3 has MUCH more communication than ZeRO-1/2 because it happens per layer, not per step. This is why fast interconnect (NVLink 900 GB/s) matters more for ZeRO-3.

**FSDP (Fully Sharded Data Parallel) — Zhao et al. 2023:**

PyTorch's native implementation of ZeRO-3, integrated into the PyTorch training loop. Key differences from DeepSpeed:

- **Simpler API** — wraps modules with `FSDP()`, chooses sharding strategy
- **Full pytorch integration** — plays well with torch.compile, mixed precision, etc.
- **Similar performance** to DeepSpeed ZeRO-3 in practice

FSDP sharding strategies:
- `FULL_SHARD`: full ZeRO-3 (weights + grads + opt state sharded)
- `SHARD_GRAD_OP`: ZeRO-2 equivalent
- `NO_SHARD`: standard DDP
- `HYBRID_SHARD`: FULL_SHARD within nodes, no shard across nodes (uses NVLink for the expensive part, avoids Ethernet for it)

**When to use which:**

| Model Size | Method | Reason |
|---|---|---|
| < 7B, fits on 1 GPU | DDP | Simplest, no communication overhead |
| 7-13B, doesn't fit on 1 GPU | ZeRO-2 / FSDP with SHARD_GRAD_OP | Weights fit, sharding grads+opt gives room for larger batch |
| 70B+, standard training | ZeRO-3 / FSDP with FULL_SHARD | Weights don't fit, must shard |
| 70B+, multi-node | FSDP with HYBRID_SHARD | Full shard within node (NVLink), replicated across nodes (avoid Ethernet penalty) |
| 175B+ | ZeRO-3 + CPU offloading + tensor parallelism | Beyond single node, needs more tricks |

**Unsloth (unsloth.ai, 2024):**

Not distributed training — a *single-GPU* optimization. Custom Triton kernels for attention, layer norms, and RoPE that achieve 2× training speed and 60% VRAM reduction vs HuggingFace on the same GPU. **Only supports single-GPU training** (as of 2026 — check current status). Use case: LoRA/QLoRA fine-tuning on 1×A100 or 1×H100 up to ~70B models with QLoRA.

**Axolotl and LlamaFactory:**

Higher-level trainers built on top of PyTorch + FSDP/DeepSpeed + PEFT + TRL. They handle:
- Configuration (YAML/JSON)
- Multi-GPU orchestration
- LoRA/QLoRA/full-tune switching
- Preference data formatting
- Chat template management

Use for production fine-tuning pipelines where you don't want to write PyTorch by hand.

**The staff+ signal:**

Junior: 'I use FSDP for large models.'
Staff+: 'For our 70B model on 8×H100 with NVLink, FSDP HYBRID_SHARD gives us ~145 GB/GPU with room for batch size 4 at 4K context; if we go multi-node we lose 30% throughput to Ethernet, so we stay single-node. If we had InfiniBand HDR we'd consider ZeRO-3 across 16 GPUs for a 405B model.'"

---

## 🏆 Question 4: Explain CLIP's Contrastive Learning. Why Does It Need Huge Batches? How Does It Enable VLMs? ⭐⭐⭐⭐

### A strong answer covers:

- **CLIP architecture** (dual encoder)
- **InfoNCE contrastive loss** derivation
- **Why batch size matters** (negatives)
- **Temperature parameter**
- **SigLIP's improvement**
- **How CLIP embeddings enable LLaVA-style VLMs**

### Full Answer:

"CLIP (Contrastive Language-Image Pretraining) from **Radford et al. 2021 ('Learning Transferable Visual Models From Natural Language Supervision')** is one of the most influential vision models ever. Every modern VLM builds on this foundation.

**Architecture:**

Two encoders:
- **Image encoder**: ViT or ResNet, produces embedding `v ∈ R^d`
- **Text encoder**: Transformer, produces embedding `t ∈ R^d`

Both project into the same d-dimensional space (typically d=512 or 1024). L2-normalized.

Training: **400M image-text pairs** scraped from the internet (WebImageText dataset).

**The InfoNCE contrastive loss:**

Given a batch of N image-text pairs `{(I_i, T_i)}`, form the similarity matrix `S ∈ R^{N×N}` where:

```
S_ij = (v_i · t_j) / τ
```

`τ` is a learned temperature scalar (starts at 0.07).

The loss is **symmetric cross-entropy** — treating each row and column of the similarity matrix as a classification over 'which text matches which image':

```
L = -0.5 · [ (1/N) Σ_i log(exp(S_ii) / Σ_j exp(S_ij))  +  (1/N) Σ_j log(exp(S_jj) / Σ_i exp(S_ij)) ]
```

The first term: image i should match text i (against all other texts as negatives).
The second term: text j should match image j (against all other images).

**Why huge batches?**

The number of negatives per positive = N-1 (all other pairs in batch). More negatives = harder classification = better representation learning.

CLIP trained with **batch size 32,768**. On modern hardware, this requires:
- Distributed training across many GPUs
- 'All-gather' the batch across workers for the contrastive matrix
- Massive activation memory

Small batch = few negatives = collapsed representations. This is the fundamental constraint of contrastive learning.

**Empirical: doubling batch size gives ~2 point ImageNet accuracy improvement, plateauing around 32K.**

**Temperature parameter τ:**

Controls the sharpness of the softmax. Lower τ = sharper distribution = stronger separation. Learned during training. Typically converges to ~0.01, meaning very peaked distributions.

The temperature also affects how the model handles hard negatives (visually similar but semantically different).

**SigLIP (Zhai et al. 2023) — the improvement:**

Softmax-based InfoNCE requires all-gather across GPUs (expensive). SigLIP replaces the multi-class softmax with a **sigmoid loss per pair**:

```
L = -Σ_{i,j} log σ(z_ij · (v_i · t_j) - b)
```

Where `z_ij = +1` if i==j (positive pair), `-1` otherwise. `b` is a learned bias.

Advantages:
- **No all-gather needed** — each pair is evaluated independently
- **Better at small batch sizes** — no reliance on huge negative counts
- **Faster training** — 4× throughput at same compute

SigLIP has largely superseded CLIP for new work.

**How CLIP enables LLaVA-style VLMs:**

**LLaVA (Liu et al. 2023 'Visual Instruction Tuning')** connects a vision encoder to an LLM via a simple projection layer.

Architecture:
```
Image  →  CLIP ViT encoder  →  patch embeddings (v_patches ∈ R^{n×d_vision})
                                                     ↓
                             Projection MLP  →  visual tokens (∈ R^{n×d_llm})
                                                     ↓
        Text (tokenized)  →  text token embeddings  ↓
                                                     ↓
                             LLM (Llama)  →  response
```

The visual tokens are literally inserted into the token sequence like text tokens. The LLM attends over them uniformly.

**Two-stage training:**

Stage 1 (feature alignment): freeze both CLIP and LLM. Train only the projection layer on image-caption pairs. Cheap, fast.

Stage 2 (instruction tuning): freeze CLIP, train projection + LLM on instruction data ('describe the image', 'answer the question').

**Why this works:**

CLIP's pretrained embeddings are aligned with language semantics (from 400M image-text pairs). The projection layer only needs to translate the CLIP embedding space to the LLM's embedding space — a much easier task than learning vision from scratch.

**Cost comparison:**

- Training CLIP from scratch: ~$1M in GPU-hours (400M pairs, large batches)
- Training LLaVA on top of CLIP + Llama: ~$1K in GPU-hours (much smaller data, only projection + LLM)

The composition of pretrained models is what makes modern VLMs economically feasible.

**Alternative architectures:**

- **Flamingo (Alayrac 2022)** — cross-attention layers in the LLM attend over vision features. More complex, better for interleaved image-text training.
- **Chameleon (Meta 2024)** — early-fusion, images tokenized to discrete tokens, treated identically to text. Cleanest architecture but requires from-scratch training.
- **GPT-4o / Gemini (native multimodal)** — trained from scratch with all modalities interleaved. Best quality, only feasible for frontier labs.

**When to use what:**

- **CLIP embeddings for search:** cheap, no LLM needed. Product search, image dedup.
- **LLaVA / open VLMs:** enterprise multimodal on-prem. Fine-tuneable.
- **GPT-4o / Claude via API:** highest quality, no infra burden.
- **ColPali:** document retrieval where layout matters (see Q22).

**The staff+ signal:**

Understanding that CLIP is a *foundation*, not a product — and every VLM either uses it, uses its successor SigLIP, or trains a similar objective from scratch."

---

# 📚 Q5-Q32: Additional Deep Conceptual Questions

## Q5. Explain QLoRA's NF4 quantization and Double Quantization. Why the specific choices? ⭐⭐⭐⭐

QLoRA (Dettmers et al. 2023) enables 65B fine-tuning on a single 48GB GPU. Three techniques:

**1. NF4 (NormalFloat 4-bit)** — 4-bit quantization designed for normally-distributed weights (which pretrained weights approximately are). Standard INT4 uses uniform quantization; NF4 uses quantiles of a normal distribution, so it allocates more precision where weights actually cluster. Result: 1-2% quality preservation vs INT4.

**2. Double Quantization** — the quantization constants themselves (scales and zeros) are also quantized. Each block of 64 weights has an FP32 scale (32 bits per 64 weights = 0.5 bits per weight overhead). Double quantization compresses these scales to 8 bits: 8/64 = 0.125 bits per weight. Saves ~0.4 bits per parameter across the entire model.

**3. Paged optimizers** — Adam optimizer state paged between GPU and CPU RAM using NVIDIA unified memory. Prevents OOM during gradient spikes.

**Cost math:** Llama-70B in FP16 = 140GB. In NF4 with double quantization = 35GB. LoRA adapters (~200MB). Fits on 1×A100 40GB.

**Quality:** empirically 99% of full fine-tune performance on GLUE, GSM8K. The residual gap is worth the 40× cost reduction.

---

## Q6. Explain DoRA (Weight-Decomposed LoRA). Why is it better? ⭐⭐⭐⭐

**DoRA (Liu et al. 2024 'DoRA: Weight-Decomposed Low-Rank Adaptation')** decomposes each weight matrix into magnitude and direction:

```
W = ||W||_c · (W / ||W||_c)
```

Where `||W||_c` is the column-wise norm (magnitude vector) and `W/||W||_c` is the direction.

DoRA adapts both:
- **Magnitude:** trained as a small vector `m ∈ R^d`
- **Direction:** adapted via LoRA (`V + BA`)

Empirical result: DoRA closes the gap between LoRA and full FT. On commonsense reasoning, DoRA with r=8 matches LoRA with r=16 or full FT.

**Why it works:** LoRA can't independently control the magnitude of updates. DoRA decouples these, matching what full fine-tuning naturally does. Analysis in the DoRA paper shows the magnitude direction is where full-FT deviates most from LoRA.

**Trade-off:** DoRA is ~10% slower than LoRA per step (extra magnitude computation). Worth it when quality matters.

**When to use:** production LoRA where quality is critical. Use r=8 DoRA instead of r=16 LoRA.

---

## Q7. Prompt tuning vs Prefix tuning vs P-Tuning v2 vs IA³ — the PEFT landscape beyond LoRA ⭐⭐⭐

**Prompt Tuning (Lester 2021):** learns soft prompt embeddings prepended to input. Only these embeddings are trained. Tiny parameter count (~0.01% of model). Works well for large models (T5-XXL), degrades for smaller ones.

**Prefix Tuning (Li & Liang 2021):** learns prefix vectors prepended to KEYS and VALUES at every transformer layer, not just input. More parameters than prompt tuning, more expressive.

**P-Tuning v2 (Liu 2021):** essentially prefix tuning with additional MLP reparameterization. Trained a bit more stably.

**IA³ (Liu et al. 2022):** learns scaling vectors for K, V, and MLP activations. Even fewer parameters than LoRA (~0.01%). Works surprisingly well for classification but weaker on generation.

**When to use each:**
- **LoRA/QLoRA/DoRA:** default for generation, code, instruction following
- **Prompt tuning:** classification with very large models
- **Prefix tuning:** legacy, use LoRA instead
- **IA³:** memory-constrained scenarios, classification

**Frontier lab practice:** LoRA/QLoRA dominates production. Others are research curiosities except in specific niches.

---

## Q8. Catastrophic Forgetting — the mechanism and mitigations ⭐⭐⭐⭐

**Definition:** fine-tuning on a narrow task causes the model to lose general capabilities.

**Mechanism:** during fine-tuning, weights shift to minimize the task loss. Since the task doesn't test general capabilities, those weights drift arbitrarily. Empirically: fine-tuning Llama-70B on 10K math problems can drop MMLU by 5+ points and MT-Bench by 1+ points.

**Mitigations:**

1. **Elastic Weight Consolidation (EWC, Kirkpatrick 2017):** penalize weight changes proportional to Fisher information. Weights important for pretraining get pinned. Rarely used at LLM scale (Fisher matrix expensive to estimate).

2. **Replay:** mix in samples from a general instruction-following dataset (e.g., 20% of batch is general, 80% is task). Cheap and effective. Standard practice.

3. **LoRA (implicitly):** by only training low-rank updates, the model's base weights are preserved. Empirically less catastrophic forgetting than full FT.

4. **Lower learning rate:** slow, careful updates preserve more general capability. LR 1e-5 for full FT vs 2e-4 for LoRA.

5. **Freezing early layers:** early transformer layers encode general syntax; late layers encode task-specific reasoning. Freeze first 60% of layers.

6. **Regularization to reference model:** add KL divergence from pretrained model to loss. Essentially what DPO's β does.

**Detection:** always evaluate on MMLU + IFEval + MT-Bench + task-specific holdout AFTER every fine-tuning run. If MMLU drops > 2 points, forgetting is real.

**Frontier lab practice:** replay + LoRA + careful LR is the standard playbook. Full FT + high LR + narrow data is a recipe for forgetting.

---

## Q9. Instruction tuning vs Chat tuning vs SFT — the distinctions ⭐⭐⭐

**SFT (Supervised Fine-Tuning):** the umbrella term. Any (input, output) supervised training on a pretrained LLM.

**Instruction tuning:** SFT specifically on (instruction, response) pairs, teaching the model to follow instructions. Introduced by **FLAN (Wei 2021)** and **T0 (Sanh 2021)**. Enables zero-shot task performance.

**Chat tuning:** SFT specifically on multi-turn conversations with speaker roles. Teaches the model conversational flow, role separation. Started with InstructGPT.

**The evolution:**
- Base LLM → SFT on instruction pairs → instruction-following model
- Base LLM → SFT on conversations → chat model  
- Modern: SFT on both + preference optimization

**Loss masking:**

Critical detail: during chat tuning, only compute loss on the **assistant's tokens**, not the user's or system's. Otherwise the model learns to imitate user queries.

```python
# Simplified:
loss_mask = torch.zeros_like(input_ids)
loss_mask[assistant_token_positions] = 1
loss = cross_entropy(logits, targets) * loss_mask
```

**Chat templates:**

Different models use different templates:
- **ChatML (OpenAI):** `<|im_start|>role\ncontent<|im_end|>`
- **Llama-3:** `<|start_header_id|>role<|end_header_id|>\ncontent<|eot_id|>`
- **Mistral:** `[INST] {user} [/INST] {assistant}`

Using the wrong template breaks generation. Verify template matches base model.

---

## Q10. Continued Pretraining vs Fine-Tuning — when each is appropriate ⭐⭐⭐

**SFT:** teaches the model NEW BEHAVIOR on existing knowledge. (query, response) pairs. Thousands of examples. Days of compute.

**Continued Pretraining:** teaches the model NEW KNOWLEDGE. Raw unlabeled text in target domain. Millions-billions of tokens. Weeks of compute. Same objective as pretraining (next-token prediction), just on a different corpus.

**When to use continued pretraining:**

1. **Large domain shift:** English pretraining, but need Turkish medical text. SFT can't inject that much new vocabulary.
2. **Massive unlabeled corpus:** you have 10GB of legal text but no labels. Continued pretraining extracts value from it.
3. **New language:** extending Llama to a low-resource language.
4. **Code specialization:** e.g., CodeLlama is Llama with continued pretraining on code.

**When SFT is enough:**

1. **Behavior adaptation:** style, format, tone, response length.
2. **Small domain shift:** you have (question, answer) pairs in the target domain.
3. **Small dataset:** < 100K examples.
4. **Time/budget constrained:** SFT is 10-100× cheaper than continued pretraining.

**Cost math:**

- SFT LoRA on 100K examples: $200
- SFT full fine-tune on 100K examples: $2K
- Continued pretraining on 10B tokens: $50K-100K

**Empirical results:**

Anthropic's constitutional AI, OpenAI's InstructGPT, Meta's Llama-3-Instruct all use SFT + preference optimization on top of massive pretraining. Only Meta releases continued pretraining publicly (CodeLlama).

**Common failure:** teams do continued pretraining when they only needed SFT. Waste of $100K.

---

## Q11-Q32: Additional Deep Conceptual Questions (Condensed)

### Q11. Synthetic data generation techniques and the model collapse risk ⭐⭐⭐
**Self-Instruct (Wang 2022):** LLM generates instructions and responses from a seed set. Bootstrapping. Used to train Alpaca. **Evol-Instruct (Xu 2023, WizardLM):** iteratively rewrites instructions to increase complexity. **Distillation:** stronger model (GPT-4) generates data for weaker model (Llama-8B). **Model collapse (Shumailov 2023):** training on generated data recursively degrades quality — tails of distribution disappear. Mitigation: mix synthetic + real data (min 20% real), diverse generators, quality filtering.

### Q12. RLHF full pipeline — SFT, reward model, PPO ⭐⭐⭐⭐
Three stages: (1) SFT on demonstrations. (2) Reward model: another LLM trained on preference pairs `(y_w, y_l)` to output scalar scores. Bradley-Terry loss. (3) PPO fine-tuning of the SFT model, using reward model + KL penalty from SFT baseline. Compute: SFT ~$5K, RM ~$5K, PPO ~$200K+ for 70B. PPO is unstable; requires careful hyperparameter tuning (KL coefficient β, clip range, epoch count). Modern practice: DPO replaces this in most enterprise cases; RLHF stays at frontier labs for hardest problems.

### Q13. Constitutional AI (Anthropic 2022) — how it differs from RLHF ⭐⭐⭐⭐
Uses AI feedback instead of human labels. Model critiques its own outputs against a written 'constitution' (e.g., 'be helpful, harmless, honest'). Two stages: SL from AI critiques (RLAIF-style SFT), then RL from AI feedback (RLAIF). Advantage: scalable (no human labelers per iteration), aligned with explicit principles. Disadvantage: quality depends on the critic model's own alignment. Used by Anthropic Claude models. Paper: Bai et al. 2022 'Constitutional AI: Harmlessness from AI Feedback.'

### Q14. Dataset construction principles — the unsexy 80% ⭐⭐⭐⭐
Rules that matter more than model choice: (1) **Coverage:** train distribution must cover eval distribution. (2) **Quality > quantity:** 1K high-quality > 100K noisy. LIMA (Zhou 2023) proved this — 1K carefully curated examples beat 52K noisy Alpaca. (3) **Deduplication:** exact + near-duplicate (MinHash). Duplicates cause memorization. (4) **Contamination check:** test set can't be in train set. Non-trivial with synthetic data. (5) **Distribution balance:** if 90% of your data is Q&A but 10% is summarization, model biases toward Q&A. Rebalance or reweight. (6) **Preference data cleanliness:** 15%+ label noise typical; multiple annotators help. (7) **Format consistency:** wrong chat template = broken model.

### Q15. Model Merging — Model Soups, TIES, DARE, Task Arithmetic ⭐⭐⭐
Training-free way to combine fine-tuned models. **Model Soups (Wortsman 2022):** average weights of multiple fine-tuned models. Works when models fine-tuned from same base. **Task Arithmetic (Ilharco 2022):** `W_new = W_base + Σ_i (W_i - W_base)` treats fine-tuning as vector arithmetic. Can subtract capabilities. **TIES (Yadav 2023):** trim, elect sign, disjoint merge — reduces conflict between task vectors. **DARE (Yu 2023):** randomly drops params and rescales, then merges. Empirically effective. **Use case:** combine 3 tenant-specific LoRAs into one adapter without retraining. **Limitation:** models must share tokenizer and architecture.

### Q16. LoRA Adapter Serving at Scale — vLLM's multi-LoRA support ⭐⭐⭐
Serve 100+ LoRAs on one base model. vLLM implementation: base weights shared, LoRA weights loaded per-request. Request specifies which adapter. Punica kernels (Chen 2023) enable efficient batched multi-LoRA. **Memory cost:** each LoRA r=16 on 70B ≈ 200 MB. 100 LoRAs = 20 GB. **Latency cost:** minimal (LoRA compute is small vs base). **Use case:** per-tenant fine-tuned models in SaaS. One base serves N tenants without deploying N model copies. **Cost savings:** ~90% vs dedicated deployments.

### Q17. Evaluation of fine-tuned models — perplexity's lies ⭐⭐⭐⭐
**Perplexity is misleading.** Lower perplexity ≠ better model. A model that memorized the training data has low perplexity but bad generalization. Modern eval stack: (1) **MT-Bench:** 80 challenging multi-turn tasks, LLM-judged. (2) **IFEval (Zhou 2023):** verifiable instruction-following (measurable rules). (3) **MMLU:** general knowledge regression check. (4) **GSM8K:** math regression. (5) **HumanEval:** code regression. (6) **Task-specific holdout:** your actual production task. Regression on any of #3-#5 signals catastrophic forgetting. Always eval on all after fine-tuning.

### Q18. ViT (Vision Transformer) — patches, positions, why it works ⭐⭐⭐
**ViT (Dosovitskiy 2020 'An Image is Worth 16x16 Words'):** treats images as sequences of patches. 224×224 image → 14×14 = 196 patches of 16×16 pixels. Each patch flattened + linear projected to embedding. Learnable positional encoding. Standard transformer processes as sequence. **Why it works despite no convolution inductive bias:** with enough data (JFT-300M), pure attention learns visual features. **Trade-offs:** ViT needs more data than ResNet to reach comparable quality (~10M+ images), but scales better beyond that. **Patch size matters:** 16×16 for standard vision, 14×14 for CLIP, 8×8 for finer detail (more compute). SigLIP uses 14×14.

### Q19. VLM Architectures — projection vs cross-attention vs native ⭐⭐⭐⭐
**Projection-based (LLaVA, Liu 2023):** simplest. CLIP encoder → MLP projection → LLM. Visual tokens inserted into text sequence. Train-cost: cheap (only projection layer initially). **Cross-attention (Flamingo, Alayrac 2022):** dedicated cross-attention layers in the LLM attend to vision features. Vision features stay separate. More parameters, better for interleaved image-text. **Native multimodal (GPT-4o, Gemini, Chameleon):** trained from scratch with modalities interleaved. Best quality, requires frontier-lab compute. **Discrete tokenization (Chameleon, Meta 2024):** vision tokenized to discrete tokens like text. Everything is tokens. Clean architecture, requires new tokenizer. **Choice for enterprise:** LLaVA-style for on-prem fine-tuneable, GPT-4o/Claude via API for max quality.

### Q20. LLaVA training recipe — the two-stage approach ⭐⭐⭐
**Stage 1 — Pretraining projection:** freeze CLIP + LLM, train only MLP projection on 558K image-caption pairs. ~1 hour on 8×A100. Aligns CLIP embedding space to LLM embedding space. **Stage 2 — Visual instruction tuning:** freeze CLIP, train projection + LLM on 150K instruction-following examples (GPT-4 generated). ~15 hours on 8×A100. Teaches conversational visual QA. **Total cost:** ~$500. **Result:** competitive with GPT-4V on visual QA benchmarks. **The lesson:** modular pretraining beats end-to-end training when components are already good.

### Q21. Whisper (OpenAI 2022) — the ASR paradigm shift ⭐⭐⭐⭐
**Whisper (Radford 2022 'Robust Speech Recognition via Large-Scale Weak Supervision'):** encoder-decoder transformer trained on **680,000 hours** of multilingual audio scraped from the web. **Input:** log-mel spectrogram (80 mel bins, 25ms window, 10ms stride). **Encoder:** conv stems + transformer, produces 1500 audio tokens per 30s chunk. **Decoder:** transformer that generates text conditioned on audio tokens. **Multitask:** single model handles ASR, translation, language ID, voice activity detection via special task tokens (`<|transcribe|>`, `<|translate|>`, `<|en|>`, etc.). **Robustness:** weak supervision at scale > careful supervision at small scale. Whisper handles noise, accents, code-switching where prior models failed. **Size options:** tiny (39M) to large-v3 (1.5B). **Distil-Whisper:** distilled version, 6× faster, 50% smaller, 99% quality.

### Q22. ColPali — the document retrieval breakthrough ⭐⭐⭐⭐
**ColPali (Faysse 2024 'ColPali: Efficient Document Retrieval with Vision Language Models'):** replaces OCR + text embedding with direct vision embedding of document pages. Uses PaliGemma VLM to embed each page as multiple vision tokens (not one). Query embedded normally. **Late interaction (ColBERT-style):** for each query token, compute max similarity over document tokens; sum across query tokens. `Score(q, d) = Σ_i max_j (q_i · d_j)`. **Advantages:** (1) No OCR (handles handwriting, formulas, layout natively). (2) Preserves visual structure (tables, charts, figures). (3) Better on layout-heavy documents (financial reports, scientific papers). **Benchmark:** ViDoRe. ColPali beats OCR + BM25 + rerank by 10-20 points nDCG@5 on layout-heavy retrieval. **Cost:** ~5× storage vs text embedding (multiple embeddings per page).

### Q23. Voice agent latency budget — from mic to response ⭐⭐⭐⭐
Human perception: <1s response feels natural, >2s feels broken. Budget breakdown:
- Voice activity detection: 100ms (buffer end-of-speech)
- STT (Whisper-medium streaming): 300ms
- LLM first token: 500ms (this is the largest component)
- TTS first audio: 200ms
- **Total to first sound: 1100ms** (achievable, tight)
Optimizations: (1) Streaming STT with chunk-and-emit (don't wait for full utterance). (2) LLM with speculative decoding. (3) Streaming TTS (e.g., ElevenLabs Flash). (4) End-of-turn detection tuned aggressively. (5) Skip STT/TTS entirely for voice-native models (GPT-4o Realtime API). **Advanced:** interruptibility — detect user starting to speak, gracefully stop TTS and process new input.

### Q24. Streaming ASR — chunk-and-emit vs full context ⭐⭐⭐
Standard Whisper: full 30s context, batch. Not streaming. **Streaming techniques:** (1) **Chunk-and-emit:** process audio in 200-500ms chunks, emit tokens progressively. Downstream latency but early results. (2) **Overlap chunks:** each chunk includes 100ms of previous audio to avoid boundary errors. (3) **Beam search delay:** wait N tokens before committing (allows correction). **Trade-off:** streaming reduces accuracy 3-5% vs batch (less context per decision). **Use case:** voice agents need streaming; podcast transcription doesn't.

### Q25. Video understanding — frame sampling, temporal reasoning ⭐⭐⭐
**Frame sampling:** uniform (every Nth frame), scene-change-based (adaptive), keyframe-only. Trade-off: more frames = better quality + more tokens/cost. **Typical:** 1 frame/second for VLM analysis, 8 frames/second for action recognition. **Long video challenge:** memory quadratic in tokens (attention). A 10-minute video at 1 fps = 600 frames = ~600K tokens (each frame ~1K tokens after encoding). Exceeds most LLM contexts. **Mitigations:** (1) Frame subsampling with adaptive selection. (2) Temporal pooling. (3) Long-context models (Gemini 2M tokens). (4) Hierarchical processing (summarize segments, aggregate). **Models:** VideoLLaMA, LLaVA-Video, Qwen2-VL support video natively.

### Q26. Multimodal embeddings — cross-modal search ⭐⭐⭐
Same principle as CLIP: image and text encoded to shared space, retrieve by cosine similarity. **Use cases:** product search (photo → matching listings), brand safety (image → policy category), reverse image search, video search. **Models:** CLIP (500M+ dimensions), SigLIP-2, OpenCLIP variants (H, G, L). **Fine-tuning:** contrastive learning on domain data. E.g., fashion retailer trains on their catalog with product-description pairs. **Cost:** embedding computation is dominant (~$0.10/1K images). Storage: 1KB/image typical.

### Q27. Multimodal generation — Diffusion, video, audio ⭐⭐⭐
**Diffusion models (DDPM, Ho 2020):** learn to denoise Gaussian-corrupted images. Iteratively denoises from pure noise to image. Stable Diffusion adds latent-space diffusion (compress image → VAE → denoise in latent space → decompress). **Text-to-image:** cross-attention from text embeddings to diffusion U-Net. **Modern:** Stable Diffusion 3, Flux, DALL-E 3. **Video generation (Sora, Runway):** diffusion in space-time. Extremely compute-heavy. **Audio (ElevenLabs, XTTS):** transformer-based, can clone voices. Ethics: voice cloning consent required.

### Q28. Multimodal production costs — the reality ⭐⭐⭐
**Storage:** text is ~1KB per message. Image is 200KB-2MB. Video is GB. 1000× ratio. Storage costs matter. **Bandwidth:** transferring images 1000× cost of text. CDN important. **Compute:** image processing 10× text (encoder pass). Video 100× text. **Latency:** loading + processing media adds seconds vs milliseconds for text. **Multi-modal cache:** cache CLIP embeddings for images that reappear. **Frontier lab practice:** aggressive caching, CDN, media pipelines optimized separately from text pipelines.

### Q29. Fine-tuning for domain terminology — the tokenizer problem ⭐⭐⭐
Pretrained tokenizers may split domain words inefficiently. E.g., 'thrombocytopenia' might tokenize as 15 tokens in Llama, 3 in a medical-domain tokenizer. **Impact:** more tokens = more compute, more likely to lose semantic coherence. **Solutions:** (1) **Continued pretraining with tokenizer extension** — add domain vocabulary, extend embedding matrix, warm-up new tokens on domain corpus. (2) **Accept the cost** — most domains work fine with base tokenizer. (3) **Full retrain with domain tokenizer** — rare, expensive. **When it matters:** medical, legal, code — domains with highly specialized vocabulary.

### Q30. Packing — training efficiency trick ⭐⭐⭐
Sample packing: concatenate multiple short training examples into one sequence up to max length. Padding is waste; packing eliminates it. **Naive packing:** just concat with separator token. Problem: cross-example attention leaks information. **Attention masking:** ensure examples don't attend to each other. Requires custom attention implementation. **Impact:** 2-3× training throughput on datasets with variable-length examples.

### Q31. When NOT to fine-tune — the honest answer ⭐⭐⭐⭐
Fine-tune ONLY if all these are true: (1) Prompt engineering fully explored and insufficient. (2) RAG can't provide missing knowledge. (3) You have 500+ high-quality examples (or 5000+ for full FT). (4) You can measure quality objectively. (5) The quality gap is worth $200-$10K + weeks of engineering. (6) Base model + prompt has been benchmarked on your task. If ANY of these fails, don't fine-tune. Most 'we need fine-tuning' turns out to be 'we need better prompts.'

### Q32. Frontier lab post-training pipelines — what they actually do ⭐⭐⭐⭐
Confirmed public info: **Anthropic:** SFT → Constitutional AI SL → RLAIF. Emphasis on safety training. **OpenAI:** SFT → RM training → PPO. Constant iteration, weekly releases. **Meta:** SFT → RLHF for Llama-Instruct, released publicly. **Mistral:** DPO-based for open-weight models. **Common patterns:** massive human labeling ($10M+ budgets), red-teaming as ongoing process, quality regression testing on every checkpoint, safety training as first-class priority. The gap between frontier lab post-training and enterprise fine-tuning is scale of data + scale of compute + specialized talent.
