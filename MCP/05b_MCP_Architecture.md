# Module 05b -- MCP Architecture
> Model Context Protocol Series · Hosts, clients, servers, transport layer, connection lifecycle, security model

---

## Table of Contents
1. [The Three Components](#1-the-three-components)
2. [How They Fit Together](#2-how-they-fit-together)
3. [The Transport Layer](#3-the-transport-layer)
4. [The Connection Lifecycle](#4-the-connection-lifecycle)
5. [The Security Model](#5-the-security-model)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. The Three Components

Every MCP setup involves three distinct components: a host, one or more clients, and one or more servers. Understanding what each one does and who is responsible for what is the foundation of understanding MCP.

### Host

The host is the application the user is actually interacting with. Claude Desktop is a host. Cursor is a host. Any AI application that wants to use MCP to connect to external tools is a host.

The host is the top-level coordinator. It is responsible for:
- Presenting the user interface and managing the user experience
- Deciding which MCP servers to connect to
- Creating and managing the MCP clients that talk to those servers
- Receiving tool calls from the LLM and deciding whether to execute them
- Enforcing what is and is not allowed
- Mediating all communication between the LLM and the external world

The host owns the security boundary. Nothing goes to an MCP server without the host approving it.

### Client

The client is a component that lives inside the host application. It manages the connection to one specific MCP server. If the host connects to three MCP servers, it creates three separate client instances, one for each server.

The client is responsible for:
- Establishing and maintaining the connection to its server
- Sending requests to the server (list tools, call a tool, read a resource)
- Receiving responses and passing them back to the host
- Handling reconnection if the connection drops

Each client has a one-to-one relationship with one server. The client does not know about other servers or other clients.

### Server

The server is a separate process or remote service that exposes capabilities to any MCP client that connects to it. It runs independently from the host application.

The server is responsible for:
- Declaring what tools, resources, and prompts it offers
- Executing tool calls when the client requests them
- Returning resource content when asked
- Running with only the permissions it needs for its job

The server does not know about the LLM, the user, or anything beyond its own capabilities and the requests it receives.

---

## 2. How They Fit Together

The relationship flows in one direction: the host creates clients, clients connect to servers. The LLM never talks to the server directly.

```
USER
  |
  v
HOST APPLICATION
  |
  |-- creates and manages --> MCP CLIENT 1 <-- MCP protocol --> SERVER A (GitHub)
  |
  |-- creates and manages --> MCP CLIENT 2 <-- MCP protocol --> SERVER B (Postgres)
  |
  |-- creates and manages --> MCP CLIENT 3 <-- MCP protocol --> SERVER C (Internal Docs)

LLM
  ^
  |
  (host sends messages to LLM and receives responses)
  (host executes tool calls by forwarding to the right MCP client)
```

When the LLM decides to call a tool, the sequence is:

```
1. LLM outputs a tool call (e.g. search_documents with query="vacation policy")

2. Host receives the tool call

3. Host looks up which MCP server handles the search_documents tool

4. Host tells Client 3 (Internal Docs) to call search_documents

5. Client 3 sends the request to Server C over the MCP protocol

6. Server C executes the search and returns results

7. Client 3 passes results back to the host

8. Host formats the results and sends them back to the LLM as an observation

9. LLM continues its reasoning
```

The LLM sees a tool call and then a result. Everything in between -- the MCP protocol, the server, the actual execution -- is invisible to it.

---

## 3. The Transport Layer

MCP defines the rules for communication but does not prescribe a specific communication channel. The same protocol can run over different transports depending on where the server is located.

### stdio Transport

stdio stands for standard input and output. This transport is used when the MCP server is a local process running on the same machine as the host.

The host launches the server as a child process. The two processes communicate by writing to and reading from standard input and standard output -- the same channels that command-line programs use to receive input and print output.

```
Host process
    |
    | spawns as child process
    v
Server process
    |
    Communication via stdin / stdout pipe
```

This is the most common setup for developer tools. When you configure Claude Desktop to connect to a local MCP server, it uses stdio transport. The configuration tells Claude Desktop what command to run to start the server, and Claude Desktop manages the process lifecycle.

stdio is simple, fast, and does not require network configuration. The downside is that both processes must be on the same machine.

### HTTP with Server-Sent Events

This transport is used when the server is remote -- running in the cloud, on a different machine, or as a shared service.

The client sends requests over HTTPS POST. The server can send responses and streaming updates over a persistent Server-Sent Events connection. Server-Sent Events is a standard web technology that keeps an HTTP connection open so the server can push data to the client as it becomes available.

```
Host application (your machine)
    |
    | HTTPS requests and SSE responses
    v
Remote MCP Server (cloud service, company server, etc.)
```

This transport is used when you want to run an MCP server as a shared service that multiple users or applications connect to, or when the server needs to run in a controlled environment with specific resources.

---

## 4. The Connection Lifecycle

When an MCP client first connects to a server, they go through a handshake before doing any real work. This ensures both sides agree on how to communicate before either side starts making requests.

### Step 1: Connect

The client establishes the communication channel. For stdio this means launching the server process. For HTTP this means establishing the SSE connection.

### Step 2: Initialize Request

The client sends an initialize message to the server. This message contains:
- The version of the MCP protocol the client supports
- What capabilities the client has (for example, whether it supports sampling)
- Information identifying the client (name, version)

### Step 3: Initialize Response

The server responds with its own initialize message. This contains:
- The protocol version the server will use (may be different if versions differ)
- What capabilities the server supports (tools, resources, prompts, sampling)
- Information identifying the server (name, version)

### Step 4: Initialized Notification

The client sends a final notification to confirm the handshake is complete. After this, normal operation begins.

### Step 5: Normal Operation

The client can now:
- Ask the server to list its tools
- Call specific tools with arguments
- Ask the server to list its resources
- Read specific resources by URI
- Ask the server to list its prompts
- Get specific prompts with arguments

```
CLIENT                              SERVER
  |                                   |
  |--- initialize request ----------->|
  |                                   |
  |<-- initialize response -----------|
  |                                   |
  |--- initialized notification ----->|
  |                                   |
  |  (handshake complete)             |
  |                                   |
  |--- list_tools request ----------->|
  |<-- list_tools response -----------|
  |                                   |
  |--- call_tool request ------------>|
  |<-- call_tool response ------------|
  |                                   |
```

### Disconnection

When the host is done with a server (application closing, user disconnecting), it closes the connection cleanly. For stdio servers, the host terminates the child process.

---

## 5. The Security Model

MCP gives servers the ability to execute code and access external systems. This creates real security responsibilities. The protocol places the security responsibility primarily on the host.

### The Host is the Gatekeeper

The LLM never talks to an MCP server directly. All communication goes through the host. This means the host can:
- Inspect every tool call before forwarding it to a server
- Reject tool calls that do not meet its criteria
- Ask the user to confirm before executing sensitive operations
- Log all tool calls and results for auditing
- Disconnect from a server that behaves unexpectedly

The host should never blindly forward everything the LLM requests. It should apply its own judgment about what is safe to execute.

### Servers Should Have Minimal Permissions

Each MCP server should run with only the permissions it needs. A server that reads files should not have network access. A server that queries a database should use a read-only database connection. A server that searches documents should not have the ability to delete them.

This limits the damage if a server is exploited or if a prompt injection attack causes the LLM to make unexpected tool calls.

### User Consent

For operations that create, modify, or delete data, good host applications ask the user for confirmation before executing. The LLM is a reasoning system, not an autonomous agent with unrestricted authority. Significant actions should involve the human.

### Trust Boundaries

MCP servers are external processes. Treat them with the same trust you would give any external dependency. Do not give them access to secrets, credentials, or data beyond what they need for their specific function.

---

## 6. Key Takeaways

- MCP has three components: the host (the AI application the user interacts with), the client (a component inside the host that manages one server connection), and the server (the external process that exposes tools and data).
- The LLM never talks to the server directly. All communication goes through the host, which acts as a gatekeeper.
- Each client has a one-to-one connection with one server. If the host connects to multiple servers, it creates multiple clients.
- MCP supports two main transports: stdio for local servers on the same machine, and HTTP with Server-Sent Events for remote servers.
- Every connection starts with an initialization handshake where both sides declare what protocol version and capabilities they support.
- Security responsibility sits with the host. It decides what servers to connect to, what tool calls to forward, and whether to require user confirmation for sensitive operations.

---

