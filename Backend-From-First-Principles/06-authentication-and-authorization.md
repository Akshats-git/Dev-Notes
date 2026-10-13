# 06 — Authentication and Authorization

**Video:** [#8 Authentication and authorization for backend engineers](https://www.youtube.com/watch?v=A95rliroC8Q) (1h 36m)

> **Authentication** answers **"who are you?"**. **Authorization** answers **"what can you do?"**.

The lecture starts with a short history (to explain *why* each mechanism exists), then covers sessions, JWTs and cookies, the four main authentication types (stateful, stateless, API keys, OAuth 2.0 / OpenID Connect), authorization with RBAC, and two security details: generic error messages and timing attacks.

---

## Contents

1. [A short history of authentication](#a-short-history-of-authentication)
2. [Three building blocks: sessions, JWTs, cookies](#three-building-blocks-sessions-jwts-cookies)
3. [Stateful authentication](#stateful-authentication)
4. [Stateless authentication](#stateless-authentication)
5. [API key authentication](#api-key-authentication)
6. [OAuth 2.0 and OpenID Connect](#oauth-20-and-openid-connect)
7. [Which one to use](#which-one-to-use)
8. [Authorization and RBAC](#authorization-and-rbac)
9. [Generic error messages](#generic-error-messages)
10. [Timing attacks](#timing-attacks)
11. [Summary](#summary)
12. [Interview questions](#interview-questions)

---

## A short history of authentication

[1:31](https://www.youtube.com/watch?v=A95rliroC8Q&t=91s)

Each era invented a mechanism because the previous one stopped scaling or was attacked.

| Era | Mechanism | Principle | Problem that ended it |
|---|---|---|---|
| Pre-industrial | A village elder **vouches** for you; handshake | **Human trust** (implicit authentication) | Doesn't scale beyond a community |
| Medieval | **Wax seals** on letters | Something you **have** (possession) | Seals could be **forged**. First "authentication bypass" |
| Industrial / telegraph | Pre-agreed **pass phrases** | Something you **know** | Static shared secret |
| 1961, MIT CTSS | **Passwords** on multi-user computers | Something you know | Stored in **plaintext**; someone printed the password file. This led to **hashing** (irreversible, fixed-length output) |
| 1970s | Diffie-Hellman → **asymmetric cryptography** (PKI); Kerberos | Shared secrets over untrusted channels; **ticket-based** auth (precursor to tokens) | |
| 1990s | **MFA**: know + have + are (password, OTP/smart card, biometrics) | Layered factors | Passwords alone weak against brute-force/dictionary attacks. Biometrics bring false positives/negatives |
| 21st century | **OAuth 2.0, JWT, zero-trust, passwordless (WebAuthn)** | Cloud / mobile / API-era needs | |
| Future candidates | Decentralised identity (blockchain), behavioural biometrics, **post-quantum cryptography** | Quantum computers will break RSA-style algorithms | |

**Hashing** in one line: a deterministic algorithm that turns any input into a **fixed-length** output. The same input always gives the same hash, and you can't reverse it.

---

## Three building blocks: sessions, JWTs, cookies

[18:32](https://www.youtube.com/watch?v=A95rliroC8Q&t=1112s)

### Sessions

HTTP is **stateless**: every request is independent. That was fine for the early static web. Then websites became dynamic: shopping carts, staying logged in. So the web needed **memory** about a user, and the **session** was invented: *a temporary server-side context for a user.*

```
1. User logs in.
2. Server creates a unique SESSION ID and stores it + user data (role, cart, ...)
   in a persistent store (database / Redis).
3. Server sends the session ID to the browser as a COOKIE.
4. Every later request carries the cookie → server looks up the session ID → knows who you are.
5. Sessions expire (e.g. 15 min); the server then creates a new one.
```

How session storage evolved: **files on the server** (doesn't scale) → **database** (survives restarts) → **distributed in-memory stores like Redis/Memcached** (fast).

### JWT (JSON Web Token)

[25:46](https://www.youtube.com/watch?v=A95rliroC8Q&t=1546s)

By the mid-2000s apps were global distributed systems and sessions hurt:

1. **Memory**: storing session data for millions of users is expensive.
2. **Replication**: syncing session data across servers/regions adds latency and consistency headaches.

So JWTs (standardised ~2015) moved the state **into the token itself**: a stateless, self-contained token carrying claims, **base64-encoded**, in three parts:

| Part | Contents |
|---|---|
| **Header** | Metadata, e.g. the signing algorithm |
| **Payload** | The claims: `sub` (user ID), `iat` (issued at), optional `name`, `role`, ... |
| **Signature** | Computed with a **secret key only the server knows**. If anyone alters the token, verification fails |

The server verifies the token with its secret and reads the user from the payload. **No lookup**.

**Advantages:** stateless (no storage cost), scalable (many servers/microservices share the secret and each verify independently), portable (URL-safe, lightweight).

**Disadvantages:**

1. **No revocation.** Whoever holds the token can impersonate the user until it expires. The only global kill-switch is changing the secret, which logs out *every* user.
2. **Can't track status** per token, since the server is stateless.

**Hybrid approach:** verify the JWT, *then* check a **blacklist** (in a DB or Redis) to block compromised tokens. The video points out the irony: if you do a storage lookup anyway, why not just use sessions, which are considered more secure? That is a common debate.

> **Practical advice (from the speaker):** it's great to implement auth yourself *to learn*. In production, for anything medium-to-complex, **use an auth provider** (Auth0, Clerk, ...). Hashing, salting and algorithm choice are their headache, unless you're very confident.

### Cookies

[38:33](https://www.youtube.com/watch?v=A95rliroC8Q&t=2313s)

A **cookie** is a small piece of data the **server stores in the user's browser**. Key properties:

- It's scoped to the server that set it. Other servers can't see it.
- The browser **automatically sends it** with every subsequent request to that server.

That makes cookies the ideal transport for auth tokens: the server sets a cookie with a session ID (or JWT) at login, and the browser returns it on every request, with no client code needed.

---

## Stateful authentication

[42:12](https://www.youtube.com/watch?v=A95rliroC8Q&t=2532s)

```
Client                         Server                              Redis/DB
  │ POST /login (email, pw) ───►│ verify credentials                    │
  │                             │ create session ID + user data ───────►│ store
  │◄── Set-Cookie: sid=...; HttpOnly ─────────────────────────────────  │
  │ any request + Cookie: sid ─►│ look up sid ─────────────────────────►│
  │                             │◄── user data / role / expiry ─────────│
  │◄── response                 │                                       │
```

- Cookie is **`HttpOnly`** so JavaScript can't read it.
- The session ID is just a cryptographically random string.

| Pros | Cons |
|---|---|
| **Centralised control**: real-time view of all active sessions | Limited scalability |
| **Easy to revoke / log out** a user instantly | Higher operational complexity in distributed, multi-region systems (latency) |
| Good for strict session requirements | |

The speaker's view: **most applications should go stateful** for security and revocation convenience.

---

## Stateless authentication

[44:46](https://www.youtube.com/watch?v=A95rliroC8Q&t=2686s)

```
Client                           Server
  │ POST /login ────────────────►│ verify credentials
  │                              │ sign a JWT {sub, role, iat} with the secret key
  │◄── JWT ──────────────────────│
  │ request + Authorization: Bearer <JWT> ►│ verify signature with secret → user known. No storage lookup
```

- Sent in the **`Authorization`** header (the standard way).
- All needed info lives **in the token**, so it's "stateless".

| Pros | Cons |
|---|---|
| Scales easily; no session store dependency | **Revocation is hard** (see above) |
| Great for distributed architectures, microservices | If a user's account is compromised, you can't invalidate their token until expiry unless you rotate the secret (logging everyone out) |
| Mobile-friendly (cookies aren't a thing there) | |

### Hybrid by client type

[49:27](https://www.youtube.com/watch?v=A95rliroC8Q&t=2967s)

Use **stateful** sessions for the browser web app, and **stateless** tokens for mobile apps and third-party/server integrations. This covers the weaknesses of both.

---

## API key authentication

[50:28](https://www.youtube.com/watch?v=A95rliroC8Q&t=3028s)

You click "Generate API key" in a platform's UI and get a long random string. You attach it (in a header, or as the server defines) to requests to **access the server programmatically**, with a set of **permissions** and an **expiry**.

**Example:** ChatGPT's UI is for humans. OpenAI's **API key** lets *your server* call the models directly, with your plan and quota applied to the key.

Why use them:

- Easy to generate.
- Perfect for **machine-to-machine** communication: no login form, no human in the loop. The server stores the key in an environment variable or secret store and sends it with each call.

```
User → your UI → your server ──(API key)──► OpenAI server → response → your server → user
```

---

## OAuth 2.0 and OpenID Connect

[58:13](https://www.youtube.com/watch?v=A95rliroC8Q&t=3493s)

### The problems

1. **Password fatigue and reuse**: one breach compromises many accounts.
2. **The delegation problem**: App A needs access to your data in Platform B. Examples: a travel app scanning your Gmail for flight tickets, a social app importing your Google contacts.

The first "solution" was **sharing your password** with App A. Disastrous:

- It grants **full access** to everything, with no way to limit permissions.
- You can't revoke access without **changing your password everywhere**.

### OAuth (2007, "Open Authorization")

Idea: **share a token with limited permission, not the password.** A password opens everything. A token can be scoped to "read contacts only".

**Roles:**

| Role | Example (Facebook importing Google contacts) |
|---|---|
| **Resource owner** | You (own the data) |
| **Client** | Facebook (wants access) |
| **Resource server** | Google (holds the contacts) |
| **Authorization server** | Google's auth server (issues tokens after you approve) |

**OAuth 1.0 flow:**

```
1. Client redirects you to the authorization server.
2. You authenticate and grant specific permissions ("Allow").
3. Authorization server gives the client a token.
4. Client uses the token to call the resource server.
```

### OAuth 2.0 (~2010)

OAuth 1.0 was complex and used error-prone cryptographic signatures. OAuth 2.0 introduced:

- **Bearer tokens** (simpler; more vulnerable if stolen).
- **Different flows by app type**:

| Flow | Use case |
|---|---|
| **Authorization code** | Server-side apps |
| **Implicit** | Browser apps (now **discouraged**, security risks) |
| **Client credentials** | **Machine-to-machine**, no user |
| **Device code** | Limited-input devices (smart TVs) |

### OpenID Connect (OIDC, ~2014)

OAuth solves **authorization** (delegated access), **not authentication** (who is this user?). OIDC sits on top of OAuth 2.0 and adds an **ID token**, typically a **JWT**, with the user's identity: user ID, issued-at, issuer, name, email.

This is what powers **"Sign in with Google / Facebook / Discord"**.

**Flow (a note-taking app with "Sign in with Google"):**

```
1. You click "Sign in with Google" → redirected to Google's authorization server.
2. You log in at Google (not at the note app) and grant permissions (profile, email, Keep notes).
3. Google returns an authorization CODE (and possibly an ID token) to the app.
4. App exchanges the code for an ACCESS TOKEN (+ ID token).
5. App uses the ID token to know who you are, and the access token to call Google
   on your behalf, but only for the permissions you granted.
```

OAuth 2.0 + OIDC turned the password-sharing mess into a secure, interconnected system.

---

## Which one to use

[1:17:11](https://www.youtube.com/watch?v=A95rliroC8Q&t=4631s)

| Type | Best for |
|---|---|
| **Stateful** (session + persistent store) | Web apps (most SaaS): user session data lives on the server |
| **Stateless** (JWT) | APIs and distributed/scalable systems; mobile clients |
| **OAuth / OIDC** | Third-party integrations and "login with Google/Facebook/Discord" |
| **API keys** | Server-to-server / machine-to-machine; single-purpose API access |

In practice, you'll use stateful and stateless most when building your own APIs.

---

## Authorization and RBAC

[1:19:46](https://www.youtube.com/watch?v=A95rliroC8Q&t=4786s)

After a user is authenticated, **not all users can do the same things**.

Example: a note-taking app. Deleted notes go to a trash for 30 days, then to a "dead zone" that only the **admin** can inspect. How do you give the admin extra powers?

**Bad idea:** a secret "god mode" string sent with each API call. If intercepted, an attacker could wipe the DB. And giving the same powers to a friend means sharing or creating more strings: unmanageable.

**Authorization** = giving specific permissions to specific users. The famous model is **RBAC: Role-Based Access Control**.

| Role | Permissions on notes | Permissions on dead zone |
|---|---|---|
| user | read, write, delete | none |
| moderator | read, write | none |
| admin | all | read |

You can create custom roles and be as granular as you want, per resource.

**Flow:**

```
1. User signs up → server assigns a role (user / admin).
2. User sends a token (session ID or JWT).
3. Early in the request, the server works out the user's role
   (from the token's claims or a DB lookup) and attaches it to the request.
4. Later middleware/handlers check the role.
   Not allowed → 403 Forbidden.
```

(The roadmap also mentions ABAC and ReBAC as alternatives.)

---

## Generic error messages

[1:27:59](https://www.youtube.com/watch?v=A95rliroC8Q&t=5279s)

Friendly messages help users and help **attackers**:

| Message | What the attacker learns |
|---|---|
| "User not found" | That username doesn't exist. Move on to the next one |
| "Incorrect password" | That the username **does** exist. Now brute-force only the password |
| "Account locked" | Confirms the account is real |

**Rule:** in authentication flows, always return a **generic message** like "Authentication failed" for every failure type. Friendly, specific messages are fine in other flows (validation, other APIs).

---

## Timing attacks

[1:30:32](https://www.youtube.com/watch?v=A95rliroC8Q&t=5432s)

A typical login does:

1. Find user by username/email.
2. Check if the account is locked.
3. Hash the provided password and compare with the stored hash.

If the user **doesn't exist**, step 1 fails and returns **fast**. If the user exists but the password is wrong, the server does the (slow) hash in step 3 and returns **slower**. An attacker measuring response time (e.g. 200 ms difference) can tell which usernames are valid.

**Defences:**

1. **Constant-time operations**: use cryptographic functions whose run time doesn't depend on how similar the input is.
2. **Simulate a delay**: for an unknown username, still wait a similar amount (e.g. a sleep) or do a dummy hash, so both paths take about the same time.

(Servers also compare by hashing the provided password with the same algorithm and comparing against the stored hash. They can never read the original password.)

---

## Summary

1. **Authentication = who you are. Authorization = what you may do.**
2. History: trust → seals → pass phrases → passwords → hashing → asymmetric crypto → MFA → OAuth/JWT.
3. **Sessions** store state on the server (ID in a cookie). **JWTs** put signed state in the token. **Cookies** are the browser-stored value automatically sent back.
4. **Stateful**: easy to revoke, centralised control, more infra. **Stateless**: scales, hard to revoke.
5. A **hybrid** (sessions for web, JWT for mobile/integrations) is common.
6. **API keys**: machine-to-machine access. **OAuth 2.0**: delegated access via scoped tokens. **OIDC**: adds identity (ID token) on top of OAuth.
7. **RBAC**: roles → permissions. Missing permission → **403**.
8. Auth errors must be **generic**. Defend against **timing attacks** with constant-time comparison or simulated delay.
9. In production, prefer a proven **auth provider** unless you're very confident.

## Interview questions

**Q: Authentication vs authorization?**
Authentication verifies identity ("who are you?"). Authorization determines permissions ("what can you do?").

**Q: Session vs JWT?**
A session keeps state on the server (looked up by an ID in a cookie). A JWT carries signed state inside the token, so the server doesn't need storage. JWTs scale easily but are hard to revoke.

**Q: How do you revoke a JWT?**
You can't directly. Options: short expiry plus refresh tokens, a blacklist in Redis/DB (making it a hybrid), or rotating the signing key (logs out everyone).

**Q: What are the parts of a JWT?**
Header (algorithm metadata), payload (claims such as `sub`, `iat`, `role`), signature (proves it hasn't been tampered with).

**Q: Why use `HttpOnly` cookies?**
So JavaScript can't read the session ID, which limits theft via XSS.

**Q: What problem did OAuth solve?**
The delegation problem: letting one app access your data on another platform without sharing your password, using limited-scope, revocable tokens.

**Q: OAuth 2.0 vs OpenID Connect?**
OAuth 2.0 is for authorization (access tokens). OIDC adds authentication on top of it via an ID token (JWT) with the user's identity.

**Q: Which OAuth 2.0 flow for server-to-server?**
Client credentials.

**Q: What is RBAC?**
Role-Based Access Control: permissions are attached to roles, and users are assigned roles. The server checks the user's role on each protected action and returns 403 if it isn't allowed.

**Q: Why return "Authentication failed" instead of "wrong password"?**
Specific messages let attackers enumerate valid usernames and focus brute-force attacks. Generic messages leak nothing.

**Q: What is a timing attack on login and how do you prevent it?**
Different code paths (unknown user vs wrong password) take different times, revealing valid usernames. Prevent with constant-time comparisons and equalising response time (dummy hash or delay).

**Q: When use API keys?**
For machine-to-machine access, where there's no human to log in. Scoped, expiring keys stored in environment variables or a secret manager.
