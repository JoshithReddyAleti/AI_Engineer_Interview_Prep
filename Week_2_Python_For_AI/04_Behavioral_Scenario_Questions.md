# 🎭 Week 2 — Behavioral & Scenario Questions

> **Focus:** Python-specific production incidents, dependency management, code quality debates, debugging war stories
>
> **How to use:** Practice these as conversational answers. Interviewers want to hear your reasoning process, not a script.

---

## Q1. The Dependency Disaster ⭐⭐

**Scenario:** You deploy on Friday afternoon. Monday morning, the service is down. Investigation reveals that a transitive dependency (`langchain` updated, which pulled a new `pydantic` version) broke your validation layer. 300+ requests failed over the weekend.

**Strong answer:**

**Immediate fix:**
- Pin the last known working versions in `requirements.txt` (exact pins, not ranges)
- Redeploy with pinned versions
- Verify the service recovers

**Root cause:** We weren't pinning transitive dependencies. `langchain>=0.1.0` resolved to a new version on deploy that required `pydantic>=2.5`, but our code was written for Pydantic v2.3 patterns.

**Prevention:**
1. **Use a lockfile** (`poetry.lock`, `pip-compile` from `pip-tools`, or `uv.lock`). This pins EVERY dependency including transitive ones.
2. **Never deploy on Friday.** This is an organizational rule, not just folklore — weekend incidents have the longest time-to-resolution.
3. **Separate dependency updates from feature deploys.** Update dependencies in a dedicated PR, run the full test suite, then merge separately from feature work.
4. **Pin AI library versions aggressively.** Libraries like `langchain`, `openai`, `anthropic` release breaking changes frequently. Pin exact versions and update deliberately.

**The bigger lesson:** Python dependency management is uniquely fragile in the AI ecosystem because AI libraries move fast and have deep dependency trees. Treat dependency updates as potentially breaking changes, not routine maintenance.

---

## Q2. Sync vs Async Debate ⭐⭐

**Scenario:** A senior engineer on your team wants to rewrite your FastAPI service from async to sync because "async is confusing and we keep getting event loop errors." The service makes 5 LLM API calls per request and handles 200 concurrent users. How do you respond?

**Strong answer:**

"I understand the frustration with async bugs — they're harder to debug. But switching to sync would be a serious performance regression for our workload. Let me explain why, and propose a better solution."

**The math:**
- Each LLM call takes ~2 seconds
- 5 calls per request, sequential = 10 seconds per request
- With sync + threading: 200 concurrent users × 1 thread each = 200 threads (heavy, OS scheduling overhead, memory-hungry)
- With async: 200 concurrent users × 5 API calls = 1000 concurrent I/O operations managed by a single thread's event loop (lightweight, efficient)

"Async is the right choice for I/O-bound workloads. What we should fix instead:"

1. **Audit for sync-in-async bugs.** The most common event loop error is calling a sync function inside an async endpoint. Let's grep for `requests.` and `time.sleep(` in async code — these are the usual culprits.

2. **Use `asyncio.to_thread()` for unavoidable sync code.** If a library only offers sync APIs, wrap it: `result = await asyncio.to_thread(sync_function, arg)`.

3. **Add an async linter rule.** Catch sync calls in async functions during CI, not in production.

4. **Training session.** Spend 2 hours teaching the team the 5 async patterns they'll actually use. Most AI engineering async is: `await`, `asyncio.gather`, `async for`, `async with`, and `asyncio.to_thread`.

---

## Q3. The Memory Leak Mystery ⭐⭐⭐

**Scenario:** Your AI service's memory usage grows from 500MB to 8GB over 24 hours, then gets OOM-killed by Kubernetes. The restart cycle repeats daily. How do you diagnose and fix this?

**Strong answer:**

**Step 1 — Reproduce and measure:**
```python
# Add memory tracking to the service
import tracemalloc
tracemalloc.start()

# Endpoint to check memory
@app.get("/debug/memory")
async def memory_stats():
    snapshot = tracemalloc.take_snapshot()
    top = snapshot.statistics("lineno")[:10]
    return {"top_allocations": [str(s) for s in top]}
```

**Step 2 — Common AI service memory leaks:**

1. **Conversation history growing unbounded.** If you store chat history in a Python list and never trim it, each user session accumulates indefinitely.
   - Fix: `ContextWindowManager` with max token budget (we built this in Week 1!)

2. **Response caching without TTL or max size.** Caching LLM responses is great, but if the cache dict grows forever...
   - Fix: Use `cachetools.TTLCache` or `functools.lru_cache(maxsize=1000)`

3. **Embedding vectors accumulating in memory.** If you load embeddings into NumPy arrays and never release them.
   - Fix: Use a vector database instead of in-memory arrays

4. **HTTP client connection pool leaks.** Creating new `httpx.Client()` per request without closing them.
   - Fix: Shared client via FastAPI lifespan (we built this in Q8 of coding questions!)

5. **Logging handlers accumulating.** Adding a new logging handler per request instead of configuring once.

**Step 3 — Production monitoring:**
- Add Prometheus metrics for process memory (`process_resident_memory_bytes`)
- Alert at 80% of pod memory limit
- Periodic heap dumps during off-peak for analysis

---

## Q4. Code Review Conflict — "Your Code Is Too Over-Engineered" ⭐⭐

**Scenario:** You submit a PR with Pydantic schemas, retry decorators, structured logging, and type hints for a new LLM integration. A teammate comments: "This is way too complex for a simple API call. Just use `requests.post()` and move on."

**Strong answer:**

"I appreciate the feedback, and I agree we should avoid unnecessary complexity. Let me explain what each piece does and let's decide together which ones are worth keeping."

Then walk through pragmatically:

**Keep — non-negotiable for production AI code:**
- **Pydantic schema for the response:** LLM outputs are unreliable. Without schema validation, a malformed response silently corrupts downstream logic. This has bitten us before.
- **Retry with backoff:** LLM APIs return 429 (rate limit) and 503 (overloaded) regularly. Without retry, we'd fail on ~5-10% of requests.
- **Type hints:** They cost nothing at runtime and prevent bugs at development time.

**Potentially simplifiable:**
- **Structured logging:** If we're not yet shipping to a log aggregator, `print()` is fine for now. But let's put a TODO to upgrade before production.
- **Custom exception hierarchy:** If we only have 2 error types, Python's built-in exceptions might be enough. I'll simplify.

"I'd rather have 'too much structure' on day 1 than add it after a production incident. But I'm happy to trim the parts that don't earn their complexity today."

**What NOT to do:** Don't get defensive. Don't say "you'll thank me later." Engage with the feedback, defend what matters, concede what doesn't.

---

## Q5. The "It Works on My Machine" Problem ⭐⭐

**Scenario:** A teammate's AI pipeline works perfectly on their Mac but fails in the Docker container with cryptic errors about missing system libraries. They're blocked and the feature is due tomorrow.

**Strong answer:**

**Immediate unblock (30 minutes):**
1. Check the error — likely a missing C library that a Python package depends on (common: `libmagic`, `poppler`, `tesseract`, `ffmpeg`)
2. Add the library to the Dockerfile: `RUN apt-get update && apt-get install -y libmagic1 poppler-utils`
3. Rebuild and test

**Root cause:** Mac includes many libraries (via Homebrew/Xcode) that Linux slim Docker images don't have. Python packages with C extensions (`python-magic`, `pdfplumber`, `opencv-python`) need system dependencies.

**Prevention:**
1. **Development containers (devcontainers).** Everyone develops inside the same Docker container — no "works on my machine" possible.
2. **CI builds Docker image on every PR.** If it fails in Docker, the PR can't merge.
3. **Document system dependencies** in a `SYSTEM_DEPS.md` or Dockerfile comments.
4. **Use "headless" package variants** when available: `opencv-python-headless` instead of `opencv-python`, `pdfplumber` instead of `camelot`.

---

## Q6. Choosing Between LLM Libraries ⭐⭐⭐

**Scenario:** Your team is starting a new AI project. One engineer wants LangChain ("it has everything"), another wants LlamaIndex ("it's better for RAG"), and a third wants to use raw API calls ("frameworks add complexity"). The tech lead asks for your recommendation.

**Strong answer:**

"This depends on what we're building and our team's experience level. Let me break it down."

**Raw API calls (OpenAI/Anthropic SDK directly):**
- Best when: Your pipeline is straightforward (prompt → call → parse), you want full control, team is experienced
- Risk: You'll rebuild abstractions that frameworks already provide (retry, streaming, tool calling)
- My bias: Start here. Only add a framework when you hit a pain point it solves.

**LangChain:**
- Best when: You need lots of integrations quickly (50+ LLM providers, 100+ tools, vector stores)
- Risk: Deep abstraction layers make debugging hard. "LangChain errors" is a meme for a reason. API changes frequently.
- My bias: Good for prototyping, often replaced with custom code in production.

**LlamaIndex:**
- Best when: Your primary use case is RAG / document Q&A. Its data ingestion and query pipeline abstractions are excellent.
- Risk: If your project grows beyond RAG, you'll outgrow it.

**LangGraph:**
- Best when: You need stateful, multi-step agent workflows with branching and cycles.
- Risk: Overkill for simple pipelines.

"For this project, I'd recommend: start with raw SDK calls for the core pipeline. If we find ourselves reimplementing framework-level features (complex tool orchestration, multi-agent coordination), then we evaluate adding LangGraph or LlamaIndex for that specific subsystem. Don't framework-first."

---

## Q7. Production Python Performance Investigation ⭐⭐⭐

**Scenario:** Your AI endpoint's p99 latency has crept from 3 seconds to 12 seconds over the last month. No code changes were deployed. What's your investigation process?

**Strong answer:**

**Phase 1 — External causes (most likely):**
1. **LLM API latency increase.** Check provider status pages. Run a standalone API call to measure raw latency. If their p99 went up, your p99 went up.
2. **Increased input length.** Are users sending longer prompts? More context in RAG? Check median token count over time.
3. **Traffic increase.** More concurrent users → connection pool saturation → queuing. Check concurrent connection metrics.

**Phase 2 — Internal causes:**
4. **Cache degradation.** Check cache hit rate over time. If it dropped from 30% to 5%, more requests hit the slow path (real API calls).
5. **Database slow queries.** If RAG retrieval involves a database, check query performance. Index degradation, table growth, or connection pool exhaustion.
6. **Python process health.** Memory growth (triggering GC pauses), thread exhaustion, or connection pool leaks.

**Phase 3 — Profiling:**
```python
# Add timing middleware
@app.middleware("http")
async def timing_middleware(request, call_next):
    start = time.time()
    response = await call_next(request)
    elapsed = time.time() - start
    response.headers["X-Process-Time-Ms"] = str(int(elapsed * 1000))
    
    # Log slow requests with breakdown
    if elapsed > 5.0:
        logger.warning(f"Slow request: {elapsed:.1f}s — {request.url.path}")
    
    return response
```

"In my experience, 80% of these creeping latency issues are caused by increased input size (token count) or LLM provider latency changes — not code bugs. Always check external dependencies first."

---

## Q8. Teaching Python to Non-Engineer AI Team Members ⭐⭐

**Scenario:** Your company hired a prompt engineer who has great AI intuition but minimal Python skills. They need to write and test prompts, run evals, and modify system prompts. How do you enable them without creating a support burden for engineering?

**Strong answer:**

**Build a developer experience, not a training program:**

1. **One-command setup:** `make setup` installs everything, creates `.env`, verifies API key works.

2. **Prompt-as-config, not prompt-as-code:** Put prompts in YAML/text files, not embedded in Python. The prompt engineer edits files, not code.
```yaml
# prompts/customer_support.yaml
system: |
  You are a helpful customer support agent for Acme Corp.
  Always be polite and professional.
  If you don't know the answer, say so.
temperature: 0.3
model: gpt-4o
```

3. **CLI commands for common tasks:**
```bash
$ make test-prompt FILE=prompts/customer_support.yaml
$ make run-eval SUITE=support_quality
$ make compare-prompts OLD=v1.yaml NEW=v2.yaml
```

4. **Jupyter notebooks for exploration:** Prompt engineers are comfortable in notebooks. Pre-build notebooks with: "Change the system prompt below and run all cells to see the effect."

5. **Git-based workflow:** Teach them to: create branch → edit prompt file → push → PR gets auto-tested → merge if eval passes. They never need to touch Python logic.

"The goal is to give them the maximum leverage with the minimum Python. Their domain expertise is the valuable part — our job is to remove the Python barrier, not add training overhead."
