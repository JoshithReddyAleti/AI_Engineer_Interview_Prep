# 🎭 Week 15 — Behavioral & Scenario Questions

> **Focus:** $180K/month cost explosion, reasoning model 30× more expensive than expected, cross-tenant data leak in production, SOC 2 audit finding, GDPR request paralyzed team for 2 weeks, EU AI Act blocked launch
>
> **Depth:** frontier-lab reasoning process + enterprise architect reasoning process — mechanism → diagnostic path → fix → prevention.

---

## Q1. The $180K/Month LLM Bill That Nobody Predicted ⭐⭐⭐⭐

**Scenario:** Your team shipped an AI-powered support agent 3 months ago. It's working — customers love it, resolution rates up 40%. Finance sends you a Slack: 'LLM API bill last month was $180K. Budget was $20K. What happened?'

**Strong answer:**

"Cost blowup is the second most common way AI features die in production (first is a security incident). My response:

**Hour 0-4: Immediate cost triage**

1. **Pull the bill breakdown.** By provider, by model, by day. Look for spike patterns.

2. **Correlate with product usage.** Did query volume 9× overnight? Or is per-query cost 9× baseline?

3. **Emergency circuit breaker if still active.** Set a hard daily budget limit. Route to cheapest model above budget. Don't let bill compound while investigating.

**Day 1-2: Root-cause the cost drivers**

**Hypothesis 1: Agent runaway loops.**
Support agent enters loop, generates 20 LLM calls per resolution instead of 3. Check per-conversation LLM call distribution. Look for outliers (conversations with 50+ calls).

**Hypothesis 2: Context bloat.**
Retrieval returns 20 documents per query, all in prompt. Or conversation history accumulates without summarization. Check average input token count over time.

**Hypothesis 3: Wrong model tier.**
Everything routed to Opus when Haiku would suffice for classification steps. Check model usage distribution.

**Hypothesis 4: No prompt caching.**
System prompt sent uncached every query. If system prompt is 2000 tokens and there are 10M queries/month, that's 20B tokens = $60K uncached on Sonnet vs $6K cached.

**Hypothesis 5: Adversarial usage.**
Bad actor discovered the endpoint, running expensive queries at scale. Check per-user query volume distribution.

**Hypothesis 6: Volume growth.**
Legitimately 9× traffic. Success caused cost. Check user growth curve.

**Real diagnosis in a typical case:**

Usually a combination:
- 3× volume growth (product success)
- 2× per-query cost (context bloat: retrieval returns more docs since we added new sources)
- 1.5× because prompt caching not enabled (didn't ship with the feature)

Compounds to 9× baseline = $180K vs $20K.

**Day 3-7: Deploy fixes in priority order**

**1. Enable prompt caching** (Anthropic prompt cache or OpenAI equivalent). Immediate 30-50% reduction on input tokens.

**2. Add model tier router.** Classify each step. Route Haiku-appropriate to Haiku, Sonnet to Sonnet. Expected 40% cost reduction.

**3. Add context budget.** Cap retrieved context at top-5 docs, max 3000 tokens. Summarize conversation history above N turns.

**4. Add per-user + per-tenant rate limits.** Prevents abuse and runaway. Free tier: 100 queries/day. Paid: 5000. Enterprise: contract.

**5. Add cost budget alerts + hard cutoff.** Alert at 50%, 80%. Hard cutoff at 100% until manual override.

**6. Add cost attribution.** Per-tenant, per-feature. Enables informed decisions about which features are worth their cost.

**Week 2-4: Systemic changes**

**Cost engineering as first-class discipline:**
- Every AI feature has a unit economics doc BEFORE launch
- Per-query cost target explicit in feature spec
- Cost dashboards reviewed weekly
- Cost anomaly alerting always on

**Post-mortem:**
- What did we not know that we should have?
- How did we ship without cost budgets?
- What's the process change to prevent recurrence?

**Communication to leadership:**

'The AI support agent is generating strong customer value (40% resolution improvement). But we shipped without cost engineering, and cost scaled non-linearly with success. We're implementing tiered routing, prompt caching, and budget controls. Expected steady-state cost: $30-40K/month at current scale. Long-term unit economics: $0.15 per resolution vs $12 per human-handled ticket. The feature is profitable; we just weren't measuring it right.'

**Prevention (long-term):**

1. **Every AI feature launches with unit economics analysis.** What's the cost per user, per session, per feature interaction? What's the acceptable cost per user?

2. **Cost dashboards from day one.** Not built after fire. Built into the launch checklist.

3. **Router pattern by default.** No AI feature ships without model tier routing.

4. **Prompt caching by default.** Enable in the LLM gateway; features inherit automatically.

5. **Budget alerts by default.** Every feature has per-tenant + per-user budgets.

**The staff+ signal:**

Understanding that cost is a first-class architectural concern, not an operational afterthought. The engineers who bake cost engineering in from day one prevent this fire. The engineers who ship 'and figure out cost later' cause $180K/month surprises."

---

## Q2. Cross-Tenant Data Leak in Production ⭐⭐⭐⭐

**Scenario:** Enterprise customer emails support: 'Your AI just showed my confidential Q3 revenue projections when I asked about competitors.' Turns out they saw ANOTHER tenant's confidential data. What do you do?

**Strong answer:**

"This is one of the worst incidents an AI SaaS can have. My response:

**Hour 0-1: Immediate containment**

1. **Take the feature offline.** Kill switch. Prevent additional leaks while investigating.

2. **Assemble the response team:** eng lead, security, legal, PR, exec sponsor.

3. **Preserve evidence.** Full logs of the incident. The exact prompt, the exact response, the source of the leaked data.

4. **Notify legal + DPO immediately.** This may trigger breach notification obligations under GDPR (72 hours), state laws, contract SLAs.

**Hour 1-8: Investigation**

Root cause hypotheses:

**Hypothesis 1: Cross-tenant vector search.**
Vector DB shared across tenants. Query from Tenant A matched documents from Tenant B semantically. Missing tenant_id filter or filter bypassed.

**Hypothesis 2: Cache poisoning.**
Semantic cache shared across tenants. Response cached from Tenant A's session served to Tenant B.

**Hypothesis 3: Prompt injection.**
Malicious user included instructions that manipulated the LLM into accessing other data. Tool-use scenario.

**Hypothesis 4: Session mix-up.**
Session state accidentally shared. User's session pointer got another user's context.

**Hypothesis 5: RAG ACL misconfiguration.**
ACLs sync failed for the affected document. Access control bypassed.

**Investigation approach:**
1. Trace the exact request end-to-end
2. Identify where the leaked data entered the response
3. Check tenant_id at every layer where it should be present
4. Identify the guilty layer

**Real cases (industry precedent):**
- 2023-2024: multiple RAG deployments discovered leaking data across tenants
- Root cause almost always: missing tenant_id filter in retrieval OR shared semantic cache without namespacing

**Hour 8-24: Assess scope**

- How many other tenants could this have affected?
- How many actual incidents occurred (search logs for pattern)?
- What data was potentially leaked?
- Which tenants' data was involved?

**Day 2-5: Remediation + Disclosure**

**Technical fix:**
- Add tenant_id filter at every retrieval layer
- Enforce via contract tests, not just code review
- Add per-tenant cache namespaces
- Add integration tests for cross-tenant scenarios

**Disclosure:**

To affected tenants (may include multiple):
'On [date], we identified an incident where information from your tenant may have been shown to another tenant. Specifically: [detail]. We've fixed the root cause and are implementing additional controls. Under our contract and applicable law, this may constitute a security incident. Full incident report attached. We're available for a call to discuss.'

To regulators (if PII involved):
- GDPR: 72-hour notification to supervisory authority + affected data subjects
- HIPAA (if PHI): notification to HHS + affected individuals
- State laws: per-state notification requirements

**Do not underestimate legal exposure:**
- Contractual breach → potential termination + damages
- GDPR fines up to €20M or 4% global revenue
- HIPAA fines up to $50K per record
- Reputation damage compounds

**Week 2+: Systemic hardening**

**1. Tenant isolation as architectural invariant.**
Not 'code discipline.' Enforced via infrastructure: per-tenant DB schemas, per-tenant cache namespaces, per-tenant vector DB collections for enterprise.

**2. Chaos engineering for tenant isolation.**
Regularly simulate: 'what if a query lost its tenant_id?' Result should be: rejection, not cross-tenant leak.

**3. Canary tenants.**
Special tenants with distinctive test data. If Tenant A's canary data appears in Tenant B's responses, alert immediately.

**4. Audit every retrieval.**
Log tenant_id, user_id, retrieved doc IDs. Automated anomaly detection: if retrieval returned docs from tenant B for a tenant A query, alert.

**5. Contract tests in CI.**
Test suite explicitly verifies: no code path returns cross-tenant data. Runs on every PR.

**Communication:**

**Internal:**
'We had a cross-tenant data leak. It's serious. Feature offline. Investigation underway. Blameless postmortem coming. Systemic changes in progress. If you see anything else concerning, report immediately.'

**External:**
'We identified a security incident. Immediate remediation applied. Systemic controls being enhanced. Full disclosure to affected parties. Independent security audit engaged.'

**The staff+ signal:**

Understanding that this is not an ops incident. It's an architectural incident. The response is not 'add a filter.' It's 'redesign the isolation model.' The engineers who see the systemic vs surface fix are the ones who prevent recurrence.

Also: knowing that legal and PR response run in parallel with technical response. Coordination across dimensions is what staff+ engineers do.

**Frontier lab context:**

Every AI provider (Anthropic, OpenAI, Google, etc.) has had cross-tenant or data-boundary incidents. The industry pattern: strong incident response + transparent disclosure + systemic hardening. Not: cover up. Not: minimize."

---

## Q3. EU AI Act Blocked Launch Two Weeks Before Go-Live ⭐⭐⭐⭐

**Scenario:** Product team spent 8 months building an AI-powered candidate screening tool. Launch scheduled for next month. Legal review flags: 'This is high-risk under EU AI Act. Cannot launch in EU without conformity assessment.' Product is furious. What do you do?

**Strong answer:**

"This is the emerging class of AI compliance surprises. My approach:

**Day 1: Understand the finding**

Meet with legal to understand:
- What specifically triggers 'high-risk' classification?
- What does conformity assessment require?
- What's the timeline (weeks? months?)?
- What's the risk of launching without it?

EU AI Act (2024) classifies employment decision AI (including CV screening) as high-risk. Requirements:
- Risk management system
- Data governance
- Technical documentation
- Transparency to users
- Human oversight
- Accuracy monitoring
- Cybersecurity
- Conformity assessment (self-assessment for most; third-party for critical)
- CE marking
- Registration in EU database

Timeline: 3-6 months typically for first conformity assessment.

**Day 2-3: Assess options**

**Option A: Delay EU launch.**
Launch in US and non-EU markets on schedule. EU launches after conformity assessment complete.

**Option B: Redesign to reduce risk classification.**
Can we position as 'assistive tool' (limited risk) rather than 'decision-making tool' (high-risk)? Requires:
- Human always in the loop, no automated rejection
- Transparency to candidates
- Not the sole basis for decisions
- Documented human review process

**Option C: Full high-risk compliance sprint.**
Fast-track the conformity assessment. 3-6 months of work compressed. Requires: dedicated team, external counsel, likely third-party assessor.

**Option D: Don't launch this feature in EU.**
Some products decide EU risk isn't worth compliance cost.

**Day 4-7: Recommendation to leadership**

'The AI Act reality: CV screening AI is regulated. There's no way around this.

Options in order of feasibility:
1. Redesign to limited-risk (2-4 weeks): keep AI as decision-support, add human review requirement, launch on schedule with adjusted UX
2. Delay EU (0 cost, 3-6 months to add EU market): launch other regions on schedule
3. Full compliance (6+ months, $500K+ cost): launch EU with high-risk certification

I recommend Option 1. It preserves the product value, aligns with the AI Act's intent, and is achievable in the launch window.'

**Week 1-2: Execute Option 1**

**Product changes:**
- AI provides ranked candidates + reasoning; does not reject
- Human recruiter must review every AI-ranked candidate before rejection
- Candidates informed AI is used (transparency requirement)
- Appeal mechanism for candidates
- Audit log of every AI decision + human review

**Documentation:**
- AI system technical documentation (per AI Act Article 11)
- Risk assessment
- Data governance documentation
- Human oversight procedures

**Testing:**
- Accuracy monitoring on training data + production
- Fairness testing across demographic segments (protected characteristics)
- Documented bias mitigation

**Week 3-4: Legal review + launch prep**

- Legal signs off on limited-risk classification
- DPA drafted for EU customers
- Terms of service updated with AI transparency
- Candidate communications drafted

**Prevention (long-term):**

**1. Compliance-by-design.**
Every product spec includes regulatory classification EARLY. Not two weeks before launch.

**2. Regulatory landscape monitoring.**
EU AI Act is not the last one. Colorado AI Act, CCPA extensions, US federal AI regulation coming. Track proactively.

**3. Cross-functional compliance rituals.**
Legal + product + engineering meet monthly on AI regulation impact.

**4. Compliance architecture patterns.**
Human-in-the-loop, transparency, appeal mechanisms, audit trails — these should be default architectural patterns, not per-project scrambles.

**Communication:**

**To product:**
'The AI Act is real. It's not going away. We can launch on schedule with the redesign. It's a better product anyway — humans in the loop for high-stakes decisions is what we should be building.'

**To engineering:**
'Regulatory work is engineering work. It's not 'legal's problem.' We build compliance into the product architecture. This is a chance to build the pattern for future features.'

**To leadership:**
'We identified an EU AI Act compliance requirement in flight. Recommended redesign preserves the launch date and improves the product. Long-term, we need compliance-by-design as standard practice.'

**The staff+ signal:**

Understanding that AI regulation is architectural, not legal-department-only. The engineers who see the technical implications of regulations (human-in-the-loop = specific UX pattern; transparency = specific data disclosure; audit = specific data flow) are the ones who lead compliant AI development.

Also: staying calm when legal blocks a launch. This is normal. The response is not fight legal or blame legal — it's understand the requirement and design for compliance.

**Frontier lab context:**

Anthropic, OpenAI, Google all navigate EU AI Act complexity. Their approach: proactive compliance, published safety cards, transparent design documents. Emerging pattern for all AI companies."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. Reasoning Model 30× More Expensive Than Expected ⭐⭐⭐⭐
Team migrated to o1 for 'quality.' Cost 30×'d. But quality gain was 5%. Approach: (1) measure ROI properly (accuracy gain × cost of error × volume vs cost premium), (2) identify tasks where reasoning helps vs doesn't, (3) build router that uses reasoning selectively, (4) most tasks don't need reasoning — the router will send 90%+ to standard.

### Q5. SOC 2 Audit Finding on LLM Data Handling ⭐⭐⭐
Auditor: 'Your LLM provider sees customer data without proper agreements.' Approach: (1) verify contractual coverage (DPA, no-training clause), (2) if inadequate, migrate to BAA-covered provider, (3) update security posture documentation, (4) proactively address before formal finding.

### Q6. GDPR Request Paralyzed the Team for 2 Weeks ⭐⭐⭐⭐
User requests Article 15 access + Article 17 erasure. Team spent 2 weeks manually pulling data. Approach: (1) build automated erasure workflow (see coding Q13), (2) data catalog with per-user provenance, (3) standard operating procedure documented, (4) target: complete within 30 days automated.

### Q7. Enterprise Customer Requires On-Prem Deployment ⭐⭐⭐
$5M contract contingent on on-prem. Currently SaaS-only. Approach: (1) understand the specific requirement (data residency? sovereignty? air-gap?), (2) evaluate options: hybrid (self-hosted inference + our control plane), VPC deployment (dedicated in their cloud), full air-gap, (3) build only if TAM justifies, (4) alternative: silo tier with EU/US-Gov residency may satisfy.

### Q8. Explaining Compliance Cost to Product Manager ⭐⭐⭐
PM: 'Why does adding this feature cost 3 months for compliance?' Approach: quantify. 'SOC 2 CC6.1 requires access controls: 3 weeks. HIPAA §164.312 audit controls: 2 weeks. GDPR Article 17 workflow: 2 weeks. Testing + docs: 3 weeks. Alternative: launch without and face contract termination + regulatory fines. Compliance is a feature, not a tax.'

### Q9. Multi-Region Deployment Timeline Blown ⭐⭐⭐
Promised EU region in 2 months; 6 months in and still not shipped. Approach: (1) reset expectations honestly with leadership, (2) identify the blocker (usually provider region availability + data flow architecture), (3) minimum viable EU (deploy just inference in EU, control plane still US, with clear residency documentation), (4) phase full sovereignty.

### Q10. Reasoning Model Choice Landmine ⭐⭐⭐
Team picked o1 without evaluating alternatives. Costs are hurting. Approach: (1) evaluate DeepSeek-R1 (open, cheaper), Claude thinking (competitive), (2) provider-agnostic routing so future switches don't require rewrite, (3) evaluate per task, not blanket choice.

### Q11. LLM Vendor Locked-In and Prices Just Jumped 40% ⭐⭐⭐
Provider announces 40% price increase. Approach: (1) provider abstraction was should have been built earlier — case study for it now, (2) evaluate alternative providers (Anthropic, OpenAI, Azure OpenAI, self-hosted OSS), (3) migrate in phases with A/B validation, (4) target: never single-provider dependent again.

### Q12. Customer Discovers Their Data Was Used to Train Provider Model ⭐⭐⭐⭐
Public model release includes patterns clearly from customer data. Fury. Approach: (1) verify the claim, (2) audit provider contract (no-training clauses?), (3) if provider violated, contractual remedies, (4) if we misconfigured (didn't opt out where required), acknowledge + remediate, (5) proactively audit all providers' training policies quarterly.

### Q13. Feature Works Great in US, Broken in Asia ⭐⭐⭐
Users in Japan, Korea, Singapore report worse quality. Investigation: model trained mostly on English data, non-English performance degrades. Approach: (1) evaluate models with multilingual training, (2) fine-tune or route to multilingual models for non-English, (3) transparent quality metrics per region, (4) target: quality parity across supported languages.

### Q14. Enterprise Security Team Blocking Every AI Feature ⭐⭐⭐
Security team requires 6-week review per feature. Bottleneck. Approach: (1) build shared secure-by-default patterns pre-approved, (2) tiered review (low-risk features fast-track, high-risk full review), (3) security team involved in early design (not late gate), (4) trust built through good outcomes.

### Q15. Reasoning Model Gave Different Answer for Same Question ⭐⭐⭐
User complains: 'Yesterday it said X, today it says Y.' Nondeterminism. Approach: (1) explain that LLM responses are stochastic — expected, (2) temperature settings, seed values, cache invalidation contribute, (3) for critical answers: cache the answer + document as canonical, (4) for exploratory: variance is fine, (5) transparency in UX.

### Q16. Data Sovereignty Requirement Blocks Global Enterprise Deal ⭐⭐⭐
Fortune 100 requires 'data never leaves EU.' Current architecture routes through US for observability. Approach: (1) design per-tenant residency (silo tier), (2) EU-native inference + logging + backups, (3) documented in contract, (4) auditable via location metadata on every request.

### Q17. Bridging to Agentic AI Roadmap 2027 ⭐⭐⭐
Series ended (Week 15 = final). Team wants direction for next generation. Approach: (1) foundations carry (all Missing Layer work applies to agents), (2) new dimensions (autonomy scope, multi-agent coordination, long-horizon memory, cascading failure), (3) new specialties (agent evaluators, tool designers, agent safety engineers), (4) same discipline mindset, new territory. This bridges from AI Engineering Roadmap 2026 → Agentic AI Roadmap 2027.
