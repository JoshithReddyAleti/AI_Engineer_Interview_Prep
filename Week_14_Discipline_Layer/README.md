# 🎯 Week 14 — The Discipline Layer: Data Engineering + Prompt Engineering + LLM Security + A/B Testing

> **Maps to:** [Episode_14_The_Discipline_Layer_What_Separates_Shippers_from_Demoers](https://github.com/JoshithReddyAleti/Episode_14_The_Discipline_Layer_What_Separates_Shippers_from_Demoers)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the layer where careers plateau — or accelerate.**

Every AI engineer eventually confronts these four disciplines. Most treat them as afterthoughts. The teams that treat them as first-class engineering are the ones that ship durable systems and get promoted.

**The four crafts of Week 14:**
1. **Data Engineering for AI** — vector DB internals deep enough to CHOOSE and TUNE (not just call an SDK)
2. **Prompt Engineering as Engineering** — DSPy compilation, TextGrad optimization, structured decoding, compression
3. **LLM Security** — OWASP LLM Top 10, injection defense stack, red teaming, exfiltration prevention
4. **A/B Testing on Stochastic Systems** — power calculations, CUPED, sequential testing, causal inference

**Signal calibration (what interviewers detect):**
- **Junior:** "We use Pinecone / it works fine."
- **Mid:** "I know HNSW is used for vector search."
- **Senior:** "We tuned HNSW with M=32, efConstruction=400 to hit 0.95 recall at 8ms p95."
- **Staff+ / Frontier Lab:** "For our 100M vector corpus at 0.95 recall@10 SLO, HNSW-only would need 200GB RAM; we combined HNSW with PQ 8-bit codebook, dropped to 32GB, 2ms recall degradation. The trade-off was justified because our budget was $6K/mo and disk-ANN was too slow for interactive."

Week 14 gets you to the staff+ / frontier-lab bar across all four disciplines.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 32 | HNSW math from first principles, DSPy compilation, 9-layer injection defense, CUPED variance reduction, MinHash/LSH, contamination detection, OWASP LLM Top 10, PAIR/TAP attacks, structured decoding internals, LLMLingua compression |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 17 | Implement MinHash LSH deduplication, DSPy Signature + Module, prompt injection classifier, sample size calculator, CUPED estimator, Wilson CI, sequential testing with alpha spending, HNSW parameter tuner |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | 100M-vector search system, enterprise prompt management platform, defense-in-depth for LLM security, A/B testing platform for stochastic AI, red team automation |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 17 | Vector DB recall dropped silently, prompt injection made news, A/B test showed "no effect" that was real, contamination invalidated 6 months of eval, DSPy migration failed |
| [05_Terminology_Differentiation.md](05_Terminology_Differentiation.md) | Reference | HNSW vs IVF vs Flat, exact vs approximate search, MinHash vs SimHash vs semantic dedup, DSPy vs LangChain vs manual prompts, injection types, Bonferroni vs FDR, novelty vs primacy effects |

## The 31 Sections Covered (Full Episode 14)

### Part A — Data Engineering For AI (7 sections)

| # | Section | Interview Territory |
|---|---|---|
| 1 | Foundations | Data as product, 80% of AI work, org patterns for AI data teams |
| 2 | Vector DB Internals | HNSW (Malkov & Yashunin 2016), IVF, Product Quantization (Jégou 2011), quantization tiers, disk-ANN, hybrid search, sharding |
| 3 | Data Versioning | DVC, lakeFS, Delta Lake, Iceberg — versioning as CI/CD prerequisite for AI |
| 4 | Deduplication | Exact hash, MinHash + LSH, SimHash, semantic dedup — Jaccard thresholds that matter |
| 5 | Contamination Detection | Test-in-train, benchmark leakage, exact/n-gram/embedding-similarity checks |
| 6 | Synthetic Data | Self-Instruct, Evol-Instruct, distillation, model collapse mitigation |
| 7 | Data Pipelines | Reindexing at scale, streaming updates, lineage, quality gates |

### Part B — Prompt Engineering as Engineering (8 sections)

| # | Section | Interview Territory |
|---|---|---|
| 8 | Foundations | Prompts as versioned artifacts, Golden set, prompt registry |
| 9 | DSPy | Khattab 2023 — Signatures, Modules, Optimizers (BootstrapFewShot, MIPROv2, COPRO), compilation |
| 10 | TextGrad | Yuksekgonul 2024 — text as gradient descent, when it beats manual optimization |
| 11 | Structured Decoding | Outlines FSM (Willard & Louf 2023), XGrammar (2024), LMFormatEnforcer, LLGuidance, Instructor |
| 12 | Compression | LLMLingua (Jiang 2023) — 20× compression at 5% quality loss |
| 13 | Meta-Prompting | LLMs generating prompts, prompt-of-prompt patterns |
| 14 | Advanced Patterns | Tree of Thoughts (Yao 2023), Graph of Thoughts, Program of Thoughts, self-consistency (Wang 2022), Chain of Verification (Dhuliawala 2023) |
| 15 | Management Platforms | LangSmith, LangFuse, Promptfoo, Helicone, PromptLayer — when each |

### Part C — LLM Security (8 sections)

| # | Section | Interview Territory |
|---|---|---|
| 16 | Foundations | OWASP LLM Top 10 2024, threat modeling for LLM apps |
| 17 | Injection | Direct vs indirect injection, taxonomy, real attack vectors |
| 18 | Defenses | The 9-layer defense stack, dual-LLM pattern (Willison 2023), spotlighting (Hines 2024) |
| 19 | Exfiltration | Training data extraction, prompt-based exfil, markdown/image side-channels |
| 20 | Jailbreaking | Many-shot (Anthropic 2024), PAIR (Chao 2023), TAP (Mehrotra 2023), automated red teaming |
| 21 | Supply Chain | Model weights, tokenizer swaps, malicious datasets, dependency risks |
| 22 | Red Teaming | Garak, PyRIT, methodology, blue-team feedback loops |
| 23 | Security Ops | Incident response, forensics on stochastic systems, disclosure |

### Part D — A/B Testing on Stochastic Systems (8 sections)

| # | Section | Interview Territory |
|---|---|---|
| 24 | Foundations | Why LLM A/B testing is different (stochastic, subjective, heavy-tailed) |
| 25 | Statistical Foundations | Power formula from first principles, MDE, Bonferroni, FDR/Benjamini-Hochberg, CUPED (Deng 2013) |
| 26 | Experiment Design | Four LLM change types (prompt, model, RAG, agent), randomization unit, allocation |
| 27 | Running Experiments | SRM checks, novelty effects, sequential testing, alpha spending |
| 28 | Analyzing | Segment analysis, heterogeneous treatment effects, guardrail metrics |
| 29 | Rollout Strategies | Canary → gradual → full, rollback triggers, feature flags |
| 30 | Causal Inference for AI | When observational data matters, instrumental variables, DiD |
| 31 | A/B Testing at Scale | Platform design, interleaving, holdout populations |

## The Discipline Layer Cost Reality

- **Vector DB miscalibration:** 30-50% higher hosting cost or 10-20% recall drop. At scale, $50K-$500K/year.
- **Contamination undetected:** 6 months of eval invalidated. Retrain and re-evaluate. $100K+ waste.
- **Prompt injection incident:** SEC disclosure, customer lawsuits, brand damage. $M+ per incident (see Chevrolet chatbot, Air Canada rulings).
- **Underpowered A/B tests:** ship wrong changes with confidence. Compounds silently over years.
- **No versioning:** unreproducible bugs, no rollback, prompt regressions blamed on model.

Every discipline in Week 14 has direct dollar consequences. The teams that treat these as afterthoughts pay for it in incidents, cost overruns, and shipped regressions. The teams that treat them as first-class engineering ship durable systems.

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1-13 | LLM Fundamentals → Fine-Tuning + Multimodal | (see main README) | ✅ |
| **14** | **The Discipline Layer: Data + Prompt + Security + A/B Testing** ← you are here | [Episode_14](https://github.com/JoshithReddyAleti/Episode_14_The_Discipline_Layer_What_Separates_Shippers_from_Demoers) | ✅ |

**Week 14 completes the AI Engineering Roadmap 2026.** From here, the series bridges to the **Agentic AI Roadmap 2027**.
