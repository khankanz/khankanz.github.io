---
layout: post
title: "Using Decoder-Only LLMs for PHI De-Identification: A Minimal Setup"
date: 2025-06-17
categories: [nlp, llm, medical-nlp, phi]
tags: [LLM, PHI, Qwen, De-Identification, Clinical NLP, Jupyter, Transformers]
---

This was originally a notebook I built for a summer student. The goal: test if decoder-style LLMs can help extract PHI from clinical text — and wrap their output with simple postprocessing to recover spans and labels in a structured format.

All results below use **Qwen‑2.5‑3B‑Instruct** (≈3B parameters) on a single desktop GPU. Bigger models will likely perform better, but this quick test shows the core idea.

We’ve already fine-tuned several encoder models for this. We also built a baseline using regex and dictionary heuristics. But decoder models like Qwen, trained on broad corpora, offer a chance to extract PHI with very little supervision if we guide them properly.

## The Setup

LLMs struggle with exact character positions. Tokenizers don’t operate at character granularity. Asking for `"start"` and `"end"` positions directly leads to unreliable outputs. So instead, we ask for just this:

```
"exact span from input text" [label]
```

Then we match that span against the original input using `re.search()` to recover offsets ourselves. This avoids all alignment issues and keeps things clean.

## The Prompt

```
Extract all PHI (Protected Health Information) entities from the "Content" using the exact text spans as they appear. Follow the steps below:

1. Identify PHI entities in the input text.
2. Assign a label to each entity from this list: hospital, date, month, day, year, holiday, name, person, initials, MRN, medical record number, SIN, social_security_number, social security number, OHIP, identity card number, phone number, address, email, e-mail, email address, fax, location, street, postal code, health insurance id number, health insurance number, insurance number, landline phone number.
3. Return the exact span of the entity from the input, without rephrasing or normalizing.
4. Present the results as a list of strings in JSON format, where each string follows this structure:
   "text span" [label]

Examples:

Content: The patient, MR. JOHN DOE, visited St. VINCENT Hospital on December 12th, 2022. His OHIP: 1234-567-890.
Output: [
  "MR. JOHN DOE" [name],
  "St. VINCENT Hospital" [hospital],
  "December 12th, 2022" [date],
  "OHIP: 1234-567-890" [health insurance number]
]

Content: Contact Dr. LEE-WONG via email: lwong@caremail.org or fax: 555-432-8765. Address: 99 Queen's Blvd., Toronto, ON.
Output: [
  "Dr. LEE-WONG" [name],
  "lwong@caremail.org" [email address],
  "555-432-8765" [fax],
  "99 Queen's Blvd., Toronto, ON" [address]
]

Input: Content: {your clinical text here}
```

## Real Example Input

```
LEFT KNEE
Small joint effusion with lipohemarthrosis. Marginal osteophytosis along the lateral compartment. No fracture.
_____________
This report was electronically signed by DONALD, ALPHA, Staff Radiologist on 2010/13/25 at 22:30
```

### Model Output (verbatim)

```json
[
  "LEFT KNEE" [location],
  "DONALD, ALPHA" [name],
  "2010/13/25" [date],
  "22:30" [time]
]
```

### Postprocessed Output

```json
[{"start":0,"end":9,"text":"LEFT KNEE","labels":["Location"]},
 {"start":794,"end":809,"text":"DONALD, ALPHA","labels":["Name"]},
 {"start":832,"end":842,"text":"2010/13/25","labels":["Date"]},
 {"start":846,"end":851,"text":"22:30","labels":["time"]}]
```

A few notes:
- The model invented a `"time"` label. We never gave it one.
- It tagged `"LEFT KNEE"` as a `location`. Technically true, but not PHI.
- `"DONALD, ALPHA"` and `"2010/13/25"` were both captured exactly and matched correctly.

## Another Example

**Input:**
```
Dr. CHARMS,LUCKY reviewed the patient's file.
```

**Output:**
```json
[
  "Dr. CHARMS,LUCKY" [name]
]
```

**Postprocessed:**
```json
[{"start":4,"end":20,"text":"CHARMS,LUCKY","labels":["Name"]}]
```

## Takeaways

Use decoder LLMs for what they’re good at: labeling spans. Don’t ask them to think in characters. Instead, extract raw spans and reconstruct structure with simple Python logic. This gives you:
- Exact offsets
- Clean, valid JSON
- Full control over downstream formatting

Even a 3B model can produce usable PHI annotations quickly. The approach still needs deeper evaluation, better prompt tuning, and strict label control, but the core idea works and is light to implement.

If you’re working on clinical de‑identification and want to dig deeper, reach out.

