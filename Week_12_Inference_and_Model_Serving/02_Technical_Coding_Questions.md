# 💻 Week 12 — Technical / Coding Questions

> **Focus:** Compute KV cache memory from first principles, build continuous batching scheduler, prefix cache, speculative acceptance rate calculator, cost-per-token math, quantization pipeline, benchmark harness with proper percentile handling
>
> **How to use:** These are live-code and whiteboard questions for inference infrastructure roles at MAANG, frontier labs, and specialty inference companies (Together, Fireworks, Groq, Anyscale).

---

## Q1. Compute KV Cache Memory Requirements From First Principles ⭐⭐⭐⭐

**Prompt:** Given a model config (num_layers, num_kv_heads, head_dim, dtype), compute per-token, per-request, and batch KV cache size. Return which GPU configurations can serve it.

**Solution:**

```python
from dataclasses import dataclass
from enum import Enum

class Dtype(Enum):
    FP32 = 4
    FP16 = 2
    BF16 = 2
    FP8 = 1
    INT8 = 1
    INT4 = 0.5  # 4 bits

@dataclass
class ModelConfig:
    name: str
    num_layers: int
    num_attention_heads: int
    num_kv_heads: int  # Different from num_attention_heads for GQA/MQA
    head_dim: int
    hidden_dim: int
    weights_bytes_per_param: float  # e.g., 2 for FP16, 0.5 for INT4
    num_parameters: int  # total param count
    
    @property
    def weight_memory_gb(self) -> float:
        return (self.num_parameters * self.weights_bytes_per_param) / (1024**3)

@dataclass
class GPUConfig:
    name: str
    hbm_gb: int
    hbm_bandwidth_tbs: float  # TB/s
    nvlink_gbs: float  # GB/s per GPU, 0 if no NVLink

# Reference model configs
LLAMA_70B_FP16 = ModelConfig(
    name="Llama-3.1-70B",
    num_layers=80,
    num_attention_heads=64,
    num_kv_heads=8,  # GQA with group size 8
    head_dim=128,
    hidden_dim=8192,
    weights_bytes_per_param=2,  # FP16
    num_parameters=70_000_000_000,
)

LLAMA_70B_INT4 = ModelConfig(
    name="Llama-3.1-70B-INT4",
    num_layers=80,
    num_attention_heads=64,
    num_kv_heads=8,
    head_dim=128,
    hidden_dim=8192,
    weights_bytes_per_param=0.5,  # INT4
    num_parameters=70_000_000_000,
)

# Reference GPU configs
H100_SXM = GPUConfig(name="H100 SXM", hbm_gb=80, hbm_bandwidth_tbs=3.35, nvlink_gbs=900)
H200 = GPUConfig(name="H200", hbm_gb=141, hbm_bandwidth_tbs=4.8, nvlink_gbs=900)
B200 = GPUConfig(name="B200", hbm_gb=192, hbm_bandwidth_tbs=8.0, nvlink_gbs=1800)


def compute_kv_cache_per_token(
    model: ModelConfig,
    kv_dtype_bytes: float = 2,  # FP16 default; use 1 for FP8
) -> float:
    """
    KV cache bytes per token.
    
    Formula: 2 (K and V) × num_layers × num_kv_heads × head_dim × dtype_bytes
    """
    return 2 * model.num_layers * model.num_kv_heads * model.head_dim * kv_dtype_bytes


def compute_kv_cache_per_request(
    model: ModelConfig,
    sequence_length: int,
    kv_dtype_bytes: float = 2,
) -> float:
    """KV cache bytes for a single request at given sequence length."""
    per_token = compute_kv_cache_per_token(model, kv_dtype_bytes)
    return per_token * sequence_length


def compute_batch_kv_cache(
    model: ModelConfig,
    sequence_length: int,
    batch_size: int,
    kv_dtype_bytes: float = 2,
) -> float:
    """Total KV cache bytes for a batch of requests."""
    per_request = compute_kv_cache_per_request(model, sequence_length, kv_dtype_bytes)
    return per_request * batch_size


def can_serve(
    model: ModelConfig,
    gpu: GPUConfig,
    num_gpus: int,
    sequence_length: int,
    batch_size: int,
    kv_dtype_bytes: float = 2,
    activation_overhead_gb: float = 10.0,  # Rough estimate per GPU
) -> tuple[bool, dict]:
    """
    Can this GPU config serve the model at this batch size + sequence length?
    Assumes tensor parallelism sharding weights + KV evenly.
    """
    total_hbm = gpu.hbm_gb * num_gpus
    weight_gb = model.weight_memory_gb
    kv_gb = compute_batch_kv_cache(model, sequence_length, batch_size, kv_dtype_bytes) / (1024**3)
    activation_gb = activation_overhead_gb * num_gpus
    
    total_needed = weight_gb + kv_gb + activation_gb
    fits = total_needed <= total_hbm
    
    # Per-GPU check (with TP, load is spread but KV cache also spread)
    per_gpu_weight = weight_gb / num_gpus
    per_gpu_kv = kv_gb / num_gpus
    per_gpu_activation = activation_overhead_gb
    per_gpu_total = per_gpu_weight + per_gpu_kv + per_gpu_activation
    per_gpu_fits = per_gpu_total <= gpu.hbm_gb
    
    return per_gpu_fits, {
        "total_hbm_gb": total_hbm,
        "weight_gb": weight_gb,
        "kv_cache_gb": kv_gb,
        "activation_gb": activation_gb,
        "total_needed_gb": total_needed,
        "per_gpu_needed_gb": per_gpu_total,
        "per_gpu_available_gb": gpu.hbm_gb,
        "headroom_gb": gpu.hbm_gb - per_gpu_total,
    }


# Example usage
if __name__ == "__main__":
    per_token = compute_kv_cache_per_token(LLAMA_70B_FP16)
    print(f"Llama-70B FP16 KV per token: {per_token / 1024:.1f} KB")
    # Expected: ~320 KB (2 × 80 × 8 × 128 × 2 = 327,680 bytes)
    
    # 32 concurrent × 8K context on 4×H100
    fits, details = can_serve(
        model=LLAMA_70B_FP16,
        gpu=H100_SXM,
        num_gpus=4,
        sequence_length=8192,
        batch_size=32,
    )
    print(f"Fits: {fits}, details: {details}")
    
    # With FP8 KV cache
    fits_fp8, details_fp8 = can_serve(
        model=LLAMA_70B_INT4,  # Weights INT4
        gpu=H100_SXM,
        num_gpus=4,
        sequence_length=8192,
        batch_size=64,
        kv_dtype_bytes=1,  # FP8 KV
    )
    print(f"With INT4 weights + FP8 KV @ batch 64: {fits_fp8}, {details_fp8}")
```

**What this demonstrates:**
- KV cache formula from first principles
- Awareness of GQA vs MHA (using num_kv_heads correctly)
- Quantization impact on both weights and KV
- Tensor parallelism sharding math
- Per-GPU budget checking (not just total)

---

## Q2. Implement Continuous Batching Scheduler ⭐⭐⭐⭐

**Prompt:** Build a continuous batching scheduler for LLM inference. Each request has a prompt and target output length. Scheduler decides which requests are in the batch at each iteration.

**Solution:**

```python
from dataclasses import dataclass, field
from enum import Enum
from collections import deque
import time
import uuid

class RequestState(Enum):
    QUEUED = "queued"
    PREFILLING = "prefilling"
    DECODING = "decoding"
    FINISHED = "finished"

@dataclass
class Request:
    id: str
    prompt_tokens: int
    max_output_tokens: int
    output_tokens_generated: int = 0
    state: RequestState = RequestState.QUEUED
    kv_blocks_used: int = 0
    arrival_time: float = field(default_factory=time.time)
    prefill_started_at: float = 0
    decode_started_at: float = 0
    finished_at: float = 0
    
    @property
    def is_finished(self) -> bool:
        return self.output_tokens_generated >= self.max_output_tokens
    
    @property
    def total_tokens(self) -> int:
        return self.prompt_tokens + self.output_tokens_generated

@dataclass
class SchedulerConfig:
    max_batch_size: int = 128
    max_kv_blocks: int = 2048  # e.g., 32K tokens with block_size=16
    kv_block_size: int = 16
    max_context_length: int = 8192
    chunked_prefill_size: int = 512  # For chunked prefill
    prefill_priority: bool = True  # Prefill requests take priority

class ContinuousBatchingScheduler:
    """
    Iteration-level (continuous) batching scheduler.
    At each iteration, decides which requests are in the batch.
    """
    
    def __init__(self, config: SchedulerConfig):
        self.config = config
        self.queue: deque[Request] = deque()
        self.active_batch: list[Request] = []
        self.finished: list[Request] = []
        self.kv_blocks_used: int = 0
    
    def add_request(self, prompt_tokens: int, max_output_tokens: int) -> str:
        """Add a new request to the queue."""
        req = Request(
            id=str(uuid.uuid4())[:8],
            prompt_tokens=prompt_tokens,
            max_output_tokens=max_output_tokens,
        )
        self.queue.append(req)
        return req.id
    
    def _kv_blocks_needed(self, req: Request, extra_tokens: int = 0) -> int:
        """Blocks needed for a request after generating extra_tokens more."""
        total = req.total_tokens + extra_tokens
        return (total + self.config.kv_block_size - 1) // self.config.kv_block_size
    
    def _can_admit_new(self, req: Request) -> bool:
        """Can we admit this queued request into the batch?"""
        if len(self.active_batch) >= self.config.max_batch_size:
            return False
        needed = self._kv_blocks_needed(req)
        if self.kv_blocks_used + needed > self.config.max_kv_blocks:
            return False
        return True
    
    def _remove_finished(self):
        """Remove finished requests and reclaim their KV blocks."""
        new_active = []
        for req in self.active_batch:
            if req.is_finished:
                req.state = RequestState.FINISHED
                req.finished_at = time.time()
                self.finished.append(req)
                self.kv_blocks_used -= req.kv_blocks_used
                req.kv_blocks_used = 0
            else:
                new_active.append(req)
        self.active_batch = new_active
    
    def _admit_new_requests(self):
        """Admit as many queued requests as possible."""
        while self.queue and self._can_admit_new(self.queue[0]):
            req = self.queue.popleft()
            blocks = self._kv_blocks_needed(req)
            req.kv_blocks_used = blocks
            self.kv_blocks_used += blocks
            req.state = RequestState.PREFILLING
            req.prefill_started_at = time.time()
            self.active_batch.append(req)
    
    def schedule_step(self) -> dict:
        """
        Execute one scheduler iteration.
        Returns the batch to process this step.
        """
        # 1. Remove finished requests
        self._remove_finished()
        
        # 2. Admit new requests
        self._admit_new_requests()
        
        # 3. Categorize batch
        prefilling = [r for r in self.active_batch if r.state == RequestState.PREFILLING]
        decoding = [r for r in self.active_batch if r.state == RequestState.DECODING]
        
        # 4. Simulate one step (in real vLLM, this would call the model)
        # Prefill: process the full prompt in this step (or chunked)
        for req in prefilling:
            req.state = RequestState.DECODING
            req.decode_started_at = time.time()
        
        # Decode: generate one token
        for req in decoding:
            req.output_tokens_generated += 1
            # Grow KV cache if crossing block boundary
            new_blocks = self._kv_blocks_needed(req)
            if new_blocks > req.kv_blocks_used:
                delta = new_blocks - req.kv_blocks_used
                if self.kv_blocks_used + delta <= self.config.max_kv_blocks:
                    self.kv_blocks_used += delta
                    req.kv_blocks_used = new_blocks
        
        return {
            "prefill_count": len(prefilling),
            "decode_count": len(decoding),
            "kv_utilization": self.kv_blocks_used / self.config.max_kv_blocks,
            "queue_depth": len(self.queue),
            "active": len(self.active_batch),
        }
    
    def metrics(self) -> dict:
        """Aggregate metrics across finished requests."""
        if not self.finished:
            return {}
        
        ttfts = [r.decode_started_at - r.arrival_time for r in self.finished if r.decode_started_at]
        e2es = [r.finished_at - r.arrival_time for r in self.finished]
        
        return {
            "requests_finished": len(self.finished),
            "avg_ttft_ms": (sum(ttfts) / len(ttfts) * 1000) if ttfts else 0,
            "avg_e2e_ms": (sum(e2es) / len(e2es) * 1000) if e2es else 0,
            "total_output_tokens": sum(r.output_tokens_generated for r in self.finished),
        }
```

**Enterprise features demonstrated:**
- Iteration-level batching (not request-level)
- KV block allocation/reclamation
- Prefill and decode phase distinction
- Backpressure via KV budget
- Support for chunked prefill (config-driven)
- Observability metrics

---

## Q3-Q17: Additional Coding Challenges (Condensed)

### Q3. Implement Prefix Cache With Radix Tree ⭐⭐⭐⭐
Store prefix→KV block mappings in a radix tree. On new request, walk tree matching prompt prefix. Reuse matched blocks. Insert new prefix segments. Reference counting for eviction. Track hit rate.

### Q4. Compute Speculative Decoding Acceptance Rate & Speedup ⭐⭐⭐⭐
Given: target model latency, draft model latency, K speculation length, acceptance probability α per token. Compute expected speedup = (1 + α + α² + ... + α^K) / (1 + K × draft_ratio). Show when speculation loses (low α, expensive draft).

### Q5. Cost-Per-Token Calculator ⭐⭐⭐⭐
Given: GPU config, hourly cost, achieved throughput (tok/sec), utilization %. Compute: cost per 1M tokens. Compare to API pricing. Compute break-even token volume. Include amortized TCO options.

### Q6. Build a Benchmark Harness With Proper Percentiles ⭐⭐⭐⭐
Concurrent request generation (asyncio). Warm-up phase separate. Track per-request TTFT + TPOT + E2E. Compute p50, p95, p99 (not averages). Sequence-length distribution. Report throughput at each percentile.

### Q7. Implement PagedAttention Block Allocation ⭐⭐⭐
Simulate paged KV cache: blocks of 16 tokens, page table per request, allocate blocks on demand as request grows. Free blocks when request finishes. Handle out-of-memory (backpressure). Fragmentation metrics.

### Q8. GPU Memory Budget Calculator ⭐⭐⭐
Given: GPU HBM, model params, quantization format, activation buffer estimate. Compute available KV cache budget. Derive max batch size × max context combinations that fit. Output planning table.

### Q9. Implement Simple Draft-And-Verify Speculative Decoder ⭐⭐⭐⭐
Draft model generates K token candidates. Target verifies in parallel (single forward pass). Accept longest matching prefix. Compute per-request acceptance rate. Handle backfill after rejection.

### Q10. Continuous Batching With Chunked Prefill ⭐⭐⭐⭐
Extend scheduler from Q2. When request has long prompt, break prefill into chunks. Interleave prefill chunks with decode steps for other requests. Prevent long prefills from starving decodes.

### Q11. Multi-LoRA Serving Adapter Manager ⭐⭐⭐
Track loaded LoRAs, LRU eviction, memory budget. On request, look up tenant's LoRA. Apply to base model output. Manage adapter cold-start (lazy load from disk).

### Q12. Quantization Quality Regression Framework ⭐⭐⭐⭐
Given: baseline model output on golden set, quantized model output on same set. Compute quality metrics (BLEU, ROUGE, LLM-judge, exact match). Detect regressions. Block deploy if quality drops > threshold.

### Q13. Roofline Model Calculator ⭐⭐⭐⭐
Given: operation FLOPs, memory bytes moved, GPU peak FLOPS, GPU memory bandwidth. Compute arithmetic intensity. Determine compute-bound vs memory-bound. Identify optimization axis.

### Q14. Warm Pool Manager for GPU Inference ⭐⭐⭐
Manages N warm model instances. Predictive scaling based on traffic forecast. Cold-standby vs warm-standby vs active. Health checks. Graceful drain on scale-down.

### Q15. Multi-Tenant Cost Attribution ⭐⭐⭐
Per-request tagging (tenant, user, feature). Real-time aggregation to Redis. Cost breakdown by tenant. Budget enforcement (soft warning at 80%, hard limit). Monthly attribution reports.

### Q16. Disaggregated Prefill/Decode Router ⭐⭐⭐⭐
Router receives request, sends prefill to prefill cluster. Transfers KV cache to decode cluster. Continues decode there. Manages state transitions. Handles failures on either side.

### Q17. Full-Stack Inference Observability ⭐⭐⭐⭐
OpenTelemetry instrumentation covering: request received → auth → routing → prefill (with token count) → decode iterations → KV cache metrics → response. Per-span attributes for gen_ai.model, tokens, cost. Sampling strategy: 100% errors + 10% success.
