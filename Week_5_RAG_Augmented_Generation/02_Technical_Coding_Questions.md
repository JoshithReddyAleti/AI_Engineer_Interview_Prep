# 💻 Week 5 — Technical / Coding Questions

> **Focus:** Build RAG components from scratch — chunking, embeddings, retrieval, reranking, evaluation, complete pipelines
>
> **How to use:** These are the coding challenges that appear in MAANG AI engineering live-coding rounds. Build before reading.

---

## Q1. Build a Complete Naive RAG Pipeline from Scratch ⭐⭐⭐

**Prompt:** Implement an end-to-end RAG pipeline: load documents, chunk, embed, store in FAISS, retrieve, and generate with citations. No frameworks — raw Python.

**Solution:**

```python
import numpy as np
import json
import re
from dataclasses import dataclass, field
from typing import Generator

@dataclass
class Chunk:
    text: str
    source: str
    chunk_index: int
    embedding: np.ndarray | None = None

class NaiveRAGPipeline:
    def __init__(self, embedding_fn, llm_fn, chunk_size=500, chunk_overlap=50):
        self.embedding_fn = embedding_fn  # text → np.ndarray
        self.llm_fn = llm_fn              # prompt → str
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.chunks: list[Chunk] = []
        self.embeddings: np.ndarray | None = None
    
    # === INGESTION ===
    
    def ingest(self, documents: list[dict]):
        """documents = [{"text": "...", "source": "file.pdf"}, ...]"""
        for doc in documents:
            doc_chunks = self._chunk_text(doc["text"], doc["source"])
            self.chunks.extend(doc_chunks)
        
        # Embed all chunks
        texts = [c.text for c in self.chunks]
        embeddings = [self.embedding_fn(t) for t in texts]
        self.embeddings = np.array(embeddings)
        
        # Normalize for cosine similarity
        norms = np.linalg.norm(self.embeddings, axis=1, keepdims=True)
        self.embeddings = self.embeddings / norms
        
        print(f"Ingested {len(self.chunks)} chunks from {len(documents)} documents")
    
    def _chunk_text(self, text: str, source: str) -> list[Chunk]:
        """Recursive character splitting with overlap."""
        sentences = re.split(r'(?<=[.!?])\s+', text)
        
        chunks = []
        current_chunk = []
        current_length = 0
        char_target = self.chunk_size * 4  # ~4 chars per token
        overlap_target = self.chunk_overlap * 4
        
        for sentence in sentences:
            if current_length + len(sentence) > char_target and current_chunk:
                chunk_text = " ".join(current_chunk)
                chunks.append(Chunk(text=chunk_text, source=source, chunk_index=len(chunks)))
                
                # Keep overlap
                overlap_text = chunk_text[-overlap_target:]
                current_chunk = [overlap_text]
                current_length = len(overlap_text)
            
            current_chunk.append(sentence)
            current_length += len(sentence)
        
        if current_chunk:
            chunks.append(Chunk(text=" ".join(current_chunk), source=source, chunk_index=len(chunks)))
        
        return chunks
    
    # === RETRIEVAL ===
    
    def retrieve(self, query: str, top_k: int = 5) -> list[tuple[Chunk, float]]:
        """Dense retrieval via cosine similarity."""
        query_embedding = self.embedding_fn(query)
        query_embedding = query_embedding / np.linalg.norm(query_embedding)
        
        # Cosine similarity (dot product since vectors are normalized)
        scores = self.embeddings @ query_embedding
        
        top_indices = np.argsort(scores)[::-1][:top_k]
        
        results = []
        for idx in top_indices:
            results.append((self.chunks[idx], float(scores[idx])))
        
        return results
    
    # === GENERATION ===
    
    def query(self, question: str, top_k: int = 5) -> dict:
        """Full RAG pipeline: retrieve → format → generate."""
        # Retrieve
        retrieved = self.retrieve(question, top_k)
        
        # Format context with citations
        context_parts = []
        for i, (chunk, score) in enumerate(retrieved):
            context_parts.append(
                f"[Source {i+1}: {chunk.source}, Chunk {chunk.chunk_index}] "
                f"(relevance: {score:.3f})\n{chunk.text}"
            )
        context = "\n\n".join(context_parts)
        
        # Generate
        prompt = f"""Answer the question based ONLY on the provided context.
If the context doesn't contain enough information, say "I don't have enough information."
Cite sources using [Source N] notation.

Context:
{context}

Question: {question}

Answer:"""
        
        answer = self.llm_fn(prompt)
        
        return {
            "answer": answer,
            "sources": [{"text": c.text[:100], "source": c.source, "score": s} for c, s in retrieved],
            "context_tokens": len(context) // 4,
        }


# Usage
def mock_embed(text: str) -> np.ndarray:
    """Placeholder — in production use OpenAI/Sentence Transformers."""
    np.random.seed(hash(text) % 2**32)
    return np.random.randn(384)

def mock_llm(prompt: str) -> str:
    return "Based on the provided context, the answer is..."

pipeline = NaiveRAGPipeline(embedding_fn=mock_embed, llm_fn=mock_llm)
pipeline.ingest([
    {"text": "Our refund policy allows returns within 30 days...", "source": "policy.pdf"},
    {"text": "Shipping takes 3-5 business days...", "source": "shipping.pdf"},
])
result = pipeline.query("What is the refund policy?")
```

---

## Q2. Implement 4 Chunking Strategies and Compare Them ⭐⭐⭐

**Prompt:** Build fixed-size, recursive, semantic, and parent-child chunking. Show how the same document produces different chunks.

**Solution:**

```python
import re
import numpy as np

class FixedSizeChunker:
    """Split by character count with overlap."""
    def __init__(self, chunk_size: int = 1000, overlap: int = 200):
        self.chunk_size = chunk_size
        self.overlap = overlap
    
    def chunk(self, text: str) -> list[str]:
        chunks = []
        start = 0
        while start < len(text):
            end = start + self.chunk_size
            chunks.append(text[start:end])
            start = end - self.overlap
        return chunks

class RecursiveChunker:
    """Split by hierarchy: paragraphs → sentences → words."""
    def __init__(self, chunk_size: int = 1000):
        self.chunk_size = chunk_size
        self.separators = ["\n\n", "\n", ". ", " "]
    
    def chunk(self, text: str) -> list[str]:
        return self._split(text, 0)
    
    def _split(self, text: str, sep_idx: int) -> list[str]:
        if len(text) <= self.chunk_size:
            return [text] if text.strip() else []
        
        if sep_idx >= len(self.separators):
            return [text[:self.chunk_size]]
        
        sep = self.separators[sep_idx]
        parts = text.split(sep)
        
        chunks = []
        current = ""
        for part in parts:
            candidate = current + sep + part if current else part
            if len(candidate) > self.chunk_size and current:
                chunks.append(current.strip())
                current = part
            else:
                current = candidate
        if current.strip():
            chunks.append(current.strip())
        
        # Recursively split any chunks that are still too large
        final = []
        for chunk in chunks:
            if len(chunk) > self.chunk_size:
                final.extend(self._split(chunk, sep_idx + 1))
            else:
                final.append(chunk)
        return final

class SemanticChunker:
    """Group sentences by embedding similarity."""
    def __init__(self, embed_fn, threshold: float = 0.5):
        self.embed_fn = embed_fn
        self.threshold = threshold
    
    def chunk(self, text: str) -> list[str]:
        sentences = re.split(r'(?<=[.!?])\s+', text)
        if len(sentences) <= 1:
            return [text]
        
        embeddings = [self.embed_fn(s) for s in sentences]
        
        chunks = []
        current_chunk = [sentences[0]]
        
        for i in range(1, len(sentences)):
            similarity = np.dot(embeddings[i], embeddings[i-1]) / (
                np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[i-1])
            )
            
            if similarity < self.threshold:
                # Topic shift detected — start new chunk
                chunks.append(" ".join(current_chunk))
                current_chunk = [sentences[i]]
            else:
                current_chunk.append(sentences[i])
        
        if current_chunk:
            chunks.append(" ".join(current_chunk))
        
        return chunks

class ParentChildChunker:
    """Small chunks for retrieval, large parents for context."""
    def __init__(self, parent_size: int = 2000, child_size: int = 500):
        self.parent_size = parent_size
        self.child_size = child_size
        self.parent_chunker = RecursiveChunker(parent_size)
        self.child_chunker = RecursiveChunker(child_size)
    
    def chunk(self, text: str) -> list[dict]:
        parents = self.parent_chunker.chunk(text)
        results = []
        
        for parent_idx, parent in enumerate(parents):
            children = self.child_chunker.chunk(parent)
            for child_idx, child in enumerate(children):
                results.append({
                    "child_text": child,         # Used for retrieval (small, precise)
                    "parent_text": parent,        # Returned as context (large, complete)
                    "parent_idx": parent_idx,
                    "child_idx": child_idx,
                })
        
        return results
```

---

## Q3. Build a Hybrid Search Engine (Vector + BM25 + RRF) ⭐⭐⭐⭐

**Prompt:** Implement hybrid retrieval combining dense vector search and sparse BM25, fused with Reciprocal Rank Fusion.

**Solution:**

```python
import math
from collections import Counter, defaultdict

class BM25:
    """Sparse keyword retrieval."""
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        self.k1 = k1
        self.b = b
        self.doc_freqs = Counter()
        self.doc_lengths = []
        self.avg_dl = 0
        self.corpus_size = 0
        self.term_freqs = []  # per-document term frequencies
    
    def index(self, documents: list[str]):
        self.corpus_size = len(documents)
        for doc in documents:
            terms = doc.lower().split()
            self.doc_lengths.append(len(terms))
            tf = Counter(terms)
            self.term_freqs.append(tf)
            for term in set(terms):
                self.doc_freqs[term] += 1
        self.avg_dl = sum(self.doc_lengths) / self.corpus_size
    
    def search(self, query: str, top_k: int = 10) -> list[tuple[int, float]]:
        query_terms = query.lower().split()
        scores = []
        
        for doc_idx in range(self.corpus_size):
            score = 0
            dl = self.doc_lengths[doc_idx]
            
            for term in query_terms:
                if term not in self.term_freqs[doc_idx]:
                    continue
                
                tf = self.term_freqs[doc_idx][term]
                df = self.doc_freqs[term]
                idf = math.log((self.corpus_size - df + 0.5) / (df + 0.5) + 1)
                
                numerator = tf * (self.k1 + 1)
                denominator = tf + self.k1 * (1 - self.b + self.b * dl / self.avg_dl)
                
                score += idf * numerator / denominator
            
            scores.append((doc_idx, score))
        
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:top_k]


def reciprocal_rank_fusion(
    *ranked_lists: list[tuple[int, float]],
    k: int = 60,
) -> list[tuple[int, float]]:
    """Fuse multiple ranked lists using RRF."""
    rrf_scores = defaultdict(float)
    
    for ranked_list in ranked_lists:
        for rank, (doc_idx, _) in enumerate(ranked_list, start=1):
            rrf_scores[doc_idx] += 1.0 / (k + rank)
    
    fused = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
    return fused


class HybridSearchEngine:
    """Combines dense vector search + BM25 sparse search + RRF fusion."""
    
    def __init__(self, embed_fn):
        self.embed_fn = embed_fn
        self.bm25 = BM25()
        self.documents: list[str] = []
        self.embeddings: np.ndarray | None = None
    
    def index(self, documents: list[str]):
        self.documents = documents
        
        # Build BM25 index
        self.bm25.index(documents)
        
        # Build vector index
        embeddings = [self.embed_fn(doc) for doc in documents]
        self.embeddings = np.array(embeddings)
        norms = np.linalg.norm(self.embeddings, axis=1, keepdims=True)
        self.embeddings = self.embeddings / norms
    
    def search(self, query: str, top_k: int = 5) -> list[dict]:
        # Dense retrieval
        query_emb = self.embed_fn(query)
        query_emb = query_emb / np.linalg.norm(query_emb)
        dense_scores = self.embeddings @ query_emb
        dense_ranked = sorted(enumerate(dense_scores), key=lambda x: x[1], reverse=True)[:20]
        
        # Sparse retrieval
        sparse_ranked = self.bm25.search(query, top_k=20)
        
        # RRF fusion
        fused = reciprocal_rank_fusion(dense_ranked, sparse_ranked, k=60)
        
        results = []
        for doc_idx, rrf_score in fused[:top_k]:
            results.append({
                "text": self.documents[doc_idx],
                "rrf_score": rrf_score,
                "dense_score": float(dense_scores[doc_idx]),
                "doc_index": doc_idx,
            })
        
        return results
```

---

## Q4. Build a Cross-Encoder Reranker ⭐⭐⭐

**Prompt:** Implement a reranking step that takes candidates from initial retrieval and re-scores them using a cross-encoder approach.

**Solution:**

```python
class CrossEncoderReranker:
    """
    Re-score (query, document) pairs using an LLM or cross-encoder model.
    In production: use a dedicated cross-encoder (ms-marco-MiniLM) or Cohere Rerank API.
    Here: LLM-based relevance scoring as a universal fallback.
    """
    
    def __init__(self, scoring_fn=None):
        self.scoring_fn = scoring_fn or self._llm_based_scorer
    
    def rerank(
        self,
        query: str,
        candidates: list[dict],
        top_k: int = 5,
    ) -> list[dict]:
        scored = []
        for candidate in candidates:
            score = self.scoring_fn(query, candidate["text"])
            scored.append({**candidate, "rerank_score": score})
        
        scored.sort(key=lambda x: x["rerank_score"], reverse=True)
        return scored[:top_k]
    
    def _llm_based_scorer(self, query: str, document: str) -> float:
        """
        Use an LLM to score relevance.
        In production: use Cohere Rerank API or a cross-encoder model for speed.
        """
        prompt = f"""Rate the relevance of this document to the query on a scale of 0-10.
Return ONLY a number.

Query: {query}
Document: {document[:500]}

Relevance score (0-10):"""
        
        # In production: actual LLM call
        # response = llm.generate(prompt, max_tokens=3, temperature=0)
        # return float(response.strip()) / 10.0
        
        # Placeholder
        return 0.5


# Two-stage retrieval pipeline
def two_stage_retrieve(query: str, search_engine, reranker, initial_k=50, final_k=5):
    """
    Stage 1: Fast approximate search (bi-encoder) → 50 candidates
    Stage 2: Precise reranking (cross-encoder) → 5 results
    """
    # Stage 1: Cast wide net
    candidates = search_engine.search(query, top_k=initial_k)
    
    # Stage 2: Precise reranking
    reranked = reranker.rerank(query, candidates, top_k=final_k)
    
    return reranked
```

---

## Q5. Build a RAG Evaluation Pipeline ⭐⭐⭐⭐

**Prompt:** Implement evaluation metrics for a RAG system: Faithfulness, Answer Relevancy, and Context Precision. Use LLM-as-judge methodology.

**Solution:**

```python
from dataclasses import dataclass

@dataclass
class EvalCase:
    query: str
    retrieved_contexts: list[str]
    generated_answer: str
    ground_truth: str | None = None

@dataclass
class EvalResult:
    faithfulness: float
    answer_relevancy: float
    context_precision: float
    details: dict = None

class RAGEvaluator:
    """LLM-as-judge evaluation for RAG pipelines."""
    
    def __init__(self, judge_fn):
        self.judge_fn = judge_fn  # prompt → str (LLM call)
    
    def evaluate(self, case: EvalCase) -> EvalResult:
        faithfulness = self._evaluate_faithfulness(case)
        relevancy = self._evaluate_answer_relevancy(case)
        precision = self._evaluate_context_precision(case)
        
        return EvalResult(
            faithfulness=faithfulness,
            answer_relevancy=relevancy,
            context_precision=precision,
        )
    
    def _evaluate_faithfulness(self, case: EvalCase) -> float:
        """Is every claim in the answer supported by the context?"""
        context = "\n---\n".join(case.retrieved_contexts)
        
        prompt = f"""Evaluate the faithfulness of the answer to the given context.
Faithfulness = every claim in the answer is supported by the context.

Context:
{context}

Answer:
{case.generated_answer}

Rate faithfulness from 0.0 (completely unfaithful/hallucinated) to 1.0 (every claim is grounded).
Respond with ONLY a decimal number."""
        
        score_str = self.judge_fn(prompt).strip()
        try:
            return max(0, min(1, float(score_str)))
        except ValueError:
            return 0.5
    
    def _evaluate_answer_relevancy(self, case: EvalCase) -> float:
        """Does the answer address the question?"""
        prompt = f"""Evaluate whether this answer is relevant to the question.

Question: {case.query}
Answer: {case.generated_answer}

Rate relevancy from 0.0 (completely off-topic) to 1.0 (directly answers the question).
Respond with ONLY a decimal number."""
        
        score_str = self.judge_fn(prompt).strip()
        try:
            return max(0, min(1, float(score_str)))
        except ValueError:
            return 0.5
    
    def _evaluate_context_precision(self, case: EvalCase) -> float:
        """Of the retrieved contexts, how many were actually relevant?"""
        relevant_count = 0
        for ctx in case.retrieved_contexts:
            prompt = f"""Is this context relevant to answering the question?

Question: {case.query}
Context: {ctx[:500]}

Answer YES or NO only."""
            
            result = self.judge_fn(prompt).strip().upper()
            if "YES" in result:
                relevant_count += 1
        
        return relevant_count / max(len(case.retrieved_contexts), 1)
    
    def evaluate_dataset(self, cases: list[EvalCase]) -> dict:
        """Run evaluation across a full dataset."""
        results = [self.evaluate(case) for case in cases]
        
        return {
            "num_cases": len(results),
            "avg_faithfulness": sum(r.faithfulness for r in results) / len(results),
            "avg_answer_relevancy": sum(r.answer_relevancy for r in results) / len(results),
            "avg_context_precision": sum(r.context_precision for r in results) / len(results),
            "below_threshold": sum(1 for r in results if r.faithfulness < 0.7),
        }
```

---

## Q6. Implement Self-RAG (Retrieval Only When Needed) ⭐⭐⭐⭐

**Prompt:** Build a Self-RAG system that decides whether to retrieve or answer directly, then self-evaluates groundedness.

**Solution:**

```python
class SelfRAG:
    """
    Self-RAG: The model decides when to retrieve.
    1. Assess: Do I need external info for this query?
    2. If YES → retrieve → generate from context → verify groundedness
    3. If NO → generate directly from model knowledge
    """
    
    def __init__(self, llm_fn, retriever, confidence_threshold=0.7):
        self.llm_fn = llm_fn
        self.retriever = retriever
        self.confidence_threshold = confidence_threshold
    
    def query(self, question: str) -> dict:
        # Step 1: Assess retrieval necessity
        needs_retrieval = self._assess_retrieval_need(question)
        
        if not needs_retrieval:
            # Direct generation — skip retrieval (saves cost + latency)
            answer = self.llm_fn(f"Answer this question:\n{question}")
            return {
                "answer": answer,
                "retrieval_used": False,
                "reason": "Model confident in direct answer",
            }
        
        # Step 2: Retrieve
        chunks = self.retriever.retrieve(question, top_k=5)
        context = "\n\n".join([c["text"] for c in chunks])
        
        # Step 3: Generate from context
        answer = self.llm_fn(
            f"Answer based ONLY on this context:\n\n{context}\n\nQuestion: {question}"
        )
        
        # Step 4: Self-evaluate groundedness
        is_grounded = self._check_groundedness(answer, context)
        
        if not is_grounded:
            # Retry with stricter prompt or more context
            answer = self.llm_fn(
                f"The previous answer may contain unsupported claims. "
                f"Answer STRICTLY from this context. If unsure, say 'I don't know'.\n\n"
                f"Context:\n{context}\n\nQuestion: {question}"
            )
        
        return {
            "answer": answer,
            "retrieval_used": True,
            "is_grounded": is_grounded,
            "chunks_used": len(chunks),
        }
    
    def _assess_retrieval_need(self, question: str) -> bool:
        """Does this question need external context?"""
        prompt = f"""Assess whether this question requires external information to answer correctly.

Question: {question}

Consider:
- Factual questions about specific data/policies → NEEDS RETRIEVAL
- General knowledge questions (capital of France) → NO RETRIEVAL NEEDED
- Questions about current events or specific documents → NEEDS RETRIEVAL
- Simple math or logic → NO RETRIEVAL NEEDED

Answer ONLY 'RETRIEVE' or 'DIRECT':"""
        
        result = self.llm_fn(prompt).strip().upper()
        return "RETRIEVE" in result
    
    def _check_groundedness(self, answer: str, context: str) -> bool:
        """Is the answer grounded in the provided context?"""
        prompt = f"""Is every claim in this answer supported by the context?

Context: {context[:2000]}
Answer: {answer}

Answer ONLY 'GROUNDED' or 'NOT_GROUNDED':"""
        
        result = self.llm_fn(prompt).strip().upper()
        return "GROUNDED" in result
```

---

## Q7-Q16: Additional Coding Challenges (Condensed)

### Q7. Build a Query Expansion System ⭐⭐⭐
Generate 3-5 query variations using an LLM, retrieve for each, merge results with deduplication, rerank against original query. Key: captures different phrasings that match different documents.

### Q8. Implement HyDE (Hypothetical Document Embedding) ⭐⭐⭐
LLM generates a hypothetical answer → embed THAT instead of the query → search for similar real documents. Bridges vocabulary gap between queries and documents.

### Q9. Build a Document-Level Retrieval + Chunk Refinement Pipeline ⭐⭐⭐
Stage 1: search document summaries to find relevant DOCUMENTS. Stage 2: within those documents, search for relevant CHUNKS. Prevents missing documents that are relevant at the topic level but not at the chunk level.

### Q10. Implement Adaptive RAG Router ⭐⭐⭐⭐
Classify query complexity → route to: no-retrieval (simple), single-pass RAG (standard), or multi-hop iterative RAG (complex). Use a lightweight classifier or LLM to categorize.

### Q11. Build a Streaming RAG Response with Progressive Source Display ⭐⭐⭐
Show sources immediately after retrieval → stream LLM generation tokens → display citations as they appear. User sees progress at every stage, never waits for full completion.

### Q12. Implement a Chunk Deduplication System ⭐⭐
Content-hash and near-duplicate detection (MinHash/SimHash) during ingestion. Prevents the same content from appearing as multiple top-K results due to duplicate indexing.

### Q13. Build a RAG Pipeline with Metadata Filtering ⭐⭐
Add metadata (date, department, document_type, author) to chunks. At query time, filter BEFORE vector search to narrow the search space. Key: `{"document_type": "policy", "year": 2024}` before similarity search.

### Q14. Implement Conversational RAG (Multi-Turn with History) ⭐⭐⭐
Maintain conversation history → on each turn, combine history + current question → contextualized query for retrieval. Handle follow-ups: "What about their competitor?" requires resolving "their" from history.

### Q15. Build a RAG Cost Calculator and Optimizer ⭐⭐
Track: embedding cost + retrieval infra cost + LLM input tokens + output tokens per query. Identify optimization levers: smaller model for simple queries, caching, fewer chunks, shorter system prompt.

### Q16. Implement a Simple Graph RAG with Entity Extraction ⭐⭐⭐⭐
Use LLM to extract (entity, relationship, entity) triples from documents → build NetworkX graph → for queries mentioning entities, traverse graph for connected nodes → combine graph context with vector-retrieved text → generate.
