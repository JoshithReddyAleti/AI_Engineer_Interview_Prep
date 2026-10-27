# 💻 Week 16 — Technical / Coding Questions

> **Focus:** Risk scoring engine, lane router, threat register generator, guardrail calibrator (confusion matrix), OPA policy engine, ADR/RDR schemas, faithfulness scorer, PII redactor with tokenization, hash-chained audit trail
>
> **Depth:** Senior AI Security Architect bar. Every solution reflects production considerations, not tutorial code.

---

## Q1. Multi-Factor Risk Scoring Engine with Lane Routing ⭐⭐⭐⭐

**Prompt:** Implement the 7-factor risk scoring model with dominant-factor logic that routes to Standard / Elevated Risk / Experimentation lanes.

**Solution:**

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class Lane(Enum):
    EXPERIMENTATION = "experimentation"
    STANDARD = "standard"
    ELEVATED_RISK = "elevated_risk"


@dataclass
class RiskFactors:
    """7-factor multi-dimensional risk scoring. Each factor 1-5."""
    data_sensitivity: int       # public=1, internal=2, confidential=3, PII/PHI=4, restricted=5
    model_autonomy: int         # passive=1, suggestion=2, recommendation=3, action=4, agent=5
    decision_impact: int        # informational=1, minor=2, moderate=3, major=4, life-safety=5
    regulatory_scope: int       # none=1, general=2, industry=3, HIPAA/GDPR=4, EU AI Act high-risk=5
    user_population: int        # dev=1, internal=2, partner=3, customer=4, public/vulnerable=5
    external_exposure: int      # air-gapped=1, VPN=2, partner=3, authenticated-web=4, public=5
    reversibility: int          # seconds=1, minutes=2, hours=3, days=4, never=5
    
    def __post_init__(self):
        for name, val in self.__dict__.items():
            if not 1 <= val <= 5:
                raise ValueError(f"{name} must be 1-5, got {val}")
    
    def all_scores(self) -> dict[str, int]:
        return {
            "data_sensitivity": self.data_sensitivity,
            "model_autonomy": self.model_autonomy,
            "decision_impact": self.decision_impact,
            "regulatory_scope": self.regulatory_scope,
            "user_population": self.user_population,
            "external_exposure": self.external_exposure,
            "reversibility": self.reversibility,
        }


@dataclass
class LaneAssignment:
    lane: Lane
    dominant_factors: list[str]
    rationale: str
    all_scores: dict[str, int]
    max_score: int
    top_3_sum: int


class RiskTieringEngine:
    """
    Multi-factor risk scoring with dominant-factor routing.
    
    Rules (in priority order):
    1. ANY factor = 5 → Elevated Risk (dominant factor override)
    2. Top-3 sum >= 12 → Elevated Risk (compound risk)
    3. All factors <= 3 AND total <= 15 → Experimentation
    4. Everything else → Standard
    """
    
    def assess(self, factors: RiskFactors) -> LaneAssignment:
        scores = factors.all_scores()
        max_score = max(scores.values())
        sorted_scores = sorted(scores.items(), key=lambda x: -x[1])
        top_3 = sorted_scores[:3]
        top_3_sum = sum(v for _, v in top_3)
        total = sum(scores.values())
        max_factor = max(scores.values())
        
        # Rule 1: any factor at maximum → Elevated Risk
        if max_score == 5:
            dominant = [name for name, val in scores.items() if val == 5]
            return LaneAssignment(
                lane=Lane.ELEVATED_RISK,
                dominant_factors=dominant,
                rationale=f"Factor(s) at maximum severity: {', '.join(dominant)}. "
                          f"Any single dimension at 5 requires Elevated Risk controls "
                          f"regardless of other factors.",
                all_scores=scores,
                max_score=max_score,
                top_3_sum=top_3_sum,
            )
        
        # Rule 2: compound top-3 risk
        if top_3_sum >= 12:
            dominant = [name for name, _ in top_3]
            return LaneAssignment(
                lane=Lane.ELEVATED_RISK,
                dominant_factors=dominant,
                rationale=f"Top-3 factors sum to {top_3_sum} (threshold: 12). "
                          f"Compound risk across {', '.join(dominant)}.",
                all_scores=scores,
                max_score=max_score,
                top_3_sum=top_3_sum,
            )
        
        # Rule 3: low risk across the board
        if total <= 15 and max_factor <= 3:
            return LaneAssignment(
                lane=Lane.EXPERIMENTATION,
                dominant_factors=[],
                rationale=f"Total score {total} (threshold: 15) and max factor {max_factor} "
                          f"(threshold: 3). Suitable for Experimentation lane.",
                all_scores=scores,
                max_score=max_score,
                top_3_sum=top_3_sum,
            )
        
        # Rule 4: default Standard
        return LaneAssignment(
            lane=Lane.STANDARD,
            dominant_factors=[name for name, val in sorted_scores[:2]],
            rationale=f"Total {total}, max {max_factor}. Standard lane appropriate.",
            all_scores=scores,
            max_score=max_score,
            top_3_sum=top_3_sum,
        )


# Test cases with real scenarios
if __name__ == "__main__":
    engine = RiskTieringEngine()
    
    scenarios = {
        "Internal chatbot for engineering docs": RiskFactors(
            data_sensitivity=2, model_autonomy=2, decision_impact=1,
            regulatory_scope=1, user_population=2, external_exposure=2, reversibility=1
        ),
        "Customer service agent (public, PII, autonomous)": RiskFactors(
            data_sensitivity=4, model_autonomy=4, decision_impact=3,
            regulatory_scope=4, user_population=5, external_exposure=5, reversibility=3
        ),
        "Healthcare diagnostic assistant": RiskFactors(
            data_sensitivity=5, model_autonomy=3, decision_impact=5,
            regulatory_scope=5, user_population=4, external_exposure=3, reversibility=5
        ),
        "Marketing copy generator (internal review)": RiskFactors(
            data_sensitivity=2, model_autonomy=2, decision_impact=2,
            regulatory_scope=2, user_population=2, external_exposure=2, reversibility=2
        ),
    }
    
    for name, factors in scenarios.items():
        result = engine.assess(factors)
        print(f"\n{name}")
        print(f"  Lane: {result.lane.value}")
        print(f"  Dominant: {result.dominant_factors}")
        print(f"  Rationale: {result.rationale}")
```

**Production notes:**
- Add per-initiative RDR persistence with hash chain (see Q5)
- Log every scoring event for audit trail
- Version the scoring rules (calibration changes over time)
- Add regulatory-specific overrides (EU AI Act high-risk → automatic Elevated regardless of scoring)

---

## Q2. Threat Register Generator (STRIDE + OWASP + ATLAS + MAESTRO) ⭐⭐⭐⭐

**Prompt:** Build a threat register that consolidates STRIDE-for-AI, OWASP LLM Top 10, MITRE ATLAS, and MAESTRO into a single prioritized list.

**Solution:**

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class Severity(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4


class Likelihood(Enum):
    RARE = 1
    UNLIKELY = 2
    POSSIBLE = 3
    LIKELY = 4
    ALMOST_CERTAIN = 5


@dataclass
class Threat:
    """Multi-framework threat with cross-references."""
    threat_id: str
    component: str  # which system component
    description: str
    stride_category: Optional[str] = None  # Spoofing, Tampering, Repudiation, Info, DoS, EoP
    owasp_llm_ref: Optional[str] = None    # LLM01-LLM10
    atlas_technique: Optional[str] = None  # AML.T####
    maestro_layer: Optional[str] = None    # Foundation, Fine-tuning, ..., Ecosystem
    severity: Severity = Severity.MEDIUM
    likelihood: Likelihood = Likelihood.POSSIBLE
    existing_controls: list[str] = field(default_factory=list)
    identified_gaps: list[str] = field(default_factory=list)
    mitigation_adr: Optional[str] = None  # ADR-YYYY-NNNN
    
    @property
    def risk_score(self) -> int:
        """Simple risk = severity × likelihood."""
        return self.severity.value * self.likelihood.value
    
    @property
    def priority(self) -> str:
        s = self.risk_score
        if s >= 16: return "P0 — immediate"
        elif s >= 12: return "P1 — this quarter"
        elif s >= 8: return "P2 — planned"
        elif s >= 4: return "P3 — backlog"
        else: return "P4 — accept"


class ThreatRegister:
    """Consolidated threat register spanning STRIDE + OWASP + ATLAS + MAESTRO."""
    
    def __init__(self, system_name: str):
        self.system_name = system_name
        self.threats: list[Threat] = []
    
    def add(self, threat: Threat):
        self.threats.append(threat)
    
    def prioritized(self) -> list[Threat]:
        return sorted(self.threats, key=lambda t: -t.risk_score)
    
    def by_framework(self, framework: str) -> list[Threat]:
        attr = {
            "stride": "stride_category",
            "owasp": "owasp_llm_ref",
            "atlas": "atlas_technique",
            "maestro": "maestro_layer",
        }[framework.lower()]
        return [t for t in self.threats if getattr(t, attr) is not None]
    
    def coverage_matrix(self) -> dict[str, int]:
        return {
            "STRIDE covered": len(self.by_framework("stride")),
            "OWASP LLM covered": len(self.by_framework("owasp")),
            "MITRE ATLAS covered": len(self.by_framework("atlas")),
            "MAESTRO covered": len(self.by_framework("maestro")),
        }
    
    def gap_report(self) -> list[str]:
        """Threats with identified gaps and no mitigation ADR."""
        return [
            f"{t.threat_id}: {t.description} — gaps: {t.identified_gaps}"
            for t in self.threats
            if t.identified_gaps and not t.mitigation_adr
        ]


# Example: build register for a customer-facing RAG agent
if __name__ == "__main__":
    register = ThreatRegister("Customer Service RAG Agent v2")
    
    register.add(Threat(
        threat_id="T-2026-047",
        component="Retrieval layer",
        description="Indirect prompt injection via user-uploaded document",
        stride_category="Tampering",
        owasp_llm_ref="LLM01",
        atlas_technique="AML.T0051",
        severity=Severity.HIGH, likelihood=Likelihood.LIKELY,
        existing_controls=["Input classifier on user query"],
        identified_gaps=["No filtering on retrieved doc content"],
        mitigation_adr="ADR-2026-1207",
    ))
    
    register.add(Threat(
        threat_id="T-2026-048",
        component="Agent tool layer",
        description="Excessive agency: agent can execute refunds without approval",
        stride_category="Elevation of privilege",
        owasp_llm_ref="LLM08",
        atlas_technique="AML.T0053",
        maestro_layer="Tool / plugin",
        severity=Severity.CRITICAL, likelihood=Likelihood.POSSIBLE,
        existing_controls=["Refund amount cap $50"],
        identified_gaps=["No human approval for repeated refunds to same customer"],
        mitigation_adr="ADR-2026-1208",
    ))
    
    register.add(Threat(
        threat_id="T-2026-049",
        component="Response layer",
        description="Sensitive information disclosure via hallucination",
        stride_category="Information disclosure",
        owasp_llm_ref="LLM06",
        severity=Severity.HIGH, likelihood=Likelihood.LIKELY,
        existing_controls=["PII detector in output"],
        identified_gaps=["No detection for training-data verbatim reproduction"],
    ))
    
    print(f"Threat register: {register.system_name}\n")
    print("Coverage:")
    for framework, count in register.coverage_matrix().items():
        print(f"  {framework}: {count}")
    
    print("\nPrioritized threats:")
    for t in register.prioritized():
        print(f"  {t.threat_id} [{t.priority}] score={t.risk_score}: {t.description}")
    
    print("\nGaps needing ADRs:")
    for gap in register.gap_report():
        print(f"  {gap}")
```

---

## Q3. Guardrail Threshold Calibrator (Confusion Matrix) ⭐⭐⭐⭐

**Prompt:** Given labeled test data (safe vs unsafe), sweep guardrail thresholds and recommend the best threshold per delivery lane based on lane-specific cost function.

**Solution:**

```python
import numpy as np
from dataclasses import dataclass
from typing import Callable


@dataclass
class ConfusionMatrix:
    tp: int  # correctly allowed
    fp: int  # incorrectly allowed (BREACH — worst case)
    tn: int  # correctly blocked
    fn: int  # incorrectly blocked (usability harm)
    
    @property
    def fpr(self) -> float:
        return self.fp / (self.fp + self.tn) if (self.fp + self.tn) else 0
    
    @property
    def fnr(self) -> float:
        return self.fn / (self.fn + self.tp) if (self.fn + self.tp) else 0
    
    @property
    def precision(self) -> float:
        return self.tp / (self.tp + self.fp) if (self.tp + self.fp) else 0
    
    @property
    def recall(self) -> float:
        return self.tp / (self.tp + self.fn) if (self.tp + self.fn) else 0
    
    @property
    def f1(self) -> float:
        p, r = self.precision, self.recall
        return 2*p*r/(p+r) if (p+r) else 0


@dataclass
class LaneCostFunction:
    """Relative cost of FP (breach) vs FN (blocked legitimate)."""
    lane: str
    cost_fp: float  # cost per false positive (breach)
    cost_fn: float  # cost per false negative (usability)
    
    def total_cost(self, cm: ConfusionMatrix) -> float:
        return cm.fp * self.cost_fp + cm.fn * self.cost_fn


# Cost calibration per lane
LANE_COSTS = {
    "experimentation": LaneCostFunction("experimentation", cost_fp=10, cost_fn=1),
    "standard": LaneCostFunction("standard", cost_fp=100, cost_fn=1),
    "elevated_risk": LaneCostFunction("elevated_risk", cost_fp=10_000, cost_fn=1),
}


def evaluate_threshold(
    scores: np.ndarray,
    labels: np.ndarray,  # 1 = truly unsafe, 0 = truly safe
    threshold: float,
) -> ConfusionMatrix:
    """
    Score >= threshold → predicted unsafe (block).
    Score < threshold → predicted safe (allow).
    """
    predicted_unsafe = scores >= threshold
    
    tp = int(((~predicted_unsafe) & (labels == 0)).sum())  # allowed, was safe
    fp = int(((~predicted_unsafe) & (labels == 1)).sum())  # allowed, was unsafe — BREACH
    tn = int(((predicted_unsafe) & (labels == 1)).sum())   # blocked, was unsafe
    fn = int(((predicted_unsafe) & (labels == 0)).sum())   # blocked, was safe
    
    return ConfusionMatrix(tp=tp, fp=fp, tn=tn, fn=fn)


def find_optimal_threshold(
    scores: np.ndarray,
    labels: np.ndarray,
    cost_fn: LaneCostFunction,
    threshold_range: tuple = (0.0, 1.0),
    n_steps: int = 101,
) -> tuple[float, ConfusionMatrix, float]:
    thresholds = np.linspace(*threshold_range, n_steps)
    
    best_threshold = thresholds[0]
    best_cost = float("inf")
    best_cm = None
    
    for t in thresholds:
        cm = evaluate_threshold(scores, labels, t)
        cost = cost_fn.total_cost(cm)
        if cost < best_cost:
            best_cost = cost
            best_threshold = t
            best_cm = cm
    
    return float(best_threshold), best_cm, best_cost


# Example usage
if __name__ == "__main__":
    np.random.seed(42)
    # Simulate labeled test data: 5000 samples, 15% truly unsafe
    n = 5000
    labels = (np.random.rand(n) < 0.15).astype(int)
    # Scores: unsafe biased higher, safe biased lower, with overlap
    scores = np.where(labels == 1, 
                      np.random.beta(6, 2, n),
                      np.random.beta(2, 6, n))
    
    for lane, cost_fn in LANE_COSTS.items():
        t, cm, cost = find_optimal_threshold(scores, labels, cost_fn)
        print(f"\nLane: {lane}")
        print(f"  Recommended threshold: {t:.3f}")
        print(f"  FPR (breach rate): {cm.fpr:.4f}")
        print(f"  FNR (block rate on safe): {cm.fnr:.4f}")
        print(f"  Precision: {cm.precision:.3f}, Recall: {cm.recall:.3f}, F1: {cm.f1:.3f}")
        print(f"  Total cost: {cost:.1f}")
```

**Production notes:**
- Recalibrate quarterly with fresh production traffic
- Segment calibration by tenant tier (enterprise gets stricter thresholds)
- Add drift monitoring: alert if runtime FPR diverges from calibration set
- Adversarial red team traffic labeled separately

---

## Q4. OPA Policy Engine for AI Deployment ⭐⭐⭐⭐

**Prompt:** Implement a policy engine using Rego rules that enforces deployment policies per lane at API gateway.

**Solution:**

```python
# rego_policies.py — showing the policy structure; real deployment uses opa binary

REGO_POLICY = """
package ai_deployment.v1

# Default deny
default allow = false

# Lane-based provider restrictions
allow {
    input.action == "deploy"
    input.lane == "experimentation"
    input.data_classification in {"public", "internal_nonpii"}
}

allow {
    input.action == "deploy"
    input.lane == "standard"
    input.data_classification in {"public", "internal_nonpii", "confidential"}
    input.provider in approved_standard_providers
    input.has_adr == true
    input.has_rdr == true
}

allow {
    input.action == "deploy"
    input.lane == "elevated_risk"
    input.provider in approved_elevated_providers
    input.has_adr == true
    input.has_rdr == true
    input.has_threat_model == true
    input.has_red_team_report == true
    input.security_signoff == true
    valid_baa_if_phi
}

# BAA required for PHI
valid_baa_if_phi {
    input.data_classification != "phi"
}
valid_baa_if_phi {
    input.data_classification == "phi"
    input.provider in baa_covered_providers
    input.baa_reference != ""
}

# Approved provider lists (versioned, updated quarterly by CoE)
approved_standard_providers := {
    "openai", "anthropic", "azure_openai", "aws_bedrock", "google_vertex"
}

approved_elevated_providers := {
    "anthropic_enterprise", "openai_enterprise", "azure_openai", 
    "aws_bedrock", "google_vertex"
}

baa_covered_providers := {
    "anthropic_enterprise", "openai_enterprise", "azure_openai",
    "aws_bedrock", "google_vertex"
}

# Denial reasons for audit
denial_reasons[msg] {
    input.action == "deploy"
    input.lane == "elevated_risk"
    input.has_threat_model != true
    msg := "Elevated Risk deployments require a threat model (missing)"
}

denial_reasons[msg] {
    input.action == "deploy"
    input.data_classification == "phi"
    not input.provider in baa_covered_providers
    msg := sprintf("PHI requires BAA-covered provider; %v is not BAA-covered", [input.provider])
}
"""


# Python client wrapping OPA
import json
import subprocess

def evaluate_policy(input_data: dict) -> dict:
    """
    Evaluate OPA policy on input.
    Returns {allow: bool, denial_reasons: [...]}
    """
    # In production: use opa's Python SDK or REST endpoint
    # This is illustrative
    result = subprocess.run(
        ["opa", "eval", "-d", "policy.rego", "-i", "-", 
         "data.ai_deployment.v1"],
        input=json.dumps({"input": input_data}),
        capture_output=True, text=True,
    )
    return json.loads(result.stdout)["result"][0]["expressions"][0]["value"]


if __name__ == "__main__":
    # Test scenarios
    scenarios = [
        {
            "action": "deploy",
            "lane": "experimentation",
            "data_classification": "public",
            "provider": "openai",
        },
        {
            "action": "deploy",
            "lane": "elevated_risk",
            "data_classification": "phi",
            "provider": "openai",  # not BAA — should fail
            "has_adr": True,
            "has_rdr": True,
            "has_threat_model": True,
            "has_red_team_report": True,
            "security_signoff": True,
        },
        {
            "action": "deploy",
            "lane": "elevated_risk",
            "data_classification": "phi",
            "provider": "azure_openai",
            "has_adr": True,
            "has_rdr": True,
            "has_threat_model": True,
            "has_red_team_report": True,
            "security_signoff": True,
            "baa_reference": "BAA-2026-0184",
        },
    ]
    
    for scenario in scenarios:
        # In production, call opa; here we simulate the logic
        print(f"Scenario: {scenario}")
        # ... evaluate
```

---

## Q5. Risk Decision Record (RDR) with Hash Chain ⭐⭐⭐⭐

```python
import hashlib
import json
import time
from dataclasses import dataclass, asdict, field


@dataclass
class RiskDecisionRecord:
    rdr_id: str
    initiative_name: str
    sponsor: str
    timestamp: float
    reviewers: list[str]
    risk_factors: dict[str, int]
    dominant_factors: list[str]
    lane_assigned: str
    framework_citations: list[str]  # ["NIST AI RMF MAP-2.3", "EU AI Act Art 6"]
    linked_adr_ids: list[str] = field(default_factory=list)
    linked_threat_model_id: str = ""
    guardrail_config_ref: str = ""
    review_frequency_days: int = 90
    next_review_date: str = ""
    auditor_signoff: str = "pending"
    prev_hash: str = ""
    record_hash: str = ""
    
    def compute_hash(self) -> str:
        d = asdict(self)
        d.pop("record_hash", None)
        canonical = json.dumps(d, sort_keys=True)
        return hashlib.sha256(canonical.encode()).hexdigest()


class RDRRegistry:
    """Hash-chained RDR store. Tamper-evident."""
    GENESIS = "0" * 64
    
    def __init__(self):
        self.records: list[RiskDecisionRecord] = []
    
    def append(self, rdr: RiskDecisionRecord) -> RiskDecisionRecord:
        rdr.prev_hash = self.records[-1].record_hash if self.records else self.GENESIS
        rdr.record_hash = rdr.compute_hash()
        self.records.append(rdr)
        return rdr
    
    def verify_chain(self) -> tuple[bool, int]:
        prev = self.GENESIS
        for i, r in enumerate(self.records):
            if r.prev_hash != prev: return False, i
            if r.compute_hash() != r.record_hash: return False, i
            prev = r.record_hash
        return True, -1
```

---

## Q6-Q17: Additional Coding Challenges (Condensed)

### Q6. Faithfulness Scorer (RAGAS-style) ⭐⭐⭐
Decompose response into claims via LLM. Verify each against retrieved context (NLI or LLM-as-judge). Return faithfulness score = supported / total. Support per-lane threshold.

### Q7. PII Detector with Tokenization ⭐⭐⭐
Layered detection: Presidio + regex + LLM verifier. Reversible tokenization for downstream re-identification. Track token vault with access logs.

### Q8. Prompt Injection Classifier Ensemble ⭐⭐⭐⭐
Ensemble Rebuff + Lakera Guard + custom fine-tuned classifier. Return injection probability + confidence. Escalate borderline cases to secondary review.

### Q9. Spotlighting Wrapper for Retrieved Content ⭐⭐⭐
Wrap every retrieved doc with distinct markers + instruction (Hines 2024). Support multi-marker for defense-in-depth.

### Q10. Dual-LLM Pattern Implementation ⭐⭐⭐⭐
Privileged operations (e.g., writing to database) require a second LLM's confirmation (Willison 2023). Privileged LLM never sees untrusted content directly; only structured commands from primary LLM.

### Q11. Canary Token Injection ⭐⭐⭐
Insert canary strings into prompts; monitor outputs for canary presence. Presence indicates prompt injection or model regurgitation. Rotate canaries.

### Q12. ADR Generator ⭐⭐⭐
Structured ADR template. Fields: context, decision, consequences, status, linked artifacts. Store git-tracked. Support diff view across ADR versions.

### Q13. Guardrail Metrics Emitter ⭐⭐⭐
Per-guardrail: trigger rate, TP/FP rates, latency. Emit to observability stack (see Week 11). Alert on threshold drift.

### Q14. Policy Test Harness ⭐⭐⭐
Positive + negative test cases per OPA policy. Run in CI. Fail build on policy regression.

### Q15. Incident Response Automation ⭐⭐⭐
Detect security event → auto-page on-call → gather context (recent logs, guardrail triggers, threat register) → create incident record → notify SIEM.

### Q16. Red Team Traffic Generator ⭐⭐⭐⭐
Generate adversarial test cases: prompt injection (direct + indirect), jailbreaks, data extraction attempts. Categorized by attack type. Track model resilience over time.

### Q17. Model Inventory Query Interface ⭐⭐⭐
Central register of all AI initiatives. Query by: lane, framework classification, data classification, provider, last review date. Feed board dashboards and audit responses.
