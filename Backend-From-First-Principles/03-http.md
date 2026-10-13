# 03 — Understanding HTTP for Backend Engineers

**Video:** [#5 Understanding HTTP for backend engineers, where it all starts](https://www.youtube.com/watch?v=a3C1DMswClQ) (1h 18m)

HTTP is the medium through which browsers talk to servers. This lecture covers statelessness, the client-server model, HTTP versions, message anatomy, headers, methods and idempotency, CORS (simple vs preflight), status codes, HTTP caching, content negotiation, compression, persistent connections, large uploads/downloads, and a short note on SSL/TLS/HTTPS.

The demos use a simple frontend, a server (any language, no code shown) and **Burp Suite** (an HTTP interceptor) to look at real requests.

---

## Contents

1. [Two core ideas](#two-core-ideas)
2. [HTTP and TCP](#http-and-tcp)
3. [HTTP versions](#http-versions)
4. [Anatomy of HTTP messages](#anatomy-of-http-messages)
5. [Headers](#headers)
6. [Methods](#methods)
7. [CORS](#cors)
8. [Status codes](#status-codes)
9. [HTTP caching](#http-caching)
10. [Content negotiation](#content-negotiation)
11. [Compression](#compression)
12. [Persistent connections](#persistent-connections)
13. [Large requests and responses](#large-requests-and-responses)
14. [SSL, TLS and HTTPS](#ssl-tls-and-https)
15. [Summary](#summary)
16. [Interview questions](#interview-questions)

---

## Two core ideas

[0:32](https://www.youtube.com/watch?v=a3C1DMswClQ&t=32s)

### 1. HTTP is stateless

The server has **no memory of past interactions**. Each request carries everything the server needs: URL, method, headers (cookies, tokens). After responding, the server forgets. The next request is treated as brand new.

Example: to get a user profile, the client must send credentials (cookie or token) **on every request**.

| Benefit | Why |
|---|---|
| **Simplicity** | The server doesn't have to store session state |
| **Scalability** | Any server can handle any request, so load balancing is trivial. A crashing server doesn't lose a "session" |

Since real apps need continuity (logins, carts), developers layer **cookies, sessions and tokens** on top. Covered in the auth lecture.

### 2. Client-server model

- The **client** (browser, mobile app, another service) **always initiates** the communication, sending a request with everything the server needs.
- The **server** hosts resources (pages, APIs, files) and waits. It processes the request and sends a response (HTML, JSON, text, error...).

> The server never starts a conversation. HTTP is **client-initiated**. (This is why webhooks and WebSockets are different stories.)

HTTP and HTTPS are treated as interchangeable here. HTTPS is HTTP with encryption (TLS).

---

## HTTP and TCP

[3:38](https://www.youtube.com/watch?v=a3C1DMswClQ&t=218s)

Client and server first need a **connection**. HTTP only needs a **reliable** transport that doesn't silently lose messages. Of the two common transports, **TCP** (reliable, connection-oriented) and UDP, HTTP uses TCP. TCP sets up a connection with a **3-way handshake**.

In the OSI model, backend engineers mostly live at **Layer 7, the application layer**. TCP handshakes and TLS are network-engineering topics, so look them up separately if curious. For now: *a network connection exists, and messages go back and forth.*

---

## HTTP versions

[5:42](https://www.youtube.com/watch?v=a3C1DMswClQ&t=342s)

| Version | Key changes |
|---|---|
| **HTTP/1.0** | A **new connection for every request** and response. Slow. |
| **HTTP/1.1** | **Persistent connections**: many requests over one TCP connection. Chunked transfer encoding, better caching. Still the most used. |
| **HTTP/2** | **Multiplexing** (many requests/responses in parallel on one connection), **binary framing** instead of text, **header compression (HPACK)**, **server push**. |
| **HTTP/3** | Built on **QUIC**, which runs over **UDP**. Faster connection setup, lower latency, better packet-loss handling, multiplexing **without head-of-line blocking** (an issue in HTTP/2). |

---

## Anatomy of HTTP messages

[7:13](https://www.youtube.com/watch?v=a3C1DMswClQ&t=433s)

**Request** (client → server):

```http
PUT /api/resource HTTP/1.1            ← method, resource URL, HTTP version
Host: api.example.com                 ← headers (key: value)
Origin: https://example.com
Authorization: Bearer <token>
Content-Type: application/json
                                      ← blank line = headers end, body starts
{ "name": "..." }                     ← request body
```

**Response** (server → client):

```http
HTTP/1.1 200 OK                       ← version, status code, status text
Content-Type: application/json
Cache-Control: max-age=10
                                      ← blank line
{ "id": 1, ... }                      ← response body
```

---

## Headers

[8:45](https://www.youtube.com/watch?v=a3C1DMswClQ&t=525s)

Headers are **key-value pairs** carrying metadata about the request or response.

**Why separate them from the body/URL?** The parcel analogy: the recipient's address is written **on the outside of the box**, so couriers can route it without opening it. Headers are the outside label. Anything that needs to read the message's metadata (proxies, caches, browsers) can do so without parsing the body.

### Categories

| Category | Purpose | Examples |
|---|---|---|
| **Request headers** | Info about the request/client | `User-Agent` (browser/Postman/app), `Authorization` (bearer token), `Accept` (JSON, HTML...) |
| **General headers** | Apply to both request and response | `Date`, `Cache-Control` (`no-cache`, `max-age`), `Connection` (`keep-alive`/`close`) |
| **Representation headers** | Describe the **body** | `Content-Type` (media type), `Content-Length` (bytes), `Content-Encoding` (gzip, deflate), `ETag` (unique id for caching) |
| **Security headers** | Harden the browser's behaviour | see below |

### Security headers

| Header | What it does |
|---|---|
| `Strict-Transport-Security` (HSTS) | Client must use HTTPS only. Prevents protocol-downgrade attacks |
| `Content-Security-Policy` | Restricts where JS/CSS/images can load from. Helps prevent XSS |
| `X-Frame-Options` | Stops the page being embedded in an iframe. Prevents clickjacking |
| `X-Content-Type-Options` | Stops the browser guessing MIME types. Prevents MIME sniffing |
| `Set-Cookie` with `HttpOnly` / `Secure` | Cookie hidden from JS and only sent over HTTPS |

### Two ideas about headers

[14:25](https://www.youtube.com/watch?v=a3C1DMswClQ&t=865s)

- **Extensibility**: you can add headers (including custom ones like `X-Custom-Header`) without changing the protocol. Security, negotiation and new features all arrived this way.
- **Remote control**: headers let the client send instructions/preferences that change how the server behaves:
  - **Content negotiation**: `Accept`
  - **Caching and expiry**: `Cache-Control`, `Expires`
  - **Authentication**: `Authorization`

---

## Methods

[16:28](https://www.youtube.com/watch?v=a3C1DMswClQ&t=988s)

Methods express the **intent** of the request.

| Method | Intent | Body? | Idempotent? |
|---|---|---|---|
| **GET** | Fetch data; must not modify anything | no | **yes** |
| **POST** | Create data | yes | **no** |
| **PATCH** | Update (selectively replace) | yes | |
| **PUT** | Update by **completely replacing** the previous instance with the body | yes | **yes** |
| **DELETE** | Remove a resource | usually no | **yes** |
| **OPTIONS** | Ask the server about its capabilities (CORS preflight) | no | |

> **Rule of thumb:** use `PATCH` unless you really intend a full replacement (`PUT`). Many developers use `PUT` where they should use `PATCH`.

### Idempotency

[18:01](https://www.youtube.com/watch?v=a3C1DMswClQ&t=1081s)

Calling an **idempotent** method multiple times has the same result.

- GET: fetching again doesn't change anything.
- PUT: replacing with the same data twice leaves the same state.
- DELETE: a resource can only be deleted once. Further calls don't change the outcome.
- **POST** is *not* idempotent. Submitting "create note" twice creates two notes.

(The next-but-one lecture, REST API design, goes much deeper on this.)

---

## CORS

[20:34](https://www.youtube.com/watch?v=a3C1DMswClQ&t=1234s)

Browsers enforce the **same-origin policy**: a page can only make requests to the **same origin** (scheme + host + port) that served it. **CORS** (Cross-Origin Resource Sharing) is how a server explicitly allows other origins. Without it, the browser blocks a response from, say, `api.example.com` to a page on `example.com`.

> Important: **the browser enforces it**, not the server. The request still reaches the server. The browser refuses to hand the response to your JavaScript.

There are two flows.

### Simple request

[21:37](https://www.youtube.com/watch?v=a3C1DMswClQ&t=1297s)

```
Frontend: https://example.com        API: https://api.example.com

1. Browser sends GET with   Origin: https://example.com
2. Server responds with     Access-Control-Allow-Origin: https://example.com   (or *)
3. Browser sees the header matches → lets the response through to JS
```

If the server **omits** `Access-Control-Allow-Origin` (or the origin isn't allowed), the browser **blocks** the response and logs a **CORS error** in the console and the Network tab.

### Preflight request

[24:11](https://www.youtube.com/watch?v=a3C1DMswClQ&t=1451s)

A cross-origin request is **preflighted** if it is cross-origin **and any** of these is true:

1. The method is **not** `GET`, `POST` or `HEAD` (e.g. `PUT`, `DELETE`).
2. It uses **non-simple headers** (e.g. `Authorization`, custom headers).
3. The `Content-Type` is not one of `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`. **So `application/json` triggers a preflight.**

Since nearly all modern API traffic is JSON with an `Authorization` header, **most cross-origin requests are preflighted**.

**The preflight is an `OPTIONS` request with no body:**

```http
OPTIONS /api/resource HTTP/1.1
Host: api.example.com
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type
```

"I'm making a cross-origin request. Do you allow `PUT` and these headers?"

**A server that handles CORS replies:**

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com        (or *)
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

| Header | Meaning |
|---|---|
| `Allow-Origin` | This client's origin is permitted |
| `Allow-Methods` | Methods the server supports for this resource |
| `Allow-Headers` | Non-simple headers allowed |
| `Max-Age` | Cache this preflight result (seconds, e.g. 24h) so the browser doesn't preflight every request |

`204` is used because it's a general inquiry, with no content.

If everything checks out, the browser sends the **actual request**. If the server doesn't handle CORS, the preflight fails and the browser blocks the real request.

### In the demo

[30:24](https://www.youtube.com/watch?v=a3C1DMswClQ&t=1824s)

- Frontend on `localhost:5173`, API on `localhost:3000`. **Different ports = different origins.**
- Simple GET works because the response includes `Access-Control-Allow-Origin`. When the server removes that header, the browser throws a CORS error even though the server answered.
- A `PUT` with `Authorization` and JSON triggers an `OPTIONS` preflight first (`204`), then the real request. All three preflight conditions were true here.
- `Access-Control-Max-Age` was set to 0 in the demo so every preflight is visible.

---

## Status codes

[39:08](https://www.youtube.com/watch?v=a3C1DMswClQ&t=2348s)

Status codes communicate the **outcome** in a standard way, so a client can react without parsing the body: log the user out on `401`, ask them to fix a form on `400`. Before codes were standard, clients had to guess from the content.

Three digits. The first digit gives the class:

| Class | Meaning |
|---|---|
| **1xx** | Informational |
| **2xx** | Success |
| **3xx** | Redirection |
| **4xx** | Client error |
| **5xx** | Server error |

### 1xx (rare)

- `100 Continue`: server got the headers, client may send the body (large uploads).
- `101 Switching Protocols`: e.g. HTTP → WebSocket.

### 2xx

| Code | Meaning | Typical use |
|---|---|---|
| **200 OK** | Success | successful GET |
| **201 Created** | New resource created | POST / form submission |
| **204 No Content** | Success, nothing to return | DELETE, CORS preflight |

### 3xx

| Code | Meaning | Typical use |
|---|---|---|
| **301 Moved Permanently** | Use the new URL from now on | route renamed `/user` → `/person`, keep old clients working |
| **302 Found** (temporary redirect) | Temporarily elsewhere. Keep using the original URL | a short campaign redirect |
| **304 Not Modified** | Cached copy is still valid | conditional GET with ETags |

### 4xx

| Code | Meaning | Use when |
|---|---|---|
| **400 Bad Request** | Invalid or illogical data | expected a number, got an array |
| **401 Unauthorized** | Not authenticated | missing, invalid or expired token |
| **403 Forbidden** | Authenticated but **not allowed** | user A trying to delete user B's resource |
| **404 Not Found** | Resource doesn't exist | bad URL or deleted |
| **405 Method Not Allowed** | Wrong HTTP method | `PUT` on a GET-only route |
| **409 Conflict** | State conflict | creating a folder whose name already exists |
| **429 Too Many Requests** | Rate limited | exceeded e.g. 60 req/s |

### 5xx

| Code | Meaning |
|---|---|
| **500 Internal Server Error** | Unexpected failure (unhandled exception). Don't leak details to the client |
| **501 Not Implemented** | Method/feature not supported (yet) |
| **502 Bad Gateway** | A proxy/load balancer got an invalid response from upstream (set by proxies like Nginx) |
| **503 Service Unavailable** | Down, overloaded or in maintenance. Try again later |
| **504 Gateway Timeout** | The proxy didn't get a timely response from upstream |

---

## HTTP caching

[55:08](https://www.youtube.com/watch?v=a3C1DMswClQ&t=3308s)

Store copies of responses and **reuse** them, saving bandwidth, load time and server load.

### First request

Server responds `200` with three key headers:

```http
Cache-Control: max-age=10          ← keep this for at most 10 seconds
ETag: "3141"                       ← a hash/fingerprint of the response
Last-Modified: Tue, 14 Jan ...     ← when the resource last changed
```

(The demo uses a random number for the ETag. Real ETags are hashes of the response.)

### Revalidation (conditional GET)

Once the cache is stale, the browser sends the stored validators back:

```http
GET /api/resource
If-None-Match: "3141"
If-Modified-Since: Tue, 14 Jan ...
```

"Send me a new copy only if the ETag no longer matches, or it changed after this date."

| Server's answer | Meaning |
|---|---|
| **`304 Not Modified`** (no body) | Your cached copy is still good, reuse it |
| **`200`** + new body + new `ETag` | It changed. Replace your cache |

Demo: after a POST that updated the resource, the next GET with the old ETag gets `200` and `ETag: "2943"`. A GET after that sends the new ETag and gets `304`.

### Caveat

The server must manage ETags correctly. If you forget to update one, clients keep serving stale data. Modern frontends often use client-side caching (React Query), which gives the client full control over when to refetch. HTTP caching is still fine for simple cases.

---

## Content negotiation

[1:02:23](https://www.youtube.com/watch?v=a3C1DMswClQ&t=3743s)

The client and server agree on the best representation.

| Type | Request header | Example |
|---|---|---|
| **Media type** | `Accept` | `application/json` vs `application/xml` |
| **Language** | `Accept-Language` | `en`, `es` |
| **Encoding** | `Accept-Encoding` | `gzip, deflate, br, zstd` |

Demo: same endpoint, `Accept-Language: es` → the content comes back in Spanish. `Accept: application/xml` → XML.

---

## Compression

[1:07:00](https://www.youtube.com/watch?v=a3C1DMswClQ&t=4020s)

Part of the same negotiation: the client advertises `Accept-Encoding: gzip, deflate, br`; the server compresses and replies with `Content-Encoding: gzip`; the browser decompresses transparently.

Demo with an 11,000-entry JSON file:

| | Size |
|---|---|
| With gzip | **3.8 MB** |
| Without compression | **26 MB** |

Compression is a huge saving for large text responses.

---

## Persistent connections

[1:09:04](https://www.youtube.com/watch?v=a3C1DMswClQ&t=4144s)

- HTTP/1.0: one connection per request-response; opening and closing TCP is expensive.
- HTTP/1.1: **connections are persistent by default**. One TCP connection serves many requests.
- `Connection: keep-alive` can explicitly request it, with optional `timeout` and `max` parameters.
- `Connection: close` closes the connection after the response.

Good to know, but the defaults are normally fine.

---

## Large requests and responses

[1:11:08](https://www.youtube.com/watch?v=a3C1DMswClQ&t=4268s)

### Uploading big files: multipart

```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----abc123
Content-Length: ...

------abc123
Content-Disposition: form-data; name="file"; filename="photo.png"
Content-Type: image/png

<binary data>
------abc123--
```

The file is sent as **parts**. The **boundary** is the delimiter that separates parts: it marks the start of the binary data and the end.

### Downloading big files: streaming

The server sends data in **chunks** with:

```http
Content-Type: text/event-stream
Connection: keep-alive
```

The client keeps appending chunks until the whole file is received.

---

## SSL, TLS and HTTPS

[1:15:16](https://www.youtube.com/watch?v=a3C1DMswClQ&t=4516s)

| | |
|---|---|
| **SSL** | The original protocol for encrypting client-server traffic. Now **outdated** (vulnerabilities) |
| **TLS** | The modern, more secure replacement. Uses **certificates** to authenticate the server and set up an encrypted connection; prevents eavesdropping and tampering. The recommended version is TLS 1.3 |
| **HTTPS** | HTTP running over TLS. The browser and server encrypt traffic so credentials etc. can't be intercepted |

Enough for application-level work.

---

## Summary

1. HTTP is **stateless** (every request carries all it needs) and **client-initiated**. Runs over **TCP**. Versions: 1.0 → 1.1 (persistent) → 2 (multiplexing) → 3 (QUIC/UDP).
2. A message is **start line + headers + blank line + body**.
3. **Headers** = metadata: request, general, representation, security. Extensible, and act like a remote control.
4. **Methods**: GET (safe), POST (create, non-idempotent), PUT (replace), PATCH (partial update), DELETE, OPTIONS (preflight).
5. **CORS**: browsers enforce same-origin. Servers opt in via `Access-Control-Allow-*`. JSON + `Authorization` ⇒ **preflight (`OPTIONS`)**.
6. **Status codes**: 2xx ok, 3xx redirect (301/302/304), 4xx client error (400/401/403/404/405/409/429), 5xx server error (500/501/502/503/504).
7. **Caching**: `Cache-Control`, `ETag`, `Last-Modified`, `If-None-Match` → `304`.
8. **Negotiation**: `Accept`, `Accept-Language`, `Accept-Encoding`. Compression slashes payloads.
9. **Large data**: multipart/form-data with boundary upload, chunked event-stream download.
10. **TLS** encrypts HTTP into **HTTPS**.

## Interview questions

**Q: What does "HTTP is stateless" mean, and how do apps work around it?**
The server keeps no memory between requests, so every request is self-contained. Apps add state with cookies, sessions or tokens sent on every request.

**Q: HTTP/1.1 vs 2 vs 3?**
1.1: persistent connections. 2: multiplexing, binary framing, header compression. 3: QUIC over UDP, no head-of-line blocking, faster setup.

**Q: What is a CORS preflight and when does it happen?**
An `OPTIONS` request the browser sends before the real one to ask the server if the method/headers are allowed. It happens for cross-origin requests that use non-simple methods, non-simple headers, or a non-simple content type (e.g. JSON).

**Q: Is CORS a server security feature?**
No, the browser enforces it. The server just declares who's allowed with headers. Non-browser clients (curl, Postman) ignore it.

**Q: 401 vs 403?**
401: you're not authenticated (missing/invalid/expired credentials). 403: you are authenticated but not permitted.

**Q: 502 vs 503 vs 504?**
502: the proxy got a bad response from upstream. 503: the service is unavailable/overloaded. 504: the proxy timed out waiting for upstream.

**Q: How do ETags work?**
The server sends a fingerprint of the response. The client later sends it back with `If-None-Match`. If unchanged, the server returns `304` with no body and the client reuses its cache.

**Q: What does `Content-Encoding: gzip` do?**
Marks the body as gzip-compressed (the client advertised support via `Accept-Encoding`). The browser decompresses it. It dramatically reduces transfer size.

**Q: How do you upload a large file over HTTP?**
`multipart/form-data` with a `boundary` that delimits the parts. For big downloads, stream with chunked transfer / `text/event-stream`.

**Q: PUT vs PATCH vs POST regarding idempotency?**
PUT and PATCH (when setting values) are idempotent. POST is not, since repeating it creates new resources.
