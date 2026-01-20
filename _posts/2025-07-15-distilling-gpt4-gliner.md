---
layout: post
title: "When 75% Isn't Enough: Trying to Distill GPT-4 into GLiNER"
date: 2025-07-15
excerpt: "Had a PHI detection model hitting 75% F1. Wondered if knowledge distillation could push it higher. Spoiler: it didn't work. But I learned a lot about why."
tags: [knowledge-distillation, gliner, gpt-4, ner, incomplete]
---

Had a PHI detection model hitting 75% F1. Decent, but not production-ready for clinical text. Wondered if knowledge distillation could push it higher—transfer dark knowledge from GPT-4 to GLiNER.

Spoiler: it didn't work. But I learned a lot about why.

---

## What I tried

### The setup

GPT-4 as teacher, GLiNER as student. Standard KD recipe:
- Generate soft labels from teacher
- Train student with combined loss (KL divergence on soft labels + BCE on hard labels)
- Hope the "dark knowledge" in soft distributions teaches the student nuances hard labels miss

### Round 1: Pseudo-labels from GPT-4

Prompted GPT-4 to extract entities with confidence scores. Structured JSON output:
```json
{"span": "Thomas Chen", "type": "NAME", "prob": 0.92}
```

Built a pipeline: prompt → parse spans/probs → save to JSONL → load as dataset with `ner_probs` field.

Problem: GPT-4's self-reported probabilities are fabricated. It's not giving you actual model confidence—it's generating a number that sounds plausible. Weak correlation with actual correctness.

### Round 2: Patching GLiNER's trainer

GLiNER uses BCE (binary cross-entropy) for multi-label span classification. Needed to:
- Patch DataCollator to include `ner_probs` and `gold_span_indices`
- Custom compute_loss with KL divergence on soft labels + BCE on hard labels
- Gather logits for gold spans only, handle shape mismatches

Spent days debugging:
- Device mismatches (CPU/GPU tensors)
- Shape errors ([6, 2448] vs [6, 204, 12, 6])
- CUDA assertions from gather indices out of bounds
- Clamping indices, valid masks, averaging over extra dimensions

Training finally ran. KLD dropped from 0.48 to <0.1. Looked promising.

### Round 3: The confidence problem

Inference scores stayed around 0.05. Model was "aligned" to teacher distributions but wouldn't commit.

Tried temperature scaling (T=10) to soften distributions during training. Tried Platt scaling post-training for calibration.

Platt gave high scores (0.7+) on completely wrong predictions—"Thomas" as Contact, random strings as Identifier. The calibration was fitting noise.

---

## What went wrong

1. **Prompted probabilities aren't real probabilities.** GPT-4 saying "0.92 confidence" is a hallucination, not a logit distribution. The soft labels were garbage in.

2. **Alignment ≠ confidence.** KLD going down means distributions match. Doesn't mean the student learned when to be confident.

3. **Small pseudo-dataset.** ~2000 samples. Probably needed 5-10x more, with targeted generation for underrepresented entity types.

4. **BCE vs KL mismatch.** GLiNER's multi-label sigmoid doesn't map cleanly to KL divergence, which assumes normalized distributions. Spent too long forcing a square peg.

5. **Calibration can't fix bad training.** Platt scaling on a model that learned wrong patterns just makes it confidently wrong.

---

## What I'd try differently

If I revisit:

- **Actual logits from open model.** Use Qwen or Llama locally where you can capture real softmax distributions, not prompted numbers.
- **More data with better distribution.** 10k+ samples, 30-40% targeted at names/contacts/ids where the model struggled.
- **Simpler loss.** Maybe just MSE on logits (the high-T approximation from Hinton). Skip the KL/BCE hybrid complexity.
- **Feature distillation.** Align hidden states, not just output probs. Might transfer more signal.

---

## What I actually learned

The Hinton paper is elegant but assumes you have the teacher's actual probability distributions. Blackbox KD from API models—where you're prompting for confidence scores—is a different beast entirely.

"Dark knowledge" is real. Soft labels do carry signal about uncertainty and ambiguity. But you can't extract that signal by asking the model to make up numbers.

GLiNER's architecture (encoder + span classification + sigmoid) doesn't fit standard KD recipes designed for softmax classifiers. The multi-label nature breaks assumptions.

---

*Related: [On Augmented Data: Synthetic PHI for De-identification](/2025/09/03/on-augmented-data.html)*
