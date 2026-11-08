# Model Context Protocol (MCP) — Notes

**Source:** [CampusX — "Model Context Protocol"](https://www.youtube.com/playlist?list=PLKnIA16_Rmva_oZ9F4ayUu9qcWgF7Fyc0) (YouTube playlist, Nitish Singh)
**Length:** 8 videos, about 6.5 hours in total
**Language:** Taught in Hindi. Code in Python (FastMCP, LangChain MCP adapters, Streamlit), with Claude Desktop as the ready-made client.
**Code repos:** [expense-tracker-mcp-server](https://github.com/campusx-official/expense-tracker-mcp-server) · [test-remote-mcp-server](https://github.com/campusx-official/test-remote-mcp-server) · [mcp-client](https://github.com/campusx-official/mcp-client)
**Slides:** [Google Slides deck](https://docs.google.com/presentation/d/1OQzPhi73EUUbYXrIkOwGLnQYTSK8xU8T_zrtZcfgN0Q/edit) (covers videos 1–5)

---

## How to use these notes

The playlist is split into **Why → What → How**, after a trailer that shows MCP in action:

- **Why** (02): the problem MCP solves. Read this to be able to explain MCP to anyone.
- **What** (03–04): architecture and lifecycle. The conceptual core. Every later chapter leans on these.
- **How** (05–08): use existing servers, build a local server, deploy a remote one, build your own client.

Read in order the first time. Afterwards, use [09-quick-reference.md](09-quick-reference.md) as a cheat sheet.

Where the spec or tooling has changed since the videos were recorded (Aug–Nov 2025), the notes add a short **"Update"** or **"Correction"** box rather than silently changing what was taught.

---

## Contents

| # | File | Covers | Video |
|---|------|--------|-------|
| 01 | [Trailer: AI newsletter with MCP](01-trailer-ai-newsletter-with-mcp.md) | A real workflow (research → edit → design) done with Claude Desktop + MCP servers and three prompts | [#1](https://www.youtube.com/watch?v=3_TN1i3MTEU) 37m |
| 02 | [The Why](02-the-why.md) | LLM adoption waves, fragmentation, context, copy-paste hell, function calling, N × M problem, MCP, benefits, network effects | [#2](https://www.youtube.com/watch?v=Zmy439spZB4) 52m |
| 03 | [Architecture](03-architecture.md) | Host/client/server, 1:1 clients, primitives (tools, resources, prompts), JSON-RPC 2.0 data layer, stdio and HTTP transports | [#3](https://www.youtube.com/watch?v=nQa31xdXbGk) 1h 17m |
| 04 | [Lifecycle](04-lifecycle.md) | Initialize handshake, version and capability negotiation, operation, shutdown, ping, errors, timeouts, cancellation, progress | [#4](https://www.youtube.com/watch?v=sBHeMcxupmE) 55m |
| 05 | [Connecting servers to Claude Desktop](05-connecting-servers-to-claude-desktop.md) | Connectors vs config file, file system, Manim, Google Drive, Twitter, weather servers | [#5](https://www.youtube.com/watch?v=y-uPv3ltOTY) 46m |
| 06 | [Building a local MCP server](06-building-a-local-mcp-server.md) | MCP SDK vs FastMCP, uv workflow, MCP Inspector, expense tracker with SQLite, resources, FastAPI → MCP | [#6](https://www.youtube.com/watch?v=tc2oOznpdE0) 1h 12m |
| 07 | [Building and deploying a remote server](07-building-and-deploying-a-remote-mcp-server.md) | HTTP transport, FastMCP Cloud, custom connectors, read-only DB, async, proxy workaround, the multi-user/auth flaw | [#7](https://www.youtube.com/watch?v=GF7-ZzUausU) 46m |
| 08 | [Building an MCP client](08-building-an-mcp-client.md) | LangChain MCP adapters, `MultiServerMCPClient`, tool-calling loop, multiple servers, Streamlit UI | [#8](https://www.youtube.com/watch?v=o4ajsc-tSBc) 40m |
| 09 | [Quick reference](09-quick-reference.md) | Cheat sheet: terms, JSON-RPC messages, error codes, configs, commands, checklists | — |

> The playlist ends by promising advanced topics (authentication, sampling, elicitation, sessions) after the LangGraph series. They're introduced conceptually in 04 but not built in these videos.

Related notes in this repo: [FDE/CoderArmy 14 — MCP servers](../../FDE/CoderArmy/14-mcp-servers.md) and [15 — MCP resources and prompts](../../FDE/CoderArmy/15-mcp-resources-and-prompts.md) cover the same protocol from a Java/Spring angle.

---

## The short summary

LLMs went from novelty to work tool to an API inside every app. The result was **fragmentation**: many AIs, each blind to the others. The real blocker to a single, useful AI partner is **context**, which at work is scattered across Jira, GitHub, databases, Drive and Slack. **Function calling** let models fetch that context through tools, but every AI app had to write and maintain its own integration for every service: **N × M** integrations, each with its own auth, formats and secrets.

**MCP** standardizes the connection. A **host** (Claude Desktop, Cursor, your chatbot) creates one **client** per **server**. Servers offer three **primitives**: **tools** (actions), **resources** (readable, mostly static data) and **prompts** (templates). Messages are **JSON-RPC 2.0** (the data layer): requests with `id`s, responses, errors with standard codes, and fire-and-forget notifications. They travel over a **transport**: **stdio** for local servers the host launches as subprocesses, **Streamable HTTP** (POST, optionally streamed with SSE) for remote ones. Keeping data and transport separate is what lets one protocol cover both.

Every session follows a **lifecycle**: `initialize` → response → `notifications/initialized` (versions and capabilities negotiated), then discovery (`tools/list` ...) and calls (`tools/call` ...), then a transport-level **shutdown**. Pings, timeouts with cancellation, and progress notifications handle the edge cases.

In practice: popular servers plug into Claude Desktop as **connectors**; anything else goes in the **JSON config file**. **FastMCP** makes a server a few decorated Python functions; **MCP Inspector** debugs it; switching `mcp.run()` to the HTTP transport makes it remote, and **FastMCP Cloud** deploys it from GitHub. A remote server then needs things a local one doesn't: persistent storage, async I/O, and above all **authentication and per-user data**. Finally, your own app becomes a client with **`MultiServerMCPClient`**: fetch tools, bind them to an LLM, execute the calls it asks for, return `ToolMessage`s, and get the final answer.
