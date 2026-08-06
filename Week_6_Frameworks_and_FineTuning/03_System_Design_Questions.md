# 🏗️ Week 6 — System Design Questions

> **Focus:** Framework selection at scale, multi-agent architectures, fine-tuning infrastructure, model serving pipelines, cost optimization
>
> **How to use:** 30-45 min whiteboard rounds. These are the questions where "I've thought about this in production" separates you from candidates who've only read tutorials.

---

## Q1. Design a Multi-Framework Enterprise AI Platform ⭐⭐⭐⭐

**Prompt:** "Your company has 5 AI use cases: (1) customer support chatbot, (2) internal document Q&A, (3) code review assistant, (4) sales lead qualification, (5) contract analysis. Different teams built each with different frameworks. Consolidate them into a unified platform."

**Architecture approach:**

```
┌─────────────────── Unified AI Gateway (FastAPI) ────────────────┐
│                                                                  │
│  Authentication · Rate Limiting · Cost Attribution · Routing     │
│                                                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
    ┌────────────────────────┼──────────────────────────┐
    ▼                        ▼                          ▼
┌─────────────┐    ┌──────────────────┐    ┌────────────────────┐
│ Simple API  │    │ LangChain/LCEL   │    │ LangGraph          │
│ (raw SDK)   │    │ Workflows        │    │ Complex Agents     │
│             │    │                  │    │                    │
│ Support     │    │ Lead             │    │ Contract Analysis  │
│ chatbot     │    │ Qualification    │    │ Code Review        │
│             │    │                  │    │                    │
│ (fast, low  │    │ (chained         │    │ (multi-step +      │
│  overhead)  │    │  prompts)        │    │  human-in-loop)    │
└─────────────┘    └──────────────────┘    └────────────────────┘
                             │
                             ▼
                   ┌─────────────────────┐
                   │ LlamaIndex          │
                   │ RAG Layer           │
                   │                     │
                   │ (Document Q&A       │
                   │  shared across      │
                   │  use cases)         │
                   └─────────────────────┘
                             │
                             ▼
    ┌────────────────────────────────────────────────────────────┐
    │           Shared Infrastructure Layer                        │
    │                                                              │
    │  Model Router → OpenAI / Anthropic / Self-hosted            │
    │  Vector DB (Pinecone/Weaviate)                              │
    │  Observability (Langfuse — framework-agnostic)              │
    │  Prompt Registry (versioned, tested)                        │
    │  Guardrails (input + output)                                │
    │  Cost tracking + attribution                                │
    │                                                              │
    └────────────────────────────────────────────────────────────┘
```

**Key architectural decisions:**

**1. Don't mandate one framework.**
Different use cases have different optimal frameworks. Forcing everything into LangChain (or LangGraph, or nothing) is a mistake. Let teams choose — but standardize the interfaces.

**2. Standardize the OUTPUT layer, not the framework.**
Every AI feature exposes the same API contract: input schema, output schema, trace format, cost format. Internally they can use different frameworks.

**3. Shared observability (framework-agnostic).**
Langfuse works with everything — LangChain, LlamaIndex, raw SDK. This means one dashboard for all AI features, one alerting system, one cost dashboard.

**4. Shared model router.**
All frameworks call through a central router that: enforces rate limits, tracks cost per team, handles failover between providers, applies guardrails.

**5. Shared vector database.**
Multiple RAG use cases share the same vector infrastructure. Metadata filtering separates tenants and use cases.

**6. Prompt registry.**
All prompts stored centrally, versioned, tested. Any framework references prompts by ID: `registry.get("customer_support_v3")`. This enables A/B testing across teams.

**The critical insight:** The framework is an implementation detail. The infrastructure is the platform. Consolidate infrastructure, not frameworks.

---

## Q2. Design a Fine-Tuning Pipeline for Continuous Model Improvement ⭐⭐⭐⭐

**Prompt:** "You have a customer support LLM. Users provide feedback (thumbs up/down). Every month, retrain the model on new preference data. Design the pipeline."

**Pipeline design:**

```
┌────────── Data Collection (Continuous) ──────────┐
│                                                   │
│  Production traffic → conversations logged         │
│  User feedback captured (thumbs up/down)          │
│  Human review of low-quality responses            │
│                                                   │
│  Storage: PostgreSQL + S3                         │
│  Format: (query, response, feedback, metadata)    │
└──────────────────────┬────────────────────────────┘
                       │
                       ▼ (Monthly cron)
┌────────── Data Preparation Pipeline ──────────────┐
│                                                    │
│  1. Filter: last 30 days, verified feedback       │
│  2. Dedup: hash-based similarity check            │
│  3. Balance: sample per category/difficulty       │
│  4. Format: preference pairs (chosen, rejected)   │
│  5. Split: 90% train, 10% eval                    │
│                                                    │
│  Output: HF Dataset in S3                         │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────── Training (DPO on LoRA) ──────────────────┐
│                                                     │
│  Base: last month's fine-tuned model               │
│  Add: this month's preference data                 │
│  Config: LoRA rank 16 + DPO beta 0.1               │
│  Hardware: 2x A100 (~4 hour training)              │
│  Framework: TRL DPOTrainer                         │
│                                                     │
│  Output: New model checkpoint + LoRA adapters      │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌────────── Evaluation Gate ─────────────────────────┐
│                                                     │
│  Golden test set (500 curated queries)             │
│  Metrics:                                          │
│    - Response quality (LLM-as-judge)               │
│    - Safety (guardrail violation rate)             │
│    - Domain accuracy (task-specific eval)          │
│                                                     │
│  Compare: new model vs current production          │
│                                                     │
│  Gate: new model must be ≥ current on all metrics  │
│        AND better on ≥ 1 metric to promote         │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────┴────────┐
        Pass  │                 │ Fail
              ▼                 ▼
   ┌──────────────────┐   ┌──────────────────┐
   │ Canary Deploy    │   │ Alert team +     │
   │                  │   │ Investigate      │
   │ 5% → 25% → 100%  │   │ regression       │
   │ over 5 days      │   │                  │
   │                  │   │ Don't promote    │
   │ Auto-rollback if │   │                  │
   │ metrics degrade  │   │                  │
   └──────────────────┘   └──────────────────┘
```

**Key design decisions:**

**1. Never do full FT — always LoRA on top of the previous month's LoRA.**
LoRA adapters compose. Merging monthly LoRAs incrementally is much cheaper than re-training from scratch.

**2. DPO not RLHF.**
DPO is simpler, more stable, and produces comparable results with 30-50% less compute.

**3. Gated deployment.**
Never promote a model without passing an eval gate. Regressions kill user trust faster than any bug.

**4. Canary + auto-rollback.**
Even after passing eval, canary in production for 5 days. Auto-rollback triggers on: user feedback drop, latency spike, cost spike.

**5. Preference data hygiene.**
Not all thumbs-down = good training signal. Filter: verified feedback only, human-reviewed low-quality cases, dedup, and balance across categories.

**6. Store everything.**
Every version of every model, every dataset, every eval result. When something breaks, you need to reproduce.

---

## Q3. Design an Observability System for a Multi-Framework AI Platform ⭐⭐⭐⭐

**Prompt:** "You have 15 AI features across 5 teams using LangChain, LangGraph, raw SDK, and CrewAI. How do you get unified observability?"

**Approach: OpenTelemetry as the unifying standard.**

```
┌───── LangChain apps ───┐    ┌───── LangGraph apps ─────┐
│ auto-instrumentation   │    │ auto-instrumentation      │
│ via langchain-otel     │    │ via langgraph tracer      │
└─────────┬──────────────┘    └──────────┬────────────────┘
          │                              │
┌─────── Raw SDK apps ──────────┐   ┌── CrewAI apps ─────┐
│ manual OpenTelemetry SDK      │   │ OpenLLMetry adapter │
│ @traced decorator             │   │                     │
└─────────┬─────────────────────┘   └───────┬─────────────┘
          │                                 │
          └────────────────┬────────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  OTel Collector          │
              │  (OpenTelemetry spec)    │
              └────────────┬─────────────┘
                           │
        ┌──────────────────┼────────────────────────┐
        ▼                  ▼                        ▼
  ┌──────────┐      ┌──────────────┐        ┌─────────────┐
  │ Datadog  │      │ Langfuse     │        │ Prometheus  │
  │ (APM)    │      │ (LLM-specific│        │ (metrics)   │
  └──────────┘      │  UX)         │        └─────────────┘
                    └──────────────┘
```

**What each span captures (unified format):**

```json
{
  "span_id": "abc123",
  "trace_id": "xyz789",
  "operation": "llm_call",
  "framework": "langchain",  // or langgraph, raw_sdk, crewai
  "model": "gpt-4o",
  "input_tokens": 1250,
  "output_tokens": 340,
  "cost_usd": 0.0128,
  "latency_ms": 1420,
  "temperature": 0.7,
  "prompt_id": "customer_support_v3",  // Central prompt registry ref
  "user_id": "u_456",
  "session_id": "s_789",
  "feature": "support_chatbot",
  "team": "support_platform",
  "status": "success"
}
```

**What this enables:**

- **Cross-framework tracing:** Follow a request through LangChain agent → LlamaIndex RAG → raw SDK call → response.
- **Unified cost attribution:** Who spent what? Which feature? Which team?
- **Alert on any framework:** Latency threshold applies to all AI calls regardless of framework.
- **Compare frameworks empirically:** Same task, different frameworks — which has lower latency? Higher cost? Better quality?

**The critical design decision:** Choose observability BEFORE choosing frameworks. Vendor-locked observability (LangSmith) forces framework choices. Vendor-neutral (Langfuse/OTel) preserves flexibility.

---

## Q4. Design a Cost Optimization System for AI Features at Scale ⭐⭐⭐⭐

**Prompt:** "Your AI features cost $200K/month and growing. The CFO wants a 50% reduction without hurting quality. Design the system."

**Multi-layered cost optimization:**

**Layer 1 — Model routing (biggest lever, 40-60% savings):**

```
Router decision tree:
├── Query complexity: simple factual → gpt-4o-mini or claude-haiku
├── Query complexity: reasoning-heavy → gpt-4o or claude-sonnet
├── Query complexity: extreme reasoning → o1 or o1-mini
├── Volume/cost sensitive → self-hosted fine-tuned model
└── Fallback: cheaper model if primary times out
```

Rules of thumb:
- 60-80% of queries in most systems can use "small" tier
- Reserve "large" tier for genuinely complex queries
- Route via lightweight classifier or query length + keyword heuristics

**Layer 2 — Semantic caching (20-30% savings):**

```
Query arrives → 
  Check semantic cache (similarity threshold 0.95) → 
    Hit: return cached response (~0 cost)
    Miss: proceed to LLM
```

- 20-30% cache hit rate typical for support chatbots
- Cache invalidation on document updates
- Per-user or global depending on data sensitivity

**Layer 3 — Prompt optimization (10-20% savings):**

- Audit system prompts for verbosity
- Compress few-shot examples (keep essence, cut ceremony)
- Move static instructions to fine-tuned models
- Use structured outputs to reduce output tokens

**Layer 4 — Context compression (15-25% savings):**

For conversations:
- Summarize after 10 turns (keep summary + last 5 turns)
- Compress old messages into structured facts

For RAG:
- Reduce top-K from 10 to 5 (with reranker to maintain quality)
- Compress retrieved chunks (remove boilerplate)

**Layer 5 — Batching and async (10-20% savings):**

- Batch requests via provider batch APIs (50% discount typical)
- Async parallel processing for independent tasks
- Non-time-sensitive queries → batch queue (overnight processing)

**Layer 6 — Self-hosted fine-tuned models for high-volume workflows:**

Break-even math:
- 10M requests/month × $0.005/request = $50K/month API cost
- Self-hosted 8B fine-tuned on 2× A100 = $10K/month
- Savings: $40K/month, break-even in ~2 months

**Combined impact:**
- Model routing: -40%
- Caching: -20% (of what remains)
- Prompt optimization: -10%
- Context compression: -15%
- Batching: -5%
- Self-hosting for high-volume: -20% (for those workflows only)

Total: 50-60% reduction achievable without quality loss.

**Cost dashboard requirements:**
- Cost per feature (which features are expensive?)
- Cost per user (are power users profitable?)
- Cost per model tier (is routing working?)
- Cache hit rate over time
- Alerts on spend anomalies

---

## Q5. Design a Model Serving Architecture for 100K QPS ⭐⭐⭐⭐

**Prompt:** "Design serving infrastructure for a fine-tuned 13B model serving 100K queries per second."

**Architecture:**

```
┌───────────── Load Balancer (nginx/HAProxy) ──────────────┐
│         Health checks · SSL termination · rate limiting   │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ▼
┌───────────────── Request Router ─────────────────────────┐
│                                                            │
│  Priority queueing:                                        │
│  - Real-time queries → main pool                           │
│  - Batch queries → batch pool                              │
│  - Long-context queries → long-context pool                │
│                                                            │
└───────┬────────────────┬──────────────────┬────────────────┘
        │                │                  │
        ▼                ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Main Pool    │  │ Batch Pool   │  │ Long-context │
│              │  │              │  │              │
│ vLLM servers │  │ vLLM servers │  │ vLLM servers │
│              │  │ (larger      │  │ (fewer, more │
│ AWQ-quantized│  │  batches,    │  │  memory)     │
│ 13B model    │  │  higher      │  │              │
│              │  │  throughput) │  │              │
│ 50 replicas  │  │ 10 replicas  │  │ 5 replicas   │
│ 4x A100 each │  │ 8x A100 each │  │ 8x A100 each │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │                  │
        └────────────────┼──────────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │ Redis (KV cache)   │
              │ Response cache     │
              │ Rate limit state   │
              └────────────────────┘
```

**Key decisions:**

**1. Quantization: AWQ over FP16.**
- 4x memory reduction — more concurrent requests per GPU
- <2% quality loss on most tasks
- Faster inference (specialized kernels)

**2. Serving engine: vLLM.**
- PagedAttention: dramatically higher throughput than naive serving
- Continuous batching: dynamic batch size based on requests
- OpenAI-compatible API: easy client integration

**3. Multiple pools for different workloads:**
- Real-time: optimized for low latency (smaller batches)
- Batch: optimized for throughput (larger batches, less latency-sensitive)
- Long-context: separate pool because these are memory-heavy

**4. Capacity math:**
- vLLM with AWQ on 4x A100 = ~500 tokens/sec throughput
- Average query = 200 output tokens = ~2.5 QPS per server
- 100K QPS / 2.5 = 40K servers ← unrealistic

**Reality check:** 100K QPS is enormous. Actual approach:
- **Streaming responses** — user sees tokens immediately
- **Speculative decoding** — 2-3x throughput improvement
- **Response caching** — 20-30% hit rate typical
- **Semantic caching** — additional 10-15%
- **Effective QPS after caching:** 60-70K new queries → ~24K servers still needed

At this scale, cost is enormous ($M/day). Sanity check: does 100K QPS make sense? Often the answer is "reduce demand via caching, batching, and eliminating unnecessary queries" not "scale infrastructure."

**Interview signal:** Pushing back on unrealistic requirements shows maturity. "100K QPS is likely wrong — let's audit where that number came from" is a valid response.

---

## Q6-Q15: Additional System Design Questions (Condensed)

### Q6. Design an A/B Testing System for LLM Applications ⭐⭐⭐⭐
Sticky user assignment. Variant configurations (prompt, model, RAG params). Metrics: quality (LLM-judged), cost, latency, user feedback. Statistical significance testing. Auto-rollback on regression. Support: multi-armed bandits for adaptive traffic allocation.

### Q7. Design a Prompt Registry with Versioning and Testing ⭐⭐⭐
Central prompt storage. Every prompt has: unique ID, version, template, test cases, owner. On prompt change: run test suite, require approval, canary deploy, roll back on regression. All apps reference prompts by ID.

### Q8. Design a Multi-Agent Coordination System ⭐⭐⭐⭐
Message passing between agents (Kafka/Redis). Shared state (per-conversation Redis hash). Timeout and retry logic. Circular dependency detection. Cost budgets per crew. Debugging: full inter-agent conversation logs.

### Q9. Design Fine-Tuning Infrastructure for Multiple Teams ⭐⭐⭐⭐
Shared GPU pool with fair scheduling (Kubernetes + Volcano). Data versioning (DVC). Experiment tracking (W&B). Model registry (MLflow). Checkpoint storage (S3). Approval workflows before promotion.

### Q10. Design a Guardrails System for a Multi-Tenant AI Platform ⭐⭐⭐
Per-tenant guardrail configurations. Shared safety layer (toxicity, PII) + tenant-specific rules. Fast path for cached decisions. Async escalation to human review. Audit log per guardrail action.

### Q11. Design an Evaluation Pipeline for CI/CD ⭐⭐⭐
On every PR: golden test set runs (500 cases). Compare against production baseline. Block merge if any metric regresses >5%. Nightly full eval on 5000-case set. Weekly A/B test on real traffic. Human review of edge cases quarterly.

### Q12. Design a Quantization + Serving Pipeline for a Company's Model Zoo ⭐⭐⭐⭐
Automated pipeline: FT model → quantize (AWQ) → benchmark → validate quality → deploy to vLLM. Support multiple model families. Version everything. Rollback capability. Per-model performance dashboards.

### Q13. Design a RAG-Powered Enterprise Search Across All Company Data ⭐⭐⭐⭐
Data sources: SharePoint, Confluence, Slack, GitHub, Google Drive, email. Access control per user (only see permitted docs). Hybrid search + reranker. Attribution to sources. Feedback loop for quality improvement.

### Q14. Design a Cost Attribution System for Shared AI Infrastructure ⭐⭐⭐
Every AI call tagged: team, feature, environment, user. Aggregate: cost per team per day. Chargeback to team budgets. Alerts when a team exceeds allocated budget. Dashboard: cost trend per feature.

### Q15. Design a Self-Improving Fine-Tuning Loop ⭐⭐⭐⭐
Production traffic → sample low-quality responses → human review → create training data → monthly DPO retrain → evaluate → canary deploy → measure improvement. Fully automated except human review step.
