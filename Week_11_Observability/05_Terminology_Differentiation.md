# 📖 Week 11 — Terminology Differentiation Reference

> **Focus:** Similar-sounding AI/RAG/Observability terms that mean different things. Every set includes: definition, when to use, when NOT to use, common interview traps.
>
> **How to use:** Interviewers love probing "what's the difference between X and Y?" to catch candidates who use terms loosely. This file is your defense.

---

# Section 1: RAG Retrieval Terminology

These four terms sound similar but measure very different things. Confusing them signals inexperience.

## 🔍 Chunking vs Chunk Relevance vs Context Utilization vs Hit@K

### Chunking

**Definition:** The PROCESS of splitting documents into smaller pieces before embedding.

**Not a metric — an operation.** It's what you DO to prepare data for RAG.

**Parameters:**
- Chunk size (200-2000 tokens typically)
- Overlap (10-20% typically)
- Boundary strategy (character, sentence, paragraph, semantic)

**Use when:** discussing HOW documents are prepared for the vector store.

**Don't use when:** discussing retrieval quality — chunking is upstream of retrieval.

**Interview trap:** Candidates say "we improved chunking" to answer "how did you improve retrieval?" That's causally correct but analytically loose. Say: "we changed chunk size from 512 to 1024 tokens, which improved retrieval Hit@5 from 0.72 to 0.81."

---

### Chunk Relevance

**Definition:** How well an INDIVIDUAL retrieved chunk matches the query intent.

**A metric** measured per-chunk per-query.

**How measured:**
- Similarity score from vector search (cosine, dot product) — noisy proxy
- Reranker score from cross-encoder — better proxy
- LLM-judge relevance rating (1-5) — most accurate but expensive
- Human labeling — gold standard

**Use when:** discussing quality of INDIVIDUAL retrieved chunks.

**Don't use when:** discussing whether retrieval as a whole succeeded (that's Hit@K).

**Interview trap:** Confusing "high relevance score" with "correct answer." A chunk can have 0.9 similarity but be wrong; a chunk with 0.7 similarity might be correct.

---

### Context Utilization

**Definition:** Of retrieved chunks passed to the LLM, how many did the LLM ACTUALLY USE in generating the response?

**A metric** measured per-response.

**How measured:**
- Citation analysis: if response cites chunk IDs, which were cited?
- Attribution analysis: which retrieved text appears in the response?
- Overlap analysis: n-gram overlap between chunks and response

**Why it matters:** If you retrieve 5 chunks but response uses only 1, you're paying for 4 chunks of context tokens for nothing. Retrieve fewer.

**Use when:** optimizing cost/latency by right-sizing retrieval.

**Don't use when:** discussing whether the RIGHT information was retrieved (chunk might have been useful but ignored).

**Interview trap:** High context utilization ≠ good answer. Model might use retrieved chunks AND still hallucinate. It measures "was the retrieval consumed," not "was the answer correct."

---

### Hit@K (also written Hit@k or Recall@K)

**Definition:** Did the relevant document appear in the top K retrieved results?

**A metric** at the retrieval SYSTEM level.

**How measured:**
- Offline (with labels): for each query, check if labeled-relevant docs appear in top K.
- Online (no labels): use "chunks cited by LLM in final answer" or "chunks with LLM-judge relevance ≥ threshold" as ground truth proxy.

**Common Ks:** Hit@1, Hit@5, Hit@10.

**Use when:** measuring overall retrieval system quality across many queries.

**Don't use when:** debugging a single query (use trace + chunk relevance for that).

**Interview trap:** Hit@K measures "was right info retrievable," NOT "was answer correct." Right info can be retrieved and still ignored, misunderstood, or drowned out by irrelevant chunks. High Hit@K + bad answers → generation problem, not retrieval.

---

### 🎯 The Distinction That Matters

| Term | What it measures | Level | Example question it answers |
|---|---|---|---|
| **Chunking** | How docs are split (operation) | Ingestion | "How did you prepare docs?" |
| **Chunk relevance** | Individual chunk quality | Per-chunk | "How relevant was this specific chunk?" |
| **Context utilization** | Fraction of retrieved chunks used | Per-response | "Are we retrieving too much?" |
| **Hit@K** | Fraction of queries where relevant doc in top K | System | "How good is our retrieval overall?" |

**When someone asks "how good is your retrieval?" they mean Hit@K. Not chunk relevance. Not context utilization.**

---

# Section 2: RAG Quality Terminology

## Faithfulness vs Answer Relevance vs Context Precision vs Context Recall

These are the four core RAGAS metrics. Candidates confuse them constantly.

### Faithfulness

**Definition:** Is the generated answer SUPPORTED by the retrieved context?

**Measures:** hallucination (in the intrinsic sense).

**Formula (RAGAS):**
```
faithfulness = number of claims in answer supported by context / total claims in answer
```

**High faithfulness (1.0):** every claim in the answer is backed by context.
**Low faithfulness (0.3):** most claims are made up.

**Use when:** measuring hallucination rate specifically.

**Don't use when:** measuring answer quality overall (a faithful answer can still miss the point).

---

### Answer Relevance

**Definition:** Does the answer actually address the question asked?

**Measures:** on-topic-ness.

**Formula (RAGAS):**
```
1. Generate synthetic questions from the answer
2. Compare synthetic questions to original question via embedding similarity
3. High similarity = answer is relevant to the question
```

**High answer relevance:** answer is on-topic.
**Low answer relevance:** answer is off-topic (technically correct but not what was asked).

**Use when:** detecting when model rambles or misinterprets question.

**Don't use when:** measuring correctness (relevance ≠ correctness).

---

### Context Precision

**Definition:** Of the RETRIEVED chunks, how many are relevant?

**Formula:**
```
context_precision = relevant chunks retrieved / total chunks retrieved
```

**High precision:** retrieval is clean (few irrelevant results).
**Low precision:** retrieval has noise (many irrelevant chunks pollute context).

**Use when:** measuring retrieval SIGNAL-TO-NOISE.

**Don't use when:** measuring whether ALL needed info was retrieved (that's recall).

---

### Context Recall

**Definition:** Of the chunks that WOULD BE relevant, how many did we retrieve?

**Formula:**
```
context_recall = relevant chunks retrieved / total relevant chunks that exist
```

**High recall:** we got most of what's needed.
**Low recall:** missing critical information.

**Use when:** measuring retrieval COVERAGE.

**Don't use when:** measuring signal quality (that's precision).

---

### 🎯 The Distinction That Matters

| Metric | What it asks | Failure mode when low |
|---|---|---|
| **Faithfulness** | Is answer grounded in context? | Hallucination |
| **Answer Relevance** | Does answer address question? | Off-topic response |
| **Context Precision** | Are retrieved chunks relevant? | Noisy retrieval |
| **Context Recall** | Did we retrieve everything needed? | Incomplete context |

**Precision vs Recall trade-off:** Increasing K (retrieve more) → higher recall, lower precision. Reranking helps both by first casting wide (recall) then narrowing (precision).

---

# Section 3: Drift Terminology

Every AI system has drift, but drift means different things.

## Data Drift vs Concept Drift vs Model Drift vs Embedding Drift vs Prompt Drift

### Data Drift (Input Drift, Feature Drift)

**Definition:** The distribution of INPUTS changes over time.

**Example:** Launched for legal docs. Users started asking medical questions. Same model, same prompts, but the questions look different.

**Detection:** PSI/KL on input features (embeddings, categorical variables, text length distributions).

**Fix:** Update system to handle new input types (better retrieval, new prompts, potentially retraining).

---

### Concept Drift

**Definition:** The RELATIONSHIP between input and correct output changes.

**Example:** "Who is the CEO of Twitter?" — the correct answer changed multiple times. Same question, different correct answer over time.

**Detection:** User feedback trending negative on questions that used to work. Golden set with time-stamped ground truth showing accuracy degradation.

**Fix:** Update knowledge base, update embedding index with current data, potentially retrain.

**Critical distinction:** Data drift = inputs change. Concept drift = correct answers change. Different problem, different fix.

---

### Model Drift (Model Version Drift)

**Definition:** The MODEL ITSELF changed while you weren't looking.

**Example:** OpenAI updates GPT-4-turbo behind the same API name. Same prompts, same queries, but responses differ. This is real and happens.

**Detection:** Run golden set weekly. If baseline quality drops with no code changes, model may have drifted. Check provider announcements + release notes.

**Fix:** Pin model versions where possible (`gpt-4-0125-preview` not `gpt-4-turbo`). Test new versions before switching.

---

### Embedding Drift

**Definition:** The distribution of QUERY (or document) EMBEDDINGS shifts over time.

**Example:** Users start using new terminology; embedding space of queries moves; existing chunk embeddings no longer match well.

**Detection:** Compute embedding distribution statistics (mean, variance, cluster centers) over time windows. Compare with PSI/KL.

**Fix:** Re-embed corpus with newer model. If you SWITCH embedding models, ALL vectors must be re-embedded (mixed spaces = catastrophic retrieval failure).

**Critical distinction:** Embedding drift can be either data drift (inputs changing) OR model drift (embedding model changed). Root cause matters.

---

### Prompt Drift

**Definition:** Your OWN prompts changed over time (accidentally or via A/B testing) and quality changed with them.

**Example:** Someone updated a system prompt "to be more concise." Concise responses have less context to be grounded in. Hallucination rate rose.

**Detection:** Version prompts. Every prompt change linked to quality metrics. Prompt version = dimension in observability.

**Fix:** Rollback bad prompt versions. Test prompt changes on golden set BEFORE deploying.

---

### 🎯 The Distinction That Matters

| Drift Type | What changed | Under your control? | Primary detection |
|---|---|---|---|
| **Data drift** | User inputs | No | PSI on input features |
| **Concept drift** | Correct answers | No | User feedback + golden set |
| **Model drift** | Provider's model | No | Golden set weekly |
| **Embedding drift** | Query/doc embeddings | Partially | Embedding distribution PSI |
| **Prompt drift** | Your prompts | Yes | Version + track quality per version |

**"Something changed" isn't a diagnosis. Which type of drift is diagnosis.**

---

# Section 4: Latency Terminology

## TTFT vs Full Response Latency vs End-to-End Latency vs Time-to-Interactive

### TTFT (Time to First Token)

**Definition:** Time from user request to FIRST token appearing in stream.

**For streaming LLMs:** the perceived latency. Users care about TTFT because they see typing start.

**Typical TTFT:**
- Cached response: 20-50ms
- Fast LLM (Haiku, gpt-4o-mini): 300-800ms
- Larger LLM (Opus, GPT-4o): 500-1500ms

**Use when:** measuring perceived latency for streaming apps.

**Don't use when:** measuring total time (streaming apps still have "full response" time).

---

### Full Response Latency (Time to Complete Response)

**Definition:** Time from request to LAST token generated.

**For LLMs:** proportional to output length. Longer responses take longer.

**Typical:** 3-15 seconds for typical LLM responses.

**Use when:** measuring total generation time.

**Don't use when:** discussing user perception (users care about TTFT more for streaming).

---

### End-to-End Latency (E2E)

**Definition:** Time from request received to response delivered (full pipeline).

**For RAG:** includes embedding + retrieval + reranking + LLM + response formatting.

**Typical breakdown:**
- Auth: 10-50ms
- Cache lookup: 5-20ms
- Embedding: 50-200ms (if not cached)
- Retrieval: 100-500ms
- Reranking: 100-300ms
- LLM call: 3-10s (dominant)
- Response formatting: 10-50ms

**Use when:** measuring the full user experience.

**Don't use when:** debugging a specific stage (E2E hides which stage is slow).

---

### Time-to-Interactive (TTI)

**Definition:** Time from user request to user being able to READ and INTERACT with the response.

**Different from TTFT because:**
- TTFT: first token shown
- TTI: enough tokens shown to start reading

**In practice:** TTI ≈ 200-500ms after TTFT (a few sentences visible).

**Use when:** discussing UX quality holistically.

**Don't use when:** measuring backend performance (that's E2E).

---

### 🎯 The Distinction That Matters

For streaming LLM apps, TTFT is what users perceive. Marketing "3s response time" means E2E, but the user's brain says "it started responding in 500ms" (TTFT). Optimize what users actually feel.

---

# Section 5: Cost Terminology

## Cost per Request vs Cost per Token vs Cost per User vs Cost per Conversation

### Cost per Request

**Definition:** Average dollar cost of one API request to your service.

**Includes:** all LLM calls, retrieval, reranking, cache misses, etc. for one user request.

**Typical:** $0.01-$0.10 for chatbot-style apps.

**Use when:** costing individual interactions.

---

### Cost per Token

**Definition:** LLM provider's charge per 1M input or output tokens.

**Not the same as cost per request** — one request has variable tokens.

**Use when:** discussing provider pricing (comparing GPT-4o vs Claude Sonnet).

**Don't use when:** discussing app-level economics (users don't consume tokens directly).

---

### Cost per User

**Definition:** Average total spend per active user over a period.

**Formula:** total cost / active users in period.

**Typical:** $5-50/month per active user for AI-heavy products.

**Use when:** unit economics, pricing decisions ("we can charge $30/month because CAC is $50 and cost/user is $10").

---

### Cost per Conversation

**Definition:** Average cost for a full multi-turn conversation session.

**Different from cost per request because:**
- Conversation has 5-20 turns
- Later turns have longer context → higher cost per request
- Total cost per conversation compounds

**Typical:** $0.20-$2.00 depending on length + model.

**Use when:** designing for multi-turn products (support chatbots).

---

### 🎯 The Distinction That Matters

- Provider talks in cost per token.
- Engineers debug in cost per request.
- Product managers plan in cost per user or cost per conversation.
- CFOs sign off on total monthly LLM spend.

Every level of the org has a different unit. Use the right one for the audience.

---

# Section 6: Evaluation Terminology

## Online Eval vs Offline Eval vs Shadow Eval vs A/B Test

### Offline Eval

**Definition:** Evaluate model/prompt/system on a FIXED test dataset before deployment.

**When:** during development, before production.

**Requires:** labeled dataset with expected outputs (or reference outputs for comparison).

**Metrics:** accuracy, precision, recall, faithfulness on the test set.

**Use when:** validating changes before deploying.

**Limitation:** test set may not represent real production data.

---

### Online Eval

**Definition:** Evaluate model/system on LIVE PRODUCTION data.

**When:** continuously in production.

**Doesn't require labeled data — uses:**
- LLM-as-judge on sampled traffic
- User feedback signals (thumbs up/down)
- Implicit signals (session length, follow-up questions)
- Golden set queries run against production

**Use when:** detecting silent quality regressions in production.

**Limitation:** no ground truth, noisier signals.

---

### Shadow Eval (Shadow Deployment)

**Definition:** Run NEW model alongside CURRENT model on same traffic. Compare outputs. Users see current model's response.

**When:** validating a new model before switching.

**Use when:** low-risk way to test a new model on real production data.

**Metrics:** compare outputs on same inputs — quality, latency, cost side-by-side.

---

### A/B Test

**Definition:** Send a % of production traffic to each variant. Users experience the variant they're assigned. Compare metrics.

**When:** measuring impact of a change on real users.

**Use when:** you want to prove impact on user behavior (engagement, conversion, satisfaction).

**Limitation:** requires enough traffic for statistical significance. Ethics: some users get the (potentially worse) variant.

---

### 🎯 The Distinction That Matters

| Method | When | Users affected? | Ground truth? |
|---|---|---|---|
| **Offline eval** | Pre-deployment | No | Yes (labeled) |
| **Online eval** | Continuous prod | Yes (monitoring only) | No (proxies) |
| **Shadow eval** | Pre-switch | No (shadow only) | No (comparison) |
| **A/B test** | Feature validation | Yes (some see new) | Yes (behavior) |

Full development lifecycle uses ALL FOUR at different stages.

---

# Section 7: Observability Terminology

## Logs vs Metrics vs Traces vs Events vs Spans

### Logs

**Definition:** Discrete, timestamped records of things that happened.

**Format:** structured JSON preferred.

**Cardinality:** unlimited (every log line is unique).

**Use when:** debugging specific events, audit trails.

**Don't use when:** aggregating over time (use metrics for that).

---

### Metrics

**Definition:** Numeric values sampled over time.

**Format:** Prometheus-style (counter, gauge, histogram).

**Cardinality:** LIMITED — label combinations must be bounded.

**Use when:** dashboards, alerts, trends.

**Don't use when:** need per-request detail (use logs/traces).

---

### Traces

**Definition:** Sequence of operations for a single request across services.

**Format:** parent-child spans with timing.

**Cardinality:** unlimited (each trace unique) but usually sampled.

**Use when:** understanding request flow, latency breakdown.

**Don't use when:** aggregating over many requests (use metrics).

---

### Events

**Definition:** Business/domain-significant occurrences.

**Different from logs because:**
- Logs = anything worth recording
- Events = domain-significant milestones (user signed up, purchase made, agent completed task)

**Use when:** business analytics, product tracking.

---

### Spans

**Definition:** A single unit of work within a trace.

**Contains:** name, start/end time, attributes, events, parent span.

**Example spans in an LLM request:**
- auth.verify (10ms)
- cache.lookup (5ms)
- embed.query (150ms)
- vector.search (200ms)
- llm.call (3000ms)
- response.format (20ms)

Total trace = sum of relevant spans.

**Use when:** measuring individual operations within a request.

---

### 🎯 The Distinction That Matters

- Logs: what happened (rich detail, unlimited cardinality)
- Metrics: how often / how much (aggregated, low cardinality)
- Traces: how did this specific request flow (single request lifecycle)
- Events: what business-significant thing occurred
- Spans: individual work unit within a trace

**Never put high-cardinality data (user_id, request_id) in metrics. Never try to aggregate individual events without metrics.**

---

# Section 8: Alert Terminology

## Symptom vs Cause vs Error vs Warning vs SLO Burn

### Symptom Alerts

**Definition:** Alerts on user-visible pain.

**Examples:**
- p95 latency > 2s
- error rate > 5%
- quality score < 0.85

**Use when:** you want alerts to correspond to user impact.

---

### Cause Alerts

**Definition:** Alerts on internal state that MIGHT cause user pain.

**Examples:**
- CPU > 80%
- disk > 90% full
- cache hit rate < 40%

**Use when:** you want early warning before symptoms appear.

**Warning:** cause alerts often false positive. High CPU might be normal load. Use judiciously.

---

### Errors vs Warnings

**Error alerts:** something IS wrong, immediate action needed. Page on-call.

**Warning alerts:** something MIGHT be wrong, investigate when time permits. Slack notification.

**Use error alerts when:** action is required immediately.

**Use warnings when:** you want visibility without disruption.

---

### SLO Burn Rate Alerts

**Definition:** Alert when error budget is depleting faster than sustainable.

**Not the same as error alerts:**
- Error alert: "this specific error happened"
- Burn rate alert: "we're using budget faster than we can afford long-term"

**Use when:** operating under SLOs and want to know when reliability is trending badly (before violation).

---

### 🎯 The Distinction That Matters

Best-practice alerting:
- 90% symptom alerts (page on-call — user pain)
- 5% burn rate alerts (page on-call — SLO risk)
- 5% cause alerts (slack — early warning)

**If your alerts are 80% cause-based, you have alert fatigue guaranteed.**

---

# Section 9: Agent Terminology

## Agent vs Chain vs Workflow vs Trajectory vs Tool Call

### Agent

**Definition:** LLM system that DECIDES what to do next at runtime.

**Key property:** decisions made by LLM, not hardcoded.

---

### Chain

**Definition:** Predetermined sequence of steps.

**Key property:** flow decided at design time.

---

### Workflow

**Definition:** Chain with explicit conditional branching.

**Key property:** decisions made by CODE (if/else), not LLM.

---

### Trajectory

**Definition:** The FULL SEQUENCE of steps an agent took for one task.

**Contains:** thoughts, actions, tool calls, results.

**Use when:** analyzing what an agent did (debugging, evaluation).

---

### Tool Call

**Definition:** A SINGLE step where the agent invoked an external function.

**One trajectory contains many tool calls.**

---

### 🎯 The Distinction That Matters

- Chain executes a plan.
- Workflow decides via code.
- Agent decides via LLM.
- Trajectory is the record of what agent did.
- Tool call is one step within trajectory.

**Interview trap:** Candidates say "we built an agent" when they built a workflow. Different complexity, different failure modes.

---

# Section 10: Deployment Terminology

## Rolling vs Blue-Green vs Canary vs Shadow vs Feature Flag

### Rolling Deploy

**Definition:** Replace instances one at a time.

**Zero-downtime:** yes.
**Rollback speed:** slow (must roll back new version).

---

### Blue-Green

**Definition:** Two identical environments; switch traffic.

**Zero-downtime:** yes.
**Rollback speed:** instant (flip switch back).

---

### Canary

**Definition:** Deploy to small % of traffic, monitor, expand.

**Zero-downtime:** yes.
**Rollback speed:** medium (stop expansion, roll back gradual).

---

### Shadow

**Definition:** Deploy new version, receive traffic COPY, don't return responses.

**Zero-downtime:** yes.
**Rollback speed:** N/A (no user impact).

**Use for:** testing before real deployment.

---

### Feature Flag

**Definition:** Deploy code; enable/disable feature via config.

**Zero-downtime:** yes.
**Rollback speed:** instant (toggle flag).

**Use for:** decoupling deployment from release.

---

### 🎯 The Distinction That Matters

Enterprise best practice uses ALL FIVE:
- Rolling deploys for standard code changes
- Blue-green for critical infrastructure changes
- Canary for risky features
- Shadow for new models/prompts
- Feature flags for controlled release

Each solves a different problem. Not "which is best" — "which for this scenario."

---

## 🎯 The Meta-Lesson

Junior engineers use similar-sounding terms interchangeably. Senior engineers precisely distinguish. Precision in language reveals precision in thinking.

When an interviewer asks "how did you improve retrieval?" — a precise answer is:

> "We changed chunk size from 512 to 1024 tokens, which improved Hit@5 from 0.72 to 0.81. Context precision dropped slightly (0.85 → 0.78) but we compensated with a Cohere reranker that brought precision back to 0.89. Faithfulness rose from 0.78 to 0.86 because retrieved chunks had more complete context."

Every term used correctly. Every claim measurable. **That's the level.**
