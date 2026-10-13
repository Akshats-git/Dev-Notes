# 19 — Backend Scaling and Performance Engineering: Part 1

**Video:** [#21.1 Backend Scaling and Performance Engineering: Part-1](https://www.youtube.com/watch?v=z7kt_p44rjs) (1h 48m)

The goal isn't a list of tricks. It is to build **intuition**: how systems behave under load, where bottlenecks hide, and how to think clearly when things fall apart. This part covers **how to measure performance**, the **mental models** (utilization, queues, bottlenecks), **database-level** problems (N+1, indexes, connection pooling), **caching** in a performance context, and **vertical vs horizontal scaling**. Part 2 continues with load balancing and the rest of scaling ([20](20-scaling-and-performance-part-2.md)).

---

## Contents

1. [What "fast" means: latency](#what-fast-means-latency)
2. [Percentiles: P50, P90, P99](#percentiles-p50-p90-p99)
3. [Throughput](#throughput)
4. [Utilization and the latency curve](#utilization-and-the-latency-curve)
5. [Bottlenecks: measure, don't guess](#bottlenecks-measure-dont-guess)
6. [Profiling and distributed tracing](#profiling-and-distributed-tracing)
7. [Database problem 1: N+1 queries](#database-problem-1-n1-queries)
8. [Database problem 2: indexes](#database-problem-2-indexes)
9. [Database problem 3: connection cost and pooling](#database-problem-3-connection-cost-and-pooling)
10. [Caching for performance](#caching-for-performance)
11. [Vertical vs horizontal scaling](#vertical-vs-horizontal-scaling)
12. [Summary](#summary)
13. [Interview questions](#interview-questions)

---

## What "fast" means: latency

[1:30](https://www.youtube.com/watch?v=z7kt_p44rjs&t=90s)

A user clicks a button. The browser sends a request over the internet → server processes → maybe queries the DB and calls other APIs (e.g. an email service) → returns JSON → the browser parses and renders. The total time from click to render is **latency**: what users feel as "slow".

Latency is **not a single number**. It varies per request:

- One request hit a **cache/CDN** → 50 ms.
- Another needed a DB query, or arrived when the server was already handling 50 concurrent requests → 200 ms.

### Why averages mislead

Say 1,000 requests average 100 ms. But 99% finished in under 50 ms while **1% took 5 seconds**. The average hides this. At 1M requests/day, **10,000 users waited 5 seconds**. The average says "fine"; those users disagree.

---

## Percentiles: P50, P90, P99

[10:24](https://www.youtube.com/watch?v=z7kt_p44rjs&t=624s)

| Term | Meaning (example) |
|---|---|
| **P50** (median) | P50 = 400 ms → half of requests are faster than 400 ms |
| **P90** | P90 = 900 ms → 10% of requests are slower than 900 ms |
| **P99** | P99 = 2 s → 99% of requests are faster than 2 s, **1% are slower** |

Backend teams focus on **P95/P99**. The slowest requests are usually the **most complex ones**: heavy queries, external service calls, payment workflows. Those users are often the **most valuable** (the person making a purchase hits more complex logic than someone browsing).

---

## Throughput

[14:33](https://www.youtube.com/watch?v=z7kt_p44rjs&t=873s)

> **Throughput** = how many requests the system handles per time unit (requests/second or per minute).

Latency and throughput interact: at 10 req/s your latency may be 150 ms, but at 1,000 req/s it may jump to 2 s. Together they answer questions like: *Can we survive Black Friday? A sudden email campaign? A podcast mention? How many concurrent users before we need more resources?*

As throughput rises, latency rises slowly at first, then sharply.

---

## Utilization and the latency curve

[17:37](https://www.youtube.com/watch?v=z7kt_p44rjs&t=1057s)

**Ice-cream shop analogy:**

- Sunday evening, shop empty: you're served instantly (low utilization, low latency).
- Tuesday lunch, long line: the worker is just as fast (2 min/ice cream) but **you wait for everyone ahead of you**. Perceived latency is much higher even though the *service* time is unchanged.

Servers are the same: low utilization → instant. Higher utilization → requests **queue**, and wait time grows.

> **Utilization** = percentage of the system's capacity in use. 0% = idle, 100% = maxed out.

The relationship is **not linear**. Latency grows slowly, then **exponentially** as utilization approaches 100%.

**Highway analogy:** at 50% capacity traffic flows; at 80% you notice slowdowns; at 90% it's unpredictable (one brake causes a ripple of honking); at 100% nobody moves.

**Consequences:**

- **Never run at 100%.** Real production systems typically run at **60–80% utilization**, keeping headroom.
- **Traffic comes in bursts**, not a steady metronome. Even if average load is 40%, a burst can push past 100% without headroom.

---

## Bottlenecks: measure, don't guess

[28:28](https://www.youtube.com/watch?v=z7kt_p44rjs&t=1708s)

When the system is "slow", something *specific* is slow: the **bottleneck**. Engineers often skip finding it and jump to by-the-book fixes: add caching, upgrade Postgres 16 → 18, add servers. Sometimes that helps. Often you spend weeks solving a problem you don't have.

**Story:** a `GET /products/:id` is slow. Assume the DB, so spend a week adding Redis caching. Still slow. Finally add timing around each step:

| Step | Time |
|---|---|
| DB query | 10 ms |
| Redis cache | 5 ms |
| **Synchronous call to a remote logging service** | **~500 ms** |

The culprit was a **blocking log call**, not the database. Other hidden culprits: JSON/XML serialization, or a huge response payload where network transfer dominates.

> **Rule: never guess, always measure.**

---

## Profiling and distributed tracing

[36:45](https://www.youtube.com/watch?v=z7kt_p44rjs&t=2205s)

**Profiling** = measuring where your app spends its time. A profiler attaches to the running app and samples what executes. Output is overwhelming, so a **flame graph** shows the call stack over time: wider boxes = functions taking more time; callers stack on top of callees.

The bottleneck is rarely where you expect: you assume complex business logic, but it's JSON serialization. You assume DB queries, but it's an external API call inside a loop.

**Limitation:** profilers are best for **CPU-bound** work. Most backend slowness is **I/O-bound** (DB queries, file handling, serialization, external APIs), which profilers capture poorly.

**Distributed tracing** (an observability tool) follows a request through your system and records timing at each component: *"This `/products/5` request spent 2 ms in business logic and 800 ms in the DB query."* Now you know where to look. (See [16](16-logging-monitoring-observability.md).)

---

## Database problem 1: N+1 queries

[43:54](https://www.youtube.com/watch?v=z7kt_p44rjs&t=2634s)

Databases are the first suspect because they do hard work: durable storage, consistency, concurrency, locking, complex queries over millions of rows.

**Example:** a blog homepage shows 20 posts, each with the author's name. The list API returns posts without author info, so you do **1 query for the posts + N queries for each author** = 21 queries. This is the **N+1 problem**. The number of queries grows linearly with the items shown.

Why that's bad: each query has **overhead**: network round trip, (possibly) TCP setup, parse → plan → execute → return. At 5 ms per query, 1,000 items = 1,000 queries ≈ **5 seconds** of loader.

**Fix: fetch in bulk.** Collect the author IDs, run one query for all authors (or a `JOIN`). It's **2 queries** whether it's 20 or 1,000 posts.

It happens on the **server ↔ database** side (your server is the "frontend" of the DB). It's often caused by **ORMs**, which make code like this look innocent:

```ts
const posts = await db.select().from(posts);
for (const post of posts) {
  post.author = await db.select().from(users).where(eq(users.id, post.authorId)); // N queries!
}
```

Most ORMs have bulk-fetch features: Django `select_related` / `prefetch_related`, Rails `includes`, Prisma/Drizzle joins and relation loading. And modern ORMs can **print the actual SQL**: check it during development.

---

## Database problem 2: indexes

[53:35](https://www.youtube.com/watch?v=z7kt_p44rjs&t=3215s)

(Intro in [10](10-databases-with-postgres.md).) Library analogy: a million books, no catalog. To find all books by an author you walk the whole library: a **sequential scan / full table scan**. With a catalog (index) organised by author with locations, you go straight there.

An index is typically a **B-tree**: a sorted copy of a column's values with pointers to rows. Example: 1M rows, a lookup by author takes ~4 s without an index, ~40 ms with one.

### Costs of indexes

1. **Space** for the structure.
2. **Every `INSERT` / `UPDATE` / `DELETE` must update every index.** Index every column and your writes slow down significantly.

### What to do

- **Primary keys are indexed automatically** (in Postgres).
- Index obvious ones up front (e.g. `author_id`, a foreign key you will certainly search by).
- For non-obvious ones, **wait**. Use logs and traces to find which endpoints are hit most and which queries are slow, then index.
- Be cautious, not generous.

### Composite and covering indexes

| | Meaning |
|---|---|
| **Composite index** | Covers multiple columns, e.g. `(user_id, created_at)`. **Order matters.** It helps queries on `user_id` alone or on both, but **not on `created_at` alone** |
| **Covering index** | Includes all columns a query needs (e.g. department `name` + `id`), so the DB answers from the index without touching the table. Bigger index, so weigh it |

### How to know what to index: `EXPLAIN ANALYZE`

[1:06:00](https://www.youtube.com/watch?v=z7kt_p44rjs&t=3960s)

Prefix a query with `EXPLAIN ANALYZE` to see the query plan: which tables, joins and indexes it used. If you see a **sequential scan** on a large table, add an index and re-run. You should now see an **index scan** and a faster query.

Workflow: **distributed tracing → find slow query → `EXPLAIN ANALYZE` → add index → verify.**

---

## Database problem 3: connection cost and pooling

[1:08:04](https://www.youtube.com/watch?v=z7kt_p44rjs&t=4084s)

Connections are cheap locally and expensive at scale. A new connection involves:

1. A **TCP three-way handshake**
2. Authentication
3. Encryption (TLS) negotiation
4. Session state setup
5. The database **allocating memory** (MBs) for it

If you open and close a connection per query you pay this latency every time, and a traffic spike can **exhaust the database's connection limit** (a Postgres instance might allow only a few hundred, e.g. 400–600).

### Connection pooling

A **pool** is a middleman holding a set of open connections:

```
server ──borrow──► POOL ──(reused open connection)──► database
       ◄─return───
```

The pool keeps connections open for reuse, so you skip the setup cost and cap the total connections.

### Internal vs external pooling

| | Internal | External |
|---|---|---|
| What | Pool inside each app instance (the driver provides it) | A separate pooler between all instances and the DB, e.g. **PgBouncer** (lightweight, open source, for Postgres) |
| Problem | With horizontal scaling, 3 instances × 150 connections = **450** > DB limit of 300 → crash on a spike | One pool of e.g. 250–300 serves **all** instances, so the DB limit can't be exceeded |

In production, **prefer an external pooler**.

---

## Caching for performance

[1:17:18](https://www.youtube.com/watch?v=z7kt_p44rjs&t=4638s)

(Details in [11](11-caching.md).) Store the result of an expensive operation so you can skip it: e.g. a complex query taking 800 ms is served from Redis in ~50 ms.

### Cache invalidation

> "There are only two hard things in programming: cache invalidation and naming things."

It's hard because caches touch many layers (servers, CDNs, browsers, reverse proxies). Two techniques:

| | How | Trade-off |
|---|---|---|
| **Time-based expiration (TTL)** | Cache for e.g. 5–10 minutes | Hard to pick the optimal duration; depends on access patterns per endpoint |
| **Event-based invalidation** | When the underlying data changes (e.g. profile update), delete the cache entry in the same handler | Exact, but you must remember to invalidate in **every** place that writes. Miss one and you serve stale data |

### Where the cache lives

| | |
|---|---|
| **Local cache** (a map in the server's memory) | Near-instant, but with 10 instances you have 10 separate caches → **inconsistency** |
| **Distributed cache** (Redis, Memcached, Valkey) | One shared cache, consistent, but each access is a **network round trip** |
| **Tiered caching** | A small local cache for the hottest data in front of the distributed cache |

### Caching patterns

[1:26:28](https://www.youtube.com/watch?v=z7kt_p44rjs&t=5188s)

| Pattern | How | Trade-off |
|---|---|---|
| **Cache-aside (lazy loading)**, the most common | Check cache → miss → query DB → store in cache → return. On update, **delete** the cache entry | Simple and intuitive |
| **Write-through** | On write, update **both** DB and cache | Never a miss, but writes take longer |
| **Write-behind** | On write, update the cache and return; update the DB **asynchronously** | Fastest writes, but if the DB write fails, cache and DB are **inconsistent** |

### Cache hit rate

The % of requests served from cache. ~90% is very good. 20% means something's wrong with the strategy. Factors: **TTL** (long = more hits but staleness), **cache size**, and **how well you understand access patterns**.

---

## Vertical vs horizontal scaling

[1:34:12](https://www.youtube.com/watch?v=z7kt_p44rjs&t=2052s)

As traffic grows, servers run out of capacity. Two approaches:

### Vertical scaling (scale up)

Replace the server with a **bigger one**: more CPU cores, more RAM, more/faster storage (NVMe SSDs), faster network card.

| Pros | Cons |
|---|---|
| **Simple**: architecture and code don't change | **Hard ceiling**: there's a biggest machine your provider offers |
| ~2× CPUs ≈ 2× requests; 2× RAM ≈ 2× local cache | **Single point of failure** (standby/failover helps but doesn't remove the risk) |
| One big server often costs less than two half-size ones; less ops overhead (security, backups, load balancers, distributed state) | **No geographic distribution** (e.g. a US server serving a 40% India user base) |

### Horizontal scaling (scale out)

Add **more instances** of the server that work together.

| Pros | Cons |
|---|---|
| 5 servers at 1,000 req/s each → 5,000 req/s | **Complexity**, which vertical scaling avoids |
| No hard limit: keep adding | Need a **load balancer**, and a **balancing algorithm** |
| **Redundancy**: if one dies, the others take the traffic | How do instances stay **in sync** (user updates a name on server 1; server 2 must know)? |
| **Geographic distribution**: route users to the nearest instance | What happens when the **network between servers fails**, and they make conflicting decisions? |
| | How do you detect a dead instance, stop routing to it, and resume when it's back? |

Every question has an answer, but each answer involves **trade-offs**. Distributed systems don't remove problems: they **trade one set of problems for another**, hopefully a more favourable one. (Continued in [Part 2](20-scaling-and-performance-part-2.md).)

---

## Summary

1. **Latency** = time from click to result. It varies, so **averages mislead**. Use **percentiles (P50, P90, P99)**, and watch the tail.
2. **Throughput** = requests per time. Together with latency it tells you whether you'll survive a traffic spike.
3. Latency grows **exponentially** near 100% **utilization**. Keep **60–80%** and headroom, since traffic comes in **bursts**.
4. **Find the bottleneck by measuring**, not guessing: **profiling / flame graphs** (CPU-bound), **distributed tracing** (I/O-bound).
5. Database fixes: avoid **N+1** (bulk fetch / joins), add the **right indexes** (verify with `EXPLAIN ANALYZE`; indexes slow writes), use **connection pooling** (prefer an **external** pooler like PgBouncer when scaling out).
6. **Caching**: pick a pattern (**cache-aside / write-through / write-behind**), handle **invalidation** (TTL vs events), choose **local vs distributed vs tiered**, monitor **hit rate**.
7. **Vertical scaling** is simple but capped and a single point of failure. **Horizontal scaling** is unlimited and redundant but adds distributed-systems complexity.

## Interview questions

**Q: Why are averages a poor latency metric?**
They hide the tail. 1% of users could be waiting seconds while the average looks fine. Use percentiles (P95/P99).

**Q: What does P99 = 2 s mean?**
99% of requests complete within 2 s. 1% take longer.

**Q: Why does latency spike near 100% utilization?**
Requests queue behind each other, and queueing delay grows non-linearly as capacity is exhausted. Plus bursts push utilization over 100%.

**Q: What utilization do production systems target?**
Roughly 60–80%, leaving headroom for bursts.

**Q: A request is slow. What do you do first?**
Measure. Use tracing to see which component takes the time, rather than assuming (e.g. adding a cache to a fast query).

**Q: What is the N+1 problem and how do you fix it?**
One query for N items plus N queries for related data. Fix with a join or a bulk `WHERE id IN (...)` fetch, or the ORM's eager-loading feature.

**Q: Downsides of indexes?**
Extra storage and slower writes, since all indexes must be updated on each insert/update/delete.

**Q: What does the column order in a composite index mean?**
The index helps queries that use a leading prefix of the columns. `(user_id, created_at)` serves `user_id` and `user_id + created_at`, but not `created_at` alone.

**Q: How do you check whether a query uses an index?**
`EXPLAIN ANALYZE`: look for index scan vs sequential scan.

**Q: Why use connection pooling, and internal vs external?**
Reuse open connections to avoid setup cost and cap total connections. Internal pools multiply with each app instance, so an external pooler (PgBouncer) caps the total for the whole fleet.

**Q: Cache-aside vs write-through vs write-behind?**
Cache-aside: populate on miss, invalidate on write. Write-through: write DB and cache together. Write-behind: write the cache first, persist to the DB asynchronously (risk of inconsistency).

**Q: TTL vs event-based invalidation?**
TTL expires entries automatically (simple, but hard to tune). Event-based deletes entries when data changes (exact, but requires discipline at every write path).

**Q: Vertical vs horizontal scaling?**
Vertical: bigger machine, simple, but capped and a single point of failure. Horizontal: more machines, scalable and redundant, but needs load balancing and handling distributed state.
