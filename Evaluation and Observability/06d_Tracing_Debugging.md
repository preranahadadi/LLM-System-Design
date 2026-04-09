# Module 06d -- Tracing and Debugging
> Evaluation and Observability Series · What tracing is, spans and traces, LangSmith, debugging multi-step pipelines

---

## Table of Contents
1. [What Tracing Is](#1-what-tracing-is)
2. [Spans and Traces](#2-spans-and-traces)
3. [Why Tracing Matters for LLM Systems](#3-why-tracing-matters-for-llm-systems)
4. [LangSmith](#4-langsmith)
5. [OpenTelemetry for LLM Systems](#5-opentelemetry-for-llm-systems)
6. [Debugging a Bad Response](#6-debugging-a-bad-response)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. What Tracing Is

Tracing is a technique from distributed systems engineering that records exactly what happened when processing a specific request, step by step. A trace is a complete record of everything that occurred to handle one user request, from the moment it arrived to the moment the response was returned.

For traditional web applications, tracing records which services were called, in what order, and how long each took. For LLM applications, tracing does the same thing but captures the specific context needed to understand LLM behavior: what prompt was sent, what was retrieved, what the model responded, how many tokens were used, and what any intermediate decisions looked like.

Tracing is the difference between knowing that something went wrong and being able to see exactly what went wrong. Without tracing, when a user reports a bad response, your only option is to guess or reproduce the issue from scratch. With tracing, you look up the specific request by its ID and see every step that was taken to produce that response.

---

## 2. Spans and Traces

A trace is made up of individual spans. Each span represents one step or operation in the processing of a request.

For a typical RAG request, the spans might be:

The query rewriting step, which takes the user's message and conversation history and produces a standalone query for retrieval.

The embedding step, which converts the query into a vector.

The vector search step, which searches the vector database and returns candidate chunks.

The reranking step, which takes the candidate chunks and scores them for relevance.

The prompt assembly step, which combines the system instructions, retrieved chunks, and user query into the final prompt.

The LLM generation step, which is the call to the language model. This span captures the full prompt, the full response, the token counts, and the latency.

The groundedness check step, which verifies the response is supported by the retrieved context.

All these spans together form one trace for one request. The trace shows you the sequence, the timing of each step, and the data flowing between them.

---

## 3. Why Tracing Matters for LLM Systems

LLM systems are harder to debug than traditional software for a specific reason: the intermediate state is not obvious from the inputs and outputs alone.

If a user reports that the answer was wrong, there are many possible causes. The query rewriter might have produced a bad standalone query that led to wrong retrieval. The retrieval might have returned irrelevant chunks. The prompt might have been assembled incorrectly. The model might have ignored the retrieved context. The groundedness check might have failed to catch a hallucination.

Looking at just the input and the output does not tell you which of these happened. You need to see the intermediate state at each step. That is what tracing provides.

Tracing also makes debugging reproducible. Instead of trying to recreate the conditions of a failure from memory, you pull up the exact trace and see exactly what state existed at each step.

For multi-agent systems, tracing becomes even more critical. An agent might make 10 or 15 tool calls before producing a final answer. Without tracing, the only thing you can see is the final answer. With tracing, you see every thought the agent had, every tool it called, every result it received, and how it responded to each result. This is the only way to understand why an agent took a particular path.

---

## 4. LangSmith

LangSmith is Anthropic and LangChain's observability platform designed specifically for LLM applications. It provides automatic tracing, a web interface for viewing traces, evaluation tools, and a dataset management system.

### What LangSmith Captures

When you configure LangSmith for your application, it automatically captures traces for LangChain and LangGraph code without requiring you to instrument every function manually.

For each trace it shows you the complete prompt that was sent to the LLM, the complete response that came back, the token counts for input and output, the cost of the call, the latency, and any errors that occurred.

For multi-step pipelines and agents, it shows you the full execution tree: which steps called which other steps, in what order, and what data flowed between them.

### How You Use It in Practice

The most common workflow is: a user reports a bad response, you find the trace for that request, and you step through it to understand what went wrong.

You look at the retrieved chunks to see whether they actually contained the right information. If they did not, the problem is in retrieval -- maybe the query was embedded poorly, maybe the wrong documents are in the index, maybe the similarity threshold is too strict.

If the retrieved chunks did contain the right information, you look at the full prompt to see how those chunks were presented to the model. Was the context clearly labeled? Were the grounding instructions clear?

If the prompt looks right, you look at the raw model output to see if the error was in the generation itself or in post-processing.

This methodical step-through of the trace is the fastest path from "the answer was wrong" to "here is the specific thing that caused it."

### Using LangSmith for Evaluation

Beyond debugging individual requests, LangSmith lets you create evaluation datasets from traces. If you see a trace where something went wrong, you can save that trace to a dataset and use it as a test case going forward. This makes it easy to build a regression dataset from real production failures.

LangSmith also lets you run evaluations directly in the platform and track eval scores over time, so you can see how quality changes across deployments.

---

## 5. OpenTelemetry for LLM Systems

OpenTelemetry is the industry-standard open source framework for distributed tracing. If your organization already uses OpenTelemetry for tracing other services, you can integrate your LLM application into the same tracing infrastructure rather than using a separate tool.

The advantage of OpenTelemetry is that it works with any compatible observability backend: Jaeger, Honeycomb, Datadog, Grafana, New Relic, and many others. If your organization already has one of these, your LLM traces can appear alongside traces from your other services, giving you a unified view of your entire system.

OpenLLMetry is an open source extension to OpenTelemetry that adds LLM-specific context to traces, including prompt and response content, model names, and token counts. It supports the major LLM providers and frameworks.

The tradeoff compared to LangSmith is that OpenTelemetry requires more setup and configuration, and the LLM-specific features are less polished. LangSmith is easier to get started with and has more LLM-focused features out of the box. OpenTelemetry is better if you need to integrate with existing infrastructure or if you want to avoid vendor lock-in.

---

## 6. Debugging a Bad Response

When a user reports a bad response, the debugging process should be systematic. Following the same steps every time makes debugging faster and more reliable.

### Step 1: Find the Trace

Use the request ID from your logs to pull up the full trace for that specific request. If you did not log request IDs, use the timestamp, user ID, and the content of the user's message to find it. This is one of the strongest arguments for logging request IDs: without them, finding a specific request in a large volume of logs is painful.

### Step 2: Check What Was Retrieved

Look at the chunks that were retrieved from the knowledge base. Ask a simple question: do these chunks actually contain the information that would be needed to answer the user's question correctly?

If the answer is no, the problem is in the retrieval pipeline. The right information exists somewhere in your knowledge base but did not get retrieved. This could be because the query was embedded poorly, because the relevant document is not in the index, because the similarity threshold is too strict, or because the relevant information is in a part of the document that was not chunked well.

If the answer is yes, retrieval worked and the problem is downstream. Move on to the next step.

### Step 3: Check the Full Prompt

Look at the complete prompt that was sent to the LLM. Are the retrieved chunks presented clearly? Are they labeled with their sources? Is the grounding instruction explicit -- does it clearly say to answer only from the provided context?

Subtle issues in prompt assembly can cause the model to ignore retrieved context or to treat it differently than intended.

### Step 4: Check the Raw Model Output

Look at the LLM's response before any post-processing, parsing, or formatting was applied. Was the hallucination or error present in the raw output, or was it introduced by something your code did to the output afterward?

This distinction matters. If the problem is in the raw output, you need to fix the prompt, the retrieval, or the model. If the problem is in post-processing, you need to fix your parsing or formatting code.

### Step 5: Determine Whether It Is Deterministic

Try running the same input again a few times. Does the problem happen every time, or only sometimes?

Problems that happen every time are usually structural: something in the retrieval, the prompt, or the post-processing is consistently wrong for this type of input. Fix the structure and it will stop happening.

Problems that happen only sometimes are usually stochastic: the model is uncertain about this type of input and sometimes takes the wrong path. These can often be addressed by lowering the temperature to reduce randomness, by adding stronger grounding instructions, or by adding a post-generation check that catches and retries this class of error.

### Step 6: Add to the Regression Dataset

Regardless of what caused the problem, once you understand and fix it, add this case to your eval dataset. You want to be sure that future changes do not reintroduce the same issue.

---

## 7. Key Takeaways

- A trace is a complete record of every step taken to process one user request, broken down into individual spans. It is the primary tool for debugging LLM applications.
- Tracing matters because LLM systems have many intermediate steps, and the correct answer is not obvious from inputs and outputs alone. You need to see what was retrieved, what prompt was built, and what the model actually received.
- For multi-agent systems, tracing becomes even more important because the agent takes many steps that are invisible without explicit instrumentation.
- LangSmith is the easiest way to get started with LLM tracing. It captures traces automatically for LangChain and LangGraph code and provides a web interface for exploration and debugging.
- OpenTelemetry is the alternative for organizations that already have observability infrastructure and want to integrate LLM traces into it.
- When debugging a bad response, follow the same systematic process every time: find the trace, check what was retrieved, check the full prompt, check the raw model output, determine whether the problem is deterministic or stochastic, and add the case to the regression dataset.

---

