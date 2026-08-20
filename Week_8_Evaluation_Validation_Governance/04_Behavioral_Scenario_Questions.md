# 🎭 Week 8 — Behavioral & Scenario Questions

> **Focus:** Hallucination-caused lawsuits, bias findings after launch, regulatory audits, eval-vs-user disagreements, ground truth disputes, migration crises, convincing leadership to invest in eval
>
> **How to use:** These are the 3 AM crisis scenarios. Practice reasoning out loud — interviewers score HOW you think through them, not whether you have a canned answer.

---

## Q1. The Hallucination That Cost the Company Money ⭐⭐⭐⭐

**Scenario:** Your customer support LLM confidently told a customer they'd get a $500 refund. The customer's post went viral. The company honored the refund. Legal is asking questions. Your CEO wants to know: "How did our AI make up a $500 refund policy?" How do you respond?

**Strong answer:**

"This is a hallucination incident with legal and financial impact. My response has three timeframes:

**Immediate (Hours 0-24):**

1. **Stop the bleeding.** Disable auto-response for anything mentioning money/refunds/credits. Route those queries to human agents until we've fixed the root cause. Better to be slow than to be wrong-and-viral.

2. **Preserve evidence.** Pull the exact conversation logs, retrieved context, model version, prompt version, timestamp. Legal will need this.

3. **Reproduce the failure.** Run the exact query against the exact model version. Does it hallucinate the $500 refund again? If yes, we understand the vector. If no, we have a stochasticity problem.

4. **Communicate up.** CEO gets a factual summary within 4 hours: What happened, our current understanding, containment status, ETA on root cause.

**Diagnosis (Days 1-3):**

Root cause is usually one of:

- **Retrieval failure:** The RAG system pulled the wrong document (maybe an outdated refund policy from 2019 mentioning $500 promos)
- **Generation failure:** Retrieval was correct, but the LLM invented specifics not in the context
- **Prompt weakness:** System prompt didn't strongly enforce "only quote from context"
- **Combined:** Marginal retrieval + weak grounding = confident hallucination

Investigate by:
- Examining what was retrieved for that query
- Testing the model with the exact context on a fixed seed
- Reviewing recent prompt/model changes

**Remediation (Days 3-14):**

1. **Hard fix immediately:** Add explicit guardrail. Regex/classifier detects money-amount promises. Route to human review.

2. **Root cause fix:** Depending on diagnosis:
   - Retrieval fix: audit corpus for stale docs, add freshness filter
   - Grounding fix: strengthen system prompt, add faithfulness check post-generation
   - Systemic fix: add domain-specific evals for money-related queries

3. **Add eval regression:** The exact failing query becomes a permanent regression test. Ships to every future release.

4. **Sample audit:** Look at last 30 days of money-related responses. Any other hallucinations we missed? Proactive customer outreach if so.

**Communication to CEO:**

'On [date], our AI incorrectly promised a customer a $500 refund. Root cause: [specific technical cause]. We've deployed a fix that: (1) blocks any AI response promising money amounts, (2) adds automated eval to catch this class of failure. We've audited the last 30 days for similar incidents — [X other cases, or none]. Long-term: adding stronger grounding checks and expanding red team suite to include financial promise scenarios.'

**What I would NOT say to the CEO:**
- 'AI is unpredictable, this can happen'
- 'The model is a black box, we can't fully control it'
- Vague reassurances without specifics

**The lesson:** Money-related outputs need higher scrutiny than general responses. Systems handling financial commitments need explicit guardrails, not just faithfulness checks."

---

## Q2. Discovering Bias After Launch ⭐⭐⭐⭐

**Scenario:** Six months after launching your AI hiring tool, an external audit reveals that it recommends male candidates 40% more often than equally-qualified female candidates. It's in the news. Regulatory letters are arriving. What do you do?

**Strong answer:**

"This is a critical compliance incident that requires immediate action AND long-term reform.

**Day 1 — Contain:**

1. **Suspend the system.** Not paused with 'we're investigating' — actually stopped for hiring decisions. Users must go back to human-only screening until this is fixed.

2. **Assemble response team.** Legal, HR, ML, product, communications. This is not just an engineering fix.

3. **Preserve everything.** Model versions, training data, evaluation history, all decisions the system ever made. Regulators will want the full paper trail.

4. **External comms.** Coordinated with legal: 'We've identified a critical fairness issue in our AI tool. It's been suspended pending investigation. We're conducting a full audit and will disclose findings.'

**Days 2-14 — Investigate:**

**Root cause analysis (multiple hypotheses to test):**

*Hypothesis 1: Training data bias.* If we trained on historical hiring data, and historical hires were biased, model learned the bias. Test: analyze training data demographics. Very likely true.

*Hypothesis 2: Feature bias.* Model is using proxy features that correlate with gender (e.g., names, career gaps, certain schools). Test: SHAP analysis on features.

*Hypothesis 3: Evaluation blindness.* We evaluated overall accuracy but not fairness across groups. Test: how did we evaluate this? Almost certainly the primary issue.

**The uncomfortable answer:** We probably knew this was possible and didn't test rigorously enough. Own it in the analysis.

**Days 14-60 — Remediation:**

1. **Retroactive impact assessment.** How many hiring decisions were influenced by the biased AI? Identify affected candidates. Legal will guide remediation (potential redress).

2. **Rebuild with fairness first:**
   - Bias-corrected training data (or synthetic data augmentation)
   - Fairness constraints in model
   - Continuous fairness monitoring (not just accuracy)
   - Third-party bias audit before re-launch

3. **New governance:**
   - Fairness metrics gate: no deploy passes without equalized odds within threshold
   - Monthly external fairness audit
   - Diverse review panel for hiring AI decisions

4. **Transparency:**
   - Publish detailed post-mortem
   - Publish model card with fairness metrics
   - Ongoing public fairness dashboards

**Days 60+ — Systemic changes:**

1. **Fairness is now first-class in evaluation.** Every AI system gets fairness testing before deploy, not as an afterthought.

2. **Cross-functional review board.** Legal, ethics, ML, product review all high-stakes AI applications quarterly.

3. **Culture shift.** Bias isn't a technical bug — it's a systemic risk. Everyone treats it that way.

**Communication with regulators:**

Full cooperation. Provide everything they ask for. Show remediation plan. Time-bounded commitments. Regulators respect transparency and action, not defensiveness.

**What I would tell my team:**

'This isn't a technical failure — it's an evaluation failure. We had the tools to detect this. We didn't use them. Going forward, no AI system ships without fairness testing. No exceptions.'

**The lesson:** Bias in AI hiring isn't a hypothetical risk. It's a lawsuit waiting to happen. Any system making decisions about protected classes needs fairness testing from day 1, not as remediation after damage."

---

## Q3. Regulatory Audit Finds Gaps ⭐⭐⭐⭐

**Scenario:** EU regulators are auditing your AI system under the EU AI Act. They ask: "Show us your risk management documentation for the last 12 months." Your team has been building fast — the documentation is thin. What do you do?

**Strong answer:**

"Regulatory audits require honesty, not creativity. Fabricating documentation is worse than admitting gaps. My approach:

**Immediate response to auditors:**

'Thank you for the audit. We can share what we have. I want to be transparent that our documentation practices haven't kept pace with our development velocity. Here's what we have, and here's our plan to close the gaps.'

Note: this is legally advisable. Fabricated documentation is fraud. Missing documentation is a compliance gap that can be remediated.

**Immediate provisioning:**

1. **What we have:** Pull whatever risk management artifacts do exist. Git commits, Slack discussions of risk, meeting notes, incident reports. Not formal, but evidence of consideration.

2. **What we can reconstruct:** From logs, we can show: eval runs, incident responses, mitigation deployments. These constitute risk management even if not documented in the required format.

3. **What's missing:** Formal risk register, DPIA, systematic mitigation tracking. Be honest about these.

**Remediation plan to present:**

'Within 30 days: Formal risk register documented, DPIA completed, existing artifacts organized into required format. Within 90 days: Full compliance with Article 9 (risk management), Article 12 (record-keeping), Article 15 (accuracy and robustness). We'll provide monthly progress updates and welcome verification visits.'

**Internal actions:**

**Week 1:**
- Compile all existing risk-related artifacts (emails, docs, code, logs)
- Formalize into standard template
- Set up ongoing documentation practice

**Weeks 2-4:**
- Formal risk assessment for the AI system
- Data protection impact assessment (DPIA)
- Model card with all required disclosures

**Ongoing:**
- Standing weekly meeting: risk register review
- Every eval run tagged with risk implications
- Every incident linked to risk register updates

**Communication with leadership:**

'We have compliance gaps. The regulators know. We've committed to remediation with clear timelines. Building this documentation is necessary work — I need [team members, budget] to do it right. The alternative is a formal enforcement action that costs more and creates greater risk.'

**What I would NOT do:**

- Backdate documentation (fraud)
- Downplay gaps to regulators
- Blame previous team members
- Promise unrealistic timelines to make it go away

**The lesson:**

Compliance documentation isn't overhead — it's evidence that risk management is happening. In a regulated space, if it isn't documented, it didn't happen from the auditor's perspective. Build the practice into normal work, not as post-hoc scrambling."

---

## Q4. Eval Scores Are Good But Users Complain ⭐⭐⭐⭐

**Scenario:** Your eval metrics look great — 92% faithfulness, 89% answer relevancy, low hallucination rate. But user satisfaction scores are dropping, complaints are increasing. Users say the AI 'feels off' but can't articulate why. What do you do?

**Strong answer:**

"When metrics disagree with users, users are usually right. Metrics are proxies. The real quality signal is user behavior and satisfaction.

**Investigation approach:**

**Step 1 — Talk to users.**
Not surveys — actual conversations. Get 20 users on video calls. Ask them to show you specific interactions where they felt the AI was 'off.' Watch what they see. Their intuition captures things our metrics don't.

**Step 2 — Analyze complaint patterns.**
Categorize the last 100 complaints. What's the actual issue?

Common patterns metrics miss:
- **Tone:** Response is factually correct but robotic, formal, or off-putting
- **Length:** Response is right but too verbose, or too brief
- **Format:** Right info but hard to scan
- **Personality:** Response is technically accurate but feels different from previous interactions
- **Timing:** Feels slow even if latency numbers are fine (due to when users receive it)
- **Missing subtlety:** Correct main answer but misses nuance user cared about

**Step 3 — Correlate complaints with eval results.**
Do the complaint cases score well on our metrics? If yes → our metrics don't capture what matters.

**Step 4 — Identify the metric gap.**
What quality dimension is unmeasured? Common gaps:
- **User experience:** Is the response USEFUL to the user, not just correct?
- **Consistency:** Does the AI behave the same way across sessions?
- **Personality:** Does the AI feel appropriate for our brand?
- **Actionability:** Can the user actually DO something with the answer?

**Step 5 — Build a new metric.**

Example: 'Usefulness' — beyond faithfulness/relevancy. Score: 'Would the user be able to accomplish their goal from this response alone?'

Build LLM-judge for this. Correlate with user satisfaction. If it correlates → adopt as primary metric.

**Step 6 — Root cause the drift.**

Why did quality drop? Common causes:
- Model was updated (silent change from provider)
- Prompt was tweaked (unintended side effects)
- Corpus changed (RAG results shifting)
- User expectations grew (as users learn, they expect more)
- Feature drift (new use cases stress the system)

**Communication approach:**

To engineering team: 'Our metrics are good, but they're not measuring what users care about. We need to expand our eval framework.'

To leadership: 'User satisfaction is dropping despite improving technical metrics. This means our eval is measuring the wrong things. Investing 2 weeks to build user-experience metrics that reflect actual satisfaction.'

**The senior insight:**

Every eval framework has blind spots. When users disagree with your metrics, don't defend the metrics — expand them. The gap between 'what we measure' and 'what matters' is where real quality improvement lives."

---

## Q5. Ground Truth Disputes Between Labelers ⭐⭐⭐

**Scenario:** You're building a golden eval dataset. Your two most experienced labelers systematically disagree on ~20% of examples. Both defend their positions vigorously. Your dataset is going into production. What do you do?

**Strong answer:**

"20% disagreement is a rubric problem, not a labeler problem. When smart, experienced people disagree, the rubric is ambiguous. Fix the rubric.

**Step 1 — Diagnose the disagreement.**

Pull 50 disputed items. Analyze the pattern:
- Are they disagreeing on the same TYPE of case? (e.g., edge cases, ambiguous inputs)
- Are they applying different unstated rules?
- Are they weighting different factors?

Discover WHY they disagree, not just THAT they disagree.

**Step 2 — Have them articulate their reasoning.**

Interview each labeler:
- 'Walk me through 5 cases where you disagreed. What's your reasoning on each?'
- 'What's your mental model for this task?'

Often you'll find they're actually applying DIFFERENT tasks:
- Labeler A: 'Is this response literally correct?'
- Labeler B: 'Is this response what a user would want?'

Both valid — but different tasks. The rubric didn't specify.

**Step 3 — Redefine the rubric with explicit examples.**

Not just 'rate faithfulness 1-5' — but specific examples:
- 'Score 5: Every claim directly stated in context (example: [X])'
- 'Score 4: Every claim inferable from context, no external facts (example: [Y])'
- 'Score 3: Some claims are inferences that might not be intended (example: [Z])'
- ...

Explicit examples anchor labeling far better than descriptions.

**Step 4 — Calibration session.**

Both labelers together, work through the disputed cases. Discuss. Reach consensus. Update rubric based on discoveries. Repeat.

**Step 5 — Measure improvement.**

After rubric refinement, re-label a fresh 50 items. If agreement is now >85%, the rubric is fixed.

**Step 6 — What about the original 20% disputed?**

Options:
- Re-label all disputed items with new rubric (correct approach if practical)
- Mark disputed items in dataset (transparent, allows analysis)
- Remove from dataset (safest — don't train on ambiguous cases)

**Step 7 — Adjudicate remaining disagreements.**

Even with clear rubric, some cases will be genuinely 50/50. That's fine — mark them and either:
- Third labeler tie-breaks
- Mark as 'ambiguous' and exclude from training but keep for eval
- Product decision: which side is safer?

**What NOT to do:**

- 'Just average the scores' — averaging 4 and 2 gives 3, which represents neither labeler's view
- 'Trust the more senior labeler' — dismisses valid disagreement
- 'Add more labelers' — doesn't solve rubric ambiguity, just introduces more noise

**Communication with team:**

'Disagreement isn't a bug in our process — it's information. It's telling us the task is under-specified. We fix it by making the task clearer, not by forcing agreement.'

**The senior insight:**

Inter-annotator agreement is a measure of rubric quality, not annotator quality. High disagreement = fix the rubric. Never proceed to production with a dataset showing < 0.7 IAA. The labels aren't trustworthy."

---

## Q6. The Metrics Improved But Revenue Dropped ⭐⭐⭐

**Scenario:** You launched a new prompt that dramatically improved eval scores (faithfulness 78% → 91%, answer relevancy 82% → 89%). But conversion rate dropped 6% and revenue is down. Data team is asking what happened. What do you do?

**Strong answer:**

"Improved eval scores + declining business metrics = we optimized for the wrong thing.

**Investigation:**

**Step 1 — Correlate specific behaviors.**

What did the prompt change? Compare responses before/after:
- More cautious refusals?
- More disclaimers?
- Different tone?
- Longer/shorter?
- More/less specific?

**Step 2 — Hypothesize why 'better' hurt conversion.**

Common causes:
- **More refusals:** Higher faithfulness might mean more 'I don't have enough information' responses. Users bounce.
- **More hedging:** 'Might be' and 'could be' feel less confident. Users don't trust.
- **Verbosity:** More thorough answers are less scannable. Users lose patience.
- **Overly precise:** Correct but not compelling. 'Yes, we offer premium plans' isn't as engaging as 'Absolutely, let me show you our premium options!'

**Step 3 — Get actual data.**

Segment conversion analysis:
- Query types: which see biggest conversion drop?
- Response length: correlation with conversion?
- User feedback: qualitative themes in complaints/praise?

**Step 4 — The realization.**

Our eval optimized for FAITHFULNESS. Sales/support optimizes for CONVERSION. These aren't the same thing.

A response that's:
- Cautious ✓ (faithfulness)
- Balanced ✓ (relevancy)
- ...but boring ✗ (conversion)

Scores well on our metrics but loses the sale.

**Step 5 — Decide.**

Two paths:

**Path A: Revert.** Old prompt had lower faithfulness but better conversion. Business impact matters. Revert. Faithfulness improvement wasn't worth the revenue loss.

**Path B: Rebuild.** Design a prompt that's BOTH faithful AND engaging. This is harder but better long-term.

Path B requires:
- New metric: 'engagement' or 'conversion likelihood' (measured via A/B)
- Prompt engineering to balance both
- More rigorous evaluation before deploying

**Communication:**

To data team: 'We improved technical quality but hurt business outcomes. Our metrics didn't include business impact. Building that gap into our eval framework so we don't repeat this.'

To leadership: 'Learned an important lesson: technical eval scores aren't a proxy for business value. Investing in business-outcome metrics as first-class in our eval.'

**The senior insight:**

Metrics have unintended consequences. Optimizing for measured metrics can hurt unmeasured outcomes. Always ask: 'If we max this metric, what breaks?' Eval frameworks should include business/UX metrics, not just technical ones."

---

## Q7-Q16: Additional Behavioral Scenarios (Condensed)

### Q7. The Framework Migration Crisis ⭐⭐⭐
Migrating from RAGAS to DeepEval. Metrics compute differently. Numbers change. Team unsure if quality actually changed or just measurement. Approach: run BOTH in parallel for 30 days, calibrate mappings, communicate 'metric change != quality change,' rebuild dashboards.

### Q8. Convincing Leadership to Invest in Eval Infrastructure ⭐⭐⭐
Leadership sees eval as cost, not value. Approach: quantify past incidents that eval would have caught, show competitors' investments, present tiered investment options (minimum viable eval, mid-tier, comprehensive), frame as risk reduction not overhead. ROI: cost of one lawsuit >> cost of eval infrastructure.

### Q9. Debating Human Eval vs LLM-as-Judge ⭐⭐⭐
Team split: some want human eval only (rigorous), others want LLM-judge (scalable). Approach: hybrid — LLM-judge for scale (5-10% sample), human eval for calibration + high-stakes decisions. Show correlation data: LLM-judge with well-designed rubric achieves 0.8+ correlation with humans.

### Q10. The Production-Only Bug ⭐⭐⭐⭐
Bug that only appears in production, not eval environment. Approach: what's different? Real user queries have edge cases, different distribution, different context. Fix: run online eval on production traffic sample, expand eval dataset with production examples, build shadow eval to test changes safely.

### Q11. Vendor Eval Results Conflict With Yours ⭐⭐⭐
Third-party evaluator ranks your AI lower than your internal eval. Approach: understand their methodology, check if they're testing your actual use case, verify their eval dataset quality. If they're right, revise. If they're testing something different, document that. Never dismiss without investigation.

### Q12. The Fired Engineer Took the Golden Dataset ⭐⭐⭐
Key engineer leaves, and the golden dataset that took months to build is on their laptop or in their personal cloud. Approach: prevent via governance from day 1 (datasets in git/S3 with access control, never on personal machines). If it happens: rebuild from scratch, faster this time with lessons learned, treat as security incident with legal review.

### Q13. Eval on Day 1 vs Shipping Fast Debate ⭐⭐⭐
Startup CTO: 'We can't afford eval infrastructure. Ship first, eval later.' Approach: MVP eval (5-10 canonical examples + manual review) can be added in <1 day. Cost of retrofitting eval later is 10x higher. Ship with minimum viable eval, expand over time, but never skip entirely.

### Q14. Explaining Eval to Non-Technical Executives ⭐⭐
CEO: 'Why can't we just look at user feedback?' Approach: analogy — user feedback is like sales calls (small sample, self-selected, delayed). Eval is like quality control on the factory floor (systematic, immediate, catches issues before customers see them). Both matter; eval catches issues before user feedback would.

### Q15. The Compliance Deadline Pressure ⭐⭐⭐⭐
EU AI Act deadline approaching. Your system will be classified 'high-risk.' Team is behind on compliance work. Options: request extension (rarely granted), simplify system to lower-risk tier (may hurt product), work overtime to comply (burnout risk), stop deployment in EU (business impact). Recommend: honest timeline to leadership, tiered plan (comply for critical markets first, expand), get external compliance consultants.

### Q16. The Contradictory Feedback From Different User Segments ⭐⭐⭐
Enterprise users say the AI is too casual. Consumer users say it's too formal. Same eval scores across both. Approach: user segmentation in eval (test with each user type), differentiated prompts per segment, don't average — optimize per segment. Metric: satisfaction PER SEGMENT, not aggregate.
