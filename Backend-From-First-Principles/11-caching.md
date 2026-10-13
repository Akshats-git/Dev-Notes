# 11 — Caching: The Secret Behind It All

**Video:** [#13 Caching, the secret behind it all](https://www.youtube.com/watch?v=estH64OkwxU) (1h 4m)

What caching is, why every high-performance system relies on it (Google, Netflix, Twitter/X), the three levels of caching a backend engineer meets (network, hardware, software), how Redis-style in-memory caches work, caching strategies, eviction policies, and the use cases you'll actually build.

---

## Contents

1. [What caching is](#what-caching-is)
2. [Three real-world examples](#three-real-world-examples)
3. [The pattern](#the-pattern)
4. [Levels of caching](#levels-of-caching)
5. [Network caching: CDN](#network-caching-cdn)
6. [Network caching: DNS](#network-caching-dns)
7. [Hardware caching](#hardware-caching)
8. [Software caching: in-memory key-value stores](#software-caching-in-memory-key-value-stores)
9. [Caching strategies](#caching-strategies)
10. [Eviction policies](#eviction-policies)
11. [Backend use cases](#backend-use-cases)
12. [Summary](#summary)
13. [Interview questions](#interview-questions)

---

## What caching is

[0:00](https://www.youtube.com/watch?v=estH64OkwxU&t=0s)

> **Caching** decreases the time and effort needed to perform some work.

Technically: keep a **subset** of some primary data in a location that is **faster and cheaper to access**. The subset is chosen by how often it's used and how likely it is to be needed next.

---

## Three real-world examples

### 1. Google search

[1:33](https://www.youtube.com/watch?v=estH64OkwxU&t=93s)

A search runs crawling, indexing and ranking over billions of pages: **computationally expensive**. "What is the weather today" is searched millions of times a day. Without caching, every query would redo all of it, giving high latency and huge server load.

Google uses a **distributed in-memory cache** (servers spread across the world) that stores results.

- Query arrives → check cache.
- **Cache hit** (found) → return instantly.
- **Cache miss** → run the full pipeline, **store the result**, return it. The next identical query is a hit.

### 2. Netflix

[5:42](https://www.youtube.com/watch?v=estH64OkwxU&t=342s)

Each title is encoded into multiple resolutions (1080p, 720p, 480p...) for different devices and bandwidths. That's terabytes per title. Netflix serves it using a **CDN (Content Delivery Network)**: its origin servers hold everything, but **edge locations** around the world cache popular content close to viewers, so buffering and latency stay low and origin load drops.

It does *not* cache everything everywhere. Cache memory is expensive. It analyses what's popular in each region and caches that subset. Vercel's edge network works the same way for web assets.

### 3. Twitter / X trending topics

[12:23](https://www.youtube.com/watch?v=estH64OkwxU&t=743s)

Finding trending topics means analysing millions of tweets with ML: a very expensive computation. If every user opening the trending tab triggered it, the servers would crash in seconds.

Trends don't change by the second, so every few minutes X computes them once and stores the result in an in-memory key-value store (like Redis). Users read the cached result, so it loads instantly.

---

## The pattern

[14:27](https://www.youtube.com/watch?v=estH64OkwxU&t=867s)

Caching shows up in two situations:

1. You want to avoid **heavy computation** repeated again and again.
2. You want to avoid sending **large amounts of data** to many users.

And a cache holds only a **subset** of the data. Caching everything defeats the point, since cache memory costs more than disk.

---

## Levels of caching

[17:29](https://www.youtube.com/watch?v=estH64OkwxU&t=1049s)

| Level | Examples |
|---|---|
| **Network** | CDN, DNS |
| **Hardware** | CPU L1/L2/L3 caches, RAM |
| **Software** | Redis, Memcached, AWS ElastiCache (still backed by hardware caches, but you interact via a library) |

---

## Network caching: CDN

[19:32](https://www.youtube.com/watch?v=estH64OkwxU&t=1172s)

**Goal:** cache content on servers **geographically close to users** (edge nodes / edge servers).

```
1. User types a URL for an image/video/page.
2. Browser sends a DNS query to resolve the domain.
3. The CDN's DNS routes it to the NEAREST PoP (Point of Presence =
   a region with a cluster of edge servers), based on location and network conditions.
4. The edge server checks its cache:
     HIT  → serve immediately.
     MISS → fetch from the ORIGIN server, cache it, serve it.
5. Entries expire via TTL (time to live) so stale content gets refreshed.
```

DNS is central to a CDN's routing. A user with a bad connection can even be routed to a PoP that holds lower-quality versions.

---

## Network caching: DNS

[26:13](https://www.youtube.com/watch?v=estH64OkwxU&t=1573s)

DNS resolution is **recursive** and expensive, so DNS caches at several layers:

```
Browser types example.com
   │
   ▼  1. Browser's DNS cache?       → hit: done
   ▼  2. OS-level DNS cache?        → hit: done
   ▼  3. Recursive resolver          (from your ISP, or public: Google DNS, Cloudflare)
          checks its own cache?     → hit: done
          miss ↓
          Root server  → "ask the .com TLD server"
          TLD server   → "ask the authoritative name server for example.com"
          Authoritative name server → returns the IP
   ◄──  result cached at each layer on the way back
```

Each layer caches, so the full recursion is rarely repeated. (Some authoritative servers cache too.)

---

## Hardware caching

[34:54](https://www.youtube.com/watch?v=estH64OkwxU&t=2094s)

```
CPU  →  L1 cache  →  L2 cache  →  L3 cache (shared between cores)  →  RAM (main memory)  →  Disk / network storage
fastest, smallest                                                                         slowest, largest
```

CPUs cache frequently/soon-to-be-used data. Example: iterating an array sequentially is fast because the CPU predictively loads neighbouring elements into cache.

### Why RAM is fast

- **Random access memory**: any address is reached directly with an electrical signal, in roughly constant time regardless of direction.
- A hard disk is **mechanical** (a head moves over a platter).

### But RAM has trade-offs

| | RAM | Disk |
|---|---|---|
| Speed | very fast | slower |
| Capacity | limited | large |
| Persistence | **volatile**: lost when power goes | persistent |

So RAM can't replace disk. In-memory cache systems use disk behind the scenes for persistence (reloading data at startup), but reads and writes happen in memory.

---

## Software caching: in-memory key-value stores

[41:04](https://www.youtube.com/watch?v=estH64OkwxU&t=2464s)

**Redis**, **Memcached**, **AWS ElastiCache** (and **Valkey**, the open-source Redis alternative, mentioned at the end).

Why "**in-memory key-value NoSQL database**":

- **In-memory**: data lives in RAM, not disk, so access is very fast.
- **Key-value**: no tables, rows or strict schema. You store a **value** (string, number, JSON, list...) under a **key**. Different tools offer different data types.
- **NoSQL**: no strict SQL schema, and no joins or aggregations. Usage is simple: `set(key, value)`, `get(key)`. In Node you'd just use a client library.

---

## Caching strategies

[42:36](https://www.youtube.com/watch?v=estH64OkwxU&t=2556s)

### Lazy caching (cache-aside)

```
read(key):
    value = cache.get(key)
    if value exists:        # HIT
        return value
    value = database.read(key)   # MISS
    cache.set(key, value)        # store for next time
    return value
```

Called "lazy" because you only cache data **when someone actually asks for it**, not by prediction.

### Write-through

On every write (POST / PUT / PATCH), update **both** the database and the cache in the same operation.

| | |
|---|---|
| **Pro** | The cache is always fresh. Never serves stale data |
| **Con** | Every write has extra overhead: two updates |

(The roadmap video also lists write-behind / read-through as variants.)

---

## Eviction policies

[45:09](https://www.youtube.com/watch?v=estH64OkwxU&t=2709s)

RAM is limited. When the cache is full and a new key arrives, something has to go: the **eviction policy** decides what.

| Policy | Evicts | Example |
|---|---|---|
| **No eviction** | nothing. Inserts just **error** when memory is full | |
| **LRU** (Least Recently Used) | the key not accessed for the longest time | Keys 1–3 accessed today, key 4 yesterday → evict 4 |
| **LFU** (Least Frequently Used) | the key accessed the fewest times | Key 1 used 5 times, others 10/6/23 → evict 1 |
| **TTL-based** | the key closest to expiring | the lowest remaining TTL goes first |

(FIFO was in the roadmap list too.) You can also set a **TTL on each key** so entries expire automatically, independent of memory pressure.

---

## Backend use cases

[51:20](https://www.youtube.com/watch?v=estH64OkwxU&t=3080s)

| Use case | How caching helps |
|---|---|
| **Database query results** | A query with many joins/aggregations over millions of rows, called often (landing page, dashboard). Cache the result for e.g. 1 hour. Invalidate or update the cache on changes. Amazon caches product details/prices so a sale doesn't hammer the DB. Social platforms cache celebrity profiles (read-heavy, rarely changing). |
| **Session storage** | After login, the session token/data is in Redis, since every request needs to look it up. Faster than a DB read and lowers DB load. |
| **External API responses** | Weather API (not real-time-critical): cache for an hour. Avoids hitting rate limits and reduces billing. |
| **Rate limiting** | Key = client IP (from the `X-Forwarded-For` header set by the reverse proxy), value = request counter for the current window. Over the limit (e.g. 50 req/min) → **`429`**. In Redis, not a relational DB, because it's checked on **every request**: it needs low latency and shouldn't flood the DB. |

> Rule: caching shines for **read-heavy** data that changes **infrequently**.

---

## Summary

1. **Caching** = keep a hot **subset** of data somewhere faster/cheaper to access.
2. Use it to avoid **heavy computation** and **moving lots of data**.
3. **Hit** = found in cache. **Miss** = not found, so go to the source and store it.
4. Levels: **network** (CDN, DNS), **hardware** (L1/L2/L3, RAM), **software** (Redis, Memcached).
5. **CDN**: edge servers/PoPs near users, DNS routes you to the closest, TTL expires entries.
6. **DNS** caches at browser → OS → resolver → authoritative levels.
7. RAM is fast but small and volatile, so caches are in-memory **key-value** stores.
8. Strategies: **lazy / cache-aside** and **write-through**.
9. Eviction: **no-eviction, LRU, LFU, TTL**.
10. Typical uses: DB query results, sessions, external API results, rate limiting.

## Interview questions

**Q: What is caching and why use it?**
Storing a frequently used subset of data in faster storage to reduce latency, compute and load on the primary source.

**Q: Cache hit vs miss?**
Hit: the data is in the cache and returned immediately. Miss: it isn't, so the system goes to the primary store and (usually) populates the cache.

**Q: Why is Redis fast?**
It keeps data in RAM (random access, no mechanical seek) with a simple key-value model, avoiding disk I/O and query planning.

**Q: Cache-aside vs write-through?**
Cache-aside (lazy) populates on read misses. Write-through updates the cache on every write, so it's always fresh but writes cost more.

**Q: LRU vs LFU?**
LRU evicts the least recently accessed key. LFU evicts the least frequently accessed.

**Q: What does TTL do?**
Expires a key after a set time, so stale data is refreshed. It also bounds memory growth.

**Q: How does a CDN reduce latency?**
It serves cached copies from an edge server near the user (found via DNS routing to the nearest PoP) and only goes to the origin on a miss.

**Q: Why use Redis for rate limiting instead of Postgres?**
The counter is checked and updated on every request. Redis keeps latency low and avoids loading the primary database.

**Q: When is caching a bad idea?**
For rapidly changing or write-heavy data where stale reads are unacceptable, or when invalidation complexity outweighs the benefit.

**Q: Why not cache everything?**
Cache memory is more expensive and limited. Caching data that isn't re-read wastes it.
