# 01 — Roadmap, Philosophy and Why First Principles

**Videos:**
- [#1 Roadmap for backend from first principles](https://www.youtube.com/watch?v=0Rwb4Xmlcwc) (31m)
- [#2 Walk the path of a true backend engineer](https://www.youtube.com/watch?v=3qFjZbFRSAU) (4m)
- [#4 Benefits of learning backend engineering from first principles](https://www.youtube.com/watch?v=6fqZs5Z3k9A) (10m)

(Video #3, "What is a backend", is in [02](02-what-is-a-backend.md).)

These three videos set up the whole series: what "backend engineering" means, why the series avoids any single language, how it is planned, and the full map of topics. No code here. It is the "why" and the table of contents.

---

## Contents

1. [What backend engineering actually is](#what-backend-engineering-actually-is)
2. [Why learn from first principles](#why-learn-from-first-principles)
3. [The three phases of the series](#the-three-phases-of-the-series)
4. [The full roadmap](#the-full-roadmap)
5. [What "first principles" means here](#what-first-principles-means-here)
6. [Summary](#summary)
7. [Interview questions](#interview-questions)

---

## What backend engineering actually is

[0:00](https://www.youtube.com/watch?v=0Rwb4Xmlcwc&t=0s)

> Backend engineering is much more than building a set of CRUD APIs. It is about building **reliable, scalable, fault-tolerant, maintainable** code bases and **efficient** systems.

Two problems the series is trying to fix:

| Problem | What happens |
|---|---|
| **Too many resources, no map** | There are thousands of tutorials. You can't tell what to learn, in what order, or how the pieces connect. People pick it up through trial and error over years. |
| **Language-first learning** | You learn through Express, Spring Boot or Rails, and see every problem through that one ecosystem. Switch language (say Rails → Go for performance) and you can't tell how much of your knowledge transfers, because you never separated the *concept* from the *tool*. |

The fix is a language-agnostic set of videos built from books and from reading many open-source code bases.

---

## Why learn from first principles

[#4 · 0:00](https://www.youtube.com/watch?v=6fqZs5Z3k9A&t=0s)

The video opens with situations where syntax knowledge is not enough:

- You're a frontend dev asked to fix a bug in a backend in a language you don't know. Where do you even look?
- You must build an API from scratch. How do you build a mental map and not break things?
- You're a TypeScript or Go engineer, suddenly moved to Rust or Python. Do you read hours of library docs (FastAPI, Pydantic, Axum, SQLAlchemy, Diesel)?

First principles means breaking a system into its **basic, universal components**. The payoffs:

| Benefit | What it means |
|---|---|
| **See the big picture** | In an unfamiliar code base you can mentally separate routing, core logic, DB connections and over-engineered parts, ignore the noise, and fix things with confidence. Senior engineers do this by pattern recognition built over years. You can practise it deliberately from day one. |
| **Faster onboarding** | Once you know how HTTP works, how requests flow through middleware, and how auth and DB access are structured, the syntax is secondary. You cut through the noise. |
| **10x faster on new projects** | You build MVPs with production-quality structure because you know what the system needs (routes, DB connections, caching, error handling, logging), not just a tutorial's boilerplate. |
| **Less syntax fatigue** | When you know *which problem* each piece solves, switching language is only "apply the right syntax and library". |
| **Right tool for the job** | You stop being "a Node dev" or "a Ruby dev". You know when Redis, Postgres, MongoDB or Kafka fits, regardless of stack. |
| **More employable** | Teams want engineers who can think independently and contribute in any environment. |

### The Node → Rust example

Say you know Node and want to move to Rust, but there are no 5-hour production-quality Rust project tutorials. First principles solve this:

1. You already understand each layer as a *concept*: routing, middleware, validation, DB access, repository pattern, handlers, auth, logging, error handling, async work.
2. Start a Rust project with the community's recommended layout.
3. Take one component at a time. Know what production-quality validation looks like, then look up *how Rust does validation* (usually a library).
4. Repeat for each module.

In 2–3 days you have a production-quality Rust code base. The concepts transfer. Only the syntax is new.

> **Framework developer → software engineer.** You're no longer limited by a stack. You understand the core problems backend engineering solves.

---

## The three phases of the series

[#2 · 0:00](https://www.youtube.com/watch?v=3qFjZbFRSAU&t=0s)

| Phase | What it is | Language |
|---|---|---|
| **1. Story and philosophy** (this playlist) | The big questions, the inner workings, how components collaborate. It explains what frameworks and runtimes abstract away. | None. Language-agnostic |
| **2. Implementation** (next playlist) | Each principle gets a deep-dive in code. For databases, that means Postgres with a JS driver (postgres.js) and with Go's `pgx`: drivers, migrations, everything around them. | **Node.js** and **Go** |
| **3. Production projects** | Everything comes together: end-to-end projects with industry standards and best practices. | Follow along in either |

The promise: if you follow all three phases you can build systems that scale from zero users to a million, and that other people can maintain for a long time.

---

## The full roadmap

[#1 · 2:02](https://www.youtube.com/watch?v=0Rwb4Xmlcwc&t=122s)

The roadmap video walks the entire topic map. The series follows it in this order (the later notes in this folder cover each one):

| Area | Topics promised |
|---|---|
| **How requests travel** | Browser → network hops → firewalls → AWS server → response. What a response looks like. |
| **HTTP** | Raw messages, headers (request, representation, general, security), methods and their semantics, CORS and preflight, status codes, caching (ETag, max-age), HTTP/1.1 vs 2 vs 3, content negotiation, persistent connections, compression (gzip, deflate, brotli), TLS/HTTPS |
| **Routing** | URLs → server logic, path and query params, static/dynamic/nested/catch-all/regex routes, API versioning and deprecation, route groups, securing routes, route-matching performance |
| **Serialization** | Text (JSON, XML) vs binary (Protobuf), per-language implementation, JSON data types, missing/extra fields, nulls, date and timezone issues, custom serializers, schema validation, performance trade-offs |
| **Authentication / authorization** | Stateful vs stateless, basic, bearer tokens, sessions, JWT, cookies, OAuth, OpenID Connect, API keys, MFA, hashing and salting, ABAC / RBAC / ReBAC, CSRF/XSS/MITM, audit logging, obscuring errors, timing attacks |
| **Validation and transformation** | Syntactic, semantic, type validation; client vs server side; fail fast; type casting, normalisation, sanitisation; relationship, conditional and chained validation; error aggregation |
| **Middleware and request context** | Pre/post middleware, chaining, `next`, ordering (log → auth → validate → handle → errors), common middlewares (security headers, CORS, CSRF, rate limit, compression, body parsing), request-scoped context (request ID, user, trace ID, timeouts, cancellation) |
| **Handlers, controllers, services** | MVC, CRUD to HTTP method mapping, pagination, search, sort, filter, consistent response formats |
| **REST** | Resource-oriented design, HTTP semantics, versioning styles (URI, header, query, media type), OpenAPI-first design, content negotiation, ETags |
| **Databases** | Relational vs non-relational, ACID, CAP, joins, schema design, indexing, query optimisation, connection pooling, constraints, transactions, concurrency, ORMs and trade-offs, migrations |
| **Business logic layer** | Presentation → business → data-access layering, separation of concerns, SRP, open/closed, dependency inversion, services, domain models, error propagation |
| **Caching** | Memory, browser, DB caching; cache-aside, write-through, write-behind, read-through; LRU, LFU, TTL, FIFO; invalidation; L1/L2/hierarchical; hit and miss ratio |
| **Transactional email** | Anatomy (subject, preheader, body, CTA, footer), personalisation |
| **Task queues and scheduling** | Producer / queue / consumer / broker / backend; chains and groups; retries; priority and rate limiting; scheduled jobs |
| **Elasticsearch** | Inverted index, TF-IDF, segments and shards, analyzers, relevance, fuzzy search, aggregation, Kibana, best practices |
| **Error handling** | Fail-safe vs fail-fast, graceful degradation, custom error types, global handlers, Sentry/ELK, alerting |
| **Config management** | Static, dynamic (feature flags) and sensitive config; env vars vs flags vs files |
| **Logging, monitoring, observability** | Log levels, structured logs, rotation, Prometheus/Grafana, alert fatigue, logs + metrics + traces |
| **Graceful shutdown** | Signals (SIGTERM/SIGINT/SIGKILL); stop accepting → drain in-flight → close resources → exit |
| **Security** | SQL and NoSQL injection, XSS, CSRF, broken auth, insecure deserialization; least privilege, defense in depth, fail-secure, separation of duties |
| **Scaling and performance** | Response time, N+1 queries, indexes, batching, memory leaks, payload size, load testing, profiling, graceful degradation, offloading to queues |
| **Concurrency and parallelism** | IO-bound vs CPU-bound |
| **Object storage and large files** | S3, chunking, streaming, multipart upload |
| **Real-time backends** | WebSockets, Server-Sent Events, pub/sub |
| **Testing and code quality** | Unit, integration, e2e, regression, load, stress, UAT, security; TDD; CI; linting; cyclomatic complexity; maintainability index |
| **12-Factor App** | The principles for cloud-native apps |
| **OpenAPI** | Swagger → OpenAPI 3.0/3.1, paths, schemas, components, tooling, API-first development |
| **Webhooks** | Polling vs push, signature verification, retries, testing with ngrok, Stripe/GitHub/Slack/Twilio |
| **DevOps** | CI/CD (integration, delivery, deployment), IaC, Docker, Kubernetes, horizontal vs vertical scaling, blue-green and rolling deploys |

> The roadmap was a plan. The final playlist has **30 videos** and follows this order closely. A few items (transactional emails as their own video, a separate business-logic-layer video) were folded into other videos. See the [index](00-index.md) for what actually exists.

---

## What "first principles" means here

[#4 · 9:43](https://www.youtube.com/watch?v=6fqZs5Z3k9A&t=583s)

Not a list of rules. It means **foundational building blocks** around which every code base revolves, no matter how big or small. A generic map of the backend territory.

```
 Request ──► Routing ──► Middleware ──► Validation ──► Handler/Controller
                                                           │
                                                      Service (business logic)
                                                           │
                                                   Repository / DB / Cache / Queue
                                                           │
 Response ◄── Serialization ◄───────────────────────────────┘

 Cross-cutting: auth, logging, errors, config, security, observability
```

Every backend you will ever read has some version of this picture.

---

## Summary

1. Backend engineering is about **reliability, scalability, fault tolerance and maintainability**, not just CRUD.
2. Learning through one framework gives you blind spots. Learning the **concepts** first lets you move between languages.
3. First principles give you: the big picture, fast onboarding, speed on new projects, less syntax fatigue, tool-choice judgement, employability.
4. The series has 3 phases: **philosophy (this)** → **Node.js and Go implementations** → **production projects**.
5. The map covers HTTP, routing, serialization, auth, validation, middleware, REST, databases, caching, queues, search, errors, config, observability, shutdown, security, scaling, concurrency, object storage, real-time, testing, 12-factor, OpenAPI, webhooks and DevOps.

## Interview questions

**Q: Why learn backend "from first principles" instead of through a framework?**
Frameworks change. The problems they solve (routing, validation, auth, persistence, caching, error handling) do not. If you understand the problem, moving to a new framework or language only means learning new syntax for a known pattern.

**Q: You're moved from Node to a language you've never used. How do you become productive quickly?**
Treat each layer as a known concept (routing, middleware, validation, repository, auth, logging). Learn the language basics, then for each layer find that ecosystem's idiomatic library and map your existing pattern onto it.

**Q: How would you approach an unfamiliar backend code base?**
Split it mentally into layers (entry point and routing, middleware, validation, handlers, services, data access, config) and find the layer where the bug or feature lives, ignoring the rest.

**Q: What makes a backend "production grade"?**
It's reliable, scalable, fault tolerant and maintainable, which means it also has proper logging and monitoring, error handling, config management, security, graceful shutdown and tests, beyond just working endpoints.
