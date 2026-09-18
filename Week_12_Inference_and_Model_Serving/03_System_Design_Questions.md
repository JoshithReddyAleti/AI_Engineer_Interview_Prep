# 🏗️ Week 12 — System Design Questions

> **Focus:** Enterprise inference platform, multi-model gateway, disaggregated prefill/decode, multi-tenant with per-tenant SLOs, cost-optimized migration from API
>
> **How to use:** These are 45-60 minute whiteboard rounds at MAANG, frontier labs, and specialty inference companies. The bar: multi-layer reasoning across hardware, framework, and cost.

---

## Q1. Design a Self-Hosted Inference Platform Serving 100M Tokens/Day Across Multiple Models ⭐⭐⭐⭐

**Prompt:** "Design the complete inference infrastructure for a product doing 100M tokens/day. Multiple models (Llama-8B for classification, Llama-70B for chat, Mixtral for reasoning). Multiple tenants. Sub-500ms p95 TTFT SLO. Cost under $30K/month all-in."

**Architecture:**

```
┌────────────── CLIENT LAYER ───────────────────────────────────┐
│  Web app · Mobile · API partners                              │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── EDGE / API GATEWAY ─────────────────────────────┐
│  Cloudflare / AWS API Gateway                                 │
│  - Auth (JWT, API keys) · Rate limits · WAF                   │
│  - Request routing by model/tenant                            │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       ▼
┌────────────── LLM GATEWAY ────────────────────────────────────┐
│  Custom + LiteLLM base                                        │
│  - Model routing (cheap → expensive)                          │
│  - Semantic cache (Redis + vector search)                     │
│  - Cost tracking per tenant/request                           │
│  - Circuit breakers per inference cluster                     │
│  - Fallback to OpenAI on cluster failure                      │
│  - Observability: OTel traces, LangFuse                       │
└─────┬─────────────────┬─────────────────┬─────────────────────┘
      │                 │                 │
      ▼                 ▼                 ▼
┌───────────┐    ┌───────────┐    ┌───────────┐
│ Llama-8B  │    │ Llama-70B │    │ Mixtral   │
│ Cluster   │    │ Cluster   │    │ Cluster   │
│           │    │           │    │           │
│ vLLM      │    │ vLLM      │    │ vLLM      │
│ 4×L40S    │    │ 8×H100    │    │ 8×H100    │
│ TP=1      │    │ TP=4      │    │ EP+TP     │
│ 2 replicas│    │ 3 replicas│    │ 2 replicas│
│           │    │           │    │           │
│ INT8      │    │ INT4 AWQ  │    │ FP8       │
│ FP8 KV    │    │ FP8 KV    │    │ FP8 KV    │
└───────────┘    └───────────┘    └───────────┘

┌─── Observability Stack (Prometheus, Grafana, LangFuse) ─────┐
┌─── Cost dashboard, drift detection, quality monitoring ─────┐
```

**Key design decisions with justification:**

**1. Multi-model architecture (not one-size-fits-all).**

Different queries need different models. Small classification query → Llama-8B (cheap, fast). Standard chat → Llama-70B. Complex reasoning → Mixtral (MoE, capable). Router at gateway decides.

Cost impact: ~40% of traffic to Llama-8B at 1/10 the cost = 36% cost reduction vs Llama-70B for everything.

**2. Hardware selection per model:**

- Llama-8B: L40S is enough. 48GB HBM, moderate bandwidth. Lower $/hour.
- Llama-70B: 8×H100 SXM. NVLink for TP=4 sharding. HBM3 bandwidth.
- Mixtral: 8×H100 with expert parallelism. Larger memory footprint (MoE all experts must be in memory even if only 2 active).

**3. Quantization per model (calibrated to quality bar):**

- Llama-8B: INT8 weights (quality less sensitive at 8B scale).
- Llama-70B: INT4 AWQ (validated on golden set, < 3% quality drop).
- Mixtral: FP8 (native on H100, best quality preservation).
- FP8 KV cache universally (doubles batch size, minor quality impact).

**4. LLM Gateway is the critical piece:**

- Semantic cache (Redis with vector search): 30-45% hit rate on production traffic. HUGE cost saver.
- Circuit breakers per cluster: if Llama-70B cluster degrades, fall back to Mixtral (worse cost, but availability).
- OpenAI fallback: emergency safety net if all self-hosted clusters unhealthy.
- Cost tracking: every request tagged with tenant, model, feature.

**5. Autoscaling strategy:**

- Warm pools (min replicas ≥ 2 per model) to avoid cold starts.
- HPA scales on custom metric: queue depth per cluster.
- Peak vs off-peak: scale down at night, up during business hours.
- Predictive scaling: forecast next hour's demand from historical.

**Cost math:**

| Cluster | Config | Hourly | Monthly | Monthly Tokens |
|---|---|---|---|---|
| Llama-8B | 2 × 4×L40S | $9.6 | $6,900 | ~40M (40% of traffic) |
| Llama-70B | 3 × 8×H100 spot | $86 | $18,500 | ~50M (50% of traffic) |
| Mixtral | 2 × 8×H100 spot | $57 | $12,300 | ~10M (10% of traffic) |
| Total infra | | $152.6 | ~$37,700 | 100M |
| With 40% cache hit | | | **~$22,600** | 100M |

Under $30K target with semantic cache. Add observability tooling ($3K/mo), engineering time amortization → real all-in ~$28K.

**vs API (comparison):**

At $6/M tokens blended: 100M × $6 = $600/day = $18K/month. But sub-500ms p95 TTFT unreliable, data residency issues for some tenants, no fine-tuning path.

Self-host wins on: cost (barely, thanks to cache), latency (much better), control (huge).

**Observability requirements:**

Per-request trace: gateway → routing decision → model → cluster → GPU. Cost per request. Quality LLM-as-judge sample. Drift detection weekly.

**Failure modes to handle:**

- Cluster fails: circuit breaker → route to alternative cluster or OpenAI
- Model quality regression: golden set alerts, roll back model
- Cost anomaly: alerts, auto-throttle tenant
- Cache poisoning across tenants: strict namespacing

**Interview signal:** discussing "the semantic cache is what makes the cost math work" shows real production experience.

---

## Q2. Design a Multi-Model LLM Gateway for a SaaS Platform ⭐⭐⭐⭐

**Prompt:** "Design an LLM gateway that routes traffic across 6 models (OpenAI, Anthropic, self-hosted Llama, fine-tuned tenant models) with per-tenant policies, cost tracking, quality routing, and fallbacks."

**Core capabilities:**

**Routing engine:**
- Per-tenant model policies (some tenants restricted to certain models)
- Query classification for smart routing (cheap model for simple queries)
- Cost-based routing (cheapest capable model)
- Latency-based routing (fastest available)
- Quality-based routing (highest capability for critical requests)

**Fallback chains:**
```
Primary: Llama-70B (self-hosted)
  → Anthropic Claude (API fallback, higher cost)
    → OpenAI GPT-4o (emergency fallback)
      → Cached response (last resort)
```

**Semantic cache:**
- Per-tenant namespace (prevent cross-tenant leaks)
- Vector similarity threshold ~0.95
- 24h TTL for user content, 7d for stable content
- Hit rate dashboard

**Cost management:**
- Real-time cost per tenant
- Daily/monthly budgets with soft (80%) and hard (100%) limits
- Anomaly detection (2σ over baseline)
- Chargeback reports for enterprise tenants

**Observability:**
- Every request: trace_id, tenant_id, model_id, cost, latency (TTFT + E2E), tokens, quality signals
- Cluster health metrics per backend
- LangFuse for LLM traces

**Enterprise features:**
- Prompt injection detection
- PII redaction in logs
- Audit trail (immutable, tamper-evident)
- Regional routing (data residency)

**Existing solutions vs build:**
- Portkey, LiteLLM, Cloudflare AI Gateway: get you 70% there
- Build custom: full control, extensibility, deep integration
- Middle: start with LiteLLM open source + build custom layers

**Interview signal:** discussing "semantic cache namespacing per tenant is critical for security AND cost" shows enterprise depth.

---

## Q3. Design Disaggregated Prefill/Decode Architecture ⭐⭐⭐⭐

**Prompt:** "Design a large-scale inference system using disaggregated prefill/decode (DistServe/Splitwise pattern). Show the request flow, KV cache transfer, and failure handling."

**Architecture:**

```
Request
  │
  ▼
Router
  │
  ▼
Prefill Cluster (H100 SXM — compute-optimized)
  ├── Process prompt tokens in parallel
  ├── Generate KV cache
  └── Emit "prefill done, KV location = X"
      │
      ▼
KV Cache Transfer (NVLink within node, or RDMA across nodes)
      │
      ▼
Decode Cluster (H200 — memory-bandwidth-optimized)
  ├── Load KV cache
  ├── Continue autoregressive generation
  └── Stream output back
      │
      ▼
Response to user
```

**Why this pattern:**

Prefill: compute-bound. Loves H100 FLOPS.
Decode: memory-bandwidth-bound. Loves H200 bandwidth (4.8 TB/s vs H100's 3.35 TB/s).

Running them together means one is always underutilized. Disaggregation runs each at peak.

**KV cache transfer options:**

- Intra-node: NVLink (~900 GB/s, near-zero cost)
- Inter-node: RDMA over InfiniBand (~25 GB/s, ~10ms per GB)
- Fallback: TCP/IP (slow, avoid)

For Llama-70B request at 8K context: ~2.6 GB KV. RDMA transfer: ~100ms. Acceptable overhead if decode benefits outweigh.

**Optimization: locality-aware routing:**

Route requests to decode nodes in the same rack as their prefill node (minimize inter-node transfer).

**Failure handling:**

- Prefill node fails during processing: retry on another prefill node.
- Decode node fails mid-generation: re-prefill on new decode node (expensive, but rare).
- KV transfer fails: retry, then re-prefill.

**When it's worth the complexity:**

- 10M+ requests/day
- Cost pressure demanding maximum GPU utilization
- Long prompts + short outputs (prefill dominates cost)
- Willing to invest in infra engineering

**When it's NOT worth it:**

- < 1M requests/day (complexity outweighs benefit)
- Short prompts + long outputs (decode dominates)
- Team can't operationalize the complexity

**Interview signal:** discussing "we only disaggregated after our request mix showed prefill dominated at 60% of GPU time" shows measurement-driven decision.

---

## Q4. Design Enterprise Multi-Tenant Inference Platform With Per-Tenant SLOs ⭐⭐⭐⭐

**Prompt:** "Design a multi-tenant inference platform serving 200+ tenants across free/business/enterprise tiers. Different SLOs, different models available, cost isolation, compliance requirements."

**Tier definitions:**

| Tier | SLO | Models | Rate limit | Isolation |
|---|---|---|---|---|
| Free | p95 < 2s TTFT | Llama-8B only | 100 req/day | Shared cluster, tenant tagged |
| Business | p95 < 800ms | Llama-70B, GPT-4o-mini | 10K req/day | Shared cluster, priority queue |
| Enterprise | p95 < 500ms | All models + custom LoRAs | Unlimited | Dedicated cluster capacity |
| Regulated | Enterprise SLO + HIPAA | Enterprise + audit | Unlimited | Fully isolated cluster |

**Cluster architecture:**

- Shared inference clusters: serve free + business tenants. Tenant tags on every request.
- Enterprise-reserved capacity: pre-allocated slots ensure SLO under load.
- Regulated tenants: entirely separate infrastructure. Own K8s namespace, own GPUs, own observability.

**Priority queuing:**

At scheduler level: enterprise > business > free. Under load, free tier queued first.

**Cost attribution:**

- Every request: tenant_id, tokens, model, GPU-seconds
- Aggregate: real-time to Redis, batch to warehouse
- Chargeback: enterprise tenants get monthly cost reports
- Multi-tenant cost dashboards

**Compliance:**

- HIPAA tenants: PHI never in logs (redaction), audit trail immutable, BAA in place
- SOC 2: access controls, encryption at rest + transit
- GDPR: EU tenants on EU infrastructure, right-to-delete cascades to observability

**Isolation guarantees:**

- Cache namespaced per tenant (prevent cross-tenant leaks)
- Model access enforced at gateway (tenant policy check)
- Automated cross-tenant leak tests every deploy

**Failure modes:**

- One tenant abuses free tier: rate limit + cost cap + eventually kick off
- Enterprise cluster degrades: fallback to shared cluster with priority
- Regulated cluster fails: no fallback (compliance), escalate to human

**Interview signal:** "Regulated tenants get physically separate infrastructure — you can't tier compliance" shows enterprise SaaS reality.

---

## Q5. Design a Complete API-to-Self-Host Migration Plan ⭐⭐⭐⭐

**Prompt:** "The company has been on OpenAI/Anthropic APIs, spending $150K/month. Design the 6-month migration plan to self-hosting, preserving quality and reducing cost by 60%+."

**Phase 1 (Month 1) — Baseline & Assessment:**

- Instrument every API call with cost tracking + quality metrics
- Build golden dataset (500-1000 diverse queries with expected quality tier)
- Analyze traffic: peak QPS, avg tokens per request, prompt vs output ratio, cache-hittable prefix ratio
- Compute break-even math for candidate models
- Select target open-weight model (Llama-70B, Mixtral, Qwen, DeepSeek-V3)

**Phase 2 (Month 2) — Infrastructure prep:**

- Provision GPUs (spot + on-demand mix)
- Deploy vLLM with target model
- Baseline benchmarks (TTFT, TPOT, throughput)
- Deploy semantic cache + LLM gateway
- Set up observability stack

**Phase 3 (Month 3) — Shadow mode:**

- Route 100% of traffic to API (unchanged) + duplicate to self-hosted
- Compare responses on same inputs
- LLM-as-judge scores both
- Identify quality gaps
- Fine-tune prompts for self-hosted model if needed
- Measure real self-hosted cost per token vs API

**Phase 4 (Month 4) — Canary migration:**

- Start with lowest-risk traffic (internal tools, non-customer-facing)
- Gradually: 5% → 25% → 50% of that traffic segment
- Monitor: quality, latency, cost, error rate
- Full cutover of that segment when confidence high
- Repeat for next-lowest-risk segment

**Phase 5 (Month 5) — Full migration + optimization:**

- Migrate customer-facing traffic
- Deploy speculative decoding
- Enable APC
- Add quantization (calibrated on golden set)
- Iteratively improve cost-per-token

**Phase 6 (Month 6) — Steady-state operations:**

- Cost dashboard reviews weekly
- Quality regression monitoring
- Capacity planning based on trend
- Cost report to leadership monthly
- API remains as emergency fallback (< 5% traffic)

**Communication cadence:**

- Weekly team updates
- Bi-weekly to VP Eng
- Monthly to CFO (cost) and CPO (quality)

**Risk mitigation:**

- Kill switch: revert to 100% API in 15 minutes if catastrophic
- Quality regression: threshold that pauses migration
- Cost overrun: alert if projected savings won't materialize

**Success metrics (after 6 months):**

- Cost: $150K → $50K (67% reduction)
- Quality: maintained or improved on golden set
- Latency: p95 improved from 1200ms to 400ms
- Team capability: can now fine-tune, quantize, deploy models internally

**Interview signal:** "Shadow mode for a month before ANY user traffic migrates" shows risk-managed thinking.

---

## Q6-Q16: Additional System Design Questions (Condensed)

### Q6. Design GPU Autoscaling for Bursty AI Workload ⭐⭐⭐
Cold start: 30s-3min for 70B model load. HPA reactive scaling can't handle. Solution: warm pool (N always warm), predictive scaling from traffic curves, warm standby models (loaded but not serving), graceful degradation to smaller model or API if scale-out too slow.

### Q7. Design Cost-Optimized Inference for Long-Tail Traffic ⭐⭐⭐
Long tail = many small models with sparse usage. Options: (a) multi-LoRA on shared base (most efficient), (b) cold-loaded models (accept latency penalty), (c) route to API for infrequent models. Selection by usage frequency threshold.

### Q8. Design Model Version Rollout System ⭐⭐⭐⭐
New model version arrives. Test on golden set, canary rollout (5% → 25% → 50% → 100%), quality monitored at each stage, automated rollback if regression. Different from code deploys because model artifacts are huge (50-200 GB) and load times long.

### Q9. Design Regional Inference Deployment for Data Residency ⭐⭐⭐⭐
EU tenants → EU inference clusters. US tenants → US clusters. Same model, different physical infrastructure. Regional gateway routes based on tenant. Cross-region observability aggregation on non-PII metrics only. Compliance evidence per region.

### Q10. Design High-Availability Inference With Zero Downtime ⭐⭐⭐
Multiple replicas per model. Rolling model updates (drain, update, verify, next). Graceful shutdown draining in-flight requests. Circuit breakers between replicas. Load balancer removes unhealthy nodes fast.

### Q11. Design Batch Inference System for 100M Daily Documents ⭐⭐⭐
Different from serving. Optimize throughput not latency. Large batches, no streaming, aggressive quantization OK, spot GPUs, checkpointing for interrupt tolerance. Kafka input queue, Kubernetes Jobs, results to warehouse.

### Q12. Design Speculative Decoding Deployment ⭐⭐⭐⭐
Target 70B + draft 1B. Colocate on same GPU (small extra memory) or separate. Route requests conditionally (speculation on/off per request based on workload). Metrics: acceptance rate, speedup, quality preservation. A/B test to prove value.

### Q13. Design GPU Cost Anomaly Detection ⭐⭐⭐
Baseline: rolling 7-day cost per hour. Detect: current > mean + 2σ. Break down by tenant/model/feature. Common causes: bad user (attack), agent loop, prompt bloat. Automated response: rate limit, kill switch. Escalation: page on-call for severe.

### Q14. Design Multi-Cluster GPU Management ⭐⭐⭐⭐
Multiple K8s clusters across regions. Global scheduler places workloads by cost, latency, capacity. Cross-cluster failover. Terraform manages infrastructure. ArgoCD for GitOps deployments. Multi-cluster observability aggregation.

### Q15. Design Prompt Cache Invalidation Strategy ⭐⭐⭐
Semantic cache holds responses. When source data updates, invalidate related cache entries. Options: TTL only (simple but stale), event-driven invalidation (complex but fresh), hybrid (TTL + explicit invalidation on important updates). Track hit rate over time.

### Q16. Design Inference SLO Framework ⭐⭐⭐⭐
SLIs: availability (99.5%), TTFT p95 (< 500ms), TPOT p95 (< 40ms), quality score (> 0.85). Error budget calculation per month. Fast burn (5min at 10x) + slow burn (1hr at 2x) alerts. Monthly SLO reviews. Customer-facing SLO status page for enterprise tenants.
