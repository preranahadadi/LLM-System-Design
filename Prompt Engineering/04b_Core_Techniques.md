# Module 04b -- Core Prompting Techniques
> Prompt Engineering Series · Zero-shot, few-shot, chain-of-thought, positive instructions, clarity

---

## Table of Contents
1. [Zero-Shot Prompting](#1-zero-shot-prompting)
2. [Few-Shot Prompting](#2-few-shot-prompting)
3. [Chain-of-Thought Prompting](#3-chain-of-thought-prompting)
4. [Positive vs Negative Instructions](#4-positive-vs-negative-instructions)
5. [Instruction Clarity](#5-instruction-clarity)
6. [Combining Techniques](#6-combining-techniques)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Zero-Shot Prompting

Zero-shot means giving the model a task with no examples. You describe what you want and the model figures out how to do it from its training.

```
Classify the sentiment of this product review as Positive, Negative, or Neutral.

Review: "The product arrived on time but the packaging was damaged."

Sentiment:
```

### When Zero-Shot Works

Zero-shot works well when:
- The task is common and well-understood (translation, summarization, classification of everyday text)
- The model has seen many examples of this task type during training
- The output format is obvious or does not matter much
- You want a quick baseline before investing in more complex prompting

### When Zero-Shot Fails

Zero-shot tends to fail when:
- The task is ambiguous and different people would interpret it differently
- You need a very specific output format the model might not guess correctly
- The task is domain-specific or uses terminology the model may not be familiar with
- The decision boundary is subtle (for example, distinguishing sarcasm from genuine praise)

When zero-shot fails, add examples before adding more instructions. Examples are usually more effective than longer descriptions.

---

## 2. Few-Shot Prompting

Few-shot means including two to five input-output examples in your prompt before the actual task. The model learns the pattern, format, and decision boundaries from those examples.

```
Classify the sentiment. Output exactly one word: Positive, Negative, or Neutral.

Review: "Absolutely loved it, will buy again."
Sentiment: Positive

Review: "Terrible quality, broke after one day."
Sentiment: Negative

Review: "It is okay, nothing special."
Sentiment: Neutral

Review: "The product arrived on time but the packaging was damaged."
Sentiment:
```

### Why Few-Shot Works

The examples show the model:
- Exactly what the input looks like
- Exactly what the output looks like
- The format to use (one word, not a sentence)
- The tone to use
- Where the decision boundaries are

Without examples, the model must infer all of this from your description. With examples, it just matches the pattern.

### How Many Examples to Use

Two to five examples is the typical sweet spot for most tasks. More examples cost more tokens and the improvement becomes marginal after about five. The quality of examples matters much more than the quantity.

For tasks with multiple output classes or edge cases, include at least one example per class. If you only show positive examples, the model tends to over-predict positive.

### Choosing Good Examples

The examples you choose significantly affect performance.

Cover different cases. If your task has three sentiment classes, include at least one example of each. Do not use three examples that all have the same label.

Choose examples similar to what you expect in production. If your real inputs are short social media posts, do not use long formal review paragraphs as examples.

Use examples with unambiguous expected outputs. If an example is something a human would disagree on, the model will also be uncertain about it. Save the ambiguous cases for the actual task.

Use your hardest cases as examples. If certain input patterns cause the model to fail in zero-shot, include an example of that pattern in your few-shot examples.

### Dynamic Few-Shot Selection

For best results in production, do not use the same static examples for every query. Select examples dynamically based on similarity to the current input.

```python
def select_few_shot_examples(current_input, example_pool, top_k=3):
    query_embedding = embed(current_input)
    example_embeddings = [embed(ex["input"]) for ex in example_pool]

    similarities = [
        cosine_similarity(query_embedding, ex_emb)
        for ex_emb in example_embeddings
    ]

    top_indices = sorted(range(len(similarities)), key=lambda i: similarities[i], reverse=True)[:top_k]
    return [example_pool[i] for i in top_indices]
```

This retrieves the most similar examples from your pool for each new input, which works much better than fixed examples for tasks with varied input types.

---

## 3. Chain-of-Thought Prompting

Chain-of-thought prompting gets the model to show its reasoning step by step before producing the final answer. This substantially improves accuracy on tasks that require multiple reasoning steps.

The core insight is that when the model writes out intermediate reasoning steps, it has to commit to each step before moving to the next. This reduces the chance of jumping to a wrong conclusion.

### Zero-Shot Chain-of-Thought

Add "think step by step" or "let us think through this carefully" to your prompt. This alone is often enough to trigger useful reasoning behavior.

```
A store sells apples for 1.20 each and oranges for 0.80 each.
If you buy 5 apples and 3 oranges, what is the total cost?
Think step by step.
```

The model will now show its work:

```
5 apples at 1.20 each: 5 x 1.20 = 6.00
3 oranges at 0.80 each: 3 x 0.80 = 2.40
Total: 6.00 + 2.40 = 8.40
The total cost is 8.40.
```

Without chain-of-thought, the model often pattern-matches to a wrong number. With it, the explicit calculation forces the correct path.

### Few-Shot Chain-of-Thought

Include examples that show the reasoning process, not just the final answer. The model learns to reproduce the same reasoning structure.

```
Q: A store sells apples for 1.20 each and oranges for 0.80 each.
   If you buy 5 apples and 3 oranges, what is the total?

A: 5 apples at 1.20 each is 5 times 1.20, which is 6.00.
   3 oranges at 0.80 each is 3 times 0.80, which is 2.40.
   The total is 6.00 plus 2.40, which is 8.40.

Q: A shop sells pens for 0.50 each and notebooks for 3.00 each.
   If you buy 8 pens and 2 notebooks, what is the total?

A:
```

The model follows the same structure: compute each component, then sum.

### When Chain-of-Thought Helps

Chain-of-thought is most valuable for:
- Math problems and numerical reasoning
- Logic puzzles and deductive reasoning
- Multi-step analysis where each step depends on the previous
- Tasks where the model tends to jump to wrong conclusions
- Complex code generation where the approach needs to be thought through first

Chain-of-thought does not help much for:
- Simple factual lookups
- Single-step tasks
- Tasks where the model already performs reliably without it
- Very short outputs where there is no room for reasoning

### Extracting the Final Answer

When using chain-of-thought, the model's output includes both reasoning and the final answer. If you need to parse just the answer, prompt the model to put the final answer in a specific format.

```
Think step by step. After your reasoning, state the final answer on a new line
starting with "Answer: "
```

---

## 4. Positive vs Negative Instructions

Tell the model what to do, not what to avoid. Negative instructions are consistently less reliable than positive ones.

### Why Negative Instructions Are Weaker

To follow a negative instruction, the model must:
1. Understand what is forbidden
2. Generate a response
3. Check the response against what is forbidden
4. Inhibit that behavior

This is harder than simply activating the behavior you want directly. The model also tends to focus on the forbidden thing, which can paradoxically increase its presence.

### Examples

```
Weak negative:   "Do not use technical jargon."
Better positive: "Use plain English that a non-technical manager can understand."

Weak negative:   "Do not write long responses."
Better positive: "Keep your response to three sentences or fewer."

Weak negative:   "Do not make up information you are not sure about."
Better positive: "Answer only using the provided context. If the answer is not
                  in the context, say: I do not have information on this."

Weak negative:   "Do not be vague."
Better positive: "Be specific. Include numbers, dates, and names where relevant."
```

In each case, the positive version tells the model exactly what to produce. The negative version only tells it what to avoid, leaving room for many ways to still be wrong.

### When Negative Instructions Are Acceptable

Negative instructions are useful when the prohibited behavior is very specific and easy to check, and when a positive alternative is not obvious.

```
Do not include the user's password in any response, even if they ask you to repeat it.
```

This is a specific, checkable prohibition where the behavior to avoid is clear. Use this form for hard safety rules where the boundary is unambiguous.

---

## 5. Instruction Clarity

Vague instructions produce vague results. The model interprets ambiguous instructions in ways that may not match what you intended.

A useful test: if three different people read this instruction independently, would they produce similar outputs? If not, the instruction is ambiguous and needs to be more specific.

### Making Instructions Specific

Specify format explicitly:

```
Vague:    "Summarize this article."
Specific: "Summarize this article in exactly three bullet points.
           Each bullet should be one sentence.
           Focus on: the main finding, the method used, and the practical implication."
```

Specify length:

```
Vague:    "Write a short response."
Specific: "Write a response of 50 to 100 words."
```

Specify the audience:

```
Vague:    "Explain this concept."
Specific: "Explain this concept as if the reader is a first-year university student
           who has never studied programming."
```

Specify what to do with edge cases:

```
Vague:    "Extract the customer name from the email."
Specific: "Extract the customer name from the email. If no name is present,
           return null. If multiple names are present, return the first one."
```

### One Instruction Per Rule

Do not stack multiple requirements into a single sentence.

```
Unclear: "Summarize the document briefly and accurately in a professional tone
          using bullet points without any technical jargon."

Clearer:
  Format: Use bullet points.
  Length: Three to five bullets, one sentence each.
  Tone: Professional, written for a non-technical audience.
  Content: Accurate summary of the key points. Do not add information not in the document.
```

Separating instructions makes each one easier for the model to follow and easier for you to debug when something goes wrong.

---

## 6. Combining Techniques

These techniques work together. A production prompt will typically combine several of them.

A strong prompt for a classification task might use:
- A specific system message (what the model is, what rules apply)
- Positive instructions (what to output, not what to avoid)
- An explicit output format specification
- Few-shot examples covering each class
- Chain-of-thought if the task requires reasoning before classifying

```
You are a customer support triage assistant for a software company.

Your job is to classify incoming support tickets into one of these categories:
  billing -- questions about invoices, payments, subscriptions, or refunds
  technical -- questions about bugs, errors, installation, or API issues
  account -- questions about login, passwords, user settings, or access
  other -- anything that does not fit the above categories

Output the category name only, with no other text.

Examples:

Ticket: "I was charged twice for my subscription this month."
Category: billing

Ticket: "My API key stopped working after your update yesterday."
Category: technical

Ticket: "I need to transfer my account to a different email address."
Category: account

Ticket: "Can you recommend a good tutorial for getting started?"
Category: other

Now classify this ticket:

Ticket: "{ticket_text}"
Category:
```

This combines a clear role definition, specific positive instructions, explicit output format, and four-shot examples covering each class.

---

## 7. Key Takeaways

- Zero-shot works for common, well-defined tasks. Add examples when it fails before adding more instructions.
- Few-shot examples teach format, tone, and decision boundaries more effectively than longer descriptions. Two to five high-quality examples outperform ten mediocre ones. Cover all output classes.
- Chain-of-thought improves accuracy on multi-step reasoning by having the model show its work before answering. Add "think step by step" as the simplest way to trigger it.
- Positive instructions are more reliable than negative ones. Tell the model what to produce, not what to avoid.
- Specific instructions produce specific outputs. Define format, length, audience, and edge case handling explicitly.

---

