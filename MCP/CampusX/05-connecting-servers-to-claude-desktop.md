# 05 — Using MCP: Connecting Servers to Claude Desktop

**Video:** [How to connect MCP Servers to Claude Desktop](https://www.youtube.com/watch?v=y-uPv3ltOTY) (46m)
**Links from the video:** [Claude Desktop](https://claude.ai/download) · [Manim MCP server](https://github.com/abhiemj/manim-mcp-server) · [Twitter MCP server](https://github.com/EnesCinr/twitter-mcp) · [Weather MCP server](https://github.com/adhikasp/mcp-weather) · [awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers)

First "How" video. No code yet: a **ready-made client** (Claude Desktop) talks to **ready-made servers**, so you can experience MCP before building anything. Along the way: the two ways of connecting a server (**connectors** vs the **JSON config file**) and why both exist.

---

## Contents

1. [The plan for the "How" part](#the-plan-for-the-how-part)
2. [Two ways to connect a server](#two-ways-to-connect-a-server)
3. [Why not make everything a connector?](#why-not-make-everything-a-connector)
4. [Installing Claude Desktop](#installing-claude-desktop)
5. [Server 1: File system (local, via connector)](#server-1-file-system-local-via-connector)
6. [Server 2: Manim (local, via config file)](#server-2-manim-local-via-config-file)
7. [Server 3: Google Drive (remote, via connector)](#server-3-google-drive-remote-via-connector)
8. [Server 4: Twitter/X (via config file)](#server-4-twitterx-via-config-file)
9. [Server 5: Weather (via config file)](#server-5-weather-via-config-file)
10. [Finding more servers](#finding-more-servers)
11. [Config file cheat sheet](#config-file-cheat-sheet)
12. [Gotchas](#gotchas)
13. [Interview questions](#interview-questions)

---

## The plan for the "How" part

[0:50](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=50s)

| Video | Client | Server |
|---|---|---|
| 05 (this) | Ready-made: Claude Desktop | Ready-made: existing servers |
| 06, 07 | Claude Desktop | **Our own** server (local, then remote) |
| 08 | **Our own** client | Our own + existing servers |

Plan for today: install Claude Desktop, then connect **two local** and **two remote** servers, one of each through a connector and one through the config file, to cover every combination. (As it turned out, the "remote" config-file servers were actually local. See below.)

---

## Two ways to connect a server

[4:08](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=248s)

### 1. Config file

AI hosts like Claude Desktop and Cursor have a **JSON config file**. You open it and add the server's details (copied from the server's README). This is the normal, original way.

### 2. Connectors

[4:59](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=299s)

> A **connector** is a built-in feature that links Claude to MCP servers automatically, without manual setup or configuration.

- Most Claude Desktop users are **non-technical**. They want Claude to "talk" to Notion, Drive, GitHub, Slack, without running servers, editing JSON or worrying about transports.
- A connector **wraps an MCP server behind the scenes** and handles **authentication via OAuth** (sign in with Google, GitHub...), API keys and so on. You click a button.
- Anthropic writes and maintains this code, so it's **safer, easier and more consistent**. Hand-edited configs sometimes fail to load; connectors work reliably.
- Think of connectors as the **"App Store" for MCP servers**: user-friendly, click-based, pre-curated.

Where to find them: the **search and tools** icon in the chat box → *Add connectors*. Two kinds:

- **Desktop extensions** → local MCP servers (e.g. File system)
- **Web** → remote MCP servers (e.g. Google Drive, Gmail, Calendar, Notion, Canva, Asana, Box, Hugging Face...)

---

## Why not make everything a connector?

[8:47](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=527s)

Two reasons:

1. **Connectors are curated and managed.** They're for popular SaaS tools. Anthropic writes, hosts and maintains the wrapper code, handles OAuth flows, security patches, rate limits, and keeps it stable. Thousands of MCP servers exist and 10–15 new ones appear daily. Anthropic can't build a connector for every one. It **doesn't scale**.
2. **MCP is an open standard.** Anyone can write a client or a server. If every server had to be reviewed and wrapped by Anthropic before it could be used, Anthropic would decide whose server runs. That would **close the ecosystem**.

> The **config file path exists to preserve freedom and extensibility.** Popular tools → connectors. Your own or your company's servers → config file, usable the same day you build them.

---

## Installing Claude Desktop

[12:40](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=760s)

Download from [claude.ai/download](https://claude.ai/download) for macOS or Windows, install, sign in. The UI looks like any chat app.

**Opening the config file:** *Settings → Developer → Edit Config*. This reveals `claude_desktop_config.json`:

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

> **Restart Claude Desktop after adding any new MCP server**, whether by connector or config file. Fully quit, don't just close the window.

---

## Server 1: File system (local, via connector)

[15:15](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=915s)

1. *Add connectors → Desktop extensions → Filesystem → Install*.
2. **Choose which directories it can access.** By default it gets nothing. He allowed only the **Desktop**. You could add Downloads or a project folder. This is a safety feature: giving it your whole disk risks real damage if something goes wrong.
3. Enable it, restart Claude.
4. Under the tools icon you now see all its tools (that's `tools/list` being called). You can switch individual tools off.

**Demo:**

- *"Are there any PDF files on my desktop?"* → Claude asks permission (**every tool use asks first**) → lists 5 PDFs.
- *"Write a Python program to print Fibonacci numbers"* → *"Write the first version in a .py file and save it on my desktop"* → file appears on the desktop.

**Real uses:** organize a messy Downloads folder, summarize a code project folder, bulk rename files, etc.

---

## Server 2: Manim (local, via config file)

[19:41](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=1181s)

**Manim** is the Python library 3Blue1Brown uses for his maths visualizations. Hand-coding those animations is hard. LLMs are good at writing Manim code, and with an MCP server you go from **an English sentence to a rendered video**: Claude writes the code, the Manim server runs it and returns the video.

This server is community-built, not official, and **local**: you clone it to your machine. No connector exists, so it goes in the config file.

### Steps

```bash
pip install manim
pip install mcp
cd ~/Desktop
git clone https://github.com/abhiemj/manim-mcp-server.git
```

Then in `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "manim-server": {
      "command": "/absolute/path/to/python3",
      "args": ["/absolute/path/to/manim-mcp-server/src/manim_server.py"],
      "env": {
        "MANIM_EXECUTABLE": "/absolute/path/to/manim"
      }
    }
  }
}
```

Finding the absolute paths:

```bash
which python3     # macOS/Linux  (Windows: where python)
which manim       # Windows: usually ...\Scripts\manim.exe
cd ~/Desktop/manim-mcp-server/src && pwd
```

Save, quit and restart Claude.

### Demo prompt

> Use the Manim server to create an animation showing the concept of **vector transformation in linear algebra**. First create a 2D coordinate grid with two basis vectors î and ĵ. Then apply this matrix transformation [...]. Show the whole grid bending and the new vectors. Add this title.

Claude wrote the Manim code, executed it, and returned a video: the grid transforming with î′ and ĵ′ shown. One issue: **LaTeX wasn't installed**, so maths symbols used a fallback. Install LaTeX (~10–15 GB) for perfect output.

> Tip: if a concept in ML/DL is hard, ask ChatGPT/Claude to write a Manim prompt for it, paste it into Claude Desktop, and get a visualization in minutes.

---

## Server 3: Google Drive (remote, via connector)

[30:25](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=1825s)

1. Tools icon → **Drive search** → sign in with Google (OAuth).
2. Done. Drive search is now enabled.

**Demo:** *"Summarize the AI newsletter content ideas document from my Google Drive"* → Claude searched Drive, found the doc, summarized it.

Note: this connector is **read-only**. You can read any document but can't create or edit files.

---

## Server 4: Twitter/X (via config file)

[33:30](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=2010s)

1. Create an **X developer account** and open the developer portal dashboard.
2. Default project → **Keys and tokens** → regenerate and copy: API key, API key secret, access token, access token secret.
3. Add to the config file **inside** the existing `mcpServers` object (comma after the previous entry):

```json
"twitter-mcp": {
  "command": "npx",
  "args": ["-y", "@enescinar/twitter-mcp"],
  "env": {
    "API_KEY": "...",
    "API_SECRET_KEY": "...",
    "ACCESS_TOKEN": "...",
    "ACCESS_TOKEN_SECRET": "..."
  }
}
```

4. If it doesn't work, install Node/npm (`npx` comes with it). Restart Claude.

Tools: `post_tweet`, `search_tweets`.

**Demo:** *"What are the top tweets on AI this week?"* → worked. *"Post a tweet saying hello from CampusX"* → **authentication error**: the app only had **read** permission.

> Fix: in the developer portal, *User authentication settings* → set app permissions to **Read and write**, then **regenerate the access token and secret** (old tokens keep the old permissions) and update the config.

**Local or remote?** He first called this remote, then corrected himself: `npx` downloads the package and **runs it on your machine** over stdio. It's a **local** server that calls the X API.

---

## Server 5: Weather (via config file)

[40:25](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=2425s)

1. Install **uv** (`pip install uv`).
2. Get a free **AccuWeather API key** (free tier ~15 days).
3. Add:

```json
"weather": {
  "command": "/absolute/path/to/uvx",
  "args": ["--from", "git+https://github.com/adhikasp/mcp-weather.git", "mcp-weather"],
  "env": { "ACCUWEATHER_API_KEY": "..." }
}
```

`which uvx` gives the path. Restart.

**Demo:** *"What's the current weather in Gurgaon?"* → the weather tool errored (an API-side issue on his machine), so Claude **fell back to web search** and answered anyway.

> **Correction:** the video calls this a remote server because the code comes from a GitHub URL. It isn't. `uvx --from git+https://...` **downloads the code and runs it locally**, talking to Claude over **stdio**. Where the code is *fetched from* doesn't matter; where it *runs* does. A true remote server runs on someone else's machine and you connect to a URL (like the Drive connector, or the server you deploy in chapter 07).

---

## Finding more servers

[44:45](https://www.youtube.com/watch?v=y-uPv3ltOTY&t=2685s)

Search **"awesome MCP servers"** → [punkpeye/awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers), an up-to-date list organized by category. The Manim server came from here. Other directories and marketplaces exist too.

Homework: install Claude Desktop, think of a workflow you'd like to automate (like the newsletter in chapter 01) and build it with MCP.

---

## Config file cheat sheet

```json
{
  "mcpServers": {
    "<any-name>": {
      "command": "<program to launch>",
      "args": ["<arg1>", "<arg2>"],
      "env": { "SECRET": "value" }
    },
    "<another-name>": { "...": "..." }
  }
}
```

| Field | Meaning |
|---|---|
| key (`"manim-server"`) | Display name you choose |
| `command` | Executable the host launches as a subprocess (`python3`, `npx`, `uvx`, `uv`, `node`...). Absolute paths avoid PATH problems |
| `args` | Arguments: script path, package name, flags |
| `env` | Environment variables for the server, typically API keys |

Every `command` entry = a **local stdio server**. The host spawns it (chapter 03). Common launcher patterns:

| Launcher | Pattern | Where the code comes from |
|---|---|---|
| `python3` | `python3 /path/server.py` | Local clone |
| `npx -y <pkg>` | npm package | Downloaded from npm, run locally |
| `uvx <pkg>` / `uvx --from git+URL <entry>` | Python package | Downloaded from PyPI/GitHub, run locally |
| `uv run fastmcp run main.py` | Your own FastMCP project | Your folder (chapter 06) |

---

## Gotchas

- **Restart** Claude Desktop after every config change.
- Paste new entries **inside** the `mcpServers` braces, with commas between entries. Invalid JSON = no servers load.
- Use **absolute paths** for `command` and script paths. Claude Desktop doesn't inherit your shell's PATH.
- Each tool call asks for **permission** (Allow once / Always allow).
- File-system access is limited to directories you **explicitly allow**.
- Don't commit API keys. The video shows real keys and deletes them after recording.

---

## Interview questions

**Q: What are Claude Desktop connectors?**
Built-in, Anthropic-maintained integrations that wrap an MCP server, handle auth (usually OAuth) and connect with a click. Like an app store for popular MCP servers.

**Q: Why does the config-file method still exist?**
Connectors are curated and can't scale to thousands of servers, and MCP is an open standard. The config file lets anyone connect any server immediately without approval.

**Q: What does each config-file entry tell the host?**
The command to launch the server process, its arguments, and environment variables (usually secrets). The host spawns it and talks to it over stdio.

**Q: Is a server installed with `npx` or `uvx` local or remote?**
Local. The package is downloaded and executed on your machine; only the code's origin is remote.

**Q: Why restrict the file-system server to certain directories?**
Least privilege. An AI with write access to your entire disk can cause serious damage by mistake.
