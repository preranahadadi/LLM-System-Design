# Module 05d -- Building an MCP Server
> Model Context Protocol Series · Python SDK, FastMCP, defining tools and resources, connecting to Claude Desktop

---

## Table of Contents
1. [Setup](#1-setup)
2. [Minimal Server](#2-minimal-server)
3. [Using FastMCP](#3-using-fastmcp)
4. [Defining Tools](#4-defining-tools)
5. [Defining Resources](#5-defining-resources)
6. [A Real-World Example](#6-a-real-world-example)
7. [Connecting to Claude Desktop](#7-connecting-to-claude-desktop)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Setup

Anthropic publishes an official Python SDK for building MCP servers. Install it with:

```bash
pip install mcp
```

For most servers you will also want FastMCP, which is included in the SDK and provides a simpler, decorator-based API.

The SDK handles all the protocol details -- the handshake, message serialization, transport management -- so you can focus on writing the tools and resources your server exposes.

---

## 2. Minimal Server

Here is the simplest possible MCP server written with the low-level SDK. It exposes one tool that adds two numbers.

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types
import asyncio

# Create the server with a name
app = Server("my-first-server")

# Define what tools this server offers
@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="add_numbers",
            description="Add two numbers together and return the result.",
            inputSchema={
                "type": "object",
                "properties": {
                    "a": {
                        "type": "number",
                        "description": "The first number"
                    },
                    "b": {
                        "type": "number",
                        "description": "The second number"
                    }
                },
                "required": ["a", "b"]
            }
        )
    ]

# Handle tool calls
@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "add_numbers":
        result = arguments["a"] + arguments["b"]
        return [types.TextContent(type="text", text=str(result))]

    raise ValueError(f"Unknown tool: {name}")

# Run the server over stdio
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

This is a complete working server. A host application can launch it as a subprocess and immediately call the `add_numbers` tool. The low-level API is verbose but shows exactly what is happening.

---

## 3. Using FastMCP

FastMCP is a higher-level wrapper that comes with the SDK. It uses Python decorators and reads function signatures and docstrings automatically. It removes most of the boilerplate.

The same server from above, written with FastMCP:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-first-server")

@mcp.tool()
def add_numbers(a: float, b: float) -> str:
    """Add two numbers together and return the result."""
    return str(a + b)

if __name__ == "__main__":
    mcp.run()
```

FastMCP:
- Reads the function's type hints to build the input schema automatically
- Uses the docstring as the tool description
- Handles the protocol boilerplate
- Runs over stdio by default

For almost all practical server development, use FastMCP. The low-level API is good to understand conceptually, but FastMCP is what you will actually write.

---

## 4. Defining Tools

### Basic Tool

```python
@mcp.tool()
def search_documents(query: str, max_results: int = 5) -> str:
    """
    Search the document store for content matching the query.
    Use this to find specific information in the knowledge base.
    Returns the top matching documents with their content and source.
    """
    results = document_store.search(query, limit=max_results)

    if not results:
        return "No documents found matching that query."

    output = []
    for r in results:
        output.append(f"Source: {r.source}\n{r.content}")

    return "\n\n---\n\n".join(output)
```

### Tool with Enum Parameter

Use Python's type system to constrain what values a parameter can take.

```python
from typing import Literal

@mcp.tool()
def get_tickets(
    status: Literal["open", "closed", "all"] = "open",
    assigned_to: str = None
) -> str:
    """
    Get support tickets filtered by status and optionally by assignee.
    status: filter by ticket status -- open, closed, or all
    assigned_to: optional email address to filter by assignee
    Returns a list of matching tickets with their ID, title, and status.
    """
    tickets = ticket_system.query(status=status, assignee=assigned_to)
    return format_ticket_list(tickets)
```

### Tool with Error Handling

Always handle errors gracefully. Return error information as a string result, not as a raised exception. If the tool raises an exception, the error handling depends on the host. If it returns a descriptive error string, the LLM receives that as an observation and can decide what to do next.

```python
@mcp.tool()
def run_query(sql: str) -> str:
    """
    Run a read-only SQL SELECT query and return the results.
    Only SELECT statements are allowed. Returns results as a formatted table.
    """
    # Validate input
    if not sql.strip().upper().startswith("SELECT"):
        return "Error: only SELECT queries are permitted. Modify operations are not allowed."

    try:
        results = database.execute(sql)

        if not results.rows:
            return "Query returned no results."

        return format_as_table(results.columns, results.rows)

    except database.SyntaxError as e:
        return f"SQL syntax error: {str(e)}"

    except database.PermissionError:
        return "Permission denied. You do not have access to this data."

    except Exception as e:
        return f"Query failed: {str(e)}"
```

### Async Tools

FastMCP supports both synchronous and asynchronous tools. Use async when your tool needs to make network requests or do I/O that would block if run synchronously.

```python
import httpx

@mcp.tool()
async def fetch_url(url: str) -> str:
    """
    Fetch the content of a web page and return the text.
    Use this to retrieve current information from a specific URL.
    """
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, timeout=10.0)
            response.raise_for_status()
            return response.text[:5000]   # limit to first 5000 chars
        except httpx.TimeoutException:
            return "Error: the request timed out."
        except httpx.HTTPStatusError as e:
            return f"Error: received status {e.response.status_code}."
```

---

## 5. Defining Resources

### Static Resource

A resource with content that does not change or changes very rarely.

```python
@mcp.resource("docs://getting-started")
def getting_started_guide() -> str:
    """The getting started guide for new users."""
    return """
    # Getting Started

    Welcome to the system. Here is what you need to know...
    """
```

### Dynamic Resource

A resource whose content is generated fresh each time it is read.

```python
@mcp.resource("database://schema")
def database_schema() -> str:
    """The current database schema showing all tables and their columns."""
    conn = get_database_connection()
    tables = conn.execute(
        "SELECT name, sql FROM sqlite_master WHERE type='table'"
    ).fetchall()
    conn.close()

    schema_parts = []
    for name, create_sql in tables:
        if create_sql:
            schema_parts.append(f"-- Table: {name}\n{create_sql}")

    return "\n\n".join(schema_parts)
```

### File Resource

```python
import os

@mcp.resource("file:///{path}")
def read_file(path: str) -> str:
    """Read the contents of a file from the allowed directory."""
    # Security: restrict to a specific directory
    base_dir = "/allowed/directory"
    full_path = os.path.realpath(os.path.join(base_dir, path))

    if not full_path.startswith(base_dir):
        return "Error: access to that path is not permitted."

    if not os.path.isfile(full_path):
        return f"Error: file not found: {path}"

    with open(full_path, "r", encoding="utf-8") as f:
        return f.read()
```

---

## 6. A Real-World Example

A complete MCP server for a SQLite database. It exposes the schema as a resource and two tools for exploring and querying the database.

```python
from mcp.server.fastmcp import FastMCP
from typing import Literal
import sqlite3
import os

DB_PATH = os.environ.get("DB_PATH", "data.db")
mcp = FastMCP("database-server")

def get_connection():
    return sqlite3.connect(DB_PATH)

# Resource: the database schema
@mcp.resource("db://schema")
def get_schema() -> str:
    """
    The complete database schema showing all tables, columns, and relationships.
    Read this first to understand what data is available before writing queries.
    """
    conn = get_connection()
    cursor = conn.execute(
        "SELECT name, sql FROM sqlite_master WHERE type='table' ORDER BY name"
    )
    tables = cursor.fetchall()
    conn.close()

    if not tables:
        return "No tables found in the database."

    parts = []
    for name, create_sql in tables:
        if create_sql:
            parts.append(f"-- {name}\n{create_sql}")

    return "\n\n".join(parts)

# Tool: list tables with row counts
@mcp.tool()
def list_tables() -> str:
    """
    List all tables in the database along with their row counts.
    Use this to get an overview of what data exists before querying.
    """
    conn = get_connection()
    cursor = conn.execute(
        "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
    )
    tables = [row[0] for row in cursor.fetchall()]

    if not tables:
        return "No tables found."

    results = []
    for table in tables:
        count = conn.execute(f"SELECT COUNT(*) FROM {table}").fetchone()[0]
        results.append(f"{table}: {count:,} rows")

    conn.close()
    return "\n".join(results)

# Tool: run a SELECT query
@mcp.tool()
def run_query(sql: str, limit: int = 50) -> str:
    """
    Run a read-only SELECT query against the database.
    Only SELECT statements are allowed -- no INSERT, UPDATE, DELETE, or DROP.
    The limit parameter caps the number of rows returned (default 50, max 500).
    Returns results as a formatted table with column headers.
    """
    # Safety checks
    sql_upper = sql.strip().upper()
    if not sql_upper.startswith("SELECT"):
        return "Error: only SELECT queries are permitted."

    forbidden = ["INSERT", "UPDATE", "DELETE", "DROP", "CREATE", "ALTER", "TRUNCATE"]
    for keyword in forbidden:
        if keyword in sql_upper:
            return f"Error: {keyword} statements are not allowed."

    # Cap the limit
    limit = min(limit, 500)

    conn = get_connection()
    try:
        cursor = conn.execute(sql)
        columns = [desc[0] for desc in cursor.description]
        rows = cursor.fetchmany(limit)
        conn.close()

        if not rows:
            return "Query returned no results."

        # Format as a readable table
        col_widths = [
            max(len(col), max((len(str(row[i])) for row in rows), default=0))
            for i, col in enumerate(columns)
        ]

        header = " | ".join(col.ljust(col_widths[i]) for i, col in enumerate(columns))
        separator = "-+-".join("-" * w for w in col_widths)
        data_rows = [
            " | ".join(str(row[i]).ljust(col_widths[i]) for i in range(len(columns)))
            for row in rows
        ]

        output = [header, separator] + data_rows
        if len(rows) == limit:
            output.append(f"\n(showing first {limit} rows)")

        return "\n".join(output)

    except sqlite3.Error as e:
        conn.close()
        return f"Query error: {str(e)}"

if __name__ == "__main__":
    mcp.run()
```

This server is useful as-is. Point it at any SQLite database and a host application can immediately explore the schema, list tables, and run read-only queries.

---

## 7. Connecting to Claude Desktop

Once you have a server, you configure Claude Desktop to connect to it by editing its configuration file.

**Location of the config file:**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

The config file lists the MCP servers Claude Desktop should connect to. Each entry specifies the command to run to start the server.

```json
{
  "mcpServers": {
    "my-database": {
      "command": "python",
      "args": ["/Users/you/projects/database_server.py"],
      "env": {
        "DB_PATH": "/Users/you/data/company.db"
      }
    }
  }
}
```

After saving the config file, restart Claude Desktop. It will launch the server as a subprocess and you will see the tools available in the interface. Claude can now call your tools during conversations.

### Connecting Multiple Servers

You can connect to as many servers as you need. Each one gets a name in the config.

```json
{
  "mcpServers": {
    "database": {
      "command": "python",
      "args": ["/path/to/database_server.py"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/Documents"]
    }
  }
}
```

Claude Desktop connects to all of them at startup and aggregates all their tools and resources.

### Using an Existing Server

Many MCP servers are published as npm packages (JavaScript) or Python packages. For these, you typically run them with `npx` or `uvx` rather than running a Python file directly.

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://user:password@localhost/mydb"
      ]
    }
  }
}
```

Check the server's documentation for the exact command to use.

---

## 8. Key Takeaways

- Install the MCP SDK with `pip install mcp`. Use FastMCP for all practical server development -- it removes most of the boilerplate.
- Tools are defined with the `@mcp.tool()` decorator. FastMCP reads the type hints for the input schema and the docstring for the description.
- Resources are defined with the `@mcp.resource("uri://pattern")` decorator. The URI identifies the resource.
- Always handle errors gracefully in tools. Return error information as a string result so the LLM receives it as an observation and can decide what to do.
- Validate all inputs in your tools before using them. Do not trust that the LLM will always provide safe or valid arguments.
- Connect your server to Claude Desktop by editing the config file. Restart Claude Desktop after making changes.

---

