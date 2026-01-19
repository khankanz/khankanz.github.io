---
layout: post
title: "Framing Survival Prediction as Next-Token: A Failed Experiment"
date: 2025-01-19
excerpt: "Read the Cell2Sentence paper and thought: what if survival prediction could be framed as next-token prediction? I never finished it, but the idea was too interesting not to document."
tags: [cell2sentence, kaggle, llm, tabular, incomplete]
---

I never finished this, but the idea was too interesting not to document.

---

## The Spark

Read the [Cell2Sentenceence paper](https://github.com/vandijklab/cell2sentence) and something clicked. They take single-cell gene expression data—thousands of genes per cell—and convert it into natural language sentences that LLMs can process.

The key insight: **ordering matters**. They sort genes by expression level (highest to lowest) and serialize them into a sentence. The mathematical formula was something like ranking by z-score or log-fold-change. This ordering preserves the relative importance of each gene while making it digestible by a language model trained on text.

I was staring at the CIMBTR Kaggle competition at the time—bone marrow transplant survival prediction. Tabular data. Clinical features. The usual stuff you'd throw XGBoost at.

And I thought: what if survival prediction could be framed as next-token prediction?

---

## The Half-Baked Idea

Language models are probability distributions over sequences. They predict the next token given the previous tokens. What if clinical features could be serialized into a sequence where "survival" or "death" was the natural next token to predict?

The Cell2Sentence ordering trick seemed relevant. If you sort features by some measure of importance—maybe absolute value of correlation with outcome, or feature importance from a tree model—you get a canonical ordering. Then:

`age:67 | comorbidity:high | donor_match:partial | conditioning:myeloablative | ...`

Becomes a sequence where the model learns patterns like "when you see these tokens in this order, the next token is more likely to be [outcome_X]."

The LLM isn't learning tabular patterns—it's learning sequence patterns. And maybe, just maybe, the attention mechanism would pick up on feature interactions that tree models miss.

---

## What I Actually Did

1. Downloaded the CIMBTR data
2. Wrote some code to serialize features into sentences
3. Tried a few ordering schemes (by feature importance, by correlation, alphabetically as baseline)
4. Fine-tuned a small model on the serialized data
5. Got distracted by work and never finished the evaluation

---

## Why I Stopped

Honestly? I didn't have a clear hypothesis for *why* this would beat XGBoost. The Cell2Sentence paper worked because gene expression data has inherent sequential structure—pathways, regulatory cascades, biological ordering. Clinical tabular data doesn't have that. The ordering I was imposing was artificial.

Also, the CIMBTR competition ended and I had actual work to do.

---

## What Might Be Worth Revisiting

1. **The ordering hypothesis**: Is there a principled way to order clinical features that captures something meaningful? Maybe temporal order (things that happen first come first in the sequence)?

2. **The attention visualization**: Even if the model doesn't beat XGBoost, looking at attention patterns might reveal feature interactions that are clinically meaningful.

3. **The pretraining question**: Cell2Sentence works partly because the LLM has prior knowledge about genes from biomedical text. For clinical features, what prior knowledge would help? Maybe pretraining on clinical notes where these features are discussed?

---

## The Honest Conclusion

This was a "what if" exploration that I abandoned. The Cell2Sentence insight about ordering was genuinely interesting. Applying it to survival prediction was a reach. I learned something about how these papers translate (or don't) to other domains.

Sometimes the negative result—or in this case, the incomplete result—is worth documenting. At minimum, if someone else has this idea, they'll find this post and either learn from my false starts or tell me what I missed.

---

*If you've tried something similar or have thoughts on principled feature ordering for tabular-to-sequence conversion, I'd love to hear about it.*
