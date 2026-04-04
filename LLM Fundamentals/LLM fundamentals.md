# 01 — LLM Fundamentals

> Everything you need to know about how Large Language Models work — simple explanations with keywords for interviews.

---

## Table of Contents

1. [What is an LLM?](#1-what-is-an-llm)
2. [How LLMs Work — Transformers](#2-how-llms-work--transformers)
3. [Tokens](#3-tokens)
4. [Embeddings](#4-embeddings)
5. [Context Window](#5-context-window)
6. [Attention Mechanism](#6-attention-mechanism)
7. [Temperature, Top-p, Top-k](#7-temperature-top-p-top-k)
8. [Prompting Basics](#8-prompting-basics)
9. [Fine-tuning vs RAG vs Prompting](#9-fine-tuning-vs-rag-vs-prompting)
10. [How Models Are Trained](#10-how-models-are-trained)
11. [Model Families & When to Use What](#11-model-families--when-to-use-what)
12. [Bonus: MoE & Running LLMs Locally](#12-bonus-moe--running-llms-locally)

---

## 1. What is an LLM?

An LLM is a **deep learning model** trained on massive text. Its one core skill: **next-token prediction** — given some text, predict the next word. Everything else (chatting, coding, summarizing) is a side effect of being really good at that one thing.

```
"The cat sat on" → [LLM, 175B params] → P("the") = 0.72 → "the"
                                          Append, repeat.
```

### Traditional Software vs LLM

| Traditional Software | LLM |
|---|---|
| You write explicit rules: `if order > $100: free_shipping()` | No rules written. You give it text, it gives text back. |
| Does exactly what you coded. Nothing more. | Learned to summarize, code, translate — all from patterns in data. |
| Deterministic | **Statistical model** — it's all probabilities |

### What Makes It "Large"?

| Dimension | What It Means | Example |
|---|---|---|
| **Large Parameters** | Weights inside the model stored in **weight matrices** | GPT-3 = 175B parameters |
| **Large Training Data** | Trillions of tokens. Called the **pre-training corpus** | Books, web crawls, code, Wikipedia |
| **Large Compute** | Thousands of GPUs for weeks. Measured in **FLOPs** | GPT-4 training cost: tens of millions $ |

**Scaling Laws** (Kaplan et al., OpenAI): If you increase parameters + data + compute together, performance improves *predictably*. That's why everyone kept building bigger.

### Why LLMs Matter Now

Before LLMs, NLP needed separate models for every task — one for sentiment, one for translation, one for NER. Each with its own **task-specific dataset** and pipeline.

Three things came together:
- **Transformer architecture** (2017, Google's "Attention Is All You Need" paper)
- Enough **GPU/TPU compute** to train at scale
- Enough **internet data** to learn from

Now one **foundation model** does many tasks. You just change the **prompt**. This is **in-context learning** — the model figures out what you want from your instructions, no retraining needed.

> **Interview one-liner:** "An LLM is an **autoregressive model** that predicts the next token given all previous tokens, trained on massive text data using the **Transformer architecture**."

---

## 2. How LLMs Work — Transformers

The **Transformer** is the architecture behind every modern LLM. Here's how text flows through it:

```
Step 1: "The cat sat" → [The] [cat] [sat] → [0.12, -0.8, ...] [0.45, 0.3, ...] [0.7, -0.2, ...]
        TEXT            → TOKENS             → EMBEDDINGS (vectors)

Step 2: + POSITIONAL ENCODING
        Add position info → "this token is at position 1, 2, 3..." (RoPE)

        ┌──────────────────────────────────────────────────────────┐
        │  TRANSFORMER BLOCK (×96 layers for GPT-3)                │
        │                                                          │
        │  Step 3: SELF-ATTENTION                                  │
        │  "Which other tokens should I pay attention to?"         │
        │           ↓                                              │
        │  Step 4: FEED-FORWARD NETWORK (FFN)                     │
        │  Process gathered context. Factual knowledge stored here.│
        │                                                          │
        │  Step 5: Stack 96 of these layers.                       │
        │  Early layers = grammar. Later layers = reasoning.       │
        └──────────────────────────────────────────────────────────┘

Step 6: LINEAR + SOFTMAX → probability distribution over vocabulary
        P("the")=0.72  P("a")=0.11  P("his")=0.08  ...
        Pick token → append → repeat (AUTOREGRESSIVE GENERATION)
```

### The Three Transformer Variants

| Variant | Architecture | Examples | Best For |
|---|---|---|---|
| **Encoder-only** | Reads all tokens at once. **Bidirectional attention**. | BERT | Classification, NER |
| **Decoder-only** | Generates one token at a time. **Causal attention**. | GPT, Llama, Claude | Text generation (almost all modern LLMs) |
| **Encoder-Decoder** | Encoder reads input, decoder generates output. | T5, original Transformer | Translation |

---

## 3. Tokens

A **token** is a chunk of text the model treats as one unit. Not always a full word — could be a word, a subword, a character, or punctuation.

```
"Tokenization is surprisingly tricky!"
→ [Token] [ization] [·is] [·surpris] [ingly] [·trick] [y] [!]
= 8 tokens
```

### Tokenization Methods

| Method | Used By | How It Works |
|---|---|---|
| **BPE (Byte Pair Encoding)** | GPT, Llama | Start with characters, merge most frequent pairs, build vocabulary (32K–100K tokens) |
| **WordPiece** | BERT | Similar to BPE but uses likelihood instead of frequency |
| **SentencePiece** | T5, multilingual models | Language-agnostic, works on raw text without pre-tokenization |

### Why Tokens Matter

| Concern | Why |
|---|---|
| **Cost** | API pricing is per token. Input + output tokens both count. |
| **Context Limits** | Prompt + response must fit within the model's max token window. |
| **Speed** | Each token = one forward pass through the model. More tokens = slower. |
| **Accuracy** | Weird subword splits hurt performance. LLMs can't count characters because they see tokens, not letters. |

> **Rule of thumb:** 1 token ≈ 4 characters in English ≈ ¾ of a word. Non-English text often uses more tokens per word.

---

## 4. Embeddings

An **embedding** is a vector (list of numbers) representing a token/word/sentence's meaning in high-dimensional space (768 to 4096 dimensions). Similar meanings = close vectors.

```
         Dimension 2
         ↑
         │    ○ refrigerator (far away)
         │
         │              ● king
         │                  ● queen      ← royalty cluster
         │
         │     ● man
         │         ● woman               ← gender cluster
         └──────────────────────→ Dimension 1

         king − man + woman ≈ queen   (vector arithmetic!)
```

### Static vs Contextual Embeddings

| Type | Examples | Behavior |
|---|---|---|
| **Static** | Word2Vec, GloVe | One vector per word forever. "bank" always has the same vector. |
| **Contextual** | Transformer-based (BERT, GPT) | Vector for "bank" changes in "river bank" vs "bank account". |

### Why Embeddings Are the Foundation of RAG

1. Embed your documents → store in a **vector database** (Pinecone, Weaviate, pgvector, ChromaDB)
2. User asks a question → embed the question
3. Find closest document vectors via **cosine similarity** or **dot product**
4. Those docs go into the prompt → LLM answers using YOUR data

**Common embedding models:** `text-embedding-ada-002` (OpenAI), Cohere Embed, `sentence-transformers` (open source). You must use the *same* model for both documents and queries.

---

## 5. Context Window

The **context window** (or **context length**) is the max number of tokens the model can see at once — input + output combined. Beyond this, tokens are simply gone.

```
┌──────────────────────────────────────────────────────────────────┐
│ System Prompt │ Retrieved Docs (RAG) │ User Message │ Output │ ← │
│               │                      │              │        │left│
└──────────────────────────────────────────────────────────────────┘
← ─ ─ ─ ─ ─ ─ Context Window (128K for GPT-4o, 200K for Claude) ─ →
```

| Model | Context Window |
|---|---|
| GPT-3 | 4K tokens |
| GPT-4o | 128K tokens |
| Claude | 200K tokens |
| Gemini | 1M+ tokens |

### Key Concepts

**Lost in the Middle** — Models pay most attention to the **beginning and end** of the context. Info buried in the middle gets less attention. Put important stuff first or last.

**Quadratic Cost** — Attention scales **O(n²)** with sequence length. Double the context = 4× the compute.

### Strategies When You Hit the Limit

| Strategy | How It Works |
|---|---|
| **Chunking** | Split documents into small pieces, retrieve only relevant chunks (this is RAG) |
| **Summarization** | Condense earlier conversation history to fit more |
| **Sliding Window** | Keep only the most recent N tokens, drop older ones |

---

## 6. Attention Mechanism

Before Transformers, **RNNs** and **LSTMs** processed text sequentially — by word 500, they'd mostly forgotten word 1. Attention fixes this by letting every token look at every other token directly.

### The Intuition

For each token, attention asks: "Which other tokens in this sequence are relevant to me right now?"

```
"The cat sat on the mat because it was tired"

When processing "it" → attention weights:
  The(0.05)  cat(0.62)  sat(0.18)  because(0.08)  it(0.07)
                ↑
         highest attention — "it" refers to "cat"
```

### Q, K, V — The Three Vectors

Every token produces three vectors:

| Vector | Role | Analogy |
|---|---|---|
| **Query (Q)** | "What am I looking for?" | You're searching |
| **Key (K)** | "What do I contain?" | Each item's label |
| **Value (V)** | "What info do I give if you pick me?" | The actual content |

**How it works:** Compute score = Q · K (dot product). High score = relevant tokens. Apply **softmax** to get weights that sum to 1. Take weighted sum of V vectors. Result: each token now carries context from the tokens it attended to.

### Multi-Head Attention

Run 32+ attention heads in parallel. Each head focuses on different things — grammar, coreference, topic. Outputs get concatenated and projected back down.

### Causal Masking

In decoder-only LLMs (GPT, Claude, Llama), each token only sees tokens **before** it. Future tokens are masked. Makes sense — during generation, you haven't produced the future tokens yet.

### Why Attention Is Expensive

Every token attends to every other → **O(n²) complexity**. Speedup techniques: **Flash Attention**, **Multi-Query Attention (MQA)**, **Grouped-Query Attention (GQA)**.

---

## 7. Temperature, Top-p, Top-k

After the model computes probabilities for every token in the vocabulary, these parameters control *how* it picks one.

### Temperature

Controls randomness. The model outputs raw scores (**logits**). Before softmax, divide by temperature.

```
Temperature = 0.2 (focused)          Temperature = 1.0 (creative)
┃████████████████│                    ┃████████│
┃██│             │                    ┃██████│
┃█│              │                    ┃█████│
┃│               │                    ┃████│
─────────────────                     ─────────────────
"the" "a" "his" "my"                 "the" "a" "his" "my"
Almost always picks "the"             More even spread, more surprise
✓ Factual Q&A, code                  ✓ Creative writing, brainstorming
```

- **Temperature = 0** → always picks highest probability token = **greedy decoding**
- **Too high (>1.5)** → incoherent random garbage

### Top-k

Only consider the top K most probable tokens. Fixed size. k=50 = pick from the 50 best. Problem: sometimes only 3 make sense, sometimes 500 do. It's rigid.

### Top-p (Nucleus Sampling)

More adaptive. Keep the smallest set of tokens whose cumulative probability hits p (e.g., 0.9). Small set when confident, larger when unsure. Usually preferred over top-k.

### Common Setup

`temperature=0.7, top_p=0.9` is a solid default. For code/facts: `temp=0`. For creative: `temp=1.0+`.

### Other Generation Parameters

| Parameter | What It Does |
|---|---|
| **Max tokens** | Hard cap on output length |
| **Stop sequences** | Strings that tell the model to stop (e.g., `</answer>`) |
| **Frequency penalty** | Penalizes tokens proportional to how many times they've appeared. Reduces repetition. |
| **Presence penalty** | Penalizes any repeated token regardless of count. Encourages new topics. |

### 4 Text Generation Strategies

| Strategy | How | Tradeoff |
|---|---|---|
| **Greedy Decoding** | Always pick top token | Fast, deterministic, repetitive |
| **Beam Search** | Track top B sequences, pick best overall | Better than greedy, expensive |
| **Sampling** | Randomly sample with temp/top-p/top-k | Most common for chat. Good diversity. |
| **Contrastive Search** | Balance probability AND embedding diversity | Reduces degeneration loops. Newer method. |

---

## 8. Prompting Basics

Prompting is how you tell an LLM what to do — no code, no retraining, just words.

### Core Techniques

| Technique | What It Is | When to Use |
|---|---|---|
| **Zero-shot** | No examples. Just the task. `"Classify as positive/negative: 'Great food!'"` | Simple tasks where the model already knows the format |
| **Few-shot** | Give 3-5 examples first. `"'Great!' → Pos, 'Terrible' → Neg, 'OK food' → ?"` Model sees the pattern. This is **in-context learning**. | When you need a specific format or behavior |
| **Chain-of-Thought (CoT)** | "Let's think step by step." Model writes out reasoning before answering. Intermediate tokens act as **scratch space**. | Math, logic, multi-step reasoning |
| **Self-consistency** | Run the same prompt multiple times, take majority vote | Reducing errors on reasoning tasks |
| **ReAct** | Model alternates between thinking and acting (e.g., web search). Foundation for **agents**. | Agentic workflows |

### Good Prompt Structure

```
Role       → Who the model should be
  ↓
Context    → Background info
  ↓
Task       → What you want done
  ↓
Format     → How the output should look
  ↓
Examples   → Few-shot examples if needed
  ↓
Constraints → What to avoid
```

---

## 9. Fine-tuning vs RAG vs Prompting

The big decision every AI engineer faces: how do you customize an LLM?

```
                Need to customize LLM?
               /          |           \
         Simple task?  Need your data?  Need new behavior?
              ↓            ↓                  ↓
          Prompting     RAG             Fine-tune
                           \               /
                            ↘             ↙
                       Often best: RAG + Fine-tune
```

### The Three Options

| Approach | What It Does | Cost | When to Use |
|---|---|---|---|
| **Prompting** | Steer behavior with words (system prompt, few-shot) | Free | Start here always. Most problems should try this first. |
| **RAG** | Retrieve relevant docs at query time, stuff into prompt | Medium | Model needs YOUR data (company docs, recent info). Knowledge changes over time. |
| **Fine-tuning** | Actually update the model's weights on your data | High | Need consistent style/tone, specific output format, distill big model to small, novel skills |

### When to Fine-tune — Real Examples

| Situation | Why Fine-tune? |
|---|---|
| Bank support bot needs to always follow greet→acknowledge→solve→confirm flow | Prompting works 90% of the time but drifts. Fine-tune on 5,000 real conversations. |
| Model must return exact JSON schema `{"diagnosis": "...", "confidence": 0.85}` every time | Prompting gives 90% reliability. Fine-tuning gives 99.9%. |
| Using GPT-4 to classify 100K tickets/day at $3,000/day | Fine-tune Llama 3 8B on GPT-4 labels (**knowledge distillation**). 95% accuracy, nearly free. |
| Medical coding assistant needs 70,000+ ICD-10 codes | Too large for RAG context. Fine-tune so the model internalizes the patterns. |
| Custom SQL dialect your company invented | Not in training data. No amount of prompting teaches novel syntax. |

### When NOT to Fine-tune

- If **prompting works** — don't bother. Fine-tuning is expensive.
- For **factual knowledge** — use RAG. Fine-tuning bakes knowledge into weights (goes stale). RAG stays up to date.
- With **fewer than 500-1000 examples** — you'll overfit.
- Risk of **catastrophic forgetting** — model loses abilities it used to have.

### Fine-tuning Methods

| Method | What It Does | Cost |
|---|---|---|
| **Full Fine-tuning** | Update every weight. Needs full model in GPU memory + gradients + optimizer states. | Very expensive. Multiple A100 GPUs for 70B. |
| **LoRA (Low-Rank Adaptation)** | Freeze original model, train small adapter matrices (~0.1% of params). Merge at inference. Can swap adapters at runtime. | Much cheaper. Most popular method now. |
| **QLoRA** | LoRA on a quantized (4-bit) model. 70B fits in ~35GB. | Cheapest. Single GPU. Slight quality tradeoff. |

### Concrete LoRA Example

Fine-tune Llama 3 8B to write emails in your CEO's voice:
1. Collect 1,000 CEO emails
2. Format as `{"prompt": "Write an email declining a partnership", "response": "<actual CEO email>"}`
3. Load Llama 3 8B in 4-bit (QLoRA)
4. Train LoRA adapters — rank 16, alpha 32, on attention layers
5. Train 3 epochs on a single A100 — ~2 hours
6. Merge adapter into base model
7. Now it sounds like your CEO

### The Mental Model

> **Prompting controls what the model does. RAG controls what the model knows. Fine-tuning controls how the model behaves.**

---

## 10. How Models Are Trained

### The Three Stages

```
Stage 1: PRE-TRAINING
  │  Train on trillions of tokens. Next-token prediction.
  │  Self-supervised learning. AdamW optimizer.
  │  Distributed training across thousands of GPUs.
  │  Result: Base model. Can complete text but can't follow instructions.
  ▼
Stage 2: SUPERVISED FINE-TUNING (SFT)
  │  Train on curated prompt-response pairs (tens of thousands).
  │  High quality, often human-written.
  │  Result: Instruction-tuned / chat model.
  ▼
Stage 3: RLHF
  │  Humans rank model outputs →
  │  Train a reward model on those rankings →
  │  Use PPO to maximize reward.
  │  Result: More helpful, harmless, honest model.
  ▼
DEPLOYED MODEL
```

### Stage 1: Pre-training (Details)

- Task: **next-token prediction** — this is **self-supervised learning** (the next word IS the label, no human labeling needed)
- Optimizer: **AdamW** with **learning rate schedule** (warmup then decay)
- **Distributed training** strategies: **data parallelism** (each GPU processes different data), **tensor parallelism** (split layers across GPUs), **pipeline parallelism** (split layers sequentially across GPUs)
- Output: a **base model** / **foundation model** — can complete text but doesn't follow instructions. Like a student who's read everything but doesn't know how to answer questions.

### Stage 2: SFT (Supervised Fine-Tuning)

- Also called **instruction tuning**
- Train on curated **prompt-response pairs**: "user asked X, ideal response is Y"
- Much smaller dataset (tens of thousands) but very high quality, often **human-written**
- After SFT, the model can follow instructions and have conversations

### Stage 3: RLHF (Reinforcement Learning from Human Feedback)

1. **Collect preferences** — show humans two outputs for the same prompt, they pick the better one
2. **Train a reward model** — learns to score outputs the way humans would
3. **RL optimization** — use **PPO (Proximal Policy Optimization)** to fine-tune the LLM to maximize the reward

### Alternatives to RLHF

| Method | How It Works |
|---|---|
| **DPO (Direct Preference Optimization)** | Skips the reward model entirely. Trains directly on preference pairs. Simpler, cheaper, increasingly popular. |
| **RLAIF (RL from AI Feedback)** | Another LLM does the judging. Anthropic's **Constitutional AI** — model critiques its own outputs based on principles. |
| **Knowledge Distillation** | Small **student** model mimics large **teacher** model's output distributions. |

### Full Build Pipeline (from scratch)

```
Collect Data → Clean (dedup, filter) → Build Tokenizer (BPE) → Choose Architecture (decoder-only)
    → Pre-train (next-token, thousands of GPUs) → SFT → RLHF/DPO → Evaluate
```

### Common Benchmarks

| Benchmark | What It Tests |
|---|---|
| **MMLU** | General knowledge across 57 subjects |
| **HumanEval** | Code generation (Python) |
| **HellaSwag** | Commonsense reasoning |
| **TruthfulQA** | Factual accuracy, resistance to misconceptions |

---

## 11. Model Families & When to Use What

### Closed-Source (API-based)

| Model | Key Strengths | Best For |
|---|---|---|
| **GPT-4o** (OpenAI) | Strong coding, reasoning, multimodal (text+image+audio). Huge ecosystem. | General-purpose, tool use |
| **Claude** (Anthropic) | 200K context, strong reasoning, follows nuanced instructions, less hallucination. | Long documents, analysis, writing |
| **Gemini** (Google) | Natively multimodal (text, image, video, audio). 1M+ context. Google integration. | Multimodal tasks, long context |

### Open-Source

| Model | Key Strengths | Best For |
|---|---|---|
| **Llama 3** (Meta) | 8B/70B/405B sizes. Fully customizable, self-hosted. Huge community. | Privacy, fine-tuning, on-prem |
| **Mistral / Mixtral** (Mistral AI) | MoE architecture. Near-70B quality at fraction of compute cost. | Efficiency, cost-sensitive deployments |

### Quick Decision Guide

| Need | Go With |
|---|---|
| Best quality, don't mind paying | GPT-4o or Claude |
| Long documents to analyze | Claude (200K) or Gemini (1M+) |
| Images / video / audio | Gemini or GPT-4o |
| Self-host / data privacy | Llama 3 or Mistral (open source) |
| Cheap & fast for simple tasks | GPT-4o-mini, Claude Haiku, Mistral 7B |
| Fine-tune on your own data | Open source (Llama, Mistral) |

---

## 12. Bonus: MoE & Running LLMs Locally

### Mixture of Experts (MoE)

| Aspect | Standard Transformer | MoE |
|---|---|---|
| **How it works** | Every token uses ALL parameters | Multiple expert FFNs + a **router/gating network**. Top-2 experts activated per token. |
| **Example** | 70B model → all 70B active per token | Mixtral 8×7B = 46.7B total, only ~13B active per token |
| **Disk size** | Smaller | Bigger (all experts stored) |
| **Inference speed** | Slower | Faster (fewer computations per token) |
| **Training trick** | Standard | Needs **load balancing losses** so all experts get used evenly |

### 4 Ways to Run LLMs Locally

| Tool | What It Is | Best For |
|---|---|---|
| **Ollama** | Easiest. `ollama run llama3`. Handles download, quantize, serve. | Quick local experimentation |
| **llama.cpp** | C/C++ inference engine. Runs on CPU, no GPU needed. Uses **GGUF** format. | Optimized CPU inference |
| **vLLM** | High-throughput serving engine. **PagedAttention** for GPU memory. | Serving many concurrent requests |
| **HuggingFace Transformers** | Python library. Most flexible. Supports **quantization** (4-bit, 8-bit via bitsandbytes). | Experimentation, research |

### Key Concept: Quantization

**Quantization** = reducing weight precision from 32-bit float to 8-bit or 4-bit integers. Models get ~4× smaller with minimal quality loss. This is how you run a 70B model on consumer hardware.

---

## Quick Reference — Key Terms Cheatsheet

| Term | One-Line Definition |
|---|---|
| **Autoregressive** | Generate one token at a time, using all previous tokens as context |
| **Foundation Model** | A large pre-trained model that can be adapted to many tasks |
| **In-context Learning** | Model learns task from prompt examples, no weight updates |
| **Tokenizer / BPE** | Converts text to token IDs. BPE merges frequent character pairs. |
| **Embedding** | Vector representation of a token's meaning |
| **Context Window** | Max tokens the model can see at once (input + output) |
| **Self-Attention** | Mechanism where each token attends to all other tokens via Q/K/V |
| **Multi-Head Attention** | Multiple attention heads in parallel, each learning different patterns |
| **Causal Masking** | Each token only sees tokens before it (decoder-only models) |
| **Temperature** | Controls randomness. Low = focused. High = creative. |
| **Top-p (Nucleus)** | Sample from smallest set of tokens covering p% probability mass |
| **Chain-of-Thought** | "Think step by step" — model writes reasoning before answering |
| **RAG** | Retrieve docs → stuff in prompt → model answers from your data |
| **LoRA** | Fine-tune small adapter matrices instead of full model weights |
| **QLoRA** | LoRA on a 4-bit quantized model. Cheapest fine-tuning method. |
| **SFT** | Supervised Fine-Tuning on prompt-response pairs |
| **RLHF** | Reinforcement Learning from Human Feedback via reward model + PPO |
| **DPO** | Direct Preference Optimization — skips reward model, trains on pairs directly |
| **Knowledge Distillation** | Small student model mimics large teacher model's outputs |
| **MoE** | Mixture of Experts — route tokens to subset of expert FFNs |
| **Quantization** | Reduce weight precision (32-bit → 4-bit) to shrink model size |
| **Flash Attention** | Optimized attention implementation that reduces memory usage |
| **Scaling Laws** | Performance improves predictably with more params + data + compute |

---

## Cross-Pattern Reference

| Pattern | Where It Shows Up |
|---|---|
| Embeddings + Vector DB + Cosine Similarity | RAG pipeline |
| Temperature + Top-p + Stop Sequences | API calls, inference config |
| LoRA + QLoRA + Quantization | Fine-tuning pipeline |
| Pre-training + SFT + RLHF/DPO | Model training pipeline |
| BPE + Context Window + Token Count | Cost estimation, prompt design |
| Attention + Causal Masking + Multi-Head | Transformer internals |
| Foundation Model + In-context Learning + Few-shot | Prompting strategy |
| Knowledge Distillation + Synthetic Data + RLAIF | Training with another LLM |

---

*Next up: [02 — RAG](../02-rag/README.md)*
