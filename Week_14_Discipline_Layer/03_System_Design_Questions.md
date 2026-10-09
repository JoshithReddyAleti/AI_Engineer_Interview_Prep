# 🏗️ Week 14 — System Design Questions

> **Focus:** 100M-vector search system, enterprise prompt management platform, defense-in-depth for LLM security, A/B testing platform for stochastic AI, red team automation
>
> **Depth:** MAANG staff+ and frontier-lab bar. Every design includes math, cost, and specific tech choices with justification.

---

## Q1. Design a 100M-Vector Search System with 0.95 Recall@10 and Sub-10ms p99 Latency ⭐⭐⭐⭐

**Prompt:** "Design vector search for 100M documents, 1024-dim embeddings. SLO: recall@10 ≥ 0.95, p99 latency < 10ms. Budget: $10K/month. Multi-tenant with per-tenant filtering."

### Architecture:

```
Client
  ↓
Load Balancer (per region)
  ↓
API Gateway (auth, rate limit, tenant routing)
  ↓
Query Router → decides partition strategy
  ↓
Vector Search Cluster (4 nodes, sharded)
├── Node 1: HNSW-PQ8 index for shard 1 (25M vectors, ~10GB RAM)
├── Node 2: shard 2
├── Node 3: shard 3
└── Node 4: shard 4
  ↓
Result Aggregator (top-K across shards)
  ↓
Optional Rerank (cross-encoder on top-100 → top-10)
  ↓
Response
```

### Key Design Decisions:

**1. HNSW + Product Quantization (PQ8).**

Raw vectors: 100M × 1024 × 4 bytes = 400 GB. Cannot fit on single node.

With PQ 8-bit codebook (8 sub-vectors × 8-bit codes = 8 bytes per vector):
- Vector storage: 100M × 8 bytes = 800 MB total
- HNSW graph overhead (M=32): ~19 GB
- **Total: ~20 GB across cluster, ~5 GB per shard**

Fits comfortably in 16 GB RAM per node with headroom for query workload.

**2. Sharding strategy.**

4 shards, ~25M vectors each. Random shard assignment (not by tenant) to balance load.

Query: broadcast to all shards, retrieve top-K per shard, aggregate to top-K globally.

**3. HNSW parameters.**

- M = 32 (higher recall, moderate memory)
- efConstruction = 400 (higher build quality)
- efSearch = 200 at query time (tunable for recall/latency)

At these settings on quantized vectors: recall@10 ≈ 0.94. To hit 0.95, add rerank on top-100 with full-precision vectors kept on disk. Disk fetch adds ~2ms, brings recall to 0.96.

**4. Multi-tenant filtering.**

For low-selectivity filters (< 5% of docs matched): post-filter after HNSW.

For high-selectivity (per-tenant with many small tenants): partition indexes per tenant_id. Each tenant has own HNSW index. Sub-linear routing.

Multi-tenancy model: tenants < 100K vectors → per-tenant index. Larger tenants → shared shards with post-filter.

**5. Failure handling.**

- Shard failure: return partial results with degradation flag. Alternative: pause writes, restore from snapshot.
- Query timeout: return best-effort results at deadline. Log deadline_exceeded metric.
- Update lag: writes go to primary, reads from primary + replica. Eventually consistent OK for search.

### Cost Math:

Cluster: 4 × r6i.xlarge (16 GB RAM, 4 vCPU) = 4 × $180/mo = $720/mo
Plus load balancer + gateway: $200/mo
Plus storage (S3 for full-precision vectors): $100/mo
Plus monitoring: $200/mo
**Total infrastructure: ~$1.2K/mo**

Well under $10K budget. Headroom for scale.

### Latency Budget (p99 target 10ms):

- Client to API gateway: 1ms
- Auth + routing: 1ms
- Fan-out to 4 shards (parallel): 5ms per shard
- Result aggregation: 1ms
- Response: 1ms
- **Total: ~9ms p99** — hits SLO

If rerank added: 2ms extra (disk fetch + cross-encoder). Push to 12ms; may need to relax SLO or optimize rerank.

### Failure Modes:

- **Distribution shift over time:** graph quality degrades. Solution: monthly rebuild schedule.
- **Hot tenants overwhelming shared shards:** rate-limit per tenant.
- **Rare-query long-tail latency:** ef_search tuning per-query based on estimated difficulty.
- **Vendor lock-in:** portable format (write your own file format or use faiss).

### Frontier Lab Context:

At billion-vector scale, custom solutions dominate. Facebook uses FAISS variants, Google uses ScaNN. This design uses off-the-shelf HNSW-PQ from Qdrant or Milvus at 100M scale.

---

## Q2. Design an Enterprise Prompt Management Platform ⭐⭐⭐⭐

**Prompt:** "Design a platform for managing 500+ prompts across 20+ teams. Requirements: versioning, testing, deployment, cost tracking, quality regression detection, rollback."

### Architecture:

```
Prompt Registry (Postgres + object storage)
  ├── Prompts (versioned, semver)
  ├── Metadata (owner, description, tags)
  ├── Test datasets (golden sets)
  └── Deployment state (which version live where)
    ↓
CI/CD Pipeline
  ├── Prompt change PR → automated tests
  ├── Golden set evaluation (LLM-judge + task metrics)
  ├── Regression check (vs current production)
  ├── Cost estimation (token count × pricing)
  ├── Human approval gate for significant changes
  └── Deployment to staging → canary → production
    ↓
Runtime Serving
  ├── Prompt lookup by (name, version)
  ├── A/B test routing (per feature flag)
  ├── Request-level tracing
  └── Cost + latency + quality logging
    ↓
Observability
  ├── Per-prompt cost dashboard
  ├── Quality trends over time
  ├── Alert on regression
  └── Rollback trigger
```

### Key Design Decisions:

**1. Prompts as first-class artifacts.**

Not stored in code. Not scattered across repos. In a central registry with:
- Semantic versioning (v1.2.3)
- Provenance (author, PR, timestamp)
- Immutability (published versions can't be edited)
- Structured metadata

**2. Automated evaluation pipeline.**

Every prompt PR triggers:
- Syntax check (template variables valid)
- Cost estimate (tokens × pricing)
- Golden set evaluation
- Regression check (compare to production version)
- Safety check (adversarial test cases)

**3. Deployment progression.**

Staging (100% test traffic) → Canary (5% production) → Gradual (25% → 50% → 100%) → Full.

At each stage: guardrail metrics monitored. Automatic rollback if breached.

**4. Cost governance.**

Per-team budgets. Real-time attribution. Anomaly detection (unexpected token spike).

**5. Multi-tenancy.**

Teams isolated by namespace. RBAC (who can edit vs deploy vs view).

### Scale Considerations:

500 prompts × 20 teams × 10 versions each = 100K prompt versions historically.
- Cold storage: object store
- Hot storage (in-use): Postgres or Redis
- Search: full-text over prompts + metadata

Query patterns:
- Runtime lookup by (name, version) — must be fast (<10ms)
- Dashboards (aggregate cost by team) — batch OK
- CI tests — moderate volume, batch

### Existing Solutions vs Build:

- **LangSmith:** great if LangChain-heavy
- **LangFuse:** open source, model-agnostic
- **Promptfoo:** CI testing focused
- **PromptLayer, Helicone:** proxy-based

For 20+ teams, custom platform often justified for full control. Build on top of existing observability (OpenTelemetry, Datadog).

---

## Q3. Design Defense-in-Depth for a Public LLM Chatbot ⭐⭐⭐⭐

**Prompt:** "Design security architecture for a customer-facing LLM chatbot with RAG. Handles account queries, tool calls (order lookup, refund initiation). Must resist prompt injection, exfiltration, jailbreaking."

### The 9-Layer Defense Stack Applied:

```
User Input
  ↓
[Layer 1] Input filter: regex for known-bad patterns
[Layer 2] Input filter: ML classifier (Llama Guard or ProtectAI)
  ↓
RAG Retrieval (from tenant-scoped index)
  ↓
[Layer 3] Spotlighting: encode retrieved docs with delimiters
[Layer 4] Dual-LLM architecture:
  ├── Privileged LLM: orchestrates, calls tools
  └── Quarantined LLM: processes retrieved content, outputs structured data
  ↓
[Layer 5] Structured output enforcement (JSON schema)
  ↓
Tool Call (with validation)
[Layer 8] Egress control: URL whitelist, no arbitrary requests
[Layer 9] Human approval for sensitive actions (refunds > $X)
  ↓
[Layer 6] Output filter: PII scan, canary detection
[Layer 7] Canary check: system prompt not leaked
  ↓
Response to User
```

### Additional Enterprise Controls:

**Threat modeling:**
- STRIDE analysis for each component
- Data flow diagrams identifying trust boundaries
- Regular red team exercises

**Detection & response:**
- Log every prompt + response (PII redacted)
- ML-based anomaly detection on query patterns
- Rate limiting per user + IP
- Alert on canary token appearance in outputs

**Continuous testing:**
- Garak automated scans nightly
- Manual red team quarterly
- Bug bounty program

**Incident response:**
- Kill switch to disable LLM feature
- Model rollback capability
- Communication template for user disclosure
- Post-incident: attack added to red team suite

### Cost:

- ML classifiers (Llama Guard): $500-2K/mo self-hosted
- Multi-LLM architecture: 2× LLM cost vs single
- Security tooling (SIEM, Garak, PyRIT): $2-5K/mo
- Red team engagement: $10-30K/quarter
- **Total defense budget: ~$25K/quarter for enterprise-grade**

Compared to a single injection incident: $M+ (see Air Canada, Chevrolet). Cheap insurance.

---

## Q4. Design an A/B Testing Platform for LLM Systems ⭐⭐⭐⭐

**Prompt:** "Design an A/B testing platform for a company running 100+ LLM experiments per month. Support all four change types (prompt, model, RAG, agent). Statistical rigor required."

### Architecture:

```
Experiment Definition (YAML/UI)
  ├── Hypothesis
  ├── Metrics (primary, secondary, guardrails)
  ├── Power calculation → sample size
  ├── Randomization unit (user, session, request)
  ├── Duration
  └── Pre-registration (locked before start)
    ↓
Feature Flag System (LaunchDarkly, Statsig, Split, or build)
  ├── Deterministic hashing → arm assignment
  ├── Sticky per user
  └── Rollout controls
    ↓
Runtime Instrumentation
  ├── Emit experiment_id + arm_id per request
  ├── Emit primary + secondary + guardrail metrics
  └── OpenTelemetry integration
    ↓
Data Pipeline
  ├── Stream events to warehouse (Snowflake, BigQuery)
  ├── Nightly ETL: exposures, outcomes, joins
  └── Real-time aggregation for guardrails
    ↓
Analysis Engine
  ├── SRM check (chi-squared)
  ├── Primary metric CI (with CUPED)
  ├── Guardrail check
  ├── Segment analysis (with FDR)
  ├── Sequential testing (optional)
  └── Structured decision report
    ↓
Decision Layer
  ├── Auto-rollback on guardrail breach
  ├── Ship criteria (pre-registered)
  ├── Human review for edge cases
  └── Institutional learning capture
```

### Key Features:

**Statistical rigor built in:**
- Power calc UI: input MDE, get sample size + duration
- CUPED automatic when pre-experiment data available
- Multiple testing correction (FDR default)
- Sequential testing option for large effects

**LLM-specific:**
- Handle stochastic outputs (multi-sample averaging)
- LLM-judge integration for subjective metrics
- Cost per experiment tracking
- Model version tracking (which model was live per user per moment)

**Governance:**
- Pre-registration locks hypotheses before start
- Audit trail
- Peer review for high-impact experiments
- Institutional metrics library (agreed definitions)

### Scale:

100 experiments/month × 3-week average duration = ~75 concurrent experiments at any time. Requires:
- Instrumentation overhead < 1ms per request
- Data pipeline can handle 100M+ events/day
- Analysis can complete in minutes, not hours

### Existing Solutions:

- **Statsig:** LLM-friendly, growing rapidly
- **Optimizely, LaunchDarkly:** general-purpose, LLM support via feature flags
- **In-house at Meta, Netflix, Airbnb:** custom platforms

For 100+ experiments/month, sophisticated in-house or Statsig-level tooling justified.

---

## Q5-Q16: Additional System Design Questions (Condensed)

### Q5. Design a Dedup + Contamination Detection Pipeline ⭐⭐⭐⭐
Ingest 10TB training corpus. Exact hash dedup → MinHash LSH near-dup → semantic dedup. Contamination check against public benchmark corpus. Report duplicates + contamination. Support incremental updates as new data arrives.

### Q6. Design Prompt Injection Detection at Scale ⭐⭐⭐⭐
1B queries/day. Real-time classification (regex + ML). Feedback loop from human reviewers. Continuously updated attack patterns. Multi-tenant isolation. Cost < $0.001 per query.

### Q7. Design Automated Red Teaming Infrastructure ⭐⭐⭐
Continuous PAIR + TAP against production LLM endpoints. Track: attack success rate over time, novel attack discoveries, coverage. Feed findings to defense updates.

### Q8. Design DSPy Compilation Infrastructure ⭐⭐⭐
Users submit DSPy programs + training data. Cluster runs MIPROv2 compilation (LLM calls + evaluation). Cache compilation results. Return compiled prompts. Cost tracking per compilation.

### Q9. Design a Vector DB Migration ⭐⭐⭐
Move 100M vectors from Pinecone to self-hosted Qdrant. Zero-downtime. Progressive traffic shift. Recall parity validation. Rollback capability.

### Q10. Design Real-Time Contamination Detection for RAG ⭐⭐⭐
Every retrieved document checked against known benchmark corpus. Alert on hits. Filter or replace before serving.

### Q11. Design a Semantic Cache with Multi-Tenant Safety ⭐⭐⭐
Cache LLM responses by semantic similarity. Per-tenant namespacing to prevent cross-tenant data leakage. TTL policies. Hit rate monitoring.

### Q12. Design Structured Decoding Serving Infrastructure ⭐⭐⭐
Enforce JSON schema on 10K+ QPS. Grammar compilation cache. Latency < 2× unconstrained. Support Pydantic → schema → grammar pipeline.

### Q13. Design Guardrail Metrics Framework ⭐⭐⭐⭐
Define, monitor, alert on guardrails across features. Latency, cost, error, safety, fairness. Rollback triggers. Cross-experiment guardrails (some metrics must not regress across ANY experiment).

### Q14. Design Feature Flag System for LLM Experiments ⭐⭐⭐
Fine-grained targeting (user, session, geo). Deterministic hashing for sticky assignment. Rollout controls. Real-time flag evaluation < 1ms.

### Q15. Design Novelty Effect Detection ⭐⭐⭐
Track experiment metrics over time within each arm. Auto-detect: novelty (early spike then fade), primacy (early dip then recover). Suggest extended experiment duration. Weight late-period results higher.

### Q16. Design Data Versioning for AI Compliance ⭐⭐⭐⭐
Given a model in production, prove which data trained it. DVC or lakeFS integration. Immutable snapshots. Cascade delete for GDPR right-to-erasure. Audit trail for regulator queries.
