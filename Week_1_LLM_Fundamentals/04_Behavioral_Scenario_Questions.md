# 🎭 Episode 1 — Behavioral & Scenario Questions

> **Focus:** Production incidents, stakeholder communication, ethical dilemmas, team dynamics — the "what would you do if…" questions
>
> **How to use:** These test judgment, not just knowledge. Practice using the STAR format (Situation, Task, Action, Result) but emphasize the *reasoning* behind your decisions.

---

## Q1. The Hallucination Incident ⭐⭐

**Scenario:** Your AI-powered customer support bot just told a customer that they're eligible for a full refund on a non-refundable product. The customer has a screenshot. Your manager is asking what happened and what you're going to do about it.

**What the interviewer is really testing:** Incident response maturity. Do you blame the model, or do you own the system?

**Strong answer framework:**

**Immediate (next 30 minutes):**
- Honor the commitment to THIS customer (the company made a promise through our system, even if it was wrong)
- Pull the exact conversation logs: input, retrieved context, model output, and the system prompt active at that time
- Determine if this is a one-off or a pattern — search for similar outputs in the last 24 hours

**Root cause (next 2 hours):**
- Was it a retrieval failure? (Wrong refund policy document retrieved)
- Was it a generation failure? (Right document, but model synthesized incorrectly)
- Was it a prompt failure? (System prompt didn't adequately constrain refund-related responses)
- Was the refund policy ambiguous in a way that confused the model?

**Fix (next 24 hours):**
- Add the failing case to the eval suite
- If generation failure → add output validation: any response mentioning "refund" must be checked against the actual policy before sending
- If retrieval failure → improve chunking/embedding of policy documents
- If prompt failure → update system prompt with explicit refund constraints
- Deploy fix to staging, run eval suite, then canary deploy

**Process improvement (next week):**
- Add a "high-stakes response" classifier — responses about money, legal commitments, or account changes get extra validation
- Implement a confidence threshold for financial topics — below threshold, escalate to human
- Weekly review of AI-initiated commitments

**What NOT to say:** "The model hallucinated, there's nothing we can do about it." That's not engineering — that's helplessness.

---

## Q2. The Cost Explosion ⭐⭐

**Scenario:** You wake up to an alert: your LLM API costs went from $200/day to $3,000/day overnight. Your team hasn't deployed any changes. What do you do?

**Strong answer:**

**Triage (immediately):**
1. Check if it's a billing anomaly or real usage
2. Look at request volume — did traffic spike 15x, or did cost-per-request spike?
3. If cost-per-request spiked: check if someone changed the model routing (e.g., everything routing to GPT-4 instead of GPT-4o-mini)
4. Check for infinite loops — is any request generating runaway tool calls?
5. Set an emergency spend cap if available

**Common root causes:**
- A retry loop is firing on every request due to a transient API error that's now persistent
- Cache layer went down, so 100% of requests hit the API (normally 30% are cached)
- An upstream service started sending much longer inputs (e.g., including full document text instead of summaries)
- A new feature was deployed that accidentally uses the expensive model for all queries
- A bot or adversarial user is sending high-volume requests

**Resolution:**
- Fix the root cause
- Set up billing alerts at 150%, 200%, 300% of baseline
- Implement per-tenant and per-feature cost caps
- Add request-level cost logging so you can identify exactly which feature/user/query type caused the spike

---

## Q3. The Model Update Broke Everything ⭐⭐⭐

**Scenario:** OpenAI updated GPT-4o (a routine update, no version change). Your structured output parsing starts failing on 15% of requests — the JSON is subtly different (different field names, different nesting). How do you handle this?

**Strong answer:**

**This is one of the most common real-world AI engineering problems.** Foundation model providers update models regularly, and "same model name" doesn't mean "same behavior."

**Immediate:**
- Switch to the pinned snapshot version (e.g., `gpt-4o-2024-11-20` instead of `gpt-4o`) — this is why you should ALWAYS pin model versions in production
- If you weren't pinning (lesson learned), implement the fallback: catch JSON parse errors → retry with explicit format instructions → fall back to older model version

**Structural fix:**
- Always pin model versions in production code
- Use structured output enforcement (OpenAI's function calling with strict JSON schemas, or Anthropic's tool use) rather than hoping the model outputs the right JSON
- Pydantic validation on every model output — catch schema mismatches immediately
- Your eval suite should run against the new model version BEFORE you adopt it
- Have a model-version rollback mechanism that takes <5 minutes

**Process:**
- Add "model version change" to your eval trigger list
- Set up a weekly canary test: run your eval suite against the latest model version to detect breaking changes before they hit production
- Document your model version policy: "We update model versions quarterly after eval verification. Emergency rollback takes 5 minutes."

---

## Q4. The Prompt Injection Attack ⭐⭐⭐

**Scenario:** A security researcher reports that they can make your customer-facing AI assistant reveal its system prompt, access internal tool names, and craft responses that impersonate your company making legal commitments. They've published a blog post about it. Your VP of Engineering wants a response plan by end of day.

**Strong answer:**

**Hour 1 — Assess blast radius:**
- Can the injection actually cause financial/legal/safety harm, or is it just embarrassing?
- Is the system prompt itself sensitive? (Does it contain internal API endpoints, business logic, customer data?)
- Can the injection trigger tool calls? (This is the critical question — if the attacker can call tools, it's not just an information leak, it's an action vulnerability)
- Has anyone exploited this in the wild? (Check logs for the patterns described in the blog post)

**Hour 2-4 — Immediate mitigations:**
1. Add input classification layer — deploy a prompt injection classifier (even a regex-based one today, ML-based tomorrow)
2. Add output validation — any response containing system prompt fragments, internal tool names, or legal commitment language gets blocked
3. Restrict tool call permissions — tools should require server-side validation, not just LLM-initiated calls
4. Add rate limiting on suspicious patterns

**Day 2-5 — Structural hardening:**
1. Redesign system prompt to not contain sensitive information
2. Implement privilege separation: the LLM sees a "safe" view of tools, actual tool execution has separate auth
3. Add a "what would a reasonable company response look like?" output classifier
4. Red-team test the new defenses before declaring victory

**Communication to VP:**
"We've confirmed the vulnerability exists but our tools have server-side auth, so the actual blast radius is limited to information disclosure (system prompt) and social engineering (fake commitments). We've deployed input/output filters as an immediate mitigation and have a 5-day hardening plan. We should also reach out to the researcher, thank them, and discuss responsible disclosure."

---

## Q5. The Bias Report ⭐⭐⭐

**Scenario:** Your AI hiring assistant is being used to screen resumes. An internal audit finds that it rates resumes from women 12% lower on average, even with identical qualifications. The tool has been in production for 3 months. What do you do?

**Strong answer:**

**Immediate (same day):**
- Take the tool offline or switch to human-only review immediately. This is a legal and ethical emergency — continued use while aware of bias creates liability.
- Notify leadership, legal, and HR immediately
- Preserve all logs and model outputs for the audit period

**Investigation:**
- Is the bias in the model itself, the prompt, the evaluation criteria, or the training data?
- Run controlled tests: submit identical resumes with only gendered names changed. Measure score differences.
- Check if the bias exists across all job types or only certain roles
- Review what "evaluation criteria" the model was instructed to use — do they inadvertently correlate with gender? (e.g., "years of continuous experience" penalizes career gaps)

**Remediation:**
- If prompt-based: restructure evaluation criteria to be explicitly bias-resistant, blind the model to name/gender indicators
- If model-based: this is a known issue with foundation models. Mitigations include calibration (adjust scores post-generation), blind evaluation (strip identifying information), or moving to structured rubric-based evaluation where each criterion is scored independently
- Run the controlled test again after changes to verify the bias is eliminated

**Process changes:**
- Mandatory bias testing before any AI tool touches hiring, lending, or access decisions
- Regular (monthly) bias audits on production systems
- Diverse eval dataset that tests for demographic fairness

**What the interviewer wants to hear:** You recognize this is not just a technical problem — it's a legal, ethical, and trust problem. You act decisively (take it offline), investigate thoroughly, fix systematically, and put processes in place to prevent recurrence.

---

## Q6. The "Just Make It Work" Pressure ⭐⭐

**Scenario:** Your product manager wants to ship an AI feature by Friday. You've run evals and the accuracy is 72% — your team's minimum shipping bar is 85%. The PM says "good enough, ship it, we'll improve it later." What do you do?

**Strong answer:**

"I understand the urgency, and I want to ship this as much as you do. But let me show you what 72% accuracy means in practice."

*Pull up specific failure cases from the eval suite. Show the PM what bad outputs look like — not abstract numbers, concrete examples.*

"These failures will hit real users. Here's what I propose instead:"

**Option A — Scope reduction:** "We can ship a narrower version that only handles the query types where we're at 90%+ accuracy, and show a fallback message for the rest. We can have that ready by Friday."

**Option B — Confidence gating:** "We add a confidence threshold. High-confidence responses go through automatically. Low-confidence ones show 'Let me connect you with a human' instead of a bad answer. This protects quality while still shipping."

**Option C — Soft launch:** "We ship to 5% of users, monitor quality metrics for a week, and expand as accuracy improves. We learn from real traffic while limiting blast radius."

**What NOT to do:**
- Don't just say "no" without alternatives — that's not collaborative
- Don't just say "yes" and ship bad quality — you'll spend 3x the time fixing production fires
- Don't hide behind process — explain WHY the bar exists with concrete examples

---

## Q7. Explaining LLMs to Non-Technical Leadership ⭐⭐

**Scenario:** Your CEO asks in an all-hands: "Can our AI read and understand all our customer emails?" How do you answer without overpromising or being dismissive?

**Strong answer:**

"Yes and no — let me explain what it can and can't do, because the distinction matters for how we build on it.

**What it can do:** Process the text of every email, extract key topics, sentiment, and requests, route them to the right team, and draft suggested responses. It does this faster and more consistently than any human could at scale.

**What it can't do:** Truly 'understand' in the way a person does. It doesn't know your customers personally, it can't reliably detect subtle sarcasm or cultural nuance, and it can sometimes confidently get things wrong — especially on topics it hasn't seen before.

**What that means for us:** We should use it as a powerful first-pass tool that handles the 70-80% of routine emails, surfaces the important ones to humans faster, and never makes commitments or promises on its own. The combination of AI speed and human judgment is where the real value is."

This answer is honest, avoids hype, gives a concrete capability picture, and ends with a practical recommendation.

---

## Q8. The Data Privacy Question ⭐⭐⭐

**Scenario:** Your team wants to send customer support conversations to the OpenAI API for analysis. Your security team says "absolutely not — customer data cannot leave our infrastructure." The feature is a top priority. How do you navigate this?

**Strong answer:**

**Respect the security team's position first.** They're right to be cautious — customer data sent to third-party APIs may violate GDPR, SOC 2, HIPAA, or your own privacy policy.

**Explore the option space:**

1. **Data Processing Agreements (DPAs):** OpenAI and Anthropic both offer enterprise agreements with data processing guarantees (no training on your data, SOC 2 compliance, data residency). Does this satisfy the security team's requirements?

2. **Azure OpenAI / AWS Bedrock / Google Vertex:** Same models, but hosted within your cloud provider's compliance boundary. Data doesn't leave your VPC. This solves most compliance concerns.

3. **Self-hosted models:** Deploy Llama 3, Mistral, or another open model on your own infrastructure. Fully air-gapped. Trade: lower capability for full data control. May be good enough for classification/extraction tasks.

4. **Data anonymization:** Strip PII (names, emails, account numbers) before sending to the API. Re-attach after processing. This reduces risk but adds engineering complexity and may lose context the model needs.

5. **Hybrid approach:** Use self-hosted models for PII-sensitive tasks (data extraction, classification) and API models for non-sensitive tasks (help article generation, prompt optimization).

"I'd start by exploring Azure OpenAI with a DPA — it likely satisfies security requirements while giving us full model capability. If that doesn't pass review, self-hosted models for the sensitive parts of the pipeline are my fallback."

---

## Q9. The "AI Will Replace Us" Fear ⭐⭐

**Scenario:** You're building an AI tool that automates 60% of a support team's routine work. Team members are worried about their jobs. The team lead asks you directly: "Are you building something that replaces my team?"

**Strong answer:**

Be honest and empathetic — don't BS them with "AI is just a tool" while knowing the headcount implications.

"I want to be straight with you. This tool will handle the routine, repetitive tickets that take up most of your team's day — password resets, order status checks, FAQ responses. That's real work that will be automated.

But here's what it doesn't do: it can't handle escalations, it can't make judgment calls on edge cases, it can't build relationships with upset customers, and it can't improve itself without human feedback.

What I'm building changes your team's work from mostly routine to mostly high-value. Whether that means the team shrinks, stays the same size with higher output, or grows to handle more customers — that's a business decision above both of us.

What I can tell you is that the team members who learn to work WITH this tool — reviewing AI outputs, handling escalations, providing feedback that improves the system — will be far more valuable than before. I'd recommend we build training on AI oversight into the rollout plan."

---

## Q10. Choosing Between Build vs Buy ⭐⭐⭐

**Scenario:** Your company needs an AI document processing system. You can: (A) Build it with LLM APIs + custom pipeline, (B) Buy an enterprise solution (Glean, Unstructured.io, etc.), or (C) Use a managed AI platform (AWS Bedrock, Google Vertex). Your CTO asks for your recommendation.

**Strong answer:**

| Factor | Build (APIs) | Buy (SaaS) | Managed Platform |
|---|---|---|---|
| Time to MVP | 4-8 weeks | 1-2 weeks | 2-4 weeks |
| Customization | Full control | Limited | Moderate |
| Cost at low scale | Low ($) | High ($$$) | Medium ($$) |
| Cost at high scale | Medium ($$) | Very high | Medium ($$) |
| Maintenance burden | High (yours) | Low (theirs) | Medium |
| Data privacy | You control | Vendor dependency | Cloud provider |
| Vendor lock-in | Low | High | Medium |

**My recommendation depends on context:**

- **If we're a startup with <10 engineers and need to ship fast:** Buy. Don't spend engineering time on infrastructure you can purchase. Switch to build later if the vendor becomes a bottleneck.

- **If we're a mid-size company with specific requirements:** Managed platform (Bedrock/Vertex). Good balance of customization and managed infrastructure.

- **If AI is our core differentiator or we have strict data requirements:** Build. The upfront investment pays off in customization and control. But hire for it — this isn't a side project.

"For us specifically, I'd recommend [choice based on company context] because [specific reasoning]. We should commit to a 3-month evaluation period with clear success metrics, and have a pivot plan if it's not working."
