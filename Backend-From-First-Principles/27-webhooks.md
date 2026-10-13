# 27 — Webhooks: How the Server Calls You

**Video:** [#29 Webhooks: how the server calls you](https://www.youtube.com/watch?v=eWM0CVReP04) (56m)

Service-to-service communication where **the provider calls you**. Why polling fails, what a webhook is, how to **receive** one safely (HMAC signatures, replay protection, SSRF, idempotency, ordering, retries, thin handlers), how to **send** one (the outbox pattern, signing, backoff with jitter), and when webhooks are the wrong tool.

---

## Contents

1. [The problem: knowing a payment succeeded](#the-problem-knowing-a-payment-succeeded)
2. [What a webhook is](#what-a-webhook-is)
3. [Vocabulary](#vocabulary)
4. [What a delivery looks like](#what-a-delivery-looks-like)
5. [Registration, handshakes and local development](#registration-handshakes-and-local-development)
6. [Security 1: is this really from the provider?](#security-1-is-this-really-from-the-provider)
7. [Security 2: replay attacks](#security-2-replay-attacks)
8. [Security 3: SSRF (provider side)](#security-3-ssrf-provider-side)
9. [Reliability: at-least-once, idempotency and ordering](#reliability-at-least-once-idempotency-and-ordering)
10. [Retries and keeping the receiver thin](#retries-and-keeping-the-receiver-thin)
11. [Being the sender: the outbox pattern](#being-the-sender-the-outbox-pattern)
12. [When webhooks are the wrong tool](#when-webhooks-are-the-wrong-tool)
13. [Summary](#summary)
14. [Interview questions](#interview-questions)

---

## The problem: knowing a payment succeeded

[0:00](https://www.youtube.com/watch?v=eWM0CVReP04&t=0s)

Every video so far assumed **the client always initiates**. The real-time video bent that, but the client still opened the connection first. Now: two **backends** talking, with no browser involved.

Example: a user buys a paid plan; the payment is processed by **Stripe**. Success happens in *Stripe's* system, not ours. We need a signal before unlocking the plan.

- **It can't be the browser.** The user's network can drop, they can close the tab or their machine can crash. And a client-side signal can be forged. Payments are critical, so **our server** must be the source of truth.
- **Polling** (call Stripe's API every second) copies all the problems from the real-time video: average delay = half the interval; wasted requests; and, at scale, if 10,000 merchants poll every second, Stripe gets 10,000 req/s. They'll rate-limit you, say to once a minute → ~30 s average delay.

**Solution: flip the direction.** Trust the provider to tell us when it's done.

---

## What a webhook is

[5:36](https://www.youtube.com/watch?v=eWM0CVReP04&t=336s)

> A **webhook** is a plain **HTTP `POST`** that the provider sends to a **URL you registered**, when an event happens. A **reverse API call**.

History: in **2007** Jeff Lindsay proposed "web hooks", a Unix-pipe-like primitive for the web (user-defined callbacks via HTTP POST, like redirecting standard output). Two years later Google engineers built **PubSubHubbub** on a similar idea.

---

## Vocabulary

[7:10](https://www.youtube.com/watch?v=eWM0CVReP04&t=430s)

(Providers use these loosely, so check each one's docs.)

| Term | Meaning |
|---|---|
| **Provider / sender** | The system where the event happens (Stripe, GitHub) |
| **Consumer / receiver** | You |
| **Endpoint** | Your URL that receives the POST |
| **Event** | Anything predefined (payment succeeded, task moved) |
| **Delivery** | **One attempt** to send an event. One event can have **multiple deliveries** (design for this!) |
| **Subscription** | Your registration: the URL + the event types you want |

---

## What a delivery looks like

[8:42](https://www.youtube.com/watch?v=eWM0CVReP04&t=522s)

Demo: a GitHub webhook to a tiny receiver (route `POST /hooks/github`: read the body → verify the signature → store the delivery ID → respond).

```http
POST /hooks/github HTTP/1.1
Content-Type: application/json
X-GitHub-Event: push
X-GitHub-Delivery: 7f3c...-unique-id
X-Hub-Signature-256: sha256=ab12...

{ ...event as JSON... }
```

- The event type, a **unique delivery ID** and a **signature** are in headers.
- Our receiver replied **`202`**. GitHub keeps its own log of recent deliveries (headers, payload, our response).

### How the provider judges success

- **Any `2xx`** = success. **Anything else** (3xx, 4xx, 5xx) = failure.
- **Time limit** on your response. GitHub **10 s**, Shopify **5 s**, Slack **3 s**, Microsoft Graph **3 s**. Miss it = failure → retry. Check each provider's docs.

---

## Registration, handshakes and local development

[11:48](https://www.youtube.com/watch?v=eWM0CVReP04&t=708s)

A registration form takes a **URL**, a **secret** (for signing) and **event types**.

**Use one URL per provider** (`/hooks/github`, `/hooks/stripe`), never a generic `/hooks/callback`. Each provider has its own headers, signature scheme and retry behaviour that your handler must account for.

### Handshakes

Some providers verify your URL before sending real events:

| Provider | Handshake |
|---|---|
| Microsoft Graph | Sends a validation token you must return as plain text |
| Discord | Sends a `ping`, expects a `pong` |
| Amazon SNS | Sends a subscription-confirmation URL you must visit |

**Why:** without it, anyone could register a **victim's** address, subscribe to every event and make the provider flood them. It's a DoS vector.

### Local development

A webhook is an **inbound** request from the internet. Your laptop is behind a home router and `localhost` means nothing to a remote server. Use a **tunnel** (**ngrok**, **Cloudflare Tunnel**) that gives you a temporary public HTTPS URL forwarding to your local port.

---

## Security 1: is this really from the provider?

[15:21](https://www.youtube.com/watch?v=eWM0CVReP04&t=921s)

Your endpoint is a **public URL**. If anyone can guess it (`/webhooks`, `/hooks`) and POST `{"payment": "success", "user": 42}`, they unlock paid plans for free. **HTTPS doesn't help**: it hides bytes in transit but doesn't prove *who* sent them.

Five ways providers authenticate deliveries:

| # | Method | Notes |
|---|---|---|
| 1 | **Secret token in the URL** | Weakest. URLs get logged everywhere (CloudWatch, Elasticsearch...) |
| 2 | **IP allow-list** | Provider publishes its IPs; your firewall allows only those. Better, but not enough alone |
| 3 | **Mutual TLS** | Provider gives a client certificate |
| 4 | **HMAC signature** | **Most common (~65–80% of providers)** |
| 5 | **Signature with a public key** | Provider signs with a private key, you verify with its public key (Discord) |

### HMAC (hash-based message authentication code)

The provider computes `HMAC-SHA256(body, sharedSecret)` and sends it in a header. You recompute over the same body with the same secret and **compare**. Matching proves it came from someone holding the secret. A one-byte change to the body changes the whole signature.

**What exactly is signed differs per provider. Read the docs:**

| Provider | Signs |
|---|---|
| **GitHub** | the body only |
| **Stripe** | `timestamp + "." + body` (replay protection) |
| **Standard Webhooks spec** | `deliveryId + "." + timestamp + "." + body` |
| **Twilio** | the full URL + sorted form fields (SHA-1) |

### Verify the raw bytes first

The signature is over exact bytes. If your framework (e.g. `express.json()`) parses the JSON, then you re-serialise it, a different whitespace choice (`{"a":1}` vs `{"a": 1}`) changes the bytes and **verification fails**.

> **Read the body as bytes → verify → then parse.**

Demo: correct secret → `202`. Change the secret → `401`. Parse-then-re-encode-then-verify → fails even with the right secret.

---

## Security 2: replay attacks

[22:32](https://www.youtube.com/watch?v=eWM0CVReP04&t=1352s)

An attacker captures one genuine delivery (headers + body + valid signature) and **replays** it later, maybe 1,000 times. The signature is still valid.

**Defence:** include a **timestamp inside the signed string** and reject anything too old. Stripe's library rejects deliveries older than **5 minutes**. Since the timestamp is signed, altering it breaks the signature.

### Constant-time comparison

Don't compare signatures with `==`. Ordinary string comparison stops at the first mismatching byte, so response time leaks how many leading bytes matched. An attacker measuring thousands of requests can derive the signature **byte by byte** (a **timing attack**; compare [06](06-authentication-and-authorization.md)). Use:

| Language | Function |
|---|---|
| Go | `hmac.Equal` |
| Node.js | `crypto.timingSafeEqual` |
| Python | `hmac.compare_digest` |

---

## Security 3: SSRF (provider side)

[25:05](https://www.youtube.com/watch?v=eWM0CVReP04&t=1505s)

This one concerns you when **you are the sender**. The provider calls whatever URL a customer typed. An attacker registers `http://169.254.169.254/...` (the cloud **metadata service**, which hands out credentials) or an internal admin host. Your dispatcher, inside your network, makes the request: **Server-Side Request Forgery (SSRF)**.

Checking the URL at registration isn't enough: DNS can resolve to a public IP at registration and later flip to a private one, or the public host can **redirect** to a private one.

**Defences:**

1. **Resolve the hostname yourself** at send time and check the IP against a deny-list of private ranges.
2. **Never follow redirects**. Treat any non-2xx (including 3xx) as failure.
3. Send from a separate network via an **egress proxy** (Stripe open-sourced one, *Smokescreen*).

---

## Reliability: at-least-once, idempotency and ordering

[27:38](https://www.youtube.com/watch?v=eWM0CVReP04&t=1658s)

Your handler does its work and returns `200`, but the `200` is **lost in the network**. The provider thinks it failed and retries the same event.

> Providers promise **at least once**: it won't be lost, but it **may arrive more than once**. Never exactly once.

### Idempotency via the delivery ID

The **delivery/event ID stays the same across retries**:

| Provider | Where |
|---|---|
| GitHub | `X-GitHub-Delivery` header |
| Stripe | `id` in the body |
| Standard Webhooks | `webhook-id` |

Keep a `processed_deliveries` table with a **UNIQUE constraint on the delivery ID**.

> **Insert the delivery ID in the same database transaction as the side effects.**
> - Work first, ID after → crash between them → retry does the work **twice**.
> - ID first, work after → work fails → retry is skipped though the work never happened.
> One transaction means either both happen or neither.

### Ordering

[32:13](https://www.youtube.com/watch?v=eWM0CVReP04&t=1933s)

Events are **not** delivered in order. A Stripe engineer wrote about this in 2017. Object *created* fails once (you were deploying) and retries 5 min later, while *deleted* succeeds immediately. You get *deleted* then *created* and end up with a row for an object that no longer exists. (Timestamps are in seconds too, so two events within the same second tie.)

Solutions:

1. Treat the event as a **signal only**; **re-fetch the current state** from the provider's API (1 extra call). State is always the source of truth.
2. Attach a **version / `updated_at`** and apply only if newer, in one transaction.
3. **Buffer** events until earlier ones arrive (unpredictable).
4. Some providers sell **strictly ordered delivery** as a feature.

---

## Retries and keeping the receiver thin

[34:16](https://www.youtube.com/watch?v=eWM0CVReP04&t=2056s)

Every provider has its own retry schedule, so read the docs.

| Provider | Retries |
|---|---|
| **Stripe** | Exponential backoff for up to **3 days**, then **disables** your endpoint and emails you |
| **Svix** | Immediately, then 5 s, 5 min, 30 min, 2 h, 5 h, 10 h... |
| Shopify / Slack / Polar | Own schedules |
| **GitHub** | **Zero retries**. If you were down, the delivery is gone until you click *redeliver* or script the API to list failed deliveries and redeliver |

(Svix is a webhooks-as-a-service provider. Building the full sender is a lot of engineering.)

### Retries amplify load

Bursts are common: a billing system renews every subscription on the 1st; an email tool sends 500k delivery receipts. If your handler does all the work inside the request (verify + update 3 tables + call other APIs), it misses the 3–10 s deadline, **which triggers retries, which adds more load.**

> **Keep the receiver thin:** read body → verify signature → store the delivery → return `2xx` → stop. A **background worker** (queue; see [12](12-task-queues-and-background-jobs.md)) does the actual processing.

---

## Being the sender: the outbox pattern

[38:52](https://www.youtube.com/watch?v=eWM0CVReP04&t=2332s)

Now you are the provider. Your customer wants a notification when a task moves to *done*.

**Naive:** inside the `move` handler, after saving, `http.post(customerURL, event)`. Three bugs:

1. **Latency:** a slow or dead customer endpoint adds its latency to *your* handler.
2. **Lost event:** you saved the move, then crashed before sending. The task moved but no webhook.
3. **Phantom event:** you posted first, then your DB write failed or you crashed. The customer was told about a move that didn't happen.

**Outbox pattern:**

```
BEGIN TRANSACTION
  UPDATE tasks SET status='done' WHERE id=...
  INSERT INTO outbox (event ...)          -- same DB, same transaction
COMMIT                                    -- both or neither

(separate background dispatcher)
  read outbox row → sign → POST to customer → record attempt → mark delivered / schedule retry
```

This is the **write-ahead log** idea from the databases video, applied to events: write your *intention* somewhere durable, and let another process do the delivery. The handler stays as fast as before.

### Schema in the demo

Three tables: **endpoints** (URL + secret), **outbox** (the events), **deliveries** (one row per attempt).

### The dispatcher, per delivery

[43:26](https://www.youtube.com/watch?v=eWM0CVReP04&t=2606s)

1. **Sign** `id.timestamp.body` with the endpoint's secret (standard webhooks format).
2. `POST` with a **10-second timeout** (a common default).
3. **Record the attempt**: status code, first 2 KB of the response, time.
4. `2xx` → mark **delivered**. Otherwise compute the next retry.

**Backoff + jitter:** delays grow after each failure (5 s, 5 min, 15 min, 1 h...). **Jitter** adds random seconds (5 min 42 s rather than exactly 5 min) so thousands of retries don't land at the same instant.

Also:

- Respect the customer's **`Retry-After`** header.
- After the last attempt → mark **exhausted**.
- After **10 consecutive failures** → **disable the endpoint** and email the owner.
- **One queue per endpoint** so one dead customer doesn't stall others.
- The **SSRF guard** runs on **every send**, not just at registration.

Demo: a customer endpoint returns 500 → attempt 1 fails, the deliveries table schedules attempt 2 with a longer delay. Fix the endpoint → next attempt `200` → delivered. Registering the loopback admin route or a hostname resolving to a private IP is refused.

---

## When webhooks are the wrong tool

[48:31](https://www.youtube.com/watch?v=eWM0CVReP04&t=2911s)

A webhook is a **notification**: "this happened, and here's the data".

**Right use:** post to chat when X happens; trigger a CI/CD run; unlock a plan when payment succeeds.

**Wrong use:** as a **sync mechanism** to keep a copy of the provider's data in your database. Example: you use **Clerk** for auth (so identity/roles/permissions live in Clerk's DB), but copy users/organisations/roles into your own tables to avoid 50–60 ms (or 500+ ms) API calls to Clerk on every check. You subscribe to `user.created`, `user.updated`, `org.deleted` webhooks to maintain that copy. Now every pitfall above applies (timeouts, replays, ordering, duplicates) and your copy can silently go wrong.

**Better: use the provider's ordered event log.** Providers guarantee their event log is ordered because it's written transactionally inside their own system.

| Approach | How |
|---|---|
| **A. Periodic sync** (if a few minutes of staleness is OK) | Every N minutes, a worker calls the provider's `/events` API, applies the events **after your stored `last_event_id`**, and updates the cursor |
| **B. Webhook as a doorbell** | The webhook arrives just to say "a new event was added to the log". You start a worker that reads events since `last_event_id` and applies them. Within seconds your copy is in sync, **without relying on the webhook's payload, ordering or exactly-once delivery** |

---

## Summary

1. A **webhook** = the provider `POST`s to your URL when something happens. A reverse API call that replaces polling.
2. A **delivery** is one attempt. One event → many deliveries. Success = any `2xx` **within a time limit** (3–10 s).
3. One URL per provider. Handshakes prevent registration abuse. Use **ngrok/Cloudflare Tunnel** in local dev.
4. **Authenticate deliveries**: HMAC-SHA256 (most common) over **raw bytes** → **verify before parsing**. Include a signed **timestamp** (reject if > ~5 min) to stop **replays**. Use **constant-time comparison**.
5. Delivery is **at-least-once** → be **idempotent**: unique delivery ID in the **same transaction** as the work.
6. Events can arrive **out of order** → treat as a signal and re-fetch state, or use versions.
7. Keep the receiver **thin**: verify → store → `2xx`. Process in a queue.
8. As a **sender**: **outbox pattern**. Sign, timeout 10 s, exponential backoff + jitter, respect `Retry-After`, disable after repeated failures, one queue per endpoint, **SSRF guard** (resolve + deny private ranges, no redirects, egress proxy).
9. Webhooks are for **notifications**, not for syncing data. For sync, use the provider's ordered event log, with the webhook as a trigger.

## Interview questions

**Q: Webhook vs polling?**
Polling: you ask repeatedly (delay, wasted requests, rate limits). Webhook: the provider pushes an HTTP POST when an event occurs.

**Q: How do you verify a webhook is genuine?**
Recompute the HMAC (e.g. SHA-256) over the raw request body with the shared secret and compare it, in constant time, to the signature header. Also check a signed timestamp to reject replays.

**Q: Why verify before parsing JSON?**
The signature is over exact bytes. Parsing then re-serialising can alter whitespace/ordering and break verification.

**Q: What is a replay attack and how do you stop it?**
Re-sending a captured valid request. Include a timestamp in the signed content and reject deliveries older than a short window (e.g. 5 min). Also dedupe by delivery ID.

**Q: Why constant-time comparison?**
A normal `==` returns at the first mismatch, leaking timing information that can be used to forge a signature byte by byte.

**Q: Why must webhook handlers be idempotent?**
Providers guarantee at-least-once delivery. Retries mean duplicates. Record the delivery ID under a unique constraint, in the same transaction as the side effects.

**Q: Why store the delivery ID and do the work in one transaction?**
Otherwise a crash between them either duplicates the work or marks work done that never happened.

**Q: How do you handle out-of-order events?**
Treat the webhook as a signal and fetch the current state from the provider, or compare versions/`updated_at` before applying.

**Q: What should the webhook endpoint do inside the request?**
Read the body, verify the signature, persist the event, return 2xx quickly. Do the real work in a background worker.

**Q: What's the outbox pattern and what problem does it solve?**
Write the event to an `outbox` table in the same DB transaction as the state change; a separate dispatcher delivers it. It avoids lost events, phantom events and latency coupling.

**Q: What is SSRF in the context of webhooks?**
A malicious customer registers an internal/metadata URL so the sender's server makes requests inside its own network. Mitigate: resolve DNS yourself, block private ranges, don't follow redirects, use an egress proxy, and re-check at each send.

**Q: Why use exponential backoff with jitter?**
Backoff spaces retries out to avoid hammering a failing endpoint. Jitter spreads retries so they don't synchronise into bursts.

**Q: When are webhooks the wrong tool?**
As a synchronisation mechanism for copying data. Use the provider's ordered events API and use the webhook only as a trigger.
