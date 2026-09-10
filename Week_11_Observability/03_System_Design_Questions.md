# 🏗️ Week 11 — System Design Questions

> **Focus:** Enterprise observability platform, multi-tenant observability, OTel Collector architecture, cost anomaly detection systems, quality regression detection, compliance audit trails
>
> **How to use:** These are the questions that decide staff+ observability and AI infrastructure roles at Anthropic, Datadog, Google, Meta.

---

## Q1. Design an Enterprise-Grade Observability Platform for AI Workloads ⭐⭐⭐⭐

**Prompt:** "Design the complete observability stack for a company running 50 AI-powered services in production. Requirements: unified logs/metrics/traces, LLM-specific observability, cost tracking, quality monitoring, drift detection, compliance-ready audit logs, multi-tenant support."

**Architecture:**

```
┌────────────────── APPLICATION LAYER ────────────────────────┐
│  All services instrumented with OpenTelemetry SDK           │
│  ├── HTTP: auto-instrumented                                │
│  ├── DB: auto-instrumented                                  │
│  ├── LLM: custom instrumentation (GenAI semantic conv.)     │
│  ├── RAG: custom (retrieval, chunks, citations)             │
│  ├── Agents: custom (trajectory, tool calls)                │
│  └── Cost: attributes on every LLM span                     │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────── OTEL COLLECTOR (central telemetry gateway) ──┐
│  ├── Receive: OTLP (gRPC + HTTP)                            │
│  ├── Process:                                                │
│  │   ├── Tail-based sampling (keep errors, slow, sample)   │
│  │   ├── PII redaction (last-mile safety)                  │
│  │   ├── Attribute enrichment (env, region, service)       │
│  │   ├── Batching                                           │
│  │   └── Filtering (drop noise)                            │
│  └── Export: multi-backend fan-out                          │
└──────┬───────────────┬─────────────┬────────────────┬──────┘
       │               │             │                │
       ▼               ▼             ▼                ▼
┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐
│ Loki        │  │Prometheus│  │  Tempo   │  │  Langfuse   │
│ (logs)      │  │(metrics) │  │ (traces) │  │ (LLM traces)│
└──────┬──────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘
       └──────────────┴─────────────┘                │
                      │                              │
                      ▼                              ▼
              ┌────────────┐              ┌─────────────────┐
              │  Grafana   │              │  Langfuse UI    │
              │ (infra+app)│              │ (LLM-specific)  │
              └────────────┘              └─────────────────┘

┌────────────── ANALYTICS TIER ──────────────────────────────┐
│  ├── BigQuery / Snowflake (ad-hoc queries, drift analysis) │
│  ├── S3 archive (compliance retention)                     │
│  ├── ML pipeline (drift detection, anomaly detection)      │
│  └── Cost attribution warehouse                             │
└─────────────────────────────────────────────────────────────┘

┌────────────── ALERTING TIER ───────────────────────────────┐
│  ├── Alertmanager (metric-based alerts)                    │
│  ├── Custom alert engine (quality, drift, cost)            │
│  ├── PagerDuty (severity 1)                                 │
│  ├── Slack (severity 2)                                    │
│  └── Email (severity 3)                                    │
└─────────────────────────────────────────────────────────────┘

┌────────────── COMPLIANCE TIER (isolated) ──────────────────┐
│  ├── Immutable audit log (append-only)                     │
│  ├── Hash chain for tamper evidence                        │
│  ├── Encrypted storage (per-tenant keys)                   │
│  ├── 7-year retention                                       │
│  ├── External auditor read-only access                     │
│  └── SOC 2 / HIPAA / EU AI Act ready                       │
└─────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

**1. OpenTelemetry everywhere.**
Single instrumentation standard. Every service instruments once. Backend can change without touching app code. Non-negotiable for enterprise (avoid vendor lock-in on observability).

**2. OTel Collector as central gateway.**
Between apps and backends. Handles: sampling policy, PII redaction, transformation, batching, retry, multi-export. Change tools by changing Collector config, not app code.

**3. Multi-backend fan-out.**
- Grafana stack (Loki + Prometheus + Tempo): infrastructure + application layer, cost-effective at scale
- Langfuse (or LangSmith): LLM-specific traces, prompt versioning, eval integration
- BigQuery/Snowflake: ad-hoc analytics, ML on telemetry data
- Compliance-grade separate storage: audit logs cannot commingle with operational

Layered — different tools for different jobs. Don't try to force one tool to do everything.

**4. Sampling strategy.**
Tail-based at Collector level:
- 100% of errors
- 100% of slow (> p95 baseline)
- 100% of premium tier customers
- 10% of everything else

Keeps cost bounded while preserving signal.

**5. PII redaction last-mile.**
In OTel Collector, not app. Enforces org-wide policy. App developers can't accidentally leak PII. Uses Presidio or similar for detection.

**6. Cost as first-class dimension.**
Every LLM span has `gen_ai.cost.usd` attribute. Aggregates roll up to Prometheus metrics (bounded cardinality) + BigQuery for analytics. Per-tenant/user/feature dashboards.

**7. Compliance tier is separate.**
Different security posture. Different retention. Different access control. Different infrastructure. Not "just a log system with extra retention" — genuinely different tier.

**Scale considerations:**

At 50 services with 100 QPS each = 5000 QPS total:
- Traces: 5000/s × 10 spans/trace × 10% sampled = 5000 spans/s → Tempo
- Metrics: 500 series/service × 50 services = 25K series → Prometheus
- Logs: 5000/s × 5 log lines/req = 25K logs/s → Loki
- LLM traces: 5000/s × 30% AI-touching × 100% sampled = 1500 LLM traces/s → Langfuse

Storage:
- Traces: ~500GB/day
- Metrics: ~5GB/day (aggregated)
- Logs: ~1TB/day
- LLM traces (with prompts): ~200GB/day

Cost estimate: $30-50K/month total observability infrastructure at this scale.

**Interview signal:** "The compliance tier is genuinely separate — you can't just extend operational logs" shows enterprise thinking.

---

## Q2. Design a Multi-Tenant Observability System With Per-Tenant Dashboards ⭐⭐⭐⭐

**Prompt:** "Design observability for a multi-tenant SaaS AI product. Each tenant needs their own dashboards, alerts, cost tracking. Internal team sees everything. Support 500+ tenants."

**Isolation model:**

**Data ingestion (shared):**
- Single OTel Collector pipeline
- All tenants' telemetry flows through same infrastructure
- Every event tagged with `tenant_id` label/attribute

**Data storage (shared with tenant isolation via labels):**
- Prometheus: metrics with `tenant_id` label
- Loki: logs with `tenant_id` label
- Langfuse: multi-tenancy built-in

**Query layer (tenant-scoped):**
- Tenant queries auto-filtered by their tenant_id
- Internal team can query across tenants
- Row-level security in query layer

**Dashboard layer (per-tenant):**
- Template-based: one dashboard template
- Instantiated per tenant with tenant_id parameter
- Version-controlled templates (dashboard-as-code)

**Architecture:**

```
Internal team dashboards ────► Query all tenants
                              (cross-tenant analytics)

Tenant A dashboard ─────────► Query filtered: tenant_id=A
Tenant B dashboard ─────────► Query filtered: tenant_id=B
...
Tenant Z dashboard ─────────► Query filtered: tenant_id=Z
                              (single template, parameterized)
```

**Tenant tier differentiation:**

| Tier | Traces Sampled | Metrics Detail | Retention | Custom Dashboards |
|---|---|---|---|---|
| Free | 5% | Basic | 7 days | No (fixed template) |
| Business | 25% | Standard | 30 days | Limited |
| Enterprise | 100% | Full + custom | 90 days | Yes |
| Regulated | 100% + audit tier | Full | 7 years | Yes + compliance |

**Cost attribution:**

Every telemetry event has tenant_id. Aggregate storage/processing costs to tenant level. Enterprise customers get monthly reports: "You used X observability resources this month."

Chargeback: enterprise pays for observability as line item.

**Alerts per tenant:**

- Tenant's own metrics trigger alerts to tenant's Slack/email
- Internal alerts (across tenants) trigger to on-call
- Tenant can configure their own alerts within their tier

**Data isolation checks:**

- Automated: query as tenant A, verify no tenant B data returned
- Runs every deploy
- Blocks deployment if isolation regression

**GDPR / compliance:**

Right-to-delete: tenant deletion cascades to all observability data
Data residency: EU tenants' data stays in EU region
Retention: enforced per tenant tier

**Interview signal:** "Dashboards as code, parameterized by tenant, with tier-based sampling and retention" shows deep multi-tenant thinking.

---

## Q3. Design a Silent Quality Regression Detection System ⭐⭐⭐⭐

**Prompt:** "Design a system that catches when AI quality regresses in production, even when nothing was deployed and no user complains. Detect drift, model version changes, prompt regressions."

**The challenge:**

- User complaints lag reality by days/weeks
- No error signals (AI failed silently with 200 status)
- Provider models can silently update
- Slow-moving degradations invisible day-to-day

**Detection layers:**

**Layer 1: Golden set continuous evaluation**

- 100+ curated queries with known-good answers
- Runs every hour against production
- Metrics: exact match, semantic similarity to expected, LLM-judge quality score
- Baseline: rolling 7-day average
- Alert: current score drops > 5% from baseline

**Layer 2: LLM-as-judge on production sample**

- 10% of production traffic judged
- Rubric: faithfulness, relevance, helpfulness (1-5 each)
- Aggregate: rolling 24-hour quality score
- Baseline: rolling 30-day average
- Alert: quality drop > 10% from baseline

**Layer 3: Statistical drift detection**

- Input distribution: PSI on query embeddings (weekly)
- Output distribution: response length, token count (daily)
- Embedding drift: cluster analysis (weekly)
- Alert: PSI > 0.2 on any dimension

**Layer 4: User feedback correlation**

- Thumbs-down rate over time
- Complaint category classification
- Correlate: quality metrics + feedback trends
- Alert: feedback trend degrading + quality metrics degrading

**Layer 5: Model version fingerprinting**

- Weekly: run standardized probing queries
- Capture: response patterns, style markers, factual claims
- Compare: has the model's "voice" changed?
- Alert: significant fingerprint shift (provider may have updated)

**Architecture:**

```
┌─────── Continuous Golden Set ────┐
│  100 queries × hourly            │  → hourly_quality_score
└──────────────┬───────────────────┘

┌─────── Production Sampling ──────┐
│  10% sampled for LLM-as-judge   │  → 24h_quality_rolling
└──────────────┬───────────────────┘

┌─────── Drift Analysis ───────────┐
│  Weekly PSI + KL divergence     │  → drift_scores
└──────────────┬───────────────────┘

┌─────── Feedback Aggregation ─────┐
│  Thumbs-down + categorization    │  → feedback_trend
└──────────────┬───────────────────┘

┌─────── Model Fingerprinting ─────┐
│  Weekly probing queries          │  → fingerprint_delta
└──────────────┬───────────────────┘

              ▼
┌───── Composite Quality Signal ───┐
│  Weighted combination            │
│  Baseline: rolling 30d           │
│  Alert threshold: 10% drop       │
└──────────────┬───────────────────┘

              ▼
     Notification + investigation flow
```

**Investigation flow when alert fires:**

1. Correlate: what changed?
   - Recent deploys?
   - Prompt updates?
   - Model announcements from provider?
   - Traffic pattern shift?

2. Isolate: which slice is affected?
   - By tenant?
   - By feature?
   - By query type?

3. Compare: good period vs bad period
   - Same golden queries, both time windows
   - Sample bad responses, categorize

4. Hypothesize + test:
   - Suspected cause identified
   - Test with A/B (revert change on 10% traffic)
   - Confirm/reject hypothesis

5. Remediate:
   - Rollback if possible
   - Otherwise: forward fix with monitoring

**Cost of running this system:**

- Golden set: 100 queries × hourly × $0.05 = $12/day = $360/month
- LLM-as-judge on 10%: at 1M queries/day, judging 100K = $100/day = $3K/month
- Drift analysis compute: $100/month
- Total: ~$3.5K/month

For any product where AI quality matters, this is a small price for early detection of expensive regressions.

**Interview signal:** "Golden set runs continuously, LLM-judge samples production, statistical drift complements both" shows layered thinking.

---

## Q4. Design a Cost Anomaly Detection and Response System ⭐⭐⭐⭐

**Prompt:** "Design a system that detects unusual cost patterns in AI usage BEFORE they become $50K weekend disasters. Automated response where safe. Human escalation where risky."

**Detection dimensions:**

- Total organization spend (hourly, daily)
- Per-tenant spend
- Per-user spend
- Per-feature spend
- Per-model spend
- Per-endpoint spend

**Baseline computation:**

- Rolling 7-day average per dimension
- Rolling 30-day average for longer trends
- Time-of-day patterns (Tuesday 3am ≠ Monday 10am)
- Day-of-week patterns
- Seasonal patterns (Black Friday spikes expected)

**Anomaly detection methods:**

**Simple:** current > mean + 2σ (moderate) or > mean + 3σ (severe).

**Better:** decompose time series (trend + seasonal + residual), alert on residual outliers.

**Best:** ML-based (Prophet, isolation forest, LSTM) that learn normal patterns.

**Trade-off:** simpler = fewer false positives from clear anomalies, more false negatives on subtle patterns.

**Automated responses (by severity):**

**Severity 1: Spike > 10x baseline in 1 hour**
- Auto-action: apply hard cost cap immediately (protect budget)
- Notify: page on-call, notify CFO
- Investigation: who/what caused spike?

**Severity 2: Spike > 3x baseline sustained 30 min**
- Auto-action: throttle offending user/feature
- Notify: page on-call
- Investigation: dashboard alerts

**Severity 3: Spike > 2x baseline sustained 4 hours**
- Auto-action: none (may be legitimate growth)
- Notify: Slack to team
- Investigation: async during business hours

**Severity 4: Slow drift (30d cost trending up)**
- Auto-action: none
- Notify: weekly cost review meeting
- Investigation: planned analysis

**Automated response safeguards:**

- Never auto-disable a whole tenant (business risk)
- Never auto-disable production features (users notice)
- Always notify humans of auto-actions (they might override)
- Time-limited (auto-throttles expire after 1 hour unless renewed)

**Forecasting:**

Beyond anomaly detection: PROJECT end-of-month cost from current burn rate.

- Simple: (current daily average × days in month)
- Better: trending (accounting for growth)
- Best: seasonal (Black Friday, holidays, seasonal patterns)

If forecasted > budget → alert BEFORE budget breach.

**Attribution:**

When spike detected, drill into:
- Which user contributed most?
- Which feature contributed most?
- Was it one bad request or many?
- Was it expensive per-request or high-volume?

Each pattern has different remediation.

**Enterprise concerns:**

- **False positive management:** false alarms cause fatigue. Tune thresholds. Auto-suppress known-legitimate spikes (deploy caused expected cost bump).
- **Cross-tenant fairness:** one abusive tenant can't burn budget of shared infrastructure.
- **Budget planning:** cost data feeds finance planning. Reports monthly to CFO.
- **Chargeback:** enterprise tenants billed based on actual usage.

**Interview signal:** Discussing "auto-throttle with time-limited scope + human notification" shows safety-conscious automation.

---

## Q5. Design an Incident Investigation Platform for AI Systems ⭐⭐⭐⭐

**Prompt:** "Design a platform that helps engineers investigate AI incidents fast. Given a bad response, engineer should be able to root-cause in minutes. Support: hallucinations, cost spikes, quality regressions."

**Core workflow:**

```
1. Get trace_id (from user report, alert, or dashboard)
    ↓
2. Load full trace: every span, every attribute
    ↓
3. Understand what happened: reconstruct the request lifecycle
    ↓
4. Identify anomaly: what's different from normal traces?
    ↓
5. Root cause: correlate with recent changes
    ↓
6. Fix: patch, verify, prevent recurrence
```

**Trace UI features:**

**Timeline view:**
- Every span with start/end timing
- Critical path highlighted
- Slow spans visually distinct
- Errors marked

**Prompt/response view:**
- Full prompt (with PII redaction)
- LLM response
- System prompt version
- Model + parameters used

**Retrieval view:**
- Query and query embedding
- Retrieved chunks with scores
- Reranked chunks with scores
- Chunks that were cited in response

**Cost view:**
- Cost per stage
- Total cost
- Comparison to similar requests

**Context view:**
- User context (persistent memory)
- Session context
- Feature flags active

**Comparison mode:**

- Load two traces side-by-side
- Compare: what's different?
- Common: bad trace vs good trace of similar request
- Common: current trace vs baseline (from 2 weeks ago when it worked)

**Root cause suggestions (ML-assisted):**

Based on trace characteristics, suggest likely root causes:

- Slow LLM span → provider slowness, model change, prompt bloat
- Low retrieval scores → embedding drift, chunking issue, index staleness
- Empty retrieval → out-of-domain query, index issue
- Hallucinated response → context missing key info OR model ignoring context

**Correlation with observability:**

- Was there a deploy in the last hour?
- Was there a prompt change?
- Did provider announce anything?
- Are similar traces also failing?
- Is quality regression system alerting?

**Search across traces:**

- "Show me all traces where response mentioned 'CEO'" (find affected)
- "Show me traces where retrieval score < 0.5" (find quality issues)
- "Show me traces from tenant X in last 2 hours" (customer support)
- "Show me traces with cost > $1" (cost investigation)

**Reproduction:**

- Given trace, replay the exact query
- With original context and configuration
- Compare live result to historical
- Debug interactively

**Enterprise concerns:**

- Access control: engineers see only tenants they're authorized for
- PII: all views apply redaction
- Audit: every trace access logged
- Retention: traces retained per compliance requirement

**Tools that do this:**
- Langfuse (best-in-class LLM trace UI)
- LangSmith (LangChain-specific)
- Arize Phoenix (RAG-focused)
- Honeycomb (general distributed tracing)

**Interview signal:** "The most important feature is comparison mode — bad trace vs good trace side by side" shows debugging experience.

---

## Q6-Q16: Additional System Design Questions (Condensed)

### Q6. Design a Real-Time Cost Dashboard for AI Applications ⭐⭐⭐
Live cost per hour, breakdown by dimensions (user, tenant, feature, model). Forecast to end-of-month. Comparison to budget. Anomaly indicators. Refresh every minute. Historical trend last 30 days. Executive view (roll-ups) + engineering view (per-service detail).

### Q7. Design a Prompt Version Management System With Observability ⭐⭐⭐⭐
Every prompt in git, versioned. Metadata: version, author, changes, expected impact. Every LLM call tagged with prompt version. Quality metrics per prompt version. A/B testing between versions. Rollback capability. Dashboard: performance by prompt version over time.

### Q8. Design an Agent Trajectory Analytics Platform ⭐⭐⭐
Aggregate trajectories over time. Metrics: avg steps per task, tool distribution, cost per completion, failure mode categorization. Compare agents (A vs B). Identify anti-patterns (loops, wasted calls). Suggest optimization (unused tools, redundant calls). Feed improvements back.

### Q9. Design a Compliance-Grade Audit Log for AI ⭐⭐⭐⭐
Immutable append-only. Fields: timestamp, user, tenant, action, resource, prompt hash, response hash, model version, IP. Hash chain (each entry hashes previous). Encrypted at rest with per-tenant keys. 7-year retention. External read-only auditor access. Separate infra from operational logs.

### Q10. Design a Feedback Loop From Production to Training Data ⭐⭐⭐
Users provide thumbs-down + reason. Categorize (LLM-classified). Route to appropriate improvement queue: prompt improvements, model fine-tuning candidates, RAG index gaps. Golden set updates. Weekly review by team. Track: which categories are improving, which stagnating.

### Q11. Design a Multi-Region Observability With Local Compliance ⭐⭐⭐⭐
Data residency requires: EU telemetry stays in EU, US in US. Regional Collectors. Cross-region aggregation only for non-PII metrics. Regional dashboards. Global executive view (aggregates only). Compliance evidence per region.

### Q12. Design an SLO Framework and Dashboard for AI Products ⭐⭐⭐
SLIs: availability, latency (TTFT, E2E), quality score, cost efficiency. SLOs per tier (free 99%, paid 99.5%, enterprise 99.9%). Error budget calculations. Burn rate alerting (fast + slow). Monthly SLO reviews. Customer-facing SLO status page.

### Q13. Design a Data Pipeline Observability System ⭐⭐⭐
Ingestion pipeline: docs added per day, failures, latency per stage, dedup rate. Embedding pipeline: embeddings generated, failures, cost. Vector index: size growth, query latency, empty result rate. Data lineage: source to chunk traceability. Alerts on pipeline health degradation.

### Q14. Design a Drift Response Automation System ⭐⭐⭐
Drift detection triggers → automated investigation → categorized suggestions. Data drift → suggest index refresh. Model drift → suggest golden set re-run + provider check. Embedding drift → suggest re-embed. Never fully auto-remediate (human judgment needed). Track: drift → response → resolution.

### Q15. Design a Chaos Engineering System for AI ⭐⭐⭐
Chaos experiments: inject LLM provider outages, slow retrieval, corrupt context, high latency. Verify: does fallback work? Does graceful degradation activate? Does monitoring detect? Run in staging weekly, production monthly. Track: mean time to detect, mean time to recover, incidents prevented.

### Q16. Design a Reporting System for AI Executive Stakeholders ⭐⭐⭐
Executive dashboard: uptime, cost trend, quality trend, user satisfaction. Weekly PDF report generated automatically. Monthly business review deck. Anomalies flagged with context. Board-ready. Data-driven (not vanity metrics). Different audiences (CEO, CFO, CPO) see different views.
