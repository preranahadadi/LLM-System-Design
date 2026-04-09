# Module 06f -- Evaluation for Agents
> Evaluation and Observability Series · Why agent eval is different, outcome vs trajectory, key metrics, eval datasets

---

## Table of Contents
1. [Why Agent Evaluation Is Different](#1-why-agent-evaluation-is-different)
2. [Two Dimensions of Agent Evaluation](#2-two-dimensions-of-agent-evaluation)
3. [Key Metrics](#3-key-metrics)
4. [LLM-as-Judge for Trajectory Evaluation](#4-llm-as-judge-for-trajectory-evaluation)
5. [Building an Agent Eval Dataset](#5-building-an-agent-eval-dataset)
6. [Continuous Evaluation for Agents](#6-continuous-evaluation-for-agents)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Why Agent Evaluation Is Different

Evaluating a single LLM call is relatively straightforward. You have an input and an output. You check whether the output is correct. Pass or fail.

Evaluating an agent is fundamentally more complex for three reasons that make it a qualitatively different problem.

The output is not just text. When an agent completes a task, the result is not only what it said. It is also what it did. If the agent was supposed to book a meeting and it said "I have booked your meeting with Alice for tomorrow at 3pm" but no calendar event was actually created, that is a failure even though the text response sounds correct. Side effects -- the actual changes made to systems in the world -- are part of what you evaluate.

The path matters as much as the destination. An agent that produces the correct final answer after 20 unnecessary steps and tool calls is worse than an agent that produces the same answer in 3 steps. Both arrive at the same destination, but one wastes significant time and money doing it. In traditional software evaluation, the path taken through the code is usually not measured. In agent evaluation, it is a primary metric.

Non-determinism is amplified across many steps. A single LLM call might produce slightly different outputs on different runs. An agent that makes 10 LLM calls compounds this variability. Two runs of the same task can take completely different sequences of tool calls, encounter different intermediate results, and sometimes reach different final answers. This makes it harder to define a single "correct" execution and easier for agents to fail in unexpected ways.

---

## 2. Two Dimensions of Agent Evaluation

Because of these unique characteristics, agent evaluation needs to measure two independent things: whether the agent completed the task correctly, and whether it took a reasonable path to get there.

### Outcome Evaluation

Outcome evaluation asks: did the agent accomplish what it was supposed to?

For tasks with text answers, you check whether the final answer contains the expected information. A task asking the agent to summarize the remote work policy should produce a response that mentions the key rules.

For tasks with side effects, you check whether the expected changes actually happened. A task asking the agent to create a support ticket should result in an actual ticket being created in the ticketing system, with the correct fields filled in.

For all tasks, you check whether the agent completed within the allowed constraints. Did it stay within the iteration limit? Did it stay within the cost budget? Did it complete within the time limit? An agent that eventually produces the right answer after 50 iterations and $10 in API costs has not really succeeded if your limits are 10 iterations and $1.

It is possible for an agent to produce the correct outcome through a terrible path, and it is possible for an agent to take an excellent path and still fail due to factors outside its control (an external API returning incorrect data, for example). Measuring only outcome misses these cases.

### Trajectory Evaluation

Trajectory evaluation asks: did the agent take a good path?

You look at the sequence of tool calls the agent made and ask whether those were the right tools used in the right order. Did the agent use internal document search for questions about internal policies rather than going to web search? Did it avoid making redundant calls to the same tool with the same query? Did it handle errors gracefully and try an alternative approach rather than getting stuck?

Trajectory evaluation also looks at efficiency. If a task requires two pieces of information and the agent makes three tool calls to get them instead of two, that is slightly inefficient but acceptable. If the agent makes twelve tool calls when two would suffice, that is a significant efficiency failure.

The two dimensions are independent. A good outcome via a bad trajectory is still a partial failure. A good trajectory with a bad outcome due to external factors is not the agent's fault but still counts against success rate. You need both measurements to fully understand how an agent is performing.

---

## 3. Key Metrics

### Task Success Rate

The percentage of tasks where the agent completes correctly with both the outcome and key trajectory properties meeting your criteria. This is the primary headline metric for agent performance.

A reasonable target for most production agents is above 80 percent. Below this, the agent is failing too often to be reliably useful. The appropriate target depends heavily on the stakes involved -- a research assistant that sometimes misses context has a different acceptable failure rate than a system that creates records in a billing system.

### Tool Call Accuracy

The percentage of individual tool calls where the agent selected the right tool and provided valid, appropriate arguments. A high task success rate can coexist with moderate tool call accuracy if the agent manages to reach correct outcomes despite some wrong tool choices. But low tool call accuracy is a warning sign that the agent does not have a reliable understanding of when to use which tool.

### Trajectory Efficiency

Measured as the minimum number of steps needed to complete the task divided by the actual number of steps the agent took. A score of 1.0 means the agent took the optimal path. A score of 0.5 means it took twice as many steps as necessary. A reasonable target is above 0.7, meaning the agent takes at most about 40 percent more steps than the theoretical minimum.

### Cost Per Task

The average total cost of completing one task, counting all LLM calls, API calls, and any other costs incurred during execution. Track this over time and by task type. A sudden increase in cost per task without a corresponding improvement in quality indicates something changed that is causing the agent to do more work unnecessarily.

### Failure Mode Distribution

Break down failed tasks by the reason they failed. Did the agent hit the iteration limit without completing? Did a tool call error cause it to give up? Did it run out of context window? Did it produce an incorrect final answer despite completing the execution?

This breakdown tells you where to focus improvement efforts. If 60 percent of failures are hitting the iteration limit, you need to either increase the limit, improve planning efficiency, or investigate what is causing the agent to loop. If 60 percent are tool errors, you need to improve error handling or fix the tools themselves.

### Human Intervention Rate

The percentage of tasks that required a human-in-the-loop pause or correction. In systems that support human-in-the-loop, this measures how often the agent reaches a decision point where it needs human judgment. Minimize this metric while ensuring the agent still pauses appropriately for genuinely high-stakes decisions.

---

## 4. LLM-as-Judge for Trajectory Evaluation

Defining the "correct" trajectory for a task ahead of time is often difficult. Many tasks can be completed through multiple valid sequences of tool calls. A judge that penalizes any deviation from one specific expected sequence would unfairly score valid alternative approaches as failures.

LLM-as-judge is a practical solution for trajectory evaluation when the expected path is not fully specified. You give the judge the task, the full execution trace showing every step the agent took, and the final answer. You ask the judge to evaluate whether the agent's approach was reasonable and efficient.

This lets you evaluate dimensions like whether the agent used sensible tools for each step, whether it wasted effort with unnecessary repetition, whether it recovered well from errors, and whether the overall sequence made sense for the task.

The judge can score these dimensions and provide written explanations for its scores, which are useful for understanding systematic patterns in agent failures.

The same limitations that apply to LLM-as-judge for response evaluation apply here. The judge has preferences and biases. Run multiple evaluations of the same trace and average the scores for important decisions. Spot-check a sample of judge evaluations to verify they align with your own judgment.

---

## 5. Building an Agent Eval Dataset

Agent eval test cases need more information than simple LLM eval cases because you are evaluating the path and the outcome separately.

### What Each Test Case Should Specify

The task itself: a clear, specific description of what the agent should accomplish. Vague tasks produce vague evaluations.

Expected outcome properties: what facts or changes should be present in the final result. For text answers, list the key facts that should appear. For side-effect tasks, describe what state changes should have occurred.

Expected and forbidden tools: which tools the agent should use for this task, and which tools it should not use. A task about internal company policies should use internal search tools, not web search.

Budget constraints: the maximum number of iterations, maximum cost, and maximum time allowed for this specific task. Setting these per task allows you to have stricter limits for simpler tasks and more generous limits for complex ones.

Whether the task should succeed: some eval cases test that the agent handles out-of-scope or impossible requests gracefully. These should produce a polite deflection, not a hallucinated answer.

### Types of Tasks to Include

Include a mix of tasks that reflect the full range of challenges your agent will face.

Tasks that should succeed cleanly: representative of normal usage, well within the agent's capabilities, expected to complete with good tool choices and reasonable efficiency.

Tasks that test error recovery: a tool call will fail partway through, and the agent needs to try an alternative approach rather than giving up or looping.

Tasks that test scope boundaries: out of scope, ambiguous, or impossible requests that the agent should decline gracefully rather than attempting and hallucinating.

Tasks that test budget compliance: tasks that could tempt the agent toward an inefficient path, verifying it completes within iteration and cost limits.

Tasks from real production failures: every task where your agent failed in production and you understand why should become a regression test case.

### Coverage Requirements

Your eval dataset should cover all major task categories your agent handles, different difficulty levels within each category, and a range of tool combinations. If your agent has five tools, include tasks that exercise each tool individually and tasks that require combinations of tools.

A dataset that only includes easy tasks will give you high scores that do not reflect real-world performance on challenging tasks. A dataset that only includes hard tasks will give you low scores that may be overly pessimistic.

---

## 6. Continuous Evaluation for Agents

### Evaluation as a Deployment Gate

Run your full agent eval set before every deployment. A new version should meet or exceed the current version's performance on the full eval set and must not regress on any previously passing regression cases.

For agents specifically, pay attention to efficiency metrics alongside success rate. A version that improves success rate but doubles cost per task may not be an improvement in practice.

### Production Monitoring

Track your key agent metrics in production continuously: task success rate, average iterations per task, cost per task, and failure mode distribution. Dashboard these with historical baselines so you can see when they shift.

Set up alerts for the conditions that need immediate attention. If cost per task suddenly doubles, that may indicate a loop bug introduced by a recent change. If task success rate drops significantly over a few hours, there may be an issue with a tool or external dependency.

### Adding Production Failures to the Eval Dataset

When an agent fails on a real user task in production and you understand what went wrong, add that task to your eval dataset. This is the most direct way to ensure your eval set stays aligned with the actual challenges your agent faces in the real world.

Over time, a dataset built from real production failures gives you much stronger assurance that your agent handles the kinds of tasks real users actually bring to it, rather than only the idealized tasks you thought of when building the system initially.

---

## 7. Key Takeaways

- Agent evaluation is more complex than LLM eval because the output includes side effects, the path matters as much as the destination, and non-determinism is amplified across many steps.
- Agent evaluation has two independent dimensions: outcome evaluation (did the task complete correctly?) and trajectory evaluation (did the agent take a reasonable path?). A correct outcome via a terrible path is still a partial failure.
- The key metrics are task success rate, tool call accuracy, trajectory efficiency, cost per task, failure mode distribution, and human intervention rate.
- When the expected trajectory cannot be fully specified in advance, use LLM-as-judge to evaluate whether the agent's approach was reasonable, efficient, and recovered well from errors.
- Agent eval test cases need to specify the task, expected outcome properties, expected and forbidden tools, budget constraints, and whether the task should succeed or be gracefully declined.
- Include tasks covering normal usage, error recovery, scope boundaries, budget compliance, and real production failures. Run the full eval set before every deployment and monitor production metrics continuously.

---

