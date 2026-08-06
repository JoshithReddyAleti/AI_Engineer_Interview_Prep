# 💻 Week 6 — Technical / Coding Questions

> **Focus:** LangChain LCEL, LangGraph state machines, CrewAI crews, MCP servers, LoRA config, DPO training, quantization
>
> **How to use:** These test whether you've written the code, not just read about it. Build before reading.

---

## Q1. Build a LangChain LCEL Chain with Streaming, Retry, and Fallback ⭐⭐⭐

**Prompt:** Build a production-ready LCEL chain that streams responses, retries on failure, and falls back to a cheaper model if the primary times out.

**Solution:**

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda

# Primary and fallback models
primary = ChatOpenAI(model="gpt-4o", temperature=0, timeout=10, max_retries=2)
fallback = ChatOpenAI(model="gpt-4o-mini", temperature=0, timeout=5, max_retries=1)
cheap_fallback = ChatAnthropic(model="claude-3-5-haiku-20241022", timeout=3)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a customer support agent. Answer based on the context provided."),
    ("human", "Context: {context}\n\nQuestion: {question}")
])

# Fallback chain: primary → fallback → cheap_fallback
llm_with_fallbacks = primary.with_fallbacks(
    [fallback, cheap_fallback],
    exceptions_to_handle=(Exception,)  # Fall back on any exception
)

# Full LCEL chain
chain = prompt | llm_with_fallbacks | StrOutputParser()

# Usage with streaming
async def stream_response(context: str, question: str):
    async for chunk in chain.astream({"context": context, "question": question}):
        yield chunk

# Usage with retry via .with_retry()
chain_with_retry = chain.with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,
)
```

**Key concepts demonstrated:**
- `|` composition — LCEL declarative style
- `with_fallbacks()` — model tier chain
- `with_retry()` — retry with exponential backoff
- `astream()` — async streaming out of the box
- Nothing was manually implemented — LCEL provides all this

---

## Q2. Build a LangGraph Agent with Cycles and Human-in-the-Loop ⭐⭐⭐⭐

**Prompt:** Build a LangGraph workflow where an agent researches a topic, drafts a response, gets human approval, and revises based on feedback.

**Solution:**

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI

class AgentState(TypedDict):
    topic: str
    research: str
    draft: str
    feedback: str
    approved: bool
    revision_count: int

llm = ChatOpenAI(model="gpt-4o-mini")

def research_node(state: AgentState) -> AgentState:
    response = llm.invoke(f"Research this topic and provide key facts: {state['topic']}")
    return {"research": response.content}

def draft_node(state: AgentState) -> AgentState:
    context = state['research']
    if state.get('feedback'):
        context += f"\n\nPrevious feedback to address: {state['feedback']}"
    
    response = llm.invoke(
        f"Based on this research, write a response:\n{context}\n\nTopic: {state['topic']}"
    )
    return {
        "draft": response.content,
        "revision_count": state.get("revision_count", 0) + 1,
    }

def human_review_node(state: AgentState) -> AgentState:
    # In real system: pause here, wait for human input via API/UI
    # LangGraph's checkpointing allows resuming later
    print(f"DRAFT:\n{state['draft']}\n\nApprove? (yes/no) or provide feedback:")
    user_input = input()
    
    if user_input.lower() == "yes":
        return {"approved": True, "feedback": ""}
    else:
        return {"approved": False, "feedback": user_input}

def should_continue(state: AgentState) -> Literal["revise", "done", "escalate"]:
    if state["approved"]:
        return "done"
    if state["revision_count"] >= 3:
        return "escalate"  # Too many revisions, escalate to human handling
    return "revise"

# Build the graph
workflow = StateGraph(AgentState)
workflow.add_node("research", research_node)
workflow.add_node("draft", draft_node)
workflow.add_node("review", human_review_node)

workflow.set_entry_point("research")
workflow.add_edge("research", "draft")
workflow.add_edge("draft", "review")
workflow.add_conditional_edges(
    "review",
    should_continue,
    {"revise": "draft", "done": END, "escalate": END}
)

# Compile with checkpointing (state persists across pauses)
checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# Run
config = {"configurable": {"thread_id": "session_1"}}
result = app.invoke({"topic": "quantum computing basics"}, config=config)
```

**Key concepts:** Explicit graph structure, typed state, conditional routing, cycles (draft → review → draft), checkpointing for pausability.

---

## Q3. Build a CrewAI Multi-Agent Research Crew ⭐⭐⭐

**Prompt:** Build a crew of 3 agents (Researcher, Analyst, Writer) that produces a market analysis report.

**Solution:**

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, WebsiteSearchTool

# Define agents with specific roles and expertise
researcher = Agent(
    role="Senior Research Analyst",
    goal="Gather comprehensive market data on {topic}",
    backstory="You are a meticulous researcher with 15 years of experience "
              "at top consulting firms. You cross-verify every fact.",
    tools=[SerperDevTool(), WebsiteSearchTool()],
    verbose=True,
    allow_delegation=False,
)

analyst = Agent(
    role="Market Analyst",
    goal="Identify trends, opportunities, and risks from research data",
    backstory="You are a strategic analyst who transforms raw data into "
              "actionable insights. You think in frameworks like Porter's Five Forces.",
    verbose=True,
    allow_delegation=False,
)

writer = Agent(
    role="Executive Report Writer",
    goal="Produce a polished executive summary suitable for C-level audiences",
    backstory="You are a McKinsey-trained writer. You write with clarity, "
              "brevity, and impact. Executives read your reports in 5 minutes.",
    verbose=True,
    allow_delegation=False,
)

# Define tasks (each references outputs from previous)
research_task = Task(
    description="Research the market for {topic}. Include: market size, "
                "top players, recent trends, and key challenges. Provide sources.",
    agent=researcher,
    expected_output="A comprehensive research document with cited facts."
)

analysis_task = Task(
    description="Analyze the research data. Identify: 3 major opportunities, "
                "3 major risks, and the competitive landscape. Use structured frameworks.",
    agent=analyst,
    context=[research_task],  # Uses output of research_task
    expected_output="Strategic analysis with opportunities, risks, and framework analysis."
)

report_task = Task(
    description="Write a 500-word executive summary combining the research and analysis. "
                "Format: Executive Summary, Key Findings (3 bullets), Recommendations (3 bullets).",
    agent=writer,
    context=[research_task, analysis_task],
    expected_output="Polished executive report ready for C-level delivery."
)

# Assemble the crew
crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, report_task],
    process=Process.sequential,  # Or Process.hierarchical for manager-led
    verbose=True,
)

result = crew.kickoff(inputs={"topic": "generative AI enterprise adoption 2025"})
```

**Key concepts:** Role/goal/backstory pattern, task context chaining, sequential vs hierarchical process, tool assignment per agent.

---

## Q4. Build a Custom MCP Server ⭐⭐⭐⭐

**Prompt:** Build an MCP server that exposes tools for querying a company knowledge base. Server must support tool discovery, argument validation, and structured responses.

**Solution:**

```python
# server.py — MCP server implementation
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field
import sqlite3

mcp = FastMCP("KnowledgeBase")

# Database connection
db = sqlite3.connect("knowledge.db", check_same_thread=False)

# Tools exposed to MCP clients

@mcp.tool()
def search_articles(
    query: str = Field(description="Search query text"),
    category: str = Field(default="", description="Optional category filter"),
    limit: int = Field(default=5, description="Max results"),
) -> list[dict]:
    """
    Search the knowledge base for articles matching the query.
    Returns article titles, summaries, and IDs.
    """
    sql = "SELECT id, title, summary FROM articles WHERE content LIKE ?"
    params = [f"%{query}%"]
    
    if category:
        sql += " AND category = ?"
        params.append(category)
    
    sql += " LIMIT ?"
    params.append(limit)
    
    results = db.execute(sql, params).fetchall()
    return [{"id": r[0], "title": r[1], "summary": r[2]} for r in results]

@mcp.tool()
def get_article(article_id: int) -> dict:
    """Get full content of an article by ID."""
    row = db.execute(
        "SELECT id, title, content, author, published_at FROM articles WHERE id = ?",
        [article_id]
    ).fetchone()
    
    if not row:
        raise ValueError(f"Article {article_id} not found")
    
    return {
        "id": row[0], "title": row[1], "content": row[2],
        "author": row[3], "published_at": row[4],
    }

# Resources — read-only data URIs
@mcp.resource("categories://all")
def list_categories() -> str:
    """List all article categories."""
    rows = db.execute("SELECT DISTINCT category FROM articles").fetchall()
    return "\n".join([r[0] for r in rows])

# Prompts — reusable prompt templates
@mcp.prompt()
def summarize_article(article_id: int) -> str:
    """Prompt template for summarizing an article."""
    article = get_article(article_id)
    return f"Summarize this article in 3 bullet points:\n\nTitle: {article['title']}\n\n{article['content']}"

if __name__ == "__main__":
    mcp.run(transport="stdio")  # stdio for Claude Desktop, or "sse" for HTTP
```

**Claude Desktop config** (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "knowledge_base": {
      "command": "python",
      "args": ["/path/to/server.py"]
    }
  }
}
```

**Key concepts:** Tool decorator with Field descriptions (auto-schema), resources for read-only data, prompt templates, stdio transport for desktop integration.

---

## Q5. LoRA Fine-Tuning Configuration for a Specific Task ⭐⭐⭐⭐

**Prompt:** Configure LoRA for fine-tuning Llama-3-8B on a customer support classification task (5K examples). Explain each hyperparameter choice.

**Solution:**

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from peft import LoraConfig, get_peft_model, TaskType
from trl import SFTTrainer
from datasets import load_dataset

# Load model
model_name = "meta-llama/Meta-Llama-3-8B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,  # BF16 for training stability
    device_map="auto",
)

# LoRA configuration — every choice explained
lora_config = LoraConfig(
    # r=16 chosen because:
    # - r=8 underfit on 5K classification examples (validation loss plateau)
    # - r=32 showed no improvement over r=16 (overfitting risk)
    # - r=16 is the sweet spot for medium-sized tasks
    r=16,
    
    # alpha=32 (2x rank) — standard practice
    # The effective update is scaled by alpha/r = 32/16 = 2
    # Higher alpha = stronger adapter influence
    lora_alpha=32,
    
    # target_modules — which layers get LoRA
    # For quality: q_proj, k_proj, v_proj, o_proj (all attention)
    # For efficiency: just q_proj, v_proj (2x fewer params)
    # For classification, attention modules are most impactful
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    
    # Dropout on LoRA layers — prevents overfitting on small datasets
    # 0.05-0.1 typical; higher for smaller datasets
    lora_dropout=0.05,
    
    # bias="none" — don't train bias terms (rarely helps)
    bias="none",
    
    task_type=TaskType.CAUSAL_LM,
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Trainable params: ~13M (0.16% of 8B) 

# Training arguments
training_args = TrainingArguments(
    output_dir="./llama3-support-classifier",
    
    # Batch size — larger is more stable but memory-limited
    # Effective batch size = per_device × gradient_accumulation
    # 4 × 8 = 32 effective batch size
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,
    
    # Learning rate — LoRA tolerates higher LR than full FT
    # 2e-4 works well; 5e-4 for aggressive, 1e-4 for cautious
    learning_rate=2e-4,
    
    # Epochs — LoRA on small datasets: 2-4 epochs
    # More causes overfitting; less causes underfitting
    num_train_epochs=3,
    
    # LR scheduler — cosine helps LoRA converge smoothly
    lr_scheduler_type="cosine",
    warmup_ratio=0.1,  # 10% warmup prevents early instability
    
    # Precision — BF16 for A100/H100, FP16 for older GPUs
    bf16=True,
    
    # Save strategy
    save_strategy="epoch",
    save_total_limit=2,  # Keep only 2 checkpoints (disk cost)
    
    # Eval strategy
    evaluation_strategy="steps",
    eval_steps=50,
    load_best_model_at_end=True,
    
    # Logging
    logging_steps=10,
    report_to="wandb",  # or "tensorboard", "none"
)

# Train
trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=dataset["train"],
    eval_dataset=dataset["validation"],
    tokenizer=tokenizer,
    max_seq_length=1024,  # Match your data distribution
)

trainer.train()

# Save LoRA adapters (small — ~50MB, not full model)
model.save_pretrained("./llama3-support-classifier-lora")
```

**Why each choice matters** — the interviewer will ask about any of these hyperparameters. Being able to justify each one shows real fine-tuning experience.

---

## Q6. DPO Training Script for Preference Alignment ⭐⭐⭐⭐

**Prompt:** Build a DPO training script to align a model to prefer helpful responses over unhelpful ones. Include data format, loss configuration, and evaluation.

**Solution:**

```python
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer
from datasets import Dataset

# 1. Load SFT-tuned base model (must be instruction-tuned already)
model = AutoModelForCausalLM.from_pretrained("meta-llama/Meta-Llama-3-8B-Instruct")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Meta-Llama-3-8B-Instruct")

# 2. Prepare preference data
# Format: {"prompt": str, "chosen": str, "rejected": str}
preference_data = [
    {
        "prompt": "How do I improve my Python performance?",
        "chosen": "Here are 5 concrete techniques with examples: (1) use built-in functions...",
        "rejected": "Just use Python 3.11 or later.",
    },
    {
        "prompt": "What causes memory leaks in Python?",
        "chosen": "Common causes: circular references, global caches, unclosed resources...",
        "rejected": "Python has garbage collection so leaks are rare.",
    },
    # ... typically 1K-10K preference pairs
]
dataset = Dataset.from_list(preference_data)

# 3. Configure DPO training
dpo_config = DPOConfig(
    output_dir="./llama3-dpo-aligned",
    
    # Beta — KL penalty strength (0.1-0.5 typical)
    # Higher = more conservative (stays close to SFT baseline)
    # Lower = more aggressive updates (risks reward hacking)
    beta=0.1,
    
    # Standard training args
    per_device_train_batch_size=2,  # DPO is memory-heavy (2x forward passes)
    gradient_accumulation_steps=8,
    learning_rate=5e-6,  # Lower than SFT LR (fine-grained alignment)
    num_train_epochs=1,  # Usually 1 epoch is enough for DPO
    
    # Sequence lengths
    max_length=1024,      # Total sequence length
    max_prompt_length=512,  # Cap prompt length
    
    # Precision
    bf16=True,
    
    # Logging
    logging_steps=10,
    save_strategy="epoch",
)

# 4. Initialize DPO trainer
# Note: DPOTrainer creates reference model automatically (frozen copy)
# For memory efficiency, use LoRA — the reference is base model without LoRA
trainer = DPOTrainer(
    model=model,
    ref_model=None,  # Automatically uses model without LoRA as reference
    args=dpo_config,
    train_dataset=dataset,
    tokenizer=tokenizer,
    # Optional: use LoRA for memory efficiency
    peft_config=LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"]),
)

# 5. Train
trainer.train()

# 6. Evaluation — compare responses before/after
def compare_responses(prompt: str, model_before, model_after, tokenizer):
    for label, model in [("BEFORE DPO", model_before), ("AFTER DPO", model_after)]:
        inputs = tokenizer(prompt, return_tensors="pt")
        output = model.generate(**inputs, max_new_tokens=200)
        print(f"{label}: {tokenizer.decode(output[0])}\n")

# Verify alignment
compare_responses(
    "Explain how to debug a slow API endpoint",
    model_before=base_model,
    model_after=trainer.model,
    tokenizer=tokenizer,
)
```

**Key concepts demonstrated:**
- Preference data format (prompt, chosen, rejected)
- Beta hyperparameter tuning
- Reference model handling
- LoRA + DPO combination for memory efficiency
- Before/after evaluation

---

## Q7-Q15: Additional Coding Challenges (Condensed)

### Q7. Implement Function Calling with Retry and Validation ⭐⭐⭐
Build a wrapper around OpenAI function calling that validates outputs against Pydantic schemas and retries with error feedback on validation failure. Handle: schema mismatches, missing required fields, type coercion errors.

### Q8. Build a LangSmith-Compatible Tracing Decorator ⭐⭐⭐
Write a `@traced` decorator that captures inputs, outputs, latency, and cost for any function. Format traces as JSON compatible with LangSmith's API. Handle: nested function calls (parent/child spans), async functions, exception tracking.

### Q9. Implement a Structured Output Extractor with Instructor ⭐⭐
Use the Instructor library to extract structured data from unstructured text. Define Pydantic models for `ContractTerms`, `MeetingSummary`, `MedicalReport`. Handle: validation failures with automatic retry, partial extraction, confidence scoring.

### Q10. Build a Multi-Framework RAG (LangChain + LlamaIndex) ⭐⭐⭐⭐
Combine LlamaIndex for retrieval and LangChain for orchestration in a single pipeline. LlamaIndex handles: document loading, indexing, query engines. LangChain handles: prompt templates, chain composition, streaming. Show how to bridge both.

### Q11. Implement Prompt Injection Detection with Guardrails ⭐⭐⭐
Build a two-layer defense: (1) input classifier for common injection patterns using a smaller model, (2) output validator that checks if the LLM followed the system prompt vs the injection. Handle: encoded attacks, indirect injections, role-playing attacks.

### Q12. Build a Custom Evaluator for Domain-Specific Quality ⭐⭐⭐
Implement a custom evaluator for medical Q&A quality. Metrics: factual accuracy (LLM-as-judge against reference), safety (no dangerous advice), citation presence, and completeness. Integrate with DeepEval framework for CI/CD execution.

### Q13. Implement Quantization Pipeline (FP16 → AWQ) ⭐⭐⭐⭐
Take a fine-tuned Llama model in FP16 and produce an AWQ-quantized version for production serving. Include: calibration dataset selection, quantization config, quality verification (perplexity comparison), and vLLM deployment.

### Q14. Build a vLLM Serving Endpoint with Streaming ⭐⭐⭐
Deploy a quantized model with vLLM, expose OpenAI-compatible endpoint, enable continuous batching, and implement streaming response. Include: request queueing, batch size configuration, memory limits, and health checks.

### Q15. Compare 3 Prompt Engineering Approaches with Statistical Testing ⭐⭐⭐
Build an evaluation harness that tests zero-shot vs few-shot vs CoT approaches on a task. Run each 100 times. Compute: accuracy, latency, cost, statistical significance (Mann-Whitney U test) between approaches. Output: winner recommendation with confidence intervals.
