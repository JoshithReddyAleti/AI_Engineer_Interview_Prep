# 🎭 Week 7 — Behavioral & Scenario Questions

> **Focus:** Memory leak incidents, GDPR compliance emergencies, cross-user contamination, cost blowups, migration decisions, user trust breakdowns
>
> **How to use:** These are the real 3 AM incidents that memory system owners face. Practice reasoning out loud, showing your incident-response process.

---

## Q1. The Cross-User Memory Leak ⭐⭐⭐⭐

**Scenario:** A user emails support screenshots of your AI assistant mentioning ANOTHER user's private information — their name, their job, their preferences. The screenshots are on Twitter within hours. What do you do?

**Strong answer:**

"This is a severity-1 security incident. My response, in order:

**Hour 0-1 — Contain the bleeding.**
1. Disable the memory system entirely across all users. Assistant runs stateless. Better degraded experience than more leaks.
2. Alert legal, security, PR, exec team. This is a data breach that requires disclosure.
3. Preserve evidence: pull the conversation logs, memory state at the time, retrieval traces.

**Hour 1-6 — Diagnose.**
4. Trace back: which user's memories leaked into which user's response? Was it retrieval (semantic search returning wrong user's data) or context assembly (mixing memories in code)?
5. Identify the failure mode:
   - Missing tenant_id filter in retrieval query?
   - Cache poisoning (one user's cached response served to another)?
   - Async race condition (two users' contexts assembled simultaneously)?
   - Namespace collision in vector store?
6. Determine scope: was this ONE user affected, or many? Query logs for similar patterns.

**Hour 6-24 — Notification & Fix.**
7. Notify all affected users. Not just the visible one — anyone whose data may have leaked.
8. GDPR requires notification within 72 hours to authorities for personal data breaches. Legal handles this.
9. Deploy fix. Add automated cross-tenant test: attempt to retrieve user A's data as user B, must fail. Run before every deploy.
10. Re-enable memory system with fix, incrementally by user segments.

**Days 1-7 — Post-incident.**
11. Full post-mortem. Root cause. Timeline. Detection gaps. Preventive measures.
12. Publish blameless post-mortem internally.
13. Consider public disclosure depending on scope and jurisdiction.
14. External security audit if scope was large.

**Prevention going forward:**
- **Defense in depth:** Never rely on application code alone. Enforce isolation at DB level (row-level security, per-tenant namespaces).
- **Automated tests:** CI must include cross-tenant tests.
- **Access logs with alerting:** Any query lacking user_id filter should alert immediately.
- **Regular penetration testing:** Contract security firms to actively try to break tenant isolation.

**What I'd communicate to the CEO:**
'We had a memory leak affecting X users. We've contained it, notified affected users, and shipped a fix. Root cause was a missing filter in the vector search query. Preventive measures deployed. Full post-mortem in 3 days. Ongoing risk: minimal (mitigations in place). Recommend external security audit for public credibility.'

**What I would NOT do:**
- Deny the incident or downplay it
- Hide it from users
- Blame a specific engineer publicly
- Rush a fix without proper testing (creates second incident)

**The lesson:** Isolation must be enforced at multiple layers. Any single point of failure is one deploy away from catastrophe."

---

## Q2. The GDPR Deletion Request That Can't Be Completed ⭐⭐⭐⭐

**Scenario:** A user submits a GDPR right-to-be-forgotten request. Your team completes the deletion — from the DB, vector store, and caches. Two weeks later, the user's data appears in a logged conversation the AI referenced. What went wrong?

**Strong answer:**

"This is a compliance failure with regulatory implications. Let me diagnose then fix.

**Diagnosis — likely causes:**

**1. Downstream cache we missed.**
Somewhere in the system, the user's memory was cached in a location we didn't purge. Common culprits:
- Semantic cache from RAG pipeline
- CDN or edge cache
- Read replica that lagged the deletion
- Data warehouse for analytics
- Backup snapshot within retention period

**2. Compressed summary that referenced the user.**
Their raw memories were deleted, but a summarization step had compressed their data into a summary that referenced them. The summary wasn't tracked as containing the user's data.

**3. Fine-tuned model training.**
Their data was included in a training run. Now their information is baked into model weights — impossible to fully remove without retraining.

**4. Log files.**
Application logs contained their memory content. Logs weren't part of the deletion pipeline.

**5. Downstream sharing.**
Their data was legitimately shared with a partner system (analytics, another product). That partner's deletion wasn't triggered.

**Fix approach:**

**Immediate (this week):**
1. Locate the specific instance of their data. Delete it.
2. Notify the user of the residual data and its deletion. Own the mistake.
3. Audit whether other users have similar residual data. Bulk deletion pipeline.

**Medium-term (this month):**
4. Rebuild the deletion pipeline. Map every location user data can end up:
   - Primary DB, cache, vector store, backup, snapshot
   - Analytics DB, data warehouse
   - Logs, monitoring, tracing systems
   - Training data pipelines
   - Downstream partner systems
5. For each: implement automated deletion when a delete request is triggered.
6. Add validation: after deletion, sample query for the user's data across all systems. Must return nothing.

**Long-term (this quarter):**
7. Data lineage tracking. Every piece of user data tagged with source. Deletion follows the lineage graph.
8. Regular deletion audits. Run 'phantom deletion' tests: request deletion for a test user, verify complete purge.
9. Consider: don't include user data in fine-tuning. If needed, use differential privacy.

**Communication to legal team:**
'Confirmed residual data in [system]. Deleted. Root cause: gap in deletion pipeline coverage for [component]. Deployed fix. Investigating whether this affected other users' deletion requests retrospectively. Recommending we notify affected users and regulators per GDPR guidelines.'

**The systemic issue:**
Deletion isn't a single DELETE query. It's an orchestrated pipeline touching every system that could hold user data. Most companies underestimate this."

---

## Q3. The Silent Memory Degradation ⭐⭐⭐

**Scenario:** Users start complaining that "the AI is getting dumber." Support tickets increase over 3 months. Response quality metrics look normal. Latency is fine. But something is off. Where do you look?

**Strong answer:**

"Silent degradation is the hardest kind. Everything monitored looks fine, but users know something's wrong. Common causes in memory systems:

**Hypothesis 1 — Memory pollution.**
Over 3 months, memory has accumulated garbage. Bad extractions, contradicted facts, low-quality summaries. Retrieval returns 'relevant-looking' memories, but they're wrong or misleading.

Test: sample 100 recent memory retrievals. Human-verify against source conversations. Baseline vs 3 months ago.

**Hypothesis 2 — Retrieval bias.**
Vector store is returning memories that are 'similar' in embedding space but not 'useful' for the query. Semantic drift.

Test: measure precision@5 on a golden eval set. Compare to baseline.

**Hypothesis 3 — Stale facts.**
Users' situations changed. Old memories are still being retrieved and treated as current. 'You mentioned you're a data scientist' — user changed jobs 4 months ago.

Test: audit memory age distribution. Compare frequency of memories over 90 days old being surfaced now vs 3 months ago.

**Hypothesis 4 — Compression drift.**
As conversations grow, summaries compress older turns. But summarization is lossy. Over many compressions, the 'summary of a summary of a summary' has drifted from truth.

Test: for long-running users, compare current summary to reconstructed history. Semantic similarity should be high.

**Hypothesis 5 — Retrieval weight imbalance.**
Multiple memory sources (window, entity, semantic). Weights shifted over time — maybe due to a config change. Now retrieval over-weights one source.

Test: audit config changes over 3 months. Check retrieval mix (how many memories from each source per query).

**Hypothesis 6 — Model drift.**
The base LLM was updated by the provider. Different model behavior with same memories. Not a memory issue at all.

Test: compare responses between old model version (if pinned somewhere) and current version, same memories.

**Diagnostic approach:**

1. Pick 20 users who reported degradation. Look at their traces from 3 months ago vs now.
2. For each: what changed? Memory content? Retrieval scores? Model responses?
3. Categorize the failure modes across the 20.
4. That distribution points to the root cause.

**Fix — depends on the diagnosis:**

- Pollution → memory quality audit + selective cleanup + tighter extraction rules
- Retrieval bias → reweight retrieval, add reranker, update ranking model
- Stale facts → time-decay in retrieval scoring, active refresh
- Compression drift → periodic re-summarization from raw source, not from prior summary
- Weight imbalance → revert config, add regression test
- Model drift → pin model version, or update prompts for new model

**The lesson:**
'Metrics look fine' isn't validation. Track memory quality metrics explicitly (accuracy, freshness, contradiction rate). Silent degradation happens when your metrics don't cover the right dimensions."

---

## Q4. The Cost Explosion from Semantic Memory ⭐⭐⭐

**Scenario:** Your AI assistant launched with semantic memory. Cost was $500/month. 6 months later, it's $50,000/month. Storage grew from 5 GB to 5 TB. Retrieval latency doubled. CEO wants answers.

**Strong answer:**

"1000x cost growth in 6 months means unbounded write-back. Memory grows with every interaction, and nothing prunes.

**Immediate diagnosis:**

**Storage growth:**
- 5 GB → 5 TB = 1000x growth
- If users grew 10x, users aren't the cause (would be 10x cost, not 1000x)
- Memory per user is growing. Each user's memory footprint is expanding rapidly.

**Why storage grows:**
- Every conversation turn embedded and stored (no filtering)
- Duplicated embeddings (same user asking same question over months)
- No forgetting policy — memories accumulate forever
- Extraction storing extremely fine-grained facts

**Immediate cost reduction (this week):**

**1. Stop the bleeding.**
Add a filter: only embed turns > 30 tokens (skip 'ok', 'thanks', 'yes'). Estimated 30-50% reduction in new writes.

**2. Deduplicate.**
Before storing, check if a similar memory already exists (embed distance < 0.05). Skip if duplicate. Estimated 20-30% reduction in storage.

**3. Cap per-user memory.**
Max 5,000 memories per user. When limit hit, evict oldest low-value memories.

**4. Enable TTL.**
Memories older than 12 months → move to cold storage. Not accessible in normal retrieval. Estimated 60-80% reduction in active vector store size.

**Combined immediate impact: 5 TB → ~1 TB active. Cost from $50K → ~$10K/month.**

**Medium-term (this month):**

**5. Consolidate.**
Cluster similar memories per user. Replace clusters with representative memory + count. Preserves information, reduces storage.

**6. Importance scoring.**
Each memory scored by usage frequency. Prune bottom 30% quarterly.

**7. Compression at scale.**
Batch compression: for each user, compress memories older than 3 months into summaries. Retain raw only for recent.

**8. Query-side improvements.**
Reduce top-K from 10 → 5. Better reranker so top-5 is high quality. Less over-retrieval.

**Combined medium-term: additional 50% reduction. Cost ~$5K/month.**

**Long-term (this quarter):**

**9. Tiered storage.**
Hot (Redis): last 30 days. Warm (PostgreSQL): last 12 months. Cold (S3): archive.
Vector store only for hot tier. Retrievals prioritize hot, fall back to warm.

**10. Right-size the vector index.**
Move from expensive (Pinecone p1) to cost-optimized (Pinecone serverless or self-hosted). 5-10x cost reduction.

**Communication to CEO:**
'Cost was growing unbounded because we had no forgetting policy or storage caps. Immediate fix reduces cost 80% this week. Medium-term fixes reduce another 50%. Long-term: tiered storage architecture cuts vector store cost 5-10x. Final expected cost: $2-5K/month. This is a design issue, not a scaling issue — memory systems need forgetting policies from day one.'

**The lesson:**
Memory systems without forgetting are a cost time bomb. Design forgetting BEFORE launching, not after cost explodes."

---

## Q5. Explaining Memory Loss to Users ⭐⭐⭐

**Scenario:** Your team decides to reduce memory retention from unlimited to 90 days for cost reasons. Users will notice — some will complain. How do you communicate this?

**Strong answer:**

"This is a product communication problem, not just an engineering one. Users made an emotional commitment to an AI that 'remembers' them. Rolling that back requires care.

**What NOT to do:**
- Silently deploy the change and let users discover it.
- Blame technical reasons ('storage costs') — users don't care about your ops budget.
- Be vague ('we're making improvements') — users see through it and lose trust more.

**What to do:**

**Framing: focus on what stays, not what leaves.**

Not: 'We're deleting your old memories.'
Instead: 'We're focusing on what matters most — the last 90 days of context.'

Frame the change as a POSITIVE product decision, not a limitation. Explain:
- Old memories often become stale (your job, preferences, projects change)
- Focused memory = higher quality responses (less noise, more signal)
- Users can explicitly mark memories as 'always remember' (create an escape hatch)

**Timeline:**
1. Announce the change 30 days in advance.
2. Provide export tool immediately — users can download all their memories.
3. Provide 'always remember' feature for high-value memories.
4. Offer premium tier if they want unlimited memory (monetize the cost).
5. Deploy gradually — new users first, then existing.

**Communication channels:**
- Email to all users (formal notification)
- In-app notification (context-sensitive: when they interact with the AI)
- Blog post (public transparency)
- Support docs (detailed explanation, FAQ)

**Sample messaging:**

Subject: "Improving your AI's memory"

Body: "We're making changes to how [AI Name] remembers you. Starting [date], memories will focus on the last 90 days — the most relevant context for helping you.

Why the change: shorter memory means more accurate responses. Old memories can become outdated (jobs change, preferences shift). Focused memory works better.

What this means for you:
- Your recent memories (last 90 days) stay as usual
- Older memories: download them anytime before [date] via [link]
- Want to keep something forever? Mark it as 'Always Remember' in your settings.
- On our Premium tier ($X/month) memory retention is 1 year.

We know memory feels personal. Reach out if you have concerns: [support]."

**Handle the backlash:**
- Some users WILL be upset. Have empathetic support responses ready.
- Offer opt-in extension for beta users or long-term customers.
- Track sentiment. If backlash is severe, be willing to revise the plan.

**Interview signal:** Discussing the user-emotional dimension (not just technical) shows product maturity."

---

## Q6-Q16: Additional Behavioral Scenarios (Condensed)

### Q6. The Wrong Fact That Won't Go Away ⭐⭐⭐
User tells AI "I don't have kids." Weeks later, AI still says "your kids." User is furious. Root cause: extracted fact was stored with high confidence, subsequent correction wasn't picked up. Fix: prioritize explicit corrections in extraction, add correction as first-class signal, allow user to view/edit facts directly.

### Q7. The New Engineer Who Removes the User_ID Filter ⭐⭐⭐⭐
Junior engineer submits PR that inadvertently removes user_id filter from a memory query 'to simplify.' Reviewers miss it. Deployed. Cross-user contamination begins. Prevention: mandatory security review for memory code, CI test that fails on missing user_id filter, code review checklist.

### Q8. The Migration From Redis to PostgreSQL ⭐⭐⭐⭐
Current Redis-only memory system is losing data on Redis restarts. Migration to PostgreSQL takes 3 months but users notice inconsistencies during migration window. Approach: dual-write phase, shadow-read validation, gradual cutover per tenant, rollback plan for each phase.

### Q9. The Vector Store Vendor Discontinuation ⭐⭐⭐
Your vector store provider announces service shutdown in 12 months. You have 5M user memories embedded. Migration approach: choose new provider, re-embed all memories (cost: significant), dual-read during transition, validate migration integrity, cutover.

### Q10. The Compliance Auditor's Question ⭐⭐⭐
External auditor asks: 'Show me every action taken on user X's memories in the last 12 months.' Your logging doesn't capture that level of detail. Approach: honest disclosure of the gap, implement detailed audit logging, provide what you can from existing logs, remediation plan with timeline.

### Q11. Onboarding a Junior Engineer to Memory Systems ⭐⭐
Junior joins team, needs to understand memory. Approach: start with 'why memory exists' (stateless LLMs), then buffer/window/summary progression, then persistent memory (SQLite → real backends), then multi-tenant considerations. Pair on real incidents. Emphasize security from day 1.

### Q12. The Debate: More Memory vs Better Memory ⭐⭐⭐
Team split: half wants to store MORE (add graph memory, add semantic-per-turn). Half wants to focus on QUALITY (extraction accuracy, retrieval precision). Approach: measure current quality baseline, quality improvements first (extraction accuracy 70% → 90% is bigger win than adding memory types), then scope expansion after quality is solid.

### Q13. The Session Timeout Complaint ⭐⭐⭐
Users report AI 'forgets' after 30 minutes of inactivity even though it should remember. Root cause: session state expires but persistent memory should still work — but somehow doesn't. Diagnosis: session expires → new session_id → memory lookups fail because session_id was part of retrieval key. Fix: retrieval by user_id, not session_id.

### Q14. Convincing Leadership to Invest in Memory Infrastructure ⭐⭐⭐
Leadership sees memory as a cost, not a differentiator. Approach: show retention data (users with active memory have 3x retention), quality metrics (memory-aware responses score 40% higher), competitive analysis (competitors have memory). Frame as investment, not overhead.

### Q15. Explaining Memory Architecture to Non-Technical Stakeholders ⭐⭐
Board asks 'why can't the AI just remember everything like a human?' Analogy approach: a librarian remembering every conversation with every visitor is impossible even for humans — good librarians use indexes (entity memory), summaries (summary memory), and time-based filing (episodic). AI does the same. Storage/retrieval always has trade-offs. Set expectations realistically.

### Q16. The User Who Wants Total Memory Erasure Then Regrets It ⭐⭐⭐
User exercises right-to-delete. Later, changes their mind: 'I didn't mean permanently.' But you deleted everything. Approach: HONEST — deletion was completed, data is gone (compliance requirement). Offer: import from any backups they have (email, notes). Add: soft-delete option (30-day grace period before permanent deletion) as future feature. Learning: default to soft-delete when possible.
