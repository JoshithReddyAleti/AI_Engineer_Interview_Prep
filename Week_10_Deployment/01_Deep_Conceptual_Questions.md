# 🧠 Week 10 — Deep Conceptual Questions

> **Focus:** Every deployment concept with enterprise depth — FastAPI internals, all 4 rate limiting algorithms with formulas, circuit breaker state machines, JWT vs OAuth2, K8s vs serverless, SLO/SLI, semantic caching, LLM gateway patterns, cost-based rate limiting, blameless postmortems
>
> **How to use:** These questions come up in staff+ interviews at Anthropic, OpenAI, Google, Meta, and every serious AI startup. Answers include the enterprise trade-off, not just the mechanic.

---

## Q1. FastAPI vs Flask vs Django for AI backends. Why is FastAPI the modern default? ⭐⭐⭐

**What the interviewer is really testing:** Do you know WHY FastAPI won for AI, not just that it did?

**FastAPI's advantages for AI workloads:**

**1. Async-first design.**
LLM calls are I/O-bound (waiting on network). Async lets one worker handle 100s of concurrent LLM requests without extra threads. Flask/Django are synchronous by default — every LLM request blocks a whole worker.

Concrete impact: at 10 concurrent users, 5s LLM latency:
- Flask (sync, 4 workers): can handle 4 concurrent requests → users queue
- FastAPI (async, 4 workers): can handle 400+ concurrent requests → no queuing

**2. Native Pydantic integration.**
Type-safe request/response schemas. Auto-validation. Auto-OpenAPI docs. Critical for AI where you're constantly serializing structured data.

**3. Native streaming support.**
LLM responses stream token-by-token. FastAPI's `StreamingResponse` and `EventSourceResponse` are built for this. Flask needs manual chunked-encoding.

**4. Type hints as contracts.**
Function signatures = API schema. Refactoring is safe.

**5. Performance.**
Built on Starlette + Uvicorn (ASGI). ~3x faster than Flask on typical workloads.

**When Flask/Django still make sense:**
- Existing Flask/Django team, no time to migrate
- Django admin dashboard is critical
- Sync-heavy workloads (traditional CRUD without AI)

**Enterprise concerns:**

**Sync vs async pitfall:** Async requires ALL downstream code to be async. Mixing sync DB clients into async FastAPI kills performance worse than pure Flask. Interviewers love asking about this — "You used FastAPI with sync SQLAlchemy. What happens?"

Answer: sync calls block the event loop, effectively making the whole app synchronous. Use asyncpg or SQLAlchemy 2.0 async instead.

**Interview signal:** "FastAPI is fast if you use it right; slower than Flask if you use it wrong" shows real experience.

---

## Q2. Explain Docker multi-stage builds. Why do they matter for AI production? ⭐⭐⭐⭐

**What the interviewer is really testing:** Real containerization knowledge, not just `FROM python:latest`.

**Single-stage build (naive):**
```dockerfile
FROM python:3.11
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app:app"]
```

Resulting image: ~1.2GB. Includes: build tools, pip cache, dev dependencies, all source files including .git.

**Multi-stage build (production):**
```dockerfile
# Stage 1: Build environment
FROM python:3.11-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends build-essential
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime environment
FROM python:3.11-slim AS runtime
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser app/ ./app/
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Resulting image: ~200MB. Includes: only runtime dependencies + code.

**Why this matters for AI production:**

**1. Pull speed.** 200MB image pulls in 5s vs 1.2GB in 30s. Matters for auto-scaling (fast cold start), CI/CD (faster deploys), K8s (faster pod scheduling).

**2. Attack surface.** Fewer tools in image = fewer CVEs. Build tools + dev deps = attackers' toy chest.

**3. Cost.** Registry storage, egress bandwidth, node storage. Cheap per GB, expensive at scale.

**4. Cold start.** Serverless (Cloud Run, Fargate) cold starts scale with image size. 200MB image ≈ 3s cold start; 1.2GB ≈ 15s+.

**Enterprise concerns:**

- **Non-root user:** Never run as root in production. Layer above shows `USER appuser`.
- **Health check baked in:** Container reports its own health.
- **Distroless / minimal base images:** For maximum hardening — `gcr.io/distroless/python3-debian12` has no shell, no package manager, only what's needed to run Python.
- **Reproducible builds:** Pin base image by digest (`python:3.11-slim@sha256:...`) not tag (`python:3.11-slim`). Tags can change.
- **SBOM generation:** Enterprise increasingly requires Software Bill of Materials. Tools: Syft, Docker's built-in SBOM.

**Interview signal:** "Multi-stage + non-root + pinned digest + health check" is the enterprise Dockerfile spec.

---

## Q3. Explain all 4 rate limiting algorithms with formulas. Which for what? ⭐⭐⭐⭐

**What the interviewer is really testing:** Depth on a critical production topic.

### 1. Fixed Window Counter

**Formula:**
```
requests_in_current_window = counter
if counter >= limit: reject
else: counter += 1
Reset counter at window boundary (e.g., every 60s)
```

**Example:** 100 requests/minute. Counter resets at :00 of each minute.

**Pros:** Simplest. Very low memory.
**Cons:** Boundary problem — user can hit 200 requests in 2 seconds (100 at 0:59, 100 at 1:00).

**When to use:** Simple use cases, rough rate limiting. Not for AI (LLM cost is too high per request to allow the boundary spike).

### 2. Sliding Window Log

**Formula:**
```
For each request:
  log.append(current_timestamp)
  log.remove_expired(now - window_size)
  if len(log) >= limit: reject
  else: accept
```

**Example:** Store timestamps of last 100 requests in Redis sorted set. On new request, check count within last 60 seconds.

**Pros:** Perfectly accurate. No boundary problem.
**Cons:** Memory-heavy. Each request stored individually. Doesn't scale to millions of requests.

**When to use:** High-value APIs where accuracy > memory cost. Not for high-volume public APIs.

### 3. Sliding Window Counter (Weighted Combination)

**Formula:**
```
current_count = counter[current_window]
previous_count = counter[previous_window]
elapsed_in_current = time_since_current_window_start
weight = elapsed_in_current / window_size

effective_count = previous_count * (1 - weight) + current_count
if effective_count >= limit: reject
```

**Example:** 100 requests/minute. At 30 seconds into the current minute:
- Previous window: 80 requests
- Current window: 50 requests
- Weight of current: 0.5
- Effective: 80 × 0.5 + 50 = 90 → still under 100, allow

**Pros:** Memory-efficient (just 2 counters per user). Approximates sliding window log.
**Cons:** Slight inaccuracy (~1-2%). Sufficient for most cases.

**When to use:** Production default. Redis-friendly. Scales to millions.

### 4. Token Bucket

**Formula:**
```
State: tokens (starts at capacity), last_refill_time
On request:
  now = time.now()
  elapsed = now - last_refill_time
  tokens_to_add = elapsed * refill_rate
  tokens = min(capacity, tokens + tokens_to_add)
  last_refill_time = now
  if tokens >= cost_of_request:
    tokens -= cost_of_request
    accept
  else: reject
```

**Example:** Capacity = 100, refill rate = 10/second. After 5 seconds inactive, bucket has 50 tokens (or capped at 100).

**Pros:** Allows bursts (up to bucket capacity). Smooth over time. Perfect for API gateways.
**Cons:** Slightly more complex state management.

**When to use:** APIs where burst tolerance matters (user occasionally sends 10 requests in a burst, then quiet). LLM APIs love this.

### 5. Leaky Bucket (bonus)

**Formula:**
```
Queue with fixed processing rate
Requests added to queue if queue not full
Processed at constant rate regardless of arrival rate
If queue full: reject
```

**When to use:** When you need SMOOTH downstream rate (never burst downstream). E.g., protecting a fragile downstream service.

### Cost-Based Rate Limiting (LLM-specific)

The problem: 1 request ≠ 1 request in LLM systems.
- Small query: 500 input + 200 output tokens = $0.001
- Large query: 50K input + 10K output tokens = $1.00

Rate limiting by request count is meaningless. **Rate limit by COST.**

```
User budget: $10/day
Each request estimates or measures cost
Deduct from budget
Reject when budget exhausted
Reset daily
```

Implementation with Redis:
```python
async def check_cost_budget(user_id: str, estimated_cost: float) -> bool:
    key = f"cost_budget:{user_id}:{today()}"
    current = float(await redis.get(key) or 0)
    if current + estimated_cost > DAILY_LIMIT:
        return False
    await redis.incrbyfloat(key, estimated_cost)
    await redis.expire(key, 86400)  # 24h TTL
    return True
```

### Selection framework:

| Need | Algorithm |
|---|---|
| Rough per-user rate | Fixed window |
| Perfect accuracy | Sliding window log |
| Production default | Sliding window counter |
| Burst tolerance | Token bucket |
| Smooth downstream | Leaky bucket |
| LLM cost control | Cost-based |

**Enterprise pattern:** LAYERED — token bucket for burst control + cost-based for budget enforcement + fixed window for coarse abuse prevention.

**Interview signal:** "For LLM APIs, request count is the wrong dimension — cost is" shows LLM-specific depth.

---

## Q4. Explain the circuit breaker pattern with its 3 states. Why does every LLM call need one? ⭐⭐⭐⭐

**What the interviewer is really testing:** Distributed systems + LLM reliability.

**The three states:**

```
     failures < threshold        cool-down elapsed
CLOSED ────────────────────►   OPEN ─────────────────► HALF-OPEN
   ▲   failures ≥ threshold      │                        │
   │                              ▼                        │
   │                                                       │
   │◄─── success ──────────────────────────────────────────┤
   │                                                       │
   │                          failure                     │
   │◄──────────────────────────────────────────────────────┘
```

**CLOSED (normal):**
- Requests pass through
- Track failure rate
- If failures exceed threshold, transition to OPEN

**OPEN (blocking):**
- All requests immediately rejected (fast failure)
- Downstream service gets to recover
- After cool-down period, transition to HALF-OPEN

**HALF-OPEN (testing):**
- Allow single test request
- If success, transition to CLOSED
- If failure, back to OPEN

**Formula for OPEN transition:**
```
Rolling window: last N requests OR last T seconds
Failure count in window ≥ threshold → OPEN

Common thresholds:
- 5 failures in 30s
- 50% failure rate over 100 requests
- 10 consecutive failures
```

**Cool-down before HALF-OPEN:**
```
Typical: 30-60 seconds
Adaptive: exponential backoff (30s → 60s → 120s)
```

**Why LLM APIs need this:**

**1. Provider outages happen.**
OpenAI, Anthropic, Google all have outages. Without circuit breaker, your app hangs waiting for timeouts. With circuit breaker, requests fail fast and fallback triggers.

**2. Cost containment.**
Retry logic + failing provider = each request costs full price + retry costs. Circuit breaker stops the bleeding.

**3. Cascade prevention.**
Slow LLM = slow requests = piled-up requests = thread pool exhaustion = whole service down. Circuit breaker prevents cascade.

**4. Better UX.**
Fast failure (100ms) + fallback response > 30-second timeout hanging.

**Implementation with a library (aiobreaker):**
```python
from aiobreaker import CircuitBreaker

llm_breaker = CircuitBreaker(
    fail_max=5,               # Open after 5 failures
    timeout_duration=60,      # Try again after 60s
    exclude=[ClientError],    # Don't count 4xx as service failures
)

@llm_breaker
async def call_openai(prompt):
    return await openai.chat.completions.create(...)
```

**Enterprise concerns:**

**Per-provider circuit breakers.** OpenAI circuit shouldn't affect Anthropic. Each provider gets its own breaker.

**Per-endpoint circuit breakers.** Chat completions vs embeddings vs image gen — different failure modes. Separate breakers per endpoint.

**Observability.** Circuit state changes are ALERTS. "Circuit breaker OPEN for OpenAI" wakes on-call because it means production is degraded.

**Half-open concurrency.** In HALF-OPEN, allow only ONE test request. Not all traffic. Otherwise you thundering-herd the recovering service.

**Interview signal:** Discussing "per-provider, per-endpoint, with alerts on state changes" shows real LLM production experience.

---

## Q5. Compare API keys, JWT, OAuth 2.0, and sessions. When each? ⭐⭐⭐

**What the interviewer is really testing:** Auth pattern selection.

### API Keys

**How:** Long random string given to consumer. Sent as header (`X-API-Key: sk_live_abc123`).

**Storage:** Consumer stores it securely. Server hashes it (bcrypt) and compares.

**Use for:**
- Server-to-server (B2B API access)
- CLI tools
- Long-lived automation
- When you control the consumer

**Don't use for:**
- Browser JavaScript (leaks in DevTools)
- Mobile apps (extractable from binary)

**Enterprise concerns:**
- Rotation: keys should expire or be revocable
- Scope: fine-grained permissions per key
- Audit: log every use for compliance
- Rate limits: per-key, not per-user

### JWT (JSON Web Token)

**How:** Server signs a JSON payload. Client stores + sends it. Server verifies signature (no DB lookup needed).

**Anatomy:**
```
Header.Payload.Signature

{"alg": "HS256"}.{"user_id": "123", "exp": 1234567890}.hmac_sha256(...)
```

**Use for:**
- Web/mobile app auth (stateless, no session store needed)
- Microservices auth (service passes token to service)
- Short-lived tokens (15-60 min recommended)

**Advantages:**
- Stateless — no DB lookup on every request
- Scales horizontally trivially
- Standard, widely supported

**Disadvantages:**
- Can't easily revoke (until expiry)
- Payload visible (base64, not encrypted by default)
- Vulnerable to secret leak (attacker mints valid tokens)

**Best practices:**
- Short expiry (15-60 min)
- Refresh token pattern (separate long-lived token for renewals)
- Rotate signing keys periodically
- HS256 for single-service; RS256 for multi-service (asymmetric)

### OAuth 2.0

**How:** Complex flow with authorization server, resource server, client, resource owner. User authorizes third-party via login-with-Google/GitHub/etc.

**Use for:**
- Third-party integrations ("Log in with Google")
- Granting scoped access ("This app can read your calendar")
- Delegated authorization

**Don't use for:**
- Simple API auth (overkill)
- First-party auth (use JWT directly)

### Sessions

**How:** Server generates session ID, stores in cookie. Session data in Redis/DB. Every request looks up session.

**Use for:**
- Server-rendered web apps
- When you need to invalidate immediately

**Disadvantages:**
- Not scalable without shared session store
- Cookie limitations (same-origin)

### Comparison table:

| Method | Stateless | Revocable | Complexity | Use When |
|---|---|---|---|---|
| API Key | Yes | Yes | Low | B2B API |
| JWT | Yes | No (until expiry) | Medium | Web/mobile |
| OAuth 2.0 | Yes | Yes | High | Third-party auth |
| Sessions | No | Yes | Low | Server-rendered |

**Enterprise LLM pattern (layered):**
```
Layer 1: API key at gateway (identify tenant)
Layer 2: JWT after login (identify user within tenant)
Layer 3: Fine-grained scopes (RBAC)
Layer 4: Rate limits per key/user
Layer 5: Audit log every request
```

**Interview signal:** Discussing "JWT for user auth + API keys for machine auth + OAuth2 only for third-party" shows practical judgment.

---

## Q6. Explain RBAC vs ABAC. When to use each? ⭐⭐⭐

**What the interviewer is really testing:** Authorization model depth.

### RBAC (Role-Based Access Control)

**Model:** Users have Roles. Roles have Permissions.

**Example:**
```
User "alice" → Role "admin"
Role "admin" → Permissions ["read:users", "write:users", "delete:users"]
```

**Pros:** Simple. Familiar. Auditable ("who has admin?").
**Cons:** Combinatorial explosion. If you need "admin who can only edit East region users", you create a new role. Roles multiply.

**When to use:** Most SaaS apps. Enterprise IT. Clear org hierarchy.

### ABAC (Attribute-Based Access Control)

**Model:** Access decisions based on attributes of subject, resource, action, context.

**Example:**
```
Allow if:
  subject.department == resource.department
  AND action IN ["read", "write"]
  AND context.time BETWEEN 9am AND 5pm
  AND context.location == "office_network"
```

**Pros:** Fine-grained. Handles complex policies. No role explosion.
**Cons:** More complex. Harder to audit. Policy engines needed (OPA, Casbin).

**When to use:** Financial services, healthcare, government — anywhere with complex compliance requirements.

### Hybrid pattern (common enterprise):

Base access via RBAC (fast, coarse). Fine-grained decisions via ABAC (context-sensitive).

Example:
- RBAC: "Only 'admins' can access user management" (coarse)
- ABAC: "Admins can only edit users in their region during business hours" (fine)

### AI-specific concerns:

Multi-tenant AI systems have unique needs:
- Tenant isolation (never see other tenant's data) — RBAC + resource-level checks
- Per-feature access (some tenants pay for advanced features) — ABAC by subscription
- Cost limits per role (free tier = 100 requests/day, paid = unlimited) — ABAC by tier
- Model access (some tenants can use GPT-4, others only GPT-3.5) — ABAC by contract

**Enterprise implementation:**

Use a policy engine (OPA — Open Policy Agent) for ABAC. Write policies in Rego language. Test policies. Integrate with services.

**Interview signal:** "RBAC for basics, ABAC for context-sensitive decisions, OPA for policy engine" shows enterprise IAM knowledge.

---

## Q7. Compare Blue-Green vs Canary vs Rolling deployments. When each? ⭐⭐⭐⭐

**What the interviewer is really testing:** Deployment strategy depth.

### Blue-Green Deployment

**How:**
- Two identical environments: "Blue" (current prod) and "Green" (new version)
- Deploy new version to Green
- Test Green thoroughly
- Switch traffic 100% to Green (via load balancer / DNS)
- Blue becomes standby (rollback ready)

**Pros:**
- Instant rollback (switch traffic back)
- Full testing before switch
- No mixed versions

**Cons:**
- 2x infrastructure cost (both envs running)
- Database migrations tricky (both versions might need schema)
- Bad if you can't do all-or-nothing switchover

**When to use:**
- Stateless services
- Critical services where downtime unacceptable
- When you have budget for 2x infra during deploys

### Canary Deployment

**How:**
- Deploy new version to a small subset (e.g., 5% of servers or specific users)
- Monitor error rates, latency, business metrics
- If good, expand: 5% → 25% → 50% → 100%
- If bad, rollback immediately

**Pros:**
- Minimizes blast radius of bad deploys
- Real production feedback before full rollout
- Standard pattern at scale

**Cons:**
- Complexity — need traffic-splitting infrastructure
- Longer deployment time (gradual rollout takes hours)
- Mixed versions during rollout (can cause bugs)

**When to use:**
- Production default at scale (Netflix, Google, etc.)
- Any change with production risk
- New features you want to A/B test

### Rolling Deployment

**How:**
- Replace instances one at a time
- Deploy to instance 1, wait for healthy, deploy to instance 2, etc.
- K8s default deployment strategy

**Pros:**
- Simple
- No 2x infrastructure
- Zero downtime

**Cons:**
- Slower rollback (must roll new version back)
- Mixed versions during rollout
- If bad, may affect many users before detection

**When to use:**
- Standard K8s deployments
- Less risky changes
- When budget doesn't allow blue-green

### For AI systems specifically:

**Prompt changes:** Canary (5% of traffic gets new prompt, compare metrics).
**Model version bump:** Blue-green + shadow (run both, compare quality on same inputs).
**Infrastructure changes:** Rolling (standard K8s pattern).
**Emergency rollback needed:** Blue-green (instant switch).

**Canary math for AI:**

Standard canary %: 5% → 25% → 50% → 100% with 30 min at each stage.

For AI, add QUALITY metric to standard health metrics:
- Error rate (< 1%)
- Latency (p95 < baseline + 20%)
- Cost per request (< baseline + 10%)
- QUALITY score (LLM-judge on sample — must not regress > 5%)

If any metric fails at any stage → rollback.

**Interview signal:** Naming AI-specific canary metrics (quality, cost) shows LLM production experience.

---

## Q8. Explain SLO/SLI. How do you set them for LLM systems? ⭐⭐⭐⭐

**What the interviewer is really testing:** Reliability engineering depth.

**Definitions:**

**SLI (Service Level Indicator):** A measurable metric of service behavior.
**SLO (Service Level Objective):** Target for the SLI over a time window.
**SLA (Service Level Agreement):** Contractual promise (SLO + consequences).

**Standard SLIs:**

- Availability: % of successful requests
- Latency: p50, p95, p99 response time
- Throughput: requests per second
- Correctness: % of correct responses

**LLM-specific SLIs:**

- **Quality:** LLM-judge score, user thumbs-up rate
- **Cost efficiency:** cost per successful interaction
- **Hallucination rate:** measured via automated eval
- **Fallback rate:** % of requests served by fallback provider
- **Cache hit rate:** % served from semantic cache
- **Time to first token:** for streaming responses (feels like latency to user)

**Setting SLOs for AI systems:**

**Example: Chat product**
- Availability: 99.5% (36 hours/year downtime allowed)
- Latency (streaming TTFT): p95 < 1s, p99 < 3s
- Latency (full response): p95 < 8s (LLMs are slow)
- Cost per query: p95 < $0.05
- Quality: LLM-judge score >= 4.0 average
- Cache hit rate: >= 40%

**Error budgets:**

If SLO = 99.5% availability, error budget = 0.5% = 3.6 hours/month unavailable.

Error budget as decision framework:
- Budget healthy (barely used) → ship features faster
- Budget depleting (approaching limit) → freeze features, focus on reliability
- Budget exceeded → hard freeze, root cause required

**Enterprise pattern:**

Different SLOs per tier:
- Free tier: 99% availability, no SLA
- Paid tier: 99.5% + latency SLO, refund if breached
- Enterprise tier: 99.9% + custom SLOs, penalty clauses

**Multi-window SLOs:**

Not just monthly — also:
- 1-hour window (catches acute issues fast)
- 24-hour window (catches sustained issues)
- 30-day window (long-term trends)

**Interview signal:** Discussing "error budget as decision framework for feature velocity" shows SRE maturity.

---

## Q9. What is semantic caching for LLMs? How does it work? ⭐⭐⭐⭐

**What the interviewer is really testing:** LLM-specific optimization depth.

**The problem:**

Traditional cache: `key = query_string, value = response`. Only exact matches hit.

LLM queries have variance:
- "What is the capital of France?"
- "Tell me the capital of France"
- "France's capital?"

Same answer. Traditional cache misses all but the first.

**Semantic caching:**

Cache by MEANING, not exact string.

**Algorithm:**
```
On query Q:
  1. Compute embedding E(Q)
  2. Search cache for similar embeddings
  3. If nearest embedding has similarity > threshold (e.g., 0.95):
     Return cached response
  4. Else:
     Call LLM
     Store (E(Q), Q, response) in cache
     Return response
```

**Similarity threshold:**
- Too high (0.99): near-perfect matches only, low hit rate
- Too low (0.85): different queries return same response (wrong answers)
- Sweet spot: 0.93-0.97 for most tasks

**Storage:**
- Redis with vector search (Redis 8+)
- Or dedicated vector DB (Pinecone, Weaviate)
- Or in-memory (small deployments)

**Cost impact:**

Real numbers from production systems:
- Cache hit rate: 30-60% (varies by workload)
- Cost reduction: 30-60% (each cache hit = $0 LLM cost)
- Latency reduction: 5-10s → 20ms for cache hits

**Concrete example:**
- 1M requests/day, $0.02 avg per request = $20K/day = $600K/year
- 40% cache hit rate → $240K/year savings

That's often more than the entire eng team's headcount cost.

**When semantic caching FAILS:**

**1. Time-sensitive queries.**
"What's the weather now?" — same query, different correct answer each time. Don't cache.

**2. User-specific responses.**
"What's on my calendar?" — must be per-user. Cache with user_id in key.

**3. Slightly different intent.**
"How do I refund?" vs "How much for refund?" — similar embeddings, different needed answers. Might return wrong response.

**Mitigation:** Filter by category, use hybrid (semantic + metadata) caching.

**Enterprise concerns:**

- **TTL:** Cached responses stale eventually. Set TTL (24h, 7d, forever based on content type).
- **Invalidation:** When source data changes, invalidate related cache entries. Hard problem.
- **Multi-tenant:** Never share cache across tenants (data leak risk). Namespace by tenant_id.
- **Cost of the cache itself:** Vector search has cost. At scale, self-hosted (FAISS) cheaper than Pinecone.

**Interview signal:** Discussing "30-40% cache hit rate saves $200K+/year" with real numbers shows production experience.

---

## Q10. Explain the LLM Gateway pattern. Why every serious AI product needs one. ⭐⭐⭐⭐

**What the interviewer is really testing:** Production LLM architecture.

**The problem:**

Naive architecture: your app calls OpenAI directly.
```python
response = await openai.chat.completions.create(...)
```

What happens when:
- OpenAI is down? Your app is down.
- OpenAI is slow? Your app is slow.
- OpenAI raises prices? You pay more.
- You want to try Claude for one endpoint? Rewrite.
- Cost tracking? Manual per-request instrumentation everywhere.
- Prompt injection defense? Repeated in every call site.

**The LLM Gateway pattern:**

Single abstraction layer between your app and LLM providers.

```
Your App ──►  LLM Gateway  ──►  OpenAI
                    │      ──►  Anthropic
                    │      ──►  Google
                    │      ──►  Self-hosted
```

**What the gateway handles:**

**1. Multi-provider abstraction.**
Uniform API. Switch providers with config change.

**2. Automatic fallbacks.**
Primary: OpenAI. Fallback: Anthropic. If OpenAI fails, try Anthropic transparently.

**3. Retry logic.**
Exponential backoff with jitter. Configurable per provider.

**4. Circuit breakers.**
Per-provider. Fast failure when provider is down.

**5. Cost tracking.**
Every call attributed to user, tenant, feature. Aggregated in real-time.

**6. Rate limiting.**
Per-provider (respect their limits) and per-user (your policies).

**7. Semantic caching.**
Central cache benefits all callers.

**8. Prompt injection defense.**
Input sanitization at the gateway.

**9. Logging & observability.**
Every LLM call logged with full context. Traces to Langfuse/LangSmith.

**10. Model routing.**
Cheap model for simple tasks, expensive for complex. Configurable rules.

**Existing solutions:**

- **LiteLLM** (open source, unified interface)
- **Portkey** (commercial gateway)
- **Cloudflare AI Gateway** (edge-hosted)
- **AWS Bedrock** (AWS's own multi-provider gateway)
- **Custom** (build your own for full control)

**Enterprise pattern (production stack):**

```
Client
  │
  ▼
API Gateway (Kong/Nginx) — auth, rate limit, WAF
  │
  ▼
Application (FastAPI)
  │
  ▼
LLM Gateway (LiteLLM/Portkey/custom)
  ├─► Cache lookup (semantic)
  ├─► Provider router
  ├─► Fallback chain
  └─► Providers (OpenAI, Anthropic, ...)
```

**When to build vs buy:**

- Building your own: full control, custom logic, avoid vendor lock-in
- Buying (Portkey, Cloudflare): faster to production, less maintenance
- Middle ground: LiteLLM open source + your customizations

**Interview signal:** Discussing "LLM gateway is not optional at scale — it's the difference between a startup that survives an OpenAI outage and one that doesn't" shows real production wisdom.

---

## Q11-Q27: Additional Deep Conceptual Questions (Condensed)

### Q11. Kubernetes vs Serverless (Cloud Run, Fargate) vs PaaS (Railway, Render). Decision framework? ⭐⭐⭐⭐
**Start:** Railway/Render (Heroku-like, $7-50/month). **Scale:** Cloud Run/Fargate serverless containers ($0.05/M req + compute). **Complex:** Kubernetes (min $150/month, days of learning). **Rule:** don't use K8s unless you have specific needs it uniquely solves (custom networking, service mesh, GPU workloads, multi-region orchestration). The K8s tax: 20-30% of eng time on infrastructure. Justify it.

### Q12. Streaming responses (SSE vs WebSocket vs gRPC bidirectional). When each? ⭐⭐⭐
**SSE (Server-Sent Events):** unidirectional server→client. Perfect for LLM streaming (server pushes tokens). Simple HTTP, works through proxies, auto-reconnect. **WebSocket:** bidirectional, but complexity overkill for one-way streaming. Use for chat where client sends mid-stream. **gRPC bidirectional:** internal service-to-service. Not typically for browser clients. **LLM default: SSE.** LangChain, Anthropic, OpenAI all use SSE.

### Q13. Explain the 12-Factor App methodology. Why does it matter for AI? ⭐⭐⭐
1. Codebase (one per app), 2. Dependencies (explicit), 3. Config (env vars, not code), 4. Backing services (as attached resources), 5. Build/release/run (separate stages), 6. Processes (stateless), 7. Port binding (self-contained), 8. Concurrency (scale horizontally), 9. Disposability (fast start/stop), 10. Dev/prod parity, 11. Logs (as streams), 12. Admin processes (as one-off). **For AI:** especially factors 3 (secrets not in code), 6 (stateless enables scaling), 10 (dev/prod parity prevents "works on my machine" for LLM behavior).

### Q14. Explain HashiCorp Vault vs AWS Secrets Manager vs cloud-native alternatives. ⭐⭐⭐
**AWS Secrets Manager / GCP Secret Manager / Azure Key Vault:** cloud-native, integrated with IAM, automatic rotation, ~$0.40/secret/month. Best for single-cloud deployments. **HashiCorp Vault:** cloud-agnostic, most features (dynamic secrets, PKI, transit encryption), operational complexity. Self-host or HCP Vault (managed). Best for multi-cloud or advanced needs. **Doppler:** developer-friendly, syncs across envs, great DX. **Rule:** cloud-native for single-cloud simplicity; Vault for multi-cloud or compliance-heavy; Doppler for smaller teams.

### Q15. Explain graceful degradation with concrete LLM examples. ⭐⭐⭐⭐
When primary path fails, provide REDUCED but functional service instead of hard failure. **Example 1:** OpenAI down → fallback to Anthropic (transparent to user). **Example 2:** All LLM providers down → return cached response if similar query in cache. **Example 3:** All providers + cache down → return canned response ("I'm experiencing issues, please try again"). **Example 4:** Vector DB down → skip RAG, LLM answers from parametric knowledge only. Each layer is worse than ideal but better than "500 error." Design the degradation hierarchy upfront.

### Q16. Explain distributed tracing with OpenTelemetry. How does it work for AI? ⭐⭐⭐
Trace: single logical operation (e.g., "user's chat request"). Contains multiple spans (operations within): auth check, cache lookup, RAG retrieval, LLM call, response formatting. Each span has: trace_id, span_id, parent_id, start_time, duration, attributes. OpenTelemetry is the open standard. Instrument once, export to any backend (Jaeger, Datadog, Honeycomb). For AI: add attributes like llm.model, llm.tokens.input, llm.tokens.output, llm.cost. Trace shows entire request lifecycle across services. **Interview trap:** "How do you debug a slow LLM request?" Answer: distributed trace shows exactly where the 8 seconds went (retrieval? LLM? post-processing?).

### Q17. Blameless postmortems. What makes them effective? ⭐⭐⭐
Format: incident timeline, impact assessment, root cause analysis, contributing factors, action items with owners/dates, lessons learned. **Rules:** no naming individuals, focus on process failures, no "should have" language (implies blame), no punishment for honest mistakes. **Google's insight:** "the system enabled this failure to happen" — not "engineer X caused it." **Enterprise pattern:** all incidents severity-2+ get postmortems. Reviewed weekly by team. Action items tracked to completion. **Anti-pattern:** postmortem that names/blames someone → nobody reports incidents next time → hidden failures pile up → catastrophic incident.

### Q18. Explain OWASP Top 10 for LLM applications (LLM01-LLM10). ⭐⭐⭐⭐
1. Prompt Injection, 2. Insecure Output Handling, 3. Training Data Poisoning, 4. Model Denial of Service, 5. Supply Chain Vulnerabilities, 6. Sensitive Information Disclosure, 7. Insecure Plugin Design, 8. Excessive Agency, 9. Overreliance, 10. Model Theft. **Enterprise:** every AI product needs specific mitigations for each. Prompt injection → input sanitization + structural delimiters. Output handling → validate before executing. Model DoS → rate limit + cost caps. Sensitive info → filter secrets from prompts and responses. This is now the security baseline expected by enterprise procurement.

### Q19. Cold starts in serverless AI deployments. How to mitigate? ⭐⭐⭐
Cold start: first request to a scaled-to-zero function takes seconds. **Causes:** container pull, runtime init, dependency load, model load (if serving models). **Mitigations:** (1) Minimum instances (keep 1+ warm — costs but eliminates cold start), (2) Provisioned concurrency (AWS Lambda), (3) Smaller images (multi-stage Docker), (4) Lazy imports (defer heavy imports until first request), (5) Preload cache/models in startup script, (6) Use edge platforms (Cloudflare Workers) with faster starts. **For LLM APIs:** cold start rarely matters (LLM latency dominates). For model serving: cold start is huge (multi-second model load). Different strategies apply.

### Q20. Explain immutable infrastructure. Why does it matter? ⭐⭐⭐
Never modify running instances. Deploy new instances, decommission old. Every change = new deployment. **Pros:** reproducible, no config drift, easy rollback, auditable. **Cons:** slower for small changes, higher deployment overhead. **Enabled by:** containers (immutable images), IaC (declarative infrastructure), CI/CD (automated deployment). **Contrast:** mutable infrastructure = SSH into server, patch, restart = works but drift accumulates, "the pet server that nobody understands anymore." **Enterprise standard:** immutable everywhere.

### Q21. Explain CDN and edge deployment for AI. When useful? ⭐⭐⭐
CDN: content delivery network, caches at geographic edge close to users. **For AI:** less relevant for LLM responses (dynamic, unique per user). More relevant for: (1) static assets (frontend), (2) cached responses (semantic cache at edge — Cloudflare AI Gateway), (3) inference at edge for small models (Cloudflare Workers AI, Vercel Edge). **When useful:** global user base, latency-sensitive apps, high cache hit rate scenarios. **When NOT useful:** small user base, always-unique responses, complex per-user logic.

### Q22. Backup and disaster recovery for AI systems. RTO/RPO for AI? ⭐⭐⭐⭐
**RTO (Recovery Time Objective):** how quickly you recover. **RPO (Recovery Point Objective):** how much data you can lose. **For AI systems, what to back up:** (1) User data + conversation history (RPO 1h, RTO 4h typical), (2) Fine-tuned model weights (RPO daily, RTO 24h — models rarely change), (3) Prompts/config (RPO immediate — in git), (4) Vector DB indexes (RPO daily, RTO 4h — rebuild if catastrophic), (5) Golden eval datasets (RPO monthly, RTO 24h). **Not to back up:** LLM provider APIs (their responsibility), cache (rebuilds naturally). Test disaster recovery quarterly — most orgs discover their DR plan doesn't work under stress.

### Q23. Explain health checks properly. Liveness vs readiness vs startup probes. ⭐⭐⭐
**Liveness:** is the process alive? Fails → restart container. Simple check (HTTP 200 from /health). **Readiness:** is the process ready to serve traffic? Fails → remove from load balancer. Deeper check (DB connected, dependencies available). **Startup:** is startup complete? Prevents liveness kills during slow startup (model loading). **For AI systems:** readiness should check LLM provider connectivity (or fail-fast to fallbacks). Startup probe critical for services loading models (30-60s startup common). **Anti-pattern:** using `/health` returning 200 for everything. Meaningless. Real health checks verify actual functionality.

### Q24. Explain feature flags. Why critical for AI product development? ⭐⭐⭐
Feature flags: runtime toggles for features. Enable per-user, per-tenant, per-percentage. Tools: LaunchDarkly, Unleash, Split, Flagsmith. **For AI specifically:** (1) A/B test prompts without deploys, (2) Gradual rollout of new models (5% → 25% → 100%), (3) Emergency kill switch for problematic features, (4) Per-tenant feature availability (enterprise gets X, free tier gets Y), (5) Debugging (enable verbose logging for specific users). **Enterprise standard:** every risky change wrapped in a flag. Deployments are always low-risk because change activation is separate from deployment.

### Q25. Explain queue-based architecture for async LLM workloads. ⭐⭐⭐
Sync: user waits for LLM response (5-30s). Bad UX at scale. **Async:** user submits request → enters queue → worker processes → user gets notified. **Tools:** Celery + Redis, RQ, SQS, Cloud Tasks. **For AI:** ideal for (1) long-running tasks (document processing, batch analysis), (2) rate-limited providers (queue smooths bursts), (3) cost optimization (batch cheap queries with expensive ones). **Not for:** interactive chat (users expect immediate). **Enterprise pattern:** hybrid — sync API for interactive, async queue for batch. Users see progress in UI (polling or WebSocket).

### Q26. Explain zero-downtime deployment for stateful AI services. ⭐⭐⭐⭐
**Stateless services:** easy — new instances start, old stop, health checks handle. **Stateful:** WebSocket connections, in-flight LLM streams, agent executions in progress. **Techniques:** (1) Graceful shutdown (SIGTERM handler drains in-flight requests before terminating), (2) Load balancer removes instance from pool BEFORE stopping, (3) Connection draining (30-60s to finish current requests), (4) State externalization (checkpoint agent state to Redis, resume on new instance), (5) Blue-green with session migration. **For AI agents:** LangGraph checkpoints enable pause-on-old-instance, resume-on-new-instance.

### Q27. Explain enterprise "supply chain security" for AI (SBOM, dependencies, models). ⭐⭐⭐⭐
Supply chain: everything that goes into your app (libraries, models, base images, dependencies). **Attacks:** poisoned packages (event-stream), typosquatting, malicious models on Hugging Face. **Defenses:** (1) SBOM generation (Software Bill of Materials — what's in your artifact), (2) Dependency scanning (Snyk, Dependabot), (3) Container scanning (Trivy, Grype), (4) Model provenance (only download from verified publishers, verify hashes), (5) Signed artifacts (Sigstore, cosign), (6) Reproducible builds. **Enterprise requirement:** SBOMs increasingly mandatory for federal contracts, growing in private sector. AI supply chain includes model files — often overlooked but a real attack surface.
