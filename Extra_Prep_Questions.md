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
