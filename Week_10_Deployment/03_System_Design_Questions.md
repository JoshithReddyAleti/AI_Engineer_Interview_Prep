# 🏗️ Week 10 — System Design Questions

> **Focus:** Enterprise AI deployment platform, multi-tenant SaaS, global edge deployment, K8s vs serverless decisions, disaster recovery, HIPAA-compliant AI, cost governance, incident response systems
>
> **How to use:** 45-60 min whiteboard rounds. These are the questions that decide staff+ AI infrastructure roles.

---

## Q1. Design an End-to-End Production AI Platform Architecture ⭐⭐⭐⭐

**Prompt:** "Design the complete infrastructure for a production AI SaaS product. 10K enterprise customers, 100K daily active users, 10M+ LLM requests/day. Highly available, secure, cost-controlled."

**Architecture:**

```
┌─────────────────── CLIENTS ────────────────────────────────┐
│  Web App  │  Mobile App  │  Enterprise API │  Public API   │
└──────┬────────────┬────────────┬────────────┬──────────────┘
       └────────────┴────────────┴────────────┘
                          │
                          ▼
┌─────────────────── EDGE / CDN ─────────────────────────────┐
│  Cloudflare / CloudFront                                    │
│  - DDoS protection                                          │
│  - WAF (rate limiting, injection defense)                   │
│  - TLS termination                                          │
│  - Static assets                                            │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌─────────────────── API GATEWAY ────────────────────────────┐
│  Kong / AWS API Gateway / Envoy                             │
│  - Authentication (API keys, JWT)                           │
│  - Rate limiting (per key, cost-based)                      │
│  - Request routing                                          │
│  - Circuit breakers                                         │
│  - Observability (traces, metrics, logs)                    │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────── APPLICATION TIER (auto-scaled) ──────────────┐
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │ Chat Service   │  │ Agent Service   │  │ Batch Worker │ │
│  │ (FastAPI)      │  │ (LangGraph)     │  │ (Celery)     │ │
│  │                │  │                 │  │              │ │
│  │ 20-500 pods   │  │ 10-200 pods    │  │ 5-100 pods   │ │
│  └────────────────┘  └────────────────┘  └──────────────┘ │
│                                                              │
│  Deployment: K8s (EKS/GKE) with HPA                         │
│  Rolling deploys, blue-green for critical paths             │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌─────────────── LLM GATEWAY (critical layer) ───────────────┐
│  - Multi-provider routing (OpenAI, Anthropic, Google)       │
│  - Per-provider circuit breakers                            │
│  - Semantic cache (30-40% hit rate = 30-40% cost cut)      │
│  - Cost tracking per user/tenant                            │
│  - Prompt injection defense                                 │
│  - Retry with backoff + jitter                              │
└──────────────────────────┬─────────────────────────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
   ┌────────┐          ┌────────┐          ┌────────┐
   │OpenAI │          │Anthropic│          │Google │
   └────────┘          └────────┘          └────────┘

┌────────────── DATA TIER ──────────────────────────────────┐
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Postgres     │  │ Redis Cluster│  │ Vector DB     │    │
│  │ (RDS Multi-AZ)  │              │  │ (Pinecone/    │    │
│  │              │  │ - Session     │  │  self-hosted  │    │
│  │ Primary +    │  │ - Cache       │  │  Qdrant)      │    │
│  │ Read Replicas│  │ - Rate limits │  │               │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ S3 / GCS     │  │ Kafka        │  │ Data Warehouse│    │
│  │ (files)      │  │ (events)     │  │ (Snowflake/BQ)│    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└────────────────────────────────────────────────────────────┘

┌────────────── OBSERVABILITY ──────────────────────────────┐
│  Logs: CloudWatch / Loki                                    │
│  Metrics: Prometheus + Grafana                              │
│  Traces: OpenTelemetry + Jaeger/Datadog                     │
│  LLM Traces: LangSmith / Langfuse                           │
│  Errors: Sentry                                             │
│  Uptime: PagerDuty / OpsGenie                               │
│  Cost tracking: custom (per-tenant attribution)             │
└────────────────────────────────────────────────────────────┘

┌────────────── SECURITY ──────────────────────────────────┐
│  Secrets: Vault / AWS Secrets Manager                      │
│  Network: VPC, private subnets, security groups            │
│  Encryption: TLS 1.3, at-rest encryption everywhere        │
│  WAF: Cloudflare / AWS WAF                                 │
│  DDoS: Cloudflare / Shield                                 │
│  Compliance: SOC 2 Type II, ISO 27001                      │
└─────────────────────────────────────────────────────────────┘
```

**Key design decisions with justifications:**

**1. Edge/CDN first.**
Cloudflare or CloudFront terminates TLS, applies WAF rules, and serves static assets from edge. Protects origin from DDoS. Speeds up global users.

**2. API Gateway as security boundary.**
Every request authenticated + rate-limited before hitting application. Kong/Envoy handles the boring stuff so app code focuses on business logic.

**3. LLM Gateway is CRITICAL.**
Single choke point for all LLM traffic. Failures here = whole product down. Multi-provider fallback = survives single-provider outages. Semantic cache = 30-40% cost cut.

**4. K8s for application tier.**
At 10K enterprise customers, K8s justified. HPA (Horizontal Pod Autoscaler) scales based on CPU + custom metrics (queue depth, LLM latency). Rolling deploys for standard changes, blue-green for critical paths.

**5. Multi-AZ for HA.**
RDS Multi-AZ (automatic failover on primary failure). Redis cluster (sharded, resilient). App tier across 3+ AZs.

**6. Async for expensive workloads.**
Batch worker for long-running tasks (document processing, batch analysis). Users don't block, users get notification when complete.

**7. Observability from day 1.**
Three pillars (logs, metrics, traces) + LLM-specific (LangSmith). Not optional. Adding this later = years of pain.

**Cost breakdown estimate at scale:**

- Infrastructure (K8s, RDS, Redis, S3): ~$50K/month
- LLM provider costs (OpenAI + Anthropic): ~$200K-500K/month
- Observability (Datadog + LangSmith): ~$20K/month
- CDN/WAF (Cloudflare Enterprise): ~$5K/month
- **Total: ~$275K-575K/month at this scale**

Semantic caching pays for itself many times over: 40% cache hit on $500K LLM cost = $200K/month saved.

**Interview signal:** Discussing "LLM gateway is the single most important piece — it's what lets you survive provider outages" shows real production experience.

---

## Q2. Design a Multi-Tenant SaaS AI Architecture With Complete Isolation ⭐⭐⭐⭐

**Prompt:** "Design a multi-tenant AI SaaS platform where each tenant's data, model access, and cost are strictly isolated. Support 100+ tenants including regulated industries."

**Isolation strategies (choose based on tenant tier):**

**Tier 1 — Shared Infrastructure (Standard/Free):**
- All tenants on same K8s cluster
- Logical isolation (tenant_id in every query)
- Shared DB with tenant_id column
- Shared vector store with tenant namespaces
- Cost-effective; adequate for non-regulated tenants

**Tier 2 — Shared Compute, Isolated Data (Business):**
- Same K8s cluster
- Dedicated DB per tenant
- Dedicated vector store per tenant
- Better isolation; per-tenant backup independence
- 3-5x cost of Tier 1

**Tier 3 — Fully Isolated (Enterprise/Regulated):**
- Dedicated K8s namespace or cluster
- Dedicated DB, cache, vector store
- Dedicated LLM API keys (billed to tenant directly)
- Possibly single-tenant deployment in tenant's cloud
- 10-20x cost, required for HIPAA/PCI

**Architecture:**

```
┌────────────── TENANT CONTEXT LAYER ────────────────────────┐
│  Every request tagged with tenant_id (from JWT/API key)     │
│  Middleware enforces tenant context on every DB query       │
│  Impossible to bypass without explicit code path            │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼
┌────────────── ROUTING BY TENANT TIER ──────────────────────┐
│  Tier 1 → Shared cluster (cost-optimized)                   │
│  Tier 2 → Shared cluster, isolated data                     │
│  Tier 3 → Dedicated cluster/namespace                       │
└──────────────────────┬─────────────────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
   Shared         Semi-Isolated   Fully Isolated
   Cluster        Cluster          Deployment
                                   (per tenant)
```

**Data isolation enforcement:**

**Row-Level Security (PostgreSQL RLS):**
```sql
-- Every table has tenant_id
CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    user_id UUID NOT NULL,
    -- ...
);

-- RLS policy
CREATE POLICY tenant_isolation ON conversations
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

-- Application sets tenant per session
SET app.current_tenant = 'tenant_uuid_here';
```

**Vector store isolation:**
- Pinecone: separate namespace per tenant
- Qdrant/Weaviate: separate collection per tenant
- Self-hosted: dedicated index per tenant OR strict metadata filtering

**LLM cost attribution:**

Every LLM call tagged with tenant_id → per-tenant cost dashboards, per-tenant budgets, per-tenant reports.

**Cross-tenant safeguards:**

**Automated tests:**
```
Test: authenticate as Tenant A, attempt to query Tenant B's data
Expected: 403 Forbidden or empty results
Actual: must always match expected

Run these on every deploy. Any regression = block deploy.
```

**Anomaly detection:**
- User activity across multiple tenants = alert
- Cross-tenant data patterns in logs = alert
- Support ticket mentioning "seeing wrong data" = severity 1

**Compliance considerations:**

**HIPAA:** requires Tier 3 (fully isolated) OR business associate agreement + technical safeguards.

**PCI DSS:** cardholder data isolation + network segmentation.

**GDPR:** data residency (EU tenants in EU regions), right-to-be-forgotten (full delete pipeline per tenant).

**SOC 2 Type II:** annual audit of controls. All access logged.

**Interview signal:** Discussing "different tenant tiers get different isolation levels" + "compliance drives tier" shows enterprise SaaS understanding.

---

## Q3. Design Global Edge Deployment for a Latency-Sensitive AI Product ⭐⭐⭐⭐

**Prompt:** "Design deployment for an AI product that must feel fast globally. Users in US, Europe, Asia, Australia. LLM API calls are latency-heavy. How do you minimize perceived latency?"

**Latency budget:**

Users have expectations:
- Time to first token (TTFT): should feel instant (< 500ms)
- Full response: 5-15s for LLM is acceptable if streaming
- Non-LLM operations (auth, routing): < 100ms

**Sources of latency:**

- Client to edge: ~20-100ms depending on geography
- Edge to region: 100-300ms (cross-continent)
- Region to LLM provider: 50-200ms (usually same region)
- LLM inference: 1000-5000ms (dominant)
- Region to user: same as client to edge

**Architecture:**

```
User (anywhere)
    │
    ▼
Edge (Cloudflare Workers / Vercel Edge / Fastly)
    │  20-50ms
    │  - Auth verification (JWT)
    │  - Rate limiting
    │  - Cache lookup (semantic cache at edge)
    │  - Route to region
    ▼
Regional Deployment (US-East, US-West, EU, APAC)
    │  100-300ms if cross-region
    │  - LLM Gateway
    │  - Application logic
    │  - Regional DB replica
    ▼
LLM Provider (usually same region)
    │  50-200ms + 1-5s inference
    ▼
Response streams back through same path
```

**Design decisions:**

**1. Edge-side auth.**
JWT verification at edge (Cloudflare Workers can do this). No round-trip to origin for auth check. Saves 100-300ms.

**2. Edge-side rate limiting.**
Basic rate limits enforced at edge. Sophisticated rate limiting (cost-based) at origin.

**3. Edge caching for common queries.**
Semantic cache lookup at edge. Common questions get sub-100ms responses. Cloudflare AI Gateway does this.

**4. Regional deployment.**
App servers in multiple regions (US-East, EU-West, APAC). User routed to nearest via GeoDNS.

**5. Regional LLM providers.**
OpenAI/Anthropic/Google have regional endpoints. Use the region matching your app deployment.

**6. Global DB reads, regional writes.**
Primary DB in one region (writes). Read replicas globally (reads). Or Aurora Global Database. Handles most workloads.

**7. Data residency compliance.**
EU users' data stays in EU (GDPR). US users in US. Data replication respects boundaries.

**8. Streaming to hide LLM latency.**
User sees tokens as they're generated. Feels fast even though total time is 5-10s.

**Where NOT to run at edge:**

- Complex business logic (edge functions are compute-limited)
- Large model inference (edge doesn't have GPUs)
- Anything needing >100ms compute (defeats purpose)

Edge is for: routing, auth, caching, simple transformations.

**Cost implications:**

- CDN/Edge: low ($20-100/M requests)
- Multi-region app deployment: 2-3x single-region cost
- Cross-region DB replication: significant bandwidth cost
- Compliance benefit often justifies the cost

**Interview signal:** Discussing "edge for auth + cache, regional for compute + LLM" shows practical understanding of edge limitations.

---

## Q4. Design Disaster Recovery for a Production AI System ⭐⭐⭐⭐

**Prompt:** "Design the disaster recovery plan for a production AI SaaS. What RTO/RPO for each component? What's the plan when everything catches fire?"

**Component-by-component DR plan:**

**Application code (K8s deployments):**
- **RTO:** 1 hour
- **RPO:** N/A (stateless)
- **Backup:** Container images in registry, IaC in git
- **Recovery:** Redeploy from IaC to backup region

**PostgreSQL:**
- **RTO:** 30 minutes
- **RPO:** 5 minutes (point-in-time recovery)
- **Backup:** Multi-AZ replication + automated snapshots every 4h + hourly WAL archives to S3
- **Recovery:** Failover to standby (automatic) or restore from snapshot to different region

**Redis (cache + sessions):**
- **RTO:** 15 minutes
- **RPO:** 5 minutes for durable data, N/A for cache
- **Backup:** ElastiCache automated snapshots + cross-region replication for tier-1 data
- **Recovery:** Failover to replica, cache rebuilds naturally

**Vector Database (Pinecone/Qdrant):**
- **RTO:** 4 hours
- **RPO:** 24 hours
- **Backup:** Nightly export to S3
- **Recovery:** Reload from S3 export (embeddings can also be regenerated from source docs)

**Object Storage (S3):**
- **RTO:** immediate (S3 designed for 11 9's durability)
- **RPO:** N/A
- **Backup:** Cross-region replication for critical data
- **Recovery:** Automatic

**LLM Provider Access:**
- **RTO:** immediate (multi-provider fallback)
- **RPO:** N/A
- **Backup:** OpenAI + Anthropic + Google as alternatives
- **Recovery:** LLM Gateway routes to healthy provider

**User Data (conversations, files):**
- **RTO:** 1 hour
- **RPO:** 5 minutes
- **Backup:** Postgres backups + S3 replication
- **Recovery:** Restore Postgres to point-in-time

**Disaster scenarios:**

**Scenario 1: Region failure (AWS us-east-1 down)**
- Traffic redirects to us-west-2 (secondary region) via DNS/Route53 failover
- App deployments running in secondary (warm standby)
- Postgres promoted from standby
- Total recovery: ~30 min

**Scenario 2: LLM provider outage (OpenAI down)**
- LLM Gateway detects failures → circuit breaker OPEN
- Automatic failover to Anthropic
- Quality metrics tracked (may see slight quality difference)
- Users see no impact
- Recovery: automatic

**Scenario 3: Database corruption**
- Halt writes to prevent further damage
- Restore from most recent clean snapshot
- Replay WAL logs to just before corruption
- Total recovery: 1-4 hours depending on data size

**Scenario 4: Complete AWS outage (rare but happened)**
- Traffic to backup GCP deployment (if hot standby)
- Or accept downtime while restoring elsewhere (if cold DR)
- This is the "billion-to-one" scenario — plan but don't over-invest

**Testing:**

Quarterly DR drills. Actually failover to secondary region. Test that everything works. Most DR plans fail their first real test — practice reveals gaps.

**Cost of DR:**

- Warm standby (recommended): 30-50% overhead
- Hot standby: 100% overhead (2x infrastructure)
- Cold DR: 5-10% overhead but hours of RTO

Enterprise typically chooses warm standby.

**Interview signal:** Discussing "we do quarterly DR drills and always find bugs" shows engineering maturity.

---

## Q5. Design an Incident Response System for AI Production Issues ⭐⭐⭐⭐

**Prompt:** "Design the on-call and incident response system for AI production. Cover: detection, response, escalation, communication, postmortem."

**Detection layer:**

**Alerts trigger on:**
- Latency degradation (p95 > 2x baseline)
- Error rate spike (> 5% or > 3x baseline)
- LLM provider circuit breaker OPEN
- Cost spike (> 2x baseline hourly)
- Cross-tenant access attempt (security)
- Safety violation (agent doing unauthorized action)
- Data drift detected (input distribution shift)
- Quality regression (eval score drop > 10%)

**Alert routing:**

Severity 1 (page immediately): auth broken, mass hallucination, security incident, cost 10x
Severity 2 (Slack + page after 15 min): latency degraded, one provider down
Severity 3 (Slack only): minor metric shifts, non-urgent bugs
Severity 4 (email/ticket): quality issues, feature bugs

**On-call rotation:**

- Primary on-call: 1 week, then rotate
- Secondary on-call: backup if primary unreachable
- Escalation: engineering manager → VP eng → CTO
- Follow-the-sun for global teams: US → EU → APAC coverage 24/7

**Response playbook:**

```
Alert fires
    ↓
1. Acknowledge (< 5 min)
   - Prevent escalation
   - Communicate to team channel
    ↓
2. Assess (< 15 min)
   - Read runbook for this alert
   - Understand scope (users affected? severity?)
   - Decide: is this urgent enough to page others?
    ↓
3. Communicate (during response)
   - Status page update if user-facing
   - Slack updates every 15-30 min
   - Include: what's happening, what we're doing, ETA
    ↓
4. Remediate
   - Follow runbook first
   - Escalate if runbook doesn't work
   - Do the smallest safe fix (don't refactor during incident)
    ↓
5. Verify
   - Metrics returning to baseline?
   - User reports stopped?
   - Wait 30 min before declaring resolved
    ↓
6. Communicate resolution
   - Status page update
   - Slack "all clear"
   - Send to internal stakeholders
    ↓
7. Postmortem (within 48h)
   - Timeline, root cause, contributing factors
   - Action items with owners + due dates
   - Blameless — focus on process
```

**Runbook pattern (for each common alert):**

```markdown
# Runbook: LLM Provider Circuit Breaker OPEN

## Symptoms
- Alert: circuit_breaker_open{provider="openai"}
- User impact: requests routing to fallback (Anthropic)

## Diagnosis
1. Check OpenAI status: https://status.openai.com
2. Check our error logs for OpenAI: `grep "openai" logs/error.log`
3. Check recent deploys: was there a change?

## Common Causes
- OpenAI outage (external, wait)
- Auth issue (rotated key not deployed)
- Rate limit hit (increase or wait)
- Regional issue (try different region endpoint)

## Remediation
- If OpenAI outage: monitor, no action (fallback is working)
- If auth: deploy new key
- If rate limit: increase in provider dashboard OR reduce our usage
- If regional: change LLM_REGION env var

## Verification
- Circuit should close within 60s of resolution
- Verify with: /health/dependencies
```

**Postmortem template:**

```
Date:
Duration:
Severity:
Users impacted:

## Timeline (in UTC)
[Timestamp] [Event]
[Timestamp] [Detection]
[Timestamp] [Escalation]
[Timestamp] [Root cause identified]
[Timestamp] [Fix deployed]
[Timestamp] [Resolved]

## Root Cause
[What caused this?]

## Contributing Factors
[What made it worse?]

## What Went Well
[Detection speed, communication, etc.]

## What Went Poorly
[Detection gaps, delayed communication, etc.]

## Action Items
| Item | Owner | Due Date | Status |
|------|-------|----------|--------|

## Lessons Learned
[What would we do differently?]
```

**Enterprise concerns:**

- **Status page:** public (statuspage.io) — users see incidents
- **Customer comms:** proactive email for major incidents
- **Post-incident review meeting:** 1 hour with team, discuss findings
- **Metrics:** MTTR (mean time to resolve), MTTD (mean time to detect), incident count by category
- **Trend tracking:** are incidents increasing? Root cause categories shifting?

**Interview signal:** Discussing "MTTR is a key metric" + "blameless postmortem culture" shows SRE maturity.

---

## Q6-Q16: Additional System Design Questions (Condensed)

### Q6. Design CI/CD Pipeline for Production AI System ⭐⭐⭐
Every PR: lint + unit tests + type check + security scan + eval regression (Week 8) + build container + scan container. On merge to main: integration tests + staging deploy + smoke tests + prod deploy (canary). Rollback on failure. Deployment happens in parallel across regions. GitHub Actions + ArgoCD or similar.

### Q7. Design Cost Governance Platform for Enterprise AI ⭐⭐⭐⭐
Multi-level budgets: user daily, team monthly, department quarterly, org annual. Real-time cost tracking (per LLM call). Anomaly detection (2σ from baseline). Alert routing (department → team → user). Enforcement: soft limit (warning) at 80%, hard limit at 100%. Reporting: dashboards per level, weekly emails to owners, exec review monthly.

### Q8. Design K8s Deployment for AI Workloads With Autoscaling ⭐⭐⭐⭐
HPA (Horizontal Pod Autoscaler) with custom metrics: CPU + memory + queue depth + LLM latency. VPA (Vertical Pod Autoscaler) for right-sizing. Node autoscaler (Cluster Autoscaler + Karpenter). Pod Disruption Budget for graceful updates. Resource requests/limits based on actual usage. GPU node pools for self-hosted models.

### Q9. Design Feature Rollout System for AI Product Changes ⭐⭐⭐
Every change wrapped in feature flag. Gradual rollout: internal (5-10 users) → beta users (100) → 5% → 25% → 50% → 100%. Between stages: check quality metrics, error rate, user feedback. Kill switch on any flag. A/B testing framework for measuring impact. LaunchDarkly or Unleash.

### Q10. Design HIPAA-Compliant AI Infrastructure ⭐⭐⭐⭐
Business Associate Agreement with LLM providers (Azure OpenAI offers BAA). PHI never in logs, prompts, or persistent storage. Encryption at rest + in transit. Access controls (RBAC + audit). Dedicated infrastructure (not shared with non-HIPAA workloads). Data residency (US-only for US healthcare). Annual audit. Breach response plan.

### Q11. Design Streaming Chat Architecture at Scale ⭐⭐⭐
SSE for LLM token streaming to browser. WebSocket for bidirectional (user can send interrupts). Backend: FastAPI async with connection pooling. LLM Gateway with streaming pass-through. Handling: reconnection logic, mid-stream error handling, graceful degradation to non-streaming. Load balancer: sticky sessions for WebSocket, any-instance for SSE.

### Q12. Design Data Pipeline for LLM Training/Fine-Tuning ⭐⭐⭐
Data sources → ingestion (batch + streaming) → validation → PII scrubbing → deduplication → quality filtering → formatting → training dataset → version control. Tools: Airflow/Prefect for orchestration, dbt for transformations, DVC for dataset versioning. Compliance: data lineage tracking, opt-out enforcement.

### Q13. Design Model Serving Infrastructure for Self-Hosted LLMs ⭐⭐⭐⭐
GPU nodes (A100, H100). Model serving: vLLM or TGI for high throughput. Load balancing across GPUs (LB with GPU awareness). Autoscaling: scale on GPU utilization, not CPU. Model file distribution: S3 → node local storage. Warm pool of models. Cost optimization: mix of on-demand and spot GPUs.

### Q14. Design SLO/SLI Framework for AI Product ⭐⭐⭐
Define SLIs: availability, latency (TTFT + full response), quality score, cost efficiency. Set SLOs per tier: free (99%), paid (99.5%), enterprise (99.9%). Error budget: SLO violation budget in time. Dashboards: SLO burn rate. Alerts: fast burn (deplete monthly budget in <2h) → wake on-call. Monthly SLO review.

### Q15. Design Change Management Process for Production AI ⭐⭐⭐
Change categories: standard (well-tested, low risk, auto-approved), normal (staged rollout, review required), emergency (fast-track with post-approval). All changes: PR + review + tests + eval + gradual deploy. Change advisory board for high-risk. Change freeze windows (before major events). Full change log with rollback plans.

### Q16. Design Compliance and Audit Trail System ⭐⭐⭐⭐
Immutable audit log for: every LLM call, every access to sensitive data, every admin action, every deploy. Cryptographic tamper evidence (hash chain, blockchain-lite). Retention: 7 years for financial, 10+ healthcare. External auditor read-only access. Automated compliance reports (SOC 2, EU AI Act, HIPAA). Encrypted storage. Not GDPR-deletable (compliance requirement supersedes right-to-delete).
