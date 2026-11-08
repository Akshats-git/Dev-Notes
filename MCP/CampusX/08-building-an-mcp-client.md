# 08 — Building an MCP Client with LangChain MCP Adapters

**Video:** [How to build MCP Clients](https://www.youtube.com/watch?v=o4ajsc-tSBc) (40m)
**Code:** [campusx-official/mcp-client](https://github.com/campusx-official/mcp-client) · **Quick revision:** [Google Doc](https://docs.google.com/document/d/1I26xHGosv0B7zgDl0sM_GZtlX20rPWYBPZHEcoREN7A/edit) · **Async primer:** [Real Python: async IO](https://realpython.com/async-io-python/)

Until now Claude Desktop was the client. What if you're building **your own chatbot**? This video builds an MCP client in Python with **`langchain-mcp-adapters`**, connects it to a **local** math server, a **remote** expense tracker and the community **Manim** server, writes the tool-calling loop by hand, and wraps it in a **Streamlit** chat UI.

---

## Contents

1. [Choosing a client library](#choosing-a-client-library)
2. [The goal](#the-goal)
3. [The local math server](#the-local-math-server)
4. [Setup](#setup)
5. [Step 1: connect and fetch tools](#step-1-connect-and-fetch-tools)
6. [Step 2: bind tools to an LLM](#step-2-bind-tools-to-an-llm)
7. [Step 3: execute the tool call](#step-3-execute-the-tool-call)
8. [Step 4: send the result back](#step-4-send-the-result-back)
9. [Step 5: handle normal chat and multiple tool calls](#step-5-handle-normal-chat-and-multiple-tool-calls)
10. [Adding more servers: remote expense tracker and Manim](#adding-more-servers-remote-expense-tracker-and-manim)
11. [Full CLI client (client1.py)](#full-cli-client-client1py)
12. [Streamlit chat UI (client2.py)](#streamlit-chat-ui-client2py)
13. [Limitations and next steps](#limitations-and-next-steps)
14. [Interview questions](#interview-questions)

---

## Choosing a client library

[1:48](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=108s)

Three ways to build a client in Python:

| Library | Notes |
|---|---|
| Official MCP SDK (`mcp`) | `ClientSession` + transport clients. Lowest level |
| FastMCP (`fastmcp.Client`) | Simpler, same library as our servers |
| **LangChain MCP adapters** (`langchain-mcp-adapters`) | "A lightweight wrapper that makes MCP tools compatible with LangChain and LangGraph" |

He picks **LangChain MCP adapters** because it was the simplest, and it plugs straight into LangChain/LangGraph (which the channel already teaches). The concepts carry over to the other two.

---

## The goal

[3:54](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=234s)

A chatbot that chats normally **and** has MCP servers attached behind the scenes:

| Server | Type | Transport |
|---|---|---|
| Math (add, subtract, multiply, divide, power, modulus) | Local, built by him | stdio |
| Expense tracker (from chapter 07) | Remote, on FastMCP Cloud | Streamable HTTP |
| Manim (from chapter 05) | Local, built by someone else | stdio |

Plan: connect the local server first → add the remote one → add Manim → wrap it in Streamlit.

---

## The local math server

[6:13](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=373s)

A small FastMCP server in `~/Desktop/mcp-math-server/main.py` (its code isn't in the repo; it looks roughly like this):

```python
from fastmcp import FastMCP

mcp = FastMCP("Math Server")

@mcp.tool
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b

@mcp.tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

# ... subtract, divide, power, modulus

if __name__ == "__main__":
    mcp.run()
```

Tested with `uv run fastmcp dev main.py`: six tools; `modulus(21, 7)` → 0.

---

## Setup

[8:07](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=487s)

```bash
pip install uv
mkdir mcp-client && cd mcp-client        # open in VS Code
uv init .
uv add langchain langchain-openai langchain-mcp-adapters streamlit python-dotenv
```

`.env` with `OPENAI_API_KEY=...`.

The code uses Python **`async`/`await`** throughout. MCP calls are network/process I/O, so the adapter API is async.

---

## Step 1: connect and fetch tools

[9:23](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=563s)

**Skeleton:**

```python
import asyncio

async def main():
    ...

if __name__ == "__main__":
    asyncio.run(main())
```

**Server config.** The same shape as Claude Desktop's config (chapter 05), plus an explicit `transport`:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

SERVERS = {
    "math": {
        "transport": "stdio",
        "command": "/absolute/path/to/uv",          # i.e. just "uv"
        "args": ["run", "fastmcp", "run", "/Users/nitish/Desktop/mcp-math-server/main.py"],
    },
}
```

With stdio, **the client starts the server**, so you give it the command to launch it (chapter 03).

**Connect and list tools:**

```python
async def main():
    client = MultiServerMCPClient(SERVERS)     # "multi" because we'll add more servers
    tools = await client.get_tools()           # tools/list on every server → LangChain tools
    print(tools)
```

Running it: the client launches the math server (you see its stdio startup banner), then prints a list of tool objects: `add`, `subtract`, `multiply`, `divide`, `power`, `modulus`.

**Index them by name** (needed later to execute a call):

```python
named_tools = {}
for tool in tools:
    named_tools[tool.name] = tool
# {"add": <StructuredTool ...>, "subtract": <...>, ...}
```

---

## Step 2: bind tools to an LLM

[14:06](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=846s)

Tools are useless without someone to choose them: the LLM.

```python
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

llm = ChatOpenAI(model="gpt-5")
llm_with_tools = llm.bind_tools(tools)       # pass the list, not the dict

prompt = "What is the product of 12 and 15?"
response = await llm_with_tools.ainvoke(prompt)
print(response)
```

- First run: **no tool call**. The calculation was easy, so the LLM just answered.
- Changed the prompt to *"...using the math tool"*: `content` is empty and `response.tool_calls` holds:

```python
[{"name": "multiply", "args": {"a": 12, "b": 15}, "id": "call_abc123", "type": "tool_call"}]
```

It's a list because the LLM may request **several tool calls at once**. The LLM only *says* which tool to use with which input. Running it is your job.

---

## Step 3: execute the tool call

[19:18](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=1158s)

```python
selected_tool      = response.tool_calls[0]["name"]     # "multiply"
selected_tool_args = response.tool_calls[0]["args"]     # {"a": 12, "b": 15}
selected_tool_id   = response.tool_calls[0]["id"]

result = await named_tools[selected_tool].ainvoke(selected_tool_args)
print(result)   # 180
```

`ainvoke` on the adapter tool sends a `tools/call` to the right MCP server and returns the result.

---

## Step 4: send the result back

[23:32](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=1412s)

Give the result back to the LLM so it can answer the user. Wrap it in a **`ToolMessage`** and resend the **whole history**: original prompt → the AI message that asked for the tool → the tool result.

```python
from langchain_core.messages import ToolMessage
import json

tool_message = ToolMessage(tool_call_id=selected_tool_id, content=json.dumps(result))

final_response = await llm_with_tools.ainvoke([prompt, response, tool_message])
print(final_response.content)    # "The product of 12 and 15 is 180."
```

First attempt errored: a `ToolMessage` needs the **`tool_call_id`** (not the tool name). That's how the LLM matches results to the calls it made.

**The full loop:**

```
prompt ─► LLM (with tools) ─► AIMessage(tool_calls=[multiply(12,15)])
                                   │
             client executes ◄─────┘   named_tools["multiply"].ainvoke(args)  ─► MCP server
                                   │
          ToolMessage(id, "180") ──┘
[prompt, AIMessage, ToolMessage] ─► LLM ─► "The product is 180"
```

---

## Step 5: handle normal chat and multiple tool calls

[27:20](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=1640s)

**Normal questions.** *"What is the capital of India?"* → no tool calls → the code crashed on `response.tool_calls[0]`. Fix:

```python
if not getattr(response, "tool_calls", None):
    print("\nLLM Reply:", response.content)
    return
```

Now "capital of India" → *New Delhi*; *"What is the remainder of 231423 divided by 7?"* → tool call → answer.

**Multiple tool calls.** [29:29](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=1769s) The LLM may ask for several tools in one response. Loop over all of them and collect one `ToolMessage` each:

```python
tool_messages = []
for tc in response.tool_calls:
    result = await named_tools[tc["name"]].ainvoke(tc.get("args") or {})
    tool_messages.append(ToolMessage(tool_call_id=tc["id"], content=json.dumps(result)))

final_response = await llm_with_tools.ainvoke([prompt, response, *tool_messages])
```

---

## Adding more servers: remote expense tracker and Manim

[29:29](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=1769s)

Adding a server = adding a config entry. **No other code changes.**

```python
SERVERS = {
    "math": {
        "transport": "stdio",
        "command": "/absolute/path/to/uv",
        "args": ["run", "fastmcp", "run", "/Users/nitish/Desktop/mcp-math-server/main.py"],
    },
    "expense": {
        "transport": "streamable_http",          # remote server; if this fails, try "sse"
        "url": "https://splendid-gold-dingo.fastmcp.app/mcp",
    },
    "manim-server": {
        "transport": "stdio",
        "command": "/absolute/path/to/python3",
        "args": ["/Users/nitish/desktop/manim-mcp-server/src/manim_server.py"],
        "env": {"MANIM_EXECUTABLE": "/absolute/path/to/manim"},
    },
}
```

| Transport | Required keys |
|---|---|
| `stdio` (local) | `command`, `args`, optional `env` |
| `streamable_http` (remote) | `url` (plus optional `headers` for auth) |

**Remote expense tracker** (chapter 07): printing `named_tools.keys()` now shows the math tools **plus** `add_expense`, `list_expenses`, `summarize`. *"Add an expense of ₹800 for groceries on 4th November"* → expense added, with date, category and amount echoed back.

**Manim** (chapter 05): the first server someone else built. Same config as for Claude Desktop with `transport: "stdio"` added. New tools appear: `execute_manim_code`, `cleanup_manim_temp_dir`. *"Draw a triangle rotating in place using the Manim tool"* → an animation video is generated.

> Any MCP server from the internet can be attached to your own chatbot this way.

---

## Full CLI client (client1.py)

From the course repo:

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import ToolMessage
import json

load_dotenv()

SERVERS = {
    "math": {
        "transport": "stdio",
        "command": "/Library/Frameworks/Python.framework/Versions/3.11/bin/uv",
        "args": ["run", "fastmcp", "run", "/Users/nitish/Desktop/mcp-math-server/main.py"]
    },
    "expense": {
        "transport": "streamable_http",  # if this fails, try "sse"
        "url": "https://splendid-gold-dingo.fastmcp.app/mcp"
    },
    "manim-server": {
        "transport": "stdio",
        "command": "/Library/Frameworks/Python.framework/Versions/3.11/bin/python3",
        "args": ["/Users/nitish/desktop/manim-mcp-server/src/manim_server.py"],
        "env": {"MANIM_EXECUTABLE": "/Library/Frameworks/Python.framework/Versions/3.11/bin/manim"}
    }
}

async def main():
    client = MultiServerMCPClient(SERVERS)
    tools = await client.get_tools()

    named_tools = {}
    for tool in tools:
        named_tools[tool.name] = tool

    print("Available tools:", named_tools.keys())

    llm = ChatOpenAI(model="gpt-5")
    llm_with_tools = llm.bind_tools(tools)

    prompt = "Draw a triangle rotating in place using the manim tool."
    response = await llm_with_tools.ainvoke(prompt)

    if not getattr(response, "tool_calls", None):
        print("\nLLM Reply:", response.content)
        return

    tool_messages = []
    for tc in response.tool_calls:
        selected_tool = tc["name"]
        selected_tool_args = tc.get("args") or {}
        selected_tool_id = tc["id"]

        result = await named_tools[selected_tool].ainvoke(selected_tool_args)
        tool_messages.append(ToolMessage(tool_call_id=selected_tool_id, content=json.dumps(result)))

    final_response = await llm_with_tools.ainvoke([prompt, response, *tool_messages])
    print(f"Final response: {final_response.content}")

if __name__ == '__main__':
    asyncio.run(main())
```

---

## Streamlit chat UI (client2.py)

[35:21](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=2121s)

Same logic, converted into a Streamlit chat app (written with ChatGPT's help; the video is about MCP, not Streamlit). Key parts:

```python
import json, asyncio
import streamlit as st
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage, SystemMessage

SERVERS = { ... }   # same three servers

SYSTEM_PROMPT = (
    "You have access to tools. When you choose to call a tool, do not narrate status updates. "
    "After tools run, return only a concise final answer."
)

st.set_page_config(page_title="MCP Chat", page_icon="🧰", layout="centered")
st.title("🧰 MCP Chat")
load_dotenv()

# One-time init (Streamlit reruns the script on every interaction)
if "initialized" not in st.session_state:
    st.session_state.llm = ChatOpenAI(model="gpt-5")
    st.session_state.client = MultiServerMCPClient(SERVERS)
    tools = asyncio.run(st.session_state.client.get_tools())
    st.session_state.tools = tools
    st.session_state.tool_by_name = {t.name: t for t in tools}
    st.session_state.llm_with_tools = st.session_state.llm.bind_tools(tools)
    st.session_state.history = [SystemMessage(content=SYSTEM_PROMPT)]
    st.session_state.initialized = True

# Render history: user + final assistant messages only
for msg in st.session_state.history:
    if isinstance(msg, HumanMessage):
        with st.chat_message("user"):
            st.markdown(msg.content)
    elif isinstance(msg, AIMessage):
        if getattr(msg, "tool_calls", None):     # skip intermediate "calling tool..." messages
            continue
        with st.chat_message("assistant"):
            st.markdown(msg.content)

user_text = st.chat_input("Type a message…")
if user_text:
    with st.chat_message("user"):
        st.markdown(user_text)
    st.session_state.history.append(HumanMessage(content=user_text))

    first = asyncio.run(st.session_state.llm_with_tools.ainvoke(st.session_state.history))
    tool_calls = getattr(first, "tool_calls", None)

    if not tool_calls:
        with st.chat_message("assistant"):
            st.markdown(first.content or "")
        st.session_state.history.append(first)
    else:
        # ORDER MATTERS: AIMessage with tool_calls, then its ToolMessages, then the final answer
        st.session_state.history.append(first)
        tool_msgs = []
        for tc in tool_calls:
            args = tc.get("args") or {}
            if isinstance(args, str):
                try: args = json.loads(args)
                except Exception: pass
            res = asyncio.run(st.session_state.tool_by_name[tc["name"]].ainvoke(args))
            tool_msgs.append(ToolMessage(tool_call_id=tc["id"], content=json.dumps(res)))
        st.session_state.history.extend(tool_msgs)

        final = asyncio.run(st.session_state.llm.ainvoke(st.session_state.history))
        with st.chat_message("assistant"):
            st.markdown(final.content or "")
        st.session_state.history.append(AIMessage(content=final.content or ""))
```

What's new compared with the CLI version:

- **`st.session_state`** keeps the LLM, client, tools and chat history across Streamlit reruns.
- A **system prompt** stops the model narrating "let me call the tool...".
- **Full conversation history** is sent each turn, so it's a real multi-turn chat.
- **Message order** must be: `AIMessage(tool_calls)` → matching `ToolMessage`s → final answer. Break it and the OpenAI API rejects the request.
- Intermediate tool-call messages and tool results are kept in history but **not rendered**.

Run:

```bash
streamlit run client2.py
```

**Demo:** normal chat works; *"Show me all expenses from the last two weeks"* → the ₹800 grocery expense (via the remote server); *"Remainder of ... divided by 23"* → 13 (math server); *"Make an animation video of a circle using the Manim tool"* → video (Manim server). Three servers, one chatbot.

---

## Limitations and next steps

[39:16](https://www.youtube.com/watch?v=o4ajsc-tSBc&t=2356s)

Things to know about this client (beyond the video):

- **One round of tool calls only.** If the model needs a tool, looks at the result, and then wants *another* tool, this code won't do it. A real agent loops until the model stops asking for tools. LangGraph's prebuilt ReAct agent (or LangChain's agent helpers) does this for you, and accepts the tools from `client.get_tools()` directly.
- **Stateless sessions.** By default `MultiServerMCPClient` opens a **new MCP session per tool call**. Fine for stateless tools. For servers that keep per-session state, open a persistent session (`async with client.session("name") as session:` and load tools from it).
- **Absolute paths** in `command` still matter (same reason as Claude Desktop).
- **Remote auth** would go in the server entry (e.g. `headers` with a bearer token) once the server supports it.

**What's left in the playlist**, promised for later (after the LangGraph series): **authentication, sampling, elicitation, sessions**.

---

## Interview questions

**Q: What does `MultiServerMCPClient` do?**
Holds connection configs for several MCP servers (stdio or HTTP), runs the MCP handshake, and converts each server's tools into LangChain tools via `get_tools()`.

**Q: Walk through the tool-calling loop in a custom MCP client.**
Fetch tools → bind them to the LLM → send the prompt → if the response has `tool_calls`, run each tool via MCP (`tools/call`) → wrap each result in a `ToolMessage` with the matching `tool_call_id` → send prompt + AI message + tool messages back → get the final answer.

**Q: Why does a ToolMessage need `tool_call_id`?**
So the model can match each result to the specific call it requested, especially when there are several calls in one turn.

**Q: How do you connect a client to a local vs a remote server?**
Local: `transport: "stdio"` with the `command`/`args` to launch it (the client spawns it). Remote: `transport: "streamable_http"` with the server's `url`.

**Q: What breaks if you assume every response has a tool call?**
Normal questions return plain content with no `tool_calls`, so indexing `tool_calls[0]` crashes. Check first and return the content directly.

**Q: Why use LangChain MCP adapters instead of the raw SDK?**
Less code: it handles sessions and transports and returns tools in LangChain's format, so they work with `bind_tools` and LangGraph agents directly.
