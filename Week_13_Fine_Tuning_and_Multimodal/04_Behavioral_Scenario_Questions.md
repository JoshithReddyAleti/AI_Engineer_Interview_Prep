# 🎭 Week 13 — Behavioral & Scenario Questions

> **Focus:** Fine-tuning degraded general capability, DPO diverged, synthetic data collapsed the model, catastrophic forgetting on customer traffic, VLM hallucinated on medical images
>
> **Depth:** frontier-lab reasoning process — not just "what would you do" but "how you'd diagnose, what's the mechanism, how you'd prevent recurrence."

---

## Q1. Fine-Tuning Degraded General Capability by 4 MMLU Points ⭐⭐⭐⭐

**Scenario:** You LoRA fine-tuned Llama-70B on 20K customer-service examples. Task quality improved from 68% to 92% on holdout. Great. But MMLU dropped from 79.5 to 75.5. IFEval dropped 6 points. Product manager: "It's dumber than the base model on everything else. Users are complaining." What's the root cause and how do you fix it?

**Strong answer:**

"This is catastrophic forgetting — a well-known and well-documented failure mode of fine-tuning. My response, methodically:

**Understanding the mechanism (before jumping to fixes):**

The model's weights encoded general capability from pretraining. During fine-tuning, gradients pointed toward the customer service task. The optimizer moved weights to minimize task loss. But the loss function doesn't test general knowledge, so those directions in weight-space are unconstrained.

For LoRA specifically: even though only 0.1% of parameters trained, they multiplicatively modify every attention output. The effective weight change on general behavior can be much larger than the parameter count suggests.

**Confirming with diagnostics:**

Before fixing, I'd verify the pattern:
- Is it uniform degradation, or specific to certain MMLU categories? (Usually specific — e.g., math and science degrade more than history)
- Did loss curves show any sign of over-training? (Loss going too low often indicates memorization)
- What was the learning rate? What was the epoch count?

**Root cause hypotheses (ranked by likelihood):**

**Hypothesis 1 (most likely): Learning rate too high + too many epochs.**
Common mistake with LoRA: people use LR that's fine for LoRA magnitude but excessive when multiplied by the effective weight change. `2e-4` × 3 epochs on 20K examples = many strong updates.

**Hypothesis 2: Data lacks diversity.**
20K examples all customer service means the model saw ONLY customer service context for 20K iterations. It forgot other formats exist.

**Hypothesis 3: Alpha too high.**
If `alpha = 64, rank = 16`, the effective scaling is `4x`. That's aggressive. `alpha = rank` is safer.

**Hypothesis 4: Target modules include MLP.**
MLP layers encode more general capability than attention. Adapting `gate_proj`, `up_proj`, `down_proj` risks capability drift more than Q/V/K/O.

**Hypothesis 5: Base model was Instruct not Base.**
Fine-tuning Llama-3.1-70B-Instruct further pushes it away from its instruction-following training. Should have started from Base and taught the instruction format too.

**The fix (order of operations):**

**Step 1: Add replay data.**
Mix 20% general instruction data (Alpaca, OpenAssistant, WildChat) into the training batches. This preserves general behavior while still teaching customer service. **This alone typically recovers 60-80% of the regression.**

**Step 2: Reduce learning rate.**
`2e-4` → `5e-5` for LoRA. Slower training, but less catastrophic drift. Add cosine decay schedule.

**Step 3: Reduce epochs.**
3 epochs → 1-2. Early stopping based on general benchmark, not just task loss.

**Step 4: Reduce alpha.**
`alpha=64` → `alpha=16` (matching rank). Lower LoRA effective magnitude.

**Step 5: Target only attention.**
Remove MLP layers from target_modules. Q, K, V, O only.

**Step 6: Add regularization to reference.**
For DPO or later stages, KL constraint back to base. This is what β does in DPO.

**Step 7: Re-evaluate on comprehensive benchmark suite.**
- Task-specific: customer service quality
- General: MMLU, IFEval, MT-Bench, GSM8K, HumanEval
- Safety: HH-RLHF, Anthropic's harm eval
- Only ship if regression is <1 point on general benchmarks

**Communication:**

To PM: 'Root cause identified — catastrophic forgetting from over-training on narrow data. Standard failure mode with well-known mitigations. Redeploying with fix in [X] days. Expected: task quality maintained at ~90%, general benchmark regression <1 point.'

To team: honest postmortem. This is one of the top-3 fine-tuning failure modes. Add to team playbook. Update training recipe with replay data as default.

**Systemic changes:**

1. **Replay data by default.** Every training recipe includes 20% general instruction data.
2. **Comprehensive eval as CI/CD.** Every model checkpoint runs on MMLU, IFEval, MT-Bench. Block if any regresses > 1 point.
3. **Learning rate + alpha as configuration.** Not hardcoded. Recipe library with tested values per scenario.
4. **Base vs Instruct choice documented.** Team standard for when to use each.
5. **Fine-tuning launch checklist.** Includes 'have you tested general regression?' as a required gate.

**What I would NOT do:**

- Ship the degraded model with a caveat ('it's dumber but the task quality is higher!')
- Roll back to base model without fix (waste of the training investment)
- Blame the team for missing this (systematic gap, not personal)
- Immediately jump to full FT (usually the wrong fix — LoRA works with proper technique)

**The senior insight:**

Every fine-tuning practitioner faces this incident. The response separates seniors from staff+. Seniors fix this specific case; staff+ engineers turn it into a systematic prevention playbook so the team never hits it again.

Frontier lab context: at Anthropic/OpenAI/Meta, catastrophic forgetting is monitored on 20+ benchmarks simultaneously. Every checkpoint. Every training run. Never negotiated. That's the bar."

---

## Q2. DPO Training Diverged After 500 Steps ⭐⭐⭐⭐

**Scenario:** You started DPO on your SFT model with 50K preference pairs. First 500 steps looked healthy — loss decreasing, reward margin widening. Then loss exploded. Model outputs became repetitive, incoherent. What's the mechanism, how do you diagnose, how do you fix?

**Strong answer:**

"DPO divergence is a well-documented failure mode. Let me trace through the diagnostic process rigorously.

**The mechanism of DPO divergence:**

DPO's loss encourages the policy to increase log-probability of chosen over rejected. Without a proper KL constraint (β), the policy can drift arbitrarily far from the reference model. As it drifts, the log-probability ratios blow up. Once the ratio is very large, gradients become huge and unstable. The model catastrophically forgets language modeling ability.

Specifically: the DPO loss is `-log σ(β · (chosen_logratio - rejected_logratio))`. When `chosen_logratio - rejected_logratio` becomes very positive (chosen much more likely than reference), σ saturates at 1 and gradient vanishes for those samples. But the model has already deviated from being a language model.

**Immediate diagnosis:**

Load a checkpoint from step 400 (before divergence). Generate outputs. Compare to base.
Load checkpoint from step 500. Compare.

Check the metrics I would have logged:
- Chosen reward, rejected reward (over time)
- Reward margin (chosen - rejected)
- Log ratios (chosen_logratio, rejected_logratio individually)
- KL divergence from reference (implicit — should be moderate)

**What I'd expect to see with divergence:**
- Both chosen_reward and rejected_reward exploding (both very positive)
- Reward margin also growing (fine on paper, but underlying probabilities broken)
- Log ratios both huge

The trap: reward margin looks good even as the model is falling apart.

**Root cause hypotheses:**

**Hypothesis 1 (most common): β too small.**
`β = 0.1` is standard, but for some data distributions, needs to be higher (0.5). Small β means weak KL constraint, allows drift.

**Hypothesis 2: Learning rate too high.**
`5e-7` typical for DPO. Higher (`5e-6`) can work but risks instability.

**Hypothesis 3: Reference model not identical to SFT baseline.**
If reference is different model, log-ratios are meaningless. Common bug when loading wrong checkpoint.

**Hypothesis 4: Preference data has systematic bias.**
E.g., chosen always longer, always starts with 'Sure,' — model can hack these patterns without becoming better.

**Hypothesis 5: Label noise.**
If 20%+ of pairs are inverted (chosen actually worse), model receives contradictory signal, oscillates.

**Hypothesis 6: Training too long.**
DPO benefits from few epochs (1-3 typical). 5+ epochs on same data leads to overfitting the preferences.

**Fix strategy:**

**Step 1: Load pre-divergence checkpoint.** Don't try to save the diverged one.

**Step 2: Increase β.** `0.1` → `0.3` or `0.5`. Tighter KL constraint. Slower learning but stable.

**Step 3: Reduce learning rate.** Halve it. `5e-7` → `2e-7`.

**Step 4: Add length normalization.**
DPO consistently prefers longer responses due to sum of log-probs correlating with length. Divide log-probs by response length before computing loss (this is essentially SimPO).

**Step 5: Reduce epochs.** 1 pass through data. Add early stopping based on downstream benchmarks.

**Step 6: Sanity check data.**
Sample 100 pairs. Human-review: are chosen actually better than rejected? What's the label noise rate?

**Step 7: Switch to IPO or SimPO if issues persist.**
IPO has different loss shape that's more robust to overfitting. SimPO removes reference-model dependence, avoids drift issues.

**Prevention (for next time):**

1. **Continuous benchmark monitoring during training.** Every 100 steps, run 20 examples through generation, compute LLM-judge score, verify not degrading. Cost: minimal. Value: catches divergence immediately.

2. **Log KL divergence explicitly.** Compute KL(π || π_ref) on validation set every N steps. If KL grows > threshold, alert.

3. **Preference data QA.** Sample 5% of pairs, verify with second annotator (or GPT-4 judge). Estimate label noise rate before training.

4. **Curriculum learning.** Start with easier pairs (large quality gap), progress to harder (subtle differences).

**Communication:**

To team: 'DPO diverged at step 500. Root cause: β too small combined with our preference data's length bias. Redeploying with β=0.3, length normalization, and per-step benchmark monitoring. Expected: stable training, similar final quality.'

**Frontier lab context:**

At Anthropic and OpenAI, RL training runs are supervised by engineers watching dashboards in real-time. Every training curve is monitored. Divergence is caught within minutes and training halted for investigation. The stakes at frontier lab scale ($1M+ per run) demand this vigilance. At enterprise scale, per-step benchmark eval is a cheap simulation of the same discipline.

**The staff+ insight:**

DPO looks like supervised learning but has stability characteristics closer to RL. Treating it as vanilla SFT (set LR, train, done) leads to this exact failure. The senior discipline: monitor training as if it's RL, because it functionally is."

---

## Q3. Synthetic Data Collapsed the Model ⭐⭐⭐⭐

**Scenario:** You generated 100K training examples using GPT-4 (Evol-Instruct style). Fine-tuned Llama-70B. Task metrics improved on synthetic-similar test data but dropped 10 points on human-written test data. What happened, how do you fix it?

**Strong answer:**

"This is model collapse — training on synthetic data that has systematic distributional shifts vs real data. Documented in **Shumailov et al. 2023 ('The Curse of Recursion: Training on Generated Data Makes Models Forget')**.

**The mechanism:**

Synthetic data has narrower distribution than real data:
- Vocabulary bias (LLMs use certain phrases more)
- Structural bias (starts with 'Sure,' or 'Certainly')
- Reasoning bias (specific styles of explanation)
- Missing tails (rare-but-real patterns absent)

The fine-tuned model learns these shifted patterns. On synthetic-similar test data: high accuracy. On real human data: performance drops because human distribution has more variance the model wasn't exposed to.

**Diagnostic:**

Sample 100 examples from your synthetic data. Compare to 100 human-written examples of the same task type. Look for:
- **Vocabulary distribution:** what n-grams appear more often in synthetic?
- **Length distribution:** synthetic responses often longer, more structured
- **Format markers:** 'Sure,' 'Certainly,' 'Great question' openings
- **Structural patterns:** numbered lists, headers, bullet points overused
- **Reasoning style:** explicit chain-of-thought when humans wouldn't

Also: compute the model's confidence (log-probs) on both distributions. Synthetic will have higher confidence — model is memorizing surface patterns.

**Fixes (order matters):**

**Fix 1: Mix in real data.**
Add 20-30% real human examples to the training mix. Even a few thousand real examples reset the anchor to human distribution. **This is the standard mitigation.**

**Fix 2: Diverse synthetic generators.**
Use multiple LLMs (GPT-4, Claude, Gemini, Llama). Each has different biases. Averaging reduces any single-model bias contamination.

**Fix 3: Post-hoc filtering.**
Detect synthetic-flavored responses (long, structured, hedged) and filter them out or downweight.

**Fix 4: Self-Instruct rejection sampling.**
Generate, but reject responses that don't match human-written distribution on key metrics (length, structure, vocabulary).

**Fix 5: Human-in-the-loop refinement.**
Take synthetic examples, have humans edit them. Retains the coverage benefit while restoring human distribution.

**Prevention for next time:**

1. **Distribution monitoring during data collection.** Track: word frequencies, length histograms, structural markers. Alert if synthetic drifts too far from real.

2. **Held-out real test set from day 1.** Never optimize against synthetic-similar eval. Always evaluate on real distribution.

3. **Data lineage.** Every training example tagged: real or synthetic, which generator, when. Auditable.

4. **Continuous audit.** Sample 100 training examples/week for human review. Catches drift early.

**The specific data recipe I'd use:**

- 25% high-quality real human examples (whatever you have)
- 25% synthetic from GPT-4
- 25% synthetic from Claude 3.5
- 15% synthetic from Llama-3.1-70B (diversifies away from OpenAI/Anthropic style)
- 10% human-edited synthetic

**Cost consideration:**

Real human examples cost $2-$10 each (labeler time).
Synthetic examples cost $0.01 each (LLM API).

The temptation to go 100% synthetic is real. Resist it. The $5-$10K spent on real data prevents a $50K bad training run.

**Frontier lab practice:**

Meta's Llama-3 post-training used mix of ~10M human-generated + ~10M synthetic examples with careful quality filtering. Anthropic's Constitutional AI uses AI-generated critiques + human review. Neither uses 100% synthetic. Even at frontier scale, real data anchors the distribution.

**Communication:**

To team: 'Root cause: synthetic data distributional shift. Real vs synthetic test gap confirmed our training data doesn't match production. Redeploying with mixed real+synthetic recipe. Also adding distribution monitoring as CI check.'

To product: 'The model was optimized for a slightly different distribution than production. We're regenerating training data with a more representative mix. Expected quality parity within 2 weeks.'

**The staff+ signal:**

Understanding that synthetic data is a POWERFUL tool AND a subtle failure vector. Using it wisely: coverage augmentation, not replacement. Anchoring to real distribution always.

Frontier lab context: model collapse is one of the top research topics in 2024-2026 (Shumailov, Alemohammad, Bertrand papers). Enterprise engineers should know the failure exists and the standard mitigations. The teams that don't will train models that look great on their synthetic benchmarks and disappoint in production."

---

## Q4-Q17: Additional Behavioral Scenarios (Condensed)

### Q4. VLM Hallucinated Medical Details on X-Ray Images ⭐⭐⭐⭐
Enterprise medical customer complains: VLM described a fracture that wasn't in the image. Approach: (1) IMMEDIATE: disable VLM for medical images pending investigation. (2) Root cause: LLaVA-style VLMs hallucinate confidently on medical domain (out-of-distribution). Vision encoder wasn't trained on medical images. (3) Fix: domain-specific vision encoder (MedCLIP), require citations to visible regions, LLM-judge verification against image, human-in-loop for medical.

### Q5. LoRA Adapter Storage Exploded to 500GB ⭐⭐⭐
Tenant fine-tunes accumulated. No garbage collection. Storage cost spiraled. Approach: LRU eviction of unused LoRAs (>90 days no access → archive to Glacier). Per-tenant quota. Compression for cold LoRAs. Life-cycle policy. Track LoRA usage per adapter.

### Q6. Continued Pretraining Ran 6 Weeks Then Failed Eval ⭐⭐⭐⭐
$100K spent. Final model didn't beat baseline on target domain. Approach: retrospective — what went wrong? Common causes: (a) data quality issues discovered late, (b) domain data too narrow (still needed diverse pretraining mix), (c) wrong evaluation set, (d) hyperparameter miscalibration. Lessons: continuous eval during pretraining (not just at end), smaller-scale ablations before full run, diverse data mix.

### Q7. Convincing PM That Fine-Tuning Won't Solve Their Problem ⭐⭐⭐
PM: "The AI is bad, let's fine-tune." Approach: run their failing queries through better prompting first. Show that 70% improve. Only 30% remain. Those 30% — is it knowledge (RAG?), format (prompt template?), or truly missing capability (fine-tune)? Usually only 5-10% remains as fine-tune candidates. Save $50K by not fine-tuning what prompting can fix.

### Q8. Voice Agent Hit 3.5s Latency in Production ⭐⭐⭐
Target 1s. Production: 3.5s. Investigation: LLM was the bottleneck (2s TTFT). Root cause: hosted API had queuing latency during peak. Fix: (a) migrate to dedicated GPU for LLM, (b) enable speculative decoding, (c) reduce prompt (400 tokens → 200 tokens), (d) upgrade to faster model (GPT-4o-mini). Result: back to 900ms.

### Q9. Document Intelligence Confused Similar Contract Types ⭐⭐⭐
Product classified NDAs as employment contracts 15% of time. Cost: incorrect data extraction. Fix: fine-tune classifier on labeled examples of edge cases. Also: add confidence thresholds, route low-confidence to human review. Continuous learning from human corrections.

### Q10. RLHF Reward Model Learned Verbosity Bias ⭐⭐⭐⭐
After RLHF, model responses grew to 3× length. Labelers preferred longer responses regardless of quality. Fix: (a) retrain RM with length control (penalize obvious verbosity), (b) DPO with length normalization (SimPO-style), (c) explicit human labels for conciseness, (d) monitor length distribution during training.

### Q11. Fine-Tuned Model Leaked Training Data in Outputs ⭐⭐⭐⭐
Users got responses containing verbatim customer records. Security incident. Root cause: over-training on limited data → memorization. Fix: (a) IMMEDIATE: rollback, notify security team, assess exposure. (b) Systematic: differential privacy training, deduplicate training data, monitor training loss (over-low = memorization warning), evaluate for verbatim recall.

### Q12. Convincing Skeptical Legal Team on Voice Cloning ⭐⭐⭐⭐
Product wants voice cloning feature. Legal has concerns (consent, deepfakes). Approach: propose framework — explicit consent workflow, voice fingerprint for verification, watermarking of generated audio, usage audit trail, contractual protections. Legal signs off with these guardrails.

### Q13. Document Intelligence Pipeline Went Down During Quarter-End ⭐⭐⭐
100K contracts to process before quarter close. Pipeline broke. Investigation: OCR service rate-limited. Fix: (a) IMMEDIATE: scale up OCR provider, engage support, (b) prioritize highest-value contracts, (c) systematic: multi-provider OCR, circuit breakers, capacity planning for peak.

### Q14. Multimodal Cost Spiraled to $200K/Month ⭐⭐⭐
Product doubled video processing. Costs went up 10×. Investigation: naive frame sampling processed every frame. Fix: (a) adaptive frame sampling (scene-change based), (b) caching for repeat videos, (c) cheaper model tier for low-priority videos, (d) cost dashboards per feature. Reduced to $60K/month.

### Q15. Explaining Fine-Tuning Costs to Non-Technical Leadership ⭐⭐⭐
CEO: "Just fine-tune the model." Response: quantify. "Full fine-tune of 70B: $10K per run + data labeling ($20K) + eval ($5K) + serving ($10K/month new deployment) = $50K first year for one model. LoRA on same task: $200 per run + labeling + eval = $10K first year, and we can serve multiple LoRAs on shared base." Data-driven, not hand-waving.

### Q16. Preference Data Labelers Disagreed 40% of Time ⭐⭐⭐
Inter-annotator agreement dropped. Signal quality suspect. Investigation: task was too subjective, labelers using different criteria. Fix: (a) refine labeling instructions, (b) calibration sessions, (c) redundant labeling (3 labelers per pair), (d) drop pairs with disagreement, (e) simpler binary tasks vs graded quality. Agreement improved to 75%.

### Q17. Sunsetting a Deprecated Fine-Tuned Model ⭐⭐⭐
Old fine-tuned model still in production. Tenant depends on it. New base model launched. Migration path: (a) notify tenant 6 months ahead, (b) run new base + updated LoRA in shadow mode, (c) compare quality on their traffic, (d) gradual migration (10% → 50% → 100%), (e) sunset old model with data retention for audit.
