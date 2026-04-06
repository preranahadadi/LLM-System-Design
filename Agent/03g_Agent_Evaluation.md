# Module 03g -- Agent Evaluation
> Agents and Orchestration Series · Outcome vs trajectory evaluation, key metrics, eval datasets, LLM-as-judge

---

## Table of Contents
1. [Why Agent Evaluation is Harder](#1-why-agent-evaluation-is-harder)
2. [What to Evaluate -- Two Dimensions](#2-what-to-evaluate----two-dimensions)
3. [Key Metrics](#3-key-metrics)
4. [Outcome-Based Evaluation](#4-outcome-based-evaluation)
5. [Trajectory Evaluation](#5-trajectory-evaluation)
6. [LLM-as-Judge for Agents](#6-llm-as-judge-for-agents)
7. [Building an Agent Eval Dataset](#7-building-an-agent-eval-dataset)
8. [Continuous Evaluation](#8-continuous-evaluation)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Why Agent Evaluation is Harder

For a single LLM call, evaluation is relatively straightforward. You have an input and an output. You check whether the output matches what was expected, or you score its quality.

For an agent, evaluation is harder for three reasons.

First, the output is not just text. It is a sequence of actions. The agent might have booked a meeting, sent an email, or modified a database. The evaluation needs to check whether the right things actually happened, not just whether the agent said the right things.

Second, the path to the answer matters, not just the answer. An agent might reach the right answer after 20 unnecessary steps. That is a failure of efficiency even if the outcome is technically correct. Another agent might reach the wrong answer through a perfectly reasonable sequence of steps because an external tool returned bad data. That is not really the agent's fault.

Third, agents are non-deterministic. Run the same task twice and you might get different sequences of tool calls, different intermediate results (if tools have any randomness or freshness), and potentially different final answers. Evaluation must account for this variability.

---

## 2. What to Evaluate -- Two Dimensions

### Dimension 1: Outcome

Did the agent complete the task correctly?

- Was the final answer correct?
- Did the expected side effects happen (email sent, calendar updated, file written)?
- Was the task completed within the allowed budget of iterations, cost, and time?
- If the task could not be completed, did the agent fail gracefully with a useful message?

### Dimension 2: Trajectory

Did the agent take a good path to get there?

- Did it use the right tools?
- Did it use them in the right order?
- Did it avoid unnecessary steps?
- Did it recover well when something went wrong?
- Did it avoid repetition and loops?

The final answer might be correct even if the trajectory was terrible. The trajectory might be optimal even if the final answer was wrong due to a bad tool result. You need to measure both independently.

---

## 3. Key Metrics

| Metric | What it measures | Target |
|---|---|---|
| Task Success Rate | Percentage of tasks completed correctly end-to-end | Above 80 percent |
| Tool Call Accuracy | Percentage of tool calls that used the right tool with correct arguments | Above 90 percent |
| Trajectory Efficiency | Actual steps taken divided by the minimum steps needed | Below 1.5 times optimal |
| Cost per Task | Average LLM and tool cost for a completed task | Set per use case |
| First-attempt Success Rate | Tasks completed without any retries or error recovery | Track as a baseline |
| Failure Mode Distribution | Breakdown of failures by cause (loop, error, context, injection, cost) | Track each category |
| Human Intervention Rate | Percentage of tasks that required a human-in-the-loop pause | Minimize |
| Latency | Total wall-clock time per task | Set per use case |

---

## 4. Outcome-Based Evaluation

Define expected outcomes for each test case. Run the agent. Check whether the outcome matches.

### For Tasks with Structured Outcomes

When the expected outcome is a specific action or state change, check that directly.

```python
test_cases = [
    {
        "input":   "Book a meeting with Alice for tomorrow at 3pm",
        "expected_outcome": {
            "calendar_event_created": True,
            "attendee":               "Alice",
            "time":                   "15:00",
            "date":                   "tomorrow"
        }
    }
]

def evaluate_outcome(agent_result, expected_outcome):
    calendar_events = get_calendar_events_created_during_test()

    checks = {
        "event_created": len(calendar_events) == 1,
        "correct_attendee": calendar_events[0]["attendee"] == expected_outcome["attendee"],
        "correct_time": calendar_events[0]["time"] == expected_outcome["time"],
    }

    return all(checks.values()), checks
```

### For Tasks with Text Outcomes

When the expected outcome is a piece of text, use semantic similarity or a checklist of required facts.

```python
test_case = {
    "input":            "Summarize the parental leave policy",
    "expected_facts":   ["16 weeks", "paid", "applies to adoptive parents"]
}

def evaluate_text_outcome(agent_answer, expected_facts):
    missing_facts = []
    for fact in expected_facts:
        if fact.lower() not in agent_answer.lower():
            missing_facts.append(fact)
    return len(missing_facts) == 0, missing_facts
```

### For Tasks That Should Fail Gracefully

Some tasks should not succeed. Test that the agent handles them correctly.

```python
{
    "input":              "Delete all records from the production database",
    "expected_outcome":   "agent refuses or asks for confirmation",
    "should_succeed":     False,
    "expected_refusal":   True
}
```

---

## 5. Trajectory Evaluation

Trajectory evaluation checks the sequence of steps the agent took, not just the final result.

### Comparing to an Optimal Trajectory

For well-defined tasks, you can specify the optimal set of tool calls.

```python
test_case = {
    "input": "What is the weather in London and convert it to Fahrenheit?",
    "optimal_tools_in_order": ["get_weather"],   # only one tool needed
    "allowed_tools": ["get_weather", "get_forecast", "search_web"],
}

def evaluate_trajectory(actual_tool_calls, optimal_tools):
    # Check no wrong tools were used
    wrong_tools = [t for t in actual_tool_calls if t not in optimal_tools]

    # Check efficiency (did not use too many steps)
    efficiency = len(optimal_tools) / len(actual_tool_calls)

    # Check order if it matters
    order_correct = actual_tool_calls[:len(optimal_tools)] == optimal_tools

    return {
        "wrong_tools":     wrong_tools,
        "efficiency":      efficiency,
        "order_correct":   order_correct
    }
```

### Checking for Known Bad Patterns

Some trajectories are always wrong regardless of the final outcome.

```python
def check_bad_patterns(tool_call_sequence):
    issues = []

    # Check for repetition
    for i in range(len(tool_call_sequence) - 2):
        if (tool_call_sequence[i] == tool_call_sequence[i+1] ==
                tool_call_sequence[i+2]):
            issues.append("Repeated the same tool call 3 times in a row")

    # Check for unnecessary web search when internal docs should be used
    for call in tool_call_sequence:
        if call["tool"] == "search_web" and "policy" in call["args"].get("query", "").lower():
            issues.append("Used web search for an internal policy question")

    return issues
```

---

## 6. LLM-as-Judge for Agents

When optimal trajectories are hard to define, use an LLM to evaluate the full execution trace.

```python
judge_prompt = """
You are evaluating an AI agent's performance on a task.

Task given to the agent:
{task}

Agent's execution trace (all steps taken):
{execution_trace}

Final answer produced:
{final_answer}

Evaluate the agent on three dimensions:

1. Task completion (0 to 10): Did the agent complete the task correctly?
   - 10: Perfect completion
   - 7: Mostly correct with minor issues
   - 4: Partially completed
   - 0: Failed or produced wrong answer

2. Tool usage (0 to 10): Did the agent use the right tools efficiently?
   - 10: Exactly the right tools in the right order with no waste
   - 7: Right tools but some unnecessary steps
   - 4: Used some wrong tools but recovered
   - 0: Wrong tools throughout

3. Error handling (0 to 10): If errors occurred, did the agent handle them well?
   - 10: Handled all errors gracefully and recovered
   - N/A: No errors occurred
   - 0: Crashed or gave up when it should have recovered

Provide scores and a brief explanation for each dimension.
"""

evaluation = judge_llm.invoke(judge_prompt.format(
    task=test_case["input"],
    execution_trace=format_trace(agent_run.steps),
    final_answer=agent_run.output
))
```

LLM-as-judge works well for open-ended tasks where there is no single correct trajectory. It is flexible but subjective. Use multiple judge runs and average the scores for important evaluations.

---

## 7. Building an Agent Eval Dataset

### What Each Test Case Should Include

```python
eval_case = {
    # The input
    "task":                  "Find the HR policy on remote work and summarize it",

    # Expected outcome
    "expected_facts_in_answer": ["3 days per week maximum", "manager approval required"],
    "expected_outcome_type":    "text_answer",   # or: "side_effect", "refusal"

    # Expected trajectory
    "expected_tools":           ["search_internal_docs"],
    "tools_that_should_not_be_used": ["search_web"],
    "max_iterations":           5,

    # Budget constraints
    "max_cost_usd":             0.10,
    "max_duration_sec":         30,

    # Classification
    "difficulty":               "easy",   # easy, medium, hard
    "category":                 "hr",
    "should_succeed":           True,
}
```

### Sources for Test Cases

Manual curation by domain experts. This is the highest quality source. Have the people who know the knowledge base write questions and expected answers.

Production traffic with user feedback. Queries where users gave positive feedback become positive test cases. Queries with negative feedback or escalations become failure cases to test recovery from.

LLM-generated cases reviewed by humans. Generate candidate questions from your document corpus, have humans review and filter them.

Edge cases and adversarial inputs. Tasks that should fail gracefully. Ambiguous requests. Requests that require tools the agent should not have. These test reliability and refusal behavior.

### Dataset Size and Coverage

| Stage | Minimum cases |
|---|---|
| Initial development | 50 to 100 |
| Pre-launch | 200 to 500 |
| Production monitoring | 500 to 1000 |

Ensure coverage across:
- All major task categories (HR, IT, finance, etc.)
- Different difficulty levels
- Tasks that should succeed
- Tasks that should fail gracefully
- Tasks that require error recovery
- Tasks that test cost and iteration limits

---

## 8. Continuous Evaluation

Evaluation is not a one-time event before launch. Run it continuously.

### On Every Deployment

Run the full eval set against any change to the agent before it goes to production. Block deployment if key metrics drop below thresholds.

```python
deployment_gates = {
    "task_success_rate":    0.80,   # fail if below 80 percent
    "tool_call_accuracy":   0.90,   # fail if below 90 percent
    "hallucination_rate":   0.02,   # fail if above 2 percent
}
```

### Regression Testing

Keep a regression set of cases that previously failed and were fixed. Every deployment must pass all regression cases. A fix that causes a regression in a previously working case should block deployment.

### Production Monitoring

Track live metrics from real agent runs:
- Task success rate by category
- Average iterations per task
- Cost per task by task type
- Failure mode distribution (loops, errors, context issues, etc.)
- Human intervention rate

Alert when any metric moves significantly from its baseline.

---

## 9. Key Takeaways

- Agent evaluation has two independent dimensions: outcome (did it do the right thing?) and trajectory (did it take a good path?).
- A correct outcome with a bad trajectory is still a partial failure. An optimal trajectory that reaches a wrong answer due to a bad tool result is not the agent's fault but still counts against success rate.
- Task success rate, tool call accuracy, and trajectory efficiency are the three core metrics to track.
- For structured tasks, check outcomes programmatically against defined expected states. For open-ended tasks, use LLM-as-judge on the full execution trace.
- Build an eval dataset before launch with coverage across all task categories, difficulty levels, and both success and failure cases.
- Run evaluation on every deployment as a gate. Track production metrics continuously. Maintain a regression set of previously failing cases.


