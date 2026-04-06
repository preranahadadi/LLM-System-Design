# Module 03b -- Tools and Function Calling
> Agents and Orchestration Series · What tools are, function calling API, tool descriptions, parallel calls, tool routing

---

## Table of Contents
1. [What is a Tool](#1-what-is-a-tool)
2. [How Function Calling Works](#2-how-function-calling-works)
3. [Defining Tools as Schemas](#3-defining-tools-as-schemas)
4. [Tool Description Quality](#4-tool-description-quality)
5. [Parallel Tool Calls](#5-parallel-tool-calls)
6. [Tool Routing](#6-tool-routing)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. What is a Tool

A tool is any function the agent can call to interact with the world outside the LLM. The LLM cannot browse the web, query a database, or send an email on its own. Tools give it those capabilities.

A tool can be:
- A Python function you write
- An API call to an external service
- A database query
- A web search
- A code executor that runs Python or SQL
- Another LLM call for a specialized subtask
- A file reader or writer

The important thing to understand: the LLM does not execute the tool. It outputs a structured description of what tool to call and with what arguments. Your application code reads that, runs the actual function, and feeds the result back to the LLM.

```
LLM says:    I want to call get_weather with city="London"
Your code:   calls get_weather("London"), gets "18 degrees, cloudy"
LLM receives: "18 degrees, cloudy" as an observation
LLM says:    The weather in London is 18 degrees and cloudy.
```

The LLM is the decision maker. Your code is the executor.

---

## 2. How Function Calling Works

The full flow of a single tool call:

```
Step 1: You send the user query and a list of available tools to the LLM.

Step 2: The LLM reads the query and the tool descriptions.
        It decides whether a tool is needed and which one.

Step 3: The LLM returns a structured tool call response instead of a text answer.
        This includes the tool name and the arguments it wants to pass.

Step 4: Your code reads the tool call, executes the function with those arguments,
        and gets a result.

Step 5: You append the tool call and its result to the conversation history.

Step 6: You call the LLM again with the updated history.

Step 7: The LLM reads the tool result and either calls another tool or
        generates the final answer.
```

This is why agents are loops. Each tool call requires going back to the LLM.

### What the LLM Response Looks Like

When the LLM decides to call a tool, the response is not a text answer. It is a structured object.

```python
# LLM response when it wants to call a tool
{
    "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "get_stock_price",
                "arguments": '{"ticker": "AAPL", "date": "2026-04-05"}'
            }
        }
    ]
}
```

Your code reads this, calls `get_stock_price(ticker="AAPL", date="2026-04-05")`, and appends the result back into the conversation before calling the LLM again.

---

## 3. Defining Tools as Schemas

Tools are defined as JSON schemas that describe the function name, what it does, and what arguments it takes. The quality of this schema directly affects whether the LLM calls the tool correctly.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_stock_price",
            "description": "Get the current or historical stock price for a publicly traded company using its ticker symbol. Use this when the user asks about stock prices, market values, or share prices.",
            "parameters": {
                "type": "object",
                "properties": {
                    "ticker": {
                        "type": "string",
                        "description": "The stock ticker symbol in uppercase. For example: AAPL for Apple, GOOGL for Google, MSFT for Microsoft."
                    },
                    "date": {
                        "type": "string",
                        "description": "Optional. The date to get the price for, in YYYY-MM-DD format. If not provided, returns today's price."
                    }
                },
                "required": ["ticker"]
            }
        }
    }
]
```

Calling the LLM with tools:

```python
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": "What is Apple's stock price today?"}
    ],
    tools=tools,
    tool_choice="auto"   # LLM decides whether to call a tool or not
)
```

After the LLM returns a tool call, execute it and loop back:

```python
# Execute the tool
tool_call = response.choices[0].message.tool_calls[0]
tool_name = tool_call.function.name
tool_args = json.loads(tool_call.function.arguments)

result = execute_tool(tool_name, tool_args)

# Append tool call and result to history
messages.append(response.choices[0].message)        # LLM's tool call message
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": str(result)
})

# Call LLM again with updated history
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools
)
```

---

## 4. Tool Description Quality

The LLM picks which tool to use based entirely on the tool descriptions. A vague description leads to wrong tool selection or the LLM inventing arguments that do not match the schema.

### Weak Description

```python
{
    "name": "search",
    "description": "Search for things."
}
```

The LLM does not know what kind of search this is, when to use it, or what it returns.

### Strong Description

```python
{
    "name": "search_internal_docs",
    "description": "Search the company internal knowledge base including HR policies, IT documentation, legal guidelines, and product specs. Use this when the user asks about company-specific information, internal processes, or policies. Do not use this for general knowledge questions or real-time data. Returns the top 3 most relevant document excerpts with the source document name and page number."
}
```

This tells the LLM:
- What data source it searches
- What kinds of questions it is good for
- What questions it is not good for
- What the output looks like

### Rules for Good Tool Descriptions

- Say what the tool searches or operates on, not just what it does generically
- Say when to use it and when not to use it
- Describe what the return value looks like
- For arguments, include examples of valid values
- If two tools are similar, explain the difference between them

---

## 5. Parallel Tool Calls

When multiple tool calls are independent of each other, modern LLMs can call them all in a single turn instead of sequentially.

Sequential (slow):
```
Turn 1: LLM calls get_stock_price("AAPL") --> waits --> gets result
Turn 2: LLM calls get_stock_price("GOOGL") --> waits --> gets result
Turn 3: LLM calls get_stock_price("MSFT") --> waits --> gets result
Turn 4: LLM synthesizes answer

Total: 4 LLM calls
```

Parallel (fast):
```
Turn 1: LLM calls get_stock_price("AAPL"), get_stock_price("GOOGL"),
        get_stock_price("MSFT") all at once
        Your code executes all three in parallel
Turn 2: LLM receives all three results and synthesizes answer

Total: 2 LLM calls
```

Parallel tool calls reduce latency significantly for tasks that involve collecting data from multiple independent sources. GPT-4o and Claude 3.5 Sonnet both support this natively.

When to use parallel: when the tools do not depend on each other's results. If tool B needs the output of tool A to know what to search for, they must be sequential.

---

## 6. Tool Routing

When you have many tools (10 or more), passing all of them to the LLM creates two problems. The context gets long and expensive, and the LLM has more difficulty choosing the right tool from a crowded list.

The solution is tool retrieval: embed the user query and find the most relevant tools by semantic similarity. Only pass those tools to the LLM.

```
User query: "What is the weather in London?"

All tools: [get_weather, search_docs, query_database, send_email, create_ticket,
            get_stock_price, run_sql, search_web, get_calendar, ...]

Tool retrieval: embed the query, find top 3 most similar tool descriptions
Matched tools:  [get_weather, get_forecast]

Pass to LLM: only get_weather and get_forecast
LLM picks:   get_weather
```

This works because tool descriptions are text that can be embedded and compared to the query embedding.

### When Tool Routing Matters

Under 5 tools: pass all of them, no routing needed.
5 to 10 tools: optional, LLMs handle this range reasonably well.
More than 10 tools: routing is recommended.
More than 20 tools: routing is essentially required.

---

## 7. Key Takeaways

- A tool is any function the agent can call. The LLM decides to call it. Your code executes it. The result goes back to the LLM.
- Function calling works by having the LLM return a structured tool call object instead of a text answer. Your code runs the function and feeds the result back.
- Tool descriptions are the most important thing to get right. The LLM picks tools based entirely on what the description says.
- Parallel tool calls are more efficient than sequential ones when tools are independent. Always prefer them.
- With many tools, use tool retrieval to pre-filter which tools the LLM sees for a given query.
