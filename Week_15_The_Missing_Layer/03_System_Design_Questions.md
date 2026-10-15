# 🏗️ Week 15 — System Design Questions

> **Focus:** Multi-tenant SaaS AI platform, cost-tiered LLM serving with break-even math, reasoning gateway routing, permission-aware RAG across SharePoint/Confluence/Salesforce, GDPR right-to-erasure workflow, EU AI Act compliance architecture
>
> **Depth:** MAANG staff+ / enterprise principal architect bar. Every design includes cost math, compliance mapping, and specific tech choices with justification.

---

## Q1. Design a Multi-Tenant AI SaaS Platform Supporting SMB Through Fortune 500 ⭐⭐⭐⭐

**Prompt:** "Design an AI-powered platform (e.g., customer support automation) serving 10K tenants ranging from 5-person startups to Fortune 500. Support: shared/pooled/dedicated/silo tiers. HIPAA-eligible tier. EU residency. Cost-competitive at each tier."

### Architecture:

```
                    ┌─────────────────────────┐
                    │  Global Edge (CDN, WAF) │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Regional Gateway      │  (auth, tenancy resolution, routing)
                    │   (US-East, EU-West,    │
                    │    US-Gov)              │
                    └───┬────────┬────────┬───┘
                        │        │        │
       ┌────────────────▼─┐  ┌───▼─────┐  ▼
       │  SHARED TIER     │  │ POOLED  │  ┌──────────────────┐
       │  Free/SMB        │  │ TIER    │  │  ENTERPRISE TIER │
       │                  │  │ Standard│  │                  │
       │  Shared LLM      │  │ B2B     │  │  Dedicated LLM   │
       │  Shared Vector DB│  │         │  │  Isolated Vector │
       │  Shared cache    │  │  Pooled │  │  Per-tenant cache│
       │  Namespaced by   │  │  infra +│  │                  │
       │  tenant_id       │  │  logical│  │  BAA-eligible    │
       │                  │  │  iso    │  │  (Azure OpenAI,  │
       │                  │  │         │  │   Anthropic Ent) │
       └──────────────────┘  └─────────┘  └──────────────────┘
                                          
                                          ┌──────────────────┐
                                          │  SILO TIER       │
                                          │  Full isolation  │
                                          │  Per-tenant VPC  │
                                          │  In-region only  │
                                          │  Contract-driven │
                                          └──────────────────┘
```

### Tier Design and Pricing:

| Tier | Isolation | Cost/User/Mo | Price/User/Mo | Gross Margin | Compliance |
|---|---|---|---|---|---|
| Free (Shared) | Logical namespace | $0.10 | $0 (ads/upgrades) | -100% (loss leader) | Basic |
| SMB (Pooled) | Logical + cache namespace | $0.50 | $9 | 94% | SOC 2 |
| Business (Pooled+) | Same + PII redaction | $2 | $30 | 93% | SOC 2, GDPR |
| Enterprise (Dedicated) | Dedicated inference | $8 | $80 | 90% | SOC 2, HIPAA-eligible, BAA |
| Silo | Full isolation | $50 | $500 flat + usage | Variable | SOC 2 + HIPAA + FedRAMP-ready |

### Key Design Decisions:

**1. Multi-region deployment from day one.**

Regions: US-East (primary), EU-West (residency), US-Gov (federal). Each region has full stack. Cross-region only for global observability aggregation (deidentified).

**2. Tenant resolution at gateway.**

Every request → resolve tenant_id → look up tier + residency + BAA scope → route to appropriate backend cluster.

**3. Cache namespace strategy.**

- Shared/SMB: `cache:{tenant_id}:{hash}` — logical isolation
- Business+: separate cache clusters per tier (cache poisoning contained)
- Enterprise: dedicated cache per-tenant
- Silo: full separate cache infrastructure

**4. Provider routing.**

- Non-BAA workloads: any provider (cost-optimized)
- BAA workloads: only BAA-eligible providers (Azure OpenAI, Anthropic Enterprise, AWS Bedrock)
- EU residency: EU-hosted providers or self-hosted OSS models
- Gov workloads: Azure Gov or AWS GovCloud only

**5. Data plane per tier.**

- Shared: single Postgres cluster with tenant_id partitioning
- Pooled: sharded Postgres by tenant_id hash
- Enterprise: dedicated Postgres cluster per group of tenants
- Silo: per-tenant Postgres

### Cost Math (10K tenants):

Assumptions:
- 60% shared (6K tenants × 5 users × 10 queries/day)
- 30% SMB (3K tenants × 15 users × 20 queries)
- 8% enterprise (800 tenants × 100 users × 50 queries)
- 2% silo (200 tenants × 500 users × 100 queries)

Total queries/day:
- Shared: 6K × 5 × 10 = 300K
- SMB: 3K × 15 × 20 = 900K
- Enterprise: 800 × 100 × 50 = 4M
- Silo: 200 × 500 × 100 = 10M

Cost breakdown (with tiering, caching, prompt cache):
- Shared: 300K × $0.0005 = $150/day (Haiku, cached)
- SMB: 900K × $0.002 = $1,800/day (Sonnet-mostly, cached)
- Enterprise: 4M × $0.005 = $20,000/day (Sonnet + BAA overhead)
- Silo: 10M × $0.008 = $80,000/day (per-tenant dedicated overhead)

**Total: ~$3M/month LLM costs at 10K tenants**

Revenue:
- Shared: $0 (30K users, loss leader for conversion)
- SMB: $30/user × 45K users = $1.35M/mo
- Enterprise: $80/user × 80K users = $6.4M/mo
- Silo: $500 + usage × 100K users = $50M+/mo

Blended margin ~85% at scale.

### Failure Modes:

- **Free tier abuse:** rate limiting per anonymous IP + user
- **Cross-tenant contamination:** contract tests + audit trail + code review discipline
- **BAA scope creep:** dedicated infrastructure for HIPAA-eligible tenants; never mix
- **Cost blowup:** per-tenant budgets + circuit breakers
- **Provider outage:** multi-provider fallback for non-BAA; single provider with escalation SLA for BAA

---

## Q2. Design a Reasoning-Aware LLM Gateway ⭐⭐⭐⭐

**Prompt:** "Build a gateway that routes queries to the right model (standard vs reasoning) based on task, budget, latency SLA, and expected ROI. Support all major providers and open-source reasoning models."

### Architecture:

```
Client
  ↓
Gateway (auth, tenant resolution)
  ↓
Query Classifier
  ├── Extract features: task type, expected complexity, urgency, budget tier
  ├── Predict: reasoning benefit score (0-1)
  └── Latency budget check
  ↓
Router
  ├── If reasoning_benefit > threshold AND latency budget > 10s:
  │    → Reasoning tier (o1, o3, R1, Claude thinking)
  │
  ├── Else if complexity > medium AND latency budget > 3s:
  │    → Standard-large (Sonnet, GPT-4o, Opus)
  │
  └── Else:
       → Fast tier (Haiku, Flash, GPT-4o-mini)
  ↓
Provider selection (BAA scope, residency, cost)
  ↓
Execute + cost tracking + audit
  ↓
Response (with metadata: model used, cost, latency, reasoning_used)
```

### Query Classification:

Small classifier LLM (or trained model) scores each query on:
- **Reasoning benefit** (0-1): does chain-of-thought or search help?
- **Complexity** (low/medium/high): baseline model capability required
- **Verifiability**: can we check the answer? (enables best-of-N)
- **Cost tolerance**: from tenant/user config

### Reasoning Decision Table:

| Query type | Reasoning benefit | Decision |
|---|---|---|
| Classification, extraction | 0.0-0.2 | Fast tier |
| Standard RAG Q&A | 0.1-0.3 | Standard tier |
| Multi-step planning | 0.6-0.9 | Reasoning tier |
| Math, logic, code | 0.7-0.95 | Reasoning tier |
| Complex synthesis | 0.5-0.7 | Standard-large or Reasoning |
| Creative writing | 0.1-0.3 | Standard tier |

### Cost & Latency Tracking:

Per request emit:
```json
{
  "request_id": "...",
  "tenant_id": "...",
  "classifier_scores": {"reasoning": 0.72, "complexity": "high"},
  "router_decision": "reasoning_tier",
  "model_used": "o1",
  "provider": "openai",
  "cost_usd": 0.42,
  "latency_ms": 8300,
  "reasoning_thinking_tokens": 2400,
  "output_tokens": 320,
  "budget_remaining_usd": 42.10
}
```

### Fallback Logic:

- Reasoning provider down → fall back to `best available reasoning model`
- Reasoning model too slow (approaching latency SLA) → interrupt and fall back to standard
- Cost budget exceeded → refuse or route to cheaper tier with warning

### Example ROI Numbers (from Episode 15 demos):

- Legal use case: reasoning tier reduces end-to-end cost 70% (fewer expert hours)
- Customer support: reasoning ties standard — don't pay the premium
- Agent planning: reasoning saves 53% (fewer wasted tool calls)

Track these per-feature. Adjust routing based on observed ROI.

---

## Q3. Design Permission-Aware RAG Across SharePoint + Confluence + Salesforce ⭐⭐⭐⭐

**Prompt:** "Design a RAG system that ingests from SharePoint, Confluence, and Salesforce. Users query and get answers only from documents THEY can see in the source systems. Handle ACL changes in near-real-time."

### Architecture:

```
[Source Systems]                    [Sync Layer]
                                    
SharePoint  ─┐                     ┌─── SharePoint Connector
              │                     │    (Graph API, deltas)
Confluence  ──┤                     ├─── Confluence Connector
              │─── SCIM 2.0  ──────►│    (v2 API, webhooks)
Salesforce  ──┤                     ├─── Salesforce Connector
              │                     │    (Metadata + Streaming API)
IdP (SSO)  ──┘                     └─── Identity Sync
                                    
                                    [Central RAG Index]
                                    ├── Documents + embeddings
                                    ├── ACLs per document
                                    │   (allowed_users, allowed_groups)
                                    ├── Classification per document
                                    └── Source lineage
                                    
                                    [Query Path]
Client → Auth (SSO) → Resolve user's groups → Pre-filter index →
Vector search allowed subset → Rerank → LLM synthesis with citations →
Audit log emission → Response
```

### Sync Strategy:

**Full sync nightly + incremental delta every 5-15 minutes.**

- **SharePoint:** Microsoft Graph delta queries. Watch for permission changes on documents + sites.
- **Confluence:** REST API with `lastModified` filter. Space permission changes trigger cascade re-sync.
- **Salesforce:** Metadata API + Streaming API (Push Topics). Record-level sharing rules.

### The Four-Layer Enforcement:

**Layer 1 — Source ACL Sync**
Continuously sync ACL state from each source to central index metadata.

**Layer 2 — Index-Time Metadata**
Every embedded document tagged with:
```json
{
  "allowed_users": ["u1", "u2", ...],
  "allowed_groups": ["g1", "g2", ...],
  "classification": "confidential",
  "source_system": "sharepoint",
  "source_document_id": "...",
  "last_permission_sync": "iso8601"
}
```

**Layer 3 — Query-Time Pre-Filter**
Resolve user's effective groups via SCIM-provisioned graph. Pre-filter index. Vector search over authorized subset.

**Layer 4 — Post-Retrieval Audit**
Every retrieval logged with user, query, docs returned, ACL state, response hash. Hash-chained audit trail.

### Deletion Propagation:

Critical case: user removed from source group → they should immediately lose access to previously indexed docs.

Two strategies:
- **Sync on-read:** every query re-validates ACLs against source (slower, always correct)
- **Sync on-write + TTL:** ACL sync every 5-15 min, plus max-staleness TTL (~15 min) forces re-check

Enterprise SLA: 'ACL changes propagate within 15 minutes.'

### Handling Massive Enterprise Scale:

- 10M documents across all sources
- 100K users across many groups
- Group membership can be transitive (nested groups)

Precompute effective permissions: `user → all groups (including nested) → all doc IDs accessible`. Update on group membership change.

### Failure Modes:

- **Stale ACLs:** monitored + alerted (sync SLA metric)
- **Sync connector failure:** partial data view; fallback to source system on-read validation
- **Nested group cycles:** detect and break
- **Massive user with 10K group memberships:** cache resolved permissions; refresh on change

---

## Q4. Design a GDPR Right-to-Erasure Workflow Across a Multi-Region SaaS ⭐⭐⭐⭐

**Prompt:** "User submits Article 17 request. Design the workflow to delete all traces within 30 days across your multi-region AI SaaS. Include backups, analytics, audit trail exceptions."

### Architecture:

```
User submits erasure request (via UI, email, DPO)
  ↓
Erasure Coordinator (workflow engine: Temporal, Step Functions, custom)
  ↓
┌───────────────────────────────────────────────────────┐
│  Parallel deletion across all data plane systems      │
├───────────────────────────────────────────────────────┤
│  1. Identity: mark account deleted, deprovision SCIM  │
│  2. App DB: DELETE user rows (cascade)                │
│  3. Vector DB: filter delete by user_id metadata      │
│  4. Semantic cache: flush user-scoped entries         │
│  5. Prompt store: delete user prompts                 │
│  6. Logs: mask user_id, keep event structure          │
│  7. Backups: schedule exclusion on next restore       │
│  8. Analytics warehouse: DELETE + row-level policies  │
│  9. SIEM: mask user_id (retention for security)       │
│  10. Cold archive: mark for deletion on next rotation │
│  11. Audit trail: FLAG (retain per legal req)         │
└───────────────────────────────────────────────────────┘
  ↓
Verification: reconciliation queries across all systems
  ↓
Confirmation to user (< 30 days)
  ↓
Deletion evidence stored (compliance record)
```

### Legal Considerations:

**Cannot delete:**
- Audit trail entries (retention required by SOC 2, SOX)
- Records under legal hold
- Anonymized aggregate data (not personal data anymore)

**Must delete:**
- All identifiable personal data
- Both structured (DB rows) and unstructured (prompts, responses containing user info)

**Retention exception documentation:** legal basis for retention documented per record.

### Handling Backups:

Cannot practically delete from every backup snapshot. Instead:
- Restore-time exclusion: when restoring from backup, filter out deleted users
- Progressive rotation: deleted-user backups age out per retention policy (90 days typical)
- Document the process for the DPA / regulator

### Cross-Region Coordination:

User's data spans multiple regions if traveled. Erasure must run in all regions.

Regional erasure workflows coordinate via central orchestrator. Verify each region's completion.

### Special Case: LLM Provider Cache/Training Data

Some providers cache prompts for observability or (historically) training. Enterprise contracts specify no-training. But provider-side caches (5-min TTL for prompt cache) may briefly contain user data.

Best practice: notify provider of deletion (contractual requirement in DPA). Trust their compliance.

Auditable trail: 'user X requested erasure at time T; deletion completed across all systems by T+27 days; provider notified at T+2 days.'

---

## Q5-Q16: Additional System Design Questions (Condensed)

### Q5. Design an EU AI Act-Compliant High-Risk AI System ⭐⭐⭐⭐
CV screening tool (high-risk under EU AI Act). Required: risk management system, data governance, transparency to users, human oversight, accuracy monitoring, cybersecurity, conformity assessment. Design each subsystem with control mapping.

### Q6. Design a Multi-Cloud LLM Gateway ⭐⭐⭐⭐
Route between AWS Bedrock, Azure OpenAI, Anthropic, Google Vertex. Failover < 500ms. Cost optimization. Compliance-aware (residency, BAA). Cross-provider observability.

### Q7. Design a Cost-Aware Feature Flag System ⭐⭐⭐
Each feature has a cost budget. Feature flags gate rollout based on current spend. Automatic slowdown or hard cutoff. Per-tenant budgets.

### Q8. Design SAML/OIDC SSO Integration for a Consumer AI Product Going Enterprise ⭐⭐⭐
Support: Okta, Azure AD, Google Workspace, Ping. SCIM 2.0 provisioning. Just-in-time user creation. Group-based feature access. Migration path from existing accounts.

### Q9. Design a Reasoning Model Routing Gateway ⭐⭐⭐⭐
Fine-grained routing between reasoning models (o1, o3, R1, Claude thinking). Consider: cost, latency, provider availability, quality per task. Per-tenant routing preferences.

### Q10. Design a Multi-Device Session Sync System ⭐⭐⭐
Users seamlessly move conversations between mobile, web, desktop. Real-time sync. Conflict resolution. Offline mode with sync-on-reconnect.

### Q11. Design a Prompt Injection Detection Pipeline for a Multi-Layer LLM App ⭐⭐⭐⭐
Every layer (user input, RAG docs, tool outputs) monitored. Multi-classifier ensemble. Real-time alerting. Threat intelligence feed integration.

### Q12. Design an Agent Framework With Cost + Time Budgets ⭐⭐⭐⭐
Agent has hard budget (e.g., $0.50, 60 seconds). Track spend + time per iteration. Graceful degradation when budget approaches limit. Interruptible mid-run.

### Q13. Design PCI-DSS-Compliant AI System for Payment Data ⭐⭐⭐
Tokenize card data before any LLM sees it. Segment PCI zone from AI zone. Audit every access. Support authorized use cases (fraud detection).

### Q14. Design a US State Law Compliance Engine ⭐⭐⭐
CCPA, VCDPA, CTDPA, Colorado AI Act, Illinois BIPA, etc. Per-state rules encoded. Route requests through compliance middleware. Rejection with control code.

### Q15. Design Cross-Border Data Transfer Governance ⭐⭐⭐
Track every data flow across borders. Verify transfer instrument exists (adequacy, SCCs, BCRs). Alert on violations. Support regulatory audit queries.

### Q16. Design a Complete Observability Stack for a Regulated AI System ⭐⭐⭐⭐
Metrics + logs + traces + audit + compliance monitoring. Prometheus + Grafana + Loki + OpenTelemetry + SIEM integration. Per-tenant + per-feature dashboards. Alerting with runbooks. Cost dashboards. Regulatory reporting exports.
