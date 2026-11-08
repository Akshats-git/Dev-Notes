# 01 — Trailer: Building an AI Newsletter with MCP

**Video:** [Model Context Protocol | Mini Playlist | MCP Trilogy](https://www.youtube.com/watch?v=3_TN1i3MTEU) (37m)

Before any theory, Nitish shows what MCP makes possible. He builds a weekly AI newsletter for CampusX end to end (research, writing, HTML design) using only **Claude Desktop + a handful of MCP servers + three prompts**. No integration code was written. The whole point of the video: watch a real problem get solved first, then learn why and how it works in the next videos.

---

## Contents

1. [The playlist plan](#the-playlist-plan)
2. [The problem](#the-problem)
3. [Step 1: decide the structure](#step-1-decide-the-structure)
4. [Step 2: define the process](#step-2-define-the-process)
5. [Step 3: pick the AI and the tools](#step-3-pick-the-ai-and-the-tools)
6. [Step 4: research phase](#step-4-research-phase)
7. [Step 5: editing phase](#step-5-editing-phase)
8. [Step 6: design phase](#step-6-design-phase)
9. [How much code did it take?](#how-much-code-did-it-take)
10. [Takeaways](#takeaways)

---

## The playlist plan

[1:30](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=90s)

Originally planned as a trilogy, it grew to 8 videos:

| Part | Question | Videos |
|---|---|---|
| **The Why** | What problem does MCP solve? Why did it appear? | 02 |
| **The What** | Architecture: hosts, clients, servers, primitives, JSON-RPC, transports, lifecycle | 03, 04 |
| **The How** | Use ready-made servers, build local and remote servers, build a client | 05–08 |

---

## The problem

[3:49](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=229s)

AI moves so fast that a 6-month learning roadmap goes stale in 1–2 months. One thing that genuinely helps stay current is **reading AI newsletters**. So CampusX wanted its own weekly newsletter.

The blocker: a good newsletter needs a lot of **daily research**, careful **writing**, and **design**, and there was no time. MCP made it possible to hand almost all of that to an AI.

---

## Step 1: decide the structure

[7:48](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=468s)

Before touching any AI, picture the final product. After studying other newsletters he settled on nine sections:

1. **Introduction**: a short paragraph that sets expectations and builds curiosity
2. **Big story of the week**: the one piece of news that shook the AI world
3. **3–5 quick updates**: smaller news you should still know
4. **Top research papers**: summary + download link
5. **Top GitHub repos**: trending AI repositories of the week
6. **One quick tutorial**: something you can read in ~5 minutes and gain from
7. **Top AI products** of the week
8. **Top X posts**: what leading people are saying
9. **Closing notes**: summary + a teaser for the next issue

> Clear picture of the output first. Once that's fixed, everything after it gets easier.

---

## Step 2: define the process

[10:17](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=617s)

If the AI is going to do the work, you have to be explicit about the steps. Three stages:

```
Research  ──►  Editing  ──►  Designing
(gather notes    (merge notes into     (turn the draft into
 from sources)    one final draft)      a styled HTML email)
```

---

## Step 3: pick the AI and the tools

[11:33](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=693s)

**Main AI: Claude (Claude Desktop).** MCP came from Anthropic, so Claude had the strongest MCP support at the time. ChatGPT had fewer integrations and no MCP support then.

**Tools, all connected to Claude through MCP:**

| Tool | Used for |
|---|---|
| Google Drive | Reading the content-ideas file, past performance data, the sample newsletter template |
| Gmail | Reading reader feedback emails |
| Calendar | Checking when the next issue is due |
| Web search | Big story + quick updates |
| arXiv | Trending research papers |
| GitHub | Trending repos |
| Product Hunt | New AI products |
| Twitter / X | Top posts |
| Local file system | Saving research notes, drafts and HTML on the desktop |

> At this point it isn't obvious *why* you'd need MCP to connect a tool to an AI. That's exactly what the next videos answer. For now: one main AI (Claude) + many tools, connected through MCP.

---

## Step 4: research phase

[13:29](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=809s)

One long prompt ("You are an AI newsletter research agent...") with two jobs.

**Part A: decide what to research.** Claude reads three inputs:

- `content ideas` file on Google Drive (a running list of topic ideas)
- `performance data` file on Drive (per past issue: date, subject, open rate, click rate, average read time; dummy data for now, would come from a tool like Mailchimp later)
- **feedback emails** in Gmail

From these it produces a research focus (e.g. "AI in healthcare and finance performs well", "OpenAI vs Anthropic competition", "open-source LLMs").

**Part B: do the research** across five sources, writing **one Markdown file per source** into a desktop folder:

```
web search  → big story + quick updates
arXiv       → papers
GitHub      → trending repos
Product Hunt→ products
Twitter     → top posts
```

In the demo Claude asked permission before writing files, and appeared to run some research branches in parallel. Result: a folder with five research Markdown files.

---

## Step 5: editing phase

[17:01](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=1021s)

Second prompt ("You are an AI newsletter assembly agent"):

1. Read the five research files from the desktop folder.
2. Read the **sample newsletter template** from Google Drive (it has all nine sections laid out).
3. Fill each section with the latest research, smooth transitions, clear headers, engaging tone.
4. Save the final draft as Markdown back into the same folder.

---

## Step 6: design phase

[18:30](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=1110s)

Third prompt ("You are an email template builder agent"): convert the Markdown draft into a **production-ready HTML email** that works across major email clients.

The prompt spells out layout and structure, styling, content mapping, deliverability, testing, and gives an example HTML snippet. Two outputs:

- **HTML** version (the designed newsletter)
- **Plain-text** fallback, in case a client flags HTML as spam or can't render it

The HTML opened in a browser looked exactly like the final email: all nine sections, working links to sources, arXiv, GitHub, Product Hunt and X. Sending it is then just a Mailchimp-style step. (He tried Canva via MCP too, but HTML gave better results.)

---

## How much code did it take?

[32:49](https://www.youtube.com/watch?v=3_TN1i3MTEU&t=1969s)

Almost none. All tools were added through **one JSON config file** in Claude Desktop (`claude_desktop_config.json`, reached via *Settings → Developer → Edit Config*). Each server is a few lines:

```json
{
  "mcpServers": {
    "twitter": {
      "command": "npx",
      "args": ["-y", "@enescinar/twitter-mcp"],
      "env": { "API_KEY": "...", "API_SECRET_KEY": "..." }
    }
  }
}
```

- No function calls, no API client code for GitHub, arXiv, X and so on.
- If GitHub changes its API tomorrow, **you change nothing**. The GitHub MCP server's maintainers handle it.
- Adding a tool = adding a config entry.

---

## Takeaways

- A complex multi-step workflow (research → edit → design) was done in **three prompts**.
- The AI could reach Drive, Gmail, Calendar, the web, arXiv, GitHub, Product Hunt, X and the local disk **because each is exposed as an MCP server**.
- The client side (Claude Desktop) needed **configuration, not code**.
- Good results still came from human work up front: a clear output structure, a clear process, and detailed prompts.
