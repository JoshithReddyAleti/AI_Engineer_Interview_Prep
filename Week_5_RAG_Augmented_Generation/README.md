# 🔍 Week 5 — RAG & Augmented Generation Interview Prep

> **Maps to:** [Episode_5_Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the single most-tested topic in AI engineer interviews.** Every company building with LLMs — from MAANG to Series A startups — asks RAG questions. This week covers not just RAG, but the entire family of augmented generation (RAG, CAG, MAG, TAG, KAG), all 9 RAG variants, and the building blocks (chunking, embeddings, vector DBs, retrieval, reranking, evaluation) at a depth that gets offers.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 20 | RAG end-to-end, all 5 augmented generation types, 9 RAG variants, chunking, embeddings, vector DBs, retrieval strategies |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 16 | Build RAG from scratch, chunking implementations, hybrid search, reranker, Self-RAG, evaluation pipeline |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 15 | Production RAG at scale, multi-tenant RAG, enterprise document Q&A, Graph RAG, evaluation infrastructure |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 15 | RAG debugging in production, hallucination incidents, scaling crises, stakeholder communication, trade-off decisions |

## Key Topics Tested

**Augmented Generation Family:**
- RAG (Retrieval-Augmented Generation) — the main event
- CAG (Cache-Augmented Generation) — preloaded context, no retrieval
- MAG (Memory-Augmented Generation) — persistent external memory
- TAG (Tool-Augmented Generation) — live API/tool calls at inference
- KAG (Knowledge-Augmented Generation) — structured knowledge graphs

**9 RAG Variants (every one is testable):**
- Naive RAG → the baseline every interview starts with
- Advanced RAG → HyDE, query rewriting, hybrid search, reranking
- Graph RAG → entity extraction, knowledge graph traversal
- Agentic RAG → agent decides retrieval strategy per query
- Self-RAG → retrieves only when the model decides it needs context
- Corrective RAG (CRAG) → verifies retrieved documents before using them
- Adaptive RAG → routes between no-retrieval, single-pass, and multi-hop
- Multimodal RAG → images, tables, charts alongside text
- Hybrid RAG → combines vector + keyword + graph in one pipeline

**Building Blocks:**
- Chunking strategies (Fixed, Recursive, Semantic, Parent-Child, Hybrid)
- Embedding models (OpenAI, Cohere, Voyage AI, open-source BGE/MiniLM)
- Vector databases (FAISS, Pinecone, Chroma, Milvus, Weaviate, pgvector)
- Retrieval (dense, sparse/BM25, hybrid with RRF)
- Reranking (bi-encoder vs cross-encoder, Cohere Rerank)
- Evaluation (Faithfulness, Answer Relevancy, Context Precision/Recall, RAGAS, DeepEval)

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1 | [LLM Fundamentals](../Week_1_LLM_Fundamentals/) | [Understanding_LLMs_From_The_Inside_Out](https://github.com/JoshithReddyAleti/Understanding_LLMs_From_The_Inside_Out) | ✅ |
| 2 | [Python for AI](../Week_2_Python_For_AI/) | [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters) | ✅ |
| 3 | [Tool Calling, APIs & Validation](../Week_3_Tool_Calling_APIs_Validation/) | [Building_AI_Project_Blueprint_for_Beginners](https://github.com/JoshithReddyAleti/Building_AI_Project-Blueprint_for_Begin) | ✅ |
| 4 | [End-to-End AI Projects](../Week_4_End_To_End_AI_Projects/) | [Your_First_End_To_End_AI_Project](https://github.com/JoshithReddyAleti/Episode_4_Your_First_End_To_End_AI_Project/tree/main/Your_First_End_To_End_AI_Project/Your_First_End_To_End_AI_Project) | ✅ |
| **5** | **RAG & Augmented Generation** ← you are here | [Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation) | ✅ |
| 6 | Frameworks — LangChain, LlamaIndex, LangGraph, CrewAI | Coming soon | 🔜 |
| 7 | Memory & State in AI Systems | Coming soon | 🔜 |
