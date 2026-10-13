# 26 — OpenAPI: The Universal Contract Between Clients and Servers

**Video:** [#28 OpenAPI: The universal contract between clients and servers](https://www.youtube.com/watch?v=CwKsU84jIWs) (44m)

Inside one code base, the compiler protects you when a function signature changes. Across a **network boundary** (HTTP) nothing does. OpenAPI gives remote calls a **signature**: one machine-readable file describing every endpoint, which clients, servers, mocks, tests and docs can all be generated from.

---

## Contents

1. [The problem: three copies of the truth](#the-problem-three-copies-of-the-truth)
2. [What OpenAPI is](#what-openapi-is)
3. [Vocabulary: OAS, OAD, OAI, Swagger](#vocabulary-oas-oad-oai-swagger)
4. [History](#history)
5. [Anatomy of an OpenAPI file](#anatomy-of-an-openapi-file)
6. [Code-first vs design-first](#code-first-vs-design-first)
7. [The design-first pipeline](#the-design-first-pipeline)
8. [Summary](#summary)
9. [Interview questions](#interview-questions)

---

## The problem: three copies of the truth

[0:00](https://www.youtube.com/watch?v=CwKsU84jIWs&t=0s)

The task board's `POST /api/move` takes a JSON body `{ taskId, to, actor }` and returns `200`, `400`, `404` or `502`. Where does this knowledge live?

1. **The backend handler** (it reads the body).
2. **The frontend** `fetch` call (it sends the body).
3. **A doc**, if anyone wrote one.

Three copies must stay in sync. A teammate renames `to` → `column`. They update the handler and frontend, **forget the doc**. Weeks later a new engineer reads the doc (the "source of truth"), sends `to`, and gets a validation error.

**Why this only happens across a network boundary:** within your code base, changing a function's signature fails at **compile time** everywhere it's called. A remote HTTP call has **no signature**, so mismatches show up only at **runtime**.

The idea: give the remote call a signature. Freeze it: routes, query/path params, request body fields and their types, and every response shape, in a **programmatic** document (not Markdown) that can be enforced.

---

## What OpenAPI is

[5:37](https://www.youtube.com/watch?v=CwKsU84jIWs&t=337s)

An **OpenAPI description** is a **JSON or YAML file** that lists every endpoint and, for each, its inputs, outputs and behaviour.

**Motivation** (Tony Tam, the creator, at the startup Wordnik, 2010): clients wanted SDKs in Python, PHP, JS, plus docs for each. His point: *"we often code faster than we can document."* So the server describes itself in a file, and software reads it.

### What you get from one file

| Use | How |
|---|---|
| **Documentation** | Read-only or **interactive** (Swagger UI, Redoc, **Scalar**): try the API with a sample body and token |
| **Client libraries / types** | Generated in any language: no hand-written fetch code |
| **Server interfaces/boilerplate** | Generated, so your handler must satisfy them |
| **Request validation** | Libraries validate query, path and body automatically from the spec |
| **Mocks and tests** | Fake servers, contract tests |

### The spec's own definition

> Language-agnostic · for **humans and computers** · **without** source code, documentation or traffic inspection.

- *Language-agnostic*: Go, Node, Python, Java, PHP all fine.
- *Humans and computers*: the same file renders docs for people and drives code generation for tools.
- *Without source code / docs / network tab*: you don't need to read the implementation, a stale doc, or the browser Network panel to learn the API.

---

## Vocabulary: OAS, OAD, OAI, Swagger

[12:48](https://www.youtube.com/watch?v=CwKsU84jIWs&t=768s)

| Term | Meaning |
|---|---|
| **OAS**, OpenAPI **Specification** | The set of **rules** (which fields exist and what they mean) |
| **OAD**, OpenAPI **Description** | **Your file** (`openapi.json` / `openapi.yaml`) that must obey the OAS |
| **OAI**, OpenAPI **Initiative** | The organisation (under the **Linux Foundation**) that publishes and maintains the spec |
| **Swagger** | The *old name* of the specification, and today the name of a **tool** (Swagger UI) that renders the file. "Swagger file" = the OAD. "Swagger UI" = the interactive page |

JSON and YAML are interchangeable. Most people write YAML because **JSON has no comments**.

---

## History

[14:54](https://www.youtube.com/watch?v=CwKsU84jIWs&t=894s)

| When | What |
|---|---|
| 2010 | Wordnik builds a JSON description of its API + a page to render it; a designer names it **Swagger** |
| ~2011 | Swagger 1.0 open-sourced |
| 2014 | **Swagger 2.0**, adoption grows |
| 2015 | **SmartBear** acquires Swagger and donates the spec to the **Linux Foundation** (founding members include Google, Microsoft, IBM, PayPal) |
| Jan 2016 | Renamed **OpenAPI** with no content change (**OpenAPI 2.0 = Swagger 2.0**) |
| Now | Latest is **3.2.0**, but the ecosystem mostly supports **3.1**, so people write 3.1 |

---

## Anatomy of an OpenAPI file

[16:26](https://www.youtube.com/watch?v=CwKsU84jIWs&t=986s)

### Minimum valid file

```yaml
openapi: 3.1.1          # version of the SPEC the file follows
info:
  title: Task Board API
  version: 1.0.0        # version of YOUR API (different thing!)
paths: {}
```

Required at the root: `openapi` and `info`, plus at least one of `paths`, `components` or `webhooks`. Optional: `servers`, `security`, `tags`.

> Two different versions: `openapi` = the spec version; `info.version` = your API's own version.

### Servers

```yaml
servers:
  - url: http://localhost:8080      # a list: local, staging, production
```

Because the base URL is declared once, each path doesn't repeat it.

### Paths and operations

`paths` is a map from **path** (must start with `/`) to a **path item**, which has one entry per HTTP method (`get`, `post`, `put`, `patch`, `delete`, `head`, `options`). Dynamic segments use **curly braces**: `/tasks/{taskId}`.

Each method is an **operation**:

| Field | Purpose |
|---|---|
| `summary` | One-liner |
| `description` | Longer, supports Markdown |
| `tags` | Group operations in docs |
| `operationId` | **Unique** name. Write it like a **function name** (`getBoard`): code generators turn it into an actual function |
| `parameters` | Inputs in path/query/header/cookie |
| `requestBody` | Payload for POST/PUT/PATCH |
| `responses` | Outputs per status code |

### Parameters

A parameter has `name`, **`in`** (`query`, `path`, `header`, `cookie`), `required`, and a `schema`. **Path parameters are always required** (otherwise the URL makes no sense).

Example: a polling endpoint takes `since` (query, integer: last sequence number received), and the SSE endpoint can take `Last-Event-ID` as a header.

### Request body

```yaml
requestBody:
  required: true
  content:
    application/json:           # the media type (Content-Type)
      schema:
        type: object
        required: [taskId, to]
        properties:
          taskId: { type: string }
          to:     { type: string }
          actor:  { type: string, default: guest }   # optional, defaults to "guest"
```

`content` maps a **media type** to a schema: `application/json`, `multipart/form-data` (file uploads), `application/x-www-form-urlencoded` (HTML forms).

### Responses

Keys are status codes **in quotes**. Each has a `description` and optional `content`.

```yaml
responses:
  "200": { description: Moved,       content: { application/json: { schema: { $ref: '#/components/schemas/Event' } } } }
  "400": { description: Bad request, content: { application/json: { schema: { $ref: '#/components/schemas/Error' } } } }
  "404": { description: No such task }
  "502": { description: Moved, but publishing to the bus failed }
```

### Schema object

Describes the shape of data:

| `type` | Extra constraints |
|---|---|
| `string` | `minLength`, `maxLength`, `pattern`, `enum` (allowed values) |
| `number` / `integer` | `minimum`, `maximum` |
| `boolean`, `null` | |
| `array` | `items` (a schema per element) |
| `object` | `properties` (key → schema), `required` |

Schemas **compose**, so you can describe almost any structure. Reusable schemas, parameters and security schemes live under `components`. There's also a section for **security** (what auth the client uses).

---

## Code-first vs design-first

[29:13](https://www.youtube.com/watch?v=CwKsU84jIWs&t=1753s)

| | Code-first | Design-first (contract-first) |
|---|---|---|
| Order | Write handlers/services first. The spec is **derived** (annotations, comments, types) | Write the **spec first**, then implement against it |
| Spec is | A by-product of the code | The **source of truth** |
| How | Libraries per ecosystem. **FastAPI** generates `openapi.json` from handler code/annotations. Also **ts-rest** + Zod (see below) | Hand-written (or generated from code) YAML/JSON, then tooling |

The Swagger team started code-first (they already had a server) but moved to **contract-first**. The speaker also began with code-first on existing servers but now **starts new projects contract-first**:

- Frontend and backend can **start in parallel** since they've agreed the contract.
- In the agentic-coding era: agree the contract with the AI, then run two agents (server side and client side) that don't block each other.

### Code-first in practice (ts-rest example)

In the author's `tasker` app: **Zod schemas** define the shapes → a **ts-rest contract** attaches them to paths/methods → a generator function emits `openapi.json` → the Go server serves it for the docs UI. You never hand-write the JSON. You only declare the **endpoint, inputs (query/body/path/headers), and responses (schema + status code)**. This is more robust than comment-based generation because it's real code.

---

## The design-first pipeline

[34:53](https://www.youtube.com/watch?v=CwKsU84jIWs&t=2093s)

Demonstrated on the task board (which has no OpenAPI yet). **One file feeds every stage:**

```
                         openapi.yaml  (single source of truth)
                               │
   ┌──────────┬────────────────┼────────────────┬───────────────┐
   ▼          ▼                ▼                ▼               ▼
 LINT       MOCK            CLIENT           SERVER           DOCS
Spectral    Prism     openapi-typescript   oapi-codegen   Swagger UI/Redoc/Scalar
                      + openapi-fetch      (Go interfaces)
                               │
                               ▼
                    TEST the real server against the spec: Schemathesis
```

| Stage | Tool | What it does |
|---|---|---|
| 1. **Lint** | **Spectral** | Checks the spec against OpenAPI rules and style (demo: 3 errors: no `info.contact`, 2 operations without descriptions. Fix → passes) |
| 2. **Mock** | **Prism** | Spins up a fake server from the spec, returning your documented **examples**. Frontend can build against it **before the backend exists** |
| 3. **Client** | **openapi-typescript** + **openapi-fetch** | Generates TypeScript types for every path/field. A misspelled field or wrong type becomes a **compile error** |
| 4. **Server** | **oapi-codegen** (Go) | Generates **interfaces** your handlers must implement. Change the spec and regenerate → **the build fails at the handler** until it matches (the compile-time guarantee, restored across the network) |
| 5. **Docs** | **Swagger UI**, **Redoc**, **Scalar** (the speaker's pick: modern, clean) | Render the same file as interactive docs |
| 6. **Verify** | **Schemathesis** | Generates valid *and* invalid requests from the schemas, sends them to the **real server**, and checks that every response conforms to the spec |

The industry has standardised on this: **GitHub** (~344,000 lines) and **Stripe** (~171,000 lines) publish their OpenAPI files, which anyone can use to build clients. Also useful as the description given to **AI agents** that call tools.

---

## Summary

1. Inside a code base, the compiler enforces signatures. **Across HTTP nothing does**, so contracts drift among handler, client and docs.
2. **OpenAPI** = a machine-readable (JSON/YAML) description of your API: paths, parameters, request bodies, responses, schemas, security.
3. **OAS** (rules), **OAD** (your file), **OAI** (the org), **Swagger** (old name / the UI tool).
4. Minimum file: `openapi` + `info` + one of `paths`/`components`/`webhooks`. Two different versions: spec version vs API version.
5. **`operationId`** becomes a function name in generated code. **Path parameters are always required.**
6. **Code-first** derives the spec from code (FastAPI, ts-rest + Zod). **Design-first** writes the spec first and generates everything else.
7. Prefer **contract-first for new projects**: parallel frontend/backend work, one source of truth.
8. Pipeline: **lint → mock → client types → server interfaces → docs → contract-test the server.**
9. Latest spec 3.2, but 3.1 is what tools support.

## Interview questions

**Q: What problem does OpenAPI solve?**
Remote calls lack a compile-time signature, so the handler, client and docs drift apart. OpenAPI gives one machine-readable contract from which they're all generated or checked.

**Q: OpenAPI vs Swagger?**
Swagger was the original name of the specification. Since 2016 the spec is OpenAPI. "Swagger" now mostly refers to tooling such as Swagger UI.

**Q: OAS vs OAD vs OAI?**
Specification (the rules), Description (your `openapi.yaml`), Initiative (the Linux Foundation org that maintains the spec).

**Q: What are the mandatory root fields of an OpenAPI document?**
`openapi` and `info`, plus at least one of `paths`, `components`, `webhooks`.

**Q: What's the difference between `openapi` and `info.version`?**
`openapi` is the spec version the file follows. `info.version` is your own API's version.

**Q: Why should `operationId` look like a function name?**
Generators turn it into a client function (`getBoard()`), and it must be unique across the document.

**Q: Where can parameters live?**
`path`, `query`, `header`, `cookie`, indicated by `in`. Path parameters must be `required`.

**Q: Code-first vs design-first?**
Code-first derives the spec from existing code. Design-first writes the spec first and treats it as the contract. Design-first allows parallel work and gives a single source of truth.

**Q: How does design-first restore compile-time safety?**
Generate server interfaces and client types from the spec. If the spec and handler/caller diverge, the build fails.

**Q: What does a mock server (Prism) enable?**
Frontend development against documented example responses before the backend is implemented.

**Q: How do you verify that a real server matches its spec?**
Contract-test it with a tool like Schemathesis, which generates requests from the schemas and validates the responses.

**Q: Why is YAML often preferred over JSON for the spec file?**
Equivalent features, but YAML allows comments.
