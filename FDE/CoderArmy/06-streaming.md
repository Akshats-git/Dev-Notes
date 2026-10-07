# 06 — Streaming: From LLM Tokens to a Live Chat UI

**Video:** [FDE Full Course #6 — Build ChatGPT-Style Streaming](https://www.youtube.com/watch?v=r4x3Zm4WIWA) (42m)
**Instructor notes + code:** [Lecture 06](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2006) (Spring, Python, JS + a chat frontend)

ChatGPT's answers appear word by word, as if it's typing. Our Tomato bot makes you wait and then dumps the whole answer. This lecture fixes that end to end: provider → Spring → HTTP → browser → one growing chat bubble.

---

## Contents

1. [Generation vs delivery](#generation-vs-delivery)
2. [Non-streaming vs streaming](#non-streaming-vs-streaming)
3. [Time to first token](#time-to-first-token)
4. [Tokens ≠ chunks](#tokens--chunks)
5. [How streaming works over HTTP: SSE and WebSockets](#how-streaming-works-over-http-sse-and-websockets)
6. [Streaming must survive the whole pipeline](#streaming-must-survive-the-whole-pipeline)
7. [Backend: `.call()` → `.stream()` and `Flux<String>`](#backend-call--stream-and-fluxstring)
8. [Streaming + conversation history](#streaming--conversation-history)
9. [Frontend: reading the stream](#frontend-reading-the-stream)
10. [End-to-end mental model](#end-to-end-mental-model)

---

## Generation vs delivery

[0:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=0s)

From lecture 2: an LLM generates one token at a time.

```
The capital of India is         → New
The capital of India is New     → Delhi
The capital of India is New Delhi → .
```

How does it know when to stop? During training, text is deliberately marked with end-of-sequence markers, so the model learns how human text naturally ends and eventually predicts "end".

Two separate things happen in an AI app:

| | What it is | Example |
|---|---|---|
| **Generation** | What the model does | `Spring` → `Boot` → `is` → `a` → `framework` |
| **Delivery** | How output travels through *your* app | AI provider → backend → browser |

Streaming changes **delivery**. It doesn't change how the model generates. The ChatGPT website streams, but when your own backend calls the API, the default is non-streaming: you get one complete string.

---

## Non-streaming vs streaming

**Non-streaming:**

```
USER → "Explain Spring Boot" → BACKEND → LLM
       (generates Spring… Boot… is… a… framework… — backend waits)
     ← complete response ← BACKEND
```

If generation takes 8 seconds, the user stares at a spinner for 8 seconds, then everything appears at once.

**Streaming:**

```
LLM ├─ "Spring"     → Backend → Browser
    ├─ " Boot"      → Backend → Browser
    ├─ " is"        → Backend → Browser
    └─ " framework" → Backend → Browser
```

The user sees `Spring` → `Spring Boot` → `Spring Boot is` → … The full answer may still take 8 seconds, but useful text appears after about 1.

> Streaming doesn't make the model finish faster. It makes the app *feel* faster because output arrives before generation is done.

---

## Time to first token

[10:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=600s)

```
0 s  user clicks Send
1 s  first output arrives      ← Time to First Token (TTFT)
2 s  more…
…
8 s  complete answer           ← Total generation time
```

```
Without streaming: |-------------- 8 s --------------| ANSWER
With streaming:    |--1 s--| Spring Boot makes building … COMPLETE at 8 s
```

Total time stays about the same whether you stream or not. **Perceived latency** drops a lot, which is why every chat UI streams.

### Why the first token takes a moment

Before generating anything, the model has to process the whole input:

```
System prompt + history + user message
        ↓
   process the context      ← bigger input = longer wait for first token
        ↓
Generate output 1 → output 2 → output 3 …
```

So: **input processing → output generation**. Streaming helps once output generation starts. Each later token also takes a slightly different time, depending on how quickly the comparison against the vocabulary resolves.

---

## Tokens ≠ chunks

One network chunk is not necessarily one LLM token. Providers don't send every token the instant it's made. Roughly, they send "whatever tokens I have in this small time window".

```
Model generates:   T1 → T2 → T3 → T4 → T5 → T6
Provider sends:    [T1 T2] [T3 T4] [T5 T6]
```

A chunk might be `"Spring"`, `"Spring Boot"` or `"Spring Boot makes"`. The grouping depends on the provider, framework, network layer, buffering and protocol. And it can re-group again at every hop:

```
LLM token ≠ API provider chunk ≠ HTTP/network chunk ≠ reader.read() result
```

For app code, the right mental model is just: **small piece, small piece, small piece…** Your UI should never depend on chunk boundaries.

---

## How streaming works over HTTP: SSE and WebSockets

[16:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=960s)

A normal API: `request → server works → server finishes → one response` (e.g. `GET /users/10` → `{"id":10,"name":"Rohit"}`).

AI responses develop over time, so we want:

```
REQUEST → connection stays open → data → data → data → data → DONE
```

**Streaming is still HTTP.** The server can start the response without building the whole body first:

```
HTTP response begins
  chunk
  chunk
  chunk
HTTP response ends
```

### SSE vs WebSockets

| | **SSE (Server-Sent Events)** | **WebSockets** |
|---|---|---|
| Direction | Server → client | Client ↔ server (both ways) |
| Protocol | Plain HTTP, long-lived response with multiple events | Starts as HTTP, then upgrades to its own protocol |
| Format | `data: Spring` / `data: Boot` / `data: is` … | Arbitrary messages either way |
| Good for | Streaming an AI answer | Highly interactive real-time apps (games, collaboration) |

After the user sends the prompt, data flows **one way**: server → client, chunk after chunk. ChatGPT doesn't message you first. So WebSockets are usually overkill for answer streaming. Most model providers use **SSE** to stream to your backend.

---

## Streaming must survive the whole pipeline

[22:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=1320s)

There are really **two streams** in your app:

```
AI provider → Backend      (stream 1)
Backend     → Frontend     (stream 2)
```

```
Browser ──question──► Spring Boot ──AI request──► AI provider → LLM
LLM generates → provider streams chunks → Spring AI receives chunks
  → Spring Boot forwards chunks → browser receives chunks → UI appends chunks
```

If the backend **collects** all chunks into one string and then sends it, the user sees no streaming. If the backend streams perfectly but the **JavaScript** waits for the full body, streaming disappears again.

```
LLM → AI provider → Spring AI → Controller → HTTP response → JavaScript → DOM
```

> Streaming is an end-to-end property of the application, not just an LLM feature.

---

## Backend: `.call()` → `.stream()` and `Flux<String>`

[25:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=1500s)

Non-streaming:

```java
public String chat(String message) {
    return chatClient.prompt().user(message).call().content();   // one String
}
```

Streaming:

```java
public Flux<String> chat(String message) {
    return chatClient.prompt().user(message).stream().content(); // many Strings over time
}
```

The interesting change isn't `.call()` → `.stream()`. It's the **return type**: `String` → `Flux<String>`. A stream can't be stored in a single `String`, so the service *and* the controller return types have to change.

### What is `Flux<String>`?

- `String` = "eventually give me **one** string". A box: `[Hello Rohit, how can I help?]`
- `Flux<String>` = "give me **zero, one or many** strings **over time**". A conveyor belt: `→["Hello"] →[" Rohit"] →[", how"] →[" can"] →[" I help?"]`

Same idea, different names per language:

| Language | Abstraction |
|---|---|
| Java / Reactor | `Flux` |
| JavaScript | `ReadableStream` |
| Python | `AsyncIterator` (async generator) |

### Controller

```java
@PostMapping(value = "/api/chat",
             consumes = MediaType.TEXT_PLAIN_VALUE,
             produces = MediaType.TEXT_PLAIN_VALUE)
public Flux<String> chat(@RequestBody String message) {
    return chatService.chat(message);
}
```

Returning `Flux<String>` lets Spring write each piece into the HTTP response as it's emitted.

**`text/event-stream`?** If you want real SSE, use `produces = MediaType.TEXT_EVENT_STREAM_VALUE`. But streaming doesn't *require* SSE. Raw `text/plain` streaming is enough for the Tomato bot.

> Streaming = data arrives progressively. SSE = one specific protocol/format for delivering events.

In the video, the bot's system prompt is changed to *"You are a funny AI chatbot. You reply everything sarcastically."*, and Postman shows the sarcastic answer arriving piece by piece.

---

## Streaming + conversation history

[31:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=1860s)

Before, we did `history.add(new AssistantMessage(response))` with a complete string. Now the response is a stream of chunks like `"My"`, `" name"`, `" is"`, `" ChatGPT"`.

**Don't store each chunk as its own message.** Those are transport pieces. Logically the model wrote *one* assistant message: `"My name is ChatGPT"`.

Each chunk should do two jobs:

```
      chunk
   ┌────┴─────┐
   ↓          ↓
 send to    append to
 browser    buffer
```

When the stream finishes: buffer → `"Java is a language"` → store **one** `AssistantMessage`.

```java
public Flux<String> chat(String userMessage) {
    history.add(new UserMessage(userMessage));
    StringBuilder fullResponse = new StringBuilder();

    return chatClient.prompt()
            .system(SYSTEM_PROMPT)
            .messages(history)
            .stream()
            .content()
            .doOnNext(fullResponse::append)          // every chunk → buffer
            .doOnComplete(() ->                      // stream done → one message
                    history.add(new AssistantMessage(fullResponse.toString())));
}
```

- `.doOnNext(...)` — whenever a chunk arrives, also append it to the buffer. The chunk keeps flowing to the controller and browser.
- `.doOnComplete(...)` — when the stream ends successfully, save the full text as one assistant message.

The same idea in Python with the OpenAI SDK:

```python
response = await client.responses.create(
    model="gpt-4o-mini", instructions=SYSTEM_PROMPT, input=history, stream=True
)
full = ""
async for event in response:
    if event.type == "response.output_text.delta":
        full += event.delta
        yield event.delta          # stream to caller
history.append({"role": "assistant", "content": full})
```

**Production note:** a shared `ArrayList` of history is fine for teaching but wrong for real multi-user apps. You need per-conversation isolation and thread-safe state.

---

## Frontend: reading the stream

[34:00](https://www.youtube.com/watch?v=r4x3Zm4WIWA&t=2040s)

The backend now streams, but the old frontend still shows everything at once. The culprit is one line:

```js
const assistantReply = await response.text();
```

That says: *read the entire body, combine it, and give it to me when it's finished.*

```
LLM ✅ → Spring Boot ✅ → HTTP ✅ → Browser receiving ✅ → response.text() ❌ waits → UI
```

### What does `await fetch()` wait for?

Not the full body. It resolves once **headers** arrive:

```
request sent → server starts response → headers arrive → fetch() resolves
                                                         (body may still be arriving…)
```

`response.text()` is what waits for the full body. For a live UI, read the body while it arrives.

### `response.body` is a `ReadableStream`

```js
const reader = response.body.getReader();

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  // process value
}
```

Each `read()` gives `{ value, done: false }` until the end, then `{ value: undefined, done: true }`.

### Why isn't `value` a string?

HTTP moves **bytes**. `"Spring"` → UTF-8 → `[83, 112, 114, 105, 110, 103]`. So `value` is a `Uint8Array`. Decode it:

```js
const decoder = new TextDecoder();
const chunk = decoder.decode(value, { stream: true });
```

`{ stream: true }` matters. A multi-byte character (emoji, Hindi text) can be split across two network chunks, and this keeps the partial bytes until the rest arrives. After the loop, call `decoder.decode()` once more to flush anything left.

### One bubble, not one bubble per chunk

Chunks `"Spring"`, `" Boot"`, `" is"`, `" great"` are four pieces of **one** message. Create the assistant bubble once and keep appending to it:

```js
const assistantRow = addMessage("assistant", "");
const assistantBubble = assistantRow.querySelector(".message");
// for every chunk:
assistantBubble.textContent += chunk;
```

### The complete `sendMessage()`

```js
async function sendMessage() {
  const message = messageInput.value.trim();
  if (!message) return;

  addMessage("user", message);
  messageInput.value = "";
  sendButton.disabled = true;
  showTyping();

  let assistantRow = null;
  let assistantBubble = null;
  let hasStartedStreaming = false;

  try {
    const response = await fetch(API_URL, {
      method: "POST",
      headers: { "Content-Type": "text/plain" },
      body: message,
    });
    if (!response.ok) throw new Error("Request failed");

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value, { stream: true });

      if (!hasStartedStreaming) {               // first chunk: swap typing dots for a bubble
        hideTyping();
        assistantRow = addMessage("assistant", "");
        assistantBubble = assistantRow.querySelector(".message");
        hasStartedStreaming = true;
      }

      assistantBubble.textContent += chunk;
      scrollToBottom();
    }

    const remaining = decoder.decode();         // flush leftover bytes
    if (remaining && assistantBubble) assistantBubble.textContent += remaining;

  } catch (error) {
    hideTyping();
    if (assistantRow) assistantRow.remove();
    addMessage("assistant", "Sorry, I couldn't connect to Tomato Support. Please try again.", "error");
  } finally {
    hideTyping();
    sendButton.disabled = false;
  }
}
```

Flow: submit → show typing → send request → headers arrive → get `response.body` → reader → **wait for first chunk** → hide typing → create ONE bubble → decode → append → read next → … → done.

**Why `scrollToBottom()` after every chunk?** The bubble grows taller, and new text would otherwise scroll out of view. A production app should stop auto-scrolling if the user scrolls up to read.

---

## End-to-end mental model

```
USER "Explain dependency injection"
 → BROWSER ──HTTP request──► SPRING BOOT ──AI request──► AI PROVIDER → LLM
 LLM generates incrementally
 → AI PROVIDER streams chunks (SSE)
 → SPRING AI: Flux<String>
 → CONTROLLER: streaming HTTP response
 → BROWSER: ReadableStream → reader → Uint8Array → TextDecoder → string chunks
 → ONE assistant bubble: append… append… append…
 → COMPLETE RESPONSE

Meanwhile in the backend:
 incoming chunk ─┬─► browser
                 └─► StringBuilder → stream complete → AssistantMessage → history
```

```
Generation → provider streaming → backend streaming → HTTP streaming → browser streaming → live UI
```

## Key takeaways

1. LLMs generate token by token. Streaming is about **delivering** those pieces as they come.
2. Streaming improves **time to first token** / perceived latency, not total generation time.
3. Tokens, provider chunks, network chunks and `reader.read()` results are all different. Never rely on chunk boundaries.
4. Streaming is still HTTP: a response whose body arrives progressively.
5. SSE = one-way server→client events (what providers use). WebSockets = two-way, usually unnecessary for answer streaming.
6. Streaming must be preserved at **every** hop. One buffering step or one `response.text()` kills it.
7. Spring AI: `.stream().content()` returns `Flux<String>`. Change the service and controller return types.
8. Stream to the client **and** buffer at the same time. Save one assistant message on completion.
9. Browser: `response.body.getReader()` + `TextDecoder({stream:true})`, appending to one growing bubble.

## Interview questions

**Q: Does streaming make the LLM faster?**
No. Total generation time is about the same. Streaming cuts time-to-first-visible-output, so the app feels faster.

**Q: Why might a streaming backend still show the full answer at once in the browser?**
The frontend calls `await response.text()` (or `.json()`), which waits for the whole body. Read `response.body` with a reader instead. Or an intermediate layer (proxy, backend code) is buffering.

**Q: SSE or WebSockets for a chatbot?**
SSE (or plain chunked HTTP). After the prompt, data only flows server → client. WebSockets add two-way complexity you don't need.

**Q: How do you keep conversation history correct when streaming?**
Forward each chunk to the client and also append it to a buffer. When the stream completes, save the buffer as a single assistant message.

**Q: Why `TextDecoder.decode(value, { stream: true })`?**
Chunks are raw bytes, and a multi-byte UTF-8 character can be split across chunks. Streaming mode keeps partial bytes until the rest arrives.
