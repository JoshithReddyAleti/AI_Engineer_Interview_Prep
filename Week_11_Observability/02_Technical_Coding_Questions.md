# 💻 Week 11 — Technical / Coding Questions

> **Focus:** Structured logger with context propagation, OpenTelemetry instrumentation, RAG trace, agent trajectory logger, cost tracker, drift detector, PII redaction, alerting engine
>
> **How to use:** These are the coding questions for observability roles at OpenAI, Anthropic, Datadog, and every enterprise AI team. Every solution includes production concerns.

---

## Q1. Build a Structured Logger With Context Propagation ⭐⭐⭐⭐

**Prompt:** Implement a structured JSON logger that automatically includes trace_id, user_id, tenant_id in every log entry. Async-safe. Includes PII redaction.

**Solution:**

```python
import json
import logging
import re
import sys
import time
import uuid
from contextvars import ContextVar
from typing import Any

# Context variables — request-scoped, async-safe
trace_id_var: ContextVar[str] = ContextVar("trace_id", default="")
user_id_var: ContextVar[str] = ContextVar("user_id", default="")
tenant_id_var: ContextVar[str] = ContextVar("tenant_id", default="")
request_id_var: ContextVar[str] = ContextVar("request_id", default="")


class PIIRedactor:
    """Redact common PII patterns before logging."""
    
    EMAIL_PATTERN = re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b')
    PHONE_PATTERN = re.compile(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b')
    SSN_PATTERN = re.compile(r'\b\d{3}-\d{2}-\d{4}\b')
    CREDIT_CARD_PATTERN = re.compile(r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b')
    API_KEY_PATTERN = re.compile(r'\b(sk-|pk_|xoxb-)[a-zA-Z0-9-]{20,}\b')
    
    @classmethod
    def redact(cls, text: str) -> str:
        """Redact PII from text."""
        if not isinstance(text, str):
            return text
        text = cls.EMAIL_PATTERN.sub("[EMAIL_REDACTED]", text)
        text = cls.PHONE_PATTERN.sub("[PHONE_REDACTED]", text)
        text = cls.SSN_PATTERN.sub("[SSN_REDACTED]", text)
        text = cls.CREDIT_CARD_PATTERN.sub("[CARD_REDACTED]", text)
        text = cls.API_KEY_PATTERN.sub("[APIKEY_REDACTED]", text)
        return text
    
    @classmethod
    def redact_dict(cls, data: dict) -> dict:
        """Recursively redact PII in dict values."""
        redacted = {}
        for key, value in data.items():
            # Redact sensitive keys entirely
            if key.lower() in {"password", "api_key", "token", "secret", "authorization"}:
                redacted[key] = "[REDACTED]"
            elif isinstance(value, str):
                redacted[key] = cls.redact(value)
            elif isinstance(value, dict):
                redacted[key] = cls.redact_dict(value)
            elif isinstance(value, list):
                redacted[key] = [cls.redact_dict(v) if isinstance(v, dict) 
                                 else cls.redact(v) if isinstance(v, str) 
                                 else v for v in value]
            else:
                redacted[key] = value
        return redacted


class StructuredLogger:
    """
    Production-grade structured logger with context propagation.
    
    Features:
    - JSON output
    - Automatic context injection (trace_id, user_id, tenant_id)
    - PII redaction on all string values
    - Async-safe via contextvars
    - Configurable log level
    - Compatible with Loki, Elasticsearch, Datadog, etc.
    """
    
    def __init__(self, service_name: str, level: str = "INFO", redact_pii: bool = True):
        self.service_name = service_name
        self.level = getattr(logging, level.upper())
        self.redact_pii = redact_pii
    
    def _base_context(self) -> dict:
        """Standard fields in every log entry."""
        return {
            "timestamp": time.time(),
            "timestamp_iso": time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime()),
            "service": self.service_name,
            "trace_id": trace_id_var.get(),
            "user_id": user_id_var.get(),
            "tenant_id": tenant_id_var.get(),
            "request_id": request_id_var.get(),
        }
    
    def _log(self, level: str, event: str, **fields):
        """Emit a log entry."""
        if getattr(logging, level) < self.level:
            return
        
        log_entry = self._base_context()
        log_entry["level"] = level
        log_entry["event"] = event
        log_entry.update(fields)
        
        # Redact PII
        if self.redact_pii:
            log_entry = PIIRedactor.redact_dict(log_entry)
        
        # Emit as JSON to stdout (or stderr for errors)
        stream = sys.stderr if level in ("ERROR", "CRITICAL") else sys.stdout
        print(json.dumps(log_entry, default=str), file=stream, flush=True)
    
    def debug(self, event: str, **fields):
        self._log("DEBUG", event, **fields)
    
    def info(self, event: str, **fields):
        self._log("INFO", event, **fields)
    
    def warning(self, event: str, **fields):
        self._log("WARNING", event, **fields)
    
    def error(self, event: str, **fields):
        self._log("ERROR", event, **fields)
    
    def critical(self, event: str, **fields):
        self._log("CRITICAL", event, **fields)


# Context management
class LogContext:
    """Context manager for setting log context in a scope."""
    
    def __init__(self, **kwargs):
        self.kwargs = kwargs
        self.tokens = {}
    
    def __enter__(self):
        # Set each context var
        if "trace_id" in self.kwargs:
            self.tokens["trace_id"] = trace_id_var.set(self.kwargs["trace_id"])
        if "user_id" in self.kwargs:
            self.tokens["user_id"] = user_id_var.set(self.kwargs["user_id"])
        if "tenant_id" in self.kwargs:
            self.tokens["tenant_id"] = tenant_id_var.set(self.kwargs["tenant_id"])
        if "request_id" in self.kwargs:
            self.tokens["request_id"] = request_id_var.set(self.kwargs["request_id"])
        return self
    
    def __exit__(self, *args):
        # Reset all
        for var_name, token in self.tokens.items():
            {"trace_id": trace_id_var, "user_id": user_id_var,
             "tenant_id": tenant_id_var, "request_id": request_id_var}[var_name].reset(token)


# FastAPI middleware
from fastapi import Request

async def logging_middleware(request: Request, call_next):
    """Middleware that sets logging context for the request."""
    trace_id = request.headers.get("X-Trace-ID") or str(uuid.uuid4())
    request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    
    # Extract from JWT (simplified)
    user_id = getattr(request.state, "user_id", "")
    tenant_id = getattr(request.state, "tenant_id", "")
    
    with LogContext(
        trace_id=trace_id,
        user_id=user_id,
        tenant_id=tenant_id,
        request_id=request_id,
    ):
        response = await call_next(request)
    
    response.headers["X-Trace-ID"] = trace_id
    return response


# Usage
logger = StructuredLogger("chat-service")

@app.post("/chat")
async def chat_endpoint(request: ChatRequest):
    logger.info("chat_request_received", model=request.model, message_count=len(request.messages))
    
    try:
        response = await call_llm(request)
        logger.info("chat_completed", tokens=response.tokens, cost=response.cost)
        return response
    except Exception as e:
        logger.error("chat_failed", error=str(e), error_type=type(e).__name__)
        raise
```

**Enterprise features demonstrated:**
- Async-safe context propagation via contextvars
- PII redaction with common patterns
- Sensitive key detection (password, api_key, token)
- JSON output for machine parsing
- Middleware integration for auto-context
- Level filtering
- Trace ID correlation across systems

---

## Q2. Instrument an LLM Call With OpenTelemetry ⭐⭐⭐⭐

**Prompt:** Wrap an LLM API call with OpenTelemetry spans. Capture: prompt, response, tokens, cost, latency. Follow GenAI semantic conventions.

**Solution:**

```python
from opentelemetry import trace, metrics
from opentelemetry.trace import Status, StatusCode
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
import time

# Setup (once at app startup)
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(tracer_provider)

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Metrics (bounded cardinality — no user_id!)
llm_calls_counter = meter.create_counter(
    "llm_calls_total",
    description="Total LLM calls",
)
llm_latency_histogram = meter.create_histogram(
    "llm_latency_seconds",
    description="LLM call latency",
    unit="s",
)
llm_tokens_histogram = meter.create_histogram(
    "llm_tokens",
    description="Tokens used per call",
)
llm_cost_counter = meter.create_counter(
    "llm_cost_dollars_total",
    description="Total cost in dollars",
)


async def instrumented_llm_call(
    provider: str,
    model: str,
    messages: list,
    max_tokens: int = 1000,
    temperature: float = 0.7,
) -> dict:
    """LLM call with full OpenTelemetry instrumentation."""
    
    # Start span (follows GenAI semantic conventions)
    with tracer.start_as_current_span(
        name=f"llm.{provider}.chat",
        attributes={
            # GenAI semantic conventions
            "gen_ai.system": provider,
            "gen_ai.request.model": model,
            "gen_ai.request.max_tokens": max_tokens,
            "gen_ai.request.temperature": temperature,
            "gen_ai.request.message_count": len(messages),
        }
    ) as span:
        start_time = time.time()
        
        try:
            # Actual LLM call
            response = await call_llm_provider(
                provider=provider,
                model=model,
                messages=messages,
                max_tokens=max_tokens,
                temperature=temperature,
            )
            
            duration = time.time() - start_time
            
            # Record response attributes
            span.set_attributes({
                "gen_ai.usage.input_tokens": response.usage.prompt_tokens,
                "gen_ai.usage.output_tokens": response.usage.completion_tokens,
                "gen_ai.usage.total_tokens": response.usage.total_tokens,
                "gen_ai.response.finish_reasons": [response.choices[0].finish_reason],
                "gen_ai.response.id": response.id,
            })
            
            # Calculate cost
            cost = calculate_cost(
                model=model,
                input_tokens=response.usage.prompt_tokens,
                output_tokens=response.usage.completion_tokens,
            )
            span.set_attribute("gen_ai.cost.usd", cost)
            
            # Record metrics (BOUNDED CARDINALITY)
            metric_attrs = {
                "provider": provider,
                "model": model,
                "status": "success",
            }
            llm_calls_counter.add(1, metric_attrs)
            llm_latency_histogram.record(duration, metric_attrs)
            llm_tokens_histogram.record(response.usage.total_tokens, metric_attrs)
            llm_cost_counter.add(cost, {"provider": provider, "model": model})
            
            span.set_status(Status(StatusCode.OK))
            
            return {
                "response": response,
                "duration_ms": int(duration * 1000),
                "cost_usd": cost,
            }
        
        except Exception as e:
            duration = time.time() - start_time
            
            # Record error
            span.set_status(Status(StatusCode.ERROR, description=str(e)))
            span.record_exception(e)
            span.set_attribute("gen_ai.error.type", type(e).__name__)
            
            # Metrics for errors
            llm_calls_counter.add(1, {
                "provider": provider,
                "model": model,
                "status": "error",
                "error_type": type(e).__name__,
            })
            llm_latency_histogram.record(duration, {
                "provider": provider,
                "model": model,
                "status": "error",
            })
            
            raise


# Example usage - trace propagates through nested calls
async def handle_user_query(query: str, user_id: str):
    with tracer.start_as_current_span("handle_user_query") as parent_span:
        parent_span.set_attribute("user.id_hash", hash(user_id))  # hash for cardinality
        
        # Retrieval (nested span)
        with tracer.start_as_current_span("rag.retrieve"):
            chunks = await retrieve_chunks(query)
        
        # LLM call (nested span with all attributes)
        result = await instrumented_llm_call(
            provider="openai",
            model="gpt-4o",
            messages=[{"role": "user", "content": query}],
        )
        
        return result
```

**Enterprise features:**
- GenAI semantic conventions (standard attribute names)
- Bounded cardinality metrics (no user_id in labels)
- Both traces AND metrics captured
- Error paths instrumented
- Cost as first-class attribute
- Parent-child span relationships
- Compatible with any OTel backend (Grafana, Datadog, Honeycomb, LangSmith)

---

## Q3. Build a RAG Observability Layer ⭐⭐⭐⭐

**Prompt:** Implement full observability for a RAG pipeline. Track: query embedding, retrieval, reranking, chunk relevance, context utilization, generation.

**Solution:**

```python
from dataclasses import dataclass, field, asdict
from typing import Any
import time
import uuid

@dataclass
class ChunkTrace:
    chunk_id: str
    doc_id: str
    text: str  # Redacted before persistence
    retrieval_score: float
    rerank_score: float | None = None
    cited_in_response: bool = False
    position_in_context: int = 0

@dataclass
class RAGTrace:
    trace_id: str
    query: str
    query_embedding_time_ms: int = 0
    retrieval_time_ms: int = 0
    reranker_time_ms: int = 0
    llm_time_ms: int = 0
    total_time_ms: int = 0
    chunks_retrieved: list[ChunkTrace] = field(default_factory=list)
    chunks_reranked: list[ChunkTrace] = field(default_factory=list)
    chunks_in_context: list[ChunkTrace] = field(default_factory=list)
    llm_response: str = ""
    llm_tokens_input: int = 0
    llm_tokens_output: int = 0
    llm_cost_usd: float = 0.0
    quality_scores: dict = field(default_factory=dict)
    citations_found: list[str] = field(default_factory=list)

class ObservableRAGPipeline:
    """
    RAG pipeline with full observability at every stage.
    """
    
    def __init__(self, embedder, vector_db, reranker, llm, logger):
        self.embedder = embedder
        self.vector_db = vector_db
        self.reranker = reranker
        self.llm = llm
        self.logger = logger
    
    async def query(self, user_query: str, top_k_retrieval: int = 20, top_k_final: int = 5) -> dict:
        trace = RAGTrace(
            trace_id=str(uuid.uuid4()),
            query=user_query,
        )
        total_start = time.time()
        
        # Stage 1: Embed query
        embed_start = time.time()
        query_embedding = await self.embedder.embed(user_query)
        trace.query_embedding_time_ms = int((time.time() - embed_start) * 1000)
        
        # Stage 2: Retrieve
        retrieval_start = time.time()
        retrieved = await self.vector_db.search(query_embedding, top_k=top_k_retrieval)
        trace.retrieval_time_ms = int((time.time() - retrieval_start) * 1000)
        
        trace.chunks_retrieved = [
            ChunkTrace(
                chunk_id=r.chunk_id,
                doc_id=r.doc_id,
                text=r.text,
                retrieval_score=r.score,
            )
            for r in retrieved
        ]
        
        # Retrieval quality check
        if not retrieved:
            self.logger.warning("retrieval_empty", trace_id=trace.trace_id, query=user_query)
            trace.quality_scores["retrieval_status"] = "empty"
        elif retrieved[0].score < 0.5:
            self.logger.warning("retrieval_low_relevance", trace_id=trace.trace_id, 
                              top_score=retrieved[0].score)
            trace.quality_scores["retrieval_status"] = "low_relevance"
        else:
            trace.quality_scores["retrieval_status"] = "ok"
        
        # Stage 3: Rerank
        rerank_start = time.time()
        reranked = await self.reranker.rerank(user_query, retrieved, top_k=top_k_final)
        trace.reranker_time_ms = int((time.time() - rerank_start) * 1000)
        
        trace.chunks_reranked = [
            ChunkTrace(
                chunk_id=r.chunk_id,
                doc_id=r.doc_id,
                text=r.text,
                retrieval_score=r.retrieval_score,
                rerank_score=r.rerank_score,
                position_in_context=i,
            )
            for i, r in enumerate(reranked)
        ]
        trace.chunks_in_context = trace.chunks_reranked
        
        # Stage 4: LLM generation
        llm_start = time.time()
        llm_result = await self.llm.generate(
            query=user_query,
            context_chunks=reranked,
            require_citations=True,
        )
        trace.llm_time_ms = int((time.time() - llm_start) * 1000)
        
        trace.llm_response = llm_result.text
        trace.llm_tokens_input = llm_result.input_tokens
        trace.llm_tokens_output = llm_result.output_tokens
        trace.llm_cost_usd = llm_result.cost
        
        # Stage 5: Analyze citations & context utilization
        trace.citations_found = self._extract_citations(llm_result.text)
        for chunk in trace.chunks_in_context:
            chunk.cited_in_response = chunk.chunk_id in trace.citations_found
        
        context_utilization = (
            len(trace.citations_found) / len(trace.chunks_in_context)
            if trace.chunks_in_context else 0
        )
        trace.quality_scores["context_utilization"] = context_utilization
        trace.quality_scores["citations_count"] = len(trace.citations_found)
        
        # Stage 6: Optional async quality checks (sample 10%)
        if hash(trace.trace_id) % 10 == 0:
            await self._async_quality_check(trace)
        
        trace.total_time_ms = int((time.time() - total_start) * 1000)
        
        # Emit observability signal
        self._emit_trace(trace)
        
        return {
            "answer": trace.llm_response,
            "citations": trace.citations_found,
            "trace_id": trace.trace_id,
            "cost_usd": trace.llm_cost_usd,
            "latency_ms": trace.total_time_ms,
        }
    
    def _extract_citations(self, response: str) -> list[str]:
        """Extract chunk_id citations from response."""
        import re
        return list(set(re.findall(r'\[chunk_id:([a-zA-Z0-9-]+)\]', response)))
    
    async def _async_quality_check(self, trace: RAGTrace):
        """LLM-judge sample of traffic — background."""
        # Fire and forget — send to background queue
        pass
    
    def _emit_trace(self, trace: RAGTrace):
        """Emit to observability backend (LangSmith, Langfuse, etc.)."""
        self.logger.info(
            "rag_query_complete",
            trace_id=trace.trace_id,
            total_time_ms=trace.total_time_ms,
            retrieval_time_ms=trace.retrieval_time_ms,
            llm_time_ms=trace.llm_time_ms,
            cost_usd=trace.llm_cost_usd,
            chunks_retrieved=len(trace.chunks_retrieved),
            chunks_in_context=len(trace.chunks_in_context),
            citations_count=len(trace.citations_found),
            context_utilization=trace.quality_scores.get("context_utilization", 0),
            retrieval_status=trace.quality_scores.get("retrieval_status"),
            top_retrieval_score=trace.chunks_retrieved[0].retrieval_score if trace.chunks_retrieved else 0,
        )
```

**Enterprise features:**
- Full pipeline stage timing
- Retrieval quality categorization
- Citation extraction for context utilization measurement
- Sampled quality checks
- Structured trace for querying/aggregation

---

## Q4-Q17: Additional Coding Challenges (Condensed)

### Q4. Build an Agent Trajectory Logger ⭐⭐⭐⭐
Log every step: thought, tool called, arguments, result, cost, latency. Structured JSON. One trajectory per task with parent trace_id. Support replay for debugging. Compute trajectory metrics: step count, cost, tool distribution.

### Q5. Implement PSI (Population Stability Index) Drift Detector ⭐⭐⭐⭐
Two distributions (baseline, current). Bin into N buckets. Compute per-bin probability. PSI = Σ(P_current - P_baseline) × ln(P_current / P_baseline). Return: PSI value + per-bin contribution + drift severity classification (< 0.1 stable, 0.1-0.25 moderate, > 0.25 significant).

### Q6. Build a Cost Anomaly Detector ⭐⭐⭐
Track hourly cost. Compute rolling mean + stddev over last 7 days. Detect anomalies: current > mean + 2σ (moderate) or > mean + 3σ (severe). Route alerts by severity. Forecast month-end from current burn rate. Alert if forecasted overshoots budget.

### Q7. Implement LLM-as-Judge Quality Scorer ⭐⭐⭐⭐
Given (query, context, response), prompt judge LLM with rubric: faithfulness (1-5), relevance (1-5), helpfulness (1-5). Parse structured output. Aggregate scores. Cost per judgment: ~$0.001. Bias mitigation: use different model family than the one being judged.

### Q8. Build a Golden Set Runner ⭐⭐⭐
Set of curated (query, expected_answer) pairs. Runs against production every hour. Computes: exact match, semantic similarity (embedding cosine), LLM-judge similarity to expected. Alerts if quality drops below baseline. Dashboard: quality over time.

### Q9. Implement Symptom-Based Alerting Engine ⭐⭐⭐
Multiple SLIs: latency p95, error rate, quality score, cost. Each has SLO. Multi-window burn rate: fast (5min at 10x) + slow (1hr at 2x). Alert routing: severity 1 → page, severity 2 → slack. Auto-resolve when metric returns to healthy. Alert deduplication.

### Q10. Build a Distributed Trace Analyzer ⭐⭐⭐⭐
Given trace ID, fetch all spans. Compute: total duration, critical path (longest chain), fan-out (parallel spans), errors, latency by stage. Identify bottleneck (slowest span). Compare traces of similar requests (find outliers).

### Q11. Implement Sampling Decision Engine ⭐⭐⭐
Configurable sampling: head-based (X% of all traces), plus 100% of errors, 100% of slow (> threshold), 100% of paid tier users, adversarial (suspected issues). Track sampled vs total for accuracy calculations.

### Q12. Build a Prompt Injection Detector for Observability ⭐⭐⭐⭐
Analyze incoming prompts for injection patterns. Signals: instruction-like content ("ignore previous"), system prompt reveal attempts, format-breaking, unusual character patterns. Score each request. Flag high-risk. Store flagged requests for security review.

### Q13. Implement Embedding Drift Detector ⭐⭐⭐
Batch of query embeddings vs baseline batch. Compute: distribution mean/variance, cluster analysis (K-means, compare cluster centroids), PSI on PCA-reduced dimensions. Alert on significant shift. Suggest re-embedding if drift is significant.

### Q14. Build an Audit Log System ⭐⭐⭐⭐
Every access to sensitive data logged. Fields: timestamp, user, action, resource, result, IP. Append-only storage. Hash chain for tamper evidence (each entry hashes previous). Retention: 7-10 years. Encrypted at rest. Separate from operational logs. Compliance dashboard.

### Q15. Implement Dashboard-as-Code (Grafana JSON) ⭐⭐⭐
Grafana dashboards in code (JSON or jsonnet). Version-controlled. Deployed via CI/CD. Templated for multi-tenant (per-tenant dashboards from single template). Standard panels: RED metrics + LLM-specific + cost + quality. Reusable across services.

### Q16. Build a Feedback Loop Aggregator ⭐⭐⭐
User thumbs up/down + optional reason. Aggregate by: user, tenant, feature, model version. Correlate with quality scores. Category detection (LLM classifies reason). Weekly reports: worst categories, worst features, improvement candidates. Feed golden set with negative examples.

### Q17. Implement Multi-Tenant Cost Attribution ⭐⭐⭐⭐
Every LLM call tagged: tenant_id, user_id, feature. Real-time aggregation to Redis: per-user daily, per-tenant monthly. Async persist to warehouse (BigQuery/Snowflake) for analytics. Dashboards per tenant (they see only their own). Cost anomaly per tenant. Chargeback billing report.
