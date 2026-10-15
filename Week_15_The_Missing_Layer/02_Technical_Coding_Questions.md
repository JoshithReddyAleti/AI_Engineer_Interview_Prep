# 💻 Week 15 — Technical / Coding Questions

> **Focus:** Multi-tenant semantic cache, cost calculator with model tiering, best-of-N with verifier, ACL-aware retrieval, audit event with hash chain, SSE streaming handler, reasoning ROI calculator
>
> **Depth:** MAANG staff+ / enterprise architect bar. Every solution reflects production considerations, not tutorial code.

---

## Q1. Implement a Multi-Tenant Semantic Cache with Per-Tenant Isolation ⭐⭐⭐⭐

**Prompt:** Build a semantic cache that supports multi-tenant SaaS. Prevent cross-tenant data leakage while maximizing hit rate within tenant. Handle personalization boundaries. Include cost tracking.

**Solution:**

```python
import hashlib
import time
import numpy as np
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class CachedResponse:
    tenant_id: str
    user_id: Optional[str]      # None if tenant-shared
    query_embedding: np.ndarray  # for similarity check
    query_text: str              # for audit
    response: str
    model_used: str
    cost_saved_per_hit: float    # what we would have paid
    created_at: float
    hit_count: int = 0
    metadata: dict = field(default_factory=dict)


class MultiTenantSemanticCache:
    """
    Semantic cache with strict multi-tenant isolation.
    
    Design principles:
      1. Per-tenant namespace at the STORAGE layer (not just filtering)
      2. Personalized responses cached per-user, not per-tenant
      3. Similarity threshold tuned per use case
      4. Cost attribution built in
      5. Adversarial poisoning defenses (see production notes)
    """
    
    def __init__(
        self,
        similarity_threshold: float = 0.95,
        ttl_seconds: int = 3600,
        max_entries_per_tenant: int = 10_000,
    ):
        # Nested dict: tenant_id -> user_id (or 'shared') -> list of entries
        # In production: use Redis with per-tenant keyspaces or Postgres with tenant_id partitioning
        self.cache: dict[str, dict[str, list[CachedResponse]]] = {}
        self.similarity_threshold = similarity_threshold
        self.ttl_seconds = ttl_seconds
        self.max_entries_per_tenant = max_entries_per_tenant
        
        # Metrics
        self.total_hits = 0
        self.total_misses = 0
        self.total_cost_saved = 0.0
        self.per_tenant_stats: dict[str, dict] = {}
    
    def _get_scope_key(self, user_id: Optional[str], is_personalized: bool) -> str:
        """Personalized responses cannot be shared across users."""
        if is_personalized:
            assert user_id is not None, "Personalized responses require user_id"
            return f"user:{user_id}"
        return "shared"
    
    def _cosine(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
    
    def get(
        self,
        tenant_id: str,
        query_embedding: np.ndarray,
        user_id: Optional[str] = None,
        is_personalized: bool = False,
    ) -> Optional[CachedResponse]:
        """
        Look up cached response for query in tenant scope.
        
        Returns matching CachedResponse if similarity > threshold and not expired.
        """
        tenant_bucket = self.cache.get(tenant_id, {})
        scope_key = self._get_scope_key(user_id, is_personalized)
        entries = tenant_bucket.get(scope_key, [])
        
        now = time.time()
        best_match = None
        best_similarity = 0.0
        
        for entry in entries:
            # Check TTL
            if now - entry.created_at > self.ttl_seconds:
                continue
            
            similarity = self._cosine(query_embedding, entry.query_embedding)
            if similarity >= self.similarity_threshold and similarity > best_similarity:
                best_match = entry
                best_similarity = similarity
        
        if best_match:
            best_match.hit_count += 1
            self.total_hits += 1
            self.total_cost_saved += best_match.cost_saved_per_hit
            self._update_tenant_stats(tenant_id, hit=True, cost_saved=best_match.cost_saved_per_hit)
        else:
            self.total_misses += 1
            self._update_tenant_stats(tenant_id, hit=False, cost_saved=0.0)
        
        return best_match
    
    def put(
        self,
        tenant_id: str,
        query_embedding: np.ndarray,
        query_text: str,
        response: str,
        model_used: str,
        cost_per_query: float,
        user_id: Optional[str] = None,
        is_personalized: bool = False,
    ):
        """Store response in cache."""
        if tenant_id not in self.cache:
            self.cache[tenant_id] = {}
        
        scope_key = self._get_scope_key(user_id, is_personalized)
        if scope_key not in self.cache[tenant_id]:
            self.cache[tenant_id][scope_key] = []
        
        entries = self.cache[tenant_id][scope_key]
        
        # LRU eviction if over limit
        if len(entries) >= self.max_entries_per_tenant:
            # Remove oldest
            entries.sort(key=lambda e: e.created_at)
            entries.pop(0)
        
        entries.append(CachedResponse(
            tenant_id=tenant_id,
            user_id=user_id if is_personalized else None,
            query_embedding=query_embedding,
            query_text=query_text,
            response=response,
            model_used=model_used,
            cost_saved_per_hit=cost_per_query,
            created_at=time.time(),
        ))
    
    def invalidate_tenant(self, tenant_id: str):
        """Full tenant cache flush — for compliance events (GDPR erasure, etc.)."""
        if tenant_id in self.cache:
            del self.cache[tenant_id]
    
    def invalidate_user(self, tenant_id: str, user_id: str):
        """User-specific flush — for right-to-erasure requests."""
        scope_key = self._get_scope_key(user_id, is_personalized=True)
        if tenant_id in self.cache and scope_key in self.cache[tenant_id]:
            del self.cache[tenant_id][scope_key]
    
    def _update_tenant_stats(self, tenant_id: str, hit: bool, cost_saved: float):
        if tenant_id not in self.per_tenant_stats:
            self.per_tenant_stats[tenant_id] = {"hits": 0, "misses": 0, "cost_saved": 0.0}
        stats = self.per_tenant_stats[tenant_id]
        if hit:
            stats["hits"] += 1
            stats["cost_saved"] += cost_saved
        else:
            stats["misses"] += 1
    
    def stats(self) -> dict:
        total = self.total_hits + self.total_misses
        hit_rate = self.total_hits / total if total > 0 else 0
        return {
            "total_hits": self.total_hits,
            "total_misses": self.total_misses,
            "hit_rate": hit_rate,
            "total_cost_saved_usd": self.total_cost_saved,
            "per_tenant": self.per_tenant_stats,
        }
```

**Production notes:**
- Replace in-memory dict with Redis (per-tenant keyspaces) or Postgres (tenant_id partitioning)
- For adversarial poisoning defense: rate-limit cache writes per tenant, validate response quality before caching
- For GDPR: `invalidate_user` supports right-to-erasure
- For SOC 2: log every cache put/get/invalidate to audit trail
- For KV cache providers (Anthropic prompt cache): namespace cache keys with tenant_id prefix

---

## Q2. Implement a Cost Calculator with Model Tiering ⭐⭐⭐⭐

**Prompt:** Given a query volume and query characteristics, compute expected cost across a router that tiers Haiku / Sonnet / Opus / o1. Include prompt cache impact.

**Solution:**

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class ModelPricing:
    """2026 published prices per M tokens."""
    name: str
    input_price_per_M: float
    output_price_per_M: float
    supports_prompt_cache: bool = True
    cache_price_multiplier: float = 0.1  # 90% discount on cached tokens


PRICING = {
    "gpt-4o-mini": ModelPricing("gpt-4o-mini", 0.15, 0.60),
    "haiku": ModelPricing("haiku", 0.25, 1.25),
    "flash": ModelPricing("flash", 0.10, 0.40),
    "gpt-4o": ModelPricing("gpt-4o", 2.50, 10.00),
    "sonnet": ModelPricing("sonnet", 3.00, 15.00),
    "opus": ModelPricing("opus", 15.00, 75.00),
    "o1": ModelPricing("o1", 15.00, 60.00, supports_prompt_cache=False),
}


@dataclass
class QueryProfile:
    """Represents a class of queries in your workload."""
    name: str
    fraction_of_traffic: float
    fixed_prompt_tokens: int      # cacheable
    variable_prompt_tokens: int   # not cacheable
    output_tokens: int
    reasoning_thinking_tokens: int = 0  # for reasoning models


@dataclass
class RouterRule:
    """Route this query class to this model."""
    query_class: str
    model: str


def query_cost(
    profile: QueryProfile,
    model_name: str,
    cache_hit_rate: float = 0.0,
) -> float:
    """
    Cost of one query on given model with given cache hit rate.
    """
    pricing = PRICING[model_name]
    
    # Fixed prompt tokens (cacheable)
    if pricing.supports_prompt_cache:
        cached_cost = profile.fixed_prompt_tokens * cache_hit_rate * pricing.cache_price_multiplier
        uncached_cost = profile.fixed_prompt_tokens * (1 - cache_hit_rate)
        fixed_cost = (cached_cost + uncached_cost) * pricing.input_price_per_M / 1_000_000
    else:
        fixed_cost = profile.fixed_prompt_tokens * pricing.input_price_per_M / 1_000_000
    
    # Variable prompt tokens (never cached)
    variable_cost = profile.variable_prompt_tokens * pricing.input_price_per_M / 1_000_000
    
    # Reasoning thinking tokens (charged as output for o1)
    thinking_cost = profile.reasoning_thinking_tokens * pricing.output_price_per_M / 1_000_000
    
    # Output tokens
    output_cost = profile.output_tokens * pricing.output_price_per_M / 1_000_000
    
    return fixed_cost + variable_cost + thinking_cost + output_cost


def total_cost(
    queries_per_day: int,
    profiles: list[QueryProfile],
    router: dict[str, str],
    cache_hit_rate: float = 0.4,
) -> dict:
    """
    Compute expected daily cost with the given routing.
    """
    total_per_day = 0.0
    per_class: dict[str, dict] = {}
    
    for profile in profiles:
        model = router[profile.name]
        cost_per_query = query_cost(profile, model, cache_hit_rate)
        class_queries = queries_per_day * profile.fraction_of_traffic
        class_cost = cost_per_query * class_queries
        
        per_class[profile.name] = {
            "model": model,
            "queries": class_queries,
            "cost_per_query": cost_per_query,
            "daily_cost": class_cost,
        }
        total_per_day += class_cost
    
    return {
        "daily_cost_usd": total_per_day,
        "monthly_cost_usd": total_per_day * 30,
        "annual_cost_usd": total_per_day * 365,
        "per_class": per_class,
    }


# Example usage
if __name__ == "__main__":
    profiles = [
        QueryProfile("classification", 0.60, 1000, 100, 50),
        QueryProfile("standard_rag", 0.30, 1500, 500, 300),
        QueryProfile("complex_synthesis", 0.08, 1500, 1000, 800),
        QueryProfile("reasoning_heavy", 0.02, 1500, 500, 500, reasoning_thinking_tokens=3000),
    ]
    
    router_baseline = {
        "classification": "sonnet",
        "standard_rag": "sonnet",
        "complex_synthesis": "sonnet",
        "reasoning_heavy": "sonnet",
    }
    
    router_tiered = {
        "classification": "haiku",
        "standard_rag": "sonnet",
        "complex_synthesis": "opus",
        "reasoning_heavy": "o1",
    }
    
    result_baseline = total_cost(100_000, profiles, router_baseline)
    result_tiered = total_cost(100_000, profiles, router_tiered)
    
    print(f"Baseline: ${result_baseline['monthly_cost_usd']:,.0f}/mo")
    print(f"Tiered:   ${result_tiered['monthly_cost_usd']:,.0f}/mo")
    print(f"Savings:  {(1 - result_tiered['monthly_cost_usd']/result_baseline['monthly_cost_usd'])*100:.1f}%")
```

---

## Q3. Implement Best-of-N With Verifier ⭐⭐⭐⭐

**Prompt:** Build best-of-N sampling with pluggable verifier. Show log-scaling behavior. Track cost.

**Solution:**

```python
import math
from dataclasses import dataclass
from typing import Callable

@dataclass
class BoNResult:
    best_answer: str
    best_score: float
    all_candidates: list[tuple[str, float]]
    total_cost_usd: float
    n: int


def best_of_n(
    generate_fn: Callable[[str], tuple[str, float]],  # returns (response, cost)
    verify_fn: Callable[[str, str], float],  # returns score 0..1
    prompt: str,
    n: int,
) -> BoNResult:
    """
    Sample n candidates, verify each, return best.
    
    Log-scaling: quality grows approximately as log(n) up to verifier ceiling.
    """
    candidates = []
    total_cost = 0.0
    
    for i in range(n):
        response, cost = generate_fn(prompt)
        total_cost += cost
        score = verify_fn(prompt, response)
        candidates.append((response, score))
    
    # Sort by verifier score
    candidates.sort(key=lambda x: x[1], reverse=True)
    best_answer, best_score = candidates[0]
    
    return BoNResult(
        best_answer=best_answer,
        best_score=best_score,
        all_candidates=candidates,
        total_cost_usd=total_cost,
        n=n,
    )


# Demonstrate log-scaling on synthetic task
def synthetic_generate(prompt: str) -> tuple[str, float]:
    """Simulated LLM with 40% base success rate."""
    import random
    is_correct = random.random() < 0.4
    return ("correct_answer" if is_correct else f"wrong_{random.randint(0,100)}"), 0.01

def perfect_verifier(prompt: str, response: str) -> float:
    return 1.0 if response == "correct_answer" else 0.0

def weak_verifier(prompt: str, response: str) -> float:
    """80% accuracy verifier."""
    import random
    true_score = 1.0 if response == "correct_answer" else 0.0
    if random.random() < 0.8:
        return true_score
    else:
        return 1.0 - true_score  # inverted


# Empirical log-scaling curve
if __name__ == "__main__":
    for n in [1, 4, 16, 64, 256]:
        success_count = 0
        trials = 1000
        for _ in range(trials):
            result = best_of_n(synthetic_generate, perfect_verifier, "test", n)
            if result.best_answer == "correct_answer":
                success_count += 1
        
        empirical_success = success_count / trials
        theoretical = 1 - (0.6 ** n)  # 1 - (1-p)^n
        print(f"n={n:4d}  empirical={empirical_success:.3f}  theoretical={theoretical:.3f}")
    
    # With weak verifier: shows verifier ceiling effect
    print("\nWith 80% verifier (shows ceiling):")
    for n in [1, 4, 16, 64, 256]:
        success_count = 0
        trials = 1000
        for _ in range(trials):
            result = best_of_n(synthetic_generate, weak_verifier, "test", n)
            if result.best_answer == "correct_answer":
                success_count += 1
        print(f"n={n:4d}  success={success_count/trials:.3f}")
```

---

## Q4. ACL-Aware Retrieval With Pre-Filter ⭐⭐⭐⭐

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class Document:
    doc_id: str
    text: str
    embedding: 'np.ndarray'
    allowed_users: set[str]  # empty means group-based only
    allowed_groups: set[str]
    classification: str  # 'public', 'internal', 'confidential', 'secret'


@dataclass
class User:
    user_id: str
    groups: set[str]
    clearance_level: str  # to match against classification


CLASSIFICATION_HIERARCHY = {
    "public": 0,
    "internal": 1,
    "confidential": 2,
    "secret": 3,
}


def user_can_access(user: User, doc: Document) -> bool:
    """Explicit permission check."""
    # Clearance level check
    user_clearance = CLASSIFICATION_HIERARCHY.get(user.clearance_level, 0)
    doc_classification = CLASSIFICATION_HIERARCHY.get(doc.classification, 0)
    if user_clearance < doc_classification:
        return False
    
    # ACL check
    if doc.allowed_users and user.user_id in doc.allowed_users:
        return True
    if doc.allowed_groups and user.groups.intersection(doc.allowed_groups):
        return True
    if doc.classification == "public":
        return True
    return False


class ACLAwareRetriever:
    """RAG retriever with row-level security enforced at query time."""
    
    def __init__(self, documents: list[Document]):
        self.documents = documents
    
    def retrieve(
        self,
        user: User,
        query_embedding: 'np.ndarray',
        top_k: int = 5,
    ) -> tuple[list[Document], dict]:
        """
        PRE-FILTER by ACL, then vector search over allowed subset.
        
        Returns (docs, audit_metadata).
        """
        # Step 1: pre-filter by permissions
        allowed = [d for d in self.documents if user_can_access(user, d)]
        
        # Step 2: vector search over allowed subset
        import numpy as np
        if not allowed:
            return [], {"filtered_out_count": len(self.documents), "returned_count": 0}
        
        similarities = np.array([
            np.dot(query_embedding, d.embedding) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(d.embedding)
            )
            for d in allowed
        ])
        
        top_indices = np.argsort(-similarities)[:top_k]
        top_docs = [allowed[i] for i in top_indices]
        
        # Step 3: audit metadata
        audit = {
            "user_id": user.user_id,
            "user_groups": list(user.groups),
            "user_clearance": user.clearance_level,
            "total_docs_in_index": len(self.documents),
            "docs_filtered_by_acl": len(self.documents) - len(allowed),
            "docs_returned": len(top_docs),
            "returned_doc_ids": [d.doc_id for d in top_docs],
            "returned_classifications": [d.classification for d in top_docs],
        }
        
        return top_docs, audit
```

**Critical detail:** pre-filter, not post-filter. Post-filter would search full index, then remove unauthorized docs — could leave zero results even when authorized docs exist. Pre-filter searches only authorized subset.

---

## Q5. Audit Event With Hash Chain (Tamper-Evident) ⭐⭐⭐⭐

```python
import hashlib
import json
import time
from dataclasses import dataclass, asdict, field
from typing import Optional


@dataclass
class AuditEvent:
    event_id: str
    timestamp: float
    tenant_id: str
    user_id: str
    action: str
    resource: str
    data_hashes: dict  # sha256 of prompt, response, retrieved doc IDs
    compliance_controls: list[str]  # ["SOC2-CC6.1", "HIPAA-164.312a"]
    prev_hash: str  # sha256 of previous event
    event_hash: str = ""  # sha256 of this event
    
    def compute_hash(self) -> str:
        # Serialize event (except event_hash itself) deterministically
        event_dict = asdict(self)
        event_dict.pop("event_hash", None)
        canonical = json.dumps(event_dict, sort_keys=True)
        return hashlib.sha256(canonical.encode()).hexdigest()


class AuditChain:
    """
    Append-only audit log with hash chain for tamper evidence.
    
    Any modification to a past event breaks the chain and is detectable.
    In production: append events to WORM storage (S3 Object Lock, Azure Immutable Blob).
    """
    
    GENESIS_HASH = "0" * 64
    
    def __init__(self):
        self.events: list[AuditEvent] = []
    
    def append(
        self,
        tenant_id: str,
        user_id: str,
        action: str,
        resource: str,
        data_hashes: dict,
        compliance_controls: list[str],
    ) -> AuditEvent:
        prev_hash = self.events[-1].event_hash if self.events else self.GENESIS_HASH
        event = AuditEvent(
            event_id=hashlib.sha256(f"{time.time_ns()}".encode()).hexdigest()[:16],
            timestamp=time.time(),
            tenant_id=tenant_id,
            user_id=user_id,
            action=action,
            resource=resource,
            data_hashes=data_hashes,
            compliance_controls=compliance_controls,
            prev_hash=prev_hash,
        )
        event.event_hash = event.compute_hash()
        self.events.append(event)
        return event
    
    def verify(self) -> tuple[bool, Optional[int]]:
        """
        Walk the chain, verify each hash matches.
        Returns (is_valid, index_of_first_broken_link).
        """
        prev_hash = self.GENESIS_HASH
        for i, event in enumerate(self.events):
            # Verify prev_hash link
            if event.prev_hash != prev_hash:
                return False, i
            # Verify event's own hash
            if event.compute_hash() != event.event_hash:
                return False, i
            prev_hash = event.event_hash
        return True, None
```

**Production notes:**
- Store events in WORM storage (S3 Object Lock RETAIN mode)
- Periodic anchoring: publish chain root hash to public blockchain for third-party verifiability
- Digital signatures on each event (KMS-managed private key)
- Retention per regulation: 7yr HIPAA, 6yr SOX, indefinite for legal hold

---

## Q6-Q17: Additional Coding Challenges (Condensed)

### Q6. SSE Streaming Handler With Backpressure ⭐⭐⭐
FastAPI SSE endpoint that streams LLM tokens. Handle client disconnect (cancel generation). Buffer with max size. Return correct MIME type + proper heartbeat comments.

### Q7. Reasoning ROI Calculator ⭐⭐⭐⭐
Given task type, cost of error, accuracy gain, baseline model cost, reasoning model cost — compute break-even query volume. Show ROI curves. Recommend when to use reasoning.

### Q8. Tenant-Aware Rate Limiter ⭐⭐⭐
Rate limiting per tenant + per user + per feature. Token bucket algorithm. Redis-backed. Return 429 with Retry-After header. Priority tier for enterprise.

### Q9. SCIM 2.0 User/Group Endpoint ⭐⭐⭐⭐
Implement SCIM POST /Users, PATCH /Users/{id}, DELETE /Users/{id}. Handle group membership sync. Fire ACL update webhooks on user changes. Correct SCIM response format.

### Q10. Provider Fallback Circuit Breaker ⭐⭐⭐
Multi-provider LLM gateway. On provider failure (5xx, timeout), fall back to secondary. Track error rates per provider. Open circuit on sustained errors. Half-open probe pattern.

### Q11. Prompt Cache Namespace Manager ⭐⭐⭐
Manage prompt cache keys with tenant isolation. Prevent cross-tenant timing side channels. Track cache hit rate per tenant. Invalidate on prompt template updates.

### Q12. Cost Anomaly Detector ⭐⭐⭐
Monitor per-tenant LLM spend. Detect anomalies (e.g., 5× yesterday's rate). Alert. Automatic circuit breaker if threshold breached. Runaway agent scenario mitigation.

### Q13. Right-to-Erasure Coordinator ⭐⭐⭐⭐
Multi-system deletion workflow for GDPR Article 17. Delete from: app DB, vector DB, semantic cache, logs, backups (schedule), analytics, audit (flag). Track per-system status. Confirm completion within 30 days.

### Q14. PII Detection and Redaction ⭐⭐⭐
Detect PII in prompts and responses (email, phone, SSN, credit card, custom patterns). Redact before logging. Track detection rate. Support per-tenant PII policies.

### Q15. Consent Management for GDPR ⭐⭐⭐
Track user consent for: data processing, marketing, third-party sharing. Timestamped consent events. Withdraw consent workflow. Query 'is user X consented for purpose Y at time Z?'

### Q16. Data Residency Router ⭐⭐⭐⭐
Route LLM inference requests to region matching tenant's residency requirement. Fallback logic if region unavailable. Track routing per request for compliance evidence.

### Q17. Compliance Control Verifier ⭐⭐⭐
Given a request, verify all applicable SOC 2 / HIPAA / GDPR controls are met before processing. Reject with specific control code if not. Emit audit event for every check.
