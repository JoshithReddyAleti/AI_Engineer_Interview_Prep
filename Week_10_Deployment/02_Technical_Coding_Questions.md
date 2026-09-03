# 💻 Week 10 — Technical / Coding Questions

> **Focus:** Build production FastAPI with streaming, circuit breakers, cost-based rate limiter, LLM gateway with fallbacks, semantic cache, health checks, graceful shutdown, canary logic
>
> **How to use:** These are live-code questions for senior AI engineer roles. Every solution includes enterprise features — cost, safety, observability — not just the happy path.

---

## Q1. Build a Production-Grade FastAPI LLM Endpoint With Streaming ⭐⭐⭐⭐

**Prompt:** Implement a production LLM endpoint. Requirements: async streaming (SSE), Pydantic validation, rate limiting, authentication, structured logging, cost tracking, error handling with retries.

**Solution:**

```python
from fastapi import FastAPI, HTTPException, Depends, Request
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field, field_validator
from contextlib import asynccontextmanager
import structlog
import time
import uuid
import asyncio
from typing import AsyncIterator

# Structured logging
logger = structlog.get_logger()

# --- Pydantic models ---
class ChatRequest(BaseModel):
    messages: list[dict] = Field(..., min_length=1, max_length=50)
    model: str = Field(default="gpt-4o-mini")
    max_tokens: int = Field(default=1000, ge=1, le=4000)
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    stream: bool = Field(default=True)
    
    @field_validator("messages")
    @classmethod
    def validate_messages(cls, v):
        for msg in v:
            if "role" not in msg or "content" not in msg:
                raise ValueError("Each message must have 'role' and 'content'")
            if msg["role"] not in ("system", "user", "assistant"):
                raise ValueError(f"Invalid role: {msg['role']}")
            if len(msg["content"]) > 10000:
                raise ValueError("Message content too long")
        return v

class ErrorResponse(BaseModel):
    error: str
    error_type: str
    request_id: str
    retry_after: int | None = None

# --- Auth dependency ---
async def verify_api_key(request: Request) -> dict:
    api_key = request.headers.get("X-API-Key")
    if not api_key:
        raise HTTPException(status_code=401, detail="Missing API key")
    
    # In production: hash and compare against DB
    user_info = await lookup_api_key(api_key)
    if not user_info:
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    return user_info

# --- Rate limit dependency ---
async def check_rate_limit(user_info: dict = Depends(verify_api_key)) -> dict:
    user_id = user_info["user_id"]
    
    # Cost-based rate limit (see Q3)
    can_proceed, reason = await cost_limiter.check(user_id)
    if not can_proceed:
        raise HTTPException(
            status_code=429,
            detail={"error": reason, "retry_after": 3600},
            headers={"Retry-After": "3600"},
        )
    
    return user_info

# --- App lifecycle ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("startup", event="app_starting")
    app.state.llm_client = create_llm_client()
    app.state.start_time = time.time()
    yield
    # Shutdown
    logger.info("shutdown", event="app_stopping")
    await app.state.llm_client.close()

app = FastAPI(
    title="Production LLM API",
    version="1.0.0",
    lifespan=lifespan,
)

# --- Request ID middleware ---
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    request.state.request_id = request_id
    
    logger.info(
        "request_start",
        request_id=request_id,
        method=request.method,
        path=request.url.path,
    )
    
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    response.headers["X-Request-ID"] = request_id
    
    logger.info(
        "request_end",
        request_id=request_id,
        status=response.status_code,
        duration_ms=int(duration * 1000),
    )
    
    return response

# --- Health checks ---
@app.get("/health/liveness")
async def liveness():
    """Is the process alive? Simple check."""
    return {"status": "alive"}

@app.get("/health/readiness")
async def readiness():
    """Is the process ready to serve traffic? Check dependencies."""
    checks = {"llm_provider": False, "database": False, "cache": False}
    
    try:
        await asyncio.wait_for(app.state.llm_client.ping(), timeout=2.0)
        checks["llm_provider"] = True
    except Exception:
        pass
    
    try:
        await asyncio.wait_for(db.execute("SELECT 1"), timeout=1.0)
        checks["database"] = True
    except Exception:
        pass
    
    try:
        await asyncio.wait_for(redis.ping(), timeout=1.0)
        checks["cache"] = True
    except Exception:
        pass
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    return {"status": "ready" if all_healthy else "degraded", "checks": checks}

# --- Main endpoint ---
@app.post("/v1/chat/completions")
async def chat_completion(
    request: ChatRequest,
    http_request: Request,
    user_info: dict = Depends(check_rate_limit),
):
    request_id = http_request.state.request_id
    
    logger.info(
        "chat_request",
        request_id=request_id,
        user_id=user_info["user_id"],
        model=request.model,
        message_count=len(request.messages),
    )
    
    if request.stream:
        return StreamingResponse(
            stream_completion(request, user_info, request_id),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "X-Accel-Buffering": "no",  # Disable nginx buffering
            },
        )
    else:
        return await non_stream_completion(request, user_info, request_id)

async def stream_completion(
    request: ChatRequest,
    user_info: dict,
    request_id: str,
) -> AsyncIterator[str]:
    """SSE streaming response for LLM tokens."""
    total_tokens = 0
    start_time = time.time()
    
    try:
        # Call LLM with circuit breaker + retry (see Q4)
        async for chunk in llm_gateway.stream(
            messages=request.messages,
            model=request.model,
            max_tokens=request.max_tokens,
            temperature=request.temperature,
        ):
            total_tokens += chunk.token_count
            # SSE format
            yield f"data: {chunk.to_json()}\n\n"
        
        yield "data: [DONE]\n\n"
        
    except Exception as e:
        logger.error(
            "stream_error",
            request_id=request_id,
            error=str(e),
        )
        error_response = ErrorResponse(
            error=str(e),
            error_type=type(e).__name__,
            request_id=request_id,
        )
        yield f"data: {error_response.model_dump_json()}\n\n"
    
    finally:
        # Cost tracking
        duration = time.time() - start_time
        cost = calculate_cost(request.model, total_tokens)
        await cost_tracker.record(
            user_id=user_info["user_id"],
            model=request.model,
            tokens=total_tokens,
            cost=cost,
            duration_ms=int(duration * 1000),
        )
        
        logger.info(
            "chat_complete",
            request_id=request_id,
            user_id=user_info["user_id"],
            tokens=total_tokens,
            cost=cost,
            duration_ms=int(duration * 1000),
        )
```

**Enterprise features demonstrated:**
- Async streaming via SSE (production LLM pattern)
- Pydantic validation with `field_validator` for security
- Layered dependency injection (auth → rate limit → handler)
- Structured logging with request IDs (correlation across systems)
- Liveness vs readiness health checks (K8s-compatible)
- Cost tracking after every request
- Graceful error handling in stream (client gets error message, not connection drop)
- Middleware for request ID (traceability)
- Lifespan for proper startup/shutdown

---

## Q2. Implement a Production Circuit Breaker With Half-Open State ⭐⭐⭐⭐

**Prompt:** Build a circuit breaker for LLM API calls. Full state machine (CLOSED, OPEN, HALF-OPEN). Per-provider. Async-native.

**Solution:**

```python
from enum import Enum
from dataclasses import dataclass, field
from collections import deque
import asyncio
import time
from typing import Callable, Any

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreakerOpenError(Exception):
    """Raised when circuit is open — fast failure."""
    pass

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5        # Failures to open circuit
    success_threshold: int = 2        # Successes in half-open to close
    timeout_seconds: float = 60.0     # Cool-down before half-open
    window_seconds: float = 60.0      # Window for counting failures
    excluded_exceptions: tuple = ()   # Don't count these (e.g., 4xx client errors)

class CircuitBreaker:
    """
    Production-grade circuit breaker with full state machine.
    Async-native. Per-instance state.
    """
    
    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        
        self._state = CircuitState.CLOSED
        self._failures: deque = deque()  # Timestamps of recent failures
        self._successes_in_half_open = 0
        self._opened_at: float = 0
        self._half_open_lock = asyncio.Lock()  # Only one probe at a time
        self._state_lock = asyncio.Lock()
        
        # Observability
        self._state_change_callbacks: list[Callable] = []
    
    def on_state_change(self, callback: Callable):
        """Register callback for state changes — for alerting."""
        self._state_change_callbacks.append(callback)
    
    async def _transition(self, new_state: CircuitState):
        old_state = self._state
        self._state = new_state
        
        # Notify observers (alerts, metrics)
        for callback in self._state_change_callbacks:
            try:
                await callback(self.name, old_state, new_state)
            except Exception:
                pass  # Never let callback failures break the breaker
    
    def _cleanup_failures(self):
        """Remove failures outside the rolling window."""
        cutoff = time.time() - self.config.window_seconds
        while self._failures and self._failures[0] < cutoff:
            self._failures.popleft()
    
    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Execute function with circuit breaker protection."""
        await self._check_state()
        
        if self._state == CircuitState.OPEN:
            raise CircuitBreakerOpenError(
                f"Circuit breaker '{self.name}' is OPEN"
            )
        
        if self._state == CircuitState.HALF_OPEN:
            # Only one probe request at a time
            async with self._half_open_lock:
                # Re-check state (may have changed while waiting)
                if self._state != CircuitState.HALF_OPEN:
                    return await self.call(func, *args, **kwargs)
                return await self._execute_and_track(func, *args, **kwargs)
        
        # CLOSED state
        return await self._execute_and_track(func, *args, **kwargs)
    
    async def _check_state(self):
        """Check if OPEN circuit should transition to HALF_OPEN."""
        async with self._state_lock:
            if self._state == CircuitState.OPEN:
                if time.time() - self._opened_at >= self.config.timeout_seconds:
                    await self._transition(CircuitState.HALF_OPEN)
                    self._successes_in_half_open = 0
    
    async def _execute_and_track(self, func: Callable, *args, **kwargs) -> Any:
        try:
            result = await func(*args, **kwargs)
            await self._record_success()
            return result
        except self.config.excluded_exceptions:
            raise  # Don't count these against circuit
        except Exception as e:
            await self._record_failure()
            raise
    
    async def _record_success(self):
        async with self._state_lock:
            if self._state == CircuitState.HALF_OPEN:
                self._successes_in_half_open += 1
                if self._successes_in_half_open >= self.config.success_threshold:
                    await self._transition(CircuitState.CLOSED)
                    self._failures.clear()
            elif self._state == CircuitState.CLOSED:
                # Success in closed state is normal, no action
                pass
    
    async def _record_failure(self):
        async with self._state_lock:
            self._failures.append(time.time())
            self._cleanup_failures()
            
            if self._state == CircuitState.HALF_OPEN:
                # Failure in half-open → back to open
                await self._transition(CircuitState.OPEN)
                self._opened_at = time.time()
            elif self._state == CircuitState.CLOSED:
                if len(self._failures) >= self.config.failure_threshold:
                    await self._transition(CircuitState.OPEN)
                    self._opened_at = time.time()
    
    @property
    def state(self) -> CircuitState:
        return self._state
    
    @property
    def failure_count(self) -> int:
        self._cleanup_failures()
        return len(self._failures)


# Usage
openai_breaker = CircuitBreaker(
    name="openai",
    config=CircuitBreakerConfig(
        failure_threshold=5,
        timeout_seconds=60,
        excluded_exceptions=(ValueError,),  # Don't count validation errors
    ),
)

# Alert on state changes
async def alert_on_state_change(name, old_state, new_state):
    if new_state == CircuitState.OPEN:
        await send_pager_alert(f"Circuit {name} OPEN — production degraded")
    elif new_state == CircuitState.CLOSED and old_state != CircuitState.CLOSED:
        await send_slack_message(f"Circuit {name} recovered")

openai_breaker.on_state_change(alert_on_state_change)

# Wrap LLM calls
async def call_openai(prompt: str):
    return await openai_breaker.call(openai.chat.completions.create, ...)
```

**Enterprise features:**
- Full state machine (not just open/closed)
- Rolling window for failure counting (recent failures matter more)
- Half-open probe with lock (single request tests recovery)
- State change callbacks for alerts (PagerDuty, Slack)
- Excluded exceptions (client errors don't count against circuit)
- Async-native (works with FastAPI, aiohttp, etc.)

---

## Q3. Implement Cost-Based Rate Limiting With Redis ⭐⭐⭐⭐

**Prompt:** Build a rate limiter that limits by DOLLAR COST, not request count. Per-user daily budget. Track actual costs. Enforce hard cutoffs.

**Solution:**

```python
from redis.asyncio import Redis
from dataclasses import dataclass
from datetime import datetime, timezone
import json

@dataclass
class CostLimits:
    per_request_max: float = 1.0    # Reject if single request would cost > $1
    per_user_daily: float = 10.0    # Reject if user would exceed $10/day
    per_user_monthly: float = 200.0  # Reject if user would exceed $200/month
    per_tenant_monthly: float = 5000.0  # Reject if tenant would exceed $5K/month

@dataclass
class CostCheckResult:
    allowed: bool
    reason: str = ""
    current_daily: float = 0.0
    current_monthly: float = 0.0
    limit_daily: float = 0.0
    limit_monthly: float = 0.0

class CostBasedRateLimiter:
    """
    Cost-based rate limiter for LLM APIs.
    Limits by actual dollar cost, not request count.
    Multi-level: per-request, per-user daily/monthly, per-tenant monthly.
    """
    
    def __init__(self, redis: Redis, limits: CostLimits):
        self.redis = redis
        self.limits = limits
    
    def _today_key(self) -> str:
        return datetime.now(timezone.utc).strftime("%Y-%m-%d")
    
    def _this_month_key(self) -> str:
        return datetime.now(timezone.utc).strftime("%Y-%m")
    
    async def check(
        self,
        user_id: str,
        tenant_id: str,
        estimated_cost: float,
    ) -> CostCheckResult:
        """Pre-check before executing LLM call."""
        
        # Check 1: Per-request maximum
        if estimated_cost > self.limits.per_request_max:
            return CostCheckResult(
                allowed=False,
                reason=f"Request cost ${estimated_cost:.2f} exceeds per-request limit ${self.limits.per_request_max:.2f}",
            )
        
        # Check 2: User daily
        user_daily_key = f"cost:user:{user_id}:daily:{self._today_key()}"
        current_daily = float(await self.redis.get(user_daily_key) or 0)
        if current_daily + estimated_cost > self.limits.per_user_daily:
            return CostCheckResult(
                allowed=False,
                reason=f"Would exceed user daily limit (${self.limits.per_user_daily})",
                current_daily=current_daily,
                limit_daily=self.limits.per_user_daily,
            )
        
        # Check 3: User monthly
        user_monthly_key = f"cost:user:{user_id}:monthly:{self._this_month_key()}"
        current_monthly = float(await self.redis.get(user_monthly_key) or 0)
        if current_monthly + estimated_cost > self.limits.per_user_monthly:
            return CostCheckResult(
                allowed=False,
                reason=f"Would exceed user monthly limit (${self.limits.per_user_monthly})",
                current_monthly=current_monthly,
                limit_monthly=self.limits.per_user_monthly,
            )
        
        # Check 4: Tenant monthly
        tenant_monthly_key = f"cost:tenant:{tenant_id}:monthly:{self._this_month_key()}"
        current_tenant = float(await self.redis.get(tenant_monthly_key) or 0)
        if current_tenant + estimated_cost > self.limits.per_tenant_monthly:
            return CostCheckResult(
                allowed=False,
                reason=f"Would exceed tenant monthly limit (${self.limits.per_tenant_monthly})",
            )
        
        return CostCheckResult(
            allowed=True,
            current_daily=current_daily,
            current_monthly=current_monthly,
            limit_daily=self.limits.per_user_daily,
            limit_monthly=self.limits.per_user_monthly,
        )
    
    async def record(
        self,
        user_id: str,
        tenant_id: str,
        actual_cost: float,
    ):
        """Record actual cost after LLM call completes."""
        today = self._today_key()
        month = self._this_month_key()
        
        # Atomic pipeline
        pipe = self.redis.pipeline()
        
        # User daily
        pipe.incrbyfloat(f"cost:user:{user_id}:daily:{today}", actual_cost)
        pipe.expire(f"cost:user:{user_id}:daily:{today}", 86400 * 2)  # 2-day TTL
        
        # User monthly
        pipe.incrbyfloat(f"cost:user:{user_id}:monthly:{month}", actual_cost)
        pipe.expire(f"cost:user:{user_id}:monthly:{month}", 86400 * 35)  # 35-day TTL
        
        # Tenant monthly
        pipe.incrbyfloat(f"cost:tenant:{tenant_id}:monthly:{month}", actual_cost)
        pipe.expire(f"cost:tenant:{tenant_id}:monthly:{month}", 86400 * 35)
        
        # Aggregate metrics
        pipe.incrbyfloat("cost:global:total", actual_cost)
        
        await pipe.execute()
    
    async def get_usage(self, user_id: str, tenant_id: str) -> dict:
        """Get current usage for dashboard."""
        today = self._today_key()
        month = self._this_month_key()
        
        pipe = self.redis.pipeline()
        pipe.get(f"cost:user:{user_id}:daily:{today}")
        pipe.get(f"cost:user:{user_id}:monthly:{month}")
        pipe.get(f"cost:tenant:{tenant_id}:monthly:{month}")
        results = await pipe.execute()
        
        return {
            "user_daily": float(results[0] or 0),
            "user_monthly": float(results[1] or 0),
            "tenant_monthly": float(results[2] or 0),
            "limits": {
                "user_daily": self.limits.per_user_daily,
                "user_monthly": self.limits.per_user_monthly,
                "tenant_monthly": self.limits.per_tenant_monthly,
            },
        }
    
    def estimate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        """Estimate cost based on model pricing."""
        # Pricing per 1M tokens (update from provider)
        pricing = {
            "gpt-4o": {"input": 2.50, "output": 10.00},
            "gpt-4o-mini": {"input": 0.15, "output": 0.60},
            "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
            "claude-3-5-haiku": {"input": 0.80, "output": 4.00},
        }
        p = pricing.get(model, {"input": 5.0, "output": 15.0})
        return (input_tokens * p["input"] + output_tokens * p["output"]) / 1_000_000


# Usage in FastAPI
limiter = CostBasedRateLimiter(
    redis=redis_client,
    limits=CostLimits(
        per_request_max=0.50,
        per_user_daily=5.0,
        per_user_monthly=100.0,
        per_tenant_monthly=10000.0,
    ),
)

@app.post("/v1/chat")
async def chat(request: ChatRequest, user: dict = Depends(auth)):
    # Estimate cost before calling LLM
    est_cost = limiter.estimate_cost(
        model=request.model,
        input_tokens=count_tokens(request.messages),
        output_tokens=request.max_tokens,
    )
    
    check = await limiter.check(
        user_id=user["id"],
        tenant_id=user["tenant_id"],
        estimated_cost=est_cost,
    )
    if not check.allowed:
        raise HTTPException(429, detail=check.reason)
    
    # Call LLM
    response = await call_llm(request)
    
    # Record actual cost
    actual_cost = calculate_actual_cost(response)
    await limiter.record(
        user_id=user["id"],
        tenant_id=user["tenant_id"],
        actual_cost=actual_cost,
    )
    
    return response
```

**Enterprise features:**
- Multi-level limits (request, user daily, user monthly, tenant monthly)
- Atomic Redis operations (pipeline)
- TTL on counters (auto-cleanup old data)
- Estimate before call, record after (accurate cost tracking)
- Dashboard-ready usage endpoint

---

## Q4-Q17: Additional Coding Challenges (Condensed)

### Q4. Build an LLM Gateway With Multi-Provider Fallback ⭐⭐⭐⭐
Wrap OpenAI + Anthropic + Google. Primary → fallback chain. Per-provider circuit breakers. Retry with exponential backoff + jitter. Unified response format. Cost tracking per provider. Automatic failover if primary is OPEN.

### Q5. Implement Semantic Caching With Redis Vector Search ⭐⭐⭐⭐
On query, compute embedding, search Redis vector index for similar (threshold 0.95). If hit, return cached response. If miss, call LLM, store (embedding, query, response). TTL 24h. Per-tenant namespaces (never share across tenants). Track cache hit rate.

### Q6. Build Retry Logic With Exponential Backoff + Jitter ⭐⭐⭐
Retry transient errors (rate limit, timeout, 5xx). Exponential backoff: `wait = base * 2^attempt`. Add jitter: `wait *= random(0.5, 1.5)` (prevents thundering herd). Max retries: 3. Don't retry: 4xx (except 429), validation errors. Log every retry with attempt number.

### Q7. Implement Graceful Shutdown for FastAPI ⭐⭐⭐
On SIGTERM: (1) stop accepting new requests (health check returns 503), (2) drain in-flight requests (wait up to 30s), (3) close DB connections, (4) checkpoint any in-progress agent state, (5) close LLM client. Load balancer sees 503 → removes from pool before pod terminates.

### Q8. Build Blue-Green Deployment Logic ⭐⭐⭐
Two environments (blue = current, green = new). Deploy to green. Verify health checks + smoke tests. Switch load balancer to green. Keep blue as standby (24h). Rollback = switch back. Automated via CI/CD script.

### Q9. Implement Canary Deployment With Auto-Rollback ⭐⭐⭐⭐
Deploy new version to N% of traffic (5% → 25% → 50% → 100%). Between stages, check: error rate (< baseline + 20%), latency (p95 < baseline + 20%), quality score (LLM-judge, not < baseline - 5%). If any fails, auto-rollback. Otherwise, proceed to next stage.

### Q10. Build a JWT Auth System With Refresh Tokens ⭐⭐⭐
Short-lived access token (15 min). Long-lived refresh token (7 days, stored in HTTP-only cookie). On expiry, client uses refresh token to get new access token. Refresh token rotation on use (prevents stolen refresh reuse). Blacklist for logout.

### Q11. Implement Structured Logging With Correlation IDs ⭐⭐⭐
Use structlog. Every log entry has: timestamp, level, message, request_id, user_id, tenant_id, service_name. Correlation ID passed via headers across services. JSON output. Sensitive fields (passwords, keys) filtered. Log levels: DEBUG (dev), INFO (normal), WARNING (issue), ERROR (failure), CRITICAL (outage).

### Q12. Build Health Check Endpoints (Liveness + Readiness + Startup) ⭐⭐⭐
Liveness: `/health/live` — returns 200 if process alive. Readiness: `/health/ready` — returns 200 if ALL dependencies healthy (DB, cache, LLM provider). Startup: `/health/startup` — returns 200 when startup complete (model loaded). Different endpoints for different K8s probes.

### Q13. Implement Feature Flag System ⭐⭐⭐
Flags stored in Redis. Rules: user_id in list, tenant_id in list, percentage rollout (hash-based, sticky per user), date range. API: `flag_enabled(name, user_id) -> bool`. Cache per-request (avoid repeated Redis calls). Dashboard for flag management.

### Q14. Build Load Testing Framework for LLM APIs ⭐⭐⭐
Simulate concurrent users. Realistic distribution of query sizes (small/medium/large). Track: RPS achieved, p50/p95/p99 latency, error rate, cost per test run. Different from web load testing: track TOKENS/sec not just requests/sec. Alert on cost overruns during test.

### Q15. Implement Distributed Rate Limiting Across Multiple Nodes ⭐⭐⭐⭐
Multiple app instances behind load balancer. Rate limits must be consistent across all. Solution: Redis-based counter (shared state). Lua script for atomic check-and-increment. Handle Redis failure (fallback to local counter with degraded accuracy). Latency: <5ms overhead per request.

### Q16. Build Prompt Injection Defense at API Layer ⭐⭐⭐⭐
Input sanitization: strip common injection markers. Detection: classifier for known patterns. Structural delimiter: user input in XML tags for LLM. Output validation: check response doesn't leak system prompt. Layered: input filter + prompt structure + output filter.

### Q17. Implement Zero-Downtime Database Migrations ⭐⭐⭐
Rule: backwards-compatible migrations only. Add column (nullable), deploy code that writes both old + new, backfill, deploy code that reads new, later drop old. Never break the running version. Use tools: Alembic for Python, Prisma migrate. Test on staging with prod-scale data.
