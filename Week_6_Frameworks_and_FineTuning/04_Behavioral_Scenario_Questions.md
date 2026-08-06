# 🎭 Week 6 — Behavioral & Scenario Questions

> **Focus:** Framework migration decisions, fine-tuning failures, cost crises, team debates, real production incidents
>
> **How to use:** Practice these as 2-3 minute conversational answers. Interviewers watch how you reason, not just what you know.

---

## Q1. The "LangChain vs Raw Code" Team Debate ⭐⭐⭐

**Scenario:** Half your team wants to use LangChain for a new AI feature. The other half wants raw Python + OpenAI SDK. Both sides feel strongly. You're the tech lead. How do you resolve it?

**Strong answer:**

"I'd resolve it with data, not opinions. Here's my approach:

**Step 1 — Understand the actual disagreement.**
Usually the debate isn't 'LangChain vs raw' — it's 'faster development vs debuggability.' Or 'ecosystem features vs code minimalism.' Let each side articulate their concerns.

**Step 2 — Match the choice to the actual requirements.**

Ask concrete questions:
- Does this feature need 5+ integrations glued together (memory, tools, retrieval, callbacks)?
- Will the code be maintained by junior engineers?
- Is debuggability critical (regulated industry, complex workflows)?
- What's the expected iteration speed?

If we need many integrations → LangChain has real value. If it's a single-shot LLM call with structured output → LangChain adds overhead without benefit.

**Step 3 — Build a 2-day prototype in each.**
Same feature, both implementations. Compare:
- Lines of code
- Testing effort
- Debug time when it breaks (deliberately break it)
- Latency
- Team's confidence maintaining it

**Step 4 — Decide based on evidence, communicate clearly.**

The team may not like the answer, but they'll respect a data-driven decision.

**What I wouldn't do:** Choose based on hype (LangChain is popular!), on tribalism (raw code is 'pure'!), or by fiat (I said so). Any of those undermines team trust.

**The senior insight:** LangChain vs raw isn't binary. A common outcome: use LangChain for orchestration (chains, memory, observability) + raw SDK for the LLM calls themselves. Hybrid, not either/or."

---

## Q2. The Fine-Tuning That Made Things Worse ⭐⭐⭐⭐

**Scenario:** You fine-tuned a Llama-3-8B on customer support conversations. On support tasks: 15% better than baseline. But it now fails on general questions ("What is the CEO's name?" → hallucinates). Users complain. What happened?

**Strong answer:**

"Classic catastrophic forgetting. Fine-tuning updated the model to be great at support but degraded general capabilities.

**Diagnosis:**

**1. Check the training data.**
Was 100% of it support conversations? Probably yes — that's the issue. The model learned that ALL queries should be answered like support tickets.

**2. Check the LR and epoch count.**
Aggressive LR (>2e-4) or too many epochs (>3 on 5K examples) causes overwriting of pre-training knowledge.

**3. Check LoRA config.**
Was it high rank (r>32)? Or full fine-tuning? Higher parameter count = more capacity to overwrite general knowledge.

**Fix approaches:**

**Quick fix (this week):**
Revert to base model. Users get slightly worse support responses but working general capabilities. Better user experience than the fine-tuned version.

**Medium fix (this month):**
Redo fine-tuning with:
- **Mixed training data:** 70% support conversations + 30% general instruction-following data
- **Lower LR:** 1e-4 or lower
- **Fewer epochs:** Stop when eval loss plateaus, typically 1-2 epochs
- **Lower LoRA rank:** r=8 or r=16 (less capacity = less forgetting)
- **Add regularization:** LoRA dropout 0.1

**Long-term fix (next quarter):**
Deploy TWO models:
- Fine-tuned model for support queries (routed by classifier)
- Base instruction-tuned model for general queries
- Router decides per query based on classification

This eliminates the trade-off entirely.

**Lesson learned:**
Never fine-tune without measuring on a BROAD eval suite. Support accuracy went up 15%, but MMLU (general knowledge) probably dropped 20%. Only measuring support quality hid the regression.

**What I'd do differently next time:**
Before fine-tuning: define BOTH task-specific evals AND general capability evals. Only promote if both pass thresholds."

---

## Q3. The "Just Use ChatGPT" Executive ⭐⭐⭐

**Scenario:** Your VP asks: "Why are we spending 6 months and $200K building this custom AI system when we could just use ChatGPT?"

**Strong answer:**

"That's a fair question, and I want to make sure we've evaluated it properly. Let me walk through why the custom approach is worth it — or if it isn't, we should know now.

**What we get from ChatGPT (Plus/Enterprise):**
- Best-in-class model quality
- Zero infrastructure work
- $30/user/month

**What we lose:**
- **Data privacy:** Our customer data goes to OpenAI. Enterprise plans give data protection, but we'd still send data externally. Legal team has concerns.
- **Custom knowledge:** ChatGPT doesn't know our products, policies, or internal processes. Users would have to paste context repeatedly.
- **Integration:** ChatGPT is a chat interface. It doesn't integrate with our CRM, ticketing system, or internal tools.
- **Consistent behavior:** Every user gets a slightly different experience. No way to enforce brand voice, safety guardrails, or business logic.
- **Cost at scale:** $30/user × 500 users = $180K/year forever. Our custom system amortizes to less at scale.

**What we're building:**
- RAG over our knowledge base (product docs, policies, FAQs)
- Custom system prompt enforcing brand voice and safety rules
- Integration with our CRM (auto-populate customer context)
- Audit logs for compliance
- Cost per query ~$0.005 = $50K/year at our volume

**The honest question I'd ask myself back:**
Is our differentiation actually meaningful? If our RAG corpus is 50 pages of publicly available info and our system prompt is generic, then yes — ChatGPT would work.

**But if we have significant proprietary knowledge, integration needs, or compliance requirements**, the custom build pays back.

**What I'd propose:** Give me 2 weeks to do a rigorous comparison. Prototype the ChatGPT approach with the same use cases. Compare quality, cost, feasibility. If ChatGPT wins, we save the remaining budget. If we win, executive has evidence to fund the rest."

---

## Q4. The Multi-Agent System That Costs $5K/Day ⭐⭐⭐

**Scenario:** Your team built a CrewAI-powered content generation system. Works beautifully. But it costs $5K/day in LLM fees. Executive says "cut costs 80% without hurting quality." What do you do?

**Strong answer:**

"$5K/day means the crew is making a LOT of LLM calls per task. First, let me understand where the cost is going, then attack it.

**Step 1 — Instrument and measure.**

Break down cost by:
- Agent (which agent's calls cost most?)
- Task type (which content types are expensive?)
- User (any users driving disproportionate cost?)

Usually you find: one agent is doing 10x more calls than expected, one task type is 5x more expensive than average.

**Step 2 — Attack cost, layer by layer:**

**Layer 1 — Model routing (biggest win, 40-60% savings).**
Are all agents using GPT-4o? Do they need to? Research and writing agents may need GPT-4o. But the coordinator/summarizer agent might work fine with GPT-4o-mini.

For a 3-agent crew (researcher/writer/editor): if all use GPT-4o at $15/M tokens, replacing summarizer with GPT-4o-mini ($0.15/M) → 30% cost cut.

**Layer 2 — Reduce inter-agent chatter.**
Multi-agent systems often have agents 'discussing' unnecessarily. Audit: does the researcher really need 3 back-and-forth clarifications with the analyst? Often no.

Simplify handoffs. Structured outputs between agents (JSON not free-form text). Fewer clarification rounds.

**Layer 3 — Cache expensive intermediate outputs.**
If research for 'AI market 2025' has already been done in the last week, reuse it. Only regenerate if user explicitly asks for fresh.

**Layer 4 — Batch and async where possible.**
If we're generating 100 articles, batch API calls (50% discount from OpenAI). Overnight processing for non-urgent tasks.

**Layer 5 — Question the multi-agent architecture itself.**
Honest question: does this task genuinely need multi-agent? Or is it a single-agent task dressed up with 'crew' language?

If a well-prompted single agent with tools does 90% as well at 20% of the cost, multi-agent is overengineering.

**Combined impact:**
Model routing: -40%. Reduced chatter: -15%. Caching: -15%. Batching: -10%. Total: -70-80%.

**What I'd communicate to executive:**
'We can hit 80% cost reduction in 3 weeks. Week 1: instrumentation. Week 2: model routing + reduced chatter. Week 3: caching + batching. Quality gates at each step to ensure no regression.'"

---

## Q5. Migrating from LangChain to Something Else ⭐⭐⭐⭐

**Scenario:** Your team started with LangChain. It's grown into a 50K-line codebase. LangChain breaks on every version bump. Team wants to migrate. Where do you start?

**Strong answer:**

"Framework migrations are always harder than they look. Before jumping to 'let's rewrite,' I'd ask:

**Step 1 — Is the pain real or perceived?**

- **Real pain:** Breaking changes cost 2+ engineer-weeks per release, features blocked on framework limitations, framework abstractions actively harming code quality.
- **Perceived pain:** 'Some people say LangChain is bad on Twitter.'

If it's real pain, migrate. If perceived, pin the version and move on.

**Step 2 — Identify what's actually painful.**

Not all LangChain is bad. Common actual pain points:
- Agents (LangChain agents → migrate to LangGraph)
- Memory abstractions (often overengineered → replace with Redis)
- Callbacks system (leaky abstraction → replace with OTel)

Individual components can be replaced without touching others.

**Step 3 — Migration strategy: incremental, not big-bang.**

**Never rewrite from scratch.** That path leads to a 6-month project that ships nothing until it does — and then it's buggier than what it replaced.

Instead:
1. **Wrap LangChain calls** in your own abstraction layer. All app code calls YOUR interfaces, not LangChain directly.
2. **Rewrite ONE feature** at a time behind the same interface. Users don't notice.
3. **Prove out the new stack** on a low-risk feature first (internal tool, not customer-facing).
4. **Iterate.** Each migrated feature makes the next easier.

Timeline: 6-12 months for a 50K-line codebase, done incrementally, without a shipping freeze.

**Step 4 — What to migrate TO?**

Options:
- **Raw SDK + custom orchestration:** Maximum control, requires building custom infrastructure
- **LangGraph:** For agent-heavy workloads, ecosystem overlap with LangChain
- **LlamaIndex:** If RAG-heavy
- **Mix:** Different frameworks for different parts

I'd probably keep LlamaIndex for RAG, migrate agents to LangGraph, and use raw SDK for simple chains. Not a single-framework replacement — the right tool per workflow.

**Step 5 — Communicate to team and stakeholders.**

Team: 'Here's the migration plan, timeline, and how it affects your work.'
Stakeholders: 'This won't slow down feature delivery — we're refactoring alongside new features.'

Trust erodes when migrations become their own multi-month projects with no visible progress."

---

## Q6. The Quantization Quality Cliff ⭐⭐⭐

**Scenario:** You quantized your fine-tuned model from FP16 to INT4 to reduce serving costs. Perplexity looks fine on standard benchmarks. But users report the model is 'dumber' — misses subtle context, gives shorter answers. What's happening?

**Strong answer:**

"Perplexity is a general metric. It measures average token prediction quality. But specific capabilities (nuanced reasoning, subtle context understanding) can degrade while average perplexity looks fine.

**Diagnosis:**

**1. Which quantization did you use?**
- bitsandbytes 4-bit: usually decent quality
- GPTQ 4-bit: some quality loss, especially on reasoning
- AWQ 4-bit: best-in-class 4-bit quality
- 4-bit anything: still significant loss vs FP16

**2. Did you calibrate on representative data?**
GPTQ and AWQ need calibration data. If you calibrated on generic text but your task is customer support conversations, the quantization is optimized for the wrong distribution.

**3. Which layers are quantized?**
Some techniques quantize everything; some keep certain layers (embedding, output) in higher precision. Full quantization loses more nuance.

**Fixes to try:**

**Fix 1: Switch quantization method.**
If using GPTQ or bitsandbytes → try AWQ. Better quality-to-compression ratio.

**Fix 2: Use larger quantization (INT8 not INT4).**
INT8 keeps ~99% quality. INT4 loses more. If cost allows, INT8 is safer.

**Fix 3: Recalibrate with domain data.**
Recalibrate the quantized model using YOUR domain's conversations, not generic Wiki text.

**Fix 4: Task-specific evaluation, not benchmarks.**
Perplexity says 'fine.' Your task-specific eval (LLM-judged quality on 200 real queries) says 'worse.' The task-specific eval is right. Always evaluate quantized models on YOUR tasks, not generic benchmarks.

**Fix 5: Consider staying at FP16.**
Quantization saves memory and cost. But if it hurts quality noticeably, the trade-off may not be worth it. Sometimes the answer is 'quantization isn't right for this model.'

**The senior insight:**
Quantization has non-uniform quality impact across capabilities. Simple tasks (classification, extraction) barely notice. Complex tasks (multi-step reasoning, subtle understanding) can drop significantly. Always test quantized models on YOUR complex tasks, not just standard benchmarks."

---

## Q7-Q15: Additional Behavioral Scenarios (Condensed)

### Q7. The DPO Training That Made the Model Sycophantic ⭐⭐⭐
After DPO, model always agrees with users, even when they're wrong. Root cause: preference data biased toward 'agreeable' responses. Fix: rebuild preference dataset with 'disagreement done well' examples. Add: eval for calibrated disagreement.

### Q8. The Vendor Lock-In Realization ⭐⭐⭐
Team is 100% invested in LangSmith. Cost is climbing. Migrating would take 2 months. Approach: negotiate enterprise pricing, evaluate Langfuse in parallel, migrate incrementally. Lesson: vendor-neutral observability from day one.

### Q9. Onboarding a Junior Engineer to Frameworks ⭐⭐
Junior joins, feels overwhelmed by LangChain complexity. Approach: start with raw SDK to understand fundamentals, add LangChain when they hit specific pain points. Never start with the framework — start with the problem.

### Q10. The Prompt Engineering Debate ⭐⭐⭐
PM wants CoT everywhere ('reasoning improves quality!'). Data shows CoT hurts simple queries (adds latency, no accuracy gain). Approach: query classification → route simple to zero-shot, complex to CoT. Show data, not opinions.

### Q11. The Fine-Tuning vs RAG Decision Meeting ⭐⭐⭐⭐
Team split: some want to fine-tune, some want RAG. Approach: define the actual failure mode (missing knowledge → RAG; wrong behavior → fine-tune). Frame it as 'what's actually broken' not 'which technique is cooler.'

### Q12. The Multi-Agent Coordination Failure ⭐⭐⭐
CrewAI agents get stuck in infinite loops. Root cause: no max iteration limits, unclear termination conditions. Fix: max_rounds per agent, structured handoff protocols, timeout escalation. Multi-agent needs explicit termination logic.

### Q13. The Framework Version Bump That Broke Everything ⭐⭐⭐
LangChain 0.2 → 0.3 breaking changes take down production. Approach: pin versions in requirements.txt, dependabot with human review, canary deploys, gradual migration. Never upgrade frameworks in production without staging.

### Q14. The MCP Server Security Review ⭐⭐⭐
Security team blocks MCP server deployment: 'It gives the LLM access to our database.' Approach: least-privilege tool permissions, argument validation with Pydantic, tool allow-lists per user, full audit logs, no destructive tools without explicit confirmation.

### Q15. Explaining Fine-Tuning to a Business Executive ⭐⭐
Exec asks 'Why does fine-tuning take 4 weeks?' Approach: analogy (fine-tuning = training an employee for a specific role vs hiring a general contractor), phases (data prep → training → eval → deploy), risks (catastrophic forgetting, regressions). Set expectations before starting.
