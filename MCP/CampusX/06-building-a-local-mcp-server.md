# 06 — Building a Local MCP Server: Expense Tracker with FastMCP

**Video:** [How to Build Local MCP Servers](https://www.youtube.com/watch?v=tc2oOznpdE0) (1h 12m)
**Code:** [campusx-official/expense-tracker-mcp-server](https://github.com/campusx-official/expense-tracker-mcp-server) · **Quick revision:** [Google Doc](https://docs.google.com/document/d/1xLHPVi07Jy3vZUjDNnC7BivUpN1Ro0aYDxP09yD3pzs/edit)

Build your own MCP server. First a toy demo server to learn the workflow (install → write → inspect → run → connect to Claude Desktop), then an **intermediate-level, actually useful** one: an **expense tracker** you talk to in plain English, backed by SQLite. It's a **local** server: it runs on your machine and talks to Claude Desktop over **stdio**. Chapter 07 turns it into a remote server.

---

## Contents

1. [The goal: expense tracking in natural language](#the-goal-expense-tracking-in-natural-language)
2. [MCP SDK vs FastMCP: clearing the confusion](#mcp-sdk-vs-fastmcp-clearing-the-confusion)
3. [Demo server: the full workflow](#demo-server-the-full-workflow)
    - [Setup with uv](#setup-with-uv)
    - [Write the server](#write-the-server)
    - [Test with MCP Inspector](#test-with-mcp-inspector)
    - [Run it](#run-it)
    - [Install into Claude Desktop](#install-into-claude-desktop)
4. [Expense tracker v1: add and list](#expense-tracker-v1-add-and-list)
5. [v2: list by date range](#v2-list-by-date-range)
6. [v3: summarize](#v3-summarize)
7. [v4: consistent categories with a resource](#v4-consistent-categories-with-a-resource)
8. [Final server code](#final-server-code)
9. [Ideas to extend it](#ideas-to-extend-it)
10. [FastAPI → MCP server in one line](#fastapi--mcp-server-in-one-line)
11. [Command cheat sheet](#command-cheat-sheet)
12. [Interview questions](#interview-questions)

---

## The goal: expense tracking in natural language

[2:01](https://www.youtube.com/watch?v=tc2oOznpdE0&t=121s)

Expense apps make you fill forms and read graphs. With an MCP server behind Claude, you just say:

- *"Add 500 travel expense for cab ride yesterday"* → Claude works out the date, category and a note, and calls `add_expense`.
- *"Show all expenses from September in a table"* → Claude converts "September" to `2025-09-01 … 2025-09-30` and calls `list_expenses`.
- *"How much did I spend on health in August?"* → `summarize` with `category=health`.
- *"How much on travel this month?"*

In the demo it also noticed transport expenses had been split across two categories. That's the motivation for the "consistent categories" fix later.

**Roadmap:** demo server → expense tracker built incrementally (add/list → date range → summarize → categories resource) → next video: make it remote and deploy it.

---

## MCP SDK vs FastMCP: clearing the confusion

[13:59](https://www.youtube.com/watch?v=tc2oOznpdE0&t=839s)

MCP is a **protocol** (a set of rules). Implementing it from scratch in Python is complex for beginners and redundant (you'd rewrite the same rules for every server). So you use a library. YouTube tutorials use two names, **MCP SDK** and **FastMCP**, with near-identical code. Here's why:

| When | What happened |
|---|---|
| **Nov 2024** | Anthropic released MCP. People started implementing it by hand: complex and repetitive |
| | Anthropic released the official **Python MCP SDK** (`pip install "mcp[cli]"`) with sub-modules `mcp.server`, `mcp.client`, `mcp.cli` |
| | The first SDK was **verbose and boilerplate-heavy**: a simple "add two numbers" server needed lots of code, with transports handled manually |
| | **Jeremiah Lowin** (CEO of Prefect, an Airflow-like orchestration tool) built **FastMCP**, a beginner-friendly abstraction on top of the SDK |
| | FastMCP was so popular that the **official SDK adopted it**: `from mcp.server.fastmcp import FastMCP` (that's FastMCP 1.0) |
| **2025** | The two split. **FastMCP 2.0** became an independent, much larger library (`pip install fastmcp`) with extra features (client, proxies, OpenAPI/FastAPI integration, auth, deployment). The SDK still ships the 1.0 version internally |

> The video says MCP arrived "end of 2023". It was actually announced in **November 2024**.

**Analogies:**

- TensorFlow was hard → **Keras** on top → Keras became TensorFlow 2's main API.
- **WSGI** is the low-level spec for Python web apps → **Flask** made it easy → everyone uses Flask.

So: **MCP SDK ≈ WSGI (low-level spec)**, **FastMCP ≈ Flask (developer-friendly)**. The course uses **FastMCP 2.x**. The code you write looks the same either way.

Side note: FastMCP's naming and design deliberately echo **FastAPI** (more on that [at the end](#fastapi--mcp-server-in-one-line)).

---

## Demo server: the full workflow

[26:00](https://www.youtube.com/watch?v=tc2oOznpdE0&t=1560s)

Two tools: **roll dice** (values 1–6) and **add two numbers**. The point isn't the server, it's the process.

### Setup with uv

**uv** is a new Python package manager, much faster than pip, and FastMCP recommends it.

```bash
pip install uv                       # once
mkdir expense-tracker-mcp-server     # on the Desktop
cd expense-tracker-mcp-server        # open this folder in VS Code
uv init .                            # creates pyproject.toml, main.py, etc.
uv add fastmcp                       # (pip equivalent: pip install fastmcp)
uv run fastmcp version               # check: FastMCP 2.x, MCP SDK version, Python, platform
```

### Write the server

[29:00](https://www.youtube.com/watch?v=tc2oOznpdE0&t=1740s)

```python
# main.py
import random
from fastmcp import FastMCP

mcp = FastMCP(name="Demo Server")          # 1. create a server instance

@mcp.tool                                   # 2. any Python function + decorator = MCP tool
def roll_dice(n_dice: int = 1) -> list[int]:
    """Roll n_dice six-sided dice and return the results."""
    return [random.randint(1, 6) for _ in range(n_dice)]

@mcp.tool
def add_numbers(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b

if __name__ == "__main__":
    mcp.run()                               # 3. default transport = stdio
```

That's all of it: **create `FastMCP` → decorate functions with `@mcp.tool` → `mcp.run()`**.

> FastMCP builds each tool's **input schema from the type hints** and its **description from the docstring**. Those are exactly what the LLM sees in `tools/list`, so write clear docstrings and add type hints. (`@mcp.tool` and `@mcp.tool()` both work in FastMCP 2.x.)

### Test with MCP Inspector

[31:34](https://www.youtube.com/watch?v=tc2oOznpdE0&t=1894s)

**MCP Inspector** is Anthropic's debugging UI for MCP servers: **like Postman, but for MCP servers**. It comes with FastMCP (it launches via `npx`, so Node must be installed).

```bash
uv run fastmcp dev main.py
```

In the browser:

1. Transport type: **STDIO** (server and client on the same machine). Click **Connect**.
2. You see the three primitives: **Resources, Prompts, Tools**. This server only has tools.
3. **Tools → List Tools** → sends `tools/list` → shows `roll_dice`, `add_numbers`.
4. Pick `add_numbers`, enter two numbers, **Run Tool** → result `11`.
5. The **History** panel shows every JSON-RPC message (the `initialize` handshake, capability exchange, `tools/list`, `tools/call`). There are also panels for ping, sampling, elicitation, auth.

> Test in the Inspector **before** connecting to Claude. Debugging inside Claude is much harder.

### Run it

[34:13](https://www.youtube.com/watch?v=tc2oOznpdE0&t=2053s)

```bash
uv run fastmcp run main.py
```

Prints the server name, transport, FastMCP and SDK versions. Any local client could now connect. We don't have our own client yet, so we install it into Claude Desktop instead.

### Install into Claude Desktop

[35:34](https://www.youtube.com/watch?v=tc2oOznpdE0&t=2134s)

```bash
uv run fastmcp install claude-desktop main.py
# options: --name "My Server", --env KEY=value, ...
```

This writes an entry into `claude_desktop_config.json`. Restart Claude Desktop.

**Common problem:** Claude says it can't connect to the server. Open *Settings → Developer → Edit Config* and look at the entry: the `command` is just `"uv"`, and Claude Desktop can't find it on its PATH.

**Fix:** replace it with the absolute path.

```bash
which uv      # e.g. /Library/Frameworks/Python.framework/Versions/3.11/bin/uv
```

```json
{
  "mcpServers": {
    "Demo Server": {
      "command": "/absolute/path/to/uv",
      "args": ["run", "--with", "fastmcp", "fastmcp", "run", "/absolute/path/to/main.py"]
    }
  }
}
```

Save, **fully quit** Claude, reopen. Now *"Roll a die"* → 2, *"Roll two dice"* → 4 and 3, *"Add 234 and 567"* → 801.

---

## Expense tracker v1: add and list

[40:30](https://www.youtube.com/watch?v=tc2oOznpdE0&t=2430s)

Three core features: **add expense**, **list expenses**, **summarize**. (Suggested extras: edit, delete, credits/income, budgets. See below.)

**Storage: SQLite.** In production you'd use Postgres/MySQL/Oracle, but SQLite needs no setup and the DB file sits in the project folder (`expenses.db`), so you can open it and watch rows appear.

Same skeleton as the demo. What's new:

```python
import os, sqlite3
from fastmcp import FastMCP

DB_PATH = os.path.join(os.path.dirname(__file__), "expenses.db")
mcp = FastMCP("ExpenseTracker")

def init_db():
    with sqlite3.connect(DB_PATH) as c:
        c.execute("""
            CREATE TABLE IF NOT EXISTS expenses(
                id INTEGER PRIMARY KEY AUTOINCREMENT,   -- unique per transaction
                date TEXT NOT NULL,                     -- when
                amount REAL NOT NULL,                   -- how much
                category TEXT NOT NULL,                 -- e.g. travel
                subcategory TEXT DEFAULT '',            -- e.g. cab_ride
                note TEXT DEFAULT ''                    -- e.g. "cab ride to airport"
            )
        """)

init_db()          # create the table if it doesn't exist

@mcp.tool()
def add_expense(date, amount, category, subcategory="", note=""):
    '''Add a new expense entry to the database.'''
    with sqlite3.connect(DB_PATH) as c:
        cur = c.execute(
            "INSERT INTO expenses(date, amount, category, subcategory, note) VALUES (?,?,?,?,?)",
            (date, amount, category, subcategory, note)
        )
        return {"status": "ok", "id": cur.lastrowid}

@mcp.tool()
def list_expenses():
    '''List all expense entries from the database.'''
    with sqlite3.connect(DB_PATH) as c:
        cur = c.execute("SELECT id, date, amount, category, subcategory, note FROM expenses ORDER BY id ASC")
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]

if __name__ == "__main__":
    mcp.run()
```

- `sqlite3.connect` → cursor → `execute(SQL)`. Plain DB code; paste it into ChatGPT for a line-by-line explanation if it's unfamiliar.
- `?` placeholders = **parameterized queries** (safe against SQL injection).
- **Always give MCP tools a description** (the docstring).

Paste into `main.py`, run `uv run fastmcp install claude-desktop main.py` again, fix the `uv` path again, restart Claude.

**Demo:**

- *"Add an expense: cab ride to Delhi last Sunday, fare was ₹500"* → Clicked **Always allow** (the server is harmless).
- *"Add expense groceries yesterday for 500"* → no permission prompt this time.
- Open `expenses.db`: two rows. Claude worked out the dates ("yesterday" → 26th because today is the 27th) and notes. But the **subcategory column was empty**: Claude ignored it. Remember that.

---

## v2: list by date range

[49:51](https://www.youtube.com/watch?v=tc2oOznpdE0&t=2991s)

Listing everything isn't useful. You want "last month", "last week", "yesterday". Add `start_date` and `end_date` and a `WHERE` clause:

```python
@mcp.tool()
def list_expenses(start_date, end_date):
    '''List expense entries within an inclusive date range.'''
    with sqlite3.connect(DB_PATH) as c:
        cur = c.execute(
            """
            SELECT id, date, amount, category, subcategory, note
            FROM expenses
            WHERE date BETWEEN ? AND ?
            ORDER BY id ASC
            """,
            (start_date, end_date)
        )
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]
```

Replace the old function, restart Claude (**required after any server change**).

- *"Show me all expenses from September first week"* → Claude passes `2025-09-01` and `2025-09-07`.
- *"List my expenses from the last week in a tabular manner"* → 4 transactions, ₹3,500.

> Dates are stored as `TEXT`. `BETWEEN` works because ISO dates (`YYYY-MM-DD`) sort correctly as strings, so keep the format consistent.

---

## v3: summarize

[52:42](https://www.youtube.com/watch?v=tc2oOznpdE0&t=3162s)

Questions like *"How much did I spend on transport last week?"* need totals for a date range, optionally for one category.

```python
@mcp.tool()
def summarize(start_date, end_date, category=None):
    '''Summarize expenses by category within an inclusive date range.'''
    with sqlite3.connect(DB_PATH) as c:
        query = """
            SELECT category, SUM(amount) AS total_amount
            FROM expenses
            WHERE date BETWEEN ? AND ?
        """
        params = [start_date, end_date]

        if category:                       # optional filter
            query += " AND category = ?"
            params.append(category)

        query += " GROUP BY category ORDER BY category ASC"

        cur = c.execute(query, params)
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]
```

- With a category: the total for that category in the range.
- Without: totals for every category in the range.

**Demo:** *"What was my total expense on education last week?"* → Claude answered from earlier chat data without calling the tool. Rephrased: *"Total expense on education in the last 10 days"* → `summarize(start=18th, end=27th, category="education")` → ₹1,800 (an online course subscription).

> LLMs sometimes answer from context instead of calling a tool. Being specific in the prompt helps; so do good tool descriptions.

---

## v4: consistent categories with a resource

[56:47](https://www.youtube.com/watch?v=tc2oOznpdE0&t=3407s)

**The problem:** Claude picks the category itself. *"Add a new expense, Udemy course purchased last Friday, ₹999"* → today it writes `education`, tomorrow `upskilling`, or `Education` with a capital E. Messy, inconsistent data makes analysis painful later. Also the `subcategory` column keeps getting ignored.

**The fix:** give Claude a fixed list of categories and subcategories as an MCP **resource** (static data, exactly what resources are for: chapter 03).

1. Create `categories.json` in the project:

```json
{
  "food": ["groceries", "fruits_vegetables", "dairy_bakery", "dining_out", "coffee_tea", "snacks", "delivery_fees", "other"],
  "transport": ["fuel", "public_transport", "cab_ride_hailing", "parking", "tolls", "vehicle_service", "other"],
  "education": ["books", "courses", "online_subscriptions", "exam_fees", "workshops", "other"],
  "entertainment": ["movies_events", "streaming_subscriptions", "games_apps", "outing", "other"],
  "...": ["20 categories in total: housing, utilities, health, shopping, travel, taxes, investments, misc, ..."]
}
```

2. Expose it with `@mcp.resource`:

```python
CATEGORIES_PATH = os.path.join(os.path.dirname(__file__), "categories.json")

@mcp.resource("expense://categories", mime_type="application/json")
def categories():
    # Read fresh each time so you can edit the file without restarting
    with open(CATEGORIES_PATH, "r", encoding="utf-8") as f:
        return f.read()
```

- `expense://categories` is the resource's **URI** (any scheme you choose).
- Restart Claude. In the chat box, **+ → Add from Expense Tracker → categories** attaches the JSON to the conversation as context.

**Demo:** with the resource attached, *"Add a new transaction: cab ride to airport last Wednesday, 700. Pick category from the pasted resource."* → `category: transport`, `subcategory: cab_ride_hailing`. Both fields now come from the fixed list, so entries stay consistent.

> In Claude Desktop, resources are **attached by the user** (the app decides how to use them); the model doesn't fetch them on its own the way it calls tools. That's why he attaches it and asks Claude to use it. If you want the model to pull categories by itself, you can also expose the same data as a tool (e.g. `get_categories`).

---

## Final server code

From the course repo (`main.py`):

```python
from fastmcp import FastMCP
import os
import sqlite3

DB_PATH = os.path.join(os.path.dirname(__file__), "expenses.db")
CATEGORIES_PATH = os.path.join(os.path.dirname(__file__), "categories.json")

mcp = FastMCP("ExpenseTracker")

def init_db():
    with sqlite3.connect(DB_PATH) as c:
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

init_db()

@mcp.tool()
def add_expense(date, amount, category, subcategory="", note=""):
    '''Add a new expense entry to the database.'''
    with sqlite3.connect(DB_PATH) as c:
        cur = c.execute(
            "INSERT INTO expenses(date, amount, category, subcategory, note) VALUES (?,?,?,?,?)",
            (date, amount, category, subcategory, note)
        )
        return {"status": "ok", "id": cur.lastrowid}

@mcp.tool()
def list_expenses(start_date, end_date):
    '''List expense entries within an inclusive date range.'''
    with sqlite3.connect(DB_PATH) as c:
        cur = c.execute(
            """
            SELECT id, date, amount, category, subcategory, note
            FROM expenses
            WHERE date BETWEEN ? AND ?
            ORDER BY id ASC
            """,
            (start_date, end_date)
        )
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]

@mcp.tool()
def summarize(start_date, end_date, category=None):
    '''Summarize expenses by category within an inclusive date range.'''
    with sqlite3.connect(DB_PATH) as c:
        query = (
            """
            SELECT category, SUM(amount) AS total_amount
            FROM expenses
            WHERE date BETWEEN ? AND ?
            """
        )
        params = [start_date, end_date]

        if category:
            query += " AND category = ?"
            params.append(category)

        query += " GROUP BY category ORDER BY category ASC"

        cur = c.execute(query, params)
        cols = [d[0] for d in cur.description]
        return [dict(zip(cols, r)) for r in cur.fetchall()]

@mcp.resource("expense://categories", mime_type="application/json")
def categories():
    # Read fresh each time so you can edit the file without restarting
    with open(CATEGORIES_PATH, "r", encoding="utf-8") as f:
        return f.read()

if __name__ == "__main__":
    mcp.run()
```

**Improvements worth making** (not in the video):

- Add type hints and fuller docstrings, e.g. `def add_expense(date: str, amount: float, category: str, subcategory: str = "", note: str = "") -> dict:` with `"""date: YYYY-MM-DD. category: one of the keys in expense://categories."""`. The LLM then gets a typed schema and clear rules, which reduces bad inputs.
- Validate `category`/`subcategory` against `categories.json` inside `add_expense` and return an error message if invalid, instead of trusting the model.

---

## Ideas to extend it

[45:00](https://www.youtube.com/watch?v=tc2oOznpdE0&t=2700s), [1:01:18](https://www.youtube.com/watch?v=tc2oOznpdE0&t=3678s)

- `edit_expense` (cab was ₹300, not ₹500)
- `delete_expense`
- Credits/income (salary, refunds)
- Budgets and alerts
- More analysis (Claude can do detailed analysis if entries are clean)

Adding a feature = add a function in `main.py` + restart Claude.

---

## FastAPI → MCP server in one line

[1:02:49](https://www.youtube.com/watch?v=tc2oOznpdE0&t=3769s)

FastMCP's creators studied **FastAPI** and reused its design principles, and FastMCP is **compatible with FastAPI**: you can turn a FastAPI app into an MCP server, and the reverse.

**Why a business cares.** Say CampusX's expense tracker is a product on **web, Android and iPhone**, all powered by one **FastAPI backend** (`POST /expenses`, `GET /expenses`, `GET /expenses/summary`, with a Pydantic model validating each expense). Then MCP arrives, and you want the product inside Claude Desktop and ChatGPT too. Rewriting all that logic as an MCP server duplicates the work. Instead:

```python
# server.py
from fastmcp import FastMCP
from main import app                 # your existing FastAPI app

mcp = FastMCP.from_fastapi(app=app, name="Expense Tracker Server")

if __name__ == "__main__":
    mcp.run()
```

FastMCP reads the app's endpoints and turns them into MCP tools automatically.

**Demo:** `uv run fastmcp dev server.py` → Inspector → List Tools shows **three tools**, one per endpoint. Ran the add-expense tool (₹2000, shopping, "purchased cricket bat") → success → row appears in `expenses.db`. Then `uv run fastmcp install claude-desktop server.py` to put it in Claude (remove the old entry first).

> Your existing API becomes available to AI clients with almost no new code. This will likely be a heavily used feature as companies add MCP to existing products.

---

## Command cheat sheet

| Task | Command |
|---|---|
| Install uv | `pip install uv` |
| New project | `uv init .` |
| Add FastMCP | `uv add fastmcp` |
| Check versions | `uv run fastmcp version` |
| Debug in MCP Inspector | `uv run fastmcp dev main.py` |
| Run server (stdio) | `uv run fastmcp run main.py` |
| Install into Claude Desktop | `uv run fastmcp install claude-desktop main.py` |
| Find uv's path (fix "can't connect") | `which uv` → put the absolute path in the config |
| Official SDK instead | `pip install "mcp[cli]"`, `from mcp.server.fastmcp import FastMCP` |

**Rules of thumb:**

- Restart Claude Desktop after every server change.
- Test in the Inspector first.
- Every tool needs a good description; add type hints.
- Static reference data → resource. Actions and dynamic queries → tools.

---

## Interview questions

**Q: MCP SDK vs FastMCP?**
The SDK is Anthropic's official, lower-level implementation. FastMCP is a developer-friendly framework; v1 was merged into the SDK (`mcp.server.fastmcp`), and FastMCP 2.x is an independent, larger library. Server code looks nearly identical.

**Q: What are the minimum pieces of a FastMCP server?**
A `FastMCP(name)` instance, functions decorated with `@mcp.tool` (or `@mcp.resource`/`@mcp.prompt`), and `mcp.run()`.

**Q: Where does the tool's schema come from in FastMCP?**
From the function signature (names, type hints, defaults) and the docstring as the description.

**Q: What is MCP Inspector?**
A browser-based debugging tool for MCP servers (like Postman for APIs): connect over a transport, list and call tools/resources/prompts, and see the raw JSON-RPC history.

**Q: Why did Claude Desktop fail to connect after `fastmcp install`?**
The generated config used `uv` as the command, which Claude Desktop couldn't find on its PATH. Using the absolute path to `uv` fixes it.

**Q: How did the course keep categories consistent?**
By exposing a `categories.json` as an MCP resource (`expense://categories`) and having Claude pick categories and subcategories from it.

**Q: How do you expose an existing FastAPI backend to AI clients?**
`FastMCP.from_fastapi(app=app)` converts its endpoints into MCP tools.
