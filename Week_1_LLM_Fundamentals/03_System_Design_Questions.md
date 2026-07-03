# 🏗️ Episode 1 — System Design Questions

> **Focus:** LLM serving infrastructure, model selection architecture, cost optimization, latency design
>
> **How to use:** Treat each question as a 30-45 minute whiteboard exercise. Start with requirements gathering, then architecture, then deep-dive into one component.

---

## Q1. Design an LLM Gateway Service ⭐⭐⭐

**Prompt:** "Design a unified API gateway that sits between your application and multiple LLM providers (OpenAI, Anthropic, Google, local models). It should handle routing, fallback, rate limiting, cost tracking, and caching."

**What the interviewer is really testing:** Can you build production infrastructure around LLMs?

**Requirements to clarify:**
- How many requests per second? (Start: 100 RPS, design to scale to 10K)
- Latency budget? (p50 < 500ms for cached, p50 < 5s for uncached)
- Multi-tenant? (Yes — different teams have different budgets)
- Streaming support? (Yes — critical for user-facing applications)

**Architecture:**

```
Client Apps
    │
    ▼
┌─────────────────────────────────────┐
│         LLM Gateway (API Layer)     │
│  ┌─────────┐  ┌──────────────────┐  │
│  │ Auth +   │  │  Rate Limiter    │  │
│  │ Tenant ID│  │ (per-tenant,     │  │
│  │          │  │  per-model)      │  │
│  └─────┬───┘  └────────┬─────────┘  │
│        └───────┬────────┘            │
│                ▼                     │
│  ┌────────────────────────────┐      │
│  │    Request Router           │     │
│  │  - Model selection logic    │     │
│  │  - Complexity classifier    │     │
│  │  - Budget-aware routing     │     │
│  └───────────┬────────────────┘      │
│              ▼                       │
│  ┌────────────────────────────┐      │
│  │    Semantic Cache Layer     │     │
│  │  (Redis + vector similarity)│     │
│  └──────┬────────────┬────────┘      │
│    HIT  │            │ MISS         │
│         ▼            ▼              │
│  Return cached   ┌──────────┐       │
│  response        │ Provider │       │
│                  │ Adapter  │       │
│                  │ Layer    │       │
│                  └─┬──┬──┬─┘       │
│                    │  │  │          │
└────────────────────┼──┼──┼──────────┘
                     │  │  │
              ┌──────┘  │  └──────┐
              ▼         ▼         ▼
          OpenAI   Anthropic   Local
                                (vLLM)
```

**Key design decisions:**

**1. Request Router — the brain:**
- Simple requests (classification, extraction, short answers) → cheap/fast model (GPT-4o-mini, Haiku)
- Complex requests (reasoning, long-form, code generation) → capable model (GPT-4o, Sonnet)
- The classifier itself is a lightweight model or rules-based system
- Respects per-tenant budget: if Team A has used 80% of their daily budget, route to cheaper models

**2. Semantic Cache — the money saver:**
- Two tiers: exact hash match (Redis, O(1)) → semantic similarity match (vector DB, O(log n))
- Cache hit rates of 20-40% are typical and save enormous amounts
- TTL varies by query type: factual questions cached longer than creative tasks
- Cache key includes: model, temperature, system prompt hash (same question with different context = different cache entry)

**3. Provider Adapter Layer — the resilience layer:**
- Normalizes different API formats into a single internal format
- Handles retries with exponential backoff per provider
- Failover chain: if OpenAI is down → Anthropic → local model
- Streaming support via Server-Sent Events (SSE) — must work across all providers

**4. Observability (critical, often forgotten):**
- Log every request: input/output tokens, model, latency, cost, tenant, cache hit/miss
- Dashboards: cost per tenant per day, latency percentiles, error rates by provider
- Alerts: daily spend exceeds threshold, error rate > 5%, latency p99 > 30s

**Scaling considerations:**
- Gateway is stateless — horizontal scaling behind a load balancer
- Cache is shared (Redis cluster)
- Rate limiting uses distributed counters (Redis or token bucket)
- Streaming responses bypass the cache (but log for analytics)

**Follow-up:** "How do you handle a scenario where all LLM providers are down?"

**Answer:** (1) Return cached responses for similar queries with a "stale data" flag. (2) For new queries, return a graceful degradation response: "Our AI system is temporarily unavailable. Your request has been queued." (3) Queue requests in a message broker (SQS, Kafka) for processing when providers recover. (4) Alert on-call team. This is the same pattern as any distributed system — LLMs are just another external dependency.

---

## Q2. Design a Real-Time AI Content Moderation System ⭐⭐⭐⭐

**Prompt:** "Design a system that moderates user-generated content (social media posts, comments, chat messages) using LLMs. It needs to handle 50K messages per minute with <2 second latency for blocking decisions."

**Requirements:**
- Must catch: hate speech, harassment, self-harm content, spam, misinformation
- False positive rate must be < 1% (blocking legitimate content is very costly)
- Must support multiple languages
- Must handle adversarial inputs (users trying to bypass moderation)
- Appeals process required

**Architecture:**

```
User Content
    │
    ▼
┌───────────────────────────────┐
│  Tier 1: Fast Classifiers      │   < 50ms
│  - Keyword/regex blocklist     │
│  - Lightweight ML classifier   │
│  - Language detection           │
│  Result: BLOCK / PASS / REVIEW │
└──────────┬─────────┬───────────┘
     PASS   │  REVIEW │  BLOCK
      │     │         │     │
      ▼     ▼         │     ▼
  Publish  ┌─────────┐│  Block +
           │ Tier 2:  ││  Log
           │ LLM      ││
           │ Analysis  ││   < 2s
           └────┬─────┘│
          SAFE  │ UNSAFE│
           │    │       │
           ▼    ▼       ▼
        Publish Block  Block +
                 +      Escalate
                Log     to human
```

**Key design decisions:**

**Tier 1 (fast path):** Traditional ML classifiers (distilled BERT, fastText) that run in <50ms. These catch 80% of violations with high confidence. Low confidence → escalate to Tier 2.

**Tier 2 (LLM-powered):** For ambiguous cases, use an LLM with structured output:
```json
{
  "decision": "block",
  "category": "harassment",
  "confidence": 0.87,
  "reasoning": "Targets individual based on protected characteristic",
  "severity": "high"
}
```

**Why not LLM for everything?** At 50K messages/minute:
- Tier 1 only: $0 per message (self-hosted classifier)
- LLM for all: ~$0.001-0.01/message = $50-500/minute = $72K-720K/day
- Tiered: LLM only for ~20% of messages = $10-100/minute = practical

**Adversarial robustness:**
- Unicode normalization (catch homoglyph attacks: using Cyrillic 'а' instead of Latin 'a')
- Image-in-text detection (screenshots of text to bypass text classifiers)
- Paraphrase detection (same meaning, different words)
- Regular red-team testing with adversarial prompt engineers

**Appeals flow:**
User appeals → LLM re-review with human-in-the-loop → final decision within 24 hours. Log all appeals to improve classifiers.

---

## Q3. Design an LLM-Powered Search System ⭐⭐⭐

**Prompt:** "You're building a search system for a company's internal knowledge base (100K documents — Confluence pages, Slack threads, PDFs, code repos). Users should be able to ask natural language questions and get accurate answers with citations."

**This is a RAG system design question — the most common AI system design prompt in interviews.**

**Architecture:**

```
┌──────────────────────────────────────────────┐
│                  Ingestion Pipeline            │
│                                                │
│  Documents → Chunking → Embedding → Vector DB  │
│     │                                    │     │
│     └── Metadata extraction ────────────┘     │
│         (author, date, source, permissions)    │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│                  Query Pipeline                │
│                                                │
│  User Query                                    │
│     │                                          │
│     ├── Query Expansion (LLM rewrites query)   │
│     │                                          │
│     ├── Hybrid Retrieval                       │
│     │   ├── Vector search (semantic)           │
│     │   └── BM25 / keyword search (exact)      │
│     │                                          │
│     ├── Re-ranking (cross-encoder)             │
│     │                                          │
│     ├── Context Assembly (top-k chunks)        │
│     │                                          │
│     ├── LLM Generation (with citations)        │
│     │                                          │
│     └── Response + Source Links                │
└──────────────────────────────────────────────┘
```

**Critical design decisions:**

**Chunking strategy:**
- Not one-size-fits-all: code → by function/class, docs → by section/paragraph, Slack → by thread
- Chunk size: 256-512 tokens with 50-token overlap (overlap prevents losing context at boundaries)
- Each chunk stores: text, embedding, source URL, parent document ID, position, metadata

**Hybrid retrieval (this is what separates junior from senior answers):**
- Vector search alone misses exact matches ("error code E4021" → semantic search won't find this)
- Keyword search alone misses semantic understanding ("how to fix authentication" → won't match "resolve login issues")
- Combine both with Reciprocal Rank Fusion (RRF): `score = Σ 1/(k + rank_i)` across search methods

**Re-ranking:**
- Retrieval gives you 50 candidates; re-ranking picks the best 5
- Cross-encoder models (e.g., Cohere Rerank) are more accurate than bi-encoder similarity but too slow for first-pass retrieval
- This two-stage approach (fast retrieval → accurate re-ranking) is standard in production

**Permission-aware search:**
- User A should not see documents from Team B's private Confluence space
- Index permission metadata alongside content
- Filter at retrieval time, not generation time (don't even retrieve unauthorized content)

**Evaluation:**
- Retrieval metrics: Recall@k, MRR (Mean Reciprocal Rank), NDCG
- Generation metrics: Faithfulness (does answer match retrieved context?), relevance, completeness
- Track "I don't know" rate — the system SHOULD say "I don't have information about this" for out-of-scope queries

---

## Q4. Design a Multi-Agent AI System for Customer Support ⭐⭐⭐⭐

**Prompt:** "Design an AI customer support system that handles email and chat. It should autonomously resolve 70% of tickets, escalate the rest to humans with full context, and continuously improve."

**Architecture:**

```
Customer Message
       │
       ▼
┌──────────────────┐
│  Intent Classifier │  (What does the customer want?)
│  + Sentiment       │  (How urgent/frustrated are they?)
└────────┬───────────┘
         │
    ┌────┴────┐
    ▼         ▼
  Simple    Complex
  (FAQ,      (billing dispute,
   status,    bug report,
   reset)     feature request)
    │              │
    ▼              ▼
┌────────┐   ┌──────────────┐
│ Direct  │   │  Orchestrator │
│ Answer  │   │  Agent        │
│ Agent   │   │               │
└────────┘   │  ┌──────────┐ │
              │  │ Knowledge│ │  ← RAG over docs
              │  │ Agent    │ │
              │  ├──────────┤ │
              │  │ Action   │ │  ← Tool calling (CRM, billing API)
              │  │ Agent    │ │
              │  ├──────────┤ │
              │  │ Draft    │ │  ← Response generation
              │  │ Agent    │ │
              │  └──────────┘ │
              └───────┬───────┘
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
      Auto-resolve         Escalate to
      + send response      human agent
      + log resolution     + full context
                           + suggested response
```

**Critical design decisions:**

**Infinite loop prevention (from your example):**
- Maximum 5 tool calls per conversation turn
- Maximum 3 retry attempts per tool call
- Timeout after 30 seconds of agent processing
- If the orchestrator calls the same tool with the same arguments twice → break and escalate
- Circuit breaker pattern: if >10% of requests in the last 5 minutes hit the loop limit → alert + switch to human-first mode

**Confidence-gated autonomy:**
- Agent outputs a confidence score with every response
- confidence > 0.9 → send automatically
- 0.7 < confidence < 0.9 → human reviews before sending (suggested response)
- confidence < 0.7 → full escalation to human
- Start conservative (higher thresholds), lower as system proves reliability

**Continuous improvement flywheel:**
- Log every auto-resolved ticket + customer satisfaction score
- Human agents edit AI-suggested responses → these edits become fine-tuning data
- Weekly eval: accuracy, resolution rate, customer satisfaction
- A/B test prompt changes against the eval suite before deploying

---

## Q5. Design a Cost-Optimized LLM Pipeline for a Startup ⭐⭐⭐

**Prompt:** "You're the first AI engineer at a startup. You have a $5K/month LLM API budget. Design a system that serves 100K user queries per day while staying within budget."

**Budget math first:**
- $5K/month = ~$167/day
- 100K queries/day = $0.00167 per query budget
- GPT-4o at ~$5/1M input tokens: you can afford ~33M input tokens/day
- That's ~330 tokens per query average — very tight for RAG + long system prompts

**Cost optimization strategies:**

**1. Prompt compression (saves 30-50%):**
- Strip unnecessary whitespace and formatting from system prompts
- Use abbreviations in system prompts (the model understands them fine)
- Compress retrieved RAG context before injecting

**2. Response caching (saves 20-40%):**
- Exact + semantic cache (as designed in Q1)
- Cache common queries: "How do I reset my password?" hits cache after the first call
- Even a 25% cache hit rate saves $1,250/month

**3. Model routing (saves 40-60%):**
- Simple queries (80% of traffic) → GPT-4o-mini (~$0.15/1M input) = 40x cheaper
- Complex queries (20%) → GPT-4o
- A $0.001 classifier call to route saves $0.01+ on each simple query

**4. Batch processing for non-real-time tasks:**
- Anthropic and OpenAI offer batch APIs at 50% discount
- Queue summarization, classification, and analytics tasks for batch processing
- Only use real-time API for user-facing, latency-sensitive queries

**5. Output length control:**
- Set max_tokens appropriately per query type (don't default to 4096 for a yes/no question)
- Use structured outputs (JSON) to keep responses concise
- Output tokens cost 2-4x more than input tokens

---

## Q6. Design an Eval Framework for an LLM Application ⭐⭐⭐⭐

**Prompt:** "Your team ships LLM features weekly. How do you design an evaluation system that prevents regressions, catches quality degradation, and gives confidence to ship?"

**This is arguably the most important system design question for AI engineers.**

**Architecture:**

```
Code Change / Prompt Change / Model Change
              │
              ▼
┌──────────────────────────────┐
│        Eval Pipeline          │
│                                │
│  1. Unit Evals                 │  (individual test cases)
│  2. Slice Evals                │  (performance by category)
│  3. Regression Evals           │  (compare against baseline)
│  4. Human Evals (sampled)      │  (for subjective quality)
│  5. A/B Evals (production)     │  (real user impact)
│                                │
│  Gate: All metrics ≥ baseline? │
│        YES → Ship              │
│        NO → Block + alert      │
└──────────────────────────────┘
```

**Eval types in detail:**

**Unit evals:** Individual test cases with expected outputs.
```python
{"input": "What's 2+2?", "expected": "4", "metric": "exact_match"}
{"input": "Summarize: ...", "expected_keywords": ["key1", "key2"], "metric": "keyword_coverage"}
```

**Slice evals:** Group test cases by category and track per-slice performance:
- By topic: billing questions vs technical questions vs general inquiries
- By difficulty: simple vs complex
- By language: English vs Spanish vs Mandarin
- By edge case type: adversarial, ambiguous, out-of-scope

**Regression evals:** Compare new version against production baseline:
- Same inputs → are outputs ≥ as good?
- Use LLM-as-judge: "Which response is better, A or B?" with blind evaluation
- Track: win rate, tie rate, loss rate. Ship if win rate > loss rate.

**Key metrics:**
- Accuracy/correctness (factual tasks)
- Faithfulness (RAG: does the answer match the retrieved context?)
- Relevance (does it answer the question asked?)
- Harmlessness (safety violations?)
- Latency (did performance degrade?)
- Cost (did cost increase?)

**Critical insight the interviewer wants to hear:** Evals are not one-time — they're a continuously maintained dataset that grows with every production failure. When a user reports a bad output, that input/output pair becomes a new eval case. The eval suite is a living artifact, as important as the codebase.

---

## Q7. Design Token-Level Streaming for a Chat Interface ⭐⭐

**Prompt:** "Design the full stack for real-time LLM streaming from the API through to the browser. Users should see tokens appear in real-time like ChatGPT."

**Architecture:**

```
Browser (React)  ←  SSE/WebSocket  ←  Backend (FastAPI)  ←  Stream  ←  LLM API
    │                                       │
    │  Progressive rendering                │  Buffer + flush
    │  Markdown parsing                     │  Token counting
    │  Code block detection                 │  Cost tracking
    │  "Typing" indicator                   │  Timeout handling
```

**Backend (FastAPI with SSE):**

```python
@app.post("/chat/stream")
async def stream_chat(request: ChatRequest):
    async def event_generator():
        async for chunk in llm_client.stream(request.messages):
            token = chunk.choices[0].delta.content or ""
            yield {
                "event": "token",
                "data": json.dumps({"text": token})
            }
        yield {"event": "done", "data": "{}"}
    
    return EventSourceResponse(event_generator())
```

**Frontend considerations:**
- Accumulate tokens into a buffer; render markdown progressively
- Detect code blocks (triple backticks) — don't render markdown inside them
- Handle connection drops: reconnect with a "continue from token N" mechanism
- Show a typing indicator that becomes the actual response
- Parse and render markdown only after a paragraph break (not mid-sentence)

**Edge cases to handle:**
- User sends a new message while previous is still streaming → cancel previous stream
- Network interruption mid-stream → detect via heartbeat, offer retry
- Model produces an extremely long response → show a "stop generating" button
- Multiple concurrent streams (if your UI supports it) → manage via request IDs

---

## Q8. Design a Prompt Management and Versioning System ⭐⭐⭐

**Prompt:** "Your team has 50 different prompts across 10 features. Different team members edit prompts. How do you manage, version, test, and deploy prompts safely?"

**Architecture:**

```
Prompt Authors
     │
     ▼
┌─────────────────┐
│  Prompt Registry  │  (Git-backed, version-controlled)
│  ├── feature_a/   │
│  │   ├── v1.txt   │
│  │   ├── v2.txt   │
│  │   └── meta.yml │  (model, temp, max_tokens, owner)
│  ├── feature_b/   │
│  └── ...           │
└────────┬──────────┘
         │
    PR / Review
         │
         ▼
┌─────────────────┐
│  Eval Pipeline    │  (runs on every prompt change)
│  - Unit tests     │
│  - Regression     │
│  - Cost estimate  │
└────────┬──────────┘
    PASS │
         ▼
┌─────────────────┐
│  Deployment       │
│  - Canary (5%)    │
│  - Monitor        │
│  - Full rollout   │
│  - Rollback if    │
│    metrics drop   │
└─────────────────┘
```

**Key principles:**
- Prompts are code — version them, review them, test them, deploy them with the same rigor
- Every prompt change runs the eval suite before deployment
- Canary deployment: new prompt serves 5% of traffic, monitor quality metrics, roll forward or back
- Rollback is instant: point to the previous version in the registry
- Metadata per prompt: model, temperature, max_tokens, owner, last test results, production metrics
