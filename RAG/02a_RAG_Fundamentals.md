# Module 02a — RAG Fundamentals
> LLM Fundamentals Series · Why RAG exists, what it is, and the naive pipeline

---

## Table of Contents
1. [Why RAG Exists](#1-why-rag-exists)
2. [The Two Memories](#2-the-two-memories)
3. [What is RAG](#3-what-is-rag)
4. [Naive RAG — Full Pipeline](#4-naive-rag--full-pipeline)
5. [Offline vs Online Phase](#5-offline-vs-online-phase)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. Why RAG Exists

LLMs have three fundamental problems when used alone:

| Problem | What it means | Example |
|---|---|---|
| **Knowledge cutoff** | Model is frozen in time — knows nothing after training | Ask GPT-4 about yesterday's news → refuses or makes something up |
| **Hallucination** | Generates plausible-sounding but false information | Confidently cites a paper that doesn't exist |
| **No private knowledge** | Never saw your company's internal docs | Doesn't know your HR policy, codebase, or product catalog |

### Why Not Just Fine-tune the Model?

This is a common interview question. The answer:

| | Fine-tuning | RAG |
|---|---|---|
| Knowledge update | Retrain (hours/days) | Re-index (minutes) |
| Cost | High GPU compute | Cheap |
| Hallucination | Still happens — model memorizes patterns, not facts | Lower — grounded in source text |
| Citability | Cannot cite where it learned from | Can cite exact source chunk |
| Private data | Data baked into weights permanently | Data stays in your database |

> **The RAG insight:** Instead of baking knowledge into weights, fetch relevant knowledge at query time and hand it to the model as context.
>
> **Separation of concerns: model handles reasoning, database handles knowledge.**

---

## 2. The Two Memories

RAG combines two types of memory:

```
┌─────────────────────────┐        ┌─────────────────────────┐
│   PARAMETRIC MEMORY     │        │  NON-PARAMETRIC MEMORY  │
│   (LLM Weights)         │   +    │  (Vector Store)         │
│─────────────────────────│        │─────────────────────────│
│ • Trained on internet   │        │ • Updated anytime       │
│ • Fixed after training  │        │ • Your private docs     │
│ • General knowledge     │        │ • Source of truth       │
│ • Can hallucinate       │        │ • Citable               │
└─────────────────────────┘        └─────────────────────────┘
             │                                  │
             └──────────────┬───────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │      RAG      │
                    │───────────────│
                    │ Grounded      │
                    │ Current       │
                    │ Private       │
                    │ Citable       │
                    └───────────────┘
```

**Parametric memory** = knowledge baked into model weights during training. Cannot be changed without retraining.

**Non-parametric memory** = external knowledge store (vector DB). Can be updated, swapped, filtered, and cited.

---

## 3. What is RAG

**RAG = Retrieval-Augmented Generation**

An architecture that:
1. **Retrieves** relevant documents from a knowledge store
2. **Augments** the LLM prompt with those documents
3. **Generates** a grounded response based only on retrieved context

Introduced in a 2020 Meta AI paper by Lewis et al. as a way to give sequence-to-sequence models access to a non-parametric memory store.

### The Core Loop (One Sentence)

> Embed the query → find similar chunks in the vector store → stuff them into the prompt → LLM answers from those chunks only.

---

## 4. Naive RAG — Full Pipeline

Naive RAG is the baseline. Two phases: **offline indexing** and **online serving**.

### Offline Indexing Pipeline

```
Raw Documents
(PDF / DOCX / HTML / MD)
        │
        ▼
  ┌─────────────┐
  │   LOADER    │  → Parse and extract plain text
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │   CHUNKER   │  → Split text into small overlapping pieces
  └─────────────┘
        │
        ▼
  ┌─────────────────┐
  │ EMBEDDING MODEL │  → Convert each chunk to a dense vector
  └─────────────────┘       [0.12, -0.87, 0.44, 0.03, ...]
        │
        ▼
  ┌─────────────┐
  │  VECTOR DB  │  → Store vectors + metadata (source, page, section)
  └─────────────┘
```

### Online Query Pipeline

```
User Query
     │
     ▼
  ┌─────────────────┐
  │ EMBED THE QUERY │  → Query text → vector
  └─────────────────┘
     │
     ▼
  ┌───────────────────┐
  │   VECTOR SEARCH   │  → Find top-K chunks most similar to query
  └───────────────────┘
     │
     ▼
  ┌──────────────────────────────────┐
  │         PROMPT ASSEMBLER         │
  │  system prompt                   │
  │  + retrieved chunks (with source)│
  │  + conversation history          │
  │  + user query                    │
  └──────────────────────────────────┘
     │
     ▼
  ┌─────┐
  │ LLM │  → Generate grounded answer
  └─────┘
     │
     ▼
Response + Citations
```

---

## 5. Offline vs Online Phase

| | Offline Phase | Online Phase |
|---|---|---|
| **When it runs** | Once (on doc update/upload) | Every user query |
| **What it does** | Index and embed all documents | Retrieve and generate |
| **Latency** | Can be slow (batch job) | Must be fast (< 3s) |
| **Triggered by** | New document added / updated | User sends a message |
| **Output** | Vector DB index | Answer + citations |

### Freshness
New documents should be searchable within a defined SLA (e.g. 1 hour of upload). This requires a trigger mechanism — webhook, file watcher, or scheduled job — that kicks off the offline pipeline on doc change.

---

## 6. Key Takeaways

- RAG exists to solve three LLM problems: knowledge cutoff, hallucination, and private knowledge gaps.
- It combines two memories: parametric (model weights) and non-parametric (vector store).
- Fine-tuning teaches the model *behavior*. RAG gives the model *knowledge*. They are complementary.
- Naive RAG has two phases: offline indexing (build the store) and online serving (retrieve + generate).
- The model is instructed to answer **only** from retrieved context — this is what prevents hallucination.

### Interview One-Liner
> *"RAG retrieves relevant chunks from a knowledge store at query time and injects them into the LLM prompt. The model is grounded in retrieved context rather than relying on memorized weights — this makes answers citable, current, and less likely to hallucinate."*

---
*LLM Fundamentals · Module 02a · RAG Fundamentals*
