# 20 — Backend Scaling and Performance Engineering: Part 2

**Video:** [#21.2 Backend Scaling and Performance Engineering: Part-2](https://www.youtube.com/watch?v=sOhAopEwjH4) (2h 18m)

Continues from [Part 1](19-scaling-and-performance-part-1.md) (which ended with horizontal scaling). This part covers **statelessness**, **load balancers**, **database scaling** (read replicas, sharding, distributed databases), **CDNs and edge computing**, **asynchronous processing**, **microservices vs monoliths**, **serverless**, and the closing rules of thumb. (The video was split in two because of a recording-software issue.)

---

## Contents

1. [Statelessness: the enabler of horizontal scaling](#statelessness-the-enabler-of-horizontal-scaling)
2. [Load balancers](#load-balancers)
3. [Health checks](#health-checks)
4. [Database scaling: read replicas](#database-scaling-read-replicas)
5. [Database scaling: sharding](#database-scaling-sharding)
6. [Distributed databases](#distributed-databases)
7. [CDNs](#cdns)
8. [Edge computing](#edge-computing)
9. [Asynchronous processing](#asynchronous-processing)
10. [Monoliths vs microservices](#monoliths-vs-microservices)
11. [Serverless](#serverless)
12. [Closing rules of thumb](#closing-rules-of-thumb)
13. [Summary](#summary)
14. [Interview questions](#interview-questions)

---

## Statelessness: the enabler of horizontal scaling

[0:31](https://www.youtube.com/watch?v=sOhAopEwjH4&t=31s)

> Horizontal scaling only works if **no server instance holds data that other instances can't access.**

It requires planning your whole stack (code and infra), unlike vertical scaling, where you just click a bigger machine. **It must not matter which instance receives a request.** Delete instance B and A + C + D should behave identically.

Any data that must persist lives **outside** the instances, somewhere all of them can reach.

| State | Wrong | Right |
|---|---|---|
| **Sessions** | Instance A authenticates the user and stores the session in its own memory. The next request lands on B, which has no session → `401`. A confusing UX | Store sessions in **Redis**, which all instances share |
| **Uploaded files** | Saved to instance A's local disk. The next request on instance C can't find them | Upload to **object storage** (S3, Cloudflare R2, on-prem equivalent) |
| **Database** | A SQLite file on one server | A centralised **Postgres / RDS** |
| **Cache** | A map in one instance's memory | Redis (or tiered, see [19](19-scaling-and-performance-part-1.md)) |

**Rule of thumb:** with horizontal scaling, no piece of information may live in only one instance. Externalise everything.

---

## Load balancers

[9:08](https://www.youtube.com/watch?v=sOhAopEwjH4&t=548s)

If you have many instances, **who decides where each request goes?** A **load balancer (LB)** sits in front. All client traffic goes to it, it picks an instance via some algorithm, forwards the request, and relays the response back over the same HTTP connection.

```
Clients ──► Load balancer ──┬──► Server A
                            ├──► Server B
                            └──► Server C
```

### Algorithms

| Algorithm | How it works | Good for / weakness |
|---|---|---|
| **Round robin** | Rotating order: A, B, C, A, B, C... | Similar requests and similar servers. **Blind**: if requests alternate cheap/expensive, one server might get all the expensive ones and crash |
| **Weighted round robin** | Same, but bigger servers get proportionally more (e.g. A has 8 GB/4 cores, B and C have 4 GB/2 cores → A gets 2× requests) | Unequal hardware. Still blind to request cost |
| **Least connections** | Send to the server with the **fewest active connections**. A heavy request keeps a connection open longer, so that server looks busy and gets fewer new requests | Mixed workloads. Smarter than round robin |
| **Weighted least connections** | Least connections + capacity weights | |
| **Least response time** | Prefer servers responding fastest | |
| **Resource-based** | Prefer servers with the lowest CPU/RAM use | |

Example of why round robin can hurt: request type 1 is a cheap DB read (~200–300 ms); type 2 calls an external service and writes to a big, heavily indexed table (~2 s). Order: 1, 2, 2, 1... Round robin has no idea and may keep sending the expensive ones to the same server.

---

## Health checks

[22:28](https://www.youtube.com/watch?v=sOhAopEwjH4&t=1348s)

What if server A crashes? With round robin, a third of requests keep going to a dead server and fail with `502/503`.

The LB sends a **test request** (e.g. a simple `GET /health`, every second) to every server alongside real traffic. A healthy server returns `200` immediately (cheap, no load).

- Non-200 (or no response) → the LB **blacklists** that instance and sends no user traffic to it.
- It **keeps probing**. When it returns `200` again, the LB puts it back in rotation.

(Cross-reference [14](14-error-handling-and-fault-tolerance.md) on writing meaningful health checks.)

---

## Database scaling: read replicas

[27:36](https://www.youtube.com/watch?v=sOhAopEwjH4&t=1656s)

Scaling the app tier is easy once state is externalised. The **database is the hard part**, because it is **stateful** and can't just be cloned: the data must stay consistent.

### Read replicas

```
                    writes (INSERT/UPDATE/DELETE)
App servers ───────────────────────────► PRIMARY (master)
     │                                       │ replication
     │  reads (SELECT)                       ▼
     └──────────────────────────────► Replica 1 (India)  Replica 2 (China)  Replica 3 (Japan)
```

- One **primary** takes **all writes**. Multiple **replicas** (read-only copies) serve **reads**.
- Roughly 70–90% of SaaS traffic is reads, so the primary goes from 100% of load to ~30%.
- Put replicas near users (India, China, Japan) for **lower latency** too.
- Modern databases (Postgres, MySQL) have native replication.

### The catch: consistency and replication lag

[34:18](https://www.youtube.com/watch?v=sOhAopEwjH4&t=2058s)

A user in India updates their name (the write goes to the **US primary**). The client then immediately refetches the profile (a read, routed to the **India replica**). The primary → replica transfer takes, say, **200 ms** (physics: undersea cables; "we can't beat physics"). The replica hasn't caught up, so the user sees the **old name**, right after "success". For something like invoices or payments it can be worse.

This is **replication lag**. Mitigations:

1. **Route reads that follow a write to the primary** (for that entity).
2. **Track the lag** and make reads wait until replication completes.
3. **Delay the refetch** in the frontend.

Every distributed design **trades consistency for something else**. Managed providers (AWS RDS, Cloud SQL) make replicas a few toggles and have robust solutions for these problems.

---

## Database scaling: sharding

[42:28](https://www.youtube.com/watch?v=sOhAopEwjH4&t=2548s)

**Sharding (partitioning)** = splitting one big table across **multiple physical database instances**.

Example: an `orders` table with billions of rows. Two problems:

1. **Query latency**: even with an index, billions of rows are slow.
2. **Capacity**: one instance can only serve so many requests.

Split it by a **shard key**. E.g. orders from Jan–Jun in database 1, Jul–Dec in database 2 (or one shard per month). Now each instance holds fewer rows (faster queries) and you have more instances (more throughput).

The hard part is **choosing the shard key**. In your backend's database-routing layer you decide which shard holds the data and send the query there.

> **Replication** = copies of the *same* data (read scaling). **Sharding** = *different* data on different instances (write and size scaling).

---

## Distributed databases

[48:06](https://www.youtube.com/watch?v=sOhAopEwjH4&t=2886s)

Modern trend: distributed databases that handle sharding, replication and distributed transactions automatically. Examples: **PlanetScale** (MySQL, built on Vitess), **Neon** (serverless Postgres, written in Rust), **CockroachDB**, **YugabyteDB**.

**Advice:** don't run your own replication/sharding/backup infrastructure unless you really know what you're doing. Pick a managed provider (RDS, Neon, PlanetScale...), get a connection URL, and plug it in. But **learn the concepts** so you can configure them: backup frequency, replica regions, sharding plan.

---

## CDNs

[51:11](https://www.youtube.com/watch?v=sOhAopEwjH4&t=3071s)

(More in [11](11-caching.md).) A CDN is caching at the **global** level.

**Physics:** light in fibre travels ~200,000 km/s. A request from **Tokyo to Virginia (US-East-1)**, ~20,000 km round trip, costs at least **~100 ms**, and no optimisation can beat that. Add deserialization, business logic, DB (50–100 ms), external APIs (~200 ms) and you're at **500–800 ms**.

**Fix:** put content on **edge nodes / PoPs (points of presence)** near users. Tokyo → a Tokyo PoP is ~100–200 km, so latency drops from ~100 ms to **~2–3 ms**.

**Benefits:**

1. **Lower latency.**
2. **Lower origin load.** Traffic is spread across nodes (~50% less on the primary), so you scale horizontally less often.
3. **Security**: a CDN like Cloudflare sits in front of your server and absorbs **DDoS** attacks. It detects abnormal traffic, challenges suspicious users with CAPTCHAs, and its huge network absorbs terabytes. Without it, a DDoS could crash your server or (with autoscaling) run up a **$50k/day** bill.

**What to cache:**

- **Static content**: JS/CSS/HTML bundles, images, videos, fonts (how SPAs and static sites are deployed).
- Some **API responses** (e.g. a rarely changing product catalogue).
- Invalidate by **purging**, typically via **tags** (e.g. tag all of a user's blog content, purge the tag when they publish).

---

## Edge computing

[1:04:39](https://www.youtube.com/watch?v=sOhAopEwjH4&t=3879s)

"Edge" originally meant CDN nodes at the edge of the network, placed with ISPs. **Edge computing** = running **code** at those nodes, not just serving static files.

**Use cases:**

- **Authentication checks.** Instead of a 100 ms round trip to the origin just to be told `401`, the edge node validates the session and rejects in 2–3 ms. The origin only sees legit requests.
- **Localisation.** The edge sees the user's region and `Accept-Language` (Japanese) and returns a localised page.
- Routing, validation, personalisation.

**Why not run everything at the edge?** Constraints. Edge nodes are ISP-based infrastructure with limited resources (e.g. ~1 GB RAM, 1 core). Cloudflare Workers use **V8 isolates**: no file system, no raw TCP, etc. They complement the origin servers but **can't replace** them.

---

## Asynchronous processing

[1:13:22](https://www.youtube.com/watch?v=sOhAopEwjH4&t=4402s)

(Full coverage in [12](12-task-queues-and-background-jobs.md).) A cheap, early win for **perceived latency**. Unlike horizontal scaling (adopted at a traffic threshold), async processing is usually adopted from the start.

**Example: invite a team member.** DB check + insert ≈ 100 ms, then calling the email provider ≈ 300 ms, so the user waits ~400 ms. Instead: after the DB insert (~100 ms), **push a "send invite email" task to a queue** and respond `200`. A worker sends the email. The user sees "invited, status pending" in 100 ms.

**Candidates:** emails, notifications, video processing (YouTube: after upload, tasks for thumbnails, HD encoding, subtitles), image resizing, **delete account** (deleting from 8 tables with millions of rows might take 8 s; respond immediately and let a worker do it).

**Tools:** Redis-backed queues (BullMQ for Node, Upstash for managed Redis), RabbitMQ, Kafka for event-driven setups. The consumer can be in the same code base or a separate deployable you scale independently.

**Rule:** a task is a good candidate when the user **doesn't expect to see its result instantly**.

---

## Monoliths vs microservices

[1:30:47](https://www.youtube.com/watch?v=sOhAopEwjH4&t=5447s)

**Monolith:** a single deployable unit. Auth, orders, notifications, payments, webhooks are modules in one code base, one process (or several copies of it). Simple to develop, test, deploy and refactor.

You can always scale a monolith horizontally. So why microservices?

> **Microservices are mainly about scaling your *team*, not your machines.**

| Problem with monoliths at large scale | Microservices answer |
|---|---|
| **Deployment dependency**: the payments team wants to ship, but unready notification changes sit on `main`. (Small teams avoid this through coordination, feature flags and discipline. It bites at hundreds of developers) | Independent deploys |
| **Uneven scaling**: payments need CPU/RAM, notifications don't, but you scale everything together | Scale services independently |
| **Tech stack**: markdown parsing needs the Node ecosystem; image processing is 10× faster in Go/Rust (500 ms vs 50 ms) | A different language per service |

### The costs

1. **Network**: function calls become network calls (gRPC/HTTP): latency, failures, retries, timeouts.
2. **Debugging**: one request touches 4 services' logs → need distributed tracing.
3. **Data consistency**: each service has its own DB → replication lag, consistency problems.

**Go microservices only when:** you have a **large team** (the speaker mentions **100–200+ developers**), clear service boundaries, genuinely different scaling needs, or different technology needs. Otherwise the complexity outweighs the benefits.

---

## Serverless

[1:44:36](https://www.youtube.com/watch?v=sOhAopEwjH4&t=6276s)

### What came before: servers

You rent a VM (EC2), install Ubuntu, Nginx/Docker, configure, maintain. You know exactly what you have (e.g. 4 GB RAM, 2 cores, 30 GB disk, 1 TB bandwidth). Problems:

- **Capacity planning:** you must predict traffic.
  - **Under-provision** → crashes, slowness, lost customers.
  - **Over-provision** → e.g. 32 GB used at 20% costs $5,000/week vs $500 for 8 GB.

### Autoscaling (and its limits)

Spin up new instances when CPU/memory crosses a threshold (e.g. 70%). But:

1. **Time**: boot OS + build/start app + register with the LB can take seconds to minutes. A sudden spike in 3–4 seconds beats it.
2. **Limits**: max too low → under-provisioned again; max too high → a huge bill (e.g. $100k in a day).
3. **Reactive, not proactive**: it only scales after you're already loaded.
4. **Always-on cost**: you pay for the minimum 24/7, even at zero requests.

For steady, predictable traffic (most SaaS), this model is fine.

### Serverless model

You don't manage machines or OS. You provide only **functions** and the **events** (routes) that trigger them. An **API gateway** maps routes to functions.

- **Compute:** no server is running. On the first request, the provider spins up an instance, runs your function, returns a response, and keeps the instance for a few seconds (or reuses a pool) before discarding it.
- **Pricing:** pay only for **actual execution time** (CPU/memory milliseconds), not 24/7.

### Trade-offs

| Issue | Detail |
|---|---|
| **Cold starts** | Spinning up a new instance takes time. Mitigations: periodic "warm-up" pings (but too many = paying 24/7 again), **Firecracker microVMs** (AWS Lambda), **V8 isolates** (Cloudflare Workers, ~0–5 ms), interpreted languages (JS/Python start faster than JVM) |
| **Limits** | e.g. Lambda max ~15 min. Long-running requests fail |
| **Stateless** | Machines are replaced constantly: no persistent TCP connections, no in-process state, WebSockets and DB connections need rethinking (serverless databases) |

**Don't use it for:** latency-critical apps (banking, payments), long-running work, WebSockets, apps needing lots of DB connections.
**Good fits:** video/image processing, event-triggered work (queue messages, file uploads, DB changes), spiky/unpredictable workloads.

The speaker's view: the industry is **over-hyped**. Powerful for specific cases, not a universal replacement for servers.

---

## Closing rules of thumb

[2:09:42](https://www.youtube.com/watch?v=sOhAopEwjH4&t=7782s)

1. **Start with the problem, and measure.** Use observability (traces, metrics, logs) and load tests to find the *specific* bottleneck (e.g. "missing index", "no sharding"). Otherwise it's **premature optimisation**, and the worst outcome is *fixing the wrong bottleneck while believing you fixed it*.
2. **Prefer simple solutions.** Big server before horizontal scaling; index before Redis cache; monolith before microservices; avoid Kubernetes unless you need it. Complexity has costs: every component can fail and must be monitored and operated. Accept complexity only when simplicity is insufficient.
3. **Scale for the problems you have.** You probably won't reach a million users. Build for your current scale plus headroom. Learn from **measuring your system**, not from Netflix/Google blog posts.
4. **Measure from day one.** The one exception to "keep it simple": production-grade observability from the start. You'll get early warnings and never have to guess.
5. **Mindset.** Like security, performance is ongoing learning. Your job isn't to predict every problem but to build systems that **handle problems gracefully** and to diagnose and fix them quickly.

---

## Summary

1. Horizontal scaling needs **stateless** servers. Move sessions, files, DB and cache **out** of instances (Redis, S3, central DB).
2. **Load balancers** distribute traffic: round robin, weighted, **least connections**, least response time, resource-based. **Health checks** remove dead instances.
3. **Database scaling:** **read replicas** (scale reads, cut latency; beware **replication lag**), **sharding** (split data by a shard key), **distributed/managed databases** (don't self-host).
4. **CDNs** cut latency (physics), offload the origin and absorb DDoS. **Edge computing** runs small logic (auth, localisation) near users but has tight constraints.
5. **Async processing** via queues improves perceived latency for non-instant work.
6. **Microservices** solve *team* scaling and independent deploys/scaling/tech, at a high complexity cost. Default to a monolith.
7. **Serverless** removes capacity planning and bills per execution, but has cold starts, limits and statelessness.
8. Measure first, keep it simple, scale for today, build observability early.

## Interview questions

**Q: Why must servers be stateless for horizontal scaling?**
Any instance may handle any request. State kept in one instance's memory/disk is invisible to the others, causing bugs like random 401s or missing files.

**Q: Round robin vs least connections?**
Round robin rotates blindly. Least connections sends to the server with the fewest active connections, accounting for varying request cost.

**Q: How does a load balancer detect a dead server?**
Periodic health-check requests. Non-200 or a timeout → remove from rotation, keep probing, restore when healthy.

**Q: What are read replicas and what problem do they introduce?**
Read-only copies of the primary DB that serve reads, reducing primary load and latency. Replication lag can cause stale reads right after a write.

**Q: How do you mitigate read-after-write inconsistency?**
Route follow-up reads to the primary, wait for replication, or delay the refetch.

**Q: Replication vs sharding?**
Replication copies the same data to multiple nodes (read scale). Sharding partitions different data across nodes (data volume and write scale).

**Q: What is a shard key and why is it hard?**
The attribute used to decide which shard a row lives in. A poor choice leads to hot spots, cross-shard queries and painful resharding.

**Q: What do CDNs give you beyond latency?**
Reduced origin load and DDoS protection.

**Q: What is edge computing and its limits?**
Running code on CDN edge nodes (auth checks, localisation). Limited by CPU/RAM and restricted runtimes (e.g. V8 isolates).

**Q: When would you choose microservices?**
Large teams needing independent deploys, services with very different scaling profiles, or different tech stacks. Not as a default.

**Q: What is a cold start?**
The delay when a serverless platform has to start a new instance for a request. Mitigated by lighter runtimes (V8 isolates, Firecracker), warm pools, or interpreted languages.

**Q: Downsides of autoscaling?**
Boot delay, min/max misconfiguration, reactive scaling, and always-on minimum cost.

**Q: Name the single most important performance principle from this video.**
Measure before optimising. Find the actual bottleneck. Otherwise you may fix the wrong thing.
