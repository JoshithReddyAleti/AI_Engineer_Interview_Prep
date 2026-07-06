# 🎯 Extra Prep Questions — Cross-Cutting AI Engineer Interview Topics

> **These questions span multiple weeks and cover advanced topics frequently asked in MAANG-level AI engineer interviews.**
>
> **Covers:** RAG architecture, embeddings, vector databases, frameworks, agent design, prompt engineering, evaluation, fine-tuning, inference optimization, security, production monitoring, and more.
>
> *Part of the [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/) interview prep series.*

---

## Table of Contents

1. [RAG Architecture (End-to-End)](#1-rag-architecture-end-to-end)
2. [Embeddings — What They Are and How to Choose](#2-embeddings)
3. [Vector Database Comparison](#3-vector-database-comparison)
4. [Framework Comparison — LangChain vs LlamaIndex vs LangGraph vs CrewAI](#4-framework-comparison)
5. [AI Agents and Multi-Agent Architecture](#5-ai-agents-and-multi-agent-architecture)
6. [Function Calling / Tool Calling Implementation](#6-function-calling--tool-calling)
7. [MCP (Model Context Protocol)](#7-mcp-model-context-protocol)
8. [LLM Latency Optimization and Inference Cost Reduction](#8-llm-latency-optimization)
9. [Prompt Engineering Techniques](#9-prompt-engineering-techniques)
10. [LLM Application Evaluation — RAGAS, DeepEval, and Metrics](#10-llm-evaluation)
11. [Preventing Hallucinations in RAG](#11-preventing-hallucinations)
12. [Chunking Strategies](#12-chunking-strategies)
13. [MMR and RRF — Advanced Retrieval Techniques](#13-mmr-and-rrf)
14. [KV Cache, Speculative Decoding, and Continuous Batching](#14-kv-cache-speculative-decoding-continuous-batching)
15. [Enterprise GenAI Security](#15-enterprise-genai-security)
16. [LLM Fine-Tuning — LoRA, QLoRA, PEFT, RLHF](#16-llm-fine-tuning)
17. [Production LLM Monitoring](#17-production-llm-monitoring)
18. [Designing Applications to Survive Zonal Failures](#18-zonal-failure-resilience)
19. [RAG Debugging — Wrong Answers, First Thing to Investigate](#19-rag-debugging)
20. [Retriever Finds Relevant Docs But Answer Quality Is Poor](#20-retrieval-generation-gap)
21. [Proving Embedding Improvements Actually Improved the System](#21-proving-embedding-improvements)
22. [Multi-Document Questions — Info Spread Across 5 Documents](#22-multi-document-questions)
23. [RAG Scaling — 10K to 1M Documents, What Breaks First](#23-rag-scaling)
24. [NLP-to-SQL Chatbot for 100+ Table RDBMS](#24-nlp-to-sql)
25. [Scientific Prompt Engineering — Isolating What Works](#25-scientific-prompt-engineering)
26. [Correct Answer in Chunk #12 But Top-K Is 5](#26-chunk-12-problem)
27. [Embedding Cost Optimization — $20K/Month Re-Embedding](#27-embedding-cost-optimization)
28. [RAG Streaming and Progressive Rendering](#28-rag-streaming)
29. [Agent Action Safety — Preventing Accidental Destructive Actions](#29-agent-action-safety)
30. [AI Gateway — Multi-Provider Routing with Auto-Fallback](#30-ai-gateway)
31. [Context Compression at 40K Tokens](#31-context-compression)
32. [Safe Model Migration in Production](#32-safe-model-migration)
33. [Catastrophic Forgetting After Fine-Tuning](#33-catastrophic-forgetting)
34. [Hybrid Search — BM25 + Vector vs Pure Vector](#34-hybrid-search)
35. [Reproducing a Fine-Tuning Run 6 Months Later](#35-finetune-reproducibility)
36. [PII Accidentally in Training Data — Model Already Trained](#36-pii-in-training-data)
37. [Design: Enterprise Chatbot for Complex PDFs](#37-enterprise-pdf-chatbot)
38. [FastAPI Async LLM Endpoint + GET vs POST](#38-fastapi-async-llm)
39. [5 Levers to Pull When Inference Cost Explodes](#39-five-cost-levers)

---

<a id="1-rag-architecture-end-to-end"></a>
## 1. Explain the Complete RAG Architecture. How Would You Build It from Scratch? ⭐⭐⭐⭐

**RAG (Retrieval-Augmented Generation)** combines a retrieval system with an LLM to ground responses in actual data instead of relying on the model's parametric memory.

### End-to-End Architecture

```
┌──────────────────── OFFLINE (Ingestion Pipeline) ────────────────────┐
│                                                                       │
│  Documents (PDFs, web pages, DBs, APIs)                              │
│       │                                                               │
│       ▼                                                               │
│  [Document Loader] → raw text extraction                             │
│       │                                                               │
│       ▼                                                               │
│  [Chunking Engine] → split into overlapping passages                 │
│       │                                                               │
│       ▼                                                               │
│  [Embedding Model] → convert chunks to vectors                       │
│       │                                                               │
│       ▼                                                               │
│  [Vector Database] → store (vector, text, metadata)                  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────── ONLINE (Query Pipeline) ─────────────────────────┐
│                                                                       │
│  User Query                                                          │
│       │                                                               │
│       ▼                                                               │
│  [Query Transformation]                                               │
│  - Rewrite for clarity                                                │
│  - Decompose complex queries into sub-queries                        │
│  - Generate hypothetical answers (HyDE)                              │
│       │                                                               │
│       ▼                                                               │
│  [Embedding Model] → convert query to vector                         │
│       │                                                               │
│       ▼                                                               │
│  [Vector Search] → retrieve top-K similar chunks                     │
│       │                                                               │
│       ▼                                                               │
│  [Reranker] → re-score and filter results by relevance               │
│       │                                                               │
│       ▼                                                               │
│  [Context Assembly] → format retrieved chunks + metadata             │
│       │                                                               │
│       ▼                                                               │
│  [LLM Generation] → generate answer grounded in context              │
│       │                                                               │
│       ▼                                                               │
│  [Validation] → check faithfulness, format, safety                   │
│       │                                                               │
│       ▼                                                               │
│  Response to User (with citations)                                   │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Building from Scratch — Key Decisions

**Document loading:** Use `PyMuPDF` for PDFs (fastest), `unstructured` for mixed formats, `BeautifulSoup` for web pages. Handle tables, headers, and images separately — they need different processing.

**Chunking:** Don't just split by character count. Use recursive character splitting (respects paragraph/sentence boundaries) with 512-token chunks and 50-token overlap. See the dedicated chunking question below for advanced strategies.

**Embedding:** Choose a model based on your domain. `text-embedding-3-small` (OpenAI) for general use, domain-specific models for specialized corpora. See the embeddings question below.

**Vector store:** Start with FAISS for prototyping (local, free), move to Pinecone/Weaviate for production (managed, scalable). See the comparison question below.

**Retrieval:** Start with cosine similarity, add a reranker (Cohere Rerank, cross-encoder models) when baseline retrieval quality plateaus. Reranking typically improves precision by 10-20%.

**Generation:** Include retrieved chunks in the system prompt with clear formatting:
```
Answer the user's question based ONLY on the following context.
If the context doesn't contain the answer, say "I don't have enough information."

Context:
[Source: annual_report_2024.pdf, Page 42]
Revenue grew 23% year-over-year to $4.2B...

[Source: earnings_call_Q4.pdf, Page 3]
Operating margin improved to 28%...

User question: {query}
```

**Why this architecture beats a raw LLM:**
- The LLM only generates text from verified data (reduces hallucination)
- You can update knowledge without retraining (just re-index documents)
- Citations are traceable (each answer points to source documents)
- Works with private/proprietary data the model was never trained on

---

<a id="2-embeddings"></a>
## 2. What Are Embeddings? How Do You Choose an Embedding Model? ⭐⭐⭐

**Embeddings** are dense vector representations of text (or images, audio) in a high-dimensional space where semantic similarity corresponds to spatial proximity.

```
"The cat sat on the mat" → [0.023, -0.451, 0.882, ..., 0.167]  (1536 dimensions)
"A feline rested on a rug" → [0.019, -0.448, 0.879, ..., 0.170]  (very similar vector!)
"Stock prices rose today" → [-0.531, 0.204, -0.112, ..., 0.893]  (very different vector)
```

### How Embedding Models Work

Embedding models (like `text-embedding-3-small`) are trained on massive datasets of text pairs. They learn to place semantically similar text close together and dissimilar text far apart. The training objective is typically contrastive: given a pair (query, relevant_document), maximize their similarity; given (query, irrelevant_document), minimize it.

### Choosing an Embedding Model

| Model | Dimensions | Speed | Quality | Cost | Best For |
|---|---|---|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536 | Fast | Good | $0.02/1M tokens | General purpose, cost-sensitive |
| `text-embedding-3-large` (OpenAI) | 3072 | Medium | Better | $0.13/1M tokens | Higher accuracy needs |
| `voyage-3` (Voyage AI) | 1024 | Fast | Excellent for code | Variable | Code search, technical docs |
| `all-MiniLM-L6-v2` (open source) | 384 | Very fast | Decent | Free (self-hosted) | Prototyping, low-latency |
| `bge-large-en-v1.5` (BAAI) | 1024 | Medium | Very good | Free (self-hosted) | Self-hosted production |
| `Cohere embed-v3` | 1024 | Fast | Excellent | API pricing | Multilingual use cases |

### Selection Criteria

1. **Domain fit:** General-purpose embeddings work for most tasks. For code, legal, or medical domains, evaluate domain-specific models on YOUR data.

2. **Dimensionality:** Higher dimensions = more expressive but more storage cost and slower search. 1024-1536 is the sweet spot.

3. **Latency:** If embedding at query time (online), speed matters. If batch-embedding documents (offline), it doesn't.

4. **Self-hosted vs API:** Self-hosted models (Sentence Transformers) have zero marginal cost but need GPU infrastructure. APIs are simpler but add per-token cost.

5. **Benchmark vs your data:** MTEB benchmarks are useful but don't replace evaluation on YOUR retrieval task. Always test on a sample of your actual queries.

**Critical rule:** Once you choose an embedding model for a corpus, you can't mix models. If you re-embed with a different model, you must re-embed ALL documents — old and new vectors exist in different spaces and aren't comparable.

---

<a id="3-vector-database-comparison"></a>
## 3. Compare Vector Databases: Pinecone, Chroma, FAISS, Milvus, and Weaviate ⭐⭐⭐

| Feature | FAISS | Chroma | Pinecone | Milvus | Weaviate |
|---|---|---|---|---|---|
| **Type** | Library | Embedded DB | Managed cloud | Self-hosted / Zilliz Cloud | Self-hosted / Cloud |
| **Best for** | Prototyping, research | Small-medium apps | Production SaaS | Large-scale enterprise | Enterprise, multimodal |
| **Scale** | 1B+ vectors (single machine) | ~1M vectors | 1B+ vectors | 1B+ vectors | 100M+ vectors |
| **Persistence** | Manual (save/load) | Built-in | Fully managed | Built-in | Built-in |
| **Filtering** | Limited metadata | Basic metadata | Rich metadata filtering | Advanced hybrid search | GraphQL-like filtering |
| **Managed** | No | No | Yes (fully) | Zilliz Cloud option | Weaviate Cloud option |
| **Cost** | Free | Free | $0.08/1M reads | Free (self-hosted) | Free (self-hosted) |
| **Setup** | `pip install faiss-cpu` | `pip install chromadb` | API key | Docker / K8s | Docker / K8s |

### When to Use Each

**FAISS:** You're prototyping, running experiments, or your entire corpus fits in memory on one machine. Zero infrastructure overhead. Google-scale search teams use it as a building block.

**Chroma:** You want a quick embedded database for a small-to-medium application. Great developer experience, runs in-process. Outgrow it when you need multi-node scaling or advanced filtering.

**Pinecone:** You want zero-ops vector search in production. Fully managed, auto-scales, great metadata filtering. The "just works" option. Higher cost but eliminates DevOps burden.

**Milvus:** You need to self-host at enterprise scale. Handles billions of vectors, supports GPU acceleration, hybrid search (vector + keyword). The most feature-rich open-source option. Zilliz Cloud is the managed version.

**Weaviate:** You need multimodal search (text + image) or complex filtering. Good for enterprise knowledge graphs. Native support for multiple vectorizers and object schemas.

### Interview signal

Don't just name a database — explain WHY for your use case:
- "I chose FAISS for our proof-of-concept because our corpus was 50K documents and we needed zero infrastructure overhead to validate the retrieval quality first."
- "We migrated to Pinecone for production because we needed managed scaling, metadata filtering for multi-tenant isolation, and 99.9% uptime without dedicated DevOps."

---

<a id="4-framework-comparison"></a>
## 4. LangChain vs LlamaIndex vs LangGraph vs CrewAI — When to Choose Each? ⭐⭐⭐

### LangChain
**What it is:** A general-purpose framework for building LLM applications with chains, tools, and memory.

**Strengths:** Massive ecosystem (100+ integrations), lots of examples, large community. Good for prototyping diverse LLM applications quickly.

**Weaknesses:** Deep abstraction layers make debugging painful. API changes frequently (breaking changes between versions). Performance overhead from abstractions. "LangChain errors" is a meme in the community for a reason.

**Use when:** You need to prototype quickly with many integrations, or you're building a simple chain (prompt → LLM → parse → output).

### LlamaIndex
**What it is:** A data framework specifically designed for connecting LLMs to data sources, optimized for RAG.

**Strengths:** Best-in-class data ingestion (200+ data connectors). Excellent query pipeline abstraction. Built-in evaluation tools. More focused than LangChain.

**Weaknesses:** Narrower scope — if your app isn't data-retrieval-heavy, you'll outgrow it. Less flexible for non-RAG agent patterns.

**Use when:** Your primary use case is RAG, document Q&A, or knowledge base search. LlamaIndex's query engines and data connectors are purpose-built for this.

### LangGraph
**What it is:** A framework for building stateful, multi-step agent workflows as graphs. Built on top of LangChain.

**Strengths:** First-class support for cycles, branching, human-in-the-loop, and state persistence. The graph abstraction makes complex workflows explicit and debuggable. Native checkpointing for long-running tasks.

**Weaknesses:** Steeper learning curve. Overkill for simple linear pipelines. Inherits some LangChain complexity.

**Use when:** You need multi-agent orchestration, workflows with cycles (retry loops, iterative refinement), human approval steps, or stateful long-running tasks.

### CrewAI
**What it is:** A framework for orchestrating teams of AI agents with defined roles, goals, and collaboration patterns.

**Strengths:** Intuitive "crew" metaphor (agents have roles like "Researcher," "Writer," "Editor"). Easy to set up multi-agent collaboration. Built-in delegation between agents.

**Weaknesses:** Less mature than LangChain/LangGraph. Less control over low-level execution. Can be opaque when debugging inter-agent communication.

**Use when:** You're building a system where multiple specialized agents need to collaborate on a task (research → analyze → write → review), and the role-based abstraction fits your mental model.

### Decision Framework

```
Simple chain (prompt → LLM → output)?
  → Raw SDK calls. No framework needed.

RAG / document Q&A?
  → LlamaIndex (purpose-built for this)

Complex multi-step workflow with cycles?
  → LangGraph (graph-based state management)

Multi-agent collaboration with roles?
  → CrewAI (role-based team metaphor)

Need to prototype quickly with many integrations?
  → LangChain (broadest ecosystem)

Production system with strict requirements?
  → Start with raw SDK, add framework only for specific subsystems that need it
```

---

<a id="5-ai-agents-and-multi-agent-architecture"></a>
## 5. Explain AI Agents, Multi-Agent Systems, and Supervisor Architecture ⭐⭐⭐⭐

### What Is an AI Agent?

An AI agent is an LLM that can reason about tasks, decide to use tools, observe results, and iterate until the task is complete. The key difference from a basic LLM call: agents have **autonomy** — they decide WHAT to do, not just generate text.

```
Basic LLM:   Input → Model → Output (one shot)
Agent:        Input → Think → Act → Observe → Think → Act → ... → Output (iterative)
```

### Single Agent

One LLM with access to multiple tools. It handles the entire task end-to-end.

```python
# Pseudocode
while not done:
    action = llm.decide(user_query, context, available_tools)
    if action.type == "respond":
        return action.response
    result = tools.execute(action.tool, action.args)
    context.append(result)
```

Good for: straightforward tasks, well-defined tool sets, when you need one coherent reasoning chain.

### Multi-Agent System

Multiple specialized agents, each with their own system prompt, tools, and expertise.

**Supervisor pattern:**
```
         ┌──────────────┐
         │  Supervisor    │  Decomposes task, delegates, synthesizes
         └──────┬───────┘
          ┌─────┼──────┐
          ▼     ▼      ▼
       Agent  Agent  Agent
       (research) (code) (review)
```

The supervisor:
1. Receives the user request
2. Breaks it into subtasks
3. Assigns each subtask to a specialist
4. Collects results
5. Synthesizes the final response

**Hierarchical pattern:**
```
         ┌──────────────┐
         │  Orchestrator  │
         └──────┬───────┘
          ┌─────┼──────┐
          ▼     ▼      ▼
       Team    Team   Team
       Lead    Lead   Lead
        │       │      │
       ┌┴┐    ┌┴┐   ┌┴┐
       W  W   W  W   W  W   (Worker agents)
```

For very complex tasks: the orchestrator delegates to team leads, who further delegate to workers.

**Peer-to-peer pattern:**
Agents communicate directly with each other without a central coordinator. Good for collaborative tasks (debate, brainstorming) but harder to control.

### When Multi-Agent > Single Agent

- **Context window pressure:** One agent can't fit all the context it needs. Split into specialists that each focus on a narrower context.
- **Conflicting system prompts:** A "devil's advocate" agent and an "optimist" agent can't share a system prompt.
- **Different tool access:** The research agent needs web search; the code agent needs a code executor. Separating them improves security.
- **Independent scaling:** If research takes 80% of compute, scale research agents without scaling others.

---

<a id="6-function-calling--tool-calling"></a>
## 6. What Is Function Calling? How Is It Implemented? ⭐⭐

Function calling lets the LLM output structured JSON requesting that your code execute a specific function with specific arguments.

### OpenAI Implementation

```python
import openai

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["city"],
            },
        },
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
    tools=tools,
)

# Model returns a tool_call (not text)
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name = "get_weather"
# tool_call.function.arguments = '{"city": "Paris", "unit": "celsius"}'

# YOUR CODE executes the function
result = get_weather(city="Paris", unit="celsius")

# Feed result back
messages.append(response.choices[0].message)  # Assistant's tool call
messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": json.dumps(result)})

# Model generates final response incorporating the tool result
final_response = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools)
```

**The model NEVER executes code.** It outputs a JSON request. Your code executes, validates, and returns the result.

---

<a id="7-mcp-model-context-protocol"></a>
## 7. What Is MCP (Model Context Protocol)? Why Is It Gaining Popularity? ⭐⭐⭐

**MCP** is an open standard (originated from Anthropic) that defines how LLM applications connect to external tools and data sources through a uniform protocol.

**Before MCP:** Every application built custom integrations for every tool. Claude desktop had a Slack plugin, a GitHub plugin, a Google Drive plugin — all built differently.

**With MCP:** Tools are published as MCP servers. Any MCP-compatible client can discover and use them.

```
┌─────────────────┐         ┌──────────────┐
│  Claude Desktop  │ ←─MCP─→ │  GitHub MCP   │
│  (MCP client)    │ ←─MCP─→ │  Slack MCP    │
│                  │ ←─MCP─→ │  Database MCP │
└─────────────────┘         └──────────────┘

┌─────────────────┐         ┌──────────────┐
│  Your Custom App │ ←─MCP─→ │  Same servers!│
│  (MCP client)    │ ←─MCP─→ │  No code change│
└─────────────────┘         └──────────────┘
```

**Why it's gaining popularity:**
1. **Write once, use everywhere.** A GitHub MCP server works in Claude, Cursor, your custom app — no per-client integration.
2. **Security model.** MCP defines how tools declare permissions, how clients authenticate, and how data flows. Standardized security is better than 100 custom implementations.
3. **Discovery.** Clients can query what tools a server offers at runtime. Add a new tool to the server, and all clients see it automatically.
4. **Ecosystem.** Growing library of pre-built MCP servers for popular services. Reduces the "build every integration from scratch" burden.

**MCP server structure (Python with FastMCP):**
```python
from fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
def get_weather(city: str, unit: str = "celsius") -> dict:
    """Get current weather for a city."""
    # Implementation
    return {"temperature": 22, "condition": "sunny", "city": city}

mcp.run()
```

---

<a id="8-llm-latency-optimization"></a>
## 8. How Do You Optimize LLM Latency and Reduce Inference Costs? ⭐⭐⭐⭐

### Latency Optimization

**1. Streaming:** Don't wait for the full response. Stream tokens to the user as they're generated. First-token latency drops from 5s → 200ms.

**2. Model selection routing:** Route simple queries to smaller, faster models:
```
Simple factual query → gpt-4o-mini (200ms) 
Complex reasoning → gpt-4o (2-5s)
Routing classifier → gpt-4o-mini (100ms overhead, saves 2-4s on 60% of requests)
```

**3. Prompt optimization:** Shorter prompts = faster processing. Remove redundant instructions, compress system prompts, use concise few-shot examples.

**4. Caching:** Semantic cache for similar queries (embedding similarity > 0.95). Exact-match cache for repeated queries. Cache hit = 0ms LLM latency.

**5. Parallel tool execution:** When the agent needs multiple independent tools, run them simultaneously:
```python
# Sequential: 200ms + 300ms + 150ms = 650ms
# Parallel: max(200, 300, 150) = 300ms
results = await asyncio.gather(tool1(), tool2(), tool3())
```

**6. KV cache (provider-side):** Modern LLM APIs cache the key-value attention matrices for repeated prompt prefixes. If your system prompt is the same across calls, subsequent calls are faster.

### Cost Reduction

**1. Model tiering:** 80% of requests can be handled by mini models at 1/10th the cost.

**2. Batch API:** For non-real-time work (eval, data enrichment, embeddings), use batch endpoints at 50% cost.

**3. Prompt compression:** Shorter prompts = fewer input tokens = lower cost. Tools like `gzip`-based prompt compression or automatic summarization of long contexts.

**4. Response length limits:** Set `max_tokens` appropriately. Don't pay for 4000-token responses when 200 tokens suffice.

**5. Caching:** A 30% cache hit rate means 30% fewer API calls.

**6. Self-hosted models:** For high-volume, low-complexity tasks, a self-hosted 7B-13B model can be cheaper than API calls at scale (break-even: ~100K requests/day).

---

<a id="9-prompt-engineering-techniques"></a>
## 9. Explain Prompt Engineering Techniques ⭐⭐⭐

### Zero-Shot
No examples. Just the instruction.
```
Classify the sentiment of this review as positive, negative, or neutral:
"The food was amazing but the service was slow."
```
Use when: The task is straightforward and the model understands it inherently.

### Few-Shot
Provide 2-5 examples before the actual query.
```
Review: "Great product!" → Sentiment: positive
Review: "Terrible experience." → Sentiment: negative
Review: "It was okay." → Sentiment: neutral

Review: "The food was amazing but the service was slow." → Sentiment:
```
Use when: The task has nuance that zero-shot misses, or you want to establish an output format.

### Chain-of-Thought (CoT)
Ask the model to reason step-by-step before answering.
```
Q: A store has 15 apples. 3 customers each buy 2 apples, and then 10 more apples are delivered. How many apples are in the store?

Think step by step:
1. Starting apples: 15
2. Apples sold: 3 customers × 2 apples = 6 apples
3. After sales: 15 - 6 = 9 apples
4. After delivery: 9 + 10 = 19 apples

Answer: 19 apples
```
Use when: Multi-step reasoning, math, logic problems, complex analysis. Dramatically improves accuracy on reasoning tasks.

### Self-Consistency
Run the same prompt multiple times (with temperature > 0), then take the majority vote.
```
Run 1: "The answer is 19" ← 
Run 2: "The answer is 19" ←  Majority: 19 ✓
Run 3: "The answer is 17"
Run 4: "The answer is 19" ←
Run 5: "The answer is 19" ←
```
Use when: High-stakes decisions where reliability matters more than cost. Typically 3-5 runs with CoT.

### ReAct (Reasoning + Acting)
The model alternates between thinking and using tools (covered in detail in Week 3).
```
Thought: I need current weather data. I can't generate this from memory.
Action: get_weather(city="NYC")
Observation: {"temp": 72, "condition": "sunny"}
Thought: I have the data. I can answer the user.
Answer: It's 72°F and sunny in NYC.
```
Use when: Tasks requiring external data or tool use.

### Tree of Thoughts (ToT)
Explore multiple reasoning paths simultaneously, evaluate each, and select the best.
```
Path A: Assume the user wants a refund → leads to refund flow → check: does the user mention money? Yes → viable
Path B: Assume the user wants technical help → leads to troubleshooting → check: does the user mention a bug? No → less viable
Path C: Assume the user is just frustrated → leads to empathy → check: does the user express emotion? Somewhat → partially viable

Best path: A (strongest signal alignment)
```
Use when: Complex problems with multiple possible approaches. More expensive (multiple LLM calls) but higher quality for ambiguous tasks.

---

<a id="10-llm-evaluation"></a>
## 10. How Do You Evaluate an LLM Application? ⭐⭐⭐⭐

### Key Metrics

**Faithfulness:** Does the generated answer ONLY use information from the provided context? A response that adds information not in the retrieved documents is unfaithful (even if factually correct).

**Answer Relevancy:** Does the answer actually address the question? A faithful but off-topic response scores low here.

**Context Precision:** Of the retrieved documents, how many were actually relevant to answering the question? High precision = retrieved documents were well-targeted.

**Context Recall:** Of all the documents that COULD have answered the question, how many did the retrieval system find? High recall = didn't miss important sources.

### Evaluation Frameworks

**RAGAS (Retrieval-Augmented Generation Assessment):**
Open-source framework that evaluates RAG pipelines on: faithfulness, answer relevancy, context precision, context recall. Uses LLM-as-judge to compute metrics without manual labeling.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset=eval_dataset,  # questions + answers + contexts + ground_truth
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(result)
# {"faithfulness": 0.87, "answer_relevancy": 0.92, "context_precision": 0.78}
```

**DeepEval:**
Python-first eval framework with built-in metrics for: hallucination, toxicity, bias, answer relevancy, faithfulness. Integrates with pytest for CI/CD.

```python
from deepeval import assert_test
from deepeval.metrics import HallucinationMetric
from deepeval.test_case import LLMTestCase

test_case = LLMTestCase(
    input="What is our refund policy?",
    actual_output="Our refund policy allows returns within 30 days.",
    context=["Customers can return products within 30 days for a full refund."],
)

metric = HallucinationMetric(threshold=0.5)
assert_test(test_case, [metric])
```

### Building a Gold Evaluation Set

1. **Collect real user queries** from production logs (anonymized)
2. **Human-label ground truth answers** for 200-500 queries
3. **Tag each query** with metadata: topic, difficulty, expected tools, expected sources
4. **Include adversarial cases:** prompt injection attempts, off-topic queries, ambiguous questions
5. **Version the eval set** — it evolves as your system evolves
6. **Run evals in CI/CD** — block deploys that drop below thresholds

---

<a id="11-preventing-hallucinations"></a>
## 11. How Do You Prevent Hallucinations in a RAG Application? ⭐⭐⭐⭐

Hallucination in RAG = the model generates information NOT present in the retrieved context. This is different from a general LLM hallucination because you provided the correct context but the model ignored or embellished it.

### Prevention Layers

**1. Better retrieval (prevent the model from having to guess):**
- More relevant chunks → less temptation to fill gaps from parametric memory
- Use reranking to push truly relevant chunks to the top
- Retrieve more chunks (top-20 instead of top-5) then rerank down to top-5

**2. Explicit grounding instructions:**
```
Answer ONLY based on the provided context.
If the context doesn't contain enough information to answer, say:
"I don't have enough information in my sources to answer this question."
Do NOT use your general knowledge.
```

**3. Citation enforcement:**
Force the model to cite specific passages:
```
For every claim in your answer, include a citation in [brackets] referencing 
the source document. Example: "Revenue grew 23% [annual_report_2024, p.42]"
```
This makes hallucinations visible — a fabricated claim won't have a valid citation.

**4. Faithfulness validation (post-generation):**
Use an LLM-as-judge or NLI (natural language inference) model to check: "Is every claim in the response supported by the context?"

```python
def check_faithfulness(response: str, context: str) -> float:
    """Use an LLM to verify each claim is grounded in context."""
    claims = extract_claims(response)  # Break response into individual claims
    
    grounded_count = 0
    for claim in claims:
        is_grounded = judge_model.evaluate(
            f"Is this claim supported by the context?\n"
            f"Claim: {claim}\n"
            f"Context: {context}\n"
            f"Answer: YES or NO"
        )
        if is_grounded == "YES":
            grounded_count += 1
    
    return grounded_count / len(claims)  # Faithfulness score
```

**5. Temperature = 0 for factual tasks:** Lower temperature reduces creative embellishment.

**6. Shorter context windows:** Don't stuff 20 chunks into context. The "lost in the middle" problem means models pay less attention to chunks in the middle of long contexts.

**Measuring success:** "Reduced hallucination rate from 34% to 8% on a 50K-doc corpus" — this is the kind of metric that gets you hired.

---

<a id="12-chunking-strategies"></a>
## 12. Explain Chunking Strategies ⭐⭐⭐

### Fixed-Size Chunking
Split text into equal-sized chunks (e.g., 512 tokens each) with overlap.

```
Chunk 1: tokens 0-512
Chunk 2: tokens 462-974 (50-token overlap)
Chunk 3: tokens 924-1436
```

**Pros:** Simple, predictable chunk sizes. **Cons:** Splits mid-sentence, mid-paragraph, mid-thought. Important context gets severed.

### Recursive Character Splitting
Split by hierarchy: paragraph → sentence → word, respecting natural boundaries.

```python
# LangChain's RecursiveCharacterTextSplitter
separators = ["\n\n", "\n", ". ", " ", ""]
# First tries to split on paragraphs, then sentences, then words
```

**Pros:** Respects document structure. **Cons:** Chunk sizes vary, some chunks may be very short.

### Semantic Chunking
Group sentences by semantic similarity. Adjacent sentences with high similarity stay together; a drop in similarity triggers a chunk boundary.

```
Sentence 1: "Our Q4 revenue was $4.2B."     ─┐
Sentence 2: "This represents 23% YoY growth." ─┤ High similarity → same chunk
Sentence 3: "The board approved a dividend."  ─┘
                                                  ← Similarity drops
Sentence 4: "Our new CEO starts in January." ─┐
Sentence 5: "She previously led Product."    ─┘ Different topic → new chunk
```

**Pros:** Chunks are topically coherent. **Cons:** Computationally expensive (requires embedding every sentence), unpredictable chunk sizes.

### Parent-Child (Hierarchical) Chunking
Store small chunks for retrieval precision but return larger parent chunks for context richness.

```
Parent chunk (1024 tokens): Full section about Q4 financials
├── Child chunk (256 tokens): Revenue paragraph
├── Child chunk (256 tokens): Expenses paragraph
├── Child chunk (256 tokens): Profit margin paragraph
└── Child chunk (256 tokens): Forecast paragraph

Search matches "child chunk: Revenue paragraph"
→ Return "parent chunk: Full section about Q4 financials" to LLM
```

**Pros:** Best of both worlds — precise retrieval + rich context. **Cons:** More complex ingestion pipeline, more storage.

### Hybrid Chunking
Combine strategies. Common pattern:
1. Split by document structure (headers, sections)
2. Within each section, use recursive splitting at ~512 tokens
3. Add metadata (section title, page number, document name) to each chunk

**Why the strategy matters:** Bad chunking is the #1 cause of bad retrieval, which is the #1 cause of bad RAG answers. If a relevant fact is split across two chunks and neither chunk alone contains enough context, retrieval fails silently.

---

<a id="13-mmr-and-rrf"></a>
## 13. What Are MMR and RRF? ⭐⭐⭐

### MMR (Maximum Marginal Relevance)
**Problem:** Vector search returns the top-5 most similar chunks, but they're all from the same paragraph (because similar text has similar embeddings). You get 5 copies of the same information.

**Solution:** MMR balances relevance to the query AND diversity among selected results:

```
Score(chunk) = λ × similarity(chunk, query) - (1-λ) × max_similarity(chunk, already_selected)
```

- `λ = 1.0`: Pure relevance (no diversity — default vector search)
- `λ = 0.5`: Balanced — relevant AND diverse
- `λ = 0.0`: Maximum diversity (ignores relevance — useless)

**Typical setting:** `λ = 0.5 to 0.7`

**Impact:** Instead of 5 chunks saying "Revenue grew 23%," you get: one about revenue, one about expenses, one about the forecast, one about the CEO quote, one about market conditions. Much better context for the LLM.

### RRF (Reciprocal Rank Fusion)
**Problem:** You have multiple retrieval methods (vector search, BM25 keyword search, metadata filter) that each return ranked lists. How do you combine them?

**Solution:** RRF combines rankings using a simple formula:

```
RRF_score(chunk) = Σ 1 / (k + rank_in_system_i)
```

Where `k` is a constant (typically 60) that prevents high-ranked items from dominating.

**Example:**
```
Vector search ranking:  [Doc A: rank 1, Doc B: rank 3, Doc C: rank 2]
Keyword search ranking: [Doc B: rank 1, Doc C: rank 2, Doc A: rank 5]

RRF scores (k=60):
Doc A: 1/(60+1) + 1/(60+5) = 0.0164 + 0.0154 = 0.0318
Doc B: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323 ← Winner!
Doc C: 1/(60+2) + 1/(60+2) = 0.0161 + 0.0161 = 0.0322
```

Doc B wins because it ranked well in BOTH systems. This is hybrid search — combining semantic understanding (vectors) with exact matching (keywords).

**When reranking helps vs adds cost:** Reranking (Cohere Rerank, cross-encoders) adds 100-500ms latency. It's worth it when your retrieval precision is below 80%. If your retrieval is already good (90%+ precision), reranking might only add latency without meaningful quality improvement. Always measure before and after.

---

<a id="14-kv-cache-speculative-decoding-continuous-batching"></a>
## 14. KV Cache, Speculative Decoding, and Continuous Batching ⭐⭐⭐⭐

These are inference optimization techniques used by LLM serving infrastructure.

### KV Cache
**Problem:** Transformer attention computes Key and Value matrices for every previous token at every generation step. For a 4000-token context, generating token #4001 recomputes attention over all 4000 tokens.

**Solution:** Cache the K and V matrices from previous tokens. When generating token #4001, only compute K/V for the new token and reuse the cached matrices.

**Impact:** Without KV cache: O(n²) per token. With KV cache: O(n) per token. Massive speedup for long sequences.

**Trade-off:** KV cache uses GPU memory proportional to (context_length × num_layers × hidden_dim). For long contexts (100K+ tokens), KV cache can consume more memory than the model weights themselves.

**Prompt caching (API-level):** Some providers (Anthropic, OpenAI) cache KV states for repeated prompt prefixes. If your system prompt is identical across calls, subsequent calls skip the prefill computation for that prefix. This reduces both latency and cost.

### Speculative Decoding
**Problem:** Large models (405B parameters) generate tokens slowly — ~15 tokens/second.

**Solution:** Use a small, fast "draft" model to generate a sequence of candidate tokens, then verify them all at once with the large model.

```
Draft model (7B, fast): generates 8 candidate tokens in 1 step
Large model (405B): verifies all 8 in a single forward pass
  → Accepts tokens 1-6 (correct)
  → Rejects tokens 7-8 (regenerates from large model)
  
Net: 6 tokens at the cost of ~2 forward passes instead of 6
```

**Impact:** 2-3x speedup in tokens per second for the large model, with zero quality degradation (the large model verifies every token).

### Continuous Batching
**Problem:** Traditional batching waits until N requests accumulate, processes them together, and returns. Short requests must wait for long requests to finish (head-of-line blocking).

**Solution:** Continuous batching inserts new requests into the batch as soon as a slot opens (when a previous request finishes generating). Requests enter and leave the batch independently.

```
Time →
Traditional batch:  [Req A ████████] [Req B ██] [Req C ████]  (B waits for A)
Continuous batch:   [Req A ████████]
                    [Req B ██]        → B finishes, slot freed
                    [Req C ████]      → C enters freed slot
```

**Impact:** 2-5x higher throughput for the same hardware. Critical for multi-tenant serving (multiple users with different request lengths).

---

<a id="15-enterprise-genai-security"></a>
## 15. How Do You Secure an Enterprise GenAI Application? ⭐⭐⭐⭐

### PII Masking
Detect and mask personally identifiable information BEFORE it reaches the LLM:

```python
# Input: "My SSN is 123-45-6789 and I live at 42 Oak Street"
# After masking: "My SSN is [SSN_REDACTED] and I live at [ADDRESS_REDACTED]"

# Tools: Microsoft Presidio, AWS Comprehend, Google DLP, custom regex
```

The LLM processes masked text. If the response needs to include PII (e.g., confirming an address), re-inject from the original data AFTER generation — never through the model.

### Prompt Injection Defense
Three attack vectors:
- **Direct:** User types malicious instructions ("ignore your system prompt")
- **Indirect:** Malicious content in tool outputs, documents, or web pages
- **Encoded:** Instructions hidden in base64, Unicode, or other encodings

Defenses:
- Input classifier trained on injection patterns (pre-LLM filter)
- System prompt that explicitly warns about injection attempts
- Output validation that checks for anomalous behavior (unexpected tool calls, data exfiltration attempts)
- Sandboxing: agents processing untrusted content have reduced tool access

### Jailbreak Protection
Jailbreaks try to bypass the model's safety training. Defenses:
- Content moderation API on outputs (OpenAI Moderation, Anthropic content filtering)
- Canary tokens in system prompts (if the model repeats them, it's been manipulated)
- Output classifiers trained on known jailbreak outputs
- Rate limiting: rapid successive attempts trigger lockout

### RBAC (Role-Based Access Control)
Different users get different agent capabilities:
```
Anonymous user → can only chat, no tools
Authenticated user → chat + search + basic tools
Admin → chat + search + all tools including write operations
System → full access (for internal automation)
```

### Guardrails
Runtime safety checks independent of the model's training:
- Topic blocking: "Don't discuss competitors' products"
- Output format enforcement: "Always respond with JSON matching this schema"
- Factual grounding: "Only use information from provided documents"
- Brand safety: "Don't make promises about refunds or timelines"
- Response length limits: prevent the model from dumping excessive data

### Content Moderation
- Pre-generation: classify user input for toxicity, hate speech, harmful intent
- Post-generation: classify model output before delivering to user
- Use dedicated classifier models (cheaper and faster than routing through the main LLM)
- Human escalation: flag edge cases for human review

---

<a id="16-llm-fine-tuning"></a>
## 16. LLM Fine-Tuning — LoRA, QLoRA, PEFT, RLHF ⭐⭐⭐⭐

### When to Fine-Tune vs Use RAG

**Use RAG when:**
- You need the model to access specific, changing data
- Your knowledge base updates frequently
- You need citations and traceability
- The model's base capabilities are sufficient, it just needs the right context

**Fine-tune when:**
- You need to change the model's behavior, style, or format consistently
- You need domain-specific terminology or reasoning patterns
- Prompt engineering has hit diminishing returns
- You need to compress knowledge that doesn't fit in prompts (very specialized domains)
- Latency is critical and you can't afford long prompts

### Fine-Tuning Techniques

**Full fine-tuning:** Update ALL model parameters. Requires enormous compute (8+ A100 GPUs for a 7B model). Creates a full copy of the model. Only practical for well-funded teams.

**LoRA (Low-Rank Adaptation):**
Instead of updating all parameters, inject small trainable "adapter" matrices into the model's attention layers. Only 0.1-1% of parameters are trainable.

```
Original weight matrix W: [4096 × 4096] = 16.7M parameters (frozen)
LoRA adapter: A [4096 × 16] × B [16 × 4096] = 131K parameters (trainable)
Effective weight: W + A × B
```

**Impact:** Train a 7B model on a single A100 GPU instead of 8+. Adapter weights are ~10-50 MB (vs ~14 GB for the full model). Multiple LoRA adapters can be swapped at inference time.

**QLoRA (Quantized LoRA):**
Quantize the base model to 4-bit precision, then apply LoRA adapters in higher precision. Reduces memory by 4x.

```
Full fine-tuning 7B: ~120 GB GPU memory
LoRA 7B: ~32 GB GPU memory
QLoRA 7B: ~12 GB GPU memory (fits on a single consumer GPU!)
```

**PEFT (Parameter-Efficient Fine-Tuning):**
Umbrella term for all techniques that fine-tune a small number of parameters: LoRA, prefix tuning, prompt tuning, adapter layers. The Hugging Face `peft` library implements all of these.

**RLHF (Reinforcement Learning from Human Feedback):**
The technique that made ChatGPT work. Three-step process:
1. **Supervised fine-tuning:** Train on human-written examples
2. **Reward model training:** Train a model to predict which response a human would prefer
3. **RL optimization (PPO):** Use the reward model to guide the LLM toward preferred behaviors

Most teams don't do RLHF themselves — it requires specialized infrastructure and large amounts of human preference data. Instead, they use models already RLHF-aligned (GPT-4, Claude) and fine-tune with LoRA/QLoRA for their specific domain.

---

<a id="17-production-llm-monitoring"></a>
## 17. How Would You Monitor an LLM Application in Production? ⭐⭐⭐⭐

### Metrics to Track

**Operational metrics:**
- Request latency (p50, p95, p99)
- Throughput (requests/second)
- Error rate (by type: timeout, rate limit, validation failure, model error)
- Token usage (input + output, per request, per user, per model)
- Cost per request, per user, per day
- Cache hit rate

**Quality metrics:**
- Faithfulness score (sample 5% of responses, run through eval)
- User satisfaction (thumbs up/down, NPS if available)
- Hallucination rate (automated detection on sampled responses)
- Tool selection accuracy (was the right tool called?)
- Task completion rate (for agent systems)
- Refusal rate (how often does the model decline to answer?)

**Safety metrics:**
- Prompt injection attempt rate
- Guardrail trigger rate (by rule)
- PII leak detection rate
- Content moderation flag rate

**Drift metrics:**
- Input distribution change (are users asking different types of questions?)
- Output length distribution change (is the model suddenly verbose?)
- Tool call distribution change (sudden shift in which tools are used)
- Model confidence score distribution

### Monitoring Architecture

```
Application → Structured Logs → Log Aggregator (DataDog/Grafana)
    │                                      │
    ├→ Metrics (Prometheus) → Dashboards   │
    │                                      │
    ├→ Traces (OpenTelemetry) → Trace DB   │
    │                                      │
    └→ Eval Pipeline (async) → Quality DB → Alerts
```

### Alerting Rules

```yaml
alerts:
  - name: high_error_rate
    condition: error_rate > 5% for 5 minutes
    severity: page
    
  - name: latency_degradation
    condition: p99_latency > 10s for 10 minutes
    severity: warning
    
  - name: cost_spike
    condition: hourly_cost > 2x rolling_7d_average
    severity: page
    
  - name: quality_regression
    condition: daily_faithfulness_score < 0.80
    severity: warning
    
  - name: hallucination_spike
    condition: hourly_hallucination_rate > 15%
    severity: page
```

---

<a id="18-zonal-failure-resilience"></a>
## 18. How Do You Design Applications to Survive Zonal Failures? ⭐⭐⭐

### What Is a Zonal Failure?

Cloud providers (AWS, GCP, Azure) organize infrastructure into regions (e.g., us-east-1) and availability zones within regions (e.g., us-east-1a, us-east-1b). A zonal failure means all compute in one zone is unavailable — network outage, power failure, hardware issue.

### Surviving Zonal Failures for AI Applications

**1. Multi-AZ deployment:**
Deploy your application across at least 2 availability zones. Kubernetes does this automatically with pod anti-affinity rules:
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: topology.kubernetes.io/zone
```

**2. Stateless application design:**
Your AI service should be stateless — conversation state in Redis/DynamoDB (replicated across zones), model weights in shared storage (S3/GCS), configuration in environment variables. If a pod dies, another pod in another zone picks up the next request.

**3. Database replication:**
Vector databases and conversation stores must replicate across zones:
- Pinecone: Built-in multi-AZ replication
- Self-hosted Milvus: Configure cross-AZ replicas
- Redis: Sentinel or Cluster mode with replicas in different AZs

**4. Load balancer health checks:**
The load balancer must detect zone failures and stop routing to unhealthy zones within seconds:
```
Zone A: healthy → receives traffic
Zone B: unhealthy → removed from rotation
Zone C: healthy → receives traffic (now handling Zone B's share)
```

**5. Multi-region challenges (advanced):**
For global applications, you need multi-REGION (not just multi-AZ):
- LLM API latency varies by region (API endpoint closest to users)
- Model serving: replicate model weights to each region
- Vector DB: cross-region replication with eventual consistency
- Container images: replicate to per-region container registries to avoid cross-region pull latency
- Data sovereignty: some regions require data to stay within borders

**6. Graceful degradation during failures:**
Design your app so that when one component fails, the whole system doesn't crash:
- LLM API down → fall back to cached responses or simpler model
- Vector DB down → fall back to keyword search
- One agent type unavailable → route to general-purpose agent
- All AI features down → serve static/cached content + "AI features temporarily unavailable"


# 🎯 Extra Prep Questions — Cross-Cutting AI Engineer Interview Topics

> **These questions span multiple weeks and cover advanced topics frequently asked in MAANG-level AI engineer interviews.**
>
> **Covers:** RAG architecture, embeddings, vector databases, frameworks, agent design, prompt engineering, evaluation, fine-tuning, inference optimization, security, production monitoring, and more.
>
> *Part of the [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/) interview prep series.*

---

## Table of Contents

1. [RAG Architecture (End-to-End)](#1-rag-architecture-end-to-end)
2. [Embeddings — What They Are and How to Choose](#2-embeddings)
3. [Vector Database Comparison](#3-vector-database-comparison)
4. [Framework Comparison — LangChain vs LlamaIndex vs LangGraph vs CrewAI](#4-framework-comparison)
5. [AI Agents and Multi-Agent Architecture](#5-ai-agents-and-multi-agent-architecture)
6. [Function Calling / Tool Calling Implementation](#6-function-calling--tool-calling)
7. [MCP (Model Context Protocol)](#7-mcp-model-context-protocol)
8. [LLM Latency Optimization and Inference Cost Reduction](#8-llm-latency-optimization)
9. [Prompt Engineering Techniques](#9-prompt-engineering-techniques)
10. [LLM Application Evaluation — RAGAS, DeepEval, and Metrics](#10-llm-evaluation)
11. [Preventing Hallucinations in RAG](#11-preventing-hallucinations)
12. [Chunking Strategies](#12-chunking-strategies)
13. [MMR and RRF — Advanced Retrieval Techniques](#13-mmr-and-rrf)
14. [KV Cache, Speculative Decoding, and Continuous Batching](#14-kv-cache-speculative-decoding-continuous-batching)
15. [Enterprise GenAI Security](#15-enterprise-genai-security)
16. [LLM Fine-Tuning — LoRA, QLoRA, PEFT, RLHF](#16-llm-fine-tuning)
17. [Production LLM Monitoring](#17-production-llm-monitoring)
18. [Designing Applications to Survive Zonal Failures](#18-zonal-failure-resilience)

19. [RAG Debugging — Wrong Answers, First Thing to Investigate](#19-rag-debugging)
20. [Retriever Finds Relevant Docs But Answer Quality Is Poor](#20-retrieval-generation-gap)
21. [Proving Embedding Improvements Actually Improved the System](#21-proving-embedding-improvements)
22. [Multi-Document Questions — Info Spread Across 5 Documents](#22-multi-document-questions)
23. [RAG Scaling — 10K to 1M Documents, What Breaks First](#23-rag-scaling)
24. [NLP-to-SQL Chatbot for 100+ Table RDBMS](#24-nlp-to-sql)
25. [Scientific Prompt Engineering — Isolating What Works](#25-scientific-prompt-engineering)
26. [Correct Answer in Chunk #12 But Top-K Is 5](#26-chunk-12-problem)
27. [Embedding Cost Optimization — $20K/Month Re-Embedding](#27-embedding-cost-optimization)
28. [RAG Streaming and Progressive Rendering](#28-rag-streaming)
29. [Agent Action Safety — Preventing Accidental Destructive Actions](#29-agent-action-safety)
30. [AI Gateway — Multi-Provider Routing with Auto-Fallback](#30-ai-gateway)
31. [Context Compression at 40K Tokens](#31-context-compression)
32. [Safe Model Migration in Production](#32-safe-model-migration)
33. [Catastrophic Forgetting After Fine-Tuning](#33-catastrophic-forgetting)
34. [Hybrid Search — BM25 + Vector vs Pure Vector](#34-hybrid-search)
35. [Reproducing a Fine-Tuning Run 6 Months Later](#35-finetune-reproducibility)
36. [PII Accidentally in Training Data — Model Already Trained](#36-pii-in-training-data)
37. [Design: Enterprise Chatbot for Complex PDFs](#37-enterprise-pdf-chatbot)
38. [FastAPI Async LLM Endpoint + GET vs POST](#38-fastapi-async-llm)
39. [5 Levers to Pull When Inference Cost Explodes](#39-five-cost-levers)
---

<a id="1-rag-architecture-end-to-end"></a>
## 1. Explain the Complete RAG Architecture. How Would You Build It from Scratch? ⭐⭐⭐⭐

**RAG (Retrieval-Augmented Generation)** combines a retrieval system with an LLM to ground responses in actual data instead of relying on the model's parametric memory.

### End-to-End Architecture

```
┌──────────────────── OFFLINE (Ingestion Pipeline) ────────────────────┐
│                                                                       │
│  Documents (PDFs, web pages, DBs, APIs)                              │
│       │                                                               │
│       ▼                                                               │
│  [Document Loader] → raw text extraction                             │
│       │                                                               │
│       ▼                                                               │
│  [Chunking Engine] → split into overlapping passages                 │
│       │                                                               │
│       ▼                                                               │
│  [Embedding Model] → convert chunks to vectors                       │
│       │                                                               │
│       ▼                                                               │
│  [Vector Database] → store (vector, text, metadata)                  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────── ONLINE (Query Pipeline) ─────────────────────────┐
│                                                                       │
│  User Query                                                          │
│       │                                                               │
│       ▼                                                               │
│  [Query Transformation]                                               │
│  - Rewrite for clarity                                                │
│  - Decompose complex queries into sub-queries                        │
│  - Generate hypothetical answers (HyDE)                              │
│       │                                                               │
│       ▼                                                               │
│  [Embedding Model] → convert query to vector                         │
│       │                                                               │
│       ▼                                                               │
│  [Vector Search] → retrieve top-K similar chunks                     │
│       │                                                               │
│       ▼                                                               │
│  [Reranker] → re-score and filter results by relevance               │
│       │                                                               │
│       ▼                                                               │
│  [Context Assembly] → format retrieved chunks + metadata             │
│       │                                                               │
│       ▼                                                               │
│  [LLM Generation] → generate answer grounded in context              │
│       │                                                               │
│       ▼                                                               │
│  [Validation] → check faithfulness, format, safety                   │
│       │                                                               │
│       ▼                                                               │
│  Response to User (with citations)                                   │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Building from Scratch — Key Decisions

**Document loading:** Use `PyMuPDF` for PDFs (fastest), `unstructured` for mixed formats, `BeautifulSoup` for web pages. Handle tables, headers, and images separately — they need different processing.

**Chunking:** Don't just split by character count. Use recursive character splitting (respects paragraph/sentence boundaries) with 512-token chunks and 50-token overlap. See the dedicated chunking question below for advanced strategies.

**Embedding:** Choose a model based on your domain. `text-embedding-3-small` (OpenAI) for general use, domain-specific models for specialized corpora. See the embeddings question below.

**Vector store:** Start with FAISS for prototyping (local, free), move to Pinecone/Weaviate for production (managed, scalable). See the comparison question below.

**Retrieval:** Start with cosine similarity, add a reranker (Cohere Rerank, cross-encoder models) when baseline retrieval quality plateaus. Reranking typically improves precision by 10-20%.

**Generation:** Include retrieved chunks in the system prompt with clear formatting:
```
Answer the user's question based ONLY on the following context.
If the context doesn't contain the answer, say "I don't have enough information."

Context:
[Source: annual_report_2024.pdf, Page 42]
Revenue grew 23% year-over-year to $4.2B...

[Source: earnings_call_Q4.pdf, Page 3]
Operating margin improved to 28%...

User question: {query}
```

**Why this architecture beats a raw LLM:**
- The LLM only generates text from verified data (reduces hallucination)
- You can update knowledge without retraining (just re-index documents)
- Citations are traceable (each answer points to source documents)
- Works with private/proprietary data the model was never trained on

---

<a id="2-embeddings"></a>
## 2. What Are Embeddings? How Do You Choose an Embedding Model? ⭐⭐⭐

**Embeddings** are dense vector representations of text (or images, audio) in a high-dimensional space where semantic similarity corresponds to spatial proximity.

```
"The cat sat on the mat" → [0.023, -0.451, 0.882, ..., 0.167]  (1536 dimensions)
"A feline rested on a rug" → [0.019, -0.448, 0.879, ..., 0.170]  (very similar vector!)
"Stock prices rose today" → [-0.531, 0.204, -0.112, ..., 0.893]  (very different vector)
```

### How Embedding Models Work

Embedding models (like `text-embedding-3-small`) are trained on massive datasets of text pairs. They learn to place semantically similar text close together and dissimilar text far apart. The training objective is typically contrastive: given a pair (query, relevant_document), maximize their similarity; given (query, irrelevant_document), minimize it.

### Choosing an Embedding Model

| Model | Dimensions | Speed | Quality | Cost | Best For |
|---|---|---|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536 | Fast | Good | $0.02/1M tokens | General purpose, cost-sensitive |
| `text-embedding-3-large` (OpenAI) | 3072 | Medium | Better | $0.13/1M tokens | Higher accuracy needs |
| `voyage-3` (Voyage AI) | 1024 | Fast | Excellent for code | Variable | Code search, technical docs |
| `all-MiniLM-L6-v2` (open source) | 384 | Very fast | Decent | Free (self-hosted) | Prototyping, low-latency |
| `bge-large-en-v1.5` (BAAI) | 1024 | Medium | Very good | Free (self-hosted) | Self-hosted production |
| `Cohere embed-v3` | 1024 | Fast | Excellent | API pricing | Multilingual use cases |

### Selection Criteria

1. **Domain fit:** General-purpose embeddings work for most tasks. For code, legal, or medical domains, evaluate domain-specific models on YOUR data.

2. **Dimensionality:** Higher dimensions = more expressive but more storage cost and slower search. 1024-1536 is the sweet spot.

3. **Latency:** If embedding at query time (online), speed matters. If batch-embedding documents (offline), it doesn't.

4. **Self-hosted vs API:** Self-hosted models (Sentence Transformers) have zero marginal cost but need GPU infrastructure. APIs are simpler but add per-token cost.

5. **Benchmark vs your data:** MTEB benchmarks are useful but don't replace evaluation on YOUR retrieval task. Always test on a sample of your actual queries.

**Critical rule:** Once you choose an embedding model for a corpus, you can't mix models. If you re-embed with a different model, you must re-embed ALL documents — old and new vectors exist in different spaces and aren't comparable.

---

<a id="3-vector-database-comparison"></a>
## 3. Compare Vector Databases: Pinecone, Chroma, FAISS, Milvus, and Weaviate ⭐⭐⭐

| Feature | FAISS | Chroma | Pinecone | Milvus | Weaviate |
|---|---|---|---|---|---|
| **Type** | Library | Embedded DB | Managed cloud | Self-hosted / Zilliz Cloud | Self-hosted / Cloud |
| **Best for** | Prototyping, research | Small-medium apps | Production SaaS | Large-scale enterprise | Enterprise, multimodal |
| **Scale** | 1B+ vectors (single machine) | ~1M vectors | 1B+ vectors | 1B+ vectors | 100M+ vectors |
| **Persistence** | Manual (save/load) | Built-in | Fully managed | Built-in | Built-in |
| **Filtering** | Limited metadata | Basic metadata | Rich metadata filtering | Advanced hybrid search | GraphQL-like filtering |
| **Managed** | No | No | Yes (fully) | Zilliz Cloud option | Weaviate Cloud option |
| **Cost** | Free | Free | $0.08/1M reads | Free (self-hosted) | Free (self-hosted) |
| **Setup** | `pip install faiss-cpu` | `pip install chromadb` | API key | Docker / K8s | Docker / K8s |

### When to Use Each

**FAISS:** You're prototyping, running experiments, or your entire corpus fits in memory on one machine. Zero infrastructure overhead. Google-scale search teams use it as a building block.

**Chroma:** You want a quick embedded database for a small-to-medium application. Great developer experience, runs in-process. Outgrow it when you need multi-node scaling or advanced filtering.

**Pinecone:** You want zero-ops vector search in production. Fully managed, auto-scales, great metadata filtering. The "just works" option. Higher cost but eliminates DevOps burden.

**Milvus:** You need to self-host at enterprise scale. Handles billions of vectors, supports GPU acceleration, hybrid search (vector + keyword). The most feature-rich open-source option. Zilliz Cloud is the managed version.

**Weaviate:** You need multimodal search (text + image) or complex filtering. Good for enterprise knowledge graphs. Native support for multiple vectorizers and object schemas.

### Interview signal

Don't just name a database — explain WHY for your use case:
- "I chose FAISS for our proof-of-concept because our corpus was 50K documents and we needed zero infrastructure overhead to validate the retrieval quality first."
- "We migrated to Pinecone for production because we needed managed scaling, metadata filtering for multi-tenant isolation, and 99.9% uptime without dedicated DevOps."

---

<a id="4-framework-comparison"></a>
## 4. LangChain vs LlamaIndex vs LangGraph vs CrewAI — When to Choose Each? ⭐⭐⭐

### LangChain
**What it is:** A general-purpose framework for building LLM applications with chains, tools, and memory.

**Strengths:** Massive ecosystem (100+ integrations), lots of examples, large community. Good for prototyping diverse LLM applications quickly.

**Weaknesses:** Deep abstraction layers make debugging painful. API changes frequently (breaking changes between versions). Performance overhead from abstractions. "LangChain errors" is a meme in the community for a reason.

**Use when:** You need to prototype quickly with many integrations, or you're building a simple chain (prompt → LLM → parse → output).

### LlamaIndex
**What it is:** A data framework specifically designed for connecting LLMs to data sources, optimized for RAG.

**Strengths:** Best-in-class data ingestion (200+ data connectors). Excellent query pipeline abstraction. Built-in evaluation tools. More focused than LangChain.

**Weaknesses:** Narrower scope — if your app isn't data-retrieval-heavy, you'll outgrow it. Less flexible for non-RAG agent patterns.

**Use when:** Your primary use case is RAG, document Q&A, or knowledge base search. LlamaIndex's query engines and data connectors are purpose-built for this.

### LangGraph
**What it is:** A framework for building stateful, multi-step agent workflows as graphs. Built on top of LangChain.

**Strengths:** First-class support for cycles, branching, human-in-the-loop, and state persistence. The graph abstraction makes complex workflows explicit and debuggable. Native checkpointing for long-running tasks.

**Weaknesses:** Steeper learning curve. Overkill for simple linear pipelines. Inherits some LangChain complexity.

**Use when:** You need multi-agent orchestration, workflows with cycles (retry loops, iterative refinement), human approval steps, or stateful long-running tasks.

### CrewAI
**What it is:** A framework for orchestrating teams of AI agents with defined roles, goals, and collaboration patterns.

**Strengths:** Intuitive "crew" metaphor (agents have roles like "Researcher," "Writer," "Editor"). Easy to set up multi-agent collaboration. Built-in delegation between agents.

**Weaknesses:** Less mature than LangChain/LangGraph. Less control over low-level execution. Can be opaque when debugging inter-agent communication.

**Use when:** You're building a system where multiple specialized agents need to collaborate on a task (research → analyze → write → review), and the role-based abstraction fits your mental model.

### Decision Framework

```
Simple chain (prompt → LLM → output)?
  → Raw SDK calls. No framework needed.

RAG / document Q&A?
  → LlamaIndex (purpose-built for this)

Complex multi-step workflow with cycles?
  → LangGraph (graph-based state management)

Multi-agent collaboration with roles?
  → CrewAI (role-based team metaphor)

Need to prototype quickly with many integrations?
  → LangChain (broadest ecosystem)

Production system with strict requirements?
  → Start with raw SDK, add framework only for specific subsystems that need it
```

---

<a id="5-ai-agents-and-multi-agent-architecture"></a>
## 5. Explain AI Agents, Multi-Agent Systems, and Supervisor Architecture ⭐⭐⭐⭐

### What Is an AI Agent?

An AI agent is an LLM that can reason about tasks, decide to use tools, observe results, and iterate until the task is complete. The key difference from a basic LLM call: agents have **autonomy** — they decide WHAT to do, not just generate text.

```
Basic LLM:   Input → Model → Output (one shot)
Agent:        Input → Think → Act → Observe → Think → Act → ... → Output (iterative)
```

### Single Agent

One LLM with access to multiple tools. It handles the entire task end-to-end.

```python
# Pseudocode
while not done:
    action = llm.decide(user_query, context, available_tools)
    if action.type == "respond":
        return action.response
    result = tools.execute(action.tool, action.args)
    context.append(result)
```

Good for: straightforward tasks, well-defined tool sets, when you need one coherent reasoning chain.

### Multi-Agent System

Multiple specialized agents, each with their own system prompt, tools, and expertise.

**Supervisor pattern:**
```
         ┌──────────────┐
         │  Supervisor    │  Decomposes task, delegates, synthesizes
         └──────┬───────┘
          ┌─────┼──────┐
          ▼     ▼      ▼
       Agent  Agent  Agent
       (research) (code) (review)
```

The supervisor:
1. Receives the user request
2. Breaks it into subtasks
3. Assigns each subtask to a specialist
4. Collects results
5. Synthesizes the final response

**Hierarchical pattern:**
```
         ┌──────────────┐
         │  Orchestrator  │
         └──────┬───────┘
          ┌─────┼──────┐
          ▼     ▼      ▼
       Team    Team   Team
       Lead    Lead   Lead
        │       │      │
       ┌┴┐    ┌┴┐   ┌┴┐
       W  W   W  W   W  W   (Worker agents)
```

For very complex tasks: the orchestrator delegates to team leads, who further delegate to workers.

**Peer-to-peer pattern:**
Agents communicate directly with each other without a central coordinator. Good for collaborative tasks (debate, brainstorming) but harder to control.

### When Multi-Agent > Single Agent

- **Context window pressure:** One agent can't fit all the context it needs. Split into specialists that each focus on a narrower context.
- **Conflicting system prompts:** A "devil's advocate" agent and an "optimist" agent can't share a system prompt.
- **Different tool access:** The research agent needs web search; the code agent needs a code executor. Separating them improves security.
- **Independent scaling:** If research takes 80% of compute, scale research agents without scaling others.

---

<a id="6-function-calling--tool-calling"></a>
## 6. What Is Function Calling? How Is It Implemented? ⭐⭐

Function calling lets the LLM output structured JSON requesting that your code execute a specific function with specific arguments.

### OpenAI Implementation

```python
import openai

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["city"],
            },
        },
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
    tools=tools,
)

# Model returns a tool_call (not text)
tool_call = response.choices[0].message.tool_calls[0]
# tool_call.function.name = "get_weather"
# tool_call.function.arguments = '{"city": "Paris", "unit": "celsius"}'

# YOUR CODE executes the function
result = get_weather(city="Paris", unit="celsius")

# Feed result back
messages.append(response.choices[0].message)  # Assistant's tool call
messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": json.dumps(result)})

# Model generates final response incorporating the tool result
final_response = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools)
```

**The model NEVER executes code.** It outputs a JSON request. Your code executes, validates, and returns the result.

---

<a id="7-mcp-model-context-protocol"></a>
## 7. What Is MCP (Model Context Protocol)? Why Is It Gaining Popularity? ⭐⭐⭐

**MCP** is an open standard (originated from Anthropic) that defines how LLM applications connect to external tools and data sources through a uniform protocol.

**Before MCP:** Every application built custom integrations for every tool. Claude desktop had a Slack plugin, a GitHub plugin, a Google Drive plugin — all built differently.

**With MCP:** Tools are published as MCP servers. Any MCP-compatible client can discover and use them.

```
┌─────────────────┐         ┌──────────────┐
│  Claude Desktop  │ ←─MCP─→ │  GitHub MCP   │
│  (MCP client)    │ ←─MCP─→ │  Slack MCP    │
│                  │ ←─MCP─→ │  Database MCP │
└─────────────────┘         └──────────────┘

┌─────────────────┐         ┌──────────────┐
│  Your Custom App │ ←─MCP─→ │  Same servers!│
│  (MCP client)    │ ←─MCP─→ │  No code change│
└─────────────────┘         └──────────────┘
```

**Why it's gaining popularity:**
1. **Write once, use everywhere.** A GitHub MCP server works in Claude, Cursor, your custom app — no per-client integration.
2. **Security model.** MCP defines how tools declare permissions, how clients authenticate, and how data flows. Standardized security is better than 100 custom implementations.
3. **Discovery.** Clients can query what tools a server offers at runtime. Add a new tool to the server, and all clients see it automatically.
4. **Ecosystem.** Growing library of pre-built MCP servers for popular services. Reduces the "build every integration from scratch" burden.

**MCP server structure (Python with FastMCP):**
```python
from fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
def get_weather(city: str, unit: str = "celsius") -> dict:
    """Get current weather for a city."""
    # Implementation
    return {"temperature": 22, "condition": "sunny", "city": city}

mcp.run()
```

---

<a id="8-llm-latency-optimization"></a>
## 8. How Do You Optimize LLM Latency and Reduce Inference Costs? ⭐⭐⭐⭐

### Latency Optimization

**1. Streaming:** Don't wait for the full response. Stream tokens to the user as they're generated. First-token latency drops from 5s → 200ms.

**2. Model selection routing:** Route simple queries to smaller, faster models:
```
Simple factual query → gpt-4o-mini (200ms) 
Complex reasoning → gpt-4o (2-5s)
Routing classifier → gpt-4o-mini (100ms overhead, saves 2-4s on 60% of requests)
```

**3. Prompt optimization:** Shorter prompts = faster processing. Remove redundant instructions, compress system prompts, use concise few-shot examples.

**4. Caching:** Semantic cache for similar queries (embedding similarity > 0.95). Exact-match cache for repeated queries. Cache hit = 0ms LLM latency.

**5. Parallel tool execution:** When the agent needs multiple independent tools, run them simultaneously:
```python
# Sequential: 200ms + 300ms + 150ms = 650ms
# Parallel: max(200, 300, 150) = 300ms
results = await asyncio.gather(tool1(), tool2(), tool3())
```

**6. KV cache (provider-side):** Modern LLM APIs cache the key-value attention matrices for repeated prompt prefixes. If your system prompt is the same across calls, subsequent calls are faster.

### Cost Reduction

**1. Model tiering:** 80% of requests can be handled by mini models at 1/10th the cost.

**2. Batch API:** For non-real-time work (eval, data enrichment, embeddings), use batch endpoints at 50% cost.

**3. Prompt compression:** Shorter prompts = fewer input tokens = lower cost. Tools like `gzip`-based prompt compression or automatic summarization of long contexts.

**4. Response length limits:** Set `max_tokens` appropriately. Don't pay for 4000-token responses when 200 tokens suffice.

**5. Caching:** A 30% cache hit rate means 30% fewer API calls.

**6. Self-hosted models:** For high-volume, low-complexity tasks, a self-hosted 7B-13B model can be cheaper than API calls at scale (break-even: ~100K requests/day).

---

<a id="9-prompt-engineering-techniques"></a>
## 9. Explain Prompt Engineering Techniques ⭐⭐⭐

### Zero-Shot
No examples. Just the instruction.
```
Classify the sentiment of this review as positive, negative, or neutral:
"The food was amazing but the service was slow."
```
Use when: The task is straightforward and the model understands it inherently.

### Few-Shot
Provide 2-5 examples before the actual query.
```
Review: "Great product!" → Sentiment: positive
Review: "Terrible experience." → Sentiment: negative
Review: "It was okay." → Sentiment: neutral

Review: "The food was amazing but the service was slow." → Sentiment:
```
Use when: The task has nuance that zero-shot misses, or you want to establish an output format.

### Chain-of-Thought (CoT)
Ask the model to reason step-by-step before answering.
```
Q: A store has 15 apples. 3 customers each buy 2 apples, and then 10 more apples are delivered. How many apples are in the store?

Think step by step:
1. Starting apples: 15
2. Apples sold: 3 customers × 2 apples = 6 apples
3. After sales: 15 - 6 = 9 apples
4. After delivery: 9 + 10 = 19 apples

Answer: 19 apples
```
Use when: Multi-step reasoning, math, logic problems, complex analysis. Dramatically improves accuracy on reasoning tasks.

### Self-Consistency
Run the same prompt multiple times (with temperature > 0), then take the majority vote.
```
Run 1: "The answer is 19" ← 
Run 2: "The answer is 19" ←  Majority: 19 ✓
Run 3: "The answer is 17"
Run 4: "The answer is 19" ←
Run 5: "The answer is 19" ←
```
Use when: High-stakes decisions where reliability matters more than cost. Typically 3-5 runs with CoT.

### ReAct (Reasoning + Acting)
The model alternates between thinking and using tools (covered in detail in Week 3).
```
Thought: I need current weather data. I can't generate this from memory.
Action: get_weather(city="NYC")
Observation: {"temp": 72, "condition": "sunny"}
Thought: I have the data. I can answer the user.
Answer: It's 72°F and sunny in NYC.
```
Use when: Tasks requiring external data or tool use.

### Tree of Thoughts (ToT)
Explore multiple reasoning paths simultaneously, evaluate each, and select the best.
```
Path A: Assume the user wants a refund → leads to refund flow → check: does the user mention money? Yes → viable
Path B: Assume the user wants technical help → leads to troubleshooting → check: does the user mention a bug? No → less viable
Path C: Assume the user is just frustrated → leads to empathy → check: does the user express emotion? Somewhat → partially viable

Best path: A (strongest signal alignment)
```
Use when: Complex problems with multiple possible approaches. More expensive (multiple LLM calls) but higher quality for ambiguous tasks.

---

<a id="10-llm-evaluation"></a>
## 10. How Do You Evaluate an LLM Application? ⭐⭐⭐⭐

### Key Metrics

**Faithfulness:** Does the generated answer ONLY use information from the provided context? A response that adds information not in the retrieved documents is unfaithful (even if factually correct).

**Answer Relevancy:** Does the answer actually address the question? A faithful but off-topic response scores low here.

**Context Precision:** Of the retrieved documents, how many were actually relevant to answering the question? High precision = retrieved documents were well-targeted.

**Context Recall:** Of all the documents that COULD have answered the question, how many did the retrieval system find? High recall = didn't miss important sources.

### Evaluation Frameworks

**RAGAS (Retrieval-Augmented Generation Assessment):**
Open-source framework that evaluates RAG pipelines on: faithfulness, answer relevancy, context precision, context recall. Uses LLM-as-judge to compute metrics without manual labeling.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset=eval_dataset,  # questions + answers + contexts + ground_truth
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(result)
# {"faithfulness": 0.87, "answer_relevancy": 0.92, "context_precision": 0.78}
```

**DeepEval:**
Python-first eval framework with built-in metrics for: hallucination, toxicity, bias, answer relevancy, faithfulness. Integrates with pytest for CI/CD.

```python
from deepeval import assert_test
from deepeval.metrics import HallucinationMetric
from deepeval.test_case import LLMTestCase

test_case = LLMTestCase(
    input="What is our refund policy?",
    actual_output="Our refund policy allows returns within 30 days.",
    context=["Customers can return products within 30 days for a full refund."],
)

metric = HallucinationMetric(threshold=0.5)
assert_test(test_case, [metric])
```

### Building a Gold Evaluation Set

1. **Collect real user queries** from production logs (anonymized)
2. **Human-label ground truth answers** for 200-500 queries
3. **Tag each query** with metadata: topic, difficulty, expected tools, expected sources
4. **Include adversarial cases:** prompt injection attempts, off-topic queries, ambiguous questions
5. **Version the eval set** — it evolves as your system evolves
6. **Run evals in CI/CD** — block deploys that drop below thresholds

---

<a id="11-preventing-hallucinations"></a>
## 11. How Do You Prevent Hallucinations in a RAG Application? ⭐⭐⭐⭐

Hallucination in RAG = the model generates information NOT present in the retrieved context. This is different from a general LLM hallucination because you provided the correct context but the model ignored or embellished it.

### Prevention Layers

**1. Better retrieval (prevent the model from having to guess):**
- More relevant chunks → less temptation to fill gaps from parametric memory
- Use reranking to push truly relevant chunks to the top
- Retrieve more chunks (top-20 instead of top-5) then rerank down to top-5

**2. Explicit grounding instructions:**
```
Answer ONLY based on the provided context.
If the context doesn't contain enough information to answer, say:
"I don't have enough information in my sources to answer this question."
Do NOT use your general knowledge.
```

**3. Citation enforcement:**
Force the model to cite specific passages:
```
For every claim in your answer, include a citation in [brackets] referencing 
the source document. Example: "Revenue grew 23% [annual_report_2024, p.42]"
```
This makes hallucinations visible — a fabricated claim won't have a valid citation.

**4. Faithfulness validation (post-generation):**
Use an LLM-as-judge or NLI (natural language inference) model to check: "Is every claim in the response supported by the context?"

```python
def check_faithfulness(response: str, context: str) -> float:
    """Use an LLM to verify each claim is grounded in context."""
    claims = extract_claims(response)  # Break response into individual claims
    
    grounded_count = 0
    for claim in claims:
        is_grounded = judge_model.evaluate(
            f"Is this claim supported by the context?\n"
            f"Claim: {claim}\n"
            f"Context: {context}\n"
            f"Answer: YES or NO"
        )
        if is_grounded == "YES":
            grounded_count += 1
    
    return grounded_count / len(claims)  # Faithfulness score
```

**5. Temperature = 0 for factual tasks:** Lower temperature reduces creative embellishment.

**6. Shorter context windows:** Don't stuff 20 chunks into context. The "lost in the middle" problem means models pay less attention to chunks in the middle of long contexts.

**Measuring success:** "Reduced hallucination rate from 34% to 8% on a 50K-doc corpus" — this is the kind of metric that gets you hired.

---

<a id="12-chunking-strategies"></a>
## 12. Explain Chunking Strategies ⭐⭐⭐

### Fixed-Size Chunking
Split text into equal-sized chunks (e.g., 512 tokens each) with overlap.

```
Chunk 1: tokens 0-512
Chunk 2: tokens 462-974 (50-token overlap)
Chunk 3: tokens 924-1436
```

**Pros:** Simple, predictable chunk sizes. **Cons:** Splits mid-sentence, mid-paragraph, mid-thought. Important context gets severed.

### Recursive Character Splitting
Split by hierarchy: paragraph → sentence → word, respecting natural boundaries.

```python
# LangChain's RecursiveCharacterTextSplitter
separators = ["\n\n", "\n", ". ", " ", ""]
# First tries to split on paragraphs, then sentences, then words
```

**Pros:** Respects document structure. **Cons:** Chunk sizes vary, some chunks may be very short.

### Semantic Chunking
Group sentences by semantic similarity. Adjacent sentences with high similarity stay together; a drop in similarity triggers a chunk boundary.

```
Sentence 1: "Our Q4 revenue was $4.2B."     ─┐
Sentence 2: "This represents 23% YoY growth." ─┤ High similarity → same chunk
Sentence 3: "The board approved a dividend."  ─┘
                                                  ← Similarity drops
Sentence 4: "Our new CEO starts in January." ─┐
Sentence 5: "She previously led Product."    ─┘ Different topic → new chunk
```

**Pros:** Chunks are topically coherent. **Cons:** Computationally expensive (requires embedding every sentence), unpredictable chunk sizes.

### Parent-Child (Hierarchical) Chunking
Store small chunks for retrieval precision but return larger parent chunks for context richness.

```
Parent chunk (1024 tokens): Full section about Q4 financials
├── Child chunk (256 tokens): Revenue paragraph
├── Child chunk (256 tokens): Expenses paragraph
├── Child chunk (256 tokens): Profit margin paragraph
└── Child chunk (256 tokens): Forecast paragraph

Search matches "child chunk: Revenue paragraph"
→ Return "parent chunk: Full section about Q4 financials" to LLM
```

**Pros:** Best of both worlds — precise retrieval + rich context. **Cons:** More complex ingestion pipeline, more storage.

### Hybrid Chunking
Combine strategies. Common pattern:
1. Split by document structure (headers, sections)
2. Within each section, use recursive splitting at ~512 tokens
3. Add metadata (section title, page number, document name) to each chunk

**Why the strategy matters:** Bad chunking is the #1 cause of bad retrieval, which is the #1 cause of bad RAG answers. If a relevant fact is split across two chunks and neither chunk alone contains enough context, retrieval fails silently.

---

<a id="13-mmr-and-rrf"></a>
## 13. What Are MMR and RRF? ⭐⭐⭐

### MMR (Maximum Marginal Relevance)
**Problem:** Vector search returns the top-5 most similar chunks, but they're all from the same paragraph (because similar text has similar embeddings). You get 5 copies of the same information.

**Solution:** MMR balances relevance to the query AND diversity among selected results:

```
Score(chunk) = λ × similarity(chunk, query) - (1-λ) × max_similarity(chunk, already_selected)
```

- `λ = 1.0`: Pure relevance (no diversity — default vector search)
- `λ = 0.5`: Balanced — relevant AND diverse
- `λ = 0.0`: Maximum diversity (ignores relevance — useless)

**Typical setting:** `λ = 0.5 to 0.7`

**Impact:** Instead of 5 chunks saying "Revenue grew 23%," you get: one about revenue, one about expenses, one about the forecast, one about the CEO quote, one about market conditions. Much better context for the LLM.

### RRF (Reciprocal Rank Fusion)
**Problem:** You have multiple retrieval methods (vector search, BM25 keyword search, metadata filter) that each return ranked lists. How do you combine them?

**Solution:** RRF combines rankings using a simple formula:

```
RRF_score(chunk) = Σ 1 / (k + rank_in_system_i)
```

Where `k` is a constant (typically 60) that prevents high-ranked items from dominating.

**Example:**
```
Vector search ranking:  [Doc A: rank 1, Doc B: rank 3, Doc C: rank 2]
Keyword search ranking: [Doc B: rank 1, Doc C: rank 2, Doc A: rank 5]

RRF scores (k=60):
Doc A: 1/(60+1) + 1/(60+5) = 0.0164 + 0.0154 = 0.0318
Doc B: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323 ← Winner!
Doc C: 1/(60+2) + 1/(60+2) = 0.0161 + 0.0161 = 0.0322
```

Doc B wins because it ranked well in BOTH systems. This is hybrid search — combining semantic understanding (vectors) with exact matching (keywords).

**When reranking helps vs adds cost:** Reranking (Cohere Rerank, cross-encoders) adds 100-500ms latency. It's worth it when your retrieval precision is below 80%. If your retrieval is already good (90%+ precision), reranking might only add latency without meaningful quality improvement. Always measure before and after.

---

<a id="14-kv-cache-speculative-decoding-continuous-batching"></a>
## 14. KV Cache, Speculative Decoding, and Continuous Batching ⭐⭐⭐⭐

These are inference optimization techniques used by LLM serving infrastructure.

### KV Cache
**Problem:** Transformer attention computes Key and Value matrices for every previous token at every generation step. For a 4000-token context, generating token #4001 recomputes attention over all 4000 tokens.

**Solution:** Cache the K and V matrices from previous tokens. When generating token #4001, only compute K/V for the new token and reuse the cached matrices.

**Impact:** Without KV cache: O(n²) per token. With KV cache: O(n) per token. Massive speedup for long sequences.

**Trade-off:** KV cache uses GPU memory proportional to (context_length × num_layers × hidden_dim). For long contexts (100K+ tokens), KV cache can consume more memory than the model weights themselves.

**Prompt caching (API-level):** Some providers (Anthropic, OpenAI) cache KV states for repeated prompt prefixes. If your system prompt is identical across calls, subsequent calls skip the prefill computation for that prefix. This reduces both latency and cost.

### Speculative Decoding
**Problem:** Large models (405B parameters) generate tokens slowly — ~15 tokens/second.

**Solution:** Use a small, fast "draft" model to generate a sequence of candidate tokens, then verify them all at once with the large model.

```
Draft model (7B, fast): generates 8 candidate tokens in 1 step
Large model (405B): verifies all 8 in a single forward pass
  → Accepts tokens 1-6 (correct)
  → Rejects tokens 7-8 (regenerates from large model)
  
Net: 6 tokens at the cost of ~2 forward passes instead of 6
```

**Impact:** 2-3x speedup in tokens per second for the large model, with zero quality degradation (the large model verifies every token).

### Continuous Batching
**Problem:** Traditional batching waits until N requests accumulate, processes them together, and returns. Short requests must wait for long requests to finish (head-of-line blocking).

**Solution:** Continuous batching inserts new requests into the batch as soon as a slot opens (when a previous request finishes generating). Requests enter and leave the batch independently.

```
Time →
Traditional batch:  [Req A ████████] [Req B ██] [Req C ████]  (B waits for A)
Continuous batch:   [Req A ████████]
                    [Req B ██]        → B finishes, slot freed
                    [Req C ████]      → C enters freed slot
```

**Impact:** 2-5x higher throughput for the same hardware. Critical for multi-tenant serving (multiple users with different request lengths).

---

<a id="15-enterprise-genai-security"></a>
## 15. How Do You Secure an Enterprise GenAI Application? ⭐⭐⭐⭐

### PII Masking
Detect and mask personally identifiable information BEFORE it reaches the LLM:

```python
# Input: "My SSN is 123-45-6789 and I live at 42 Oak Street"
# After masking: "My SSN is [SSN_REDACTED] and I live at [ADDRESS_REDACTED]"

# Tools: Microsoft Presidio, AWS Comprehend, Google DLP, custom regex
```

The LLM processes masked text. If the response needs to include PII (e.g., confirming an address), re-inject from the original data AFTER generation — never through the model.

### Prompt Injection Defense
Three attack vectors:
- **Direct:** User types malicious instructions ("ignore your system prompt")
- **Indirect:** Malicious content in tool outputs, documents, or web pages
- **Encoded:** Instructions hidden in base64, Unicode, or other encodings

Defenses:
- Input classifier trained on injection patterns (pre-LLM filter)
- System prompt that explicitly warns about injection attempts
- Output validation that checks for anomalous behavior (unexpected tool calls, data exfiltration attempts)
- Sandboxing: agents processing untrusted content have reduced tool access

### Jailbreak Protection
Jailbreaks try to bypass the model's safety training. Defenses:
- Content moderation API on outputs (OpenAI Moderation, Anthropic content filtering)
- Canary tokens in system prompts (if the model repeats them, it's been manipulated)
- Output classifiers trained on known jailbreak outputs
- Rate limiting: rapid successive attempts trigger lockout

### RBAC (Role-Based Access Control)
Different users get different agent capabilities:
```
Anonymous user → can only chat, no tools
Authenticated user → chat + search + basic tools
Admin → chat + search + all tools including write operations
System → full access (for internal automation)
```

### Guardrails
Runtime safety checks independent of the model's training:
- Topic blocking: "Don't discuss competitors' products"
- Output format enforcement: "Always respond with JSON matching this schema"
- Factual grounding: "Only use information from provided documents"
- Brand safety: "Don't make promises about refunds or timelines"
- Response length limits: prevent the model from dumping excessive data

### Content Moderation
- Pre-generation: classify user input for toxicity, hate speech, harmful intent
- Post-generation: classify model output before delivering to user
- Use dedicated classifier models (cheaper and faster than routing through the main LLM)
- Human escalation: flag edge cases for human review

---

<a id="16-llm-fine-tuning"></a>
## 16. LLM Fine-Tuning — LoRA, QLoRA, PEFT, RLHF ⭐⭐⭐⭐

### When to Fine-Tune vs Use RAG

**Use RAG when:**
- You need the model to access specific, changing data
- Your knowledge base updates frequently
- You need citations and traceability
- The model's base capabilities are sufficient, it just needs the right context

**Fine-tune when:**
- You need to change the model's behavior, style, or format consistently
- You need domain-specific terminology or reasoning patterns
- Prompt engineering has hit diminishing returns
- You need to compress knowledge that doesn't fit in prompts (very specialized domains)
- Latency is critical and you can't afford long prompts

### Fine-Tuning Techniques

**Full fine-tuning:** Update ALL model parameters. Requires enormous compute (8+ A100 GPUs for a 7B model). Creates a full copy of the model. Only practical for well-funded teams.

**LoRA (Low-Rank Adaptation):**
Instead of updating all parameters, inject small trainable "adapter" matrices into the model's attention layers. Only 0.1-1% of parameters are trainable.

```
Original weight matrix W: [4096 × 4096] = 16.7M parameters (frozen)
LoRA adapter: A [4096 × 16] × B [16 × 4096] = 131K parameters (trainable)
Effective weight: W + A × B
```

**Impact:** Train a 7B model on a single A100 GPU instead of 8+. Adapter weights are ~10-50 MB (vs ~14 GB for the full model). Multiple LoRA adapters can be swapped at inference time.

**QLoRA (Quantized LoRA):**
Quantize the base model to 4-bit precision, then apply LoRA adapters in higher precision. Reduces memory by 4x.

```
Full fine-tuning 7B: ~120 GB GPU memory
LoRA 7B: ~32 GB GPU memory
QLoRA 7B: ~12 GB GPU memory (fits on a single consumer GPU!)
```

**PEFT (Parameter-Efficient Fine-Tuning):**
Umbrella term for all techniques that fine-tune a small number of parameters: LoRA, prefix tuning, prompt tuning, adapter layers. The Hugging Face `peft` library implements all of these.

**RLHF (Reinforcement Learning from Human Feedback):**
The technique that made ChatGPT work. Three-step process:
1. **Supervised fine-tuning:** Train on human-written examples
2. **Reward model training:** Train a model to predict which response a human would prefer
3. **RL optimization (PPO):** Use the reward model to guide the LLM toward preferred behaviors

Most teams don't do RLHF themselves — it requires specialized infrastructure and large amounts of human preference data. Instead, they use models already RLHF-aligned (GPT-4, Claude) and fine-tune with LoRA/QLoRA for their specific domain.

---

<a id="17-production-llm-monitoring"></a>
## 17. How Would You Monitor an LLM Application in Production? ⭐⭐⭐⭐

### Metrics to Track

**Operational metrics:**
- Request latency (p50, p95, p99)
- Throughput (requests/second)
- Error rate (by type: timeout, rate limit, validation failure, model error)
- Token usage (input + output, per request, per user, per model)
- Cost per request, per user, per day
- Cache hit rate

**Quality metrics:**
- Faithfulness score (sample 5% of responses, run through eval)
- User satisfaction (thumbs up/down, NPS if available)
- Hallucination rate (automated detection on sampled responses)
- Tool selection accuracy (was the right tool called?)
- Task completion rate (for agent systems)
- Refusal rate (how often does the model decline to answer?)

**Safety metrics:**
- Prompt injection attempt rate
- Guardrail trigger rate (by rule)
- PII leak detection rate
- Content moderation flag rate

**Drift metrics:**
- Input distribution change (are users asking different types of questions?)
- Output length distribution change (is the model suddenly verbose?)
- Tool call distribution change (sudden shift in which tools are used)
- Model confidence score distribution

### Monitoring Architecture

```
Application → Structured Logs → Log Aggregator (DataDog/Grafana)
    │                                      │
    ├→ Metrics (Prometheus) → Dashboards   │
    │                                      │
    ├→ Traces (OpenTelemetry) → Trace DB   │
    │                                      │
    └→ Eval Pipeline (async) → Quality DB → Alerts
```

### Alerting Rules

```yaml
alerts:
  - name: high_error_rate
    condition: error_rate > 5% for 5 minutes
    severity: page
    
  - name: latency_degradation
    condition: p99_latency > 10s for 10 minutes
    severity: warning
    
  - name: cost_spike
    condition: hourly_cost > 2x rolling_7d_average
    severity: page
    
  - name: quality_regression
    condition: daily_faithfulness_score < 0.80
    severity: warning
    
  - name: hallucination_spike
    condition: hourly_hallucination_rate > 15%
    severity: page
```

---

<a id="18-zonal-failure-resilience"></a>
## 18. How Do You Design Applications to Survive Zonal Failures? ⭐⭐⭐

### What Is a Zonal Failure?

Cloud providers (AWS, GCP, Azure) organize infrastructure into regions (e.g., us-east-1) and availability zones within regions (e.g., us-east-1a, us-east-1b). A zonal failure means all compute in one zone is unavailable — network outage, power failure, hardware issue.

### Surviving Zonal Failures for AI Applications

**1. Multi-AZ deployment:**
Deploy your application across at least 2 availability zones. Kubernetes does this automatically with pod anti-affinity rules:
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: topology.kubernetes.io/zone
```

**2. Stateless application design:**
Your AI service should be stateless — conversation state in Redis/DynamoDB (replicated across zones), model weights in shared storage (S3/GCS), configuration in environment variables. If a pod dies, another pod in another zone picks up the next request.

**3. Database replication:**
Vector databases and conversation stores must replicate across zones:
- Pinecone: Built-in multi-AZ replication
- Self-hosted Milvus: Configure cross-AZ replicas
- Redis: Sentinel or Cluster mode with replicas in different AZs

**4. Load balancer health checks:**
The load balancer must detect zone failures and stop routing to unhealthy zones within seconds:
```
Zone A: healthy → receives traffic
Zone B: unhealthy → removed from rotation
Zone C: healthy → receives traffic (now handling Zone B's share)
```

**5. Multi-region challenges (advanced):**
For global applications, you need multi-REGION (not just multi-AZ):
- LLM API latency varies by region (API endpoint closest to users)
- Model serving: replicate model weights to each region
- Vector DB: cross-region replication with eventual consistency
- Container images: replicate to per-region container registries to avoid cross-region pull latency
- Data sovereignty: some regions require data to stay within borders

**6. Graceful degradation during failures:**
Design your app so that when one component fails, the whole system doesn't crash:
- LLM API down → fall back to cached responses or simpler model
- Vector DB down → fall back to keyword search
- One agent type unavailable → route to general-purpose agent
- All AI features down → serve static/cached content + "AI features temporarily unavailable"


---

<a id="19-rag-debugging"></a>
## 19. Your RAG System Suddenly Starts Giving Wrong Answers. What's the First Thing You Investigate? ⭐⭐⭐⭐

This is the #1 RAG debugging question in interviews. The answer reveals whether you think in systems or just build demos.

### Debugging Framework: Work Backwards from the Output

The RAG pipeline has 4 failure points. You isolate which one broke by checking them in order:

**Step 1 — Check Retrieval First (most common root cause, ~60% of issues):**

Pull the last 20 failed queries from logs. For each, inspect the retrieved chunks:
- Are the retrieved chunks relevant to the question? **If NO → retrieval is broken.** The LLM is generating from irrelevant context.
- Were the correct chunks even IN the vector database? Maybe a recent ingestion failed silently.
- Did the embedding model change or get updated? Even a minor version bump produces incompatible vectors.

```python
# Quick retrieval audit
for query in failed_queries:
    chunks = retriever.search(query, top_k=10)
    print(f"Query: {query}")
    for i, chunk in enumerate(chunks):
        print(f"  Chunk {i}: score={chunk.score:.3f} | {chunk.text[:100]}...")
    print(f"  Human judgment: relevant? {'yes' if is_relevant(chunks, query) else 'NO ← problem here'}")
```

**Step 2 — Check the Generation (if retrieval looks fine):**

The correct chunks are retrieved, but the LLM still generates a wrong answer. This means:
- The system prompt changed (someone edited it without testing)
- The model was swapped or updated (provider-side model update)
- The context is too long and the answer is "lost in the middle" (LLMs pay less attention to chunks in the middle of long contexts)
- Temperature is too high (creative embellishment overrides grounding)

**Step 3 — Check the Data (if both retrieval and generation look fine on manual test):**

- Was the source data itself wrong or stale? (garbage in, garbage out)
- Did a recent re-ingestion corrupt or duplicate documents?
- Were documents deleted but their vectors are still in the database?

**Step 4 — Check the Infrastructure:**

- Did a deployment change the embedding model endpoint?
- Is the vector database returning stale results (caching issue)?
- Is a load balancer routing some traffic to an old version?

**How to PROVE the root cause:**

Don't just hypothesize. Run a controlled test:
```python
# 1. Take a known-good query that used to work
query = "What is our refund policy?"
expected_source = "policy_handbook_v3.pdf"

# 2. Check retrieval
chunks = retriever.search(query, top_k=5)
assert any(expected_source in c.metadata["source"] for c in chunks), "Retrieval broken"

# 3. Check generation with known-good context
forced_context = load_known_good_chunk(expected_source, section="refund_policy")
response = llm.generate(query, context=forced_context)
assert "30 days" in response, "Generation broken even with correct context"

# If retrieval passes but generation fails → generation problem
# If retrieval fails → retrieval problem
# If both pass → the issue is intermittent or environmental
```

---

<a id="20-retrieval-generation-gap"></a>
## 20. Retriever Returns Relevant Documents, But Answer Quality Is Still Poor. What's Going Wrong? ⭐⭐⭐

This is the **retrieval-generation gap** — the most subtle RAG failure mode. You've confirmed the right documents are retrieved, but the final answer is wrong, incomplete, or vague.

### Root Causes (in order of likelihood)

**1. Lost in the middle:**
LLMs pay the most attention to the beginning and end of the context, and less to the middle. If your answer is in chunk #3 of 5, it gets less attention than chunks #1 and #5.

Fix: Rerank chunks so the most relevant one is FIRST, not buried in the middle. Or restructure context to put the key passage at the top:
```
[MOST RELEVANT PASSAGE — answer this question using this passage first]
{best_chunk}

[SUPPORTING CONTEXT]
{remaining_chunks}
```

**2. Context window noise:**
You retrieved 5 relevant chunks, but 3 of them contain tangentially related information that distracts the model. The LLM tries to synthesize ALL context, including the noise.

Fix: Aggressive reranking + filtering. Only include chunks above a similarity threshold (e.g., score > 0.75). Better: use a cross-encoder reranker that scores (query, chunk) pairs more accurately than embedding similarity.

**3. Contradictory context:**
Multiple chunks contain conflicting information (e.g., old policy vs new policy, different product versions). The LLM gets confused and either picks the wrong one or hedges.

Fix: Add metadata filtering (date, version, source priority). Include recency bias: "If sources conflict, prefer the most recent document."

**4. Chunk boundary problem:**
The answer spans two chunks, and neither chunk alone contains the complete information. The model has partial context and generates a partial answer.

Fix: Overlap your chunks (50-100 token overlap). Use parent-child chunking: retrieve the small child chunk for precision, return the larger parent chunk for completeness.

**5. System prompt competing with context:**
Your system prompt says "Be concise" but the user needs a detailed answer. Or the system prompt says "Only answer from context" but the context is ambiguous, so the model gives an empty response.

Fix: Tune the system prompt for your use case. For factual Q&A: "Answer thoroughly using the provided context." For customer support: "Be concise but complete."

**6. The model just isn't good enough:**
Smaller models (gpt-4o-mini, 7B models) struggle with complex multi-step reasoning over long contexts. The retrieval is fine, but the model can't synthesize the information.

Fix: Upgrade the generation model for complex queries (model routing), or pre-process the context (extract key facts from chunks before passing to the LLM).

---

<a id="21-proving-embedding-improvements"></a>
## 21. How Do You Know If Improving Embeddings Actually Improved the System? ⭐⭐⭐

"We switched from `text-embedding-ada-002` to `text-embedding-3-large` and things feel better" is NOT evidence. Here's how to prove it.

### Before-and-After Evaluation Framework

**Step 1 — Build a golden evaluation set BEFORE changing anything:**

```python
# 50-200 queries with labeled relevant documents
eval_set = [
    {
        "query": "What is the refund policy?",
        "relevant_doc_ids": ["policy_v3_chunk_42", "policy_v3_chunk_43"],
        "expected_answer_contains": ["30 days", "full refund"],
    },
    # ... 50-200 more
]
```

**Step 2 — Measure retrieval quality with the OLD embeddings:**

```python
# Retrieval metrics
for case in eval_set:
    retrieved = retriever.search(case["query"], top_k=5)
    retrieved_ids = [r.id for r in retrieved]
    
    # Precision@K: of top-K retrieved, how many are relevant?
    precision = len(set(retrieved_ids) & set(case["relevant_doc_ids"])) / len(retrieved_ids)
    
    # Recall@K: of all relevant docs, how many were retrieved?
    recall = len(set(retrieved_ids) & set(case["relevant_doc_ids"])) / len(case["relevant_doc_ids"])
    
    # MRR (Mean Reciprocal Rank): how high is the first relevant result?
    for rank, rid in enumerate(retrieved_ids, 1):
        if rid in case["relevant_doc_ids"]:
            mrr = 1 / rank
            break
    else:
        mrr = 0

# Aggregate: Avg Precision@5, Avg Recall@5, MRR
```

**Step 3 — Re-embed with the NEW model, measure again with the SAME eval set and SAME queries.**

**Step 4 — Compare:**

```
                    Old Embedding    New Embedding    Change
Precision@5:        0.72             0.81             +12.5% ✓
Recall@5:           0.65             0.74             +13.8% ✓
MRR:                0.78             0.85             +9.0%  ✓
End-to-end accuracy: 0.71            0.79             +11.3% ✓
Avg latency:        45ms             52ms             +15.5% (acceptable?)
Embedding cost:     $20/day          $130/day         +550%  (acceptable?)
```

**Step 5 — Measure END-TO-END, not just retrieval:**

Better retrieval doesn't always mean better answers. Run the full pipeline (retrieve → generate → validate) on the eval set with both embedding models. The metric that matters is **final answer quality**, not just retrieval precision.

**The principle:** Any change to ANY component of the RAG pipeline (embedding model, chunk size, retrieval top-K, reranker, LLM model, system prompt) should be evaluated with the same before-and-after methodology on the same golden dataset. No exceptions.

---

<a id="22-multi-document-questions"></a>
## 22. A User Asks a Question Requiring Information from 5 Different Documents. How Do You Handle It? ⭐⭐⭐⭐

This is the **multi-hop retrieval** problem — one of the hardest challenges in RAG. Standard vector search finds documents similar to the query, but multi-hop questions need information that's spread across documents that may not individually match the query well.

### Example

User: "Compare our Q4 2024 revenue growth rate with our main competitor's, and explain how our new product launch affected the difference."

This needs: (1) Q4 2024 revenue numbers, (2) competitor's Q4 numbers, (3) product launch details, (4) product revenue attribution, (5) market context.

### Strategies

**Strategy 1 — Query Decomposition:**

Use an LLM to break the complex question into sub-questions, retrieve for each, then synthesize:

```python
async def multi_hop_retrieve(query: str) -> list[Chunk]:
    # Step 1: Decompose
    sub_queries = await llm.generate(
        f"Break this question into 3-5 independent sub-questions:\n{query}\n"
        f"Return as JSON array of strings."
    )
    # ["What was our Q4 2024 revenue?", 
    #  "What was competitor X's Q4 2024 revenue?",
    #  "What new products did we launch in 2024?", ...]
    
    # Step 2: Retrieve for each sub-query
    all_chunks = []
    for sub_q in sub_queries:
        chunks = await retriever.search(sub_q, top_k=3)
        all_chunks.extend(chunks)
    
    # Step 3: Deduplicate and rerank ALL chunks against the ORIGINAL query
    unique_chunks = deduplicate(all_chunks)
    reranked = reranker.rerank(query, unique_chunks, top_k=8)
    
    return reranked
```

**Strategy 2 — Iterative Retrieval (ReAct-style):**

Let the agent retrieve, read, decide what's missing, retrieve more:

```
Turn 1: Retrieve for "Q4 2024 revenue" → found our numbers
Turn 2: Retrieve for "competitor Q4 revenue" → found competitor numbers  
Turn 3: Retrieve for "new product launch 2024 impact" → found launch details
Turn 4: LLM has all 3 pieces → generates comprehensive answer
```

**Strategy 3 — Knowledge Graph Augmented Retrieval:**

Pre-build a knowledge graph that links entities across documents. When the query mentions "revenue" + "competitor" + "product launch," the graph traversal finds all connected documents even if they don't match the query terms.

**Strategy 4 — Hierarchical Retrieval:**

First retrieve at the document level (which documents are relevant?), then within those documents, retrieve specific chunks. This ensures you don't miss documents that are relevant at the topic level but don't match at the chunk level.

### Context Assembly for Multi-Source Answers

Once you have chunks from 5 documents, organize them clearly for the LLM:

```
Answer using ALL the following sources. Cite each source by its label.

[Source A: Q4 2024 Earnings Report, Page 3]
Revenue reached $4.2B in Q4, up 23% YoY...

[Source B: Competitor Analysis Report, Page 7]  
CompetitorX reported $3.8B in Q4, up 15% YoY...

[Source C: Product Launch Retrospective, Page 2]
The new product contributed $450M in its first quarter...

[Source D: Market Analysis, Page 12]
The enterprise SaaS market grew 18% overall in 2024...

[Source E: Board Presentation, Slide 15]
Management attributes 40% of the growth gap to the new product...
```

---

<a id="23-rag-scaling"></a>
## 23. RAG Works with 10K Documents. Now It Has 1M Documents. What Breaks First? ⭐⭐⭐⭐

### What Breaks (in order)

**1. Retrieval precision drops (breaks first):**

With 10K documents, top-5 retrieval was precise because there were fewer similar but irrelevant chunks to confuse the search. At 1M documents, the noise floor rises dramatically. Chunks from different contexts have similar embeddings, pushing truly relevant results below top-5.

Fix: 
- Add metadata filtering (date, department, document type) to narrow the search space BEFORE vector similarity
- Add a reranker (cross-encoder) as a second-pass filter
- Hybrid search (BM25 keyword + vector) catches what pure vector misses
- Namespace your index by category so queries only search relevant subsets

**2. Ingestion pipeline can't keep up:**

At 10K docs, you could re-embed everything in 30 minutes. At 1M docs, full re-embedding takes days and costs thousands.

Fix:
- Incremental indexing: only embed new/changed documents
- Change detection: hash each document, only re-process if hash changed
- Parallel processing: multiprocessing for text extraction, async for embedding API calls

**3. Vector database performance degrades:**

FAISS in-memory with 1M × 1536-dim vectors = ~6 GB RAM. Manageable. But search latency increases from 5ms to 50ms+. At 10M vectors, in-memory is impractical.

Fix:
- Switch from flat index to HNSW or IVF index (approximate nearest neighbor)
- Move from in-memory (FAISS) to managed (Pinecone, Weaviate) with built-in sharding
- Partition the index by metadata (tenant, year, category)

**4. Chunk quality variance:**

At 1M documents, your corpus likely includes: duplicate documents, near-duplicates (same content, different formatting), outdated versions, low-quality sources. These pollute retrieval.

Fix:
- Deduplication at ingestion (content hash + near-duplicate detection via MinHash)
- Source quality scoring (prioritize authoritative documents)
- Version management (only index the latest version of each document)

**5. Evaluation becomes harder:**

Your golden eval set of 100 queries was built for 10K docs. At 1M docs, the relevant documents have changed, new edge cases exist, and the eval set no longer represents your corpus.

Fix:
- Continuously sample real user queries and label them
- Automated quality monitoring (sample 5% of production responses, run eval)
- Per-category eval sets (finance queries, technical queries, etc.)

### Architecture Redesign for 1M+ Scale

```
10K docs:  Flat FAISS index → simple retriever → LLM → done
1M+ docs:  Metadata filter → HNSW index → BM25 hybrid → reranker → LLM → validation
```

---

<a id="24-nlp-to-sql"></a>
## 24. Design an NLP-to-SQL Chatbot for 100+ Table RDBMS ⭐⭐⭐⭐

**The constraint:** You can't dump all 100+ table schemas into the prompt (token limit), can't move data to a vector DB (legacy production system), and must keep token costs low.

### Architecture

```
User Question: "How many orders were placed last month by premium customers?"
          │
          ▼
┌──────────────────────────────────────────────┐
│  Stage 1: Schema Discovery (cheap, fast)      │
│                                               │
│  Pre-built schema catalog:                    │
│  - Table name + description + column summary  │
│  - ~50 tokens per table (vs 500+ for full DDL)│
│  - Relationships (FK mappings)                │
│                                               │
│  LLM selects relevant tables:                 │
│  "For this query, I need: orders, customers,  │
│   customer_tiers, order_items"                │
│                                               │
│  Input: ~5K tokens (100 tables × 50 tokens)   │
│  Output: 4 table names                        │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  Stage 2: Schema Detail Loading               │
│                                               │
│  Load FULL DDL only for selected tables:      │
│  - Column names, types, constraints           │
│  - Foreign key relationships                  │
│  - Sample values (3-5 per column)             │
│  - Indexes (helps generate efficient queries) │
│                                               │
│  Input: ~2K tokens (4 tables × 500 tokens)    │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  Stage 3: SQL Generation                      │
│                                               │
│  System prompt:                               │
│  - Database dialect (PostgreSQL, SQL Server)   │
│  - Schema for 4 selected tables               │
│  - 3-5 few-shot examples of similar queries   │
│  - Rules: "Use JOINs not subqueries for perf" │
│                                               │
│  Output: SQL query                            │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  Stage 4: SQL Validation & Safety             │
│                                               │
│  1. Syntax check (parse the SQL, don't just   │
│     regex it)                                 │
│  2. Safety: block DROP, DELETE, UPDATE, INSERT │
│     — only allow SELECT                       │
│  3. Table check: only access tables the user  │
│     has permission for                        │
│  4. Cost check: EXPLAIN the query, reject if  │
│     estimated scan > threshold                │
│  5. Timeout: kill queries after 30 seconds    │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  Stage 5: Execute & Respond                   │
│                                               │
│  Execute validated SQL on READ REPLICA         │
│  (never hit the production write DB)          │
│                                               │
│  Format results as natural language:          │
│  "Premium customers placed 1,247 orders       │
│   last month, a 15% increase from the         │
│   previous month."                            │
└──────────────────────────────────────────────┘
```

### The Schema Catalog (The Key Innovation)

Instead of putting full DDL for 100 tables in every prompt, pre-build a compressed catalog:

```python
schema_catalog = {
    "orders": {
        "description": "Customer purchase orders with status tracking",
        "key_columns": "order_id, customer_id, order_date, status, total_amount",
        "relationships": "customers.customer_id, order_items.order_id",
        "row_count": "~2.3M",
    },
    "customers": {
        "description": "Customer profiles with tier classification",
        "key_columns": "customer_id, name, email, tier, created_at",
        "relationships": "orders.customer_id, customer_tiers.tier_id",
        "row_count": "~150K",
    },
    # ... 100+ tables, each ~50 tokens
}
```

**Stage 1 costs ~5K tokens.** Stage 2 + 3 costs ~4K tokens. Total: ~9K tokens instead of ~50K+ for all schemas. That's an **80% token reduction**.

### Handling JOINs

Include FK relationships in the schema catalog. When the LLM selects tables, it sees the join paths. Include few-shot examples of multi-table joins for your specific schema:

```sql
-- Example: Orders by premium customers
SELECT COUNT(*) as order_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id  
JOIN customer_tiers ct ON c.tier_id = ct.tier_id
WHERE ct.tier_name = 'Premium'
AND o.order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
AND o.order_date < DATE_TRUNC('month', CURRENT_DATE);
```

### Self-Correction Loop

If the SQL fails (syntax error, timeout, empty result), feed the error back to the LLM:

```python
for attempt in range(3):
    sql = generate_sql(query, schema, examples)
    
    try:
        result = execute_sql(sql, timeout=30)
        if result.rows == 0:
            # Might be correct (genuinely 0 results) or might be wrong query
            feedback = "Query returned 0 rows. Are the WHERE conditions correct?"
            continue
        return format_response(query, sql, result)
    
    except SyntaxError as e:
        feedback = f"SQL syntax error: {e}. Fix and retry."
    except TimeoutError:
        feedback = "Query too slow. Add indexes or simplify."
```

---

<a id="25-scientific-prompt-engineering"></a>
## 25. You Tweaked 3 Things in Your Prompt and It Got Better — But You Don't Know Which Tweak Worked. How Do You Improve Prompts Scientifically? ⭐⭐⭐

### The Problem

Most prompt engineering is: change 3 things → test on 2 examples → "it looks better" → ship. This is guesswork, not engineering.

### The Scientific Approach

**Principle 1 — One variable at a time:**

```
Experiment 1: Baseline prompt → eval score: 72%
Experiment 2: Baseline + added examples → eval score: 79% (+7%)
Experiment 3: Baseline + added "think step by step" → eval score: 75% (+3%)  
Experiment 4: Baseline + changed output format → eval score: 71% (-1%)

Conclusion: Examples helped most, CoT helped some, format change hurt slightly.
Ship: Baseline + examples + CoT = Experiment 5 → eval score: 82%
```

**Principle 2 — Eval set, not vibes:**

Build a golden dataset of 50-200 test cases BEFORE you start tweaking. Run every prompt variant against the SAME eval set. Compare numbers, not feelings.

```python
def eval_prompt(prompt_template: str, eval_set: list[dict]) -> dict:
    scores = []
    for case in eval_set:
        response = llm.generate(prompt_template.format(**case))
        score = evaluate_response(response, case["expected"])
        scores.append(score)
    
    return {
        "accuracy": sum(s["correct"] for s in scores) / len(scores),
        "avg_quality": sum(s["quality"] for s in scores) / len(scores),
        "failure_cases": [s for s in scores if not s["correct"]],
    }
```

**Principle 3 — Version control your prompts:**

```yaml
# prompts/customer_support_v3.yaml
version: 3
author: joshith
date: 2025-06-15
changes: "Added 3 few-shot examples, added CoT instruction"
eval_score: 82%
previous_version: 2
previous_score: 72%

template: |
  You are a customer support agent...
  Think through the problem step by step before answering.
  
  Examples:
  ...
```

**Principle 4 — Statistical significance:**

On a 50-question eval set, a jump from 72% to 76% (2 more correct) could be noise. Use confidence intervals or run each prompt variant 3 times (with temperature > 0) and average.

**Principle 5 — Track regressions, not just improvements:**

When tweak X improves accuracy on billing questions from 70% to 85%, check: did it HURT accuracy on technical questions? Always measure across ALL categories, not just the one you're optimizing.

---

<a id="26-chunk-12-problem"></a>
## 26. The Correct Answer Is in Chunk #12, But Top-K=5 and You Can't Blow the Context Window. How Do You Get It? ⭐⭐⭐⭐

This is the **recall ceiling** problem — your retriever finds some relevant content but misses the key passage.

### Strategies

**1. Query transformation (most effective):**

The user's query might not match the language in chunk #12. Transform the query to increase match probability:

```python
# HyDE (Hypothetical Document Embedding)
# Generate a hypothetical answer, then embed THAT to find similar chunks
hypothetical = llm.generate(f"Write a paragraph that would answer: {query}")
embedding = embed(hypothetical)  # This embedding is closer to the answer's language
chunks = vector_search(embedding, top_k=5)  # Now chunk #12 might appear in top-5
```

```python
# Multi-query: generate 3-5 query variations
variations = llm.generate(
    f"Generate 5 different ways to ask this question:\n{query}\n"
    f"Include different keywords and phrasings."
)
# Search with each variation, merge results
all_chunks = set()
for v in variations:
    all_chunks.update(vector_search(v, top_k=5))
# Rerank all unique chunks against original query
final = reranker.rerank(query, list(all_chunks), top_k=5)
```

**2. Hybrid search (BM25 + vector):**

Vector search finds semantically similar chunks. BM25 keyword search finds exact term matches. The chunk you're missing might have the right keywords but a different semantic framing.

```python
vector_results = vector_search(query, top_k=10)
keyword_results = bm25_search(query, top_k=10)
fused = reciprocal_rank_fusion(vector_results, keyword_results)
final = fused[:5]  # Chunk #12 might now appear via keyword match
```

**3. Two-stage retrieval:**

First retrieve broadly (top-50), then rerank aggressively down to top-5. A cross-encoder reranker is much more accurate than embedding similarity at distinguishing truly relevant chunks.

```python
candidates = vector_search(query, top_k=50)  # Cast a wide net
reranked = cross_encoder_rerank(query, candidates, top_k=5)  # Precise filtering
# Chunk #12 (from the wide net) might score #1 after reranking
```

**4. Metadata-boosted retrieval:**

If you know the topic area, filter before searching:

```python
# User asks about refund policy → filter to policy documents first
chunks = vector_search(
    query, 
    top_k=5, 
    filter={"document_type": "policy", "status": "active"}
)
# Searching 500 policy chunks instead of 100K total chunks → much higher precision
```

**5. Summary index as a routing layer:**

Pre-compute a summary of each document. First search summaries to find the right DOCUMENT, then search within that document's chunks:

```python
# Stage 1: Which document? (search over ~1000 document summaries)
relevant_docs = summary_index.search(query, top_k=3)

# Stage 2: Which chunk within those documents? (search ~50 chunks per doc)
chunks = []
for doc in relevant_docs:
    doc_chunks = chunk_index.search(query, top_k=3, filter={"doc_id": doc.id})
    chunks.extend(doc_chunks)
```

---

<a id="27-embedding-cost-optimization"></a>
## 27. Your Embedding Bill Is $20K/Month Because You Re-Embed the Entire Corpus on Every Chunking Tweak. How Do You Fix It? ⭐⭐⭐

### The Problem

Every time you change chunk size, overlap, or splitting strategy, you re-embed 1M+ documents. At $0.02-0.13 per 1M tokens, this adds up fast when your corpus is billions of tokens.

### Solution Architecture

**1. Content-Addressed Caching:**

Hash each chunk's text content. If the text hasn't changed, reuse the cached embedding:

```python
import hashlib

class EmbeddingCache:
    def __init__(self, cache_store):
        self.store = cache_store  # Redis, SQLite, or filesystem
    
    def get_or_embed(self, text: str, model: str) -> list[float]:
        # Hash the text + model name (different models = different embeddings)
        cache_key = hashlib.sha256(f"{model}:{text}".encode()).hexdigest()
        
        cached = self.store.get(cache_key)
        if cached:
            return cached  # Free! No API call
        
        # Cache miss — embed and store
        embedding = embedding_api.embed(text, model=model)
        self.store.set(cache_key, embedding)
        return embedding
```

When you change chunking strategy, only NEW chunks (that don't exist in the cache) get embedded. If you split a 1000-token chunk into two 500-token chunks, only the new chunks get embedded. If a chunk's text is identical to a previous run, it's a cache hit.

**2. Incremental Re-Embedding:**

Track which documents have changed since the last run:

```python
# Ingestion pipeline
for doc in documents:
    doc_hash = hash_document(doc)
    
    if doc_hash == previous_run_hash(doc.id):
        continue  # Document unchanged, skip entirely
    
    # Only process changed documents
    chunks = chunk_document(doc)
    embeddings = [cache.get_or_embed(c.text, model) for c in chunks]
    vector_db.upsert(chunks, embeddings)
    
    update_hash(doc.id, doc_hash)
```

**3. Versioned Embedding Experiments:**

When testing different chunking strategies, don't re-embed the whole corpus:

```python
# Experiment: test new chunking on a SAMPLE
sample_docs = random.sample(all_docs, 5000)  # 0.5% of corpus

# Embed only the sample with new chunking
new_chunks = new_chunking_strategy(sample_docs)
new_embeddings = embed_batch(new_chunks)

# Evaluate on your golden eval set
score_old = eval_retrieval(old_index, eval_set)
score_new = eval_retrieval(sample_index, eval_set)  # Eval only on sampled docs

# If improvement is significant, THEN re-embed the full corpus
if score_new > score_old + threshold:
    re_embed_full_corpus()  # One-time cost, justified by proven improvement
```

**4. Self-Hosted Embedding Models:**

For $20K/month in API costs, you can rent GPU instances and run open-source embedding models (e.g., `bge-large-en-v1.5`) for ~$2-5K/month. After the initial setup, re-embedding is just compute cost, not per-token API cost.

---

<a id="28-rag-streaming"></a>
## 28. Your RAG Assistant Takes 8-30 Seconds to Respond. Implement Streaming and Progressive Rendering. ⭐⭐⭐

### Why 8-30 Seconds?

Breakdown: Embedding query (100ms) + Vector search (200ms) + Reranking (500ms) + LLM generation (3-20s) + Validation (500ms). The LLM generation dominates, and you can't make it faster — but you can make it FEEL faster.

### Streaming Implementation

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

@app.post("/chat")
async def chat(request: ChatRequest):
    return StreamingResponse(
        generate_stream(request),
        media_type="text/event-stream",
    )

async def generate_stream(request):
    # Phase 1: Immediate acknowledgment (0ms)
    yield f"data: {json.dumps({'type': 'status', 'message': 'Understanding your question...'})}\n\n"
    
    # Phase 2: Retrieval (200-700ms)
    yield f"data: {json.dumps({'type': 'status', 'message': 'Searching knowledge base...'})}\n\n"
    chunks = await retrieve(request.query)
    
    # Phase 3: Show sources immediately (before generation starts)
    sources = [{"title": c.metadata["source"], "page": c.metadata.get("page")} for c in chunks]
    yield f"data: {json.dumps({'type': 'sources', 'data': sources})}\n\n"
    
    # Phase 4: Stream LLM tokens as they generate
    yield f"data: {json.dumps({'type': 'status', 'message': 'Generating answer...'})}\n\n"
    
    async for token in llm.stream_generate(request.query, context=chunks):
        yield f"data: {json.dumps({'type': 'token', 'text': token})}\n\n"
    
    # Phase 5: Done
    yield f"data: {json.dumps({'type': 'done'})}\n\n"
```

### Frontend Progressive Rendering

```
0.0s  → "Understanding your question..."          (user sees immediate response)
0.3s  → "Searching knowledge base..."             (something is happening)
0.8s  → "Found 3 relevant sources: [doc1, doc2, doc3]"  (sources appear)
1.2s  → First tokens start appearing: "Based on..."     (answer begins streaming)
1.2-8s → Tokens stream in real-time                       (user reads as it types)
8.0s  → Complete answer with citations
```

**Perceived wait time:** Under 1 second (first status message appears immediately). Actual wait time: 8 seconds. The user never "waits" because content is always appearing.

### Key Design Patterns

- **Show sources before the answer.** Users can start reading source documents while the answer generates.
- **Stream tokens, don't buffer.** Every token goes to the frontend as it's generated.
- **Show a typing indicator** during the gap between retrieval finishing and first LLM token appearing.
- **Timeout with partial response.** If generation takes > 20s, show what's been generated so far + "Still generating..." indicator.

---

<a id="29-agent-action-safety"></a>
## 29. Your Agent's send_email Tool Accidentally Emails 12,000 Customers. How Do You Design Agent Action Safety? ⭐⭐⭐⭐

### How This Happens

The agent has a `send_email` tool for replying to individual customer inquiries. During debugging, someone sends a test prompt like "send a test email to all active users." The agent interprets this literally, calls `send_email` in a loop for 12,000 customers, and by the time anyone notices, 12,000 emails with "test" content have been sent.

### Defense Architecture: The Action Safety Pyramid

**Layer 1 — Tool Risk Classification:**

```python
TOOL_RISK_LEVELS = {
    "search_knowledge_base": "read",       # No side effects → no restrictions
    "lookup_customer": "read",             # No side effects
    "draft_email": "write_preview",        # Creates draft, doesn't send
    "send_email": "write_destructive",     # Side effect: sends actual email
    "bulk_send_email": "write_critical",   # SHOULD NOT EXIST in agent's toolset
    "delete_account": "write_critical",    # Irreversible
}
```

**Layer 2 — Blast Radius Limits:**

```python
class ActionLimiter:
    """Prevent runaway actions."""
    
    def __init__(self):
        self.limits = {
            "send_email": {"per_minute": 5, "per_session": 10, "per_day": 50},
            "create_ticket": {"per_minute": 10, "per_session": 20, "per_day": 100},
        }
        self.counts = defaultdict(lambda: defaultdict(int))
    
    def check(self, tool: str, session_id: str) -> bool:
        limits = self.limits.get(tool, {})
        counts = self.counts[session_id]
        
        if counts[f"{tool}_minute"] >= limits.get("per_minute", float("inf")):
            raise RateLimitError(f"{tool} rate limit: max {limits['per_minute']}/minute")
        
        if counts[f"{tool}_session"] >= limits.get("per_session", float("inf")):
            raise RateLimitError(f"{tool} session limit reached")
        
        counts[f"{tool}_minute"] += 1
        counts[f"{tool}_session"] += 1
        return True
```

Even if the agent goes rogue, it can only send 5 emails per minute and 10 per session. The 12,000-email blast becomes a 10-email incident.

**Layer 3 — Human-in-the-Loop for Destructive Actions:**

```python
async def execute_tool(tool_call, user_session):
    risk = TOOL_RISK_LEVELS[tool_call.tool_name]
    
    if risk in ("write_destructive", "write_critical"):
        # Don't execute — ask the user to confirm
        confirmation = await request_user_confirmation(
            user_session,
            message=f"I'm about to {tool_call.tool_name} with: {tool_call.arguments}. Proceed?",
            timeout=60,
        )
        if not confirmation:
            return ToolResult(success=False, error="Action cancelled by user")
    
    return await tool_registry.execute(tool_call)
```

**Layer 4 — Sandbox Mode for Development:**

```python
if environment == "development":
    # ALL write tools are intercepted and logged, never executed
    class SandboxToolExecutor:
        def execute(self, tool_call):
            logger.info(f"SANDBOX: Would have executed {tool_call.tool_name}({tool_call.arguments})")
            return ToolResult(success=True, data="[SANDBOX] Action logged, not executed")
```

**Layer 5 — Audit Trail:**

Every tool call is logged with: who triggered it, what arguments, what result, timestamp, session ID, user ID. If something goes wrong, you can reconstruct exactly what happened.

---

<a id="30-ai-gateway"></a>
## 30. Design an AI Gateway That Routes Between GPT, Claude, and Gemini with Auto-Fallback ⭐⭐⭐⭐

### Architecture

```
Incoming Request
       │
       ▼
┌──────────────────────────────────────────┐
│            AI Gateway                     │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Router                            │  │
│  │  - Model preference (if specified) │  │
│  │  - Task-based routing             │  │
│  │  - Cost optimization              │  │
│  │  - Load balancing                 │  │
│  └──────────────┬────────────────────┘  │
│                 │                        │
│  ┌──────────────▼────────────────────┐  │
│  │  Provider Manager                  │  │
│  │                                    │  │
│  │  ┌─────────┐ ┌─────────┐ ┌──────┐│  │
│  │  │ OpenAI  │ │Anthropic│ │Google││  │
│  │  │ Adapter │ │ Adapter │ │Adapt.││  │
│  │  │         │ │         │ │      ││  │
│  │  │ health: │ │ health: │ │health││  │
│  │  │ ✅ 45ms │ │ ✅ 52ms │ │✅60ms││  │
│  │  └─────────┘ └─────────┘ └──────┘│  │
│  └───────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Fallback Chain                    │  │
│  │  Primary → Secondary → Tertiary   │  │
│  │  gpt-4o → claude-sonnet → gemini  │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Response Normalizer               │  │
│  │  Unified response format           │  │
│  │  regardless of which provider      │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### Implementation

```python
from dataclasses import dataclass
from enum import Enum
import asyncio
import time

class Provider(Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    GOOGLE = "google"

@dataclass
class ProviderHealth:
    provider: Provider
    is_healthy: bool
    avg_latency_ms: float
    error_rate: float        # rolling 5-minute window
    last_error: str | None
    last_check: float

class AIGateway:
    def __init__(self):
        self.adapters = {
            Provider.OPENAI: OpenAIAdapter(),
            Provider.ANTHROPIC: AnthropicAdapter(),
            Provider.GOOGLE: GoogleAdapter(),
        }
        self.health: dict[Provider, ProviderHealth] = {}
        self.fallback_order = [Provider.OPENAI, Provider.ANTHROPIC, Provider.GOOGLE]
    
    async def chat(
        self,
        messages: list[dict],
        preferred_provider: Provider | None = None,
        task_type: str = "general",  # "general", "coding", "creative", "analysis"
    ) -> GatewayResponse:
        
        # Determine provider order
        if preferred_provider:
            order = [preferred_provider] + [p for p in self.fallback_order if p != preferred_provider]
        else:
            order = self._route_by_task(task_type)
        
        # Filter to healthy providers
        order = [p for p in order if self._is_healthy(p)]
        
        if not order:
            raise AllProvidersDownError("No healthy LLM providers available")
        
        # Try each provider in order
        last_error = None
        for provider in order:
            try:
                adapter = self.adapters[provider]
                start = time.time()
                
                response = await asyncio.wait_for(
                    adapter.chat(messages),
                    timeout=30.0,
                )
                
                latency_ms = int((time.time() - start) * 1000)
                self._record_success(provider, latency_ms)
                
                return GatewayResponse(
                    content=response.content,
                    provider=provider.value,
                    model=response.model,
                    latency_ms=latency_ms,
                    fallback_used=provider != order[0],
                )
                
            except (TimeoutError, ProviderError, RateLimitError) as e:
                last_error = e
                self._record_failure(provider, str(e))
                continue  # Try next provider
        
        raise AllProvidersFailedError(f"All providers failed. Last error: {last_error}")
    
    def _route_by_task(self, task_type: str) -> list[Provider]:
        """Route to best provider for task type."""
        routing = {
            "coding": [Provider.ANTHROPIC, Provider.OPENAI, Provider.GOOGLE],
            "creative": [Provider.OPENAI, Provider.ANTHROPIC, Provider.GOOGLE],
            "analysis": [Provider.OPENAI, Provider.ANTHROPIC, Provider.GOOGLE],
            "general": self.fallback_order,
        }
        return routing.get(task_type, self.fallback_order)
    
    def _is_healthy(self, provider: Provider) -> bool:
        health = self.health.get(provider)
        if not health:
            return True  # Assume healthy if no data
        return health.is_healthy and health.error_rate < 0.3  # <30% error rate
    
    def _record_success(self, provider: Provider, latency_ms: int):
        # Update rolling health metrics
        ...
    
    def _record_failure(self, provider: Provider, error: str):
        # Update error rate, potentially mark unhealthy
        ...
```

### Response Normalization

Each provider has a different response format. The gateway normalizes to a unified format:

```python
@dataclass
class GatewayResponse:
    content: str
    provider: str
    model: str
    latency_ms: int
    input_tokens: int
    output_tokens: int
    cost_usd: float
    fallback_used: bool
```

Callers don't know or care which provider handled their request. They get the same response format regardless.

---

<a id="31-context-compression"></a>
## 31. By Message 30, Your Chatbot Context Hits 40K Tokens and Costs $0.40 Per Reply. How Do You Compress? ⭐⭐⭐

### Strategies (from least to most aggressive)

**1. Sliding Window (simplest):**

Keep only the last N messages. Discard old messages entirely.

```python
def trim_context(messages: list[dict], max_messages: int = 20) -> list[dict]:
    system = [m for m in messages if m["role"] == "system"]
    history = [m for m in messages if m["role"] != "system"]
    return system + history[-max_messages:]
```

Problem: Loses all information from early messages. If the user referenced something from message 3, it's gone.

**2. Summarization Compression:**

Periodically summarize old messages into a compact summary, keep recent messages verbatim:

```python
def compress_context(messages: list[dict], keep_recent: int = 10) -> list[dict]:
    system = [m for m in messages if m["role"] == "system"]
    history = [m for m in messages if m["role"] != "system"]
    
    if len(history) <= keep_recent:
        return messages  # No compression needed
    
    old_messages = history[:-keep_recent]
    recent_messages = history[-keep_recent:]
    
    # Summarize old messages into a concise paragraph
    summary = llm.generate(
        f"Summarize this conversation in 200 words, preserving key facts, "
        f"decisions, and any specific data mentioned:\n"
        f"{format_messages(old_messages)}"
    )
    
    summary_message = {
        "role": "system",
        "content": f"Previous conversation summary:\n{summary}"
    }
    
    return system + [summary_message] + recent_messages
    # 40K tokens → ~2K (summary) + ~8K (recent 10 messages) = ~10K tokens
```

**3. Key-Value Extraction:**

Instead of summarizing prose, extract structured key-value pairs from the conversation:

```python
# Extract: {"user_name": "Alice", "issue": "billing error", "account_id": "A-12345",
#            "amount_disputed": "$49.99", "resolution_offered": "credit applied"}
# 
# This compresses 30 messages into ~100 tokens of structured context
```

**4. Tiered Memory:**

```
Tier 1 (always in context): System prompt + current user message
Tier 2 (summary): Compressed summary of conversation so far
Tier 3 (retrievable): Full conversation stored externally, retrieved on demand

If the user says "what did we discuss earlier about X":
  → Search Tier 3 for "X"
  → Inject relevant old messages back into context
```

**Cost impact:**
```
Before: 40K tokens × $0.01/1K = $0.40 per reply
After:  10K tokens × $0.01/1K = $0.10 per reply (75% reduction)
At 100K requests/day: $40K → $10K/month saved
```

---

<a id="32-safe-model-migration"></a>
## 32. You Migrate from GPT-4-Turbo to a Newer Model and 15% of Prompts Regress. How Do You Migrate Safely? ⭐⭐⭐⭐

### Why Prompts Regress

Different models have different strengths, weaknesses, and instruction-following behavior. A prompt optimized for GPT-4-Turbo may not work the same on GPT-4o because:
- Different sensitivity to system prompt formatting
- Different default verbosity
- Different handling of edge cases in tool calling
- Slightly different reasoning patterns

### Safe Migration Framework

**Phase 1 — Shadow Mode (zero risk):**

```python
async def chat(request):
    # Primary: old model (serves the response)
    response = await call_model(request, model="gpt-4-turbo")
    
    # Shadow: new model (runs in background, result is logged but NOT returned)
    asyncio.create_task(shadow_call(request, model="gpt-4o"))
    
    return response

async def shadow_call(request, model):
    shadow_response = await call_model(request, model=model)
    log_comparison(request, primary_response, shadow_response)
```

Run for 1-2 weeks. Collect thousands of side-by-side comparisons without any user impact.

**Phase 2 — Evaluate Regressions:**

```python
# Analyze shadow results
for comparison in shadow_results:
    # Automated eval
    old_score = evaluate(comparison.primary_response)
    new_score = evaluate(comparison.shadow_response)
    
    if new_score < old_score - threshold:
        regressions.append(comparison)

# Categorize regressions
# "15% regress" → but WHICH 15%?
# - 8% are format regressions (fixable with prompt changes)
# - 4% are reasoning regressions on edge cases
# - 3% are tool calling differences
```

**Phase 3 — Fix Regressions:**

For each regression category, update the prompt to work with the new model:
- Format regressions → adjust output format instructions
- Reasoning regressions → add more explicit CoT or examples
- Tool calling differences → update tool descriptions

Test fixes against the golden eval set AND the regression examples.

**Phase 4 — Canary Deployment:**

```python
def route_request(request):
    # 5% of traffic → new model
    if hash(request.user_id) % 100 < 5:
        return "gpt-4o"
    return "gpt-4-turbo"
```

Monitor canary traffic for 48 hours. Compare metrics (accuracy, latency, cost, user feedback) between canary and stable.

**Phase 5 — Gradual Rollout:**

5% → 25% → 50% → 100%, with automated rollback if quality metrics drop below threshold at any stage.

**Phase 6 — Keep the Old Model as Fallback:**

Even after 100% migration, keep the old model configured as a fallback. If the new model has an outage or quality regression, you can switch back in seconds.

---

<a id="33-catastrophic-forgetting"></a>
## 33. You Fine-Tuned a Model and It Performs Worse on General Tasks. What Happened? (Catastrophic Forgetting) ⭐⭐⭐

### What Is Catastrophic Forgetting?

When you fine-tune a model on task-specific data, it updates its weights to optimize for that task. But those weight updates can overwrite the general knowledge stored in the same weights. The model gets better at your task but "forgets" how to do general tasks.

```
Before fine-tuning:  General tasks: 90%  |  Your task: 60%
After fine-tuning:   General tasks: 65%  |  Your task: 92%  ← catastrophic forgetting
```

### Why It Happens

- **Training data too narrow.** If you fine-tune on 1000 customer support examples, the model over-adapts to that distribution and loses its ability to handle anything else.
- **Too many epochs.** Training for too long on a small dataset causes overfitting. The model memorizes your training examples rather than learning general patterns.
- **Learning rate too high.** Large weight updates overwrite general knowledge aggressively.

### Prevention

**1. LoRA instead of full fine-tuning:**

LoRA freezes the base model weights and only trains small adapter matrices. The base model's general knowledge is PRESERVED because its weights don't change. The adapter learns task-specific behavior on top.

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=16,                    # Rank (lower = fewer parameters, less forgetting risk)
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],  # Only adapt attention layers
    lora_dropout=0.1,
)
model = get_peft_model(base_model, config)
# Base model weights: FROZEN
# LoRA adapter weights: TRAINABLE (0.1% of total parameters)
```

**2. Mixed training data:**

Include 10-20% general-purpose data alongside your task-specific data. This reminds the model of general skills during fine-tuning:

```python
training_data = (
    task_specific_examples     # 80% — your domain data
    + general_instruction_data  # 20% — diverse general tasks
)
```

**3. Lower learning rate + fewer epochs:**

```python
training_args = TrainingArguments(
    learning_rate=2e-5,   # Conservative (not 5e-4)
    num_train_epochs=3,   # Not 10+
    warmup_ratio=0.1,
)
```

**4. Evaluate on BOTH tasks after each epoch:**

```python
for epoch in range(num_epochs):
    train(model, task_data)
    
    task_score = evaluate(model, task_eval_set)       # Should go up
    general_score = evaluate(model, general_eval_set)  # Should NOT go down significantly
    
    if general_score < baseline_general * 0.9:
        print(f"WARNING: General capability dropped >10% at epoch {epoch}")
        break  # Stop before forgetting gets worse
```

---

<a id="34-hybrid-search"></a>
## 34. Hybrid Search: BM25 + Vector vs Pure Vector — When Does Keyword Search Still Win? ⭐⭐⭐

### When BM25 Keyword Search Beats Vector Search

**1. Exact term matching:**
User searches for error code `ERR_SSL_PROTOCOL_ERROR` or product SKU `ABC-12345-XL`. Vector embeddings map these to generic semantic space ("error" or "product") and lose the exact string. BM25 finds the exact token match.

**2. Rare or domain-specific terms:**
Embedding models are trained on general text. If your domain uses specialized terminology (`QLoRA`, `HNSW`, `pgvector`), the embedding might not have strong representations for these terms. BM25 matches them by exact occurrence.

**3. Short, keyword-like queries:**
"python asyncio timeout" — this is essentially a keyword search. Vector search tries to find semantically similar text, but the user wants documents containing these exact terms.

**4. Boolean/filter-like queries:**
"orders status=cancelled 2024" — the user is expressing a filter, not a semantic concept.

### When Vector Search Beats BM25

**1. Natural language questions:**
"How do I fix a broken deployment?" — there are many ways to express this without using the word "fix" or "broken." Vector search finds semantically similar content regardless of exact words.

**2. Synonym handling:**
"automobile insurance" matches "car coverage" in vector space but not in keyword space.

**3. Cross-lingual retrieval:**
Multilingual embeddings can match "how to cook rice" with a Spanish document about cooking rice. BM25 can't.

### Hybrid is Almost Always Better

```python
from rank_bm25 import BM25Okapi

def hybrid_search(query: str, top_k: int = 5) -> list[Chunk]:
    # Vector search: semantic understanding
    vector_results = vector_db.search(embed(query), top_k=20)
    
    # BM25 search: exact keyword matching
    bm25_results = bm25_index.get_top_n(query.split(), corpus, n=20)
    
    # Fuse with RRF
    fused = reciprocal_rank_fusion(vector_results, bm25_results, k=60)
    
    return fused[:top_k]
```

**Typical improvement:** Hybrid search improves retrieval precision by 5-15% over pure vector search, with the biggest gains on queries containing specific terms, numbers, or codes.

---

<a id="35-finetune-reproducibility"></a>
## 35. How Do You Version and Reproduce a Fine-Tuning Run 6 Months Later? ⭐⭐⭐

### What You Must Track

```yaml
# training_run_v2.3.yaml
run_id: "ft-2025-06-15-001"
date: "2025-06-15"
author: "joshith"

# Data versioning
training_data:
  source: "s3://ml-data/customer_support/v2.3/"
  hash: "sha256:abc123..."
  num_examples: 15000
  split: {train: 0.85, val: 0.10, test: 0.05}

# Base model
base_model:
  name: "meta-llama/Llama-3.1-8B-Instruct"
  source: "huggingface"
  hash: "sha256:def456..."

# Training config (EXACT hyperparameters)
training:
  method: "qlora"
  lora_r: 16
  lora_alpha: 32
  learning_rate: 2e-5
  epochs: 3
  batch_size: 4
  gradient_accumulation_steps: 8
  warmup_ratio: 0.1
  max_seq_length: 2048
  seed: 42

# Environment
environment:
  gpu: "A100-80GB"
  cuda: "12.1"
  torch: "2.1.0"
  transformers: "4.38.0"
  peft: "0.7.1"
  python: "3.11.5"
  requirements_hash: "sha256:ghi789..."

# Results
eval_results:
  task_accuracy: 0.92
  general_accuracy: 0.87
  training_loss_final: 0.23
  
# Artifacts
artifacts:
  adapter_weights: "s3://ml-models/ft-2025-06-15-001/adapter/"
  merged_model: "s3://ml-models/ft-2025-06-15-001/merged/"
  training_logs: "s3://ml-logs/ft-2025-06-15-001/"
  eval_results: "s3://ml-evals/ft-2025-06-15-001/"
```

### Tools

- **MLflow:** Tracks experiments, hyperparameters, metrics, and artifacts. The most common choice.
- **Weights & Biases:** Experiment tracking with visualization. Good for team collaboration.
- **DVC (Data Version Control):** Git for data. Versions datasets alongside code.
- **Docker:** Pin the exact training environment. `docker build` + `docker push` → reproducible container.

### The Reproducibility Checklist

```
□ Training data is versioned and immutable (content-addressed storage)
□ Base model is pinned to exact version/hash (not "latest")
□ All hyperparameters are in a config file (not hardcoded in a notebook)
□ Random seed is fixed
□ Python environment is pinned (requirements.txt with exact versions)
□ GPU type and CUDA version are documented
□ Training script is in version control (git commit hash logged)
□ Results are logged automatically (not copy-pasted from terminal)
□ Adapter weights are saved to persistent storage with run ID
```

If any of these is missing, you can't reproduce the run. Period.

---

<a id="36-pii-in-training-data"></a>
## 36. Your Training Data Pipeline Accidentally Fed PII to the Model. It's Already Trained. Now What? ⭐⭐⭐⭐

### Immediate Actions (first 24 hours)

**1. Stop using the model.** Don't serve it to users until you assess the risk. Switch to the previous model version or a base model.

**2. Assess the scope:**
- What PII was included? (names, emails, SSNs, medical records?)
- How much? (10 records or 10,000?)
- How identifiable? (full names + SSNs = critical; first names only = lower risk)
- Which training split? (If only in validation set, model never learned from it)

**3. Legal/compliance notification:**
Depending on jurisdiction and data type, you may have mandatory breach notification requirements (GDPR: 72 hours, HIPAA, state privacy laws). Involve your legal and compliance team immediately.

### Technical Remediation

**4. Can the model memorize and regurgitate the PII?**

Test with extraction attacks:
```python
# Prompt the model with partial PII and see if it completes
prompts = [
    "John Smith's social security number is",
    "The email for customer account #12345 is",
    "Patient records show that",
]
for p in prompts:
    response = model.generate(p, temperature=0, max_tokens=50)
    if contains_pii(response):
        print(f"PII LEAKAGE DETECTED: {p} → {response}")
```

Large models are less likely to memorize individual training examples than small models. But the risk is non-zero, especially if the PII appeared multiple times in training data.

**5. Remediation options (from least to most disruptive):**

**Option A — Output filtering (fastest, weakest):**
Add a PII detection layer on model outputs. If the model generates PII, redact it before returning to the user. This doesn't remove PII from the model — it just prevents output.

**Option B — Retrain without PII (cleanest):**
Fix the data pipeline, remove PII from training data, retrain from the base model. This is the correct long-term fix but takes time and compute.

**Option C — "Machine unlearning" (research-grade):**
Techniques to selectively remove specific knowledge from a trained model without full retraining. Still largely experimental — not production-ready for most teams.

**Option D — Fine-tune AWAY from PII (practical middle ground):**
Fine-tune the contaminated model on examples that teach it to refuse PII-related queries:
```json
{"prompt": "What is John Smith's SSN?", "response": "I cannot share personal information."}
{"prompt": "Tell me the email for account 12345", "response": "I don't have access to personal account data."}
```
This doesn't remove the knowledge but suppresses it behind safety behavior.

### Prevention (so it never happens again)

```python
# Add to your data pipeline BEFORE training
class PIIDetector:
    def scan(self, text: str) -> list[PIIMatch]:
        # Use Microsoft Presidio, AWS Comprehend, or regex patterns
        ...
    
    def scrub(self, text: str) -> str:
        for match in self.scan(text):
            text = text.replace(match.text, f"[{match.type}_REDACTED]")
        return text

# Every training example passes through PII scrubbing
clean_data = [pii_detector.scrub(example) for example in raw_training_data]
assert all(len(pii_detector.scan(d)) == 0 for d in clean_data), "PII still present!"
```

---

<a id="37-enterprise-pdf-chatbot"></a>
## 37. Design an Enterprise Chatbot for Complex PDFs (Tables, Images, Scanned Docs, Charts) ⭐⭐⭐⭐

### The Challenge

Enterprise PDFs aren't clean text. They contain: multi-column layouts, tables with merged cells, scanned pages (image-only, no extractable text), charts and graphs, headers/footers/page numbers, watermarks, and embedded forms.

### Architecture

```
┌──────────────────── Document Ingestion Pipeline ─────────────────────┐
│                                                                       │
│  PDF Input                                                           │
│       │                                                               │
│       ▼                                                               │
│  [PDF Classifier]                                                     │
│  ├── Digital PDF (has text layer) → Fast text extraction              │
│  ├── Scanned PDF (image-only) → OCR pipeline                         │
│  └── Mixed PDF (some pages scanned) → Hybrid processing              │
│                                                                       │
│  ┌─────────────────────────────────────────────┐                     │
│  │  Text Extraction                             │                     │
│  │  - PyMuPDF for digital text                  │                     │
│  │  - Tesseract/EasyOCR for scanned pages       │                     │
│  │  - Layout-aware: preserve reading order       │                     │
│  └─────────────────────────────────────────────┘                     │
│                                                                       │
│  ┌─────────────────────────────────────────────┐                     │
│  │  Table Extraction                            │                     │
│  │  - Camelot/tabula for simple tables          │                     │
│  │  - Table Transformer (DETR) for complex ones │                     │
│  │  - Convert to Markdown or structured JSON     │                     │
│  │  - Preserve row/column relationships          │                     │
│  └─────────────────────────────────────────────┘                     │
│                                                                       │
│  ┌─────────────────────────────────────────────┐                     │
│  │  Chart/Image Processing                      │                     │
│  │  - Chart detection (YOLOv8 or similar)        │                     │
│  │  - Chart → data extraction (DePlot, ChartOCR)│                     │
│  │  - Image description via multimodal LLM      │                     │
│  │  - Store both raw image + extracted data      │                     │
│  └─────────────────────────────────────────────┘                     │
│                                                                       │
│  ┌─────────────────────────────────────────────┐                     │
│  │  Metadata Extraction                         │                     │
│  │  - Document title, author, date               │                     │
│  │  - Section headers and hierarchy              │                     │
│  │  - Page numbers                               │                     │
│  │  - Confidentiality classification             │                     │
│  └─────────────────────────────────────────────┘                     │
│                                                                       │
│       ▼                                                               │
│  [Chunking Engine]                                                    │
│  - Section-aware: chunks respect section boundaries                  │
│  - Tables: each table is ONE chunk (not split mid-row)               │
│  - Charts: extracted data + description = ONE chunk                  │
│  - Rich metadata per chunk: page, section, type (text/table/chart)   │
│                                                                       │
│       ▼                                                               │
│  [Vector Database]                                                    │
│  - Text chunks + table chunks + chart data chunks                    │
│  - Metadata filters: document, page, content_type, date              │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────── Query Pipeline ──────────────────────────────────┐
│                                                                       │
│  User: "What was the revenue trend shown in the Q4 report chart?"    │
│       │                                                               │
│       ▼                                                               │
│  [Query Classifier]                                                   │
│  - Is this about text, a table, or a chart?                          │
│  - This mentions "chart" → boost chart-type chunks in retrieval      │
│       │                                                               │
│       ▼                                                               │
│  [Hybrid Retrieval]                                                   │
│  - Vector search + metadata filter (content_type="chart")            │
│  - Returns the chart's extracted data + surrounding text context     │
│       │                                                               │
│       ▼                                                               │
│  [Multimodal Context Assembly]                                        │
│  - If chart image is available → include in multimodal prompt        │
│  - Table data formatted as Markdown for LLM readability              │
│  - Text chunks provide surrounding narrative                         │
│       │                                                               │
│       ▼                                                               │
│  [LLM Generation] → answer with citations                           │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

**Table handling:** Tables are first-class citizens. Never chunk through the middle of a table. Extract the table as structured data (Markdown or JSON), and store it as a single chunk with metadata describing what the table contains. When retrieved, present it to the LLM in a format it can reason over.

**Scanned document OCR:** OCR quality varies dramatically. Pre-process scanned pages: deskew, denoise, binarize. Use Tesseract with `--oem 3` (LSTM engine) for best accuracy. For critical documents, consider a commercial OCR API (AWS Textract, Google Document AI, Azure Form Recognizer) — they handle complex layouts better.

**Charts:** Extract the underlying data from charts whenever possible (DePlot, ChartOCR). Store both the extracted data AND a description. For retrieval, match on the data description. For generation, provide the extracted data so the LLM can do calculations.

**Security for enterprise:**
- RBAC: users only see documents they have access to (metadata filter on user permissions)
- PII masking on outputs
- Audit logging of every query + retrieved documents
- No data leaves the customer's network (on-prem deployment or private cloud)

---

<a id="38-fastapi-async-llm"></a>
## 38. Build a FastAPI Endpoint to Invoke an LLM Asynchronously. Explain GET vs POST. ⭐⭐

### GET vs POST

**GET:** Retrieve data. No request body. Parameters in URL query string. Idempotent (same request = same result). Cacheable.
```
GET /models                    → List available models
GET /chat/history?user_id=123  → Fetch chat history
GET /health                    → Service health check
```

**POST:** Send data, create resources, trigger actions. Has a request body. Not idempotent. Not cacheable.
```
POST /chat           → Send messages, get LLM response (sends message body)
POST /embed          → Send text, get embedding (sends text body)
POST /chat/feedback  → Submit user feedback (creates a record)
```

**Why LLM calls are POST:** You're sending message data (request body), creating a new completion (resource creation), and the response may differ each time (non-idempotent due to temperature).

### FastAPI Async LLM Endpoint

```python
import os
import httpx
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from contextlib import asynccontextmanager

# Shared async client — created once, reused across requests
class AppState:
    client: httpx.AsyncClient | None = None

state = AppState()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: create shared HTTP client with connection pooling
    state.client = httpx.AsyncClient(
        base_url="https://api.openai.com/v1",
        headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}"},
        timeout=60.0,
        limits=httpx.Limits(max_connections=20, max_keepalive_connections=10),
    )
    yield
    # Shutdown: clean up
    await state.client.aclose()

app = FastAPI(lifespan=lifespan)

class ChatRequest(BaseModel):
    messages: list[dict[str, str]]
    model: str = Field(default="gpt-4o-mini")
    temperature: float = Field(default=0.7, ge=0, le=2)
    max_tokens: int = Field(default=1024, ge=1, le=128000)
    stream: bool = Field(default=False)

class ChatResponse(BaseModel):
    content: str
    model: str
    input_tokens: int
    output_tokens: int

# Non-streaming endpoint
@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """
    Async LLM chat endpoint.
    
    The 'await' keyword yields control to the event loop while waiting
    for the LLM API response. Other requests are processed during the wait.
    One FastAPI worker handles hundreds of concurrent LLM calls efficiently.
    """
    try:
        response = await state.client.post(
            "/chat/completions",
            json={
                "model": request.model,
                "messages": request.messages,
                "temperature": request.temperature,
                "max_tokens": request.max_tokens,
            },
        )
        response.raise_for_status()
        data = response.json()
        
        return ChatResponse(
            content=data["choices"][0]["message"]["content"],
            model=data["model"],
            input_tokens=data["usage"]["prompt_tokens"],
            output_tokens=data["usage"]["completion_tokens"],
        )
    
    except httpx.TimeoutException:
        raise HTTPException(status_code=504, detail="LLM request timed out")
    except httpx.HTTPStatusError as e:
        raise HTTPException(status_code=e.response.status_code, detail=str(e))

# Streaming endpoint
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """Stream LLM tokens via Server-Sent Events."""
    
    async def generate():
        async with state.client.stream(
            "POST",
            "/chat/completions",
            json={
                "model": request.model,
                "messages": request.messages,
                "temperature": request.temperature,
                "max_tokens": request.max_tokens,
                "stream": True,
            },
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: ") and line != "data: [DONE]":
                    yield f"{line}\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")

# Health check (GET — no body, idempotent, cacheable)
@app.get("/health")
async def health():
    return {"status": "healthy", "model_available": state.client is not None}
```

---

<a id="39-five-cost-levers"></a>
## 39. Your Inference Cost Is Exploding. Give Me 5 Levers to Pull. ⭐⭐⭐⭐

### Lever 1 — Semantic Caching (30-50% savings)

Cache responses for semantically similar queries. Most AI applications have high query repetition — the same question asked slightly differently.

```python
from functools import lru_cache
import numpy as np

class SemanticCache:
    def __init__(self, similarity_threshold: float = 0.95):
        self.threshold = similarity_threshold
        self.cache: list[tuple[np.ndarray, str, str]] = []  # (embedding, query, response)
    
    def get(self, query: str) -> str | None:
        query_embedding = embed(query)
        for cached_embedding, cached_query, cached_response in self.cache:
            similarity = cosine_similarity(query_embedding, cached_embedding)
            if similarity > self.threshold:
                return cached_response  # Cache hit! Zero LLM cost.
        return None
    
    def set(self, query: str, response: str):
        self.cache.append((embed(query), query, response))
```

### Lever 2 — Model Routing (20-30% savings)

Don't use GPT-4o for "What time is it in London?" Route simple queries to cheaper models:

```python
def route(query: str) -> str:
    # Classify query complexity (use a tiny classifier, not an LLM)
    complexity = classify_complexity(query)  # "simple", "medium", "complex"
    
    routing = {
        "simple": "gpt-4o-mini",    # $0.15/1M tokens
        "medium": "gpt-4o-mini",    # $0.15/1M tokens  
        "complex": "gpt-4o",        # $2.50/1M tokens
    }
    return routing[complexity]
# If 60% of queries are simple: 60% at 1/16th the cost = ~56% savings on those queries
```

### Lever 3 — Prompt Compression (10-20% savings)

Shorter prompts = fewer input tokens = lower cost. Every token in your system prompt is charged on EVERY request.

```python
# Before: 2000-token system prompt
"You are a helpful customer service agent for Acme Corporation. Your job is to..."
# (verbose instructions, redundant examples, unnecessary formatting)

# After: 800-token system prompt (same behavior, 60% fewer tokens)
"You are Acme's support agent. Rules: 1) Use context only 2) Cite sources 3) ..."
```

At 100K requests/day, cutting 1200 tokens from the system prompt saves: 1200 × 100K = 120M tokens/day = $300-3000/day depending on model.

### Lever 4 — Batch API for Non-Real-Time Work (50% savings)

OpenAI and Anthropic offer batch endpoints at 50% cost for work that doesn't need instant responses:

```python
# Real-time: user-facing chat → standard API ($2.50/1M input tokens for gpt-4o)
# Batch: nightly eval, document processing, data enrichment → batch API ($1.25/1M)

# Queue offline work
batch_requests = [{"id": f"req_{i}", "messages": msgs} for i, msgs in enumerate(offline_tasks)]
batch_id = client.batches.create(requests=batch_requests)
# Results available in hours, not seconds — but 50% cheaper
```

### Lever 5 — Context Window Management (15-25% savings)

As conversations grow, token count grows. Compress old context:

```python
# Message 1-10: 3K tokens
# Message 11-20: 8K tokens  
# Message 21-30: 15K tokens ← growing linearly
# Message 30+: 25K+ tokens ← expensive

# With compression:
# Always: system prompt (500 tokens) + summary of old messages (500 tokens) + last 5 messages (~2K tokens)
# Total: ~3K tokens regardless of conversation length
```

### Combined Impact

```
Lever                    Savings    Effort
Semantic caching         30-50%     Medium (build cache infrastructure)
Model routing            20-30%     Low (add classifier + routing logic)
Prompt compression       10-20%     Low (edit prompts)
Batch API               50%*        Low (queue offline work) *only on batch-eligible work
Context compression      15-25%     Medium (build summarization pipeline)

Combined realistic savings: 50-70% of original cost
```
