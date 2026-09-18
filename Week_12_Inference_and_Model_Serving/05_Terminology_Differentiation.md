# 📖 Week 12 — Terminology Differentiation Reference

> **Focus:** Every set of similar-sounding inference terms that trips candidates up. Parallelism types, quantization formats, attention variants, batching strategies, precision formats.
>
> **How to use:** Frontier lab and MAANG interviewers love probing "what's the difference between X and Y?" precisely because it filters out candidates who use terms loosely. This file is your defense.

---

# Section 1: Parallelism Types (The Alphabet Soup)

## DP vs TP vs PP vs EP vs SP

### Data Parallelism (DP)

**Definition:** Every GPU has a full copy of the model. Different batches run on different GPUs.

**Communication:** none during forward pass (only gradient sync in training; nothing in inference).

**Fits when:** model fits on one GPU. Scale by adding replicas.

**Interconnect requirement:** low. Ethernet fine.

**Use when:** small-to-mid models, high concurrency, horizontal scaling.

**Don't use when:** model doesn't fit on one GPU.

---

### Tensor Parallelism (TP)

**Definition:** Shard weight matrices ACROSS GPUs. Each GPU holds a slice of every layer.

**Communication:** all-reduce EVERY layer (huge bandwidth demand).

**Fits when:** model too big for one GPU but nodes have fast interconnect.

**Interconnect requirement:** VERY HIGH. NVLink or NVSwitch mandatory. Ethernet kills TP.

**Use when:** need to shard a big model within a node (Llama-70B on 4×H100).

**Don't use when:** GPUs connected only via Ethernet — TP will be bandwidth-starved.

**Interview trap:** candidates say "use tensor parallelism" without knowing it requires NVLink. Say TP + NVLink together.

---

### Pipeline Parallelism (PP)

**Definition:** Different GPUs hold DIFFERENT LAYERS. Layer 1-10 on GPU 0, layers 11-20 on GPU 1, etc. Forward pass flows through the pipeline.

**Communication:** activation transfer between stages (small compared to TP).

**Interconnect requirement:** MEDIUM. Ethernet acceptable.

**Fits when:** model too big for one node. Use across nodes.

**Downside:** pipeline bubbles (idle time waiting for previous stage). Hurts p99 latency significantly.

**Use when:** MUST distribute model across nodes and can't use TP over network.

**Don't use when:** latency-sensitive (bubbles hurt), or model fits within a node (use TP).

---

### Expert Parallelism (EP)

**Definition:** For MoE models. Different EXPERTS on different GPUs. Router directs tokens to appropriate expert-GPU.

**Communication:** token routing overhead.

**Interconnect requirement:** MEDIUM-HIGH. Depends on routing pattern.

**Use when:** MoE model (Mixtral, DeepSeek-V3, GPT-4 architecture).

**Don't use when:** dense model (not MoE).

**Interview trap:** candidates conflate EP with TP. EP shards along the EXPERT axis; TP shards along the WEIGHT MATRIX axis. Different concepts.

---

### Sequence Parallelism (SP)

**Definition:** Shard along the SEQUENCE dimension. Combines with TP to shard activation memory too.

**Use when:** used with TP, especially for long-context training/inference. Reduces per-GPU memory ceiling.

**Don't use alone:** SP is a companion to TP, not a standalone strategy.

---

### 🎯 The Distinction That Matters

| Type | What's split | Interconnect need | When |
|---|---|---|---|
| **DP** | Batches | Low | Horizontal scaling |
| **TP** | Weight matrices | Very high (NVLink) | Fit big model in node |
| **PP** | Layers | Medium (Ethernet OK) | Fit model across nodes |
| **EP** | MoE experts | Medium-high | MoE models |
| **SP** | Sequence dimension | Same as TP | Long context + TP |

**Real production:** TP=4 or 8 within a node + DP across nodes = most common. PP only when unavoidable.

---

# Section 2: Attention Variants

## MHA vs MQA vs GQA vs MLA

### Multi-Head Attention (MHA)

**Definition:** N query heads, N key heads, N value heads. All N sets of K and V computed.

**Quality:** maximum.

**KV cache:** maximum (baseline).

**Use in:** original transformer, older models, when quality is paramount.

**Interview trap:** MHA is not "multi-headed" in a generic sense — it specifically means EQUAL numbers of Q, K, V heads.

---

### Multi-Query Attention (MQA)

**Definition:** N query heads, 1 key head, 1 value head. All queries share the single K and V.

**Quality:** slight loss vs MHA.

**KV cache:** ~N× smaller than MHA. Massive reduction.

**Use in:** Falcon, PaLM, small models where inference speed dominates.

**Don't use when:** quality is critical and you can afford MHA's KV overhead.

---

### Grouped-Query Attention (GQA)

**Definition:** N query heads in G groups. Each group shares K and V heads. Interpolates between MHA (G=N) and MQA (G=1).

**Example:** Llama-70B has 64 Q heads and 8 KV heads. Group size = 8.

**Quality:** near-MHA at reasonable group sizes.

**KV cache:** N/G× smaller than MHA. Llama-70B GQA = 8× KV cache reduction.

**Use in:** Llama-2/3/3.1, most modern models. Default choice.

---

### Multi-Head Latent Attention (MLA)

**Definition:** DeepSeek's approach. Compresses KV to a low-rank latent representation. Decompresses at attention time.

**Quality:** competitive with MHA.

**KV cache:** extreme reduction (up to 90%+).

**Use in:** DeepSeek-V2, DeepSeek-V3. Emerging.

**Trade-off:** more complex kernel implementation.

---

### 🎯 The Distinction That Matters

| Attention | Q heads | K heads | V heads | KV vs MHA | Notes |
|---|---|---|---|---|---|
| **MHA** | N | N | N | 1× (baseline) | Max quality |
| **MQA** | N | 1 | 1 | 1/N | Extreme reduction |
| **GQA (group=G)** | N | N/G | N/G | 1/G | Modern default |
| **MLA** | N | latent | latent | ~10× smaller | Cutting edge |

**Interview signal:** knowing that Llama-70B uses GQA with 64:8 ratio = 8× KV reduction shows real model literacy.

---

# Section 3: Batching Strategies

## Static vs Dynamic vs Continuous vs In-Flight vs Chunked Prefill

### Static Batching

**Definition:** Wait for exactly N requests, forward-pass together, return N responses. Whole batch waits for slowest request.

**Use for:** classifiers, embeddings, anything with fixed output length.

**Don't use for:** LLM generation. Ever.

---

### Dynamic Batching

**Definition:** Wait for N requests OR T milliseconds. Batch what arrives.

**Improvement over static:** doesn't wait forever.

**Use for:** Triton Inference Server workloads, non-generative models.

**Don't use for:** LLM generation (still has variable-length problem).

---

### Continuous Batching

**Definition:** Token-level scheduling. New requests join the batch every decode step. Finished requests drop out.

**Use for:** LLM generation. Default in vLLM, TGI, all modern LLM servers.

**Don't use when:** non-generative model (unnecessary complexity).

---

### In-Flight Batching

**Definition:** TensorRT-LLM's term for continuous batching. Same concept, NVIDIA branding.

**Interview trap:** candidates think in-flight batching is a distinct third concept. It's not — it's the same paradigm.

---

### Chunked Prefill

**Definition:** Break long prefills into chunks (e.g., 512 tokens per step), interleave with decode.

**Use with:** continuous batching. Solves the "long prompt starves decode" problem.

**Use when:** workload has variable prompt lengths, especially long prompts.

---

### 🎯 The Distinction That Matters

| Strategy | Level of granularity | Fixes | Use for LLMs? |
|---|---|---|---|
| **Static** | Request | Nothing | Never |
| **Dynamic** | Request + timeout | Wait forever | Non-generative only |
| **Continuous** | Token | Variable length | Yes (default) |
| **In-flight** | Token (TRT-LLM's name) | Same as continuous | Yes (TRT-LLM) |
| **Chunked prefill** | Sub-request | Long prompt starving decode | Yes (advanced) |

**When someone says "we use in-flight batching" they mean the same thing as "we use continuous batching" — TensorRT-LLM users say the former, vLLM users say the latter.**

---

# Section 4: Quantization Formats

## GPTQ vs AWQ vs GGUF vs SmoothQuant vs FP8

### GPTQ (Post-Training Quantization)

**Definition:** Weight quantization using Hessian-based error correction. INT4 or INT8.

**Scope:** weights only (activations remain FP16).

**Quality:** high preservation with proper calibration.

**Use for:** open-source models on NVIDIA GPUs when you want smaller weights.

---

### AWQ (Activation-Aware Weight Quantization)

**Definition:** Preserves salient weights based on activation magnitude. INT4 quantization.

**Scope:** weights only.

**Quality:** slightly better than GPTQ at INT4.

**Use for:** aggressive INT4 quantization while preserving quality.

---

### GGUF (llama.cpp Format)

**Definition:** File format from llama.cpp ecosystem. Supports various bit widths (Q4_K_M, Q5_K_M, Q8_0, etc.).

**Scope:** whole model + metadata.

**Use for:** CPU inference, edge devices, Apple Silicon, consumer GPUs.

**Don't use for:** production H100 serving (GPTQ/AWQ/FP8 better).

---

### SmoothQuant

**Definition:** Migrates quantization difficulty from activations to weights. Enables INT8 for BOTH.

**Scope:** weights AND activations at INT8.

**Use for:** INT8 throughput improvements when weight-only INT4 quality is insufficient.

---

### FP8 (E4M3, E5M2)

**Definition:** 8-bit floating point. Two variants: E4M3 (4 exponent, 3 mantissa) and E5M2.

**Scope:** weights, activations, KV cache — any tensor.

**Quality:** superior to INT8 due to wider dynamic range.

**Hardware:** native support on H100+ (Hopper), Ada Lovelace L40S, upcoming Blackwell.

**Use for:** new deployments on H100+. Future default.

---

### 🎯 The Distinction That Matters

| Format | Precision | Weights | Activations | KV | Best for |
|---|---|---|---|---|---|
| **GPTQ** | INT4/8 | ✅ | ❌ | ❌ | Standard quantization |
| **AWQ** | INT4 | ✅ | ❌ | ❌ | Aggressive INT4 |
| **GGUF** | Various | ✅ | ✅ (some) | ✅ | Edge/CPU |
| **SmoothQuant** | INT8 | ✅ | ✅ | ❌ | INT8 throughput |
| **FP8** | FP8 | ✅ | ✅ | ✅ | H100+ future default |

**Recommendation stack (2026 production):**
- New H100+ deployment: FP8 weights + FP8 KV cache
- Older hardware / max compression: AWQ INT4 + FP8 KV
- Edge/consumer: GGUF (llama.cpp)

---

# Section 5: Latency Metrics

## TTFT vs TPOT vs E2E vs Throughput

### TTFT (Time To First Token)

**Definition:** Time from request received to FIRST output token.

**Dominated by:** prefill compute + queuing.

**User perception:** responsiveness. Low TTFT = "the AI is thinking."

**Target:** < 500ms for chat, < 1s for RAG.

**Optimization levers:** prefix caching, chunked prefill, prefill hardware, batch size tuning.

---

### TPOT (Time Per Output Token)

**Definition:** Average time between consecutive output tokens.

**Dominated by:** decode phase, memory bandwidth.

**User perception:** typing speed. Low TPOT = "the AI types fast."

**Target:** < 50ms/token feels natural.

**Optimization levers:** memory bandwidth (hardware choice), quantization (weight and KV), speculative decoding.

---

### E2E Latency

**Definition:** Total time from request to complete response.

**Formula:** TTFT + (N-1) × TPOT where N = output tokens.

**Use when:** measuring total user wait.

**Don't use when:** streaming responses (users start reading before E2E completes).

---

### Throughput

**Definition:** Tokens per second aggregate across all concurrent requests.

**Different from latency:** throughput is system-level; latency is per-request.

**Trade-off:** high throughput often means larger batches which increases TTFT and TPOT for individual requests.

**Target:** dictated by capacity planning, not user experience.

---

### 🎯 The Distinction That Matters

| Metric | User feels? | Dominated by | Fix by |
|---|---|---|---|
| **TTFT** | Responsiveness | Prefill | Prefix cache, hardware |
| **TPOT** | Typing speed | Decode | Memory bandwidth, quantization |
| **E2E** | Total wait | Both | Balance both |
| **Throughput** | Nothing directly | Batching | Higher batches (trades latency) |

**Interview trap:** candidates optimize "latency" without specifying which one. Senior signal: name TTFT + TPOT + E2E specifically, know which optimizations move which.

---

# Section 6: Framework Taxonomies

## vLLM vs TGI vs TensorRT-LLM vs SGLang

### vLLM

**Origin:** UC Berkeley (Kwon et al., PagedAttention paper).

**Key innovations:** PagedAttention, continuous batching, Automatic Prefix Caching.

**Language:** Python + custom CUDA kernels.

**Sweet spot:** general-purpose LLM serving. Great docs, huge community.

**Use when:** default choice. Most workloads start here.

---

### TGI (Text Generation Inference)

**Origin:** HuggingFace.

**Key innovations:** Rust-based router, tightly integrated with HuggingFace ecosystem.

**Language:** Rust router + Python model.

**Sweet spot:** teams already deep in HuggingFace ecosystem, need production Rust performance for routing.

**Use when:** HF-centric team, want Rust ops layer.

---

### TensorRT-LLM

**Origin:** NVIDIA.

**Key innovations:** compile-based approach (engine builds), kernel fusion, in-flight batching.

**Language:** C++ + Python.

**Sweet spot:** raw NVIDIA GPU performance, locked-in on NVIDIA stack.

**Use when:** every ounce of NVIDIA GPU performance matters, willing to pay complexity cost of engine builds.

**Don't use when:** rapid iteration (rebuilding engines is slow), non-NVIDIA hardware.

---

### SGLang

**Origin:** LMSYS (Berkeley).

**Key innovations:** RadixAttention (prefix tree for KV sharing including branches), DSL frontend for complex generation patterns.

**Language:** Python.

**Sweet spot:** agent workloads, complex generation flows, prefix-heavy workloads.

**Use when:** agent systems with shared context branching, complex JSON generation.

---

### 🎯 The Distinction That Matters

| Framework | Killer feature | Best for | Weakness |
|---|---|---|---|
| **vLLM** | PagedAttention + APC | General LLM serving | None major — default choice |
| **TGI** | Rust router | HF-centric teams | Feature velocity behind vLLM |
| **TensorRT-LLM** | Compiled kernels | Peak NVIDIA perf | Engine build complexity |
| **SGLang** | RadixAttention + DSL | Agent workloads | Smaller ecosystem |

**No single winner** — workload determines choice. Interviewers respect nuanced answers over "vLLM is best."

---

# Section 7: Hardware Terminology

## HBM vs SRAM vs Memory Bandwidth vs FLOPS

### HBM (High Bandwidth Memory)

**Definition:** Stacked DRAM on GPU package. Where model weights and KV cache live.

**H100 SXM:** HBM3, 80 GB, 3.35 TB/s bandwidth.
**H200:** HBM3e, 141 GB, 4.8 TB/s bandwidth.
**B200:** HBM3e, 192 GB, 8 TB/s bandwidth.

**Interview trap:** HBM is not "RAM" in the CPU sense. Much faster, much more expensive per GB.

---

### SRAM (Static RAM, aka "on-chip")

**Definition:** Ultra-fast memory ON the GPU die. Registers, shared memory, L1/L2 cache.

**Size:** small (MB range).

**Speed:** 10× to 100× faster than HBM.

**Role:** where FlashAttention keeps intermediate values to avoid HBM round-trips.

---

### Memory Bandwidth

**Definition:** How fast data moves from HBM to compute units. Measured in TB/s.

**Why it matters for decode:** decode is memory-bandwidth-bound. Reading model weights + KV cache dominates time.

**Rule:** faster HBM = faster decode.

---

### FLOPS (Floating Point Operations Per Second)

**Definition:** Raw compute throughput.

**H100 SXM:** ~1000 TFLOPS (FP16), ~2000 TFLOPS (FP8).
**H200:** same compute as H100, more memory bandwidth.
**B200:** ~2500 TFLOPS (FP16), ~5000 TFLOPS (FP8).

**Why it matters for prefill:** prefill is compute-bound. More FLOPS = faster prefill.

---

### 🎯 The Distinction That Matters

| Concept | What it measures | Matters for | Optimize by |
|---|---|---|---|
| **HBM capacity** | Model + KV space | Batch size | Quantization, smaller model |
| **HBM bandwidth** | Data movement speed | Decode (TPOT) | H200/B200 upgrade |
| **SRAM** | On-chip fast memory | FlashAttention efficiency | Software |
| **FLOPS** | Raw compute | Prefill (TTFT) | Better GPU (B200) |

**Prefill wants FLOPS. Decode wants bandwidth. Different GPUs excel at different phases.**

---

# Section 8: Deployment Patterns

## Single-GPU vs Multi-GPU vs Multi-Node vs Disaggregated

### Single-GPU Serving

**Fits:** small models (7B) or heavily quantized larger models on H100.

**Simplest:** one process, one GPU, no coordination.

**Scale out:** replicas via DP.

---

### Multi-GPU (Within Node)

**Fits:** models needing 100+ GB (Llama-70B).

**Pattern:** TP across GPUs within node. NVLink handles communication.

**Common:** TP=4 on 4×H100, TP=8 on 8×H100.

---

### Multi-Node Serving

**Fits:** models exceeding node capacity (Llama-405B, GPT-4-scale).

**Patterns:**
- TP within node, PP across nodes (most common)
- TP across nodes (only with high-end interconnect like GB200 NVL72)

**Complexity:** high. Network configuration matters.

---

### Disaggregated Prefill/Decode

**Recognizes:** prefill and decode have different resource profiles.

**Pattern:** separate GPU pools. Prefill on H100 (compute-optimized). Decode on H200 or L40S (memory-optimized).

**KV cache:** transferred between pools when request moves from prefill to decode.

**Use when:** high scale, need optimal cost per token.

---

### 🎯 The Distinction That Matters

Progression: **single-GPU → multi-GPU (same node) → multi-node → disaggregated**

Each step adds complexity. Only take the next step when the previous can't handle load. Don't over-engineer.

---

# Section 9: Cost Terminology

## Cost per Token vs Cost per Request vs GPU-Hour vs Amortized TCO

### Cost per Token

**Definition:** dollar cost per output token (or blended input+output).

**Formula:** total infrastructure cost / total tokens generated.

**Use for:** provider comparison, self-host vs API break-even.

---

### Cost per Request

**Definition:** dollar cost per user-facing request.

**Formula:** cost per token × avg tokens per request.

**Use for:** feature economics, per-user analysis.

---

### GPU-Hour

**Definition:** cost of one GPU running for one hour.

**H100 SXM spot:** ~$3.60/hr per GPU (2026).
**H100 on-demand:** ~$6-8/hr per GPU.
**Reserved 1yr:** ~40% off on-demand.

**Use for:** capacity planning.

---

### Amortized TCO (Total Cost of Ownership)

**Definition:** ALL costs allocated to inference: GPU depreciation, power, cooling, engineering time, observability tools, incident response.

**Use for:** honest comparison to API pricing.

**Interview trap:** self-host math ignores TCO half the time. Include engineering headcount cost when comparing to API.

---

### 🎯 The Distinction That Matters

| Metric | What it captures | When to use |
|---|---|---|
| **Cost per token** | Provider pricing comparison | Break-even math |
| **Cost per request** | Feature economics | Product decisions |
| **GPU-hour** | Raw capacity cost | Capacity planning |
| **Amortized TCO** | Honest total cost | Executive decisions |

**Presenting cost per token without acknowledging TCO is junior. Presenting TCO with engineering time included is senior.**

---

## 🎯 The Meta-Lesson

Frontier lab compensation lives at the intersection of these terminologies. When you can speak in the specific vocabulary:

> "For Llama-70B decode phase which is bandwidth-bound, we chose H200 (HBM3e at 4.8 TB/s) over H100 (HBM3 at 3.35 TB/s). Combined with FP8 KV cache quantization, we got 43% TPOT improvement without touching FLOPS-limited prefill hardware."

vs the loose version:

> "We used better GPUs and quantization to make it faster."

The specific version gets the offer. The loose version gets the "we'll be in touch."
