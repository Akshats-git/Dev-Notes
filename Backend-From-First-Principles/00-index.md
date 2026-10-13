# Backend from First Principles — Notes

**Source:** [Sriniously — "Backend from first principles"](https://www.youtube.com/playlist?list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1) (YouTube playlist)
**Length:** 30 videos, about 35 hours in total
**Language:** Language-agnostic. The speaker shows examples in Go and Node.js, but the ideas apply to any stack. These notes keep code minimal and mostly language-neutral.
**Companion series:** the playlist announces two follow-ups, a Node.js + Go implementation playlist and a set of production projects. These notes cover only this concepts playlist.

---

## How to use these notes

The series is a **map of the backend territory**: every backend you will ever read has some version of routing → middleware → validation → handler → service → repository, with auth, logging, errors, config and security cutting across. The goal is to separate the *concept* from the *tool* so you can move between languages.

Read in order the first time. The early files (HTTP, routing, serialization) are used by everything after them. After that, use [29-quick-reference.md](29-quick-reference.md) as a cheat sheet.

If you only have time for one block:

- **Request path:** 03 → 09 (HTTP to REST design).
- **Data and speed:** 10, 11, 19, 20 (Postgres, caching, scaling).
- **Production readiness:** 14–18 and 24–25 (errors, config, observability, shutdown, security, testing, 12-factor).

---

## Contents

| # | File | Covers | Video |
|---|------|--------|-------|
| 01 | [Roadmap and mindset](01-roadmap-and-mindset.md) | What backend engineering is, why first principles, the 3-phase plan, the full topic map | [#1](https://www.youtube.com/watch?v=0Rwb4Xmlcwc) 31m · [#2](https://www.youtube.com/watch?v=3qFjZbFRSAU) 4m · [#4](https://www.youtube.com/watch?v=6fqZs5Z3k9A) 10m |
| 02 | [What is a backend](02-what-is-a-backend.md) | Request journey (DNS → firewall → Nginx → app), why a backend exists, why the frontend can't do it | [#3](https://www.youtube.com/watch?v=6Ss4dJD9Kzg) 19m |
| 03 | [HTTP](03-http.md) | Stateless, headers, methods, CORS and preflight, status codes, caching and ETags, negotiation, compression, TLS | [#5](https://www.youtube.com/watch?v=a3C1DMswClQ) 1h 18m |
| 04 | [Routing](04-routing.md) | Method + path, static/dynamic/nested routes, path vs query params, versioning, catch-all | [#6](https://www.youtube.com/watch?v=SubuU1iOC2s) 24m |
| 05 | [Serialization and deserialization](05-serialization-and-deserialization.md) | Common wire formats, text vs binary, JSON rules | [#7](https://www.youtube.com/watch?v=vzg90tY3uM0) 22m |
| 06 | [Authentication and authorization](06-authentication-and-authorization.md) | History, sessions/JWT/cookies, stateful vs stateless, API keys, OAuth 2.0/OIDC, RBAC, timing attacks | [#8](https://www.youtube.com/watch?v=A95rliroC8Q) 1h 36m |
| 07 | [Validation and transformation](07-validation-and-transformation.md) | Syntactic/semantic/type/complex validation, transformation, client vs server validation | [#9](https://www.youtube.com/watch?v=qedj_JjjL-U) 43m |
| 08 | [Handlers, services, repositories, middleware, context](08-handlers-services-repositories-middleware-context.md) | Request lifecycle inside the server, layering, middleware and `next`, ordering, request context | [#10](https://www.youtube.com/watch?v=hyc-7w3pee8) 1h |
| 09 | [REST API design](09-rest-api-design.md) | REST history, URL rules, idempotency, CRUD design, pagination/sort/filter, custom actions, consistency | [#11](https://www.youtube.com/watch?v=RG6q57DwV8Y) 2h 4m |
| 10 | [Databases with Postgres](10-databases-with-postgres.md) | Relational vs NoSQL, types, migrations, schema design, constraints, queries, parameterization, indexes, triggers | [#12](https://www.youtube.com/watch?v=F7Vwp2Xo5Do) 2h 45m |
| 11 | [Caching](11-caching.md) | CDN, DNS and hardware caches, Redis, cache-aside/write-through, eviction, use cases | [#13](https://www.youtube.com/watch?v=estH64OkwxU) 1h 4m |
| 12 | [Task queues and background jobs](12-task-queues-and-background-jobs.md) | Producer/broker/consumer, retries, acks, task types, idempotency | [#14](https://www.youtube.com/watch?v=r-nQsyguU1Y) 56m |
| 13 | [Full-text search with Elasticsearch](13-full-text-search-with-elasticsearch.md) | Inverted index, BM25, typo tolerance, Postgres FTS vs ES, benchmark | [#15](https://www.youtube.com/watch?v=7_sovzAhRSM) 32m |
| 14 | [Error handling and fault tolerance](14-error-handling-and-fault-tolerance.md) | Error types, health checks, recovery strategies, global error handler, safe messages | [#16](https://www.youtube.com/watch?v=8NaM_9aKS24) 1h 9m |
| 15 | [Configuration management](15-configuration-management.md) | Config types, sources, per-environment differences, secrets, startup validation | [#17](https://www.youtube.com/watch?v=GR9NtirPXyc) 36m |
| 16 | [Logging, monitoring, observability](16-logging-monitoring-observability.md) | Logs/metrics/traces, levels, structured logs, OpenTelemetry, New Relic demo | [#18](https://www.youtube.com/watch?v=5PEuwgLOQQM) 40m |
| 17 | [Graceful shutdown](17-graceful-shutdown.md) | Signals, SIGTERM/SIGINT/SIGKILL, connection draining, resource cleanup | [#19](https://www.youtube.com/watch?v=6rfBgphiCWM) 36m |
| 18 | [Backend security](18-backend-security.md) | Injection, password hashing, sessions/JWT pitfalls, BOLA/BFLA, XSS, CSRF, misconfiguration, defence in depth | [#20](https://www.youtube.com/watch?v=xB1C1xZZW4k) 2h 50m |
| 19 | [Scaling and performance, part 1](19-scaling-and-performance-part-1.md) | Latency, percentiles, utilization, bottlenecks, N+1, indexes, pooling, caching, vertical vs horizontal | [#21.1](https://www.youtube.com/watch?v=z7kt_p44rjs) 1h 48m |
| 20 | [Scaling and performance, part 2](20-scaling-and-performance-part-2.md) | Statelessness, load balancers, read replicas, sharding, CDN/edge, async, microservices, serverless | [#21.2](https://www.youtube.com/watch?v=sOhAopEwjH4) 2h 18m |
| 21 | [Concurrency and parallelism](21-concurrency-and-parallelism.md) | I/O-bound vs CPU-bound, threads, event loop, goroutines, async/await, race conditions | [#22](https://www.youtube.com/watch?v=bs9MEYRTA30) 1h 28m |
| 22 | [Object storage](22-object-storage.md) | S3 model, key design, durability, pre-signed URLs, multipart, downloads, range requests | [#23](https://www.youtube.com/watch?v=Ie0TjKI9cDI) 1h 29m · [#24](https://www.youtube.com/watch?v=iWrVCxexUWY) 50m |
| 23 | [Real-time backends](23-real-time-backends.md) | Polling, long polling, SSE, WebSockets, connection limits, pub/sub | [#25](https://www.youtube.com/watch?v=wUQryt697cs) 52m |
| 24 | [Testing for backend engineers](24-testing-for-backend-engineers.md) | Test sizes, test doubles, DI, real DBs, flaky tests, TDD, coverage and mutation testing | [#26](https://www.youtube.com/watch?v=HgVrfHX92y4) 1h 25m |
| 25 | [The Twelve-Factor App](25-twelve-factor-app.md) | All 12 factors, history and what aged | [#27](https://www.youtube.com/watch?v=2_fcv3-XfN4) 1h 33m |
| 26 | [OpenAPI](26-openapi.md) | The contract problem, OpenAPI structure, code-first vs design-first, toolchain | [#28](https://www.youtube.com/watch?v=CwKsU84jIWs) 44m |
| 27 | [Webhooks](27-webhooks.md) | Receiving (HMAC, replay, idempotency, ordering) and sending (outbox, retries, SSRF) | [#29](https://www.youtube.com/watch?v=eWM0CVReP04) 56m |
| 28 | [DevOps for backend engineers](28-devops-for-backend-engineers.md) | CI/CD, containers, images, Kubernetes, rollouts, GitOps, IaC, SRE basics | [#30](https://www.youtube.com/watch?v=_E8X5RIJfQ4) 1h 48m |
| 29 | [Quick reference](29-quick-reference.md) | Cheat sheet: status codes, patterns, rules of thumb, checklists | — |

> The roadmap video (#1) promised some topics that were folded into other videos or not made as separate videos (e.g. transactional emails, a standalone business-logic-layer video). The notes follow the 30 videos that actually exist.

---

## The short summary

A **backend** is a computer listening on a port, serving centralised **data** to clients whose code runs in a sandboxed browser. Requests reach it via **DNS → firewall → reverse proxy → app**. **HTTP** is stateless and client-initiated. A method + path is **routed** to a **handler**.

Data crosses the wire as **serialized** text (JSON). Everything entering is **validated and transformed** at the door (client validation is UX, server validation is security). Inside, a **handler** (HTTP concerns) calls a **service** (business logic) calling a **repository** (database), with **middleware** (CORS, auth, rate limit, logging, error handling) around it and a per-request **context** for shared state.

APIs follow **REST** conventions: plural nouns, correct methods and status codes, pagination with defaults, consistent naming, documented by **OpenAPI**. **Authentication** (stateful sessions or stateless JWT, OAuth/OIDC for delegation, API keys for machines) proves who you are. **Authorization** (RBAC, plus object-level checks at the data layer) decides what you can do. Defences like parameterized queries, slow salted password hashes, `HttpOnly`/`Secure`/`SameSite` cookies and layered rate limits come from asking "where does data cross a boundary?".

Data lives in **Postgres** with migrations, constraints and indexes. **Caching** (Redis, CDN) saves repeated work. **Task queues** move slow or unreliable work out of the request. **Elasticsearch** handles ranked, typo-tolerant search. **Object storage** holds files, uploaded directly from the browser with pre-signed URLs. **WebSockets/SSE** push data to clients, scaled with pub/sub. **Webhooks** let providers call you, so verify, dedupe and answer fast.

Production readiness comes from: structured **errors** with a global handler; validated **config** (12-factor); **logs/metrics/traces**; **graceful shutdown**; **tests** at the right sizes; **concurrency** matched to I/O vs CPU; **scaling** informed by percentiles and measurement, not guesses.

Finally, **DevOps** gets it all to the internet: reproducible builds, **containers** (namespaces + cgroups + layered FS), registries, systemd/Kamal or **Kubernetes**, safe rollouts, **GitOps**, **infrastructure as code**, and operating with SLOs, error budgets and blameless postmortems.
