# 03 — Calling an LLM From Your Application

**Video:** [FDE Full Course #3 — Integrating LLM with Your Application](https://www.youtube.com/watch?v=SFD22DT0wCI) (48m)
**Instructor notes + code:** [Lecture 03](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2003) (Spring Boot, FastAPI and Express versions)

So far it has all been theory. This lecture makes the first real call: from Postman, then from a Spring Boot app, to build a support-ticket summariser API.

---

## Contents

1. [ChatGPT is not the LLM](#chatgpt-is-not-the-llm)
2. [Why an LLM needs tools](#why-an-llm-needs-tools)
3. [The most important principle](#the-most-important-principle)
4. [Talking to the model directly](#talking-to-the-model-directly)
5. [Doing it in Postman](#doing-it-in-postman)
6. [Reading the response and token usage](#reading-the-response-and-token-usage)
7. [Doing it in code: the ticket summariser](#doing-it-in-code-the-ticket-summariser)
8. [Why use Spring AI at all?](#why-use-spring-ai-at-all)
9. [The problem we've left open](#the-problem-weve-left-open)

---

## ChatGPT is not the LLM

[0:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=0s)

When you type "Explain Docker to me like I'm a beginner" into ChatGPT, you are not sending text straight to a Transformer. There's a whole application in between: a frontend, a server, guardrails that block junk requests, tokenization, memory, tools. ChatGPT, Claude, Gemini and DeepSeek are all **applications** built around a model.

> An LLM is a capability. A GenAI application is software built around that capability.

**The car analogy:** the LLM is the engine. An engine makes power but it isn't a car. You also need steering, brakes, transmission, a dashboard, sensors, navigation and safety systems.

```
          GenAI application
     ┌─────────┼─────────┐
   Memory    Search    Tools
     └─────────┼─────────┘
               ▼
              LLM
```

The LLM provides language intelligence. The application decides **what information reaches the model and what happens with its answer**. Controlling that flow is most of what GenAI engineering is.

### Raw LLM vs app with tools

Ask "What's the weather in Delhi right now?"

**Raw LLM:** `prompt → LLM → answer`. It only has what's in your request plus what it learned in training. Every model has a **knowledge cutoff date**, the point where training on internet data stopped. It has no live weather.

**ChatGPT (an app with tools):** the video shows ChatGPT saying "Searching the web…" and answering "Delhi is about 31°C with hazy sunshine".

```
User → Application → detects live info is needed
     → Weather API / web search tool → current data
     → LLM → natural-language answer → User
```

The *wording* comes from the LLM. The *live data* comes from the application and its tools. Tools are just normal deterministic functions: weather API, calculator, database query, web search, email service, payment service.

---

## Why an LLM needs tools

[12:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=720s)

An LLM generates tokens. It's not a calculator, database, search engine or OS.

The video tests this. Ask a raw model to multiply two big random numbers and you get a big number that may or may not be right. Ask ChatGPT and it shows "Calculating the product of two integers…". It used a tool, so the answer is reliably correct. Same with "count the `*` characters in this string": ChatGPT writes a tiny script, runs it in a code-execution tool (the LLM can't run code itself), and reports the right count, 32.

Simple maths often comes out right anyway because the pattern showed up a lot in training. When you need *reliable* computation, give the model a real tool.

### The LLM as translator

A calculator doesn't understand "what is this times this?". A weather API doesn't understand English. So the LLM sits in the middle:

```
Natural-language input
  → LLM → structured tool input
  → Tool → tool output
  → LLM → natural-language response
```

The LLM is useful twice: once to understand what the user wants and shape the tool input, and again to phrase the result. Something has to tell the model which tools it has access to. Later lectures (5, 14, 15) cover how.

---

## The most important principle

> **The LLM only knows what reaches the LLM.**

| If the model needs… | …then the application has to |
|---|---|
| Conversation history | Send the relevant history with the request |
| Database information | Fetch it and pass it in |
| Current weather | Call a live weather source |
| Reliable calculations | Provide a calculator / code-execution tool |
| To perform an action | Actually execute the action itself |

The model doesn't magically get the capabilities of the application around it. Much of this course is about getting the right information into the request.

---

## Talking to the model directly

[23:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=1380s)

You don't need ChatGPT to use an LLM. Your app can call the provider's API like any other backend call (`POST /payments` with some JSON). There's nothing special about it being AI:

```
Our application ──HTTP request──► LLM provider ──HTTP response──► Our application
```

When your backend calls the API, ChatGPT's web search and other app features don't come along. You're talking to the raw model. Any tools are yours to build.

### Four things every LLM request needs

| Part | Question it answers |
|---|---|
| **Endpoint** | Where should the request go? |
| **API key** | Who is making the request? |
| **Model** | Which model should process it? |
| **Input** | What should it process? |

**Why an API key?** Running models costs a lot of compute. If anyone could call the model freely, the provider couldn't bill or rate-limit anyone. The key identifies and authorises your app, and usage is charged to it. Keys are paid (OpenAI needs a minimum top-up, around $5). Some providers have free tiers, but free models often become paid later, so the instructor recommends using a paid one.

**Never expose keys publicly or commit them to GitHub.**

**Why choose a model?** Providers offer many models that trade off cost, speed, reasoning ability, context size and multimodality. The latest flagship is expensive. A small one ("Luna" in the video) is cheap and fine for learning.

The input you send eventually goes through everything from lecture 2:

```
Application → HTTP request → input text → tokens → Transformer
            → generated tokens → API response → Application
```

---

## Doing it in Postman

[26:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=1560s)

The FDE scenario: you're embedded with a SaaS company's support team. A ticket arrives:

> Hi team, Our production deployment has failed three times since yesterday. It looks like the payment-service container keeps restarting. Customers are occasionally getting 502 errors. We already tried restarting the deployment manually but the issue returned after approximately twenty minutes. Can someone investigate urgently?

Instead of an engineer reading every long ticket, ask the model to *"Summarize this support ticket in 2 lines."* Expected output:

> Production payment-service is repeatedly restarting and causing intermittent 502 errors. Manual restarts only temporarily resolve the issue.

Steps:

1. Create an API key on the provider's platform. Keep it private.
2. Postman → New → HTTP Request → `POST https://api.openai.com/v1/responses` (OpenAI's Responses API).
3. Authorization → Type: **Bearer Token** → paste the key. That adds `Authorization: Bearer <API_KEY>`.
4. Header `Content-Type: application/json` (Postman adds it for a JSON body).
5. Body → raw → JSON:

```json
{
  "model": "gpt-5.6-luna",
  "input": "Explain Docker in two simple sentences."
}
```

What happens when you hit Send:

```
Postman ──HTTPS──► OpenAI API
                     ├─ validates credential
                     ├─ processes request
                     └─ selects requested model
                          ▼
                         LLM: processes input tokens, generates output tokens
                          ▼
                    OpenAI API ──JSON response──► Postman
```

You've now talked to an LLM without ChatGPT. (In the video the first attempt returned a 404 because of a URL typo. Always double-check the endpoint.)

The video also sends "Hi, my name is Aditya" and then "What's my name?" in a second request. The model doesn't know. Each API call is independent. Lecture 4 explains why and fixes it.

---

## Reading the response and token usage

[32:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=1920s)

```json
{
  "id": "resp_...",
  "object": "response",
  "status": "completed",
  "model": "gpt-5.6-luna",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [
        { "type": "output_text", "text": "Docker packages an application..." }
      ]
    }
  ],
  "usage": {
    "input_tokens": 15,
    "output_tokens": 30,
    "total_tokens": 45
  }
}
```

Two kinds of stuff:

- **Generated content** — the actual answer (`output[].content[].text`). This is what reaches the user.
- **Metadata** — response ID, model, status, token usage. This is what you use to manage and monitor the interaction.

### Tokens are now an engineering problem

> **Input tokens** are what the model reads. **Output tokens** are what it generates.

Send a 10-page incident report and ask for a summary: the 10 pages are input tokens, the 10-line summary is output tokens. Your bill is input tokens + output tokens. So tokens drive:

```
Tokens → usage, cost, latency, context size
```

That's why the token theory from lecture 2 matters even when you're "just calling an API".

---

## Doing it in code: the ticket summariser

[34:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=2040s)

The course uses **Spring AI** for the main code (the repo also has FastAPI and Express versions). The course is language independent and the concepts carry over.

Spring AI doesn't provide intelligence. OpenAI, Anthropic and Gemini provide the models. Spring AI gives Java abstractions for talking to them.

### 1. Dependencies (`pom.xml`)

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

### 2. Config — the same things you gave Postman

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}   # read from an env var
      chat:
        model: gpt-5-mini
```

```bash
export OPENAI_API_KEY="your-api-key"
```

Don't hard-code `api-key: sk-xxxx`. It *will* end up on GitHub.

Postman needed a key, a model and an input. The config now covers the key ✓ and model ✓. The input comes from the Java code, through `ChatClient`.

### 3. The service

```java
@Service
public class SummarizerService {

    private final ChatClient chatClient;

    public SummarizerService(ChatClient.Builder builder) {
        this.chatClient = builder.build();   // pre-configured from application.yml
    }

    public String summarize(String ticket) {
        return chatClient.prompt()
                .user("Summarize this support ticket in two sentences:\n\n" + ticket)
                .call()
                .content();
    }
}
```

Don't memorise it as syntax. Read it as the Postman request:

| Call | Meaning |
|---|---|
| `prompt()` | Start building a model request |
| `.user(...)` | The text for the model (the `input`) |
| `.call()` | Send it to the configured model |
| `.content()` | Give me just the generated text |

### 4. The controller

```java
@RestController
@RequestMapping("/api")
public class SummarizerController {

    private final SummarizerService summarizerService;

    public SummarizerController(SummarizerService summarizerService) {
        this.summarizerService = summarizerService;
    }

    @PostMapping("/summarize")
    public String summarize(@RequestBody SummarizeRequest request) {
        return summarizerService.summarize(request.text());
    }
}
```

```
POST http://localhost:8080/api/summarize
{ "text": "Payment was deducted three days ago but the order still shows
           Payment Processing. I contacted support twice and need the
           product tomorrow for my daughter's birthday." }
```

### The full flow

```
Client / Postman
  → POST /api/summarize → Controller
  → SummarizerService → ChatClient
  → OpenAI API → LLM
  → generated summary
  → ChatClient → Service → Controller → Client
```

Your first GenAI-powered API. In the video, a long angry Zomato-style food-delivery complaint comes back as a two-line summary a support agent can read in seconds.

### The same thing in Python (from the repo)

```python
from openai import AsyncOpenAI

client = AsyncOpenAI()  # reads OPENAI_API_KEY from env

async def summarize_ticket(ticket: str) -> str:
    response = await client.responses.create(
        model="gpt-5.6-luna",
        input=f"Summarize this support ticket in 2 lines:\n\n{ticket}",
        store=False,
    )
    return response.output_text
```

Node is nearly identical: `client.responses.create({ model, input })` and read `response.output_text`.

---

## Why use Spring AI at all?

You could use `RestClient` and build the same JSON request by hand. For one call, that's fine. The abstraction pays off later when you need:

- Multiple messages and conversation memory
- Streaming
- Structured outputs
- Tool calling
- RAG
- Multiple model providers
- Observability

If you hand-roll provider-specific HTTP calls and response parsing for all of that, you get tightly coupled to one provider's API format. An abstraction lets you swap OpenAI for Anthropic or Gemini without rewriting everything.

When you need more than the text (token usage, metadata), ask for the full response:

```java
ChatResponse response = chatClient.prompt()
        .user(prompt)
        .call()
        .chatResponse();
```

---

## The problem we've left open

[46:00](https://www.youtube.com/watch?v=SFD22DT0wCI&t=2760s)

```java
.user("Summarize this support ticket in two sentences:\n\n" + ticket)
```

Two very different things are glued into one user message:

```
"Summarize this support ticket in two sentences"  ← application instruction
"Payment was deducted…"                           ← end-user data
```

What stops a user from sending "What is 2 + 2?" as the "ticket"? Nothing. The video tries it and gets back *"Customer asked what is 2 + 2. Answer: 4."* That's useless output that shouldn't be returned.

Should instructions from *our app* and text from *the user* be treated the same? No. That's where **system prompts vs user prompts** come in, the start of lecture 4.

---

## Key takeaways

1. ChatGPT is an application. The LLM is the engine inside it.
2. Live data, reliable maths and real actions come from **tools** the application provides, not the model.
3. The LLM acts as a translator between natural language and structured tool input/output.
4. **The LLM only knows what reaches the LLM.**
5. Calling an LLM is a normal HTTP API call: endpoint, API key, model, input.
6. Keep API keys in environment variables, never in code or Git.
7. Responses have content and metadata. Token usage (input + output) drives cost, latency and context limits.
8. Spring AI's `ChatClient` (`prompt().user().call().content()`) wraps that same HTTP call and becomes valuable once you need memory, streaming, tools, RAG or multiple providers.
9. Mixing app instructions and user data in one message is a problem. Next lecture fixes it.

## Interview questions

**Q: What's the difference between an LLM and a GenAI application?**
The LLM is a model that generates tokens. The application around it handles UI, auth, guardrails, memory, retrieval, tools and actions, and decides what reaches the model.

**Q: How does ChatGPT know today's weather if models have a training cutoff?**
The app detects that live data is needed, calls a search or weather tool, and passes the result to the model, which phrases the answer.

**Q: What decides the cost of an LLM API call?**
Mainly input tokens plus output tokens, at the chosen model's per-token rates. Bigger models cost more per token.

**Q: Why use an SDK/framework instead of raw HTTP?**
To avoid coupling to one provider's request/response format, and to get memory, streaming, tool calling, structured output, RAG and observability without building them yourself.
