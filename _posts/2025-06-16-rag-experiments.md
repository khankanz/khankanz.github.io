---
layout: post
title: "RAG Experiments: Chunking, Retrieval, Reformulation"
date: 2025-06-16
excerpt: "Chunking, hybrid retrieval, and reformulation strategies for clinical RAG pipelines, including Dense X Retrieval and proposition-based methods."
---

This is a summary of my ongoing experiments applying RAG techniques to clinical NLP problems. It covers chunking strategies, retrieval architecture, and how query reformulation improves alignment with structured text.

---

## RAG Recap

Classic Retrieval-Augmented Generation (RAG) works like this:

1. Embed each document in your corpus using a model `M` → store vectors.
2. Embed a user query `U` using the same model.
3. Use cosine similarity to retrieve top-k matches.
4. Feed top-k documents + query into an LLM for generation.

This works well in open-domain QA but breaks down in clinical NLP.

---

## Retrieval Challenges in Clinical Text

Clinical documents use structured, formal language. Queries are messy, conversational, and vague. Embedding models trained on generic corpora struggle to bridge this gap.

I tested general, biomedical, and radiology-specific embedding models. None reliably separated reports with and without microcalcifications — unless explicitly tuned for that task.

---

## Chunking: Trial, Error, Proposition

Naive chunking strategies I tried:

- Newline-based
- Paragraph-based
- Sentence-by-sentence

All failed. Small chunks hallucinate. Arbitrary splits weaken signal. This aligns with Skylar Payne's notes and Anton from ChromaDB.

Inspired by **Dense × Retrieval**, I used GPT-4 to decompose clinical notes into **propositions**:

- One idea per chunk
- De-referenced pronouns
- Syntactically simple
- Self-contained
- Output as JSON list

This created interpretable, retrieval-friendly text units.

---

## Hybrid Retrieval

Dense similarity search alone was slow and imprecise. I switched to a **hybrid method**:

1. **BM25**: Get top 100 matches fast using keyword overlap.
2. **Embed**: Use `Qwen3-Embedding-0.6B` for query + chunk embeddings.
3. **Rerank**: Cosine similarity → top-k for generation.

This reduced semantic drift between natural language queries and structured clinical notes.

---

## Query Reformulation

Even with better chunks, vague user queries underperform. To fix this, I used GPT-4 to rewrite queries into proposition-aligned formats:

### Input:
    Does the patient have anything they're looking forward to?

### Reformulated:
```json
[
  "Is the patient looking forward to any specific events?",
  "Is the patient anticipating any particular activities?",
  "Does the patient have any future plans they are excited about?"
]
```

This improved reranking and final answer quality.

---

## Model Outcomes

### Query: *Is the patient looking forward to any specific events?*

| Model        | Output                                                                                                         | Result |
|--------------|----------------------------------------------------------------------------------------------------------------|--------|
| Qwen2.5-7B   | “Revisiting discharge planning on June 3, 2025.”                                                              | ❌ Too literal |
| Qwen2.5-14B  | “Jordan M. is looking forward to the intake appointment…”                                                    | ✅ Grounded |
| Qwen2.5-32B  | “Jordan M. is looking forward to structured therapy…”                                                        | ✅ Strong |
| GPT-4        | “Jordan M. appears to be looking forward to continuing therapy… calendar reminder…”                          | ✅ Best synthesis |

Removing “with just a few words” from the prompt improved generation. Expanded queries + reranked proposition chunks made a large difference.

---

## Next Steps

- Train a FlanT5 propositionizer
- Experiment with other chunking units (e.g., entity-level)
- Add filtering by patient ID or date
- Test end-to-end QA across more clinical tasks

---

More to come.
