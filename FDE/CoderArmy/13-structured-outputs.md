# 13 — Structured Outputs: LLM Responses Your Code Can Use

**Video:** [FDE Full Course #13 — Structured Outputs in LLM](https://www.youtube.com/watch?v=yKsgSHJ3Jfo) (41m)
**Instructor notes + code:** [Lecture 13](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2013) (Spring, Python, JS)

The shortest lecture in the series, and the groundwork for MCP. So far LLM output went to a **human**. Now it goes to **code**, and code needs structure, not prose.

---

## Contents

1. [Plain text is fine for humans, useless for code](#plain-text-is-fine-for-humans-useless-for-code)
2. [What structured output is](#what-structured-output-is)
3. [Schemas: the form analogy](#schemas-the-form-analogy)
4. [Why not just say "return JSON"?](#why-not-just-say-return-json)
5. [Structured output vs tool calling](#structured-output-vs-tool-calling)
6. [Practical: a meeting scheduler](#practical-a-meeting-scheduler)
7. [Where MCP comes in](#where-mcp-comes-in)

---

## Plain text is fine for humans, useless for code

[0:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=0s)

```java
String response = chatClient.prompt().user("Explain Docker").call().content();
// "Docker is a platform that allows developers to package applications into containers..."
```

For a chatbot that's perfect, because the reader is a human, and humans are great at unstructured text. Read this:

> "Rahul wants a meeting tomorrow at 4 PM for 30 minutes to discuss Docker deployment."

You instantly see: person → Rahul, date → tomorrow, time → 4 PM, duration → 30 min, topic → Docker deployment.

To a Java app it's just one `String`. You can't do:

```java
meeting.getTime();
if (meeting.getDuration() > 60) { ... }
calendar.createEvent(meeting);
```

Think of a normal REST API: you call `GET /user/details` and expect `{"name": …, "email": …, "mobile": …}`, not *"The user's name is Rahul and his email is…"*. Machines never liked natural language. **We** did, which is why we built LLMs to speak it.

> **Humans prefer meaning. Software prefers structure.**

LLMs naturally produce language. Applications work with objects, arrays, numbers, booleans, enums, dates and IDs. We need a bridge:

```
Natural language → LLM → Structured data → Application logic
```

That bridge is **structured output**.

---

## What structured output is

[5:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=300s)

> Asking the model to produce information in a **predefined structure** that the application can reliably consume.

A REST API returns `{"name": "Rahul", "age": 28, "email": "rahul@gmail.com"}`, not a paragraph. That's a **contract** for the response. Apply the same idea to an LLM:

```
Without: Prompt → LLM → free-form text
With:    Prompt → LLM → expected schema → structured data
```

### Example: meeting scheduler

The old way: a form with *Meeting with*, *Time*, *Invite?*, and a Submit button. The new way: the user just types **"Schedule a meeting at 4 PM today with Ravi."** (ChatGPT's agent mode can already put things on your Google Calendar like this.)

The server can't read that string. Hand-written string matching ("4 PM" → 16:00, "today" → a date…) would explode into edge cases. So the LLM converts it into something the server *can* use:

```
MeetingRequest
  title           → String
  attendee        → String
  date            → String
  time            → String
  durationMinutes → Integer
  type            → MeetingType (enum)
```

```json
{
  "title": "Docker Deployment Discussion",
  "attendee": "Rahul",
  "date": "2026-10-02",
  "time": "16:00",
  "durationMinutes": 30,
  "type": "ONE_ON_ONE"
}
```

Now the app has `MeetingRequest meeting` and can call `meeting.attendee()`, `meeting.date()`, `meeting.time()`, `meeting.durationMinutes()`. The response isn't something to display anymore. **It's data that takes part in your logic.**

### It's not just "make it return JSON"

JSON is a common representation, but the real idea is **the output follows a structure your application understands**. In Java you ultimately want an object:

```java
public record MeetingRequest(String title, String attendee, String date,
                             String time, int durationMinutes) {}
```

```
LLM → structured representation (maybe JSON in between) → MeetingRequest object
```

---

## Schemas: the form analogy

[17:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=1020s)

A **schema** defines what the structured data must look like: `title → text`, `attendee → text`, `durationMinutes → number`…

Think of a form:

- **No schema:** "Tell me about yourself." Everyone writes something different: profession first, projects, experience, in any order. That's free-form output.
- **With a schema:** `Name: ____  Age: ____  Email: ____  Top 5 skills: 1.__ 2.__ …`. The fields are fixed. The person just fills the values.

> **We define the slots. The LLM fills the values.**

Once the response is a real object, normal programming works:

```java
if (meeting.type() == MeetingType.INTERVIEW) sendInterviewInstructions();
if (meeting.durationMinutes() > 60)          requireApproval();
googleCalendar.createEvent(meeting);
```

The LLM is no longer just a chatbot. It's one component inside a larger system.

---

## Why not just say "return JSON"?

[19:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=1140s)

```
Extract the meeting information. Return JSON in this format:
{ "attendee": "", "date": "", "time": "" }
```

For a demo it often works. But going from **demo → software**, LLMs are probabilistic. You might get:

```
Sure! Here's the JSON you requested:
{ "attendee": "Rahul", "date": "2026-10-02", "time": "16:00" }
```

A human ignores the first line. A JSON parser crashes on it. Or next time the model renames fields on its own: `meetingTime` becomes `timeOfMeeting`, `attendee` becomes `visitor`, or a field gets dropped. Your server needs the **same field names every time**.

| "Please return JSON" | Structured output |
|---|---|
| An **instruction**: "please behave like this" | A **contract**: "this is the structure the application expects" |
| LLM → *hopefully* valid JSON | LLM → defined schema → expected output → application object |

The second is how normal software is designed.

---

## Structured output vs tool calling

[14:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=840s)

Both involve structured data, so they look alike. They solve different problems.

**Structured output** is about the **format of the final result**:

```
User → LLM → MeetingRequest {...}
```

Nothing was executed. The object *is* the result we wanted.

**Tool calling** (lecture 5) means the model **decides to invoke a function**:

```
User "Weather in Delhi?" → LLM → decides to call weather tool → {"city": "Delhi"}
  → Weather API → tool result → LLM → final answer
```

The `{"city": "Delhi"}` is structured too, but it exists **so a tool can run**.

| | Structured output | Tool calling |
|---|---|---|
| The structured data is… | The final result | Arguments for an external function |
| Something executed? | No | Yes, the tool runs |

In the meeting example, structured output only turns *"Schedule a Docker discussion with Rahul tomorrow at 4 PM for 30 minutes"* into a `MeetingRequest`. **Nothing has been scheduled yet.** It's human language → application data.

---

## Practical: a meeting scheduler

[28:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=1680s)

### The schema class

```java
@Getter @Setter @AllArgsConstructor @NoArgsConstructor
public class MeetingDetails {
    private String  title;
    private String  attendee;
    private String  date;
    private String  time;
    private Integer durationMinutes;
}
```

### The service

```java
@Service
public class MeetingService {

    private final ChatClient chatClient;

    public MeetingService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public MeetingDetails schedule(String message) {
        String systemPrompt = """
                You extract meeting information from the user's request.

                Today's date is %s.

                Rules:
                - Convert relative dates like today and tomorrow into yyyy-MM-dd format.
                - Convert time into 24-hour HH:mm format.
                - If title is missing, create a simple title.
                - If duration is missing, use 30 minutes.
                - Do not invent attendee, date or time.
                - If information is missing, keep it blank.
                """.formatted(LocalDate.now());

        return chatClient.prompt()
                .system(systemPrompt)
                .user(message)
                .call()
                .entity(MeetingDetails.class);   // ← instead of .content()
    }
}
```

Things to notice:

- **`.entity(MeetingDetails.class)`** replaces `.content()`. Spring AI derives a schema from the class, tells the model to follow it, and converts the reply straight into a `MeetingDetails` object.
- **Pass today's date.** The model doesn't know what "today" or "tomorrow" means unless you tell it (no live clock, remember its knowledge cutoff).
- **Normalise formats** in the prompt: `yyyy-MM-dd`, 24-hour `HH:mm`.
- **Defaults vs blanks.** Missing title → make one up. Missing duration → 30. But *never invent* attendee, date or time. Leave them blank so the app can ask the user.

### The controller

```java
@RestController
@RequestMapping("/api")
public class MeetingController {

    @PostMapping("/schedule")
    public MeetingDetails schedule(@RequestBody String message) {
        return meetingService.schedule(message);
    }
}
```

`POST /api/schedule` with *"Schedule my meeting today at 4:00 PM with Ravi"* →

```json
{
  "title": "Meeting with Ravi",
  "attendee": "Ravi",
  "date": "2026-10-01",
  "time": "16:00",
  "durationMinutes": 30
}
```

4 PM became `16:00`, "today" became the actual date, duration defaulted to 30, and a title was generated.

### Same thing in Python (OpenAI SDK + Pydantic)

```python
from pydantic import BaseModel

class MeetingDetails(BaseModel):
    title: str
    attendee: str
    date: str
    time: str
    durationMinutes: int

response = client.responses.parse(
    model="gpt-4o-mini",
    input=[{"role": "system", "content": system_prompt},
           {"role": "user",   "content": message}],
    text_format=MeetingDetails,      # the schema
)
meeting = response.output_parsed      # a MeetingDetails instance
```

### Extension ideas from the video

- **Missing info → ask the user.** If the date or time is blank, don't schedule. Ask a follow-up. Keep conversation history, collect fields until complete, *then* build the object.
- **Two LLM roles.** One call extracts structured data. Another turns the result into a friendly confirmation for the user ("Your meeting with Ravi is set for 4 PM today").

---

## Where MCP comes in

[24:00](https://www.youtube.com/watch?v=yKsgSHJ3Jfo&t=1440s)

So far: `User → LLM → MeetingRequest`. Structured data exists, but **no external action has happened**. Next:

```
User → LLM → MeetingRequest → external calendar system → calendar event created
```

How does the app talk to Google Calendar, or any outside system, in a standard way? That's tools, external integrations, and **MCP** (lectures 14–15).

> Structured output tells the application **what the user wants**. The external integration is responsible for **actually doing it**.

## Final mental model

| | What happens |
|---|---|
| **Plain text** | LLM generates information for a human to read |
| **Structured output** | LLM converts information into a predictable structure software can use |
| **Tool calling / integration** | The app (or model) uses structured info to perform an external action |

```
"Meet Rahul tomorrow at 4 PM for 30 minutes"
  → LLM → MeetingRequest → app uses the data → calendar integration (later)
```

Structured output is the step from **AI that talks** to **AI that participates in software systems**.

## Key takeaways

1. Humans want meaning, software wants structure. LLMs produce language, so you need a bridge.
2. Structured output = the model fills a **predefined schema** that maps to your types.
3. It's a contract, not "please return JSON". Prompting for JSON breaks on preambles, renamed fields and missing fields.
4. A schema is a form: you define the slots, the LLM fills the values.
5. Spring AI: `.call().entity(MyClass.class)`. OpenAI Python: `responses.parse(..., text_format=Model)`.
6. Give the model what it can't know (today's date). Specify formats. Define defaults vs "leave blank, don't invent".
7. Structured output ≠ tool calling. Structured output is the result. Tool calling's structure is arguments for something that executes.
8. Structured output says *what* to do. Integrations and MCP actually *do* it.

## Interview questions

**Q: Why not just prompt the LLM to "return JSON"?**
It's only an instruction. The model may add prose, rename or omit fields, or use the wrong types, which breaks parsers. Structured output enforces a schema/contract and maps directly to typed objects.

**Q: Structured output vs function calling?**
Structured output shapes the model's final answer into a schema. Function calling produces arguments so the app can execute a function, then usually feeds the result back to the model.

**Q: How do you handle relative dates like "tomorrow" in extraction?**
Put the current date in the system prompt and specify an output format (e.g. ISO `yyyy-MM-dd`).

**Q: The user didn't give a time. What should the extractor do?**
Leave it blank rather than invent one. Then the application can ask a follow-up question before acting.
