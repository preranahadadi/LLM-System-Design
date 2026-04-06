# Module 04f -- Prompt Optimization and Testing
> Prompt Engineering Series · Prompt eval sets, A/B testing, versioning, DSPy, the optimization loop

---

## Table of Contents
1. [Why You Need to Test Prompts](#1-why-you-need-to-test-prompts)
2. [Building a Prompt Eval Set](#2-building-a-prompt-eval-set)
3. [Running the Eval](#3-running-the-eval)
4. [A/B Testing Prompts](#4-ab-testing-prompts)
5. [Prompt Versioning](#5-prompt-versioning)
6. [DSPy -- Automatic Prompt Optimization](#6-dspy----automatic-prompt-optimization)
7. [The Prompt Engineering Loop](#7-the-prompt-engineering-loop)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Why You Need to Test Prompts

A prompt that works well in your initial testing may fail on 15 percent of production inputs. Without systematic testing you do not know this until users complain.

Prompts also degrade. Model providers update their models. The same prompt text can produce different outputs after a model update even if you changed nothing on your end. Without a baseline eval, you will not notice when this happens.

There is also a subtler problem: manual iteration on prompts can improve performance on the cases you are looking at while silently degrading performance on cases you are not looking at. Changing a system prompt instruction to fix one failure can break three other cases you did not check.

Treat prompts like code. Version them, test them against a defined set of cases, and track performance over time. A prompt that passes your eval is not necessarily perfect, but a prompt that fails your eval is definitely not ready for production.

---

## 2. Building a Prompt Eval Set

An eval set is a collection of test cases: input plus expected output or expected properties of the output. You run your prompt against all of them and measure how many it passes.

### Structure of a Test Case

```python
eval_case = {
    # The input to the prompt
    "input": "The packaging was damaged but the product itself works fine.",

    # What the output should be (exact match, or properties to check)
    "expected_output": {"sentiment": "Neutral"},

    # Description of what this case is testing
    "description": "Mixed signal case: physical damage but product functional",

    # Optional: the category this case belongs to (for analysis by category)
    "category": "mixed_signal",
}
```

### Types of Test Cases to Include

Include a mix that covers the full range of real inputs:

Typical cases: inputs that represent the most common thing users will send. These should be easy. If your prompt fails on typical cases it is fundamentally broken.

Edge cases: inputs near decision boundaries where the right classification is not obvious. A review that is mildly positive. A question that is almost but not quite in scope.

Adversarial cases: inputs designed to trip up the model. Very short inputs. Very long inputs. Inputs with unusual formatting. Inputs in a language you did not expect. Inputs that contain partial instructions.

Previously failed cases: every time your prompt fails on a production input, add that input to the eval set. Over time this becomes a regression suite that catches regressions when you make changes.

Cases that should be refused or deflected: if your system should say "I cannot help with that" for certain inputs, include those cases and verify the model handles them correctly.

### How Many Cases You Need

| Stage | Minimum number |
|---|---|
| Early development | 20 to 30 |
| Pre-production | 100 to 200 |
| Production monitoring | 300 to 500 |

Start small. A small high-quality eval set is more useful than a large noisy one. Grow it over time from production failures.

### Coverage Check

Before launching, verify your eval set covers:
- All output categories (at least 5 to 10 examples per category for classification tasks)
- Easy, medium, and hard examples
- Both success cases and failure/refusal cases
- Different input lengths and styles

---

## 3. Running the Eval

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class EvalResult:
    case: dict
    actual_output: str
    passed: bool
    score: float
    notes: str

def run_eval(
    prompt_template: str,
    eval_cases: list[dict],
    llm,
    metric_fn: Callable
) -> tuple[float, list[EvalResult]]:

    results = []

    for case in eval_cases:
        # Format the prompt with this case's input
        prompt = prompt_template.format(input=case["input"])

        # Get the model's output
        actual_output = llm.invoke(prompt)

        # Score it against the expected output
        score, notes = metric_fn(actual_output, case["expected_output"])
        passed = score >= 0.8

        results.append(EvalResult(
            case=case,
            actual_output=actual_output,
            passed=passed,
            score=score,
            notes=notes
        ))

    pass_rate = sum(r.passed for r in results) / len(results)
    return pass_rate, results
```

### Metric Functions

The right metric depends on your task.

Exact match for classification:
```python
def exact_match_metric(actual: str, expected: dict) -> tuple[float, str]:
    actual_clean = actual.strip().lower()
    expected_value = expected["sentiment"].lower()

    if actual_clean == expected_value:
        return 1.0, "Correct"
    else:
        return 0.0, f"Expected {expected_value}, got {actual_clean}"
```

Structured output check:
```python
def json_field_metric(actual: str, expected: dict) -> tuple[float, str]:
    try:
        parsed = json.loads(actual)
    except json.JSONDecodeError:
        return 0.0, "Output is not valid JSON"

    correct_fields = 0
    notes = []

    for field, expected_value in expected.items():
        if field not in parsed:
            notes.append(f"Missing field: {field}")
        elif parsed[field] != expected_value:
            notes.append(f"{field}: expected {expected_value}, got {parsed[field]}")
        else:
            correct_fields += 1

    score = correct_fields / len(expected)
    return score, "; ".join(notes) if notes else "All fields correct"
```

LLM-as-judge for open-ended output:
```python
def llm_judge_metric(actual: str, expected: dict, judge_llm) -> tuple[float, str]:
    judge_prompt = f"""
    Expected properties of the output: {expected["requirements"]}
    Actual output: {actual}

    Score the output from 0 to 10 based on how well it meets the requirements.
    Return only a number from 0 to 10.
    """
    score_str = judge_llm.invoke(judge_prompt).strip()
    score = float(score_str) / 10.0
    return score, f"Judge score: {score_str}/10"
```

### Analyzing Failures

After running the eval, look at the failures before changing anything.

```python
def analyze_failures(results: list[EvalResult]):
    failures = [r for r in results if not r.passed]

    print(f"Failed {len(failures)} of {len(results)} cases")
    print()

    # Group failures by category
    by_category = {}
    for failure in failures:
        cat = failure.case.get("category", "uncategorized")
        by_category.setdefault(cat, []).append(failure)

    for category, cat_failures in by_category.items():
        print(f"Category '{category}': {len(cat_failures)} failures")
        for f in cat_failures[:3]:   # show up to 3 per category
            print(f"  Input:    {f.case['input'][:80]}")
            print(f"  Expected: {f.case['expected_output']}")
            print(f"  Got:      {f.actual_output[:80]}")
            print(f"  Notes:    {f.notes}")
            print()
```

Group failures by category to find patterns. If 8 of your 10 failures are in the "mixed signal" category, that tells you what to fix. Blind edits to the prompt without this analysis often fix one category while breaking another.

---

## 4. A/B Testing Prompts

When you have two candidate prompts, run them against the same eval set and compare.

```python
def ab_test_prompts(
    prompt_a: str,
    prompt_b: str,
    eval_cases: list[dict],
    llm,
    metric_fn: Callable,
    num_runs: int = 1   # increase for tasks with randomness
) -> dict:

    scores_a = []
    scores_b = []

    for run in range(num_runs):
        rate_a, results_a = run_eval(prompt_a, eval_cases, llm, metric_fn)
        rate_b, results_b = run_eval(prompt_b, eval_cases, llm, metric_fn)
        scores_a.append(rate_a)
        scores_b.append(rate_b)

    avg_a = sum(scores_a) / len(scores_a)
    avg_b = sum(scores_b) / len(scores_b)

    return {
        "prompt_a_score": avg_a,
        "prompt_b_score": avg_b,
        "winner": "A" if avg_a > avg_b else "B",
        "margin": abs(avg_a - avg_b)
    }
```

### What to Compare

Score on the full eval set. A prompt that improves performance on some cases while regressing on others may have a similar overall score but a different failure profile. Look at which specific cases each prompt fails on, not just the aggregate score.

Run multiple times if the task has any randomness (temperature > 0). A single run may not accurately represent average performance.

Test on cases the model has not seen before. If you tuned your prompt specifically on your eval cases, the eval may overestimate performance. Keep a held-out test set that you never use during prompt development.

### Statistical Significance

For small eval sets, differences in score may be noise rather than real improvement. If your eval set has 50 cases and one prompt scores 84% versus another at 86%, that 2 percentage point difference is probably not meaningful.

As a rough rule: you need at least 100 cases for 5 percentage point differences to be reliable, and at least 300 cases for 2 percentage point differences.

---

## 5. Prompt Versioning

Store prompts in version control. Track what changed and what effect the change had.

### File Organization

```
prompts/
  sentiment_classifier/
    v1.txt
    v1_eval_results.json
    v2.txt
    v2_eval_results.json
    CHANGELOG.md
    current_version.txt   -- contains "v2"
```

### Changelog

```markdown
# Sentiment Classifier Prompt Changelog

## v2 (2026-03-15)
Changed: Added instruction to treat sarcasm as Negative.
Reason: v1 was classifying sarcastic positive phrases as Positive.
Eval results: 87% pass rate (up from 79% in v1).
Regressions: None found on existing eval cases.

## v1 (2026-02-01)
Initial version.
Eval results: 79% pass rate on 150 test cases.
```

### Loading the Current Version

```python
import os

def load_prompt(prompt_name: str) -> str:
    prompt_dir = f"prompts/{prompt_name}"
    current_version = open(f"{prompt_dir}/current_version.txt").read().strip()
    return open(f"{prompt_dir}/{current_version}.txt").read()
```

### Regression Gate

Before promoting a new prompt version, it must pass all cases that the previous version passed.

```python
def check_for_regressions(
    new_prompt: str,
    old_results: list[EvalResult],
    llm,
    metric_fn: Callable
) -> list[dict]:

    previously_passed = [r.case for r in old_results if r.passed]
    _, new_results = run_eval(new_prompt, previously_passed, llm, metric_fn)

    regressions = [r for r in new_results if not r.passed]
    return regressions
```

If `check_for_regressions` returns any cases, you have a regression. Fix it before promoting the new version.

---

## 6. DSPy -- Automatic Prompt Optimization

DSPy is a framework that treats prompts as learnable parameters and optimizes them automatically using your eval set. Instead of writing prompt text by hand, you define the task signature and the metric, and DSPy finds the prompt that maximizes it.

### Core Concepts

Signature: the input-output specification of your task.
Module: a component that uses LLM calls to implement the signature.
Optimizer (Teleprompter): searches for the best prompt and few-shot examples.

### Basic Example

```python
import dspy

# Configure the LLM
lm = dspy.OpenAI(model="gpt-4o-mini", max_tokens=500)
dspy.settings.configure(lm=lm)

# Define the task signature
class SentimentClassifier(dspy.Signature):
    """Classify the sentiment of a product review."""
    review = dspy.InputField(desc="A customer product review")
    sentiment = dspy.OutputField(desc="Exactly one of: Positive, Negative, Neutral")

# Define the module
class SentimentModule(dspy.Module):
    def __init__(self):
        super().__init__()
        self.classify = dspy.Predict(SentimentClassifier)

    def forward(self, review):
        return self.classify(review=review)

# Define the metric
def sentiment_metric(example, prediction, trace=None):
    return example.sentiment.lower() == prediction.sentiment.lower()

# Set up training data
train_data = [
    dspy.Example(review="Loved it!", sentiment="Positive").with_inputs("review"),
    dspy.Example(review="Terrible.", sentiment="Negative").with_inputs("review"),
    # ... more examples
]

# Optimize
from dspy.teleprompt import BootstrapFewShot

optimizer = BootstrapFewShot(metric=sentiment_metric)
optimized_module = optimizer.compile(SentimentModule(), trainset=train_data)

# Use it
result = optimized_module(review="The packaging was damaged but the product works.")
print(result.sentiment)   # Neutral
```

DSPy searches through candidate prompts and few-shot example selections to maximize your metric. You do not write the prompt text manually.

### When to Use DSPy

DSPy is a good fit when:
- The task is well-defined with a clear programmatic metric
- Manual prompt iteration is not converging on good results
- You have at least 50 labeled examples to optimize against
- You want to automate prompt improvement as a continuous process

DSPy is not a good fit when:
- The metric is hard to define programmatically (you need human judgment)
- You have fewer than 30 labeled examples
- The task is too open-ended for automatic optimization

---

## 7. The Prompt Engineering Loop

This is the practical workflow for developing and maintaining prompts.

```
Step 1: Write an initial prompt based on your understanding of the task.
        Start simple. Do not over-engineer the first version.

Step 2: Build a small eval set of 20 to 30 cases covering typical inputs,
        edge cases, and any cases you already know are hard.

Step 3: Run the eval. Record the pass rate and look at every failure.

Step 4: Understand why each failure happened before changing anything.
        Is the instruction unclear?
        Is there a case the prompt does not handle?
        Is the model doing something unexpected?

Step 5: Make one targeted change based on your analysis.
        Do not make multiple changes at once or you will not know which one helped.

Step 6: Re-run the eval. Confirm the failure is fixed.
        Check that you did not introduce regressions.

Step 7: Add the previously failing case to your eval set permanently.

Step 8: When you have production traffic, add failures from production
        to the eval set. Repeat from step 3.
```

The most important discipline in this loop is step 4. Do not change the prompt until you understand why it failed. The most common mistake is making a change that sounds like it should help, re-running, seeing the score improve slightly, and concluding the change was good -- when in fact you fixed two cases but broke one that you are not checking.

---

## 8. Key Takeaways

- Test prompts systematically before launch and continuously after. A prompt that passes your eval may still fail in production, but a prompt that fails your eval is definitely not ready.
- Build your eval set before you start iterating on the prompt. Cases should cover typical inputs, edge cases, adversarial inputs, and previously failed production examples.
- Always analyze failures before making changes. Understand why each case failed. Make one targeted change at a time.
- Use A/B testing when comparing two prompt candidates. Look at which specific cases each fails on, not just the aggregate score.
- Version prompts in version control. Record eval results with each version. Run regression checks before promoting a new version.
- DSPy automates prompt optimization if you have a well-defined metric and enough labeled examples. Start with manual iteration; move to DSPy when manual approaches stop improving results.

---
