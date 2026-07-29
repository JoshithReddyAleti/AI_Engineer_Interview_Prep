# 🧠 Week 5 — Deep Conceptual Questions

> **Focus:** RAG architecture, all 5 augmented generation types, 9 RAG variants, chunking, embeddings, vector DBs, retrieval, reranking, evaluation
>
> **How to use:** RAG is the #1 interview topic for AI engineers in 2025-2026. Every question here has been asked in real interviews.

---

## Q1. Explain the complete RAG architecture. How would you build it from scratch? ⭐⭐⭐

**What the interviewer is really testing:** Can you explain the full pipeline, not just "I used LangChain"?

**Answer:**

RAG has two pipelines — offline ingestion and online query:

**Offline Ingestion Pipeline (runs once, or on document updates):**
```
Documents (PDFs, web pages, DBs)
    → Document Loader (extract raw text)
    → Chunking Engine (split into passages — 512 tokens, 50-token overlap)
    → Embedding Model (convert chunks to vectors — e.g., text-embedding-3-small)
    → Vector Database (store vector + text + metadata — e.g., FAISS, Pinecone)
```

**Online Query Pipeline (runs on every user question):**
```
User Query
    → Query Transformation (optional: rewrite, expand, HyDE)
    → Embedding Model (convert query to vector)
    → Vector Search (retrieve top-K similar chunks)
    → Reranker (optional: cross-encoder re-scores for precision)
    → Context Assembly (format chunks with source metadata)
    → LLM Generation (generate answer grounded in context)
    → Validation (faithfulness check, format check, safety)
    → Response with citations
```

**Why each step exists:**
- **Chunking:** LLMs have context limits. You can't stuff 10,000 pages into one prompt. Chunks make documents retrievable at the passage level.
- **Embeddings:** Convert text to numbers so you can compute similarity. "What is the refund policy?" and "Returns within 30 days for full refund" have high vector similarity.
- **Vector search:** Find the chunks most relevant to the question. O(log n) with HNSW index.
- **Reranking:** Vector similarity is approximate. A cross-encoder scores (query, document) pairs more accurately but is too slow to run against 1M documents — so you use it on the top-50 from vector search.
- **Context assembly:** Format the retrieved chunks clearly for the LLM with source attribution.
- **Generation:** The LLM reads the context and generates an answer grounded in it.
- **Validation:** Catches hallucination, format errors, and safety violations before the response reaches the user.

**Follow-up:** "What's the most common failure point?" → Retrieval. If you retrieve the wrong chunks, the LLM generates from wrong context. Everything downstream is corrupted.

---

## Q2. Explain all 5 types of augmented generation. When do you use each? ⭐⭐⭐⭐

**What the interviewer is really testing:** Do you know the full landscape, or just RAG?

**Answer:**

Most people only know RAG. There are actually 5 types:

**RAG (Retrieval-Augmented Generation):**
Retrieve relevant documents from an external corpus at query time → inject into LLM context → generate.
Use when: Large, dynamic knowledge base. The default for most enterprise AI.

**CAG (Cache-Augmented Generation):**
Preload ALL context into the model's extended context window or KV-cache. No retrieval step — the model sees everything upfront.
Use when: Corpus is small (<100 pages) and static. Avoids retrieval pipeline complexity. Think: product manual Q&A, company handbook chat.

**MAG (Memory-Augmented Generation):**
LLM reads and writes to an external memory store that persists across sessions. Short-term (conversation buffer) + long-term (user preferences, facts learned over time).
Use when: Personal assistants that learn about users over time. "You mentioned you're vegetarian last month" — that's MAG.

**TAG (Tool-Augmented Generation):**
LLM calls tools (APIs, calculators, databases, code interpreters) at inference time to get real-time data.
Use when: The answer requires live data (stock prices, weather), computation (math), or actions (sending email). This is what Episodes 3-4 covered.

**KAG (Knowledge-Augmented Generation):**
LLM queries a structured knowledge graph (entities + relationships) instead of or alongside unstructured text retrieval.
Use when: The question requires relational reasoning ("Which suppliers of Company X also supply Company Y?"). Graph traversal finds connections that vector similarity misses.

**Decision framework:**
```
Is the data structured (DB/API)?          → TAG (call it directly)
Is the corpus small and static?           → CAG (preload into context)
Does the user need personal memory?       → MAG (persistent memory)
Do you need entity relationships?         → KAG (knowledge graph)
Large unstructured corpus, dynamic?       → RAG (retrieve and generate)
Complex query needing multiple types?     → Combine them
```

---

## Q3. Explain all 9 RAG variants. When would you use each? ⭐⭐⭐⭐

**What the interviewer is really testing:** Deep RAG expertise. This is a staff-level question.

**Answer:**

**1. Naive RAG** — The baseline. Embed query → vector search → stuff top-K into prompt → generate. Simple but: no query optimization, no reranking, no verification. Precision drops on complex queries.

**2. Advanced RAG** — Adds pre-retrieval optimization (query rewriting, HyDE, expansion), hybrid search (vector + BM25 with RRF fusion), and post-retrieval reranking (cross-encoder). The production standard for most applications.

**3. Graph RAG** — Extracts entities and relationships from documents into a knowledge graph. Retrieves via graph traversal + optional vector boost. Excels at multi-hop relational queries.

**4. Agentic RAG** — An agent decides the retrieval strategy per query. Simple queries → single retrieval. Complex queries → decompose, multi-step retrieve, iterate. The agent adapts instead of following a fixed pipeline.

**5. Self-RAG** — The model DECIDES whether it needs retrieval or can answer from its own knowledge. Avoids unnecessary retrieval for simple factual questions (saving cost and latency), retrieves only when needed.

**6. Corrective RAG (CRAG)** — After retrieval, a verification step checks each chunk for relevance and groundedness. Irrelevant chunks are discarded. If all chunks fail verification, falls back to web search or abstention.

**7. Adaptive RAG** — Routes queries to different pipelines based on complexity: no-retrieval (simple factual), single-pass RAG (standard), or multi-hop iterative RAG (complex multi-document). A query classifier selects the path.

**8. Multimodal RAG** — Retrieves and reasons over text, images, tables, and charts together. Converts non-text modalities to text descriptions for embedding, stores original images for multimodal LLM generation.

**9. Hybrid RAG** — Combines vector search + keyword search (BM25) + optional graph traversal in one pipeline. Uses RRF (Reciprocal Rank Fusion) to merge ranked results. Catches what any single method alone misses.

**Selection framework:**

| If you need... | Use... |
|---|---|
| Quick prototype | Naive RAG |
| Production-quality retrieval | Advanced RAG |
| Entity/relationship queries | Graph RAG |
| Varying query complexity | Agentic or Adaptive RAG |
| Cost optimization | Self-RAG |
| Maximum accuracy in high-stakes | CRAG |
| Images, tables, charts | Multimodal RAG |
| Both keyword and semantic match | Hybrid RAG |

---

## Q4. Explain chunking strategies: Fixed, Recursive, Semantic, Parent-Child, and Hybrid. ⭐⭐⭐

**What the interviewer is really testing:** Do you understand that bad chunking is the #1 cause of bad RAG?

**Answer:**

**Fixed-Size:** Split every N tokens with M token overlap. Fast, predictable sizes. But: splits mid-sentence, mid-paragraph, mid-thought. Relevant context gets severed.

**Recursive Character Splitting:** Split by hierarchy — first try paragraphs (`\n\n`), then sentences (`. `), then words (` `). Respects natural document structure. The most commonly used strategy.

**Semantic:** Embed each sentence. Group consecutive sentences with high similarity into chunks. Drop in similarity triggers a chunk boundary. Result: topically coherent chunks. Cost: requires embedding every sentence (expensive for large corpora).

**Parent-Child (Hierarchical):** Store small chunks (256 tokens) for precise retrieval, but when a small chunk matches, return its larger parent chunk (1024 tokens) for richer context. Best of both worlds: search precision + context richness.

**Hybrid:** Combine strategies. Common pattern: split by document structure (headers/sections) → within each section, use recursive splitting at ~512 tokens → add metadata (section title, page, source).

**The interview killer follow-up:** "Why did you choose THIS chunking strategy?"
→ "I chose recursive splitting at 512 tokens with 50-token overlap because our documents are structured prose (not tables or code), our embedding model (text-embedding-3-small) was trained on ~500-token passages, and our eval showed 512 outperformed 256 and 1024 on retrieval precision by 8-12%."

That answer demonstrates: awareness of embedding model training, empirical evaluation, and principled decision-making.

---

## Q5. What are embeddings? How do you choose an embedding model? ⭐⭐

**What the interviewer is really testing:** Practical selection, not just theory.

**Answer:**

Embeddings convert text into dense vectors in a high-dimensional space where semantic similarity equals spatial proximity.

**Selection criteria:**

1. **Domain fit:** General embeddings work for most tasks. For code, legal, or medical domains, test domain-specific models against general ones on YOUR queries.

2. **Dimensionality vs storage:** Higher dims = more expressive but more storage and slower search. 1024-1536 is the sweet spot. 384 (MiniLM) is fine for prototyping.

3. **Speed vs accuracy:** Local models (Sentence Transformers) = zero marginal cost. API models (OpenAI) = per-token cost but higher quality.

4. **Critical rule:** Once you embed a corpus with model X, you cannot query it with model Y. Embedding spaces are incompatible. Changing models = re-embedding everything.

| Model | Dims | Best For |
|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536 | General purpose, cost-effective |
| `text-embedding-3-large` (OpenAI) | 3072 | Higher accuracy |
| `voyage-3` (Voyage AI) | 1024 | Code and technical docs |
| `all-MiniLM-L6-v2` (open source) | 384 | Prototyping, self-hosted, low-latency |
| `bge-large-en-v1.5` (BAAI) | 1024 | Self-hosted production |
| `Cohere embed-v3` | 1024 | Multilingual |

---

## Q6. Compare vector databases: FAISS, Pinecone, Chroma, Milvus, Weaviate, pgvector. ⭐⭐⭐

**What the interviewer is really testing:** Can you justify your choice for a given use case?

**Answer:**

| | FAISS | Chroma | Pinecone | Milvus | Weaviate | pgvector |
|---|---|---|---|---|---|---|
| **Type** | Library | Embedded | Managed | Self-hosted | Self-hosted | PostgreSQL extension |
| **Scale** | 1B+ (single machine) | ~1M | 1B+ | 1B+ | 100M+ | 10M (practical) |
| **Managed** | No | No | Yes | Zilliz Cloud | Weaviate Cloud | Any Postgres host |
| **Setup** | `pip install` | `pip install` | API key | Docker/K8s | Docker/K8s | `CREATE EXTENSION` |
| **Best for** | Prototyping, research | Small apps | Production SaaS | Enterprise scale | Multimodal | Already using PostgreSQL |
| **Key advantage** | Fastest search, free | Great DX | Zero-ops | Most features | Multimodal native | No new infrastructure |

**Decision shortcut:**
- Prototyping → FAISS
- Small app, simple setup → Chroma
- Production, zero-ops → Pinecone
- Enterprise, self-hosted → Milvus
- Already have PostgreSQL → pgvector
- Multimodal search → Weaviate

---

## Q7. What is HyDE (Hypothetical Document Embedding)? When does it help? ⭐⭐⭐

**What the interviewer is really testing:** Advanced retrieval technique knowledge.

**Answer:**

The problem: the user's query uses different language than the documents. "How do I fix a broken deployment?" won't match a document titled "Deployment Troubleshooting Guide: Common Rollback Procedures" because the words don't overlap well in embedding space.

**HyDE solution:** Instead of embedding the query directly, first ask the LLM to generate a hypothetical answer, then embed THAT answer to find similar real documents.

```python
# Step 1: Generate hypothetical answer
hypothetical = llm.generate(
    "Write a paragraph answering this question: How do I fix a broken deployment?"
)
# "When a deployment fails, first check the error logs in your CI/CD pipeline.
#  Common issues include failed health checks, misconfigured environment variables..."

# Step 2: Embed the hypothetical answer (not the original query)
embedding = embed(hypothetical)

# Step 3: Search — the hypothetical is written in the SAME language as real docs
results = vector_search(embedding, top_k=5)
# Now "Deployment Troubleshooting Guide" appears in results!
```

**When it helps:** Queries that are short, vague, or use different vocabulary than the documents. HyDE bridges the vocabulary gap because the hypothetical answer uses document-style language.

**When it hurts:** Factual lookup queries ("What is the phone number for customer support?"). The LLM generates a hypothetical phone number, and the embedding of a fake number might not match the real document. Also adds latency (one extra LLM call per query).

---

## Q8. What is hybrid search? How does RRF (Reciprocal Rank Fusion) work? ⭐⭐⭐

**What the interviewer is really testing:** Understanding why vector search alone isn't enough.

**Answer:**

**Hybrid search** combines vector (semantic) search with keyword (BM25/sparse) search. Each catches what the other misses.

Vector search excels at: semantic similarity, synonym handling, meaning-based matching.
Vector search fails at: exact term matching (error codes, product SKUs, proper nouns).

BM25 keyword search excels at: exact token matches, rare terms, specific identifiers.
BM25 fails at: paraphrasing, synonyms, conceptual similarity.

**RRF formula:**
```
RRF_score(doc) = Σ  1 / (k + rank_in_system_i)
```
Where `k` is a constant (typically 60).

**Worked example:**
```
Vector search: [Doc A: rank 1, Doc B: rank 3, Doc C: rank 2]
BM25 search:   [Doc B: rank 1, Doc C: rank 2, Doc A: rank 5]

RRF (k=60):
Doc A: 1/(60+1) + 1/(60+5) = 0.0164 + 0.0154 = 0.0318
Doc B: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323 ← Winner
Doc C: 1/(60+2) + 1/(60+2) = 0.0161 + 0.0161 = 0.0322

Final ranking: B, C, A
```

Doc B wins because it ranked well in BOTH systems. RRF naturally boosts documents that appear in multiple retrieval methods.

**Typical improvement:** Hybrid search improves retrieval precision by 5-15% over pure vector search.

---

## Q9. Bi-encoder vs Cross-encoder — what's the difference and when do you use each? ⭐⭐⭐

**What the interviewer is really testing:** The accuracy/speed trade-off in reranking.

**Answer:**

**Bi-encoder:** Encodes query and document INDEPENDENTLY. Pre-compute document embeddings. At query time, embed the query and compute cosine similarity. Fast (milliseconds for millions of docs). Approximate accuracy.

**Cross-encoder:** Encodes query and document TOGETHER as one input. Sees the full interaction between them. Slow (must score each pair individually). Much more accurate.

**Production pattern — two-stage retrieval:**
```
Stage 1: Bi-encoder search → 1M documents → top-50 candidates (50ms)
Stage 2: Cross-encoder rerank → 50 candidates → top-5 results (200-500ms)
Total: ~300-550ms for highly accurate results
```

**Impact:** Cross-encoder reranking improves precision@5 by 10-25% over bi-encoder alone.

---

## Q10. How do you evaluate a RAG application? What metrics matter? ⭐⭐⭐⭐

**What the interviewer is really testing:** Eval is the #1 gap in most candidates' knowledge.

**Answer:**

**Retrieval metrics (is the right context being found?):**
- **Context Precision:** Of retrieved chunks, how many were actually relevant?
- **Context Recall:** Of all relevant chunks in the corpus, how many did we find?
- **MRR (Mean Reciprocal Rank):** How high is the first relevant result?

**Generation metrics (is the answer correct and grounded?):**
- **Faithfulness:** Does every claim in the answer come from the retrieved context? (Not from the model's memory)
- **Answer Relevancy:** Does the answer actually address the question?
- **Completeness:** Are all parts of the question answered?

**Frameworks:**
- **RAGAS:** Open-source, LLM-as-judge metrics. Computes faithfulness, relevancy, precision, recall automatically.
- **DeepEval:** Python-first, pytest-integrated. Built-in hallucination, toxicity, bias metrics.

**Building a golden eval set:**
1. Collect 200+ real user queries
2. Human-label: relevant chunks + correct answers
3. Run eval pipeline before every deploy
4. Block deploys that drop below thresholds

**The metric that gets you hired:**
❌ "Built RAG pipeline with LangChain and Pinecone"
✅ "Improved faithfulness from 0.72 to 0.91 and reduced hallucination rate from 34% to 8% on a 50K-doc corpus"

---

## Q11. How do you prevent hallucinations in RAG? ⭐⭐⭐

**What the interviewer is really testing:** Production safety thinking.

**Answer:**

RAG hallucination = the model generates information NOT in the retrieved context.

**Prevention layers:**

1. **Better retrieval:** More relevant chunks → less temptation to fill gaps. Add reranking.
2. **Explicit grounding instructions:** "Answer ONLY from the provided context. If insufficient, say 'I don't have enough information.'"
3. **Citation enforcement:** Force the model to cite specific passages for every claim. Fabricated claims won't have valid citations.
4. **Faithfulness validation:** Post-generation check — does every claim in the response exist in the context? Use NLI models or LLM-as-judge.
5. **Temperature = 0** for factual tasks.
6. **Shorter context:** Don't stuff 20 chunks. 5 highly relevant chunks > 20 marginally relevant ones.

---

## Q12. What is Self-RAG? How does the model decide when to retrieve? ⭐⭐⭐⭐

**What the interviewer is really testing:** Knowledge of cutting-edge RAG variants.

**Answer:**

Standard RAG retrieves on EVERY query — even "What is 2+2?" where retrieval is wasteful. Self-RAG adds a decision layer: the model first assesses whether it needs external information.

**Flow:**
```
User query arrives
    → Model self-assessment: "Can I answer this from my training data?"
    → If YES (high confidence) → generate directly (skip retrieval, save cost)
    → If NO (low confidence) → retrieve → generate from context
    → After generation: self-evaluate groundedness
    → If grounded → return answer
    → If not grounded → retrieve again with refined query → regenerate
```

**Implementation signals:**
- Special tokens trained into the model: `[Retrieve]`, `[No Retrieve]`, `[IsGrounded]`, `[NotGrounded]`
- Or: use a separate classifier to decide retrieval necessity
- Or: prompt-based — ask the model to rate confidence before retrieval

**When to use:** High-volume systems where 50%+ of queries are simple enough to answer without retrieval. Self-RAG cuts cost and latency on those queries while maintaining quality on complex ones.

---

## Q13. What is Corrective RAG (CRAG)? Why verify after retrieval? ⭐⭐⭐

**What the interviewer is really testing:** Defensive architecture thinking.

**Answer:**

Standard RAG assumes retrieved chunks are relevant. They often aren't — vector similarity is approximate, and marginally related chunks pollute the context.

CRAG adds a verification step AFTER retrieval, BEFORE generation:

```
Retrieve top-K chunks
    → For each chunk: "Is this chunk relevant to answering the query?"
    → Score: Correct (relevant) / Ambiguous / Incorrect (irrelevant)
    → If most chunks = Correct → proceed to generation
    → If most chunks = Ambiguous → refine query, re-retrieve
    → If most chunks = Incorrect → fall back to web search or abstain
```

**Why this matters:** In production, ~20-30% of retrieved chunks are marginally relevant (high cosine similarity but don't actually answer the question). CRAG filters them out before the LLM sees them, dramatically reducing hallucination.

**Cost:** One extra LLM call per retrieval (to evaluate chunk relevance). Worth it in high-stakes applications (legal, medical, financial).

---

## Q14. What is Agentic RAG? How is it different from standard RAG? ⭐⭐⭐

**What the interviewer is really testing:** Understanding of adaptive retrieval.

**Answer:**

Standard RAG: fixed pipeline. Every query → same embedding → same search → same top-K → same generation.

Agentic RAG: an agent decides HOW to retrieve based on the query:

```python
# The agent's decision process
if query_is_simple(query):
    # "What is the refund policy?" → single vector search, top-5
    return single_pass_rag(query, top_k=5)

elif query_needs_decomposition(query):
    # "Compare Q3 vs Q4 revenue and explain the driver" → decompose into sub-queries
    sub_queries = decompose(query)  # ["Q3 revenue", "Q4 revenue", "revenue drivers"]
    chunks = [retrieve(sq) for sq in sub_queries]
    return synthesize(query, merge(chunks))

elif query_needs_structured_data(query):
    # "How many orders last month?" → SQL query, not vector search
    return sql_rag(query)

elif query_needs_web_search(query):
    # "What's the latest news about..." → web search, not local corpus
    return web_search_rag(query)
```

**Key insight:** The agent adapts the retrieval strategy to the query. Simple queries get fast, cheap retrieval. Complex queries get multi-step, thorough retrieval. The fixed pipeline treats every query the same — wasteful for simple queries, insufficient for complex ones.

---

## Q15. Explain MMR (Maximum Marginal Relevance). Why is diversity important in retrieval? ⭐⭐⭐

**What the interviewer is really testing:** Understanding beyond basic cosine similarity.

**Answer:**

Without MMR, top-5 results might all be from the same paragraph (because similar text has similar embeddings). You get 5 copies of the same information.

MMR balances relevance AND diversity:
```
MMR_score(chunk) = λ × sim(chunk, query) - (1-λ) × max_sim(chunk, already_selected)
```

- `λ = 1.0`: Pure relevance (default vector search — no diversity)
- `λ = 0.5`: Balanced — relevant AND diverse
- `λ = 0.7`: Slightly favor relevance (common production setting)

**Impact:** Instead of 5 chunks all saying "Revenue grew 23%," you get: revenue, expenses, forecast, CEO quote, market conditions. Much richer context for the LLM.

---

## Q16. When should you use RAG vs fine-tuning vs long context? ⭐⭐⭐

**What the interviewer is really testing:** Architectural decision-making at the system level.

**Answer:**

| | RAG | Fine-Tuning | Long Context |
|---|---|---|---|
| **Best for** | Large, dynamic corpus | Changing model behavior/style | Small, static corpus |
| **Updates** | Add docs instantly | Retrain (hours/days) | Update context window |
| **Cost** | Per-query retrieval | Training cost upfront | High per-query token cost |
| **Citations** | Natural (source tracking) | Impossible | Difficult |
| **Hallucination** | Controllable (grounding) | Can worsen | Moderate (lost in middle) |
| **Latency** | +200-500ms (retrieval) | Baseline | Baseline (but slow for 200K+) |

**Decision:** Use RAG when knowledge changes, needs citations, or corpus is large. Fine-tune when you need to change HOW the model writes (style, format, domain terminology). Use long context when corpus is <50 pages and static.

---

## Q17. What is KV Cache in the context of RAG? How does prompt caching reduce cost? ⭐⭐⭐

**What the interviewer is really testing:** Inference optimization knowledge.

**Answer:**

When you send a RAG prompt, the system prompt + retrieved context is often the same across many queries (especially the system prompt portion). The LLM must process these tokens every time.

**KV Cache (provider-level):** The provider caches the Key/Value attention matrices for repeated prompt prefixes. If your system prompt (500 tokens) is identical across calls, subsequent calls skip the prefill computation for those tokens.

**Practical impact for RAG:**
- System prompt: cached (same every call) → ~50% faster prefill
- Retrieved context: NOT cached (different every call)
- Savings: 20-40% latency reduction on the prefill phase

**Anthropic's prompt caching:** Explicitly mark cacheable sections. Pay 25% extra on first use, get 90% discount on subsequent reads of the same prefix. For RAG with a long system prompt + few-shot examples, this is significant.

---

## Q18. How do you handle documents with tables, images, and charts in RAG? ⭐⭐⭐

**What the interviewer is really testing:** Real-world document complexity.

**Answer:**

**Tables:** Extract as structured data (Markdown or JSON). Store as a single chunk — never split mid-row. Add a description: "Table: Q4 2024 revenue by region."

**Images:** Describe via multimodal LLM (GPT-4o, Claude Vision). Store the description as text for embedding. Optionally store the original image for multimodal generation.

**Charts:** Extract underlying data using chart-to-data tools (DePlot, ChartOCR). Store extracted data + text description. Both are searchable; the data enables calculations.

**Key principle:** Everything becomes text for retrieval. The original modality (image, table) is preserved for generation.

---

## Q19. What is Graph RAG? When do you need it over standard vector RAG? ⭐⭐⭐⭐

**What the interviewer is really testing:** Understanding of relational retrieval.

**Answer:**

Standard RAG retrieves chunks similar to the query. Graph RAG extracts entities and relationships from documents into a knowledge graph, then retrieves via graph traversal.

**When you NEED Graph RAG:**
- "Which executives at our partner companies also sit on competitor boards?" → requires traversing relationships across multiple documents
- "What's the chain of custody for this chemical compound?" → entity-relationship chain
- "Show me all contracts connected to Vendor X" → entity-centric query

**When standard RAG is fine:**
- "What is the refund policy?" → simple factual lookup
- "Summarize the Q4 earnings call" → content-based, not relationship-based

**Architecture:**
```
Documents → LLM Entity Extraction → Knowledge Graph (Neo4j/NetworkX)
                                          ↓
Query → Entity Recognition → Graph Traversal → Relevant Subgraph
                                          ↓
                                   + Vector search for supporting text
                                          ↓
                                   LLM Generation with graph + text context
```

**Trade-off:** Graph RAG is expensive to build (LLM call per document for entity extraction) and complex to maintain. Only use when relational queries are a core requirement.

---

## Q20. Your RAG system works with 10K documents. Now it has 1M. What breaks first and how do you fix it? ⭐⭐⭐⭐

**What the interviewer is really testing:** Scaling experience.

**Answer:**

**Breaks first — retrieval precision:**
At 10K docs, top-5 retrieval was precise (fewer distractors). At 1M, the noise floor rises. More chunks compete for top-K slots, pushing truly relevant results below the threshold.

Fix: Add reranking (cross-encoder), hybrid search (BM25+vector with RRF), and metadata filtering (narrow search space BEFORE similarity search).

**Breaks second — ingestion pipeline:**
Full re-embedding at 1M docs takes days and costs thousands.

Fix: Incremental indexing (only embed new/changed docs), content-addressed caching (hash each chunk, skip if cached).

**Breaks third — vector DB performance:**
FAISS in-memory at 1M × 1536-dim = ~6GB. Manageable. At 10M, impractical.

Fix: Move from flat index to HNSW/IVF (approximate nearest neighbor), partition by metadata, or move to managed DB (Pinecone/Weaviate).

**Breaks fourth — chunk quality variance:**
At 1M docs: duplicates, near-duplicates, outdated versions, low-quality sources pollute retrieval.

Fix: Deduplication at ingestion, version management (index only latest), source quality scoring.
