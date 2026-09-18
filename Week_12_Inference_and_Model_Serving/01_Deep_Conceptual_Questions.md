# 🧠 Week 12 — Deep Conceptual Questions

> **Focus:** KV cache math from first principles, PagedAttention, continuous batching mechanics, all quantization formats, all parallelism types, framework comparisons, hardware topology
>
> **Level:** Written to the frontier-lab and MAANG staff+ bar. Each answer includes the numbers you need to defend your position.

---

# 🌟 The 3 Questions Worth Mastering

> If you can answer these three with quantitative depth, you'll clear 80% of inference-focused interview loops at MAANG and frontier labs.

---

## 🏆 Question 1: Design a Self-Hosted LLM Inference Platform to Replace an $87K/Month OpenAI Bill. Hit Sub-200ms p99 TTFT and Beat API Cost by 60%+. Justify Every Decision. ⭐⭐⭐⭐

### A strong answer should cover:

- **Break-even math** — at what token volume does self-hosting beat the API? Show the calculation.
- **Model selection** — which model matches the quality bar? Open-weight options.
- **Hardware selection** — GPU family, memory tier, interconnect (NVLink vs Ethernet)
- **Serving framework** — vLLM vs TGI vs TensorRT-LLM vs SGLang, and why
- **Parallelism strategy** — TP, PP, DP, or combinations
- **Optimization stack** — quantization (which format), speculative decoding, KV cache tricks, prefix caching
- **Batching strategy** — continuous, in-flight, chunked prefill
- **Latency budget** — where does the 200ms go? What are the levers?
- **Quality regression testing** — golden set, quantization validation
- **Cost dashboard** — per-request, per-tenant, per-model attribution
- **Fallback plan** — API fallback for quality regression or GPU outage

**Interviewers want to see: quantitative reasoning, hardware awareness, framework comparison, and cost math — not just "we used vLLM."**

### Model Answer (Full Depth):

"This is the migration I'd own end-to-end. My approach:

**Step 1 — Break-even math (before touching hardware)**

OpenAI: GPT-4o at $2.50/M input + $10/M output = blended ~$6/M tokens (rough).
$87K/month → ~14.5B tokens/month.

Self-host target: Llama-3.1-70B on 4×H100 SXM.
- Hardware cost: ~$96K/year per 4×H100 node (spot ~$3.60/hr × 4 GPUs × 8760h × 0.85 for spot factor) OR ~$180K/year on-demand
- At 65% GPU utilization + optimizations: ~35 tokens/sec/GPU × 4 GPUs = 140 tokens/sec sustained
- Monthly capacity: 140 × 3600 × 24 × 30 = 363M tokens/month per node
- 14.5B tokens/month → need ~40 node-months = 40 nodes if serving continuously

Wait — that's more expensive than the API. So this scenario requires OPTIMIZATIONS to work.

Real math after optimizations:
- Continuous batching: 3-5× throughput
- PagedAttention: enables higher batch sizes without OOM
- INT4 AWQ quantization: 2× throughput, minimal quality loss (verified separately)
- Speculative decoding with 1B draft: 1.8-2.5× speedup on typical workloads

Combined: 12-20× throughput vs naive. So one 4×H100 node handles 4.5-7B tokens/month at 65% util.

Node count needed: 2-3 nodes for baseline load, 4 nodes with redundancy.
Cost: 4 × $96K = $384K/year on spot = $32K/month → **63% reduction vs $87K API bill.**

If we can't tolerate spot: $180K × 4 = $720K/year = $60K/month → still 31% reduction. Worth doing for the latency and control, but the math is tighter.

Decision: proceed with spot + on-demand mix (2 spot, 2 on-demand) = $50K/month savings. Also captures the LATENCY win separately.

**Step 2 — Model and quality baseline**

Llama-3.1-70B chosen because:
- Open weights, no licensing burden
- Quality within 5-10% of GPT-4o on our workload (measured on golden set, 500 queries)
- Well-supported by every serving framework
- Broad quantization support

Golden set testing is mandatory BEFORE migration. If Llama-3.1-70B doesn't clear our quality bar, we consider Llama-3.1-405B (much more expensive) or Mixtral 8x22B (MoE, different serving pattern) or stay on API. Never migrate to save cost while sacrificing product.

**Step 3 — Hardware selection**

4×H100 SXM per node:
- HBM3: 80 GB × 4 = 320 GB total
- Memory bandwidth: 3.35 TB/s per GPU — CRITICAL for decode phase (memory-bound)
- NVLink 4: 900 GB/s intra-node — enables tensor parallelism without communication bottleneck
- Ethernet 100 GbE for pod-to-pod (data parallelism only, not TP)

Why not H200 or B200? Available on H100 in current cloud market. When B200 supply matures (2026-2027), migration path is straightforward.

Why not A100? 40% less memory bandwidth. Decode-heavy workloads suffer disproportionately.

Why not L40S? 48GB HBM per card, less bandwidth. Better for smaller models or throughput-optimized batch workloads.

**Step 4 — Serving framework**

vLLM chosen because:
- Best-in-class continuous batching + PagedAttention (their invention)
- Automatic Prefix Caching (APC) — critical if we have stable system prompts
- Multi-LoRA serving support (future: per-tenant fine-tuned models)
- Best Python integration
- Mature ecosystem, widely deployed

Not TGI: Rust router is elegant but vLLM has broader feature velocity.
Not TensorRT-LLM: compile-based approach gives raw speed but engine building is painful; only choose if we're locked in on NVIDIA and need every ounce.
Not SGLang: RadixAttention is brilliant for prefix-heavy workloads; use if system prompts are our dominant token cost. For general chat, vLLM's APC is close enough.

**Step 5 — Parallelism strategy**

Tensor Parallelism (TP=4) across the 4 GPUs within a node:
- Llama-70B in FP16 = 140 GB weights — must shard across GPUs
- After INT4 AWQ: ~35 GB weights, could fit on 1×80GB GPU, but TP=4 gives more KV cache headroom AND lower latency (parallel matmuls)
- NVLink bandwidth handles the all-reduces in TP without becoming bottleneck

Data Parallelism (DP=N) across nodes:
- Each node is a full replica
- Load-balancer distributes requests
- N scales with traffic

No Pipeline Parallelism needed at this scale. PP adds complexity, hurts p99 latency (idle bubbles).

**Step 6 — Optimization stack**

Layer 1: INT4 AWQ quantization on weights
- ~2× throughput (smaller weights, more KV cache room)
- < 1% quality loss on our golden set (validated)
- Weight prep once, cached artifacts

Layer 2: FP8 KV cache
- 2× KV cache capacity
- Higher batch size sustainable
- Minor quality impact (measured)

Layer 3: Speculative decoding
- Draft model: Llama-3.1-1B
- Acceptance rate on our workload: ~65%
- Speedup: 1.8-2.2× depending on request

Layer 4: Automatic Prefix Caching
- Our system prompt is ~2K tokens, few-shot ~4K tokens = 6K tokens per request identical
- APC hits ~85% on production traffic
- ~40% reduction in prefill compute

Layer 5: Continuous batching (default in vLLM)
- Max batch size: 128 (KV cache capacity constrained)
- Requests join and leave batches token-by-token
- 3-4× throughput vs static batching

**Step 7 — Latency budget breakdown**

Target: 200ms p99 TTFT

- Network in (edge → app): 20ms
- Auth + routing: 15ms
- vLLM prefill for 6K tokens: 90ms (with APC hit) or 300ms (miss)
- With 85% APC hit rate: ~120ms average, ~300ms p99 on cache miss

To hit 200ms p99, we need APC hit rate high. Solution: warm the cache proactively with common system prompts on startup.

Alternatively: disaggregated prefill/decode architecture. Prefill on compute-optimized GPUs, decode on memory-bandwidth-optimized GPUs. Adds complexity but hits the SLO more consistently.

**Step 8 — Quality regression testing**

Golden set of 500 queries with expected quality tier.
- Every quantization change: run golden set, block deploy if score drops > 3%
- Every model version: same
- Weekly automated run against production model, track quality over time
- Comparison to API baseline (parallel API calls on 5% of traffic)

**Step 9 — Cost dashboard**

Per-request cost attribution:
- prefill_tokens × (weight_cost + KV_alloc_cost)
- decode_tokens × (decode_cost)
- GPU_hours × $rate

Rolled up per: tenant, feature, model version.
Alerting: cost per user > $X threshold, monthly forecast overshooting budget.

**Step 10 — Fallback plan**

Circuit breaker to OpenAI:
- Trigger on: quality regression detected, GPU cluster unhealthy, cost anomaly
- Fallback latency: higher, but service continues
- Budget: emergency $10K/month reserve for fallback traffic

**The senior insight:**

The mistake I've watched teams make is treating this as 'set up vLLM and switch traffic.' It's not. It's a systems-engineering problem across hardware, framework, optimization, and cost. Every decision compounds. Get quantization wrong → quality drops. Get parallelism wrong → latency spikes. Get batching wrong → cost stays high.

The winning migrations are: measure baseline, incrementally add optimizations, validate quality at each step, monitor cost from day 1, keep API fallback until confidence is proven. Six weeks minimum. Not a sprint."

---

## 🏆 Question 2: Compute the KV Cache Memory Requirement for Llama-3.1-70B at 8K Context Serving 32 Concurrent Requests. Show Your Work. ⭐⭐⭐⭐

### A strong answer should cover:

- **The KV cache formula** — layers × 2 (K and V) × sequence length × heads × head_dim × dtype_size
- **Llama-70B architecture parameters** — 80 layers, 64 KV heads (GQA), head_dim 128
- **Per-token math** — 328 KB/token derived
- **Per-request math** — at 8K context
- **Batch total** — 32 concurrent
- **Why this matters** — GPU memory budgeting, batch size ceiling
- **Optimization levers** — FP8 KV, GQA vs MQA, prefix caching, PagedAttention

**Interviewers want to see: architecture literacy + hardware awareness. This is the question that separates candidates who read papers from candidates who wave their hands.**

### Model Answer:

"KV cache math is where inference reality lives. Let me derive from first principles.

**Formula:**

```
KV_cache_bytes_per_token = 2 × num_layers × num_kv_heads × head_dim × dtype_bytes
```

The 2 is for K and V (separate tensors). We use num_kv_heads (not num_attention_heads) because Llama-70B uses Grouped-Query Attention where multiple query heads share KV heads.

**Llama-3.1-70B parameters:**

- num_layers: 80
- num_attention_heads: 64
- num_kv_heads: 8 (GQA with ratio 8:1)
- head_dim: 128
- default dtype: FP16 (2 bytes)

**Per-token computation:**

```
KV per token = 2 × 80 × 8 × 128 × 2 bytes
            = 327,680 bytes
            = 320 KB (some sources cite 328 KB depending on rounding)
```

**Per-request at 8K context:**

```
KV per request = 320 KB × 8,192 tokens
              = 2.62 GB
```

That's per request in FP16. Serious number.

**Batch total at 32 concurrent:**

```
Total KV = 2.62 GB × 32 = 83.9 GB
```

**Why this matters:**

Llama-70B in FP16 weights = 140 GB. Sharded across 4×H100 (80 GB each) via TP=4 = 35 GB weights per GPU.

Per GPU:
- Weights: 35 GB
- Activation buffers: ~10 GB
- Available for KV cache: 80 - 35 - 10 = ~35 GB

Sharded across 4 GPUs, total KV budget: 4 × 35 = 140 GB.

At 32 concurrent × 8K tokens each = 84 GB used. Fits with headroom.

But: what if 4 requests want 32K context each? That's 320 KB × 32K × 4 = 42 GB. Same overhead but only 4 concurrent. Batch size collapses.

**This is the KV cache economics problem.** Long context kills concurrency. Short context enables high concurrency.

**Optimization levers:**

1. **FP8 KV cache** — halves KV memory to 160 KB/token. Batch 32 at 8K = 42 GB. Doubles headroom.

2. **INT4 KV** (recent research) — quarters KV. Aggressive; validate quality.

3. **Grouped-Query Attention already applied** in Llama-70B — 8 KV heads vs 64 query heads → 8× reduction vs Multi-Head Attention on same base model.

4. **Multi-Query Attention** (single KV head) — extreme case, some quality trade-off, ~64× smaller KV than MHA.

5. **Multi-Head Latent Attention (MLA)** — DeepSeek's approach, KV cache in low-rank compressed form. Radical reduction.

6. **Prefix caching** — shared prefix across requests, KV computed once. If system prompt is 4K of 8K context, and shared across all 32 requests, save 4K × 320 KB × 31 (other requests) = ~40 GB per batch.

7. **PagedAttention** — allocates KV in blocks (pages) of 16 tokens. Handles variable-length requests without fragmentation. Enables higher effective batch size than naive contiguous allocation.

**Concrete recommendations for this workload:**

- Enable FP8 KV cache (2× capacity, minor quality impact if validated)
- Enable Automatic Prefix Caching in vLLM
- Use PagedAttention (default in vLLM)
- Consider offloading old KV to CPU/NVMe for very long contexts

At 8K context × 32 concurrent with all optimizations: comfortable batch size, room for burst to 48-64 concurrent.

**The senior insight:**

Every inference decision routes back to KV cache math. Choosing GQA over MHA in the model architecture is a KV cache decision. Choosing FP8 quantization on KV is a decision. Choosing max context length is a decision that CAPS your concurrency. Frontier labs live and die by this arithmetic.

Junior candidate answer: 'vLLM handles the KV cache.'
Senior candidate answer: 'Llama-70B GQA gives 320 KB/token, at 8K × 32 concurrent we need 84 GB batch KV, which fits our 4×H100 budget with FP8 headroom.'"

---

## 🏆 Question 3: Continuous Batching vs Static Batching vs Dynamic Batching vs In-Flight Batching. What Problem Does Each Solve? When Do You Use Which? ⭐⭐⭐⭐

### A strong answer should cover:

- **Static batching** — the CPU-era baseline, why it fails for LLM serving
- **Dynamic batching** — the incremental improvement, still fails on LLM's variable length
- **Continuous batching** — the paradigm shift, why LLM serving is different
- **In-flight batching** — TensorRT-LLM's flavor, subtle distinctions
- **Chunked prefill** — the newest addition, what problem it solves
- **Throughput math** — actual numbers per strategy

**Interviewers want to know if you understand WHY LLM serving is different from traditional ML serving. This is the first question of every senior inference interview.**

### Model Answer:

"This is my favorite question because it maps the entire evolution of LLM serving in one conversation.

**Static Batching (the ML-serving baseline)**

Pattern: wait for N requests to arrive, forward-pass them together, return N responses.

Works for: image classification, embedding models, anything where all inputs have similar shape and you return one output per input.

**Fails catastrophically for LLMs because:**

- Requests have variable prompt length
- Responses have variable output length
- Fast requests (short output) finish quickly, but the batch waits for slow ones (long output)
- Padding to max length wastes compute on padding tokens
- Long-tail latency = mean latency

Concrete: request A wants 20 tokens, request B wants 500 tokens. Static batch: A waits for B. p99 latency = long-tail request. GPU utilization tanks.

**Never use for LLM serving.**

**Dynamic Batching (Triton-era, still used in image/embedding servers)**

Pattern: wait for either N requests OR T milliseconds, then batch what's arrived.

Improvement: doesn't wait forever if traffic is low.

**Still fails for LLMs because:**

- Same variable-length problem
- Once batch starts, it runs to completion of slowest request
- Better than static, still bad

Use for: embedding models, classifier models, anything with fixed output length. Don't use for LLM generation.

**Continuous Batching (the paradigm shift — vLLM, TGI, etc.)**

The insight: LLM generation is TOKEN-BY-TOKEN. Each token is a separate forward pass through the model. You don't need to batch entire REQUESTS — you batch TOKENS.

Pattern:
- At each decode step, the scheduler forms a batch of tokens (one per active request)
- Requests that just started (in prefill) are handled separately or interleaved
- When a request finishes (EOS token generated), it drops out of the batch
- New requests join the batch immediately at the next step

Result: no request waits for another to finish. GPU stays busy. Throughput improves 3-5× over static.

**Also called:** iteration-level batching, token-level scheduling.

**Invented by** the Orca paper (2022) → productionized by vLLM.

**Now the default for every serious LLM serving framework.**

**In-Flight Batching (TensorRT-LLM's term)**

Semantically similar to continuous batching. NVIDIA calls it 'in-flight batching' in TensorRT-LLM to emphasize that requests are in-flight (mid-execution) rather than queued.

Subtle difference: TensorRT-LLM's implementation has kernels compiled for batch-of-requests, and manages the request slots at the kernel level for maximum efficiency.

**Interview trap:** don't confuse in-flight batching (TensorRT-LLM's name for continuous batching) with 'in-flight requests' (general term for pending requests). Context matters.

**Chunked Prefill (the newest evolution)**

Problem continuous batching didn't fully solve: prefill and decode have different compute profiles.

- Prefill (processing the prompt): parallel over prompt tokens, COMPUTE-BOUND. Loves large batch sizes.
- Decode (generating one token at a time): sequential, MEMORY-BANDWIDTH-BOUND. Small batches leave GPU idle.

Naive continuous batching mixes prefill and decode in the same forward pass. Prefill dominates when present, decode requests wait.

**Chunked prefill splits long prefills into chunks and interleaves them with decode steps.** So instead of processing a 32K prompt in one shot (blocking decode for other requests), it processes 512 tokens per step, allowing decode to continue for other requests.

Result: smoother tail latency, better GPU utilization, especially for mixed workloads (some requests with long prompts, others generating).

Introduced in Sarathi (2023), now available in vLLM, TensorRT-LLM.

**Selection framework:**

| Workload | Strategy |
|---|---|
| Image classification | Dynamic batching (Triton) |
| Embedding models | Dynamic batching |
| LLM chat/completion | Continuous batching (in-flight for TensorRT-LLM) |
| LLM with mixed short/long prompts | Continuous batching + chunked prefill |
| Extreme throughput target | Continuous + chunked + speculative decoding + FP8 KV |

**Throughput math (typical numbers, workload-dependent):**

For Llama-70B on 4×H100 at 8K context:
- Static batching (batch=8): ~200 tok/s aggregate
- Dynamic batching (batch=8): ~250 tok/s aggregate
- Continuous batching: ~800-1200 tok/s aggregate (3-5×)
- Continuous + chunked prefill: ~1500 tok/s aggregate on mixed workload
- Continuous + chunked + INT4 AWQ: ~2500-3000 tok/s aggregate
- Full stack (+ speculative decoding, prefix caching): ~4000-5000 tok/s aggregate on stable workload

12× improvement over naive is real, with the full optimization stack.

**The senior insight:**

Every generation of batching solved a problem the previous generation created. Static → dynamic solved 'wait forever.' Dynamic → continuous solved 'variable output length.' Continuous → chunked prefill solved 'long prompts starve decode.' Understanding this evolution IS understanding LLM serving.

The candidate who says 'we use continuous batching' passes the phone screen. The candidate who says 'we use continuous batching with chunked prefill because we have mixed 500-32K prompt lengths, and I tuned the chunk size to 512 tokens after benchmarking p99 latency vs GPU utilization' gets the offer."

---

# 📚 Q4-Q30: Deep Conceptual Questions (Structured Depth)

---

## Q4. Prefill vs Decode — why is this distinction the most important concept in LLM inference? ⭐⭐⭐⭐

**Prefill (also called: prompt processing, context encoding):**
- Processes the prompt tokens in parallel
- Compute-bound (matrix multiplications dominate)
- Loves large batch sizes and high FLOPS hardware (H100 shines)
- One forward pass produces KV cache entries for the entire prompt
- Duration: depends on prompt length, roughly linear

**Decode (also called: generation, autoregressive step):**
- Generates one token at a time
- Memory-bandwidth-bound (loading model weights + KV cache dominates)
- Loves high memory bandwidth (H200's 4.8 TB/s is 40% better than H100's 3.35 TB/s)
- N tokens = N forward passes
- Duration: linear in output length, per-step latency dominates

**Why this matters:**

- The best GPU for prefill is not the best GPU for decode
- The best batching strategy for prefill is different from decode
- The dominant cost differs: prefill is compute cost, decode is memory bandwidth cost
- Disaggregated architectures (DistServe, Splitwise) split them onto DIFFERENT GPUs

**Roofline model:**

Every operation is either compute-bound or memory-bound based on arithmetic intensity (FLOPs per byte moved). Prefill: high arithmetic intensity (batched matmuls) → compute-bound. Decode: low arithmetic intensity (one query dot-producted against KV cache) → memory-bound.

**Interview signal:** discussing "we run prefill on H100s and decode on H200s because the roofline differs" shows frontier-lab thinking.

---

## Q5. Explain PagedAttention. Why did it change everything? ⭐⭐⭐⭐

**The problem before PagedAttention:**

KV cache was allocated contiguously per request. Length-guessed at max context. Wasted memory on unused capacity. Fragmentation when requests finished at different times.

Result: batch size ceiling far below what memory could theoretically hold.

**PagedAttention (Kwon et al., 2023, vLLM paper):**

Inspired by virtual memory paging in operating systems. KV cache is allocated in FIXED-SIZE BLOCKS (typically 16 tokens). Blocks are not contiguous. A page table maps logical positions in a request's sequence to physical blocks.

**Wins:**
- Near-zero memory waste (only last partial block per request)
- Requests grow as they generate; new blocks allocated dynamically
- Requests share blocks (prefix caching works naturally at block granularity)
- Batch size 2-4× higher than naive contiguous allocation

**Cost:**
- Custom attention kernel needed (indirect memory access)
- Block table overhead (small)

**Impact:**
- vLLM leveraged this to become the default open-source LLM serving framework
- TensorRT-LLM, TGI have adopted similar patterns
- The single biggest memory optimization in LLM serving history

**Interview signal:** knowing that PagedAttention block size is typically 16, and understanding why (balances fragmentation vs table overhead) shows depth.

---

## Q6. Explain FlashAttention v1 vs v2 vs v3. Why do they matter? ⭐⭐⭐⭐

**The core insight (v1, Dao et al., 2022):**

Standard attention computes Q×K^T (huge intermediate matrix), softmax, times V. The intermediate matrix is written to and read from HBM (slow).

FlashAttention: reorder the computation. Tile the attention over blocks of Q, K, V. Compute softmax incrementally. Never materialize the full attention matrix in HBM.

Result: 2-4× faster attention, LESS memory (linear in sequence length instead of quadratic).

**v2 (2023):**

Better parallelism strategy — parallelize across sequence length in addition to batch and heads. Better work partitioning between warps in a CUDA block. Optimized for A100 and H100.

Result: another ~2× speedup over v1.

**v3 (2024):**

Optimized specifically for H100's Hopper architecture:
- Warpgroup Matrix Multiply-Accumulate (WGMMA) instructions
- Async operations to overlap computation with memory
- Better use of Tensor Memory Accelerator (TMA)

Result: ~1.5-2× over v2 on H100.

**Interview signal:** naming the specific hardware features (WGMMA, TMA) shows real hardware literacy.

**Practical:** FlashAttention is now default in every serious inference framework. You get it for free. Understanding it matters for interviews and for debugging when it doesn't work (unsupported attention variants, custom shapes).

---

## Q7-Q30: Additional Deep Conceptual Questions (Condensed)

### Q7. Explain MHA vs MQA vs GQA vs MLA — the attention taxonomy. ⭐⭐⭐⭐
**MHA (Multi-Head Attention):** original transformer, N Q heads and N KV heads. Maximum quality, maximum KV cache. **MQA (Multi-Query Attention):** N Q heads, 1 KV head. Massive KV reduction (~64×), small quality loss. Used in Falcon, PaLM. **GQA (Grouped-Query Attention):** N Q heads in G groups sharing KV heads. Middle ground. Llama-2/3/3.1 use GQA (Llama-70B: 64 Q heads, 8 KV heads, group size 8). **MLA (Multi-Head Latent Attention):** DeepSeek's approach. Compresses KV to low-rank latent representation. Extreme KV reduction with retained quality. **Selection:** GQA is the modern default; MLA is emerging for extreme contexts.

### Q8. Explain speculative decoding acceptance rate math. ⭐⭐⭐⭐
Draft model generates K tokens fast (e.g., 5 tokens ahead). Target model verifies all K in parallel (one forward pass). Accepted tokens use draft's fast generation; rejected tokens require redo. **Speedup formula:** `speedup ≈ (α + α² + ... + α^K) / cost_ratio` where α is per-token acceptance rate. Typical: α = 0.65, K = 5, speedup ≈ 2.2×. Higher acceptance → higher speedup. Draft model choice matters: too small = low acceptance; too large = draft cost eats speedup. Sweet spot: draft ~10% size of target. **When it fails:** highly random workloads (creative writing), where draft model's guesses rarely match target.

### Q9. GPTQ vs AWQ vs GGUF vs SmoothQuant vs FP8 — quantization decision framework. ⭐⭐⭐⭐
**GPTQ:** post-training weight quantization, INT4 or INT8. Uses Hessian-based error correction. High quality preservation. Weight-only (activations FP16). **AWQ:** activation-aware quantization. Preserves salient weights based on activation magnitudes. Slightly better quality than GPTQ at INT4. **GGUF:** llama.cpp's format for CPU/mixed inference. Various bit widths. Best for edge/self-hosted-on-consumer-hardware. **SmoothQuant:** activation smoothing enables INT8 quantization of both weights AND activations. Better throughput than weight-only. Quality nuance. **FP8:** native FP8 support on H100/H200. INT4 accuracy, INT8 throughput, no calibration needed. The future. **KV cache quantization (INT4/FP8):** separate axis — quantize the KV cache. Enables higher batch sizes. **Selection:** on H100+ use FP8 weights + FP8 KV. Fallback: AWQ INT4 weights + FP8 KV. Consumer GPU: GGUF.

### Q10. Tensor Parallelism vs Pipeline Parallelism vs Data Parallelism vs Expert Parallelism vs Sequence Parallelism ⭐⭐⭐⭐
**Data Parallelism (DP):** each GPU has full model, different batches. Easy to scale, unlimited by model size. **Tensor Parallelism (TP):** shard weight matrices across GPUs, high communication. Requires fast interconnect (NVLink 900 GB/s). Latency-friendly (parallel matmuls). Common: TP=4 or TP=8 within a node. **Pipeline Parallelism (PP):** different GPUs hold different layers. Sequential execution creates bubbles (idle time). Latency-hostile. Useful across nodes (Ethernet-tolerant). **Expert Parallelism (EP):** for MoE models — different experts on different GPUs. Routing overhead. Requires careful load balancing. **Sequence Parallelism (SP):** splits sequence dimension. Used with TP to shard activations too. Reduces per-GPU activation memory. **Selection:** TP within a node (NVLink), DP across nodes (Ethernet OK), PP only when model exceeds node capacity, EP for MoE, SP as complement to TP.

### Q11. NVLink vs InfiniBand vs Ethernet — why interconnect drives architecture. ⭐⭐⭐
**NVLink 4:** 900 GB/s per GPU, intra-node. Enables TP across 8 GPUs in a node. **NVSwitch:** full-mesh NVLink between 8+ GPUs. **InfiniBand HDR:** 200 Gbps = 25 GB/s per link. Multi-node communication. **InfiniBand NDR:** 400 Gbps. Latest. **Ethernet 100 GbE:** 12.5 GB/s. 30-70× slower than NVLink. **RDMA (RoCE):** RDMA over Ethernet. Close to InfiniBand latency at Ethernet cost. **Impact on architecture:** TP requires all-reduce EVERY LAYER (huge bandwidth demand). Fits on NVLink, dies on Ethernet. PP requires activation transfer BETWEEN STAGES (much less bandwidth). Fits on Ethernet. So: TP within a node (fast), PP or DP across nodes (tolerant).

### Q12. Automatic Prefix Caching (APC) — what is it and when does it help? ⭐⭐⭐⭐
APC detects when incoming requests share a prefix (common system prompt, few-shot examples). Reuses the KV cache from previous requests instead of recomputing. **Hit rate depends on workload:** stable system prompts → 80-95% hit rate. Random user queries → near 0%. **Savings:** prefill compute proportional to hit rate. 85% hit rate → 85% prefill savings. **Implementation:** hash the prefix tokens, store block references in a trie or hashmap. vLLM's APC is transparent (no config needed). **When it wins:** RAG products, agent systems, chatbots with system instructions. **When it doesn't:** ad-hoc chat without shared context.

### Q13. RadixAttention (SGLang) vs vLLM's APC — what's different? ⭐⭐⭐
Both share KV across requests. **APC:** prefix-only sharing. Radix trees or hash-based lookups. **RadixAttention (SGLang):** more general prefix tree that supports branching (e.g., agent that forks execution and shares common context across branches). Better for complex agent workflows where sub-tasks share context. **Practical:** SGLang wins on agent-heavy workloads; APC wins on standard chat. **Interview signal:** knowing that SGLang's DSL is designed around this makes RadixAttention especially useful.

### Q14. KV cache offloading — HBM to CPU to NVMe. ⭐⭐⭐
When KV won't fit in GPU HBM, tier storage:
- HBM (80 GB per H100, 3.35 TB/s): active requests
- CPU DRAM (up to TB, ~100 GB/s over PCIe): recent-but-idle requests
- NVMe (10s of TB, ~10 GB/s): long-context or paused requests
**When to use:** very long context (100K+), high concurrency at moderate context, paused conversations. **Trade-off:** offloaded requests pay high latency when resumed (transfer back to HBM). **Frameworks:** vLLM has CPU offloading; production usage still emerging.

### Q15. Constrained decoding — JSON mode internals. ⭐⭐⭐
Every next-token distribution filtered against a grammar (allowed states). At each step, invalid tokens get logit=-inf. Model can only sample valid tokens. **Implementations:** **Outlines** — precomputed FSM over vocabulary, fast at inference. **XGrammar** — better perf on JSON, adopted by vLLM. **guidance** — Microsoft's template-based approach. **Speed impact:** near-zero overhead when grammar is precompiled; 5-15% slowdown for on-the-fly grammar compilation. **When NOT to use:** free-form creative output where structure isn't required (adds constraint, may hurt quality subtly).

### Q16-Q30: Additional Deep Conceptual Questions (Condensed)

### Q16. When to self-host vs stay on API — the decision framework. ⭐⭐⭐⭐
Stay on API when: <1M tokens/day, unpredictable spikes, no ML infra team, cutting-edge models needed (GPT-4, Claude Opus). Self-host when: >10M tokens/day sustained, cost is critical, data residency required (regulated industry), custom fine-tuned models needed, extreme latency requirements. Break-even math typically shows benefit above 100M tokens/month for teams with existing infra.

### Q17. MoE (Mixture of Experts) inference challenges. ⭐⭐⭐⭐
MoE routes tokens to different experts per layer. Sparse activation (2 of 8 experts active) → lower compute per token but total weights massive. Serving challenges: **memory** (must hold all experts even if only 2 active), **expert parallelism** (shard experts across GPUs), **load balancing** (some experts more popular), **routing overhead** (gating decision every layer). Mixtral 8x7B, DeepSeek-V3 examples. Frameworks: vLLM and TensorRT-LLM both support MoE now.

### Q18. torch.compile for inference — helps or hurts? ⭐⭐⭐
torch.compile fuses operations, generates optimized kernels. **Helps** when: fixed model shape, warm cache after initial compilation. **Hurts** when: dynamic shapes (recompilation cost), development iteration (compile overhead), custom kernels already optimized. vLLM/TGI/TensorRT-LLM have their own kernel optimization; torch.compile mostly redundant with them. Useful for standalone HuggingFace serving.

### Q19. Kernel fusion — what's actually fused? ⭐⭐⭐
Combines multiple GPU operations into one kernel launch, reducing kernel launch overhead + memory round-trips. Common fusions: **QKV projection into one matmul**, **rotary position embedding into attention**, **layer norm + linear**, **SwiGLU activation + gate**. Impact: 10-30% latency reduction. Automatic in TensorRT-LLM (compiler-driven). Manual in vLLM (hand-written kernels). Modern inference frameworks fuse aggressively.

### Q20. Disaggregated prefill/decode (DistServe, Splitwise). ⭐⭐⭐⭐
Recognizes prefill vs decode compute asymmetry. Separates them onto different GPU pools. Prefill cluster: H100s (compute-optimized). Decode cluster: L40S or H200s (memory-bandwidth-optimized). **KV cache transfer** between pools when request moves from prefill to decode. **Wins:** each pool at high utilization on native workload, better cost per token, better p99 latency. **Costs:** complexity (KV transfer, request routing, failure modes), typically only worth it at 10M+ req/day scale.

### Q21. Multi-LoRA serving — how does it work? ⭐⭐⭐
Base model shared, LoRA adapters (small rank-decomposed matrices) applied per request. vLLM supports 100s of LoRAs on a single base model. **Serving pattern:** load base model once, load LoRAs on demand, apply LoRA to each request. **Cost:** modest per-request overhead. **Wins:** per-tenant fine-tuned models without deploying 100 model copies. **Enterprise use case:** SaaS product where each customer has fine-tuned adapter.

### Q22. Speculative decoding techniques — Medusa vs EAGLE vs draft model. ⭐⭐⭐⭐
**Draft model:** small separate model generates draft. Simple, works. Cost: draft model latency. **Medusa:** additional decoding heads on the base model predict multiple future tokens. No separate draft model. Requires training the Medusa heads. **EAGLE / EAGLE-2:** more sophisticated — uses base model's own features to predict future tokens. Higher acceptance rates. **Lookahead decoding:** parallel guess-and-verify without draft model. **N-gram speculation:** uses common n-grams from context as draft. Cheap, works for repetitive text.

### Q23. FP8 vs INT8 vs INT4 — precision trade-offs. ⭐⭐⭐
**INT8:** 2× memory savings vs FP16, well-supported, tools mature (SmoothQuant, etc.). **INT4:** 4× memory savings, quality risk, requires calibration (GPTQ, AWQ). **FP8 (E4M3, E5M2):** floating-point format at 8 bits. Wider dynamic range than INT8. Native H100+ support. Better quality than INT8 in practice. **INT4 KV cache:** aggressive KV compression, quality validation critical.

### Q24. TTFT vs TPOT — what interviewers really want you to explain. ⭐⭐⭐
**TTFT (Time To First Token):** dominated by prefill. Users perceive as responsiveness. Target: < 500ms for chat, < 1s for RAG. **TPOT (Time Per Output Token):** dominated by decode. Determines streaming speed. Target: < 50ms for chat feels natural. **E2E (End-to-End):** TTFT + N × TPOT. Total wait. Different optimizations for each: TTFT improved by prefix caching, chunked prefill, prefill hardware. TPOT improved by memory bandwidth, decode hardware, batch size (up to bandwidth ceiling).

### Q25. Benchmark pitfalls — the 10 mistakes candidates make. ⭐⭐⭐⭐
1. Not measuring warm-up separately (first request slow). 2. Sequential requests instead of concurrent (misses batching benefits). 3. Using averages instead of percentiles. 4. Single sequence length (real workloads have distribution). 5. Ignoring APC hit rate variance. 6. Client-side latency (network/serialization pollutes). 7. GPU not at steady state (thermal throttling later). 8. Comparing different quantization at different quality bars. 9. Cold start included in "throughput" measurement. 10. Not measuring quality — throughput at what quality?

### Q26. Model gateway pattern for enterprise inference. ⭐⭐⭐⭐
Single ingress for multiple models: base + fine-tuned + third-party APIs. Handles: routing (small model for easy queries, big for hard), auth (per-tenant), cost attribution, rate limits, retries, circuit breakers, semantic caching, request/response logging, PII scrubbing. Existing solutions: LiteLLM, Portkey, Cloudflare AI Gateway. Enterprises often build custom for full control.

### Q27. Multi-tenant inference isolation — what interviewers probe. ⭐⭐⭐⭐
Options: **Shared cluster with tenant tagging** — cost-efficient, weak isolation. **Dedicated LoRA per tenant on shared base** — good middle ground. **Dedicated model instances per tenant** — expensive, strong isolation. **Cross-tenant KV cache contamination:** MUST NOT SHARE. Prefix cache namespacing per tenant. Compliance drivers (HIPAA, regulated data) force stronger isolation.

### Q28. Warm pools and autoscaling for LLM inference. ⭐⭐⭐
Cold start on LLM inference = model load time (30s-3min for 70B models). Can't rely on Kubernetes HPA reactively. **Warm pool:** N model instances always running, absorbing spikes. **Predictive autoscaling:** scale ahead of traffic curves. **Warm standby models:** loaded on GPU but not serving (fast activation). **Graceful degradation:** if scaling can't keep up, fall back to smaller model or API.

### Q29. Blue-green deployment for GPU inference clusters. ⭐⭐⭐
Standard blue-green doesn't work — model instances take minutes to warm up. **Pattern:** warm green cluster with new model, verify golden set passes, canary 5% traffic to green, monitor quality + latency, gradual shift, keep blue as instant rollback. Cost: 2× GPU during transition. Necessary for quality-critical changes (quantization, model version).

### Q30. The frontier lab compensation signal — what makes you a hire. ⭐⭐⭐⭐
Frontier labs (Anthropic, OpenAI, Google DeepMind, Meta AI FAIR) hire people who can: **compute** hardware requirements from first principles, **debate** attention variants with paper citations, **quantify** improvements with percentiles + confidence intervals, **navigate** the systems stack across CUDA → framework → application, **balance** research literacy with production discipline. The specific signal isn't 'knows vLLM' — it's 'can defend WHY vLLM given workload X on hardware Y with SLO Z.' If your answer contains numbers, hardware families, and trade-off math, you're at the bar.
