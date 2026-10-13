# 09 — Complete REST API Design

**Video:** [#11 Complete REST API Design](https://www.youtube.com/watch?v=RG6q57DwV8Y) (2h 03m)

One of the most important videos in the series. It covers where REST came from, what the words mean, the rules for URLs and HTTP methods (including idempotency), and then designs a full API for a project-management app: CRUD, pagination, sorting, filtering, custom actions, status codes and consistency rules.

The goal is not to invent new standards. It is to **extract rules from the existing ones** so you stop re-debating the same questions: plural or singular? `PUT` or `PATCH`? Which method for a non-CRUD action? Which status code?

---

## Contents

1. [Why the confusion exists](#why-the-confusion-exists)
2. [A short history of REST](#a-short-history-of-rest)
3. [What "Representational State Transfer" means](#what-representational-state-transfer-means)
4. [Anatomy of an API URL](#anatomy-of-an-api-url)
5. [Idempotency](#idempotency)
6. [HTTP methods through the idempotency lens](#http-methods-through-the-idempotency-lens)
7. [How to start designing an API](#how-to-start-designing-an-api)
8. [Designing the CRUD endpoints](#designing-the-crud-endpoints)
9. [Pagination, sorting, filtering](#pagination-sorting-filtering)
10. [Update, get one, delete](#update-get-one-delete)
11. [404 vs empty list](#404-vs-empty-list)
12. [Custom actions](#custom-actions)
13. [Consistency and conventions](#consistency-and-conventions)
14. [Final checklist](#final-checklist)
15. [Summary](#summary)
16. [Interview questions](#interview-questions)

---

## Why the confusion exists

[0:00](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=0s)

Even experienced developers argue about:

- Plural or singular path segments?
- `PUT` or `PATCH` to update?
- Which method for a **custom action** that isn't create/read/update/delete?
- Which status code in which situation?

The standards were written when the web was made of **multi-page applications (MPAs)**. Today we mostly build **single-page applications (SPAs)**: the browser downloads all the JS once, routes on the client, and talks to JSON APIs. The old guidance doesn't map neatly, so we need a stable set of conventions.

---

## A short history of REST

[4:43](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=283s)

- **1990** — Tim Berners-Lee starts the World Wide Web. Within about a year he invents: **URI**, **HTTP**, **HTML**, the first **web server**, the first **web browser**, and the first WYSIWYG HTML editor. We still use evolved versions of all of them (HTTP/1.1 → 2 → 3).
- The web grew exponentially. He hadn't planned for that scale and it was heading for breakdown.
- **~1993** — **Roy Fielding** (co-founder of the Apache HTTP Server project) proposed **constraints** to make the web scalable.
- He worked with Berners-Lee on **HTTP/1.1**, the first standard version.
- **2000** — Fielding named the architectural style **REST** in his PhD dissertation. (Worth reading the original paper.)

### Fielding's six constraints

| # | Constraint | Meaning |
|---|---|---|
| 1 | **Client-server** | Separation of concerns. The client handles UI/UX, the server handles data and business logic. Each evolves independently. |
| 2 | **Uniform interface** | A standard way for components to talk. Four sub-constraints: *resource identification*, *manipulation of resources through representations*, *self-descriptive messages*, *hypermedia as the engine of application state* (HATEOAS). |
| 3 | **Layered system** | The architecture is hierarchical layers. Each only sees the layer directly below. This lets you insert load balancers, proxies, etc. without changing core behaviour. |
| 4 | **Cacheable** | Responses must be labelled cacheable or non-cacheable. Cuts server load, improves latency. |
| 5 | **Stateless** | Each request must carry **everything** needed to process it. The server keeps no client context between requests. Any server can handle any request, so you can add servers behind a load balancer (round robin, etc.). |
| 6 | **Code on demand** (optional) | The server can send executable code (e.g. JavaScript) to extend the client. Rarely emphasised. |

---

## What "Representational State Transfer" means

[15:00](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=900s)

| Word | Meaning |
|---|---|
| **Representational** | A resource (data or object) is exposed through a **representation**: JSON, XML or HTML. The same `user` can be JSON for an API client and an HTML page for a browser. |
| **State** | The current condition/attributes of a resource. E.g. a shopping cart's state = items + quantities + total price. |
| **Transfer** | Moving those representations between client and server over HTTP using standard methods (GET, POST, PUT, DELETE, ...). |

So REST = clients and servers exchange **representations of a resource's state**, following the constraints above.

---

## Anatomy of an API URL

[21:42](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=1302s)

A website URL:

```
https://example.xyz/path/to/page?key=value#section
└scheme┘ └authority┘ └ resource ┘ └ query ┘ └fragment┘
```

- **Scheme**: `http` or `https`
- **Authority**: domain (with optional subdomain)
- **Path**: the resource. `/` represents a **hierarchical relationship**
- **Query**: key-value pairs (mostly on GET)
- **Fragment**: jumps the browser to a section of the page

The industry-standard shape for an API URL:

```
https://api.example.com/v1/books
        └ subdomain ┘ └ver┘ └resource┘
```

### URL design rules

[24:48](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=1488s)

| Rule | Example |
|---|---|
| **Resource names are plural nouns**, even when fetching one item | `/books` and `/books/42`, **not** `/book/42` |
| Use `/` for **hierarchy** | `/organizations/123/projects` |
| **No spaces or underscores** in the path | use hyphens: `/books/harry-potter` |
| **Lowercase** only | URLs travel across servers, OSes and clients; case mismatches cause bugs |
| Slugs: lowercase and replace spaces with `-` | "Harry Potter" → `harry-potter` |

`/books` means "the collection of books" (first level), `/books/42` means "one item from that collection" (second level).

---

## Idempotency

[31:00](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=1860s)

> An operation is **idempotent** if performing it once or many times has **the same effect on the server's state**.

It's about **side effects the client causes**, not whether the response bytes are identical. If another user adds a book between your two `GET /books` calls, the list changes, but your *call* caused no change. It's still idempotent.

---

## HTTP methods through the idempotency lens

[32:33](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=1953s)

The main five: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. Others: `HEAD` (headers only), `OPTIONS` (used in CORS preflight).

| Method | Purpose | Idempotent? | Why |
|---|---|---|---|
| **GET** | Retrieve data | **Yes** | No change to server state |
| **PUT** | **Replace the whole** resource with the payload | **Yes** | Applying the same full payload again leaves it unchanged |
| **PATCH** | **Partially update** a resource | **Yes** (in the typical "set field to X" case) | Setting `name` to `B` twice still leaves it `B` |
| **DELETE** | Remove a resource | **Yes** | First call deletes. Next calls return `404`, but **no new side effect** |
| **POST** | Create a resource / custom action | **No** | Each call creates another resource (new ID each time) |

### PUT vs PATCH

[35:05](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=2105s)

- **PATCH**: send only the fields to change, e.g. `{ "name": "B" }`.
- **PUT**: send the **complete** representation (id, name, createdAt, ...) which fully replaces what the server has.

People often use them interchangeably, which is okay for an internal API. For a **public** API, stick to the semantics so integrators aren't misled. In practice, SPAs send partial JSON, so **`PATCH` is the usual choice**.

### DELETE: a subtle case

First call: user deleted. Second call: user not found → `404`. But the second call **changed nothing**, so it's still idempotent.

### POST is the "open-ended" method

[43:54](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=2634s)

Any action that doesn't fit read/create/update/delete is a **custom action** and goes under `POST`. Example: `POST /send-email` with `{ "target": "a@b.com" }`.

---

## How to start designing an API

[46:27](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=2787s)

**Design the interface before writing any business logic.** It should be intuitive, delightful, and not vague.

Why follow standards? An integrator can only learn your API by reading your code or trial-and-error. If you follow common conventions, ~80% of behaviour is already known: less guesswork, fewer bugs, fewer sync calls.

### The workflow

[50:35](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=3035s)

```
1. UI wireframes / Figma / product requirements
        ↓
2. Pull out the NOUNS → these are your RESOURCES
        ↓
3. Design the DB schema (next video)
        ↓
4. List the ACTIONS per resource (CRUD + extras)
        ↓
5. Design the API interface (routes, methods, payloads, responses)
        ↓
6. Only then: implementation
```

Example: a Jira/Linear-like project-management SaaS. Nouns from the requirements: **organizations**, **users**, **projects**, **tasks**, **tags**. For the demo: organization, project, task tables.

Tool used: **Insomnia** (a lighter Postman alternative). The point is to design the *interface*, language- and framework-independent.

---

## Designing the CRUD endpoints

[58:54](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=3534s)

For each resource, the template:

| Action | Method | Route | Success status | Body |
|---|---|---|---|---|
| **List** | GET | `/organizations` | **200** | paginated list |
| **Create** | POST | `/organizations` | **201 Created** | the created entity |
| **Get one** | GET | `/organizations/:id` | **200** | the entity |
| **Update** | PATCH | `/organizations/:id` | **200** | the updated entity |
| **Delete** | DELETE | `/organizations/:id` | **204 No Content** | empty |
| **Custom action** | POST | `/organizations/:id/archive` | 200 (or 201) | depends |

Key observations:

- **List and create share the same URL**. The method tells the server which handler to use.
- **Get one, update and delete share the same URL** (`/organizations/:id`). Again, only the method differs.
- Resource name in the path: **lowercase, plural**.

### Create

```http
POST /organizations
Content-Type: application/json

{ "name": "Org 1", "status": "active", "description": "..." }
```

- The client **doesn't send** `id`, `createdAt`, `updatedAt`. The server owns those.
- Response: **`201 Created`** with the full new entity, including generated `id` and `createdAt`.

### JSON conventions

[42:00](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=2520s)

JSON field names are **camelCase** (`organizationId`, `createdAt`) in both requests and responses. Follow the JSON norm so clients don't have to guess.

---

## Pagination, sorting, filtering

[1:05:04](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=3904s)

A list endpoint should support all three.

### Pagination — why

If there are 1,000 organizations and you return all of them:

1. **JSON serialization is expensive**, and big payloads slow the API.
2. The user perceives the **delay**, even though the first screen shows 10–20 items anyway.
3. The network and the client are overwhelmed.

So the server returns a **slice** at a time.

```http
GET /organizations?page=2&limit=2
```

```json
{
  "data": [ { "name": "organization 3" }, { "name": "organization 2" } ],
  "total": 5,
  "page": 2,
  "totalPages": 3
}
```

| Field | Meaning |
|---|---|
| `data` | the current slice |
| `total` | total rows in DB (independent of the page), useful for UI like "10 of 50" |
| `page` | which page this is |
| `totalPages` | total pages. Helps infinite scroll know when to stop (when `page == totalPages`) |

`limit` = items per page, `page` = which slice. With 5 rows and `limit=2`, you get 3 pages (2, 2, 1).

### Sane defaults

[1:12:48](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=4368s)

The client shouldn't need to send obvious parameters:

| Param | Default |
|---|---|
| `page` | `1` |
| `limit` | `10` or `20` |
| `sortBy` | `createdAt` |
| `sortOrder` | `desc` (newest first) |

> **Always sort by something.** Databases don't store rows in a guaranteed order. Without an explicit sort, the same query can return a different order each time.

### Sorting

```http
GET /organizations?sortBy=name&sortOrder=asc
```

If the client passes `sortBy` only, `sortOrder` still defaults to `desc`.

### Filtering

[1:21:06](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=4866s)

```http
GET /organizations?status=archived
GET /organizations?status=active&name=Org
```

Each filterable field becomes a query param. Filters can be combined.

---

## Update, get one, delete

[1:23:44](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=5024s)

All three use **`/organizations/:id`**.

```http
PATCH  /organizations/abc123   { "status": "active" }   →  200 + updated entity
GET    /organizations/abc123                             →  200 + entity
DELETE /organizations/abc123                             →  204, empty body
```

- **PATCH** returns **200** (nothing created) and the updated resource.
- **DELETE** returns **204 No Content**: success, but nothing meaningful to send back.
- After deleting, `GET /organizations/abc123` → **404** with an "organization not found" message.

---

## 404 vs empty list

[1:30:24](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=5424s)

| Request | Result | Status |
|---|---|---|
| `GET /organizations/xyz` where `xyz` doesn't exist | Not found | **404** |
| `GET /organizations?status=nonsense` (nothing matches) | `{ "data": [], "total": 0, "page": 1, "totalPages": 0 }` | **200** |

> **Rule of thumb:** `404` is for a request about a **specific entity** that doesn't exist. A **list** that is empty is still a valid answer, so return `200` with an empty array.

---

## Custom actions

[1:32:57](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=5577s)

An action that isn't CRUD, e.g. **archive** an organization.

You might think "just `PATCH` the status to `archived`". But archiving usually means **more than a field change**: archiving the org's projects and tasks, notifying members, sending emails. That's a *business operation*, not a data update.

So it's a **custom action**:

```http
POST /organizations/:id/archive
```

The path stays hierarchical: `collection → one item → action`. Response: the organization with `status: "archived"`, **status 200** (nothing was created).

Another example: **clone a project**.

```http
POST /projects/:id/clone      →  201 Created + the new project
```

You *could* simulate cloning with a client-side "read, then create", but the server might also need to clone the tasks, email the owner, etc. So it needs its own endpoint.

> **Don't assume `POST` ⇒ `201`.** Return **201** when a new resource is created (the clone), **200** when nothing is created (the archive). Choose the status from what happened.

---

## Consistency and conventions

[1:44:42](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=6282s)

Once someone integrates your first endpoint, they **assume** everything else works the same way.

| Keep consistent | Example |
|---|---|
| **Field names across resources** | If `description` is used in `organizations`, don't call it `desc` in `projects` |
| **Route shape** | If it's `/organizations` (plural), then `/projects`, not `/project` |
| **Payload style** | camelCase JSON everywhere |
| **Response shape** | same pagination envelope on every list |

> Consistency of style is one of the most important traits of a good backend engineer. Even if you don't follow the global standard, follow *something* and stick to it.

Org of the routes: list/create share a URL, get/update/delete share a URL, custom action = id + verb.

---

## Final checklist

[1:56:35](https://www.youtube.com/watch?v=RG6q57DwV8Y&t=6995s)

1. **Interactive docs**: Swagger / OpenAPI from day one. Keep it updated. It doubles as a test playground and documentation.
2. **Intuitive and consistent**: routes, parameters, payloads, responses.
3. **Sane defaults**: on GET (page/limit/sort) and on POST (only require what is truly needed; e.g. `status` defaults to `active`).
4. **No abbreviations**: `description`, not `desc`. Integrators don't share your context.
5. **Design first, code second**: dedicate a session to the interface, independent of language and framework.

---

## Summary

1. REST comes from Roy Fielding's constraints: **client-server, uniform interface, layered, cacheable, stateless, (optional) code on demand**.
2. **Representational State Transfer** = exchanging representations of a resource's state over HTTP.
3. URLs: `https://api.example.com/v1/<plural-noun>/<id>`; lowercase, hyphens, `/` = hierarchy.
4. **Idempotent**: GET, PUT, PATCH, DELETE. **Not idempotent**: POST.
5. `PATCH` = partial update, `PUT` = full replacement, `POST` also handles **custom actions**.
6. Statuses: **200** OK, **201** created, **204** no content, **404** specific resource missing, **400** bad input.
7. List endpoints need **pagination, sorting, filtering** with sensible **defaults**. Empty list = `200`, not `404`.
8. Design from nouns in the requirements, interface first, then DB and code.
9. **Be consistent** and **document** with OpenAPI.

## Interview questions

**Q: Why are REST resource names plural, even for a single item?**
The path names the *collection* the item belongs to (`/books/42` = item 42 of the books collection). It keeps one consistent pattern and shows the `/` hierarchy.

**Q: What does idempotent mean, and which methods are?**
Repeating the request has the same effect on server state as doing it once. GET, PUT, PATCH, DELETE are; POST is not.

**Q: DELETE returns 404 the second time. Is it still idempotent?**
Yes. The second call causes no additional change. Idempotency is about server state, not identical responses.

**Q: PUT vs PATCH?**
PUT replaces the entire resource and needs a full payload. PATCH changes only the supplied fields.

**Q: Which method for a non-CRUD action such as "archive" or "send email"?**
`POST` on a verb sub-resource, e.g. `POST /organizations/:id/archive`.

**Q: Should every POST return 201?**
No. 201 means a resource was created. A custom action that creates nothing returns 200.

**Q: A list endpoint matches nothing. 200 or 404?**
200 with an empty array and `total: 0`. 404 is only for a missing *specific* resource.

**Q: Why paginate?**
To avoid expensive serialization of huge payloads, reduce latency and bandwidth, and keep clients responsive.

**Q: Why must list endpoints have a default sort?**
Databases don't guarantee row order. Without a sort, results can come back in a different order each call, which breaks pagination.

**Q: Which status for a successful DELETE?**
204 No Content, with an empty body.

**Q: What are the six REST constraints?**
Client-server, uniform interface, layered system, cacheable, stateless, code on demand (optional).

**Q: Why design the API before coding?**
The interface is a contract with other engineers. Designing it independently of language/framework (e.g. in Insomnia/Swagger) surfaces usability problems early and gives consumers predictable behaviour.
