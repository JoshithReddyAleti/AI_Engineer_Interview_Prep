# 🏗️ Week 13 — System Design Questions

> **Focus:** Enterprise fine-tuning platform, multi-tenant LoRA serving at scale, document intelligence pipeline, voice agent system, video understanding at scale
>
> **Depth:** MAANG staff+ and frontier-lab bar. Every design includes cost math, failure modes, and specific technology choices with justification.

---

## Q1. Design an Enterprise Fine-Tuning Platform Serving 50+ Tenants ⭐⭐⭐⭐

**Prompt:** "Design the complete platform for a SaaS product where 50+ enterprise customers can fine-tune their own LoRA adapters on top of a shared base model, then serve them at production quality."

### Architecture:

```
┌────────────── TENANT PORTAL ──────────────────────────────────┐
│  Web UI · API · CLI                                            │
│  - Upload training data                                        │
│  - Configure job (rank, epochs, target modules)                │
│  - Monitor progress                                            │
│  - View eval results                                           │
│  - Deploy adapter                                              │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── DATA VALIDATION LAYER ──────────────────────────┐
│  - Format check (chat template, ChatML/Llama-3/etc.)         │
│  - PII detection (Presidio)                                    │
│  - Contamination check (against public benchmarks)            │
│  - Duplicate detection (MinHash)                              │
│  - Quality filters (length, language, format)                 │
│  - Distribution analysis (histograms of task types)           │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── TRAINING ORCHESTRATOR ──────────────────────────┐
│  - Kubernetes with GPU node pools                             │
│  - Job scheduling (priority by tier)                          │
│  - Cost tracking per job                                      │
│  - Fault tolerance (checkpoint every N steps)                 │
│  - Multi-tenant isolation (namespace + resource quotas)       │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── TRAINING RUNTIME ───────────────────────────────┐
│  - Base image: Axolotl + Unsloth for LoRA/QLoRA               │
│  - Or DeepSpeed/FSDP for full FT                              │
│  - Base model: Llama-3.1-70B FP16 (shared, cached)           │
│  - LoRA adapters: saved to S3 per tenant                      │
│  - Metrics: loss curves, eval scores → W&B                    │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── EVALUATION LAYER ───────────────────────────────┐
│  - Golden set: tenant-provided + shared safety set            │
│  - Metrics: task-specific + MMLU regression + IFEval          │
│  - LLM-as-judge (GPT-4 as evaluator, different family)        │
│  - Quality gate: block deploy if regression > threshold       │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── INFERENCE SERVING ──────────────────────────────┐
│  - vLLM with multi-LoRA support                               │
│  - Shared base model across all tenants                       │
│  - LoRAs loaded per-request                                    │
│  - Punica kernels for efficient batched multi-LoRA            │
│  - LRU eviction of unused LoRAs                               │
└────────────────────────────────────────────────────────────────┘
```

### Key Decisions:

**1. LoRA over full FT.** Cost of full FT on 70B ≈ $10K per job. LoRA ≈ $200. At 50 tenants running monthly, savings are $500K/year. Only offer full FT for premium tier.

**2. Shared base model at serving.** One 70B loaded, 50+ LoRAs applied per request. vLLM multi-LoRA. Alternative: 50 dedicated deployments = $200K/month vs $10K/month.

**3. Data validation before training.** Common failures: wrong chat template (15% of tenant submissions), duplicates (bad training signal), PII (compliance risk), contamination (invalidates eval).

**4. Quality gates.** Every LoRA must pass: (a) task quality on tenant golden set, (b) MMLU regression < 2 points, (c) safety eval on shared harmful-prompt set. Block deploy on any failure.

**5. Cost tracking per job.** GPU-hours × rate = tenant charge. Billing integrated.

### Multi-Tenant Isolation:

- **Data:** each tenant's training data in isolated S3 prefix, encrypted with per-tenant KMS keys.
- **Compute:** K8s namespace per tenant, resource quotas, no cross-tenant node sharing on regulated tier.
- **Adapters:** LoRA files in isolated storage. Access controlled by IAM.
- **Inference:** every request tagged with `tenant_id`, LoRA looked up from tenant-scoped registry.

### Cost Math:

**Training side (per tenant per month):**
- Data validation: $10 (compute)
- LoRA fine-tune (r=16, 10K examples): $200
- Evaluation (golden set + regression tests): $50
- Total: ~$260 per tenant per training run

**Serving side (shared, all tenants):**
- 4×H100 for base model, always warm: $6K/month
- Amortized across 50 tenants active concurrently: $120/tenant/month
- Multi-LoRA overhead: negligible

**Revenue:** charge $500-$2K/tenant/month depending on tier. Healthy margin.

### Failure Modes:

- **Adapter serving pool exhausted:** LRU evict cold LoRAs, alert if churning
- **Base model corrupted / needs update:** version pinning per tenant, upgrade paths
- **Tenant fine-tunes their way to bad model:** quality gates catch this; also provide "revert to previous adapter" one-click
- **Cross-tenant cache poisoning:** semantic cache must be namespaced by tenant_id (mandatory audit)

### Interview signal:

"The economic argument for LoRA is that you can offer fine-tuning as a service where the marginal cost per tenant is $260/month training + $120/month serving = $380 all-in. That's what lets you sustainably charge $500-2K/month with margin."

---

## Q2. Design a Multi-Tenant LoRA Serving System at Scale ⭐⭐⭐⭐

**Prompt:** "Design serving for 500 LoRA adapters on top of a shared 70B base model. 10K QPS aggregate. Sub-500ms p95 TTFT. Explain how you keep unit economics."

### Serving Architecture:

```
Request (with tenant_id + adapter_id)
    │
    ▼
Gateway (auth, rate limit, route to region)
    │
    ▼
Multi-LoRA Inference Cluster
├── Base Model: Llama-3.1-70B FP16 on 4×H100 (shared, sharded via TP=4)
├── LoRA Registry: 500 adapters, ~200MB each (~100GB total)
│   - Hot pool: 50 most-used adapters kept in GPU memory
│   - Warm pool: 200 adapters in CPU memory, load-on-request
│   - Cold: rest on S3, load-on-first-request (~1s cold load)
├── Punica Kernels: batched multi-LoRA compute (Chen 2023)
└── Continuous Batching with LoRA-aware scheduling
```

### The Punica Trick (Chen et al. 2023 'Punica: Multi-Tenant LoRA Serving'):

Standard multi-LoRA: each request applies its LoRA sequentially. Not batched. Bad throughput.

Punica: **batched grouped-matrix-multiplication** across different LoRAs simultaneously.

```
For a batch with requests using different LoRAs [A_1, B_1, A_2, B_2, ...]:
Standard: sequential loop, slow
Punica: BGMV kernel groups them into one CUDA kernel call
```

Result: 12× throughput vs naive multi-LoRA. Enables serving 100+ LoRAs efficiently on one GPU.

### LoRA Loading Tiers:

- **Hot (GPU HBM):** 50 adapters, ~10GB. Zero-latency swap.
- **Warm (CPU DRAM):** 200 adapters, ~40GB. ~50ms swap-in.
- **Cold (S3):** 250 adapters. ~1s cold-load, then promoted to warm.

Eviction policy: LRU. Promote to warmer tier on repeated use.

### Cost Math:

**Serving infrastructure (per month):**
- 4×H100 SXM: $22K/month on-demand, $9K/month reserved (1yr)
- Redundancy: 2 replicas: $18K/month reserved
- Load balancer + monitoring: $1K/month
- **Total: ~$19K/month for the base infrastructure**

**Per-adapter cost:**
- 500 adapters serving 10K QPS aggregate = 20 QPS per adapter average
- Per-adapter serving cost: $19K / 500 = $38/adapter/month
- With semantic cache (40% hit rate): $23/adapter/month effective

Adapter serving is the killer feature economically. **10-30× cheaper than dedicated deployments** while maintaining quality.

### Latency Budget:

- Auth + routing: 20ms
- LoRA lookup + load (hot tier): 5ms
- LoRA load (warm tier): 50ms  
- LoRA load (cold tier): 1000ms — flag as cold start
- Base model prefill (6K prompt with APC): 100ms
- LLM decode 300 tokens: 300ms
- **Total: ~450ms p95 TTFT** (hot LoRA); ~500ms with warm loads

### Failure Modes:

**Cold-load surge (traffic spike on cold LoRAs):**
- Detection: cold-load queue depth
- Response: aggressive promotion to hot pool, page on-call if sustained
- Prevention: predictive pre-warming based on tenant activity patterns

**Base model crash:**
- Auto-restart with warm pool preservation
- Fallback: route to secondary cluster (in different region)
- Adapters must be re-loaded on restart — track load times

**One tenant's LoRA is buggy (produces bad outputs):**
- Isolate their impact via LLM-as-judge sampling per tenant
- Alert tenant, don't affect others
- Fast disable via feature flag

### Frontier lab context:

Together AI, Fireworks, Anyscale all offer multi-LoRA serving as core products in 2026. The technology stack is standardized: vLLM + Punica kernels + LoRA registry. Differentiation is on economics and operational excellence.

---

## Q3. Design a Document Intelligence Pipeline (Enterprise PDF/Contract Processing) ⭐⭐⭐⭐

**Prompt:** "Design a system that processes 100K legal contracts/day. Extracts structured data (parties, dates, terms), answers questions about clauses, flags risks. Must handle scanned PDFs, tables, signatures. Latency: <30s per document."

### Architecture:

```
Document Ingestion
    │
    ▼
Document Type Router
├── Text-based PDF → text extraction pipeline
├── Scanned PDF → OCR pipeline
└── Complex layout → vision-based pipeline (ColPali)
    │
    ▼
Structural Analysis
├── Page detection
├── Reading order (layout parser)
├── Table detection (Camelot, PubTables)
├── Figure/signature detection
    │
    ▼
Extraction Pipeline
├── Named entities (parties, dates, monetary amounts)
├── Clause segmentation (LLM-based)
├── Key-value pair extraction (VLM-based for forms)
    │
    ▼
Knowledge Graph Population
├── Contract → entities → relationships
├── Store in graph DB (Neo4j)
    │
    ▼
Retrieval + QA
├── Chunked storage + embeddings (dense + BM25 hybrid)
├── ColPali for layout-heavy documents
├── LLM answers with citations
├── Risk flagging (fine-tuned classifier)
```

### Key Technology Choices:

**1. Vision-first for layout-heavy documents.**
Traditional pipeline: PDF → OCR → text → NLP. Loses layout, tables become garbled.
Modern pipeline: PDF → page images → VLM (Qwen2-VL, Claude 3.5) → structured extraction. Better on legal docs with complex formatting.

**2. Domain-specific models.**
- **LayoutLM/Donut** for form-like documents (invoices, applications)
- **ColPali** for retrieval on layout-heavy documents
- **Fine-tuned Llama** for clause classification and risk detection
- **GPT-4 or Claude** for complex reasoning and question answering

**3. Hybrid retrieval.**
- BM25 for exact clause matching (legal language is precise)
- Dense embeddings for semantic search
- ColPali for questions where layout matters
- Rerank with cross-encoder for final ordering

### Performance Requirements:

**Latency budget per document (target 30s):**
- OCR (if needed): 5s per page × 10 pages = 50s. TOO SLOW for scanned.
  - Optimization: parallel page processing → 5s aggregate
- Layout analysis: 2s
- LLM extraction: 15s (multiple LLM calls for different clauses)
- Storage + indexing: 3s
- **Total: 25s achievable with parallelism**

**Throughput: 100K/day = 1.16 QPS sustained. But bursty:**
- Peak: 5-10 QPS
- Off-peak: 0.1 QPS
- Solution: queue-based processing with autoscaling

### Cost Math:

**Per document:**
- OCR (Tesseract or Textract): $0.01-$0.05
- VLM analysis: $0.10 (Claude 3.5 for accurate layout understanding)
- LLM clause extraction (multiple calls): $0.20
- Storage: $0.001
- **Total: ~$0.30 per document**

**At 100K/day:** $30K/day = $900K/month. Big number. Optimization mandatory.

**Cost optimization:**
- Semantic cache for duplicate documents: 20% hit → $180K/month savings
- Cheaper LLM for simple clauses (Llama-70B self-hosted): $50K/month savings
- ColPali eliminates OCR cost for many docs: $20K/month savings
- **Optimized: ~$500K/month** for 100K docs/day, still significant

### Enterprise Requirements:

**Compliance:**
- HIPAA if healthcare contracts: BAA, PHI encryption, audit
- SOC 2 audit trail
- Data residency (contracts may be regulated by region)
- Right-to-delete cascade

**Accuracy SLAs:**
- Entity extraction: >95% precision, >90% recall on validation set
- Question answering: >85% factual accuracy on customer-graded holdout
- Risk flagging: >95% recall on known risk categories (false positives OK, false negatives costly)

**Failure modes:**
- OCR fails on handwriting: route to VLM
- LLM hallucinates on obscure clause: require citation, flag if no citation
- Bad extraction on new document type: continuous eval, weekly review

### Interview signal:

"For enterprise document intelligence, ColPali is a game-changer — you replace the fragile OCR + BM25 + rerank stack with direct vision embeddings that handle layout, tables, and figures natively. On our benchmark, this improved retrieval nDCG@5 from 0.62 to 0.81 on layout-heavy legal documents."

---

## Q4. Design a Voice Agent System With <1s Response Latency ⭐⭐⭐⭐

**Prompt:** "Design a customer service voice agent that handles phone calls. Sub-1s response time, natural interruptibility, handoff to human on complex queries. Scale: 10K concurrent calls."

### Latency Budget (Total <1000ms to first sound):

```
Mic input → 
  Voice Activity Detection: 100ms
  Streaming ASR (partial): 200ms
  End-of-turn detection: 100ms  
  LLM first token: 400ms (dominant)
  Streaming TTS first audio: 200ms
Total: ~1000ms
```

**Optimizations to hit budget:**

1. **Streaming everything.** Don't wait for full utterance to send to LLM. Feed partial ASR results to LLM, cancel if user still talking.

2. **LLM with speculative decoding.** 1.5-2× TTFT reduction.

3. **Streaming TTS (ElevenLabs Flash, XTTS).** First audio available in 200ms.

4. **VAD tuned aggressively.** Detect end-of-turn within 300ms of silence (natural conversation pause is 500-800ms).

5. **Skip STT/TTS for voice-native models.** GPT-4o Realtime API processes audio→audio directly, cutting 400ms.

### Architecture:

```
Phone call (SIP/Twilio) → Voice Gateway → Agent Runtime
                                              │
                                              ├── VAD (WebRTC VAD)
                                              ├── Streaming ASR (Deepgram or Distil-Whisper)
                                              ├── Dialog State Manager
                                              ├── LLM (Claude Haiku or GPT-4o-mini)
                                              │    ├── RAG for knowledge queries
                                              │    ├── Tools (order lookup, refund, etc.)
                                              │    └── Guardrails
                                              ├── Streaming TTS (ElevenLabs Flash)
                                              └── Handoff Logic → human agent queue
```

### Concurrency Model:

10K concurrent calls with <1s latency requires:
- **Async processing:** each call is a lightweight coroutine, not thread
- **Connection pooling** to LLM, TTS, STT services  
- **Backpressure:** if downstream services degrade, queue with timeout
- **Horizontal scaling:** stateless agent runtime, scale to thousands of instances

**GPU capacity for streaming ASR:**
- Distil-Whisper: ~50 concurrent streams per H100
- 10K concurrent = 200 H100s dedicated to STT alone
- Alternative: cloud STT API (Deepgram) at $0.005/min

**LLM capacity:**
- Claude Haiku or GPT-4o-mini via API: ~$0.001 per interaction
- 10K concurrent × 60min/hour × avg 20 exchanges/hour = 12M interactions/day = $12K/day LLM cost
- Self-hosted Llama-8B alternative: cheaper at scale

### Interruptibility:

**The critical UX feature:** if TTS is playing and user starts talking, immediately stop TTS and process new input.

Implementation:
- VAD monitors user audio during TTS playback
- On speech detected, send cancellation signal to TTS
- Buffer partial LLM response (may still be useful context)
- Process new user input, generate response acknowledging context

**Frontier level:** GPT-4o Realtime API handles this natively — the model itself decides to yield when it detects speech.

### Handoff to Human:

Triggers:
- Explicit user request ("human agent")
- Multiple consecutive failed intents
- Sentiment detection (frustrated user)
- Complex query outside agent's tools
- Escalation on sensitive topics (billing dispute over $X)

Implementation:
- Save conversation state to Redis
- Route to human queue (Zendesk, Salesforce)
- Human agent gets full context transcript
- Return-to-bot handoff also supported

### Cost Math:

**Per call (average 5min):**
- STT: 5min × $0.005 = $0.025
- LLM (20 exchanges × $0.001): $0.02
- TTS (5min of audio): $0.03
- Infrastructure amortized: $0.02
- **Total: ~$0.10 per call**

At 10K concurrent calls, 24/7:
- Daily volume: 10K × 24 × 12 = 2.88M calls/day (theoretical max)
- Realistic: 500K calls/day
- Cost: 500K × $0.10 = $50K/day = $1.5M/month

**vs human agents:** each human handles ~50 calls/hour, cost $15-25/hour. Human cost for equivalent: $15K/hour × 24 = $360K/day = $10M/month.

**7× cost reduction with voice agent** at maintained quality (for well-scoped tasks).

### Interview signal:

"The 1s latency target is achievable but tight. Every 100ms matters. Voice-native models like GPT-4o Realtime eliminate the STT/TTS round-trip and get you to 500ms. That's a step-change UX improvement, not incremental."

---

## Q5-Q16: Additional System Design Questions (Condensed)

### Q5. Design a Video Understanding Platform for Content Moderation ⭐⭐⭐⭐
Scale: 100M videos/day. Frame sampling: adaptive (scene-change based). VLM analysis per keyframe. Aggregate across frames. Classify (safe/borderline/violating). Human review for borderline. Cost: $0.01 per video via optimized frame sampling. Storage: keyframe hashes for dedup.

### Q6. Design Fine-Tuning Infrastructure With Cost Governance ⭐⭐⭐
GPU cluster shared across tenants. Priority queue (paid > free). Per-tenant budget enforcement. Real-time cost tracking. Anomaly detection (unusually long jobs = investigate). Fair scheduling (no tenant starves). Preemption for higher-priority jobs (checkpoint + resume).

### Q7. Design a Continued Pretraining Pipeline ⭐⭐⭐⭐
Corpus: 100B tokens domain text. Deduplication (MinHash, exact). Quality filtering. Contamination check against benchmarks. Data mixing (task + domain data). Training: FSDP + ZeRO-3, hundreds of H100s, weeks. Continuous eval on domain and general benchmarks. Cost: $2M+ for 7B model.

### Q8. Design Synthetic Data Generation at Scale ⭐⭐⭐
Seed instructions → LLM generates variations (Evol-Instruct pattern). Filter for quality (LLM-judge). Deduplicate. Diversity injection (multiple generator models). Model collapse prevention: mix with real data (min 20%). Track lineage for auditing.

### Q9. Design RLHF Infrastructure at Frontier-Lab Scale ⭐⭐⭐⭐
Multi-week training runs. Distributed PPO across 100+ H100s. Reward model updated in loop. Human labeling pipeline ($10M+ budget). Safety training at every checkpoint. Red team continuously testing. Continuous eval on 1000+ benchmarks. Rollback if regression.

### Q10. Design Preference Data Collection Pipeline ⭐⭐⭐
Prompt generation, response generation, human labeling interface, quality control (redundant labeling), inter-annotator agreement tracking, active learning (label uncertain samples). Cost: $0.50-$5 per preference pair depending on complexity.

### Q11. Design Multi-Modal Search System ⭐⭐⭐
Text query and image query support. CLIP embeddings for both. Cross-modal retrieval. Optional fine-tuning on domain (e.g., fashion, e-commerce). Faceted search combining vector search with metadata filters. Sub-100ms retrieval on 100M items.

### Q12. Design an Image Generation Product Backend ⭐⭐⭐
Diffusion model serving (Stable Diffusion / Flux). GPU pool for inference. Queue-based (image gen takes seconds). Prompt safety filter (before generation). Output safety filter (before returning). Storage (S3 with lifecycle policies). Cost: $0.01-$0.05 per image self-hosted.

### Q13. Design Model Merging Service ⭐⭐⭐
Tenants specify: base model + list of fine-tuned models to merge + merge strategy (TIES, DARE, task arithmetic). Compute merged model. Evaluate on relevant benchmarks. Return merged model (as LoRA delta if possible). Cost: minutes of compute per merge.

### Q14. Design Cross-Region Fine-Tuning Deployment ⭐⭐⭐
Data residency: EU tenants train on EU GPUs. US tenants on US GPUs. LoRA artifacts stay in region. Cross-region observability aggregates only non-PII metrics. Regional gateways route by tenant.

### Q15. Design an Evaluation-as-a-Service Product ⭐⭐⭐
Customers upload their models + eval sets. Run: task-specific metrics, LLM-as-judge (multiple family judges to avoid bias), regression on public benchmarks, safety eval, latency/cost benchmarks. Compare vs baseline. Generate report.

### Q16. Design a Whisper Streaming Transcription Service ⭐⭐⭐⭐
Real-time transcription for meetings, calls, live streams. Support 10K concurrent streams. Distil-Whisper on GPU. Chunk-and-emit with overlap. Speaker diarization (pyannote). Punctuation + capitalization inference. Multi-language auto-detect. Latency: <300ms per chunk. Cost: $0.005/min competitive with Deepgram.
