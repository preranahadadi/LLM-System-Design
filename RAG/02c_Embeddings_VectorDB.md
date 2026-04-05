# Module 02c — Embeddings & Vector Databases
> LLM Fundamentals Series · Embedding models, similarity metrics, HNSW, vector DB comparison

---

## Table of Contents
1. [What are Embeddings](#1-what-are-embeddings)
2. [How Embedding Models Work](#2-how-embedding-models-work)
3. [Bi-encoder vs Cross-encoder](#3-bi-encoder-vs-cross-encoder)
4. [Key Embedding Models](#4-key-embedding-models)
5. [Similarity Metrics](#5-similarity-metrics)
6. [What is a Vector Database](#6-what-is-a-vector-database)
7. [Indexing Algorithm — HNSW](#7-indexing-algorithm--hnsw)
8. [Vector Database Comparison](#8-vector-database-comparison)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What are Embeddings

An embedding is a dense numerical vector that represents the **semantic meaning** of a piece of text.

```
"vacation policy"   →  [0.12, -0.87,  0.44, 0.03, -0.21, ...]  (768 or 1536 numbers)
"annual leave"      →  [0.11, -0.85,  0.47, 0.01, -0.19, ...]
"expense report"    →  [0.73,  0.42, -0.31, 0.88,  0.55, ...]
```

The key property: **semantically similar texts produce similar vectors.**

```
cosine_similarity("vacation policy", "annual leave") = 0.94  ← very close
cosine_similarity("vacation policy", "expense report") = 0.21 ← far apart
```

This is what allows retrieval to work. A user asking "how much PTO do I get?" will produce a vector close to a chunk about "annual leave entitlement" — even though they share no keywords.

---

## 2. How Embedding Models Work

An embedding model is a neural network (transformer encoder) that maps text to a fixed-size vector.

```
Input text
    │
    ▼
[Tokenizer]         → split text into tokens
    │
    ▼
[Token Embeddings]  → each token → initial vector
    │
    ▼
[Transformer Encoder Layers]
  → self-attention: each token attends to every other token
  → feed-forward layers
  → layer normalization
    │
    ▼
[Pooling]           → aggregate all token vectors into one sentence vector
  → CLS token pooling (use first token's output)
  → Mean pooling (average all token vectors)
    │
    ▼
Output vector [d₁, d₂, ..., d₇₆₈]   ← the embedding
```

The model is trained on pairs of similar/dissimilar texts using **contrastive learning** — similar pairs are pushed close together, dissimilar pairs are pushed apart.

---

## 3. Bi-encoder vs Cross-encoder

This distinction is critical in RAG — they are used at different stages.

### Bi-encoder (Embedding Model)

Encodes query and document **independently** into vectors. Similarity is computed with a simple dot product or cosine.

```
Query  →  [Encoder]  →  q_vector
                                  → cosine_similarity(q_vector, d_vector)
Doc    →  [Encoder]  →  d_vector
```

- ✅ Very fast — vectors can be precomputed and indexed offline
- ✅ Scales to millions of documents (ANN search)
- ❌ Less accurate — query and doc never see each other during encoding
- **Used for:** Stage 1 retrieval

### Cross-encoder (Reranker)

Takes query and document **together** as a single input. Sees both simultaneously.

```
[Query + Document]  →  [Encoder]  →  relevance score (0 to 1)
```

- ✅ Much more accurate — models the interaction between query and doc
- ❌ Slow — cannot precompute, must run at query time for each candidate
- ❌ Cannot scale to millions of docs (100x slower than bi-encoder)
- **Used for:** Stage 2 reranking (on top-50 candidates only)

### Why Two Stages?

```
10M docs
   │
   ▼ Bi-encoder ANN search (~10ms)
Top 50 candidates
   │
   ▼ Cross-encoder reranker (~100ms on 50 docs)
Top 5 chunks
   │
   ▼ LLM
Answer
```

You use the fast-but-approximate bi-encoder to filter from 10M to 50, then the slow-but-accurate cross-encoder to find the best 5 from those 50. You can't run the cross-encoder on 10M docs — it would take hours.

---

## 4. Key Embedding Models

### Closed Source (API)

| Model | Provider | Dims | Notes |
|---|---|---|---|
| `text-embedding-3-large` | OpenAI | 3072 | Best quality, data leaves infra |
| `text-embedding-3-small` | OpenAI | 1536 | Cheaper, still very good |
| `embed-v3` | Cohere | 1024 | Excellent multilingual support |

### Open Source (Self-hosted)

| Model | Dims | Notes |
|---|---|---|
| `BGE-large-en-v1.5` (BAAI) | 1024 | Best OSS general-purpose English |
| `BGE-m3` (BAAI) | 1024 | Multilingual, multi-granularity |
| `E5-large-v2` (Microsoft) | 1024 | Strong MTEB benchmark |
| `all-mpnet-base-v2` (SBERT) | 768 | Fast, lightweight, good quality |
| `GTE-large` (Alibaba) | 1024 | Strong retrieval performance |

### How to Choose

| Situation | Choice |
|---|---|
| Fastest time to market, data not sensitive | `text-embedding-3-large` (OpenAI) |
| Private data, self-hosted | `BGE-large-en-v1.5` |
| Multilingual corpus | `BGE-m3` or Cohere `embed-v3` |
| Resource-constrained / edge | `all-mpnet-base-v2` |
| Domain-specific (legal, medical) | Fine-tune `BGE-large` on domain pairs |

### Domain Fine-tuning
When off-the-shelf embeddings miss domain-specific terminology (legal latin, medical jargon, internal acronyms), fine-tune the embedding model on labeled query-passage pairs from your domain.

```
Training pair example:
  Query:    "What is our EBITDA calculation methodology?"
  Positive: "EBITDA is calculated as operating income plus..."
  Negative: "The quarterly sales report shows..."
```

---

## 5. Similarity Metrics

How do you measure how close two vectors are?

### Cosine Similarity
Measures the **angle** between two vectors. Ignores magnitude — only direction matters.

```
cosine(A, B) = (A · B) / (|A| × |B|)

Range: -1 (opposite) to 1 (identical)
Typical threshold for "similar": > 0.8
```

- ✅ Most common for text embeddings
- ✅ Not affected by vector length (magnitude)
- **Use when:** Standard text similarity

### Dot Product
Measures both angle and magnitude.

```
dot(A, B) = Σ (aᵢ × bᵢ)
```

- ✅ Faster than cosine (no normalization step)
- ✅ If vectors are L2-normalized, dot product = cosine similarity
- **Use when:** Vectors are normalized (which most embedding models output)

### Euclidean Distance (L2)
Raw geometric distance in vector space.

```
L2(A, B) = √ Σ (aᵢ - bᵢ)²
```

- Less common for text
- Sensitive to vector magnitude
- **Use when:** Image embeddings, some specialized models

> **Default for RAG:** Cosine similarity. Most vector DBs use this by default for text.

---

## 6. What is a Vector Database

A vector database stores embedding vectors and supports **Approximate Nearest Neighbor (ANN)** search — finding the vectors most similar to a query vector, fast, at scale.

```
STORE:
  chunk_id: "hr_policy_chunk_12"
  vector:   [0.12, -0.87, 0.44, ...]
  metadata: {source: "HR_Policy.pdf", page: 4, department: "HR"}

QUERY:
  query_vector = embed("vacation policy")
  results = db.search(query_vector, top_k=10)
  → returns top 10 most similar chunks by cosine similarity
```

### Why "Approximate"?
Exact nearest neighbor search requires comparing the query to every vector — O(N) per query. At 10M vectors with 1024 dimensions, this is too slow.

ANN algorithms trade a small amount of recall for massive speed gains. In practice, well-tuned ANN has 95%+ recall at 10–100x the speed of exact search.

---

## 7. Indexing Algorithm — HNSW

**Hierarchical Navigable Small World** — the standard algorithm for ANN in production.

### How it Works

HNSW builds a multi-layer graph:
- **Top layers** — sparse, long-range connections. Used for coarse navigation.
- **Bottom layers** — dense, short-range connections. Used for fine-grained search.

```
Layer 2 (sparse):   A ──────────── E ──────── I
                    │                          │
Layer 1 (medium):   A ──── C ──── E ──── G ── I
                    │      │      │      │     │
Layer 0 (dense):    A─B─C─D─E─F─G─H─I─J─K─L

Query: start at top layer, navigate toward query region,
       descend to bottom layer for precise neighbors
```

### Key Parameters

| Parameter | Effect |
|---|---|
| `ef_construction` | Higher = better recall at index time, slower build, more memory |
| `ef_search` | Higher = better recall at query time, slower search |
| `M` | Number of connections per node — higher = better recall, more memory |

**Default safe values:** `ef_construction=200`, `M=16`, `ef_search=100`

---

## 8. Vector Database Comparison

| DB | Hosting | Hybrid Search | Filtering | Best for |
|---|---|---|---|---|
| **Pinecone** | Fully managed | Yes (sparse+dense) | Yes | Fast start, zero ops overhead |
| **Weaviate** | OSS / Managed | Built-in BM25 | Yes | Hybrid without extra infra |
| **Qdrant** | OSS / Managed | Yes | Yes (rich payload filters) | Best OSS performance |
| **pgvector** | OSS (Postgres ext) | Manual (+ PG FTS) | Yes (SQL WHERE) | Already running Postgres |
| **FAISS** | Library (self-host) | No | No (manual) | Max speed, no managed layer |
| **Chroma** | OSS / Managed | Basic | Yes | Local dev and prototyping |
| **Milvus** | OSS / Managed | Yes | Yes | High-scale OSS option |

### How to Choose

| Situation | Choice |
|---|---|
| Already using Postgres | `pgvector` — one less system to operate |
| Need hybrid search out of the box | `Weaviate` or `Qdrant` |
| Managed, no infra work | `Pinecone` |
| Max performance, self-host | `FAISS` + custom metadata layer |
| Prototyping / local dev | `Chroma` |

### pgvector Example (Postgres)
```sql
-- Store a chunk
INSERT INTO documents (content, embedding, source, page)
VALUES ('Employees get 15 days vacation...', '[0.12, -0.87, ...]'::vector, 'HR_Policy.pdf', 4);

-- Query top-5 similar chunks
SELECT content, source, page
FROM documents
WHERE department = 'HR'                          -- metadata filter
ORDER BY embedding <=> '[0.11, -0.85, ...]'      -- cosine distance operator
LIMIT 5;
```

---

## 9. Key Takeaways

- Embeddings are dense vectors where semantic similarity = geometric closeness.
- **Bi-encoder** (fast, approximate) is used for retrieval at scale. **Cross-encoder** (slow, accurate) is used for reranking the top candidates.
- **Cosine similarity** is the default metric for text. **BGE-large** is the default open-source embedding model.
- **HNSW** is the standard ANN indexing algorithm — trades tiny recall loss for massive speed gain.
- Choose vector DB based on your constraints: `pgvector` if already on Postgres, `Weaviate`/`Qdrant` for hybrid, `Pinecone` for managed simplicity.
- Domain fine-tuning of embedding models significantly helps when your corpus uses specialized terminology.

### Interview One-Liner
> *"Embedding models convert text to dense vectors where semantic similarity maps to geometric closeness. I use a bi-encoder for fast ANN retrieval across all documents, then a cross-encoder reranker on the top-50 candidates for accuracy. The vector store uses HNSW indexing for approximate nearest neighbor search at scale."*

---
*LLM Fundamentals · Module 02c · Embeddings & Vector Databases*
