# 🎭 Week 16 — Behavioral & Scenario Questions

> **Focus:** CISO demanding proof it's safe, SVP demanding launch bypass, regulator audit finding, agent exfiltrated data via tool call, model went rogue, PII leak in prod, red team broke every guardrail, board wants AI velocity
>
> **Depth:** senior AI security architect + VP AI compliance level. C-suite communication, regulator engagement, incident command.

---

## Q1. The CISO Meeting: "Prove Your AI Program Is Safe" ⭐⭐⭐⭐

**Scenario:** New CISO joined 3 months ago. Schedules 90 minutes with you: *"I need to be able to defend the AI program to the board next quarter. Show me the program is safe. And no vendor slide decks — I want to see the actual framework."*

**Strong answer:**

"This is the meeting every AI security architect prepares for. My approach:

**Pre-meeting (day before):**

Prepare a 40-minute presentation with 4 sections + 50 minutes for questions. Print the artifacts I'll reference (RDRs, threat models, ADRs) so we can look at real examples.

**Section 1 (10 min): The Governance Framework**

Show the 7-factor risk model + lane routing. Walk through 3 real initiatives:
- Marketing content generator → Experimentation lane, self-service
- Customer service RAG → Standard lane, champion review
- Credit decision assistance → Elevated Risk lane, full CoE review + SR 11-7 tier 1

Show the Risk Decision Record for each. Show the ADR chain. Show the threat model for Elevated Risk one.

Message: 'We don't approve AI initiatives by vibes. Every one has an auditable decision record traceable to specific framework citations.'

**Section 2 (10 min): The Control Stack**

Show live dashboard:
- Guardrail trigger rates per lane
- Recent incidents (with resolution)
- Framework attestation status (SOC 2 CC, ISO 42001 progress, EU AI Act readiness by initiative)

Message: 'Controls aren't paper — they're enforced at deploy time (OPA policies) and runtime (guardrails), with observability confirming they work.'

**Section 3 (10 min): The Threat Landscape We're Defending Against**

Show consolidated threat register with STRIDE + OWASP LLM + MITRE ATLAS + MAESTRO coverage. Walk through 2-3 high-priority threats and their mitigations.

Reference real 2024-2026 incidents (Chevrolet chatbot, Air Canada tribunal) as concrete examples of what these controls prevent.

Message: 'We know what the attack surface looks like. We track it against public threat intelligence. Our controls map to specific threats.'

**Section 4 (10 min): What Keeps Me Up At Night**

Honest section. The gaps we're actively closing:
- Agentic capabilities coming — the security surface expands 10× (bridge to 2027 territory)
- One tenant's shadow IT deployment discovered last month — what we're doing about it
- Regulatory landscape moving faster than our current review cadence
- Talent gap: only 3 people fully cross-trained on the framework

Message: 'I'm not selling you a finished product. I'm showing you a program that's real, disciplined, and honest about its gaps.'

**Section 5 (50 min): Q&A**

Prepared for likely questions:

**Q: 'What's our exposure to prompt injection?'**
Show OWASP LLM01 coverage: layered defense (input classifier + spotlighting + dual-LLM for high-risk operations + canary tokens). Real test rate: red team achieves injection in <2% of attempts on Elevated Risk lane; <5% on Standard.

**Q: 'What happens when a critical incident occurs?'**
Walk through the IR playbook. Show a real recent incident and timeline. Show notification SLAs (regulator, customers, board).

**Q: 'What's our EU AI Act readiness?'**
Show inventory of EU-facing AI initiatives. Show classification per Act. 87% of high-risk have complete conformity docs; timeline for remaining.

**Q: 'How does this compare to peer companies?'**
Reference industry benchmarks (Gartner AI governance maturity, if applicable). Position honestly: 'We're ahead of peers on framework maturity, behind on automation.'

**Q: 'What do you need from me?'**
Specific asks: staffing (2 more champions), platform investment ($500K for continuous compliance automation), sponsor for cross-BU standardization (need CISO backing to reduce BU pushback).

**Follow-up (post-meeting):**

Send a written summary within 24 hours. Include:
- Decisions made
- Action items with owners + dates
- Artifacts referenced (links)
- Next check-in date

Update board deck template with what we discussed — CISO will use for board.

**Long-term relationship building:**

Weekly 15-min check-in for first 90 days. Then monthly 30-min. Include CISO on incident briefings. Invite to red team readouts.

**The staff+ signal:**

Understanding that this meeting is not about impressing the CISO with technical depth. It's about giving them confidence they can defend the program upward. Frame everything in terms they can repeat to the board. Show honesty about gaps — CISOs distrust anyone who claims perfection. Give them ammunition and give them problems they can help solve. That's how you become their trusted advisor, not just the person who runs their AI security."

---

## Q2. SVP Demands Launch Bypass: "This Compliance Process Is Killing Us" ⭐⭐⭐⭐

**Scenario:** SVP Customer Experience walks into your office: *"The customer service AI launch is 6 weeks delayed because you keep asking for threat models and red team results. Our competitor just launched. If you don't approve this by Friday, I'm going over your head. This has to ship."*

**Strong answer:**

"This scenario is the reality of AI security architect work. My approach:

**Immediate response (in the meeting):**

'I hear you. Let me be clear: I want this to ship too. What I need is 30 minutes to walk through what's actually gating this and what our options are. Then let's decide together.'

Don't get defensive. Don't lecture on process. Show respect for the business pressure.

**Walk through the actual gates:**

Pull up the specific initiative's RDR + threat model + gaps list. Concrete, not abstract:
- 'Threat model identifies indirect prompt injection risk via user-uploaded content. We have no mitigation yet. Real risk: a customer uploads a document with injection payload, agent leaks other customer data.'
- 'Red team ran last week. Found 3 exploitable paths in agent tool access. High-severity findings.'
- 'PII detection tested at 92% recall. Elevated Risk lane requires 99%. Gap.'

These are not process items. These are actual security gaps.

**Present options honestly:**

**Option A: Ship as-is.**
Risk: high probability of one of these paths being exploited in production. Business impact: brand damage (Chevrolet-style incident), potential regulatory issue, potential lawsuit if customer data leaks. Financial exposure: legal costs $500K-$5M + churn.

**Option B: Ship with mitigations (this week or next).**
Deploy spotlighting for user uploads: 3 days engineering.
Restrict agent tool access to whitelisted actions only: 2 days.
Improve PII detection with additional detector: 4 days.
**Total: 5-7 engineering days. Ship 1-2 weeks from now.**

**Option C: Reduce scope.**
Ship a lower-risk version (no autonomous actions, human-in-loop for everything). Full autonomy in v2 after mitigations.

**Option D: Accept risk.**
SVP formally accepts risk in writing. Risk Decision Record documents the residual risk and business justification. Signed by SVP + CIO + CISO. Filed for audit.

**My recommendation:** Option B. Achieves launch in 1-2 weeks. Addresses the actual risks. No regulatory exposure. Not a bureaucratic 6-week delay — it's a 1-2 week engineering commitment.

**Handle the escalation threat:**

'If you feel I'm wrong here, going up the chain is legitimate. Let's do it together — I'll set up a 45-minute meeting with CISO + your CEO + the sponsor. Show them the same options. If they choose Option A or D, I'll sign the RDR and we ship. My job is to give them the choice with full context, not to block indefinitely.'

Don't threaten to block. Frame it as escalation together, not adversarial.

**What NOT to do:**

- Don't cite policy without showing risk
- Don't say 'my hands are tied' — you have judgment; use it
- Don't stall or hope the pressure passes
- Don't hide the trade-offs from leadership
- Don't pull rank ('CISO said no') without engagement

**What TO do:**

- Present specific, engineered options
- Quantify the residual risk in business terms
- Offer a path to ship (Option B or C)
- If leadership chooses risk acceptance (Option D), document it cleanly

**Following meeting outcome:**

If SVP accepts Option B: engineering starts today, launch in 7-10 days. Weekly check-ins until launch.

If SVP escalates: schedule joint meeting with CIO + CISO within 48 hours. Come prepared with same options. Let leadership decide.

If leadership approves Option D: draft RDR with full residual risk documentation, get signatures, monitor closely post-launch, red team weekly for first month.

**The relationship dimension:**

SVPs remember two things about security architects: (1) did you help them ship, or (2) did you make their life harder. If every interaction becomes an obstacle, you're on the losing side. Even when you say no, help them see how to get to yes.

**Frame every interaction as:** 'How do we ship this responsibly?' Not: 'Why you can't ship this.'

**The staff+ signal:**

Understanding that security architect work is 20% technical and 80% negotiation, communication, and organizational influence. The technical answer is rarely the hard part. The hard part is presenting choices in a way leadership can act on. And knowing when the business risk actually justifies acceptance — because sometimes it does. Frameworks are guardrails, not walls.

**The seniority test:** can you tell a story where the security architect and the SVP end the meeting aligned, not adversarial? If every scenario is adversarial, you're doing security wrong."

---

## Q3. Regulator Audit Finding: "Your Model Governance Is Insufficient" ⭐⭐⭐⭐

**Scenario:** Bank exam by OCC (Office of Comptroller of Currency). Auditor finding: *'Your model risk management program does not adequately govern LLM-based systems. Non-compliance with SR 11-7. Matter Requiring Attention (MRA) issued. Response required within 30 days; remediation plan within 90 days.'*

**Strong answer:**

"OCC MRA is serious. It doesn't shut down the bank, but it's a formal regulatory finding that goes to the Board and stays on the record. Escalates to Consent Order if not remediated. My approach:

**Day 1: Assemble response team**

- Chief Risk Officer (leads regulator engagement)
- Head of MRM (owns SR 11-7 program)
- General Counsel (regulatory strategy)
- Me (AI security architect, technical lead)
- Head of Applied AI (business context)

Weekly cadence with the CRO. Daily standups for the response team for first 2 weeks.

**Day 1-3: Understand the finding precisely**

Meet with the exam team (or their designated point of contact). Ask:
- What specific LLM systems triggered the finding?
- Which SR 11-7 sections do they consider under-addressed?
- What would 'adequate' look like from their perspective?
- Precedent — has OCC published guidance on LLM MRM?

Don't argue. Listen. Regulators reward candor and engagement.

**Day 4-7: Assess the actual gap**

Compare our program to SR 11-7 requirements + emerging OCC guidance on AI:

Likely gaps for LLMs specifically:
- **Model inventory:** are all LLM initiatives registered? Or is 'ChatGPT wrapper' hidden from inventory?
- **Tiering:** are LLMs tiered per SR 11-7 materiality framework?
- **Validation:** are LLM systems independently validated? Statistical models had clear validation methodology; LLMs need adapted approach.
- **Ongoing monitoring:** are we monitoring for drift, provider updates, performance changes?
- **Documentation:** is the level of documentation for LLM systems equivalent to statistical models?

**Day 8-30: Draft the 30-day response**

Written response addressing:
1. Acknowledgment of finding + commitment to remediate
2. Root cause analysis (why the gap existed)
3. Interim controls in place while remediation planned
4. Governance changes (who owns, cadence, accountability)
5. Timeline for full remediation plan (90-day deliverable)

Tone: professional, non-defensive, showing genuine engagement. Not: 'We disagree with the finding.'

**Day 31-90: Full remediation plan**

Specific, dated, owned actions:

**Track 1: Model inventory completeness (30 days)**
- Sweep all BUs, identify unregistered LLM systems
- Register per SR 11-7 requirements
- Establish deployment gate: no LLM in prod without registry entry

**Track 2: LLM-adapted tiering (30 days)**
- Extend SR 11-7 tiering criteria for LLM-specific factors (autonomy, external exposure)
- Re-tier existing LLM inventory
- Update governance policy

**Track 3: LLM validation methodology (60 days)**
- Define validation approach for LLMs (adversarial testing, RAG faithfulness, drift monitoring)
- Independent validation team stands up capability
- Backlog: validate Tier 1 first (60 days), Tier 2 (next 60), Tier 3 (annual cycle)

**Track 4: LLM ongoing monitoring (30 days)**
- Deploy monitoring for: provider model changes, output drift, guardrail effectiveness
- SLA: detect drift within 24 hours, respond within 72

**Track 5: Documentation (60 days)**
- LLM-specific documentation templates (aligned to SR 11-7 sections)
- Backfill documentation for existing inventory
- Enforce via deployment gate

**Track 6: Governance (30 days)**
- MRM committee expanded to include AI security architect
- LLM-specific MRM sub-committee
- Board-level reporting on AI/LLM risk quarterly

**Day 90-180: Execution**

Weekly progress reports to CRO. Monthly to OCC point of contact. Show tangible progress.

**Ongoing (post-remediation):**

Third-party MRM audit annually. Continuous evidence collection. Ready for next exam.

**Communication upward:**

Board Risk Committee update at next meeting:
'OCC exam identified LLM MRM gap. MRA issued. Remediation plan committed. Estimated cost: $3M investment + 4 FTE headcount. Timeline: 6 months to full remediation. Risk mitigation in place immediately. Regular updates through remediation period.'

Frame it as:
- We took the finding seriously
- We're remediating faster and more comprehensively than required
- We're using this as opportunity to lead peers in AI governance
- Investment ask is proportional to the risk exposure

**What NOT to do:**

- Don't argue the finding at length
- Don't attempt to minimize scope
- Don't drag remediation timeline
- Don't attempt separate response for each LLM system — comprehensive program response

**What TO do:**

- Engage regulator constructively
- Over-deliver on remediation
- Use as catalyst for improvements you wanted to make anyway
- Document everything with regulator-facing evidence quality

**Long-term:**

MRA closed if remediation complete. Consent Order (much worse) avoided. Program stronger. Board impressed with response. Career-defining execution.

**The staff+ signal:**

Understanding that regulator findings are not disasters if handled well — they're catalysts for necessary changes. The AI security architect who leads a strong response becomes trusted by the CRO and CISO for years. The one who fights the finding or tries to minimize it damages the relationship with regulators (which is career-limiting).

Regulators talk to each other. Peer regulators watch each other's exam findings. How you handle one exam sets the tone for all future exams. Play the long game."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. Agent Exfiltrated Data via Tool Call ⭐⭐⭐⭐
Customer service agent used the 'send email' tool to email confidential doc to attacker's address via injection. Approach: immediate containment (disable tool), incident response, forensic analysis (what data, how many customers), notifications (customer, regulator per GDPR/state law), root cause (tool permission too broad, no output validation), remediation (least-privilege tool access, HITL for external comms, output classifier).

### Q5. Model Went Rogue: Started Making Wild Recommendations ⭐⭐⭐
LLM feature started giving nonsensical, potentially harmful advice overnight. Approach: (1) immediate rollback to last-known-good, (2) investigate provider (Anthropic/OpenAI silent model update?), (3) audit prompt cache (stale cached responses?), (4) rerun eval suite (drift detection), (5) systemic fix (version-pin models, drift monitoring, canary golden set).

### Q6. PII Leak in Production ⭐⭐⭐⭐
Guardrail missed PII in LLM output. Customer email address exposed to other customer. Approach: containment, incident response, notification workflows (72-hr GDPR clock), audit trail investigation, root cause (guardrail regex missed a format), remediation (add LLM-based PII verifier, ensemble scoring, false-negative testing quarterly).

### Q7. Red Team Broke Every Guardrail ⭐⭐⭐
Contracted red team firm reports: 95% of jailbreaks succeeded, PII detection bypassed via base64 encoding, prompt injection via clever formatting. Approach: (1) don't defensively minimize — this is why we red team, (2) prioritize by severity + likelihood, (3) 90-day remediation plan, (4) partner with red team on iterative testing, (5) transparently report to leadership.

### Q8. CIO Demands "AI Velocity" Despite Governance ⭐⭐⭐⭐
CIO memo: 'We need to ship AI 5x faster. Governance is bottleneck.' Approach: (1) don't argue against velocity — enable it, (2) analyze actual bottleneck (usually Elevated Risk lane review time), (3) invest in tooling (automated evidence collection, policy-as-code enforcement, guardrail library), (4) hub-and-spoke with champion program to distribute load, (5) commit to metrics: '80% of Standard lane initiatives ship in <3 weeks by end of quarter.'

### Q9. Board Member Asks "Should We Be Worried About AI?" ⭐⭐⭐
Not technical question — governance question. Approach: (1) confident, non-defensive answer, (2) show governance framework in 3 slides, (3) frame in business terms (risk exposure, mitigation ROI), (4) offer deep-dive session if interested, (5) build ongoing board relationship.

### Q10. Employee Deployed Shadow AI ⭐⭐⭐
Discover BU team deployed ChatGPT integration for HR data outside governance framework. Approach: (1) don't punish first — investigate why they bypassed process, (2) immediate rollback, (3) incident response (assess data exposure), (4) systemic fix (make sanctioned path faster than shadow path), (5) BU-level accountability with sponsor.

### Q11. Vendor Discovered Using Our Data For Training ⭐⭐⭐⭐
LLM provider clause allowed training; renewal missed it. Discovered when public model showed patterns clearly from our data. Approach: (1) contractual remedies, (2) immediate provider switch to no-training tier, (3) legal review of exposure, (4) customer notification if their data affected, (5) contract review process improvement.

### Q12. Framework Certification Failed Audit ⭐⭐⭐
ISO 42001 external audit found substantial nonconformities. Approach: (1) understand the specific findings, (2) formal response with corrective action plan, (3) re-audit within 90 days, (4) address root cause (usually implementation-documentation gap), (5) communicate to leadership without spin.

### Q13. Competitor's AI Incident Puts Focus on Ours ⭐⭐⭐
Competitor's AI made news for privacy breach. CIO/CISO ask: 'Are we exposed to this?' Approach: (1) rapid assessment (24 hours) of similar patterns in our systems, (2) honest gap analysis, (3) proactive remediation of any similar gaps, (4) written brief for board, (5) use as catalyst for framework improvements.

### Q14. Talent: Senior Governance Engineer Quits ⭐⭐⭐
Champion in critical BU leaves for another company. Approach: (1) don't panic — succession planning matters, (2) knowledge transfer with cross-trained backup, (3) champion program's redundancy is the design goal (no single point of failure), (4) recruit + train replacement, (5) if pattern emerges (multiple champions leaving), understand why.

### Q15. Regulatory Framework Changes Mid-Deployment ⭐⭐⭐⭐
EU AI Act guidance issued that changes classification of your live system from limited to high-risk. Approach: (1) immediate impact assessment, (2) work with legal on transition period options, (3) redesign or accept high-risk classification (with conformity assessment), (4) transparent communication to customers, (5) frame as normal regulatory-landscape evolution, not crisis.

### Q16. C-Suite Wants To Fire Someone After Incident ⭐⭐⭐
Incident occurred; C-suite pressure to fire engineer who deployed. Approach: (1) advocate for blameless post-mortem culture, (2) root cause is usually systemic not individual, (3) firing chills future disclosure, (4) if there's genuine willful violation, that's HR not engineering, (5) frame consequences at the system level (process improvement) not the individual level.

### Q17. Bridging to Agentic AI 2027 ⭐⭐⭐⭐
Series ending Week 16 = final. Team wants direction: what's next? Approach: (1) foundations carry (all Missing Layer + Security Layer work applies), (2) agentic frontier expands threat surface 10× (MAESTRO, excessive agency, cascading failure), (3) new specialties emerging (agent safety engineers, autonomous system compliance), (4) framework updates coming (agentic-specific EU AI Act guidance, expanded OWASP LLM Top 10), (5) invest in agent evaluation, agent security, agent governance now — this is the 2027 frontier.
