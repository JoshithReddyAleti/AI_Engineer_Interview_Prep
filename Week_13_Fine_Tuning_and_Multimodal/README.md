# 🧪 Week 13 — Fine-Tuning as a Discipline + Multimodal AI

> **Maps to:** [Episode_13_Fine_Tuning_and_Multimodal_The_Training_Layer](https://github.com/JoshithReddyAleti/Episode_13_Fine_Tuning_and_Multimodal_The_Training_Layer)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**This is the training layer. Fine-tuning is where AI engineering meets ML research. Multimodal is where AI meets the physical world.**

Fine-tuning is the specialty where post-training researchers at Anthropic, OpenAI, Meta AI, Google DeepMind, and Mistral spend careers. Multimodal is where the next $500B of enterprise value ships — document intelligence, voice agents, video understanding.

Both disciplines have a signature interview pattern: **the interviewer probes whether you understand the mathematics, or just the API**. If you can derive LoRA from the parameter-efficient optimization perspective, articulate why DPO reformulates RLHF's reward maximization as a classification loss, explain why CLIP's contrastive loss requires large batch sizes, or reason about why LLaVA's projection layer works — you're at the frontier-lab bar.

If you can't — you're building on someone else's abstraction.

**Signal calibration:**
- **Junior:** "I used LoRA in unsloth."
- **Mid:** "I chose LoRA rank=16 because the loss curve stabilized."
- **Senior:** "For our domain shift on 15K examples, r=8 gave 96% of full-fine-tune quality at 0.4% of the parameters, and I targeted only Q/V projections because attention captures the domain shift while MLP layers retain general capability."
- **Staff+ / Frontier Lab:** "We chose LoRA over full fine-tune because our 15K SFT examples couldn't move a 70B model without catastrophic forgetting on MT-Bench. r=8 targeting Q/V/K/O + optional gate_proj gave us the intrinsic rank of the domain shift (Hu et al. 2021 shows most task-specific adaptations have effective rank <8). Full-fine-tune baseline degraded MMLU by 3.2 points; LoRA held it flat."

Week 13 gets you to the staff+ / frontier-lab level.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 32 | LoRA math from first principles, QLoRA's NF4 quantization, DPO derivation from RLHF, RLHF pipeline mechanics, catastrophic forgetting, FSDP vs DeepSpeed ZeRO, CLIP contrastive loss, ViT patches, VLM architectures (LLaVA/Flamingo/native), Whisper internals, ColPali late interaction |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 17 | Implement LoRA from scratch, DPO loss, KL-constrained PPO update, chat template + loss masking, FSDP wrapping strategy, ColPali late interaction retriever, Whisper streaming buffer |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | Enterprise fine-tuning platform, multi-tenant LoRA adapter serving at scale, document intelligence pipeline, voice agent system, video understanding at scale |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 17 | Fine-tuning degraded general capability, DPO training diverged, synthetic data collapsed the model, catastrophic forgetting on customer traffic, VLM hallucinated on medical images |
| [05_Terminology_Differentiation.md](05_Terminology_Differentiation.md) | Reference | LoRA vs QLoRA vs DoRA vs IA³ vs prefix tuning; SFT vs continued pretraining vs instruction tuning; RLHF vs DPO vs IPO vs KTO vs ORPO vs SimPO; ViT vs CLIP vs SigLIP; and more |

## The 19 Sections Covered (Full Episode 13)

### Part A — Fine-Tuning as a Discipline (11 sections)

| # | Section | Interview Territory |
|---|---|---|
| 1 | Foundations | When to fine-tune vs prompt vs RAG. The economics: $10K-$100K per fine-tuning run at scale, weeks of dataset work. Prompt/RAG solve 80% of problems. Fine-tune only when quality bar unmet by both. |
| 2 | Dataset Construction | The unsexy 80%. Distribution matching, coverage, label quality, deduplication, contamination detection. Instruction format standards (Alpaca, ShareGPT, ChatML). Preference data for DPO. |
| 3 | Synthetic Data Generation | Self-Instruct (Wang et al. 2022), Evol-Instruct (WizardLM), distillation from stronger models, the model-collapse risk (Shumailov et al. 2023). |
| 4 | Parameter-Efficient Fine-Tuning (PEFT) | LoRA (Hu et al. 2021), QLoRA (Dettmers et al. 2023), DoRA (Liu et al. 2024), IA³, prompt tuning. Rank selection theory. Target module choice. |
| 5 | Supervised Fine-Tuning (SFT) | Loss masking on user turns, chat templates (ChatML, Llama-3 format), sample packing, catastrophic forgetting, learning rate schedules for base vs instruct models. |
| 6 | Preference Optimization | RLHF pipeline (Christiano 2017, InstructGPT 2022). DPO (Rafailov 2023). IPO, KTO, ORPO, SimPO. Bradley-Terry model. When DPO fails. |
| 7 | Continued Pretraining | Domain adaptation. Learning rate warmup considerations. When to use vs SFT (large unlabeled corpora that shift distribution). |
| 8 | Training Infrastructure | DDP, FSDP (Zhao et al. 2023), DeepSpeed ZeRO-1/2/3 (Rajbhandari 2020), Unsloth (2× speedup), Axolotl, LlamaFactory. Communication cost by stage. |
| 9 | Fine-Tuning Evaluation | Perplexity is misleading. MT-Bench, IFEval, task-specific holdouts. Preference model eval for RLHF. Regression on general benchmarks (MMLU, GSM8K, HumanEval). |
| 10 | LoRA Adapter Serving | Multi-LoRA serving in vLLM. Hot-swappable adapters at inference. Per-tenant fine-tuned model economics. |
| 11 | Model Merging | Model soups (Wortsman 2022), TIES (Yadav 2023), DARE, task arithmetic (Ilharco 2022). Training-free combination of fine-tuned models. |

### Part B — Multimodal AI (8 sections)

| # | Section | Interview Territory |
|---|---|---|
| 12 | Multimodal Foundations | Modality alignment strategies. Cross-attention (Flamingo, Alayrac 2022) vs projection (LLaVA, Liu 2023) vs native training (GPT-4o, Gemini). Discrete tokenization (Chameleon). |
| 13 | Vision-Language Models | ViT (Dosovitskiy 2020) — patch tokenization mechanics. CLIP (Radford 2021) — contrastive InfoNCE loss, 400M image-text pairs. SigLIP (Zhai 2023) — sigmoid loss, better small-batch. LLaVA family. |
| 14 | Document Intelligence | LayoutLM (Xu 2019) — text + layout + image. Donut (Kim 2021) — OCR-free. ColPali (Faysse 2024) — late interaction over vision embeddings. |
| 15 | Audio and Speech | Whisper (Radford 2022) — encoder-decoder, 680K hours, log-mel spectrograms. Streaming ASR (chunk-and-emit, Distil-Whisper). TTS (XTTS, ElevenLabs). Voice agent latency budget. |
| 16 | Video Understanding | Frame sampling strategies. VideoLLaMA, LLaVA-Video. Long-video challenges (memory quadratic in tokens). Temporal reasoning. |
| 17 | Multimodal Embeddings | CLIP embeddings for cross-modal search. Vision search, product search, brand safety. |
| 18 | Multimodal Generation | Diffusion (DDPM, Ho 2020, Stable Diffusion). Video generation (Sora, Runway). Voice cloning ethics. |
| 19 | Multimodal Production | Storage cost realities (media 1000× text), latency (media processing 10× text), streaming architectures, cross-modal cache. |

## The Multi-Axis Comparison Framework (What Interviewers Probe)

Every fine-tuning interview reduces to this decision tree:

```
Problem: LLM output quality insufficient
    │
    ├─→ Can prompt engineering fix it? (Try first — days, cheap)
    │       │
    │       └─ If not:
    │
    ├─→ Can RAG fix it? (Missing knowledge? — weeks, moderate cost)
    │       │
    │       └─ If not:
    │
    ├─→ SFT: Do we have (query, ideal_response) pairs? (Format/style adaptation)
    │       │
    │       └─ Yes: LoRA rank 8-32, 1K-100K examples, days-weeks
    │
    ├─→ DPO/RLHF: Do we have preference data or reward signal? (Behavior alignment)
    │       │
    │       └─ Yes: DPO first (simpler), RLHF if DPO plateaus
    │
    ├─→ Continued Pretraining: Large distribution shift with unlabeled data?
    │       │
    │       └─ Yes: 100M+ tokens minimum, weeks-months, GPU-heavy
    │
    └─→ Full Fine-Tune: Fundamental capability change needed?
            │
            └─ Rarely justified. 100× cost of LoRA, 90%+ can be done with LoRA.
```

Interviewers respect candidates who arrive at fine-tuning as the LAST option, not the first.

## Cost Reality (What Frontier Labs Actually Spend)

- **Small LoRA (8B model, 10K examples, r=16):** 1 × A100 for 4h ≈ $8. Trivial.
- **Medium LoRA (70B model, 50K examples, r=32):** 4 × A100 for 12h ≈ $200. Reasonable.
- **QLoRA (70B model, 50K examples):** 1 × A100 for 24h ≈ $50. QLoRA's memory savings enable this.
- **Full fine-tune (70B, 100K examples):** 32 × H100 for 24h ≈ $10K. Serious spend.
- **RLHF (70B, PPO with reward model):** 100+ H100s for 1-2 weeks ≈ $200K-$500K. Frontier-lab scale.
- **Continued pretraining (7B, 100B tokens):** 512 × H100s for 2 weeks ≈ $2M. Rare outside labs.

Fine-tuning is engineering, not clicking a button. Cost, timeline, and risk are all real.

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1-11 | LLM Fundamentals → Observability | (see main README) | ✅ |
| 12 | Inference & Model Serving | [Episode_12](https://github.com/JoshithReddyAleti/Episode_12_Inference_and_Model_Serving_The_Systems_Under_the_Model) | ✅ |
| **13** | **Fine-Tuning + Multimodal: The Training Layer** ← you are here | [Episode_13](https://github.com/JoshithReddyAleti/Episode_13_Fine_Tuning_and_Multimodal_The_Training_Layer) | ✅ |
