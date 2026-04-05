# Module 02f — Advanced RAG Techniques
> LLM Fundamentals Series · Query rewriting, HyDE, Self-RAG, CRAG, Agentic RAG

---

## Table of Contents
1. [Why Go Beyond Naive RAG](#1-why-go-beyond-naive-rag)
2. [Query Rewriting](#2-query-rewriting)
3. [HyDE — Hypothetical Document Embeddings](#3-hyde--hypothetical-document-embeddings)
4. [Self-RAG](#4-self-rag)
5. [Corrective RAG (CRAG)](#5-corrective-rag-crag)
6. [Agentic RAG](#6-agentic-rag)
7. [When to Use Each](#7-when-to-use-each)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Why Go Beyond Naive RAG

Naive RAG fails in predictable ways. Each advanced technique is a targeted fix for one of these failure modes.

| Failure mode | Symptom | Fix |
|---|---|---|
| Multi-turn query is ambiguous | "Can I carry those over?" retrieves wrong chunks | Query Rewriting |
| Abstract query doesn't embed well | Short questions produce weak embeddings | HyDE |
| Every query triggers retrieval even when unnecessary | Slow, expensive for simple questions | Self-RAG |
| Knowledge base is incomplete | No relevant docs found → hallucination | CRAG |
| Answer requires synthesizing multiple sources | Single retrieval call misses multi-hop connections | Agentic RAG |

> **Rule:** Start with naive RAG. Add advanced techniques only when you measure a specific failure mode. Don't over-engineer upfront.

---

## 2. Query Rewriting

### The Problem

In a multi-turn conversation, the current query often refers to context established earlier. Retrieval of the raw query fails because it's ambiguous without the history.

```
Turn 1: "What is the parental leave policy?"
         → good retrieval 

Turn 2: "How long is it?"
         → "it" = parental leave, but the raw query "How long is it?" retrieves nothing useful 

Turn 3: "Does it apply to adoptive parents too?"
         → again, "it" is unresolved 
```

### The Fix: Standalone Query Rewriting

Before retrieval, use a small LLM to rewrite the current query into a complete, self-contained question using conversation history.

```
History:
  User: "What is the parental leave policy?"
  Assistant: "Employees receive 16 weeks of paid parental leave..."

Current query: "Does it apply to adoptive parents too?"

Rewritten:  "Does the parental leave policy apply to adoptive parents?"
```

Now retrieval gets a complete, unambiguous query.

### Implementation

```python
rewrite_prompt = """
Given the conversation history below, rewrite the latest user query
into a complete, standalone question that can be understood without
the history.

History:
{conversation_history}

Latest query: {current_query}

Rewritten query:
"""

rewritten_query = small_llm.generate(rewrite_prompt)
chunks = retrieve(rewritten_query)   # use rewritten query for retrieval
```

### Model for Rewriting
Use a **small, fast model** — GPT-4o-mini, Llama 3.1 8B. The rewrite doesn't need to be creative, just standalone. Cost: ~$0.0001 per rewrite.

### Multi-query Expansion (Variant)
Generate **multiple** rewritten queries and retrieve for all of them. Merge results. Improves recall at the cost of latency and token usage.

```
Original query: "How does our vacation policy compare to industry standard?"

Generated queries:
  1. "What is our company's vacation policy?"
  2. "What is the industry standard for paid vacation days?"
  3. "How many vacation days do companies typically offer?"

→ Retrieve for all three, merge results
```

---

## 3. HyDE — Hypothetical Document Embeddings

### The Problem

Short factual questions often produce weak, vague embeddings because they lack the vocabulary and context of the actual documents.

```
Query:  "What is our data retention policy?"
Embed → [0.21, -0.44, 0.18, ...]   ← sparse signal, generic embedding

Actual relevant doc:
  "Data Retention Policy: All customer data shall be retained for a 
   period of 7 years in compliance with regulatory requirements. 
   Deletion requests must be processed within 30 days..."
Embed → [0.19, -0.51, 0.22, ...]   ← much richer, more specific
```

The question and the answer have different "shapes" in embedding space. Short question ≠ dense document passage.

### The Fix: Generate a Hypothetical Answer First

Use an LLM to generate a *hypothetical* (possibly imperfect) answer, then embed **that** instead of the raw query. The hypothetical answer is in document-like language, so it embeds much closer to actual relevant documents.

```
Query: "What is our data retention policy?"
  │
  ▼
LLM generates hypothetical:
  "Our data retention policy states that customer data is retained for
   7 years in compliance with GDPR and CCPA. Personal data deletion 
   requests are processed within 30 days of submission..."
  │
  ▼
Embed the hypothetical  →  [0.20, -0.49, 0.23, ...]   ← closer to real docs
  │
  ▼
Retrieve using hypothetical embedding  →  better recall
  │
  ▼
Use retrieved chunks (not the hypothesis) to generate the actual answer
```

> The hypothetical is just for retrieval — it is never shown to the user. The actual answer still comes from retrieved ground-truth chunks.

### When HyDE Helps

| Query type | HyDE benefit |
|---|---|
| Short, abstract questions | High — question is far from document vocabulary |
| Specific questions with key terms | Low — keyword already guides retrieval well |
| Domain-specific jargon in the question | Low — question already uses document language |
| Complex conceptual questions | High |

### Cost
One extra LLM call (small model, ~50 tokens) per query. Worth it when abstract queries show low retrieval recall.

---

## 4. Self-RAG

### The Problem

Naive RAG always retrieves, for every query. But:
- "What's 2 + 2?" doesn't need retrieval
- "When was the company founded?" might be answerable from the model's knowledge
- Running retrieval for every query wastes latency and cost

### The Fix: Let the LLM Decide

Self-RAG trains the model with special reflection tokens that let it decide:
1. **Should I retrieve?** (`[Retrieve]` / `[No Retrieve]`)
2. **Is this retrieved chunk relevant?** (`[Relevant]` / `[Irrelevant]`)
3. **Is my answer grounded?** (`[Fully supported]` / `[Partially supported]` / `[Not supported]`)

```
Query: "What is 15% of 240?"
  │
  LLM: [No Retrieve]   ← simple math, no retrieval needed
  │
  ▼
Answer: "36"   ← answered from model knowledge

---

Query: "What is our Q3 budget allocation?"
  │
  LLM: [Retrieve]   ← needs internal knowledge
  │
  ▼
Retrieved chunks
  │
  LLM evaluates each: [Relevant] / [Irrelevant]
  │
  ▼
Generate using relevant chunks only
  │
  LLM: [Fully supported]  ← answer checks out
  │
  ▼
Return answer
```

### In Practice (Without Full Self-RAG Training)

You can approximate Self-RAG behavior with prompting:

```python
routing_prompt = """
Does answering this question require looking up information
from the company knowledge base?

Question: {query}

Answer YES if it needs internal company information.
Answer NO if it's a general question or simple calculation.
"""

needs_retrieval = classifier(query)   # YES or NO

if needs_retrieval == "NO":
    answer = llm.generate(query)           # direct answer
else:
    chunks = retrieve(query)
    answer = llm.generate(query, chunks)   # RAG answer
```

---

## 5. Corrective RAG (CRAG)

### The Problem

RAG assumes the knowledge base contains the answer. But what if it doesn't?

```
User: "What did our CEO say about the Q4 results in last week's call?"
      ← This might not be in the indexed knowledge base at all

Naive RAG retrieves whatever is most similar → LLM hallucinates the rest
```

### The Fix: Evaluate Retrieved Docs — Fall Back to Web Search

After retrieval, an evaluator scores each retrieved chunk:
- **Correct** — clearly relevant and accurate
- **Ambiguous** — might be relevant, unclear
- **Incorrect** — irrelevant, or contradicts query

Based on the evaluation:

```
Retrieved chunks evaluated:
  │
  ├── All CORRECT        →  Use as-is, proceed to generation
  │
  ├── Some AMBIGUOUS     →  Supplement with web search, combine
  │
  └── All INCORRECT      →  Discard all, fall back entirely to web search
                              (or return "I don't know" if web search not available)
```

### The Full CRAG Flow

```
Query
  │
  ▼
Retrieve chunks from vector store
  │
  ▼
[Evaluator LLM] scores each chunk: Correct / Ambiguous / Incorrect
  │
  ├── Correct   →  Use retrieved chunks
  │                     │
  │                     ▼ Generate answer
  │
  ├── Ambiguous →  Reformulate query → Web Search → Combine with vector results
  │                     │
  │                     ▼ Generate from combined context
  │
  └── Incorrect →  Query reformulation → Web Search only
                        │
                        ▼ Generate from web results
```

### When to Add CRAG
- Knowledge base is incomplete or not frequently updated
- Queries can legitimately be about very recent events
- Cost of hallucination is high (legal, medical, financial)
- Web access is acceptable from a compliance standpoint

---

## 6. Agentic RAG

### The Problem

Some questions require multiple retrieval steps from multiple sources, and the right sequence can't be determined upfront.

```
"Compare our Q3 performance against industry benchmarks,
 and summarize what changes our CEO proposed in response."

This requires:
  Step 1: Retrieve internal Q3 performance docs
  Step 2: Retrieve industry benchmark data (maybe from web)
  Step 3: Retrieve CEO statements about Q3 response
  Step 4: Synthesize all three into a coherent answer

→ You can't do this in one retrieval call
```

### The Fix: An Agent That Plans and Executes

An LLM acts as an agent — it decides what to retrieve, from where, in what order, and loops until it has enough information to answer.

```
Query
  │
  ▼
[Agent / Planner LLM]
  │
  ├── Tool 1: search_internal_docs("Q3 performance metrics")
  │     → Retrieved: Revenue report, cost breakdown
  │
  ├── Tool 2: search_web("industry Q3 benchmarks 2025")
  │     → Retrieved: Industry comparison report
  │
  ├── Tool 3: search_internal_docs("CEO Q3 response strategy")
  │     → Retrieved: CEO memo, board presentation
  │
  └── Agent decides: "I have enough context now"
          │
          ▼
     Synthesize answer from all retrieved context
```

### Tools Available to the Agent

```python
tools = [
    search_vector_store,     # search internal knowledge base
    search_web,              # internet search (CRAG fallback)
    query_sql_database,      # structured data lookup
    call_api,                # live data (inventory, pricing)
    read_file,               # specific document lookup
    run_code,                # calculation or data analysis
]
```

### Implementation with LangGraph

```python
from langgraph.graph import StateGraph

# Define agent state
class AgentState(TypedDict):
    query: str
    retrieved_docs: list
    tool_calls: list
    final_answer: str

# Nodes
def plan(state):      # LLM decides what to retrieve next
def retrieve(state):  # execute retrieval tool call
def evaluate(state):  # do I have enough context?
def generate(state):  # synthesize final answer

# Graph
graph = StateGraph(AgentState)
graph.add_node("plan", plan)
graph.add_node("retrieve", retrieve)
graph.add_node("evaluate", evaluate)
graph.add_node("generate", generate)

# Conditional loop: keep retrieving until agent is satisfied
graph.add_conditional_edges("evaluate", lambda s: "generate" if done(s) else "plan")
```

### Agentic RAG vs Normal RAG

| | Normal RAG | Agentic RAG |
|---|---|---|
| Retrieval calls | 1 | Multiple (planned dynamically) |
| Tools | Vector store only | Vector store + web + SQL + APIs |
| Latency | Low (~200ms retrieval) | High (multiple LLM + retrieval rounds) |
| Complexity | Simple | High |
| Best for | Single-hop questions | Multi-hop, multi-source synthesis |

---

## 7. When to Use Each

```
Start with: Naive RAG (always)
     │
     ├── Multi-turn queries fail?
     │      └── Add: Query Rewriting
     │
     ├── Abstract queries have low retrieval recall?
     │      └── Add: HyDE
     │
     ├── Too many unnecessary retrieval calls (simple queries)?
     │      └── Add: Self-RAG (intent routing)
     │
     ├── Knowledge base is incomplete / queries go beyond it?
     │      └── Add: CRAG (with web search fallback)
     │
     └── Questions require synthesizing multiple sources / multi-hop?
            └── Add: Agentic RAG
```

### Complexity vs Benefit

| Technique | Complexity | Benefit | When to add |
|---|---|---|---|
| Query Rewriting | Low | High for multi-turn | Almost always |
| HyDE | Low | Medium | When abstract queries fail |
| Self-RAG | Medium | Medium | When retrieval is overused |
| CRAG | Medium | High | When corpus is incomplete |
| Agentic RAG | High | High for complex | Multi-hop queries only |

---

## 8. Key Takeaways

- **Query Rewriting** — makes multi-turn conversations work by resolving ambiguous references before retrieval.
- **HyDE** — generates a hypothetical answer to use as the retrieval query; helps when short questions embed poorly.
- **Self-RAG** — the model decides whether retrieval is needed, which chunk is relevant, and whether the answer is grounded.
- **CRAG** — evaluates retrieved chunks and falls back to web search when the knowledge base is insufficient.
- **Agentic RAG** — an LLM agent dynamically plans multi-step retrieval from multiple sources to answer complex questions.
- **Don't add complexity without measuring a failure mode.** Each technique has a cost (latency, complexity, money).

### Interview One-Liner
> *"I start with naive RAG and add techniques only when I measure specific failures. Query rewriting for multi-turn ambiguity. HyDE when abstract queries have low retrieval recall. CRAG when the knowledge base is incomplete and I need a web search fallback. Agentic RAG only for multi-hop questions that require synthesizing information from multiple sources — the latency cost is only justified for genuinely complex queries."*

---

