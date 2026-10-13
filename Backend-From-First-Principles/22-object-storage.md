# 22 — Object Storage: Everything You Need to Know

**Videos:**
- [#23 Object Storage — Everything You Need to Know: Part 1](https://www.youtube.com/watch?v=Ie0TjKI9cDI) (1h 29m)
- [#24 Object Storage — Everything You Need to Know: Part 2](https://www.youtube.com/watch?v=iWrVCxexUWY) (50m)

Part 1 (sections 1–10): why local-disk storage fails, the three storage types, what an object is, how S3-style storage works internally (metadata plane vs data plane, erasure coding, consistency, conditional writes), and upload architectures (via your server vs **pre-signed URLs**).
Part 2 (sections 11–15): multipart uploads for big files, resumable uploads, downloads, range requests, video streaming and cost.

Timestamps below are marked **[P1]** or **[P2]** for the part.

---

## Contents

1. [The naive approach and why it breaks](#the-naive-approach-and-why-it-breaks)
2. [Three kinds of storage](#three-kinds-of-storage)
3. [What object storage is](#what-object-storage-is)
4. [Key design](#key-design)
5. [How it works inside](#how-it-works-inside)
6. [Durability is not backup](#durability-is-not-backup)
7. [Consistency and conditional writes](#consistency-and-conditional-writes)
8. [Upload architecture 1: through your server](#upload-architecture-1-through-your-server)
9. [Upload architecture 2: pre-signed URLs](#upload-architecture-2-pre-signed-urls)
10. [The two-phase upload flow and orphan cleanup](#the-two-phase-upload-flow-and-orphan-cleanup)
11. [Large files: multipart upload](#large-files-multipart-upload)
12. [Combining multipart + pre-signed URLs; resuming](#combining-multipart--pre-signed-urls-resuming)
13. [Downloads](#downloads)
14. [Range requests and video streaming](#range-requests-and-video-streaming)
15. [Cost](#cost)
16. [Summary](#summary)
17. [Interview questions](#interview-questions)

---

## The naive approach and why it breaks

[P1 0:00](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=0s)

Requirement: users upload a profile picture. The intuitive solution:

1. Receive the file (a `multipart/form-data` upload).
2. Save it in the **local file system** of the server.
3. Store the path in the database.

For small profile pictures (300 KB–10 MB) it works. Then videos arrive (up to GBs). Six problems:

| # | Problem | Detail |
|---|---|---|
| 1 | **Ephemeral disk** | Containers (Docker/Kubernetes, Railway/Render, serverless) have a file system that exists only as long as the container. Every restart or redeploy wipes everything. (A **persistent volume** fixes only this one.) |
| 2 | **Horizontal scaling breaks** | Three instances behind a round-robin load balancer. You upload to #1; two minutes later the profile picture request lands on #3, which doesn't have it → `404`. Probability of hitting the right instance ≈ 1/N. Violates **statelessness** |
| 3 | **Fixed, vertically-scaled disk** | You can't "add" disks horizontally. You resize, often with downtime, and there's a **ceiling**. Uploaded files are the fastest-growing data (1,000 users × 100 MB = 100 GB/month, never shrinking) |
| 4 | **Durability** | One disk dies → files gone. **Availability** (can I reach it now?) can be recovered by waiting. **Durability** (does it exist anywhere?) cannot. Durability needs **replication**, checksums, repair processes. A hard engineering area you shouldn't build for a to-do app |
| 5 | **Serving downloads** | A 50 MB file to a 2 Mbps mobile user takes ~3.5 min, occupying a server worker (goroutine/event-loop slot) as an "expensive, slow CDN". Servers aren't built for this |
| 6 | **No transaction between files and DB rows** | Write file then insert row; if the insert fails → orphan file. Insert row then write file; if that fails → row with a dead path. (Object storage doesn't magically fix this, but makes it easier) |

Also the **buffering memory** problem: a framework that reads a 4 GB upload into memory on a 4 GB instance → out-of-memory → container killed.

> The wrong assumption: *a file is something our application owns.* Files have a completely different shape, access pattern and cost profile than JSON, and need a different storage system: **object storage**.

---

## Three kinds of storage

[P1 25:29](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=1529s)

| | Block storage | File storage | Object storage |
|---|---|---|---|
| Interface | Raw array of fixed-size **blocks** (512 B–4 KB), numbered. Read/write block *N*. No names, no folders | **POSIX** file system on top of blocks: directories, files, paths, permissions, inodes; open/close/seek/overwrite 4 bytes/atomic rename/locking/append | Just **4 operations**: PUT (by name), GET (by name), DELETE, LIST |
| Examples | SSD, **Amazon EBS** | ext4, NTFS; network FS: NFS | **S3**, Cloudflare R2, Backblaze B2, MinIO |
| Latency | **µs**; databases sit on it | µs–ms | **ms to tens of ms** (every op is an HTTP request over a network) |
| Limits | Attaches to **one machine at a time**; too primitive to build apps on | Rich features need **coordination** (atomic rename, consistent partial writes, locking) which doesn't scale across data centres | No in-place edit, no rename, no folders, no locks |
| Scales to | A machine | A machine / NFS cluster | Effectively **unlimited** |

---

## What object storage is

[P1 31:39](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=1899s)

Object storage asks: *what's the **minimum** interface we can get away with so it scales without limit?*

**Trade-offs you accept:**

- **Immutable**: you can't modify, append or seek-update. Change 1 byte of a 1 GB object → re-upload the whole object under the same key.
- **No real directories.**
- **No rename**: copy to the new key + delete the old one (costs the full size).
- **No locks.**
- **Higher latency.**

**What you get:**

- Nothing to coordinate → a whole class of distributed-systems problems vanishes.
- No tree to maintain, lock or rebalance.
- Each operation is a **self-contained HTTP request**, so any of 10,000 servers can serve it → unlimited horizontal scaling.
- **Infinite capacity**, no provisioning or resizing.
- S3's **11 nines of durability**.
- Directly addressable over HTTP by any device with no server in between.

(This "give up mutability to scale infinitely" trade-off recurs across distributed systems: caching, stateless servers, immutable deployments.)

### Anatomy of an object

[P1 38:17](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=2297s)

| Part | Description |
|---|---|
| **Key** | A string. The object's entire identity |
| **Value** | A blob of bytes |
| **System metadata** | size, last-modified, content type, **ETag** |
| **User metadata** | your own key-value pairs (e.g. `uploaded-by: <user id>`) |

Objects live in a **bucket** (a named container; bucket name must be globally unique on the platform).

### Folders are an illusion

[P1 39:52](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=2392s)

The console shows `uploads/ → 2026/ → cat.png`. In reality there is **one object whose key is the string `uploads/2026/cat.png`**. The `/` is an ordinary character.

"Folders" come from **`ListObjects` with a `prefix` and `delimiter`**: the server finds all keys with that prefix, chops each at the next delimiter and reports each distinct chunk once as **`CommonPrefixes`**. Folders are a **group-by on strings**.

Consequences:

- "Moving a folder" with 40,000 objects = 40,000 copies + deletes: slow and costs money.
- You **can't have an empty folder** (tools fake it with a zero-byte object ending in `/`).

---

## Key design

[P1 43:29](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=2609s)

Since the key *is* the identity:

1. **Never use the user's filename as the key.**
   - **Collisions:** `resume.pdf` from two users → the second silently overwrites the first (`PUT` replaces without warning).
   - **Path traversal:** filenames can contain dots/special characters.
   - **Weird characters:** emoji, right-to-left override, null bytes.
   - Instead **generate the key yourself** (random) and store the original filename in your DB (and optionally user metadata).
2. **Put randomness at the front** of the key. The index is partitioned by key prefix. A time-stamped prefix (`uploads/2026-01-01-10/...`) sends an hour of writes to the same partition: a **hot partition** → `503 Slow Down`. Use high entropy early (UUID, tenant ID, hash prefix). (AWS auto-partitioning has improved, but it costs nothing to do right.)
3. **Make the key carry ownership**: `tenantId/userId/objectId`. Then authorization can be a **string comparison in middleware** before touching S3, and you can scope credentials to one prefix, saving a DB round-trip.

---

## How it works inside

[P1 49:12](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=2952s)

A `PUT` of 100 MB lands on one of thousands of **stateless front-end nodes**, which authenticates it (recomputes the signature). Then it splits into two systems:

| | Data plane | Metadata plane |
|---|---|---|
| Job | Store the bytes durably | Remember `bucket + key → where the shards are, size, ETag, content type` |
| Character | Huge volume, simple operations | Small volume, **enormous** operation count, must be **strongly consistent and fast** |
| Difficulty | Easier (solved for decades) | **The hard engineering problem**: a globally distributed, sorted, strongly-consistent index over trillions of keys with millisecond lookups |

The metadata plane is conceptually a **distributed B-tree**, partitioned by key range (hence hot partitions).

### Consequences

- **`LIST` is a range scan** over a distributed index: slow and expensive compared with a lookup. **Never use `LIST` in the request path.** Your **database** should be the index of "which files exist for this user". The bucket just holds the bytes.
- No atomic rename: the key decides the partition.
- Object size is returned instantly from metadata without touching drives.

### Data plane: replication vs erasure coding

[P1 52:44](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=3164s)

| | Replication | Erasure coding |
|---|---|---|
| How | 3 copies on 3 machines in different failure domains | Split into e.g. **14 data shards**, compute **6 parity shards** (Reed-Solomon) → 20 shards on 20 drives across racks/power domains/buildings |
| Overhead | ~3× (200% extra) | ~1.43× |
| Tolerates | 2 failures | **Any 14 of the 20** shards rebuild the object, so 6 simultaneous failures |
| Cost | Expensive drives | CPU to compute parity + reads on recovery. Cheaper overall |

---

## Durability is not backup

[P1 56:55](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=3415s)

S3's "11 nines" (99.999999999%) means roughly: store 10 million objects and expect to lose one every 10,000 years. That protects against **drive failure**, **not your actions**:

- `DELETE` of the wrong prefix → S3 will very durably delete it.
- A bug overwrites objects → S3 durably stores the corrupted versions.

For real protection:

1. Enable **versioning** (overwrites and deletes create new versions).
2. **Replicate to a second bucket in a different cloud account** (so a credential compromise can't reach both).

---

## Consistency and conditional writes

[P1 1:02:04](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=3724s)

- **Strong vs eventual consistency.** *Strong*: if you upload and someone reads a second later, they get it. *Eventual*: there's a delay before reads see the write.
- S3 was eventually consistent. Since **December 2020** it's **strongly read-after-write consistent**. Old advice about retry loops/sleeps after upload is obsolete.

### Conditional writes (2024)

[P1 1:03:37](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=3817s)

For ~20 years, `PUT` always overwrote silently, with no compare-and-swap. Now:

| Header | Meaning | On conflict |
|---|---|---|
| `If-None-Match: *` | Create **only if the key doesn't already exist** | `412 Precondition Failed` |
| `If-Match: <etag>` | Replace **only if the current ETag matches** what I read | `412 Precondition Failed` |

**Compare-and-swap scenario:** client A reads an object (ETag `e1`), edits locally, `PUT`s with `If-Match: e1`. Meanwhile client B already updated it (ETag now `e2`). A's request gets `412`, so **B's write is not silently lost**.

This gives a **distributed-lock-like** primitive. That's why people build leader election, coordination, and **write-ahead logs for databases on top of S3** (exactly one writer wins + strong consistency). For your app: uploads never silently overwrite an existing object.

> "S3 compatible" ≠ identical features. R2 is compatible via the AWS SDK. Backblaze B2 (S3-compatible) doesn't support conditional writes. **Check the docs** before relying on it.

---

## Upload architecture 1: through your server

[P1 1:10:20](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=4220s)

Browser → your API → S3. Pros: you see every byte (validate, scan, hash, reject), normal auth middleware, no CORS config. Fine for small files (profile picture, CSV, documents under a few MB).

**The hidden trap: buffering.** `ParseMultipartForm` (Go) or the default body parser (Node) reads the **whole file into RAM**. 20 concurrent 100 MB uploads = 2 GB → container hits its memory limit → OOM kill. It's invisible locally (small files, fast network, 16 GB laptop).

**Fix: stream.** Read the request body in chunks and write to the S3 client as chunks arrive, using one small buffer. Always wrap the body in a **size limit** before reading (`MaxBytesReader`).

| Demo: 500 MB upload, ×2 | Memory |
|---|---|
| Buffered | climbs ~500 MB |
| Streaming (same time, ~6.5 s) | flat |

**Even streamed, limits remain:** your bandwidth is the bottleneck; load balancer idle timeouts (~60 s) on slow mobile connections; **Nginx `413 Request Entity Too Large`**; serverless/API-gateway payload caps (a few MB, not raisable). Not recommended for production-grade large files.

---

## Upload architecture 2: pre-signed URLs

[P1 1:16:31](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=4591s)

> Let the **browser talk directly to the bucket**; your server never touches the bytes.

Problem: the bucket must be **private**, and you can't give the browser your credentials. The browser needs permission for **exactly one operation, on exactly one key, for the next ~5 minutes**.

A **pre-signed URL** is a normal URL with query params: algorithm, access-key ID, timestamp, expiry (seconds), signed headers, and a **signature**: an **HMAC** over a string describing the request (method, bucket, key, expiry...) computed with your secret key.

```
1. Browser asks your API for an upload URL.
2. Your API authorises, builds the URL (signing locally with the secret).
3. Browser PUTs the file directly to the bucket URL (no credentials).
4. S3 recomputes the HMAC with its copy of the secret. Match → accepted.
```

Change one character of the key or method, or use it after expiry → signature mismatch → `403`. The credential never leaves your server.

### The size bug in plain pre-signed PUT

A pre-signed `PUT` signs method/bucket/key but **doesn't limit body size**: someone given a URL for a profile picture can upload a 5 GB file. (Demo: a 300 MB file went through.)

**Fix: pre-signed POST with a policy.** You sign a JSON policy with conditions; the object store itself enforces them:

| Condition | Effect |
|---|---|
| `content-length-range` (e.g. 1 KB–5 MB) | Hard size bounds → `400 EntityTooLarge` |
| `Content-Type` starts-with `image/` | Type restriction |
| `key` starts-with `tenant/user/` prefix | Can't write outside their own folder |

Client notes: append the **`file` field last** in `FormData`, and **don't set `Content-Type`** (let the browser add the multipart boundary).

> **Always use pre-signed POST with `content-length-range`; never a plain pre-signed PUT.** A plain PUT is an unlimited write capability.

---

## The two-phase upload flow and orphan cleanup

[P1 1:24:46](https://www.youtube.com/watch?v=Ie0TjKI9cDI&t=5086s)

If the browser uploads directly, how does your server/DB find out?

```
Phase 1: POST /uploads/init
   Backend: authorize, verify, GENERATE the object key, INSERT row {status: pending, key, expected limits}
   Returns: pre-signed POST URL (+ upload id)

Browser uploads the bytes directly to the bucket.

Phase 2: POST /uploads/complete  { uploadId }       ← the upload ID, not the key
   Backend: HEAD the object (metadata call, free): exists? size OK? content type OK?
   Then UPDATE row → status: completed/ready
```

Your DB records the **intent** before any data moves.

### Cleanup (a must)

Some uploads get abandoned (closed tabs, network drops, or completion never called). So:

- A **bucket lifecycle rule** deletes anything under the `pending/` prefix after 24 h.
- A **background job** marks stale `pending` DB rows `expired`.

The **DB is the source of truth for "which files exist"**, and the **bucket is the source of truth for the bytes**. When they disagree you need a job to reconcile them.

### CORS

Browser → another origin (S3/R2/B2) is cross-origin. The **bucket's CORS config** must allow your frontend origin, `PUT`/`POST`, and **expose the `ETag` header** (needed for multipart). "Works in curl/Postman but not the browser" → check bucket CORS.

---

## Large files: multipart upload

[P2 0:00](https://www.youtube.com/watch?v=iWrVCxexUWY&t=0s)

Four reasons a single `PUT` stops working:

1. **Hard cap:** a single `PUT` ≤ **5 GB** (enforced by the API).
2. **All or nothing:** a Wi-Fi blink at 4.8 GB means starting over (20–30 min on mobile).
3. **Throughput:** one TCP connection can't saturate a link (bounded by window size / RTT, and one lost packet collapses the congestion window). A 200 Mbps line may give ~30 Mbps on one stream. Multiple parallel streams each get their own window. That's how download managers speed up.
4. **UX:** no precise progress, no pause/resume.

### The protocol: three calls (plus abort)

| Step | Call | Detail |
|---|---|---|
| 1 | **CreateMultipartUpload** | Returns an **upload ID** (a staging area, like git's staging; nothing exists at the key yet, a `GET` still `404`s) |
| 2 | **UploadPart** × N | Each part: upload ID + part number + bytes. **Independent HTTP requests**: parallel, any order, individually retryable. Each returns an **ETag** (keep them) |
| 3 | **CompleteMultipartUpload** | Send upload ID + ordered list of `(partNumber, ETag)`. Takes the same time for 50 GB as for 50 MB because it's a **metadata operation** (records which parts compose the object, doesn't copy data) |
| – | **AbortMultipartUpload** | Discards the parts |

### Part-size rules

[P2 0:08](https://www.youtube.com/watch?v=iWrVCxexUWY&t=488s)

- Hard cap **10,000 parts**. With a fixed 5 MB part size the max file is 50 GB, and a 60 GB upload fails at part 10,001 after 50 GB transferred.
- **Compute part size dynamically:** `partSize = max(defaultFloor, ceil(fileSize / 10000))`. Speaker's floor: **16–32 MB** (supports 160–320 GB with no thought).
- Don't make parts huge: part size = **retry granularity**. 16–64 MB is the sweet spot.

### The ETag trap

[P2 0:11:58](https://www.youtube.com/watch?v=iWrVCxexUWY&t=718s)

For a single `PUT`, ETag = MD5 of the content. For **multipart**, ETag = MD5 of the concatenated part MD5s plus `-<number of parts>` (e.g. `...-42`). So "hash locally and compare to ETag" **breaks**.

For integrity use S3's **checksum feature** (CRC32C/SHA-256 per part; the SDK can request a whole-object checksum), or compute your own SHA-256 client-side, send it as user metadata, and verify in a background job (which also gives free **deduplication**).

### Abort incomplete uploads: lifecycle rule

[P2 0:14:01](https://www.youtube.com/watch?v=iWrVCxexUWY&t=841s)

Abandoned multipart uploads leave parts in the staging area. You **keep paying forever**, and they **don't appear in the console or object listings** (the demo showed 3 forgotten uploads totalling ~460 MB). Find them with `list-multipart-uploads`.

> **On every bucket from day one, add a lifecycle rule: abort incomplete multipart uploads after 7 days.**

---

## Combining multipart + pre-signed URLs; resuming

[P2 0:16:06](https://www.youtube.com/watch?v=iWrVCxexUWY&t=966s)

```
1. Browser → POST /uploads/init  {filename, size, type}
2. Backend: authN/Z, compute part size, CreateMultipartUpload, INSERT row (pending),
            pre-sign a URL per part, return {uploadId, partSize, urls}
3. Browser: uploads N parts in parallel directly to the bucket (N ≈ 4–6)
4. Browser → POST /uploads/complete {uploadId, parts:[{partNumber, etag}]}
5. Backend: CompleteMultipartUpload → HeadObject to verify → row = ready
            → emit "file.uploaded" event to a queue (audit log / websocket notify)
```

Notes:

- **Don't pre-sign 10,000 URLs up front.** Sign in batches of ~100 as the browser progresses (otherwise `/init` returns megabytes of JSON).
- **4–6 concurrent parts** is the sweet spot. More just fights for bandwidth.
- `file.slice(start, end)` returns a **lazy `Blob`**, so browser memory ≈ `partSize × concurrency`, not the file size.
- The **bucket CORS must expose `ETag`**, or JS can't read it and can't complete.
- **Resume for free:** after a refresh the client asks "do I have a pending upload for this file?"; the backend finds the pending row, calls **`ListParts`** with the upload ID to see which parts the bucket has, and returns fresh pre-signed URLs only for the **missing** parts.

(Demo: a 2 GB file → 16 MB parts, 125 parts, 5 in flight at a time. Progress comes from computing completed/in-flight parts rather than the server. Final `complete` goes to your API, not S3.)

---

## Downloads

[P2 0:25:25](https://www.youtube.com/watch?v=iWrVCxexUWY&t=1525s)

> **Don't relay downloads through your server.** It ties up a worker for the whole transfer, you pay bandwidth (egress), and it's slower than a CDN.

Three patterns:

| Pattern | When |
|---|---|
| **1. Public content via CDN** | Public assets (images, CSS/JS bundles): put a **CDN in front of a private bucket** (only the CDN can read), set a long `Cache-Control`, use **content-addressed keys** (key = hash of content, so a new URL when content changes). Cheapest and fastest |
| **2. Pre-signed GET** | Private files: authorize on your API, return a URL valid ~5–15 min. **Problem:** each URL has a different signature/timestamp, so the CDN sees a different resource each time → **0% cache hit rate**. Fix A (cheap): sign to a coarse boundary (e.g. "valid until top of the next hour") so users within the hour share a URL. Fix B (proper): use the **CDN's own signed URLs / signed cookies**, so authorization is at the edge and the CDN controls caching |
| **3. Signed cookies for video** | A video session fetches hundreds of segments. Signing each via your backend would be absurd. Let the CDN handle it |

---

## Range requests and video streaming

[P2 0:32:32](https://www.youtube.com/watch?v=iWrVCxexUWY&t=1952s)

HTTP **range requests**: `Range: bytes=1000000-2999999` → server replies **`206 Partial Content`** with `Content-Range: bytes 1000000-2999999/TOTAL`. Invalid range → **`416 Range Not Satisfiable`**. These are plain HTTP semantics, supported natively by S3.

Uses:

- **Video seeking:** drag the scrubber to 30:00 → the browser uses the file's index to find the byte offset and requests from there, without downloading the first 30 minutes.
- **Resumable downloads:** retry from byte 400,000,000.
- **Parallel downloads:** open ~6 connections, each fetching a different range.
- **Peek at a file:** request the first 8 bytes; `xxd` shows the PNG header, and you know it's a PNG without downloading it.

```bash
curl -H "Range: bytes=0-99" -i http://.../object   # → 206
```

### Adaptive streaming (HLS / DASH)

One file for all users is a limitation. Netflix/YouTube-style streaming:

1. **Transcode** the original into multiple qualities.
2. **Chop** each into segments of a few seconds.
3. Write a **manifest** listing the segments.
4. The player fetches the manifest, then segments one at a time, **measuring download speed** and switching quality up/down (**adaptive bitrate**).

From the object store's perspective nothing changes: segments and the manifest are just objects. **Object storage + CDN = the serving part.** The hard part is the **transcoding pipeline** (queues + workers triggered on upload). If video is a small feature, use a hosted video service (Mux, Bunny Stream). If video is 90% of your product, build the pipeline.

---

## Cost

[P2 0:43:18](https://www.youtube.com/watch?v=iWrVCxexUWY&t=2598s)

You pay for:

1. **Storage** (per GB-month).
2. **Operations** (per request). Writes/mutations are roughly ~10× the price of reads.
3. **Egress** (data going out of the provider). **Cloudflare R2 has free egress.** Prices change often, so check pricing pages.

---

## Summary

1. Local disk fails for user files: ephemeral, breaks statelessness, fixed size, no durability, slow serving, no DB transactionality.
2. **Object storage** = key + bytes + metadata in a flat bucket. PUT / GET / DELETE / LIST. **Immutable** in exchange for infinite scale, 11-nines durability, HTTP access.
3. "Folders" are prefix/delimiter illusions. **Generate keys yourself**, random-first, ownership in the key.
4. Inside: **metadata plane** (hard, strongly consistent index) vs **data plane** (**erasure coding**, cheaper and stronger than replication).
5. **Durability ≠ backup**: use versioning and cross-account replication.
6. S3 is **strongly consistent** (2020) and has **conditional writes** (`If-None-Match`, `If-Match`; 2024) enabling locks/CAS.
7. **Never `LIST` in the request path**; the DB is the index.
8. Uploads: small → stream via server with a size limit. Large → **pre-signed POST** with a policy (size/type/prefix), a **two-phase flow** (`pending` row → `complete` + `HEAD`), cleanup jobs, CORS exposing `ETag`.
9. Big files: **multipart upload** (dynamic part size, parallel parts, resume via `ListParts`, don't trust ETag, **lifecycle rule to abort incomplete uploads**).
10. Downloads: never proxy. Public content via CDN with content-addressed keys; private via pre-signed GET (mind CDN caching) or CDN-signed URLs/cookies. **Range requests** (`206`) enable seeking, resuming, parallel downloads. Video uses **HLS/DASH** segments.

## Interview questions

**Q: Why not store user uploads on the app server's disk?**
Ephemeral container storage, breaks horizontal scaling (state on one instance), fixed capacity, no durability, and it ties up workers serving downloads.

**Q: Block vs file vs object storage?**
Block: raw numbered blocks, fastest, attaches to one machine (databases). File: POSIX hierarchy on blocks (rich ops, hard to scale). Object: flat key→bytes with 4 operations, immutable, effectively infinite scale over HTTP.

**Q: Are folders real in S3?**
No. Keys are flat strings. Folders are produced by listing with a prefix and delimiter (`CommonPrefixes`).

**Q: Why not use the user's filename as the object key?**
Collisions overwrite silently, path traversal and unsafe characters. Generate your own key and store the original name in the DB.

**Q: How does object storage achieve 11 nines durability cheaply?**
Erasure coding: split into data + parity shards spread across failure domains. Any k of n shards reconstruct the object, with ~1.43× overhead instead of 3×.

**Q: Is durability a backup?**
No. It protects against hardware loss, not deletes or overwrites. Use versioning and cross-account replication.

**Q: What are `If-None-Match: *` and `If-Match` for?**
Conditional writes: create only if the key doesn't exist, or replace only if the ETag still matches. Prevents silent overwrites and enables compare-and-swap/locking.

**Q: Why should uploads use pre-signed URLs?**
The bytes go straight from the browser to the bucket, so your server doesn't buffer or relay them. The URL is a time-limited, scoped, signed capability.

**Q: What's wrong with a plain pre-signed PUT, and what replaces it?**
It can't limit body size. Use a pre-signed POST with a policy (`content-length-range`, content type, key prefix).

**Q: Describe the production upload flow.**
Init (create `pending` row, generate key, return pre-signed POST/multipart URLs) → direct upload → complete (verify with `HEAD`, mark `ready`) → background cleanup of abandoned uploads.

**Q: How does multipart upload work, and what are its limits?**
Create → upload parts in parallel (each returns an ETag) → complete with the ordered list. Max 10,000 parts, 5 GB per part limit on single PUT; compute part size from file size.

**Q: Why is the multipart ETag not an MD5 of the file?**
It's the MD5 of the part MD5s with a `-N` suffix. Use checksums or your own SHA-256 for integrity.

**Q: What lifecycle rule must every bucket have?**
Abort incomplete multipart uploads after ~7 days. They're invisible in listings but billed.

**Q: How do range requests enable video seeking?**
The player maps a timestamp to a byte offset and sends `Range: bytes=...`, and the server answers `206 Partial Content`, so no need to download earlier data.

**Q: Why avoid `LIST` on a bucket in an API handler?**
It's a range scan over a distributed index: slow and costly. Keep your own file index in the database.

**Q: Why might pre-signed GET URLs defeat CDN caching, and how do you fix it?**
Each URL has a unique signature, so the CDN sees distinct resources (0% hit rate). Sign to coarse expiry boundaries or use CDN-native signed URLs/cookies.
