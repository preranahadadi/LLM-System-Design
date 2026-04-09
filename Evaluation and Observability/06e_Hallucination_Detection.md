# Module 06e -- Hallucination Detection
> Evaluation and Observability Series · Types of hallucinations, detection approaches, production monitoring, prevention

---

## Table of Contents
1. [What Hallucination Is](#1-what-hallucination-is)
2. [Types of Hallucinations](#2-types-of-hallucinations)
3. [Why Hallucination Is Hard to Detect](#3-why-hallucination-is-hard-to-detect)
4. [Detection Approaches](#4-detection-approaches)
5. [Measuring Hallucination Rate in Production](#5-measuring-hallucination-rate-in-production)
6. [Preventing Hallucination at the Source](#6-preventing-hallucination-at-the-source)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. What Hallucination Is

Hallucination in LLMs refers to the model generating content that is factually incorrect or not supported by the provided context, presented with the same confidence and fluency as content that is accurate.

The term comes from the idea that the model is perceiving or generating something that is not grounded in reality. Unlike a human who might say "I think" or "I am not sure" when uncertain, an LLM produces hallucinated content in the same confident, fluent style as accurate content. There is no obvious tell.

In a RAG system specifically, hallucination typically means the model's response contains claims that are not present in or cannot be inferred from the retrieved documents. The model was given source material to work from and generated content that goes beyond or contradicts that source material.

Hallucination is considered the most serious failure mode in RAG systems because it actively misleads users. A system that says "I don't know" is unhelpful but honest. A system that confidently states something false is actively harmful.

---

## 2. Types of Hallucinations

Understanding the different types of hallucination helps you choose the right detection and prevention strategies.

### Factual Hallucination

The model states something that is simply not true and was not present in the retrieved context. This is the most straightforward type.

For example, the retrieved context might say "The policy was updated in 2024" and the model says "The policy was updated in March 2023." The model generated a specific date that was not in the source material and that happens to be wrong.

### Context Contradiction

The model contradicts information that was explicitly in the retrieved context. This is a particularly bad form of hallucination because the correct information was right there in front of the model and it said the opposite.

For example, the retrieved context says "Remote work is permitted up to three days per week" and the model says "Employees can work remotely full-time." The model has directly contradicted its source material.

### Unsupported Inference

The model draws a conclusion that goes beyond what the retrieved context supports. The individual facts in the response may be correct, but the model has combined them in a way or drawn implications that the source material does not support.

For example, the retrieved context says "The office closes at 6pm" and the model says "Since the office closes at 6pm, overtime work is not permitted." The second claim is not in the source material and is not a logical necessity from the first claim. The model has invented a policy that was not stated.

### Fabricated Citations

The model cites a source that does not exist or attributes a quote to a document that does not contain it. This happens when the model has learned that citing sources is a mark of quality and generates plausible-sounding but false citations.

For example, the model says "According to the HR Policy document, Section 7.3, employees are entitled to 20 days of annual leave." But there is no Section 7.3, and the actual entitlement is 15 days.

---

## 3. Why Hallucination Is Hard to Detect

Hallucination is hard to detect for several interconnected reasons.

The output always looks correct. Unlike a code error that produces an exception or a database error that returns a null value, a hallucinated response looks exactly like a correct response. The text is fluent, grammatically correct, and formatted appropriately. There is nothing in the surface form that distinguishes it from an accurate response.

Detection requires knowing the truth. To know whether a claim is hallucinated, you need to know what the truth actually is. In a RAG system, you can check whether a claim is in the retrieved context. But what if the retrieved context itself is wrong, or if the right context was not retrieved? Checking faithfulness to context only catches one type of hallucination.

Scale makes manual checking impossible. In a production system processing thousands of queries per day, manually reading every response to check for hallucinations is not feasible. You need automated approaches that scale.

The model is calibrated to be confident. LLMs are trained to produce helpful, confident responses. This training makes them useful but also makes them more prone to expressing uncertain or hallucinated content with the same confidence they use for well-grounded content.

---

## 4. Detection Approaches

### NLI-Based Detection

Natural Language Inference (NLI) is a technique for determining whether one piece of text supports, contradicts, or is unrelated to another piece of text. NLI models are trained specifically for this judgment task.

For hallucination detection, you use an NLI model to check whether the generated response is entailed by (supported by) the retrieved context. You break the response into individual claims and check each one against the context.

The NLI model assigns one of three labels to each (context, claim) pair: entailment means the context supports the claim, contradiction means the context contradicts the claim, and neutral means the context neither supports nor contradicts the claim. Claims labeled as contradiction or neutral are potential hallucinations.

NLI-based detection is fast, cheap to run at scale, and does not require an additional LLM call. Its main limitation is that it works at the sentence level and can miss subtle hallucinations that span multiple sentences or require understanding context beyond individual claims.

### LLM-as-Judge for Faithfulness

Using a capable LLM to judge whether the response is grounded in the retrieved context is more nuanced than NLI but more expensive. You write a prompt that gives the judge the retrieved context and the generated response, asks it to identify any claims not supported by the context, and returns a faithfulness score.

This approach handles complex, multi-sentence reasoning better than NLI models. It can catch unsupported inferences and subtle contradictions that NLI might miss. The downside is that it adds another LLM call to every request you evaluate, which adds latency and cost if done inline.

In practice, LLM-as-judge for faithfulness is most useful for offline evaluation and for sampling a subset of production responses rather than checking every response in real time.

### Token Probability (Confidence) Signals

Some LLM APIs can return the probability that the model assigned to each generated token. Tokens generated with very low probability are ones the model was uncertain about. Clusters of low-probability tokens in factual claims can indicate the model is generating content it is not confident about.

This signal is imperfect because a model can be confident and wrong. But very low probability on a specific token that represents a factual claim is a useful signal worth investigating. Combined with NLI or LLM-as-judge, it can help prioritize which responses to check more carefully.

Not all APIs expose token probabilities, and using this signal requires additional processing. It is a secondary signal rather than a primary detection mechanism.

### Post-Generation Groundedness Check

A practical inline detection mechanism is a post-generation check that runs immediately after the LLM produces its response, before the response is returned to the user. The check determines whether the response is grounded in the retrieved context. If not, the system either retries with stricter grounding instructions or returns a deflection response instead.

This works well as a safety net for the most obvious hallucinations. It adds some latency (for the NLI model or lightweight LLM call), but the cost of returning a hallucinated response to a user is usually worth more than the latency.

---

## 5. Measuring Hallucination Rate in Production

Because checking every response for hallucination is expensive, use sampling. Check a random sample of production responses -- 5 to 10 percent is typically enough to give you a reliable estimate of the overall hallucination rate.

The hallucination rate you should aim for depends on your use case. For general knowledge assistants, below 5 percent is reasonable. For high-stakes use cases like medical information or legal guidance, you need to be much more strict -- below 1 percent or even lower.

Track the hallucination rate over time and alert when it changes significantly. A sudden increase in hallucination rate is a signal worth investigating immediately. Gradual drift over weeks can indicate that your knowledge base is becoming stale relative to the questions being asked, or that a model update changed behavior.

When you find a hallucinated response in your sample, save it to your evaluation dataset. Understanding the pattern of hallucinations in your specific system -- what kinds of questions trigger them, what kinds of content in the retrieved context leads to them -- helps you fix them at the source rather than just detecting them after the fact.

---

## 6. Preventing Hallucination at the Source

Detection is one layer of defense. Prevention is better. The following approaches reduce hallucination frequency rather than catching it after it happens.

### Temperature

Use a low temperature setting for factual RAG tasks. Temperature controls how much randomness is in the model's token selection. Higher temperatures make the model more creative and varied, but also more prone to generating content that is not grounded in the context. For factual question answering, low temperature makes the model more likely to stay close to what the retrieved context actually says.

### Explicit Grounding Instructions

Be very explicit in your system prompt that the model must answer only from the provided context. Vague instructions like "use the context below" are weaker than explicit ones like "Answer only using the information in the provided context. If the answer is not present in the context, say so clearly. Do not add information from your training knowledge."

The difference in wording matters. Models follow strong, specific negative constraints more reliably than general positive instructions.

### Retrieval Confidence Threshold

If the highest similarity score among retrieved chunks is below a threshold, the knowledge base probably does not contain a good answer to this question. Rather than sending low-quality context to the LLM and hoping it handles the uncertainty well, return a deflection response directly without calling the LLM at all. This eliminates one class of hallucinations -- the ones where the model tries to answer a question for which it has no good source material.

### Source Attribution

Requiring the model to cite a specific source for each claim acts as a natural check on hallucination. A model that cannot point to a specific sentence in the context for a claim is more likely to omit that claim when citation is required. Attribution does not eliminate hallucination, but it reduces it by making the model more explicitly connect each claim to its source.

### Knowledge Base Quality

Hallucination is more likely when the retrieved context is ambiguous, contradictory, or low quality. Invest in the quality of your documents. Remove outdated content that contradicts more recent documents. Improve chunking to ensure relevant context is not split across chunks. Make sure your knowledge base is comprehensive enough to actually answer the questions your users ask.

---

## 7. Key Takeaways

- Hallucination is when the model generates content that is not supported by or contradicts the retrieved context, presented with the same confidence as accurate content. It is the most dangerous failure mode in RAG systems because it actively misleads users.
- The main types are factual hallucination (stating something false), context contradiction (directly contradicting the retrieved content), unsupported inference (drawing conclusions beyond what the context supports), and fabricated citations.
- Hallucination is hard to detect because the output always looks correct, detecting it requires knowing the truth, and scale makes manual checking impossible.
- Detection approaches include NLI-based checking (fast, cheap, works well for sentence-level claims), LLM-as-judge (more nuanced, better for complex reasoning, more expensive), and post-generation groundedness checks (inline safety net before returning responses to users).
- Measure hallucination rate in production by sampling 5 to 10 percent of responses and checking them. Track the rate over time and alert on significant changes.
- Prevent hallucination through low temperature, explicit grounding instructions, retrieval confidence thresholds, source attribution requirements, and high-quality knowledge base content.

---

