# 02 — The Why: What Problem MCP Solves

**Video:** [Model Context Protocol - The Why](https://www.youtube.com/watch?v=Zmy439spZB4) (52m)
**Slides:** [Google Slides deck](https://docs.google.com/presentation/d/1OQzPhi73EUUbYXrIkOwGLnQYTSK8xU8T_zrtZcfgN0Q/edit) (the same deck is used for the whole playlist)

Told as a story, from ChatGPT's launch to today: LLMs spread into every app → we ended up with many disconnected AIs → the real blocker to one "unified AI partner" is **context** → function calling fixed context but created an **N × M integration problem** → MCP fixes that with one standard protocol.

---

## Contents

1. [The arrival of LLMs](#the-arrival-of-llms)
2. [Three waves of adoption](#three-waves-of-adoption)
3. [The problem of fragmentation](#the-problem-of-fragmentation)
4. [What context is](#what-context-is)
5. [Example: a software engineer's day](#example-a-software-engineers-day)
6. [Copy-paste hell](#copy-paste-hell)
7. [Function calling and the rise of tools](#function-calling-and-the-rise-of-tools)
8. [The new problem: N × M integrations](#the-new-problem-n--m-integrations)
9. [Enter MCP](#enter-mcp)
10. [MCP vs tool calling vs plain APIs](#mcp-vs-tool-calling-vs-plain-apis)
11. [Benefits](#benefits)
12. [Why the ecosystem is exploding](#why-the-ecosystem-is-exploding)
13. [Summary](#summary)
14. [Interview questions](#interview-questions)

---

## The arrival of LLMs

[2:16](https://www.youtube.com/watch?v=Zmy439spZB4&t=136s)

- ChatGPT launched **30 Nov 2022**. 1 million users in 5 days, 100 million in 2 months. No software (Google, Facebook, Twitter) had grown like that.
- It was a new **class** of software. For 500–600 years our relationship with machines (mechanical → electrical → computers) was **transactional**: press a switch, type into a form, press a button, get a result.
- With ChatGPT you talk to a machine **in natural language**, the way you'd talk to a person. It can express itself back. It can become a work partner.

---

## Three waves of adoption

[4:36](https://www.youtube.com/watch?v=Zmy439spZB4&t=276s)

| Wave | What people did | Impact |
|---|---|---|
| **1. Pure wonder** | Silly prompts: explain quantum physics from a cat's view, a Shakespearean song about pizza | Social media exploded with screenshots. Curiosity satisfied, nothing meaningful built |
| **2. Professional adoption** | Lawyers summarized 50-page contracts, developers debugged code, teachers planned lessons | Individual productivity boom. A 6-hour task took 3 |
| **3. The API revolution** | OpenAI also shipped its models as an **API**. Microsoft put Copilot in Word/Excel/PowerPoint/Outlook, Google put AI in Gmail/Docs/Drive, AI-first tools like Cursor and Perplexity appeared | AI became accessible inside the software you already use |

---

## The problem of fragmentation

[9:48](https://www.youtube.com/watch?v=Zmy439spZB4&t=588s)

Every app became "AI-enabled", but each AI lived in its own world:

- Notion's AI doesn't know what's happening in Slack's AI.
- The coding assistant in VS Code knows nothing about the discussion in MS Teams.

Users ended up juggling five AI assistants, merging information by hand.

**Vision vs reality.** What people wanted after first using ChatGPT was **one unified AI partner** that understands their whole work end to end and can solve any problem in it. What they got was five separate AI tools. The biggest obstacle to the unified version is **context**.

---

## What context is

[12:21](https://www.youtube.com/watch?v=Zmy439spZB4&t=741s)

> **Context is everything an AI can "see" when it generates a response.**
> More formally: the information (conversation history, external documents, etc.) the LLM uses to generate a response.

Simple case: you ask ChatGPT about quantum physics, then about books, then "how hard is this to learn?". It knows "this" means quantum physics because the **conversation history** is its context.

Context is important enough to be in the name: **Model *Context* Protocol**.

---

## Example: a software engineer's day

[14:37](https://www.youtube.com/watch?v=Zmy439spZB4&t=877s)

You work at an ed-tech startup (think Udemy). Task: **add two-factor authentication (2FA)**. The context for that one task lives in many places:

```
Jira         → the ticket and its requirements
GitHub       → the latest codebase, existing auth code
MySQL        → the current database schema
Google Drive → the company's security guidelines doc
Slack        → what the team has discussed about it
```

In a real professional scenario, **context is scattered across systems**.

---

## Copy-paste hell

[18:21](https://www.youtube.com/watch?v=Zmy439spZB4&t=1101s)

To get ChatGPT's help you copy the Jira ticket, paste. Copy 10–12 code files, paste. Copy the schema, paste. Copy the security doc, paste. Copy the Slack thread, paste. Twenty minutes later you ask your first question.

- You paste thousands of lines to ask **one** question.
- **Developers have become human APIs**, assembling context for the AI.
- **Context assembly time > development time.**
- You also have to track what the AI remembers, what it forgot, what to summarize.
- It doesn't scale: you cannot paste a 500,000-line codebase.

> Ideal: the AI fetches the context itself from all those places.

---

## Function calling and the rise of tools

[21:20](https://www.youtube.com/watch?v=Zmy439spZB4&t=1280s)

In **mid-2023 OpenAI introduced function calling**: a way for LLMs to call external functions.

- You give the LLM a set of functions, each with a **description** (e.g. `load_file`: "use this when the user wants to load a file's content").
- User says "read the content of `abc.txt`". The LLM sees this is a task, scans the function descriptions, and replies with **which function to call and with what arguments**: `load_file("abc.txt")`.
- Your app executes it and the task is done.

Small idea, path-breaking result: an LLM can now **do tasks**, not just chat.

```
                 ┌──► weather API tool
        LLM  ────┼──► database query tool
                 ├──► GitHub repo tool
                 └──► web search tool
```

### The flood of tools

[23:20](https://www.youtube.com/watch?v=Zmy439spZB4&t=1400s)

- **Enterprise integrations:** Salesforce, Slack bots, Google Drive connectors, database query tools, Git.
- **Internal tools:** HR (employee data), finance (accounting), marketing (campaigns), IT (infrastructure).
- **AI-first products:** Cursor (file-system access), Perplexity (browsing and real-time retrieval), ChatGPT Plus (browsing, uploads, code execution), Claude Computer Use (controls your computer).

### The 2FA example, with tools

[26:28](https://www.youtube.com/watch?v=Zmy439spZB4&t=1588s)

Now ChatGPT is connected to everything through tools:

- "Check whether I have a new Jira ticket" → it fetches it.
- "Fetch the latest codebase from GitHub" → done.
- "Get the schema" (MySQL tool), "Get the security guidelines" (Drive tool), "What's the team saying on Slack?" (Slack tool).
- Then: "Now you can see everything. How do I build 2FA?"

Context is no longer scattered. It's **connected**. For a while it felt like the context problem was solved.

---

## The new problem: N × M integrations

[30:02](https://www.youtube.com/watch?v=Zmy439spZB4&t=1802s)

Every tool needs a **function written inside the chatbot's code** (he shows his LangGraph chatbot with a calculator tool and a stock-price tool, each a separate function).

Say your company uses **3 AI chatbots** (a general chatbot, a coding agent, an analytics agent) and **20 SaaS tools** (Jira, Slack, GitHub, MySQL, Drive, ...):

> **N chatbots × M tools = N × M integrations.** 3 × 20 = **60 functions** to write.

### Problems with that

| Problem | Why it hurts |
|---|---|
| **Development nightmare** | Each integration has its own auth method, data format, API pattern, error handling. You need a separate team just to write integrations |
| **Maintenance** | 60 functions to maintain. Google Drive changes its API → every Drive integration in every chatbot breaks |
| **Security fragmentation** | 60 integrations each with their own auth and API keys spread across files. No single place to manage or audit |
| **Cost and time** | Months to wire up 20 tools, plus salaries for the integration team. You set out to make developers' lives easier and ended up hiring more developers |

**The overview:** every AI tool was building its own way to call every API. If a company uses Perplexity, Cursor and ChatGPT and wants all three talking to GitHub, that's three separate GitHub integrations that look nothing alike.

> What we want: **GitHub builds one integration that any AI tool can use.**

---

## Enter MCP

[36:10](https://www.youtube.com/watch?v=Zmy439spZB4&t=2170s)

MCP has two entities, and all communication happens between them:

- **Client**: your AI chatbot (ChatGPT, Cursor, Perplexity, or your own app).
- **Server**: the service you want the chatbot to reach (GitHub, Google Drive, Slack).

```
                ┌──► GitHub MCP server
  AI chatbot ───┼──► Google Drive MCP server
  (MCP client)  └──► Slack MCP server

  the "language" they speak = Model Context Protocol
```

The **language** the client and servers speak is MCP. You could implement the protocol from scratch by reading the spec, but Anthropic provides ready-made libraries:

- **MCP Client SDK**: install it where your chatbot runs, and the chatbot becomes an MCP-compatible client.
- **MCP Server SDK**: use it to build your tools as MCP servers.

---

## MCP vs tool calling vs plain APIs

[39:08](https://www.youtube.com/watch?v=Zmy439spZB4&t=2348s)

**Function/tool calling:** there's still a client-server shape. A weather service has an API on a server (say, FastAPI). **You** write a function inside your chatbot's code that hits that API and parses the result. Server code (written by the company) + client code (written by you) together complete the task.

**MCP:** the weather server is built with the MCP library (internally it can call the same API). On the **client side you write no code**. You configure the connection once, both sides speak the same protocol, and the AI can now discover and use whatever tools that server offers.

### The server does the heavy lifting

[42:00](https://www.youtube.com/watch?v=Zmy439spZB4&t=2520s)

For a GitHub MCP server, the **server** is responsible for:

- all business logic
- authentication
- API rate limiting
- data-format translation (what the LLM can understand)
- error handling and returning proper error codes to the client

The client just connects and speaks MCP.

---

## Benefits

[43:26](https://www.youtube.com/watch?v=Zmy439spZB4&t=2606s)

1. **N + M instead of N × M.** 3 chatbots × 10 services used to be 30 integrations. Now 10 servers exist (built by the service providers themselves) and each chatbot just connects. The integration code moved to the server side.
2. **No maintenance overhead on the client.** API changes are the server's problem. Nothing changes on your side.
3. **Reduced cost and time.** No integration team. Connect to 10 existing servers on day one.
4. **Better security.** All connections live in **one JSON config file**: e.g. the GitHub entry with a personal access token, the Notion entry with an API key. One file to manage and audit, instead of keys spread across 10 integrations.

> Everything comes back to one fact: **in MCP the server does the heavy lifting; the client just maintains a config to connect.**

---

## Why the ecosystem is exploding

[47:28](https://www.youtube.com/watch?v=Zmy439spZB4&t=2848s)

A **network effect**:

- Big AI clients (Claude Desktop, Cursor, Windsurf, Perplexity) announced MCP support.
- Services (GitHub, Slack, Google Drive) saw that future users will reach them **through AI tools**, not by opening `drive.google.com`. So they built official MCP servers.
- More servers make MCP more valuable for new chatbots. A new chatbot only has to become an MCP client to reach thousands of servers on day one with no extra code.
- More clients make it more valuable for services to build servers.

```
More AI clients support MCP  ──►  more value in building servers
           ▲                                    │
           └──────── more servers exist ◄───────┘

More adoption → more standardization → more ecosystem value
```

Not supporting MCP means being cut off from the ecosystem and writing custom code for everything. Nitish's bet: MCP becomes the industry standard within 3–5 years.

---

## Summary

1. ChatGPT made machines conversational. Adoption came in three waves: wonder → professional use → APIs everywhere.
2. AI everywhere produced **fragmentation**: many AIs, none sharing context.
3. **Context** is what the model can see. At work it's scattered across Jira, GitHub, DBs, Drive, Slack.
4. Manually assembling it is **copy-paste hell**. Developers became human APIs.
5. **Function calling** (mid-2023) let LLMs fetch context through tools, but every chatbot had to write every integration: **N × M**.
6. **MCP** standardizes the connection: **clients** (AI apps) and **servers** (services) speak one protocol. Providers build the server once; any client can use it.
7. Benefits: N + M integrations, no client-side maintenance, lower cost, centralized config and secrets.
8. Network effects are driving fast adoption.

## Interview questions

**Q: What is "context" for an LLM?**
Everything the model can see when generating a response: conversation history, documents, tool results, etc.

**Q: Why wasn't function calling enough?**
It solved getting context, but each AI app had to write and maintain its own integration for every service. With N apps and M services you get N × M integrations, each with its own auth, formats, errors and secrets.

**Q: How does MCP change the integration count?**
Each service ships one MCP server, and each AI app implements the MCP client once. N × M becomes N + M.

**Q: In MCP, who does the heavy lifting?**
The server: business logic, auth, rate limiting, translating data for the LLM, error handling. The client only connects and speaks the protocol.

**Q: What are the two SDKs Anthropic provides?**
An MCP client SDK (to make an AI app MCP-compatible) and an MCP server SDK (to expose a service as an MCP server).

**Q: Why is MCP adoption accelerating?**
Network effects: more clients make servers more valuable and vice versa. Not supporting MCP cuts you off from the ecosystem.
