# Module 03e -- Multi-Agent Systems
> Agents and Orchestration Series · Orchestrator-worker, supervisor, handoffs, agent communication

---

## Table of Contents
1. [Why Multiple Agents](#1-why-multiple-agents)
2. [The Orchestrator-Worker Pattern](#2-the-orchestrator-worker-pattern)
3. [The Supervisor Pattern](#3-the-supervisor-pattern)
4. [Handoffs](#4-handoffs)
5. [Agent Communication](#5-agent-communication)
6. [Parallelism in Multi-Agent Systems](#6-parallelism-in-multi-agent-systems)
7. [When Not to Use Multiple Agents](#7-when-not-to-use-multiple-agents)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Why Multiple Agents

A single agent trying to do everything runs into problems as tasks grow more complex.

**Tool overload.** If you give one agent 30 tools, it struggles to choose the right one for each step. The tool selection error rate goes up. The context window fills with tool descriptions.

**Context overflow.** A long complex task accumulates a lot of intermediate results. By the time you are on step 15, the context is enormous and earlier results start getting pushed out or diluted.

**No parallelism.** A single agent works sequentially. If a task has independent subtasks, a single agent must do them one after the other even when they could be done at the same time.

**Hard to debug.** When everything is in one agent, a failure anywhere in a long run is hard to isolate. You do not know which step failed or why.

**Cannot specialize.** Some subtasks need a powerful expensive model. Others need only a cheap fast one. With one agent, you use the same model for everything.

Multiple agents solve all of these by dividing responsibility.

---

## 2. The Orchestrator-Worker Pattern

This is the most common multi-agent architecture.

One orchestrator agent receives the high-level task. It breaks the task into subtasks and delegates each to a specialized worker agent. Workers report their results back to the orchestrator. The orchestrator assembles the final output.

```
                ORCHESTRATOR
                    |
       -------------|-------------
       |             |           |
  WORKER A       WORKER B    WORKER C
  (researcher)   (writer)    (fact-checker)
       |             |           |
  web search    draft text   validate claims
  internal docs format output flag issues
```

### Orchestrator Responsibilities

The orchestrator does not do detailed work itself. Its job is:
- Understand the high-level task
- Break it into concrete subtasks
- Assign each subtask to the right worker
- Track which subtasks are complete
- Handle failures (retry, reassign, abort)
- Assemble the final answer from worker outputs

The orchestrator is typically a capable model like GPT-4o because it needs to reason about task decomposition and handle unexpected situations.

### Worker Responsibilities

Each worker has a narrow scope and a specific set of tools. It receives a well-defined subtask, executes it, and returns a structured result.

Workers do not need to know about the bigger picture. They just handle their subtask. Because their job is simpler and narrower, you can often use a smaller cheaper model for workers.

### Example: Research Report

```
High-level task: "Write a competitive analysis report for our product launch."

Orchestrator plan:
  Subtask 1 -> Research Worker: "Find the top 5 competitors and their pricing"
  Subtask 2 -> Research Worker: "Find the top 5 competitors and their main features"
  Subtask 3 -> Research Worker: "Find recent news about each competitor"
  Subtask 4 -> Internal Search Worker: "Find our own pricing and roadmap docs"

After workers complete:
  Subtask 5 -> Writing Worker: "Write the comparison table from these results"
  Subtask 6 -> Writing Worker: "Write the executive summary from these results"

After writing:
  Subtask 7 -> Review Worker: "Check all facts in this report against sources"

Orchestrator assembles: final report
```

Workers for subtasks 1, 2, 3, and 4 can run in parallel. Worker for subtask 5, 6, and 7 must wait for previous steps. The orchestrator manages this dependency.

---

## 3. The Supervisor Pattern

The supervisor pattern is similar to orchestrator-worker but the supervisor actively monitors worker progress and can intervene mid-task.

In the orchestrator-worker pattern, the orchestrator assigns work and waits for completion. In the supervisor pattern, the supervisor can:
- Interrupt a worker that is stuck or going in the wrong direction
- Send a worker's output back for revision if quality is insufficient
- Reassign a task to a different worker if one fails
- Redirect the overall approach if early results reveal the plan needs to change

```
Supervisor loop:
  1. Assign subtask to worker
  2. Monitor worker progress
  3. If worker is stuck after N iterations -> interrupt, give hint or reassign
  4. If worker finishes -> evaluate output quality
  5. If quality is insufficient -> send back with feedback
  6. If quality is good -> move to next subtask
```

The supervisor pattern is better for tasks where output quality is hard to specify upfront and requires judgment during execution. The downside is complexity: you need the supervisor to be a capable model running in a monitoring loop, which adds latency and cost.

---

## 4. Handoffs

A handoff is when one agent passes control and context to another agent.

In a customer service system, the triage agent receives the user's message and routes to the appropriate specialist. Each routing decision is a handoff.

```
User message -> Triage Agent
                    |
    ----------------|------------------
    |                |                |
Billing Agent  Tech Support Agent  FAQ Agent
    |                |
 resolved      not resolved
                    |
            Escalation Agent
```

### What Gets Passed in a Handoff

The receiving agent needs everything the previous agent knew. A handoff message should include:

```python
handoff = {
    "from_agent":     "triage_agent",
    "to_agent":       "billing_agent",
    "task":           "Resolve billing dispute for order 12345",
    "context": {
        "user_id":         "u789",
        "conversation":    [...],      # full conversation history
        "retrieved_docs":  [...],      # any docs already fetched
        "steps_completed": [           # what has already been tried
            "Checked FAQ -- no relevant answer found"
        ]
    },
    "urgency":        "high",
    "deadline":       "2026-04-05T18:00:00Z"
}
```

If the handoff message is incomplete, the receiving agent has to redo work the previous agent already did. A good handoff message means no repeated steps.

### Handoff Criteria

The triage agent needs clear rules for when to hand off and to whom. These rules should be in the triage agent's system prompt.

```
If the user question is about an invoice, payment, subscription, or refund:
  hand off to billing_agent.

If the user question is about a technical error, API, or integration:
  hand off to tech_support_agent.

If the user question can be answered from the FAQ:
  answer directly without handing off.

If the issue is not resolved after two handoffs:
  hand off to escalation_agent.
```

---

## 5. Agent Communication

Agents communicate through structured messages, not freeform text. Freeform text is ambiguous and hard to parse programmatically.

### Message Format

Every message between agents should have a consistent structure.

```python
message = {
    "sender":    "research_agent",
    "receiver":  "orchestrator",
    "type":      "task_result",
    "task_id":   "subtask_001",
    "status":    "success",       # or: "failed", "partial", "needs_clarification"
    "result": {
        "data":    [...],          # the actual output
        "sources": [...]           # where the data came from
    },
    "metadata": {
        "duration_sec":   12.4,
        "llm_calls_made": 3,
        "cost_usd":       0.04
    },
    "error":     None             # or error details if status is "failed"
}
```

### Why Structure Matters

The orchestrator processes many worker messages programmatically. If each worker returned freeform text like "I found the pricing, it is 99 dollars per month for the basic plan", the orchestrator would need an LLM call just to extract the data. Structured output makes processing fast, reliable, and cheap.

Tell workers to return structured output in their system prompt:

```
Return your result as JSON with the following fields:
  status: "success" or "failed"
  data: your findings as a structured object
  sources: list of source URLs or document names you used
  summary: one sentence summary of what you found
```

---

## 6. Parallelism in Multi-Agent Systems

The main performance advantage of multi-agent systems is the ability to run independent subtasks in parallel.

### Identifying Independent Subtasks

Subtasks are independent when:
- They do not use each other's outputs as inputs
- They access different data sources
- The order does not matter

```
Research report example:
  Independent (run in parallel):
    - Search for competitor pricing
    - Search for competitor features
    - Search for recent competitor news
    - Search internal docs for our own specs

  Sequential (must wait for above):
    - Write comparison table (needs all competitor data)
    - Write executive summary (needs the comparison table)
    - Fact-check the report (needs the full draft)
```

### Running Agents in Parallel

```python
import asyncio

async def run_parallel_workers(tasks):
    workers = [research_agent.run(task) for task in independent_tasks]
    results = await asyncio.gather(*workers)
    return results

# All independent tasks run simultaneously
independent_results = asyncio.run(run_parallel_workers(independent_tasks))

# Then run sequential tasks with the results
comparison_table = writing_agent.run(
    task="Write comparison table",
    context=independent_results
)
```

---

## 7. When Not to Use Multiple Agents

Multi-agent systems are complex to build, debug, and maintain. Do not add them unless you have a clear reason.

Use multiple agents when:
- A single agent's tool list is too large and causing selection errors
- Tasks are long enough that context overflow is a real problem
- Independent subtasks exist that can genuinely run in parallel
- Different subtasks need different model capabilities or costs
- You need clear separation for debugging and reliability

Do not use multiple agents when:
- A single agent with a clear system prompt handles the task reliably
- The task is short and sequential
- The overhead of coordination adds more latency than parallelism saves
- The added complexity is not justified by the scale of the task

A well-designed single agent is almost always simpler to build and cheaper to run than a poorly motivated multi-agent system.

---

## 8. Key Takeaways

- Multiple agents solve the problems of tool overload, context overflow, lack of parallelism, and difficulty debugging that arise with single agents on complex tasks.
- The orchestrator-worker pattern is the most common structure. The orchestrator plans and delegates. Workers execute narrow subtasks.
- The supervisor pattern adds active monitoring and the ability to intervene mid-task.
- Handoffs pass full context between agents so the receiving agent does not repeat work.
- Agent communication should be structured, not freeform. JSON output from workers is easier to process than natural language.
- Parallelism is the main performance benefit of multi-agent systems. Identify which subtasks are independent and run them simultaneously.


