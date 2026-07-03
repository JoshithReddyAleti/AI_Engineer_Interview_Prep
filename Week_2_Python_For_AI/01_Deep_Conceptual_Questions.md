# 🧠 Week 2 — Deep Conceptual Questions

> **Focus:** Python patterns for AI engineering — async, data modeling, env management, error philosophy
>
> **How to use:** These test whether you understand *why* Python patterns exist, not just *how* to type them.

---

## Q1. Why is Python the dominant language for AI engineering, despite being "slow"? ⭐

**What the interviewer is really testing:** Do you understand the ecosystem trade-off, not just the syntax?

**Answer:**

Python is slow at raw computation — ~100x slower than C/C++ for tight loops. But that's irrelevant for AI engineering because:

**1. Python is glue, not the engine.** The actual compute happens in C/C++/CUDA libraries (PyTorch, TensorFlow, NumPy, FAISS). Python just orchestrates calls to these optimized backends. When you call `model.generate()`, Python sends the request and waits — the GPU does the work.

**2. The bottleneck is I/O, not CPU.** AI applications spend most of their time waiting — waiting for LLM API responses (500ms-10s), waiting for database queries (5-50ms), waiting for HTTP requests (100ms-5s). Python's "slowness" at computation is invisible when you're spending 99% of time in I/O wait.

**3. The ecosystem is unmatched.** Every LLM provider ships a Python SDK first. Every ML framework is Python-first. Every vector database has a Python client. Choosing another language means fighting the ecosystem.

**4. Development velocity matters more than execution speed.** For AI systems, the ability to iterate quickly on prompts, pipelines, and architectures is more valuable than saving 2ms of Python overhead.

**When Python's speed actually matters:**
- Processing millions of documents for embedding (use batch processing, multiprocessing)
- Real-time inference serving at >1000 RPS (consider Rust/Go for the serving layer, Python for the logic)
- Tokenization at scale (this is why `tiktoken` is written in Rust with Python bindings)

---

## Q2. Explain the difference between sync, async, threading, and multiprocessing in Python. When would you use each for AI workloads? ⭐⭐⭐

**What the interviewer is really testing:** This is one of the most-asked Python questions at MAANG AI interviews.

**Answer:**

**Synchronous (default):**
```python
# One request at a time — each waits for the previous to finish
response1 = call_llm(prompt1)  # waits 2 seconds
response2 = call_llm(prompt2)  # waits 2 seconds
# Total: 4 seconds
```
Use when: Simple scripts, sequential pipelines, prototyping.

**Async/await (asyncio):**
```python
# Concurrent I/O — start all requests, wait for all to finish
response1, response2 = await asyncio.gather(
    call_llm_async(prompt1),
    call_llm_async(prompt2),
)
# Total: ~2 seconds (parallel I/O wait)
```
Use when: **Multiple LLM API calls, multiple HTTP requests, multiple database queries.** This is the #1 pattern for AI engineering because AI workloads are I/O-bound. A single Python thread manages hundreds of concurrent network connections.

**Threading (concurrent.futures.ThreadPoolExecutor):**
```python
# Multiple threads — useful for I/O-bound work with sync libraries
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(call_llm, p) for p in prompts]
    results = [f.result() for f in futures]
```
Use when: You need concurrency but the library is synchronous (no async support). Also useful for: file I/O, calling legacy APIs.

**The GIL caveat:** Python's Global Interpreter Lock means only one thread executes Python bytecode at a time. But threads *release the GIL during I/O operations*, so threading works well for I/O-bound tasks. It does NOT help for CPU-bound work.

**Multiprocessing (concurrent.futures.ProcessPoolExecutor):**
```python
# Separate processes — true parallelism, bypasses GIL
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(process_document, documents))
```
Use when: **CPU-bound work** — document parsing, text preprocessing, embedding computation on CPU, data transformation. Each process has its own Python interpreter and GIL.

**AI engineering decision matrix:**

| Task | Best Approach | Why |
|---|---|---|
| 10 LLM API calls | async/await | I/O-bound, need concurrency |
| 100K document embedding | multiprocessing + async | CPU for preprocessing, async for API calls |
| FastAPI endpoint calling LLM | async | Native async framework, non-blocking |
| Parsing 10K PDFs | multiprocessing | CPU-bound text extraction |
| Streaming LLM response | async generator | Non-blocking stream consumption |
| Background model health checks | threading | Simple periodic I/O tasks |

---

## Q3. What's the difference between a `dict`, a `dataclass`, and a `Pydantic BaseModel`? When do you use each in AI systems? ⭐⭐

**What the interviewer is really testing:** Do you think about data contracts and validation — or do you just pass dicts around?

**Answer:**

**`dict` — unstructured, flexible, zero safety:**
```python
response = {"role": "assistant", "content": "Hello", "tokens": 5}
# No type checking, no validation, keys can be misspelled
response["tokns"] = 5  # No error — silent bug
```
Use when: Quick prototyping, dynamic data where the shape isn't known ahead of time, passing through data you don't own.

**`dataclass` — structured, typed, lightweight:**
```python
from dataclasses import dataclass

@dataclass
class LLMResponse:
    role: str
    content: str
    tokens: int
    
response = LLMResponse(role="assistant", content="Hello", tokens=5)
response.tokns = 5  # AttributeError — caught immediately
```
Use when: Internal data structures where you trust the data source (your own code). Type hints provide documentation and IDE support but **no runtime validation**.

**`Pydantic BaseModel` — structured, validated, serializable:**
```python
from pydantic import BaseModel, Field

class LLMResponse(BaseModel):
    role: str
    content: str
    tokens: int = Field(ge=0)  # must be >= 0
    
# Runtime validation
response = LLMResponse(role="assistant", content="Hello", tokens=-1)
# ValidationError: tokens must be >= 0

# Free serialization
response.model_dump()       # → dict
response.model_dump_json()  # → JSON string
LLMResponse.model_validate_json(json_str)  # → object from JSON
```
Use when: **Every boundary in an AI system.** API inputs, API outputs, LLM response parsing, tool call arguments, database records. Pydantic catches malformed data before it corrupts your pipeline.

**The AI engineering principle:** Use dicts inside exploratory notebooks. Use dataclasses for internal-only data. Use Pydantic at every boundary where data crosses a trust boundary (external API → your code, LLM output → your processing, user input → your system).

---

## Q4. How do you manage secrets and API keys in AI projects? What's wrong with just putting them in the code? ⭐

**What the interviewer is really testing:** Do you have basic security hygiene?

**Answer:**

**What's wrong with hardcoding:**
```python
# NEVER DO THIS
client = OpenAI(api_key="sk-proj-abc123...")
```
- One `git push` and your key is in version history forever (even if you delete the line later)
- GitHub has automated secret scanners — providers like OpenAI auto-revoke leaked keys
- Different environments (dev, staging, prod) need different keys
- Team members shouldn't all share one key

**The correct approach — layered:**

**Layer 1: `.env` file + `python-dotenv` (development):**
```python
# .env (in .gitignore — never committed)
OPENAI_API_KEY=sk-proj-abc123
LLM_MODEL=gpt-4o-mini

# code
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv("OPENAI_API_KEY")
if not api_key:
    raise ValueError("OPENAI_API_KEY not set — check .env file")
```

**Layer 2: Platform secrets (production):**
- AWS Secrets Manager / Parameter Store
- Google Cloud Secret Manager
- Azure Key Vault
- Kubernetes Secrets
- Environment variables set by CI/CD pipeline

**Layer 3: Secret rotation:**
- Keys should rotate on a schedule (quarterly minimum)
- Application reads from secret manager on startup, not from static config
- If a key is compromised, rotation takes minutes, not a code deploy

**Layer 4: `.env.example` for onboarding:**
```bash
# .env.example (committed to repo — shows what's needed without values)
OPENAI_API_KEY=sk-your-key-here
LLM_MODEL=gpt-4o-mini
LOG_LEVEL=INFO
```

---

## Q5. What are the most important error handling patterns for AI applications? How is it different from traditional software? ⭐⭐⭐

**What the interviewer is really testing:** Do you build resilient systems or fragile scripts?

**Answer:**

AI applications have a unique error profile compared to traditional software:

**1. Non-deterministic failures.** The same input can succeed 9 times and fail on the 10th (API rate limits, model timeouts, inconsistent outputs). This means: **retry with backoff is not optional — it's baseline.**

**2. Partial failures are common.** The LLM returns 90% correct JSON with one malformed field. Traditional code would throw and abort. AI code should: parse what you can, flag what you can't, decide if it's usable.

**3. "Soft" failures that look like success.** The API returns 200 OK, the response parses correctly, but the content is wrong (hallucination, wrong format, irrelevant answer). You need output validation, not just HTTP error checking.

**The patterns:**

**Pattern 1 — Retry with exponential backoff:**
```python
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=30),
    retry=retry_if_exception_type((ConnectionError, TimeoutError, RateLimitError)),
)
def call_llm(prompt: str) -> str:
    return client.chat.completions.create(...)
```

**Pattern 2 — Graceful degradation:**
```python
def get_answer(query: str) -> Response:
    try:
        return call_primary_model(query)
    except PrimaryModelError:
        try:
            return call_fallback_model(query)
        except FallbackModelError:
            return Response(
                answer="I'm unable to process this right now.",
                confidence=0.0,
                fallback_used=True,
            )
```

**Pattern 3 — Output validation (not just error catching):**
```python
def process_llm_response(raw: str) -> ValidatedOutput:
    try:
        parsed = json.loads(raw)
        validated = OutputSchema.model_validate(parsed)
        
        # Semantic validation — catches "soft" failures
        if validated.confidence < 0.3:
            raise LowConfidenceError("Model is uncertain")
        if len(validated.answer) < 10:
            raise IncompleteResponseError("Answer suspiciously short")
            
        return validated
    except json.JSONDecodeError:
        # Retry with stricter formatting instructions
        return retry_with_format_enforcement(raw)
    except ValidationError as e:
        # Log the failure for eval improvement
        log_validation_failure(raw, e)
        raise
```

**Pattern 4 — Structured error responses (not bare exceptions):**
```python
@dataclass
class AIResult:
    success: bool
    data: Any = None
    error: str = None
    error_type: str = None  # "api_error", "validation_error", "timeout", etc.
    retries_used: int = 0
    model_used: str = ""
    latency_ms: int = 0
```

---

## Q6. Explain Python's `json` module limitations for AI workloads. What alternatives exist? ⭐⭐

**What the interviewer is really testing:** Do you know the sharp edges of the tools you use daily?

**Answer:**

**Limitation 1: No datetime/date serialization.**
```python
import json
json.dumps({"created": datetime.now()})  # TypeError
# Fix: use default=str, or orjson which handles it natively
```

**Limitation 2: No NumPy/tensor serialization.**
```python
json.dumps({"embedding": np.array([1.0, 2.0])})  # TypeError
# Fix: convert to list first, or use a custom encoder
```

**Limitation 3: Speed.** For large payloads (embeddings, batch responses), `json` is slow.
```python
# Standard library — ~10MB/s for large JSON
import json
data = json.loads(huge_response)

# orjson — ~200MB/s, handles bytes, datetime, numpy natively
import orjson
data = orjson.loads(huge_response)

# ujson — ~50MB/s, drop-in replacement API
import ujson
data = ujson.loads(huge_response)
```

**Limitation 4: No schema validation.** `json.loads()` tells you the JSON is syntactically valid but says nothing about whether it has the right fields, types, or values. This is why Pydantic exists in the AI stack.

**Limitation 5: No streaming JSON parsing.** When you receive a large JSON response from an API, `json.loads()` needs the entire string in memory before parsing. For streaming LLM responses that include JSON, you need incremental parsing:
```python
# ijson for streaming JSON parsing
import ijson
for item in ijson.items(response_stream, 'results.item'):
    process(item)  # Processes each item as it arrives
```

**What good AI engineers do:** Use `orjson` for performance, `Pydantic` for validation, and the standard `json` module only in prototyping.

---

## Q7. What is the difference between a generator and a list in Python? Why do generators matter for AI pipelines? ⭐⭐

**What the interviewer is really testing:** Do you think about memory efficiency at scale?

**Answer:**

**List:** Computes and stores ALL elements in memory at once.
```python
# All 10M embeddings in memory simultaneously
embeddings = [embed(doc) for doc in documents]  # Could use 40GB+ RAM
```

**Generator:** Computes elements one at a time, on demand (lazy evaluation).
```python
# Only one embedding in memory at a time
def embed_stream(documents):
    for doc in documents:
        yield embed(doc)

for embedding in embed_stream(documents):
    store_in_vector_db(embedding)  # Process and discard
```

**Why this matters for AI:**

1. **Embedding 1M documents:** A list of 1M embeddings (1536-dim float32) = ~5.7 GB RAM. A generator uses ~6 KB.

2. **Streaming LLM responses:** You want to display tokens as they arrive, not wait for the full response.
```python
async for chunk in client.chat.completions.create(stream=True, ...):
    token = chunk.choices[0].delta.content
    yield token  # Send to UI immediately
```

3. **Batch processing with backpressure:** Generators naturally create backpressure — the producer only generates the next item when the consumer is ready. This prevents OOM in data pipelines.

4. **Chaining pipeline stages:**
```python
def read_documents(path):
    for file in Path(path).glob("*.pdf"):
        yield parse_pdf(file)

def chunk_documents(docs):
    for doc in docs:
        yield from split_into_chunks(doc, size=512)

def embed_chunks(chunks):
    batch = []
    for chunk in chunks:
        batch.append(chunk)
        if len(batch) >= 32:
            yield from embed_batch(batch)
            batch = []
    if batch:
        yield from embed_batch(batch)

# The full pipeline — constant memory regardless of corpus size
pipeline = embed_chunks(chunk_documents(read_documents("/data")))
for embedding in pipeline:
    vector_db.upsert(embedding)
```

---

## Q8. How do you test AI applications where outputs are non-deterministic? ⭐⭐⭐

**What the interviewer is really testing:** This is one of the hardest practical problems in AI engineering.

**Answer:**

Traditional testing: input → expected output → assert match. AI testing: input → variable output → assert *properties*.

**Strategy 1 — Property-based testing:**
Instead of checking for an exact answer, check for properties the answer must have:
```python
def test_summarization():
    result = summarize(long_document)
    
    # Property: shorter than input
    assert len(result) < len(long_document) * 0.3
    
    # Property: contains key entities
    assert "Anthropic" in result  # key entity from the document
    
    # Property: valid structure
    assert result.endswith(".")
    assert not result.startswith(" ")
```

**Strategy 2 — Mock the LLM for unit tests:**
```python
def test_router_selects_calculator(mocker):
    # Mock the LLM to return a predictable routing decision
    mocker.patch("app.router.call_llm", return_value='{"tool": "calculator", "args": {"expression": "2+2"}}')
    
    result = router.route("What is 2+2?")
    assert result.tool == "calculator"
    # Tests YOUR code, not the model
```

**Strategy 3 — Snapshot testing:**
```python
def test_response_format():
    # Pin temperature=0 for reproducibility
    result = generate(prompt, temperature=0)
    
    # Validate schema, not content
    parsed = ResponseSchema.model_validate_json(result)
    assert parsed.answer is not None
    assert 0 <= parsed.confidence <= 1
```

**Strategy 4 — Statistical testing for eval suites:**
```python
def test_accuracy_above_threshold():
    results = [evaluate(case) for case in eval_dataset]
    accuracy = sum(r.correct for r in results) / len(results)
    
    assert accuracy >= 0.85, f"Accuracy {accuracy:.2%} below 85% threshold"
    # Also track per-category accuracy to catch hidden regressions
```

**Strategy 5 — LLM-as-judge (for subjective quality):**
```python
def test_response_quality():
    response = generate(prompt)
    
    # Use a separate model to evaluate
    judge_prompt = f"""Rate this response 1-5 on relevance and helpfulness:
    Question: {prompt}
    Answer: {response}
    Return JSON: {{"relevance": int, "helpfulness": int}}"""
    
    scores = judge_model.evaluate(judge_prompt)
    assert scores["relevance"] >= 3
    assert scores["helpfulness"] >= 3
```

---

## Q9. What's the difference between `pip`, `pip install --break-system-packages`, virtual environments, `poetry`, and `uv`? When do you use each? ⭐⭐

**What the interviewer is really testing:** Can you manage Python dependencies without causing incidents?

**Answer:**

**`pip` (bare):** Installs packages globally. On modern Ubuntu/Debian, `pip install X` fails with "externally-managed-environment" error to prevent breaking system Python.

**`pip install --break-system-packages`:** Forces global install, bypassing the protection. Use only in: Docker containers, CI/CD, disposable environments. Never on your local machine.

**Virtual environments (`venv`):**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
Creates an isolated Python environment per project. This is the minimum acceptable practice. Each project has its own dependencies that don't interfere with each other.

**`poetry`:** Dependency resolver + lockfile + virtual env manager. `pyproject.toml` defines dependencies, `poetry.lock` pins exact versions.
```bash
poetry add openai pydantic
poetry install  # Installs exact versions from lockfile
```
Better than pip+requirements.txt because: deterministic installs (lockfile ensures everyone gets exactly the same versions), dependency resolution (catches conflicts before install).

**`uv`:** Rust-based Python package manager. 10-100x faster than pip. Drop-in replacement.
```bash
uv pip install -r requirements.txt  # Same as pip, but fast
uv venv                              # Create venv
```
Use when: speed matters (CI/CD, Docker builds, large dependency sets). It's the future direction of the ecosystem.

**What production AI projects should use:**
- `poetry` or `uv` + lockfile for dependency management
- Docker for deployment (isolated environment guaranteed)
- Pin major versions of AI libraries (breaking changes are common in fast-moving libraries like `langchain`, `openai`)
- Separate `requirements.txt` for prod vs dev vs test dependencies

---

## Q10. How do you handle rate limits when calling LLM APIs? ⭐⭐

**What the interviewer is really testing:** Have you actually built production systems that hit scale?

**Answer:**

LLM APIs have multiple rate limit dimensions:
- Requests per minute (RPM)
- Tokens per minute (TPM)
- Requests per day (RPD)

**The naive approach fails:**
```python
# This will get rate-limited immediately
results = [call_llm(p) for p in thousand_prompts]
```

**Production approaches:**

**1. Token bucket / leaky bucket rate limiter:**
```python
import asyncio
from asyncio import Semaphore

class RateLimiter:
    def __init__(self, max_concurrent: int = 10, requests_per_minute: int = 60):
        self.semaphore = Semaphore(max_concurrent)
        self.delay = 60.0 / requests_per_minute
    
    async def __aenter__(self):
        await self.semaphore.acquire()
        return self
    
    async def __aexit__(self, *args):
        await asyncio.sleep(self.delay)
        self.semaphore.release()

limiter = RateLimiter(max_concurrent=5, requests_per_minute=50)

async def rate_limited_call(prompt):
    async with limiter:
        return await call_llm_async(prompt)
```

**2. Retry with backoff on 429 errors:**
```python
from tenacity import retry, retry_if_exception_type, wait_exponential

@retry(
    retry=retry_if_exception_type(RateLimitError),
    wait=wait_exponential(multiplier=2, min=4, max=60),
    stop=stop_after_attempt(5),
)
async def call_with_retry(prompt):
    return await client.chat.completions.create(...)
```

**3. Batch API for non-real-time work:**
OpenAI and Anthropic offer batch endpoints at 50% cost — queue thousands of requests, get results in hours. Perfect for: offline eval, document processing, data enrichment.

**4. Request queuing with priority:**
Real-time user requests → high priority, immediate processing.
Background tasks → low priority, process when headroom allows.

---

## Q11. Why is `requests` synchronous and when should you use `httpx` or `aiohttp` instead? ⭐⭐

**What the interviewer is really testing:** Do you pick the right HTTP library for the job?

**Answer:**

**`requests` — synchronous, blocking:**
```python
import requests
response = requests.get("https://api.openai.com/v1/models")
# Thread is blocked until response arrives
```
Great for: simple scripts, sequential API calls, prototyping. Bad for: concurrent calls, FastAPI endpoints (blocks the event loop), high-throughput pipelines.

**`httpx` — sync AND async, modern:**
```python
import httpx

# Sync (like requests)
response = httpx.get("https://api.openai.com/v1/models")

# Async
async with httpx.AsyncClient() as client:
    response = await client.get("https://api.openai.com/v1/models")
```
Use when: You need both sync and async, or you want one library for everything. Supports HTTP/2, streaming, timeouts out of the box.

**`aiohttp` — async-only, high performance:**
```python
import aiohttp

async with aiohttp.ClientSession() as session:
    async with session.get("https://api.openai.com/v1/models") as response:
        data = await response.json()
```
Use when: Maximum async performance, high-concurrency scenarios. Slightly more verbose but highly optimized for async workloads.

**AI engineering recommendation:**
- Prototyping / scripts → `requests`
- FastAPI applications → `httpx` (async mode)
- High-throughput data pipelines → `aiohttp`
- If the LLM SDK you're using handles HTTP internally (OpenAI Python, Anthropic Python) → let the SDK manage it, just use their async methods

---

## Q12. What are type hints and why do MAANG-level AI teams require them? ⭐⭐

**What the interviewer is really testing:** Do you write professional, maintainable code?

**Answer:**

Type hints document expected types for function arguments and return values:
```python
# Without hints — what does this return? What's the format of messages?
def call_llm(messages, model, temperature):
    ...

# With hints — self-documenting, IDE autocomplete, catches bugs
def call_llm(
    messages: list[dict[str, str]],
    model: str = "gpt-4o",
    temperature: float = 0.7,
) -> dict[str, Any]:
    ...
```

**Why MAANG teams require them:**

1. **Catch bugs before runtime.** `mypy` static type checking catches type mismatches during development, not in production at 2 AM.

2. **Self-documenting code.** Types ARE documentation. When you read `embedding: np.ndarray` you know exactly what it is without reading the implementation.

3. **IDE superpowers.** VSCode/PyCharm use type hints for autocomplete, refactoring, and error highlighting. This is a genuine productivity multiplier in large codebases.

4. **Pydantic integration.** Type hints power Pydantic validation, which is the backbone of structured LLM output parsing:
```python
class ToolCall(BaseModel):
    tool_name: str
    arguments: dict[str, Any]
    confidence: float = Field(ge=0, le=1)
```

5. **Team scaling.** With 50+ engineers touching the same codebase, type hints prevent entire categories of integration bugs.

**The minimum bar for production AI code:**
- All function signatures should have type hints
- Return types should be specified
- Pydantic models for all data boundaries
- Run `mypy` in CI (even in `--ignore-missing-imports` mode)
