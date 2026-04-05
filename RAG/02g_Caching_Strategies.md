# Module 02g — Caching Strategies
> LLM Fundamentals Series · Semantic cache, exact cache, hybrid cache, TTL, invalidation

---

## Table of Contents
1. [Why Cache in RAG](#1-why-cache-in-rag)
2. [Exact Cache](#2-exact-cache)
3. [Semantic Cache](#3-semantic-cache)
4. [Threshold Tuning](#4-threshold-tuning)
5. [What Gets Stored](#5-what-gets-stored)
6. [Hybrid Cache — Combining Both](#6-hybrid-cache--combining-both)
7. [TTL — Cache Expiry](#7-ttl--cache-expiry)
8. [Cache Invalidation](#8-cache-invalidation)
9. [Where to Store the Cache](#9-where-to-store-the-cache)
10. [Cache in the Full Pipeline](#10-cache-in-the-full-pipeline)
11. [Multi-tenant Caching](#11-multi-tenant-caching)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Why Cache in RAG

The full RAG pipeline for one query:

```
Embed query       ~20ms
Vector search     ~10ms
BM25 search       ~5ms
Reranking         ~100ms
LLM generation    ~500–2000ms   ← biggest cost
─────────────────────────────
Total             ~650–2200ms   per query
```

For repeated or semantically similar questions, re-running this entire pipeline wastes time and money.

**The problem:**
```
User 1: "What is our parental leave policy?"         → full pipeline, $0.05
User 2: "How many weeks maternity leave do we get?"  → same answer, $0.05
User 3: "Tell me about parental leave benefits"       → same answer, $0.05

At 10,000 similar queries/day → $500/day on identical answers
```

**The solution:** Cache at the query level. Similar questions get cached answers instantly.

---

## 2. Exact Cache

The simplest form. Cache answers keyed by the exact query string (hash).

```
cache = {
    hash("What is our parental leave policy?"): "Employees receive 16 weeks...",
    hash("How do I submit an expense report?"): "Submit via the Finance portal...",
}

On query:
  key = hash(query)
  if key in cache:
      return cache[key]   ← instant, ~1ms
  else:
      run full pipeline
      cache[key] = answer
```

### Strengths
- Near-zero latency (~1ms hash lookup)
- Zero compute — no embedding needed
- Perfect precision — identical queries always get the same answer

### Weaknesses
-  Only catches **verbatim** repeated queries
- "What is vacation policy?" and "Tell me about vacation policy" = cache miss despite same intent

### When Exact Cache Is Useful
- High-traffic APIs where the same query is repeated thousands of times
- Programmatic clients that send identical queries
- First layer in a two-layer cache (cheap check before semantic search)

---

## 3. Semantic Cache

Instead of matching on exact text, match on **meaning**. If an incoming query is semantically close enough to a previously answered query, return the cached answer.

```
Incoming query: "How many weeks of maternity leave do I get?"
    │
    ▼
Embed query → vector [0.11, -0.85, 0.47, ...]
    │
    ▼
Search the CACHE (vector similarity search, not the knowledge base)
    │
    ├── Found cached query: "What is our parental leave policy?"
    │   Similarity = 0.94  > threshold (0.92)
    │
    └── CACHE HIT → Return cached answer instantly (~10ms)
```

### How It Works

```
Cache miss (first time):
  Query → embed → vector search cache → MISS
                                          │
                                          ▼
                                    Run full RAG pipeline
                                          │
                                          ▼
                                    Store: (query_vector, answer) in cache
                                          │
                                          ▼
                                    Return answer to user

Cache hit (similar query later):
  Query → embed → vector search cache → HIT (similarity > threshold)
                                          │
                                          ▼
                                    Return cached answer (~10ms) 
```

### Semantic Cache vs RAG Retrieval

| | Semantic Cache | RAG Retrieval |
|---|---|---|
| What's indexed | Previous query-answer pairs | Document chunks |
| Query | Embedded and searched | Embedded and searched |
| Hit | Returns the cached full answer | Returns document chunks for LLM |
| Miss | Falls through to full pipeline | Passes chunks to LLM |

The cache sits **before** the RAG pipeline, not inside it.

---

## 4. Threshold Tuning

The similarity threshold determines what counts as "same question."

| Threshold | Behavior | Risk |
|---|---|---|
| 0.99 | Almost exact match only | Too strict — barely any cache hits |
| **0.92–0.95** | Same intent, different phrasing | ✅ Sweet spot for most cases |
| 0.85–0.90 | Loosely related questions match | May return wrong cached answer |
| < 0.85 | Very different questions match | Dangerous — wrong answers served |

### Domain-Specific Tuning

| Domain | Recommended threshold | Reason |
|---|---|---|
| Legal / Medical | 0.95+ | Precision critical — close ≠ same |
| Enterprise HR FAQ | 0.92 | Moderate precision needed |
| General customer support | 0.88–0.90 | More flexibility acceptable |
| Coding assistant | 0.95+ | Different code questions are very different |

### Monitoring Threshold Health

```
Too high threshold:  cache hit rate < 5%   → threshold is too strict, lower it
Right threshold:     cache hit rate 20–40% → good balance
Too low threshold:   user complaints about wrong answers → threshold too loose, raise it
```

---

## 5. What Gets Stored

Each cache entry stores:

```python
cache_entry = {
    # The key for lookup
    "query_embedding": [0.12, -0.87, 0.44, ...],   # vector for similarity search
    "original_query":  "What is our parental leave policy?",

    # The cached answer
    "answer": "Employees receive 16 weeks of paid parental leave...",
    "source_chunks": [
        {"doc": "HR_Policy.pdf", "page": 8},
        {"doc": "Employee_Handbook.docx", "section": "3.2"}
    ],

    # Cache management
    "created_at":  "2026-04-01T10:00:00Z",
    "ttl_seconds": 86400,           # expire after 1 day
    "doc_ids":     ["hr_policy_v3", "employee_handbook_v2"],  # for invalidation

    # Optional: for multi-tenant
    "user_id":     None,            # None = shared across all users
    "tenant_id":   "acme_corp",
}
```

---

## 6. Hybrid Cache — Combining Both

Two cache layers in sequence. Cheapest check first, fall through to expensive only when necessary.

```
Query arrives
     │
     ▼
[LAYER 1] Exact Cache       hash lookup, ~1ms
     │
     ├── HIT  →  return instantly 
     │
     └── MISS
          │
          ▼
     [LAYER 2] Semantic Cache    embed + vector search, ~10ms
          │
          ├── HIT  →  return instantly 
          │
          └── MISS
               │
               ▼
          Full RAG Pipeline    embed + retrieve + rerank + LLM, ~500–2000ms
               │
               ▼
          Store in BOTH caches
               │
               ▼
          Return answer
```

### Why Layer 1 If Layer 2 Catches Everything?

Layer 2 (semantic) still requires:
1. Embedding the query (~20ms, small GPU/CPU cost)
2. Vector similarity search in cache

Layer 1 is a hash lookup — 1ms, zero compute. For truly identical queries (bots, programmatic clients, popular exact queries), Layer 1 is faster and free.

```
Layer 1 catches:  "What is parental leave policy?"  (verbatim repeat, 1ms)
Layer 2 catches:  "How many weeks maternity leave?"  (same intent, 10ms)
Both miss:        "Does parental leave apply if my partner gives birth?"  (new angle, full pipeline)
```

### The Mental Model
Same as CPU cache hierarchy:
- L1 cache (exact) → fastest, smallest, catches identical
- L2 cache (semantic) → fast, larger, catches similar
- Main memory (RAG pipeline) → slow, has everything

---

## 7. TTL — Cache Expiry

Cached answers go stale when underlying documents are updated. TTL (Time To Live) sets an automatic expiry.

| Knowledge type | TTL | Reason |
|---|---|---|
| Static contracts, legal docs | 7–30 days | Rarely changes |
| HR policies, product specs | 1–3 days | Changes monthly |
| Pricing, feature flags | 1–6 hours | Changes weekly |
| News, live inventory, real-time | No cache / 15 minutes | Changes constantly |
| Personalized answers | 0 (no cache) | User-specific, shouldn't be shared |

### Setting TTL in Redis

```python
import redis
import json

r = redis.Redis()

# Store with TTL
r.setex(
    name=f"sem_cache:{query_hash}",
    time=86400,              # TTL in seconds (1 day)
    value=json.dumps(cache_entry)
)

# Check and retrieve
entry = r.get(f"sem_cache:{query_hash}")
if entry:
    return json.loads(entry)   # cache hit
```

---

## 8. Cache Invalidation

TTL handles time-based expiry. But what if a document is updated before TTL expires?

### Tag-Based Invalidation

When storing a cache entry, tag it with the `doc_id`s of the source documents used to generate the answer.

```python
cache_entry["doc_ids"] = ["hr_policy_v3", "employee_handbook_v2"]
```

When a document is updated → invalidate all cache entries that used that document:

```python
def on_document_updated(doc_id):
    # Find all cache entries tagged with this doc_id
    entries_to_invalidate = cache.find_by_doc_id(doc_id)

    # Delete them
    for entry in entries_to_invalidate:
        cache.delete(entry.id)

    # Re-trigger ingestion of updated document
    ingestion_pipeline.run(doc_id)
```

### Invalidation Strategies

| Strategy | How | Best for |
|---|---|---|
| **TTL only** | Set appropriate TTL, wait for expiry | Simple, works when updates are predictable |
| **Tag-based** | Track doc_ids per entry, invalidate on update | Best for production — precise |
| **Full flush** | Clear entire cache on any doc update | Simple but wasteful |
| **Version-based** | Add doc version to cache key, new version = automatic miss | Good for versioned docs |

---

## 9. Where to Store the Cache

### Option 1: Redis (Production Default)

```python
# Redis with vector similarity via RediSearch module
from redis import Redis
from redisvl.index import SearchIndex

# Store vector + metadata
index.load([{
    "id": entry_id,
    "query_embedding": query_vector,
    "answer": answer_text,
    "doc_ids": doc_ids,
    "ttl": 86400
}])

# Query by similarity
results = index.query(VectorQuery(
    vector=query_embedding,
    vector_field_name="query_embedding",
    num_results=1,
    return_score=True
))
```

### Option 2: Qdrant (Dedicated Vector Cache)

```python
from qdrant_client import QdrantClient

client = QdrantClient(host="localhost", port=6333)

# Search cache
hits = client.search(
    collection_name="query_cache",
    query_vector=query_embedding,
    limit=1,
    score_threshold=0.92   # threshold built into search
)
```

### Option 3: GPTCache (Open Source Library)

Purpose-built semantic cache for LLMs. Plugs directly into LangChain and OpenAI client.

```python
from gptcache import cache
from gptcache.adapter import openai

cache.init(
    similarity_evaluation=ExactMatchEvaluation(),
    # or: EmbeddingSimilarityEvaluation(threshold=0.92)
)

# Now all openai calls go through cache automatically
response = openai.ChatCompletion.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": query}]
)
```

### Comparison

| Store | Speed | Ease | Vector Search | Best for |
|---|---|---|---|---|
| **Redis + RediSearch** |  Fast | Medium | Yes | Production, already using Redis |
| **Qdrant** | Fast | Easy | Yes | Dedicated cache store |
| **pgvector** | Medium | Easy | Yes | Already on Postgres |
| **GPTCache** | Fast | Very easy | Yes | Quick integration |
| **FAISS in-memory** |  Fastest | Easy | Yes | High throughput, cache lost on restart |

---

## 10. Cache in the Full Pipeline

```
User Query
    │
    ▼
[1] Exact Cache lookup             ~1ms
    ├── HIT  →  return answer 
    └── MISS ↓

[2] Embed query                    ~20ms

[3] Semantic Cache lookup          ~10ms
    ├── HIT  →  return answer 
    └── MISS ↓

[4] Query rewriting (if multi-turn)   ~100ms

[5] Hybrid retrieval (dense + BM25)   ~15ms

[6] Reranker                       ~100ms

[7] LLM generation                 ~500–2000ms

[8] Groundedness check             ~50ms

[9] Store in Exact + Semantic cache

[10] Return answer to user
```

### Expected Cache Hit Rates (Production)

| Query type | Hit rate |
|---|---|
| High-traffic FAQ (top 50 questions) | 60–80% |
| General enterprise Q&A | 20–40% |
| Highly personalized queries | 5–15% |
| Real-time / unique queries | < 5% |

---

## 11. Multi-tenant Caching

If your RAG serves multiple tenants (companies, teams, users), cache entries must be scoped correctly.

### Problem
User A from Acme Corp asks about their vacation policy.
User B from Beta Corp asks the same question.
They should NOT share a cache entry — their policies are different.

### Solution: Tenant-Scoped Cache Keys

```python
# Include tenant_id in the cache lookup
cache_key = {
    "query_embedding": embed(query),
    "tenant_id": "acme_corp",   # scope to tenant
}

# Only search cache entries for this tenant
results = cache.search(
    vector=query_embedding,
    filter={"tenant_id": "acme_corp"},
    threshold=0.92
)
```

### When NOT to Cache

| Scenario | Why |
|---|---|
| Personalized answers (user-specific data) | Answer differs per user |
| Real-time data queries | Answer changes too fast |
| PII-sensitive questions | Privacy risk if shared |
| Low-frequency unique questions | Cache overhead > benefit |

---

## 12. Key Takeaways

- **Semantic cache** catches same-intent, different-phrasing queries — the most common repetition pattern in real usage.
- **Exact cache** catches verbatim repeats — near-zero cost, always worth adding as Layer 1.
- **Hybrid cache** = exact first (1ms), semantic second (10ms), full pipeline last (500–2000ms). Cheapest check always first.
- **Threshold 0.92–0.95** is the sweet spot. Domain-specific systems need higher (legal, medical). General FAQ can go lower.
- **Tag cache entries with doc_ids** — this is what enables precise invalidation when a document is updated.
- **Multi-tenant systems** must scope cache by tenant — never share entries across tenants with different knowledge bases.

### Interview One-Liner
> *"I put a two-layer cache in front of the RAG pipeline. Layer 1 is an exact hash match — trivial cost, catches verbatim repeats. Layer 2 is a semantic cache — embed the query, search cached query-answer pairs by cosine similarity with a 0.92 threshold. Hits return in ~10ms instead of the full ~1500ms pipeline. Cache entries are tagged with source doc_ids and invalidated when those documents are updated. TTL is set based on knowledge freshness — 1 day for HR policies, 7+ days for stable contracts."*

---
