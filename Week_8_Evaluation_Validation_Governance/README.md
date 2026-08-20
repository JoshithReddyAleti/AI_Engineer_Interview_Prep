# 📊 Week 8 — Evaluation, Validation & Governance Interview Prep

> **Maps to:** [Episode_8_AI_Evaluation_Validation_and_Governance](https://github.com/JoshithReddyAleti/Episode_8_AI_Evaluation_Validation_and_Governance)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the interview surface where junior candidates die and senior candidates get offers.**

Every AI engineer can build a chatbot. Almost none of them can PROVE it works. Enterprise interviewers specifically probe evaluation, validation, and governance because that's where AI systems fail catastrophically in production — hallucinations that trigger lawsuits, bias findings that violate regulations, prompt injections that leak data, silent drift that erodes user trust.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 25 | All eval metrics with exact formulas, LLM-as-judge, RAGAS/DeepEval, hallucination taxonomy, bias/fairness, red teaming, drift, governance frameworks |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 16 | Build eval metrics from scratch, hallucination detectors, bias detectors, prompt injection tests, drift detection, snapshot tests, statistical significance |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 15 | Enterprise eval platform, continuous eval CI/CD, EU AI Act compliance, red teaming pipeline, A/B testing infrastructure, trust dashboards |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 16 | Hallucination lawsuits, bias findings after launch, regulatory audits, eval-vs-user disagreement, ground truth disputes, migration crises |

## The 11 Questions Every Production AI Must Answer

| # | Question | Section |
|---|---|---|
| 1 | Is the LLM output good? | `llm_evaluation` |
| 2 | Is retrieval good? Is the answer grounded? | `rag_evaluation` |
| 3 | Does the agent make good decisions? | `agent_evaluation` |
| 4 | Does data match expected schemas? | `validation` |
| 5 | Is the AI making things up? | `hallucination` |
| 6 | Is the AI fair and safe? | `bias_and_safety` |
| 7 | Can the AI be broken or abused? | `red_teaming` |
| 8 | How do we measure quality numerically? | `metrics` |
| 9 | How do we test non-deterministic systems? | `testing_strategies` |
| 10 | Is the system still working after deployment? | `production_monitoring` |
| 11 | Are we compliant, auditable, and responsible? | `governance` |

## Key Topics Tested

**Evaluation Methodologies:**
- LLM-as-judge (how it works, biases, mitigation)
- Human evaluation (inter-annotator agreement, calibration)
- Reference-based vs reference-free eval
- Pairwise comparison (A/B judgment)
- Golden datasets (construction, governance, evolution)

**RAG Evaluation Metrics (RAGAS):**
- Faithfulness (grounding to context)
- Answer Relevancy (alignment with question)
- Context Precision (are retrieved chunks relevant?)
- Context Recall (are all needed chunks retrieved?)
- Answer Correctness (matches ground truth)
- Groundedness (claims traceable to sources)

**Agent Evaluation:**
- Tool selection accuracy
- Task completion rate
- Step efficiency (unnecessary detours)
- Trajectory analysis (path optimality)
- Error recovery quality

**Classical NLP Metrics:**
- BLEU (n-gram precision, translation)
- ROUGE (n-gram recall, summarization)
- METEOR (BLEU + synonyms + stemming)
- BERTScore (embedding-based semantic similarity)
- Perplexity (probability of test set)

**Hallucination:**
- Intrinsic vs extrinsic hallucinations
- Closed-domain vs open-domain
- Claim extraction and verification
- Source attribution
- Factual consistency scoring (NLI-based)

**Bias & Safety:**
- Representation bias, aggregation bias, measurement bias
- Fairness metrics (demographic parity, equalized odds, equal opportunity)
- Toxicity classification
- Stereotype detection
- Safety guardrails

**Red Teaming:**
- Prompt injection (direct, indirect)
- Jailbreak techniques (roleplay, encoding, gradient)
- Data exfiltration attempts
- Adversarial inputs
- Automated red team frameworks

**Production Monitoring:**
- Concept drift, data drift, prediction drift
- Alert threshold design
- A/B testing statistical rigor
- Shadow deployment
- Online evaluation with sampling

**Governance & Compliance:**
- EU AI Act (risk tiers, obligations)
- NIST AI Risk Management Framework
- ISO 42001 (AI management systems)
- SOC 2 Type II considerations for AI
- Model cards, datasheets, transparency
- Audit trails, data lineage
- Right-to-explanation (Article 22 GDPR)

**Frameworks & Tools:**
- RAGAS — RAG-focused, LLM-as-judge based
- DeepEval — pytest-style eval framework
- LangSmith — LangChain-native observability & eval
- Arize Phoenix — open-source observability & drift
- Promptfoo — CLI-first prompt eval
- Giskard — bias & compliance testing
- TruLens — feedback functions, LLM observability

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1 | LLM Fundamentals | [Understanding_LLMs_From_The_Inside_Out](https://github.com/JoshithReddyAleti/Understanding_LLMs_From_The_Inside_Out) | ✅ |
| 2 | Python for AI | [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters) | ✅ |
| 3 | Tool Calling, APIs & Validation | [Building_AI_Project-Blueprint_for_Begin](https://github.com/JoshithReddyAleti/Building_AI_Project-Blueprint_for_Begin) | ✅ |
| 4 | End-to-End AI Projects | [Your_First_End_To_End_AI_Project](https://github.com/JoshithReddyAleti/Episode_4_Your_First_End_To_End_AI_Project) | ✅ |
| 5 | RAG & Augmented Generation | [Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation) | ✅ |
| 6 | Frameworks & Fine-Tuning | [AI_Frameworks_and_Fine_Tuning_Complete_Guide](https://github.com/JoshithReddyAleti/Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide) | ✅ |
| 7 | Memory & State in AI Systems | [Memory_and_State_in_AI_Systems](https://github.com/JoshithReddyAleti/Episode_7_Memory_and_State_in_AI_Systems) | ✅ |
| **8** | **Evaluation, Validation & Governance** ← you are here | [AI_Evaluation_Validation_and_Governance](https://github.com/JoshithReddyAleti/Episode_8_AI_Evaluation_Validation_and_Governance) | ✅ |
