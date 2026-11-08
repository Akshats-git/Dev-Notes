# 03 — MCP Architecture: Host, Client, Server, Primitives, Data and Transport Layers

**Video:** [MCP Architecture](https://www.youtube.com/watch?v=nQa31xdXbGk) (1h 17m)

"The What", part 1. Built up from first principles: start with the simplest two-box picture (host ↔ server), then add clients, primitives, the **data layer** (JSON-RPC 2.0) and the **transport layer** (stdio, HTTP), until you reach the full diagram. By the end you should be able to explain every box to anyone.

---

## Contents

1. [Simplest version: host and server](#simplest-version-host-and-server)
2. [Adding the client](#adding-the-client)
3. [One client per server](#one-client-per-server)
4. [Benefits of this design](#benefits-of-this-design)
5. [Primitives: tools, resources, prompts](#primitives-tools-resources-prompts)
6. [Standard operations on primitives](#standard-operations-on-primitives)
7. [Summary so far](#summary-so-far)
8. [The data layer and JSON-RPC 2.0](#the-data-layer-and-json-rpc-20)
9. [JSON-RPC in MCP: worked examples](#json-rpc-in-mcp-worked-examples)
10. [Why JSON-RPC instead of REST?](#why-json-rpc-instead-of-rest)
11. [The transport layer](#the-transport-layer)
12. [Local servers and stdio](#local-servers-and-stdio)
13. [Remote servers and HTTP (+ SSE)](#remote-servers-and-http--sse)
14. [Why separating data and transport is the clever part](#why-separating-data-and-transport-is-the-clever-part)
15. [The full architecture](#the-full-architecture)
16. [Interview questions](#interview-questions)

---

## Simplest version: host and server

[2:00](https://www.youtube.com/watch?v=nQa31xdXbGk&t=120s)

Forget "client" from the last video for a moment. The simplest picture has two things:

- **Host**: the AI chatbot the user talks to. Backed by an LLM (OpenAI, Anthropic, Gemini). Can be ready-made (Claude Desktop, Cursor) or your own (e.g. a LangGraph chatbot).
- **Server**: a tool that can perform a specific kind of task. A GitHub server (manage repos), a Slack server (read/write channel messages), a Google Drive server (manipulate files).

**Flow for "Are there any new commits on the GitHub repo?":**

```
User ──prompt──► Host ──► LLM: "not in my training data; I have a GitHub server"
                 Host ──query──► GitHub server ──checks repo──► list of commits
                 Host ──result──► LLM ──answer──► Host ──UI──► User
```

---

## Adding the client

[5:30](https://www.youtube.com/watch?v=nQa31xdXbGk&t=330s)

In reality **the host never talks to a server directly**. It goes through a helper: the **MCP client**.

The client's superpower: it **speaks the same MCP language the server speaks**.

```
User ─► Host ─(high-level request)─► Client ─(MCP request)─► Server
                                       ▲                         │
Host ◄─(something host understands)── Client ◄─(MCP response)───┘
```

1. Host turns the prompt into a high-level request: "find recent commits in the repo".
2. **Client converts it into an MCP-compatible request** and sends it to the server.
3. Server does the work and returns a **structured MCP response**.
4. **Client translates the response** back into something the host can use.

---

## One client per server

[8:30](https://www.youtube.com/watch?v=nQa31xdXbGk&t=510s)

The client-server relationship is **one-to-one**. A client connected to the GitHub server can't also talk to the Slack server. Add a second server → the host creates a second client. A third server → a third client.

**Analogy: phone, SIM, network.**

| Phone world | MCP world |
|---|---|
| Phone | Host |
| SIM card | MCP client |
| Network (Airtel, Jio) | MCP server |

The phone doesn't talk to the network directly. The SIM does. Want Airtel, Jio and Vodafone? Install three SIMs, each tied to one network.

> A host can connect to many servers, but it never does the low-level communication itself. It creates **one client per server**, and each client handles all communication with its server end to end.

---

## Benefits of this design

[12:17](https://www.youtube.com/watch?v=nQa31xdXbGk&t=737s)

| Benefit | Why |
|---|---|
| **Decoupling** | Separation of concerns. The GitHub channel knows nothing about the Drive or Slack channel |
| **Safety** | If the GitHub line breaks, Slack and Drive keep working |
| **Parallelism** | Separate clients can work in parallel. A request that needs Slack *and* GitHub runs both at once |
| **Scalability** | Add as many servers as you like: one more client per server |

---

## Primitives: tools, resources, prompts

[15:30](https://www.youtube.com/watch?v=nQa31xdXbGk&t=930s)

> **Primitives are the things a server can offer to the host.**

Three kinds:

### 1. Tools: actions the AI asks the server to perform

GitHub server: count commits, list active issues, count repos in a profile.
Drive server: search for a file or folder, create a file.

### 2. Resources: structured data sources the AI can read

Documents or static knowledge sitting on the server. GitHub: a repo's `README`. Database server: the schema.

**Rule of thumb:** if the thing is **static** (doesn't change), it's a **resource**. If it's **dynamic** (commit count, open issues, number of repos), you fetch it with a **tool**.

### 3. Prompts: predefined templates/instructions that shape the AI's behaviour

[18:32](https://www.youtube.com/watch?v=nQa31xdXbGk&t=1112s)

The confusing one, so an example. User says: *"Create an issue for a bug: the login button doesn't work."*

Without a prompt, the LLM generates something vague: title "Login bug", body "the login button doesn't work". Real issues should be more descriptive, but nobody told the LLM the format.

So the server stores a prompt template:

```json
{
  "name": "issue_report_prompt",
  "description": "Write clear, detailed GitHub issues",
  "messages": [{
    "role": "system",
    "content": "Always include: Title, Steps to Reproduce, Expected, Actual, Environment"
  }]
}
```

Now the generated issue looks like:

```
Title: Bug in login button
Steps to reproduce: open login page → enter credentials → press Login
Expected: user gets logged in
Actual: nothing happens
Environment: Chrome, macOS
```

> Prompts let the **server teach the host how to use it well**. Not needed everywhere, but very useful in specialized cases.

| Primitive | In one line |
|---|---|
| Tools | Make the server **do** something |
| Resources | **Read** static data from the server |
| Prompts | **Guide** how the AI responds / uses the server |

---

## Standard operations on primitives

[22:40](https://www.youtube.com/watch?v=nQa31xdXbGk&t=1360s)

MCP also defines standard methods for working with each primitive:

| Primitive | Method | What it does |
|---|---|---|
| Tools | `tools/list` | "What tools do you provide?" Server returns all its tools (create issue, push code, commit, ...) |
| | `tools/call` | "Run this tool with these arguments" |
| Resources | `resources/list` | Which static documents are available |
| | `resources/read` | Give me the content of this resource |
| | `resources/subscribe` | Tell me when this resource changes |
| | `resources/unsubscribe` | Stop telling me |
| Prompts | `prompts/list` | All prompt templates on the server |
| | `prompts/get` | Fetch one specific template |

You'll see these implemented when building servers (chapter 06).

---

## Summary so far

[25:37](https://www.youtube.com/watch?v=nQa31xdXbGk&t=1537s)

```
                    ┌──────── Host (AI chatbot + LLM) ────────┐
                    │  Client 1      Client 2      Client 3   │
                    └────┬──────────────┬─────────────┬───────┘
                         │ 1:1          │ 1:1         │ 1:1
                    Slack server   GitHub server   Drive server
                    (tools, resources, prompts) on each
```

- Host = the chatbot. Connects to many servers.
- One **client per server**.
- Each server offers **primitives**: tools, resources, prompts.
- MCP gives **standard methods** for them: `*/list`, `tools/call`, `resources/read`, ...

Two things left: the **data layer** and the **transport layer**.

---

## The data layer and JSON-RPC 2.0

[27:20](https://www.youtube.com/watch?v=nQa31xdXbGk&t=1640s)

> **The data layer is the language and grammar of the MCP ecosystem that everyone agrees upon to communicate.**

We kept saying "client and server speak the same language". The data layer is what that language looks like: its grammar, its rules.

> **In MCP, JSON-RPC 2.0 is the foundation of the data layer.** JSON-RPC gives the rules for writing MCP's language.

### What is RPC?

[29:35](https://www.youtube.com/watch?v=nQa31xdXbGk&t=1775s)

**Remote Procedure Call**: lets a program execute a function on another computer *as if it were local*, hiding network communication and data transfer. It makes distributed applications (components on different machines) easier to build.

Instead of writing `add(2, 3)` locally, you send a request: *"please run `add` with parameters 2 and 3"*.

### What is JSON-RPC?

**JavaScript Object Notation – Remote Procedure Call.** RPC + JSON: RPC requests and responses written in a standard JSON format. Current version: **2.0**.

### Request and response

```json
// request (client → server)
{ "jsonrpc": "2.0", "method": "add", "params": [2, 3], "id": 1 }

// response (server → client)
{ "jsonrpc": "2.0", "result": 5, "id": 1 }
```

| Field | Meaning |
|---|---|
| `jsonrpc` | Protocol version, must be `"2.0"` and must match |
| `method` | Name of the function to run on the server |
| `params` | Arguments |
| `id` | Lets the client match a response to its request |
| `result` | The return value (response only) |

It's a bit like a REST API (two machines communicating), but a different flavour: REST is built around HTTP verbs (GET, POST...), JSON-RPC around **calling a named method**.

### Errors

If the method doesn't exist:

```json
{ "jsonrpc": "2.0", "error": { "code": -32601, "message": "Method not found" }, "id": 1 }
```

Just like HTTP has 404/500, JSON-RPC has standardized error codes (more in chapter 04).

---

## JSON-RPC in MCP: worked examples

[37:00](https://www.youtube.com/watch?v=nQa31xdXbGk&t=2220s)

The video shows simplified versions. Below they're written closer to what MCP actually sends, so you recognise them in an inspector.

### Discover tools: `tools/list`

```json
// client
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }

// server
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "tools": [
      { "name": "list_issues", "description": "List issues in a repository",
        "inputSchema": { "type": "object",
          "properties": { "owner": {"type": "string"}, "repo": {"type": "string"} },
          "required": ["owner", "repo"] } },
      { "name": "list_pulls", "description": "List pull requests", "inputSchema": { "...": "..." } }
    ]
  }
}
```

The list tells the client **which parameters each tool needs**. That's how it knows what to send in `tools/call`.

### Call a tool: `tools/call`

```json
// client
{ "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "list_issues", "arguments": { "owner": "campusx", "repo": "demo" } } }

// server
{ "jsonrpc": "2.0", "id": 2,
  "result": { "content": [ { "type": "text", "text": "2 open issues: #1 Login bug, #2 ..." } ] } }
```

### Resources: `resources/list` and `resources/read`

```json
{ "jsonrpc": "2.0", "id": 3, "method": "resources/list" }

{ "jsonrpc": "2.0", "id": 4, "method": "resources/read",
  "params": { "uri": "file:///repo/README.md" } }

// response
{ "jsonrpc": "2.0", "id": 4,
  "result": { "contents": [ { "uri": "file:///repo/README.md", "mimeType": "text/markdown", "text": "# Demo..." } ] } }
```

### Batching

[41:28](https://www.youtube.com/watch?v=nQa31xdXbGk&t=2488s)

JSON-RPC lets you send several requests in one array (e.g. list issues **and** list PRs), and the server returns the responses together.

```json
[
  { "jsonrpc": "2.0", "id": 5, "method": "tools/call", "params": { "name": "list_issues", "arguments": {"...": "..."} } },
  { "jsonrpc": "2.0", "id": 6, "method": "tools/call", "params": { "name": "list_pulls",  "arguments": {"...": "..."} } }
]
```

> **Update since the video:** JSON-RPC itself supports batching, and MCP's 2025-03-26 spec allowed it, but the **2025-06-18 spec removed batching** from MCP. Treat it as a JSON-RPC feature, not something to rely on in MCP today.

### Notifications

[42:22](https://www.youtube.com/watch?v=nQa31xdXbGk&t=2542s)

- A **request** always gets a **response**.
- A **notification** is **fire-and-forget**: no response expected, so **no `id`**.
- Either side can send one.

Example: you subscribed to a resource, the file changed, the server tells you:

```json
{ "jsonrpc": "2.0", "method": "notifications/resources/updated",
  "params": { "uri": "file:///repo/README.md" } }
```

### Error example

[45:02](https://www.youtube.com/watch?v=nQa31xdXbGk&t=2702s)

Client called `list_issues` with only the owner, no repo name:

```json
{ "jsonrpc": "2.0", "id": 7,
  "error": { "code": -32602, "message": "Invalid params: missing required field 'repo'" } }
```

You'll rarely write JSON-RPC by hand. SDKs do it. But knowing the shape helps when debugging.

---

## Why JSON-RPC instead of REST?

[46:45](https://www.youtube.com/watch?v=nQa31xdXbGk&t=2805s)

What MCP really does: run a function on a remote machine and bring the result back. REST APIs do that too. So why did Anthropic pick JSON-RPC? Five reasons:

| # | Reason | vs REST |
|---|---|---|
| 1 | **Lightweight** | REST over HTTP carries lots of headers and metadata. A JSON-RPC message is a tiny JSON object: easy to write, debug and ship |
| 2 | **Bi-directional** | REST is one-way: client requests, server responds. In JSON-RPC the **server can also send requests to the client** (used by advanced features later) |
| 3 | **Transport-agnostic** | REST is tied to HTTP. JSON-RPC works over HTTP, stdio, WebSockets, or a custom transport |
| 4 | **Batching** | Several requests at once (see the note above about MCP) |
| 5 | **Notifications** | REST is strictly one request → one response. JSON-RPC has fire-and-forget messages |

---

## The transport layer

[53:23](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3203s)

> **The transport layer is the mechanism that moves JSON-RPC messages between the client and server.**

The data layer says what messages look like. The transport layer answers: *through what medium do they travel?* The choice depends on the **type of server**.

### Two types of servers

[55:27](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3327s)

| | Local server | Remote server |
|---|---|---|
| Runs where | On the **same computer** as the host | On **another computer**, over a network or the internet |
| Demo | File-system server: "list the folders on my desktop" (it *must* be local to see your files) | GitHub server: "list my top 5 most-starred repos" |
| Transport | **stdio** | **HTTP** (+ SSE for streaming) |

---

## Local servers and stdio

[59:36](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3576s)

**stdio = standard input/output**, the built-in streams every program has:

- **stdin**: input the program reads (your `input()` from the keyboard)
- **stdout**: output the program writes (your `print()` to the screen)

MCP smartly uses these streams as the transport between client and a local server.

### How it works

[1:01:45](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3705s)

1. The **host launches the server as a subprocess** on the same machine. This creates a **parent-child relationship**, and the host now controls the server's stdin and stdout.
2. The host's client **writes JSON-RPC messages into the server's stdin**.
3. The server reads them, processes them, and **writes responses to its stdout**, which the host reads.

### The demo

[1:03:00](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3780s)

```python
# hello.py
name = input()
print(f"Hello {name}")
```

```bash
$ python3 hello.py
Nitish
Hello Nitish
```

Nothing new, but look at it this way: the **terminal is the host**, `hello.py` is the **server**. The terminal launched it as a subprocess (step 1), wrote "Nitish" to its stdin (step 2), and got "Hello Nitish" back on stdout (step 3). That's exactly what Claude Desktop does with the file-system server, except the messages are JSON-RPC.

### Benefits of stdio

[1:06:22](https://www.youtube.com/watch?v=nQa31xdXbGk&t=3982s)

- **Fast**: data passes directly between two processes on one machine.
- **Secure**: no network port is opened, so there's nothing to attack from outside.
- **Simple**: every language supports stdin/stdout. No extra libraries.

---

## Remote servers and HTTP (+ SSE)

[1:07:36](https://www.youtube.com/watch?v=nQa31xdXbGk&t=4056s)

HTTP is the most popular application protocol on the internet, so the host can reach a server **anywhere**.

- The host sends JSON-RPC messages as the **JSON payload of HTTP POST requests**.
- You get all the standard **HTTP auth methods** (API keys, tokens, OAuth).

**SSE (Server-Sent Events)** is an extension of HTTP for **streaming**:

- The server sends **multiple messages over a single open connection**.
- Instead of one big JSON blob, it streams chunks as they're ready.
- Ideal for **long-running tasks** (an agent doing a series of steps) and **incremental updates**.

> **Update since the video:** the original "HTTP + SSE" transport (2024-11-05 spec) used two endpoints. The **2025-03-26 spec replaced it with *Streamable HTTP***: one endpoint (e.g. `/mcp`). The client POSTs JSON-RPC, and the server answers either with plain JSON or by opening an SSE stream on that response. Same ideas (POST + optional SSE), simpler shape. The later videos in this course (FastMCP Cloud, `streamable_http` in the client) use Streamable HTTP. The old SSE transport is deprecated.

---

## Why separating data and transport is the clever part

[1:12:09](https://www.youtube.com/watch?v=nQa31xdXbGk&t=4329s)

Remember "JSON-RPC is transport-agnostic"? Now it makes sense:

- Local servers use **stdio**, remote servers use **HTTP**. In **both**, the messages are written the same way: **JSON-RPC**.
- Anthropic knew some servers would be local and some remote. Had they chosen **REST**, remote would work but local wouldn't, unless you ran a web server on your own machine for every local tool (hectic, extra complexity).
- If a new transport is needed tomorrow (WebSockets?), JSON-RPC works with it, and **the data layer doesn't change**.

> The brilliance of MCP's architecture: **data layer and transport layer are separate.** Swap the transport and the data layer stays the same.

---

## The full architecture

[1:14:49](https://www.youtube.com/watch?v=nQa31xdXbGk&t=4489s)

```
 User ──► HOST (AI chatbot) ◄──► LLM
          ┌──────────┬──────────┬──────────┐
          │ Client A │ Client B │ Client C │      one client per server (1:1)
          └────┬─────┴────┬─────┴────┬─────┘
     data:     │  JSON-RPC 2.0 messages │
     transport:│ stdio    │ HTTP     │ HTTP
               ▼          ▼          ▼
        Local server   Remote      Remote
        (file system)  (GitHub)    (Slack)
        ┌───────────────────────────────────┐
        │ each offers: Tools · Resources · Prompts │
        └───────────────────────────────────┘
```

1. **Host**: the chatbot. Takes user messages, sends them to an LLM.
2. Needs external servers but **doesn't talk to them directly**: it keeps **one client per server**.
3. Servers are **local** (same machine) or **remote** (another machine/internet).
4. Every server offers three **primitives**: tools, resources, prompts.
5. Client ↔ server messages are **JSON-RPC 2.0** (data layer).
6. They travel over **stdio** (local) or **HTTP/SSE** (remote) (transport layer).

### Beyond this video

The course covers server primitives. The spec also defines **client primitives**, which servers can request from clients (this is what the "bi-directional" point enables): **sampling** (server asks the host's LLM for a completion), **roots** (which directories/URIs the server may work in), and **elicitation** (server asks the user for more input). Worth knowing for interviews.

---

## Interview questions

**Q: What are the host, client and server in MCP?**
Host: the AI app the user interacts with (Claude Desktop, Cursor, your chatbot). Client: a connector inside the host that speaks MCP to exactly one server. Server: a program exposing capabilities (tools, resources, prompts).

**Q: Can one MCP client talk to multiple servers?**
No. Client-server is one-to-one. A host connecting to three servers creates three clients.

**Q: Why have a client at all, rather than the host talking to servers?**
Decoupling and isolation (one broken connection doesn't affect others), parallel calls to different servers, and easy scaling by adding clients.

**Q: Tools vs resources vs prompts?**
Tools are actions the model can invoke (often dynamic data or side effects). Resources are readable data, typically static (a README, a DB schema). Prompts are server-provided templates that guide how the model performs a task.

**Q: What is JSON-RPC and why does MCP use it?**
A lightweight RPC protocol encoded in JSON (`jsonrpc`, `method`, `params`, `id`). MCP uses it because it's lightweight, bi-directional, transport-agnostic and supports notifications.

**Q: Request vs notification in JSON-RPC?**
A request has an `id` and must get a response. A notification has no `id` and expects no response.

**Q: Which transports does MCP use?**
stdio for local servers (the host launches the server as a subprocess and talks via stdin/stdout). HTTP for remote servers: today "Streamable HTTP" (POST to one endpoint, optional SSE streaming), which replaced the older HTTP+SSE transport.

**Q: Why is stdio good for local servers?**
Fast (direct inter-process communication), secure (no open network port), and simple (every language supports it).

**Q: What's the advantage of separating the data layer from the transport layer?**
The same JSON-RPC messages work over any transport. New transports can be added without changing message formats.
