# Module 03d -- Planning and Reasoning
> Agents and Orchestration Series · ReAct, Chain-of-Thought, Plan-and-Execute, Tree of Thought, Self-Reflection

---

## Table of Contents
1. [Why Planning Matters](#1-why-planning-matters)
2. [ReAct](#2-react)
3. [Chain-of-Thought](#3-chain-of-thought)
4. [Plan-and-Execute](#4-plan-and-execute)
5. [Tree of Thought](#5-tree-of-thought)
6. [Self-Reflection](#6-self-reflection)
7. [Choosing the Right Strategy](#7-choosing-the-right-strategy)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Why Planning Matters

LLMs jump to answers. Given a question, the model produces whatever completion has the highest probability. For simple questions this works fine. For complex tasks that require multiple steps, gathering information from multiple sources, or making decisions based on earlier results, jumping straight to an answer produces wrong or shallow outputs.

Planning strategies give the LLM structured ways to slow down and think before acting. Every technique in this module is essentially a different answer to the same question: how do we get the LLM to reason carefully before generating a final response?

---

## 2. ReAct

ReAct is covered in detail in Module 03a. The short version:

ReAct is the standard planning pattern for agents that use tools. The LLM alternates between writing a Thought (reasoning about what to do next) and an Action (calling a tool). The result of the tool call is an Observation that feeds into the next Thought.

```
Thought:      What do I need to find out?
Action:       Call a tool
Observation:  What did the tool return?
Thought:      What does this mean? What do I do next?
Action:       Call another tool or answer
...
Final Answer: The answer to the user's question
```

ReAct works because the Thought step forces the LLM to articulate its reasoning before acting. This catches mistakes early and makes the agent's behavior interpretable.

---

## 3. Chain-of-Thought

Chain-of-Thought is not a loop. It is a prompting technique that gets the LLM to show its reasoning in a single pass before producing a final answer.

You prompt the model to think step by step, and it produces a sequence of intermediate reasoning steps before the answer.

### Without Chain-of-Thought

```
Question: A store sells apples for 1.20 each and oranges for 0.80 each.
          If you buy 5 apples and 3 oranges, what is the total?

Answer: 8.40
```

The model often gets this wrong by pattern matching rather than actually computing.

### With Chain-of-Thought

```
Question: A store sells apples for 1.20 each and oranges for 0.80 each.
          If you buy 5 apples and 3 oranges, what is the total?
          Let's think step by step.

Answer: 5 apples at 1.20 each is 5 times 1.20 which is 6.00.
        3 oranges at 0.80 each is 3 times 0.80 which is 2.40.
        Total is 6.00 plus 2.40 which is 8.40.
```

Just adding "let's think step by step" to the prompt substantially improves accuracy on multi-step reasoning tasks.

### When to Use Chain-of-Thought

- Math problems
- Logic puzzles
- Multi-step reasoning where each step depends on the previous
- Questions where showing work is valuable for the user to verify
- Any task where you find the model is making mistakes by jumping to conclusions

### Few-Shot Chain-of-Thought

You can include examples of step-by-step reasoning in your prompt to teach the model the format you want. This is especially useful for domain-specific reasoning patterns.

```
Example:
Q: [example question]
A: First I will check X. X is 5. Then I will compute Y using X. Y is 5 times 3 which is 15.
   Therefore the answer is 15.

Now:
Q: [actual question]
A: [model follows the same pattern]
```

---

## 4. Plan-and-Execute

Plan-and-Execute splits the task into two distinct phases. A planner produces a complete list of steps upfront. An executor carries out each step one by one.

### Phase 1: Planning

A capable LLM (often GPT-4o or similar) reads the high-level task and produces a concrete plan.

```
Task: "Research our top 3 competitors and write a comparison report."

Plan produced by the planner:
  Step 1: Search the web for the top 3 competitors in our market
  Step 2: For each competitor, search for their pricing page
  Step 3: For each competitor, search for their main feature list
  Step 4: Search internal docs for our own pricing and features
  Step 5: Create a comparison table with pricing and features
  Step 6: Write an executive summary highlighting our strengths and gaps
```

### Phase 2: Execution

A smaller, cheaper model (or the same model with simpler prompting) executes each step in order, using the tools it needs.

```
Executor receives: Step 1: Search the web for the top 3 competitors
Executor calls:    search_web("top competitors in [our market] 2026")
Executor returns:  "Competitor A, Competitor B, Competitor C"

Executor receives: Step 2: For each competitor, search for their pricing page
Executor calls:    search_web("Competitor A pricing")
                   search_web("Competitor B pricing")
                   search_web("Competitor C pricing")
...
```

### Why This Is Better Than Straight ReAct for Complex Tasks

ReAct does not produce a plan. It just figures out the next step at each iteration. For a complex 10-step task, this means the agent might take wrong turns, need to backtrack, or forget the overall goal by step 7.

Plan-and-Execute ensures the overall strategy is right before any execution begins. The executor just needs to follow steps, which is simpler.

### Advantages

- You can validate the plan before executing it. A human can review the steps and approve or modify them before any tools are called.
- The executor can be a cheaper, smaller model since each step is simple and scoped.
- If a step fails, you can re-plan from that point rather than starting over.
- The plan gives you a clear audit trail of what the agent intended to do.

### Disadvantages

- The plan is created without seeing actual tool results. If step 2 reveals something that changes what step 3 should be, the plan may need to be revised.
- Two LLM calls are needed just to get started (planning + first execution step).
- Not suitable for tasks that are highly dynamic where the right next step is completely dependent on previous results.

---

## 5. Tree of Thought

Tree of Thought extends Chain-of-Thought by exploring multiple reasoning paths at the same time rather than committing to a single chain of thinking.

The LLM generates several candidate next thoughts at each step, evaluates which ones are most promising, continues only those, and prunes the rest. It is a search over possible reasoning paths.

### How It Works

```
Problem
  |
  |--> Thought branch A  -->  evaluate (promising / not promising)
  |         |
  |         |--> Sub-branch A1 --> evaluate
  |         |--> Sub-branch A2 --> evaluate (pruned, dead end)
  |
  |--> Thought branch B  -->  evaluate (pruned, weak)
  |
  |--> Thought branch C  -->  evaluate (promising)
            |
            |--> Sub-branch C1 --> Answer
```

At each node, the LLM generates two or three candidate continuations. It also evaluates each one with a score or a judgment of "promising, keep going" or "not promising, stop here". The search continues on the promising branches.

### When Tree of Thought Helps

Tree of Thought is expensive because it makes many LLM calls per step. It is only worth it for tasks where:

- A single chain of reasoning often leads to wrong answers
- Exploring alternative approaches is important
- The task has a clear success condition you can evaluate partway through
- Correctness matters more than cost or speed

Good examples: complex math proofs, programming challenges, strategic planning problems, creative writing with specific constraints.

Bad examples: simple Q&A, document summarization, standard RAG queries. The overhead is not justified.

---

## 6. Self-Reflection

Self-Reflection is a loop where the LLM generates an answer and then critiques its own output, then revises based on the critique, and repeats until the critique finds no significant issues.

### The Pattern

```
Step 1: Generate an initial answer or plan.
Step 2: Critique it. What is wrong, incomplete, or risky?
Step 3: Revise based on the critique.
Step 4: Critique the revision.
Step 5: Repeat until the critique is satisfied, or until a maximum number of iterations.
```

### Example in Code Generation

```
Step 1 - Generate:
  "Here is a function that sorts a list and removes duplicates: [code]"

Step 2 - Critique:
  "Issues: the function modifies the input list in place which is unexpected.
   It also does not handle the case where the input is None.
   The sorting is O(n^2) when it could be O(n log n)."

Step 3 - Revise:
  "Revised function that copies the input, handles None, and uses sorted(): [new code]"

Step 4 - Critique:
  "The revised function looks correct. No significant issues found."

Done.
```

### When to Use Self-Reflection

- Code generation tasks where you can automatically run the code and check if it passes tests
- Content generation where quality standards are defined and checkable
- Planning tasks where you want to catch obvious flaws before execution begins
- Any task where a single generation pass is not reliable enough

### Self-Reflection with Automated Checks

The most powerful version is when the critique step is not another LLM call but an actual test or validation.

```python
def code_generation_loop(task, max_iterations=3):
    code = generate_code(task)

    for i in range(max_iterations):
        test_result = run_tests(code)

        if test_result.passed:
            return code

        # LLM fixes the code using the test failure as feedback
        code = fix_code(code, test_result.error_message)

    return code   # return best attempt after max iterations
```

The test failure is objective feedback. The LLM does not need to judge whether its own code is correct. The tests do that.

---

## 7. Choosing the Right Strategy

| Task type | Best strategy |
|---|---|
| Single-step questions | No planning needed, direct answer |
| Multi-step with tools | ReAct |
| Complex reasoning, no tools | Chain-of-Thought |
| Long multi-step tasks with a clear structure | Plan-and-Execute |
| Hard problems where first attempts often fail | Tree of Thought |
| Code generation, plan validation | Self-Reflection |

These strategies can be combined. For example: Plan-and-Execute where each execution step uses ReAct, and the overall plan is validated with Self-Reflection before execution begins.

The cost goes up as you move down the list. ReAct is cheap. Tree of Thought is expensive. Use the minimum strategy that solves your problem.

---

## 8. Key Takeaways

- ReAct is the standard for tool-using agents. Thought-Action-Observation loop.
- Chain-of-Thought prompts the LLM to show its reasoning step by step in a single pass. Adding "think step by step" to a prompt substantially improves accuracy on reasoning tasks.
- Plan-and-Execute produces a complete plan upfront before any execution. Good for long structured tasks where you want to validate the strategy before starting.
- Tree of Thought explores multiple reasoning paths simultaneously. High quality, high cost. Use for hard problems where single-path reasoning often fails.
- Self-Reflection generates, critiques, and revises in a loop. Most powerful when the critique step is an automated test rather than another LLM call.


