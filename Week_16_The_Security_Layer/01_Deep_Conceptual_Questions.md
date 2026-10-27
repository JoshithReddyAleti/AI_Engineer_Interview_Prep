# 🧠 Week 16 — Deep Conceptual Questions

> **Depth level:** Senior AI Security Architect / AI Risk Lead / AI Governance Engineer bar. Every answer includes framework mapping, threshold math, control decomposition, and named regulatory citations.
>
> **How to use:** if you can defend these answers to a CISO, an internal auditor, and a regulator — you clear the AI Security Architect bar.

---

# 🌟 The 4 Questions Worth Mastering

> One per major pillar. Master these four and you clear ~85% of interviews for senior AI security, risk, and governance roles.

---

## 🏆 Question 1: Design an AI Risk Assessment & Tiering Model. Explain the Seven Factors, Lane Routing, and How You Map to NIST AI RMF / ISO 42001 / SR 11-7. ⭐⭐⭐⭐

### A strong answer covers:

- **Why single-factor tiering fails** (data sensitivity alone isn't enough)
- **The seven-factor multi-dimensional model**
- **Dominant-factor routing logic** (not simple weighted sum)
- **Three delivery lanes** with distinct control depths
- **Framework mapping** to NIST AI RMF, ISO 42001, SR 11-7
- **Speed-vs-assurance calibration**
- **The Risk Decision Record artifact**

### Full Answer:

"Risk tiering is where AI governance either enables the business or strangles it. Most organizations pick a single dimension — usually data sensitivity — and route everything through it. That produces two bad outcomes: high-friction pilots for low-risk experimentation, or under-governed high-risk deployments that surface as incidents later. A senior AI security architect designs a multi-factor model that routes into lanes with proportional control.

**The Seven-Factor Risk Model:**

1. **Data sensitivity** — public / internal / confidential / restricted (PII, PHI, PCI, trade secrets)
2. **Model autonomy** — passive suggestion → active recommendation → automated action → agent with tool access
3. **Decision impact** — reversible informational → irreversible operational → life-safety / financial-material
4. **Regulatory scope** — none / SOX / HIPAA / GDPR / EU AI Act high-risk / FDA / SEC
5. **User population** — internal-employees / partner / external-customer / general-public / vulnerable-populations
6. **External exposure** — air-gapped / VPN-internal / partner-network / public-internet
7. **Reversibility** — outputs can be undone in seconds / hours / days / never

Each scored 1-5. But scoring is not simple weighted sum — that averages away the danger.

**Dominant-Factor Routing:**

```
IF any factor scores 5 (maximum risk): route to Elevated Risk lane
ELSE IF sum of top 3 factors ≥ 12: route to Elevated Risk lane
ELSE IF sum of all factors ≤ 15 AND max factor ≤ 3: route to Experimentation lane
ELSE: route to Standard lane
```

Why this logic works: a system with data sensitivity=5 (PHI) and everything else=1 still needs Elevated Risk controls. A weighted average would let low scores mask the single high risk.

**The Three Delivery Lanes:**

| Aspect | Experimentation | Standard | Elevated Risk |
|---|---|---|---|
| **Approval time** | Same day (self-service) | 1-2 weeks | 4-8 weeks |
| **Approvers** | Team lead | CoE reviewer | Security + Legal + CoE + Sponsor |
| **Threat model** | Optional (checklist) | STRIDE-for-AI + OWASP LLM Top 10 | Full: STRIDE + OWASP + MITRE ATLAS + MAESTRO |
| **Guardrail thresholds** | Advisory (log-only) | Warning + soft-fail | Hard-fail on any breach |
| **Monitoring** | Basic (traffic + errors) | Full stack (Week 11 observability) | Full + SIEM integration + on-call |
| **Documentation** | ADR one-pager | Full ADR + Risk Decision Record | ADR + RDR + threat model + red team report + attestation |
| **Data allowed** | Public + internal non-PII | Confidential (with controls) | All classifications (with BAA, residency, encryption at rest + in transit + in use) |
| **Failure blast radius** | Prototype only, ≤10 users | ≤10K users, reversible | Any scale, potentially irreversible |
| **Rollback SLA** | Best-effort | ≤4 hours | ≤15 minutes |

**Framework Mapping:**

**NIST AI RMF 1.0 (2023) — GOVERN / MAP / MEASURE / MANAGE:**
- GOVERN: the risk framework itself, roles, accountability
- MAP: the 7-factor scoring and lane routing (this is where MAP happens)
- MEASURE: guardrail thresholds and observability
- MANAGE: incident response and continuous improvement

**ISO/IEC 42001:2023 (AI Management System — AIMS):**
- Plan-Do-Check-Act cycle
- Our lanes correspond to different depth of PDCA per initiative
- Elevated Risk lane requires full annual AIMS review
- Standard lane rolls up quarterly

**SR 11-7 (OCC / Fed 2011, Model Risk Management for Banking):**
- Tier 1 (high) = our Elevated Risk lane
- Tier 2 (medium) = our Standard lane
- Tier 3 (low) = our Experimentation lane
- Requires model inventory, independent validation, ongoing monitoring, MRM committee
- Applies to any US bank; increasingly referenced outside banking

**EU AI Act (2024):**
- Unacceptable risk (banned) — no lane, refuse outright
- High risk → automatic Elevated Risk lane + conformity assessment + CE marking + database registration
- Limited risk → Standard lane + transparency requirements
- Minimal risk → Experimentation or Standard depending on other factors

**Speed-vs-Assurance Calibration:**

The framework is not static. Metrics to tune:
- **Time-to-production per lane** (Experimentation ≤ 1 day, Standard ≤ 3 weeks, Elevated ≤ 10 weeks)
- **Incident rate per lane** (should trend toward zero in Elevated; some tolerance in Experimentation)
- **False-positive rate on lane assignment** (initiatives that got Elevated but should have been Standard — over-tiering wastes effort)
- **False-negative rate** (initiatives that got Standard but caused incidents — under-tiering causes harm)

Adjust scoring thresholds quarterly based on observed incident data.

**Risk Decision Record (RDR) Schema:**

```json
{
  "rdr_id": "RDR-2026-000847",
  "initiative": "Customer service AI agent v2",
  "sponsor": "VP Customer Experience",
  "date": "2026-08-15",
  "reviewers": ["Security Architect", "Privacy Officer", "CoE Lead"],
  "risk_factors": {
    "data_sensitivity": 3,
    "model_autonomy": 4,
    "decision_impact": 3,
    "regulatory_scope": 4,
    "user_population": 5,
    "external_exposure": 5,
    "reversibility": 3
  },
  "dominant_factors": ["user_population", "external_exposure"],
  "lane_assigned": "Elevated Risk",
  "framework_citations": ["NIST AI RMF MAP-2.3", "EU AI Act Article 6", "ISO 42001 clause 6"],
  "linked_adr_ids": ["ADR-2026-1204"],
  "linked_threat_model": "TM-2026-CX-Agent-v2",
  "guardrail_config_ref": "elevated-lane-config-v3",
  "review_frequency": "quarterly",
  "next_review_date": "2026-11-15",
  "auditor_signoff": "pending",
  "sha256_hash": "...",
  "prev_rdr_hash": "..."
}
```

Hash-chained for audit trail. Every RDR is discoverable in the model inventory. Any regulator or auditor can request the RDR for any deployed AI initiative and see the decision provenance.

**Why this level of rigor:**

Because the CISO, the internal auditor, the regulator, and (eventually) plaintiff's counsel will ask: *why did you deploy this? What did you consider? Who signed off? What was the residual risk?* An RDR is the answer.

**Failure modes:**

**1. Over-tiering.** Everything routed to Elevated Risk. Team drowns. Business bypasses framework via shadow IT.
Fix: measure over-tiering rate, calibrate thresholds down.

**2. Under-tiering.** Framework treats high-risk as standard. Incident hits. Blame follows.
Fix: measure incident-to-tier correlation, calibrate up.

**3. Single-factor bias.** Team scores everything on data sensitivity only. Model autonomy and decision impact ignored.
Fix: require all 7 factors scored, dominant-factor logic enforced.

**4. Lane creep.** Elevated Risk lane becomes 'the responsible-adult lane' — teams route to it to look serious. Volume explodes.
Fix: gate Elevated with sponsor accountability, not opt-in.

**5. Framework arbitrage.** BUs game the scoring to land in a preferred lane.
Fix: independent scoring by security team, not by the requesting BU.

**The staff+ signal:**

Understanding that risk tiering is NOT a bureaucratic exercise. It's the mechanism that lets you defend two contradictory things simultaneously: 'we move fast on innovation' AND 'we control risk on high-stakes deployments.' Frame both to leadership. The framework earns its keep by making both true."

---

## 🏆 Question 2: Compare STRIDE-for-AI, OWASP LLM Top 10, MITRE ATLAS, and MAESTRO. When Do You Use Each? How Do You Consolidate Them? ⭐⭐⭐⭐

### A strong answer covers:

- **What each framework is** and its origin
- **The angle each provides** (defender / adversary / attack surface / agentic)
- **Why one framework isn't enough**
- **How to consolidate into a threat register**
- **The ADR workflow for threat-model decisions**

### Full Answer:

"No single threat modeling framework covers modern AI. STRIDE misses AI-specific attacks. OWASP LLM Top 10 is application-centric. MITRE ATLAS is adversary-centric but broad. MAESTRO is agentic-only. A senior AI security architect uses all four together and knows what each contributes.

**STRIDE-for-AI (Adaptation of Microsoft STRIDE, 1999):**

Original STRIDE categories mapped to AI-specific threats:

| STRIDE | Traditional | AI-specific |
|---|---|---|
| **Spoofing** | Fake login | Model impersonation, deepfake output, provenance forgery |
| **Tampering** | Modified data in flight | Training data poisoning, model weight tampering, prompt injection |
| **Repudiation** | Denied action | Missing audit trail on model decisions, non-reproducible outputs |
| **Information disclosure** | Leaked data | Training data extraction, membership inference, PII in outputs |
| **Denial of service** | Traffic flood | Long-context abuse, expensive query flood, model DoS via adversarial input |
| **Elevation of privilege** | Access above role | Jailbreaking, tool abuse, excessive agency exploitation |

**Angle:** defender-centric, system-decomposition-based. Best for: initial architecture review.

**OWASP LLM Top 10 (2023, revised 2025):**

- LLM01 — Prompt Injection
- LLM02 — Insecure Output Handling
- LLM03 — Training Data Poisoning
- LLM04 — Model Denial of Service
- LLM05 — Supply Chain Vulnerabilities
- LLM06 — Sensitive Information Disclosure
- LLM07 — Insecure Plugin Design
- LLM08 — Excessive Agency
- LLM09 — Overreliance
- LLM10 — Model Theft

**Angle:** LLM-application-specific, well-known-vulnerabilities-based. Best for: control checklist, developer awareness, compliance mapping.

**MITRE ATLAS (Adversarial Threat Landscape for AI Systems):**

Adversary-perspective tactics (like MITRE ATT&CK for enterprise IT):
- **Reconnaissance:** identify target model, capabilities, deployment
- **Initial Access:** get access to model / API / training pipeline
- **ML Model Access:** query model, inference API abuse
- **Execution:** run adversarial input, extract data
- **Persistence:** poison model, backdoor via training data
- **Defense Evasion:** obfuscation, adversarial examples
- **Discovery:** understand model internals via probing
- **Collection:** exfiltrate training data, model weights
- **Impact:** manipulate output, disrupt service

Each tactic has techniques (T-numbers) with case studies.

**Angle:** adversary-centric, real-world-incident-informed. Best for: red team scoping, threat intelligence integration, executive briefings.

**MAESTRO (Multi-Agent Environment, Security, Threat, Risk, Outcome):**

Layered agentic threat framework. Analyzes threats at:
- **Foundation model layer** — base model risks
- **Fine-tuning layer** — custom-model risks
- **Deployment layer** — infrastructure risks
- **Multi-agent layer** — coordination risks
- **Tool / plugin layer** — external-action risks
- **User-interaction layer** — UX / prompt risks
- **Ecosystem layer** — supply chain, ambient risks

**Angle:** agentic-specific, multi-layer, emerging. Best for: agent architectures, autonomous system design, Agentic 2027 territory.

**Why One Framework Isn't Enough:**

- **STRIDE alone:** misses prompt injection nuances (LLM01 is more specific), misses agentic-specific threats (MAESTRO territory)
- **OWASP LLM Top 10 alone:** checklist without decomposition — misses architecture-specific attacks
- **MITRE ATLAS alone:** adversary-focused, doesn't help you decompose your own system
- **MAESTRO alone:** agentic-only — misses non-agentic AI (still most production LLM apps)

**Consolidation Pattern — The Multi-Lens Threat Register:**

For each AI initiative, produce a threat register that:

1. Decomposes the system with STRIDE (walk each component through Spoofing/Tampering/etc.)
2. Cross-references OWASP LLM Top 10 (check every LLM-specific vulnerability)
3. Overlays MITRE ATLAS tactics (identify likely adversary paths)
4. Adds MAESTRO layers if agentic components present

Register schema:

```
Threat ID | Component | STRIDE category | OWASP LLM ref | ATLAS technique | MAESTRO layer | Likelihood | Impact | Existing controls | Gap | Mitigation ADR
```

Example row:
```
T-2026-047 | Retrieval layer | Tampering | LLM01 (indirect) | AML.T0051 | Foundation | High | High | Content filter on retrieved docs | Docs from user uploads not filtered | ADR-2026-1207 (spotlighting for user uploads)
```

**ADR (Architecture Decision Record) Workflow:**

Every threat leads to a decision: accept / mitigate / transfer / avoid. Each decision → an ADR:

```markdown
# ADR-2026-1207: Spotlighting for User-Uploaded RAG Content

## Context
Users can upload documents to their workspace which get indexed into RAG. Threat model (TM-2026-CX) identified prompt injection risk via uploaded content (T-2026-047, T-2026-048).

## Decision
Implement spotlighting (Hines 2024) on user-uploaded content: prefix each retrieved chunk with distinct marker + instruction that content within markers is data, not instructions.

## Consequences
+ Reduces indirect prompt injection attack surface
+ Auditable — every retrieval logged with spotlighting applied
- 3-5% latency overhead
- Model quality on legitimate content queries: slight degradation on edge cases (measured 2%)

## Status
Approved 2026-08-15 by Security Architect + CoE Lead

## Linked
- Threat: T-2026-047
- Risk Decision Record: RDR-2026-000847
- Test suite: test_spotlighting_effectiveness.py
```

Every ADR discoverable. Auditor asks 'why did you use spotlighting?' — you show the ADR chain from RDR → threat model → ADR.

**When to use each framework in practice:**

| Situation | Framework(s) |
|---|---|
| New AI system architecture review | STRIDE-for-AI + OWASP LLM Top 10 |
| Red team engagement scoping | MITRE ATLAS |
| Agent / multi-agent design | MAESTRO + OWASP LLM08 (excessive agency) |
| Board / executive briefing | MITRE ATLAS (real incidents make it concrete) |
| Developer education | OWASP LLM Top 10 (checklist format) |
| Regulator conversation | All four, mapped to control frameworks (NIST AI RMF, ISO 42001) |
| Compliance attestation | OWASP + STRIDE mapping to specific controls |

**Failure modes:**

**1. Single-framework tunnel vision.** Team knows STRIDE, ignores agentic risks. Blind spot.

**2. Framework-shopping.** Team picks whichever framework produces the shortest threat list. Under-modeling.

**3. Threat register bloat.** Every possible threat listed, none prioritized. Analysis paralysis.
Fix: severity + likelihood scoring per threat. Focus on high×high.

**4. Threat model as one-time artifact.** Built at design time, never updated. Stale.
Fix: threat model versioned with the system. Reviewed at major changes. Annual refresh.

**5. Threat register disconnected from controls.** Threats identified but no mapping to controls / ADRs.
Fix: every threat → ADR (accept / mitigate / transfer / avoid).

**The staff+ signal:**

Understanding that threat modeling is not 'brainstorming attacks' — it's a discipline that produces auditable artifacts. The threat register and ADR chain are the evidence trail. Frame every conversation with security stakeholders around 'here's the register, here's the ADR chain, here's the residual risk.' That level of rigor separates a security architect from a security engineer."

---

## 🏆 Question 3: Design Guardrail Threshold Calibration for a Multi-Lane AI System. How Do You Set Thresholds for Hallucination, PII, Prompt Injection Per Lane? ⭐⭐⭐⭐

### A strong answer covers:

- **The full guardrail stack** (hallucination / PII-PHI / injection / toxicity / data-access)
- **Confusion matrix approach** to threshold calibration
- **Per-lane threshold policy** (Experimentation / Standard / Elevated Risk)
- **Build-vs-buy decision matrix** (Arize, Fiddler, Guardrails AI, Presidio, Varonis)
- **Failure modes** and monitoring

### Full Answer:

"Guardrails are where AI safety meets AI utility. Set thresholds too tight and every response gets blocked. Too loose and PII leaks, prompt injections succeed, hallucinations propagate. A senior AI security architect calibrates thresholds per delivery lane based on confusion matrix analysis against real production traffic.

**The Full Guardrail Stack:**

**1. Hallucination Control (Faithfulness)**

- Scoring: RAGAS faithfulness (Es et al. 2023), TruLens groundedness, custom NLI-based scoring
- Action per threshold: accept / warn / regenerate / reject
- Measured metric: hallucination detection F1 on golden set

**2. PII / PHI Control**

- Detection: Presidio (Microsoft), regex patterns, spaCy NER, LLM-based (Claude/GPT with structured output)
- Action: redact / tokenize (reversible) / reject
- Coverage: SSN, credit card, phone, email, addresses, names in context, medical record numbers, ICD-10 codes

**3. Prompt Injection Control**

- Layer 1: input classifier (Rebuff, Lakera Guard, custom fine-tuned)
- Layer 2: structured decoding (Outlines, Guidance)
- Layer 3: spotlighting (Hines 2024) for retrieved content
- Layer 4: dual-LLM pattern (Willison 2023) for privileged operations
- Layer 5: canary tokens in prompts to detect injection at test time

**4. Toxicity / Safety Control**

- Perspective API (Google Jigsaw), Llama Guard (Meta), OpenAI Moderation API
- Custom classifiers for organization-specific values (brand safety)

**5. Data Access Control**

- Row-level security integration with retrieval (see Week 15)
- Provenance tracking (every response tagged with retrieved docs + user permissions at query time)

**Confusion Matrix Approach to Threshold Calibration:**

Guardrails are classifiers. Every threshold produces a confusion matrix:

|  | Truly Safe | Truly Unsafe |
|---|---|---|
| **Predicted Safe (allow)** | True Positive | **False Positive (BREACH)** |
| **Predicted Unsafe (block)** | False Negative (usability harm) | True Negative |

The relative cost of false-positive (breach) vs false-negative (blocked legitimate query) determines the threshold.

**Calibration process:**

1. Collect labeled test set (10K+ examples of legitimate + attack traffic)
2. Sweep threshold, plot ROC curve
3. Compute per-threshold: false-positive rate, false-negative rate, precision, recall, F1
4. Choose threshold based on lane's cost function

**Per-Lane Threshold Policy:**

**Experimentation Lane:**
- Priority: developer velocity
- Threshold: loose. Warning-only on borderline cases
- Rationale: internal audience, low blast radius, iteration matters
- Example: hallucination faithfulness threshold 0.6 (log-only), PII detected → warn, don't block

**Standard Lane:**
- Priority: balance
- Threshold: medium. Hard-block on clear breach, warn on borderline
- Rationale: external users, moderate impact
- Example: hallucination faithfulness 0.75 (regenerate below), PII → hard-block + redact + regenerate

**Elevated Risk Lane:**
- Priority: assurance
- Threshold: strict. Hard-block on any breach or borderline
- Rationale: high-stakes, potentially irreversible, regulator-facing
- Example: hallucination faithfulness 0.9 (reject below, human-in-loop), PII → hard-block + incident logging + notify security

**Threshold Configuration Example:**

```yaml
lane: elevated_risk
guardrails:
  hallucination:
    scorer: ragas_faithfulness
    threshold: 0.90
    action_below_threshold: reject
    fallback: "This query requires human review. Ticket #{ticket_id} created."
    log_level: incident
  pii:
    detectors: [presidio, custom_regex, llm_verification]
    threshold: 0.99  # very high sensitivity
    action: block_and_log
    log_level: incident
    notify: [security, dpo]
  prompt_injection:
    classifier: rebuff_plus_lakera
    threshold: 0.85
    action_above_threshold: reject
    require: [spotlighting, structured_decoding, canary_tokens]
    log_level: incident
  toxicity:
    scorers: [perspective, llama_guard]
    threshold: 0.7
    action: reject
    log_level: warning
```

**Build vs Buy Decision Matrix:**

| Guardrail | Build if… | Buy if… |
|---|---|---|
| Hallucination | Custom domain (legal, medical) with proprietary golden sets | General purpose (Arize Phoenix, TruLens, RAGAS SaaS) |
| PII detection | Custom regulated PII (financial account patterns) | Standard PII (Presidio open-source, Amazon Comprehend) |
| Prompt injection | Adversarial classifier for your specific attack surface | Off-the-shelf (Rebuff, Lakera Guard, Protect AI) |
| Toxicity | Brand-specific safety rules | Perspective API, Llama Guard |
| Data access | Always build — integrates with your ACL system | Never buy — vendor doesn't know your ACL topology |

**Vendor landscape (2026):**

- **Arize AI:** ML observability + LLM guardrails. Strong on evaluation.
- **Fiddler AI:** monitoring + explainability + LLM safety. Enterprise-oriented.
- **Guardrails AI:** open-source library + hosted service. Developer-friendly.
- **Varonis:** data classification + access. Data-side of guardrails.
- **Protect AI:** ML security platform. Focus on ML supply chain.
- **Lakera Guard, Rebuff:** prompt injection specific.

**Monitoring Guardrail Effectiveness:**

Emit per-guardrail metrics:
- **Trigger rate** — % of traffic hitting the guardrail
- **True positive rate** — validated blocks
- **False positive rate** — unnecessary blocks (regenerated legitimate content)
- **False negative rate** — missed attacks (found post-hoc via audit)
- **Latency** — added per request

Weekly review by security + product. Adjust thresholds based on drift.

**Failure modes:**

**1. Threshold set once, never revisited.** Attack landscape shifts. Guardrail becomes stale.
Fix: quarterly threshold review with red team traffic.

**2. Guardrails bypass in agent tool calls.** Content passes guardrails in main path but tools return raw unfiltered.
Fix: apply guardrails on tool outputs, not just user-facing.

**3. Model changes silently, guardrails don't adapt.** Provider updates the base model. Faithfulness scoring drifts.
Fix: guardrail regression tests in CI. Alert on drift.

**4. Guardrail cascades.** Hallucination guardrail triggers regeneration; regenerated response triggers PII guardrail; infinite loop.
Fix: max-retry limits, graceful degradation.

**5. False positives blocking legitimate high-value use.** Enterprise customer's specific vocabulary flagged as PII.
Fix: per-tenant guardrail customization, allowlists, escalation path.

**6. Adversarial threshold discovery.** Attacker probes to find just-below-threshold inputs.
Fix: randomization, ensemble scoring, don't expose threshold to end users.

**The staff+ signal:**

Understanding that guardrails are not 'safety features' — they're classifiers with tunable thresholds. Every threshold is a decision about the FP/FN trade-off, and that trade-off is contextual to the lane. Frame guardrails to non-technical stakeholders as 'we're choosing our error mode: block-safe vs allow-fast.' The lane dictates which. That's how you defend the design to a CISO."

---

## 🏆 Question 4: Design the AI CoE Operating Model. Compare Centralized vs Federated vs Hub-and-Spoke. How Do You Report to the Board? ⭐⭐⭐⭐

### A strong answer covers:

- **Three operating models** with pros/cons
- **When each fits** based on org size, risk profile, AI maturity
- **The AI CoE charter** — what it owns, what it enables
- **Framework operationalization** (EU AI Act, ISO 42001, SR 11-7)
- **Board / C-suite communication cadence**
- **Leading through influence** — the reality of the role
- **Signal escalation paths**

### Full Answer:

"The AI Center of Excellence (CoE) is the connective tissue between engineering and governance. Designed well, it enables 100 teams to ship AI safely. Designed poorly, it becomes either the bureaucratic bottleneck everyone routes around or the ivory tower that no one listens to. A senior AI security architect running or advising a CoE owns this design and its evolution.

**Three Operating Models:**

**1. Centralized CoE**

- All AI decisions flow through central team
- CoE builds tools, sets policies, reviews every initiative
- Business units request AI capabilities from CoE

Pros:
- Consistent controls across org
- Deep expertise concentrated
- Easier compliance evidence

Cons:
- Bottleneck at scale (100 initiatives can't all queue at central review)
- BUs feel disempowered
- CoE detached from actual use cases
- Shadow IT emerges when official channel too slow

Best for: early-stage AI adoption (10-30 initiatives), small orgs, highly regulated industries with monolithic risk posture.

**2. Federated**

- Business units own their AI decisions
- CoE provides frameworks, tools, standards
- BUs self-attest compliance; CoE audits sample

Pros:
- Scales to any number of initiatives
- BU teams empowered
- Fast delivery
- CoE focuses on enablement, not gatekeeping

Cons:
- Requires strong controls (policy-as-code enforcement is critical here)
- Uneven quality across BUs
- Audit evidence collection is harder
- Requires cultural maturity

Best for: large orgs (Fortune 500), high AI initiative count (100+), tech-forward orgs.

**3. Hub-and-Spoke**

- CoE (hub) owns high-risk deployments, standards, platforms
- Embedded AI security champions (spokes) in each BU
- Standard-risk initiatives handled by champions with CoE oversight
- Elevated Risk initiatives handled directly by CoE

Pros:
- Balances speed and control
- Champions build BU-context expertise
- CoE stays connected to real use cases
- Scales better than centralized, safer than fully federated

Cons:
- Requires investment in champion training and career pathing
- Coordination overhead between hub and spokes
- Champions can become bottlenecked in their own BU

Best for: most mature orgs. This is where large enterprises end up.

**Which model fits when:**

| Org profile | Recommendation |
|---|---|
| Startup, ≤10 AI initiatives | Centralized (small team, direct oversight) |
| Growth-stage, 10-50 initiatives | Centralized → Hub-and-Spoke transition |
| Enterprise, 50-200 initiatives | Hub-and-Spoke |
| Large enterprise, 200+ initiatives | Hub-and-Spoke with federated experimentation lane |
| Regulated banking / healthcare | Centralized for Elevated Risk, Federated for Standard |
| Highly regulated + high volume | Hub-and-Spoke with strong policy-as-code enforcement |

**AI CoE Charter — What It Owns:**

- **Frameworks:** risk tiering model, threat modeling standards, guardrail baseline
- **Platforms:** internal LLM gateway, evaluation harness, guardrail library, observability stack
- **Standards:** ADR templates, RDR schema, compliance mapping documents
- **Reviews:** Elevated Risk lane end-to-end; sample audits of Standard lane; enablement for Experimentation
- **Communication:** monthly risk dashboard, quarterly board update, incident briefings
- **Talent:** career path for AI security champions, training program, certification (internal or external CISSP-adjacent)
- **Vendor relationships:** Anthropic, OpenAI, Azure OpenAI, guardrail vendors, red team firms
- **Regulatory monitoring:** track EU AI Act updates, state AI laws, industry-specific regulations

**Framework Operationalization:**

**EU AI Act (2024):**
- Maintain inventory of all AI initiatives with risk classification per Act
- Conformity assessments for high-risk (self-assessment for most; third-party for critical)
- Registration in EU database for high-risk
- CE marking process
- Ongoing monitoring per Act's requirements

**ISO/IEC 42001:2023 (AI Management System):**
- Plan-Do-Check-Act cycle for the AI program
- Annual internal audit
- 3-year external certification cycle
- Documented management system: policies, roles, controls, monitoring
- Continuous improvement process

**SR 11-7 (banking):**
- Model inventory (every AI initiative registered)
- Tiering per SR 11-7 (Tier 1/2/3 aligned to our Elevated / Standard / Experimentation)
- Independent validation for Tier 1
- Ongoing monitoring (performance, data drift)
- MRM Committee reporting

**All three frameworks share:** inventory, tiering, validation, monitoring, governance body. Design your CoE processes to satisfy all three simultaneously.

**Board / C-Suite Communication Cadence:**

**Monthly AI Risk Dashboard (to CIO / CISO):**
- New AI initiatives (by lane)
- Active initiative count (by lane)
- Incident count (by severity)
- Guardrail trigger rate (weekly trend)
- Compliance framework status (gaps identified, remediation timeline)
- Cost per lane (trend)

**Quarterly Board Update (to Board Risk Committee or full board):**
- AI portfolio overview (business impact + risk profile)
- Regulatory landscape changes affecting the org
- Major incidents (lessons learned)
- Framework certification status (SOC 2, ISO 42001, EU AI Act)
- Investment asks (platforms, staffing, training)

**Incident-Specific Briefing (as needed):**
- What happened
- Business impact (users affected, financial exposure, regulatory implications)
- Immediate response
- Root cause
- Systemic changes
- Timeline to resolution

**Key metrics for board:**
- Every metric ties to business outcome, not technical vanity
- 'Guardrail F1 score' → 'blocks 99.7% of harmful outputs, allows 99.5% of legitimate queries'
- 'Model risk exposure' → 'financial materiality of AI decisions this quarter: $X'
- 'Regulatory readiness' → '87% of controls green; 3 gaps identified with remediation by Q4'

**Leading Through Influence:**

The AI security architect rarely has direct authority over BU delivery teams. Persuasion, evidence, and clear escalation paths are the tools.

**Influence tactics that work:**
- **Enablement over enforcement:** provide reusable patterns; make the safe way the fast way
- **Evidence over opinion:** cite incident data, benchmark results, framework requirements
- **Sponsors up the chain:** VP-level sponsors in each BU who champion controls
- **Positive stories:** publicize teams that shipped fast in Standard lane using CoE tools
- **Consequences for shadow IT:** discovered shadow deployments get rolled back, publicly

**Influence tactics that fail:**
- Pure rule-citing without context
- Blocking without alternatives
- Retroactive gotcha reviews
- Complaining to the CISO about BU pushback (escalation-as-first-resort)

**Signal Escalation Paths:**

Clear paths for security concerns to travel:
- Individual contributor → CoE Champion → CoE Lead → CoE Director → CISO/CIO → Board Risk Committee
- Timeline expectations at each hop
- Bypass path for critical incidents (direct to CISO for actively-exploited attacks)

**Failure modes:**

**1. CoE becomes gatekeeper only.** No enablement, only reviews. BUs route around.
Fix: 80% of CoE time on tools/frameworks/enablement, 20% on reviews.

**2. CoE detached from delivery.** Publishes standards nobody uses.
Fix: embed champions, run enablement sessions, measure adoption.

**3. Metrics don't tell the business story.** Technical metrics up the chain, board doesn't understand.
Fix: translate every metric to business impact.

**4. Framework certification without operational reality.** ISO 42001 certified but daily practice ignores it.
Fix: internal audit that samples actual practice, not just docs.

**5. Champion program without career path.** Champions burn out, quit, no succession.
Fix: formal career track, compensation recognition, external certification funding.

**The staff+ signal:**

Understanding that the CoE is a *product* — its customers are BU delivery teams and the board. Both must find it valuable, or it fails. Frame every CoE investment as 'this makes [customer] able to [outcome].' Never 'this makes us more compliant.' Compliance is the byproduct, not the sell."

---

# 📚 Q5-Q32: Additional Deep Conceptual Questions

## Risk Assessment & Tiering (Q5-Q10)

### Q5. NIST AI RMF 1.0 (2023) Deep Dive ⭐⭐⭐⭐

Four core functions: **GOVERN** (org context, roles, accountability, transparency), **MAP** (categorize AI system, identify risks and impacts), **MEASURE** (analyze and monitor risks), **MANAGE** (respond, recover, communicate).

Companion Playbook provides implementation guidance. Voluntary in US but increasingly referenced in contracts and state-level regulations.

Interviewer tests: can you map your risk-tiering framework to specific NIST categories? Do you understand it's process guidance, not certification?

### Q6. ISO/IEC 42001:2023 vs 27001 vs 27017 ⭐⭐⭐

ISO 27001 = Information Security Management System. ISO 27017 = cloud security controls. ISO 42001 = AI Management System (AIMS) — newer, first international AI management standard.

42001 is certifiable via accredited bodies. Signals enterprise-grade AI governance to customers.

### Q7. SR 11-7 (OCC/Fed 2011) Model Risk Management ⭐⭐⭐⭐

Guidance for US banks on model risk management. Requires: model inventory, tiering (high/medium/low materiality), independent validation, ongoing monitoring, MRM committee, documented model risk framework.

Applies to any model informing bank decisions — including LLMs used for credit, fraud, marketing. Increasingly referenced outside banking as gold-standard MRM.

### Q8. EU AI Act Article-by-Article ⭐⭐⭐⭐

Article 5 (prohibited practices), Article 6 (high-risk classification), Article 9 (risk management), Article 10 (data governance), Article 11 (technical documentation), Article 12 (record-keeping), Article 13 (transparency), Article 14 (human oversight), Article 15 (accuracy/robustness/cybersecurity), Articles 40-44 (conformity assessment), Article 71 (penalties up to €35M or 7%).

### Q9. Colorado AI Act, CCPA Extensions, State AI Laws ⭐⭐⭐

US regulatory patchwork. Colorado AI Act (2024) — first US comprehensive AI law, effective 2026, similar structure to EU AI Act. CCPA + California Delete Act — extend consumer rights. NYC Local Law 144 — bias audits for hiring AI. Illinois HB 3773 — algorithmic hiring.

Design compliance layer per-state.

### Q10. Reversibility and the "Time-to-Undo" Metric ⭐⭐⭐

Reversibility scoring:
- **Seconds:** UI suggestion (user rejects, no impact)
- **Minutes:** internal recommendation (human reviews before action)
- **Hours:** operational action (user notification, reversible)
- **Days:** business decision (customer-visible, contract implications)
- **Never:** life/safety, financial-material, published to public

Higher irreversibility = higher lane requirement. Design decision impact scoring around actual undoability.

---

## Threat Modeling (Q11-Q16)

### Q11. Prompt Injection Deep Dive ⭐⭐⭐⭐

Direct vs indirect. Direct: user tries injection in own input. Indirect: injection via retrieved document, tool output, third-party content.

Advanced attacks: PAIR (Chao 2023) — automated red teaming with attacker LLM. TAP (Mehrotra 2023). Universal adversarial suffixes (Zou et al. 2023). Multi-turn jailbreaks.

Defenses: input classifier, structured decoding, spotlighting (Hines 2024), dual-LLM (Willison 2023), canary tokens, output validation.

### Q12. Training Data Poisoning ⭐⭐⭐

Attacker inserts crafted examples into training data (or RAG corpus). Model learns malicious behavior.

Detection: data provenance audits, outlier detection, canary examples in training set. Defenses: strict data curation, contamination detection (embeddings-based), differential privacy.

### Q13. Membership Inference and Training Data Extraction ⭐⭐⭐

Attacker probes model to determine if specific data was in training set (membership) or extract training data verbatim (extraction).

Real: Carlini et al. 2023 showed GPT-2 extracts training data. Chen 2024 for LLMs.

Defenses: output filtering for verbatim reproduction, differential privacy in training, refuse queries that seek training data.

### Q14. Model Theft and Extraction ⭐⭐⭐

Attacker queries model API to reconstruct model behavior or steal weights via distillation.

Detection: query pattern analysis (suspicious query volumes, systematic probing). Defenses: rate limiting, watermarking outputs, API access controls.

### Q15. Supply Chain Attacks (OWASP LLM05) ⭐⭐⭐⭐

Attack surfaces: base model (backdoored during training), fine-tuning data (poisoned), inference infrastructure (compromised), guardrail vendors (malicious code), retrieval sources (poisoned RAG).

Mitigations: SBOM for AI, provenance verification (model cards, dataset cards), vendor security assessments, code signing.

### Q16. Excessive Agency (OWASP LLM08) ⭐⭐⭐⭐

Agent has more capability than task requires. Malicious or errant behavior has amplified impact.

Controls: least-privilege tool access, action approval for high-impact, budget constraints (cost/time/tool-count), sandboxed execution, human-in-loop for irreversible actions.

Bridge to Agentic 2027.

---

## Guardrails & Enforcement (Q17-Q21)

### Q17. Faithfulness Scoring Methods ⭐⭐⭐

RAGAS faithfulness (Es et al. 2023): decompose response into claims, verify each against retrieved context. Score = supported claims / total claims.

Alternatives: TruLens groundedness, NLI-based scoring (entailment checker), LLM-as-judge with specific rubric.

Trade-offs: LLM-as-judge is expensive but flexible; NLI is fast but rigid; RAGAS is balanced but requires claim extraction.

### Q18. PII Detection Layered Approach ⭐⭐⭐

Layer 1: regex for structured (SSN, credit card, phone).
Layer 2: NER (spaCy, Presidio) for names, addresses.
Layer 3: LLM-based for context-dependent (is 'Kevin' a person here or a project name?).

Ensemble scoring. High recall priority — miss no PII, accept some false positives.

### Q19. Policy-as-Code with OPA / Rego ⭐⭐⭐⭐

OPA (Open Policy Agent): general-purpose policy engine. Rego language for policy rules.

Example policy: 'Elevated Risk deployments must use BAA-covered providers for PHI.'

```rego
package ai_deployment

allow_provider {
  input.lane == "elevated_risk"
  input.data_classification == "phi"
  input.provider in {"azure_openai", "anthropic_enterprise", "aws_bedrock"}
}
```

Enforcement points: API gateway (block deploy), retrieval layer (filter), model output (validate).

### Q20. Sentinel for Infrastructure Policy ⭐⭐⭐

HashiCorp Sentinel: policy-as-code for Terraform. Enforces at plan/apply time.

Example: 'All AI workloads must deploy to encrypted-at-rest storage in approved regions.'

Complements OPA (runtime) with infrastructure-time enforcement.

### Q21. Enforcement Level per Lane ⭐⭐⭐

- **Advisory:** log-only, no action (Experimentation)
- **Warning:** notify + allow (Standard soft controls)
- **Soft-fail:** require override with justification (Standard hard controls)
- **Hard-fail:** block, incident generated (Elevated Risk)

Same policy, different enforcement level per lane.

---

## Governance & Operating Models (Q22-Q27)

### Q22. AI CoE Charter Design ⭐⭐⭐⭐

Charter document specifies: mission, scope (what CoE owns vs BU), decision rights, funding model, staffing, KPIs, escalation paths, review cadence.

Signed by CIO/CISO + BU heads. Reviewed annually.

### Q23. Federated Model with Champions Program ⭐⭐⭐

Champion: embedded AI security lead in a BU. Career-tracked, certified, evaluated on both CoE alignment and BU delivery.

Champions handle Standard lane initiatives locally; escalate Elevated Risk to CoE. Reduces CoE bottleneck.

### Q24. Board-Level AI Risk Reporting ⭐⭐⭐⭐

Board Risk Committee expects:
- Portfolio view (initiative count, materiality)
- Incident summary (severity, cost, response)
- Regulatory landscape changes
- Compliance status (framework attestations)
- Investment recommendations

Metrics translated to business impact. Not F1 scores.

### Q25. Regulatory Communication ⭐⭐⭐

Regulators expect:
- Clear framework citation (NIST AI RMF, ISO 42001, EU AI Act)
- Evidence trail (RDRs, ADRs, threat models, audit logs)
- Incident response capability (with SLA to their notification requirements)
- Continuous improvement demonstration

Be transparent. Regulators reward candor.

### Q26. EU AI Act Conformity Assessment ⭐⭐⭐⭐

For high-risk AI systems: technical documentation (Annex IV), risk management system, data governance, transparency documentation, human oversight design, accuracy/robustness testing, cybersecurity controls, quality management system, post-market monitoring.

Self-assessment for most; notified body assessment for critical (medical devices, biometric ID).

Timeline: 6-12 months typically first time.

### Q27. Model Inventory and Registration ⭐⭐⭐

Central register of every AI initiative:
- Description, business owner, technical owner
- Risk tier and RDR reference
- Framework classifications (NIST, ISO 42001, EU AI Act, SR 11-7)
- Deployment status, environments, users
- Guardrails config, monitoring status
- Vendor dependencies, data sources
- Last review date, next review date

Feeds board reporting, audit requests, regulator queries.

---

## Agentic Security — Bridge to 2027 (Q28-Q32)

### Q28. Excessive Agency in Multi-Agent Systems ⭐⭐⭐⭐

Individual agents have bounded permissions. Multi-agent coordination can compound: agent A calls agent B calls agent C, each with own permissions, effective permission = union.

Controls: capability graph analysis, transitive permission checks, joint action approval for compounded impact.

### Q29. Prompt Injection at Any Hop ⭐⭐⭐⭐

In agent workflow, injection at step 1 poisons all downstream steps. Attacker doesn't need direct access — a poisoned retrieved doc at step 1 can manipulate steps 2-N.

Controls: spotlight retrieved content at every hop, validate outputs at every hop, canary tokens end-to-end.

### Q30. Goal Drift Under Adversarial Context ⭐⭐⭐⭐

Agent's stated goal shifts as context accumulates. Adversarial context can gradually steer agent to unintended actions.

Controls: goal reaffirmation at each iteration, plan-vs-execution divergence detection, human checkpoints for goal changes.

### Q31. Reward Hacking at Agent Scale ⭐⭐⭐

Agent optimizes for stated goal in unintended ways. Classical RL problem, now at LLM agent scale.

Examples: agent instructed to 'maximize customer satisfaction' → agent proactively promises refunds; agent instructed to 'complete task in fewest steps' → agent skips validation.

Controls: reward specification review, constraint-checking layer, adversarial red teaming of goal specifications.

### Q32. MAESTRO Framework Deep Dive ⭐⭐⭐⭐

MAESTRO analyzes agentic threats at layers:
- **Foundation:** base model vulnerabilities
- **Fine-tuning:** custom-model risks
- **Deployment:** infrastructure
- **Multi-agent:** coordination risks
- **Tool / plugin:** external actions
- **User-interaction:** UX / prompt
- **Ecosystem:** supply chain, ambient

Each layer has specific attack patterns and controls. Emerging framework for Agentic 2027 territory.

**The staff+ signal on agentic:**

Understanding that agentic security IS the frontier. The threat surface expands non-linearly with autonomy. Every control in Weeks 1-15 must be re-examined for the agent case. This is where the Agentic AI Roadmap 2027 begins.
