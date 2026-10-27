# 📖 Week 16 — Terminology Differentiation Reference

> **Focus:** Every set of similar-sounding Security Layer terms across risk, threat modeling, guardrails, policy, governance, and agentic security.
>
> **Why it matters:** these distinctions are how senior AI security architects communicate precisely with CISOs, auditors, and regulators. Confusing risk with threat loses credibility instantly. Confusing NIST AI RMF with ISO 42001 confuses regulators. Confusing guardrail with control confuses auditors.

---

# Section 1: Risk Foundations

## Risk vs Threat vs Vulnerability vs Hazard

### Risk

**Definition:** the potential for loss or harm from an event, quantified as likelihood × impact.

**Example:** "The risk of prompt injection leading to data leak in our customer service AI is medium-high."

**Property:** risk is a probability + consequence, not an event.

---

### Threat

**Definition:** a potential cause of an unwanted event (an actor, action, or circumstance).

**Example:** "The threat is a malicious user embedding injection payloads in uploaded documents."

**Property:** threats are agents or actions, not outcomes.

---

### Vulnerability

**Definition:** a weakness that could be exploited by a threat.

**Example:** "The vulnerability is that we don't sanitize retrieved content before passing to the LLM."

**Property:** vulnerabilities are properties of your system.

---

### Hazard

**Definition:** an inherent characteristic that could cause harm (used more in safety-critical domains).

**Example:** "LLM non-determinism is a hazard for medical decision systems."

**Property:** hazards are baseline properties, not attacks or weaknesses.

---

### 🎯 The Distinction That Matters

| Term | What It Is | Example |
|---|---|---|
| **Risk** | Probability × impact | High risk of data leak |
| **Threat** | Actor/action causing harm | Injection attacker |
| **Vulnerability** | System weakness | Unsanitized retrieval |
| **Hazard** | Inherent characteristic | Model non-determinism |

**Interview signal:** senior security architects use these precisely. "The risk is high because the vulnerability is exploitable by the threat under normal conditions." That's a full sentence with all four in correct roles.

---

# Section 2: Governance Frameworks

## NIST AI RMF vs ISO/IEC 42001 vs SR 11-7 vs EU AI Act

### NIST AI RMF 1.0 (2023)

**Full name:** National Institute of Standards and Technology AI Risk Management Framework

**Nature:** voluntary process guidance. Not certifiable. Widely referenced.

**Structure:** GOVERN / MAP / MEASURE / MANAGE + Companion Playbook

**Best for:** US organizations, general-purpose AI risk framework

---

### ISO/IEC 42001:2023

**Full name:** International Standard for AI Management Systems (AIMS)

**Nature:** certifiable via accredited bodies. International standard.

**Structure:** Plan-Do-Check-Act cycle. Documented management system requirements.

**Best for:** enterprise signaling to customers, EU market, international operations

---

### SR 11-7 (OCC/Fed, 2011)

**Full name:** Federal Reserve/OCC Supervisory Guidance on Model Risk Management

**Nature:** regulatory guidance (mandatory for US banks). Enforced by exams.

**Structure:** model inventory, tiering, independent validation, ongoing monitoring, MRM committee.

**Best for:** US banks. Increasingly referenced outside banking as gold-standard MRM.

---

### EU AI Act (2024)

**Full name:** Regulation (EU) 2024/1689 on AI

**Nature:** binding law. Penalties up to €35M or 7% global revenue.

**Structure:** risk tiers (unacceptable/high/limited/minimal), conformity assessment, CE marking, database registration.

**Best for:** required for any EU market participation with AI

---

### 🎯 The Distinction That Matters

| Framework | Nature | Certifiable | Enforcement |
|---|---|---|---|
| **NIST AI RMF** | Voluntary guidance | No | Reputational |
| **ISO 42001** | Standard | Yes (external audit) | Contract requirement |
| **SR 11-7** | Regulatory guidance | Enforced via exam | Bank supervision |
| **EU AI Act** | Binding law | Conformity assessment | Fines |

**Enterprise reality:** most large orgs need all four. NIST as process baseline, ISO 42001 as certification, SR 11-7 for banking divisions, EU AI Act for EU market.

---

# Section 3: Threat Modeling Frameworks

## STRIDE vs OWASP LLM Top 10 vs MITRE ATLAS vs MAESTRO

### STRIDE

**Origin:** Microsoft, 1999. General threat modeling.

**Perspective:** defender, system-decomposition-based.

**Categories:** Spoofing, Tampering, Repudiation, Information disclosure, DoS, Elevation of privilege

**Best for:** initial architecture threat modeling

---

### OWASP LLM Top 10

**Origin:** OWASP Foundation, 2023 (revised 2025).

**Perspective:** application-specific, known-vulnerabilities.

**Format:** Top 10 list with mitigations, applicability, examples.

**Best for:** developer education, control checklist, compliance mapping

---

### MITRE ATLAS

**Full name:** Adversarial Threat Landscape for AI Systems

**Origin:** MITRE, ongoing (analog to ATT&CK for AI)

**Perspective:** adversary-centric, tactics + techniques.

**Format:** matrix of tactics (columns) × techniques (rows), with real-world case studies.

**Best for:** red team scoping, executive briefings, threat intelligence

---

### MAESTRO

**Full name:** Multi-Agent Environment, Security, Threat, Risk, Outcome

**Origin:** emerging framework for agentic systems.

**Perspective:** layered agentic threat analysis.

**Layers:** Foundation / Fine-tuning / Deployment / Multi-agent / Tool / User / Ecosystem

**Best for:** agent architectures, autonomous system design

---

### 🎯 The Distinction That Matters

| Framework | Perspective | Best For |
|---|---|---|
| **STRIDE** | Defender, system-decomp | Initial arch review |
| **OWASP LLM** | Vulnerability checklist | Developer awareness |
| **ATLAS** | Adversary tactics | Red team, exec briefing |
| **MAESTRO** | Layered agentic | Agent design |

**These are NOT alternatives — they're complementary.** A comprehensive threat register uses all four. See Q2 in the technical coding file for the consolidation pattern.

---

# Section 4: Controls Terminology

## Guardrail vs Policy vs Control vs Countermeasure

### Guardrail

**Definition:** a runtime mechanism that filters or modifies AI inputs/outputs.

**Examples:** hallucination scorer, PII detector, prompt injection classifier.

**Property:** applied per-request.

---

### Policy

**Definition:** a rule or set of rules governing behavior. Often policy-as-code.

**Examples:** "Elevated Risk lane requires BAA-covered provider for PHI."

**Property:** applied at decision points (deploy time, runtime).

---

### Control

**Definition:** a broader category — mechanism, process, or procedure to reduce risk.

**Examples:** access control, audit control, guardrail. All are controls.

**Property:** the compliance framework term.

---

### Countermeasure

**Definition:** a specific action or mechanism to defeat or mitigate a specific threat.

**Examples:** spotlighting countermeasure against indirect prompt injection.

**Property:** threat-specific.

---

### 🎯 The Distinction That Matters

| Term | Scope | Compliance Context |
|---|---|---|
| **Guardrail** | Runtime input/output | Specific mechanism |
| **Policy** | Decision-time rule | Codified requirement |
| **Control** | Broad category | Framework language (SOC 2, ISO 42001) |
| **Countermeasure** | Threat-specific | Threat model language |

**Interview trap:** using 'guardrail' when you mean 'control' or vice versa. In compliance conversations, use 'control.' In engineering, use 'guardrail' for the mechanism.

---

# Section 5: Access Control Models

## RBAC vs ABAC vs ReBAC vs Zero-Trust

### RBAC (Role-Based Access Control)

**Model:** users have roles; roles have permissions.

**Property:** coarse, well-understood.

---

### ABAC (Attribute-Based Access Control)

**Model:** access rules use attributes (user, resource, context).

**Property:** flexible, complex.

---

### ReBAC (Relationship-Based Access Control)

**Model:** access based on relationships (Google Zanzibar).

**Property:** fine-grained sharing.

---

### Zero-Trust

**Model:** no implicit trust; verify every request.

**Property:** architectural principle applied over any of RBAC/ABAC/ReBAC.

**For AI:** every LLM call authenticated + authorized. No trust in network position.

---

### 🎯 The Distinction That Matters

| Model | Type | Granularity |
|---|---|---|
| **RBAC** | Access model | Coarse |
| **ABAC** | Access model | Context-aware |
| **ReBAC** | Access model | Fine-grained sharing |
| **Zero-Trust** | Principle | Applied over any model |

Zero-Trust is a paradigm; RBAC/ABAC/ReBAC are implementation models. Modern AI systems use zero-trust with RBAC + ABAC hybrid.

---

# Section 6: Operating Models

## Centralized vs Federated vs Hub-and-Spoke

### Centralized

**Model:** all AI decisions through central team.

**Trade-off:** consistent controls; bottleneck at scale.

**Best for:** early adoption, small orgs, highly regulated.

---

### Federated

**Model:** BUs own AI decisions; CoE provides standards.

**Trade-off:** scales fast; requires strong controls.

**Best for:** large orgs, high volume, tech-forward.

---

### Hub-and-Spoke

**Model:** CoE (hub) owns high-risk + standards; embedded champions (spokes) in each BU.

**Trade-off:** balanced.

**Best for:** most mature enterprises.

---

### 🎯 The Distinction That Matters

| Model | Speed | Consistency | Scale |
|---|---|---|---|
| **Centralized** | Slow | High | Poor |
| **Federated** | Fast | Variable | Excellent |
| **Hub-and-Spoke** | Balanced | Balanced | Good |

**Common evolution:** startup uses centralized → grows to hub-and-spoke → very large orgs add federated experimentation lane.

---

# Section 7: Defense Concepts

## Defense in Depth vs Zero-Trust vs Principle of Least Privilege

### Defense in Depth

**Principle:** multiple overlapping controls.

**Example:** input classifier + spotlighting + output validator + audit trail.

**Property:** each layer catches what previous missed.

---

### Zero-Trust

**Principle:** no implicit trust; verify every request.

**Example:** no trusted network zone; every API call authenticated.

**Property:** architectural.

---

### Principle of Least Privilege

**Principle:** grant minimum permission required.

**Example:** agent gets access only to tools it needs for the task.

**Property:** applied at grant time.

---

### 🎯 The Distinction That Matters

| Principle | Applies To |
|---|---|
| **Defense in Depth** | Control stack design |
| **Zero-Trust** | Trust model / architecture |
| **Least Privilege** | Permission grants |

All three are complementary. Modern security uses all three.

---

# Section 8: Incident Terminology

## Incident vs Event vs Alert vs Breach

### Event

**Definition:** any observable occurrence in a system.

**Example:** guardrail triggered.

**Property:** neutral, not necessarily bad.

---

### Alert

**Definition:** a notification generated when a condition is met.

**Example:** injection classifier triggered above threshold.

**Property:** requires triage.

---

### Incident

**Definition:** an event that requires response (has actual or potential impact).

**Example:** confirmed prompt injection with data exposure.

**Property:** formally tracked, has SLA.

---

### Breach

**Definition:** an incident involving unauthorized access to or disclosure of protected data.

**Example:** PII disclosed to unauthorized user.

**Property:** may trigger legal notification obligations (GDPR 72-hr, HIPAA, state laws).

---

### 🎯 The Distinction That Matters

| Term | Severity | Response |
|---|---|---|
| **Event** | Neutral | Log |
| **Alert** | Potential concern | Triage |
| **Incident** | Real impact | IR process |
| **Breach** | Regulatory event | Notification obligations |

**In interviews:** don't call an event a breach. Precision here signals seniority. Regulators use these words carefully; so should you.

---

# Section 9: Regulatory Instruments

## Guidance vs Regulation vs Law vs Standard

### Guidance

**Definition:** interpretive advice from a regulator or standards body.

**Nature:** non-binding but influential.

**Examples:** NIST AI RMF, OCC Bulletin 2011-12.

---

### Regulation

**Definition:** binding rules issued by a regulatory body under legal authority.

**Nature:** enforceable.

**Examples:** SR 11-7, HIPAA regulations.

---

### Law

**Definition:** binding rules enacted by legislature.

**Nature:** highest authority.

**Examples:** GDPR, EU AI Act, CCPA, state AI laws.

---

### Standard

**Definition:** technical specifications developed by consensus bodies.

**Nature:** voluntary unless referenced in regulation.

**Examples:** ISO 27001, ISO 42001, SOC 2 (Trust Services Criteria).

---

### 🎯 The Distinction That Matters

| Instrument | Authority | Binding |
|---|---|---|
| **Guidance** | Regulator/standards body | No |
| **Regulation** | Regulatory body | Yes (in scope) |
| **Law** | Legislature | Yes |
| **Standard** | Consensus body | Voluntary unless required |

**In interviews:** using precise regulatory terminology signals you can operate in the regulator meeting room. 'The EU AI Act is a law, not a regulation.' 'ISO 42001 is a standard, certifiable.' 'NIST AI RMF is guidance, voluntary.' These distinctions matter.

---

# Section 10: Agentic Security Terms (Bridge to 2027)

## Autonomy vs Agency vs Excessive Agency

### Autonomy

**Definition:** the degree to which a system operates without human intervention.

**Spectrum:** none / passive / suggestive / automated / autonomous.

---

### Agency

**Definition:** capability to take actions in the world.

**Property:** including tool use, external effects.

---

### Excessive Agency (OWASP LLM08)

**Definition:** agent has more capability than the task requires. Risk of amplified harm from errant or malicious behavior.

**Fix:** least-privilege tool access, HITL for irreversible actions, budget constraints.

---

## Cascading Failure vs Goal Drift vs Reward Hacking

### Cascading Failure

**Definition:** an error at step 1 compounds through steps 2-N of an agent workflow.

**Example:** wrong retrieval at step 1 → wrong plan at step 2 → wrong action at step 3.

---

### Goal Drift

**Definition:** agent's objective shifts under adversarial context or accumulated context.

**Example:** agent starts with 'help customer'; drifts to 'satisfy customer even at company cost.'

---

### Reward Hacking

**Definition:** agent optimizes stated goal in unintended ways.

**Example:** agent told 'maximize satisfaction' → offers everyone free refunds.

---

### 🎯 The Distinction That Matters

| Failure Mode | When | Detection |
|---|---|---|
| **Cascading failure** | Multi-step error propagation | Divergence detection |
| **Goal drift** | Adversarial context influence | Goal reaffirmation |
| **Reward hacking** | Optimization gaming | Constraint checking |

**These are the 2027 territory.** Agentic AI Roadmap 2027 opens where this pillar closes.

---

## 🎯 The Meta-Lesson

The Security Layer has terminology that trips up 90% of candidates because it spans risk management, compliance, threat modeling, controls engineering, and organizational design.

**Risk vs threat vs vulnerability** — three different concepts, not synonyms.
**NIST AI RMF vs ISO 42001 vs SR 11-7 vs EU AI Act** — different natures, complementary.
**Guardrail vs policy vs control** — engineering vs compliance vs framework language.
**Event vs alert vs incident vs breach** — escalating severity + response.
**Autonomy vs agency vs excessive agency** — the 2027 territory.

Loose terminology → loose thinking → wrong control → wrong lane assignment → wrong framework mapping → CISO doesn't trust you → board doesn't trust the program → regulator finds gaps.

Precise terminology → precise thinking → right controls → right lane → right framework citations → CISO trusts you → board trusts the program → regulators reward you.

**The vocabulary IS the seniority signal. And Week 16's vocabulary IS the AI Security Architect signal.**
