# 04 — Routing: How Requests Find Their Way Home

**Video:** [#6 What is Routing in Backend? How Requests Find Their Way Home](https://www.youtube.com/watch?v=SubuU1iOC2s) (24m)

Routing is how the server decides *which code* handles an incoming request. The lecture builds from the idea of "method + route" to static, dynamic and nested routes, path vs query parameters, versioning and deprecation, and catch-all routes.

---

## Contents

1. [The what and the where of a request](#the-what-and-the-where-of-a-request)
2. [Static routes](#static-routes)
3. [Dynamic routes and path parameters](#dynamic-routes-and-path-parameters)
4. [Query parameters](#query-parameters)
5. [Path vs query parameters](#path-vs-query-parameters)
6. [Nested routes](#nested-routes)
7. [Route versioning and deprecation](#route-versioning-and-deprecation)
8. [Catch-all routes](#catch-all-routes)
9. [Summary](#summary)
10. [Interview questions](#interview-questions)

---

## The what and the where of a request

[0:00](https://www.youtube.com/watch?v=SubuU1iOC2s&t=0s)

From the previous video (HTTP):

| Part | Expresses | Example |
|---|---|---|
| **HTTP method** | The **what**: your intent / action | `GET` = fetch, `POST` = add, `PUT/PATCH` = update, `DELETE` = remove |
| **Route** (URL path) | The **where**: which resource the action applies to | `/users` |

```
GET /users      →  "fetch ... users"   →  server returns an array of users
```

The server takes the method and the route, **maps the combination to a handler** (a set of instructions), runs the business logic and database work, and returns the result.

> **Routing = mapping a URL (plus method) to server-side logic.** That's all it is.

### Method + route is the key

[3:37](https://www.youtube.com/watch?v=SubuU1iOC2s&t=217s)

```
GET  /api/books   →  handler A  (list books)
POST /api/books   →  handler B  (create a book)
```

Same route, different methods. The server builds a unique key from the two, and they never clash. The method is what separates them.

---

## Static routes

[4:37](https://www.youtube.com/watch?v=SubuU1iOC2s&t=277s)

A route with **no variable part**: always the same constant string.

```
GET  /api/books
POST /api/books
```

Nothing changes between requests, so it always maps to the same handler and returns the same kind of data. The two above are static routes.

---

## Dynamic routes and path parameters

[5:09](https://www.youtube.com/watch?v=SubuU1iOC2s&t=309s)

A route with a **variable segment**:

```
GET /api/users/123
```

`123` is the user's ID. The server extracts it from the path, fetches that one user from the DB and returns it.

How the server declares it (every language and framework does roughly this):

```
router.get("/api/users/:id", handler)
```

- `GET` must match the method.
- `/api/users/` must match literally.
- `:id` is a **placeholder**: any string in that slot is accepted and exposed to the handler as `id`.

> The `:name` convention for a dynamic segment is near-universal (Node, Go routers, Rust, Java, Python frameworks...). Some use `{id}` instead, but the idea is identical.

**Everything in a path is a string.** `123` looks like a number but arrives as the string `"123"`. Convert it yourself when you need a number (that is a *transformation*, see the validation notes).

It reads like English: "GET me data, from `/api/users`, for the user with ID 123". That human-readable shape is a core idea of REST.

The dynamic part is called a **path parameter** (or route parameter), because it sits in the path after a `/`.

---

## Query parameters

[8:45](https://www.youtube.com/watch?v=SubuU1iOC2s&t=525s)

```
GET /api/search?query=some+value
        └─ route ─┘ └ query param ┘
```

- Everything after `?` is the **query string**: key-value pairs (`key=value`), joined with `&`.
- The server still routes using the **method and the path** (`/api/search`). The query string is *not* part of matching. It's extra data passed to the handler.

### Why they exist

`POST`/`PUT` requests have a **body** to carry data. In REST, **`GET` requests don't have a body**. So how do you send extra values with a GET?

You could stuff it into the path (`/api/search/some-value`). It technically works, but path parameters have a *semantic* job (they identify a resource). Overloading them hurts maintainability and defeats REST's goal of expressive endpoints.

So: **query parameters carry metadata/options about the request** on GET.

### Typical uses

| Use | Example |
|---|---|
| **Pagination** | `/api/books?page=2&limit=20` |
| **Filtering** | `/api/books?genre=fantasy` |
| **Sorting** | `/api/books?sort=price&order=desc` |
| **Search text** | `/api/search?query=...` |

### Pagination walkthrough

[12:26](https://www.youtube.com/watch?v=SubuU1iOC2s&t=746s)

`GET /api/books` returns:

```json
{
  "data": [ /* 20 books */ ],
  "total": 100,
  "currentPage": 1,
  "totalPages": 5
}
```

- The server has a **default limit** (say 20) and defaults to page 1, so no query params were needed.
- The response includes metadata (`total`, `currentPage`, `totalPages = 100 / 20`).
- The client uses that metadata to ask for more: `GET /api/books?page=2`.

---

## Path vs query parameters

| | Path parameter | Query parameter |
|---|---|---|
| Looks like | `/users/123` | `/users?role=admin` |
| Identifies | **Which resource** | **How to retrieve / filter / shape** the response |
| Part of route matching? | Yes | No |
| Typical | IDs | pagination, filter, sort, search |

A quick test: if removing it changes *what thing* you're talking about, it's a path parameter. If it only changes how you see a collection, it's a query parameter.

---

## Nested routes

[15:01](https://www.youtube.com/watch?v=SubuU1iOC2s&t=901s)

Not a separate routing mechanism, just a **practice**. REST APIs nest resources to express relationships:

```
GET /api/users                       → all users
GET /api/users/123                   → user 123
GET /api/users/123/posts             → all posts of user 123
GET /api/users/123/posts/456         → post 456 of user 123
```

Each level is a **different route** matched to a **different handler**, even though they share a prefix. Stopping at a different depth expresses a different meaning:

| Route | Meaning |
|---|---|
| `/users` | all users |
| `/users/123` | one user |
| `/users/123/posts` | posts belonging to that user |
| `/users/123/posts/456` | one specific post of that user |

Seen in almost any API of medium complexity.

---

## Route versioning and deprecation

[18:41](https://www.youtube.com/watch?v=SubuU1iOC2s&t=1121s)

```
GET /api/v1/products    →  { data: [ { id, name,  price } ] }
GET /api/v2/products    →  { data: [ { id, title, price } ] }
```

The shape changed (`name` → `title`). That's a **breaking change** for existing clients.

**Why:** requirements change. Your API served a web app. Now you also serve a React Native / Android / Flutter app that needs a different response. You have two choices:

1. Invent new routes (`/api/new-products`). Messy, and the naming says nothing about intent.
2. **Version the API.** `v1` keeps serving the old shape, `v2` serves the new one.

**Benefits:**

- It states the intent clearly: this is the v2 contract.
- You don't redesign your routes for each change.
- You can **deprecate** safely: announce that v1 is deprecated once v2 ships, give client engineers a **migration window**, then remove v1 (and v2 becomes the baseline).

> Versioning plus deprecation gives you a stable workflow for introducing breaking changes without breaking existing clients.

---

## Catch-all routes

[22:16](https://www.youtube.com/watch?v=SubuU1iOC2s&t=1336s)

Request: `GET /api/v3/products` where there is no handler for `v3`.

By default you'd get an empty or meaningless response. Instead, register a handler for `*` **last**, after all real routes:

```
router.all("*", (req, res) => res.status(404).json({ message: "Route not found" }))
```

- Routes are tried in order. Anything that matches none of them falls through to `*`.
- The handler returns a **friendly "this route doesn't exist"** message (and a `404`).

This is a catch-all (wildcard) route.

---

## Summary

1. **Method = what** (intent), **route = where** (resource). Together they select a handler.
2. **Static route**: fixed string (`/api/books`).
3. **Dynamic route**: has a placeholder (`/api/users/:id`). The captured piece is a **path parameter**. It's always a string.
4. **Query parameters** (`?page=2&limit=20`) are key-value options on the request, mainly for GET: pagination, filter, sort, search. They are not part of route matching.
5. **Nested routes** express relationships (`/users/123/posts/456`), one handler per depth.
6. **Versioning** (`/v1`, `/v2`) allows breaking changes. **Deprecation** gives clients a migration window.
7. **Catch-all** (`*`) at the end returns a clean "not found" for anything unmatched.

## Interview questions

**Q: What is routing?**
Mapping an incoming request (method + URL path) to the server-side handler that should process it.

**Q: Why does the HTTP method matter in routing?**
The method plus the path forms the unique key. `GET /books` and `POST /books` share a path but map to different handlers.

**Q: Path parameter vs query parameter?**
Path parameters identify a resource and are part of the matched route (`/users/123`). Query parameters carry optional modifiers like pagination, filtering and sorting and are not part of matching (`/users?page=2`).

**Q: Why do GET requests use query parameters?**
GET requests don't carry a body in REST. Query parameters are the clean way to pass extra data without overloading path semantics.

**Q: What type is a path parameter?**
A string, always. Convert and validate it before use.

**Q: How would you introduce a breaking response change without breaking mobile clients on the old format?**
Version the API (`/v2/...`), keep `/v1` running, announce deprecation with a migration window, then retire v1.

**Q: What is a catch-all route and where do you register it?**
A wildcard route (`*`) registered after all others to return a consistent 404 for unmatched paths.
