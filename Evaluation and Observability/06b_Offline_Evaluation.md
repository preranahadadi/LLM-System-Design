# Module 06b -- Offline Evaluation
> Evaluation and Observability Series · Eval datasets, metrics by task type, LLM-as-judge, running evals before deployment

---

## Table of Contents
1. [What Offline Evaluation Is](#1-what-offline-evaluation-is)
2. [Building an Eval Dataset](#2-building-an-eval-dataset)
3. [Metrics for Different Task Types](#3-metrics-for-different-task-types)
4. [LLM-as-Judge](#4-llm-as-judge)
5. [Running the Eval and Analyzing Results](#5-running-the-eval-and-analyzing-results)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. What Offline Evaluation Is

Offline evaluation means measuring the quality of your system before it goes live, using a defined dataset of test cases with known expected outputs. It is how you decide whether a change is safe to deploy.

When you make any change to your LLM system -- a new prompt, a different model, a new chunking strategy, a different retrieval approach -- you need a way to know whether that change made things better or worse. Without an eval dataset and defined metrics, you are guessing.

Offline evaluation gives you a reproducible, quantitative answer. You run your system on the same set of test cases before and after the change and compare the scores. If the score improves, the change is likely good. If it drops, the change hurt something. If it improves on some cases but hurts others, you have a tradeoff to reason about.

The word "offline" distinguishes this from monitoring live production traffic. Offline evaluation runs on a controlled dataset where you already know what the right answer should be.

---

## 2. Building an Eval Dataset

The eval dataset is the foundation of everything. Without it, you cannot evaluate systematically. Building a good one takes real effort, but a small high-quality dataset is worth far more than a large noisy one.

### What a Test Case Contains

Every test case has at minimum an input and a description of what a correct output looks like. For classification tasks, the correct output might be an exact label. For text generation tasks, it might be a list of facts that should appear in the answer or a description of what quality means for this case.

Good test cases also carry metadata: what category does this case belong to, what is the difficulty level, and where did this case come from. This metadata lets you break down your eval results by category and understand which types of inputs your system handles well versus poorly.

### What Makes a Good Eval Dataset

Coverage is the most important property. Your eval dataset should represent the full range of inputs your system will encounter in production. If your system handles HR questions, IT questions, and finance questions, your eval set needs cases from all three areas. A dataset that only covers HR questions will give you misleading confidence about IT and finance performance.

Difficulty distribution matters. Include easy cases that the system should handle perfectly, medium cases that represent typical usage, and hard cases that represent edge cases and tricky inputs. If all your cases are easy, you will have high scores that do not tell you anything useful about how the system handles real challenges.

Balance matters for classification tasks. If your system classifies queries into categories and 90 percent of your eval cases are one category, a model that always predicts that category will score 90 percent even though it is completely useless. Make sure each category is represented appropriately.

Failure cases are the most valuable long-term investment. Every time your system produces a wrong answer on a real user query, add that query to your eval dataset. Over time this builds a regression suite that catches the same kind of failures if they come back.

### Sources for Eval Cases

The best source is domain experts writing questions and expected answers by hand. This produces the highest quality test cases because the people writing them actually know what correct answers look like. It is slow and expensive, but the quality is worth it for the core of your eval set.

A practical middle ground is generating candidates with an LLM and having humans review them. You can use GPT-4o or Claude to generate plausible questions from your documents, then have a subject matter expert filter out bad ones and correct any wrong expected answers. This is much faster than fully manual creation and can produce hundreds of cases in the time it would take to write dozens by hand.

Once your system is live, production traffic is the most valuable source of eval cases. Look at queries where users gave positive feedback and add them as positive examples. Look at queries where users complained, escalated, or asked the same question again, and add those as cases to fix. This keeps your eval set aligned with what real users actually care about.

### How Large the Eval Set Should Be

You do not need thousands of cases to start. At the development stage, 50 to 100 carefully chosen cases is enough to give you meaningful signal about whether a change is an improvement. Before launch, aim for 200 to 500 cases with good coverage across your main use cases. In production monitoring, grow toward 500 to 1000 cases over time as you add regression cases from real failures.

---

## 3. Metrics for Different Task Types

The right metric depends on what your system is supposed to do. Using the wrong metric can give you high scores that do not reflect actual quality, or low scores that make a good system look bad.

### Classification Tasks

When your system categorizes inputs into defined categories, exact match accuracy is the right starting point. It measures what fraction of predictions match the expected label exactly. Simple and interpretable.

For tasks where some categories are much rarer than others, accuracy can be misleading. If only 5 percent of queries are "billing" questions, a system that never predicts "billing" still gets 95 percent accuracy. In these cases, use F1 score, which balances how often the system correctly identifies each class against how often it incorrectly assigns that class to cases that do not belong.

### Text Generation Tasks

For open-ended text generation, exact match fails because there are many valid ways to express the same answer. Checking whether the generated text matches the expected text character by character is too strict and will penalize perfectly good answers.

Semantic similarity is a better approach. You convert both the generated answer and the expected answer into embeddings and measure how close they are in vector space. Two answers that express the same idea in different words will have high similarity. Two answers that say completely different things will have low similarity.

Avoid BLEU and ROUGE for LLM outputs. These metrics count how many word sequences appear in both the generated and reference text. They were designed for machine translation where you have one correct translation. For LLM outputs where there are many valid responses, BLEU and ROUGE correlate poorly with whether the answer is actually good.

### RAG Systems

For RAG systems specifically, the RAGAS framework measures four things that matter independently.

Context Recall measures whether retrieval found all the information needed to answer the question. A low score here means your retrieval pipeline is missing relevant documents.

Context Precision measures whether the retrieved documents were actually relevant. A low score here means retrieval is pulling in noisy, irrelevant chunks alongside the useful ones.

Faithfulness measures whether the generated answer is supported by the retrieved context. A low score here means the model is hallucinating content that was not in the retrieved documents.

Answer Relevance measures whether the answer actually addresses the question that was asked. A low score here means the answer is off-topic even if it is factually correct relative to the context.

Each of these can fail independently. Good retrieval with bad generation gives you high Context Recall but low Faithfulness. Bad retrieval with good generation gives you high Faithfulness (the model faithfully reports what it retrieved) but low Context Recall. You need to measure all four to understand where your system is actually failing.

### Factual Accuracy

For tasks where the answer should contain specific facts, check whether those facts appear in the output. Define a list of facts that must be present in a correct answer and count what fraction appear. This is more robust than exact match because it allows for different phrasings while still checking for the essential content.

---

## 4. LLM-as-Judge

For many real-world tasks, no programmatic metric captures quality well enough on its own. The answer is not a label or a structured field. It is a piece of text that needs to be judged for correctness, helpfulness, clarity, and appropriateness in context. Human judgment is the gold standard, but having humans evaluate every output is slow and expensive.

LLM-as-judge is the practical solution. You use a capable LLM to evaluate the outputs of your system by giving it a carefully written evaluation prompt that describes what quality means for your specific task.

### How It Works

You write a prompt that gives the judge LLM the question, the retrieved context, and the system's answer. You describe the dimensions of quality you care about and ask the judge to score each one. The judge returns scores and optionally an explanation of why it gave those scores.

The key is specificity. A vague prompt asking "is this a good answer" produces vague scores. A prompt that defines exactly what a score of 5 versus 3 versus 1 looks like on each dimension produces consistent, meaningful scores.

### Pairwise Comparison

An alternative to absolute scoring is pairwise comparison. Instead of asking "how good is this answer," you ask "which of these two answers is better." You run two versions of your system on the same eval cases -- for example, two different prompts -- and ask the judge to pick the better response for each case.

Pairwise comparison is more reliable than absolute scoring because it avoids the calibration problem. Different judge LLMs have different ideas of what a "7 out of 10" means. But most judges are reasonably consistent at saying which of two responses is better. This makes pairwise comparison particularly useful for A/B testing prompt changes or model changes.

### Limitations

LLM-as-judge has real biases you need to account for.

Length bias: judge LLMs tend to prefer longer, more detailed responses even when conciseness is better.

Style bias: a judge tends to prefer responses that match its own writing style, which can disadvantage responses from different models.

Self-preference: a model asked to judge outputs of the same model tends to rate them higher than it would rate outputs of other models.

To reduce these biases, use a different model as judge than the model being evaluated. Run the same evaluation multiple times and average the scores. Use pairwise comparison rather than absolute scoring for important comparisons. And always spot-check a sample of the judge's evaluations manually to catch systematic errors.

---

## 5. Running the Eval and Analyzing Results

### The Eval as a Deployment Gate

The primary purpose of offline eval is to gate deployments. Before any change goes to production, it must pass the eval. If the overall score drops, or if it drops significantly in any important category, the change does not deploy until the regression is understood and fixed.

This requires discipline. The temptation when you have made a change that you believe is an improvement is to deploy it and see what happens. The problem is that LLM systems fail silently. You will not see the damage until users complain.

Treat the eval pass rate as a hard gate. Decide in advance what score counts as passing. A reasonable starting point is that the new version must meet or exceed the current version's score on the full eval set, and must not regress on any category by more than a small margin.

### Analyzing Failures Before Making Changes

The most important step in the eval cycle is looking at failures carefully before making any changes. Group the failures by category. Look at the specific inputs that failed and the outputs that were produced. Ask what they have in common.

If 8 of your 10 failures are questions about a specific topic, that tells you something specific to investigate. If all failures involve very long or very short inputs, that points to a different issue. If failures are scattered randomly, the problem may be in the model's general quality rather than a specific case type.

Making changes without this analysis leads to a common trap: you change something that fixes the cases you are looking at while breaking other cases you were not checking. The overall score might not change, but the failure profile has shifted. Only by looking at individual failures can you understand whether your changes are genuine improvements or just redistributing errors.

### Regression Testing

Every case that previously failed and was fixed should stay in the eval set permanently as a regression test. Before promoting any new version, it must pass all regression cases. A change that fixes 3 new cases while breaking 2 previously fixed cases is not an improvement.

Over time, the regression test portion of your eval dataset becomes the most valuable part. It is a record of every failure mode you have encountered and fixed, ensuring you never accidentally reintroduce the same problems.

---

## 6. Key Takeaways

- Offline evaluation measures quality before deployment by running your system on a dataset of test cases with known expected outputs. It is how you decide whether a change is safe to deploy.
- A good eval dataset has coverage across all use cases, a mix of difficulty levels, balanced representation of each category, and regression cases from real failures. Start small and high-quality rather than large and noisy.
- Match the metric to the task. Accuracy for classification, semantic similarity for open-ended text, RAGAS for RAG systems, factual coverage for fact-based answers.
- LLM-as-judge fills the gap where no programmatic metric captures quality. Use pairwise comparison for comparing two versions. Be aware of length bias, style bias, and self-preference.
- Use the eval as a hard deployment gate. Make changes only after understanding why cases fail. Maintain regression cases for every failure you have fixed.

---
