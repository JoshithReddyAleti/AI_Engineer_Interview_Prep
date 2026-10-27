# 🏗️ Week 16 — System Design Questions

> **Focus:** Enterprise AI governance platform, EU AI Act high-risk system, SR 11-7 model risk management for banking, federated AI CoE, agentic security architecture
>
> **Depth:** Senior AI Security Architect / VP AI Compliance bar. Every design includes framework mapping, control decomposition, cost math, and board-level metrics.

---

## Q1. Design an Enterprise AI Governance Platform for 500 AI Initiatives Across 30 Business Units ⭐⭐⭐⭐

**Prompt:** "Fortune 100 company, 30 BUs, 500 AI initiatives active, another 200 in pipeline. Regulated (banking + healthcare divisions). Design the platform that lets CoE govern this at scale without becoming a bottleneck."

### Architecture:

```
┌────────────────────────────────────────────────────────────────┐
│                   BUSINESS UNITS (30 total)                     │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐   │
│  │ Bank │  │Health│  │Retail│  │  HR  │  │Fin CX│  │  ..  │   │
│  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘   │
│      │        │         │         │         │         │       │
│  Embedded AI Security Champions (30 total)                     │
└──────┼────────┼─────────┼─────────┼─────────┼─────────┼───────┘
       │        │         │         │         │         │
       ▼        ▼         ▼         ▼         ▼         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AI CoE PLATFORM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  INITIATIVE INTAKE (self-service portal)                │   │
│  │  → 7-factor scoring → lane routing → RDR generation      │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │                                │
│  ┌────────────────────────────┴────────────────────────────┐   │
│  │  MODEL INVENTORY & REGISTRY                             │   │
│  │  Every initiative registered. Framework-tagged.          │   │
│  │  Queryable for board / audit / regulator.                │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │                                │
│  ┌──────────────┬──────────────┼──────────────┬─────────────┐  │
│  ▼              ▼              ▼              ▼             ▼  │
│ Policy       Guardrail      Threat Model    Red Team     Audit │
│ Engine       Library        Templates       Program      Trail │
│ (OPA)        (Arize/etc)    (STRIDE/OWASP)                     │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  OBSERVABILITY & SIEM INTEGRATION                       │   │
│  │  Per-initiative telemetry. Guardrail triggers. Incidents │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  DASHBOARDS                                             │   │
│  │  Champion / CoE Lead / CIO / CISO / Board Risk Committee │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Design Decisions:

**1. Hub-and-Spoke operating model.**
- CoE (hub): platform, standards, Elevated Risk reviews
- Champions (spokes): embedded in each BU, handle Standard lane
- 30 champions + 15-person CoE = 45 people governing 500 initiatives

**2. Self-service intake with automated routing.**
- BU teams enter 7 factors → engine auto-routes → lane-appropriate workflow triggered
- Standard: champion review + auto-provision guardrail baseline
- Elevated Risk: CoE review + full threat model + red team

**3. Model inventory as single source of truth.**
- Every initiative registered, linked to RDR, ADRs, threat model, code repo
- Framework-tagged: NIST AI RMF categories, ISO 42001 controls, SR 11-7 tier (banking), EU AI Act classification (EU-facing)
- Feeds all downstream: board metrics, audit responses, regulator requests

**4. Guardrail library as shared platform capability.**
- Baseline guardrails available as SDK / gateway
- BU teams don't reimplement PII detection or prompt injection defense
- Threshold policy per lane; automatic based on lane assignment

**5. Policy-as-code enforcement at deploy gates.**
- OPA policies gate CI/CD
- Deployment blocked if RDR missing, ADR missing, guardrails not configured per lane, etc.
- Automated attestation collection for audit

**6. Champion program with career ladder.**
- Champion I → Champion II → Senior Champion → CoE Engineer
- Certifications encouraged (CISSP, CIPP, ISO 42001 Lead Auditor)
- Compensation recognition + external conference budget

### Cost Math:

Team investment:
- CoE 15 people × $200K fully-loaded = $3M/yr
- 30 champions × $180K partial FTE (40%) × $180K = $2.16M/yr
- Platform infra: $500K/yr
- Vendors (Arize, guardrails, red team firm, etc.): $1.5M/yr
- **Total: $7.16M/yr for AI governance**

Return:
- 500 initiatives shipping safely (baseline non-shipped would be $0 revenue)
- Board-defensible AI program (protects vs $35M EU AI Act penalty exposure)
- Enterprise sales unlock (SOC 2 + ISO 42001 = $50M+ in RFPs won)
- Incident cost avoidance ($10M+ per major AI incident based on industry)

**Ratio:** $7M invested → $50M+ protected/enabled = strong ROI

### Metrics for Board:

- Portfolio: 500 active, distribution across lanes (target: 80% Standard, 15% Elevated, 5% Experimentation)
- Time-to-production per lane
- Incident count + severity (trending down)
- Framework attestation status (SOC 2 green, ISO 42001 in progress, EU AI Act 87% ready)
- Cost per initiative governed

---

## Q2. Design an EU AI Act High-Risk AI System — Full Compliance Architecture ⭐⭐⭐⭐

**Prompt:** "AI-powered candidate screening tool serving EU customers. High-risk under EU AI Act (Article 6, Annex III). Design end-to-end compliance."

### Full Compliance Stack:

**1. Risk management system (Article 9)**
- Documented risk assessment process
- 7-factor scoring → Elevated Risk lane automatically for hiring AI
- Ongoing risk monitoring in production

**2. Data governance (Article 10)**
- Training data documentation (source, license, consent basis)
- Bias analysis across protected characteristics (gender, race, age, disability)
- Data quality metrics (representative sample?)
- Removal of illegal / discriminatory data
- Documentation of data preprocessing

**3. Technical documentation (Article 11, Annex IV)**
- System description
- Intended purpose
- Methods & training approach
- Data specifications
- Model performance metrics
- Human oversight design
- Compliance evidence

**4. Record-keeping (Article 12)**
- Automatic logging of every system operation
- Retention 10 years
- Tamper-evident (hash chain)
- Auditable

**5. Transparency (Article 13)**
- Users informed AI is used
- Instructions for use
- Performance characteristics disclosed
- Human oversight measures

**6. Human oversight (Article 14)**
- Human reviews every AI-ranked candidate before rejection
- Human can override AI ranking with documented justification
- Human trained on system's limitations
- Appeals mechanism for candidates

**7. Accuracy, robustness, cybersecurity (Article 15)**
- Accuracy metrics measured continuously
- Robustness testing (adversarial inputs)
- Cybersecurity controls per Week 15 + Week 16 patterns

**8. Conformity assessment (Articles 40-44)**
- Self-assessment for standard high-risk
- Third-party (notified body) for critical categories
- Technical documentation reviewed
- CE marking issued
- Declaration of Conformity

**9. Registration in EU database (Article 71)**
- Public registration required for high-risk systems
- Includes: system purpose, provider, market surveillance authority

**10. Post-market monitoring (Article 61)**
- Continuous monitoring of system performance
- Reporting of serious incidents
- Framework for continuous improvement

### Concrete System Design:

```
Candidate Application
        ↓
[Transparency Notice: "AI-assisted screening in use"]
        ↓
Feature Extraction (documented per Article 10)
        ↓
AI Ranking Model (in-scope Elevated Risk lane)
        ↓
Bias Monitoring (continuous, per-decision)
        ↓
Human Recruiter Review (MANDATORY — Article 14)
        ↓
Decision (with documented rationale)
        ↓
[Candidate notification with appeal mechanism]
        ↓
Audit Log (Article 12 — 10-year retention)
        ↓
Post-Market Monitoring Feedback Loop
```

### Estimated Timeline & Cost:

- Timeline: 6-9 months first time compliance (subsequent: 3 months)
- Cost: $500K-$2M depending on complexity (documentation, testing, external assessor if required, ongoing compliance ops)
- Ongoing: 10-20% of engineering capacity dedicated to compliance maintenance

### Penalty Exposure:

- Non-compliance high-risk: up to €15M or 3% global revenue
- Prohibited AI: up to €35M or 7%
- Providing incorrect information: up to €7.5M or 1%

---

## Q3. Design Model Risk Management for a Bank Under SR 11-7 ⭐⭐⭐⭐

**Prompt:** "US bank deploying LLMs across customer service, fraud detection, credit decisions, marketing. SR 11-7 applies. Design MRM program."

### MRM Framework Under SR 11-7:

**Model Inventory (foundation):**
- Every model registered with unique ID
- Materiality tier: Tier 1 (high — credit decisions), Tier 2 (medium — fraud alerts), Tier 3 (low — internal knowledge base)
- Includes: purpose, methodology, data, validation history, ongoing performance

**Independent Model Validation:**
- Separation of development from validation (independence)
- Tier 1 validation before production (mandatory)
- Tier 2 validation at deployment + annually
- Tier 3 validation at deployment
- Validation covers: conceptual soundness, ongoing monitoring, outcomes analysis

**Ongoing Monitoring:**
- Performance metrics (accuracy, false-positive rate, drift)
- Population stability (input distribution unchanged from training)
- Champion-challenger testing
- Alert on threshold breaches

**MRM Committee:**
- Meets monthly
- Reviews new model requests
- Approves Tier 1 validations
- Escalates issues to Chief Risk Officer

**Documentation:**
- Model documentation per SR 11-7 sections
- Validation reports (archived per regulator retention)
- Committee minutes

**LLM-Specific Considerations:**

Traditional SR 11-7 was written for statistical models. LLM-specific adaptations:
- Base model changes (silent provider updates) require re-validation trigger
- Prompt as model version (change to prompt = new model version, technically)
- Emergent behavior harder to validate — need robust adversarial testing
- Third-party model (OpenAI/Anthropic) documentation depth limited — supplement with own analysis

### Federal Guidance References:

- SR 11-7 (2011) — Guidance on Model Risk Management
- OCC Bulletin 2011-12 — companion for national banks
- Fed / OCC / FDIC joint guidance on AI (evolving)

### Cost Considerations:

- MRM team: 8-15 people ($2-4M/yr)
- External validators (Tier 1 requires independence): $500K-2M/yr
- Documentation and process tools: $500K
- Ongoing monitoring infrastructure: shared with observability stack

**Total: ~$4-8M/yr for MRM in mid-sized bank**

Return: regulatory compliance (avoids consent orders, MRAs, restrictions on business activity)

---

## Q4. Design Agentic AI Security Architecture (Bridge to 2027) ⭐⭐⭐⭐

**Prompt:** "Enterprise deploying autonomous agents that plan, execute, and coordinate. Security design for cascading failure, tool abuse, goal drift."

### Architectural Principles:

**1. Least-privilege by default.**

Every agent gets minimum tool set required. Explicit approval for capability additions.

Tool permission model:
- Read-only vs read-write
- Rate-limited per tool
- Data-scope limited (agent can query own customer's data, not others)
- Auditable per invocation

**2. Budget constraints.**

Every agent run has:
- Cost budget (dollars)
- Time budget (seconds)
- Tool call budget (count)
- Iteration budget (planning cycles)

Approaching limit → graceful degradation → escalate to human if unresolved.

**3. Human-in-the-loop for irreversible actions.**

Refund > $X: approval required
Data deletion: approval required
External communication (email, ticket creation): approval required
Financial transactions: approval required

Reversible actions (internal queries, informational responses) proceed autonomously.

**4. Multi-hop injection defense.**

At every step of agent workflow:
- Validate outputs before they become next-step inputs
- Spotlight all retrieved / tool-returned content
- Canary tokens end-to-end
- Anomaly detection on tool call patterns

**5. Goal drift detection.**

Track: initial goal → intermediate goals → current action
Divergence detection: if current action can't be traced to initial goal, halt + escalate.

**6. Kill switches at multiple levels.**

- Per-agent-run: user or admin can halt in-progress agent
- Per-agent-type: disable all instances of a particular agent
- Per-tenant: disable all agents for a specific customer
- Global: disable all agents (regulatory emergency)

**7. Sandboxed execution for code-generating agents.**

Agents that generate and execute code run in isolated sandboxes:
- Network restrictions (no arbitrary egress)
- Filesystem restrictions (scoped tmpdir)
- Compute limits (CPU, memory, time)
- No access to secrets

**8. Behavior monitoring & anomaly detection.**

- Baseline agent behavior patterns
- Detect anomalies (unusual tool call frequency, unusual data access patterns)
- Alert + optionally halt

**9. Comprehensive audit.**

Every agent decision, tool call, output logged:
- User who invoked
- Agent instance ID
- Goal specification
- All intermediate reasoning + tool calls + results
- Final action taken

Hash-chained. Queryable for post-incident analysis.

### The Six MAESTRO Layers Applied:

- **Foundation:** base LLM security (prompt injection, jailbreak resistance) — inherit from Week 15 patterns
- **Fine-tuning:** custom-model risks — data provenance, adversarial robustness testing
- **Deployment:** infrastructure — network isolation, secrets management
- **Multi-agent:** coordination risks — capability graph, transitive permission enforcement
- **Tool / plugin:** external actions — least-privilege, budget constraints, HITL for irreversible
- **User-interaction:** UX / prompt — clear agent status communication, halt controls
- **Ecosystem:** supply chain — tool provider security, third-party integration audits

### Metrics:

- Agent success rate per task type
- Human intervention rate (per lane target)
- Tool call anomaly rate
- Goal-execution divergence rate
- Incident rate + severity

**This design bridges directly into the Agentic AI Roadmap 2027 opening episodes.**

---

## Q5-Q16: Additional System Design Questions (Condensed)

### Q5. Design a Federated AI Governance Platform for a Multi-Region Enterprise ⭐⭐⭐⭐
30-country deployment. Per-region regulatory variance (EU AI Act, US state laws, APAC frameworks). Design so each region operates locally while central CoE maintains global standards.

### Q6. Design an AI Incident Response Program ⭐⭐⭐⭐
Detection → triage → containment → investigation → remediation → post-mortem. Prompt injection incident. Data leak. Model going rogue. Regulator notification workflow. Playbooks per incident class.

### Q7. Design SOC 2 CC + AI-Specific Controls ⭐⭐⭐⭐
Extend SOC 2 Common Criteria (CC5, CC6, CC7) with AI-specific controls. AI change management, guardrail effectiveness, model inventory. Audit-ready.

### Q8. Design a Red Team Program for LLM Applications ⭐⭐⭐
Regular red team cadence per lane (monthly Elevated, quarterly Standard, annual Experimentation). Attack patterns catalog. Remediation SLAs. Trend tracking.

### Q9. Design a Vendor Security Assessment Process for AI Providers ⭐⭐⭐
Evaluate: SOC 2, data handling, training practices, incident history, contractual terms (no-training, BAA, DPA). Continuous monitoring. Exit plans.

### Q10. Design a Board Risk Dashboard for AI Program ⭐⭐⭐⭐
Portfolio view, incident summary, framework attestation status, investment recommendations. Metrics in business terms. Monthly + quarterly cadence.

### Q11. Design SIEM Integration for LLM Observability ⭐⭐⭐
Guardrail triggers, injection attempts, anomalous access → SIEM (Splunk, Sentinel, Chronicle). Correlation rules. Playbook automation.

### Q12. Design Continuous Compliance Automation ⭐⭐⭐
Automated evidence collection for SOC 2, ISO 42001, EU AI Act. Continuous control testing. Attestation-ready without emergency scramble.

### Q13. Design a Data Loss Prevention (DLP) Layer for AI ⭐⭐⭐
Detect + prevent sensitive data flowing to LLM providers. Classification-aware. Provider allowlists. Egress inspection.

### Q14. Design an AI Bug Bounty Program ⭐⭐⭐
Scoped to prompt injection, jailbreak, data extraction. Reward tiers. Triage process. Legal safe harbor.

### Q15. Design a Global Regulatory Monitoring System ⭐⭐⭐
Track regulatory changes across 30+ jurisdictions. EU AI Act updates, US state laws, APAC frameworks. Impact assessment. Compliance roadmap updates.

### Q16. Design the Bridge from AI Engineering Roadmap 2026 to Agentic AI Roadmap 2027 ⭐⭐⭐⭐
How the security foundation supports the agent frontier. What changes. What must be re-designed. What carries forward. Transition strategy for teams.
