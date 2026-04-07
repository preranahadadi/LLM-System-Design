# Module 05e -- MCP in Agentic Systems
> Model Context Protocol Series · How agents use MCP, multi-server setups, MCP vs direct tool calling, security

---

## Table of Contents
1. [How Agents Use MCP](#1-how-agents-use-mcp)
2. [Multi-Server Setups](#2-multi-server-setups)
3. [MCP vs Direct Tool Calling](#3-mcp-vs-direct-tool-calling)
4. [Practical Patterns](#4-practical-patterns)
5. [Security in Production](#5-security-in-production)
6. [The MCP Ecosystem](#6-the-mcp-ecosystem)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. How Agents Use MCP

In an agentic system, MCP servers are the mechanism through which the agent accesses external capabilities. Instead of defining tools directly in application code and passing them as function schemas to the LLM, you connect to MCP servers and the agent uses whatever those servers expose.

From the LLM's perspective, nothing changes. It still sees tools described as JSON schemas and it still outputs structured tool calls. MCP is entirely invisible to the LLM. It is infrastructure that the host application manages.

### The Full Sequence

```
1. Host application starts up

2. Host connects to all configured MCP servers
   Each connection goes through the initialization handshake

3. Host asks each server to list its tools
   Aggregates all tools from all servers into one combined list

4. Host starts the agent loop and passes the combined tool list to the LLM

5. LLM processes the user's request and decides to call a tool
   Example: "I need to call search_internal_docs with query=vacation policy"

6. Host receives the tool call
   Looks up which MCP server handles search_internal_docs
   Forwards the call to that server via MCP protocol

7. MCP server executes the tool
   Returns the result to the host

8. Host formats the result and sends it back to the LLM as an observation

9. LLM reads the result and decides what to do next
   May call another tool or produce the final answer

10. Loop continues until the task is complete
```

The agent loop structure is the same as without MCP. The difference is that tools are discovered dynamically from servers rather than being hardcoded in the application.

---

## 2. Multi-Server Setups

A production agentic system typically connects to several MCP servers simultaneously. The host aggregates all their tools into one list for the LLM. From the LLM's perspective, there is just one pool of tools. It does not know or care which server each tool comes from.

### How the Host Manages Multiple Servers

```python
class AgentHost:
    def __init__(self):
        self.clients = {}          # server_name -> MCPClient
        self.tool_registry = {}    # tool_name -> server_name

    async def connect_servers(self, configs: dict):
        for server_name, config in configs.items():
            client = MCPClient(config)
            await client.connect()

            # Discover tools from this server
            tools = await client.list_tools()
            self.clients[server_name] = client

            # Record which server handles each tool
            for tool in tools:
                self.tool_registry[tool.name] = server_name

    async def call_tool(self, tool_name: str, arguments: dict):
        # Find which server handles this tool
        server_name = self.tool_registry.get(tool_name)
        if not server_name:
            return {"error": f"Unknown tool: {tool_name}"}

        client = self.clients[server_name]
        return await client.call_tool(tool_name, arguments)

    def get_all_tools(self) -> list:
        # Return combined tool list from all servers
        all_tools = []
        for client in self.clients.values():
            all_tools.extend(client.available_tools)
        return all_tools
```

The host is the router. When the LLM outputs a tool call, the host looks up which server handles it and forwards the request there.

### Tool Name Collisions

When connecting to multiple servers, different servers might expose tools with the same name. A GitHub server and a GitLab server might both have a tool called `create_issue`. The host must handle this.

The simplest approach is namespacing: prefix each tool name with the server name when aggregating the tool list.

```python
def load_tools_with_namespace(server_name: str, tools: list) -> list:
    namespaced = []
    for tool in tools:
        namespaced_tool = tool.copy()
        namespaced_tool["name"] = f"{server_name}__{tool['name']}"
        namespaced.append(namespaced_tool)
    return namespaced

# Results in tool names like:
# github__create_issue
# gitlab__create_issue
# github__list_repos
```

The double underscore is a common convention but any separator that does not appear in normal tool names works.

---

## 3. MCP vs Direct Tool Calling

Function calling lets agents use tools without MCP. MCP is an additional layer on top. It is worth being clear about when each approach is the right choice.

### Direct Tool Calling

You define tools as Python functions in your application code. You pass their schemas directly to the LLM. Your application code executes them when called.

```python
# Direct tool calling -- no MCP involved

tools = [
    {
        "name": "search_docs",
        "description": "Search the document store.",
        "parameters": { ... }
    }
]

def search_docs(query: str) -> str:
    return document_store.search(query)

response = llm.call(messages=messages, tools=tools)
if response.tool_calls:
    result = search_docs(**response.tool_calls[0].arguments)
```

### MCP Tool Calling

Tools are defined in a separate MCP server. The host discovers them at runtime and calls them through the protocol.

```python
# MCP tool calling

host = AgentHost()
await host.connect_servers(server_configs)

tools = host.get_all_tools()   # discovered from servers

response = llm.call(messages=messages, tools=tools)
if response.tool_calls:
    result = await host.call_tool(
        response.tool_calls[0].name,
        response.tool_calls[0].arguments
    )
```

### When to Use Each

| Situation | Approach |
|---|---|
| Small number of custom tools for a specific application | Direct tool calling |
| Tools tightly coupled to your application logic | Direct tool calling |
| Quick prototype or internal tool | Direct tool calling |
| Tools need to be reusable across multiple AI applications | MCP |
| Tools maintained by a different team than the AI application | MCP |
| You want users to bring their own tools | MCP |
| Building a platform where tool providers and AI apps are separate | MCP |
| Open source tool that others will consume | MCP |

The honest answer is: for simple applications, direct tool calling is simpler and sufficient. MCP adds value when you have a genuine reusability or separation-of-concerns requirement. Do not add MCP complexity just because it is newer or sounds more sophisticated.

---

## 4. Practical Patterns

### Dynamic Tool Discovery

Rather than configuring a fixed list of tools in your application, let users add their own MCP servers. The host discovers what tools are available at startup by querying all connected servers.

```python
async def start_agent(user_id: str):
    # Load this user's configured MCP servers
    user_servers = load_user_server_configs(user_id)

    host = AgentHost()

    for server_name, config in user_servers.items():
        try:
            await host.connect_server(server_name, config)
        except Exception as e:
            # Log the failure but continue with other servers
            print(f"Could not connect to {server_name}: {e}")

    # Run the agent with whatever tools are available
    tools = host.get_all_tools()
    return await run_agent_loop(tools, host)
```

This is how Claude Desktop works. Users configure their own set of MCP servers and Claude discovers what tools are available based on what servers are running.

### Graceful Degradation

MCP servers can fail. A server might not start, might crash mid-session, or might become unreachable. Handle these failures gracefully.

```python
async def call_tool_safely(
    host: AgentHost,
    tool_name: str,
    arguments: dict
) -> str:
    try:
        result = await host.call_tool(tool_name, arguments)
        return result

    except MCPConnectionError:
        return (
            f"The tool '{tool_name}' is temporarily unavailable. "
            "Try a different approach or ask the user to check the server."
        )

    except MCPTimeoutError:
        return (
            f"The tool '{tool_name}' timed out after waiting for a response. "
            "The operation may or may not have completed."
        )

    except Exception as e:
        return f"Tool call failed with an unexpected error: {str(e)}"
```

Return the error as a tool result string. The LLM receives it as an observation and can decide how to proceed -- try a different tool, ask the user for help, or report that the task cannot be completed.

### Caching Tool Lists

Calling `list_tools` on every request adds latency. Cache the tool list after the initial connection and refresh it periodically or when the server sends a notification that its tools have changed.

```python
class MCPClient:
    def __init__(self, config):
        self.config = config
        self._cached_tools = None
        self._cache_time = None
        self._cache_ttl = 300   # refresh every 5 minutes

    async def get_tools(self) -> list:
        now = time.time()
        if (
            self._cached_tools is None or
            now - self._cache_time > self._cache_ttl
        ):
            self._cached_tools = await self._fetch_tools()
            self._cache_time = now

        return self._cached_tools
```

---

## 5. Security in Production

MCP servers execute code with real effects. Prompt injection attacks, bugs in the LLM's reasoning, and misuse can all lead to unintended tool calls. Take security seriously.

### Validate All Inputs in Your Server

The LLM generates tool arguments. It can be manipulated via prompt injection to generate unexpected or malicious arguments. Never trust inputs blindly.

```python
@mcp.tool()
def read_file(path: str) -> str:
    """Read a file from the project directory."""
    import os

    # Resolve the full path
    base = "/safe/project/directory"
    full_path = os.path.realpath(os.path.join(base, path))

    # Reject path traversal attempts
    if not full_path.startswith(base):
        return "Error: access to that path is not permitted."

    # Check the file exists and is a file
    if not os.path.isfile(full_path):
        return f"Error: {path} is not a file or does not exist."

    with open(full_path, "r") as f:
        return f.read()
```

Without the path traversal check, an attacker could inject a tool call like `read_file(path="../../etc/passwd")` and read sensitive system files.

### Run Servers with Minimal Permissions

Each MCP server should run with only the OS-level permissions it needs.

A server that reads a specific directory does not need access to the entire filesystem. A server that queries a database should use a read-only database user. A server that calls one API does not need network access to anything else.

On Linux and macOS, you can restrict what a process can do using tools like `firejail`, containers, or simply by running the server as a dedicated system user with limited permissions.

### Require Confirmation for Destructive Operations

Tools that create, modify, or delete data should not execute silently. Give users a way to approve actions before they happen.

```python
@mcp.tool()
def delete_record(table: str, record_id: str) -> str:
    """
    Delete a record from the database.
    This operation cannot be undone.
    The host application will ask the user to confirm before this executes.
    """
    # The host is responsible for asking the user before calling this tool.
    # The server trusts that confirmation has happened and proceeds.
    result = database.delete(table=table, id=record_id)
    return f"Deleted record {record_id} from {table}."
```

In your host application, intercept tool calls for destructive operations and show the user what is about to happen:

```python
DESTRUCTIVE_TOOLS = {"delete_record", "drop_table", "send_email", "post_message"}

async def call_tool_with_confirmation(tool_name, arguments):
    if tool_name in DESTRUCTIVE_TOOLS:
        confirmed = await ask_user_for_confirmation(tool_name, arguments)
        if not confirmed:
            return "User cancelled the operation."

    return await host.call_tool(tool_name, arguments)
```

### Log Everything

Log every tool call, its arguments, and its result. This is essential for debugging, auditing, and detecting abuse.

```python
import logging
import json

logger = logging.getLogger("mcp_calls")

async def call_tool_with_logging(tool_name, arguments):
    logger.info(json.dumps({
        "event":     "tool_call_start",
        "tool":      tool_name,
        "arguments": arguments,
        "timestamp": time.time()
    }))

    result = await host.call_tool(tool_name, arguments)

    logger.info(json.dumps({
        "event":     "tool_call_complete",
        "tool":      tool_name,
        "result_length": len(str(result)),
        "timestamp": time.time()
    }))

    return result
```

---

## 6. The MCP Ecosystem

MCP has grown quickly since its release in November 2024. As of early 2026, there are hundreds of published MCP servers.

### Official Reference Servers

Anthropic maintains a collection of reference MCP servers at `github.com/modelcontextprotocol/servers`. These are well-tested implementations for common services.

Some of the most useful ones:

**filesystem** -- read and write files within a specified directory
**github** -- create and manage GitHub issues, PRs, and repositories
**gitlab** -- similar to github but for GitLab
**postgres** -- connect to a PostgreSQL database and run queries
**sqlite** -- connect to a SQLite database
**slack** -- send messages and read channels in Slack
**brave-search** -- search the web using Brave Search API
**fetch** -- fetch web pages and return their content
**memory** -- a simple persistent key-value store for conversation memory

### Community Servers

Beyond the official collection, there are community-maintained servers for services like Notion, Linear, Stripe, AWS, GCP, and many others. Search for "MCP server [service name]" to find them.

### Before Building a Custom Server

Always check whether an MCP server already exists for what you need before writing one from scratch. Using an existing well-maintained server saves time and you benefit from bug fixes and updates from the community.

If you do build a custom server, consider publishing it. The community benefits from more high-quality servers, and others maintaining the same integration means you share the maintenance burden.

---

## 7. Key Takeaways

- In agentic systems, MCP is invisible to the LLM. The LLM sees tools and calls them. The host manages the MCP connections and routes calls to the right server.
- When connecting to multiple servers, the host aggregates all tools into one list. It uses a registry to track which server handles each tool. Use namespacing to avoid tool name collisions.
- MCP adds real value when tools need to be reusable across multiple applications or when the tools and the AI application are maintained by different teams. For simple single-application tools, direct function calling is simpler.
- Handle server failures gracefully. Return errors as tool result strings so the LLM can observe them and decide what to do next.
- Validate all inputs in your server implementations. Do not trust that the LLM will always generate safe or valid arguments.
- Run servers with minimal permissions. Require user confirmation for destructive operations. Log all tool calls.
- Check the official MCP server collection and community servers before building a custom integration.

---

