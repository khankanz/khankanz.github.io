---
layout: post
title: "Do LLMs Think in English?"
date: 2026-01-20
excerpt: "Models use multilingual concepts as rhetorical devices, not reasoning primitives."
tags: [llm, multilingual, reasoning, chain-of-thought, exploratory]
---

I speak three languages. When I think, I don't translate.

*museebat mein gadhe ko bhi baap bana dete hain*—my family uses this to mean people will shamelessly elevate anyone, even a donkey, to the status of father. The common translation is softer: in hardship, you work with what you have. Six words, two completely different meanings depending on who raised you.

English can't do that.

So why does chain-of-thought reasoning always happen in English?

---

## The question

Test-time scaling works by giving models more tokens to "think." o1, R1—they all use long reasoning chains. But those chains are almost always in English, even when the input is in another language.

If some languages encode concepts more efficiently, shouldn't multilingual reasoning be better?

Current LLMs show massive performance gaps across languages. 60% accuracy in English vs. 5% in Swahili on identical logic problems. Training artifact? Tokenization bottleneck? Architectural constraint?

I read ~11 papers to find out.

---

## What the literature says

**Latent reasoning is viable.** Models don't need to emit every token to reason. HRPO: 1.5B models match 7B by reasoning in hidden states. CoLaR: 53-83% shorter chains with only 4.8% accuracy drop. System-1.5: 20x speedup, 92% token reduction.

Reasoning doesn't need English tokens. It can happen in latent space.

**The English constraint is real.** mCoT found 60% English vs. 5% Swahili on the same problems. But training on translated reasoning data equalizes performance. The gap isn't architectural—it's training data.

LinguaLIFT: mixing English + target language helps reasoning transfer. Code-switching during chain-of-thought improves non-English performance.

**Tokenization is a root cause.** BPE fragments semantically dense non-English concepts. The English paraphrase is more tokens but more *predictable* tokens. Models learn to avoid non-English paths—not because English is semantically better, but because it's tokenizer-cheaper.

Meta's BLT eliminates the tokenizer entirely and matches token-level performance.

**Even o1-style models fall back to English.** January 2025 paper: reasoning models internally switch to English even when prompted in other languages. Hidden chain-of-thought is English regardless of input.

**The unexplored gap:** Multilingual training + latent reasoning + byte-level architecture. Each piece exists. Nobody's combined them.

---

## What I tried

Prompted GPT-4 and Grok:

```
Solve this step by step. You may use words or phrases from
any language if they express the idea more precisely.

Problem: A friend keeps making promises they don't keep.
How should I think about whether to trust them again?
```

They code-switch. GPT-4 used German (*Zuverlässigkeit*), Urdu (*Baatein kam, amal zyada*), Japanese (*現実を見る*), Spanish (*No es maldad; es límite*).

The models know these concepts. They use them appropriately.

---

## What I found

The code-switching is rhetorical, not computational.

Models use non-English phrases as punctuation—to land points with cultural resonance. But the reasoning scaffold remains entirely English. Strip out the multilingual phrases and the logical structure is unchanged.

They're citing wisdom, not thinking in it.

GPT-4 knows *Schadenfreude* exists. It doesn't reason through the concept. It reasons in English and decorates the output with a German word for flavor.

The knowledge is there. The cognition isn't.

---

## Stress-testing my assumptions

I developed "Karpathy Razors" to challenge whether my intuition points at something real:

**Representation:** Do LLMs have genuine multilingual conceptual representations, or just translation layers? If the latter, "thinking in Urdu" is meaningless—there's no Urdu to think in.

**Biological Analogy:** Am I projecting human multilingual cognition onto a different architecture? My brain developed cross-lingual shortcuts through lived experience. Transformers might not have the same pressure or solution.

**Less Is More:** Should I add capability (give models idioms) or constraint (force derivation)? The value of Urdu idioms isn't that they exist—it's that speakers were forced to compress complex ideas over centuries. Giving idioms to a model is giving trivia. Making a model derive compressions under constraint is the real test.

**Memory vs. Generalization:** Prompting with multilingual concepts just tests whether models can use what I hand them. That's not a deep capability test.

The problem: access-based experiments (prompting) are easy but might be trivial. Constraint-based experiments (training under pressure) are the real test—but require compute I don't have.

---

## Why this might matter

Tokenization creates systematic bias. If BPE fragments non-English concepts and makes English paraphrases cheaper, models learn to reason in English even when it's semantically less efficient.

Multilingual users get worse performance not because their problems are harder, but because the model's internal reasoning is English-centric.

---

## What I learned

**The finding:** Models use multilingual concepts as rhetorical devices, not reasoning primitives. The question sharpens from "would multilingual reasoning help?" to "is English-only reasoning a training artifact (fixable) or an architectural constraint (fundamental)?"

**The method:** Stress-testing intuitions before investing in experiments saves time. The Karpathy Razors framework is reusable.
