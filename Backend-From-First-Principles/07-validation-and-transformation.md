# 07 — Validations and Transformations

**Video:** [#9 Validations and transformations for backend engineers](https://www.youtube.com/watch?v=qedj_JjjL-U) (43m)

Validation and transformation are the gate at the front of your server. They're about data integrity and security: make sure everything the client sends is in the exact shape your business logic expects *before* any real work happens.

---

## Contents

1. [Where validation sits](#where-validation-sits)
2. [Why validate at the entry point](#why-validate-at-the-entry-point)
3. [Types of validation](#types-of-validation)
4. [Transformation](#transformation)
5. [Client-side vs server-side validation](#client-side-vs-server-side-validation)
6. [Summary](#summary)
7. [Interview questions](#interview-questions)

---

## Where validation sits

[0:00](https://www.youtube.com/watch?v=qedj_JjjL-U&t=0s)

A typical backend has three layers:

```
Client
  │  HTTP request (JSON, query params, path params, headers)
  ▼
Route matching ──► Controller layer     ← VALIDATE + TRANSFORM here, first
                       │
                   Service layer        ← business logic (calls repos, sends email, webhooks...)
                       │
                   Repository layer     ← DB queries, Redis, any persistent storage
```

| Layer | Job |
|---|---|
| **Repository** | Talks to the database (queries, inserts, deletes). |
| **Service** | Executes the business logic: call repo methods, send notifications/emails, store data, fire webhooks. |
| **Controller** | Handles **HTTP concerns**: what status code to return, response format, what data comes in and goes out. Calls the service and returns its result. |

The controller and service are separate so that HTTP details stay in one layer.

Validation and transformation run **right after the route is matched, before any significant logic** in the controller or service. They're usually one reusable **middleware or utility** driven by a **schema**.

---

## Why validate at the entry point

[4:38](https://www.youtube.com/watch?v=qedj_JjjL-U&t=278s)

Everything a client sends (JSON body, query params, path params, headers) must be in the format your API expects **before it enters your system**.

Example requirement: the body has a field `name`, a string, 5 to 100 characters. The validation step walks the schema:

1. Is `name` present? If not → error "name is required".
2. Is it a **string**? If not → error "expected string, received boolean".
3. Is its **length** in 5..100? If not → error "too long" / "too short".

### What happens without it

[9:47](https://www.youtube.com/watch?v=qedj_JjjL-U&t=587s)

Client sends `{ "name": 0 }`:

```
Controller (no validation) → Service → Repository → INSERT INTO books (name) VALUES (0)
                                                         │
                       Postgres: column "name" is TEXT NOT NULL → ERROR
                                                         ▼
                                   Unhandled DB error → client gets 500
```

The DB's type constraint catches it, but too late. The client gets **500 Internal Server Error**, which says "something broke on our side" when the real problem was bad input. That's poor UX, wastes work, and might leave state half-changed.

With validation at the entry point:

```
{ "name": 0 } → validation fails immediately → 400 Bad Request
                "name must be a string"
```

> Validation turns **client mistakes into 4xx errors** (400 Bad Request) instead of **server failures** (500).

A side benefit: validation error messages work as **self-documenting API contracts**. Send an empty payload and the error lists every required field and its expected type.

---

## Types of validation

[15:56](https://www.youtube.com/watch?v=qedj_JjjL-U&t=956s)

The video groups them into three types. They aren't strict categories, they're a way of thinking about how strict to be.

### 1. Syntactic validation — does it look right?

Is the value in the right **structure**?

| Check | What the algorithm verifies |
|---|---|
| **Email** | `local-part` `@` `domain` `.` `TLD` |
| **Phone** | country code + the right number of digits for that country |
| **Date** | follows the expected format (e.g. `YYYY-MM-DD`) |

Demo: `POST /api/valid/syntactic` with `{}` → errors for all three required fields. Add random strings → "invalid email format", plus "phone: expected string, received number". Fix those → `200`.

### 2. Semantic validation — does it make sense?

[17:29](https://www.youtube.com/watch?v=qedj_JjjL-U&t=1050s)

The structure is fine, but is the **meaning** valid?

| Field | Rule |
|---|---|
| Date of birth | cannot be in the **future** |
| Age | must be realistic, e.g. 1 to 120. `365` or `430` is rejected |

Demo: `dateOfBirth: 2026-...` → "date of birth cannot be in the future". `age: 430` → "number must be less than or equal to 120".

### 3. Type validation — is it the right data type?

[19:32](https://www.youtube.com/watch?v=qedj_JjjL-U&t=1172s)

Is the field a string / number / boolean / array / object? And you can go deeper: **each element of an array must be a string**.

Demo: a payload with `stringField`, `numberField`, `arrayField`, `booleanField`. Send strings everywhere → "expected number received string", "expected array received string", ... Fix one at a time. Also an inner check: "array element 0 expected string, received number".

### 4. Complex validation (relationships and conditions)

[30:12](https://www.youtube.com/watch?v=qedj_JjjL-U&t=1812s)

Rules that involve more than one field:

| Kind | Example |
|---|---|
| **Cross-field (relationship)** | `password` and `passwordConfirmation` must match (otherwise "passwords don't match") |
| **Conditional** | `partner` is optional unless `married` is `true`, then "partner name is required when married is true" |
| **Field constraint** | password must be at least 8 characters |

> How strict and how specific you are is a design decision. The only rule is that your service layer's expectations are guaranteed to hold by the time it runs.

---

## Transformation

[20:34](https://www.youtube.com/watch?v=qedj_JjjL-U&t=1234s)

Validation answers "is it in the expected form?". **Transformation** changes the data into the form the service layer wants. It often happens **before** validation (to make validation possible) or **after** (to clean up).

### The query-parameter problem

`GET /bookmarks?page=2&limit=20`

Rules you want: `page` is a number, 1 < page < 500; `limit` is a number, 1 < limit < 10,000.

But **all query-param values arrive as strings**: `"2"`, `"20"`. If you validate "is it a number?" right away, **it fails**, even though the client did nothing wrong.

So you first **cast** (force one data type into another) the string into a number, then run the number validations.

```
"2"  ──cast──►  2  ──validate (number, > 0, < 500)──►  OK
```

### Other transformations (from the demo)

[33:21](https://www.youtube.com/watch?v=qedj_JjjL-U&t=2001s)

Input sent by the client → what the service sees:

| Field | Sent | After transformation |
|---|---|---|
| Email | `TesT@Example.COM` | `test@example.com` (**lower-cased**) |
| Phone | `919876543210` | `+919876543210` (**prefix added**) |
| Date | some string format | converted to the format the service uses |

So transformations include type casting, normalising case, adding prefixes, reformatting dates, and setting defaults.

> **Put validation and transformation in one pipeline** so all the "input data layer" rules live in one place. You don't have to hunt across files to see what the data must look like and what's done to it before business logic.

---

## Client-side vs server-side validation

[37:28](https://www.youtube.com/watch?v=qedj_JjjL-U&t=2248s)

A common mistake: *"The frontend already validates, so I don't need to on the backend."*

| | Frontend validation | Backend validation |
|---|---|---|
| **Purpose** | **User experience**: instant feedback in the form | **Security and data integrity** |
| **Skippable?** | Yes. Anyone can bypass it | No. It's the real gate |
| **Needed?** | Yes, for good UX | **Always**, mandatory |

Why you can't rely on the frontend: your API can be called by **Postman, Insomnia, curl, a mobile app, another service**. None of them run your form validation. If the backend trusted the frontend, a new client would break the server.

> **Rule:** treat the server as if there is no frontend validation at all. Be as strict and specific as you need, independent of what the client does.

Demo: in the frontend, typing a bad email and clicking submit shows an error and **makes no API call**. Fix it and the call is made, returning `200`. Both layers cooperate: frontend for speed and clarity, backend for safety.

---

## Summary

1. Validation and transformation run **first**, after route matching and before controller/service logic.
2. They protect **data integrity and security** and turn bad input into **400**, not **500**.
3. Types: **syntactic** (format of email/phone/date), **semantic** (does it make sense: no future birthdate, age ≤ 120), **type** (string/number/array/boolean, even per-element), **complex** (cross-field and conditional).
4. **Transformation** adjusts data to what the service needs: cast query-param strings to numbers, lower-case emails, add `+` to phones, reformat dates.
5. Keep both in **one pipeline** driven by a schema.
6. **Frontend validation = UX. Backend validation = security.** You need both.
7. Validation errors double as living documentation of the API.

## Interview questions

**Q: Why validate at the controller entry rather than letting the DB reject bad data?**
Failing early returns a clear 4xx and avoids partial work. A DB error bubbles up as a confusing 500, wastes resources, and can leave state inconsistent.

**Q: 400 vs 500 on bad input?**
400 means the client sent something invalid, so fix the request. 500 means the server failed unexpectedly. Bad input should never produce a 500.

**Q: Syntactic vs semantic validation?**
Syntactic: does the value have the right form (valid email/date/phone)? Semantic: does it make sense (a birthdate in the future is a valid date but not a valid birthdate).

**Q: Why are query parameters a validation gotcha?**
They're always strings. You must transform (cast) them to the expected type before validating numeric constraints.

**Q: Frontend already validates. Do I still need backend validation?**
Yes. Frontend validation is a UX nicety and can be bypassed. The API must defend itself against any client.

**Q: What's the difference between validation and transformation?**
Validation checks whether data meets rules and rejects it if not. Transformation changes the data (cast types, normalise, set defaults) so it is in the form business logic expects.

**Q: How do you check "password must equal confirm password" or "partner required if married"?**
Cross-field and conditional (complex) validation, expressed in the validation schema or pipeline with access to multiple fields.
