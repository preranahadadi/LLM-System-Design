# Module 02d — Retrieval & Reranking
> LLM Fundamentals Series · Vector search vs BM25, hybrid search, RRF, metadata filtering, two-stage reranking

---

## Table of Contents
1. [The Goal of Retrieval](#1-the-goal-of-retrieval)
2. [Dense Retrieval — Vector Search](#2-dense-retrieval--vector-search)
3. [Sparse Retrieval — BM25](#3-sparse-retrieval--bm25)
4. [Why Neither Alone is Enough](#4-why-neither-alone-is-enough)
5. [Hybrid Search](#5-hybrid-search)
6. [RRF — Reciprocal Rank Fusion](#6-rrf--reciprocal-rank-fusion)
7. [Metadata Filtering](#7-metadata-filtering)
8. [Two-Stage Retrieval — Reranking](#8-two-stage-retrieval--reranking)
9. [Reranker Models](#9-reranker-models)
10. [Full Retrieval Pipeline](#10-full-retrieval-pipeline)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. The Goal of Retrieval

Given a user query, find the most relevant chunks from the knowledge base to pass to the LLM.

**What "relevant" means:**
- The chunk contains information needed to answer the question
- The chunk is specific enough to be useful (not too generic)
- The chunk is accurate and up-to-date

**The retrieval problem:**
- Knowledge base can have millions of chunks
- Must find the right ones in under ~200ms
- Must handle synonyms, paraphrases, domain jargon, and exact keywords

---

## 2. Dense Retrieval — Vector Search

Convert both the query and all document chunks into embeddings. Find chunks whose vectors are closest to the query vector.

```
Query: "How many vacation days do employees get?"
    │
    ▼
embed(query) = [0.12, -0.87, 0.44, ...]
    │
    ▼
ANN search in vector DB
    │
    ▼
Returns chunks with vectors closest by cosine similarity:
  1. "Employees are entitled to 15 days of paid leave per year." (sim=0.93)
  2. "Annual leave accrues at 1.25 days per month." (sim=0.91)
  3. "PTO policy: full-time staff receive 15 days." (sim=0.89)
```

### Strengths
-  Captures semantic meaning — finds "annual leave" when you search "vacation"
- Handles paraphrases, synonyms, different phrasings
- Language-agnostic (cross-lingual models)
- No need for exact keyword overlap

### Weaknesses
- Misses exact keyword matches for rare terms
- Embedding model quality directly limits retrieval quality
- Struggles with proper nouns, product names, codes, IDs

---

## 3. Sparse Retrieval — BM25

Classic information retrieval algorithm. Ranks documents based on term frequency and inverse document frequency.

```
BM25 score(doc, query) = Σ IDF(term) × (TF(term, doc) × (k1 + 1))
                                        ─────────────────────────────
                                        TF(term, doc) + k1 × (1 - b + b × |doc|/avgdl)

Where:
  TF  = term frequency in doc
  IDF = log(N / docs_containing_term)  → rare terms score higher
  k1  = 1.2–2.0 (term frequency saturation)
  b   = 0.75 (document length normalization)
```

You don't need to memorize the formula — understand the intuition:
- Terms that appear **frequently** in a chunk but **rarely** across the corpus get high scores
- Longer documents are penalized (normalized by length)
- Exact keyword match required

### Strengths
-  Exact keyword matching — finds "FDA approval" or "SOC-2 compliance" precisely
- Great for proper nouns, product codes, IDs, rare technical terms
- Fast — inverted index lookup
- No GPU needed
- Fully interpretable

### Weaknesses
-  No semantic understanding — "vacation" and "annual leave" are unrelated to BM25
-  Vocabulary mismatch — user must use the same words as the document

---

## 4. Why Neither Alone is Enough

| Query type | Vector Search | BM25 |
|---|---|---|
| "What is the vacation policy?" |  Finds "annual leave" semantically |  Misses if doc uses "PTO" |
| "SOC-2 Type II certification process" |  May return vaguely related security docs |  Finds exact "SOC-2" mentions |
| "How does BERT differ from GPT?" |  Semantic understanding |  Only if doc literally contains both words |
| "error code E-4021 troubleshooting" |  May not embed error codes well | Exact match on "E-4021" |

No single method dominates. Hybrid is consistently better.

---

## 5. Hybrid Search

Run both vector search and BM25 in parallel, then fuse the ranked result lists.

```
User Query
    │
    ├──────────────────────────────────────────────────────────┐
    │                                                          │
    ▼                                                          ▼
[Embed query]                                          [Tokenize query]
    │                                                          │
    ▼                                                          ▼
[Vector Search]                                        [BM25 Search]
Top-50 dense results                                   Top-50 sparse results
  1. doc_A (sim=0.93)                                    1. doc_B (score=18.2)
  2. doc_C (sim=0.91)                                    2. doc_A (score=15.7)
  3. doc_E (sim=0.88)                                    3. doc_D (score=12.1)
    │                                                          │
    └──────────────────────┬───────────────────────────────────┘
                           │
                           ▼
                   [RRF FUSION]
                   Merged, re-ranked top-K
```

**Consistent finding in research:** Hybrid search outperforms either method alone, especially on domain-specific corpora with specialized terminology.

### Where to Run Hybrid Search

| Tool | How |
|---|---|
| **Weaviate** | Built-in BM25 + vector, hybrid out of the box |
| **Qdrant** | Sparse + dense vectors, hybrid search API |
| **Elasticsearch + pgvector** | ES for BM25, pgvector for dense, custom fusion |
| **Pinecone** | Sparse-dense hybrid index |

---

## 6. RRF — Reciprocal Rank Fusion

The standard algorithm for merging multiple ranked lists.

### Formula

```
RRF_score(document) = Σ  1 / (k + rank_i(document))
                      i

Where:
  k    = 60  (constant, prevents high scores from top-ranked docs dominating)
  rank_i = rank of the document in result list i
  Σ    = sum over all result lists (dense, sparse, etc.)
```

### Example

```
Document A:
  Dense rank  = 1  →  1 / (60 + 1) = 0.0164
  Sparse rank = 2  →  1 / (60 + 2) = 0.0161
  RRF score   = 0.0164 + 0.0161 = 0.0325  ← appears in both lists, scores high

Document B:
  Dense rank  = 50  →  1 / (60 + 50) = 0.0091
  Sparse rank = 1   →  1 / (60 + 1)  = 0.0164
  RRF score   = 0.0091 + 0.0164 = 0.0255

Document C:
  Dense rank  = 2   →  1 / (60 + 2)  = 0.0161
  Sparse rank = N/A (not in sparse results)
  RRF score   = 0.0161
```

Documents that appear in **both** lists (like A) get boosted. Documents only in one list get their rank contribution only.

### Why k=60?
It's an empirical constant from the original RRF paper. It smooths the scores so a rank-1 document doesn't completely dominate. Works well in practice — rarely needs tuning.

---

## 7. Metadata Filtering

Before or during retrieval, filter the search space using stored metadata.

### Pre-filtering (Filter → Search)
Apply filter first, then run ANN search only over matching documents.

```python
results = vector_db.query(
    query_embedding=embed("vacation policy"),
    filter={
        "department": {"$eq": "HR"},
        "updated_at": {"$gt": "2025-01-01"},
        "access_level": {"$in": ["all_employees", "managers"]}
    },
    top_k=20
)
```

- Faster — smaller search space
- If filter is too strict, may return 0 results

### Post-filtering (Search → Filter)
Run ANN search first, then discard results that don't match the filter.

- Always returns results (if any matching docs exist)
- Slower — searches full index, discards after

### When to Use Filtering
- **Access control** — user in Sales dept should only search Sales docs
- **Freshness** — exclude documents older than 1 year
- **Version** — only search docs tagged as "current version"
- **Language** — only return documents in user's language

---

## 8. Two-Stage Retrieval — Reranking

Vector search is fast but approximate. The bi-encoder never lets query and document see each other — it independently encodes both and compares vectors. This limits accuracy.

A **reranker** (cross-encoder) takes both the query and each candidate chunk as a combined input. It scores relevance much more accurately.

### The Pipeline

```
STAGE 1 — Bi-encoder (fast, approximate)
10M documents
    │
    ▼ ANN search (~10ms)
Top 50 candidates
    │
    ▼
STAGE 2 — Cross-encoder reranker (slow, accurate)
    │   Takes (query, chunk_1) → scores relevance: 0.92
    │   Takes (query, chunk_2) → scores relevance: 0.45
    │   Takes (query, chunk_3) → scores relevance: 0.88
    │   ... (repeat for all 50 candidates, ~100ms total)
    ▼
Top 5 highest-scoring chunks
    │
    ▼
Sent to LLM prompt
```

### Why Can't We Just Use Cross-encoder Everywhere?

```
Cross-encoder on 10M docs:
  100ms per pair × 10,000,000 docs = 277 hours per query  

Cross-encoder on 50 candidates:
  100ms total 
```

The bi-encoder is a fast filter. The cross-encoder is the precise scorer. You need both.

### What Cross-encoder Sees

```
Input:  [CLS] query: "vacation policy" [SEP] 
        Employees are entitled to 15 days of paid leave per year... [SEP]

Output: relevance score = 0.92
```

The model sees both together in self-attention — every word in the query can attend to every word in the chunk and vice versa. This interaction is what makes it accurate.

---

## 9. Reranker Models

### Open Source

| Model | Notes |
|---|---|
| `BGE-Reranker-v2-m3` | Best OSS, multilingual, strong performance |
| `BGE-Reranker-large` | Strong English, slightly faster than m3 |
| `cross-encoder/ms-marco-MiniLM-L-12-v2` | Fast, good quality, widely used |
| `cross-encoder/nli-deberta-v3-large` | Also used for groundedness checking |

### Managed API

| Model | Notes |
|---|---|
| `Cohere Rerank API` | Easy plug-in, excellent quality, managed |
| `Jina Reranker API` | Good quality, multilingual |

### When to Skip Reranking
If vector similarity score for all top candidates is already very high (> 0.9), the reranker adds latency without much gain. Add a threshold:

```python
if max(similarity_scores) > 0.92:
    skip_reranker = True   # top result is clearly relevant
else:
    run_reranker = True    # disambiguation needed
```

---

## 10. Full Retrieval Pipeline

Putting it all together:

```
User Query
    │
    ▼
[1] Embed query (bi-encoder)
    │
    ▼
[2] Hybrid Search in parallel:
      ├── Vector Search (dense)  → Top-50
      └── BM25 Search (sparse)   → Top-50
    │
    ▼
[3] RRF Fusion → Merged Top-50
    │
    ▼
[4] Metadata Filter → remove restricted / stale docs
    │
    ▼
[5] Cross-encoder Reranker → Top-5 chunks
    │
    ▼
[6] Context Assembly → passed to LLM
```

### Latency Budget (approximate)

| Step | Time |
|---|---|
| Embed query | ~20ms |
| Vector search | ~10ms |
| BM25 search | ~5ms |
| RRF fusion | ~1ms |
| Reranking (50 candidates) | ~80–150ms |
| **Total retrieval** | **~150–200ms** |

---

## 11. Key Takeaways

- **Dense (vector) search** handles semantic similarity. **Sparse (BM25)** handles exact keyword matching. Use both.
- **Hybrid search with RRF fusion** consistently outperforms either method alone.
- **Metadata filtering** enables access control, freshness, and domain-specific scoping.
- **Two-stage retrieval** is the standard: bi-encoder for fast candidate recall (top-50), cross-encoder reranker for precise scoring (top-5).
- The bi-encoder sees query and doc separately. The cross-encoder sees both together — that's why it's more accurate.
- Total retrieval budget: ~150–200ms. The LLM generation step is the main latency.

### Interview One-Liner
> *"I run hybrid retrieval — dense vector search for semantic similarity and BM25 for exact keyword matching — in parallel, then fuse results with Reciprocal Rank Fusion. The top-50 candidates go through a cross-encoder reranker to get the final top-5 chunks. This two-stage approach balances speed and accuracy: bi-encoder for recall, cross-encoder for precision."*

---

