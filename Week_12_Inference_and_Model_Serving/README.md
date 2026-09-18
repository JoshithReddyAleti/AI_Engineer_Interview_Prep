# ⚙️ Week 12 — Inference and Model Serving: The Systems Under the Model

> **Maps to:** [Episode_12_Inference_and_Model_Serving_The_Systems_Under_the_Model](https://github.com/JoshithReddyAleti/Episode_12_Inference_and_Model_Serving_The_Systems_Under_the_Model)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the layer under every LLM API call. This is where frontier lab compensation lives.**

If you can explain to a hiring manager why continuous batching + PagedAttention + speculative decoding + tensor parallelism together get 12× throughput over naive HuggingFace serving — with numbers, trade-offs, and hardware awareness — you can walk into any inference role at Anthropic, OpenAI, Google DeepMind, Meta AI, Together AI, Databricks, Fireworks, Groq, or any serious inference infrastructure team.

Most AI engineers can't. They know the API. They don't know what's under it.

Week 12 fixes that.

**The signal you're sending to interviewers:**
- Layer 1 (junior): "I use OpenAI's API."
- Layer 2 (mid): "I know about vLLM."
- Layer 3 (senior): "I can compare vLLM vs TGI vs TensorRT-LLM vs SGLang and pick one per workload."
- Layer 4 (staff+): "I've computed Llama-70B's KV cache at 328 KB/token, run the break-even math on API vs self-host, tuned continuous batching to hit 12× throughput, and can debate MHA vs MQA vs GQA vs MLA memory trade-offs from first principles."

Layer 4 is where MAANG and frontier labs hire from. That's what this week teaches.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 30 | KV cache math, PagedAttention, continuous batching, speculative decoding, all quantization formats, all parallelism types, framework comparisons — plus 3 flagship "worth mastering" questions |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 17 | Compute KV cache memory, continuous batching scheduler, prefix cache, speculative acceptance rate, cost-per-token calculator, quantization pipeline, benchmark harness |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | Enterprise inference platform, multi-model gateway, disaggregated prefill/decode, multi-tenant with per-tenant SLOs, cost-optimized migration from API |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 17 | $87K/month OpenAI bill, GPU shortage forcing hardware pivot, quantization broke quality, KV cache OOM in prod, tensor parallelism deadlock, cold model swap |
| [05_Terminology_Differentiation.md](05_Terminology_Differentiation.md) | Reference | TP vs PP vs DP vs EP vs SP, MHA vs MQA vs GQA vs MLA, GPTQ vs AWQ vs GGUF vs SmoothQuant vs FP8, static vs dynamic vs continuous vs in-flight batching — every set of similar-sounding inference terms |

## The 19 Sections Covered (Full Episode 12)

| # | Section | Key Interview Territory |
|---|---|---|
| 1 | Foundations | Inference vs training profiles, memory-bound vs compute-bound, prefill vs decode asymmetry, when to self-host |
| 2 | Transformer Inference Internals | KV cache math (Llama-70B = 328 KB/token derived from first principles), FlashAttention v1/v2/v3, PagedAttention, MHA/MQA/GQA/MLA |
| 3 | Batching Strategies | Static → dynamic → continuous → in-flight → chunked prefill (each solves a specific problem the previous didn't) |
| 4 | KV Cache Management | Paged allocation, prefix caching, Automatic Prefix Caching, RadixAttention, offloading tiers (HBM→CPU→NVMe) |
| 5 | Speculative Decoding | Draft-and-verify, Medusa heads, EAGLE/EAGLE-2, lookahead, n-gram, acceptance rate math |
| 6 | Quantization | GPTQ vs AWQ vs GGUF vs SmoothQuant vs FP8 vs INT4 KV — quality vs throughput vs deployment trade-offs |
| 7 | Parallelism | DP/TP/PP/EP/SP with NVLink 900GB/s vs Ethernet numbers — when each, why each |
| 8 | vLLM Deep Dive | Production setup, LoRA serving, top 10 production gotchas |
| 9 | TGI Deep Dive | Rust router architecture, honest vs-vLLM comparison |
| 10 | TensorRT-LLM Deep Dive | Compile-based approach, engine building, kernel fusion, Triton integration |
| 11 | SGLang Deep Dive | RadixAttention, DSL frontend, when it wins |
| 12 | Constrained Decoding | JSON mode internals, Outlines, XGrammar, logit masking speed impact |
| 13 | Model Optimization | Kernel fusion, KV compression (H2O, KIVI), MoE serving, torch.compile |
| 14 | Serving Architectures | Single-GPU → multi-GPU → multi-node → disaggregated prefill/decode → multi-LoRA → multi-model |
| 15 | Benchmarking & Measurement | TTFT, TPOT, throughput, 10 benchmark pitfalls |
| 16 | Cost Economics | Worked Llama-70B on 4×H100 = $0.97/M tokens, break-even math, TCO |
| 17 | Hardware Deep Dive | H100/H200/B200/A100/L40S/MI300X — memory bandwidth ranking, when each |
| 18 | Production Operations | Canary, shadow deploys, autoscaling GPUs, warm pools, graceful degradation |
| 19 | Enterprise Patterns | Model gateway, multi-tenant isolation, cross-region, platform design |

## Why This Week Is Different

Previous weeks covered the AI ENGINEERING stack: how to build LLM applications.

**Week 12 covers the SYSTEMS ENGINEERING under the models.** This is a distinct discipline. It's where:

- ML engineers meet HPC (high-performance computing) engineers
- Software engineering meets hardware topology
- Cost engineering becomes GPU-hour engineering
- Latency engineering becomes memory-bandwidth engineering

**Every serious AI infrastructure role at MAANG or frontier labs probes this layer hard.** Because at scale — Anthropic's Claude, OpenAI's GPT, Google's Gemini, Meta's Llama, Databricks' MosaicML — inference cost is the biggest line item, and inference latency is the biggest UX signal.

## The 4-Layer Mental Model of Inference

```
┌─────────────────────────────────────────────────────────┐
│ Layer 4: OPTIMIZATION LAYER                             │
│ Speculative decoding, quantization, KV compression,     │
│ continuous batching, prefix caching                     │
│ — Answers: "How do I get more throughput per GPU?"     │
├─────────────────────────────────────────────────────────┤
│ Layer 3: SERVING FRAMEWORK LAYER                        │
│ vLLM, TGI, TensorRT-LLM, SGLang                        │
│ — Answers: "What runtime hosts my model?"               │
├─────────────────────────────────────────────────────────┤
│ Layer 2: PARALLELISM LAYER                              │
│ Tensor/Pipeline/Data/Expert/Sequence parallelism        │
│ — Answers: "How do I fit and scale this model?"         │
├─────────────────────────────────────────────────────────┤
│ Layer 1: HARDWARE LAYER                                 │
│ GPUs, memory bandwidth (HBM3e = 4.8 TB/s), NVLink,     │
│ InfiniBand, RDMA                                        │
│ — Answers: "What silicon am I running on?"              │
└─────────────────────────────────────────────────────────┘
```

Every interview question in this week maps to one or more of these layers. The senior signal is being able to reason **across** layers — "we hit a memory bandwidth ceiling at Layer 1, so we changed our attention pattern at Layer 3 to MQA, quantized the KV cache at Layer 4, and reduced TP from 8 to 4 at Layer 2." That kind of multi-layer reasoning is the frontier lab bar.

## Enterprise Themes (What Interviewers Actually Probe)

**Cost as a first-class engineering constraint:**
- Break-even math: at what token volume does self-hosting beat API?
- Cost per million tokens (Llama-70B on 4×H100 at 65% util = ~$1.50/M)
- TCO including GPU depreciation, power, cooling, engineering time
- Cost dashboards per tenant, per model, per feature

**Latency as a compound metric:**
- TTFT (time to first token) — prefill-dominated
- TPOT (time per output token) — decode-dominated
- E2E latency — user-perceived
- Different optimizations attack different components

**Hardware topology awareness:**
- NVLink 4 = 900 GB/s intra-node
- InfiniBand HDR = 200 Gbps inter-node
- Ethernet 100 GbE = 12.5 GB/s (30x slower than NVLink)
- Model parallelism strategy depends heavily on this

**Silent failure modes:**
- KV cache OOM under bursty traffic
- Quantization quality regression detected weeks later
- Model version drift (subtle even in self-hosted)
- Continuous batching starvation for long-context requests

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1 | LLM Fundamentals | [Understanding_LLMs_From_The_Inside_Out](https://github.com/JoshithReddyAleti/Understanding_LLMs_From_The_Inside_Out) | ✅ |
| 2 | Python for AI | [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters) | ✅ |
| 3 | Tool Calling, APIs & Validation | [Building_AI_Project-Blueprint_for_Begin](https://github.com/JoshithReddyAleti/Building_AI_Project-Blueprint_for_Begin) | ✅ |
| 4 | End-to-End AI Projects | [Your_First_End_To_End_AI_Project](https://github.com/JoshithReddyAleti/Episode_4_Your_First_End_To_End_AI_Project) | ✅ |
| 5 | RAG & Augmented Generation | [Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation) | ✅ |
| 6 | Frameworks & Fine-Tuning | [AI_Frameworks_and_Fine_Tuning_Complete_Guide](https://github.com/JoshithReddyAleti/Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide) | ✅ |
| 7 | Memory & State in AI Systems | [Memory_and_State_in_AI_Systems](https://github.com/JoshithReddyAleti/Episode_7_Memory_and_State_in_AI_Systems) | ✅ |
| 8 | Evaluation, Validation & Governance | [AI_Evaluation_Validation_and_Governance](https://github.com/JoshithReddyAleti/Episode_8_AI_Evaluation_Validation_and_Governance) | ✅ |
| 9 | Agents: When AI Systems Make Decisions | [Agents_When_AI_Systems_Make_Decisions](https://github.com/JoshithReddyAleti/Episode_9_Agents_When_AI_Systems_Make_Decisions) | ✅ |
| 10 | Deployment: Taking AI Systems to Production | [Deployment_Taking_AI_Systems_to_Production](https://github.com/JoshithReddyAleti/Episode_10_Deployment_Taking_AI_Systems_to_Production) | ✅ |
| 11 | Observability: Knowing What Your AI is Doing | [Observability_Knowing_What_Your_AI_is_Doing](https://github.com/JoshithReddyAleti/Episode_11_Observability_Knowing_What_Your_AI_is_Doing) | ✅ |
| **12** | **Inference & Model Serving: The Systems Under the Model** ← you are here | [Inference_and_Model_Serving_The_Systems_Under_the_Model](https://github.com/JoshithReddyAleti/Episode_12_Inference_and_Model_Serving_The_Systems_Under_the_Model) | ✅ |
