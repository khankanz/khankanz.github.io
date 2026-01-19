---
layout: post
title: "MedColBERT: Late Interaction Retrieval for Clinical Text"
date: 2024-12-15
excerpt: "Adapting JaColBERT's training recipe to medical retrieval. Built datasets from BioASQ and OMOP concept mappings. Got stuck at indexing. Incomplete."
tags: [colbert, retrieval, medical-nlp, bioasq, omop, incomplete]
---

Adapting JaColBERT's training recipe to medical retrieval. Incomplete.

---

Read Benjamin Clavié's [JaColBERTv2.5 papers](https://github.com/bclavie/JaColBERT) and got obsessed. He achieved SOTA Japanese retrieval with 110M parameters, trained for 15 hours on 4 A100s, using only 40% of the data from the previous version. The gains came entirely from fixing inefficiencies in how ColBERT models are typically trained.

## The training recipe insights that mattered

- Knowledge distillation from cross-encoders is immensely powerful—more than data volume
- Dynamic query length beats fixed padding (ColBERT pads queries with [MASK] tokens)
- In-batch negatives are useless for multi-vector models and waste memory
- Score normalization (min-max on both teacher and student) stabilizes training
- KL-Div loss strictly outperforms MarginMSE for ColBERT
- Schedule-free learning works better than linear decay and lets you stop/restart easily
- Single strong teacher beats ensembled teachers in low-resource settings

ColBERT's core idea: instead of compressing a document into one vector, keep token-level embeddings. At query time, compute MaxSim—for each query token, find its maximum similarity to any document token, then sum. This preserves more information than single-vector approaches and generalizes better out-of-domain, though historically it underperformed on in-domain tasks. The JaColBERTv2.5 recipe fixed that.

---

## The datasets I built

### BioASQ

Medical question-answering dataset from PubMed. Each question has associated snippets containing the answer. I extracted (query, positive_snippets) pairs, built a BM25 index over all snippets, and mined hard negatives by retrieving top-100 similar snippets and filtering out the actual positives. The result: triplets where negatives share medical terminology but don't answer the question.

### OMOP Concept Mappings

Our clinical data warehouse has source_to_concept_map tables that map local terms to SNOMED concepts. I pulled the top 500 most frequent concepts from measurement, observation, and procedure tables, then extracted their source-to-target mappings. Query is the source value ("BREAST: Biomarker Reporting Template"), positive is the mapped concept ("Immunohistochemistry procedure"). Same BM25 approach for hard negatives—similar SNOMED concepts that aren't the correct mapping.

---

## The pipeline I implemented

- TREC format conversion: `qid Q0 docid rank score tag`
- Separate TSV files for queries and document collections
- Four query augmentation strategies to test the dynamic length ablation:
  - `none`: no [MASK] padding
  - `fixed8`: always append 8 [MASK] tokens
  - `baseline`: pad to max_length with [MASK]
  - `dynamic`: minimum 8 masks, pad to nearest multiple of 32
- ColBERTv2 loaded from HuggingFace with custom tokenization

---

## Where I stopped

Got stuck at the indexing step. ColBERT needs to encode all documents and build a searchable index before you can run queries. I had the data formatted, the model loaded, the augmentation strategies implemented—but never actually ran indexing or evaluation. No NDCG@10 or MRR@10 numbers. No ablation comparison.

A second notebook went even further down the rabbit hole: what if instead of fine-tuning ColBERTv2, I trained a medical-domain BERT from scratch? Explored ModernBERT's architecture (8192 context, RoPE, GeGLU), the "Don't Stop Pretraining" paper on domain-adaptive vs task-adaptive pretraining, and whether continued pretraining with 10% medical reports mixed into the original distribution would work. Also incomplete.

---

## What I'd do if I picked this back up

1. Finish the indexing step and run baseline evaluation
2. Compare augmentation strategies on both datasets
3. Try the two-stage training: pretrain on MS MARCO filtered to medical domain, post-train on BioASQ + OMOP
4. Benchmark against MedCPT on BEIR medical tasks

---

*If you've worked on medical retrieval or have thoughts on adapting ColBERT training recipes to new domains, I'd be interested to hear what worked.*
