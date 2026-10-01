# 📖 Week 13 — Terminology Differentiation Reference

> **Focus:** Every set of similar-sounding fine-tuning + multimodal terms with precise definitions, when to use, when NOT to use, common interview traps.
>
> **Frontier lab context:** these distinctions matter because the wrong choice wastes $10K-$100K in compute. Interviewers probe them to see if you'll blow that money.

---

# Section 1: Parameter-Efficient Fine-Tuning Methods

## LoRA vs QLoRA vs DoRA vs IA³ vs Prefix Tuning vs Prompt Tuning

### LoRA (Low-Rank Adaptation, Hu et al. 2021)

**Definition:** Frozen base model + trainable low-rank matrices `A, B` where `ΔW = BA`, `r << d`.

**Params trained:** ~0.1-1% of base model.

**Use when:** default for task adaptation. Works for 90% of enterprise fine-tuning.

**Don't use when:** large distribution shift requiring new capability injection.

---

### QLoRA (Quantized LoRA, Dettmers et al. 2023)

**Definition:** LoRA where the frozen base is quantized to 4-bit (NF4). Enables 65B fine-tuning on single 48GB GPU.

**Params trained:** same as LoRA.

**VRAM impact:** 4× reduction vs FP16 base.

**Use when:** GPU memory constrained. Fine-tuning 70B on 1×A100 or 1×H100.

**Don't use when:** you have abundant GPU memory — vanilla LoRA is slightly faster.

**Interview trap:** QLoRA is LoRA + quantization + double quantization + paged optimizers. Not just "quantized LoRA."

---

### DoRA (Weight-Decomposed LoRA, Liu et al. 2024)

**Definition:** Decomposes weights into magnitude + direction. Adapts both separately.

**Params trained:** LoRA + small magnitude vector.

**Quality:** matches full FT closer than LoRA at same rank.

**Use when:** quality-critical LoRA fine-tuning.

**Don't use when:** speed/simplicity matters more than 1-2% quality bump.

---

### IA³ (Infused Adapter, Liu et al. 2022)

**Definition:** Learns scaling vectors for K, V, MLP activations. `h = h * λ` where `λ` is learned.

**Params trained:** ~0.01% of base.

**Use when:** extreme parameter efficiency, classification tasks.

**Don't use when:** generation tasks — weaker than LoRA.

---

### Prefix Tuning (Li & Liang 2021)

**Definition:** Prepends learned vectors to K and V at every transformer layer.

**Use when:** legacy — mostly superseded by LoRA.

**Don't use when:** you're starting fresh. Use LoRA.

---

### Prompt Tuning (Lester 2021)

**Definition:** Learns soft prompt embeddings prepended to input.

**Params trained:** tiniest (~0.01%).

**Use when:** classification with very large models (T5-XXL, 100B+).

**Don't use when:** smaller models or generation — quality gap significant.

---

### 🎯 The Distinction That Matters

| Method | % params | Quality | VRAM | Speed | Default use |
|---|---|---|---|---|---|
| **LoRA** | 0.1-1% | 96% of full FT | 4× reduced | Fast | Yes — production default |
| **QLoRA** | 0.1-1% | 95% of full FT | 8× reduced | Slower than LoRA | When VRAM constrained |
| **DoRA** | 0.1-1.1% | 98% of full FT | 4× reduced | 10% slower than LoRA | When quality > speed |
| **IA³** | 0.01% | Weaker on gen | Minimal | Fast | Classification only |
| **Prefix Tuning** | 0.1% | Similar to LoRA | Minimal | Fast | Legacy |
| **Prompt Tuning** | 0.01% | Only large models | Minimal | Fast | 100B+ classification |

**Rule:** LoRA/QLoRA/DoRA cover 99% of cases.

---

# Section 2: Fine-Tuning Types

## SFT vs Instruction Tuning vs Chat Tuning vs Continued Pretraining

### SFT (Supervised Fine-Tuning)

**Definition:** Umbrella term. Any supervised training on pretrained LLM.

**Use when:** referring to the general category.

---

### Instruction Tuning

**Definition:** SFT on (instruction, response) pairs.

**Purpose:** Teaches following instructions in general.

**Landmark:** FLAN (Wei 2021), T0 (Sanh 2021).

**Use when:** taking a base LLM and making it instruction-following.

---

### Chat Tuning

**Definition:** SFT on multi-turn conversations with roles.

**Purpose:** Teaches conversational flow, role separation.

**Loss masking:** ONLY on assistant tokens.

**Use when:** creating a chat model from a base or instruction-tuned model.

---

### Continued Pretraining

**Definition:** Same objective as pretraining (next-token prediction) on a new corpus. Unsupervised.

**Purpose:** Injects NEW KNOWLEDGE. Domain adaptation.

**Data:** raw unlabeled text. Millions-billions of tokens.

**Cost:** 10-100× SFT.

**Use when:** large domain shift with unlabeled corpus available.

**Don't use when:** you just need behavior change — SFT is enough.

---

### 🎯 The Distinction That Matters

| Type | Teaches | Data format | Data volume | Cost |
|---|---|---|---|---|
| **SFT (general term)** | Anything | (input, output) | Variable | Variable |
| **Instruction tuning** | Follow instructions | (instruction, response) | 10K-1M examples | $500-$10K |
| **Chat tuning** | Conversational flow | Multi-turn conversations | 10K-1M examples | $500-$10K |
| **Continued pretraining** | Domain knowledge | Raw text | 100M-100B tokens | $50K-$5M |

**Interview trap:** "we fine-tuned" is ambiguous. Specify SFT vs continued pretraining. They cost 100× differently.

---

# Section 3: Preference Optimization

## RLHF vs DPO vs IPO vs KTO vs ORPO vs SimPO

### RLHF (Reinforcement Learning from Human Feedback)

**Definition:** Three-stage pipeline: SFT → reward model → PPO. Policy trained with reward + KL constraint.

**Cost:** $200K-$500K for 70B.

**Use when:** frontier lab, hardest problems, willing to invest in RL infrastructure.

**Don't use when:** enterprise, small teams, tight budgets.

---

### DPO (Direct Preference Optimization, Rafailov 2023)

**Definition:** Reformulates RLHF as classification loss on preference pairs. No reward model, no RL.

**Cost:** ~$5K for 70B.

**Use when:** default for enterprise preference optimization.

---

### IPO (Identity Preference Optimization, Azar 2023)

**Definition:** Different loss function that's more robust to overfitting.

**Use when:** small preference datasets, DPO showing overfitting.

---

### KTO (Kahneman-Tversky Optimization, Ethayarajh 2024)

**Definition:** Uses binary desirable/undesirable signal, not paired preferences.

**Use when:** you have thumbs-up/thumbs-down data (not pairs).

---

### ORPO (Odds Ratio Preference Optimization, Hong 2024)

**Definition:** Combines SFT + preference optimization into ONE training run.

**Use when:** simplify pipeline, single-stage training preferred.

---

### SimPO (Simple Preference Optimization, Meng 2024)

**Definition:** No reference model, length-normalized loss.

**Use when:** DPO shows verbosity bias, memory-constrained.

---

### 🎯 The Distinction That Matters

| Method | Ref model needed? | Data format | Cost | Best for |
|---|---|---|---|---|
| **RLHF** | Yes (SFT baseline) | Pairs | $200K+ | Frontier labs |
| **DPO** | Yes | Pairs | $5K | Enterprise default |
| **IPO** | Yes | Pairs | $5K | Small datasets |
| **KTO** | Yes | Binary signals | $5K | Rating data |
| **ORPO** | No | Pairs + SFT | $10K | Simpler pipeline |
| **SimPO** | No | Pairs | $5K | Length issues |

**Rule:** DPO by default. Move to others for specific reasons.

---

# Section 4: Distributed Training

## DDP vs FSDP vs ZeRO-1 vs ZeRO-2 vs ZeRO-3 vs Tensor Parallelism vs Pipeline Parallelism

### DDP (Distributed Data Parallel)

**What's split:** Data batches.

**What's replicated:** Full model, gradients, optimizer state — on every GPU.

**Communication:** All-reduce gradients per step.

**Use when:** model fits on one GPU. Scale horizontally.

---

### FSDP FULL_SHARD (= ZeRO-3)

**What's split:** Weights, gradients, optimizer state.

**Communication:** All-gather weights per layer (much more).

**Use when:** model doesn't fit on one GPU. Need fast interconnect.

---

### FSDP SHARD_GRAD_OP (= ZeRO-2)

**What's split:** Gradients, optimizer state. Weights replicated.

**Use when:** weights fit, but full replication of grad+opt state doesn't.

---

### ZeRO-1

**What's split:** Optimizer state only.

**Use when:** optimizer state is the bottleneck (Adam with large models).

---

### Tensor Parallelism

**What's split:** Individual weight matrices across GPUs.

**Communication:** All-reduce per matmul. Very high bandwidth need.

**Use when:** during INFERENCE for large models. NVLink required.

**Don't use when:** across nodes without ultra-fast interconnect.

---

### Pipeline Parallelism

**What's split:** Different layers on different GPUs.

**Communication:** Activations between stages.

**Use when:** model too big for a node. Latency-tolerant workloads.

**Don't use when:** latency-critical inference — pipeline bubbles hurt.

---

### 🎯 The Distinction That Matters

| Strategy | Memory savings | Communication | Best for |
|---|---|---|---|
| **DDP** | None | Low | Small models |
| **ZeRO-1 / FSDP grad_op** | Some | Medium | Medium models |
| **ZeRO-3 / FSDP FULL_SHARD** | Most | High | Large model training |
| **Tensor Parallel** | Yes (per matmul) | Very high (per matmul) | Inference of huge models |
| **Pipeline Parallel** | Yes (per layer) | Low | Cross-node training |

---

# Section 5: Vision Encoders

## CLIP vs SigLIP vs ViT vs DINO vs BLIP

### ViT (Vision Transformer, Dosovitskiy 2020)

**Definition:** Transformer over image patches. General-purpose vision backbone.

**Trained on:** supervised classification (ImageNet) or self-supervised.

**Use when:** general vision backbone. Foundation for everything.

---

### CLIP (Radford 2021)

**Definition:** Dual encoder (ViT + text transformer) trained with contrastive loss on 400M image-text pairs.

**Enables:** cross-modal search, VLM initialization.

**Use when:** vision-language tasks, cross-modal retrieval.

---

### SigLIP (Zhai 2023)

**Definition:** CLIP successor with sigmoid loss. Works better at small batch sizes.

**Use when:** modern default over CLIP. 4× training efficiency.

---

### DINO / DINOv2 (Caron 2021, Oquab 2023)

**Definition:** Self-supervised ViT. No text supervision.

**Use when:** vision features without needing image-text pairs. Segmentation, detection.

---

### BLIP / BLIP-2 (Li 2022, 2023)

**Definition:** Vision-language model with bootstrapping captions.

**Use when:** image captioning, visual QA (older architecture).

---

### 🎯 The Distinction That Matters

| Model | Objective | Training data | Best for |
|---|---|---|---|
| **ViT** | Classification/self-supervised | ImageNet or unlabeled | General backbone |
| **CLIP** | Contrastive image-text | 400M pairs | Cross-modal search |
| **SigLIP** | Sigmoid contrastive | 4B pairs | Modern CLIP replacement |
| **DINOv2** | Self-supervised | 142M images | Detection, segmentation |
| **BLIP-2** | Multi-objective vision-language | 100M+ pairs | Captioning, VQA |

---

# Section 6: VLM Architectures

## LLaVA vs Flamingo vs GPT-4o vs Chameleon

### LLaVA (Liu 2023)

**Architecture:** CLIP encoder → MLP projection → LLM. Simplest VLM design.

**Training:** two-stage (projection alignment, then instruction tuning).

**Use when:** cost-effective, fine-tuneable enterprise VLM.

---

### Flamingo (Alayrac 2022)

**Architecture:** Cross-attention layers in LLM attend to vision features.

**Use when:** interleaved image-text training, few-shot visual QA.

---

### GPT-4o / Gemini (Native Multimodal)

**Architecture:** Trained from scratch with all modalities interleaved.

**Use when:** best quality, willing to pay API costs.

---

### Chameleon (Meta 2024)

**Architecture:** Discrete tokenization of images. Everything is tokens.

**Use when:** research; unified generation across modalities.

---

### 🎯 The Distinction That Matters

| Architecture | Design | Training cost | Quality | Fine-tuneable? |
|---|---|---|---|---|
| **LLaVA** | Projection + frozen LLM | Cheap ($500) | Good | Yes (LoRA) |
| **Flamingo** | Cross-attention | Medium | Better | Complex |
| **Native (GPT-4o)** | Trained together | $M+ | Best | No (closed) |
| **Chameleon** | Discrete tokens | Medium | Emerging | Yes |

---

# Section 7: Speech and Audio

## ASR vs TTS vs Streaming ASR vs Voice Cloning

### ASR (Automatic Speech Recognition)

**Definition:** Speech → text. Whisper, Google Speech-to-Text, AssemblyAI.

**Standard:** Whisper large-v3, offline batch.

---

### Streaming ASR

**Definition:** ASR with incremental output as audio arrives.

**Trade-off:** ~5% quality loss vs batch. Enables voice UX.

**Use when:** voice agents, live captions.

---

### TTS (Text-to-Speech)

**Definition:** Text → speech.

**Modern:** ElevenLabs, XTTS, Bark. Voice cloning capable.

---

### Voice Cloning

**Definition:** Generate speech in a specific voice from a sample.

**Ethics:** consent required. Deepfake concerns.

**Use when:** with proper permissions and safeguards.

---

### 🎯 The Distinction That Matters

| Task | Direction | Latency | Best model 2026 |
|---|---|---|---|
| **Batch ASR** | Speech → text | High | Whisper large-v3 |
| **Streaming ASR** | Speech → text (live) | Low | Distil-Whisper, Deepgram |
| **TTS** | Text → speech | Low | ElevenLabs Flash |
| **Voice cloning** | Text + sample → speech | Medium | ElevenLabs, XTTS |

---

# Section 8: Fine-Tuning Alternatives

## Fine-Tune vs Prompt Engineer vs RAG vs Model Merging

### Fine-Tune

**Definition:** Modify model weights via training.

**When:** behavior/style needs to change consistently, RAG can't provide it.

**Cost:** $500-$10K+.

---

### Prompt Engineering

**Definition:** Modify input to get better outputs. No training.

**When:** default first try. Solves 60-80% of problems.

**Cost:** engineering time only.

---

### RAG (Retrieval-Augmented Generation)

**Definition:** Retrieve relevant context, inject into prompt.

**When:** need new/current knowledge not in training data.

**Cost:** infrastructure for retrieval, per-query cost.

---

### Model Merging

**Definition:** Combine multiple fine-tuned models' weights without training.

**When:** you already have multiple fine-tuned variants. Want combined capability.

**Cost:** compute for merging (minutes) + evaluation.

---

### 🎯 The Distinction That Matters

**Decision order:** prompt → RAG → SFT → DPO → continued pretraining. Try left-most first.

| Approach | Data need | Cost | Timeline | Reversibility |
|---|---|---|---|---|
| **Prompt eng** | None | ~$0 | Days | Instant |
| **RAG** | Corpus | Infra + query | Weeks | Instant |
| **SFT (LoRA)** | 1K-100K examples | $500-$5K | Weeks | Yes (swap adapter) |
| **Continued pretraining** | 100M+ tokens | $50K+ | Months | Difficult |
| **Merging** | Multiple models | Minutes | Days | Instant |

---

# Section 9: Data Concepts

## Preference Data vs SFT Data vs Contamination vs Distillation

### SFT Data

**Format:** (input, ideal_output) pairs.

**Volume:** 1K-1M examples.

---

### Preference Data

**Format:** (input, chosen_output, rejected_output).

**Volume:** 1K-100K pairs typical.

**Used for:** DPO, RLHF, KTO.

---

### Contamination

**Definition:** Test set data appears in training set. Invalidates evaluation.

**Detection:** exact match check, n-gram overlap, embedding similarity.

**Common:** synthetic data generated from LLM that saw benchmarks.

---

### Distillation

**Definition:** Train smaller model on outputs of larger model.

**Use for:** creating cheaper serving model.

**Risk:** hard-coded errors from teacher propagate to student.

---

### 🎯 The Distinction That Matters

| Data type | Format | Use for | Watch for |
|---|---|---|---|
| **SFT** | (input, output) | Supervised training | Quality, coverage |
| **Preference** | (input, pair) | Preference optimization | Label noise, bias |
| **Contamination** | Test in train | (anti-pattern) | Detection required |
| **Distillation** | Teacher outputs | Model compression | Error inheritance |

---

# Section 10: Evaluation

## Perplexity vs Task Accuracy vs Human Eval vs LLM-Judge vs Preference Rate

### Perplexity

**Definition:** exp(cross-entropy) on held-out data. Measures how surprised the model is.

**Use when:** monitoring training convergence.

**Don't use when:** measuring fine-tune quality — misleading. Low perplexity ≠ good model.

---

### Task Accuracy

**Definition:** exact match on holdout of your specific task.

**Use when:** measuring specific capability (math, code, classification).

---

### Human Eval

**Definition:** humans rate outputs directly.

**Cost:** $5-$50 per rating.

**Use when:** ground truth quality assessment. Expensive but reliable.

---

### LLM-as-Judge

**Definition:** stronger LLM (GPT-4) rates outputs.

**Cost:** ~$0.01 per rating.

**Bias:** same-family judges biased toward same-family outputs.

**Use when:** scalable proxy for human eval.

---

### Preference Rate (Win Rate)

**Definition:** in head-to-head comparison, what % of the time is model A preferred over model B?

**Use when:** comparing two models. Cleaner signal than absolute scores.

---

### 🎯 The Distinction That Matters

| Metric | Reliability | Cost | Use for |
|---|---|---|---|
| **Perplexity** | Low for quality | Free | Training monitoring |
| **Task accuracy** | High for specific tasks | Free | Regression tests |
| **Human eval** | Gold standard | High | Final validation |
| **LLM-judge** | Good proxy | Cheap | Continuous eval |
| **Win rate** | Great for comparison | Medium | A/B decisions |

**Best practice:** use ALL of these. Perplexity for training, task accuracy for regression, LLM-judge for continuous, human eval for launch, win rate for model comparison.

---

## 🎯 The Meta-Lesson

The training layer has more alphabet soup than any other layer of AI engineering. Frontier lab interviewers probe these distinctions BECAUSE they map to real dollar decisions.

Loose terminology → loose thinking → $50K wasted on continued pretraining when SFT would have done it → didn't get the offer.

Precise terminology → precise thinking → chose LoRA rank 8 targeting Q,V with 15K carefully-deduped examples → deployed in a week for $200 → got the offer.

The vocabulary IS the skill.
