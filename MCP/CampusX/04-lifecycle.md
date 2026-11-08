# 04 — The MCP Lifecycle: Initialize, Operate, Shut Down

**Video:** [The MCP Lifecycle](https://www.youtube.com/watch?v=sBHeMcxupmE) (55m)

"The What", part 2. The architecture (chapter 03) told you *what* the boxes are. The lifecycle tells you *in what order things happen* when they work together during a session. Everything is shown live by tailing Claude Desktop's logs while it talks to a local file-system server. The later coding chapters build on this.

---

## Contents

1. [Session and lifecycle](#session-and-lifecycle)
2. [Phase 1: initialization](#phase-1-initialization)
3. [Two rules during initialization](#two-rules-during-initialization)
4. [Watching it live in Claude Desktop](#watching-it-live-in-claude-desktop)
5. [Version negotiation](#version-negotiation)
6. [Capability negotiation](#capability-negotiation)
7. [Phase 2: operation](#phase-2-operation)
8. [Phase 3: shutdown](#phase-3-shutdown)
9. [Special cases](#special-cases)
    - [Ping](#ping)
    - [Error handling](#error-handling)
    - [Timeouts and cancellation](#timeouts-and-cancellation)
    - [Progress notifications](#progress-notifications)
10. [Summary](#summary)
11. [Interview questions](#interview-questions)

---

## Session and lifecycle

[1:25](https://www.youtube.com/watch?v=sBHeMcxupmE&t=85s)

- **Session**: one continuous connection between a client and a server. Start Claude Desktop with a GitHub server configured → it connects in the background → that connection lasts until you close Claude Desktop. That's one session.
- **Lifecycle**: *the complete sequence of steps that govern how a host (client) and a server establish, use, and end a connection during a session.* The rulebook for how the architecture runs step by step.

Three stages:

```
1. Initialization  ──►  2. Operation  ──►  3. Shutdown
   (handshake)            (real work)         (end session)
```

Shutdown is usually triggered by closing the **host**. Whether the server goes down is often not in your control.

---

## Phase 1: initialization

[4:10](https://www.youtube.com/watch?v=sBHeMcxupmE&t=250s)

> The initialization phase **MUST be the first interaction** between client and server.

Two jobs:

1. **Protocol version compatibility.** MCP keeps evolving and has dated versions. If client and server speak different versions, calls might break later. So this is checked first.
2. **Exchange and negotiate capabilities.** Client says what it can do; server says what it can do.

Think of it as a **handshake**: agreeing on terms for how they'll communicate. It happens in three steps.

### Step 1: client sends `initialize`

[5:50](https://www.youtube.com/watch?v=sBHeMcxupmE&t=350s)

```json
{
  "jsonrpc": "2.0", "id": 0, "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "roots": { "listChanged": true }, "sampling": {} },
    "clientInfo": { "name": "claude-ai", "version": "0.1.0" }
  }
}
```

Three things go to the server:

- **Protocol version** (a date string)
- **Client capabilities**: what the client can do *for the server* (here `roots` and `sampling`, explained below)
- **Implementation info**: client name and version

### Step 2: server responds

[7:27](https://www.youtube.com/watch?v=sBHeMcxupmE&t=447s)

```json
{
  "jsonrpc": "2.0", "id": 0,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "tools": { "listChanged": true }, "resources": { "subscribe": true, "listChanged": true } },
    "serverInfo": { "name": "secure-filesystem-server", "version": "0.2.0" }
  }
}
```

The same three things from the server's side: its protocol version, its capabilities (tools, resources, ...), its name and version.

### Step 3: client sends `initialized`

[8:12](https://www.youtube.com/watch?v=sBHeMcxupmE&t=492s)

```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

A **notification**, so **no `id`**, and the server doesn't reply. After this the client and server are connected for the whole session and the client can start real work.

---

## Two rules during initialization

[9:00](https://www.youtube.com/watch?v=sBHeMcxupmE&t=540s)

1. The client **SHOULD NOT** send requests other than **pings** before the server has responded to `initialize`. No tool calls, no `tools/list`.
2. The server **SHOULD NOT** send requests other than **pings and logging** before it receives the `initialized` notification.

In short: until all three steps finish, neither side exchanges any other kind of message. There's no proper handshake yet.

---

## Watching it live in Claude Desktop

[10:56](https://www.youtube.com/watch?v=sBHeMcxupmE&t=656s)

Setup: Claude Desktop with the **file-system** MCP server installed (local, so stdio). A terminal on the side tails Claude's MCP log files with a filter that pretty-prints the JSON-RPC messages.

> On macOS the logs live in `~/Library/Logs/Claude/` (e.g. `tail -n 20 -F ~/Library/Logs/Claude/mcp*.log`). On Windows, `%APPDATA%\Claude\logs`.

Starting Claude Desktop showed exactly the three steps:

1. Client → `initialize` (protocol version, no special capabilities, `clientInfo: claude-ai`), `id: 0`.
2. Server → its protocol version (**different** from the client's: see version negotiation), capabilities: **tools only** (no prompts, no resources), `serverInfo: secure-filesystem-server`, same `id: 0`.
3. Client → `notifications/initialized`, no `id`.

More messages appeared right after. Those belong to the operation phase.

---

## Version negotiation

[15:08](https://www.youtube.com/watch?v=sBHeMcxupmE&t=908s)

What if the client's `protocolVersion` and the server's differ?

1. The client checks whether the server's version is in **its list of supported versions**.
2. **Yes** → send `initialized` and carry on.
3. **No** → **disconnect**. No further communication.

In the demo the versions differed, but the client supported the server's version, so the connection went ahead.

> Per the spec: the client sends the latest version it supports. If the server supports it, it echoes it back; otherwise it replies with another version it supports. If the client can't work with that, it should disconnect.

---

## Capability negotiation

[16:57](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1017s)

> Client and server capabilities establish **which protocol features will be available** during the session.

Each side learns what it may ask of the other, so expectations are set.

### Client capabilities (what the client offers the server)

| Capability | Meaning | Example |
|---|---|---|
| **Roots** | Gives the server access to the host's **base directories** | Cursor connected to a file-system server shares your project folder as a root, so "create a new file" works inside it |
| **Sampling** | The server can ask the **client's AI** to do LLM work (MCP is bi-directional) | Server holds thousands of docs and is asked to summarize them. Instead of running its own LLM, it asks the client to generate the summaries |
| **Elicitation** | The server can ask the client/user for **missing information** | Server needs more details to complete a request and asks for them |

> Note on the video's elicitation example (asking for a GitHub API key): the spec says elicitation **must not be used to request sensitive information** like passwords or API keys. Use it for things like "which repo?" or "confirm this date", and handle credentials through proper auth.

### Server capabilities (what the server offers the client)

| Capability | Meaning |
|---|---|
| **Tools** | Functions the client can ask the server to run |
| **Resources** | Static documents to read |
| **Prompts** | Templates on how to get work done with this server |
| **Logging** | Server sends log messages to the client, e.g. a long train-booking task: "form filled" → "payment done" → "confirmation received" |

### Sub-capabilities

[22:13](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1333s)

| Sub-capability | Meaning |
|---|---|
| `listChanged` | The server notifies the client when its list of tools/resources/prompts changes (5 tools at connect time → a 6th is added → "I have a new tool now") |
| `subscribe` | For resources: client subscribes to one resource and gets notified when it changes (e.g. a README in a repo) |

---

## Phase 2: operation

[24:47](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1487s)

> During the operation phase, client and server exchange messages according to the negotiated capabilities.

Two rules:

- **Respect the negotiated protocol version** when building JSON-RPC messages.
- **Only use capabilities that were successfully negotiated.**

Two parts:

### Part 1: capability discovery

[25:30](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1530s)

The server said "I support tools", but not *which* tools. So the client asks:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }
{ "jsonrpc": "2.0", "id": 2, "method": "prompts/list" }
{ "jsonrpc": "2.0", "id": 3, "method": "resources/list" }
```

This happens **automatically right after initialization**. In the demo:

- `id: 1` → the server returned all its tools with descriptions and input schemas: `read_file`, `read_multiple_files`, `write_file`, `edit_file`, `create_directory`, `list_directory`, `directory_tree`, ...
- `id: 2` and `id: 3` → **error `-32601 Method not found`**, because this server only declared tools. (Claude asked anyway.)

The client passes the tool list to the host, which stores it. When a user request comes in, the host maps it to the best tool.

### Part 2: tool calling

[29:47](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1787s)

User: *"Can you tell me what's written in `hello.py` on my desktop?"*

```json
{ "jsonrpc": "2.0", "id": 4, "method": "tools/call",
  "params": { "name": "read_file", "arguments": { "path": "/Users/nitish/Desktop/hello.py" } } }
```

What actually happened in the logs (after Nitish clicked **Allow once**):

1. Client called `list_allowed_directories` → server: "you can only access the Desktop".
2. Client called `read_file` with the `hello.py` path → server returned the file content.
3. Claude displayed the code.

---

## Phase 3: shutdown

[32:30](https://www.youtube.com/watch?v=sBHeMcxupmE&t=1950s)

- One side initiates, **typically the client**.
- **No JSON-RPC shutdown message is defined.**
- The **transport layer** is responsible for signalling termination.

### Shutdown over stdio (local servers)

[34:06](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2046s)

**Client-initiated (the normal case).** Because the client launched the server as a subprocess, it controls its streams:

1. Close the server's **input stream** (stdin).
2. Wait for the server to exit.
3. If it doesn't, send **`SIGTERM`** (ask the OS to terminate it politely: "pack up and leave").
4. If still unresponsive, send **`SIGKILL`** (force it: "get out now").

**Server-initiated (rare).** The server *may* close its output stream to the client and exit. (The slide says "client-initiated / SHOULD" for this case; he corrects it in the video to "server-initiated / MAY".)

### Shutdown over HTTP (remote servers)

[36:45](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2205s)

- **Client-initiated (common):** the host closes the HTTP connection(s) it opened.
- **Server-initiated (possible):** the server closes the connection from its side (overloaded, crashed, mid-process). The client must detect the dropped connection, close running tasks gracefully, and reconnect if appropriate.

**Demo:** quitting Claude Desktop logged `Server transport closed`, and **no JSON-RPC message** was exchanged. The whole lifecycle (init → operation → shutdown) was visible in one terminal window.

---

## Special cases

### Ping

[39:33](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2373s)

A lightweight request/response to check that **the other side is alive** and the connection is responsive. **Either side** can send it.

```json
{ "jsonrpc": "2.0", "id": "123", "method": "ping" }      // request
{ "jsonrpc": "2.0", "id": "123", "result": {} }          // response: empty result
```

**When it's used:**

- To check the other side is up **before full initialization**.
- During long periods of no activity, the client may send **periodic pings**. Otherwise the OS, a proxy or a firewall might decide the idle connection isn't needed and drop it silently.

### Error handling

[42:33](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2553s)

How host and server signal that a request went wrong. **MCP inherits JSON-RPC's standard error object.**

**Common causes:**

- Unsupported/mismatched protocol version (during init)
- Calling a method for a capability that wasn't negotiated
- Invalid arguments to a tool
- Internal server failure while processing
- Timeout exceeded → client cancels
- Malformed JSON-RPC messages

**Error object:**

```json
{ "jsonrpc": "2.0", "id": 2,
  "error": { "code": -32601, "message": "Method not found", "data": { "...": "optional debug info" } } }
```

| Field | Meaning |
|---|---|
| `code` | Integer category of the error |
| `message` | Short human-readable explanation |
| `data` | Optional extra structured info for debugging |

**Standard codes** (from JSON-RPC 2.0):

| Code | Name | Typical cause |
|---|---|---|
| `-32700` | Parse error | Body isn't valid JSON |
| `-32600` | Invalid request | JSON is valid but not a valid request object (e.g. missing `method`) |
| `-32601` | Method not found | Method doesn't exist / capability not offered (the `prompts/list` case above) |
| `-32602` | Invalid params | Wrong or missing arguments (tool needed a file, you sent something else) |
| `-32603` | Internal error | Server failed while handling the request |
| `-32000` to `-32099` | Server error (implementation-defined) | Auth failures, rate limits, quota errors, other app-specific issues |

You don't need to memorize all of these, but know the common ones.

> Small correction: the video describes `-32600` as "the method doesn't exist". That's `-32601`. `-32600` means the request object itself is malformed.

> Also worth knowing: a **tool that runs but fails** (e.g. API returned an error) usually isn't a JSON-RPC error. The server returns a normal `tools/call` result with `"isError": true` and the error text in `content`, so the LLM can see what went wrong and react.

### Timeouts and cancellation

[47:15](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2835s)

> Timeouts make sure requests don't **hang forever**.

**Why:**

- Protects against unresponsive or overloaded servers (a busy remote server with many clients).
- Frees memory and CPU held for the request.
- Gives the user feedback instead of leaving them waiting.

**How:**

1. MCP SDKs let the client set a **per-request timeout** (e.g. 30 s).
2. No result before the deadline → the client triggers a timeout.
3. The client sends a **cancellation notification**.
4. The server **must stop** processing that request and **not return a result**.

[49:57](https://www.youtube.com/watch?v=sBHeMcxupmE&t=2997s) Example: request `id: 7` asked to scan the whole codebase for files containing "MCP". It took too long:

```json
{ "jsonrpc": "2.0", "method": "notifications/cancelled",
  "params": { "requestId": 7, "reason": "Timeout exceeded (30s)" } }
```

### Progress notifications

[50:57](https://www.youtube.com/watch?v=sBHeMcxupmE&t=3057s)

Let the client know a **long-running request is still making progress**. Example: "scan every file in this repo for security vulnerabilities". Much better UX to see "200 of 400 files... 250 of 400...".

1. The client includes a **`progressToken`** in the request's `_meta`.
2. The server sends `notifications/progress` updates against that token while it works.

```json
// request
{ "jsonrpc": "2.0", "id": 8, "method": "tools/call",
  "params": { "name": "search_code", "arguments": { "repo": "..." },
              "_meta": { "progressToken": "7" } } }

// progress updates (notifications: no id)
{ "jsonrpc": "2.0", "method": "notifications/progress",
  "params": { "progressToken": "7", "progress": 60, "total": 100,
              "message": "Searching 600 of 1000 files" } }
```

The client formats these for the user, like the progress bar in a "deep research" mode.

---

## Summary

```
INITIALIZATION                  OPERATION                         SHUTDOWN
client → initialize             client → tools/list,              transport closes it
server → result (version,                prompts/list,            stdio: close stdin →
          capabilities, info)            resources/list             wait → SIGTERM → SIGKILL
client → notifications/         client → tools/call ...           http: close connection
          initialized           (only negotiated features)        (no JSON-RPC message)

Anytime: ping · errors (JSON-RPC codes) · notifications/cancelled · notifications/progress
```

1. A **session** is one continuous client-server connection; the **lifecycle** is its rulebook.
2. **Initialization** = handshake: `initialize` → response → `notifications/initialized`. Only pings (and server logging) are allowed before it completes.
3. **Version negotiation**: the client disconnects if it can't support the server's version.
4. **Capability negotiation**: client offers roots/sampling/elicitation; server offers tools/resources/prompts/logging; sub-capabilities `listChanged` and `subscribe`.
5. **Operation** = discovery (`*/list`, automatic) then calls (`tools/call` ...), using only negotiated features.
6. **Shutdown** is done by the transport, not by a JSON-RPC message.
7. Special cases: ping, standard error objects, timeouts with `notifications/cancelled`, progress via `progressToken`.

## Interview questions

**Q: What are the three phases of the MCP lifecycle?**
Initialization (handshake: versions and capabilities), operation (discovery and calls), shutdown (transport closes the connection).

**Q: Walk through the initialization handshake.**
Client sends `initialize` with its protocol version, capabilities and client info. Server replies with its version, capabilities and server info. Client sends `notifications/initialized`. Only then does normal traffic start.

**Q: What can be sent before initialization completes?**
The client may only send pings. The server may only send pings and log messages.

**Q: What happens on a protocol version mismatch?**
The client checks whether it supports the server's version. If yes, it continues; if not, it disconnects.

**Q: Name the client capabilities and what each allows.**
Roots (server can work within the host's directories), sampling (server can request LLM completions via the client), elicitation (server can ask the user for more information).

**Q: What do `listChanged` and `subscribe` mean?**
`listChanged`: the server will notify when its tools/resources/prompts list changes. `subscribe`: the client can subscribe to a resource and be notified when it changes.

**Q: How is an MCP session shut down?**
No JSON-RPC message; the transport handles it. Over stdio the client closes the server's stdin, waits, then escalates to SIGTERM and SIGKILL. Over HTTP the client closes the connection; clients must handle server-side drops.

**Q: How does a timeout work in MCP?**
The client sets a per-request timeout. When it expires, the client sends `notifications/cancelled` with the request ID, and the server must stop and not send a result.

**Q: How do progress updates work?**
The client sends a `progressToken` in the request's `_meta`. The server sends `notifications/progress` messages carrying that token, progress, optional total and message.

**Q: Which error code means "method not found"? "invalid params"?**
`-32601` and `-32602`.
