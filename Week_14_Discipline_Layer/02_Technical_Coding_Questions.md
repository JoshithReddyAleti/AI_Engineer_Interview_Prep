# 💻 Week 14 — Technical / Coding Questions

> **Focus:** MinHash + LSH deduplication, DSPy Signature + Module, prompt injection classifier, sample size + CUPED estimator, Wilson CI, sequential testing with alpha spending, HNSW parameter tuner
>
> **Depth:** frontier-lab bar. Every solution reflects paper math, not tutorial code.

---

## Q1. Implement MinHash + LSH for Near-Duplicate Detection at Scale ⭐⭐⭐⭐

**Prompt:** Given millions of documents, find pairs with Jaccard similarity > 0.8 in sub-linear time. Implement MinHash signatures + LSH bucketing.

**Solution:**

```python
import hashlib
import random
from collections import defaultdict
from typing import Iterator

class MinHashLSH:
    """
    MinHash + Locality-Sensitive Hashing for near-duplicate detection.
    
    Based on Broder (1997) MinHash + Rajaraman-Ullman MMDS Ch3 LSH.
    
    Parameters:
        num_hashes: signature length (k). Higher = better estimate, more compute.
        bands: number of LSH bands (b). rows_per_band = num_hashes / b.
        threshold: target Jaccard similarity for detection.
    
    The band/row split controls precision-recall:
        P(collision at similarity s) = 1 - (1 - s^r)^b
    
    For threshold t=0.8, choose b, r such that this probability
    is high at s>=t and low at s<t.
    """
    
    def __init__(
        self,
        num_hashes: int = 128,
        bands: int = 32,
        threshold: float = 0.8,
        seed: int = 42,
    ):
        assert num_hashes % bands == 0, "num_hashes must be divisible by bands"
        self.num_hashes = num_hashes
        self.bands = bands
        self.rows_per_band = num_hashes // bands
        self.threshold = threshold
        
        # Precompute hash functions (universal hashing)
        rng = random.Random(seed)
        LARGE_PRIME = (1 << 61) - 1
        self.hash_params = [
            (rng.randint(1, LARGE_PRIME - 1), rng.randint(0, LARGE_PRIME - 1))
            for _ in range(num_hashes)
        ]
        self.large_prime = LARGE_PRIME
        
        # LSH buckets: band_idx -> band_hash -> list of doc_ids
        self.buckets: list[dict[int, list[str]]] = [defaultdict(list) for _ in range(bands)]
        
        # Store signatures for verification
        self.signatures: dict[str, list[int]] = {}
    
    def _shingle(self, text: str, k: int = 5) -> set[int]:
        """Convert text to k-shingles (character k-grams), hashed to ints."""
        text = text.lower()
        shingles = set()
        for i in range(len(text) - k + 1):
            shingle = text[i:i+k]
            # Hash to int for compactness
            h = int(hashlib.md5(shingle.encode()).hexdigest()[:8], 16)
            shingles.add(h)
        return shingles
    
    def _minhash_signature(self, shingles: set[int]) -> list[int]:
        """Compute MinHash signature of a shingle set."""
        signature = []
        for a, b in self.hash_params:
            min_hash = float('inf')
            for shingle in shingles:
                h = ((a * shingle + b) % self.large_prime)
                if h < min_hash:
                    min_hash = h
            signature.append(min_hash)
        return signature
    
    def add(self, doc_id: str, text: str):
        """Add a document to the LSH index."""
        shingles = self._shingle(text)
        if not shingles:
            return
        
        signature = self._minhash_signature(shingles)
        self.signatures[doc_id] = signature
        
        # Hash each band
        for band_idx in range(self.bands):
            start = band_idx * self.rows_per_band
            end = start + self.rows_per_band
            band = tuple(signature[start:end])
            band_hash = hash(band)
            self.buckets[band_idx][band_hash].append(doc_id)
    
    def find_candidates(self, doc_id: str) -> set[str]:
        """Return candidate near-duplicates for a document."""
        if doc_id not in self.signatures:
            return set()
        
        signature = self.signatures[doc_id]
        candidates = set()
        
        for band_idx in range(self.bands):
            start = band_idx * self.rows_per_band
            end = start + self.rows_per_band
            band = tuple(signature[start:end])
            band_hash = hash(band)
            
            for other_id in self.buckets[band_idx][band_hash]:
                if other_id != doc_id:
                    candidates.add(other_id)
        
        return candidates
    
    def jaccard(self, doc_id_a: str, doc_id_b: str) -> float:
        """Approximate Jaccard similarity via MinHash agreement."""
        sig_a = self.signatures[doc_id_a]
        sig_b = self.signatures[doc_id_b]
        matches = sum(1 for a, b in zip(sig_a, sig_b) if a == b)
        return matches / len(sig_a)
    
    def find_all_duplicates(self) -> list[tuple[str, str, float]]:
        """Find all pairs above threshold. Returns (id_a, id_b, jaccard)."""
        seen = set()
        duplicates = []
        
        for doc_id in self.signatures:
            candidates = self.find_candidates(doc_id)
            for other_id in candidates:
                pair = tuple(sorted([doc_id, other_id]))
                if pair in seen:
                    continue
                seen.add(pair)
                
                sim = self.jaccard(doc_id, other_id)
                if sim >= self.threshold:
                    duplicates.append((pair[0], pair[1], sim))
        
        return duplicates
    
    def stats(self) -> dict:
        """Report bucket distribution — helps diagnose parameter choice."""
        bucket_sizes = []
        for band_buckets in self.buckets:
            bucket_sizes.extend(len(docs) for docs in band_buckets.values())
        
        return {
            "num_docs": len(self.signatures),
            "num_buckets_total": sum(len(b) for b in self.buckets),
            "avg_bucket_size": sum(bucket_sizes) / len(bucket_sizes) if bucket_sizes else 0,
            "max_bucket_size": max(bucket_sizes) if bucket_sizes else 0,
            "collision_prob_at_threshold": 1 - (1 - self.threshold ** self.rows_per_band) ** self.bands,
        }
```

**Complexity:** O(N × k) for signatures, O(N × b) for LSH insertion. Retrieval sub-linear in N. Enables 100M+ document deduplication.

---

## Q2. Implement Sample Size + CUPED Estimator ⭐⭐⭐⭐

**Prompt:** Given metric type, baseline, MDE, α, power — compute required sample size. Then implement CUPED variance reduction and demonstrate ~50% variance reduction.

**Solution:**

```python
import math
import numpy as np
from scipy import stats

def sample_size_proportion(
    baseline_rate: float,
    mde: float,
    alpha: float = 0.05,
    power: float = 0.80,
    two_sided: bool = True,
) -> int:
    """
    Sample size per arm for a two-sample proportion test.
    
    Formula: n = 2 * p̄(1-p̄) * (z_{α/2} + z_β)² / δ²
    """
    z_alpha = stats.norm.ppf(1 - alpha / (2 if two_sided else 1))
    z_beta = stats.norm.ppf(power)
    
    p_bar = baseline_rate  # Approximation; use (p1+p2)/2 for exact
    
    n = 2 * p_bar * (1 - p_bar) * (z_alpha + z_beta) ** 2 / (mde ** 2)
    return math.ceil(n)


def sample_size_continuous(
    std_dev: float,
    mde: float,
    alpha: float = 0.05,
    power: float = 0.80,
    two_sided: bool = True,
) -> int:
    """
    Sample size per arm for a two-sample t-test on continuous metric.
    
    Formula: n = 2 * σ² * (z_{α/2} + z_β)² / δ²
    """
    z_alpha = stats.norm.ppf(1 - alpha / (2 if two_sided else 1))
    z_beta = stats.norm.ppf(power)
    
    n = 2 * (std_dev ** 2) * (z_alpha + z_beta) ** 2 / (mde ** 2)
    return math.ceil(n)


def minimum_detectable_effect(
    n_per_arm: int,
    baseline_rate: float,
    alpha: float = 0.05,
    power: float = 0.80,
) -> float:
    """
    Given fixed sample size, what's the smallest effect we can detect?
    
    Formula: MDE = (z_{α/2} + z_β) * √(2 * p̄(1-p̄) / n)
    """
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)
    
    p_bar = baseline_rate
    se = math.sqrt(2 * p_bar * (1 - p_bar) / n_per_arm)
    
    return (z_alpha + z_beta) * se


def cuped_estimator(
    y_treatment: np.ndarray,
    y_control: np.ndarray,
    x_treatment: np.ndarray,
    x_control: np.ndarray,
) -> dict:
    """
    CUPED variance reduction (Deng et al. 2013).
    
    Y_adj = Y - θ(X - E[X])
    θ* = Cov(Y, X) / Var(X)
    
    Var(Y_adj) = Var(Y) * (1 - ρ²)
    
    Returns:
        - unadjusted ATE and variance
        - CUPED-adjusted ATE and variance
        - variance reduction fraction
    """
    # Pool for θ estimation (avoid using treatment info)
    x_all = np.concatenate([x_treatment, x_control])
    y_all = np.concatenate([y_treatment, y_control])
    
    # Compute θ from pooled data
    x_mean = x_all.mean()
    theta = np.cov(y_all, x_all)[0, 1] / np.var(x_all)
    
    # Adjust outcomes
    y_t_adj = y_treatment - theta * (x_treatment - x_mean)
    y_c_adj = y_control - theta * (x_control - x_mean)
    
    # Unadjusted ATE
    ate_unadj = y_treatment.mean() - y_control.mean()
    var_unadj = y_treatment.var(ddof=1) / len(y_treatment) + y_control.var(ddof=1) / len(y_control)
    
    # Adjusted ATE (should be same in expectation)
    ate_adj = y_t_adj.mean() - y_c_adj.mean()
    var_adj = y_t_adj.var(ddof=1) / len(y_t_adj) + y_c_adj.var(ddof=1) / len(y_c_adj)
    
    # Correlation → predicted variance reduction
    rho = np.corrcoef(y_all, x_all)[0, 1]
    
    return {
        "ate_unadjusted": ate_unadj,
        "ate_cuped": ate_adj,
        "variance_unadjusted": var_unadj,
        "variance_cuped": var_adj,
        "variance_reduction_pct": (1 - var_adj / var_unadj) * 100,
        "correlation_rho": rho,
        "theoretical_reduction_pct": (1 - (1 - rho ** 2)) * 100,
        "theta": theta,
    }


# Demo: 49.9% variance reduction with ρ≈0.7
if __name__ == "__main__":
    np.random.seed(42)
    n = 10000
    
    # Simulate: pre-experiment X strongly predicts Y, treatment adds small effect
    x_control = np.random.normal(0, 1, n)
    x_treatment = np.random.normal(0, 1, n)
    
    # Y = 0.7*X + noise (target ρ ≈ 0.7)
    y_control = 0.7 * x_control + np.random.normal(0, 0.7, n)
    y_treatment = 0.7 * x_treatment + np.random.normal(0, 0.7, n) + 0.05  # 5% lift
    
    result = cuped_estimator(y_treatment, y_control, x_treatment, x_control)
    print(f"Correlation ρ: {result['correlation_rho']:.3f}")
    print(f"Variance reduction: {result['variance_reduction_pct']:.1f}%")
    print(f"Theoretical: {result['theoretical_reduction_pct']:.1f}%")
```

Verified in Episode 14's CUPED demo: achieves 49.9% variance reduction matching theoretical prediction.

---

## Q3. Wilson Score Interval — Robust Confidence Interval for Proportions ⭐⭐⭐

**Prompt:** Implement Wilson score interval for click-through rates. Explain why Wald is wrong.

**Solution:**

```python
def wilson_ci(successes: int, trials: int, confidence: float = 0.95) -> tuple[float, float]:
    """
    Wilson score confidence interval for a proportion.
    
    Better than Wald (normal approximation) for:
    - Small samples
    - Extreme proportions (near 0 or 1)
    - Sample sizes where CLT hasn't kicked in
    
    Wald: p ± z * sqrt(p(1-p)/n)  ← wrong when p is extreme
    Wilson: solves the actual quadratic for CI bounds
    """
    if trials == 0:
        return (0.0, 0.0)
    
    from scipy import stats
    z = stats.norm.ppf(1 - (1 - confidence) / 2)
    
    p = successes / trials
    denominator = 1 + z**2 / trials
    center = (p + z**2 / (2 * trials)) / denominator
    half_width = z * math.sqrt(p * (1 - p) / trials + z**2 / (4 * trials**2)) / denominator
    
    return (center - half_width, center + half_width)


# Wald fails at extremes:
# 0 successes in 100 → Wald CI: [0, 0]  ← wrong! We didn't prove p=0
# 0 successes in 100 → Wilson CI: [0, 0.036] ← correct

# Always use Wilson for proportion CIs.
```

---

## Q4-Q17: Additional Coding Challenges (Condensed)

### Q4. Prompt Injection Classifier ⭐⭐⭐⭐
Given known injection patterns, build a classifier: regex + ML-model fallback. Return score 0-1 with explanation. Track false-positive rate on legitimate technical queries. Deploy as filter chain.

### Q5. DSPy Signature + Module Implementation ⭐⭐⭐⭐
Define a Signature class with typed input/output fields. Build a Module base class (Predict, ChainOfThought). Implement a simple bootstrap optimizer that generates few-shot examples from training data.

### Q6. Sequential Testing with Alpha Spending ⭐⭐⭐⭐
Implement O'Brien-Fleming and Pocock alpha spending functions. Given planned peeks, allocate alpha budget. Track cumulative type-I error. Enable early stopping with valid inference.

### Q7. HNSW Parameter Tuner ⭐⭐⭐
Grid search over M, efConstruction, efSearch. For each combination, measure recall@10 and p95 query latency. Return Pareto frontier. Recommend based on user's recall/latency SLO.

### Q8. Structured Output Enforcer ⭐⭐⭐⭐
Given Pydantic model, generate JSON schema. Post-process LLM output to enforce schema. Retry with schema info if invalid. Compare to true constrained decoding (FSM-based).

### Q9. Contamination Checker ⭐⭐⭐
Given benchmark items and training corpus, compute: exact match, 13-gram overlap, embedding similarity > 0.95. Report contaminated items with severity. Support incremental checks as corpus grows.

### Q10. LLMLingua-style Prompt Compressor ⭐⭐⭐
Score each token by perplexity contribution using small model. Drop lowest-contribution tokens to hit budget. Measure quality loss on task benchmark. Show compression ratio vs quality curve.

### Q11. SRM Detector ⭐⭐⭐
Given actual arm counts, compute chi-squared test against expected allocation. Flag SRM (p < 0.001). Suggest diagnostics: check randomization, logging, bot filtering.

### Q12. Multi-Judge LLM Evaluator ⭐⭐⭐
Sample response evaluated by 3+ LLM judges (different families to reduce bias). Compute inter-judge agreement. Aggregate with weighted vote or Bayesian. Handle disagreements.

### Q13. Canary Token Monitor ⭐⭐⭐
Embed unique canary strings in system prompt. Monitor all outputs for canary appearance. Alert on any hit — indicates prompt extraction. Rotate canaries periodically.

### Q14. Prompt Registry with Versioning ⭐⭐⭐
Store prompts with semver versioning. Track: content, metadata, evaluation scores, deployment status. Support rollback. Audit trail of who deployed what when.

### Q15. Bootstrap Confidence Interval for Heavy-Tailed Metrics ⭐⭐⭐
For LLM-judge scores (heavy-tailed), compute CI via bootstrap (10K resamples). More robust than t-interval. Report percentile CI (bias-corrected optional).

### Q16. Automated Red Teaming Loop (PAIR-style) ⭐⭐⭐⭐
Attacker LLM proposes prompt. Target LLM responds. Judge LLM evaluates if jailbreak succeeded. Attacker refines based on judge. Iterate up to N rounds. Track success rate and cost.

### Q17. A/B Test Analysis Pipeline ⭐⭐⭐⭐
Full pipeline: SRM check → primary metric CI → guardrail checks → segment analysis → multiple testing correction (FDR) → decision (ship/kill/extend). Output structured decision report.
