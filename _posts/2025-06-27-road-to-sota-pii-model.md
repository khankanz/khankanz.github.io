---
layout: post
title: "Road to a SOTA PII Model"
date: 2025-06-27
excerpt: "From PhysioNet heuristics tagging 'pain' as a name to 75% F1."
categories: [nlp, ner, medical-ai]
---

Yes, I’m using PHI and PII interchangeably. Yes, I know they're not the same. No, I don't care. It's my blog.

The journey started with a test run of a tool from the Gemini group—a Python port of some dusty Perl heuristics from PhysioNet. I liked its ability to swap out PII with fake “surrogate” data. What I didn’t like was it tagging the word **pain** as a person’s name. Context matters. Heuristics alone weren’t going to cut it.

I pulled data from our FHIR store: patient demographics, radiology reports, pathology notes. I queried everything I could—account numbers (probably?), patient and clinician names, DOBs, OHIP/JHN/MRN/CMR identifiers, timestamps, addresses, even terms like “tattoo” (until I found out that was also a medical procedure—thanks for that, oncology).

First attempt: train GLiNER on this data. It's billed as SOTA for NER, built from Pile spans annotated by GPT, then filtered for PII. I tried finetuning it on 2,000 samples. Results: garbage. Turns out feeding a model endless account numbers with no context doesn’t teach much. Who knew.

So I pivoted. Could I inject context into training by generating full-text sentences around each PII type? I wrote prompts, called the largest LLM I could cram into my 8GB setup (don’t laugh), and started generating. The catch: data can’t leave hospital servers. So now I’m juggling slow inferences, a crowded SLURM cluster, and my dusty desktop GPU.

Validation? Initially fine—my napkin math said we’re good. But when scaled up, the model began repeating itself. Still, I pushed forward. A thousand examples were manually annotated by me, a summer student, and a research assistant. Later discovered span inconsistencies while debugging training. Fantastic. Relabeling time.

Then came a win: someone whispered **ollama** to me. With some coaxing, I got **Qwen3-30B** running locally. Inference takes 20 seconds per call, but the outputs? Gold. I stopped asking LLMs for start/end indices—just get me the text, I’ll regex the rest. Wrote a wrapper to convert spans into NER-ready JSON.

After 7 test batches and dozens of manual checks, Qwen3 delivered near-perfect F1 every time. This means I can automate annotation at scale. Manual review stays in the loop, but I no longer need an army of undergrads to label data.

Next problem: the repetition in augmented data...

*To be continued.*
