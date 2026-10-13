# 08 — Handlers, Services, Repositories, Middleware and Request Context

**Video:** [#10 What are controllers, services, repositories, middlewares and request context?](https://www.youtube.com/watch?v=hyc-7w3pee8) (1h)

Three closely linked topics, covered together: the **layered structure** inside a backend (handler/controller → service → repository), **middleware**, and **request context**. Together they describe the *request lifecycle inside the server*.

None of this is a hard requirement. It's a **design pattern** that makes a code base more scalable, maintainable and easier to debug and extend.

---

## Contents

1. [The request lifecycle inside the server](#the-request-lifecycle-inside-the-server)
2. [Handler / controller](#handler--controller)
3. [Service layer](#service-layer)
4. [Repository layer](#repository-layer)
5. [Putting the layers together](#putting-the-layers-together)
6. [Middleware](#middleware)
7. [Common middleware examples](#common-middleware-examples)
8. [Middleware order](#middleware-order)
9. [Request context](#request-context)
10. [Summary](#summary)
11. [Interview questions](#interview-questions)

---

## The request lifecycle inside the server

[0:31](https://www.youtube.com/watch?v=hyc-7w3pee8&t=31s)

Earlier videos covered the lifecycle *outside* the server (HTTP, DNS, etc). This one is what happens inside.

```
Client ──HTTP──► OS forwards to the port your server listens on  (entry point)
                      │
                   Routing  →  picks the handler for method + path
                      │
                   Handler / Controller
                      │
                   Service
                      │
                   Repository  →  Database
```

A **handler** is a function mapped to a route. In every language/framework it gets (at least) a **request object** and a **response object**, supplied by the runtime. It reads from the request and writes the response.

---

## Handler / controller

[4:41](https://www.youtube.com/watch?v=hyc-7w3pee8&t=281s)

**Responsibility: everything HTTP-shaped.** Its steps:

1. **Extract data from the request**:
   - `GET` → query params and path params
   - `POST` / `PUT` / `PATCH` (and sometimes `DELETE`) → request body
2. **Deserialize ("bind")** the JSON into the language's native type: a struct in Go, a dict or class in Python, a struct in Rust. (In Node/Express, a body-parser middleware already did this upstream.) If binding fails → **`400 Bad Request`** and stop.
3. **Validate** everything coming from outside (path params, query params, body). See [07](07-validation-and-transformation.md).
4. **Transform**: cast types, **set defaults**. Example: `GET /books` with an optional `sort` (`name` | `date`). If the client sent none, the transformation step sets `sort = "date"`. Good API habit: **make query parameters optional**.
5. **Call the service**, passing clean data plus auth info (user ID, permissions).
6. **Choose the response**: map the service's result or error to a status code (`200/201/204`, `400`, `500`) and a body.

> The controller controls the data flow in and out. It deals with *data formats and HTTP*, not business logic.

---

## Service layer

[18:38](https://www.youtube.com/watch?v=hyc-7w3pee8&t=1118s)

**Responsibility: the actual business logic.**

- It's a plain function. From looking at a service method you should **not** be able to tell it's used by an HTTP API. It knows nothing about status codes, request/response objects, or validation of HTTP input.
- It takes data, processes it, returns data.
- It **orchestrates**: can call several repository methods, merge results, call external APIs, send emails/notifications.
- Some services don't touch a DB at all (e.g. "send an email" → `{ success: true }`).

---

## Repository layer

[20:40](https://www.youtube.com/watch?v=hyc-7w3pee8&t=1240s)

**Responsibility: talk to the database, only.**

- Takes the data it needs (what to insert, which filters, which sort) and builds/executes the query.
- Returns the result.
- **Single responsibility per method.** One repository method returns one kind of thing. Don't make `getBooks(id?)` return either one book or all books depending on an optional param. Write `getAllBooks(...)` and `getBookById(id)`.

The service layer is where results from different repository calls get combined. A **service may call many repositories; a repository method does one thing.**

---

## Putting the layers together

[24:17](https://www.youtube.com/watch?v=hyc-7w3pee8&t=1457s)

`GET /books?sort=date`:

```
Handler:    bind → validate → transform (sort default)  ──►  calls service
Service:    business logic                              ──►  calls repository
Repository: builds SELECT ... ORDER BY ...              ──►  DB
                  ▲ rows
Service:    shapes/merges result                        ◄───┘
                  ▲ data
Handler:    chooses status 200, serializes JSON, sends response
```

| Layer | Knows about | Doesn't know about |
|---|---|---|
| Handler | HTTP, formats, status codes | business rules, SQL |
| Service | business rules, orchestration | HTTP, SQL details |
| Repository | the database | HTTP, business rules |

---

## Middleware

[26:55](https://www.youtube.com/watch?v=hyc-7w3pee8&t=1615s)

Redraw the lifecycle with extra functions in the boundaries between stages:

```
request → [mw] → [mw] → routing → [mw] → [mw] → Handler → Service → Repository
                                                      │
response ◄ [mw error handler] ◄───────────────────────┘
```

These are **middleware**: functions that run **in the middle** of the request lifecycle (before the handler, or after).

A middleware receives **three** things:

| Argument | Purpose |
|---|---|
| `request` | read and modify the request |
| `response` | modify the response, or **send a response early** |
| **`next`** | a function that passes execution to the next processing step |

- A middleware can **read/modify the request**, **modify the response**, **terminate the request** by sending a response itself, or **call `next()`** to continue.
- Middlewares are **optional**. A route can go straight to its handler.

### Why middleware exists

The same reason functions exist: **avoid duplication**. Thousands of requests and hundreds of endpoints all need the same cross-cutting work (security, logging, auth, parsing). Without middleware, every handler repeats it, or at least calls a shared function. Middleware moves that work out of the handlers so it applies uniformly. And if it fails (e.g. unauthenticated), it can **reject the request before it reaches the handler**, saving server resources.

> A good test: if code must run for (almost) **every request** and needs to touch the request/response, it's a middleware.

---

## Common middleware examples

[37:15](https://www.youtube.com/watch?v=hyc-7w3pee8&t=2235s)

| Middleware | What it does |
|---|---|
| **CORS** | Reads the `Origin` header. If it's an allowed origin (your frontend), adds the `Access-Control-Allow-*` headers to the response, then calls `next()`. Otherwise adds nothing, so the browser blocks the call. Runs on every request. |
| **Security headers** | Adds headers like `Content-Security-Policy` to every response |
| **Authentication** | Reads the token (JWT / session ID) from the request and verifies it. **Failure → `401` immediately, no `next()`**. **Success →** stores user ID and role in the **request context**, then `next()` |
| **Rate limiting** | Counts requests per IP (e.g. > 30 calls in 2 s) → **`429 Too Many Requests`**, otherwise `next()` |
| **Logging / monitoring** | Logs path, method, query params, etc. for every request, for debugging and auditing |
| **Global error handling** | Catches an error from *anywhere* (handler, service, other middleware), decides client (4xx) vs server (5xx) error, and sends a **consistently structured** error body (message, code, status) |
| **Compression** | Compresses large responses (gzip, etc.). The browser decompresses transparently |
| **Body parsing / validation** | Deserialize JSON, bind, validate/transform can be delegated to middleware instead of every handler |

---

## Middleware order

[47:32](https://www.youtube.com/watch?v=hyc-7w3pee8&t=2852s)

**Order matters**, because `next()` moves execution forward in sequence.

Typical order:

1. **CORS**, first, so requests from disallowed origins are rejected at the earliest possible moment.
2. **Logging**
3. **Authentication**
4. (permission checks, validation, routing, handler...)
5. **Global error handler**, **last**.

Why the error handler is last: an error raised in the handler can only be caught by a middleware that sits **after** the point where it occurred. Execution only flows one direction. If you put the error handler in the middle, errors from later steps never reach it.

---

## Request context

[50:35](https://www.youtube.com/watch?v=hyc-7w3pee8&t=3035s)

**Request context** = a store of **state scoped to one request**, typically key-value pairs, accessible by **all middlewares and handlers** that process that request. Every language/framework has some version (e.g. Go's `context.Context`, Express's `req`/`res.locals`).

Each of the thousands of simultaneous requests has its **own** context.

**Why:** the pieces of the pipeline need to share data **without being tightly coupled** to each other.

### Uses

| Use | How |
|---|---|
| **Authenticated identity** | Auth middleware verifies the token, stores `userId` and `role` in context. The handler reads them later. |
| **Request ID / trace ID** | A middleware generates a UUID and stores it. Every log line includes it. Pass it to downstream services in a header (`X-Request-Id`) so you can trace one request across microservices. |
| **Cancellation signals, abort signals, deadlines/timeouts** | So your server doesn't hang forever on slow downstream calls |

### A security point worth remembering

For `POST /books`, the book has a `user_id`. **Take it from the authenticated context, not from the client's payload.** If you trust the payload, a malicious client can send someone else's user ID. The identity from the auth middleware is the trusted one.

---

## Summary

1. Inside the server: **routing → handler → service → repository**, with **middleware** wrapped around.
2. **Handler/controller**: HTTP only. Extract, bind, validate, transform, call service, pick status code and body.
3. **Service**: business logic and orchestration. HTTP-agnostic.
4. **Repository**: database access only. One thing per method.
5. **Middleware** receives `request`, `response` and `next`. It can modify things, stop the request, or pass on. It exists to avoid duplicating cross-cutting code.
6. Typical middlewares: CORS, security headers, auth, rate limiting, logging, compression, parsing, **global error handling**.
7. **Order matters.** CORS early, error handler last.
8. **Request context** is per-request shared state: user, role, request ID, cancellation. Take trusted identity from it, not from the client.

## Interview questions

**Q: Why split a backend into handler, service and repository?**
Separation of concerns. HTTP details, business rules, and DB access change for different reasons. Splitting makes the code easier to maintain, test, extend and debug.

**Q: What should a service method not know about?**
HTTP: status codes, request/response objects. It should read like an ordinary function.

**Q: What does a repository method return?**
One kind of data. Don't overload one method to return different shapes depending on optional parameters.

**Q: What is middleware and what does `next` do?**
Middleware is a function run between lifecycle stages with access to request, response and `next`. `next()` hands control to the next middleware or handler. Not calling it (after sending a response) ends the request.

**Q: Why is the error-handling middleware last?**
Control flows forward only, so an error can only be caught by middleware positioned after where it occurred.

**Q: Why is CORS typically the first middleware?**
So unwanted origins are rejected as early as possible, before any other work is done.

**Q: What is request context and when would you use it?**
Request-scoped shared state. Use it to pass authenticated user info, a request/trace ID, and cancellation signals through the middleware chain to the handler without coupling them.

**Q: Why take `user_id` from context instead of the request body?**
The client can lie. Identity must come from the verified auth step.

**Q: Where does validation belong?**
At the entry of the handler layer (or in a validation middleware) before any service logic runs.
