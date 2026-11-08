# 09 — MCP Quick Reference

Cheat sheet for the whole playlist. Details and context are in chapters 01–08.

---

## Contents

1. [Core terms](#core-terms)
2. [Architecture at a glance](#architecture-at-a-glance)
3. [Primitives and their methods](#primitives-and-their-methods)
4. [Capabilities](#capabilities)
5. [JSON-RPC message shapes](#json-rpc-message-shapes)
6. [Lifecycle](#lifecycle)
7. [Error codes](#error-codes)
8. [Transports](#transports)
9. [Claude Desktop config](#claude-desktop-config)
10. [FastMCP server templates](#fastmcp-server-templates)
11. [FastMCP / uv commands](#fastmcp--uv-commands)
12. [Client template (LangChain MCP adapters)](#client-template-langchain-mcp-adapters)
13. [Troubleshooting](#troubleshooting)
14. [Checklists](#checklists)

---

## Core terms

| Term | One line |
|---|---|
| **MCP** | Open protocol (from Anthropic, Nov 2024) for connecting AI apps to external tools and data |
| **Context** | Everything the model can see when it generates a response |
| **N × M problem** | N AI apps × M services = N×M custom integrations. MCP makes it N + M |
| **Host** | The AI app the user talks to (Claude Desktop, Cursor, your chatbot) |
| **Client** | Connector inside the host; one per server; speaks MCP |
| **Server** | Program exposing tools/resources/prompts over MCP. Does the heavy lifting |
| **Local server** | Runs on the host's machine; launched by the host; stdio |
| **Remote server** | Runs elsewhere; reached by URL; Streamable HTTP |
| **Data layer** | The "language": JSON-RPC 2.0 messages |
| **Transport layer** | How messages move: stdio or HTTP |
| **Session** | One continuous client-server connection |
| **Lifecycle** | Initialization → operation → shutdown |
| **Connector** | Claude Desktop's built-in, curated, one-click server integration |
| **FastMCP** | Developer-friendly Python framework for MCP servers and clients |
| **MCP Inspector** | Browser debugger for MCP servers ("Postman for MCP") |

---

## Architecture at a glance

```
 User ─► HOST ◄─► LLM
         ├─ Client A ──stdio──► Local server  (e.g. file system)
         ├─ Client B ──HTTP───► Remote server (e.g. GitHub)
         └─ Client C ──HTTP───► Remote server (e.g. Slack)
                each server: Tools · Resources · Prompts
                messages:    JSON-RPC 2.0
```

- Host never talks to servers directly; always via a client.
- Client ↔ server is **1:1**.
- Benefits: decoupling, safety (isolated failures), parallelism, scalability.

---

## Primitives and their methods

| Primitive | What | Methods | Example |
|---|---|---|---|
| **Tools** | Actions the model asks the server to perform (often dynamic) | `tools/list`, `tools/call` | `create_issue`, `list_commits`, `add_expense` |
| **Resources** | Readable data, usually static | `resources/list`, `resources/read`, `resources/subscribe`, `resources/unsubscribe` | README, DB schema, `categories.json` |
| **Prompts** | Server-provided templates that shape behaviour | `prompts/list`, `prompts/get` | "Issue report: title, steps, expected, actual, environment" |

Static → resource. Dynamic or side effects → tool.

---

## Capabilities

| Side | Capability | Meaning |
|---|---|---|
| Client | `roots` | Server may work within these directories/URIs |
| Client | `sampling` | Server may ask the client's LLM for a completion |
| Client | `elicitation` | Server may ask the user for more (non-sensitive) input |
| Server | `tools` / `resources` / `prompts` | Offers that primitive |
| Server | `logging` | Sends log messages to the client |
| Sub-capability | `listChanged` | Notifies when the list of tools/resources/prompts changes |
| Sub-capability | `subscribe` | Resource updates can be subscribed to |

---

## JSON-RPC message shapes

```json
// request: has id, expects a response
{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "add", "arguments": { "a": 2, "b": 3 } } }

// success response: same id
{ "jsonrpc": "2.0", "id": 1, "result": { "content": [ { "type": "text", "text": "5" } ] } }

// error response
{ "jsonrpc": "2.0", "id": 1, "error": { "code": -32602, "message": "Invalid params", "data": {} } }

// notification: no id, no response
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

Why JSON-RPC (not REST): lightweight, bi-directional, transport-agnostic, notifications (and batching in JSON-RPC itself, though MCP's 2025-06-18 spec removed batching).

---

## Lifecycle

| Phase | Messages |
|---|---|
| **Init 1** | client → `initialize` {protocolVersion, capabilities, clientInfo} |
| **Init 2** | server → result {protocolVersion, capabilities, serverInfo} |
| **Init 3** | client → `notifications/initialized` |
| Rules | Before init completes: client sends only pings; server only pings + logs |
| Version mismatch | Client keeps going if it supports the server's version, else disconnects |
| **Operation** | Discovery: `tools/list`, `resources/list`, `prompts/list` (automatic) → calls: `tools/call`, `resources/read`, ... Only negotiated features |
| **Shutdown** | No JSON-RPC message. stdio: close stdin → wait → SIGTERM → SIGKILL. HTTP: close connection; handle server-side drops |

**Special cases:**

| Case | Message |
|---|---|
| Ping | `{"method":"ping","id":..}` → `{"result":{}}`. Either side; liveness and keeping idle connections alive |
| Cancel (e.g. timeout) | `notifications/cancelled` {requestId, reason} → server stops, sends no result |
| Progress | request `_meta.progressToken` → `notifications/progress` {progressToken, progress, total, message} |
| List changed | `notifications/tools/list_changed` (and resources/prompts) |
| Resource changed | `notifications/resources/updated` {uri} |

---

## Error codes

| Code | Meaning |
|---|---|
| `-32700` | Parse error: not valid JSON |
| `-32600` | Invalid request: not a valid JSON-RPC request object |
| `-32601` | Method not found / capability not offered |
| `-32602` | Invalid params |
| `-32603` | Internal error |
| `-32000`…`-32099` | Implementation-defined server errors (auth, rate limits, quota...) |

A tool that runs but fails → normal result with `"isError": true`, not a JSON-RPC error.

---

## Transports

| | stdio | Streamable HTTP |
|---|---|---|
| For | Local servers | Remote servers |
| How | Host spawns server as subprocess; JSON-RPC over stdin/stdout | Client POSTs JSON-RPC to one endpoint (e.g. `/mcp`); server replies with JSON or an SSE stream |
| Pros | Fast, secure (no open port), simple | Reach anywhere, many clients, standard HTTP auth, streaming |
| Cons | One user, your machine only | Latency; must handle auth, multi-tenancy, persistence |
| Note | | Replaced the older two-endpoint "HTTP + SSE" transport (deprecated) |

---

## Claude Desktop config

`Settings → Developer → Edit Config` → `claude_desktop_config.json`
(macOS `~/Library/Application Support/Claude/`, Windows `%APPDATA%\Claude\`)

```json
{
  "mcpServers": {
    "expense-tracker": {
      "command": "/absolute/path/to/uv",
      "args": ["run", "--with", "fastmcp", "fastmcp", "run", "/absolute/path/to/main.py"]
    },
    "twitter-mcp": {
      "command": "npx",
      "args": ["-y", "@enescinar/twitter-mcp"],
      "env": { "API_KEY": "...", "API_SECRET_KEY": "...", "ACCESS_TOKEN": "...", "ACCESS_TOKEN_SECRET": "..." }
    }
  }
}
```

- Config entries = local stdio processes (`npx`/`uvx` packages also run locally).
- Remote servers: *Settings → Connectors → Add custom connector* (URL), or a local proxy.
- Restart Claude Desktop after every change. Logs: `~/Library/Logs/Claude/mcp*.log`.

---

## FastMCP server templates

**Local (stdio):**

```python
from fastmcp import FastMCP

mcp = FastMCP("ExpenseTracker")

@mcp.tool
def add_expense(date: str, amount: float, category: str, note: str = "") -> dict:
    """Add an expense. date: YYYY-MM-DD."""
    ...

@mcp.resource("expense://categories", mime_type="application/json")
def categories() -> str:
    return open("categories.json").read()

@mcp.prompt
def monthly_review(month: str) -> str:
    return f"Summarize my spending for {month} by category and flag anything unusual."

if __name__ == "__main__":
    mcp.run()                                               # stdio
```

**Remote:** same code, last line becomes

```python
    mcp.run(transport="http", host="0.0.0.0", port=8000)   # endpoint: /mcp
```

**Proxy a remote server as a local one:**

```python
mcp = FastMCP.as_proxy("https://<server>.fastmcp.app/mcp", name="My Proxy")
mcp.run()
```

**Existing FastAPI app → MCP server:**

```python
from main import app
mcp = FastMCP.from_fastapi(app=app, name="Expense Tracker Server")
mcp.run()
```

Tips: type hints → input schema; docstring → description. Async tools + async DB driver for remote servers.

---

## FastMCP / uv commands

| Task | Command |
|---|---|
| Install uv | `pip install uv` |
| New project | `uv init .` |
| Add FastMCP | `uv add fastmcp` |
| Versions | `uv run fastmcp version` |
| Inspector | `uv run fastmcp dev main.py` |
| Run (stdio) | `uv run fastmcp run main.py` |
| Run (HTTP) | `uv run fastmcp run main.py --transport http --host 0.0.0.0 --port 8000` or `uv run main.py` |
| Install in Claude Desktop | `uv run fastmcp install claude-desktop main.py` (then fix `uv` → absolute path) |
| Find uv | `which uv` |
| Deploy | push to GitHub → FastMCP Cloud → *Deploy from your own code* → entry point `main.py` |
| Official SDK instead | `pip install "mcp[cli]"`, `from mcp.server.fastmcp import FastMCP` |

---

## Client template (LangChain MCP adapters)

```python
import asyncio, json
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langchain_core.messages import ToolMessage

SERVERS = {
    "math":    {"transport": "stdio", "command": "/abs/path/uv",
                "args": ["run", "fastmcp", "run", "/abs/path/math/main.py"]},
    "expense": {"transport": "streamable_http", "url": "https://<server>.fastmcp.app/mcp"},
}

async def main(prompt: str):
    client = MultiServerMCPClient(SERVERS)
    tools = await client.get_tools()
    by_name = {t.name: t for t in tools}
    llm = ChatOpenAI(model="gpt-5").bind_tools(tools)

    ai = await llm.ainvoke(prompt)
    if not ai.tool_calls:
        return ai.content
    msgs = [ToolMessage(tool_call_id=tc["id"],
                        content=json.dumps(await by_name[tc["name"]].ainvoke(tc["args"])))
            for tc in ai.tool_calls]
    return (await llm.ainvoke([prompt, ai, *msgs])).content

print(asyncio.run(main("Add an expense of ₹800 for groceries on 4 November")))
```

Message order: prompt → AIMessage(tool_calls) → ToolMessage per call (matching `tool_call_id`) → final answer. For multi-step tool use, loop until there are no tool calls (or use a LangGraph agent).

---

## Troubleshooting

| Symptom | Likely fix |
|---|---|
| Claude Desktop: "could not connect" to your server | Use absolute path for `command` (`which uv`); restart Claude fully |
| New server/tool not showing | Restart Claude Desktop; check JSON is valid and entry is inside `mcpServers` |
| Tool works in Claude but behaves oddly | Test in MCP Inspector first; check JSON-RPC history |
| LLM answers without calling the tool | Be explicit ("using the X tool"); improve tool descriptions |
| Inspector form says a param is "required" oddly | Add type hints to the tool signature |
| Deployed SQLite is read-only | Write to a writable path; for persistence use a managed DB |
| Remote server slow for many users | Async tools + async DB driver |
| No "Add custom connector" | Plan limits; use a local proxy (`FastMCP.as_proxy` / `mcp-remote`) |
| `-32601 Method not found` | Capability not offered by that server |
| Twitter post fails with auth error | App permissions → Read and write, then regenerate access tokens |
| ToolMessage error | Pass `tool_call_id`, not the tool name |

---

## Checklists

**Writing a good tool:** clear name · docstring that says when to use it · typed params with formats (YYYY-MM-DD) · validate inputs · return structured data or a clear error · no secrets in output.

**Before sharing a remote server:** HTTP transport · writable *and persistent* storage · async I/O · error handling · **authentication** · per-user data (`user_id`, enforced server-side) · secrets in env vars · DB files in `.gitignore` · tested in Inspector.

**Explaining MCP in 30 seconds:** AI apps need context from many systems. Function calling made every app integrate every service (N × M). MCP is a standard protocol: services ship one server, apps implement one client, and any client can use any server (N + M). Servers expose tools, resources and prompts over JSON-RPC, via stdio locally or HTTP remotely.
