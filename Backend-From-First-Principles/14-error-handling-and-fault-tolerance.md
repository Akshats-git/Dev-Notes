# 14 — Error Handling and Building Fault-Tolerant Systems

**Video:** [#16 Error Handling and Building Fault Tolerant Systems](https://www.youtube.com/watch?v=8NaM_9aKS24) (1h 9m)

No tools or code here. This is a **mindset**: errors *will* happen (DB queries fail, external APIs time out, users send bad data, business logic hits edge cases). The question is how ready you are to detect, contain and fix them.

---

## Contents

1. [Types of errors](#types-of-errors)
2. [Prevention: find errors before they spread](#prevention-find-errors-before-they-spread)
3. [Monitoring and observability](#monitoring-and-observability)
4. [Philosophies for responding](#philosophies-for-responding)
5. [Global error handling: the final safety net](#global-error-handling-the-final-safety-net)
6. [Security aspects of error handling](#security-aspects-of-error-handling)
7. [Summary](#summary)
8. [Interview questions](#interview-questions)

---

## Types of errors

[2:00](https://www.youtube.com/watch?v=8NaM_9aKS24&t=120s)

### 1. Logic errors

The sneakiest and most dangerous. The code runs fine, **but does the wrong thing**. It doesn't crash, so it can go unnoticed for weeks.

*Example:* a discount is applied twice, giving **negative shipping costs**. Every order loses money.

*Causes:* misunderstood requirements, incorrectly implemented algorithms, unconsidered edge cases (or user behaviour). Dangerous with payments, money, security, and can silently corrupt data.

### 2. Database errors

Backends depend heavily on the DB, so these can take everything down.

| Kind | Meaning |
|---|---|
| **Connection errors** | App can't reach the DB: network down, DB overloaded, **connection pool exhausted**. Backend returns 500s, frontend shows empty screens |
| **Constraint violations** | Operation breaks DB rules: **unique** (email already exists), **foreign key** (order references a customer that doesn't exist). Mostly a validation-layer gap. Unique violations can't always be pre-checked (only the DB knows), so **format the error** for the user |
| **Query errors** | Malformed SQL (typo in a table name), queries too complex → time out |
| **Deadlocks** | Operations wait on each other in a circular dependency |

(Connection pools keep open TCP connections to avoid the handshake cost on every request.)

### 3. External service errors

Payment processors, email providers, S3, Redis, auth providers (Auth0, Clerk). Each is a **point of failure you don't control**, and you can't avoid dependencies.

| Cause | What to do |
|---|---|
| **Network**: timeouts, DNS failures, partitions | Expect and plan for them |
| **Authentication errors**: bad credentials, expired tokens, insufficient permissions | Also, using an auth provider doesn't remove *your* responsibility (e.g. don't log sensitive info) |
| **Rate limiting (HTTP 429)**: e.g. your logic bug floods their API | **Exponential backoff**: wait 1 min, then 2, then 4... until success |
| **Service outage** (even AWS/GCP have incidents) | **Fallbacks**: e.g. Redis down → in-memory cache or a second node, without breaking payments/orders |

### 4. Input validation errors

Users send data that doesn't meet your rules. Validation is the **first line of defence**.

| Rule | Examples |
|---|---|
| **Format** | email/phone/date look right; be exact about custom formats |
| **Range** | max/min numbers, string too long/short, array with fewer than 3 or more than 100 items |
| **Required fields** | mandatory field missing |

Respond with **`400 Bad Request`**. These are the **easiest** errors to handle since you already know the rules. So make this layer robust.

### 5. Configuration errors

Show up when moving between dev → staging → prod.

*Example:* you add `OPENAI_API_KEY` to `.env` in dev, merge, and forget to add it to production.

| If you... | Then... |
|---|---|
| **Validate all required config at startup** and fail if anything's missing | The new deployment fails to start. With blue-green deploys, the **old one keeps running**. **Best case.** |
| Don't validate | The app starts fine. Later a user hits the OpenAI endpoint → config missing → **500 at runtime**. **Worst case.** |

> **Prefer crashing at startup over failing at runtime in front of real users.**

(Config management has its own lecture.)

---

## Prevention: find errors before they spread

[24:09](https://www.youtube.com/watch?v=8NaM_9aKS24&t=1449s)

> **The best error handling starts before the error happens.**

### Health checks

Expose an endpoint like `/health` or `/status`. The important thing is the **status code**: `200` = running; `4xx/5xx` = problem. External tools can ping it continuously.

But "the server is up" isn't enough. Verify it's **doing its job**:

| Check | What |
|---|---|
| **Database** | Connectivity, **query performance** (a query that went from 500 ms to 4 s is a signal), data integrity. Run a *representative query*, not just a ping |
| **External services** | Payments → a **test transaction** (so you know the last test 5 min ago worked). Email → send test messages to internal addresses. Auth → generate a test token and validate it against the provider's endpoint |
| **Core functionality** | Config loaded, required caches populated, internal data structures consistent |

This is **proactive error detection**.

---

## Monitoring and observability

[29:44](https://www.youtube.com/watch?v=8NaM_9aKS24&t=1784s)

(Own lecture later: [16](16-logging-monitoring-observability.md).) Key points here:

- Detect errors **quickly, while they happen**, with enough context to debug.
- **Don't track only error rates.** Track **performance** too (response times, resource usage, throughput). Degradation often **precedes failure**.
- Cover all error sources: HTTP, database, external services, business logic.
- Track **business metrics**: a sudden drop in successful transactions or logins may show a problem even if technical error rates look normal.
- **Structured logging** (JSON) so logs are parseable and rich. Aggregate with tools like Grafana / Loki.

---

## Philosophies for responding

[33:21](https://www.youtube.com/watch?v=8NaM_9aKS24&t=2001s)

### 1. Immediate error response

| Error type | Strategy |
|---|---|
| **Recoverable** (network errors, temporary resource problems, email send fails, out of DB connections) | **Retry** with exponential backoff. Be careful **not to overwhelm an already stressed system** |
| **Non-recoverable** | **Containment and graceful degradation**: serve cached data, disable non-essential features, provide alternative functionality, limit the blast radius |

### 2. Recovery strategies

- **Automatic recovery** handles many cases: restart a failed service, clean corrupted caches, switch to backups. Design carefully, since some "fixes" can make things worse. Learn by trial and error which works for which service.
- **Manual recovery** is needed when judgment is required. **Document** these runbooks and **test** them so they work under stress.
- **Data recovery**: data is the most important thing (code and servers are replaceable). Prioritise data integrity: backups at key moments, restore, replay transaction logs, specialised recovery tools.

### 3. Propagation control

Not every error should be handled where it occurs. Sometimes it should bubble up to a level with **more context**. Use exception hierarchies: catch low-level exceptions, **wrap with context**, re-throw upward. Keep the bubbling under your control, or it can spread and take down the main service.

### 4. Error boundaries

Stop errors from propagating between services: separate processes, **timeouts** at service boundaries, **message queues** (RabbitMQ) to decouple services so a bug in one doesn't crash another.

---

## Global error handling: the final safety net

[41:38](https://www.youtube.com/watch?v=8NaM_9aKS24&t=2498s)

The speaker's favourite mechanism. A **one-time effort that pays off immediately**: the most important error-handling strategy in a backend app.

### The architecture

```
Routing → Handler → Service → Repository (leaf: DB queries)
                                  │
 errors thrown / returned from ANY layer
                                  ▼
        ───►  GLOBAL ERROR HANDLER (middleware, last in the chain)  ───►  HTTP error response
```

- In languages with exceptions (JS, Python): **throw**, and catch in the global middleware.
- In Go-style languages: **return** the error up through repository → service → handler to the middleware.
- Create **custom error types** in each layer (validation errors from the handler, internal errors from the service, DB errors from the repository).

### What the middleware does with each error

Example API: create a book (name ≤ 500 chars, optional description, `author_id`).

| Error | Where it came from | Response |
|---|---|---|
| Name is 700 chars | Handler validation | **400** + field errors so the user can fix the data |
| **Unique constraint violation** (book name exists) | DB / repository | **400** (or 409), message: *"book already exists"* |
| **No rows** (`GET /books/123`, the row doesn't exist) | DB / repository | **404**, *"book with ID 123 does not exist"*. Resource not found |
| **Foreign key violation** (`author_id` doesn't exist) | DB / repository | **404** (referenced author doesn't exist) |
| Anything unrecognised | anywhere | **500**, *"something went wrong"* (generic) |

An error structure usually carries: `code` (HTTP status), `message`, and optional field errors (an array of objects).

### Two advantages

1. **More robust.** Without it, each repository method must remember to translate DB errors. Forget once and the user gets a meaningless `500` instead of *"book already exists"*. A central handler can't forget.
2. **Less redundancy.** One place for the unique/FK/no-rows translation logic rather than repeated in every layer. Fewer bugs.

---

## Security aspects of error handling

[1:00:01](https://www.youtube.com/watch?v=8NaM_9aKS24&t=3601s)

### 1. Don't leak internals in error messages

If the global handler forwards whatever message bubbled up, a DB error can reveal **table names, constraint names, index names**. Attackers use them for more targeted attacks (e.g. SQL injection).

**Rule:** control your messages. Understand the error type and write a message **meant for the user**. For the default fallthrough case, return `500` + *"something went wrong"* (or just "Internal Server Error"). If you reached the default branch you don't know what the error was, and it may contain internal details.

### 2. Authentication errors must be vague

For login, never say *"no user with this email"* vs *"incorrect password"*. An attacker tries many emails with one password. When the message changes to *"incorrect password"*, they've found a valid email and switch to guessing passwords. Always: **"Invalid email or password."** (See the [auth notes](06-authentication-and-authorization.md) for timing attacks too.) The **OWASP Cheat Sheet** series has per-topic guidance.

### 3. Don't log sensitive data

Logs flow through multiple external services (storage, analysis, observability). In many major breaches the *logs* leaked. Never log emails, passwords, API keys, credit-card numbers. Log the **user ID** and a **correlation (request) ID** instead.

---

## Summary

1. Errors are normal. Build a **fault-tolerant mindset**.
2. Error types: **logic** (silent, dangerous), **database** (connection, constraint, query, deadlock), **external service** (network, auth, rate limit, outage), **input validation** (easiest), **configuration**.
3. **Validate config at startup**: crash early, not at runtime.
4. **Best error handling begins before the error**: health checks that test real work (DB, external services, core functionality).
5. Monitor performance and **business metrics**, not just error rates.
6. **Recoverable** → retry with backoff. **Non-recoverable** → contain and degrade gracefully.
7. Control **propagation**: wrap with context, bubble up to a boundary. Use **error boundaries** (timeouts, queues).
8. **Global error handler** (last middleware) maps DB/validation/internal errors to the right HTTP responses in one place.
9. Never expose internals in messages. Keep auth errors vague. Don't log secrets or PII.

## Interview questions

**Q: Why are logic errors the most dangerous?**
They don't crash anything, so they go unnoticed while quietly corrupting data or losing money.

**Q: How do you handle a third-party API returning 429?**
Exponential backoff with a retry cap (and respect `Retry-After` if provided). Ideally do it in a background job so the user request isn't blocked.

**Q: What should happen when a required environment variable is missing?**
Fail at startup with a clear message. With blue-green deploys the old version keeps serving. Never discover it at runtime.

**Q: What should a good health check do?**
Verify the service is doing its job: connect to the DB and run a representative query, test critical external services, confirm config/caches loaded. Not just "returns 200".

**Q: Recoverable vs non-recoverable errors?**
Recoverable (transient): retry with backoff. Non-recoverable: contain damage and degrade gracefully (cached data, disabled features, fallbacks).

**Q: What is a global error handler and why use one?**
A final middleware that receives errors from every layer and converts them into consistent HTTP responses. It makes error mapping robust (can't forget a case) and avoids duplication.

**Q: Map these to status codes: validation failure, unique violation, no rows on GET-by-id, unknown error.**
400, 400/409, 404, 500 (generic message).

**Q: Why not return the raw DB error message to clients?**
It exposes internals (table/constraint names) that help attackers.

**Q: What should you log on an auth failure?**
A user ID and correlation ID, never emails, passwords, or tokens.

**Q: What are error boundaries?**
Mechanisms that stop failures spreading between components: separate processes, timeouts, message queues, circuit-breaking.
