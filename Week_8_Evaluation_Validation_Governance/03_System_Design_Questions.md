# 🏗️ Week 8 — System Design Questions

> **Focus:** Enterprise eval platforms, continuous eval CI/CD, regulatory compliance systems, red teaming pipelines, A/B testing infrastructure, trust dashboards, golden dataset governance
>
> **How to use:** These are staff+ whiteboard questions. Interviewers give you a scenario, expect you to draw the architecture, discuss trade-offs, and dive deep on the critical components.

---

## Q1. Design an Enterprise-Wide LLM Evaluation Platform ⭐⭐⭐⭐

**Prompt:** "Your company has 15 teams shipping LLM features. Each has their own ad-hoc eval scripts. Some don't eval at all. Design a centralized eval platform that all teams use — supporting offline batch eval, CI/CD gates, online production monitoring, and executive reporting."

**Architecture:**

```
┌────────────────────── EVAL PLATFORM ─────────────────────────────┐
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           GOLDEN DATASET REGISTRY                            │  │
│  │  Version-controlled datasets (S3 + Postgres metadata)        │  │
│  │  - Per team + shared datasets                                │  │
│  │  - Approval workflow for changes                             │  │
│  │  - Access control (RBAC)                                     │  │
│  │  - Full audit history                                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           METRIC REGISTRY                                    │  │
│  │  Built-in metrics: RAGAS, DeepEval, custom G-Eval           │  │
│  │  Custom metrics: per-team domain metrics                     │  │
│  │  Version-controlled metric definitions                       │  │
│  │  Metric ownership + documentation                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           EVAL ORCHESTRATOR                                  │  │
│  │  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐   │  │
│  │  │ Offline Batch   │  │ CI/CD Gates  │  │ Production   │   │  │
│  │  │                 │  │              │  │ Monitoring   │   │  │
│  │  │ - On-demand     │  │ - Pre-deploy │  │ - Streaming  │   │  │
│  │  │ - Scheduled     │  │ - Block if   │  │ - Sampling   │   │  │
│  │  │ - Notebook API  │  │   regression │  │ - Alerts     │   │  │
│  │  └─────────────────┘  └──────────────┘  └──────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           JUDGE POOL                                         │  │
│  │  LLM judges: GPT-4o, Claude Sonnet, Gemini Pro               │  │
│  │  Router: chooses judge based on task, cost, quality          │  │
│  │  Fallback chain if primary judge fails                       │  │
│  │  Cost tracking per eval run                                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           RESULTS STORE + WAREHOUSE                          │  │
│  │  Postgres: recent results, fast queries                      │  │
│  │  Snowflake/BigQuery: historical eval data                    │  │
│  │  Vector DB: eval example embeddings (for similar-case lookup)│  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           DASHBOARDS + ALERTS                                │  │
│  │  Team dashboards: current metrics, regressions               │  │
│  │  Exec dashboard: quality trends, compliance status           │  │
│  │  Alerts: Slack + PagerDuty on threshold breach               │  │
│  │  Reports: weekly quality reports per team                    │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

**1. Centralized registry, decentralized ownership.**
Every dataset and metric has an owner (team). Central platform provides infrastructure; teams provide domain expertise. Prevents "not invented here" while ensuring consistency.

**2. Three execution modes with shared infrastructure:**
- Offline batch: research/one-off evals (notebook-friendly API)
- CI/CD gates: automated pre-deploy quality checks
- Production monitoring: real-time sampling on production traffic

Same metric definitions, same golden datasets, same judge pool — different triggering mechanisms.

**3. Judge pool with routing.**
Don't hardcode "use GPT-4o." Route based on:
- Task type (code eval → different judge than creative writing)
- Cost sensitivity (bulk eval → cheaper model)
- Quality bar (compliance eval → most capable judge)
- Circuit breaker (fallback if primary judge rate-limits)

**4. Metric versioning.**
Faithfulness v1 (basic) vs Faithfulness v2 (with claim decomposition) — teams can pin versions. Prevents silent behavior changes.

**5. Cost governance.**
Every eval run costs $ (LLM judge calls). Track:
- Cost per eval run
- Cost per team per month
- Enforce budgets — team pays for their eval usage
- ROI analysis: which evals are actually preventing regressions?

**6. Data classification.**
Some eval data is sensitive (customer PII, proprietary content). RBAC enforced at dataset level. Anonymization pipeline for shared datasets.

**Interview signal:** Discussing metric versioning + team ownership + judge routing shows real platform-thinking.

---

## Q2. Design a Continuous Evaluation System in CI/CD ⭐⭐⭐⭐

**Prompt:** "Design the eval-in-CI/CD system. Every PR that touches AI code runs evals. Regressions block merge. Cost stays reasonable. Team can iterate fast."

**Design:**

```yaml
# .github/workflows/ai-eval.yml
name: AI Quality Gate

on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'src/ai/**'
      - 'config/models.yaml'

jobs:
  quick-eval:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Fast smoke test
        run: |
          # Runs on 50 canonical examples in <5 min
          # Blocks catastrophic regressions
          python -m eval.smoke_test \
            --dataset eval/canonical_50.json \
            --threshold-drop 0.10 \
            --baseline-run latest_production
      
      - name: Prompt injection defense check
        run: |
          # Runs 20 known injections
          # Must maintain 95%+ defense rate
          python -m eval.security \
            --min-defense-rate 0.95

  full-eval:
    if: contains(github.event.pull_request.labels.*.name, 'run-full-eval')
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - name: Full offline eval
        run: |
          # Runs on 2000 examples across all categories
          # Full RAGAS metrics + custom domain metrics
          python -m eval.full_eval \
            --dataset eval/production_v3.json \
            --metrics faithfulness,answer_relevancy,context_precision,context_recall \
            --output eval-results.json
      
      - name: Compare against baseline
        run: |
          python -m eval.compare \
            --current eval-results.json \
            --baseline latest_production \
            --require-approval-if-drop 0.05
      
      - name: Human review sample
        run: |
          # Post 20 examples to human reviewers via Slack
          python -m eval.human_sample --n 20

  security-eval:
    runs-on: ubuntu-latest
    steps:
      - name: Adversarial red team suite
        run: |
          python -m eval.red_team \
            --tests attacks/comprehensive.yaml \
            --min-defense-rate 0.90

  compliance-eval:
    runs-on: ubuntu-latest
    steps:
      - name: Bias & fairness check
        run: |
          python -m eval.fairness \
            --dimensions gender,race,age \
            --max-disparity 0.20

  gate:
    needs: [quick-eval, full-eval, security-eval, compliance-eval]
    runs-on: ubuntu-latest
    steps:
      - name: Aggregate results
        run: |
          echo "All eval gates passed"
          # Any failed job auto-blocks the PR
```

**Design principles:**

**1. Tiered evals by cost:**
- Quick eval (every PR): 50 examples, <5 min, $0.10
- Full eval (labeled or main branch): 2000 examples, ~40 min, $10-30
- Nightly: 10K examples, full suite, $100-200

**2. Compare against baseline.**
Not absolute threshold (">0.85 faithfulness") but RELATIVE ("no more than 5% drop from current production"). Accounts for baseline variance.

**3. Multiple gates:**
- Quality (faithfulness, relevancy)
- Security (injection defense)
- Bias (fairness across groups)
- Cost (per-query cost budget)

Any gate failing blocks merge.

**4. Human review integration.**
Automated eval catches most regressions but misses subtle qualitative issues. Auto-post 20 examples to reviewers in Slack for eyeball check on every deploy.

**5. Cost caps.**
Enforce per-team eval budget. If a PR eval would exceed budget, require justification. Prevents runaway eval costs from a busy team.

---

## Q3. Design a Production Monitoring System for LLM Applications ⭐⭐⭐⭐

**Prompt:** "Design monitoring for a live customer support LLM. 500K requests/day. Detect quality issues before users complain. Balance signal vs noise."

**Architecture:**

```
User Request
    │
    ▼
LLM Application
    │
    ├───────► Response to User
    │
    └───────► Async logging:
              - Request ID
              - Input (hashed for privacy)
              - Retrieved context
              - Generated response
              - Metadata (user_id, session, model version)
                        │
                        ▼
              ┌──── Kafka / Event Bus ────┐
              │                            │
              ├──► Sampling Router:        │
              │    - 100% events (metadata)│
              │    - 5% sampled (full)     │
              │    - 100% flagged          │
              │                            │
              └────────────────────────────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ Real-time    │ │ Async Eval   │ │ Warehouse    │
    │ Metrics      │ │ Pipeline     │ │              │
    │              │ │              │ │              │
    │ - Latency    │ │ - LLM judge  │ │ - Historical │
    │ - Error rate │ │ - NLI-based  │ │   analysis   │
    │ - Cost/req   │ │   hallucin.  │ │ - Drift      │
    │ - Volume     │ │   detection  │ │ - Cohort     │
    │              │ │ - Toxicity   │ │   analysis   │
    │ Prometheus + │ │              │ │              │
    │ Grafana      │ │ 5% sample    │ │ Snowflake +  │
    │              │ │              │ │ dashboards   │
    └──────────────┘ └──────────────┘ └──────────────┘
              │              │              │
              ▼              ▼              ▼
    ┌─────────────────────────────────────────────┐
    │             ALERTING LAYER                   │
    │  - Threshold-based (error rate > 5%)         │
    │  - Anomaly-based (traffic spike, cost spike) │
    │  - Quality-based (hallucination rate > 10%)  │
    │  - Compliance (bias metric out of range)     │
    │                                              │
    │  Routes to: Slack (info), PagerDuty (crit)   │
    └─────────────────────────────────────────────┘
```

**Sampling strategy (critical for cost):**

- **100% metadata:** Every request logs latency, cost, tokens (cheap)
- **5% full eval:** Random sample gets LLM judge scoring
- **100% flagged:** Auto-flag suspicious cases (long response, refusal, low confidence), evaluate all of these
- **User-reported issues:** Always evaluate + human review

**Alert thresholds (dial in over time):**

| Metric | Warning | Critical |
|---|---|---|
| P95 latency | > 2s | > 5s |
| Error rate | > 2% | > 5% |
| Hallucination rate (sampled) | > 8% | > 15% |
| Refusal rate | > 5% | > 15% (user experience issue) |
| Cost per query | > baseline + 20% | > baseline + 50% |
| Cross-tenant leak signal | > 0 | > 0 (always critical) |

**Dashboards (three levels):**

**1. Operational dashboard (SREs, on-call):**
- Real-time latency, errors, throughput
- Active alerts
- Recent incidents

**2. Product dashboard (PM, eng leads):**
- Quality metrics over time
- User feedback distribution
- Cost trends
- Feature usage

**3. Executive dashboard (leadership, compliance):**
- Weekly quality trend
- Compliance status (bias, fairness metrics)
- Incident count and severity
- ROI (cost per successful interaction)

**Interview signal:** Sampling strategy (not 100% eval) shows cost awareness.

---

## Q4. Design an EU AI Act Compliance System for a High-Risk AI Application ⭐⭐⭐⭐

**Prompt:** "You're building AI for hiring (classified 'high-risk' under EU AI Act). Design the compliance architecture."

**Compliance requirements per EU AI Act Article 9-15:**

**1. Risk management system**
Continuous identification, assessment, mitigation of risks.

**2. Data governance**
Training/eval data must be relevant, representative, complete, and free of bias.

**3. Technical documentation**
Comprehensive docs for regulators (design, testing, monitoring, updates).

**4. Record-keeping**
Automatic logging of all operations for traceability.

**5. Transparency**
Users know when AI is involved. Explanations available.

**6. Human oversight**
Meaningful human review, not rubber-stamping.

**7. Accuracy, robustness, cybersecurity**
Documented performance levels. Adversarial robustness.

**System architecture:**

```
┌──────────── HIRING AI PLATFORM ───────────────────────────────┐
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  RISK MANAGEMENT MODULE                                  │  │
│  │  - Continuous risk assessment                            │  │
│  │  - Incident log                                          │  │
│  │  - Mitigation tracking                                   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  DATA GOVERNANCE                                          │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │  │
│  │  │ Training data│  │ Data card    │  │ Bias audit    │   │  │
│  │  │ - Lineage    │  │ - Provenance │  │ - Demographic │   │  │
│  │  │ - Consent    │  │ - Statistics │  │   parity      │   │  │
│  │  │ - Retention  │  │ - Limitations│  │ - Equal opp.  │   │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  AI SYSTEM                                                │  │
│  │  Candidate → LLM Assessment → Human Reviewer → Decision   │  │
│  │                              ↑ MANDATORY                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  AUDIT & LOGGING (all decisions retained 10 years)       │  │
│  │  - Input to AI                                           │  │
│  │  - AI recommendation + confidence                        │  │
│  │  - Human reviewer decision                               │  │
│  │  - Discrepancies between AI and human                    │  │
│  │  - Timestamp, model version, prompt version              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  TRANSPARENCY LAYER                                       │  │
│  │  - Candidate notified AI is used                         │  │
│  │  - Right to explanation (per candidate request)          │  │
│  │  - Right to human review (mandatory anyway)              │  │
│  │  - Right to contest                                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  CONTINUOUS MONITORING                                    │  │
│  │  - Fairness metrics tracked per month                    │  │
│  │  - Model card updated quarterly                          │  │
│  │  - External audit annually                               │  │
│  │  - Incident reporting to EU database                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

**Human oversight design (critical):**

The human review CAN'T be rubber-stamping. Design forces meaningful engagement:
- Reviewer sees candidate profile WITHOUT AI recommendation initially
- Reviewer forms own opinion, records rationale
- AI recommendation revealed with confidence
- If human disagrees with AI, must justify
- All decisions and disagreements logged for audit

**Compliance evidence trail:**

Every regulatory question must be answerable from logs:
- "Show all decisions for candidates in protected class X" → filter query
- "What was the model's accuracy on candidates aged 50+ last quarter?" → cohort analysis
- "Prove human oversight was meaningful" → discrepancy rate + rationale samples
- "How did you address bias finding from last audit?" → mitigation tracker

**Regulatory reporting:**

Automatic incident detection:
- Significant fairness metric change
- Human-AI disagreement rate spike
- Complaint from candidate about AI decision

Auto-reports to EU database within timelines required.

---

## Q5. Design a Red Teaming Pipeline for LLM Applications ⭐⭐⭐⭐

**Prompt:** "Design an automated red teaming system. Continuously discovers new vulnerabilities. Integrated with development lifecycle."

**System:**

```
┌──────────── RED TEAM PIPELINE ────────────────────────────────┐
│                                                                 │
│  ┌────────── ATTACK LIBRARY ──────────┐                       │
│  │  Known attack patterns (versioned)  │                       │
│  │  - Prompt injections (curated)      │                       │
│  │  - Jailbreaks (from research)       │                       │
│  │  - Encoded attacks                  │                       │
│  │  - Domain-specific attacks          │                       │
│  │  - Community-sourced (PyRIT, Garak) │                       │
│  └─────────────────────────────────────┘                       │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  ATTACK GENERATOR (adversarial LLM)                      │  │
│  │  Given: target system prompt + example attacks           │  │
│  │  Generates: novel attack variants                        │  │
│  │  Uses gradient-based optimization (GCG) for hard targets │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  TARGET SYSTEM (production or staging)                   │  │
│  │  Runs the actual application under test                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  ATTACK EVALUATOR                                         │  │
│  │  For each attack response:                                │  │
│  │  - Did the attack succeed? (LLM judge)                    │  │
│  │  - What defense fired? (guardrail logs)                   │  │
│  │  - Severity of leak?                                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  ┌───────────────────────┼─────────────────────────────────┐  │
│  │  RESULTS + PROMOTION                                      │  │
│  │  - New successful attacks → added to test suite          │  │
│  │  - CVE-style ID assigned                                 │  │
│  │  - Reported to security team                             │  │
│  │  - Fix required before next release                      │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

**Automation cadence:**

- **Every PR:** Baseline red team suite (200 known attacks, ~5 min)
- **Nightly:** Full suite (2000 attacks) + generated variants (500)
- **Weekly:** Novel attack generation session (LLM generates new attacks based on latest research)
- **Quarterly:** External red team engagement (contract firm)

**Attack categories to cover:**

1. Direct injection (100+ variants)
2. Indirect injection via retrieved content
3. Jailbreaks (roleplay, hypothetical, encoded)
4. Data exfiltration (system prompt, other users)
5. PII extraction attempts
6. Denial of service (long inputs, expensive operations)
7. Prompt leaking (revealing internal instructions)
8. Multi-turn escalation
9. Tool abuse (in agent systems)
10. Compliance violations (getting the model to violate safety training)

**Metrics tracked:**

- Attack success rate per category
- Time-to-detect newly successful attacks
- Time-to-patch after discovery
- Regression: previously-defended attacks that now succeed
- Novel attack discovery rate

---

## Q6-Q15: Additional System Design Questions (Condensed)

### Q6. Design an A/B Testing Platform for LLM Applications ⭐⭐⭐⭐
Support variants: prompt changes, model swaps, RAG param changes. Sticky user assignment (same variant per session). Track: quality (LLM judge), user feedback, latency, cost. Statistical significance testing. Auto-rollback on regression. Support multi-armed bandit for adaptive allocation.

### Q7. Design a Trust Dashboard for Executive Reporting ⭐⭐⭐
Weekly rollup for C-suite. Include: quality trend (single number), incident count (severity-weighted), compliance status (green/yellow/red per regulation), user trust metrics (complaint rate, retention), business ROI (cost per successful interaction). Avoid noise — 5 numbers, not 50.

### Q8. Design a Golden Dataset Governance System ⭐⭐⭐⭐
Version-controlled datasets. Ownership per dataset. Change approval workflow (2-reviewer requirement). Access control (some datasets are sensitive). Automatic detection of dataset drift (are we still testing what matters?). Anonymization pipeline for shared datasets.

### Q9. Design a Multi-Tenant Evaluation Infrastructure ⭐⭐⭐⭐
Each tenant (customer) has their own eval config, datasets, metric thresholds. Shared underlying compute. Isolation: tenant A's eval never sees tenant B's data. Cost attribution per tenant. Per-tenant SLAs on eval speed.

### Q10. Design a Cost-Optimized Eval System ⭐⭐⭐
Eval costs can rival production costs. Optimize: sampling instead of 100% eval, cheaper models for coarse eval + expensive for finalization, cache eval results for unchanged prompts, batch eval overnight (batch API discounts), reserved eval capacity from providers.

### Q11. Design a Human Evaluation Workforce Management System ⭐⭐⭐
Scale human eval to 1000s of examples/day. Recruit annotators. Training. Task assignment. Quality control (inter-annotator agreement tracking). Payment. Feedback loops (annotators disagree → rubric refinement). Integration with eval platform.

### Q12. Design a Model Card and Datasheet Generator ⭐⭐⭐
Automated: pulls from eval results, training data metadata, model configs. Generates: model card (Mitchell et al. format), datasheet (Gebru et al. format). Fields: intended use, out-of-scope uses, performance across groups, ethical considerations, environmental impact. Version-controlled.

### Q13. Design a Cross-Team Evaluation Standard ⭐⭐⭐
15 teams, 15 ways of evaluating. Design: common metric definitions, shared judge configurations, unified reporting format, cross-team quality dashboard. Migration path: teams gradually adopt standards, backward compatibility maintained.

### Q14. Design a Real-Time Feedback Loop System ⭐⭐⭐
User provides feedback (thumbs up/down, corrections). System: aggregates feedback, identifies patterns (which query types get most downvotes), routes patterns to improvement queue (dataset addition, prompt fix, retraining), closes loop (users see improvements).

### Q15. Design an Audit-Ready Evaluation System for Regulatory Compliance ⭐⭐⭐⭐
Immutable audit log of all eval runs. Cryptographic tamper evidence. Timestamped, signed. Retained per regulation (7 years financial, 10+ healthcare). External auditor read-only access. Report generator for common compliance requests. Cross-references: eval results ↔ deployed models ↔ user impact.
