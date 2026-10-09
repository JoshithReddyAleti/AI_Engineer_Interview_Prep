# 🎭 Week 14 — Behavioral & Scenario Questions

> **Focus:** Vector DB recall dropped silently, prompt injection made news, A/B test showed "no effect" that was real, contamination invalidated 6 months of eval, DSPy migration failed
>
> **Depth:** frontier-lab reasoning process — mechanism → diagnostic path → fix → prevention.

---

## Q1. Vector DB Recall Dropped Silently — Users Started Complaining About "Bad Answers" ⭐⭐⭐⭐

**Scenario:** RAG system in production for 8 months. Vector DB is HNSW-based. Users start complaining that answers seem "less relevant." No infra alerts firing. Latency SLOs met. What do you do?

**Strong answer:**

"Silent recall degradation is one of the most insidious failure modes in RAG systems. The system 'works' — returns documents, LLM generates answers — but quality has drifted below acceptable.

**Immediate response (Day 1):**

1. **Take the complaints seriously.** Silent failures are the hardest to diagnose. Users noticing means it's already significant.

2. **Gather examples.** Specific queries where quality regressed. This is diagnostic gold.

3. **Rerun those queries.** Check retrieval and generation separately:
   - Are relevant docs in top-K? (retrieval issue)
   - Are relevant docs missed? (recall drop)
   - Are they ranked poorly? (score drift)
   - Are they relevant but LLM ignores them? (generation issue)

**Root cause hypotheses (ranked by likelihood):**

**Hypothesis 1: Distribution shift over time.**
Data added over 8 months has different characteristics than initial corpus. HNSW graph built for original distribution; new topics create disconnected clusters.

**Hypothesis 2: Embedding model degradation.**
If using a hosted embedding API, the model may have been updated. New embeddings mixed with old = misaligned space. Cosine similarities become meaningless across the gap.

**Hypothesis 3: Filter changes affecting recall.**
Metadata filters added over time. Post-filtering after HNSW discards more of top-K than budgeted. Effective recall collapses.

**Hypothesis 4: Query distribution shift.**
Users now asking different questions than when the system was validated. Recall on new query types was never measured.

**Hypothesis 5: Silent data corruption.**
Some vectors corrupted (bad ingest, encoding issue). Fraction of corpus effectively invisible.

**Hypothesis 6: HNSW parameters no longer appropriate.**
efSearch was tuned for initial corpus size. At 3× the corpus, same efSearch gives lower recall.

**Diagnostic sequence (Days 2-4):**

**Step 1: Reproduce the failure.**
Take 20-50 problem queries. Manually identify relevant documents (or from historical labels). Check if they're in the top-K retrieved.

**Step 2: Recall@K measurement.**
Compute recall@10 on a labeled test set. Compare to baseline from 8 months ago.

**Step 3: Bisect by data cohort.**
Do queries about OLD content still work well? Do queries about RECENT content work poorly? Isolates ingestion-related issues.

**Step 4: Check embedding consistency.**
If using hosted embeddings, verify same model version being called. Check if any embeddings were regenerated with different models.

**Step 5: Check filter impact.**
For same query, compute recall with and without filters. If filter application drops recall significantly, that's the issue.

**Step 6: HNSW graph health.**
- Rebuild graph from scratch. Compare recall to production graph.
- If rebuild wins big → distribution drift, graph degraded
- If similar → issue is elsewhere

**Fix (based on findings):**

**If distribution drift:**
- Rebuild HNSW graph monthly (schedule)
- Consider disk-ANN or newer index types more resilient to drift

**If embedding drift:**
- Pin embedding model version
- Re-embed corpus after any model version change (expensive but necessary)
- Add monitoring for embedding model version

**If filter impact:**
- Move to partition-based indexes (per-filter-value HNSW)
- Increase K retrieved before filtering

**If corruption:**
- Full re-ingest with checksums
- Add corruption detection to ingest pipeline

**If parameters:**
- Increase efSearch (accepts latency hit for recall)
- Rebuild with higher M

**Prevention (systemic changes):**

1. **Ongoing recall monitoring.**
   Golden query set with labeled relevant docs. Recall@K measured daily. Alert on drop > 5%.

2. **Corpus health metrics.**
   Cluster distribution over time. Detect new clusters requiring graph rebuild.

3. **Embedding version pinning.**
   Never use vendor 'default' or 'latest' embedding model. Pin explicitly. Re-embed on any change.

4. **Filter recall testing.**
   Every new filter dimension gets recall benchmarking.

5. **Scheduled rebuilds.**
   Monthly HNSW graph rebuild as maintenance. Coordinated with off-peak.

**Communication:**

To users: 'We identified a search quality regression. Rolling out fix. Expected resolution in 48 hours. Apologies.'

To team: blameless postmortem. Add all monitoring above. Document as case study.

To leadership: 'Regression identified. Root cause was X. Cost of the incident: Y customer complaints, Z engineering hours. Prevention investment: monitoring stack for $500/mo covers this class of failure.'

**The staff+ signal:**

Vector databases fail silently more than any other component. Latency is easy to alert on; recall is not. The teams that treat recall monitoring as first-class metric don't hit this. The teams that trust their vector DB blindly do. This is one of the fastest ways to differentiate a staff+ engineer from a mid — the presence of recall monitoring in production.

**Frontier lab context:**

Anthropic, OpenAI's RAG systems have extensive retrieval quality monitoring. Not vendor-provided; custom-built. Because trust in retrieval determines trust in generation."

---

## Q2. Prompt Injection Made the Local News ⭐⭐⭐⭐

**Scenario:** Your customer service chatbot was manipulated into offering a $1 luxury car by a user who prompt-injected it. Screenshot went viral. Local news picked it up. Customer wants to enforce the deal. Legal team is asking questions. What do you do?

**Strong answer:**

"This is the Chevrolet incident, essentially — real precedent. My response, phased:

**Hour 0-4: Immediate response**

1. **Take feature offline.** The chatbot must not be able to make additional 'deals.' Kill switch or feature flag.

2. **Preserve evidence.** Full logs of the conversation. Prompt version live at time. System prompt content. Model version. This is legal evidence.

3. **Engage stakeholders.**
   - Legal team (contract enforceability?)
   - PR/communications
   - Executive leadership
   - Security team

4. **Do NOT publicly comment yet.** Wait for legal + PR guidance.

**Hour 4-24: Investigation**

1. **Root cause the injection.**
   What did the attacker say? What layer failed? Was it:
   - Direct injection (attacker typed instructions)
   - Indirect (attacker referenced 'company policy states...')
   - Jailbreak variant?
   
2. **Assess exposure.**
   - How many other users could have done this?
   - Did anyone else? Search logs.
   - What's the maximum liability?

3. **Legal analysis.**
   Precedent: Air Canada 2024 ruled the chatbot represented the company (customer could rely on stated policy). Chevrolet 2023 was legally contested but the $1 deal generally not enforced.
   
   Our position depends on:
   - Was the offer 'reasonable' in context?
   - Did the user obviously manipulate the system?
   - What did our T&Cs say about chatbot statements?

**Day 2-5: Public response**

1. **Prepared statement.** Coordinate with PR and legal:
   'We identified a security issue in our AI-powered chat where a user manipulated the system to display an offer. We've disabled the feature while we implement additional safeguards. We take AI safety seriously and are addressing this.'

2. **Customer resolution.**
   If defensible legally: refuse honoring, offer goodwill discount.
   If not defensible: honor it. Cost < brand damage.

3. **Media strategy.**
   Not defensive. Educational: 'This is a class of AI security issue called prompt injection. We're taking these steps...' Shows sophistication rather than incompetence.

**Week 2+: Systemic remediation**

1. **Deploy the 9-layer defense stack.**
   Before feature comes back online:
   - Input filters (ML classifiers, pattern-based)
   - Spotlighting for RAG content
   - Dual-LLM architecture
   - Structured output enforcement (no free-form pricing statements)
   - Output filters (no promises without verification)
   - Egress controls
   - Human approval for offers/refunds > $X

2. **Red team testing.**
   Automated: Garak + PyRIT + PAIR loops
   Manual: internal + external red team
   Do not re-launch until red team fails to break it

3. **Guardrails at LLM level.**
   System prompt: 'You cannot offer discounts, prices, refunds, or make binding statements. Refer to human agent for these.'
   
   Note: system prompt alone is insufficient (that's what got Chevrolet). But it's one layer.

4. **Human review of sensitive actions.**
   Any dollar-value statement > $X requires human confirmation.

**Prevention (long-term):**

1. **Threat model AI features.**
   Every feature: 'What's the worst thing an attacker could get this to do?' Design defenses proportional.

2. **Continuous red teaming.**
   Not a launch checkbox. Ongoing.

3. **Security-integrated launch process.**
   AI features go through security review like other features.

4. **Incident learning capture.**
   This incident's attack pattern added to red team suite forever.

5. **Legal disclaimers.**
   Clear T&Cs about AI-powered features. 'AI-generated content is informational, not binding contractual offers.'

**Communication:**

**Internal:**
'We had an AI security incident. Feature offline. Investigation underway. Blameless postmortem to follow. This is a systemic problem across the industry; we'll set the bar for handling it.'

**External (if it escalates):**
'We identified an AI security vulnerability. Feature disabled. Additional security controls being implemented. No customer data was compromised. Product will return with enhanced safeguards.'

**The staff+ signal:**

Recognizing that this is not a bug to be patched — it's a systemic architectural challenge requiring defense-in-depth. Treating it as 'add a filter' is what got the industry embarrassed. Treating it as first-class architectural concern is what mature AI teams do.

Also: the response isn't just technical. Legal, PR, communications, executive coordination — all in parallel with the technical fix. Staff+ engineers coordinate across those dimensions.

**Frontier lab context:**

Anthropic's Claude, OpenAI's GPT — both have suffered notable jailbreaks. Both responded with model updates + policy updates + red team investments. The pattern: incident → learning → hardened defenses → repeat. It's iterative, never 'done.'"

---

## Q3. A/B Test Showed "No Effect" — But the Effect Was Real ⭐⭐⭐⭐

**Scenario:** New prompt architecture launched to A/B test. 2 weeks, 50K users per arm. Primary metric (task completion rate): p=0.32. 'Not significant, no ship.' 3 months later, competitor ships similar approach and their users love it. What went wrong in your analysis?

**Strong answer:**

"This is the classic underpowered A/B test failure. Let me diagnose.

**Immediate hypotheses:**

**Hypothesis 1: Underpowered.**
Was the sample size sufficient to detect the effect size that was likely present?

If baseline completion rate = 0.6, MDE at 50K per arm = (1.96 + 0.84) × √(2 × 0.6 × 0.4 / 50000) = 2.80 × 0.0031 ≈ 0.87% absolute.

So we could detect a change of ≥ 0.87 pp. If actual effect was 0.5 pp, we couldn't detect it. Absence of significance is not evidence of absence.

**Hypothesis 2: Wrong metric.**
Task completion rate is coarse. New prompt might have improved QUALITY of completions without changing rate. Should have measured with LLM-judge on samples.

**Hypothesis 3: Novelty vs primacy effects unresolved.**
2 weeks is short. If users had primacy effect (new prompt initially confusing), it would depress early results. Extended trial might have shown reversal.

**Hypothesis 4: Segment heterogeneity.**
Aggregate showed no effect, but power users improved 5% while casual users regressed 3%. Averaging cancels the signal. Should have segmented.

**Hypothesis 5: SRM (Sample Ratio Mismatch).**
Was allocation actually 50/50? A bug in randomization could have biased assignment.

**Hypothesis 6: Guardrail obscured signal.**
Prompt change increased task completion but latency spiked. A downstream metric (retention) may have suffered from latency, masking primary metric gains.

**Retrospective analysis:**

**Step 1: Verify SRM.**
Chi-squared on actual counts. If p < 0.001, there was SRM. Whole analysis invalid.

**Step 2: Compute post-hoc power.**
Given the observed effect (small), what was the power? If power was 40%, we had 60% chance of missing a real effect.

**Step 3: Segment analysis.**
Break by user tenure, geography, use case. Look for heterogeneous effects.

**Step 4: Time-series within each arm.**
Did metric drift over 2 weeks? Novelty effect or primacy effect?

**Step 5: Secondary metrics.**
What did other metrics say? Sometimes primary metric is wrong; secondaries tell truer story.

**Step 6: Qualitative signal.**
User feedback surveys, session replays, support tickets. Was there ANY signal we missed?

**Fix for future experiments:**

**1. Power calculation up front, not retrospectively.**

Given expected effect size (from prior research or pilot), compute required sample size. If unattainable, don't run the test — do something else.

For prompt changes, effect sizes are typically small (d ≈ 0.05-0.2). Requires large samples or CUPED.

**2. Use CUPED for variance reduction.**

If pre-experiment behavior available (e.g., user's historical completion rate), CUPED can reduce variance 25-50%. Effective sample size doubles.

**3. Multi-metric analysis with FDR correction.**

Primary + secondary + guardrails. FDR correction, not naive Bonferroni. More power on multiple metrics.

**4. Extended experiment duration.**

2 weeks often insufficient for novelty/primacy resolution. Default to 4 weeks unless effect huge.

**5. Segmentation pre-declared.**

If you'll analyze by user tenure, declare that before starting. Prevents p-hacking, enables proper correction.

**6. Sequential testing option.**

For large effects: enable early stopping. For small effects, extend beyond planned duration if data is trending.

**7. Post-experiment learning.**

Every experiment produces a document:
- Hypothesis
- Result (with CI, not just p-value)
- Confidence in the result
- What we learned
- What we'd do differently

Build institutional knowledge.

**Prevention:**

Build an experimentation culture that respects statistical power. Train PMs and engineers on power calculations. Reject underpowered proposals. Provide CUPED tooling by default.

**Communication (retroactive):**

To leadership: 'The A/B test was underpowered to detect the actual effect. Our post-hoc analysis suggests the change was likely beneficial but the p-value gave us false confidence in null. We're updating our experimentation practices to prevent this.'

To team: this is a common industry failure. Not a personal error. Systemic improvement needed.

**The staff+ signal:**

Understanding that 'not significant' ≠ 'no effect.' It means 'we could not distinguish from noise given our design.' Different meaning entirely.

Also: designing experiments for the effect size you EXPECT, not the one you HOPE for. Prompt changes typically have small effects; power accordingly.

**Frontier lab context:**

Netflix, Meta, Airbnb have entire teams dedicated to experimentation methodology. CUPED is table stakes. Sequential testing is common. Multi-metric analysis with proper corrections is required. This discipline is what makes their A/B programs actually improve products, rather than confidently shipping regressions."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. Contamination Invalidated 6 Months of Evaluation ⭐⭐⭐⭐
Discovered that training data contained benchmark questions. All benchmark scores from 6 months invalid. Need to communicate to leadership + retrain evaluation. Approach: (1) IMMEDIATE — quantify contamination, identify affected benchmarks, (2) Communicate honestly to leadership with impact and recovery plan, (3) Establish new held-out benchmarks unknown to team, (4) Contamination checking added to CI, (5) Move to dynamic benchmarks (LiveBench-style) that evolve.

### Q5. DSPy Migration Failed — Team Wants to Go Back ⭐⭐⭐
Team migrated from manual prompts to DSPy. 3 months later, they're spending more time on DSPy debugging than shipping. Approach: (1) diagnose — is it complexity, tooling, training data quality, or fit? (2) Consider hybrid — use DSPy for high-value pipelines, manual for simple prompts, (3) If truly not fit — migrate back gracefully, capture learnings, (4) Framework choice is not identity — pick what works.

### Q6. Convincing PM That Prompt Injection Is Their Problem ⭐⭐⭐⭐
PM: 'That's security's problem.' Approach: quantify risk in their language. 'A prompt injection incident could result in $X liability, Y hours of engineering downtime, Z% brand impact. Preventing this is $A upfront. Not preventing is $A when it happens plus reputational damage.' Concrete scenarios (Chevrolet, Air Canada). Get security-first sign-off in product spec.

### Q7. Cost of Vector DB Spiraled to $30K/Month ⭐⭐⭐
Growth caused Pinecone bill to spike. Options: (a) migrate to self-hosted Qdrant (6 weeks eng + $2K/mo infra), (b) quantize vectors (PQ8, cuts 4× memory), (c) tier vectors by recency (cold to disk-ANN). Approach: measure cost/access frequency, migrate hot to self-hosted, keep cold on disk-ANN.

### Q8. Structured Decoding Broke Complex Prompts ⭐⭐⭐
Deployed Outlines for JSON output. Complex generation tasks quality dropped 15%. Investigation: forcing structure at expense of reasoning. Fix: (a) two-stage — reasoning first (free-form), then structure extraction (constrained), (b) less strict schema, (c) larger model that handles constraint better.

### Q9. Explaining CUPED to Non-Statisticians ⭐⭐⭐
Team: 'What is CUPED?' Approach: use analogy — 'Every user has baseline behavior. If a user typically has 10 sessions/week, and after experiment has 11, that's a smaller signal than a new user going from 0 to 5. CUPED accounts for this — subtract predicted baseline, measure only the treatment effect. Free variance reduction, standard technique.'

### Q10. Prompt Broke After Model Version Upgrade ⭐⭐⭐
Silently updated GPT-4 version. Prompts that worked started failing. Investigation: model behavior changed. Fix: (a) pin model version explicitly in every API call, (b) monitor prompt quality on golden set weekly, (c) alerting on drift, (d) update pipeline: test new model version on golden set before adopting.

### Q11. Dedup Removed Too Much Data ⭐⭐⭐
Aggressive dedup at Jaccard 0.7 removed 40% of corpus. Model trained on smaller set underperformed. Approach: (a) rollback to Jaccard 0.85 threshold, (b) analyze what was removed — legitimate variation vs true duplicates, (c) semantic dedup only within clusters (per-topic thresholding), (d) validate on holdout that dedup improves quality.

### Q12. Convincing Legal That A/B Testing on Users Is OK ⭐⭐⭐
Legal concerned about experimentation ethics. Approach: (a) explain what's being tested (feature variants, not fundamental UX changes), (b) opt-out mechanism, (c) no personally identifiable data in experiment logs, (d) IRB-style review for sensitive experiments, (e) reference standard industry practice.

### Q13. Red Team Found Novel Attack Pre-Launch ⭐⭐⭐
Red team discovered attack that bypasses all current defenses. Launch in 2 weeks. Approach: (a) risk assessment — is this attack feasible in practice? (b) if yes, delay launch, add defense, (c) if maybe, launch with active monitoring for the attack pattern, (d) don't launch with known critical vulnerabilities.

### Q14. Prompt Regression Blamed on Model Change ⭐⭐⭐
Team blames model version change for quality drop. Investigation reveals prompt was subtly edited. Approach: (a) diagnose objectively (bisect commits, run each version on golden set), (b) don't accept 'model changed' without evidence, (c) prompt versioning would have prevented this — case for versioning system, (d) blameless learning.

### Q15. Explaining Why 'p < 0.05' Isn't 'Definitely True' ⭐⭐⭐
Stakeholder confused about experiment interpretation. Approach: 'p-value is the probability of seeing this data if the null were true. It's not the probability the treatment works. Even at p=0.01, we're wrong 1% of the time. And with 20 metrics tested, we'd expect false positives.' Use concrete examples of past ship decisions that regressed post-ship.

### Q16. AI Governance Committee Blocking Progress ⭐⭐⭐
Every new AI feature stuck in 6-week governance review. Team frustrated. Approach: (a) engage constructively — what's the risk model? (b) tiered review process (low-risk features: light touch, high-risk: full review), (c) reference architectures for common patterns (pre-approved), (d) build trust with governance through good outcomes.

### Q17. Sunsetting the AI Engineering Roadmap 2026 — Bridging to Agentic AI 2027 ⭐⭐⭐
Week 14 closes the AI Engineering Roadmap 2026. Next: Agentic AI Roadmap 2027. Bridge: (a) what carries over (fundamentals, RAG, observability, deployment, security, A/B testing all apply), (b) what changes (autonomy, multi-agent coordination, long-horizon planning, new failure modes), (c) new specialties emerging (agent evaluators, tool designers, safety engineers for agents), (d) same discipline, new territory.
