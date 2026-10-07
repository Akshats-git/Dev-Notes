# 14 — MCP: Model Context Protocol, From First Principles

**Video:** [FDE Full Course #14 — MCP Server Explained](https://www.youtube.com/watch?v=hSxdqkKZoqw) (1h 10m)
**Instructor notes + code:** [Lecture 14](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2014) (Spring, Python, JS. Each has a separate `taskServer` and `taskClient`.)

Tool calling (lecture 5) works great inside **one** app. MCP is about what happens when **many** AI apps need the **same** tools. It's a standard protocol for plugging capabilities into any AI application.

---

## Contents

1. [Recap: tool calling inside one app](#recap-tool-calling-inside-one-app)
2. [The integration problem (N × M)](#the-integration-problem-n--m)
3. [What MCP is](#what-mcp-is)
4. [MCP vs tool calling](#mcp-vs-tool-calling)
5. [Architecture: host, client, server](#architecture-host-client-server)
6. [What a server exposes: tools, resources, prompts](#what-a-server-exposes-tools-resources-prompts)
7. [The full tool lifecycle: `tools/list` and `tools/call`](#the-full-tool-lifecycle-toolslist-and-toolscall)
8. [Why not just REST? Security? Caching?](#why-not-just-rest-security-caching)
9. [Practical: a Task MCP server + an AI client](#practical-a-task-mcp-server--an-ai-client)

---

## Recap: tool calling inside one app

[0:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=0s)

"What's the weather in Delhi?" The LLM doesn't know, so we give it a tool:

```java
@Tool
public String getWeather(String city) { /* call weather API */ }
```

```
User → LLM → chooses getWeather() → application executes the tool → Weather API
     → result → LLM → final response
```

For one application this works perfectly. The problem shows up when the **same capabilities** need to be used by **many** AI applications.

---

## The integration problem (N × M)

[8:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=480s)

Say Coder Army has five capabilities:

```
getCourseDetails()  searchVideos()  createStudentTicket()
getStudentProgress()  scheduleMentorshipSession()
```

And several AI apps want them: the Coder Army chatbot, an internal teacher assistant, a VS Code assistant, a desktop assistant, a support agent, a CLI agent. Each app has its own server and LLM, and each would need its **own adapter** for each service:

```
              Course Service
      ┌──────────────┼──────────────┐
   AI App A       AI App B       AI App C
   Adapter A      Adapter B      Adapter C
```

Every app has to figure out: what capabilities exist, where they are, what parameters they need, how to call them, what they return, how auth works. With 5 apps × 10 external systems you're writing **50 adapters** (App A → GitHub, App A → Slack, App A → Jira, App B → GitHub…).

This isn't really an AI problem. It's a classic software problem. Imagine every device needing its own connector: keyboard → A, mouse → B, hard drive → C. We solved that with **standards**: USB, HTTP, JDBC, SQL, REST, SMTP.

> A standard doesn't add a new capability. It gives a **common way to access** a capability.

---

## What MCP is

[13:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=780s)

**MCP = Model Context Protocol.** Note the word *protocol*.

> MCP is a standard protocol through which AI applications connect to external tools and data.

```
Before:            OUR SERVICE                After:            MCP SERVER
                  custom integration                            MCP protocol
                 /       |       \                             /     |     \
             App A    App B    App C                       App A  App B  App C
```

With a common protocol, clients speak MCP and servers expose capabilities through MCP. The N × M mess becomes a clean boundary:

```
                     MCP
       ┌──────────────┼──────────────┐
  GitHub server  Slack server   Jira server
```

### An MCP server

A program that exposes capabilities via MCP. A Task MCP server might expose `createTask`, `listTasks`, `completeTask`. A client can ask it *"what tools do you provide?"* and get back descriptions like:

```json
{
  "name": "createTask",
  "description": "Creates a new task",
  "inputSchema": { "title": "string", "description": "string" }
}
```

Then invoke it: `createTask(title = "Record MCP Video", description = "Record next FDE lecture")`. The server runs the business logic.

### An MCP server does **not** need an LLM

No OpenAI, no Claude, no `ChatClient`. It can be plain business logic:

```java
public Task createTask(String title) {
    Task task = new Task(title);
    tasks.add(task);
    return task;
}
```

MCP just makes that capability reachable by other apps through a standard protocol.

### Where's the LLM then?

Inside the **AI application**, next to an **MCP client**:

```
            User
              ↓
       AI Application
         /        \
       LLM     MCP Client
                   │
               MCP Server
```

The app discovers the server's capabilities and hands them to the LLM. "Add 'Record Docker Lecture' to my tasks" → the LLM decides `createTask(title = "Record Docker Lecture")` → the app executes it **through MCP**.

---

## MCP vs tool calling

[29:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=1740s)

The most important distinction in the lecture.

**Tool calling** — the model sees a tool definition:

```json
{ "name": "getWeather", "description": "Get current weather", "parameters": { "city": "string" } }
```

and requests:

```json
{ "tool": "getWeather", "arguments": { "city": "Delhi" } }
```

It doesn't execute anything. It says *"please run getWeather('Delhi')"*.

> **Tool calling answers:** *Which capability should the model use, and with what arguments?*

**MCP** answers a different question:

> *How can an AI application **discover and communicate with** externally provided capabilities through a standard protocol?*

| | Question |
|---|---|
| **Tool calling** | Which tool should the model use? |
| **MCP** | Where do the tools come from, and how does the app access them? |

They're **complementary**. MCP does not replace tool calling.

```
MCP server exposes tools
  → MCP client discovers them
  → AI app gives tools to the LLM
  → LLM chooses a tool               ← tool calling
  → app invokes it through MCP       ← MCP
  → MCP server executes
  → result → LLM → final response
```

### Connection to structured output

Tool schemas should look familiar from lecture 13:

```
Structured output:  LLM → schema       → predictable response
Tool calling:       Tool → input schema → predictable arguments
```

`createTask` expects `{"title": "string"}`, so "Create a task called Record MCP Video" → `createTask(title = "Record MCP Video")`. You don't hand-write those JSON schemas. Frameworks generate them from your function name, description and parameter annotations.

---

## Architecture: host, client, server

[17:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=1020s)

```
USER
  ↓
+-----------------------+
|       MCP HOST        |
|        LLM            |
|         |             |
|     MCP CLIENT        |
+----------|------------+
           | MCP
+----------▼------------+
|      MCP SERVER       |
|   createTask()        |
|   listTasks()         |
|   completeTask()      |
+-----------------------+
```

### Host

The main AI application the user talks to: Claude Code, Cursor, a VS Code AI extension, a desktop assistant, or **your own Spring AI app**. It's the **orchestrator**:

```
receive user input → send input + available tools to the LLM
→ get a tool request back → use the MCP client → call the MCP server
→ get the result → send it back to the LLM → show the final response
```

### Client

The component **inside the host** that speaks MCP. Analogy: **JDBC**.

```
Application → JDBC driver → Database
AI application → MCP client → MCP server
```

It knows how to discover tools, invoke tools, read resources and fetch prompts.

### Server

The **capability provider**. Behind it can be anything: a database, a REST API, the filesystem, GitHub, Google Calendar, Slack, internal services. It exposes only what it wants clients to access.

**An MCP server is not necessarily a microservice.** A microservice exposes `POST /tasks`, `GET /tasks`. An MCP server exposes MCP primitives. They coexist happily:

```
AI app → MCP client ──MCP──► Task MCP Server ──REST──► Task Microservice → DB
```

The MCP server can be an **AI-friendly layer over an existing system**.

### One host, many servers

```
                AI HOST (LLM)
     ┌──────────────┼──────────────┐
 MCP client     MCP client     MCP client
     ↓              ↓              ↓
 GitHub MCP    Calendar MCP    Slack MCP
```

Now the model can `createGitHubIssue()`, `scheduleMeeting()`, `sendSlackMessage()` without any of that code living inside the AI app.

---

## What a server exposes: tools, resources, prompts

[34:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=2040s)

```
MCP SERVER
├── Tools      → I want the AI to DO something
├── Resources  → I want the AI to KNOW something
└── Prompts    → I want reusable instructions
```

### Tools

Executable capabilities: `createTask()`, `completeTask()`, `listTasks()`, `deleteTask()`. Each has a name, a description and an **input schema**:

```json
{
  "name": "createTask",
  "description": "Creates a new task",
  "inputSchema": {
    "type": "object",
    "properties": {
      "title":   { "type": "string" },
      "dueDate": { "type": "string" }
    },
    "required": ["title"]
  }
}
```

→ `title` is a required string, `dueDate` an optional string. Use tools when something needs to be executed or queried.

### Resources

Sometimes you don't want an operation, just **information**: `company-policy.txt`, `employee-handbook.pdf`, app config, repo info, a customer profile, a DB record.

> A **resource** is addressable context or data exposed by an MCP server.

```
company://policies/return-policy
file:///project/README.md
```

**Tool vs resource:** there's overlap. `getReturnPolicy()` *could* be a tool. But `company://policy/returns` represents the policy itself as addressable content.

| Tool | Resource |
|---|---|
| "What operation can I perform?" | "What information can I access?" |

**Resources ≠ RAG.** RAG solves *retrieval*: chunk → embed → vector DB → semantic search → relevant chunks. Resources expose addressable content the client can fetch. They solve different problems and can be **combined**. For example, a `searchKnowledge` MCP tool can be backed by a whole RAG pipeline.

### Prompts

Reusable prompt templates or workflows. A GitHub MCP server might offer `review_pull_request`, `summarize_issue`, `prepare_release_notes`:

```
review_pull_request(prNumber) →
  "You are reviewing pull request {prNumber}. Check for bugs, maintainability issues…"
```

Clients discover and fetch them via `prompts/list` and `prompts/get`. (Lecture 15 goes deeper on resources and prompts.)

### Who typically controls each

| Primitive | Typically controlled by | Example |
|---|---|---|
| Tools | **Model** | User says "create a task for tomorrow" → model decides to call `createTask` |
| Resources | **Application** | App sees you have `README.md` open and attaches it as context |
| Prompts | **User** | UI shows `/review-code`, the user picks it |

A useful mental model, not a strict rule.

---

## The full tool lifecycle: `tools/list` and `tools/call`

[39:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=2340s)

### 1. Discovery — `tools/list`

The client asks "what tools do you have?":

```json
{
  "tools": [
    { "name": "createTask",   "description": "Creates a new task",
      "inputSchema": { "type": "object",
                       "properties": { "title": { "type": "string" } },
                       "required": ["title"] } },
    { "name": "listTasks",    "description": "Returns all tasks",
      "inputSchema": { "type": "object" } },
    { "name": "completeTask", "description": "Marks a task as completed",
      "inputSchema": { "type": "object",
                       "properties": { "id": { "type": "integer" } },
                       "required": ["id"] } }
  ]
}
```

Not just names: **name + description + input schema**, which the host passes to the LLM.

### 2. Tool calling (the LLM's part)

```
USER MESSAGE:     Create a task called Record MCP Video.
AVAILABLE TOOLS:  createTask(title) — Creates a new task.
                  listTasks() — Returns existing tasks.
                  completeTask(id) — Marks a task as complete.
```

User wants something created → `createTask` matches → it needs `title` → `createTask(title = "Record MCP Video")`.

### 3. Execution through MCP — `tools/call`

The host knows this tool lives on the MCP server, so the client sends:

```json
{ "name": "createTask", "arguments": { "title": "Record MCP Video" } }
```

The server runs `createTask("Record MCP Video")` → `{"id": 42, "title": "Record MCP Video", "status": "PENDING"}` → back through client → host → LLM → *"I've added 'Record MCP Video' to your tasks."*

```
LLM chooses the tool             → Tool calling
Application invokes it remotely  → MCP
```

### Dynamic discovery

The server starts with `createTask` and `listTasks`. Later it adds `completeTask`. It just shows up in `tools/list`, and **every** connected AI app gets it, with no `completeTask(...)` coded into any of them. The app doesn't even know the tools exist until it asks. This is one of MCP's biggest wins.

---

## Why not just REST? Security? Caching?

**Why not just REST?** MCP doesn't make REST obsolete. An MCP server can call your existing `POST /tasks` internally. REST exposes HTTP endpoints. MCP exposes **AI-oriented** concepts: tools, descriptions, input schemas, resources, prompts and capabilities, all discoverable in a standard way. They work together. You *could* put all your tools in a plain microservice yourself. MCP is what lets **any** host (Claude Code, Cursor, your Spring app…) plug into it without a custom adapter.

**Security still belongs to the host.** Discovering a tool doesn't mean the model may run it freely. The host applies permissions, user approval, tool filtering, authentication and security policies. `readFile` might be auto-allowed, while `deleteRepository` needs explicit confirmation (lecture 5's human-in-the-loop). **MCP standardises communication. It doesn't remove your responsibility for security and authorisation.**

**Does `tools/list` run on every message?** Not necessarily. If the catalogue hasn't changed, the host/client can **cache** the definitions and reuse them. `tools/list`, `resources/list` and `prompts/list` are the discovery operations.

---

## Practical: a Task MCP server + an AI client

[45:00](https://www.youtube.com/watch?v=hSxdqkKZoqw&t=2700s)

Two **separate** applications, to make one thing obvious: **MCP server ≠ LLM application**.

```
1. TASK MCP SERVER (port 8081)          2. AI APPLICATION / HOST (port 8080)
   no LLM, no OpenAI                       contains the LLM + an MCP client
   exposes create/list/complete            connects to the task server, discovers tools,
                                           gives them to the LLM
```

### 1. The MCP server

Dependency: `spring-ai-starter-mcp-server-webmvc`.

```properties
spring.application.name=taskServer
server.port=8081

spring.ai.mcp.server.name=task-mcp-server
spring.ai.mcp.server.version=1.0.0
spring.ai.mcp.server.protocol=STREAMABLE
```

**Transport = Streamable HTTP.** The client and server need to agree how messages travel. Streamable HTTP is just normal HTTP request/response, with the option to stream on top. (MCP also supports a local `stdio` transport for servers launched as subprocesses.)

```java
@Component
public class TaskTools {

    private final List<String> tasks = new ArrayList<>();

    @McpTool(name = "create_task", description = "Create a new task")
    public String createTask(
            @McpToolParam(description = "Title of the task", required = true) String title) {
        tasks.add(title);
        return "Task created: " + title;
    }

    @McpTool(name = "list_tasks", description = "List all pending tasks")
    public List<String> listTasks() {
        return List.copyOf(tasks);          // return a copy, keep the original safe
    }

    @McpTool(name = "complete_task", description = "Complete a task using its exact title")
    public String completeTask(
            @McpToolParam(description = "Exact title of the task", required = true) String title) {
        return tasks.remove(title) ? "Task completed: " + title
                                   : "Task not found: " + title;
    }
}
```

Plain Java functions. `@McpTool` (name + description) and `@McpToolParam` (parameter description) are all Spring AI needs to generate the JSON schema for `tools/list`.

Python equivalent (official `mcp` SDK):

```python
mcp = MCPServer("task-mcp-server")
tasks: list[str] = []

@mcp.tool(name="create_task", description="Create a new task")
def create_task(title: Annotated[str, Field(description="Title of the task")]) -> str:
    tasks.append(title)
    return f"Task created: {title}"

# list_tasks, complete_task similarly…

mcp.run(transport="streamable-http", host="127.0.0.1", port=8081)
```

**Test the server without any LLM.** The video hits it from the terminal with the MCP Inspector CLI, calling `tools/list` against `http://localhost:8081/mcp` over HTTP transport. The tool list with schemas comes back, and you can `tools/call` `create_task` with arguments the same way:

```bash
npx @modelcontextprotocol/inspector --cli http://localhost:8081/mcp \
    --transport http --method tools/list
```

### 2. The AI client (host)

Dependencies: `spring-ai-starter-mcp-client` + `spring-ai-starter-model-openai`.

```properties
spring.application.name=taskClient
server.port=8080
spring.ai.openai.api-key=${OPENAI_API_KEY}

# where the MCP server lives
spring.ai.mcp.client.streamable-http.connections.task-server.url=http://localhost:8081
```

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder, ToolCallbackProvider mcpTools) {
        this.chatClient = builder.defaultTools(mcpTools).build();   // ← all MCP tools
    }

    public String chat(String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

```java
@RestController
public class ChatController {
    @GetMapping("/ask")
    public String ask(@RequestParam("message") String message) {
        return chatService.chat(message);
    }
}
```

**`ToolCallbackProvider`** is the key piece. Spring AI's MCP client auto-configuration connects to the server, runs discovery, and wraps every remote tool as a callback. Compare with lecture 5, where we passed `new WeatherTool()` / `calculatorTool` objects ourselves. Here we pass `mcpTools` and **the client has no idea which tools exist** until it asks the server.

### Running it

```
GET localhost:8080/ask?message=Add a task: Record next Spring Boot project video
→ "Task 'Record next Spring Boot project video' created. Would you like to add details?"

GET /ask?message=List all my tasks
→ lists it

GET /ask?message=Add a task to buy groceries
→ created

GET /ask?message=Mark all my tasks as completed
→ completes both; "There are no pending tasks."
```

Behind each call: host → LLM (with the discovered tool definitions) → LLM picks `create_task` / `list_tasks` / `complete_task` → MCP client `tools/call` → task server runs the Java method → result → LLM → friendly reply.

Add history (lecture 4) and more tools (due dates, priorities…) to grow it. Any other host, such as Claude Code or Cursor, could connect to the same task server and use these tools too.

---

## Final mental model

```
Tool calling = the LLM decides which capability to use.
MCP          = the AI application gets access to external capabilities
               through a standard protocol.
```

```
MCP server exposes capabilities → MCP client discovers them
→ host gives relevant tools to the LLM → LLM selects a tool
→ MCP client invokes it → MCP server executes it
→ result returns to the LLM → user gets the final response
```

## Key takeaways

1. **MCP = Model Context Protocol**, a standard way for AI apps to connect to external capabilities and context.
2. It solves the **N × M integration problem**, the same way USB/JDBC/HTTP standardised other connections.
3. An **MCP server doesn't need an LLM**. It's a capability provider, often a thin AI-friendly layer over existing systems.
4. **Host** = the AI app/orchestrator. **Client** = the MCP-speaking component inside it (like a JDBC driver). **Server** = exposes capabilities.
5. Servers expose **tools** (do), **resources** (know) and **prompts** (reusable instructions).
6. `tools/list` = discovery (name, description, input schema). `tools/call` = invocation.
7. **Tool calling and MCP are complementary.** Tool calling picks the tool. MCP standardises how it's reached.
8. Dynamic discovery: add a tool to the server and every connected host gets it, with no client code changes.
9. MCP coexists with REST, databases, RAG and existing services.
10. Security, permissions and approval remain the **host's** job. Discovery can be cached.

## Interview questions

**Q: What problem does MCP solve?**
Many AI apps each building custom adapters to many tools and data sources (N × M). MCP gives one protocol: servers expose capabilities, any MCP-capable host can discover and use them.

**Q: Is MCP a replacement for function/tool calling?**
No. Tool calling is the model choosing a function and its arguments. MCP is how the host discovers and invokes functions provided by external servers. They work together.

**Q: Does an MCP server need an LLM?**
No. It's ordinary code exposing tools/resources/prompts. The LLM lives in the host.

**Q: Host vs client vs server?**
Host: the user-facing AI app that orchestrates the LLM and tools. Client: the MCP protocol component inside the host, one per server connection. Server: the capability provider.

**Q: Name the MCP primitives and when to use each.**
Tools for actions/queries (model-driven). Resources for addressable data/context (application-driven). Prompts for reusable instruction templates (user-driven).

**Q: Why not just expose a REST API?**
REST endpoints aren't self-describing for LLMs in a standard way. MCP adds discoverable tool descriptions and input schemas plus resources and prompts, so any host can plug in. An MCP server can wrap the REST API.

**Q: Who enforces security in MCP?**
The host: permissions, user approval, tool filtering, auth. MCP only standardises communication.
