# 29 — Quick Reference

Everything condensed. Use this after the first read-through. Each section points back to the file with the full explanation.

---

## The request path

```
Client → DNS → firewall (80/443) → reverse proxy (Nginx/Caddy) → app on localhost:PORT
   → routing → middleware chain → handler → service → repository → DB / cache / queue
   ← serialization (JSON) ← ... ← response
```

| Layer | Owns | Doesn't know about |
|---|---|---|
| **Handler/controller** | HTTP: extract, bind, validate, transform, status codes | business rules, SQL |
| **Service** | business logic, orchestration | HTTP |
| **Repository** | database access, one thing per method | HTTP, business rules |
| **Middleware** | cross-cutting: CORS → logging → auth → rate limit → ... → **global error handler (last)** | |
| **Request context** | per-request state: user, role, request ID, cancellation | |

See [02](02-what-is-a-backend.md), [08](08-handlers-services-repositories-middleware-context.md).

---

## HTTP

| Method | Intent | Idempotent |
|---|---|---|
| GET | read | yes |
| POST | create / custom action | **no** |
| PUT | replace whole resource | yes |
| PATCH | partial update | yes (set semantics) |
| DELETE | remove | yes |
| OPTIONS | CORS preflight | |

### Status codes

| Code | Use |
|---|---|
| **200** OK · **201** Created · **204** No Content (DELETE, preflight) | success |
| **301** permanent · **302** temporary · **304** Not Modified (ETag) | redirect/cache |
| **400** bad input · **401** not authenticated · **403** authenticated but not allowed · **404** not found · **405** wrong method · **409** conflict · **429** rate limited | client error |
| **500** unexpected · **501** not implemented · **502** bad upstream response · **503** unavailable · **504** upstream timeout | server error |

Also **206** Partial Content / **416** Range Not Satisfiable (range requests), **101** Switching Protocols (WebSocket), **412** Precondition Failed (conditional writes).

**CORS:** the *browser* enforces it. JSON + `Authorization` ⇒ preflight `OPTIONS` → server returns `Access-Control-Allow-Origin/Methods/Headers/Max-Age`.

**Caching:** `Cache-Control`, `ETag` / `If-None-Match` → `304`. **Negotiation:** `Accept`, `Accept-Language`, `Accept-Encoding`. See [03](03-http.md).

---

## REST design rules

- Plural lowercase nouns, hyphens, `/` = hierarchy: `/organizations/:id/projects`.
- List+create share one URL. Get/update/delete share `/:id`. Custom action = `POST /:id/<verb>`.
- Create → **201** + entity. Update → **200** + entity. Delete → **204** empty. Custom action → 200 *or* 201 depending on what happened.
- List endpoints: `page`, `limit`, `sortBy`, `sortOrder`, filters, with **defaults** (page 1, limit 10–20, `createdAt desc`). Always sort. Empty list = **200**, missing single resource = **404**.
- JSON in **camelCase**; same field names everywhere; no abbreviations.
- Document with **OpenAPI**; design the interface before coding.

See [04](04-routing.md), [09](09-rest-api-design.md), [26](26-openapi.md).

---

## Validation

| Kind | Example |
|---|---|
| Syntactic | is it an email / date / phone |
| Semantic | birth date not in the future; age ≤ 120 |
| Type | string / number / array / each element |
| Complex | password == confirm; partner required if married |

Query/path params are **strings** → cast first. **Server validation is mandatory** (client validation is UX only). Fail fast with **400**. See [07](07-validation-and-transformation.md).

---

## Auth

| Need | Use |
|---|---|
| Web app, revocation matters | **Stateful sessions** (ID in `HttpOnly; Secure; SameSite` cookie, state in Redis/DB) |
| Distributed APIs, mobile | **Stateless JWT** (short expiry + refresh tokens, no sensitive payload) |
| "Login with Google" / delegated access | **OAuth 2.0 + OIDC** (access token + ID token) |
| Machine-to-machine | **API keys** / OAuth client credentials |
| Permissions | **RBAC**; plus **object-level checks in the repository** |

- Passwords: **Argon2id / bcrypt / scrypt + per-user salt**, never SHA-256/MD5, never plaintext.
- Auth errors are **generic** ("Invalid credentials"). Guard against **timing attacks** (constant-time compare).
- Layered rate limits on auth: per IP, per account, global.
- Prefer an **auth provider** in production.

See [06](06-authentication-and-authorization.md), [18](18-backend-security.md).

---

## Security checklist ("where does data cross a boundary?")

- [ ] **Parameterized queries** everywhere; DB user without DDL rights.
- [ ] Command execution via **argument arrays**, never string concatenation.
- [ ] Sanitize user markup; **CSP**, `HttpOnly` cookies against XSS.
- [ ] `SameSite` cookies (CSRF).
- [ ] **Authorization at the point of access**: `WHERE id = $1 AND user_id = $2`; **404 not 403** to avoid leaking existence; UUIDs; **role checks** for admin functions; **default deny**.
- [ ] No secrets in Git (rotate if leaked); no debug logging in prod; security headers (`X-Frame-Options`, CSP...).
- [ ] Never log passwords/tokens/PII; return generic 500 messages.
- [ ] Automated authorization tests; audit logs.

---

## Database (Postgres)

- Types: `text` (not `varchar(255)`), `numeric` for money, `timestamptz`, `uuid`, `jsonb`, enums for fixed sets.
- Constraints: `PRIMARY KEY` (NOT NULL+UNIQUE), `UNIQUE`, `NOT NULL` (most columns), `CHECK`, `FOREIGN KEY ... ON DELETE RESTRICT|CASCADE|SET NULL`.
- Relationships: **1:1** child PK = parent key · **1:N** FK on the many side · **N:M** linking table with composite PK.
- **Migrations**: ordered up/down SQL files in Git (dbmate, golang-migrate).
- `LEFT JOIN` when the related row may not exist. **Always `ORDER BY`** lists. `LIMIT/OFFSET` = `(page-1)*limit`.
- **Indexes**: for `WHERE`/`JOIN`/`ORDER BY` columns on frequent queries; PKs are auto-indexed, **FKs are not**; indexes slow writes. Verify with **`EXPLAIN ANALYZE`**.
- **Triggers** keep `updated_at` correct.
- **Connection pooling** (external pooler like PgBouncer when scaled out). Avoid **N+1**.

See [10](10-databases-with-postgres.md), [19](19-scaling-and-performance-part-1.md).

---

## Caching

| Pattern | Flow |
|---|---|
| **Cache-aside** | read: cache → miss → DB → fill cache. write: update DB, **delete** key |
| **Write-through** | write DB and cache together |
| **Write-behind** | write cache, persist to DB async (inconsistency risk) |

Eviction: **LRU, LFU, TTL**, none. Invalidation: **TTL** vs **event-based**. Local vs distributed vs tiered. Watch **hit rate** (~90% good). Uses: DB query results, sessions, external API results, **rate limiting counters**. CDN/DNS caches exist too. See [11](11-caching.md).

---

## Async work and search

- **Queue**: producer → broker (RabbitMQ/Redis/SQS) → consumer. **Exponential backoff** retries, acks, **visibility timeout**. Task types: one-off, recurring, chain, batch. Make tasks **idempotent**, small, observable. See [12](12-task-queues-and-background-jobs.md).
- **Search**: `ILIKE '%x%'` scans everything. **Inverted index** + **BM25** relevance + fuzzy matching → Elasticsearch / Postgres FTS. See [13](13-full-text-search-with-elasticsearch.md).

---

## Errors, config, logs, shutdown

| Topic | Rule |
|---|---|
| **Errors** | Throw/return up to a **global error handler** (last middleware) that maps validation→400, unique/FK→400/404, no rows→404, unknown→500 (generic). Retry recoverable errors with backoff; contain non-recoverable. Real **health checks**. Fail fast on bad config at startup |
| **Config** | Env vars; no hard-coded values; **validate at startup** (Zod etc.); secrets in a secret manager; least-privilege access; rotate. Different values per environment, same code |
| **Observability** | **Logs + metrics + traces**. Levels DEBUG/INFO/WARN/ERROR/FATAL. JSON logs in prod. Alert → metrics → logs → trace |
| **Shutdown** | On SIGTERM/SIGINT: stop accepting → drain in-flight (≈30 s timeout) → close resources in **reverse** order → exit. SIGKILL can't be caught |

See [14](14-error-handling-and-fault-tolerance.md)–[17](17-graceful-shutdown.md).

---

## Performance and scaling

- Measure **P50/P90/P99**, not averages. Keep utilization at ~60–80%.
- **Never guess the bottleneck**: tracing, profiling, `EXPLAIN ANALYZE`.
- Order of attack: fix queries/indexes/N+1 → pooling → caching → async work → vertical scale → horizontal scale (stateless servers + load balancer) → read replicas → sharding.
- **Load balancing**: round robin, weighted, least connections; **health checks** remove dead instances.
- **Replication** (copies, scales reads, replication lag) ≠ **sharding** (splits data).
- **CDN/edge** for latency and DDoS; **microservices** for team scaling, not default; **serverless** for spiky/event work (cold starts).
- Principles: simple first, scale for today, observability from day one.

See [19](19-scaling-and-performance-part-1.md), [20](20-scaling-and-performance-part-2.md).

---

## Concurrency

| | I/O-bound | CPU-bound |
|---|---|---|
| Examples | DB, network, files | image/video, hashing, ML |
| Best | **concurrency** (event loop, goroutines, `async`) | **parallelism** (threads across cores) |

Event loop: never block it. `await` = callback via state machine. Shared state ⇒ race conditions (locks, channels, DB atomic updates). See [21](21-concurrency-and-parallelism.md).

---

## Files and real time

**Object storage (S3-style)**

- Flat key→bytes; immutable; "folders" are prefix illusions. **Generate your own keys** (random first, ownership in key).
- Uploads: small via server **streaming** with size limit; large via **pre-signed POST with policy** (never plain PUT) + two-phase `pending → ready` flow + cleanup.
- Multipart: part size = `max(16–32 MB, size/10000)`, 4–6 parallel parts, resume via `ListParts`, **abort-incomplete lifecycle rule (7 days)**, don't trust multipart ETag.
- Downloads: never proxy. CDN + private bucket; pre-signed GET (mind cache hit rate); **range requests** (`206`).
- Durability ≠ backup (versioning + cross-account replication). Never `LIST` in the request path.

**Real time**

| Technique | Direction | Notes |
|---|---|---|
| Polling | pull | cost scales with viewers |
| Long polling | pull-ish | gap between requests |
| **SSE** | server→client | HTTP, auto-reconnect, `Last-Event-ID`; LLM streaming |
| **WebSocket** | both | HTTP upgrade → `101` → frames; ping/pong heartbeat |

Scale across instances with **pub/sub** (Redis Pub/Sub is at-most-once; streams/Kafka for at-least-once). Sequence IDs for catch-up. Mind fan-out and reconnection storms.

See [22](22-object-storage.md), [23](23-real-time-backends.md).

---

## Webhooks

**Receive:** read raw bytes → verify **HMAC** (constant-time) → check signed timestamp (replay) → store **unique delivery ID in the same transaction** as side effects → return **2xx fast** → process in a queue. Expect duplicates and out-of-order events (re-fetch state). One URL per provider.

**Send:** **outbox** table in the same transaction; dispatcher signs, 10 s timeout, exponential backoff + jitter, respect `Retry-After`, disable after repeated failures, **SSRF guard** (resolve, block private ranges, no redirects).

**Don't** use webhooks to sync data; use the provider's ordered event log. See [27](27-webhooks.md).

---

## Testing

- Think **small / medium / large** by what a test may touch. Test the public behaviour; prefer **state over interaction** verification.
- Doubles: dummy, stub, spy, mock, fake. **Don't mock what you don't own** (wrap + contract test).
- DI + **functional core / imperative shell**. Use a **real DB** (Testcontainers, transaction-per-test or template DB). Never `sleep`: wait for conditions or inject a clock.
- **TDD**: red → green → refactor (best when the answer is known). **Flaky tests**: quarantine with owner/date/reason or delete.
- **Coverage** shows what ran, not what was checked. **Mutation testing** tests your tests. Cyclomatic vs cognitive complexity are flags.

See [24](24-testing-for-backend-engineers.md).

---

## 12-Factor App

I Codebase · II Dependencies · III Config (env) · IV Backing services · V Build/Release/Run · VI Stateless processes · VII Port binding · VIII Concurrency (process model) · IX Disposability · X Dev/prod parity · XI Logs → stdout · XII Admin one-offs. See [25](25-twelve-factor-app.md).

---

## DevOps

| Topic | Remember |
|---|---|
| **DORA** | lead time, deploy frequency, change-failure rate, recovery time. Speed and stability go together |
| **CI** | merge daily, trunk-based, feature flags |
| **Build** | reproducible, pinned, commit hash stamped. Pipeline: cheap/fast stages first |
| **Cloud creds in CI** | **OIDC**, not stored secrets |
| **Container** | process + **namespaces** (view) + **cgroups** (limits) + **OverlayFS** (layers) |
| **Image** | layers + config + manifest (hash). Order Dockerfile for cache; multi-stage; non-root; pinned; scanned |
| **Run it** | systemd + Nginx/Caddy + ACME (Let's Encrypt). Multi-host: Kamal. Immutable infra |
| **Kubernetes** | declarative; Pod → ReplicaSet → Deployment; Service; Ingress; requests/limits; probes (readiness/liveness/startup); HPA |
| **Rollouts** | recreate (downtime), rolling, blue-green, canary, progressive |
| **GitOps** | Git = desired state; Argo CD/Flux reconcile |
| **IaC** | Terraform/OpenTofu (plan/apply, state, drift), Ansible, Packer |
| **Ops** | SLI → SLO → SLA; error budget; on-call; incident roles; blameless postmortems; chaos engineering; pin + scan dependencies |

See [28](28-devops-for-backend-engineers.md).

---

## Handy numbers and rules

| | |
|---|---|
| Availability 99.9% | ~43 min downtime / month |
| Availability 99.99% | ~52 min / year |
| Typical utilization target | 60–80% |
| Shutdown timeout | ~30 s |
| Webhook response deadlines | GitHub 10 s, Shopify 5 s, Slack/MS Graph 3 s |
| Replay window (Stripe) | 5 min |
| Multipart | ≤ 10,000 parts; single PUT ≤ 5 GB |
| ACME cert validity | 90 days (renew ~60) |
| WebSocket per-connection memory (Go demo) | ~9.7 KB |
| K8s rolling update defaults | `maxSurge` 25%, `maxUnavailable` 25% |
