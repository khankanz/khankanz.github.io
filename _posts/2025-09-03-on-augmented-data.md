---
layout: post
title: "On Building Augmented Datasets: A Practical Case Study"
date: 2025-09-03
excerpt: "Why hospital-specific training data matters more than MIMIC."
tags: [data-augmentation, phi-deidentification, ner, synthetic-data, medical-nlp, data-quality]
---

I built a PHI de-identification pipeline that transforms 18,000 proprietary database entries into contextually-rich training data by augmenting real PHI entities with GPT-4 generated medical text. After failing to match real-data performance with purely synthetic data, I pivoted to this hybrid approach.

This isn't synthetic data generation—it's data augmentation. The distinction matters.

---

## Results First

The augmented approach delivered:
- F1 score improved from 40% to 75% over 8 iterations
- Model now detects multiple PHI entities per sample (was limited to 1 before)
- Eliminated systematic false positives on common medical terms

---

## Why Augmented Beats Synthetic: The Microcalcification Lesson

Before this project, I spent months generating synthetic radiology reports for microcalcification classification. Training on 600 real, manually-labeled reports achieved 95% accuracy. I created 6 clusters to identify different report formats, used these as few-shot examples for GPT-4/Gemini, and scaled to 1200 samples. 

Despite looking perfect individually, the synthetic-trained model couldn't break 65% on held-out real data—nowhere close to the 95% from real training data.

> This failure taught me: **you can't synthetic-data your way out of a domain shift problem**. Real medical text has idiosyncratic patterns, institutional quirks, and long-tail edge cases that models struggle to generate from scratch.

---

## The Pipeline

### Core Insight: Separate Entity from Context

Our proprietary advantage wasn't just having PHI—it was having *real PHI distributions*. Account numbers like `A123456` appear in specific contexts. Patient IDs follow institutional formats. But GPT-4 can't see this PHI (privacy constraints), and manual annotation doesn't scale (took 3 people a week for 2000 samples using Label Studio).

Solution: Templates with placeholders.

```python
def create_phi_template_prompt(placeholders: list[str], inspo: str) -> str:
    ph = ', '.join(f'<{p}>' for p in placeholders)
    return f"""# TASK
Write **one** free‑flowing hospital‑note paragraph (3–8 sentences).

## MUST‑USE PLACEHOLDERS
{ph}
(Use each **once**, keep the angle brackets.)

## INSPIRATION
"{inspo}"
[... style rules ...]
"""
```

GPT-4 generates text with `<patient_name>`, `<encounter_timestamp>` placeholders. I fill these locally with real PHI, maintaining security while leveraging LLM creativity.

### Fighting Repetition: Two Noise Sources

Initially, my 7B model produced variations of "Dr. X discussed the patient's treatment plan with nursing staff" ad nauseam. Even 72B models disappointed at scale. I needed diversity mechanisms:

**1. Random PHI combinations**: Sample 2-N entity types per generation
```python
def random_phi_combination(labels, min_n=5, max_n=10):
    return [random.choice(labels) for _ in range(random.randint(min_n, max_n))]
```

**2. Inspiration injection**: I filtered the mistral-pii dataset to English-only texts, creating 3200 stylistic seeds. The model must incorporate 2-4 exact phrases, mirror sentence rhythms, but reimagine scenarios clinically.

### Auto-annotation via Position Tracking

The clever bit: Since I control placeholder positions, I can automatically generate NER annotations:

```python
def fill_and_annotate(template, label_vars):
    pattern = re.compile(r'<(.*?)>')
    spans = []
    filled_out = []
    
    # Stream-build output & record spans using output positions
    for m in pattern.finditer(template):
        current_len = sum(len(x) for x in filled_out)
        val = replacements.get(key, m.group(0))
        filled_out.append(val)
        
        if key in replacements:
            spans.append({
                "start": current_len,
                "end": current_len + len(val),
                "text": val,
                "labels": [standardize_deid_label(key)]
            })
```

This eliminates manual annotation while preserving character positions for NER training.

---

## Design Decisions

**Why not fine-tune an open model locally?** I tried. 7B and 72B models produced repetitive outputs even with temperature tuning. The quality gap between GPT-4 and open models was worth the $30-40 API cost for 9000 samples.

**Why not regex everything?** That's essentially recreating Physionet's approach. Regex catches "Patient" as PHI, misses context-dependent entities, and fails on procedures named after people.

**Why GLiNER-small for iteration?** Training time: minutes not hours. Watching F1 climb from 40% to 75% over 8 iterations required rapid feedback cycles.

**When does this approach make sense?** You need proprietary data with privacy constraints. Public datasets alone won't give you an edge—unless you've discovered architectural insight to bake in as inductive bias, you need a data advantage.

---

## Future Work & Open Questions

- **Dataset diversity metrics**: How diverse is the augmented data really? Haven't measured unique n-grams, similarity between samples, or entropy of entity distributions.
- **Synthetic vs real n-gram analysis**: Need to compare phrase distributions between synthetic radiology reports and real ones to quantify domain shift.
- **Quality vs quantity tradeoff**: Would 1000 carefully curated augmented samples outperform 5000 repetitive synthetic ones?
- **Inspiration text scaling**: I picked 3200 templates for 9000 samples without formal analysis. Diminishing returns unclear.
- **Cross-institutional transfer**: My augmentation uses single-hospital PHI patterns. Generalization unknown.
- **Publishing resources**: Need to release filtered mistral-pii dataset on HuggingFace and sanitized code repository.

---

The graveyard of dead synthetic datasets taught me this: augmentation isn't about generating perfect medical text. It's about preserving what's real (entity distributions) while varying what's malleable (context). 

Sometimes the best synthetic data is barely synthetic at all.
