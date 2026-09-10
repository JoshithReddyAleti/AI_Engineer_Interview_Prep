# 🎭 Week 11 — Behavioral & Scenario Questions

> **Focus:** Silent quality regression, hallucination in production, cost spike investigation, model version drift, PII leak in logs, alert fatigue, unmonitored deployments, incident response
>
> **How to use:** These are the 3 AM incidents that separate senior engineers from principals. Practice reasoning out loud — interviewers score your incident response process, not just your solution.

---

## Q1. The Silent Quality Regression ⭐⭐⭐⭐

**Scenario:** A key enterprise customer calls: "Your AI has been giving worse answers for two weeks. We're considering churning." You check — no incidents, no error rate spikes, no deploys. All monitoring shows green. What went wrong, what do you do?

**Strong answer:**

"This is the classic silent failure — the exact scenario observability is supposed to catch. Two-week lag from degradation to detection is a serious observability gap. My response:

**Immediate (First 4 hours):**

**1. Acknowledge customer, buy time.**
'We're taking this seriously. Investigating now with dedicated team. I'll update you in 4 hours.'

Don't dismiss ('we don't see any issues'). Don't overpromise ('we'll fix it today'). Buy time to actually investigate.

**2. Confirm the regression is real.**

- Ask customer for SPECIFIC examples with dates
- Run those exact queries through current system, compare responses
- If they refuse to share (some enterprises won't), give them a golden set and ask which are now bad

**3. Check quality metrics (which we should have but likely don't at sufficient granularity):**
- LLM-as-judge sampled scores over time — any trend?
- User feedback rates — any change?
- Faithfulness scores per tenant

If we DO have these and they show degradation → we missed our own signal. If we DON'T → observability gap is the real story.

**Deep investigation (Days 1-3):**

**Since nothing was deployed, my prime suspects:**

**Suspect 1: Model version drift.**
The provider silently updated the model behind a stable name. Common with OpenAI/Anthropic — model 'gpt-4-turbo' can point to different weights over time.

**Test:**
- Check: are we using version-pinned model IDs or stable aliases?
- If aliases: run golden set now vs 2 weeks ago (if we have historical data)
- Check provider release notes for the timeframe
- Community forums: are other teams seeing similar?

**Fix if this is it:** pin to specific version (`gpt-4-0125-preview` not `gpt-4-turbo`). Set up weekly golden-set runs against production to catch this proactively.

**Suspect 2: Embedding drift.**
Users started asking different types of questions. Embedding space of queries shifted. Existing chunk embeddings no longer match well.

**Test:**
- Compute PSI on query embeddings — this week vs 3 weeks ago
- Any significant drift (PSI > 0.2)?
- Check top query terms — new terminology entering?

**Fix if this is it:** re-embed corpus with current model, or expand corpus to cover new query types.

**Suspect 3: Data freshness / index staleness.**
Our RAG corpus is stale. Users ask about recent stuff, index has old info.

**Test:**
- When did the index last update?
- Are users asking about recent events?
- Check retrieval scores — are they consistently low (relative to baseline)?

**Fix if this is it:** trigger index refresh. Set up automated freshness monitoring.

**Suspect 4: Prompt drift.**
Someone edited a system prompt. 'To be more concise' or 'to be safer.' Impact wasn't measured.

**Test:**
- Git log on prompt files — any changes in last 3 weeks?
- Feature flags on prompt versions — any changes?
- Compare quality per prompt version

**Fix if this is it:** rollback. Add: mandatory eval on prompt changes before deploy.

**Suspect 5: Retrieval regression.**
Reranker or embedding model was updated. Vector DB config changed.

**Test:**
- Git log on retrieval code
- Vector DB configuration changes
- Cache hit rate changes (indirect signal)

**Root cause identified — remediation (Week 1):**

Whatever the specific cause, the LESSON is the same: we didn't have monitoring that would have caught this.

**Immediate fixes:**
1. Rollback / patch the specific issue
2. Communicate to customer honestly ('root cause was X, we've fixed it, here's what we're doing to prevent recurrence')
3. Offer service credit or extension

**Systemic fixes:**

**1. Golden set continuous evaluation.**
100+ curated queries with expected quality. Runs hourly against production. Baseline quality tracked. Alert on drift.

**2. Model version pinning + monitoring.**
Never use stable aliases in production. Weekly probing queries to detect provider changes.

**3. Prompt version tracking.**
Every prompt change → mandatory eval on golden set → deploy only if quality maintained. Prompt version is a dimension in observability.

**4. Drift detection.**
Weekly PSI on query embeddings. Alert on drift.

**5. Customer-visible quality dashboard.**
For enterprise customers, expose their own quality metrics. Builds trust. Also creates accountability.

**Communication with customer:**

'We identified the root cause: [specific technical explanation]. It was silent — no error, no crash, just gradual quality drop from [X to Y timing]. We should have caught this ourselves and didn't. Here's what we've deployed to catch it next time: [specific monitoring]. Also, we're offering [credit/extension] for the impact.'

Honesty + technical specificity + concrete prevention. Enterprise customers respect this.

**Postmortem:**

Blameless. Focus:
- Why did our observability miss this?
- What signals SHOULD have alerted?
- What are we deploying to prevent recurrence?
- Action items with owners.

**What I would NOT do:**
- 'Everything looks fine, must be user error' (dismissive, customer churns)
- 'It's the LLM provider's fault' (not accountable)
- Deploy a fix without understanding root cause (may just move the problem)
- Hide the observability gap (worse when discovered later)

**The senior insight:**

Silent quality regression is the hardest failure mode. Standard monitoring misses it entirely. Preventing it requires: golden set eval running continuously, LLM-as-judge on production sample, drift detection, prompt versioning, and model version pinning. Any product without these WILL have this incident. Design observability to catch what monitoring misses."

---

## Q2. The $30K Weekend Cost Spike ⭐⭐⭐⭐

**Scenario:** Monday morning. Finance pings: 'We spent $30K on OpenAI over the weekend. Normal weekend is $2K. What happened?' No dashboards show anything unusual because... wait, you don't have cost anomaly detection. What do you do?

**Strong answer:**

"This is a cost observability gap. My response, in phases:

**Phase 1 — Stop the bleeding (Hour 1):**

1. **Confirm the spike.** Check OpenAI billing dashboard directly — is spend still accruing? What model? What endpoint?

2. **Emergency actions if still accruing:**
   - Rotate API key (stops all usage temporarily)
   - Wait 5 min, verify usage stopped
   - Now investigate safely

3. **Preserve evidence.** Snapshot logs, traces, metrics from the weekend window BEFORE they rotate.

**Phase 2 — Diagnose (Hours 2-6):**

Without proper cost observability, I have to reconstruct from logs:

**Query 1: When did spike start?**
Aggregate LLM API calls per hour from logs. Look for step change.

**Query 2: What's dominating?**
- By model: cheap or expensive model?
- By endpoint: which endpoint drove volume?
- By user: which user(s) caused it?
- By tenant: which tenant?

**Query 3: What was different?**
- Compare weekend to previous weekends
- Volume difference? Cost-per-request difference?

**Common root causes for 15x weekend spike:**

**Cause 1: Bad user (or attacker).**
One user discovered they could hammer an endpoint. No rate limiting caught it. Common if rate limits are request-based, not cost-based.

**Cause 2: Agent stuck in loop.**
Autonomous agent got stuck. No iteration limit. Ran for 48 hours per user session.

**Cause 3: Feature launch cost misjudged.**
New feature launched Friday. Cost projections wrong (10x). Nobody noticed over weekend.

**Cause 4: Prompt bloat.**
Someone updated a prompt adding 10K tokens of context. Every request now costs 10x. Rolled out Friday.

**Cause 5: Provider price change.**
Rare but possible. Model prices changed. We didn't notice.

**Cause 6: Wrong model routing.**
Config change caused traffic to route to expensive model (GPT-4) instead of cheap (GPT-4o-mini).

**Phase 3 — Immediate remediation (Days 1-3):**

Once root cause identified:

**Specific fix for THIS incident:**
- If loop: kill affected sessions, add iteration limits
- If bad user: revoke access, investigate their intent
- If feature: apply per-feature cost cap
- If prompt: rollback
- If routing: fix config

**Systemic fixes (the real work):**

**1. Cost observability from day 1 (should have had this):**
- Per-hour cost dashboard
- Per-user/tenant/feature/model breakdown
- Real-time (not billing-report lag)

**2. Cost anomaly detection:**
- Baseline: rolling 7-day per hour
- Alert: current > 3x baseline (moderate) or > 10x (severe)
- Route: severe → page on-call

**3. Cost-based rate limiting:**
- Per-user daily budget
- Per-tenant monthly budget
- Hard cutoff, not just warning

**4. Weekend/off-hours monitoring:**
- On-call for cost alerts, not just service outages
- Or: automatic throttle when spike detected without human on-call

**5. Cost review in every feature launch:**
- Estimate cost per request
- Test at scale
- Set caps before deploy

**Phase 4 — Recovery (Week 1):**

**Financial:**
- Contact OpenAI. Legitimate case for partial cost forgiveness (especially first-time customer). Not guaranteed but worth asking.
- Report to CFO with root cause + prevention
- Budget adjustment if needed

**Postmortem:**
- Blameless
- Focus: our observability gaps allowed this
- Concrete actions: cost dashboards, anomaly detection, alerts

**Communication:**

To CEO/CFO: 'Cost spike root cause: [specific]. Fixed. Adding cost observability we lacked. Weekly cost reviews going forward. Estimated recovery from provider: [X]. Long-term: this class of issue prevented.'

To team: blameless — the SYSTEM allowed this, not individuals. Cost observability was a known gap; we're closing it.

**The senior insight:**

Every AI startup has this incident. The difference between $500 and $30K is whether cost observability exists. Cost is a first-class signal in AI systems — treat it like error rate or latency. Dashboards, alerts, anomaly detection, forecasting. Not optional."

---

## Q3. The Hallucination That Went to a Regulator ⭐⭐⭐⭐

**Scenario:** Your customer support AI told a customer in a regulated industry that they qualified for a benefit they don't qualify for. Customer took action based on this. Regulator now investigating. Legal calls. What do you do?

**Strong answer:**

"This is severity-1: a compliance incident AND a potential legal liability. My response:

**Hour 0-4 — Contain and preserve:**

**1. Preserve evidence.**
- Full trace of the offending interaction
- User's conversation history
- Model version at time of interaction
- System prompt at time of interaction
- Retrieved context at time of interaction
- Any related interactions from same user

Snapshot IMMEDIATELY before anything rotates. Legal will need this.

**2. Assess scope.**
- Only this user? Or others got similar wrong information?
- Search logs for similar patterns: same query type + wrong benefit + affirmative response
- Any other affected customers?

**3. Notify legal and compliance.**
This is their incident to lead now. Engineering supports.

**4. Communicate carefully with the customer.**
Legal will craft the exact wording. Do NOT admit fault engineering-side without legal review. Do acknowledge, do apologize, do commit to investigation.

**Investigation (Days 1-7):**

**Understanding the failure:**

**Was this a retrieval or generation failure?**

Load the trace. What was in retrieval?

If retrieval had CORRECT information → generation failure (model ignored context OR misinterpreted)
If retrieval had NO information about this → grounding failure (model should have said 'I don't know')
If retrieval had INCORRECT information → data quality failure (our source docs are wrong)

Each has different remediation.

**Category-specific investigation:**

**Category 1: Model hallucinated with no basis.**
- No supporting context, model made it up
- Root cause: no 'I don't know' behavior, no grounding requirement
- Fix: prompt requiring refusal when info missing, post-generation validation

**Category 2: Model contradicted context.**
- Context said 'not eligible if X,' model said 'eligible'
- Root cause: model interpreted incorrectly
- Fix: stronger model, structured output, post-generation fact-check against context

**Category 3: Correct output, wrong source data.**
- Our knowledge base had wrong info
- Root cause: data quality
- Fix: audit source data, correct, add data quality gates

**Category 4: Ambiguous question, wrong interpretation.**
- User asked ambiguously, model picked wrong interpretation
- Root cause: no clarification-seeking behavior
- Fix: prompt to clarify before affirming eligibility

**Root cause identified — remediation:**

**Immediate (Days 1-3):**

1. **Disable AI for this class of question.** If it's about legal/regulatory eligibility, route to human. Non-negotiable in regulated industries.

2. **Deploy human-in-loop for high-stakes questions.** AI can draft, human must approve before sending.

3. **Add specific guardrails.**
   - Anti-affirmation prompt: 'Never confirm eligibility or make binding statements. Always advise users to verify with a human representative.'
   - Refusal on eligibility questions.
   - Warning banner: 'This is AI-generated. Verify with official documentation.'

**Systemic (Weeks 2-4):**

**1. Categorize risk of every AI-facing question type.**
- Low risk: general product info (AI OK)
- Medium risk: policy explanations (AI OK with disclaimers)
- High risk: eligibility, legal, compliance (HUMAN ONLY or HIL required)

**2. Observability specifically for regulated interactions.**
- Every regulated interaction logged with full context
- Retention: 7-10 years per regulator
- Immutable, tamper-evident
- Auditor access ready

**3. Compliance-grade eval:**
- Golden set of regulated questions with correct answers
- Every model / prompt change → mandatory eval
- Regression = block deploy
- Regulator can review our eval process

**4. External audit before re-enabling AI for regulated content.**
- Third-party review of guardrails
- Sign-off from legal
- Regulator notified of remediation

**Regulator engagement:**

Cooperate fully. Provide:
- Timeline of events
- Root cause analysis
- Remediation plan
- Ongoing controls
- Attestation

Do NOT: obstruct, minimize, blame the LLM provider. This is our system, our responsibility.

**Customer engagement:**

- Restitution if applicable (whatever the customer lost/spent based on wrong info)
- Direct apology from leadership
- Ongoing communication until resolved

**Communication with team:**

Legal + compliance + engineering + product all involved. Blameless internally but rigorous. Not: 'the AI hallucinated' (unaccountable). Yes: 'our AI system produced incorrect information in a regulated context; here's why our safeguards didn't catch it; here's what we're changing.'

**What I would NOT do:**

- Blame the LLM ('these models sometimes hallucinate' — true but not our excuse)
- Fix quietly without disclosure (regulators find out worse when you hide)
- Rush to re-enable AI (compliance safety > engineering velocity)
- Minimize to customer or regulator
- Blame individuals internally

**The senior insight:**

Regulated industries have zero tolerance for AI errors. The safeguards must be layered: input classification (avoid high-risk topics), grounding requirements (refuse if unsure), post-generation validation (fact-check against context), human-in-loop (for high-stakes), audit trails (evidence for compliance). If ANY of these are missing, AI shouldn't touch regulated content."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. Deploying to Production Without Observability ⭐⭐⭐
Team launched new AI service quickly, no observability. Now debugging is a nightmare. Approach: (1) Emergency: add MINIMUM viable observability (structured logs, basic metrics), (2) Systematic: OTel instrumentation, dashboards, alerts, (3) Cultural: no service reaches prod without observability. It's not "add it after launch" — it's launch requirement.

### Q5. PII Leaked in Logs ⭐⭐⭐⭐
Discovery: production logs contain user emails, credit card numbers, medical info in plaintext. Approach: (1) Immediate: rotate credentials in logs, secure log access, (2) Purge affected logs (or move to secure tier), (3) Add redaction at logging layer (Presidio), (4) Audit: what data was accessed by whom? (5) Compliance disclosure per GDPR/HIPAA. Never trust upstream code to redact — enforce at edge.

### Q6. Alert Fatigue Killed the Team ⭐⭐⭐
On-call rotation is burning out engineers. Too many alerts, most non-actionable. Approach: (1) Alert audit — for last 30 days, which alerts were actionable? (2) Kill non-actionable alerts (auto-resolved, false positive prone), (3) Consolidate related alerts (grouping), (4) Symptom-based over cause-based, (5) Track "alert quality" as metric. Best teams have 3-5 pages/week; more = broken system.

### Q7. The Provider Silently Updated Their Model ⭐⭐⭐⭐
Discovery: quality regressed with no changes on our side. Provider updated model behind stable name. Approach: (1) Pin to specific version (`gpt-4-0125-preview`), (2) Add weekly probing queries (canary), (3) Track model fingerprints (response patterns), (4) Alert on fingerprint changes, (5) Communicate concern to provider (they may have policies against silent updates for enterprise customers).

### Q8. Debugging a Trace That Doesn't Exist ⭐⭐⭐
Customer reports bad response. You look for trace_id — no traces retained (sampling dropped it). Approach: (1) Immediate: extract everything from logs (worse but partial), (2) Systemic: sampling strategy issue — 100% of errors + slow + user complaints should be kept, (3) Trace retention should support incident investigation lag (30+ days). "We dropped the trace" is an unacceptable excuse.

### Q9. Convincing Team to Adopt OpenTelemetry ⭐⭐⭐
Team using vendor-specific instrumentation (Datadog SDK everywhere). Migration seems expensive. Approach: quantify vendor lock-in cost (if we ever move: months of work). Show OTel benefits: single API, backend flexibility, community momentum. Pilot on one service. Prove it works. Migrate gradually. Don't force big-bang rewrite.

### Q10. The Cost Dashboard Nobody Watches ⭐⭐⭐
Beautiful dashboards built, nobody uses them. Costs still surprise finance monthly. Approach: (1) Understand the workflow — when would someone use it? (2) Push cost data to where decisions happen (Slack alerts, email reports), (3) Weekly cost review meeting with team, (4) Cost as regular part of feature planning. Dashboards alone don't drive behavior; workflows do.

### Q11. Onboarding New Team to Observability Practices ⭐⭐⭐
Team joins from company with weaker observability. Habits from previous job don't transfer. Approach: (1) Not lecture — pair programming through real incidents, (2) Show don't tell — how good observability made debugging 10x faster, (3) Code review culture — every PR checked for observability quality, (4) Runbook contributions expected. Culture is caught, not taught.

### Q12. LLM Judge Bias Discovered ⭐⭐⭐⭐
Discovery: LLM-as-judge is biased toward same-family responses (GPT-4 judging thinks GPT-4 responses are better). Skewing quality metrics. Approach: (1) Cross-family judging (Claude judges GPT, GPT judges Claude), (2) Human calibration on sample (validate judge agreement), (3) Multiple judges + consensus, (4) Track bias explicitly. Don't blindly trust LLM-as-judge — validate.

### Q13. Observability System Went Down During Incident ⭐⭐⭐
The system you use to debug outages is itself down during outage. Approach: (1) Immediate: work with limited info (application logs directly), (2) Systemic: observability infra must be MORE reliable than the systems it observes, (3) Redundant observability paths (Grafana + Datadog for critical services), (4) Practice: run incidents assuming observability partial failure.

### Q14. Explaining Observability to Non-Technical Leadership ⭐⭐⭐
CFO asks: "Why do we need $50K/month observability tools?" Approach: quantify incident value. "Our Q3 outage cost $200K in refunds. Better observability would have detected it 3 hours earlier, containing cost to $30K." Business case in dollars. "It's insurance — we hope not to need it, but when we do, it's the difference between $30K and $200K."

### Q15. The Trace That Reveals a Bug in a Third-Party Library ⭐⭐⭐
Investigation reveals a bug in a widely-used LLM SDK. Approach: (1) Workaround for our systems (patch or version pin), (2) Report to library maintainers with full details, (3) Contribute fix if possible, (4) Communicate to team (avoid similar issues), (5) Track library dependencies more carefully. Good citizenship in open source ecosystem.

### Q16. Observability Costs More Than the Product ⭐⭐⭐
Observability infrastructure costs $80K/month for a product doing $30K/month revenue. Approach: right-sizing — sampling more aggressively, retention shorter, some tools moved to lower tier. Balance observability value vs cost. Free tier customers get less observability, paid tier gets more. Observability is business decision, not just technical.

### Q17. The Migration to New Observability Tool ⭐⭐⭐⭐
Moving from Datadog to Grafana stack for cost. Team resists ("we know Datadog"). Approach: (1) Not "Grafana is better" — "Grafana matches our needs at 1/5 cost", (2) OTel abstraction makes migration easier than expected, (3) Migrate one service first, prove it, (4) Time boxed (6 months), (5) Training + documentation for team. Framework migrations are 80% change management.
