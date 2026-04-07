# Module 05c -- MCP Primitives
> Model Context Protocol Series · Tools, resources, prompts -- what each one is and when to use which

---

## Table of Contents
1. [Overview of Primitives](#1-overview-of-primitives)
2. [Tools](#2-tools)
3. [Resources](#3-resources)
4. [Prompts](#4-prompts)
5. [Resources vs Tools for Data Access](#5-resources-vs-tools-for-data-access)
6. [Sampling](#6-sampling)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Overview of Primitives

MCP defines three types of things that a server can expose to clients. These are called primitives. Each primitive serves a different purpose and is used in a different way by the host application.

| Primitive | What it is | Who initiates | Used for |
|---|---|---|---|
| Tool | A callable function | LLM or application | Actions, computation, live data |
| Resource | Addressable data | Host application | Documents, schemas, static content |
| Prompt | A reusable template | User or application | Standardized task prompts |

Most MCP servers expose tools. Resources and prompts are used less often but are important to understand.

---

## 2. Tools

Tools are functions that the LLM can call to take actions or retrieve computed results. They are the most commonly used primitive.

A tool has four things:
- A name that uniquely identifies it
- A description that tells the LLM what it does and when to use it
- An input schema that defines what arguments it accepts
- An implementation that runs when called and returns a result

### What Tools Are Good For

Tools are the right choice when:
- Accessing data requires parameters (a search query, a user ID, a date range)
- The result depends on live or frequently changing data
- You need to take an action (create a record, send a message, run a query)
- Computation is involved before returning a result

### Examples

```
search_codebase(query, file_extension)
  --> searches through files and returns matching snippets

run_sql_query(sql)
  --> executes a SELECT query and returns formatted results

create_github_issue(title, body, labels)
  --> creates an issue in a GitHub repository

get_weather(city, units)
  --> fetches current weather data for a location

send_slack_message(channel, message)
  --> sends a message to a Slack channel
```

### How the LLM Uses Tools

The host passes tool definitions to the LLM as part of the conversation. The LLM reads the name and description to understand what each tool does. When the LLM decides a tool is needed, it outputs a tool call with the tool name and arguments. The host forwards this to the MCP server, gets the result, and feeds it back to the LLM.

The description is the most important part of the tool definition. Write it clearly, say what the tool does, say when to use it, and say what it returns. A vague description leads to the LLM calling the wrong tool or calling it with wrong arguments.

```python
# Weak description -- LLM does not know when to use this
@mcp.tool()
def search(query: str) -> str:
    """Search for things."""
    ...

# Strong description -- LLM knows exactly what this is for
@mcp.tool()
def search_internal_docs(query: str, department: str = None) -> str:
    """
    Search the company internal knowledge base including HR policies,
    IT documentation, legal guidelines, and product specifications.
    Use this for questions about company-specific information and internal processes.
    Do not use this for general knowledge questions or real-time data.
    Optionally filter by department: HR, IT, Legal, Product.
    Returns the top 3 most relevant document excerpts with source and page number.
    """
    ...
```

---

## 3. Resources

Resources are pieces of data that the server makes available at a specific URI. Unlike tools, resources are not called with arguments -- they are read by URI, like reading a file by its path.

A resource has:
- A URI that uniquely identifies it
- A name and human-readable description
- A MIME type (text/plain, application/json, text/markdown, etc.)
- Content -- either text or binary data

### What Resources Are Good For

Resources are the right choice when:
- The data can be naturally identified by a location or identifier
- The content does not require computation or parameters to produce
- The data is relatively static or can be read as-is
- The host might want to proactively inject the content into context

### Examples

```
file:///project/README.md
  --> the README file of the current project

database://mydb/schema
  --> the full schema of a database

config://app/settings
  --> the application configuration

docs://api/endpoints
  --> the list of all API endpoints

git://repo/main/CHANGELOG
  --> the changelog of a git repository
```

### How the Host Uses Resources

The host can discover resources by asking the server to list them. It can then read specific resources by URI. The content gets injected into the LLM's context -- either automatically by the host, or in response to the LLM requesting a specific URI.

Resources are particularly useful for giving the LLM background knowledge that applies to the entire session. For example, a coding assistant might read the project's README and main configuration file at the start of every session and inject them into context so the LLM always has that context available.

### Static vs Dynamic Resources

Resources can be static (the same content every time) or dynamic (the server generates content when the resource is read).

A static resource might be a file on disk. A dynamic resource might be a database schema that the server queries and formats fresh each time it is read.

The difference is invisible to the client. It just reads the URI and receives content.

---

## 4. Prompts

Prompts are reusable message templates that the server exposes. They accept optional arguments and return a formatted message or set of messages that the host can use to start or continue a conversation.

A prompt has:
- A name
- A description
- Optional argument definitions
- An implementation that generates the message content

### What Prompts Are Good For

Prompts are useful when:
- You want to standardize how certain tasks are started across a team
- You have parameterized prompt templates that you want to maintain centrally
- You want server maintainers to be able to update prompt text without changing the host application
- You are building a tool like Claude Desktop where users can select from available prompt templates

### Example

```python
@mcp.prompt()
def code_review(language: str, focus: str = "all") -> str:
    """
    Generate a code review prompt for a specific language and focus area.
    language: the programming language being reviewed
    focus: what to focus on -- security, performance, readability, or all
    """
    return f"""
    Please review the following {language} code.

    Focus particularly on {focus} issues.

    For each issue you find:
    1. Identify where in the code the issue occurs
    2. Explain why it is a problem
    3. Provide a specific suggested fix
    """
```

When a user selects this prompt in Claude Desktop, they can provide the language and focus arguments, and the host uses the returned text as the starting message for the conversation.

### Prompts in Practice

Prompts are the least commonly used primitive. Most MCP servers only expose tools. Prompts are most relevant for applications like Claude Desktop that present a user interface where users can browse and select from available templates.

If you are building an MCP server for programmatic use (connecting to an agent or an application), you likely do not need to implement prompts. Focus on tools and resources.

---

## 5. Resources vs Tools for Data Access

A common question when designing an MCP server is whether to expose data as a resource or as a tool. The answer depends on how the data is accessed.

### Use a Resource When

The data has a natural identifier and can be read directly:

```
Read the contents of a specific file:
  --> resource: file:///path/to/file.py

Read the database schema:
  --> resource: database://mydb/schema

Read the application configuration:
  --> resource: config://app/settings
```

The data is relatively stable and the host might want to cache it or inject it proactively:

```
The project README does not change often.
The host can read it once at session start and keep it in context.
  --> better as a resource
```

### Use a Tool When

Accessing the data requires parameters:

```
Search across all files matching a query:
  --> tool: search_codebase(query="authentication logic")

Get records matching a filter:
  --> tool: query_database(table="users", filter="created_at > 2026-01-01")

Get the latest data that changes frequently:
  --> tool: get_current_stock_price(ticker="AAPL")
```

The operation involves computation or transformation:

```
Summarize a document:
  --> tool: summarize_document(document_id="xyz")

Run a query and format results:
  --> tool: run_sql(query="SELECT ...")
```

### Quick Decision Rule

If you can identify the data by a URI and read it without extra parameters, use a resource. If you need parameters to find or compute the data, use a tool.

---

## 6. Sampling

Sampling is an optional and advanced MCP capability that works in the opposite direction from normal: the server requests that the client run an LLM completion on its behalf.

Normally, the LLM calls tools through the MCP server. With sampling, the MCP server asks the LLM to generate something.

This lets MCP servers incorporate LLM reasoning into their tool implementations without needing direct API access to the LLM. The server sends a sampling request with a prompt, the host decides whether to approve it, runs the LLM call using its own credentials and context, and returns the result to the server.

### When Sampling Is Useful

A tool that needs to process natural language before doing something:

```
Server has a tool: organize_notes(notes)
Internally, the tool needs to categorize and structure the notes.
Instead of having its own LLM API key, the server requests a sampling call
from the client to do the categorization, then uses the result.
```

### Important Notes on Sampling

The host must explicitly support sampling. Most host applications do not implement it.

The host has full control over whether to approve sampling requests. It can reject them, modify the prompt before sending it, or apply content filtering to the result.

Sampling is rarely needed. Most MCP servers do their work without it. Only consider implementing sampling if your server genuinely needs LLM reasoning as part of its tool implementation and cannot have its own API access for some reason.

---

## 7. Key Takeaways

- MCP has three primitives: tools, resources, and prompts.
- Tools are callable functions. The LLM calls them with arguments to get computed results or take actions. They are the most commonly used primitive.
- Resources are addressable data identified by URIs. The host reads them and can inject their content into the LLM's context. Good for files, schemas, and other relatively static content.
- Prompts are reusable message templates with optional parameters. Least commonly used, most relevant for applications with user-facing template selection interfaces.
- The key decision between resource and tool: if you need parameters to find or compute the data, use a tool. If you can identify the data by a URI and read it directly, use a resource.
- Sampling lets servers request LLM completions from the client. It is an advanced feature and rarely needed.

---

