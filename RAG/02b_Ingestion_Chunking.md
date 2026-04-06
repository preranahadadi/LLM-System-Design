# Module 02b — Ingestion & Chunking
> LLM Fundamentals Series · Document loading, cleaning, chunking strategies, overlap

---

## Table of Contents
1. [The Ingestion Pipeline](#1-the-ingestion-pipeline)
2. [Document Loading](#2-document-loading)
3. [Cleaning & Preprocessing](#3-cleaning--preprocessing)
4. [Why Chunking Matters](#4-why-chunking-matters)
5. [Chunking Strategies](#5-chunking-strategies)
6. [Chunk Size Tradeoff](#6-chunk-size-tradeoff)
7. [Overlap](#7-overlap)
8. [Parent-Child Chunking](#8-parent-child-chunking)
9. [Metadata Per Chunk](#9-metadata-per-chunk)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. The Ingestion Pipeline

The offline pipeline that converts raw documents into a searchable vector index.

```
Raw Docs
    │
    ▼
[LOADER]         → read files, extract text
    │
    ▼
[CLEANER]        → remove noise, fix encoding
    │
    ▼
[CHUNKER]        → split into retrievable pieces   ← most impactful step
    │
    ▼
[EMBEDDER]       → text → dense vector
    │
    ▼
[VECTOR DB]      → store vectors + metadata
```

> Quality of ingestion = ceiling of your entire RAG system.
> Garbage in → garbage retrieval → garbage answers, regardless of LLM quality.

---

## 2. Document Loading

Different parsers handle different file formats:

| Format | Library / Tool |
|---|---|
| PDF | `PyMuPDF`, `PDFMiner`, `pdfplumber` |
| DOCX | `python-docx` |
| HTML | `BeautifulSoup`, `trafilatura` |
| Markdown | Direct text read |
| Confluence | Confluence REST API + webhook |
| Notion | Notion SDK |
| Google Docs | Google Drive API |
| CSV / JSON | `pandas`, direct parse |

### Source Triggers
The pipeline runs when documents are added or updated. Trigger mechanisms:
- **Webhook** — Confluence/Notion sends event on doc change
- **File watcher** — watches S3 bucket or filesystem for new files
- **Scheduled job** — polls source every N minutes
- **Manual upload** — admin UI triggers ingestion

---

## 3. Cleaning & Preprocessing

Raw documents contain a lot of noise. Clean before chunking.

### What to Remove
- Headers and footers (page numbers, company logos, navigation menus)
- Boilerplate repeated across pages ("Confidential — Internal Use Only")
- HTML tags, CSS, JavaScript
- Broken encoding characters (`\x84`, `â€™`)
- Excessive whitespace and empty lines

### What to Preserve
- Section headings (useful for structure-aware chunking and metadata)
- Tables (convert to text or Markdown table format)
- Lists (preserve bullet structure)
- Code blocks (keep as-is, don't split mid-code)

### Table Handling
Tables are tricky. Options:
1. Convert to Markdown table → embed as text
2. Extract each row as a separate chunk
3. Summarize the table with an LLM → embed the summary

---

## 4. Why Chunking Matters

You cannot embed an entire 200-page document as one vector — the resulting embedding would be too diluted to match any specific query.

You also cannot embed one sentence at a time — the chunk would be too specific and lack the surrounding context needed to answer a question.

Chunking is the decision of **how to split** documents into pieces that are:
- **Small enough** to be precisely retrieved for a specific query
- **Large enough** to contain enough context for the LLM to answer from

> Chunking is arguably the single most impactful parameter in RAG.
> A great embedding model on bad chunks performs worse than a decent model on good chunks.

---

## 5. Chunking Strategies

### Strategy 1: Fixed-Size Chunking
Split every N tokens with M tokens of overlap.

```
Document: [token_1 ... token_512 | token_462 ... token_974 | ...]
                                   ↑ overlap zone ↑
```

-  Simple, fast, predictable
-  God general-purpose baseline
- May cut sentences or ideas mid-thought
- **Default:** 512 tokens, 10–20% overlap

---

### Strategy 2: Sentence-Aware Chunking
Split on sentence boundaries, group sentences until size limit is reached.

```
Sentence 1. Sentence 2. Sentence 3.  → Chunk 1 (under limit)
Sentence 4. Sentence 5.              → Chunk 2
```

- Never cuts mid-sentence
- More natural reading units
- Variable chunk sizes
- **Best for:** Articles, conversational text, documentation

---

### Strategy 3: Recursive Character Splitting
Try to split on `\n\n` first, then `\n`, then spaces, then characters. Preserves document structure as much as possible.

```python
# LangChain's RecursiveCharacterTextSplitter default separators
separators = ["\n\n", "\n", " ", ""]
```

- Respects paragraph and line structure
- LangChain default — widely used
- Still character/token-based, not semantic
- **Best for:** General prose with paragraph breaks

---

### Strategy 4: Document-Structure Chunking
Split on headings (H1 / H2 / H3). Each section becomes its own chunk.

```
# Section 1: Vacation Policy       → Chunk 1
## 1.1 Annual Leave Entitlement    → Chunk 2
## 1.2 Carry-Over Rules            → Chunk 3
# Section 2: Sick Leave            → Chunk 4
```

- Sections are naturally coherent units
- Headings make great metadata labels
- Sections can be very long (needs secondary split) or very short
- **Best for:** PDFs, wikis, handbooks, structured documentation

---

### Strategy 5: Semantic Chunking
Embed each sentence, then split where cosine similarity between adjacent sentences drops below a threshold. Split at topic boundaries, not arbitrary token counts.

```
Sentence 1 → embed → sim(S1, S2) = 0.91  → same chunk
Sentence 2 → embed → sim(S2, S3) = 0.91  → same chunk
Sentence 3 → embed → sim(S3, S4) = 0.43  → SPLIT HERE  ← topic changed
Sentence 4 → embed → ...
```

- Chunks align with actual topic shifts
- Best quality for dense technical text
- Slow — requires embedding every sentence at index time
- Expensive
- **Best for:** Academic papers, legal documents, research reports

---

### Strategy 6: Propositional / Agentic Chunking
Use an LLM to extract atomic propositions from the text. Each fact becomes a standalone chunk.

```
Input paragraph:
"Employees get 15 days of vacation. This accrues monthly.
 Unused days can be carried over up to 5 days."

Output chunks:
- "Employees receive 15 days of paid vacation per year."
- "Vacation days accrue at 1.25 days per month."
- "Unused vacation days can be carried over, up to a maximum of 5 days."
```

- Maximum retrieval precision
- Each chunk answers exactly one question
- Very expensive (LLM call per passage)
- Slow
- **Best for:** High-stakes FAQ systems, medical/legal, when precision > cost

---

### Strategy Comparison

| Strategy | Speed | Quality | Semantic Awareness | Best Default? |
|---|---|---|---|---|
| Fixed-size |  Fast | Good | None | Yes |
| Sentence-aware | Fast | Better | Low | For articles |
| Recursive | Fast | Good | Low | LangChain default |
| Document-structure | Fast | Good | Medium | For structured docs |
| Semantic | Slow | Best | High | When budget allows |
| Propositional | Very slow | Highest | Highest | High-stakes only |

---

## 6. Chunk Size Tradeoff

```
                SMALL CHUNKS              LARGE CHUNKS
                (< 128 tokens)            (> 1024 tokens)

Retrieval       Very precise              Imprecise (diluted signal)
Context         Loses surrounding info    Rich context
LLM Input       Many chunks needed        Few chunks needed
Noise           Low                       High (irrelevant content included)
```

### Sweet Spot
**256–512 tokens** works well for most use cases.

Rule of thumb:
- If questions are specific and narrow → smaller chunks
- If questions require broad context → larger chunks
- If unsure → start at 512, measure Recall@5, adjust

---

## 7. Overlap

Always add overlap between adjacent chunks. This prevents important information at chunk boundaries from being split and lost.

```
Chunk 1:  [token 1  ........  token 512]
Chunk 2:             [token 462 ........  token 974]
                      ↑     overlap     ↑
                      (50–100 tokens)
```

**Why overlap matters:**
If a key sentence falls at the end of Chunk 1 and its explanation starts at the beginning of Chunk 2, without overlap you'd retrieve one without the other. Overlap ensures boundary information appears in at least one chunk that retrieval can return.

**Typical overlap:** 10–20% of chunk size.
- Chunk size 512 → overlap 50–100 tokens
- Chunk size 256 → overlap 25–50 tokens

---

## 8. Parent-Child Chunking

An advanced strategy that maintains two levels of chunks:
- **Child chunks** (small, ~128 tokens) → used for retrieval precision
- **Parent chunks** (large, ~512 tokens) → sent to the LLM for context

```
RETRIEVAL: search using small child chunks  →  high precision match
GENERATION: return the parent chunk to LLM  →  full context preserved

Child:   "The return window is 30 days."
          ↑ this gets retrieved precisely

Parent:  "Electronics return policy: Items may be returned within
          30 days of purchase. Original packaging required.
          Refund issued within 5–7 business days. Exceptions
          apply for damaged or defective items..."
          ↑ this gets sent to the LLM
```

Best of both worlds: **precision in retrieval, context in generation.**

### Implementation in LangChain
```python
from langchain.retrievers import ParentDocumentRetriever

retriever = ParentDocumentRetriever(
    vectorstore=child_vectorstore,    # small chunks indexed here
    docstore=parent_docstore,         # full parent chunks stored here
    child_splitter=child_splitter,    # 128 token splitter
    parent_splitter=parent_splitter,  # 512 token splitter
)
```

---

## 9. Metadata Per Chunk

Every chunk must carry metadata. This enables citations, filtered search, freshness-based ranking, and access control.

```python
chunk = {
    "text": "Employees are entitled to 15 days of paid vacation per year...",

    # Identity
    "doc_id":          "hr_policy_v3",
    "chunk_id":        "hr_policy_v3_chunk_12",

    # Source information (for citations)
    "source_url":      "confluence.company.com/HR/vacation",
    "source_filename": "HR_Policy_v3.pdf",
    "page_number":     4,
    "section_heading": "Annual Leave Entitlement",

    # Temporal (for freshness filtering)
    "created_at":      "2026-01-15",
    "updated_at":      "2026-03-01",

    # Access control
    "department":      "HR",
    "access_level":    "all_employees",

    # Author
    "author":          "HR Team"
}
```

### Using Metadata for Filtered Search
```python
# Only search HR docs updated in the last year
results = vector_db.query(
    query_embedding=embed("vacation policy"),
    filter={
        "department": "HR",
        "updated_at": {"$gt": "2025-04-01"}
    },
    top_k=10
)
```

---

## 10. Key Takeaways

- Ingestion quality determines the ceiling of your RAG system — no retrieval algorithm can recover from bad chunks.
- **Fixed-size with overlap** is the safe default (512 tokens, 10–20% overlap).
- **Semantic chunking** gives best quality but is expensive — use when precision is critical.
- **Parent-child chunking** is the best advanced strategy — small chunks for retrieval, large for generation.
- **Overlap** is non-negotiable — always add it to prevent boundary information loss.
- **Metadata** is as important as the chunk text — it enables citations, filtering, and freshness.

### Interview One-Liner
> *"I use recursive or sentence-aware chunking at 512 tokens with 10–20% overlap. For high-precision systems I use parent-child chunking — small child chunks for retrieval, large parent chunks sent to the LLM. Every chunk carries metadata: source, page, section, timestamp, and access level."*

---
