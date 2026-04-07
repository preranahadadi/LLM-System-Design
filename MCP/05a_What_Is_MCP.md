# Module 05a -- What is MCP?
> Model Context Protocol Series · The problem MCP solves, what it is, how it compares to function calling

---

## Table of Contents
1. [The Problem MCP Solves](#1-the-problem-mcp-solves)
2. [What MCP Actually Is](#2-what-mcp-actually-is)
3. [MCP vs Function Calling](#3-mcp-vs-function-calling)
4. [Why It Matters](#4-why-it-matters)
5. [Key Takeaways](#5-key-takeaways)

---

## 1. The Problem MCP Solves

Before MCP, every AI application that needed to connect to an external tool or data source had to build that integration from scratch. If you wanted Claude to read from your database, you wrote custom code. If you wanted it to search your files, more custom code. If you wanted it to call your internal API, more custom code on top of that.

This creates what is called the M x N problem. If you have M different AI applications and N different tools or data sources, you potentially need M x N separate custom integrations. Every new AI application has to re-implement the same integrations from scratch. Every new tool has to write a separate connector for every AI system that wants to use it.

```
Without MCP:

Claude Desktop  --> custom code --> GitHub
Claude Desktop  --> custom code --> Postgres
Claude Desktop  --> custom code --> Slack
Cursor          --> custom code --> GitHub
Cursor          --> custom code --> Postgres
Cursor          --> custom code --> Slack
Other AI tool   --> custom code --> GitHub
...

Result: M applications times N tools = M*N integrations to build and maintain
```

This does not scale. Every team rebuilds the same connectors. When a tool's API changes, every integration breaks independently.

MCP solves this by introducing a standard protocol. Tools and data sources expose themselves as MCP servers using one standard interface. AI applications connect to those servers as MCP clients using the same standard interface. You write the integration once on each side and it works everywhere.

```
With MCP:

Claude Desktop  --> MCP protocol --> GitHub MCP Server
Cursor          --> MCP protocol --> Postgres MCP Server
Other AI tool   --> MCP protocol --> Slack MCP Server

Result: M + N integrations instead of M * N
```

A useful analogy is USB. Before USB, every peripheral device needed its own port and cable. A printer used one connector, a keyboard another, a mouse another. USB introduced one standard that works for all peripherals. MCP does the same thing for AI tools and data sources.

---

## 2. What MCP Actually Is

MCP stands for Model Context Protocol. It is an open protocol published by Anthropic in November 2024.

A protocol is a set of rules that two parties agree to follow so they can communicate. HTTP is a protocol that defines how web browsers and web servers talk to each other. MCP is a protocol that defines how AI applications and tools talk to each other.

MCP is not a library you install and use directly. It is a specification -- a document that describes the rules. Anyone can write software that follows those rules. Anthropic provides official SDKs in Python and TypeScript to make it easier to build MCP servers and clients, but the protocol itself is open and anyone can implement it.

The protocol defines:
- How an AI application connects to a tool or data source
- How the tool exposes what it can do
- How the AI application discovers those capabilities
- How requests and responses flow between them
- How errors are handled

MCP is also transport-agnostic. It can run over standard input and output (for local processes on the same machine), over HTTP (for remote services), or other communication channels. The same protocol rules apply regardless of the transport.

---

## 3. MCP vs Function Calling

Function calling and MCP both let an LLM interact with tools. They operate at different levels and work together rather than competing.

Function calling is a feature of the LLM API. When you make an API call to Claude or GPT-4o, you can include a list of tool definitions. The LLM reads those definitions and can respond with a structured tool call instead of plain text. Your application code then executes the tool and feeds the result back.

MCP is infrastructure that connects your application to where the tools actually live. It standardizes how your application discovers what tools are available and how it sends calls to them.

| | Function Calling | MCP |
|---|---|---|
| What it is | LLM API feature | Communication protocol |
| Where it operates | Inside the LLM API call | Between application and tool server |
| Who defines tools | Developer writes them into the API call | MCP server exposes them dynamically |
| Portability | Custom per application | Standard across any MCP-compatible app |
| Dynamic discovery | No | Yes, client asks server what it offers |
| Who executes the tool | Your application code | The MCP server |

They work together like this: the LLM uses function calling syntax to say "I want to call this tool with these arguments." The host application receives that, forwards it to the right MCP server, gets the result back, and feeds it to the LLM as an observation. The LLM never knows or cares about MCP. It just sees tool calls and results.

---

## 4. Why It Matters

The reason MCP is gaining adoption quickly is that AI applications are becoming more agentic. An agent that can only use tools built specifically for it is limited. An agent that can connect to any MCP server gains access to a growing ecosystem of capabilities without any custom integration work.

Concretely:

A company can build one MCP server for their internal knowledge base. Every AI tool their employees use -- Claude Desktop, Cursor, custom internal tools -- gets access to that knowledge base automatically, without each tool needing its own custom connector.

An open source project can publish an MCP server for their API. Every MCP-compatible AI application can use it without any custom integration work on either side.

A developer can switch between AI applications without losing access to their tools. If they set up a GitHub MCP server and a database MCP server, those work the same whether they are using Claude Desktop or Cursor or any other MCP-compatible host.

As of early 2026, there are hundreds of published MCP servers for common services including GitHub, GitLab, Slack, Linear, Postgres, Notion, Stripe, and many others. Before building a custom integration, it is always worth checking whether an MCP server already exists.

---

## 5. Key Takeaways

- MCP solves the M x N integration problem. Without it, every AI application needs custom code to connect to every tool. With it, each side implements the protocol once and they interoperate.
- MCP is a protocol specification, not a library. It defines rules for communication. Official SDKs make it easier to implement those rules.
- Function calling and MCP work together. Function calling is how the LLM expresses tool calls. MCP is how the application connects to where tools are implemented.
- MCP is transport-agnostic. It works over stdio for local servers and over HTTP for remote ones.
- The ecosystem is growing quickly. Check for an existing MCP server before building a custom integration.

---
