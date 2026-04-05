# Module 02h — Evaluation
> LLM Fundamentals Series · RAGAS, MRR, Recall@K, hallucination rate, building eval datasets

---

## Table of Contents
1. [Why RAG Evaluation is Hard](#1-why-rag-evaluation-is-hard)
2. [What to Evaluate — Three Layers](#2-what-to-evaluate--three-layers)
3. [RAGAS — The Standard Framework](#3-ragas--the-standard-framework)
4. [Retrieval Metrics](#4-retrieval-metrics)
5. [Generation Metrics](#5-generation-metrics)
6. [System-Level Metrics](#6-system-level-metrics)
7. [LLM-as-Judge](#7-llm-as-judge)
8. [Building an Eval Dataset](#8-building-an-eval-dataset)
9. [Continuous Evaluation Pipeline](#9-continuous-evaluation-pipeline)
10. [Common Failure Modes and Fixes](#10-common-failure-modes-and-fixes)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. Why RAG Evaluation is Hard

With a traditional ML model you have a label and a prediction — evaluation is straightforward. RAG has multiple components that can each fail independently, and the "answer" is open-ended text.

### The RAG Evaluation Problem

```
A RAG system can fail at multiple independent points:

1. Wrong chunks retrieved  → even a perfect LLM gives a wrong answer
2. Right chunks retrieved  → but LLM ignores them and halluccinates
3. Right answer generated  → but it doesn't address the actual question
4. Correct answer          → but doesn't cite the source

You need metrics for each failure mode separately.
```

### Why BLEU/ROUGE Don't Work

BLEU and ROUGE measure n-gram overlap between generated and reference text. For RAG:
- "Employees get 15 days of vacation" vs "Staff are entitled to 15 annual leave days" → different n-grams, same meaning → BLEU says wrong, it's actually correct.
- Open-ended answers have many valid phrasings.
- BLEU rewards short safe answers, penalizes elaboration.

> **RAG needs semantic metrics, not surface-form overlap metrics.**

---

## 2. What to Evaluate — Three Layers

```
┌─────────────────────────────────────────────────────────┐
│  RETRIEVAL LAYER                                        │
│  Did we get the right chunks?                           │
│  Metrics: Recall@K, Precision@K, MRR, NDCG             │
├─────────────────────────────────────────────────────────┤
│  GENERATION LAYER                                       │
│  Is the answer good and grounded?                       │
│  Metrics: Faithfulness, Answer Relevance                │
├─────────────────────────────────────────────────────────┤
│  SYSTEM LAYER                                           │
│  Is the overall system healthy?                         │
│  Metrics: Latency, Hallucination Rate, "I don't know"   │
│           rate, User satisfaction                        │
└─────────────────────────────────────────────────────────┘
```

---

## 3. RAGAS — The Standard Framework

RAGAS (Retrieval Augmented Generation Assessment) is the de-facto evaluation framework for RAG systems. It measures 4 core metrics using LLM-as-judge.

```
Input per test case:
  - question:        "What is the vacation policy?"
  - answer:          generated answer from your RAG system
  - contexts:        retrieved chunks used to generate the answer
  - ground_truth:    expected correct answer (from your eval dataset)

Output:
  - context_recall:     0.0 to 1.0
  - context_precision:  0.0 to 1.0
  - faithfulness:       0.0 to 1.0
  - answer_relevance:   0.0 to 1.0
```

### Quick Setup

```python
from ragas import evaluate
from ragas.metrics import (
    context_recall,
    context_precision,
    faithfulness,
    answer_relevance,
)
from datasets import Dataset

data = {
    "question":     ["What is the vacation policy?"],
    "answer":       ["Employees get 15 days of paid vacation per year."],
    "contexts":     [["HR_Policy.pdf: Employees are entitled to 15 days..."]],
    "ground_truth": ["Employees receive 15 days of annual paid leave."],
}

dataset = Dataset.from_dict(data)
results = evaluate(dataset, metrics=[
    context_recall, context_precision, faithfulness, answer_relevance
])
print(results)
```

---

## 4. Retrieval Metrics

### Context Recall
**Did retrieval find all the information needed to answer the question?**

Measures: What fraction of the ground truth answer's claims are supported by the retrieved chunks?

```
Ground truth: "Employees get 15 days vacation. Unused days carry over up to 5."

Retrieved chunks contain:
   "Employees entitled to 15 days vacation" → covered
   "carry over up to 5 days" → NOT in retrieved chunks

Context Recall = 1/2 = 0.50
```

Low recall → retrieval is missing relevant documents. Fix: improve chunking, embedding model, or add more chunks (increase top-K).

---

### Context Precision
**Were the retrieved chunks actually relevant?**

Measures: Of all the retrieved chunks, what fraction were actually useful for answering?

```
Retrieved 5 chunks:
  Chunk 1: About vacation policy -relevant
  Chunk 2: About sick leave - not relevant
  Chunk 3: About vacation accrual - relevant
  Chunk 4: About expense reports - not relevant
  Chunk 5: About carry-over rules - relevant

Context Precision = 3/5 = 0.60
```

Low precision → retrieval is pulling noisy, irrelevant chunks. Fix: improve embedding model, add reranker, tighten metadata filtering.

---

### MRR — Mean Reciprocal Rank
**How high is the first relevant document in the result list?**

```
MRR = (1/N) × Σ (1 / rank of first relevant doc)

Query 1: relevant doc at rank 1 → 1/1 = 1.0
Query 2: relevant doc at rank 2 → 1/2 = 0.5
Query 3: relevant doc at rank 4 → 1/4 = 0.25

MRR = (1.0 + 0.5 + 0.25) / 3 = 0.58
```

Target: MRR > 0.8 for production.

---

### Recall@K
**Is the relevant document somewhere in the top-K retrieved results?**

```
Recall@5: Is the relevant chunk in the top 5?  → 1 if yes, 0 if no
Recall@10: Is it in the top 10?

Average over all queries in eval set.
```

Target: Recall@5 > 0.85.

Difference from MRR: Recall@K doesn't care about position within top-K (it's binary: in or out). MRR rewards finding it earlier.

---

### NDCG — Normalized Discounted Cumulative Gain
**How well are the relevant chunks ordered within the results?**

Takes into account both whether relevant docs are retrieved AND their position. More nuanced than Recall@K.

```
Highly relevant at rank 1 → best
Highly relevant at rank 5 → still good, small penalty
Irrelevant at rank 1 → bad
```

Target: NDCG@5 > 0.75.

---

## 5. Generation Metrics

### Faithfulness
**Is every claim in the generated answer supported by the retrieved context?**

The most important hallucination metric.

```
Retrieved context: "Employees get 15 days vacation per year."

Generated answer:  "Employees get 15 days vacation per year.  ← supported 
                   This is in line with the industry average  ← NOT in context 
                   of 14 days for similar-sized companies."

Faithfulness = 1/2 = 0.50   (1 supported claim out of 2 total)
```

How RAGAS measures it: LLM extracts all claims from the answer, then checks each claim against the context. Claims not in context = hallucinations.

Target: Faithfulness > 0.90.

---

### Answer Relevance
**Does the answer actually address the question asked?**

A faithful answer can still miss the point. Answer relevance checks that the answer is on-topic.

```
Question: "Can I carry over unused vacation days?"
Answer:   "Employees receive 15 days of vacation per year, accruing at
           1.25 days per month."

This answer is faithful (supported by context) but doesn't answer the
carry-over question → Answer Relevance = low
```

How RAGAS measures it: LLM generates hypothetical questions that the answer would be a good response to, then measures similarity to the original question.

Target: Answer Relevance > 0.85.

---

## 6. System-Level Metrics

### Hallucination Rate
Percentage of answers containing at least one unsupported claim.

```
Hallucination Rate = (queries with faithfulness < 0.9) / (total queries)
```

Target: < 2%. Hard limit — track as a primary safety metric.

---

### "I Don't Know" Rate
Percentage of queries where the system returns a deflection instead of an answer.

```
"I Don't Know" Rate = (deflected queries) / (total queries)
```

| Rate | Interpretation |
|---|---|
| < 5% | Possibly overconfident — system may be hallucinating instead of deflecting |
| 5–20% | Healthy range for most knowledge bases |
| > 30% | Knowledge base may be too sparse or retrieval threshold too high |

---

### Latency

| Percentile | Target |
|---|---|
| P50 | < 1.5s |
| P95 | < 3s |
| P99 | < 5s |

Break down by component: embedding, retrieval, reranking, generation. The bottleneck is almost always LLM generation.

---

### User Satisfaction Signals

| Signal | How to collect |
|---|---|
| **Thumbs up/down** | Inline UI rating |
| **Follow-up question** | Did user need to ask again? (bad sign) |
| **Copy/paste rate** | Did user use the answer? (good sign) |
| **Escalation rate** | % of queries escalated to human agent |
| **Session abandonment** | User left after receiving answer |

---

## 7. LLM-as-Judge

For most RAGAS metrics, the evaluator is itself an LLM (typically GPT-4). This works well but has limitations.

### How It Works

```python
# RAGAS faithfulness check (simplified)
judge_prompt = """
Context: {retrieved_chunks}
Answer: {generated_answer}

Extract all factual claims from the Answer.
For each claim, determine if it is SUPPORTED or NOT SUPPORTED by the Context.
Return a list of (claim, supported: bool).
"""

result = judge_llm.generate(judge_prompt)
faithfulness_score = sum(r.supported for r in result) / len(result)
```

### Limitations of LLM-as-Judge

| Limitation | Impact |
|---|---|
| **Judge model biases** | Prefers its own style of writing |
| **Cost** | Adds an LLM call per evaluated query |
| **Inconsistency** | Same input can get different scores on retries |
| **Can't catch factual errors it doesn't know** | Judge may not know if a specific fact is wrong |

### Mitigations

- Use temperature=0 for the judge model
- Average over 3 judge runs for important evals
- Supplement with human review for high-stakes decisions
- Use multiple judge prompts (ensemble)

---

## 8. Building an Eval Dataset

You need a golden dataset: `(question, expected_answer, relevant_doc_ids)` triples.

### Sources

**Option 1: Manual curation (gold standard)**
Domain experts write questions and expected answers. Slow, expensive, highest quality.

**Option 2: LLM-generated + human reviewed**
```python
# Generate test questions from your documents
generation_prompt = """
Read the following document passage and generate 3 realistic questions
that a user might ask, where the answer is clearly in this passage.
Also write the expected answer for each question.

Passage: {chunk_text}

Output as JSON: [{question, answer, relevant_passage}]
"""

test_cases = []
for chunk in knowledge_base:
    generated = llm.generate(generation_prompt.format(chunk_text=chunk.text))
    test_cases.extend(json.loads(generated))
```

Then human reviewers filter out bad questions.

**Option 3: Mine from production logs**
After launch, collect queries where users gave thumbs up → those are your positive examples. Queries with thumbs down or escalation → negative examples.

### Dataset Size

| Use case | Minimum size |
|---|---|
| Initial launch / sanity check | 50–100 questions |
| Pre-production evaluation | 200–500 questions |
| Continuous monitoring | 500–1000+ questions |
| Per-domain coverage | ~50 questions per major topic area |

### Ensuring Coverage

Your eval set must cover:
- All major topic areas in your knowledge base
- Different question types (factual, procedural, comparative, hypothetical)
- Short simple questions and long multi-part questions
- Questions the system should answer and questions it should deflect ("I don't know")

---

## 9. Continuous Evaluation Pipeline

Don't just evaluate once at launch. Run evals continuously.

```
Code / config change
     │
     ▼
[Offline Eval] runs RAGAS on golden dataset
     │
     ├── Metrics below threshold?  →  Block deployment, flag for review
     │
     └── Metrics pass?
          │
          ▼
     [A/B Test] new vs old version on 5–10% of traffic
          │
          ├── New version worse?   →  Roll back
          │
          └── New version better?  →  Full rollout
               │
               ▼
     [Production Monitoring] real-time metrics
          │
          ├── Faithfulness drops?  →  Alert, investigate knowledge base
          ├── Latency spikes?      →  Alert, scale or optimize
          └── "I don't know" rate up?  →  Knowledge gap detected, add docs
```

### Regression Testing

Maintain a regression test set: queries that previously failed and were fixed. Every deployment must pass all regression tests.

```python
regression_cases = [
    # Cases that previously hallucinated
    {"question": "What is our D&O insurance limit?", "expected_contains": "$10M"},
    # Cases that previously returned "I don't know" incorrectly
    {"question": "How do I reset my password?", "expected_not_deflect": True},
]
```

---

## 10. Common Failure Modes and Fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Low Context Recall | Wrong chunks retrieved, top-K too small | Improve embedding, increase K, add HyDE |
| Low Context Precision | Too many irrelevant chunks retrieved | Add reranker, tighten metadata filters |
| Low Faithfulness | LLM hallucinating despite context | Stronger grounding instructions, lower temperature, add groundedness check |
| Low Answer Relevance | Answer is off-topic | Improve system prompt, add "address the question directly" instruction |
| High "I don't know" rate | Knowledge gaps, threshold too strict | Add missing docs, lower retrieval threshold |
| Low "I don't know" rate | System hallucinating instead of deflecting | Stricter grounding, lower confidence threshold for deflection |
| High latency | LLM generation bottleneck | Add semantic cache, use smaller model for simple queries |

---

## 11. Key Takeaways

- RAG has three evaluation layers: retrieval (right chunks?), generation (good answer?), system (healthy overall?).
- **RAGAS** is the standard framework: Context Recall, Context Precision, Faithfulness, Answer Relevance.
- **Faithfulness** is the most critical metric — low faithfulness means hallucination.
- **Recall@5 > 0.85** and **Faithfulness > 0.90** are the production targets to aim for.
- LLM-as-judge powers most modern RAG eval — powerful but has biases and costs.
- Build a golden eval dataset before launch; keep it growing with regression cases.
- Run evaluation continuously as part of your deployment pipeline, not just once.

### Interview One-Liner
> *"I evaluate RAG with RAGAS across four dimensions: Context Recall (did retrieval find the right docs?), Context Precision (were retrieved docs relevant?), Faithfulness (is the answer grounded in context?), and Answer Relevance (does it address the question?). I maintain a golden eval dataset of 500+ query-answer pairs, run RAGAS on every deployment, and track hallucination rate as a hard production metric with a < 2% target."*

---
