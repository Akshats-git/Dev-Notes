# 15 — MCP Resources and Prompts

**Video:** [FDE Full Course #15 — MCP Resources & Prompts Explained](https://www.youtube.com/watch?v=MUttM7fIqO4) (53m)
**Instructor notes + code:** [Lecture 15](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2015) (Spring, Python, JS)

Lecture 14 built a Task MCP server that exposed **tools**. An MCP server can expose two more primitives, **resources** and **prompts**, and this lecture adds both to the same task server.

---

## Contents

1. [Three primitives, three controllers](#three-primitives-three-controllers)
2. [Resources: why not just use a tool?](#resources-why-not-just-use-a-tool)
3. [Discovering and reading resources](#discovering-and-reading-resources)
4. [Anatomy of a resource](#anatomy-of-a-resource)
5. [Practical: adding a resource](#practical-adding-a-resource)
6. [Static, dynamic and templated resources](#static-dynamic-and-templated-resources)
7. [How does the app choose a resource?](#how-does-the-app-choose-a-resource)
8. [Prompts: reusable workflows](#prompts-reusable-workflows)
9. [Practical: adding a prompt](#practical-adding-a-prompt)
10. [All three working together](#all-three-working-together)

---

## Three primitives, three controllers

[0:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=0s)

```
            MCP Server
       /        |        \
   Tools    Resources   Prompts
```

| Primitive | Simple idea | **Primarily controlled by** |
|---|---|---|
| **Tools** | Capabilities | **Model** — the LLM decides to call them |
| **Resources** | Context | **Application** — the host decides what to read and supply |
| **Prompts** | Reusable AI workflows | **User** — the user explicitly picks one |

The "who controls it" column is the more useful distinction.

---

## Resources: why not just use a tool?

[3:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=180s)

The server holds some reference info:

```
Coder Army Coding Guidelines
1. Use Java 21.
2. Prefer constructor injection.
3. Controllers should not contain business logic.
4. Services should contain business logic.
5. Use meaningful variable names.
```

You *could* expose it as a tool. Tools can retrieve information, not only perform actions.

```java
@McpTool(...)
public String getCodingGuidelines() { return "..."; }
```

That would work. So the right question isn't *"does it return data?"* It's **"what is the intent of this capability?"**

- `createTask()` → something the model **calls or executes**.
- `coding-guidelines.txt` → **information useful as context**.

| Tool | Resource |
|---|---|
| A capability the **model** can invoke | Context/data the **application** can make available |

### What counts as a resource

> **An MCP resource is data or content exposed by an MCP server that a client can discover and read.**

Company policies, READMEs, project docs, Git history, DB records, app config, API docs, logs, knowledge-base articles, architecture docs.

**A resource does not automatically go into the LLM context.**

```
MCP server → exposes resources
MCP client / host → discovers them
Application → decides which one to read
Relevant resource → can be supplied to the LLM as context
```

That's why resources are **application-controlled**.

---

## Discovering and reading resources

[7:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=420s)

Two separate operations.

**`resources/list`** asks "what resources exist?"

```
CLIENT ── resources/list ──► SERVER
       ◄── coding-guidelines, URI: company://coding-guidelines ──
```

Now the client knows it exists (name, URI, description, MIME type), but has no content yet.

**`resources/read`** says "give me this one":

```
CLIENT ── resources/read  company://coding-guidelines ──► SERVER
       ◄── "Use Java 21. Prefer constructor injection. …" ──
```

Docs are small here so the whole thing comes back at once. For big ones you could chunk or RAG them (more below).

Compare the flows:

```
TOOLS:      server → tool descriptions → LLM → model decides which tool to call
RESOURCES:  server → resource descriptions → client/host → relevant one is read
            → content can be supplied to the LLM
```

---

## Anatomy of a resource

[10:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=600s)

```
Resource
├── Name
├── URI
├── Description
├── MIME type
└── Content
```

- **Name** — identifies it: `coding-guidelines`, `project-info`, `company-policy`, `database-schema`.
- **URI** — its unique address: `company://coding-guidelines`, `company://leave-policy`, `project://architecture`, `docs://spring-boot-guide`, `config://application`.
- **Description** — what it represents, e.g. *"Coding standards used by our Java development team."* Very useful when a server exposes many resources.
- **MIME type** — what kind of content: `text/plain`, `application/json`, `text/html`, `image/png`.
- **Content** — the actual information.

### URI ≠ URL

`company://coding-guidelines` doesn't mean the client does `HTTP GET company://coding-guidelines` like a browser. A URI is an identifier/path that may or may not be a URL. The client asks the **MCP server** to `resources/read` that URI, and the **server** resolves it and returns the content.

> A resource is an **addressable piece of context** plus metadata describing it.

---

## Practical: adding a resource

[26:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=1560s)

### Server — `TaskResources.java`

```java
@Component
public class TaskResources {

    @McpResource(
            uri = "task://guidelines",
            name = "task-guidelines",
            description = "Guidelines for managing and prioritizing tasks",
            mimeType = "text/plain")
    public String taskGuidelines() {
        return """
                Task Management Guidelines:
                1. Deep work tasks should preferably be done before 12 PM.
                2. Meetings should preferably be scheduled after 2 PM.
                3. Urgent tasks should be completed on the same day.
                4. Keep a maximum of three high-priority tasks per day.
                5. Complete high-priority tasks before low-priority tasks.
                """;
    }
}
```

The content could just as well be read from a file. It's hardcoded to keep things simple. On startup Spring sees `@McpResource` and registers it:

```
taskServer
├── Tools:     create_task, list_tasks, complete_task
└── Resources: task://guidelines
```

### Client — read it explicitly

Unlike tools, a resource is **not** handed to the model via `defaultTools(...)`. The **application** reads it:

```java
@Service
public class ChatService {

    private final ChatClient chatClient;
    private final McpSyncClient mcpClient;      // direct access to the MCP connection

    public ChatService(ChatClient.Builder builder,
                       ToolCallbackProvider mcpTools,
                       List<McpSyncClient> mcpClients) {
        this.chatClient = builder.defaultTools(mcpTools).build();   // tools → model
        this.mcpClient  = mcpClients.get(0);                        // our task server
    }

    private String readTaskGuidelines() {
        McpSchema.ReadResourceResult result = mcpClient.readResource(
                McpSchema.ReadResourceRequest.builder("task://guidelines").build());
        var content = (McpSchema.TextResourceContents) result.contents().getFirst();
        return content.text();
    }

    public String chat(String message) {
        String guidelines = readTaskGuidelines();                   // app decides to read it
        return chatClient.prompt()
                .system("""
                        You are a task management assistant.
                        Use the following task management guidelines
                        whenever they are relevant to the user's question.

                        TASK GUIDELINES:
                        %s
                        """.formatted(guidelines))
                .user(message)
                .call()
                .content();
    }
}
```

```
ChatService → readResource("task://guidelines") → MCP server → guidelines
           → app puts them in the system prompt → LLM
```

"When should I schedule meetings?" → *"Meetings should preferably be scheduled after 2 PM."* That answer comes from the MCP resource.

---

## Static, dynamic and templated resources

[15:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=900s)

`task://guidelines` is **static** (fixed text). A resource can also be **dynamic**: `user://123/profile`, `database://customer/938`, `project://coder-army/status`. The method can fetch from a database, another API, the filesystem, app state or another service:

```java
return database.findById(...);
```

### Resource templates

One definition per user (`user://1`, `user://2`, …) doesn't scale. Use a **URI template**:

```java
@McpResource(uri = "user://{username}", name = "user-profile")
public String userProfile(String username) {
    return "Profile of " + username;
}
```

Now `user://aditya` and `user://ravi` resolve dynamically.

---

## How does the app choose a resource?

[17:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=1020s)

"Application-controlled" doesn't mean the server magically understands natural language. The server has no LLM. **MCP doesn't prescribe how a host picks a resource.** Common strategies:

**1. User selection.** The UI shows the list from `resources/list`:

```
Attach context
[ ] Coding Guidelines
[✓] Leave Policy
[ ] Architecture
```

User ticks Leave Policy → app calls `resources/read company://leave-policy` → adds it to context.

**2. Application state.** In an IDE, `OrderController.java` is open and the user asks "Explain this function". The IDE already knows which file → `resources/read file://project/src/OrderController.java` → LLM. **No AI decision needed.**

**3. Retrieval / search.** With hundreds or thousands of resources, embed the query and compare it against resource **metadata/descriptions** ("leave-policy: Company policies on leave, holidays, maternity leave…"). "What's the maternity leave policy?" → match → `company://leave-policy` → read → LLM. Basically RAG over resources, with no LLM needed for the matching.

**4. LLM-based routing.** Show an LLM the resource list with descriptions and ask which one fits. It picks `leave-policy`, then the app reads it. Note: **the protocol didn't make the resource model-controlled.** The *application* chose to use an LLM inside its own routing logic.

### Resource or tool? A rule of thumb

| Should the **model** decide to call it? | Should the **application** provide it as context? |
|---|---|
| → **Tool**: `searchDocumentation(query)`, `getWeather(city)`, `lookupOrder(id)` | → **Resource**: `company://coding-guidelines`, `file://README.md`, `project://architecture` |

---

## Prompts: reusable workflows

[34:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=2040s)

Resources solve **reusable context**. Prompts solve **reusable ways of interacting with the AI**.

The task server wants to offer workflows: *Plan My Day*, *Prioritize My Tasks*, *Review My Workload*, *Create Weekly Plan*. "Plan My Day" needs instructions like:

```
Look at my pending tasks.
I have 4 hours available today.
Prioritize them by importance and give me a concise ordered plan.
```

You could hardcode that in every AI app, but then every client of the server has to recreate it. Instead, the **server** ships it.

> **An MCP prompt is a reusable message template exposed by an MCP server.**

```
MCP SERVER
  TOOLS:     create_task, list_tasks
  RESOURCES: task://guidelines
  PROMPTS:   plan_day
```

Prompts are **user-controlled**. Clients surface them as buttons, menus, **slash commands** or forms:

```
Available prompts
[ Plan My Day ]  [ Prioritize Tasks ]  [ Review Workload ]
```

The user picks "Plan My Day". The model doesn't independently decide to invoke `plan_day`. (ChatGPT's suggested prompts and Claude Code's slash commands are the same kind of idea.)

### A prompt is not the answer

```
MCP prompt → generates messages → MCP client → LLM → final response
```

**MCP prompt ≠ LLM response.** The server defines a reusable interaction pattern. The LLM still writes the answer.

### Why keep prompts on the server?

A server can advertise both **"what this system can do"** (tools) and **"good ways to work with it"** (prompts):

| Server | Prompts it might offer |
|---|---|
| Task server | `plan_day`, `prioritize_tasks`, `review_workload` |
| GitHub server | `review_pull_request`, `summarize_issue`, `prepare_release_notes` |
| Database server | `analyze_sales`, `explain_schema`, `investigate_anomaly` |

### Prompt arguments

Don't make `plan_day_2_hours`, `plan_day_4_hours`, `plan_day_8_hours`. Make **one** prompt with an argument: `plan_day(availableHours)`.

```
Plan My Day
Available hours: [ 4 ]   [ Run ]
```

Client requests `plan_day` with `availableHours = 4` → server fills the template:

```
Plan my pending tasks for today.
First check my current pending tasks.
I have 4 hours available.
Give me a concise ordered plan.
```

### `prompts/list` and `prompts/get`

```
CLIENT ── prompts/list ──► SERVER
       ◄── plan_day (availableHours, required) ──      → UI shows "Plan My Day"

CLIENT ── prompts/get  name=plan_day, availableHours=4 ──► SERVER
       ◄── prompt messages ──
```

---

## Practical: adding a prompt

[42:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=2520s)

```
taskServer
├── TaskTools.java
├── TaskResources.java
└── TaskPrompts.java
```

### Server — `TaskPrompts.java`

```java
@Component
public class TaskPrompts {

    @McpPrompt(name = "plan_day",
               description = "Plan pending tasks for the available time")
    public McpSchema.GetPromptResult planDay(
            @McpArg(name = "availableHours",
                    description = "Number of hours available today",
                    required = true)
            String availableHours) {

        String message = """
                Help me plan my pending tasks for today.
                First check my current pending tasks using the available task tools.
                I have %s hours available today.
                Give me a concise and ordered plan.
                """.formatted(availableHours);

        var promptMessage = McpSchema.PromptMessage.builder(
                McpSchema.Role.USER,
                McpSchema.TextContent.builder(message).build()).build();

        return McpSchema.GetPromptResult.builder(List.of(promptMessage))
                .description("Plan pending tasks for the day")
                .build();
    }
}
```

`@McpTool` → tools, `@McpResource` → resources, `@McpPrompt` (+ `@McpArg`) → prompts. It returns **messages** (here one `USER` message), not an answer.

### Prompt and tool cooperate

*"First check my current pending tasks using the available task tools."* The prompt **does not run** `list_tasks`. It only instructs the LLM. The LLM sees `list_tasks` among its tools, decides it needs the current tasks, and calls it.

```
User selects prompt → prompt guides LLM → LLM decides a tool is needed → tool executes
```

### Client — fetch the prompt, then call the LLM

```java
public String planDay(String hours) {
    McpSchema.GetPromptResult result = mcpClient.getPrompt(
            McpSchema.GetPromptRequest.builder("plan_day")
                    .arguments(Map.of("availableHours", hours))
                    .build());

    var content = (McpSchema.TextContent) result.messages().getFirst().content();
    String generatedPrompt = content.text();

    return chatClient.prompt()          // ← now talk to the LLM
            .user(generatedPrompt)
            .call()
            .content();
}
```

Don't confuse the two:

| Call | Talks to |
|---|---|
| `mcpClient.getPrompt(...)` | The **MCP server**: only retrieves the prompt messages |
| `chatClient.prompt(...)` | The **LLM**: happens afterwards |

### Simulating the "user picks a prompt" button

No real MCP UI here, so an endpoint stands in for the button:

```java
@GetMapping("/plan")
public String plan(@RequestParam("hours") String hours) {
    return chatService.planDay(hours);
}
```

`GET /plan?hours=4` = the user clicked **Plan My Day** and entered 4. In the video: add tasks ("Read about Docker", "Attend college seminar"…), then `/plan?hours=4` returns a time-boxed plan for them. A big prompt stored on the server, triggered with a tiny call. (The LLM also made up specific timings on its own. With little info it fills gaps, a small reminder of lecture 4's hallucination point.)

---

## All three working together

[46:00](https://www.youtube.com/watch?v=MUttM7fIqO4&t=2760s)

```
taskServer
├── Tools:     create_task, list_tasks, complete_task
├── Resources: task://guidelines
└── Prompts:   plan_day
```

`GET /plan?hours=4`:

1. **User controls the prompt.** Picks `plan_day` with `availableHours = 4`. The server generates the prompt.
2. **Application controls the resource.** Reads `task://guidelines` and puts it in context (deep work before noon, meetings after 2 PM…).
3. **Model controls the tool.** The prompt says "check pending tasks". The LLM sees `list_tasks` and calls it.

```
USER ── selects ──► PROMPT plan_day(4)               (user-controlled)
                          ▼
APPLICATION ── reads ──► task://guidelines           (application-controlled)
                          ▼
LLM ── decides it needs tasks ──► list_tasks         (model-controlled)
                          ▼
MCP SERVER → pending tasks → LLM → final day plan
```

### Summary table

| Primitive | Main purpose | Controlled by | Example |
|---|---|---|---|
| Tool | Expose callable capabilities | Model | `list_tasks` |
| Resource | Expose context/reference data | Application | `task://guidelines` |
| Prompt | Expose reusable interaction templates | User | `plan_day` |

Three questions to ask when designing a server:

- **Tools:** what should the **model** be able to call?
- **Resources:** what context should the **application** be able to provide?
- **Prompts:** what reusable workflows should the **user** be able to choose?

Why put all this in an MCP server instead of inside one app? Because then **any** AI application can use the same tools, resources and prompts. Resources can even be backed by RAG for large document sets.

## Key takeaways

1. MCP servers expose **tools** (capabilities), **resources** (context) and **prompts** (workflows).
2. Tools can retrieve info too. The deciding question is intent and **who controls it**.
3. Resources are discovered (`resources/list`) and read (`resources/read`) separately, identified by **URIs** (not necessarily URLs), with name, description and MIME type.
4. A resource does **not** automatically enter LLM context. The host decides when to read and supply it.
5. Resource selection strategies: user selection, application state, retrieval/search over metadata, or LLM routing. The last is still the app's choice.
6. Resources can be static, dynamic, or templated (`user://{username}`).
7. Prompts are reusable, parameterised message templates (`prompts/list`, `prompts/get`), surfaced to users as buttons or slash commands.
8. A prompt produces **messages**, not the final answer. The LLM still answers.
9. Prompts can instruct the LLM to use tools, so all three primitives compose in one flow.
10. **Tools → model controlled. Resources → application controlled. Prompts → user controlled.**

## Interview questions

**Q: When would you expose something as a resource rather than a tool?**
When it's reference context the application should decide to include (docs, config, the open file). Use a tool when the model should decide to call it, e.g. a parameterised search or an action.

**Q: Does reading a resource put it in the model's context automatically?**
No. The host reads it and chooses whether and how to add it to the prompt.

**Q: How might a host decide which of 500 resources to read?**
Let the user attach one, use app state (current file), run semantic search over resource descriptions, or ask an LLM to route.

**Q: What's an MCP prompt, and does it call the LLM?**
A server-provided, possibly parameterised message template. It returns messages. The host then sends them to the LLM.

**Q: Explain the control model of MCP primitives.**
Tools are model-controlled (the LLM decides to call them), resources are application-controlled (the host decides what context to load), and prompts are user-controlled (the user explicitly picks a workflow).
