# 🧰 Week 6 — Frameworks & Fine-Tuning Interview Prep

> **Maps to:** [Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide](https://github.com/JoshithReddyAleti/Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the widest interview surface in AI engineering.** Every enterprise AI job posting mentions at least three of these tools. Interviewers test whether you can PICK the right framework, JUSTIFY the choice, and know the TRADE-OFFS — not just recite features.

Week 6 covers two massive topics that belong together:
- **Part 1 — Frameworks:** LangChain, LlamaIndex, LangGraph, CrewAI, MCP, prompt engineering, structured outputs, observability, guardrails, evaluation
- **Part 2 — Fine-Tuning:** LoRA, QLoRA, PEFT, RLHF, DPO, quantization (GPTQ/AWQ/GGUF), HuggingFace ecosystem, model serving

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 25 | Each framework in depth, prompt patterns, structured output approaches, fine-tuning methods, when-to-use-what |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 15 | LangChain LCEL, LangGraph state, CrewAI crews, MCP servers, LoRA config, DPO training |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 15 | Framework selection at scale, multi-agent architectures, fine-tuning infrastructure, model serving |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 15 | LangChain vs raw code debates, framework migrations, fine-tuning failures, cost decisions |

## Key Topics Tested

**Framework Selection (the #1 question):**
- When LangChain is the right choice vs when it's overkill
- LangChain vs LlamaIndex — the actual difference in production
- Why LangGraph exists when LangChain already has agents
- When multi-agent (CrewAI) beats single-agent
- The role of MCP in the ecosystem

**Prompt Engineering (deep patterns):**
- Zero-shot, few-shot, chain-of-thought, self-consistency
- ReAct pattern (reasoning + acting)
- Tree of Thoughts vs linear reasoning
- Meta-prompting and prompt chaining
- Role-based prompting and system prompt design

**Structured Output (6 approaches):**
- Prompt-based JSON (weakest)
- JSON mode
- Function calling (OpenAI, Anthropic)
- Instructor library
- Outlines (constrained decoding)
- Pydantic schemas — production standard

**Observability Stack:**
- LangSmith (LangChain-native)
- Langfuse (open source, framework-agnostic)
- Arize Phoenix (evaluation-focused)
- OpenTelemetry for LLMs
- Cost tracking as a first-class concern

**Fine-Tuning (the full spectrum):**
- Full fine-tuning vs PEFT — cost math
- LoRA — how it actually works, rank selection
- QLoRA — 4-bit base model + LoRA adapters
- Adapters, prefix tuning, prompt tuning — the PEFT family
- RLHF — 3-stage complexity
- DPO — the practical alternative to RLHF
- When to fine-tune vs RAG vs prompt engineering

**Quantization:**
- Why quantize (memory, latency, cost)
- INT8 vs INT4 vs mixed precision
- bitsandbytes (dynamic quantization for QLoRA)
- GPTQ (post-training, GPU-optimized)
- AWQ (activation-aware, best quality-speed)
- GGUF (llama.cpp, CPU/laptop deployment)

**Model Serving:**
- Ollama — local development
- vLLM — production high-throughput
- TGI (Text Generation Inference) — HuggingFace's serving
- ONNX Runtime — edge/mobile
- Triton — enterprise multi-model

**Guardrails & Evaluation:**
- Input validation and output filtering
- NeMo Guardrails
- RAGAS, DeepEval, custom eval frameworks
- A/B testing LLM applications

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1 | [LLM Fundamentals](../Week_1_LLM_Fundamentals/) | [Understanding_LLMs_From_The_Inside_Out](https://github.com/JoshithReddyAleti/Understanding_LLMs_From_The_Inside_Out) | ✅ |
| 2 | [Python for AI](../Week_2_Python_For_AI/) | [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters) | ✅ |
| 3 | [Tool Calling, APIs & Validation](../Week_3_Tool_Calling_APIs_Validation/) | [Building_AI_Project-Blueprint_for_Begin](https://github.com/JoshithReddyAleti/Building_AI_Project-Blueprint_for_Begin) | ✅ |
| 4 | [End-to-End AI Projects](../Week_4_End_To_End_AI_Projects/) | [Your_First_End_To_End_AI_Project](https://github.com/JoshithReddyAleti/Episode_4_Your_First_End_To_End_AI_Project) | ✅ |
| 5 | [RAG & Augmented Generation](../Week_5_RAG_Augmented_Generation/) | [Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation) | ✅ |
| **6** | **Frameworks & Fine-Tuning** ← you are here | [AI_Frameworks_and_Fine_Tuning_Complete_Guide](https://github.com/JoshithReddyAleti/Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide) | ✅ |
| 7 | Memory & State in AI Systems | Coming soon | 🔜 |
