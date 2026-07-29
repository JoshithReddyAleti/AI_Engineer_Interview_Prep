# 🎭 Week 5 — Behavioral & Scenario Questions

> **Focus:** RAG debugging in production, hallucination incidents, scaling crises, stakeholder communication, chunking/embedding trade-off decisions
>
> **How to use:** These are the real scenarios that RAG engineers face weekly. Practice your reasoning process out loud.

---

## Q1. The RAG Hallucination Incident ⭐⭐⭐

**Scenario:** Your RAG-powered customer support bot tells a user "Your order qualifies for a full refund plus 20% bonus credit." This is completely wrong — your company doesn't offer bonus credit. The user screenshots it and posts it on Twitter. It's going viral. What do you do?

**Strong answer:**

**Hour 0-1 (damage control):**
1. Take the chatbot offline or switch to "read-only FAQ mode" — stop generating new hallucinations immediately.
2. Prepare a public statement: "We're aware of an incorrect response from our AI assistant. We're investigating and the assistant has been temporarily paused."
3. Honor the incorrect promise for THIS user — fighting a viral screenshot makes it worse. Give them the refund + credit.

**Hour 1-4 (investigation):**
4. Pull the trace for this specific conversation. Check: what chunks were retrieved? Was the refund policy chunk present? Was there a "bonus credit" chunk from an outdated promotion?

Most likely root cause: an old promotional document ("Holiday 2023: 20% bonus credit on returns") was still indexed. The RAG system retrieved it alongside the current refund policy, and the LLM merged them.

**Hour 4-8 (fix):**
5. Remove the outdated promotional document from the index.
6. Audit the entire corpus for stale documents — any document older than 6 months gets flagged for review.
7. Add a faithfulness check: post-generation validation that compares the response against current policy documents only.
8. Add a guardrail: "Never promise financial compensation (refunds, credits, discounts) without explicit approval from the response template."

**Day 2+ (prevention):**
9. Implement document version management — only the LATEST version of each document is indexed.
10. Add a TTL (time-to-live) on indexed documents. Documents older than the TTL are auto-flagged for re-review.
11. Add the exact query to the eval set as a regression test.

---

## Q2. The "It Was Better Before" Regression ⭐⭐⭐

**Scenario:** You improved your RAG system — upgraded the embedding model, increased chunk overlap, and added a reranker. Metrics improved on the eval set. But 3 users report: "The old system answered my question perfectly. The new one gives irrelevant answers." What's happening?

**Strong answer:**

"Metric improvements on the eval set don't guarantee improvement on every query. The eval set is a sample — it can't represent every real user pattern. Here's how I'd investigate:

**Step 1 — Reproduce their specific queries.** Run the exact 3 failing queries through both the old and new pipeline. Compare: what chunks are retrieved? Are the correct chunks being found?

**Step 2 — Diagnose the failure:**
- **New embedding model maps differently.** A query that perfectly matched a chunk with the old model might score lower with the new model. Different models have different semantic spaces.
- **Increased overlap creates near-duplicates.** More overlap → more chunks that look similar → the top-5 might now have 3 variants of the same paragraph, pushing the actually different relevant chunk out.
- **Reranker is over-correcting.** The cross-encoder might prefer chunks that look more 'answerable' (well-structured, confident language) over chunks that are actually more relevant (buried in technical jargon).

**Step 3 — Fix without reverting:**
- Add these 3 queries to the eval set (expanding coverage)
- If the new embedding is the issue: check if the specific domain terms are underrepresented in the new model's training data
- If duplication is the issue: add deduplication post-retrieval
- If the reranker is the issue: adjust the reranker weight or use it only when bi-encoder confidence is low

**Step 4 — Communicate:**
'We identified that the upgrade improved 85% of queries but regressed on 3 specific patterns. We're fixing those while keeping the overall improvement. Should be resolved within 48 hours.'

**The lesson:** Always keep the ability to A/B test old vs new. Never delete the old pipeline configuration until the new one is validated on real traffic for at least 2 weeks."

---

## Q3. The Chunking Strategy Debate ⭐⭐⭐

**Scenario:** Your team is arguing about chunking. Engineer A says 256-token chunks for precision. Engineer B says 1024-token chunks for context richness. Engineer C wants semantic chunking. You're the tech lead. How do you decide?

**Strong answer:**

"I'd refuse to decide by opinion and instead decide by data.

**Step 1 — Build the eval set first.** Before testing any chunking strategy, we need 100+ queries with labeled relevant passages. Without this, we're comparing feelings, not quality.

**Step 2 — Test all three.** Re-chunk the corpus three ways, re-embed, and run the same eval set against each:

```
Strategy         | Precision@5 | Recall@10 | End-to-End Accuracy | Avg Chunk Tokens
256-token fixed  | 0.82        | 0.71      | 0.76                | 256
1024-token fixed | 0.65        | 0.89      | 0.79                | 1024
Semantic         | 0.78        | 0.83      | 0.81                | ~400 (variable)
```

**Step 3 — Consider the trade-offs beyond accuracy:**
- 256-token: More chunks = more storage = higher cost. But precise retrieval.
- 1024-token: Fewer chunks = cheaper. But retrieval is less precise (relevant info buried in long chunks).
- Semantic: Best quality but expensive to compute (embedding every sentence at ingestion time) and unpredictable chunk sizes.

**Step 4 — Likely recommendation:** Parent-child chunking. Use 256-token child chunks for retrieval precision, but return the 1024-token parent chunk for context richness. Best of both worlds. If semantic chunking won by a large margin AND we can afford the compute, consider it for v2.

**The principle:** Arguments about chunking are arguments about data. Run the experiment. The eval set decides, not opinions."

---

## Q4. The Cost Explosion ⭐⭐⭐

**Scenario:** Your RAG system launched at $500/month. After 3 months, it's $8,000/month. Traffic grew 5x, but cost grew 16x. The CEO asks why costs are growing faster than usage. Diagnose and fix.

**Strong answer:**

"Cost growing faster than usage means per-query cost is increasing. That's unusual — it should be roughly constant. Let me diagnose.

**Most likely cause — conversation context growth:**
Users are having longer conversations. At turn 1, the context is: system prompt + 5 chunks = ~3K tokens. At turn 20: system prompt + 5 chunks + 20 turns of history = ~15K tokens. Input tokens grew 5x per query as conversations got longer. Multiply by 5x more users = 25x cost growth (close to the 16x we're seeing).

Fix: Implement context compression. After turn 10, summarize old turns. Cap input at 8K tokens regardless of conversation length. Expected savings: 60-70%.

**Second possibility — embedding re-computation:**
If someone is re-embedding the entire corpus on every deployment (instead of incrementally), that's a recurring cost that scales with corpus size, not query volume.

Fix: Content-addressed embedding cache. Only embed new/changed chunks.

**Third possibility — model routing isn't working:**
If the routing classifier is sending 50% of queries to gpt-4o instead of the intended 10%, the per-query cost is 16x higher on those queries.

Fix: Audit routing decisions. Check what % of queries actually go to each model.

**Action plan:**
1. Add per-query cost logging (today — should have been there from day 1)
2. Implement context compression (this week — biggest savings)
3. Audit model routing (this week — quick check)
4. Add cost alerting: $200/day warning, $400/day page
5. Present to CEO: 'Cost will be back to $2K/month within 2 weeks'"

---

## Q5. Explaining RAG to a Non-Technical Executive ⭐⭐

**Scenario:** The CTO asks you to explain to the board why you're building RAG instead of just fine-tuning the model on your company's data.

**Strong answer:**

"Think of it this way:

Fine-tuning is like sending the AI to school to memorize our documents. Once it graduates, it knows what it learned — but if we update a document tomorrow, we have to send it back to school. That takes days and costs thousands every time.

RAG is like giving the AI a filing cabinet it can search through. When someone asks a question, the AI opens the cabinet, pulls out the relevant files, reads them, and answers based on what it just read. If we update a document, we just swap the file in the cabinet — instant, no retraining.

Three reasons RAG wins for us:

1. **Our data changes weekly.** New products, policy updates, pricing changes. Fine-tuning would mean retraining constantly. RAG just re-indexes the changed documents.

2. **We need citations.** When the AI answers a customer question, we need to show WHERE that answer came from — which document, which page. RAG does this naturally. A fine-tuned model can't point to its sources.

3. **We need accuracy we can verify.** A fine-tuned model might 'remember' wrong. RAG gives us a paper trail — if the answer is wrong, we check the source documents. If the source is wrong, we fix the source. Clear accountability.

The cost comparison: fine-tuning costs $5-50K per run and takes days. RAG costs ~$500/month to operate and updates in minutes."

---

## Q6. The Retrieval Quality vs Latency Trade-Off ⭐⭐⭐

**Scenario:** Your RAG system has 92% accuracy but p95 latency is 8 seconds. Product management says latency must be under 3 seconds. Removing the reranker would cut latency to 3 seconds but accuracy drops to 84%. How do you navigate this?

**Strong answer:**

"Neither extreme is acceptable. 8 seconds causes user drop-off. 84% accuracy causes wrong answers. I need to find a path that keeps accuracy >90% AND latency <3 seconds.

**Option 1 — Faster reranker.** Replace the LLM-based reranker with a dedicated cross-encoder model (ms-marco-MiniLM: 50ms vs 2000ms). Accuracy stays at ~90%, latency drops to ~4 seconds.

**Option 2 — Conditional reranking.** Only rerank when the bi-encoder confidence is low (top score < 0.8). ~40% of queries skip reranking entirely. Average latency drops to ~4.5 seconds, accuracy stays at ~91%.

**Option 3 — Parallel retrieval + streaming.** Start streaming the LLM response immediately after retrieval. Show 'Searching...' then sources, then answer tokens. Perceived latency drops even if actual computation takes 6 seconds.

**Option 4 — Cache frequent queries.** 30% of queries are repeats or near-repeats. Semantic cache returns in <100ms. Average latency drops dramatically.

**My recommendation:** Combine options 1 + 4. Faster reranker model + semantic caching. Expected: 91% accuracy, p95 latency 3.5 seconds, with 30% of queries under 200ms from cache.

Then tell product: 'We can hit <3s p95 today if we accept 84% accuracy, or 3.5s p95 at 91% accuracy. I recommend the second option and we iterate from there.'"

---

## Q7. The "Our Vector DB Is Down" Emergency ⭐⭐⭐

**Scenario:** Your Pinecone index goes down during peak hours. 5,000 users are actively chatting. Your RAG system returns errors on every query. What do you do?

**Strong answer:**

**Immediate (5 minutes):** Activate the fallback chain:
1. Switch to BM25-only search (Elasticsearch, which is still up) — degraded quality but functional
2. If BM25 is also down → serve cached responses for the top-500 most common queries
3. If nothing works → "I'm unable to search our knowledge base right now. Here are links to our most popular help articles: [FAQ], [Troubleshooting], [Contact Support]"

**The key insight:** Your RAG system should NEVER be a single point of failure. Design for graceful degradation from day 1:

```python
async def retrieve(query):
    try:
        return await vector_search(query)          # Primary: Pinecone
    except VectorDBError:
        try:
            return await keyword_search(query)      # Fallback 1: BM25
        except SearchError:
            cached = await check_cache(query)       # Fallback 2: Cache
            if cached:
                return cached
            return generate_fallback_response()     # Fallback 3: Static
```

**Post-incident:** Add the vector DB health check to the monitoring dashboard. Set up auto-failover to BM25 when Pinecone latency > 2 seconds. Maintain a read replica or local FAISS snapshot that can be activated in emergencies.

---

## Q8-Q15: Additional Behavioral Scenarios (Condensed)

### Q8. The Embedding Bill Shock ⭐⭐⭐
You changed chunking strategy from 512 to 256 tokens. Chunk count doubled. Re-embedding cost: $20K. CEO is furious. How do you prevent this? → Content-addressed cache, sample-test before full re-embed, track embedding costs as a deployment metric.

### Q9. The "RAG Can't Answer This" Complaint ⭐⭐
Users ask questions the knowledge base genuinely doesn't cover. Instead of saying "I don't know," the model hallucinates an answer. → Calibrate abstention threshold, add "I don't have information about this" as a first-class response, track abstention rate vs hallucination rate.

### Q10. The Contradictory Documents Problem ⭐⭐⭐
Two documents in your corpus directly contradict each other (old policy vs new policy). The RAG system sometimes cites the old one. → Document versioning, recency bias in retrieval, metadata filter for "status=current", human review workflow for conflicting sources.

### Q11. Joining a Team That Uses RAG Wrong ⭐⭐
Your new team has a RAG system that's essentially keyword search + LLM wrapper. No reranking, no hybrid search, no evaluation, no metadata filtering. Where do you start? → Build eval set first (measure the baseline), then add one improvement at a time (reranker, then hybrid, then eval CI), measure each change.

### Q12. The "Why Not Just Use ChatGPT?" Question ⭐⭐
A stakeholder asks why you built a custom RAG system when ChatGPT exists. → ChatGPT doesn't have your proprietary data, can't cite internal sources, has no access control, hallucinates freely, and you can't control or evaluate it. RAG gives you: grounded answers, citations, access control, evaluation, and auditability.

### Q13. The Multi-Hop Query That Stumps Your System ⭐⭐⭐
User asks a complex question requiring info from 5 documents. Your RAG returns fragments that don't connect. → Query decomposition (break into sub-queries), iterative retrieval (retrieve → reason → retrieve more), or upgrade to Agentic RAG for adaptive multi-step retrieval.

### Q14. Choosing Between RAG Variants for a New Project ⭐⭐⭐
The PM asks: "We need an AI assistant for our legal team. Which RAG variant?" → Start with Advanced RAG (hybrid search + reranking). If they need entity relationships across contracts → add Graph RAG. Legal domain requires CRAG (verification before generation) because wrong answers = malpractice risk. Always approval-first for legal.

### Q15. Presenting RAG Metrics to the Board ⭐⭐
The board wants to know if the AI investment is working. What do you show? → Don't show "faithfulness: 0.91" (means nothing to them). Show: "91% of AI answers are correct and cited. Customer ticket resolution time dropped 40%. AI handles 3,000 queries/day that would otherwise need human agents. Cost: $15,000/month vs $150,000/month for equivalent human team. ROI: 10x."
