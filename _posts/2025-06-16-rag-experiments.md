---
layout: post
title: "RAG Experiments: Chunking, Retrieval, Reformulation"
date: 2025-06-16
excerpt: "Chunking, hybrid retrieval, and reformulation strategies for clinical RAG pipelines, including Dense X Retrieval and proposition-based methods."
tags: [rag, clinical-nlp, retrieval, chunking, query-reformulation, hybrid-retrieval]
---

This post summarizes my experiments building a Retrieval-Augmented Generation (RAG) system over psychiatric discharge documentation. The goal: enable clinicians to ask meaningful, nuanced questions about patients' post-discharge outlook and care.

---

## Task

Given a set of de-identified patient charts (e.g., progress notes, treatment summaries), we want to support natural language queries such as:

- *What sentiment or tone should I approach the patient with?*
- *Does the patient have any future plans to look forward to?*
- *Has the patient acknowledged the importance of follow-up care?*

These questions require drawing subtle inferences from free-text notes, making structured QA difficult to scale manually.

---

## Background: Embedding Breakdown

In earlier work, I experimented with general, biomedical, and radiology-specific embedding models to classify microcalcification status from mammography reports — *Positive*, *Negative*, or *Not Specified*. Despite domain tuning, none of the models captured the specific, sparse signal required for this task.

> That failure was formative. It showed me that off-the-shelf embedding models often fail to preserve fine-grained clinical meaning — especially when used naïvely in dense retrieval setups.

This led me to reevaluate how retrieval pipelines are designed: how documents are chunked, how user queries are interpreted, and how semantic drift is corrected.

---

## RAG Recap (with Notation)

We want to answer user queries over a corpus `C` of clinical documents.

**Baseline setup:**

1. Each document `D ∈ C` → `M` → `D~embedded~`
2. User query `U` → `M` → `U~embedded~`
3. Compute similarity between `U~embedded~` and all `D~embedded~` (typically cosine)
4. Take top-k matches → feed into an LLM alongside the original query

> This architecture assumes `D~embedded~` captures all relevant content in a fixed-length vector. It doesn’t. Compressing complex clinical notes into dense vectors makes it hard to recover granular information — especially when queries are vague or compositional.

---

## Chunking: From Naive to Proposition

My early chunking attempts included:

- Line-by-line splits  
- Paragraphs  
- Sentence-level tokens

None worked well. Small chunks led to hallucinations. Paragraphs introduced noise. Sentence splits broke context mid-thought.

This problem was highlighted in:
- Skylar Payne’s *RAG Anti-patterns*
- ChromaDB’s chunking discussion with Anton

I pivoted to proposition-based chunking from the paper *Dense × Retrieval*. Each document was decomposed into standalone, simplified statements:

> Each proposition is a single idea. Pronouns are replaced. Structure is flattened. Every chunk stands on its own and aligns more naturally with user questions.

I used GPT-4 to generate these propositions, formatted as JSON. Example prompt:

```json
[
  "Jordan M. has been provided with a calendar reminder for the intake appointment.",
  "Jordan M. expressed motivation to continue structured therapy.",
  "The outpatient plan is acknowledged by Jordan M. as a good next step."
]
```

---

## Hybrid Retrieval: Fixing Similarity Drift

Dense-only retrieval struggled due to mismatched language between user queries and document style.

To fix this, I adopted **hybrid retrieval**:

1. Use **BM25** to retrieve top-100 candidates based on keyword overlap  
2. Embed user query + candidate chunks  
3. Rerank using cosine similarity → return top-k

> BM25 is fast and brittle. Dense models are semantic but noisy. Combining them gave the best of both — lexical recall followed by semantic precision.

This reduced irrelevant matches and ensured we only embedded promising candidates.

---

## Query Reformulation

User queries are often vague. Even with proposition chunks, retrieval underperformed.

So I used GPT-4 to rewrite user queries to match the structure of my chunks:

### Input:
> Does the patient have anything they're looking forward to?

### Reformulated:
```json
[
  "Is the patient looking forward to any specific events?",
  "Is the patient anticipating any particular activities?",
  "Does the patient have any future plans they are excited about?"
]
```

> Reformulated queries were simpler, explicit, and structurally aligned with the chunk format. This made reranking sharper and generation stronger.

---

## Generation Outcomes

Query: *Is the patient looking forward to any specific events?*

---

**🔹 Qwen2.5-7B**

> “Revisiting discharge planning on June 3, 2025.”

❌ Extracts a date but misses intent. Literal match, low relevance.

---

**🔹 Qwen2.5-14B**

> “Jordan M. is looking forward to the intake appointment and is motivated to continue with the structured outpatient therapy plan.”

✅ Clear answer grounded in chunk content. Matches meaning, not just keywords.

---

**🔹 Qwen2.5-32B**

> “Jordan M. is looking forward to the outpatient plan and is motivated to continue with structured therapy, indicating anticipation for these future activities.”

✅ Strong result with richer phrasing and implied intent. Aligns well with query.

---

**🔹 GPT-4**

> “Jordan M. appears to be looking forward to continuing structured therapy and building a routine, as indicated by their motivation and acknowledgment that the outpatient plan is 'a good next step.' They have also been provided with a calendar reminder and contact details for an intake appointment, suggesting anticipation of future care.”

✅ Best synthesis. Integrates multiple signals from chunk content. Demonstrates grounding and inference.

---

> Prompting for “a short, informative sentence or two” (instead of “just a few words”) made a significant difference. Reformulated queries + reranked proposition chunks enabled smaller models to approach GPT-4-level answers.

---

## Next Steps

- Train a FlanT5 propositionizer using GPT-generated examples
- Add filters for patient ID and note timestamp at query time
- Explore chunking beyond propositions: spans, entities, templates
- Evaluate generalizability across clinical QA tasks

---

If you're working on clinical RAG systems and these thoughts hit home, reach out. Let's discuss how to work together.
