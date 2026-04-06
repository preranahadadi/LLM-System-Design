# Module 03a -- What is an Agent?
> Agents and Orchestration Series · LLM call vs chain vs agent, ReAct loop, when to use agents

---

## Table of Contents
1. [LLM Call vs Chain vs Agent](#1-llm-call-vs-chain-vs-agent)
2. [The ReAct Loop](#2-the-react-loop)
3. [When Do You Need an Agent](#3-when-do-you-need-an-agent)
4. [Key Takeaways](#4-key-takeaways)

---

## 1. LLM Call vs Chain vs Agent

These three are commonly confused. The difference is how much control the LLM has over what happens next.

### Single LLM Call
One input goes in, one output comes out. That is the entire interaction.

```
User query --> LLM --> Answer
```

No decisions are made by the LLM about what to do next. You ask, it answers, it is done. Use this for simple question answering, summarization, or classification where you know exactly what you need.

### Chain
A fixed sequence of LLM calls where the developer decides the steps in advance.

```
Query --> LLM step 1 (summarize) --> LLM step 2 (translate) --> LLM step 3 (format) --> Output
```

The path is hardcoded before any query arrives. The LLM executes each step but does not decide the steps. The flow is always the same regardless of input. Use this when you know exactly what transformations your data needs in a fixed order.

### Agent
The LLM decides what to do next, does it, sees what happened, and decides again. This loop continues until the task is complete.

```
Query --> LLM thinks about what to do --> takes an action --> sees the result --> LLM thinks again --> ...
```

The LLM is in control of the execution path. The number of steps is not known in advance. It depends on what the agent discovers along the way. An agent might take 2 steps or 20 steps depending on the task.

The key difference from a chain: in a chain the developer writes the steps. In an agent the LLM writes the steps at runtime.

---

## 2. The ReAct Loop

ReAct stands for Reasoning and Acting. It is the core pattern behind almost every agent.

The agent alternates between two things: thinking about what to do next, and actually doing it. The output of each action becomes the input for the next round of thinking.

```
Thought:      Reason about the current situation and decide what to do
Action:       Call a tool with specific arguments
Observation:  See what the tool returned
Thought:      Reason about the result and decide what to do next
Action:       Call another tool or decide you have enough to answer
...
Final Answer: Return the answer to the user
```

### Worked Example

```
User: "What is the current stock price of Apple and how does it compare to last week?"

Thought:      I need the current Apple stock price. I will use the stock price tool.
Action:       get_stock_price(ticker="AAPL")
Observation:  213.45 as of today

Thought:      Now I need last week's price to make the comparison.
Action:       get_stock_price(ticker="AAPL", date="7 days ago")
Observation:  198.20

Thought:      I have both numbers. Apple is up 15.25 dollars which is about 7.7 percent.
              I have enough to answer now.
Final Answer: Apple is currently trading at 213.45, up 7.7 percent from 198.20 last week.
```

Notice that the agent decided on its own to make two tool calls. A chain would have needed this hardcoded. The agent figured it out from the question.

### Why This Works

The Thought step gives the LLM space to reason before acting. Without it, the LLM jumps to an action without thinking through whether it is the right one. The Observation step grounds the LLM in reality -- it cannot hallucinate what a tool returned because the actual return value is right there in the context.

---

## 3. When Do You Need an Agent

Agents add latency, cost, and complexity. Do not use them by default.

A good rule of thumb: if you can draw the steps on a whiteboard before seeing any actual query, use a chain. If the steps depend on what you discover while executing, use an agent.

| Situation | What to use |
|---|---|
| Steps are fixed and predictable | Chain |
| Single knowledge lookup from your docs | RAG |
| Real-time data is needed (weather, stock prices) | Agent with tools |
| The number of steps is unknown upfront | Agent |
| Each step depends on the result of the previous step | Agent |
| You need to interact with external systems | Agent |
| Multi-step reasoning across different data sources | Agent |

### When Agents Are Overkill

- Answering a question from a knowledge base -- just use RAG
- Summarizing a document -- single LLM call
- Extracting structured data from text -- single LLM call with a structured output prompt
- A fixed three-step data transformation -- chain

### The Cost of Agents

Every iteration of the ReAct loop costs an LLM call. A task that takes 10 iterations costs 10 LLM calls. At GPT-4o pricing that can add up fast. Always set a maximum iteration limit and a cost ceiling on any agent you deploy.

---

## 4. Key Takeaways

- A single LLM call is one input one output. A chain is a fixed sequence of calls. An agent is an LLM in a loop that decides its own steps.
- ReAct is the core agent pattern: Thought, Action, Observation, repeat until done.
- The Thought step is what separates good agents from bad ones. It gives the LLM space to reason before acting.
- Use agents only when the steps cannot be determined in advance. For everything else, chains and single calls are simpler and cheaper.
- Always set an iteration limit. Agents can loop indefinitely without one.


