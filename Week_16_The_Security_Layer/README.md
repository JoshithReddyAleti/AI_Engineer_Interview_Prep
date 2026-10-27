# 🛡️ Week 16 (Bonus) — The Security Layer: AI Risk, Governance & Enterprise Defense

> **Maps to:** [Episode_16_The_Security_Layer](https://github.com/JoshithReddyAleti/Episode_16_The_Security_Layer)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)
>
> **This is the bonus capstone.** Week 16 covers the layer that makes AI defensible to CISOs, internal auditors, and regulators — without strangling delivery speed. It bridges the AI Engineering Roadmap 2026 into the Agentic AI Roadmap 2027.

---

## Why This Bonus Week Exists

Weeks 1-15 taught you how to BUILD AI systems. Week 16 teaches you how to ANSWER the question that kills AI programs at scale:

*"Great. Now prove it's safe."*

Not "is it safe" in a hand-wavy way. **Prove it.** To a skeptical internal auditor who wants your risk-tiering criteria. To a regulator who wants your threat models and Architecture Decision Records. To a CISO who wants your guardrail thresholds for prompt injection and PII exposure. To an SVP who wants to know why Initiative A shipped in two weeks and Initiative B took two months — and wants the answer to be **auditable, not vibes**.

This is the interview surface for a growing category of senior roles:
- **AI Security Architect**
- **AI Risk Lead**  
- **AI Governance Engineer**
- **Head of AI Trust & Safety**
- **VP AI Compliance**

Roles that report to SVPs and CISOs. Roles that own the frameworks. Roles where the interview isn't "can you code" — it's "can you defend this program to the board."

**Signal calibration (what interviewers detect):**

- **Junior:** "We use guardrails."
- **Mid:** "We use NeMo Guardrails and Presidio for PII detection."
- **Senior:** "We tier AI initiatives via a 7-factor risk model into three delivery lanes with different control depths."
- **Staff+ / AI Security Architect:** "Our risk-tiering model uses seven factors — data sensitivity, model autonomy, decision impact, regulatory scope, user population, external exposure, reversibility — routed through dominant-factor logic into Standard / Elevated Risk / Experimentation lanes, each with distinct threshold policies, ADR requirements, and monitoring depth. Guardrail thresholds are calibrated per lane using confusion matrix analysis against production traffic. Every risk decision generates an auditable Risk Decision Record referenceable in SOC 2, ISO 42001, and EU AI Act attestations. This lets us move fast on low-risk experimentation while defending high-risk deployments to the board."

Week 16 gets you to the staff+ AI security architect bar across all six pillars.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 32 | 7-factor risk model, lane routing, NIST AI RMF vs ISO 42001 vs SR 11-7, STRIDE-for-AI, OWASP LLM Top 10, MITRE ATLAS, MAESTRO, guardrail thresholds, policy-as-code, AI CoE operating models, agentic security |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 17 | Risk scoring engine, lane router, threat register generator, guardrail calibrator (confusion matrix), OPA policy engine, ADR schema, faithfulness scorer, PII redactor with tokenization, hash-chained audit trail |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | Enterprise AI governance platform, multi-cloud policy-as-code enforcement, EU AI Act high-risk system, model risk management for banking (SR 11-7), federated AI CoE, agentic security architecture |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 17 | CISO demanding proof it's safe, SVP demanding launch bypass, regulator audit finding, agent exfiltrated data via tool call, model went rogue, PII leak in prod, red team broke every guardrail, board wants AI velocity |
| [05_Terminology_Differentiation.md](05_Terminology_Differentiation.md) | Reference | Risk vs threat vs vulnerability, NIST AI RMF vs ISO 42001 vs SR 11-7, STRIDE vs OWASP LLM vs MITRE ATLAS vs MAESTRO, guardrails vs policy vs control, AI CoE vs federated vs hub-and-spoke, defense-in-depth vs zero-trust for AI |

## The Six Pillar Coverage

### Pillar 1 — AI Risk Assessment & Tiering

| Sub-section | Interview Territory |
|---|---|
| Multi-factor risk model | The 7 factors: data sensitivity, model autonomy, decision impact, regulatory scope, user population, external exposure, reversibility |
| Lane routing logic | Dominant-factor combination into Standard / Elevated Risk / Experimentation lanes |
| Risk Decision Records | Auditable schema, versioning, cross-referencing with ADRs |
| Framework mapping | NIST AI RMF (Govern/Map/Measure/Manage), ISO/IEC 42001:2023 (AI management system), SR 11-7 (2011, model risk management for banking), EU AI Act risk tiers |
| Framework calibration | Speed-vs-assurance metrics, threshold tuning, false-positive rate at each lane |

### Pillar 2 — Threat Modeling for AI

**STRIDE-for-AI:** the Microsoft threat model (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege) mapped to AI-specific threats (model theft, prompt injection, membership inference, model DoS via long-context abuse, jailbreaking).

**OWASP LLM Top 10 (2023, revised 2025):** LLM01 Prompt Injection, LLM02 Insecure Output Handling, LLM03 Training Data Poisoning, LLM04 Model DoS, LLM05 Supply Chain Vulnerabilities, LLM06 Sensitive Information Disclosure, LLM07 Insecure Plugin Design, LLM08 Excessive Agency, LLM09 Overreliance, LLM10 Model Theft.

**MITRE ATLAS (Adversarial Threat Landscape for AI Systems):** adversary-perspective tactics and techniques. Reconnaissance → Initial Access → ML Model Access → Execution → Persistence → Defense Evasion → Discovery → Collection → Impact.

**MAESTRO:** layered agentic threat analysis. Specific to agent workflows — tool abuse, planner manipulation, cascading errors.

**Consolidated threat register + ADR workflow.**

### Pillar 3 — Guardrails & Control Thresholds

**Hallucination control:** faithfulness scoring (RAGAS, TruLens), threshold per lane, reject/regenerate/log logic.

**PII/PHI control:** detection (Presidio, spaCy patterns, LLM-based), redaction, tokenization for reversible workflows, provider selection for irreversible.

**Prompt injection control:** input classifier, structured decoding, spotlighting (Hines 2024), dual-LLM (Willison 2023), canary tokens.

**Toxicity/safety control:** Perspective API, Llama Guard, custom classifiers per organization values.

**Data access control:** row-level security integration with retrieval, provenance tracking.

**Threshold calibration via confusion matrix:** per lane. Standard lane may accept 5% false-positive rate; Elevated Risk must be ≤ 1%; Experimentation may accept higher for velocity.

**Build vs buy:** Arize, Fiddler, Guardrails AI, Varonis, Protect AI. When to build (differentiated use cases) vs buy (industry-standard controls).

### Pillar 4 — Policy-as-Code & Enforcement

**OPA (Open Policy Agent) / Rego:** general-purpose policy engine. Decompose "acceptable use" into policy rules. Runtime enforcement at API gateway, retrieval layer, model output.

**Sentinel (HashiCorp):** infrastructure policy-as-code. Enforce cloud region, encryption, IAM at Terraform apply time.

**Enforcement levels mapped to lanes:** advisory (log-only), warning (notify + allow), soft-fail (require override), hard-fail (block).

**Policy versioning:** git-tracked, code-reviewed, tested via policy CI.

**Testing policies:** OPA has `opa test`. Positive + negative test cases per policy.

### Pillar 5 — Governance & Operating Models

**AI Center of Excellence (CoE):** central team owns frameworks, tools, patterns. Business units execute.

**Operating model comparison:**
- **Centralized:** all AI decisions through CoE. Slow, safe.
- **Federated:** business units own AI, CoE provides guardrails + audit. Fast, requires strong controls.
- **Hub-and-spoke:** CoE for standards + high-risk projects; embedded champions in each BU.

**Frameworks operationalized:**
- **EU AI Act (2024):** risk tiers (unacceptable/high/limited/minimal), conformity assessments, CE marking, database registration
- **ISO/IEC 42001:2023:** AI Management System (AIMS) — plan/do/check/act cycle for AI programs
- **SR 11-7 (OCC/Fed 2011):** model risk management — model inventory, tiering, validation, monitoring, MRM committee

**Leading through influence:** the AI security architect rarely has direct authority over BU delivery teams. Persuasion, evidence, and clear escalation paths are the tools.

**C-suite communication:** monthly risk dashboard, quarterly board update, incident-specific briefings. Every metric ties to business outcome, not technical vanity.

### Pillar 6 — Agentic AI Security (Bridge to 2027)

Every security problem in Pillars 1-5 gets 10× harder when the AI acts autonomously.

**New threat surface:**
- **Excessive agency** (OWASP LLM08) — agent has too much access
- **Tool abuse** — agent uses tools in unexpected ways
- **Cascading errors** — small mistake compounds through multi-step plan
- **Prompt injection at any hop** — injected content at step 1 poisons steps 2-N
- **Goal drift** — agent's objective shifts under adversarial context
- **Reward hacking at agent scale** — agent optimizes for stated goal in unintended ways

**Controls:**
- Least-privilege tool access
- Budget constraints (cost + time + tool-call count)
- Human-in-the-loop checkpoints for high-impact actions
- Behavior monitoring (anomaly detection on tool call patterns)
- Kill switches
- Sandboxed execution for code-generating agents

**This pillar previews the entire Agentic AI Roadmap 2027 security surface.**

## Cost of Getting Security Wrong

- **Prompt injection in prod:** Chevrolet dealer chatbot agreed to sell an SUV for $1. Air Canada bot promised bereavement fare and tribunal held Air Canada liable. Brand damage + legal liability.
- **Data exfiltration via agent:** hypothetical still — but the first "agent leaked source code / confidential docs via tool call" incident is a matter of when, not if.
- **Regulatory penalties:** EU AI Act up to €35M or 7% global revenue. GDPR up to €20M or 4%. HIPAA up to $50K per record.
- **Failed audit:** SOC 2 findings block enterprise sales. AI-specific findings soon standard.
- **Board / regulator escalation:** the CISO who can't answer "prove AI is safe" gets replaced.

Week 16 fills these gaps at the senior architect level.

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1-15 | LLM Fundamentals → The Missing Layer | (see main README) | ✅ |
| **16** | **The Security Layer (Bonus Capstone)** ← you are here | [Episode_16_The_Security_Layer](https://github.com/JoshithReddyAleti/Episode_16_The_Security_Layer) | ✅ |
| 17+ | **Agentic AI Roadmap 2027** | (coming next) | 🚧 |

**Week 16 is the bonus capstone and the bridge into the Agentic AI Roadmap 2027.** The security layer is where the two series meet.
