# 05 — Serialization and Deserialization

**Video:** [#7 Serialization and Deserialization for backend engineers](https://www.youtube.com/watch?v=vzg90tY3uM0) (22m)

A client written in JavaScript and a server written in Rust have completely different data types. How does data cross the network and still make sense on the other side? This lecture answers that, then looks at JSON.

---

## Contents

1. [The problem](#the-problem)
2. [The solution: a common format](#the-solution-a-common-format)
3. [Which formats exist](#which-formats-exist)
4. [JSON](#json)
5. [Seeing it in a real request](#seeing-it-in-a-real-request)
6. [What you should and shouldn't worry about](#what-you-should-and-shouldnt-worry-about)
7. [Summary](#summary)
8. [Interview questions](#interview-questions)

---

## The problem

[0:00](https://www.youtube.com/watch?v=vzg90tY3uM0&t=0s)

```
Client (JavaScript app)  ── HTTP / gRPC / WebSocket ──►  Server (Rust, Go, Java, ...)
```

A request is: URL, headers, **body**. The body is the data the client wants to send. The client can be React, Angular, Vue... and the server can be written in anything.

Say the client has a JavaScript object `{ name: "Aditya", ... }` and the server is Rust.

| JavaScript | Rust |
|---|---|
| Dynamic, interpreted | Strictly typed, compiled |
| Objects, arrays, loose types | Structs, enums, exact types |

The two have **completely different type systems**. How does a JS object on one machine become something a Rust program on another machine understands?

Underneath, the data goes through the OSI model: application layer → ... → physical layer (bits as electrical, optical or radio signals) and back up on the other side. You don't need to master that. What matters: **the two ends are in different languages and can't share memory.**

---

## The solution: a common format

[6:08](https://www.youtube.com/watch?v=vzg90tY3uM0&t=368s)

If you had to design this, the obvious answer is a **shared standard**: a set of rules both sides agree on for how data is written on the wire.

```
JS object ──serialize──► [common format] ──network──► [common format] ──deserialize──► Rust struct
Rust struct ──serialize──► [common format] ──network──► [common format] ──deserialize──► JS object
```

| Term | Meaning |
|---|---|
| **Serialization** | Native in-memory data (object, struct, dict) → the common format |
| **Deserialization** | Common format → native in-memory data |

> **Serialization / deserialization = converting data to and from a common format**, so it can travel (or be stored) and be understood by any language on either end.

It makes data **language-agnostic** (more generally, domain-agnostic): machines, languages and environments that share nothing can still understand each other.

> The lecture mentions *storage* too. The same idea is used to save data in files or databases. This video focuses on transmission.

---

## Which formats exist

[9:11](https://www.youtube.com/watch?v=vzg90tY3uM0&t=551s)

There are hundreds of technologies in backend. You can't learn them all, so the series picks the **most used** one in each area and treats the rest as learn-as-you-go:

| Area | Many options | The series picks |
|---|---|---|
| Transport | HTTP, WebSockets, gRPC | **HTTP** (REST is still the most common) |
| Database | Postgres, MySQL, SQLite / MongoDB, DynamoDB | **Postgres** |
| Serialization | see below | **JSON** (~80% of client-server traffic, the speaker's estimate) |

Serialization formats fall into two families:

| Family | Formats | Notes |
|---|---|---|
| **Text-based** | **JSON**, YAML, XML | Human-readable. |
| **Binary** | **Protobuf** (most popular), Avro, others | Not human-readable. Compact and fast. |

YAML and XML are also used (config files, older APIs), but not usually for HTTP APIs today.

---

## JSON

[12:44](https://www.youtube.com/watch?v=vzg90tY3uM0&t=764s)

**JSON** = **J**ava**S**cript **O**bject **N**otation. It looks like a JS object but isn't limited to JavaScript. It's used in:

- HTTP request and response bodies (our focus)
- configuration files
- log files (a very common choice for application and server logs)

Its biggest strength: **it is completely human-readable**.

```json
{
  "id": 1,
  "title": "Atomic Habits",
  "author": "James Clear",
  "inStock": true,
  "tags": ["self-help", "habits"],
  "publisher": {
    "name": "Penguin",
    "country": "India"
  }
}
```

### The rules

| Rule | Detail |
|---|---|
| Wrapped in `{ }` | An object starts with `{` and ends with `}` |
| **Keys are strings in double quotes** | `"title"` is valid. A key can never be a number or unquoted |
| **Values** can be | a string, a number, a boolean, an array, or a **nested object** |
| Nesting | A nested object follows the same rules recursively |

That's the whole format. It's popular because it is simple, readable and every language has a parser for it.

---

## Seeing it in a real request

[16:20](https://www.youtube.com/watch?v=vzg90tY3uM0&t=980s)

In the demo, a client creates a book:

```
POST /api/books
Content-Type: application/json

{ "id": 3, "title": "...", "author": "..." }
```

The server reads the JSON, adds the book, and responds with another JSON:

```json
{ "data": [ { "id": 1, ... }, { "id": 2, ... }, { "id": 3, ... } ] }
```

The client parses that JSON into its own data structures and renders it.

### What happens between JSON and the wire

```
Client app → JSON text → HTTP → TCP segments → IP packets → frames → bits (signals)
                                                                         │
Server app ← JSON text ← HTTP ← TCP segments ← IP packets ← frames ←──────┘
```

At each layer the data is wrapped in a different form, but by the time it reaches the server's application layer, it is **the same JSON text again**.

---

## What you should and shouldn't worry about

[17:51](https://www.youtube.com/watch?v=vzg90tY3uM0&t=1071s)

The mental model for a backend engineer:

- The client **adheres to a standard** (JSON).
- The server **understands that standard** and replies in the same one.
- Everything between (frames, packets, bits) is the network's problem. **Your responsibility ends at the format.**

Serialization is not a deep topic by itself. It's a phenomenon in backend work you need to know exists: JSON goes out, the server parses it and makes sense of it, and replies with JSON the client can parse.

> Later topics that the roadmap flags around serialization: missing/extra fields, nulls, date and timezone issues, custom serializers, validating JSON before processing (see the [validation notes](07-validation-and-transformation.md)), and JSON vs Protobuf performance.

---

## Summary

1. Client and server run in **different languages** with **different type systems**. They can't share memory.
2. They agree on a **common format** for data on the wire.
3. **Serialize** = native data → common format. **Deserialize** = common format → native data.
4. Formats are **text-based** (JSON, YAML, XML) or **binary** (Protobuf, Avro).
5. **JSON** is by far the most common for HTTP APIs: human-readable, string keys in double quotes, values are string / number / boolean / array / nested object.
6. Everything below the application layer (packets, bits) is invisible to you. The server always sees the same JSON the client wrote.

## Interview questions

**Q: What is serialization and why do we need it?**
Converting in-memory data into a common transmittable or storable format (e.g. JSON). Different languages and machines don't share data types or memory, so they need a shared standard to exchange data.

**Q: Serialization vs deserialization?**
Serialization: native object/struct → bytes or text in a format. Deserialization: that format → native object/struct.

**Q: Text-based vs binary serialization formats?**
Text-based (JSON, XML, YAML) is human-readable and easy to debug. Binary (Protobuf, Avro) is not human-readable but is more compact and faster to process.

**Q: What are the rules of valid JSON?**
Objects in `{}`; keys are strings in double quotes; values are strings, numbers, booleans, arrays, objects (or null); nested objects follow the same rules.

**Q: A Go server and a JS client communicate. Who needs to know about the other's types?**
Neither. They only need to agree on the wire format (JSON schema/contract). Each side parses into its own native types.
