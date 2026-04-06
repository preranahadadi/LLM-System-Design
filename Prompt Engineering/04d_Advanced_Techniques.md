# Module 04d -- Advanced Techniques
> Prompt Engineering Series · Role prompting, self-consistency, step-back prompting, least-to-most, meta-prompting

---

## Table of Contents
1. [Role Prompting](#1-role-prompting)
2. [Self-Consistency](#2-self-consistency)
3. [Step-Back Prompting](#3-step-back-prompting)
4. [Least-to-Most Prompting](#4-least-to-most-prompting)
5. [Meta-Prompting](#5-meta-prompting)
6. [When to Use Each](#6-when-to-use-each)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Role Prompting

Role prompting means assigning the model a specific expert persona before giving it a task. The model draws on patterns from its training associated with that role, which influences the depth, style, and perspective of its response.

### Basic Form

```
You are a senior software engineer with 15 years of experience reviewing
production Python code for a financial services company. You are known for
catching subtle edge cases and security vulnerabilities that other reviewers miss.

Review the following function and identify any issues:

{code}
```

### Why It Works

The model has seen a large volume of text written by and about people in various roles. Assigning a role activates patterns associated with that role: the vocabulary they use, the concerns they prioritize, the level of depth they go into.

A general "review this code" prompt produces a generic response. A prompt that assigns a senior security-focused engineer persona produces output that is more likely to raise security concerns, mention edge cases, and use the vocabulary of experienced engineers.

### Specificity Matters

Vague roles are less effective than specific ones.

```
Less effective: "You are an expert."

More effective: "You are a board-certified cardiologist with subspecialty
                 training in interventional cardiology and 20 years of
                 clinical experience at a major academic medical center."
```

The more specific the role, the more precisely the model can match the patterns associated with it.

### Combining Role with Context

Role prompting works best when combined with specific context about the situation.

```
You are a product manager at a B2B SaaS company. Your team is evaluating
whether to build a new feature. You approach decisions by first understanding
user value, then estimating engineering effort, then assessing strategic fit.

Evaluate this feature request:
{feature_request}
```

The role tells the model who it is. The context tells it what the situation is. Together they produce much more targeted output than either alone.

---

## 2. Self-Consistency

Self-consistency means generating multiple independent responses to the same question and taking the majority answer. It trades cost for accuracy.

### How It Works

```python
from collections import Counter

def self_consistent_answer(prompt: str, llm, num_samples: int = 5) -> str:
    answers = []

    for _ in range(num_samples):
        response = llm.invoke(prompt, temperature=0.7)
        answer = extract_final_answer(response)
        answers.append(answer)

    # Take the most common answer
    most_common = Counter(answers).most_common(1)[0][0]
    return most_common
```

Use a non-zero temperature so the samples are not all identical. Temperature 0.5 to 0.7 gives enough variation to be useful without too much noise.

### Why It Works

On tasks where the model is uncertain, different runs through the reasoning process may arrive at different answers. If the model reaches the correct answer more often than not across independent runs, the majority vote will be correct even if individual runs sometimes fail.

This is most effective when:
- The task has a single correct answer that can be extracted and compared
- The model is inconsistent on this type of question (success rate around 60 to 80 percent per run)
- Accuracy matters more than cost or latency

### When It Does Not Help

If the model gets the wrong answer on every run (below 50 percent accuracy per run), majority vote will consistently pick the wrong answer. Self-consistency amplifies the most common response, not necessarily the correct one.

For tasks where all runs give the same answer, self-consistency adds cost with no benefit.

For creative tasks or tasks with many valid answers, majority vote does not make sense because there is no single correct answer to vote on.

---

## 3. Step-Back Prompting

Step-back prompting adds an intermediate step before answering a specific question. The model first identifies the underlying principle or general concept that applies, then uses that principle to answer the specific question.

### The Pattern

```
Step 1 -- Ask the model to step back and identify the general principle:
"What are the key principles for designing systems that need to handle
 high write throughput with eventual consistency?"

Step 2 -- Apply those principles to the specific question:
"Using those principles, design the write pipeline for a social media
 application that needs to handle 100,000 posts per minute."
```

### Why It Helps

When you ask a complex specific question directly, the model tends to jump to a specific solution pattern it has seen before, which may not be the right fit for your situation. Having the model first articulate the underlying principles grounds the specific answer in correct reasoning.

It is particularly effective for system design, architecture decisions, debugging approaches, and any task where the right approach depends on understanding fundamentals before applying them.

### Single-Prompt Version

You can do both steps in a single prompt by asking the model to first think about the general case.

```
Before answering, first identify the general principles that apply to this type of problem.
Then use those principles to answer the specific question.

Question: Design a rate limiter for an API that needs to handle bursts of up to
          10x normal traffic without dropping requests.
```

---

## 4. Least-to-Most Prompting

Least-to-most prompting breaks a complex problem into a sequence of subproblems ordered from simplest to most complex. Each subproblem is solved in order, and previous answers are included as context for later ones.

The name comes from the idea of starting with the least complex part and building up.

### The Pattern

```
Complex problem:
"Write a system that detects fraudulent transactions in real time
 for a payment processor handling 50,000 transactions per second."

Broken into subproblems, simplest first:

Step 1: "What signals distinguish fraudulent transactions from legitimate ones?"
Step 2: "Given those signals, what features would you engineer for the model?"
Step 3: "Given those features, what model type is appropriate for real-time scoring?"
Step 4: "Given that model, how would you deploy it to handle 50,000 transactions per second?"
Step 5: "Given that architecture, how would you handle model updates without downtime?"
```

Each question is much simpler than the whole problem. Each answer builds on the previous ones.

### Implementation

```python
def least_to_most(problem: str, subproblems: list[str], llm) -> list[str]:
    context = f"Main problem: {problem}\n\n"
    answers = []

    for subproblem in subproblems:
        prompt = context + f"Now answer this subproblem: {subproblem}"
        answer = llm.invoke(prompt)
        answers.append(answer)

        # Add the answer to context for the next subproblem
        context += f"Q: {subproblem}\nA: {answer}\n\n"

    return answers
```

### When to Use It

Least-to-most is most useful when:
- The problem genuinely has natural subproblems that build on each other
- Direct answers to the full problem tend to be shallow or miss important steps
- The task is a design or planning problem where earlier decisions constrain later ones
- You want to show your reasoning to a stakeholder step by step

It is overkill for problems that can be addressed in a single well-structured response.

---

## 5. Meta-Prompting

Meta-prompting means using an LLM to write or improve your prompts. Instead of manually iterating on prompt wording, you describe what you need and ask the model to generate a prompt that achieves it.

### Generating a New Prompt

```
I need a prompt for a task that extracts structured information from
customer support emails.

The information I need:
  - Customer name (or null if not provided)
  - Issue category: one of billing, technical, account, other
  - Urgency: one of low, medium, high
  - One sentence summary of the issue

The main failure I keep seeing: the model adds explanation text around the
JSON instead of returning only the JSON object.

Write me a system prompt that reliably produces a JSON object with those
fields and no extra text.
```

### Improving an Existing Prompt

```
Here is my current prompt:

[paste current prompt]

Here are examples of outputs that are wrong and why they are wrong:

Example 1:
Input: "..."
Output: "..." 
Problem: The urgency was classified as high but it should be medium.

Example 2:
Input: "..."
Output: "..."
Problem: The model included explanation text before the JSON.

Rewrite the prompt to fix these specific failures while keeping everything
that is currently working.
```

### Prompt Decomposition

For complex prompts, ask the model to analyze what each part is doing and whether it is necessary.

```
Here is a prompt I wrote:

[paste prompt]

For each section of this prompt:
1. Explain what it is trying to achieve
2. Assess whether it is clearly written
3. Identify any parts that might conflict with other parts
4. Suggest any improvements

Then rewrite the full prompt incorporating your suggestions.
```

### When Meta-Prompting Helps Most

Meta-prompting is most useful when:
- You are starting from scratch and want a strong first draft
- You have identified specific failure patterns but are not sure how to fix them in the prompt
- You want to simplify a prompt that has become too long and complex
- You are not familiar with what prompting patterns work well for a particular task type

It is less useful when you already have a clear idea of what you want to write. In that case, writing it directly is faster.

---

## 6. When to Use Each

| Technique | Best for | Cost |
|---|---|---|
| Role prompting | Domain-specific depth, expert perspective | Low (same as normal) |
| Self-consistency | Single-answer tasks where model is inconsistent | High (multiple calls) |
| Step-back prompting | Complex questions where jumping to answers fails | Low (one extra call) |
| Least-to-most | Multi-part problems that build on each other | Medium (multiple calls) |
| Meta-prompting | Writing and improving prompts themselves | Low (one-time cost) |

Always start with the simplest technique that works. Role prompting and step-back prompting add almost no cost. Self-consistency and least-to-most add significant cost. Use them only when simpler approaches are not sufficient.

---

## 7. Key Takeaways

- Role prompting activates patterns associated with a specific expert persona. Specificity matters -- a vague role is less effective than a precisely defined one.
- Self-consistency generates multiple independent answers and takes the majority vote. It improves accuracy when the model is inconsistent on a task but adds proportional cost.
- Step-back prompting adds an intermediate step where the model identifies the relevant principle before answering the specific question. Prevents jumping to the wrong solution pattern.
- Least-to-most breaks complex problems into ordered subproblems from simplest to most complex, using each answer to inform the next.
- Meta-prompting uses an LLM to write or improve prompts. Most useful when starting from scratch or when you have identified specific failure patterns to fix.

---

