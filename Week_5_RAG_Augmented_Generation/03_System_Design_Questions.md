# 🏗️ Week 5 — System Design Questions

> **Focus:** Production RAG at scale, multi-tenant RAG, enterprise document Q&A, evaluation infrastructure, cost optimization
>
> **How to use:** 30-45 minute whiteboard rounds. Draw the architecture, discuss trade-offs, deep-dive into the critical path.

---

## Q1. Design a Production RAG System for 50K Documents and 10K Daily Users ⭐⭐⭐⭐

**Prompt:** "Design a RAG-powered customer support chatbot. 50K documents (product manuals, FAQs, troubleshooting guides), 10K queries/day. Must answer accurately with citations, handle document updates, and cost <$500/month."

**Architecture:**

```
┌──────── Offline: Ingestion Pipeline (runs on document changes) ──────┐
│                                                                       │
│  S3 Document Store (source of truth)                                 │
│       │                                                               │
│       ▼                                                               │
│  [Change Detector] → only process new/modified documents             │
│       │                                                               │
│       ▼                                                               │
│  [Document Parser] → PyMuPDF for PDF, unstructured for mixed        │
│       │                                                               │
│       ▼                                                               │
│  [Chunker] → recursive splitting, 512 tokens, 50 overlap            │
│       │        tables as single chunks, metadata enrichment          │
│       │                                                               │
│       ▼                                                               │
│  [Embedder] → text-embedding-3-small (batched, cached)               │
│       │                                                               │
│       ▼                                                               │
│  [Vector DB] → Pinecone (managed, auto-scaling)                      │
│       │         namespace per document category                       │
│       ▼                                                               │
│  [BM25 Index] → Elasticsearch for keyword search                     │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────── Online: Query Pipeline (per user request) ───────────────────┐
│                                                                       │
│  User Query                                                          │
│       │                                                               │
│       ▼                                                               │
│  [Semantic Cache] → check if similar query was answered recently     │
│  │  Hit → return cached response (0ms LLM cost)                     │
│  │  Miss ↓                                                           │
│  │                                                                    │
│  ▼                                                                    │
│  [Query Router] → simple (single-pass) or complex (multi-hop)?      │
│       │                                                               │
│       ▼                                                               │
│  [Hybrid Search] → vector (Pinecone) + BM25 (Elasticsearch)         │
│       │             → RRF fusion → top-20 candidates                 │
│       │                                                               │
│       ▼                                                               │
│  [Cross-Encoder Reranker] → top-20 → top-5                          │
│       │                                                               │
│       ▼                                                               │
│  [Context Builder] → format chunks with source metadata              │
│       │                                                               │
│       ▼                                                               │
│  [LLM Generation] → gpt-4o-mini (90%) / gpt-4o (10% complex)       │
│       │               with citation enforcement                      │
│       │                                                               │
│       ▼                                                               │
│  [Faithfulness Check] → sample 5% for LLM-as-judge eval             │
│       │                                                               │
│       ▼                                                               │
│  Response with [Source 1], [Source 2] citations                      │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

**Cost estimation:**
```
50K docs × 20 pages avg × 500 tokens/page = 500M tokens
Embedding: 500M tokens × $0.02/1M = $10 (one-time)
Pinecone: ~$70/month (serverless, 500K vectors)
Elasticsearch: ~$50/month (managed, BM25 index)

10K queries/day:
  Embedding query: $0.002/day
  Reranker: $10/day (Cohere Rerank on 10K × 20 candidates)
  LLM (90% mini + 10% 4o): ~$8/day
  Semantic cache saves 30%: -$3/day

Monthly: $70 (Pinecone) + $50 (ES) + $15/day × 30 = $570
→ Optimize: reduce reranker calls (only for low-confidence) → ~$450/month ✅
```

---

## Q2. Design a Multi-Tenant RAG Platform ⭐⭐⭐⭐

**Prompt:** "You're building a RAG platform where 100 enterprise customers each upload their own documents. Customer A must NEVER see Customer B's documents in retrieval results."

**Isolation approaches:**

**Option A — Metadata filtering (simplest, lowest cost):**
```python
# Single shared index, filter by tenant_id on every query
results = pinecone_index.query(
    vector=query_embedding,
    filter={"tenant_id": {"$eq": "customer_a"}},
    top_k=10,
)
```
Pros: One index, shared infrastructure, lowest cost. Cons: Single point of failure, noisy-neighbor risk on search performance.

**Option B — Namespace per tenant:**
```python
# Separate namespace within same index
results = pinecone_index.query(
    vector=query_embedding,
    namespace="customer_a",
    top_k=10,
)
```
Pros: Better isolation, no filter overhead. Cons: Still shared infrastructure.

**Option C — Separate index per tenant (maximum isolation):**
Each customer gets their own vector index, their own embedding pipeline, their own evaluation. Full physical isolation.
Pros: Maximum security, independent scaling. Cons: 100× operational overhead, highest cost.

**Recommendation:** Option B (namespace) for most cases. Option C only for regulated industries (healthcare, government, financial) where compliance requires physical data separation.

**Cost allocation per tenant:**
Track: embedding tokens, storage (vectors × dimensions), query volume, LLM tokens. Bill per tenant based on usage. Set per-tenant spending limits.

---

## Q3. Design a RAG Evaluation Infrastructure That Runs in CI/CD ⭐⭐⭐⭐

**Prompt:** "Every prompt change, model change, or chunking change can regress your RAG quality. Design an evaluation system that catches regressions before they reach production."

```yaml
# .github/workflows/rag-eval.yml
name: RAG Quality Gate

on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'src/chunking/**'
      - 'src/retrieval/**'
      - 'config/models.yaml'

jobs:
  retrieval-eval:
    runs-on: ubuntu-latest
    steps:
      - name: Run retrieval precision eval
        run: |
          python eval/test_retrieval.py \
            --dataset eval/golden_set.json \
            --metric precision@5 \
            --threshold 0.80
      
      - name: Run retrieval recall eval
        run: |
          python eval/test_retrieval.py \
            --dataset eval/golden_set.json \
            --metric recall@10 \
            --threshold 0.90

  generation-eval:
    runs-on: ubuntu-latest
    needs: retrieval-eval
    steps:
      - name: Run faithfulness eval
        run: |
          python eval/test_generation.py \
            --dataset eval/golden_set.json \
            --metric faithfulness \
            --threshold 0.85
      
      - name: Run answer relevancy eval
        run: |
          python eval/test_generation.py \
            --metric answer_relevancy \
            --threshold 0.80

  safety-eval:
    runs-on: ubuntu-latest
    steps:
      - name: Run hallucination detection
        run: |
          python eval/test_safety.py \
            --max-hallucination-rate 0.05  # 5% max
      
      - name: Run prompt injection resistance
        run: python eval/test_injection.py --min-resistance 0.95

  gate:
    needs: [retrieval-eval, generation-eval, safety-eval]
    runs-on: ubuntu-latest
    steps:
      - name: Quality gate check
        run: |
          echo "All eval thresholds passed ✅"
          # If ANY eval job failed, this job won't run
          # PR is blocked from merging
```

**Golden dataset structure:**
```json
[
  {
    "id": "eval_001",
    "query": "What is the refund policy for enterprise customers?",
    "relevant_chunk_ids": ["policy_v3_chunk_42", "policy_v3_chunk_43"],
    "expected_answer_contains": ["30 days", "full refund", "enterprise"],
    "expected_answer_not_contains": ["individual plan", "no refund"],
    "category": "policy",
    "difficulty": "easy"
  }
]
```

---

## Q4. Design an Enterprise Document Q&A System for Complex PDFs ⭐⭐⭐⭐

**Prompt:** "Your client has 100K PDFs — contracts, financial reports, engineering specs. They contain tables, charts, scanned pages, and multi-column layouts. Design a RAG system that handles all of this."

**Document processing pipeline:**
```
PDF → [Classifier: digital vs scanned vs mixed]
         │
         ├── Digital → PyMuPDF text extraction
         ├── Scanned → OCR (Tesseract / AWS Textract)
         └── Mixed → Page-by-page classification
                │
                ▼
         [Layout Parser]
         ├── Text blocks → recursive chunking
         ├── Tables → extract as structured Markdown (one chunk per table)
         ├── Charts → DePlot/ChartOCR → extract data + description
         ├── Images → multimodal LLM description → text chunk
         └── Headers → section hierarchy metadata
                │
                ▼
         [Metadata Enrichment]
         ├── Document title, date, author, type
         ├── Section headers per chunk
         ├── Page numbers
         └── Content type tag (text/table/chart/image)
                │
                ▼
         [Embed + Index]
         All content types embedded as text
         Original images/tables stored separately for multimodal generation
```

**Key design: tables as first-class citizens.**
Never chunk through a table mid-row. Extract the table as structured data, add a description header, store as one chunk:
```
[Table: Q4 2024 Revenue by Region, Source: annual_report.pdf, Page 42]
| Region | Revenue | Growth |
|--------|---------|--------|
| North America | $2.1B | 25% |
| Europe | $1.4B | 18% |
| Asia Pacific | $0.7B | 32% |
```

---

## Q5. Design a Real-Time RAG System with Document Freshness ⭐⭐⭐

**Prompt:** "Your knowledge base updates hourly (support tickets, product changes, news). How do you ensure RAG answers reflect the latest information?"

**Architecture:**
```
Document Source (CMS, wiki, ticketing system)
    │
    ▼ Webhook / polling (every 15 min)
[Change Detector]
    │
    ├── New document → embed → index
    ├── Updated document → re-embed → upsert (replace old vectors)
    └── Deleted document → remove vectors from index
    │
    ▼
[Freshness Metadata]
    Every chunk has: indexed_at, source_updated_at, ttl
    
At query time:
    Retrieve → filter by freshness (prefer recent) → boost recent documents
    
    If retrieved chunk is older than TTL:
        → Flag: "This information may be outdated (last updated: 3 days ago)"
```

**Handling conflicting versions:** If the old version and new version of a document are both in the index (during the re-indexing window), metadata filtering ensures only the latest version is returned:
```python
results = vector_db.query(
    embedding, 
    filter={"version": {"$eq": "latest"}},
    top_k=5,
)
```

---

## Q6-Q15: Additional System Design Questions (Condensed)

### Q6. Design a RAG System That Handles 50 Languages ⭐⭐⭐⭐
Multilingual embedding model (Cohere embed-v3, multilingual-e5). Language detection on query → route to language-specific system prompt. Cross-lingual retrieval: English docs retrievable with Spanish queries. Translation fallback for low-resource languages.

### Q7. Design a Feedback Loop That Improves RAG Quality Over Time ⭐⭐⭐
Log (query, retrieved_chunks, response, user_feedback). Weekly pipeline: cluster negative feedback by failure type (retrieval miss, wrong answer, hallucination). Each cluster generates an improvement action (add document, fix chunking, adjust prompt). Auto-grow eval set from user corrections.

### Q8. Design a Cost-Optimized RAG for 1M Queries/Day ⭐⭐⭐⭐
Semantic cache (30% hit rate = 300K free). Model routing (80% gpt-4o-mini, 20% gpt-4o). Batch embedding for ingestion. Prompt compression (shorter system prompt). Context limit (max 5 chunks). Result: cost per query from $0.01 to $0.003.

### Q9. Design a Hybrid RAG + SQL System for Structured + Unstructured Data ⭐⭐⭐⭐
Query classifier: "structured" (numbers, aggregates, specific records) → NLP-to-SQL → query database directly. "Unstructured" (explanations, policies, procedures) → RAG pipeline. "Both" (explain why revenue dropped) → SQL for numbers + RAG for context → synthesize.

### Q10. Design a Graph RAG System for Legal Entity Relationships ⭐⭐⭐⭐
Entity extraction from contracts, filings, correspondence → build knowledge graph (Neo4j) with entities (companies, people, clauses, dates) and relationships (party_to, signed_by, references). Query: entity recognition → graph traversal → retrieve connected documents → generate with relationship context.

### Q11. Design a Multimodal RAG for Medical Imaging + Clinical Notes ⭐⭐⭐⭐
Images → medical image classifier → description via multimodal LLM → text embedding. Clinical notes → NER for medical entities → structured extraction + text chunking. Query: combine image understanding + text retrieval. Critical: never hallucinate medical information, always cite specific records.

### Q12. Design a RAG System with Role-Based Access Control ⭐⭐⭐
Different users see different documents. Metadata filter: `allowed_roles` on each chunk. At query time: filter by user's role BEFORE similarity search. Admin sees everything, manager sees team docs, employee sees public docs only. Audit log: who queried what, what was returned.

### Q13. Design a Self-Improving RAG with Automated Quality Monitoring ⭐⭐⭐⭐
Continuous monitoring: sample 5% of responses → LLM-as-judge scoring → dashboard. Alert if faithfulness drops below 0.85. Weekly auto-analysis of failure clusters. Automated prompt refinement: test candidate improvements on dev eval set → promote if better → canary deploy.

### Q14. Design a RAG System That Works Offline (Edge Deployment) ⭐⭐⭐
Self-contained: local embedding model (MiniLM, 384-dim) + FAISS index + local LLM (Llama 3 8B via llama.cpp). Sync: periodic upload of new documents when connected. Use case: field technicians with intermittent connectivity querying equipment manuals.

### Q15. Design a Benchmark Suite for Comparing RAG Approaches ⭐⭐⭐⭐
3 datasets: simple factual (easy), multi-hop reasoning (hard), adversarial (tricky). 5 metrics: precision@5, recall@10, faithfulness, answer relevancy, latency. Matrix: test each RAG variant (naive, advanced, agentic, graph) × each dataset × each metric. Auto-generate comparison report with statistical significance tests.
