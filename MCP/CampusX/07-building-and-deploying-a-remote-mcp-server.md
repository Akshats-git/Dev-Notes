# 07 — Building and Deploying a Remote MCP Server

**Video:** [How to Build & Deploy Remote MCP Servers](https://www.youtube.com/watch?v=GF7-ZzUausU) (46m)
**Code:** [campusx-official/test-remote-mcp-server](https://github.com/campusx-official/test-remote-mcp-server) · **Quick revision:** [Google Doc](https://docs.google.com/document/d/1QyQKI8s23L3QQQwaq6sabsw4dqbYJngXswK-94tsL78/edit) · **Mind map:** [Whimsical](https://whimsical.com/remote-mcp-servers-XvEKLRbGEZU8wEUPzmSBgo)

Take the local expense tracker from chapter 06, make it a **remote** server (HTTP transport), **deploy it on FastMCP Cloud**, connect it to Claude Desktop with a **custom connector**, then fix the flaws that only show up once it's on the internet: a read-only database, blocking code, free-plan access, and the big one, **no per-user data or authentication**.

---

## Contents

1. [Local vs remote servers](#local-vs-remote-servers)
2. [Plan](#plan)
3. [Step 1: a simple remote server](#step-1-a-simple-remote-server)
4. [Step 2: deploy on FastMCP Cloud](#step-2-deploy-on-fastmcp-cloud)
5. [Step 3: connect with a custom connector](#step-3-connect-with-a-custom-connector)
6. [Step 4: move the expense tracker over](#step-4-move-the-expense-tracker-over)
7. [Fix 1: database is read-only](#fix-1-database-is-read-only)
8. [Fix 2: make the server async](#fix-2-make-the-server-async)
9. [Fix 3: free plan workaround with a local proxy](#fix-3-free-plan-workaround-with-a-local-proxy)
10. [Flaw 4: everyone sees everyone's expenses](#flaw-4-everyone-sees-everyones-expenses)
11. [Final remote server code](#final-remote-server-code)
12. [Cheat sheet](#cheat-sheet)
13. [Interview questions](#interview-questions)

---

## Local vs remote servers

[0:00](https://www.youtube.com/watch?v=GF7-ZzUausU&t=0s)

| | Local | Remote |
|---|---|---|
| Runs on | Same machine as host and client | Another machine, usually on the internet |
| Speed | **Faster**: communication stays on one machine | **Slower**: network latency |
| Clients | Just you | **Many clients at once** |
| Compute | Your laptop | Powerful servers: can take **compute-heavy tasks** |
| Transport | stdio | Streamable HTTP |

> Rule of thumb: **enterprise MCP servers (big companies' servers) will mostly be remote.** So this is the important case.

---

## Plan

[2:50](https://www.youtube.com/watch?v=GF7-ZzUausU&t=170s)

1. Build a simple remote server (add numbers, random number).
2. Replace its code with the expense tracker.
3. Deploy with **FastMCP Cloud** (free at the time; AWS, Render etc. also work).
4. Find the flaws and fix them.

**Code-wise almost nothing changes** between a local and a remote FastMCP server. The difference is one line.

---

## Step 1: a simple remote server

[4:50](https://www.youtube.com/watch?v=GF7-ZzUausU&t=290s)

```bash
pip install uv
mkdir test-remote-server && cd test-remote-server     # open in VS Code
uv init .
uv add fastmcp
```

`main.py` (simple version, two tools and one resource):

```python
import json, random
from fastmcp import FastMCP

mcp = FastMCP("Simple Calculator Server")

@mcp.tool
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b

@mcp.tool
def random_number(min_val: int = 1, max_val: int = 100) -> int:
    """Generate a random number within a range."""
    return random.randint(min_val, max_val)

@mcp.resource("info://server")
def server_info() -> str:
    """Get information about this server."""
    return json.dumps({"name": "Simple Calculator Server", "version": "1.0.0",
                       "tools": ["add", "random_number"]})

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)   # ← the only real change
```

**The one change that makes it remote:**

| Local (chapter 06) | Remote |
|---|---|
| `mcp.run()` → default transport **stdio** | `mcp.run(transport="http", host="0.0.0.0", port=8000)` → **Streamable HTTP** |

- `host="0.0.0.0"`: accept requests from any IP address, not just localhost.
- `port`: where it listens. The MCP endpoint is `http://<host>:8000/mcp`.

**Run it:**

```bash
uv run main.py
# or: uv run fastmcp run main.py --transport http --host 0.0.0.0 --port 8000
```

Output shows the server name, transport **Streamable HTTP**, and the server URL.

**Debug it:** in another terminal,

```bash
uv run fastmcp dev main.py
```

In the Inspector choose **Streamable HTTP**, connect, then *Resources → List* (shows `server_info`) and *Tools → List* (shows `add`, `random_number`). `random_number(10, 100)` → 22. Working.

---

## Step 2: deploy on FastMCP Cloud

[11:55](https://www.youtube.com/watch?v=GF7-ZzUausU&t=715s)

FastMCP has its own hosting service, **FastMCP Cloud** (`fastmcp.cloud`), free at the time of the video. It deploys **from a GitHub repo**.

> Hosting offerings change. If the URL or name has moved on, any host that can run a Python web process (Render, Railway, Fly.io, AWS, a VPS) works the same way: run the server with the HTTP transport and expose the port.

**1. Push the code to GitHub.** Create a repo (`test-remote-mcp-server`), then:

```bash
git init                                    # uv init usually did this already
git status
git add .
git commit -m "initial commit"
git remote add origin <repo-url>
git push origin main
```

**2. Deploy.**

1. Sign in to FastMCP Cloud → **Deploy from your own code**.
2. Connect your GitHub account (a few OK prompts).
3. Pick the repo → **Deploy**.
4. Settings:
   - **Server name**: becomes part of the URL (random by default; use your or your company's name).
   - **Entry point**: `main.py`.
   - **Authentication**: off for now.
   - **Discoverable**: lists your server on FastMCP's curated page (not needed).
5. Wait for the build. You get a URL like `https://<name>.fastmcp.app/mcp`.

**Auto-redeploy:** every new commit pushed to the repo triggers a new build.

---

## Step 3: connect with a custom connector

[15:05](https://www.youtube.com/watch?v=GF7-ZzUausU&t=905s)

Share the URL with anyone. In Claude Desktop:

1. *Settings → Connectors → Add custom connector* (at the bottom).
2. Name (`Nitish Test MCP Server`) + the URL → **Add**.
3. Restart Claude. The connector shows tools `add` and `random_number`.

*"Generate random number between 11 and 20"* → permission prompt → a number. That's a remote MCP server built, deployed and used, in minutes.

> At recording time **Add custom connector was Pro-plan only** (beta). Check current plan limits; see [Fix 3](#fix-3-free-plan-workaround-with-a-local-proxy) for a workaround that works on any plan.

> Note: custom connectors are **remote servers by URL**. That's different from the config file in chapter 05, which launches **local** processes.

---

## Step 4: move the expense tracker over

[18:29](https://www.youtube.com/watch?v=GF7-ZzUausU&t=1109s)

Work smart: don't redo the whole process.

1. Paste the chapter 06 expense tracker code into this project's `main.py`, keeping the HTTP `mcp.run(...)` line.
2. Copy `categories.json` into the project too.
3. `uv run main.py` once (creates `expenses.db`), then `uv run fastmcp dev main.py` to inspect.

Tools and resources were discovered. Calling `list_expenses` from the Inspector form gave *"start_date is a required property"*: a formatting/type issue, because the functions have **no type hints**, so the schema is vague. (Fix: add `start_date: str, end_date: str` type hints. More reason to always type your tools.)

4. Push and let FastMCP Cloud redeploy:

```bash
git add . && git commit -m "Expense tracking code added" && git push origin main
```

He pushed `expenses.db` too, then noted you shouldn't: it's created at runtime, so add it to `.gitignore`.

**Test in Claude** (restart first): tools `add_expense`, `list_expenses`, `summarize` appear. *"Tell me about all my expenses in September"* → none (fresh DB). *"Add an expense: Navratri dinner last night, 1000"* → **fails**.

---

## Fix 1: database is read-only

[25:10](https://www.youtube.com/watch?v=GF7-ZzUausU&t=1510s)

> "I'm unable to add the expense because the database is in read-only mode."

On the hosting platform, the app's code directory isn't writable, so SQLite can't write `expenses.db` next to `main.py`. The fix (suggested by ChatGPT) puts the DB in the system **temp directory**, which is writable, and tests write access at startup:

```python
import tempfile, os
TEMP_DIR = tempfile.gettempdir()
DB_PATH = os.path.join(TEMP_DIR, "expenses.db")

def init_db():
    import sqlite3
    with sqlite3.connect(DB_PATH) as c:
        c.execute("PRAGMA journal_mode=WAL")
        c.execute("""CREATE TABLE IF NOT EXISTS expenses(...)""")
        # test write access
        c.execute("INSERT OR IGNORE INTO expenses(date, amount, category) VALUES ('2000-01-01', 0, 'test')")
        c.execute("DELETE FROM expenses WHERE category = 'test'")
```

Push → redeploy → *"Trip to railway station, 15 September, cab ride, ₹500"* → added. *"Show all my September expenses"* → both entries. Reads and writes work.

> **Important caveat (not in the video):** a temp directory on a cloud host is **ephemeral**. It's wiped when the container restarts or redeploys, so all expenses vanish with each new commit. Fine for a demo. For real use, store data in a **managed database** (Postgres on Neon/Supabase/RDS, etc.) and read its URL from an environment variable.

---

## Fix 2: make the server async

[29:40](https://www.youtube.com/watch?v=GF7-ZzUausU&t=1780s)

**Flaw:** every tool and every DB call is **synchronous (blocking)**. While one user's `add_expense` is talking to the database, the server can't serve anyone else; they queue up. Fine for a local, single-user server; bad for a shared remote one.

**Fix:** use Python's `async`/`await`:

1. Make the tools `async def`.
2. Replace `sqlite3` with **`aiosqlite`** (AIO = asynchronous I/O) so DB operations don't block the event loop.

```bash
uv add aiosqlite
```

```python
import aiosqlite

@mcp.tool()
async def add_expense(date, amount, category, subcategory="", note=""):
    '''Add a new expense entry to the database.'''
    async with aiosqlite.connect(DB_PATH) as c:
        cur = await c.execute(
            "INSERT INTO expenses(date, amount, category, subcategory, note) VALUES (?,?,?,?,?)",
            (date, amount, category, subcategory, note)
        )
        expense_id = cur.lastrowid
        await c.commit()                       # aiosqlite needs an explicit commit
        return {"status": "success", "id": expense_id}
```

`init_db()` stays synchronous (it runs once at startup). Push → the deploy failed twice on small bugs, fixed, then succeeded. *"Online course subscription last Saturday, 500"* → works. (You can't see a speed-up with one user, but the server no longer blocks.)

> Read up on [async IO in Python](https://realpython.com/async-io-python/) if `async`/`await` is new.

---

## Fix 3: free plan workaround with a local proxy

[35:42](https://www.youtube.com/watch?v=GF7-ZzUausU&t=2142s)

Free-plan users (at the time) couldn't add a custom connector. FastMCP's workaround: a **proxy server**.

```
Claude Desktop ──stdio──► local proxy server ──HTTP──► remote MCP server (FastMCP Cloud)
               ◄────────                    ◄───────
```

The proxy is a **local** server (allowed on any plan, via the config file) that forwards everything to the remote server.

```python
# proxy.py  (new uv project, same setup as chapter 06)
from fastmcp import FastMCP

# FastMCP Cloud uses Streamable HTTP, so just use the /mcp URL
mcp = FastMCP.as_proxy(
    "https://<your-server>.fastmcp.app/mcp",
    name="Nitish Server Proxy",
)

if __name__ == "__main__":
    mcp.run()          # stdio → it's a local server
```

Steps:

1. Remove the custom connector from Claude Desktop (so we know the proxy is doing the work).
2. `uv run fastmcp dev proxy.py` → Inspector, transport **STDIO** → List tools shows `add_expense`, `list_expenses`, `summarize` (slight delay: it's fetching from the remote).
3. `uv run fastmcp install claude-desktop proxy.py`.
4. Claude says the proxy failed to start → same fix as before: put the **absolute path of `uv`** (`which uv`) into the config entry.
5. Restart. *"Add an expense: grocery last Saturday, 500"* → *"List all September expenses"* → works, just a bit slower (indirect hop).

> Another common option for the same job: `npx mcp-remote https://<server>/mcp` as the config-file command, which bridges stdio to a remote HTTP server.

---

## Flaw 4: everyone sees everyone's expenses

[41:42](https://www.youtube.com/watch?v=GF7-ZzUausU&t=2502s)

The big logical flaw. The table has **no `user_id`**. It's one central database: anyone who connects adds to it, and *"show my September expenses"* returns **everyone's** expenses. The server can't tell users apart.

Adding a `user_id` column is easy. The hard part: there's **no login**. The LLM talks straight to the database. If a user says "I'm user 123", why believe them? You need **authentication** in the flow (MCP supports OAuth-based authorization for remote servers).

> As it stands this is **not a good remote server**. It's fine as a **local** server (only you use it), but you can't safely share it. Authentication, plus advanced features (sessions, sampling, elicitation), is promised for later videos.

---

## Final remote server code

From the course repo (`main.py`), lightly trimmed:

```python
from fastmcp import FastMCP
import os
import aiosqlite
import tempfile

TEMP_DIR = tempfile.gettempdir()                       # writable on the host
DB_PATH = os.path.join(TEMP_DIR, "expenses.db")
CATEGORIES_PATH = os.path.join(os.path.dirname(__file__), "categories.json")

mcp = FastMCP("ExpenseTracker")

def init_db():                                         # sync, runs once at startup
    import sqlite3
    with sqlite3.connect(DB_PATH) as c:
        c.execute("PRAGMA journal_mode=WAL")
        c.execute("""
            CREATE TABLE IF NOT EXISTS expenses(
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                date TEXT NOT NULL,
                amount REAL NOT NULL,
                category TEXT NOT NULL,
                subcategory TEXT DEFAULT '',
                note TEXT DEFAULT ''
            )
        """)
        c.execute("INSERT OR IGNORE INTO expenses(date, amount, category) VALUES ('2000-01-01', 0, 'test')")
        c.execute("DELETE FROM expenses WHERE category = 'test'")

init_db()

@mcp.tool()
async def add_expense(date, amount, category, subcategory="", note=""):
    '''Add a new expense entry to the database.'''
    try:
        async with aiosqlite.connect(DB_PATH) as c:
            cur = await c.execute(
                "INSERT INTO expenses(date, amount, category, subcategory, note) VALUES (?,?,?,?,?)",
                (date, amount, category, subcategory, note)
            )
            expense_id = cur.lastrowid
            await c.commit()
            return {"status": "success", "id": expense_id, "message": "Expense added successfully"}
    except Exception as e:
        if "readonly" in str(e).lower():
            return {"status": "error", "message": "Database is in read-only mode. Check file permissions."}
        return {"status": "error", "message": f"Database error: {str(e)}"}

@mcp.tool()
async def list_expenses(start_date, end_date):
    '''List expense entries within an inclusive date range.'''
    try:
        async with aiosqlite.connect(DB_PATH) as c:
            cur = await c.execute(
                """
                SELECT id, date, amount, category, subcategory, note
                FROM expenses
                WHERE date BETWEEN ? AND ?
                ORDER BY date DESC, id DESC
                """,
                (start_date, end_date)
            )
            cols = [d[0] for d in cur.description]
            return [dict(zip(cols, r)) for r in await cur.fetchall()]
    except Exception as e:
        return {"status": "error", "message": f"Error listing expenses: {str(e)}"}

@mcp.tool()
async def summarize(start_date, end_date, category=None):
    '''Summarize expenses by category within an inclusive date range.'''
    try:
        async with aiosqlite.connect(DB_PATH) as c:
            query = """
                SELECT category, SUM(amount) AS total_amount, COUNT(*) as count
                FROM expenses
                WHERE date BETWEEN ? AND ?
            """
            params = [start_date, end_date]
            if category:
                query += " AND category = ?"
                params.append(category)
            query += " GROUP BY category ORDER BY total_amount DESC"

            cur = await c.execute(query, params)
            cols = [d[0] for d in cur.description]
            return [dict(zip(cols, r)) for r in await cur.fetchall()]
    except Exception as e:
        return {"status": "error", "message": f"Error summarizing expenses: {str(e)}"}

@mcp.resource("expense:///categories", mime_type="application/json")   # note: three slashes here
def categories():
    try:
        with open(CATEGORIES_PATH, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        import json
        return json.dumps({"categories": ["Food & Dining", "Transportation", "Shopping",
                                          "Entertainment", "Bills & Utilities", "Healthcare",
                                          "Travel", "Education", "Business", "Other"]}, indent=2)

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)
```

Changes from the local version: HTTP transport, DB in the temp dir with a write test, `aiosqlite` + `async` tools with explicit `commit()`, try/except returning error dicts, `summarize` also returns `count` and sorts by total, and a fallback category list.

---

## Cheat sheet

| Task | How |
|---|---|
| Make a server remote | `mcp.run(transport="http", host="0.0.0.0", port=8000)` |
| Run it | `uv run main.py` or `uv run fastmcp run main.py --transport http --port 8000` |
| Inspect | `uv run fastmcp dev main.py` → transport *Streamable HTTP* |
| Deploy | Push to GitHub → FastMCP Cloud → *Deploy from your own code* → entry point `main.py` |
| Redeploy | Push a new commit |
| Connect (Pro, at the time) | Claude *Settings → Connectors → Add custom connector* → name + URL |
| Connect (any plan) | Local proxy: `FastMCP.as_proxy(url)` + `fastmcp install claude-desktop`, fix `uv` path |
| Non-blocking DB | `uv add aiosqlite`, `async def` tools, `await c.commit()` |

**Checklist before sharing a remote server:** writable *and persistent* storage, async I/O, typed tool inputs, error handling, **authentication + per-user data**, secrets in env vars, DB files in `.gitignore`.

---

## Interview questions

**Q: What changes in code between a local and remote FastMCP server?**
Essentially the transport: `mcp.run()` (stdio) becomes `mcp.run(transport="http", host="0.0.0.0", port=...)`. Tools and resources are written the same way.

**Q: Pros and cons of remote servers?**
Pros: many concurrent clients, more compute, shareable via URL. Cons: network latency, and you must handle auth, multi-tenancy, persistence and security.

**Q: Why did the deployed SQLite DB become read-only, and what's the proper fix?**
The app directory on the host isn't writable. The demo moved the DB to the temp dir, but that's ephemeral. The proper fix is a managed external database.

**Q: Why make the tools async?**
Synchronous DB calls block the server so concurrent users queue. Async tools with an async driver (`aiosqlite`) let the event loop serve other requests while waiting on I/O.

**Q: How can a client that only supports local (stdio) servers use a remote server?**
Run a local proxy that speaks stdio to the client and HTTP to the remote server, e.g. `FastMCP.as_proxy(url)` or `mcp-remote`.

**Q: What's the main security flaw in the deployed expense tracker?**
No authentication or user scoping. All users share one table, so anyone can read and write everyone's data. It needs auth (e.g. OAuth) and a `user_id` on every row, enforced server-side.
