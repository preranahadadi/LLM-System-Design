# Module 06a -- Why Evaluation and Observability Matter
> Evaluation and Observability Series · The core problem, eval vs observability, what can go wrong silently

---

## Table of Contents
1. [The Core Problem with LLM Systems](#1-the-core-problem-with-llm-systems)
2. [Evaluation vs Observability](#2-evaluation-vs-observability)
3. [What Can Go Wrong Silently](#3-what-can-go-wrong-silently)
4. [The Measurement Mindset](#4-the-measurement-mindset)
5. [Key Takeaways](#5-key-takeaways)

---

## 1. The Core Problem with LLM Systems

Traditional software is deterministic. Give it the same input and you get the same output every time. When something breaks, there is usually a clear error message, an exception, or a crash. You know immediately that something is wrong and often have a clue about where to look.

LLM systems are fundamentally different in ways that make them much harder to monitor and debug.

They are non-deterministic. The same input can produce different outputs on different runs. There is no single definitively correct answer to check against for most tasks. Two responses can both be acceptable even if they are completely different in wording.

They fail silently. A traditional function either returns a correct result or throws an error. An LLM always returns text. That text always looks like a plausible answer even when it is completely wrong. There is no exception. No error code. The system appears to be working fine while producing incorrect or misleading output.

They degrade gradually. A database query either works or it does not. An LLM system can slowly get worse over time as the underlying model gets updated by the provider, as the kinds of questions users ask start to shift, or as your knowledge base becomes stale. This degradation is invisible until you measure for it explicitly.

They have multiple independent points of failure. A RAG pipeline has document ingestion quality, chunking quality, embedding quality, retrieval quality, prompt quality, generation quality, and output formatting. Any of these can fail independently while the others partially compensate, making the overall failure mode diffuse and hard to pinpoint.

This is why evaluation and observability are not optional features you add later. They are the only way to know whether your LLM system is actually working and continuing to work over time.

---

## 2. Evaluation vs Observability

These two terms are closely related and often used interchangeably, but they mean different things and serve different purposes.

**Evaluation** is about measuring quality. It answers the question: does this system produce good outputs? Evaluation is typically done offline, before deployment, against a defined dataset of test cases with known expected outputs. It tells you whether a change you made -- a new prompt, a new model, a new retrieval strategy -- is better or worse than what you currently have. It is a gate before deployment.

**Observability** is about visibility into what is actually happening in production. It answers the question: what is my live system doing right now, and is it healthy? Observability is about logging, tracing, and monitoring real traffic. It tells you when something starts going wrong after deployment and helps you understand why.

A simple way to remember the difference:

Evaluation asks "is this good enough to deploy?"

Observability asks "is the deployed system continuing to work well?"

You need both. Without evaluation you deploy blindly and only learn about problems from user complaints. Without observability you deploy confidently but have no visibility into what is happening in production. A system with good evaluation but no observability is one where you know it worked on your test cases but have no idea what is happening to real users. A system with good observability but no evaluation is one where you can see problems happening but have no baseline to compare against or systematic way to test improvements.

---

## 3. What Can Go Wrong Silently

Understanding the specific failure modes of LLM systems helps you decide what to measure and monitor.

**Hallucination.** The model states something confidently that is not true and was not in the retrieved context. The output looks perfectly reasonable. A user who does not know the domain well will not notice. The system has no awareness that it has done anything wrong. This is the most dangerous failure mode because it actively misleads users rather than simply failing to help them.

**Retrieval failure.** In a RAG system, the wrong chunks are retrieved from the knowledge base. The model does its best to answer using irrelevant content. The answer sounds plausible because the model is good at generating fluent text, but it is based on the wrong source material. The model itself cannot tell you that retrieval failed.

**Prompt regression.** The model provider silently updates the model behind the same API endpoint. A prompt that worked well before now produces subtly worse outputs because the new model interprets instructions slightly differently. The change is not dramatic enough to be immediately obvious, but quality has degraded for every user.

**Distribution shift.** Your users gradually start asking different kinds of questions than what your system was designed and tested for. Your evaluation dataset covered the original question types well, so your metrics look fine. But on the new kinds of questions, performance is poor. You will not see this unless you monitor the types of questions being asked and evaluate on examples that reflect current usage.

**Compounding errors in agents.** In a multi-step agentic system, a small error in an early step influences all subsequent steps. The agent builds on incorrect intermediate results and by the final step the answer is completely wrong. Each individual step looked reasonable in isolation. The compounding nature of the error makes it very hard to catch without tracing the full execution.

**Latency degradation.** Your vector store grows as you add more documents. Retrieval that used to take 200ms now takes 3 seconds. Users wait noticeably longer for answers. This is invisible unless you track latency over time.

**Cost explosion.** An agentic task that should complete in 5 LLM calls starts taking 25 because of a subtle loop or inefficiency introduced by a prompt change. At low volume this is not obvious. At scale it costs significant money before anyone realizes something changed.

---

## 4. The Measurement Mindset

The fundamental shift required for working with LLM systems is moving from "does it work" to "how well does it work, how do I know, and how will I know if that changes."

For traditional software the mental model is: write code, run tests, pass or fail, deploy. The tests give you binary confidence. Either the code is correct or it is not.

For LLM systems the mental model needs to be: write the system, define what quality means for this task, build a dataset of test cases, measure quality on that dataset, deploy, monitor the same quality metrics in production, alert when metrics change significantly, investigate and improve, redeploy.

This cycle never ends. Evaluation is not a one-time activity before launch. It is an ongoing practice. The world changes, your users change, the underlying models change, and your system needs to keep up.

The practical implication is that building an LLM system requires you to invest in measurement infrastructure alongside the system itself. The eval dataset, the metrics, the logging, the dashboards, and the alerts are not optional extras. They are part of the system.

---

## 5. Key Takeaways

- LLM systems fail silently. Unlike traditional software that throws errors, an LLM always returns plausible-looking text even when that text is wrong. This makes measurement essential rather than optional.
- LLM systems degrade gradually. Model updates, distribution shift, and knowledge base staleness all cause slow quality degradation that is invisible without active monitoring.
- Evaluation and observability serve different purposes. Evaluation measures quality before deployment and gates releases. Observability monitors quality after deployment and catches problems in production.
- The main failure modes to watch for are hallucination, retrieval failure, prompt regression, distribution shift, compounding errors in agents, latency degradation, and cost explosion.
- Building an LLM system requires investing in measurement infrastructure from the start. The eval dataset, the metrics, the logging, and the alerts are as important as the system itself.

---

