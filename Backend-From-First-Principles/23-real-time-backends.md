# 23 — Real-Time Backends

**Video:** [#25 Real-Time Backends](https://www.youtube.com/watch?v=wUQryt697cs) (52m)

Everything so far followed one shape: *the browser asks, the server answers.* But what if a task dragged on a board by one user should appear **live** on another user's screen? This lecture walks the evolution: **polling → long polling → Server-Sent Events → WebSockets**, then measures how many connections one machine can hold, and how to scale real-time across multiple instances with **pub/sub**.

---

## Contents

1. [The limitation: the server can't speak first](#the-limitation-the-server-cant-speak-first)
2. [Polling](#polling)
3. [Long polling](#long-polling)
4. [Server-Sent Events (SSE)](#server-sent-events-sse)
5. [WebSockets](#websockets)
6. [How many connections can one machine hold?](#how-many-connections-can-one-machine-hold)
7. [Multiple instances: pub/sub](#multiple-instances-pubsub)
8. [Reliability: catching up after disconnects](#reliability-catching-up-after-disconnects)
9. [Fan-out and reconnection storms](#fan-out-and-reconnection-storms)
10. [Not covered](#not-covered)
11. [Summary](#summary)
12. [Interview questions](#interview-questions)

---

## The limitation: the server can't speak first

[0:00](https://www.youtube.com/watch?v=wUQryt697cs&t=0s)

In HTTP, the **client initiates** and the server only replies (a point from the HTTP video). If the server has news the browser hasn't asked for, there's no built-in way to deliver it.

The running example: a task-management board. User A moves a task from *To-do* to *In progress*, and user B must see it move live.

---

## Polling

[2:35](https://www.youtube.com/watch?v=wUQryt697cs&t=155s)

The intuitive fix: the client asks repeatedly. Every 3 seconds: *"Has board 412 changed?"* Usually the answer is no.

Demo: two browsers each poll every 3 s. The counters show most requests are "unchanged", wasted work.

Fine for internal tools with few users. As a user-facing product it has problems:

| Problem | Detail |
|---|---|
| **Delay** | Average latency = **half the interval** (3 s → ~1.5 s) |
| **"Poll faster" backfires** | 10,000 users polling every 1 s = **10,000 req/s**, each going through the load balancer, auth middleware, logging, service and a DB query. Mostly "nothing changed". You tripled responsiveness and tripled the bill |
| **Cost scales with users, not events** | A dead board costs as much as a busy one. You pay for the *medium* (requests), not for the *value* (changes) |
| **Mobile** | Every poll wakes the phone's radio (one of its most expensive operations): battery and data drain |

What we want: **(1) the server speaks first, and (2) cost scales with events, not with the number of viewers.**

---

## Long polling

[9:48](https://www.youtube.com/watch?v=wUQryt697cs&t=588s)

The first idea: ask, and let the server **take its time answering**. It holds the request open (sometimes for a minute). When something changes it responds; the browser immediately sends a new request.

(Documented in **RFC 6202**, along with its problems.) The flaw is the **gap**: after the server answers, the browser has to process the response and open a fresh request. During that window the server has no connection to the client. Average latency ≈ one network transit, but the **worst case is more than three**, and changes in the gap can be missed.

---

## Server-Sent Events (SSE)

[11:55](https://www.youtube.com/watch?v=wUQryt697cs&t=715s)

What if the response **never ends**?

```http
GET /api/events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
(connection stays open; server keeps writing text)

id: 41
event: task.moved
data: {"taskId": 7, "from": "todo", "to": "in-progress"}

id: 42
event: task.deleted
data: {"taskId": 9}
```

- A normal HTTP `GET`. It's **still HTTP**, not a new protocol.
- The server writes events whenever they occur.
- Event format has four fields: **`id`**, **`event`** (name), **`data`**, **`retry`** (how long the browser should wait before reconnecting).
- **The browser (`EventSource`) reconnects for you** if the connection drops, and sends a **`Last-Event-ID`** header with the last `id` it received, so the server can replay what was missed.

SSE isn't an inferior toy. It powered Uber's driver push platform and LinkedIn's messaging features (typing indicators, read receipts). **Every LLM API streams tokens to your CLI/browser using SSE.** It "just works everywhere".

**Limitation: one direction only** (server → client). Sending something back (the user dragging a task) needs a separate `POST`. For many products that's fine.

---

## WebSockets

[19:28](https://www.youtube.com/watch?v=wUQryt697cs&t=1168s)

When you want **two-way communication on one connection**.

### The handshake

It starts as an ordinary HTTP `GET`, so it passes through everything that understands HTTP:

```http
GET /echo HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==     ← 16 random bytes, base64

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

The server appends a **fixed GUID from the spec** to the key, takes the **SHA-1**, base64-encodes it, and returns it as `Sec-WebSocket-Accept`. It isn't security: anyone can compute it. It just proves the responder **understood the WebSocket request** and isn't a cache replaying an old `200`.

After `101`, the connection is **no longer HTTP**: no headers, methods or status codes, just a **bidirectional stream of frames** either side may write at any time.

### Frame format

[22:33](https://www.youtube.com/watch?v=wUQryt697cs&t=1353s)

| Field | Meaning |
|---|---|
| **FIN** (1 bit) | Last piece of a message |
| **Reserved** (3 bits) | |
| **Opcode** (4 bits) | `1` text, `2` binary, `8` close, `9` ping, `10` pong, `0` continuation |
| **Mask** (1 bit) | Is the payload masked |
| **Payload length** (7 bits) | < 126 → the length itself. `126` → next 2 bytes hold it. `127` → next 8 bytes hold it |

A small message has a **2-byte header** vs. hundreds of bytes of HTTP headers in each long-poll request. A big saving.

### Masking: why it exists

[24:10](https://www.youtube.com/watch?v=wUQryt697cs&t=1450s)

Every **client → server** frame is XORed with a **random 4-byte key** that is sent in clear text in the frame. It is **not encryption**: anyone can unmask with one line.

It exists to defend **intermediary caching proxies**. Researchers showed an attack: a malicious page opens a "WebSocket" to the attacker's server, then sends bytes crafted to look like an HTTP `GET` for a popular script (e.g. Google Analytics). A proxy that doesn't understand WebSockets sees a request/response pair and **caches the attacker's file**, poisoning everyone behind it. With per-frame random masking the script can't control what bytes appear on the wire, so it can't forge a valid fake request. The key must be random for every frame.

### Ping / pong (heartbeats)

[26:46](https://www.youtube.com/watch?v=wUQryt697cs&t=1606s)

Opcode `9` ping / `10` pong. **Why:** with TCP, if a client silently vanishes (walked into a lift, switched Wi-Fi → mobile), you get no error and no close. An **idle** connection and a **dead** one look identical to the server. The only way to tell is to send something and see if a reply comes. Typical rule: no pong within ~30 s → treat the connection as dead and free its resources.

---

## How many connections can one machine hold?

[28:48](https://www.youtube.com/watch?v=wUQryt697cs&t=1728s)

An experiment on a 16-core, 61 GB Linux box with a minimal Go WebSocket server (accepts and holds connections, nothing else).

### Limit 1: file descriptors

In Unix **everything is a file**, including sockets. `ulimit -n` = 1,024 open files.

- Opened 2,000 connections → stopped at ~1,017 (1,024 minus ~7 for stdin/stdout/stderr, the listener, the epoll instance...).
- Clients saw **no "connection refused"**, just **timeouts**. The server was healthy (health checks passing) but logged `too many open files` with exponential back-off.
- **Go raises its own FD limit at startup** (to ~1,048,575), while Python in the same shell stayed at 1,024. So the "1,000 connection problem" is **language/runtime-dependent**.

### Limit 2: ephemeral ports (the client's!)

With FDs raised, it stopped at **28,190** with `cannot assign requested address`. The kernel's ephemeral port range was 32,768–60,999 (≈28,232 ports, minus ~42 in use).

A TCP connection is identified by **four values**: source IP, source port, destination IP, destination port. The server has one IP + one port, so each connection must differ in the *client's* IP or port. Hence "a server can only hold 65k connections" is a myth. The limit was on the **load generator**.

Fix: give the client **3 source IPs**, 25,000 connections each → **75,000 connections, 0 failures**. (This is why load tests need several machines.)

### Limit 3: memory

75,000 idle connections: heap went from ~7.5 MB to ~722 MB → **~9.7 KB per connection** (measured ~9,750 bytes, three runs 9.5–9.9 KB). Extrapolate: **1 million connections ≈ a bit under 10 GB** of heap (plus the kernel's own socket structures). Easily fits on a 61 GB machine.

Most of that 9 KB is the **goroutine** watching the connection and its buffers. Systems chasing huge numbers avoid goroutine-per-connection and watch all sockets from one place with `epoll`.

**Takeaway:** a million idle connections is basically a question of **RAM**.

---

## Multiple instances: pub/sub

[38:42](https://www.youtube.com/watch?v=wUQryt697cs&t=2322s)

You won't have one machine. Behind a load balancer: browser A connects to instance 1, browser B to instance 2. A moves a task → **nothing happens in B**, because A's socket lives in instance 1's memory, and instance 2 has never heard of it.

> We spent the whole series making servers **stateless**. A persistent connection **is state**, pinned to one process.

**Sticky sessions don't fix it.** They keep A on instance 1, but B is still on instance 2. The problem is that instances need to **talk to each other**.

### Pub/sub

```
Instance 1 receives "task moved on board 412"
   └─ PUBLISH board:412 {event}  ─►  [ Redis / NATS / Kafka ]
                                         ├─► Instance 1 ─ writes frame to its sockets watching board 412
                                         └─► Instance 2 ─ writes frame to its sockets watching board 412
```

The receiving instance **publishes to a topic**; the broker hands the event to every subscribed instance; each looks at its own connections and writes the frame to those watching that board.

### Delivery guarantee caveat

**Redis Pub/Sub is fire-and-forget (at-most-once).** If an instance is down/disconnected when a message is published (say, during a deployment swap), **it's gone forever**. For **at-least-once** delivery you need something that stores messages and supports acknowledgements: **Redis Streams, Kafka**, or a queue.

---

## Reliability: catching up after disconnects

[44:27](https://www.youtube.com/watch?v=wUQryt697cs&t=2667s)

Connections drop all the time (phone moves Wi-Fi → mobile data, laptop lid closes, load balancer idle timeout). A production system needs a **catch-up strategy**, not just a delivery strategy.

Same shape as SSE's `Last-Event-ID`: **number every message**; the client tells the server the last sequence number it received; the server replays everything after it.

Uber does this: client connects at sequence 0; the server streams increasing sequence numbers; after a drop the client says "I got up to 2", and the server resumes from 3.

---

## Fan-out and reconnection storms

[46:31](https://www.youtube.com/watch?v=wUQryt697cs&t=2791s)

- **Fan-out cost:** work = **events × subscribers**. If 30,000 people watch a board, one move = 30,000 writes. Discord measured **900 ms to 2.1 s** to publish one message in a 30,000-member community. Fix: spread the fan-out across the machines that actually own the recipients, instead of one place.
- **Reconnection storm:** an instance holding 50,000 connections dies, all 50,000 clients reconnect at once and each wants its initial state (boards, tasks, presence) → 50,000 expensive requests in a second. Slack's fix: make connecting cheap by caching the initial state at the edge so a storm doesn't reach the core.

---

## Not covered

Presence (who's online) and **collaborative editing** (Notion / Google Docs live cursors; needs algorithms like **CRDTs**). Each could be hours of content. This video is the foundation.

---

## Summary

1. HTTP is client-initiated. Real time needs the server to push.
2. **Polling**: simple but delay = interval/2, cost scales with **viewers** not events.
3. **Long polling**: held-open request, but a gap between requests (worst case > 3 transits).
4. **SSE**: one never-ending HTTP response (`text/event-stream`), with `id/event/data/retry`, **auto-reconnect** and `Last-Event-ID`. One-way. Used by LLM APIs.
5. **WebSockets**: HTTP `GET` + `Upgrade` → `101` → a two-way **frame** stream. 2-byte minimum header. Client frames are **masked** (to defend proxies, not for secrecy). **Ping/pong** detects dead connections.
6. One machine's limits: **file descriptors** (runtime-dependent), **ephemeral ports (client side)**, then **memory** (~9.7 KB/connection).
7. Persistent connections are **state**. Scale across instances with **pub/sub** (Redis/NATS/Kafka). Redis Pub/Sub is **at-most-once**. Use streams/Kafka for at-least-once.
8. Plan **catch-up** (sequence numbers), **fan-out** cost, **reconnection storms**.

## Interview questions

**Q: Polling vs long polling vs SSE vs WebSockets?**
Polling: client asks on a timer. Long polling: server holds the request until there's data, then the client re-asks. SSE: a persistent one-way server→client HTTP stream. WebSockets: a persistent two-way connection after an HTTP upgrade.

**Q: When pick SSE over WebSockets?**
When data flows server → client only (feeds, notifications, LLM token streaming): simpler, plain HTTP, built-in reconnect and resume.

**Q: How does the WebSocket handshake work?**
HTTP `GET` with `Upgrade: websocket` and `Sec-WebSocket-Key`. Server replies `101 Switching Protocols` with `Sec-WebSocket-Accept` (SHA-1 of the key + a fixed GUID, base64). After that the connection carries frames.

**Q: Why are client WebSocket frames masked?**
To stop attackers crafting bytes that look like HTTP requests and poisoning intermediary caches. It's not encryption.

**Q: Why ping/pong?**
TCP doesn't tell you when a peer silently disappears. A heartbeat distinguishes idle from dead so you can free resources.

**Q: What limits the number of connections a server can hold?**
File descriptors (OS/runtime), ephemeral ports on the client side (in load tests), and memory per connection (~10 KB in the Go example).

**Q: Why don't sticky sessions solve multi-instance real time?**
They pin one client to one instance, but events must reach clients connected to *other* instances. You need inter-instance messaging (pub/sub).

**Q: At-most-once vs at-least-once for real-time events?**
Redis Pub/Sub drops messages to disconnected subscribers (at-most-once). Redis Streams/Kafka persist and ack (at-least-once). Pair with sequence IDs so clients can catch up.

**Q: What is a reconnection storm and how do you mitigate it?**
Many clients reconnecting at once after a failure and all requesting initial state. Mitigate with edge-cached initial state, jittered backoff and rate limits.

**Q: Why is fan-out expensive?**
Work = events × subscribers. Large channels need distributed fan-out across the nodes that own the recipients.
