# 05 — Tools and AI Agents

**Video:** [FDE Full Course #5 — Introduction to AI Agents](https://www.youtube.com/watch?v=jS3cHV43Uxs) (1h 05m)
**Instructor notes + code:** [Lecture 05](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2005) (Spring, Python, JS)

Up to now the LLM could only talk. This lecture gives it hands. First a calculator, weather and currency tools on the chatbot. Then a real agent that builds whole websites by writing files to disk.

---

## Contents

1. [Generating an action ≠ performing it](#generating-an-action--performing-it)
2. [LLM = brain, tools = hands](#llm--brain-tools--hands)
3. [The LLM as translator to structured intent](#the-llm-as-translator-to-structured-intent)
4. [What a tool is](#what-a-tool-is)
5. [Building tools with Spring AI](#building-tools-with-spring-ai)
6. [Multiple tools, multiple steps](#multiple-tools-multiple-steps)
7. [What actually happens during a tool call](#what-actually-happens-during-a-tool-call)
8. [Information tools vs action tools](#information-tools-vs-action-tools)
9. [When does it become an agent?](#when-does-it-become-an-agent)
10. [Practical: the website-builder agent](#practical-the-website-builder-agent)
11. [Security: capability, permission, sandbox](#security-capability-permission-sandbox)
12. [Human in the loop](#human-in-the-loop)

---

## Generating an action ≠ performing it

[0:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=0s)

Inside, an LLM is `input tokens → Transformer → probability distribution → next token → next token → …`. That can produce very smart text. But ask it to "create a folder called portfolio" and it produces:

```
mkdir portfolio
```

Those are characters *describing* an action. No folder exists. **The LLM generated an action. It didn't perform one.**

---

## LLM = brain, tools = hands

[5:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=300s)

Picture a brilliant person locked in a room with no hands. Ask them to turn on the AC and they'll tell you exactly which remote button to press, but they can't press it.

A raw LLM is `TEXT IN → LLM → TEXT OUT`. On its own it can't reach databases, file systems, live internet data, calculators, terminals, email, calendars, payments or weather. The surrounding application has to provide those.

```
LLM = Brain
Tools = Hands
LLM + Tools → a system that can take actions
```

```
User → Application → LLM
                      │ needs a capability
                      ▼
                    Tool
```

### Why give it a calculator?

`938475 × 736294`. A modern LLM *might* get it right from memorised patterns, but you can't be sure. (In the video the model returns a plausible but wrong big number.) Split the job:

1. **LLM:** understand the intent → multiply, `a = 938475`, `b = 736294`.
2. **Calculator:** do the actual maths deterministically.
3. **LLM again:** turn the result into a sentence.

LLM (great at language) + program (great at deterministic computation). This pattern goes far beyond calculators.

---

## The LLM as translator to structured intent

[10:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=600s)

```java
public double add(double a, double b) { return a + b; }
```

The computer understands `add(10, 20)`. Humans say "Add 10 and 20", "What's ten plus twenty?", "Calculate the sum…", "I had 10 apples and got 20 more, how many now?". Same intent, completely different strings.

Without an LLM you're back to rule-based AI:

```java
if (message.contains("add") || message.contains("sum") || ...) { ... }
```

Then someone says "plus", "combine", "total", or "I have ₹300 and got ₹200 more" with no keyword at all. Software wants a structured contract:

```json
{ "operation": "ADD", "a": 10, "b": 20 }
```

> **One of the most important capabilities of LLMs:** translating messy human intent into structured, machine-readable information.

```
Natural language → LLM → structured intent → backend code
```

Food delivery example:

| User says | Backend receives |
|---|---|
| "My burger never arrived and I want my money back." | `{"intent": "REFUND_REQUEST", "reason": "ITEM_NOT_DELIVERED", "item": "burger"}` |
| "I ordered a pizza 90 minutes ago. Where is it?" | `{"intent": "ORDER_STATUS"}` |

No thousands of if-conditions. The LLM is the translation layer.

### Why not just say "always answer in JSON"?

Usually it works. Sometimes you get `Sure! Here's the JSON: {...}`, or `"operation": "multiplication"` when your code expects `MULTIPLY`. Your code would string-match `"add"` and silently fail on `"addition"`. For software you need output that follows a defined **contract**, not "approximately correct JSON". (Lecture 13 covers structured outputs properly.)

> Non-deterministic intelligence + deterministic software.

---

## What a tool is

[18:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=1080s)

Strip away the AI vocabulary. **A tool is a normal function that the application makes available to the LLM.**

```java
public double multiply(double a, double b) { return a * b; }
```

Nothing AI about it. What matters is how it's **described** to the model:

```
Tool name:   multiply
Description: Multiplies two numbers.
Parameters:  a: number, b: number
```

User asks "What is 17 times 42?". The model sees the request plus the list of available tools (add, subtract, multiply, divide), picks one, and instead of answering it emits:

```json
{ "tool": "multiply", "arguments": { "a": 17, "b": 42 } }
```

That's a **tool call** (also called a **function call**).

A tool definition is like a REST API contract (`POST /users` needs `name` and `email`). The app tells the model "here are the actions you can take, what each does, and what parameters they need". The LLM then behaves like an **intelligent client choosing which API to call**.

### A natural-language interface to software

```
                 ┌→ Calculator
User → LLM → tool choice ├→ Database
                 ├→ Weather API
                 ├→ File system
                 └→ Email system
```

Traditionally the user adapts to the software: buttons, forms, commands, API formats. With an LLM, **the software adapts to the user's language**.

Example: "I bought 13 boxes with 24 bottles each, how many bottles?" → `multiply(13, 24)` → `312` → "You have 312 bottles."

---

## Building tools with Spring AI

[20:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=1200s)

The chatbot goes from `USER → LLM → ANSWER` to `USER → LLM → (does a tool handle this better?) → TOOL → RESULT → LLM → ANSWER`.

### Tool 1 — Calculator

```java
@Component
public class CalculatorTool {

    @Tool(description = """
            Performs arithmetic calculations.
            Supported operations: add, subtract, multiply, divide, mod, power.
            """)
    public double calculate(
            @ToolParam(description = "Operation: add, subtract, multiply, divide, mod, power")
            String operation,
            @ToolParam(description = "First number") double a,
            @ToolParam(description = "Second number") double b) {

        return switch (operation.toLowerCase()) {
            case "add"      -> a + b;
            case "subtract" -> a - b;
            case "multiply" -> a * b;
            case "divide"   -> a / b;
            case "mod"      -> a % b;
            case "power"    -> Math.pow(a, b);
            default -> throw new IllegalArgumentException("Unsupported operation");
        };
    }
}
```

- `@Component` lets Spring create the object.
- `@Tool(description=…)` tells the model what the tool does and when to use it.
- `@ToolParam(description=…)` explains each argument. Listing the exact allowed strings stops the model from sending "addition" instead of "add".

The model **never sees your Java code** or your `switch`. It only gets the name, description and parameter schema:

```
AVAILABLE TOOL
Name: calculate
Description: Performs mathematical calculations.
Inputs: operation: string, a: number, b: number
```

Hook it up:

```java
String response = chatClient.prompt()
        .user(message)
        .tools(calculatorTool)
        .call()
        .content();
```

> The model didn't change. The user's prompt didn't change. We just gave the model another capability.

Tools can be passed per request (`.tools(...)`) or set as defaults on the client (`.defaultTools(...)`).

**The video's debugging story:** at first the model sometimes still did the maths itself. The fix was a system-prompt rule, *"Always use the calculator tool, even for trivial calculations"*. A `System.out.println("Calculator tool called")` inside the tool proves in the console when it's actually used.

### Tool 2 — Weather (live data)

The calculator is about *reliability*. Weather is about *information the model doesn't have*.

```java
@Tool(description = "Get the current weather of a city.")
public String currentWeather(@ToolParam(description = "Name of the city") String city) {
    return restClient.get()
            .uri(b -> b.path("/current.json")
                       .queryParam("key", apiKey)
                       .queryParam("q", city).build())
            .retrieve()
            .body(String.class);
}
```

It calls weatherapi.com (free API key). The model only needs to know the tool's name, purpose and that it takes `city`. It doesn't care how the weather service works.

### Tool 3 — Currency exchange

```java
@Tool(description = "Gets the latest exchange rate between two currencies.")
public String getExchangeRate(
        @ToolParam(description = "Source currency code, e.g. USD") String from,
        @ToolParam(description = "Target currency code, e.g. INR") String to) {
    return restClient.get().uri("/v2/rate/{from}/{to}", from, to)
            .retrieve().body(String.class);   // Frankfurter API, no key needed
}
```

All three on one endpoint:

```java
chatClient.prompt()
        .system(SYSTEM_PROMPT)
        .messages(history)
        .tools(calculatorTool, weatherTool, currencyExchangeTool)
        .call()
        .content();
```

The system prompt from the repo:

```
You are a helpful AI assistant with access to external tools.
1. For arithmetic calculations, ALWAYS use the calculator tool.
2. Always use calculator tool for even trivial calculation.
3. For current weather, ALWAYS use the currentWeather tool.
4. For currency conversion or exchange rates, ALWAYS use the convertCurrency tool.
5. You may call multiple tools when solving a multi-step request.
6. After receiving tool results, explain the answer naturally.
7. Never invent current weather or exchange-rate information.
```

You don't need `/calculator`, `/weather`, `/currency` endpoints. One `/chat`, and the LLM figures out which capability the sentence needs.

---

## Multiple tools, multiple steps

[36:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=2160s)

**Sequential:** "Convert $120 to INR and divide it equally among 6 people."

```
LLM → convertCurrency(120, USD, INR) → ₹X
    → LLM → calculate(divide, X, 6) → ₹Y
    → LLM → final answer
```

Nobody wrote `currencyTool(); calculatorTool();`. The model worked out that the goal needs two steps in that order. In the video, "I have ₹10,000, how many dollars at today's rate?" triggers the exchange-rate tool and then the calculator, and the console logs show both.

**Parallel/independent:** "I'm travelling to Delhi. What's the weather there, and how much is $500 in INR?"

```
      ┌─ currentWeather("Delhi")
LLM ──┤
      └─ convert(500, USD, INR)
```

One sentence, two external systems, one coherent answer.

---

## What actually happens during a tool call

[40:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=2400s)

Remember the client never talks to the LLM directly. Your Spring app sits in between.

**1. User sends a message.** The app sends it to the model along with the tool definitions.

```
USER MESSAGE:    What is 53 multiplied by 29?
AVAILABLE TOOLS: calculate(...), currentWeather(...), getExchangeRate(...)
```

**2. Model requests a tool** instead of answering:

```json
{ "tool": "calculate", "arguments": { "operation": "multiply", "a": 53, "b": 29 } }
```

Meaning: *"Application, please run this for me."*

**3. Spring AI executes it.** It maps the request to `calculatorTool.calculate("multiply", 53, 29)` and the JVM runs it → `1537`.

> The LLM did not enter the JVM and run Java. The **application** ran Java on the model's behalf.

**4. Result goes back to the model:**

```
USER:      What is 53 × 29?
ASSISTANT: call calculate(multiply, 53, 29)
TOOL:      1537
```

The LLM is called *again*. **One user request can mean several model calls** (and several times the tokens).

**5. Model decides what's next.** Goal done → "53 multiplied by 29 is 1,537." Not done → request another tool.

### The tool-calling loop

```
        ┌──────────┐
   ┌───►│   LLM    │
   │    └────┬─────┘
   │    Need a tool?
   │     YES     NO → RESPONSE
   │      ▼
   │   ┌──────┐
   │   │ TOOL │
   │   └──┬───┘
   │  tool result
   └──────┘
```

```
Think → Act → Observe → Think again
```

That loop is the foundation of agents.

---

## Information tools vs action tools

| | Information tools | Action tools |
|---|---|---|
| Purpose | "Give me info I don't have" | "Do something for me" |
| Examples | weather, currency rates, DB lookups, search, customer info, inventory | `sendEmail()`, `createUser()`, `refundOrder()`, `bookFlight()`, `createFile()`, `runCommand()` |
| Effect | Lets the model **observe** the world | Lets the model **change** the world |

The difference becomes critical once you think about safety.

---

## When does it become an agent?

[44:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=2640s)

There's no single official definition. The instructor treats agency as a progression:

| Level | Shape | What's new |
|---|---|---|
| 1. Raw LLM | User → LLM → Response | Text only. Nothing outside changes. |
| 2. LLM + one tool | User → LLM → Calculator → LLM → Response | The model can delegate a task (tool calling). |
| 3. LLM + many tools | User → LLM → {calculator, weather, currency} | The model must **select** the right tool. |
| 4. Multi-step toward a goal | Understand goal → currency tool → observe → calculator → observe → answer | A **sequence of decisions** toward a goal. Now we're close to an agent. |

A single calculator call is better described as a **workflow** than an agent.

### First-principles definition: six pieces

1. **Goal** — something to achieve. *"Build me a portfolio website."*
2. **Decision maker** — the LLM, constantly asking *"What should I do next?"*
3. **Actions** — its tools. `createDirectory()`, `writeFile()`, `readFile()`, `listFiles()`.
4. **Environment** — what the tools act on. Here, the file system.
5. **Observations** — what comes back. *"Directory created." "File written." "Files: index.html, style.css, script.js."*
6. **Loop** — after each observation, decide again, until the goal is complete.

```
GOAL → LLM decides → TOOL acts → ENVIRONMENT changes → OBSERVATION → LLM decides again → … → DONE
```

> **Goal → Decide → Act → Observe → Decide again**

### Agent vs workflow

| | You define | LLM decides |
|---|---|---|
| **Workflow** | WHAT + HOW (step 1 create dir, step 2 HTML, step 3 CSS…) | — |
| **Agent** | WHAT ("build a website") + the available capabilities | Most of the HOW |

Real systems usually mix both.

---

## Practical: the website-builder agent

[48:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=2880s)

Goal: `POST /website` with *"Build a modern landing page for a premium coffee shop called BrewLab"* should produce real files:

```
generated-sites/
└── brewlab/
    ├── index.html
    ├── style.css
    └── script.js
```

The LLM already knows HTML, CSS and JS. What's missing?

> Knowing how to write code is different from having permission and capability to write files.

Without tools you get `<html>…</html>` in a chat bubble. With tools it actually creates `index.html`, `style.css`, `script.js`.

### Four tools are enough

```
createDirectory(path)
writeFile(path, content)
readFile(path)
listFiles(path)
```

Notice what it does **not** get: `executeAnyTerminalCommand(String command)`.

### WorkspaceTools

```java
@Component
public class WebsiteTools {

    private final Path workspace =
            Path.of("generated-sites").toAbsolutePath().normalize();

    @Tool(description = "Creates a new directory inside the website workspace.")
    public String createDirectory(
            @ToolParam(description = "Relative directory path, e.g. brewlab") String path) {
        try {
            Files.createDirectories(safePath(path));
            return "Directory created successfully: " + path;
        } catch (IOException e) {
            return "Failed to create directory: " + e.getMessage();
        }
    }

    @Tool(description = """
            Creates or overwrites a text file inside the website workspace.
            Use this to create HTML, CSS and JavaScript files.
            """)
    public String writeFile(
            @ToolParam(description = "Relative file path, e.g. brewlab/index.html") String path,
            @ToolParam(description = "Complete content of the file") String content) {
        try {
            Path file = safePath(path);
            Files.createDirectories(file.getParent());
            Files.writeString(file, content, StandardCharsets.UTF_8);
            return "File written successfully: " + path;
        } catch (IOException e) {
            return "Failed to write file: " + e.getMessage();
        }
    }

    @Tool(description = "Reads the contents of an existing file from the website workspace.")
    public String readFile(@ToolParam(description = "Relative file path") String path) { ... }

    @Tool(description = "Lists all files and directories inside a website project.")
    public String listFiles(@ToolParam(description = "Relative directory path") String path) { ... }

    private Path safePath(String path) {
        Path resolved = workspace.resolve(path).normalize();
        if (!resolved.startsWith(workspace)) {
            throw new IllegalArgumentException("Access outside generated-sites is not allowed");
        }
        return resolved;
    }
}
```

Points worth noticing:

- **Errors come back as strings** ("Failed to write file: …") rather than crashing, so the model can observe the failure and try again.
- **Abstraction:** the model just knows `writeFile(path, content)`. Whether it's `Files.writeString`, a DB, S3 or a remote service is hidden. Expose capabilities at the level the model needs.
- **`readFile` makes it agentic:** write → read back → notice a problem → write again. Observing the result of its own actions is a big part of agent behaviour.

### The job description (system prompt)

```
You are an expert frontend website developer.
Your job is to create complete static websites using the available tools.
1. Create a separate directory for every website.
2. Create index.html.
3. Create style.css.
4. Create script.js when JavaScript is useful.
5. Build modern, beautiful and responsive websites.
6. Use only HTML, CSS and vanilla JavaScript.
7. Do not just return website code in your response. Actually create the files using tools.
8. After creating the website, list the project files.
9. Read important files again if needed and fix obvious problems.
10. Finish only when the complete website has been created.
```

Rule 7 is the key one: **don't describe the action, perform it.** Without it the model may just paste HTML into the reply.

### The controller is tiny

```java
@PostMapping
public String generateWebsite(@RequestBody String message) {
    return websiteService.generate(message);
    // → chatClient.prompt().system(SYSTEM_PROMPT).messages(history)
    //             .tools(websiteTools).call().content();
}
```

There's almost no orchestration code. The model plus the tool-calling loop make the decisions.

### General purpose

Send a different prompt with no code changes: *"Create a futuristic landing page for NeuralForge, a company that builds AI robots. Dark space-like design, glowing cards, subtle animations."* → `neuralforge/index.html, style.css, script.js`. The video also builds an "Aditya portfolio" and a Swiggy clone. Because history is kept, you can follow up with "now add a dark-mode toggle" and it edits the existing files.

We didn't build a coffee-shop generator. We built a **website-building agent**. The goal changes. The tools stay the same.

---

## Security: capability, permission, sandbox

[52:00](https://www.youtube.com/watch?v=jS3cHV43Uxs&t=3120s)

Why not give it a `runCommand(String command)` tool? It could do `mkdir`, `cd`, `touch`, notice mistakes, `rm -rf` and retry, all fully agentic. But the process running your Spring app can reach files, env variables, network, your source code, system commands and credentials. An LLM blindly running any shell command could do things you never wanted.

> **Tool = Capability + Permission.** Exposing a function doesn't just give the model a capability. It gives it permission to affect part of your system.

### Sandbox the environment

```
project/
├── src/
└── generated-sites/   ← the only place the agent may touch
```

Not `/`, not `~/Documents`, not `src/main/java`, not `.env`. Give the agent the **minimum capabilities** it needs.

### Path traversal

If the model asks for `../../secret.txt`, `safePath` resolves and normalises the path and rejects anything outside `generated-sites/`. `brewlab/index.html` is fine. `../../secret.txt` throws.

> **The LLM decides WHAT it wants to do. The APPLICATION decides WHAT it is allowed to do.**
> The LLM is never the security boundary.

---

## Human in the loop

Not every tool should run immediately. Auto-sending a routine email might be fine. Auto-running `refundCustomer(₹50,000)` is not.

```
LLM requests refund → PAUSE → human approves?
                                YES → execute
                                NO  → reject
```

Use it for financial transactions, destructive operations, external communication, security-sensitive and irreversible actions. **The more consequential the action, the stronger the controls around the tool.**

---

## The full mental model

```
Raw LLM:        TEXT IN → LLM → TEXT OUT
+ tools:        User → LLM → Tool → Result → LLM → Response
+ many tools:   User → LLM → {Calculator | Weather | Currency}
+ env + loop:   GOAL → LLM decides → TOOL acts → ENVIRONMENT changes
                     → OBSERVATION → LLM decides again → … → GOAL DONE
```

```
LLM + Tools + Environment + Observations + Decision loop + Goal = AI Agent
```

> A chatbot mainly answers. An agent decides, acts, observes the result, and keeps working toward a goal.

## Key takeaways

1. LLMs generate tokens. They don't perform actions by themselves.
2. LLM = brain, tools = hands.
3. Tools are ordinary functions exposed through a contract (name, description, parameters).
4. The LLM turns natural language into structured tool calls.
5. The **application** executes the code, APIs, DB queries and file operations, never the model.
6. Tool results go back to the model, which decides the next step. One request can mean many LLM calls.
7. Information tools observe. Action tools change things.
8. Agent = goal + decision maker + tools + environment + observations + loop.
9. Workflow: you define what + how. Agent: you define what, the model decides much of how.
10. Tool = capability + permission. Expose the minimum, sandbox the environment, validate paths.
11. The app, not the LLM, enforces security.
12. High-impact actions need human-in-the-loop approval.

## Interview questions

**Q: What is tool/function calling?**
The app describes available functions (name, description, parameter schema) to the model. Instead of answering, the model can return a structured request to call one. The app executes it and sends the result back so the model can continue.

**Q: Does the LLM execute the tool?**
No. The model only emits a structured request. The application (e.g. Spring AI) runs the actual code and feeds the result back.

**Q: Agent vs workflow?**
A workflow hard-codes the steps. An agent gets a goal and capabilities, and the LLM decides which actions to take and in what order, looping on observations until done.

**Q: How would you make a file-writing agent safe?**
Only expose narrow tools (no raw shell), restrict them to a sandbox directory, normalise and validate every path against traversal, return errors as observations, and require human approval for destructive or costly actions.

**Q: Why return errors as strings from tools instead of throwing?**
So the model can see what went wrong and try again. That's part of the observe-then-decide loop.
