# 🏗️ Week 2 — System Design Questions

> **Focus:** Python service architecture, API design, deployment patterns, testing infrastructure
>
> **How to use:** 30-45 minute whiteboard exercises. Start with requirements, then architecture, then deep-dive.

---

## Q1. Design a FastAPI Service That Wraps Multiple LLM Providers ⭐⭐⭐

**Prompt:** "Design a Python service (FastAPI) that provides a unified REST API for your frontend team. Behind the scenes, it routes to OpenAI, Anthropic, or a local model based on request type. Must handle 500 concurrent users."

**Architecture:**

```
Frontend / Mobile App
         │
         ▼
┌────────────────────────┐
│    FastAPI Application   │
│                          │
│  ┌────────────────────┐  │
│  │  Lifespan Manager   │  │  ← Pre-loads clients, models, connections
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │  Middleware Stack    │  │
│  │  - Auth (API key)   │  │
│  │  - Rate limiter     │  │
│  │  - Request logging  │  │
│  │  - CORS             │  │
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │  Router Layer       │  │
│  │  POST /chat         │  │
│  │  POST /embed        │  │
│  │  POST /classify     │  │
│  │  GET  /health       │  │
│  └─────────┬──────────┘  │
│            │              │
│  ┌─────────▼──────────┐  │
│  │  Provider Adapter    │  │  ← Abstract base class
│  │  ├─ OpenAIAdapter    │  │
│  │  ├─ AnthropicAdapter │  │
│  │  └─ LocalAdapter     │  │
│  └────────────────────┘  │
└────────────────────────┘
```

**Key design decisions:**

**Async all the way:** FastAPI is async-native. Every LLM call must be `async def` using `httpx.AsyncClient`. A single sync call in an async endpoint blocks the entire event loop for all 500 users.

```python
# WRONG — blocks event loop
@app.post("/chat")
async def chat(req: ChatRequest):
    response = requests.post(...)  # SYNC! Blocks everything!

# RIGHT — non-blocking
@app.post("/chat")
async def chat(req: ChatRequest):
    response = await async_client.post(...)  # Yields control while waiting
```

**Connection pooling via lifespan:** Create `httpx.AsyncClient` once at startup with connection limits. Creating a new client per request wastes TCP connections and adds latency.

**Request/response models with Pydantic:**
```python
class ChatRequest(BaseModel):
    messages: list[Message]
    model: str | None = None          # Optional: auto-route if not specified
    provider: str | None = None       # Optional: force provider
    temperature: float = Field(0.7, ge=0, le=2)
    max_tokens: int = Field(1024, ge=1, le=128000)
    stream: bool = False

class ChatResponse(BaseModel):
    content: str
    model: str
    provider: str
    usage: TokenUsage
    latency_ms: int
```

**Streaming support:** SSE (Server-Sent Events) for real-time token streaming.

**GET vs POST for AI endpoints:**
- `GET /health` — idempotent status check, no body needed
- `POST /chat` — sends data (messages), creates a resource (completion), not idempotent
- `POST /embed` — sends data (text), processes it, returns result
- Rule: **GET for reads, POST for operations that send data or have side effects**

**Scaling to 500 concurrent users:**
- Single FastAPI process handles ~100 concurrent requests (CPU-bound middleware limits this)
- Use `uvicorn --workers 4` for 4 processes (1 per CPU core)
- Put behind a load balancer (nginx, Kubernetes Ingress) for horizontal scaling
- The real bottleneck is LLM API latency, not Python — async handles this naturally

---

## Q2. Design a Python Testing Pipeline for an AI Application ⭐⭐⭐

**Prompt:** "Your AI application has: LLM calls, tool execution, database queries, and a REST API. Design a comprehensive testing strategy with appropriate mocking at each layer."

**Testing pyramid for AI apps:**

```
         ╱╲
        ╱  ╲         Human Eval (monthly)
       ╱    ╲         - Subjective quality assessment
      ╱──────╲        - Edge case discovery
     ╱        ╲
    ╱ LLM-as- ╲       LLM-as-Judge Eval (weekly / on PR)
   ╱  Judge    ╲       - Quality regression detection
  ╱─────────────╲      - A/B comparison of prompt changes
 ╱               ╲
╱  Integration    ╲    Integration Tests (on PR)
╱  Tests           ╲   - Real API calls (staging keys)
╱───────────────────╲  - End-to-end pipeline validation
╱                     ╲
╱    Unit Tests         ╲  Unit Tests (on every commit)
╱    (mocked LLM)        ╲ - Business logic, routing, validation
╱──────────────────────────╲- Deterministic, fast, comprehensive
```

**Mocking strategy by layer:**

```python
# Layer 1: Mock the LLM (unit tests)
# Tests YOUR code, not the model
def test_router_parses_tool_call(mocker):
    mock_response = '{"tool": "calculator", "args": {"expr": "2+2"}}'
    mocker.patch("app.llm.call_model", return_value=mock_response)
    
    result = router.route("What is 2+2?")
    assert result.tool_name == "calculator"

# Layer 2: Mock external APIs (integration tests)
# Tests your pipeline end-to-end without external dependencies
@pytest.fixture
def mock_weather_api(httpx_mock):
    httpx_mock.add_response(
        url="https://api.weather.com/current",
        json={"temp": 72, "condition": "sunny"},
    )

# Layer 3: Real API calls (staging environment only)
# Run nightly or pre-release, never on every commit (expensive)
@pytest.mark.integration
@pytest.mark.skipif(not os.getenv("STAGING_API_KEY"), reason="No staging key")
def test_full_pipeline_real_api():
    result = pipeline.run("What's the weather in NYC?")
    assert result.status == "success"
    assert "temperature" in result.response.lower() or "degrees" in result.response.lower()

# Layer 4: Property tests for non-deterministic outputs
def test_response_properties():
    for _ in range(10):  # Run multiple times due to non-determinism
        result = generate("Summarize: The quick brown fox...")
        assert len(result) < 500          # Property: summary is shorter
        assert result.endswith(".")        # Property: complete sentence
        assert "fox" in result.lower()     # Property: key entity preserved
```

**pytest configuration for AI projects:**
```ini
# pytest.ini
[pytest]
markers =
    unit: Fast, deterministic, mocked tests
    integration: Real API calls, slower
    eval: LLM quality evaluation, expensive

# Run fast tests on every commit:   pytest -m unit
# Run integration pre-merge:        pytest -m "unit or integration"
# Run full eval weekly:              pytest -m eval
```

---

## Q3. Design a Python Package Structure for a Multi-Team AI Project ⭐⭐⭐

**Prompt:** "You have 3 teams building on shared AI infrastructure: a chatbot team, a search team, and a data processing team. Design the Python package structure."

**Structure:**

```
ai-platform/
├── pyproject.toml              # Project metadata + dependencies
├── packages/
│   ├── ai-core/                # Shared library (all teams depend on this)
│   │   ├── ai_core/
│   │   │   ├── llm/            # LLM client, retry, caching
│   │   │   ├── embeddings/     # Embedding client, vector ops
│   │   │   ├── schemas/        # Shared Pydantic models
│   │   │   ├── config/         # Configuration management
│   │   │   └── observability/  # Logging, metrics, tracing
│   │   ├── tests/
│   │   └── pyproject.toml
│   │
│   ├── chatbot-service/        # Team 1's service
│   │   ├── chatbot/
│   │   │   ├── routes/
│   │   │   ├── agents/
│   │   │   └── tools/
│   │   ├── tests/
│   │   └── pyproject.toml      # depends on ai-core
│   │
│   ├── search-service/         # Team 2's service
│   │   └── ...
│   │
│   └── data-pipeline/          # Team 3's service
│       └── ...
│
├── evals/                      # Shared evaluation suite
│   ├── datasets/
│   └── runners/
│
└── infra/
    ├── Dockerfile
    └── k8s/
```

**Key principles:**

1. **`ai-core` is the shared dependency.** It owns the LLM client, retry logic, configuration, and Pydantic schemas. Changes here require reviews from all teams.

2. **Services are independently deployable.** Each team can deploy without coordinating with others. They depend on `ai-core` via pinned version, not `main` branch.

3. **Shared schemas prevent drift.** When the chatbot and search team both need to parse LLM responses, they use the same `ChatResponse` model from `ai-core`, not their own versions.

4. **Evals are shared.** Quality regressions in `ai-core` affect everyone, so the eval suite runs against all downstream services before `ai-core` ships a new version.

---

## Q4. Design a Data Pipeline for Processing 100K PDFs ⭐⭐⭐⭐

**Prompt:** "Your company has 100K PDF documents (total 2TB). Design a Python pipeline that extracts text, chunks it, generates embeddings, and stores them in a vector database. Budget: $500 and 24 hours."

**Architecture:**

```
100K PDFs (S3/GCS)
       │
       ▼
┌──────────────────┐
│  Worker Pool       │  (multiprocessing — CPU-bound PDF parsing)
│  - 8 workers       │
│  - Extract text    │
│  - Clean/normalize │
│  - Yield chunks    │
└────────┬───────────┘
         │
    Chunk Queue (in-memory or Redis)
         │
         ▼
┌──────────────────┐
│  Embedding Batch   │  (async — I/O-bound API calls)
│  - Batch 32 chunks │
│  - Rate-limited    │
│  - Retry on failure│
└────────┬───────────┘
         │
         ▼
┌──────────────────┐
│  Vector DB Upsert  │  (batch writes)
│  - 100 vectors/call│
│  - Progress tracked│
└──────────────────┘
```

**Cost estimation:**
- 100K PDFs × ~20 pages avg × ~500 tokens/page = ~1B tokens of text
- Chunked to 512-token chunks = ~2M chunks
- Embedding at $0.02/1M tokens (text-embedding-3-small) = ~$20
- Vector DB storage (Pinecone serverless): ~$10/month for 2M vectors
- Compute (8-core instance for 12 hours): ~$50
- **Total: ~$80 — well within $500 budget**

**Critical Python patterns used:**

```python
# 1. multiprocessing for CPU-bound PDF parsing
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=8) as pool:
    for result in pool.map(extract_text, pdf_paths, chunksize=50):
        chunk_queue.put(result)

# 2. async batching for I/O-bound embedding
async def embed_batches(queue, batch_size=32):
    batch = []
    async for chunk in queue:
        batch.append(chunk)
        if len(batch) >= batch_size:
            embeddings = await embedding_client.embed(batch)
            yield embeddings
            batch = []

# 3. Generator pipeline for memory efficiency
# Never load all 2TB into memory — stream everything
def pipeline():
    for pdf_path in list_pdfs():           # yields paths
        for page in extract_pages(pdf_path): # yields pages
            for chunk in chunk_text(page):    # yields chunks
                yield chunk                    # one chunk at a time
```

**Failure handling for 24-hour jobs:**
- Checkpoint progress every 1000 documents (write processed file list to disk)
- On restart, skip already-processed files
- Dead letter queue for PDFs that fail parsing (retry later, don't block the pipeline)
- Heartbeat monitoring — alert if processing rate drops below threshold

---

## Q5. Design a Python CLI Tool for AI Workflow Management ⭐⭐

**Prompt:** "Design a CLI tool (like a mini `make` for AI projects) that manages: running evals, deploying prompts, monitoring costs, and generating reports."

**Interface:**

```bash
$ ai-tool eval run --suite regression --model gpt-4o
$ ai-tool eval compare --baseline v1.2 --candidate v1.3
$ ai-tool prompt deploy --file prompts/chat.txt --env staging
$ ai-tool prompt rollback --to v1.2
$ ai-tool cost report --period 7d --by team
$ ai-tool cost alert --threshold 500 --period daily
```

**Implementation pattern (using `click`):**

```python
import click

@click.group()
def cli():
    """AI Workflow Management Tool"""
    pass

@cli.group()
def eval():
    """Evaluation commands"""
    pass

@eval.command()
@click.option("--suite", required=True, help="Eval suite name")
@click.option("--model", default="gpt-4o-mini", help="Model to evaluate")
@click.option("--parallel", default=5, help="Concurrent eval workers")
def run(suite, model, parallel):
    """Run an evaluation suite."""
    click.echo(f"Running {suite} on {model} with {parallel} workers...")
    # Load eval dataset, run in parallel, aggregate results
    results = run_eval_suite(suite, model, max_workers=parallel)
    
    # Display results as table
    click.echo(f"\nResults: {results.accuracy:.1%} accuracy")
    click.echo(f"  Pass: {results.passed} | Fail: {results.failed}")
    
    if results.accuracy < results.threshold:
        click.secho("⚠️  Below threshold!", fg="red", bold=True)
        raise SystemExit(1)

@cli.group()
def cost():
    """Cost monitoring commands"""
    pass

@cost.command()
@click.option("--period", default="7d", help="Time period")
@click.option("--by", type=click.Choice(["team", "model", "feature"]))
def report(period, by):
    """Generate cost report."""
    data = fetch_cost_data(period, group_by=by)
    for row in data:
        click.echo(f"  {row.name}: ${row.cost:.2f} ({row.requests} requests)")

if __name__ == "__main__":
    cli()
```

---

## Q6. Design a Graceful Degradation System for AI Features ⭐⭐⭐

**Prompt:** "Your app has 5 AI-powered features. The LLM API goes down. Design a Python system that degrades gracefully — features fail individually, not catastrophically."

**Pattern: Circuit breaker + feature flags + fallbacks**

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing — use fallback
    HALF_OPEN = "half_open" # Testing recovery

class CircuitBreaker:
    """
    Per-feature circuit breaker.
    
    CLOSED → failures exceed threshold → OPEN
    OPEN → wait recovery_timeout → HALF_OPEN
    HALF_OPEN → success → CLOSED / failure → OPEN
    """
    def __init__(
        self,
        feature_name: str,
        failure_threshold: int = 5,
        recovery_timeout: float = 60.0,
    ):
        self.feature_name = feature_name
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0.0
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        elif self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        else:  # HALF_OPEN
            return True
    
    def record_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Feature registry with individual circuit breakers
class AIFeatureManager:
    def __init__(self):
        self.breakers = {}
        self.fallbacks = {}
    
    def register(self, name: str, handler, fallback, threshold=5):
        self.breakers[name] = CircuitBreaker(name, threshold)
        self.fallbacks[name] = fallback
    
    async def execute(self, name: str, *args, **kwargs):
        breaker = self.breakers[name]
        
        if not breaker.can_execute():
            # Circuit is open — use fallback immediately
            return await self.fallbacks[name](*args, **kwargs)
        
        try:
            result = await self.breakers[name]  # ... handler logic
            breaker.record_success()
            return result
        except Exception:
            breaker.record_failure()
            return await self.fallbacks[name](*args, **kwargs)

# Each feature degrades independently:
# - Chat AI fails → show "AI unavailable, connecting to human agent"
# - Search AI fails → fall back to keyword search
# - Summarization fails → show full text with "summary unavailable" badge
# - Classification fails → route to default queue
# - Recommendation fails → show popular/trending items
```
