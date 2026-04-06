# Module 03c -- Memory
> Agents and Orchestration Series · Short-term, long-term, episodic, procedural memory -- when to use each

---

## Table of Contents
1. [Why Agents Need Memory](#1-why-agents-need-memory)
2. [The Four Types of Memory](#2-the-four-types-of-memory)
3. [Short-term Memory](#3-short-term-memory)
4. [Long-term Memory](#4-long-term-memory)
5. [Episodic Memory](#5-episodic-memory)
6. [Procedural Memory](#6-procedural-memory)
7. [Combining Memory Types](#7-combining-memory-types)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Why Agents Need Memory

An LLM by itself has no memory. Every time you call it, it starts fresh. It has no idea what you talked about yesterday, what tools it tried before, or what the user prefers.

For a simple chatbot this is manageable by passing conversation history in each call. For an agent that runs long multi-step tasks, or one that needs to remember things across many sessions, this is a fundamental limitation.

Memory in the context of agents means any mechanism for retaining and retrieving information beyond what is currently in the context window.

---

## 2. The Four Types of Memory

| Type | What it stores | Where it lives | Survives session end |
|---|---|---|---|
| Short-term | Current conversation and task state | Context window (RAM) | No |
| Long-term | Facts about the user, preferences, persistent knowledge | Vector store or database | Yes |
| Episodic | Records of specific past task executions | Vector store | Yes |
| Procedural | How to do tasks, learned patterns | System prompt or fine-tuning | Yes |

---

## 3. Short-term Memory

Short-term memory is everything currently in the context window. The agent sees the system prompt, the conversation history so far, all tool calls and their results, and the current query.

This is the only memory that is always present and always free to access. No retrieval step needed. The LLM reads it all.

### The Context Window Problem

Agents accumulate a lot of content in the context window. A long task with many tool calls fills it up quickly. Most models have limits of 128K to 200K tokens, which sounds large but goes fast when you include:
- System prompt
- Many rounds of tool calls and results
- Long document extracts from retrieval steps
- Conversation history

When the context fills up, the oldest content gets cut off or the whole call fails.

### Managing Short-term Memory

Three strategies:

**Sliding window** -- keep only the last N turns of conversation history. Simple but you lose earlier context.

```python
def trim_to_window(messages, max_turns=10):
    system = messages[:1]         # always keep system prompt
    recent = messages[-max_turns * 2:]   # keep last N turns (each turn = 2 messages)
    return system + recent
```

**Summarization** -- compress older turns into a summary rather than dropping them entirely.

```python
def compress_history(old_messages, llm):
    summary = llm.invoke(
        "Summarize the key facts and decisions from this conversation: " +
        format_messages(old_messages)
    )
    return [{"role": "system", "content": "Earlier conversation summary: " + summary}]
```

**Selective retention** -- keep only the tool results that are still relevant to the current step. Discard intermediate results once they have been incorporated.

---

## 4. Long-term Memory

Long-term memory persists across sessions. When the same user comes back the next day, the agent can remember things about them: their preferences, their previous questions, facts they have shared.

This is implemented as a vector store that the agent queries at the start of each session.

### How It Works

When something worth remembering happens, the agent writes it to the memory store.

```python
def store_memory(user_id, content, memory_type, vector_store):
    vector_store.add({
        "user_id":   user_id,
        "content":   content,
        "type":      memory_type,
        "embedding": embed(content),
        "timestamp": now()
    })
```

At the start of each new session, retrieve relevant memories for this user.

```python
def retrieve_memories(user_id, current_query, vector_store, top_k=5):
    results = vector_store.search(
        query_embedding=embed(current_query),
        filter={"user_id": user_id},
        top_k=top_k
    )
    return results
```

Inject those memories into the system prompt so the agent knows them from the start.

```python
system_prompt = f"""
You are a helpful assistant.

What you know about this user from previous conversations:
{format_memories(retrieved_memories)}

Use this context to personalize your responses.
"""
```

### What to Store

Store facts that are stable and reusable across sessions:
- User preferences ("prefers Python over JavaScript", "wants concise answers")
- User role or context ("works in the sales team", "manages a team of 5")
- Decisions made in past sessions ("chose Option A for the project architecture")
- Corrections the user made ("do not use the term client, we call them customers")

Do not store things that are session-specific or that will become stale quickly.

### Memory Staleness

Memories can become outdated. A user who previously said "I am a junior developer" may now be senior. Handle this by:
- Timestamping all memories
- Giving the LLM instructions to prefer recent memories when there is a conflict
- Letting the user explicitly update or delete memories

---

## 5. Episodic Memory

Episodic memory stores records of specific past task executions. Not just facts, but what the agent did and what happened.

Think of it as a task diary. The agent can look back at how it handled similar tasks before and learn from them.

### Structure of an Episode

```python
episode = {
    "task":           "Write a report on Q3 earnings",
    "query":          "Summarize our Q3 earnings report and compare to Q2",
    "steps_taken": [
        "Searched internal docs for Q3 earnings report",
        "Searched internal docs for Q2 earnings report",
        "Generated comparison table",
        "Wrote executive summary"
    ],
    "tools_used":     ["search_internal_docs", "generate_table"],
    "what_worked":    "Searching internal docs first before generating anything",
    "what_failed":    "Tried to call SQL tool first but got permission error",
    "outcome":        "success",
    "duration_sec":   45,
    "timestamp":      "2026-03-15T10:00:00Z"
}
```

### Using Episodes at Runtime

When a new task comes in, retrieve similar past episodes and inject them as context.

```python
similar_episodes = episode_store.search(
    query_embedding=embed(current_task),
    top_k=3
)

system_prompt += f"""
Similar tasks you have handled before:
{format_episodes(similar_episodes)}

Learn from what worked and avoid what failed in those cases.
"""
```

This is how agents improve over time without retraining. Each execution becomes training data for future executions.

---

## 6. Procedural Memory

Procedural memory is knowledge about how to do things rather than knowledge about facts or events. It is the most stable type of memory.

Unlike short-term, long-term, and episodic memory which are retrieved dynamically, procedural memory is typically baked directly into the system prompt or into the model via fine-tuning.

### Examples

- How to structure a report (always start with an executive summary, then details)
- Which tools to try first for which query types
- The format to use when returning results to the user
- Company-specific conventions (always use the internal doc search before the web)

### Where It Lives

In the system prompt as standing instructions:
```
When answering questions about HR policies: always use search_internal_docs first,
never use web search for internal policy questions.

When presenting financial data: always include the source document and date.

When you are unsure: ask one clarifying question rather than guessing.
```

Or baked into the model weights via fine-tuning, so the behavior is always present without needing to be in the prompt.

---

## 7. Combining Memory Types

In a production agent, all four types work together.

```
New session starts:

1. PROCEDURAL memory is in the system prompt
   "Always search internal docs first. Return JSON for structured data requests."

2. LONG-TERM memory is retrieved for this user
   "User prefers Python. User is on the ML team. User asked about RAG last week."

3. EPISODIC memory is retrieved for similar past tasks
   "Last time we did a report like this, searching by department filter worked better."

4. SHORT-TERM memory accumulates as the task executes
   Tool call 1 result, Tool call 2 result, user follow-up, etc.
```

The agent uses all of them. Procedural and long-term shape its behavior before it starts. Episodic gives it hints from past experience. Short-term is what it actively works with during the task.

---

## 8. Key Takeaways

- Short-term memory is the context window. It is fast and always available but has a size limit. Manage it with sliding windows or summarization.
- Long-term memory persists across sessions. Store it in a vector database and retrieve relevant items at session start using semantic search.
- Episodic memory stores records of past task executions so agents can learn from experience without retraining.
- Procedural memory is how-to knowledge. Put it in the system prompt or bake it in via fine-tuning.
- All four types work together. Procedural and long-term set the context. Episodic provides relevant experience. Short-term is what the agent actively uses.

