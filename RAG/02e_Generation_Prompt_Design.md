# Module 02e — Generation & Prompt Design
> LLM Fundamentals Series · Prompt anatomy, temperature, grounding check, hallucination prevention, multi-turn

---

## Table of Contents
1. [The Generation Step in RAG](#1-the-generation-step-in-rag)
2. [Anatomy of a RAG Prompt](#2-anatomy-of-a-rag-prompt)
3. [System Prompt Design](#3-system-prompt-design)
4. [Context Assembly](#4-context-assembly)
5. [Conversation History (Multi-turn)](#5-conversation-history-multi-turn)
6. [Generation Parameters](#6-generation-parameters)
7. [Grounding Instructions](#7-grounding-instructions)
8. [Groundedness Check — Hallucination Detection](#8-groundedness-check--hallucination-detection)
9. [The "I Don't Know" Response](#9-the-i-dont-know-response)
10. [LLM Choices for Generation](#10-llm-choices-for-generation)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. The Generation Step in RAG

After retrieval gives you the top-5 chunks, those chunks are assembled into a prompt and sent to an LLM. The LLM's job is to:

1. Read the retrieved context
2. Answer the user's question **using only that context**
3. Cite which source the answer came from
4. Say "I don't know" if the answer isn't in the context

The LLM does not use its own parametric memory for facts — it reads from the chunks like an open-book exam.

```
Retrieved chunks (top-5)
        +
System prompt (instructions)
        +
Conversation history
        +
User query
        │
        ▼
     [LLM]
        │
        ▼
Grounded answer + citations
```

---

## 2. Anatomy of a RAG Prompt

```
┌─────────────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT                                                   │
│ (instructions, persona, grounding rules, citation format)       │
├─────────────────────────────────────────────────────────────────┤
│ RETRIEVED CONTEXT                                               │
│ [Source: HR_Policy_v3.pdf, Page 12]                            │
│ Employees are entitled to 15 days of paid vacation per year...  │
│                                                                 │
│ [Source: Employee_Handbook.docx, Section 4.2]                  │
│ Unused vacation days may be carried over up to 5 days...        │
├─────────────────────────────────────────────────────────────────┤
│ CONVERSATION HISTORY                                            │
│ User: What is the vacation policy?                              │
│ Assistant: Employees get 15 days of paid vacation per year...   │
├─────────────────────────────────────────────────────────────────┤
│ USER QUERY                                                      │
│ Can I carry over my unused days to next year?                   │
└─────────────────────────────────────────────────────────────────┘
```

This order matters. Most LLMs pay more attention to content near the **end** of the prompt (close to where they generate). Put the user query last, context close to the query.

---

## 3. System Prompt Design

The system prompt defines the model's behavior. For RAG, it must establish:

### Grounding Rule (most important)
```
You are a helpful internal knowledge assistant.
Answer questions ONLY using the information provided in the context below.
Do not use any knowledge from your training data.
If the answer is not present in the context, respond with:
"I don't have information on this topic in the knowledge base."
```

### Citation Format
```
Always cite your sources at the end of your answer.
Format: [Source: {filename}, {section/page}]
Example: [Source: HR_Policy_v3.pdf, Page 12]
```

### Tone and Persona
```
You are an assistant for Acme Corp employees.
Be concise, professional, and direct.
Do not speculate or add information beyond what's in the context.
```

### Full Example System Prompt
```
You are a helpful internal knowledge assistant for Acme Corp employees.

Rules:
1. Answer ONLY from the provided context. Never use outside knowledge.
2. If the answer is not in the context, say: "I don't have information
   on this in the knowledge base."
3. Always cite your source: [Source: filename, page/section]
4. Be concise and direct. Do not repeat the question.
5. If the question is ambiguous, ask one clarifying question.
```

---

## 4. Context Assembly

How you assemble the retrieved chunks into the prompt affects answer quality.

### Chunk Ordering

**Option 1: Relevance order** — highest score first
```
[Source: doc_A, score=0.94]  ← most relevant
[Source: doc_B, score=0.91]
[Source: doc_C, score=0.87]
[Source: doc_D, score=0.82]
[Source: doc_E, score=0.79]  ← least relevant
```

**Option 2: Lost-in-the-middle mitigation** — highest score first AND last, lower scores in middle
```
[Source: doc_A, score=0.94]  ← high (model pays attention here)
[Source: doc_C, score=0.87]
[Source: doc_D, score=0.82]
[Source: doc_E, score=0.79]
[Source: doc_B, score=0.91]  ← high (model pays attention here)
```

LLMs tend to underweight information in the middle of long contexts. Putting the most important chunks at beginning and end helps.

### How Many Chunks

| Scenario | Recommended chunks |
|---|---|
| Simple, single-source question | 3–5 chunks |
| Complex, multi-aspect question | 5–10 chunks |
| Context window is large (200K) | Up to 20 chunks |

More chunks = more context for the LLM but also more noise. Start with 5, measure answer quality, adjust.

### Chunk Format
Label each chunk clearly so the model knows which source it came from:

```
=== Source 1 ===
Document: HR_Policy_v3.pdf
Section: Annual Leave, Page 12
Content: Employees are entitled to 15 days of paid vacation per year.
Vacation accrues at 1.25 days per month of employment...

=== Source 2 ===
Document: Employee_Handbook.docx
Section: 4.2 Leave Carry-Over
Content: Unused annual leave days may be carried over to the following
year, subject to a maximum carry-over of 5 days...
```

---

## 5. Conversation History (Multi-turn)

RAG needs to maintain conversation context so follow-up questions make sense.

### The Problem

```
Turn 1: "What is our vacation policy?"
         → Retrieved: HR vacation policy chunks
         → Answer: "You get 15 days per year."

Turn 2: "Can I carry those over?"
         ← "those" refers to vacation days — but the query alone is ambiguous
         ← Retrieval of "Can I carry those over?" will fail
```

### Solution: Store History in Session

```python
session = {
    "session_id": "user_123_session_456",
    "history": [
        {"role": "user", "content": "What is our vacation policy?"},
        {"role": "assistant", "content": "Employees get 15 days of paid vacation per year, accruing at 1.25 days per month. [Source: HR_Policy.pdf, Page 12]"},
        {"role": "user", "content": "Can I carry those over?"}
    ]
}
```

### Session Storage

| Store | TTL | Notes |
|---|---|---|
| **Redis** | 30 min inactivity | Best for production — fast, TTL built-in |
| **DynamoDB** | Configurable | Good for serverless |
| **Postgres** | Manual cleanup | Fine if already on Postgres |

**Key:** `session:{user_id}:{session_id}`
**Value:** JSON list of `{role, content, timestamp}`

### Context Window Management

You can't stuff unlimited history into the context. Cap at:
- Last **10 turns** of conversation, OR
- Last **4,000 tokens** of history (whichever comes first)

For older turns, **summarize** instead of dropping:
```
Summary of earlier conversation:
User asked about vacation policy. Assistant explained 15 days/year entitlement
and accrual rate.
```

---

## 6. Generation Parameters

### Temperature
Controls randomness in token selection.

| Temperature | Behavior | Use in RAG |
|---|---|---|
| 0.0 | Fully deterministic — always picks highest probability token | Factual Q&A |
| 0.1–0.3 | Very low randomness — precise, consistent |  RAG default |
| 0.7–1.0 | Creative, varied responses |  Too much hallucination risk |
| > 1.0 | Very random, often incoherent |  Never for RAG |

> **RAG default: temperature = 0.1–0.2.** You want the model to read the context and report it accurately — not get creative.

### Max Tokens
Set an output limit. Long responses in RAG often mean the model is padding or elaborating beyond the context.

```python
response = llm.generate(
    prompt=assembled_prompt,
    temperature=0.1,
    max_tokens=500,      # keep answers concise
    stop=["\n\n\n"]      # stop on triple newline
)
```

### Top-p (Nucleus Sampling)
Keeps the smallest set of tokens whose cumulative probability ≥ p.

```
top_p=0.9: sample from tokens covering 90% of probability mass
```

At low temperatures (< 0.3), top-p has little effect. Keep at default (0.9–1.0) for RAG.

---

## 7. Grounding Instructions

The most important prompt engineering in RAG. The model must be explicitly told to stay grounded.

### Weak Grounding (bad)
```
Answer the user's question using the context below.
Context: ...
```
→ Model may supplement context with training knowledge

### Strong Grounding (good)
```
Answer the user's question ONLY using the context below.
Do NOT use any knowledge from your training.
If the answer is not in the context, say exactly:
"I don't have information on this in the knowledge base."
Context: ...
```
→ Model stays within the retrieved context

### Why Models Still Hallucinate Despite Instructions

1. **Training data pressure** — the model has strong priors about certain topics
2. **Ambiguous instructions** — "use the context" vs "ONLY use the context"
3. **Context doesn't contain the answer** — model fills the gap
4. **Long context** — model loses track of the grounding instruction

Solutions: explicit "I don't know" instruction, groundedness check post-generation, low temperature.

---

## 8. Groundedness Check — Hallucination Detection

After generation, verify the answer is supported by the retrieved context before returning to user.

### Method 1: NLI (Natural Language Inference)

Run a lightweight NLI model that classifies: does the answer *entail* the context?

```
Input:  premise   = retrieved chunks
        hypothesis = generated answer

Output: ENTAILMENT (supported) / NEUTRAL / CONTRADICTION (not supported)
```

**Models:**
- `cross-encoder/nli-deberta-v3-large`
- `microsoft/deberta-v3-large-mnli`

Fast, cheap, no LLM call needed.

### Method 2: LLM-as-Judge

Use a second LLM call to evaluate groundedness.

```
Prompt:
  Context: [retrieved chunks]
  Answer:  [generated answer]

  Is every claim in the answer supported by the context?
  Respond: YES or NO. If NO, identify the unsupported claim.
```

More expensive but more nuanced. Can identify *which* part of the answer is unsupported.

### The Validation Flow

```
Generated Answer
     │
     ▼
[Groundedness Check]
     │
     ├── PASS  →  Return answer + citations to user 
     │
     └── FAIL
          │
          ├──  Retry once with stricter prompt
          │       ("Base your answer SOLELY on the exact text in the context")
          │          │
          │          ├── PASS  →  Return answer 
          │          │
          │          └── FAIL  →  Return "I don't have information on this" 
          │
          └── (If time budget allows) Trigger re-retrieval with expanded query
```

---

## 9. The "I Don't Know" Response

When the knowledge base doesn't contain the answer, the model must say so clearly rather than hallucinate.

### Good "I Don't Know" Responses
```
 "I don't have information on this in the knowledge base."
"The retrieved documents don't cover this topic. You may want to contact HR directly."
"I couldn't find relevant information. Could you rephrase or provide more context?"
```

### Bad "I Don't Know" Responses
```
 "Based on general knowledge, I believe..." (mixing RAG with parametric)
Confidently stating something not in context
Vague deflection without telling user what to do next
```

### When to Trigger "I Don't Know"

1. **Pre-retrieval:** Retrieval confidence is too low (no chunk scores above threshold)
2. **Post-retrieval:** All retrieved chunks have low relevance scores
3. **Post-generation:** Groundedness check fails twice

```python
# Pre-retrieval check
if max(similarity_scores) < 0.7:
    return "I don't have information on this topic."

# Post-retrieval check
if all(score < 0.75 for score in top_k_scores):
    return "I couldn't find relevant information in the knowledge base."
```

---

## 10. LLM Choices for Generation

### Closed Source

| Model | Context | Best for |
|---|---|---|
| `GPT-4o` (OpenAI) | 128K | Best quality, fast, function calling |
| `GPT-4o-mini` | 128K | Cheap, good for simple Q&A |
| `Claude 3.5 Sonnet` | 200K | Long context, careful reasoning |
| `Claude 3 Haiku` | 200K | Very fast, cheap, good quality |
| `Gemini 1.5 Pro` | 1M | Massive context window |

### Open Source (Self-hosted)

| Model | Context | Best for |
|---|---|---|
| `Llama 3.1 70B` | 128K | Best OSS quality |
| `Llama 3.1 8B` | 128K | Fast, cheap, good for simple Q&A |
| `Mistral 7B` | 32K | Lightweight, fast |
| `Qwen2.5 72B` | 128K | Strong multilingual |

### Model Routing (Cost Optimization)

Don't use the expensive model for every query:

```
Simple Q&A (single chunk, high confidence)  →  GPT-4o-mini / Llama 8B  (cheap)
Complex reasoning (multi-chunk, nuanced)    →  GPT-4o / Llama 70B      (expensive)
```

A lightweight classifier routes queries before LLM call. Typically saves 60–80% of LLM costs.

---

## 11. Key Takeaways

- The generation step is an open-book exam — the LLM reads retrieved context and reports from it, not from memory.
- **Temperature 0.1–0.2** for RAG — low randomness, high precision.
- **Grounding instruction must be explicit and strong** — "ONLY use the provided context."
- **Groundedness check** (NLI or LLM-as-judge) is essential post-generation to catch hallucinations.
- **"I don't know"** is a first-class response in RAG — better than a confident wrong answer.
- **Multi-turn** requires session storage (Redis) and context window management.
- Model routing (small model for simple, large for complex) dramatically reduces cost.

### Interview One-Liner
> *"After retrieval I assemble a prompt: system instructions with explicit grounding rules, retrieved chunks with source labels, conversation history, then the query. I generate at low temperature (0.1–0.2) to stay factual. Post-generation I run a groundedness check via NLI to detect hallucination — if it fails twice, I return 'I don't have information on this' rather than a hallucinated answer."*

---

