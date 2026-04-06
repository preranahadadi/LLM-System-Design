# Module 04a -- Prompt Fundamentals
> Prompt Engineering Series · What a prompt is, anatomy, roles, how the model reads a prompt, token budget

---

## Table of Contents
1. [What is a Prompt](#1-what-is-a-prompt)
2. [Anatomy of a Prompt](#2-anatomy-of-a-prompt)
3. [The Three Roles](#3-the-three-roles)
4. [How the Model Reads a Prompt](#4-how-the-model-reads-a-prompt)
5. [Token Budget](#5-token-budget)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. What is a Prompt

A prompt is everything you send to the LLM before it generates a response. The model has no memory, no state, and no context beyond what is in the current prompt. Every call to the model starts fresh.

This is the foundational mental model for prompt engineering: the model only knows what you tell it right now, in this call.

Everything the model needs to know must be in the prompt:
- What its job is
- What rules it must follow
- Who the user is
- What the conversation history was
- What documents or data it should use
- What format the output should be in

If something is not in the prompt, the model does not know it. It will either guess, use patterns from its training data, or produce something unexpected.

---

## 2. Anatomy of a Prompt

Most LLM APIs structure prompts as a list of messages. Each message has a role and content. The model receives the full list and generates the next message.

A typical prompt structure looks like this:

```
System message    --> standing instructions, persona, rules, context
User message      --> the human's current input
Assistant message --> the model's previous response (in multi-turn conversations)
User message      --> the next human input
```

### What Goes Where

System message: everything about how the model should behave. Its persona, the rules it must follow, background context, output format requirements. This is written by the developer, not the user.

User message: the actual input from the human. A question, a document to process, a task to complete.

Assistant message: in multi-turn conversations, you include the model's previous responses so it has context for the next turn. The model does not remember previous API calls on its own. You reconstruct the conversation history by including previous messages in every new request.

### A Concrete Multi-Turn Example

```python
messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant for Acme Corp employees. Answer questions concisely using plain English."
    },
    {
        "role": "user",
        "content": "What is the vacation policy?"
    },
    {
        "role": "assistant",
        "content": "Employees receive 15 days of paid vacation per year, accruing at 1.25 days per month."
    },
    {
        "role": "user",
        "content": "Can I carry days over to next year?"
    }
]
```

The model sees the entire list. It knows the conversation that happened before the current question.

---

## 3. The Three Roles

### System

The system message is your primary tool for controlling model behavior. It is processed first and establishes the baseline for everything that follows.

Use the system message for:
- Role or persona definition
- Standing rules the model must always follow
- Output format requirements
- Background context that applies to every response
- Grounding instructions (for RAG systems)

```
You are a customer support assistant for a software company.
Answer only questions related to our products.
Keep responses under 150 words.
If you do not know the answer, say so clearly and suggest the user contact support@company.com.
```

### User

The user message is what the person typed or what your application is sending as the task. In agentic systems, tool results are also often formatted as user messages.

Keep user messages clean. Do not mix instructions and user input in the same message if you can help it. Instructions belong in the system message.

### Assistant

Including previous assistant messages in multi-turn conversations is how you give the model memory of what it said before. Without this, every response would be generated without awareness of previous turns.

One practical detail: if you want the model to continue from a partial response you started, you can include an incomplete assistant message at the end of the message list. The model will continue from where you left off. This is called assistant prefill and it is useful for steering the model toward a specific output format.

```python
messages = [
    {"role": "system",    "content": "Return only JSON."},
    {"role": "user",      "content": "Extract the name and age from: John Smith, 34 years old"},
    {"role": "assistant", "content": "{"}   # model continues from here
]
```

---

## 4. How the Model Reads a Prompt

The model reads the entire prompt as a single sequence of tokens from left to right. There is no special processing of system vs user vs assistant messages under the hood. They are all just text with role markers prepended.

The model generates each output token based on all the tokens that came before it, including everything in the prompt.

Two important consequences:

### Position Matters

Research consistently shows that LLMs pay more attention to content near the beginning and end of a prompt than to content in the middle. This effect gets stronger as the prompt gets longer.

Practical implications:
- Put your most critical instructions at the beginning of the system message
- Put the most relevant context close to the user query (near the end)
- If you have a critical rule, consider repeating it near the end of the prompt
- For RAG systems, the retrieved chunks closest to the user query get more attention

### Instruction Count Matters

The model has a finite amount of attention to distribute across the prompt. A system message with 20 specific rules dilutes each rule. The model cannot follow all 20 perfectly.

Prioritize ruthlessly. Five important, clearly stated rules will be followed more reliably than twenty rules of mixed importance. If a rule is truly critical, it should be prominent, specific, and possibly repeated.

---

## 5. Token Budget

Every model has a maximum context window, measured in tokens. One token is roughly 0.75 words in English, or about 4 characters. Different languages tokenize differently -- Chinese and Japanese typically use more tokens per character than English.

### Current Context Windows

| Model | Context Window |
|---|---|
| GPT-4o | 128,000 tokens |
| Claude 3.5 Sonnet | 200,000 tokens |
| Gemini 1.5 Pro | 1,000,000 tokens |
| Llama 3.1 70B | 128,000 tokens |

Your entire prompt plus the model's response must fit within this limit. For most simple tasks this is not a constraint. It becomes important for:
- Long document processing
- Multi-turn conversations with many previous turns
- Agentic tasks that accumulate many tool results
- RAG prompts with many large retrieved chunks

### Cost Implications

You pay per token in most API-based models. Input tokens are typically cheaper than output tokens, but both cost money. A long system prompt that is repeated on every single request adds up at scale.

For high-volume applications, measure the average token count per request and calculate the monthly cost before going to production.

### Counting Tokens

```python
import tiktoken

def count_tokens(text, model="gpt-4o"):
    encoder = tiktoken.encoding_for_model(model)
    return len(encoder.encode(text))

prompt_tokens = count_tokens(your_system_prompt + your_user_message)
print(f"Prompt uses {prompt_tokens} tokens")
```

---

## 6. Key Takeaways

- A prompt is everything the model sees in a single API call. It has no memory between calls. Everything it needs to know must be in the prompt.
- Prompts are structured as messages with three roles: system for instructions and rules, user for the human's input, assistant for previous model responses in multi-turn conversations.
- Position matters. Content at the beginning and end of a prompt gets more attention than content in the middle. Put critical instructions at the start and repeat the most important ones near the end.
- Fewer, clearer instructions are followed more reliably than many instructions of mixed importance.
- Context windows are large but not unlimited. Monitor token counts for long-running applications. Cost scales with tokens at high volume.

---
