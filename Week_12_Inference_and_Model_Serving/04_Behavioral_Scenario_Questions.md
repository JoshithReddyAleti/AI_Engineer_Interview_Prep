# 🎭 Week 12 — Behavioral & Scenario Questions

> **Focus:** $87K/month API bill, GPU shortage crisis, quantization broke quality, KV cache OOM in production, tensor parallelism deadlock, cold model swap disaster
>
> **How to use:** These are the war stories that hiring managers use to see if you've actually operated inference at scale. Reasoning process > final answer.

---

## Q1. The $87K/Month OpenAI Bill Made Self-Hosting a CFO Question ⭐⭐⭐⭐

**Scenario:** Your CFO calls: "Our OpenAI bill hit $87K last month. Board wants it under $30K. You have 90 days. Can you self-host?" What do you do?

**Strong answer:**

"This is a strategic decision framed as a technical one. My response, phased:

**Week 1 — Instrument and understand:**

Before touching hardware, understand the current spend:
- Break down by feature: which endpoints cost the most?
- Break down by user: is one tenant abusive?
- Break down by request type: prompt-heavy or output-heavy?
- Break down by model: GPT-4o vs GPT-4o-mini share?
- Cache-hittable ratio: how much traffic could semantic cache absorb?

Common revelation: 20-40% of spend is on requests that could use smaller models. Fixing that alone recovers half the target.

**Week 2 — Analyze options:**

Path A: **Optimization without self-hosting** (fastest path if possible)
- Semantic cache (30-40% cost reduction possible)
- Model routing (cheap model for simple queries, save 40-50% on those)
- Prompt compression (remove unnecessary tokens)
- Prompt caching (OpenAI's own prompt caching, 50% off cached input)

If Path A can hit $30K target: DO THAT. Self-hosting is expensive engineering. Don't over-engineer.

Path B: **Self-host if optimization isn't enough**
- Choose open model matching quality bar (Llama-3.1-70B typical)
- Run break-even math (I showed this in the flagship question)
- 4×H100 or 8×H100 spot instance
- vLLM + INT4 AWQ + FP8 KV + APC + speculative decoding
- Realistic savings: 60-70% cost reduction

**Week 3-4 — Prove the concept:**

Build baseline benchmark:
- Golden set of 500 real production queries
- Run through self-hosted model in shadow mode
- Compare quality (LLM-judge, exact match, user preference on sample)
- Measure real cost per token achieved

If quality holds and cost math works: proceed.

**Week 5-8 — Migration:**

- Gradual rollout: 5% → 25% → 50% → 100%
- Golden set eval at each stage
- Cost dashboard tracked daily
- API remains as fallback

**Week 9-12 — Optimize:**

- Fine-tune based on production patterns
- Semantic cache warming
- Prompt engineering for smaller model
- Weekly cost review

**Communication with CFO:**

Week 1: 'Investigating. Will present options in 2 weeks.'
Week 2: 'Recommendation: Path A (optimization) if achievable, Path B (self-host) as backup. Projected outcome: [X].'
Week 4-8: monthly updates with actual cost tracking
Week 12: 'Achieved $[actual] vs $30K target. Details: [breakdown].'

Executives want confidence, not commits. Don't promise 60% cost reduction until you've proven it in shadow mode.

**What I would NOT do:**

- Immediately buy H100s (research and prove first)
- Migrate without golden set (quality regression will hurt more than cost saved)
- Promise the CFO 65% savings on day 1 (build credibility first)
- Skip API fallback (single point of failure)
- Underestimate engineering time cost (self-hosting eats 30-50% of an engineer's time indefinitely)

**The senior insight:**

The right answer often isn't 'self-host.' It's 'diagnose the cost driver, fix the biggest lever first.' Sometimes that's semantic cache. Sometimes it's model routing. Sometimes it's a bug (one tenant burning most of the budget). Self-hosting is a big commitment; only make it when the math clearly wins."

---

## Q2. The GPU Shortage That Broke Our Launch Plan ⭐⭐⭐⭐

**Scenario:** You planned to launch on 32×H100. Cloud provider says: 'H100 out of stock in your region for 3 months.' Launch is in 4 weeks. What do you do?

**Strong answer:**

"This is a common 2025-2026 reality. My response, phased:

**Hour 0-24: Assess options**

Option 1: **Different region**
- Check H100 availability in other regions (Europe, Asia).
- Cost: data latency for users, compliance risk if crossing borders.

Option 2: **Different provider**
- AWS, GCP, Azure, CoreWeave, Lambda Labs, Runpod all sell H100.
- Multi-cloud complexity. Contract negotiations.

Option 3: **Different hardware family**
- H200: newer, better for decode, similar availability. Try.
- A100 80GB: older, less bandwidth, but often available. 25-40% throughput drop.
- L40S: 48GB HBM, moderate bandwidth. Good for smaller models.
- B200: newest, likely worse availability.
- MI300X (AMD): 192GB HBM, HUGE memory. Very hot alternative. Different software stack (ROCm vs CUDA).

Option 4: **Different model size**
- Downsize to Llama-8B: doesn't need H100. Runs on L40S or even A10. Quality trade-off.
- Or two-model strategy: 8B for most, 70B only for hard queries.

**Day 2-7: Prototype the alternatives**

Rapid benchmarking on 2-3 most viable options. Real numbers, not vendor claims.
- Cost per token achievable
- p95 TTFT achievable
- Quality on golden set
- Availability confirmed for launch quantity

**Day 8-14: Make the call**

Best case: multi-region H100 works. Extra networking cost, minimal architecture change.

Realistic case: mix of A100s and H200s from different providers. Multi-cloud LLM gateway routing.

Worst case: 8B model with fine-tuning to close quality gap. Ship, revisit when H100 supply recovers.

**Day 15-28: Execute**

Multi-cloud deployment:
- vLLM on each cluster
- LLM gateway routes by health/cost/latency
- Observability aggregates across clouds
- Terraform for reproducible infrastructure

**Communication:**

To leadership: 'GPU shortage requires plan B. Options analyzed: [table]. Recommendation: [choice]. Impact: [cost, latency, timeline].'

Don't hide it. Executives handle bad news better than surprises.

To engineers: 'H100 shortage. We're going multi-cloud on A100+H200. Different network topology. Complexity increases. Bring focus.'

**The senior insight:**

GPU supply is a first-class engineering constraint in 2025-2026. Architect for portability from day one. Same vLLM on H100, A100, MI300X. Never fully lock into one hardware family. The teams that survived the H100 shortage were the ones who could switch to MI300X on 30 days notice. That's a real capability."

---

## Q3. Quantization Broke Our Quality Silently ⭐⭐⭐⭐

**Scenario:** You deployed INT4 quantization to double throughput. Metrics look great. Two weeks later, enterprise customer complains: 'Your model got dumber. It used to be sharp; now it hedges everything.' Investigation shows quantization introduced subtle quality regression on their query type. What do you do?

**Strong answer:**

"This is the silent failure mode of quantization. Not catastrophic quality loss (that we'd catch). Subtle degradation on specific query types.

**Immediate response (Day 1):**

1. **Take the customer seriously.**
   'We're investigating. I'll have specifics within 48 hours.' No dismissal.

2. **Get their examples.**
   Specific queries where quality regressed. This is the diagnostic gold.

3. **Reproduce.**
   Run their examples through both quantized model and FP16 baseline.
   Confirm the regression.

**Investigation (Days 2-5):**

If quality regressed:
- Was it category-specific? (Legal reasoning weaker? Math worse? Style shift?)
- How large is the effect on their production traffic?
- Does it affect other customers?

Extended golden set:
- Our original golden set didn't catch this = wasn't representative enough
- Add customer's query patterns to golden set
- Re-run FP16 vs INT4 comparison across expanded set

**Root cause of quantization failures:**

- **Outlier weights** — some weights are much larger than others; naive quantization clips them, losing important information. Fix: AWQ (activation-aware) preserves outliers.
- **Activation distribution** — some layers have activations that don't fit INT4's dynamic range. Fix: mixed precision (keep sensitive layers at FP16).
- **Long-context reasoning** — quantization compounds through decoding. Long chains of reasoning degrade more than short answers.
- **Domain shift** — the calibration data used for quantization may not match this customer's distribution.

**Remediation options:**

Option 1: **Roll back to FP16.**
- Cost goes back up. Customer happy immediately.
- Not sustainable if cost was the reason for quantization.

Option 2: **Roll back on their traffic only.**
- Route their tenant to FP16 model, keep INT4 for others.
- Feasible with multi-tenant gateway.
- Enterprise tier might justify this.

Option 3: **Re-quantize with better calibration.**
- AWQ instead of naive quantization
- Include their query patterns in calibration data
- Test on expanded golden set before redeploy

Option 4: **Mixed precision.**
- Keep first/last layers in FP16 (they're more sensitive)
- Middle layers INT4
- Less throughput than pure INT4, still significant savings

Option 5: **Weight-only quantization but not KV cache.**
- Sometimes KV quantization is the real culprit
- Try INT4 weights + FP16 KV

**Communication with customer:**

'You were right. Our quantization introduced a regression on [specific query types]. Root cause: [technical explanation at their level]. Our fix: [option chosen]. Timeline: [X days]. In the interim: [temporary mitigation].'

Ideally: offer to run FP16 on their traffic while you fix, at no extra cost. Buys goodwill.

**Systemic changes:**

1. **Expanded golden set** — include diverse query types, edge cases, customer-specific patterns
2. **Per-tenant quality tracking** — not just aggregate scores, per-tenant scores
3. **Gradual rollout for quantization** — 5% → 25% → 50% → 100%, monitor per-tenant metrics at each stage
4. **Longer bake time** — 2-week bake before declaring quantization safe
5. **Rollback plan documented and tested** — automated one-command rollback

**Blameless postmortem:**

- Not 'someone deployed bad quantization'
- Not 'we should have tested more'
- Yes: 'our eval methodology missed this class of regression. Here's what we're adding to catch it next time.'

**What I would NOT do:**

- Dismiss customer feedback ('metrics look fine to us')
- Rush a fix without understanding root cause
- Redeploy fixed version to everyone at once
- Hide the incident (worse when discovered later)

**The senior insight:**

Quantization is subtly failure-prone. Aggregate quality metrics can look identical while specific query types regress. Enterprise-grade quantization requires: multi-dimensional quality evaluation, per-tenant tracking, gradual rollout, easy rollback. The teams that treat quantization as 'set and forget' hit this incident. The teams that treat it as 'ongoing quality management' don't."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. KV Cache OOM Under Traffic Burst ⭐⭐⭐⭐
Traffic spike caused KV cache to hit capacity. Requests started failing. Investigation: long-context requests filled cache faster than budgeted. Fix: (1) immediate: shed load, alert, restart, (2) short-term: cap max context per request, (3) systemic: better admission control (reject long-context if KV low), preemption support (drop lower-priority requests to make room), FP8 KV to double capacity, monitoring KV utilization as SLO.

### Q5. Tensor Parallelism Deadlock at Deploy Time ⭐⭐⭐
Cluster startup: TP=8, one GPU stuck initializing. Others waiting on NCCL barrier. Everything froze. Approach: (1) timeout on NCCL init, (2) explicit health check per GPU before starting model, (3) automated diagnostic run before serving traffic. Root cause: a GPU had memory fragmentation from previous crash. Prevention: verified startup script that flushes GPU memory + confirms full-mesh NVLink health.

### Q6. Model Version Drift From "Stable" Alias ⭐⭐⭐⭐
Using `gpt-4-turbo` alias. OpenAI silently updated behavior. Quality shifted. Users complained. Approach: (1) immediate: pin to specific version (`gpt-4-0125-preview`), (2) systemic: probing queries weekly to detect model drift, (3) alternative providers as fallback if provider changes we don't like, (4) test all provider updates on our golden set before adopting.

### Q7. Cold Model Load Killed a Product Launch ⭐⭐⭐
Launch day. Traffic hit new endpoint. Model needed to load (3 min for 70B). Users saw errors. Investigation: no warm pool. Fix: (1) warm pool with min replicas ≥ 2, (2) predictive scaling ahead of traffic, (3) launch checklist requires warm pool verified, (4) load test at expected launch traffic in staging.

### Q8. Speculative Decoding Made Things Slower ⭐⭐⭐⭐
Deployed speculative decoding. Latency went UP. Investigation: our workload had high randomness (creative writing). Draft model's guesses rejected ~85% of the time. Overhead > speedup. Fix: (1) rollback for this workload, (2) selective speculation: only enable for query types with high acceptance rate (classification, structured output), (3) measure acceptance rate per workload before enabling.

### Q9. The MoE Deployment That Wouldn't Fit ⭐⭐⭐
Mixtral 8x7B (~50GB) seemed like a 70B model. Underestimated: MoE requires ALL experts in memory even though only 2 active per token. Actual memory: 60+ GB per replica. Multiple replicas overwhelmed. Fix: (1) understand MoE memory profile before deploying, (2) expert parallelism sharding, (3) fewer replicas with more GPUs each.

### Q10. Convincing Team to Migrate From TGI to vLLM ⭐⭐⭐
Team using TGI (HF-native). Missing vLLM features (APC, better multi-LoRA). Approach: (1) don't attack TGI, (2) quantify feature gaps in dollar terms ('APC would save $8K/month'), (3) pilot vLLM on non-critical service, (4) side-by-side comparison, (5) let data drive decision. Migration approved after 2-month proof.

### Q11. Autoscaling Overshooting Cost Budget ⭐⭐⭐
HPA aggressively scaled up during traffic spike. Kept high replicas after spike. Cost overrun. Fix: (1) autoscaling with tighter scale-down policy, (2) cost-based autoscaling (don't scale beyond budget threshold), (3) predictive scaling instead of reactive, (4) periodic capacity review.

### Q12. The Interconnect That Wasn't What Vendor Said ⭐⭐⭐⭐
Vendor claimed InfiniBand between nodes. Actually it was Ethernet with RoCE. Discovered when TP across nodes performed terribly. Fix: (1) verify hardware in staging BEFORE production, (2) benchmark actual measured bandwidth, (3) architect within known constraints. Learn: vendor claims != reality; measure.

### Q13. Multi-LoRA Serving Ate All GPU Memory ⭐⭐⭐
100+ LoRAs loaded. Each small (~50MB) but total exceeded GPU memory. New requests failing. Fix: (1) LRU eviction of unused LoRAs, (2) lazy loading (load LoRA on first request for that tenant), (3) usage tracking to identify unused LoRAs, (4) tier-based LoRA allocation (enterprise tenants pinned in memory, others eviction-eligible).

### Q14. Framework Bug That Corrupted Responses ⭐⭐⭐
vLLM v0.X had bug in continuous batching under specific condition. Responses got shuffled between requests (user A got user B's response occasionally). Discovered by internal QA. Approach: (1) immediate: shed load, verify issue, roll back to previous vLLM version, (2) report to maintainers with reproduction, (3) contribute fix, (4) systematic: don't pin to bleeding-edge in production, monitor community issues.

### Q15. Cost Attribution Failing for Enterprise Customer ⭐⭐⭐
Enterprise customer disputes their bill. We can't prove their token usage precisely (aggregation lag). Approach: (1) apologize + credit, (2) improve per-request cost tracking accuracy, (3) real-time cost dashboards for enterprise tenants (they see their own spend), (4) monthly invoicing with detailed breakdown.

### Q16. Explaining GPU Economics to Non-Technical Leadership ⭐⭐⭐
CFO asks: 'Why does inference cost so much?' Approach: quantify concrete drivers. 'H100 SXM = $2.4M capital cost, depreciates over 3 years = $2/hr. Add power, cooling, networking, HR time, and it's ~$5/hr per GPU. To serve 1M req/day, we need 8 GPUs running 24/7 = $960/day. That's before we consider redundancy and headroom.' Concrete numbers > hand-waving.

### Q17. The Team That Wouldn't Adopt Observability ⭐⭐⭐⭐
New inference team focused on serving performance. Refused to instrument observability. First incident: took 6 hours to diagnose. Approach: (1) show cost of the incident (engineer hours × hourly = $X), (2) make observability part of every deploy checklist, (3) pair with team on first incident using observability, (4) show the value in the second incident when we diagnosed in 20 min. Culture change through demonstrated wins, not lectures.
