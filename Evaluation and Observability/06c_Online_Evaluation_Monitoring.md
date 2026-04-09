# Module 06c -- Online Evaluation and Monitoring
> Evaluation and Observability Series · Production metrics, logging, detecting drift and degradation, alerting

---

## Table of Contents
1. [What Online Evaluation Is](#1-what-online-evaluation-is)
2. [What to Log](#2-what-to-log)
3. [Key Production Metrics](#3-key-production-metrics)
4. [Detecting Drift and Degradation](#4-detecting-drift-and-degradation)
5. [Setting Up Alerts](#5-setting-up-alerts)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. What Online Evaluation Is

Online evaluation means measuring quality and system health while your application is running in production, processing real user requests. Unlike offline evaluation which runs on a controlled dataset before deployment, online evaluation deals with the full unpredictability of real traffic.

The goal is to detect problems as quickly as possible after they appear. The sooner you know something is wrong, the sooner you can investigate and fix it. Without online monitoring, problems accumulate silently until they are severe enough that users complain loudly or abandon the system.

Online evaluation is necessarily more approximate than offline evaluation. You cannot always know the correct answer for every live request. Instead, you measure signals that correlate with quality: how often users give positive feedback, how often the system deflects instead of answering, how long responses take, how much they cost, and whether key quality metrics sampled on a subset of traffic remain stable.

---

## 2. What to Log

The foundation of online evaluation is comprehensive logging. You cannot monitor or debug what you did not capture. Err on the side of logging more rather than less, at least initially. You can always discard data you do not need. You cannot reconstruct data you did not capture.

### What to Capture for Every Request

For every request your system processes, log enough information to reproduce and investigate what happened.

The identity of the request: a unique request ID, the session ID so you can link related messages in a conversation, the user ID, and a timestamp.

The content of the exchange: the system prompt, the user's message, and the system's response. In a RAG system, also log the retrieved chunks including their source, similarity score, and content.

The cost and performance: how many input and output tokens were used, what the estimated cost was, and how long the total request took broken down by step if possible.

Quality signals: whether the user gave explicit feedback (thumbs up, thumbs down, a rating), whether the system returned a deflection response, and whether any guardrails triggered.

### Balancing Logging and Privacy

Logging full conversation content is the most useful approach for debugging, but it raises legitimate privacy concerns. Users may share personal information in their messages. Depending on your jurisdiction and industry, there may be legal requirements around how you store conversation data.

Practical approaches include logging everything but restricting access to a small team and setting automatic deletion after a defined retention period. Alternatively, log full content for a sample of requests (say 10 percent) and log only metadata (token counts, latency, feedback signals) for the rest. For sensitive industries, avoid logging content entirely and rely only on aggregate metrics.

Whatever approach you take, document it clearly and make sure users know what is being logged. The right to deletion (GDPR and similar regulations) also applies to conversation logs, so design your logging infrastructure to support deleting a specific user's data when requested.

---

## 3. Key Production Metrics

### Latency

Track how long it takes to produce a response for every request. Report it broken down into components when possible: how long retrieval took, how long the LLM generation took, and how long any post-processing took.

Report latency at multiple percentiles. The median (P50) tells you the typical experience. P95 tells you what the slower 5 percent of requests experience. P99 tells you the worst experiences your users have.

Latency matters because users have expectations. A response that takes more than 3 or 4 seconds starts to feel slow. A response that takes more than 10 seconds causes users to wonder if the system is broken. Latency also degrades silently -- as your vector database grows or as your traffic increases, retrieval can get slower without any obvious error.

### Cost

Track the cost of every request in terms of LLM tokens used. Sum this up to a daily total cost, an average cost per request, and a cost per session. Set a budget for how much you expect to spend per day and alert when you approach or exceed it.

Cost can explode suddenly in agentic systems if a loop bug causes the agent to make many more LLM calls than expected. A runaway agent that should cost $0.10 per task might cost $5 per task, which multiplied across many users adds up to serious money before anyone notices.

### Error Rate

Track the percentage of requests that result in specific error conditions.

LLM API errors include rate limit hits, timeout errors, and content policy rejections. A sudden spike in these indicates a problem with your API configuration or a surge in traffic that exceeds your rate limits.

Parse failures happen when the model was supposed to return structured output like JSON but returned something that could not be parsed. A high parse failure rate indicates a prompt engineering issue.

Tool call failures in agentic systems happen when a tool the agent tried to use returned an error. High tool failure rates indicate problems with external dependencies.

Deflection rate is how often the system responds with something like "I don't have information on this." Some deflection is healthy and correct -- the system should say it does not know rather than hallucinate. But a sudden increase in deflection rate can indicate that retrieval is failing or that the knowledge base is missing important content.

### User Feedback Signals

Explicit feedback is the most direct signal of quality. Thumbs up and thumbs down ratings, star ratings, or any feedback mechanism you provide in your interface tells you directly whether users found the response helpful.

Implicit feedback is often more abundant than explicit feedback because most users do not rate responses. Look for behavioral signals that correlate with satisfaction and dissatisfaction.

A user asking the same question again in different words shortly after receiving an answer is a strong implicit signal that the first answer was not helpful. A user abandoning the session immediately after receiving a response may have been satisfied (got what they needed) or dissatisfied (gave up). Context helps distinguish these. A user who immediately escalates to a human agent after receiving a response is a clear negative signal.

---

## 4. Detecting Drift and Degradation

Quality does not stay constant. Several things can cause gradual degradation that accumulates unnoticed without active monitoring.

### Input Distribution Drift

The kinds of questions your users ask change over time. When you first launched, maybe most questions were about one topic. Six months later, a new product line means users are asking about completely different things. Your system was designed and optimized for the original questions. Performance on the new question types may be poor, but this will not show up in your metrics if you are only tracking overall aggregate scores.

To detect this, track the distribution of topics or categories in incoming queries over time. If a new topic starts appearing with high volume, evaluate your system's performance on examples from that topic specifically. Do not wait until users complain to discover that performance on new question types is poor.

### Model Behavior Drift

Model providers update their models frequently. The same model name -- the same API endpoint -- can produce noticeably different outputs after an update. Your prompts were tuned for a particular version's behavior. After an update, the model may interpret your instructions slightly differently, produce outputs in a different format, or handle edge cases differently.

The most reliable way to detect this is running your offline eval set automatically on a regular schedule -- daily or weekly -- not just before deployments. If the quality score drops significantly between two consecutive runs and you made no changes to your system, the most likely cause is a model update. Compare the timestamps with any change logs published by the model provider.

### Prompt Regression from Your Own Changes

When you update a prompt to fix one thing, you can break something else. The offline eval should catch this before deployment, but your eval set may not cover every case. Online monitoring of quality metrics after deployment catches regressions on cases your eval set missed.

After any deployment, watch your quality metrics closely for 24 to 48 hours. If metrics deteriorate after a deployment but were stable before it, that deployment is the likely cause.

### Knowledge Base Staleness

In a RAG system, your documents have a freshness requirement. If your knowledge base contains outdated information and users ask about things that have changed, the system will return accurate-looking but wrong answers based on the stale content.

Monitor the age of documents in your knowledge base. Set staleness alerts for document types that change frequently. Establish a regular review cycle for your knowledge base content.

---

## 5. Setting Up Alerts

Alerts are the mechanism that turns monitoring into action. Without alerts, monitoring is just data sitting in dashboards that no one looks at until something is obviously broken.

Good alerts have three properties. They are specific about what went wrong. They are actionable, meaning the person receiving the alert knows what to do. And they are not too noisy -- an alert that fires constantly gets ignored.

### What to Alert On

Latency alerts trigger when response time exceeds user-visible thresholds. A reasonable starting point is alerting when the 95th percentile latency exceeds 5 seconds for more than a few consecutive minutes. This catches real performance problems while ignoring brief spikes.

Error rate alerts trigger when the percentage of requests hitting specific error conditions exceeds a threshold. If more than 5 percent of requests are returning LLM API errors, something is wrong with the API connection or rate limit configuration.

Cost alerts trigger when daily or hourly spend exceeds your budget threshold. Set these conservatively -- cost runaway can happen fast in agentic systems.

Quality alerts based on sampled evaluation trigger when a sampled quality metric drops below a threshold. For example, if your system checks the faithfulness of a 5 percent sample of responses and the average faithfulness score drops significantly, that is worth investigating even if users have not complained yet.

Deflection rate alerts trigger when the percentage of "I don't know" responses exceeds your expected range. Both too high (the system is deflecting when it should answer) and too low (the system is answering when it should deflect) are problems.

### Alert Routing

Not all alerts need the same response. A sudden doubling of cost needs immediate human attention. A slight gradual increase in deflection rate needs investigation but not emergency response.

Route high-severity alerts (sudden large changes in key metrics) to an on-call channel that will wake someone up if necessary. Route medium-severity alerts (gradual trends, moderate threshold breaches) to a team Slack channel for investigation during business hours. Log low-severity signals as data that goes into dashboards for weekly review.

The goal is that when something actually goes wrong, someone knows about it quickly and knows what to do. When nothing is wrong, no one is being woken up unnecessarily.

---

## 6. Key Takeaways

- Online evaluation monitors quality and system health in production. Unlike offline evaluation, it deals with real, unpredictable traffic and cannot always know the correct answer for every request.
- Log everything you might need to investigate a problem: the full request and response, retrieved chunks in RAG systems, token usage, latency, cost, and user feedback signals. Logging is the foundation of observability.
- The key metrics to track are latency at multiple percentiles, cost per request and in total, error rates broken down by type, deflection rate, and user feedback signals both explicit and implicit.
- Quality degrades silently from input distribution drift, model updates by the provider, prompt regressions from your own changes, and knowledge base staleness. Active monitoring is the only way to detect these.
- Set specific, actionable alerts for the conditions that actually require a response. Good alerts get noticed and acted on. Noisy alerts get ignored.

---

