# 18 — Backend Security: Everything You Need to Know

**Video:** [#20 Backend Security: Everything You Need to Know](https://www.youtube.com/watch?v=xB1C1xZZW4k) (2h 50m)

Security is a huge field. The goal here is **not** a catalogue of fancy techniques, but a **mindset**: be paranoid, think like an attacker, and keep security at the back of your head whenever you write code. The scope is limited to the **backend code you write** (not browser, network or OS security).

> The key attacker question: **"Where did the developer make an assumption?"**
> Assumed the input would be clean? The user is who they say they are? The request came from your frontend? That no one would open the Network tab and edit a parameter?

---

## Contents

1. [Injection attacks](#injection-attacks)
2. [Authentication vulnerabilities](#authentication-vulnerabilities)
3. [Authorization vulnerabilities](#authorization-vulnerabilities)
4. [XSS (cross-site scripting)](#xss-cross-site-scripting)
5. [CSRF](#csrf)
6. [Misconfiguration](#misconfiguration)
7. [The unifying idea: boundaries](#the-unifying-idea-boundaries)
8. [Defence in depth](#defence-in-depth)
9. [Learning resources](#learning-resources)
10. [Summary](#summary)
11. [Interview questions](#interview-questions)

---

## Injection attacks

[7:48](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=468s)

> Your application **speaks multiple languages**: SQL to the database, HTML/CSS/JS to the browser, shell to the OS. Each has its own grammar and **special characters** that separate commands from data. Vulnerabilities appear when **user input in one language crosses into another** and is interpreted as **code**, not **data**.

### SQL injection

[11:25](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=685s)

A login page. The developer builds the query by **string concatenation**:

```js
query = "SELECT * FROM users WHERE email = '" + userInput + "'";
```

**Happy path:** `alice@gmail.com` → `... WHERE email = 'alice@gmail.com'` → works.

**Attack input:** `' OR '1'='1' --`

```sql
SELECT * FROM users WHERE email = '' OR '1'='1' --'
```

- The leading `'` **closes the string** the developer opened → `email = ''` (false).
- `OR '1'='1'` is **always true**, so the whole `WHERE` is true.
- `--` **comments out** the rest (the dangling `'`), avoiding a syntax error.

Result: `SELECT * FROM users`. **Every user is returned**, including (in real databases) hashed passwords, addresses, phone numbers.

**Worse:** `'; DROP TABLE users; --` deletes the table. (Many modern drivers block stacked statements by default, but older ones don't.) Attackers can also use `UNION` to read other tables, DB functions to read files from the server, and in some configurations execute OS commands.

**Why it works:** `'` ends a string, `;` separates statements, `--` starts a comment, `OR`/`DROP` have meaning. Concatenating **trusts** that the input has none of these. Even a clueless benign user might type them.

#### Fix: parameterized queries (prepared statements)

[30:39](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=1839s)

```js
db.query("SELECT * FROM users WHERE email = $1", [userInput]);
```

Send **two things** to the database: the **query template** (no user data) and the **values** for the slots. The DB treats the values **purely as data**. The malicious string becomes just a garbage "email" that matches nothing.

- Every driver and **every ORM** supports it (ORMs use it by default). You're only vulnerable if you deliberately build raw strings.
- Validation should also reject non-emails before they reach the DB.
- **Extra defence:** the DB user your app connects with should only have **DML** permissions (`INSERT`/`UPDATE`/`DELETE`/`SELECT`), **not DDL** (`DROP`/`ALTER`).

### NoSQL injection

[39:23](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=2363s)

MongoDB queries are JSON-like objects that can contain **operators** (`$ne`, `$gt`, `$exists`...). If you pass user-supplied JSON straight into a query, an attacker can send `{ "email": { "$ne": null } }` instead of a string. "Using NoSQL" doesn't make you immune.

### Command injection

[40:55](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=2455s)

Same idea targeting the **OS**. Example: an image-resize endpoint calls `ffmpeg ... -o <userChosenName>.jpg`. If the name is `x.jpg; rm -rf /`, the shell runs the second command. Attackers also use pipes and `&` (background spyware).

**Fix:** use **argument arrays**: the command as one parameter, arguments as separate parameters (`execFile`, `subprocess.run([...])`, `exec.Command(cmd, args...)`). The args go straight to the process **without passing through a shell interpreter**, so they're treated as strings.

### The mental model for all injection

[46:36](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=2796s)

Whenever user input goes into SQL, NoSQL, shell, HTML, LDAP, XML...: ensure it's **treated purely as data**. Use APIs that **separate structure from data** (parameterized queries; argument arrays). **Never** build interpreted strings by concatenating or templating user input into control syntax. If you're building a string that another system will interpret and it contains user input, **stop and find the parameterized alternative**. It almost always exists.

---

## Authentication vulnerabilities

[48:09](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=2889s)

Get authentication wrong and attackers can impersonate users, read private data, and steal money (account takeover).

### Advice: use an auth provider if you can

A production-grade auth flow is far more than "sign a JWT and verify it":

- **Stateful sessions** with per-device tracking and revocation (DB + Redis).
- **Social login** (Google, GitHub), including OAuth flows on backend *and* frontend.
- **Linking** email/password and OAuth identities for the same email so one person isn't two users.
- Permissions, timeouts, RBAC...

Providers (Clerk, Auth0) handle this, plus a dedicated security team responding to new attacks. When you're small, **buy**. If the bill reaches $10–20k/month, you'll have the revenue and bandwidth to build. Even with a provider you must understand how auth works to configure it securely.

### Password storage

[57:55](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=3475s)

| Approach | Problem |
|---|---|
| **Plaintext** | DB breaches happen daily. >70% of users reuse passwords across sites, so one breach cascades. Also every dev/DBA/contractor can read them |
| **Hashing** (one-way, fixed-length output, deterministic) | Attackers pre-compute **rainbow tables** (common password → hash) and look up your leaked hashes |
| **Hash + salt** | Add a unique, cryptographically random **salt** per user (stored alongside) before hashing. Same password → different hashes. Rainbow tables are useless |
| **Slow hash + salt** | Even salted, GPUs compute **billions of SHA-256/MD5 hashes per second**. Offline brute force cracks short passwords in days |

**Use slow, password-specific hash functions with a cost (work) factor:** **bcrypt**, **scrypt**, or **Argon2id** (current recommendation). Tune it so one hash takes e.g. ~400 ms: imperceptible for a genuine login, but it drops an attacker from billions of guesses per second to a handful, so cracking takes decades.

**Don't** use MD5/SHA-256 for passwords (fine for file-integrity checks). Lucia's docs are a good reference.

### Sessions

[1:20:03](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=4803s)

On login the server:

1. Generates a **random session ID** from a **cryptographically secure PRNG**, with ≥128 bits (so guessing it is infeasible).
2. Stores it (Redis or primary DB) with metadata: user, creation time, expiry (e.g. 7 days), IP, user-agent (to show "logged in from..." and enable revocation).
3. Sends it to the browser in a **cookie**.

Each request sends the cookie back → server looks up the session → knows the user.

### Cookie flags

[1:24:38](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=5078s)

| Flag | Effect |
|---|---|
| **`HttpOnly`** | JavaScript can't read the cookie. If you have an XSS hole, the attacker still can't steal the session ID. (Avoid `localStorage` for tokens: it's readable by any script) |
| **`Secure`** | Cookie is only sent over **HTTPS**. Prevents interception on public Wi-Fi etc. (Wireshark) |
| **`SameSite`** | Whether the cookie is sent on cross-site requests. **`Strict`**: only for requests from your own site. **`Lax`**: also top-level navigations (links), but not images/iframes/forms from other sites. **`None`**: always (requires `Secure`). Use **Strict** (or Lax). Defends against CSRF |

### JWT (stateless) pitfalls

[1:29:43](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=5383s)

A JWT is `header . payload . signature`. The payload holds **claims** (`sub`, `iat`, custom ones like `name`, `admin`). The server signs header + payload with a secret and verifies the signature on each request. If the payload is tampered with, the signature check fails. No DB lookup is needed, so it scales.

| Issue | Detail |
|---|---|
| **Revocation is hard** | Can't delete a token from the client. If an account is compromised and support is asked to log out all devices, you can't. Workarounds: a **blacklist** (DB/Redis), or **short-lived access tokens (5–10 min) + longer-lived refresh tokens (1–7 days)** with server-side storage |
| **Payload is only base64, not encrypted** | Anyone can decode it. **Never put sensitive data in it.** (But they can't change it without breaking the signature) |
| **Where to store** | `localStorage` is vulnerable to XSS. Use an **`HttpOnly` cookie**, which brings you back to the cookie-session model |

> **Recommendation:** unless you truly need horizontal scaling across servers, **prefer stateful sessions**. Even then, a shared Redis handles that. If you do use JWTs: short expiry (minutes to hours, not days), refresh tokens with server-side storage, `HttpOnly` cookies.

### Rate limiting authentication

[1:46:41](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=6401s)

Without it, attackers can try millions of passwords, or flood the server. Be **stricter on auth endpoints** than elsewhere. Use **layers**:

| Layer | Limits | Weakness (covered by next layer) |
|---|---|---|
| **Per IP** | e.g. 10 attempts/min per IP | Shared IPs (universities); attackers rotate IPs (botnets, proxies, VPNs) |
| **Per account** | e.g. 5 failures per 15 min → lock 24 h | "Credential stuffing": 1 attempt each across many accounts |
| **Global** | e.g. 100 failed logins/min system-wide → alert, block IPs, show CAPTCHA | |

---

## Authorization vulnerabilities

[1:51:49](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=6709s)

The common mistake: authentication + a coarse permission check happen in the **routing layer** (`requireAuth`, "has `read:books`"), and then the team feels safe for everything below it.

### Broken Object Level Authorization (BOLA / IDOR)

[1:56:26](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=6986s)

`GET /books?id=5`. The user passed auth and has `read books`. But **book 5 belongs to someone else**, and the repository runs:

```sql
SELECT * FROM books WHERE id = 5;       -- no ownership check
```

An attacker scripts through all IDs and downloads **every user's invoices/payment info**, and can modify them too.

**Fix:** verify access **at the point of access**, in the repository query:

```sql
SELECT * FROM invoices WHERE id = $1 AND user_id = $2;   -- $2 = user ID from request context
```

This applies to **updates, deletes and inserts**, not just reads.

#### Don't leak existence (403 vs 404)

[2:01:32](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=7292s)

A common "friendly" pattern: fetch the invoice, then `if (invoice.userId != ctx.user) return 403`. That tells the attacker "invoice 7 **exists**, you just can't see it", which lets them **enumerate** existing records and plan social engineering.

**Better:** one query with the ownership condition. No row → **404**. The attacker can't distinguish "doesn't exist" from "belongs to someone else".

### Broken Function Level Authorization (BFLA)

[2:06:38](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=7598s)

An endpoint like `GET /admin/invoices` returns **all** invoices (correctly, for admins). The "protection" is that the URL is secret: **security through obscurity**. Any logged-in user who learns the URL (e.g. from network traffic) can call it.

**Fix:** a **role check** in the routing layer in addition to permission checks (`role == admin`). The permission (`read invoices`) might be the same for users and admins, and the **role** is what differs.

### Sequential IDs

[2:11:11](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=7871s)

`/invoices/101`, `/102`, `/103`: predictable IDs make enumeration trivial. Prefer **UUIDs** for externally visible IDs (note possible performance trade-offs).

### Two categories

| | Meaning | Examples |
|---|---|---|
| **Horizontal** | User A reaches **user B's** resources | BOLA / IDOR |
| **Vertical** | A user reaches **higher-privilege functions** | BFLA (admin endpoints) |

### Mitigation framework

[2:14:45](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=8085s)

1. **Centralise** authorization logic in one layer, so it isn't scattered and forgotten.
2. **Default deny.** If no rule explicitly allows it, deny. New endpoints and resources are protected until you grant access.
3. **Test authorization specifically** with **automated tests** in CI: user A can't read B's data, a member can't call admin functions, unauthenticated users get nothing.
4. **Audit logs** for sensitive access and for failed authorization checks (flag probing).

---

## XSS (cross-site scripting)

[2:18:54](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=8334s)

> An attacker gets **their JavaScript to run in another user's browser, in your site's context.**

That JS can read the page (including sensitive data), call APIs as the logged-in user, read cookies/localStorage (unless `HttpOnly`), redirect to phishing pages, and rewrite the page.

**Stored XSS example:** a comment system supports Markdown → HTML. If the server stores HTML containing `<script>...</script>` and the client injects it into the DOM (React's `dangerouslySetInnerHTML` is deliberately named to warn you), every viewer runs the attacker's script, which can send their cookies to the attacker's server.

Other kinds: **reflected XSS** (script in the URL) and **DOM-based XSS** (client-side injection into the DOM).

**Root cause (same as SQLi):** user-supplied content is treated as **code**, not **data**.

**Defences:**

1. **Sanitize** user-supplied markup on the server before storing it (strip `<script>`, etc.). Handle all user content carefully.
2. **Content-Security-Policy (CSP)** response header: tells the browser what it may execute (only scripts from your domains, **no inline scripts**, image sources...). It's a **last line of defence**, not a substitute for fixing the vulnerability.

---

## CSRF

[2:31:10](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=9070s)

A brief note: **not a major threat in modern stacks**.

**How it works:** you're logged into `bank.com` (your browser has its cookie). You visit `evil.com`, which silently submits a form or triggers a request to `bank.com`. Your browser **automatically attaches `bank.com`'s cookie**, so the bank sees a "genuine" request.

**Why it's mostly handled today:**

- **`SameSite`** cookies. Browsers default to **`Lax`**, which blocks the cross-site attack unless you set `None`.
- **CORS** config (not a security mechanism per se, but a line of defence).

Still relevant for legacy systems.

---

## Misconfiguration

[2:35:17](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=9317s)

| Issue | Detail |
|---|---|
| **Secrets in source code** | API keys, DB passwords, JWT/encryption keys in Git are visible to anyone with repo access. Use environment variables or a secrets manager (Vault, AWS Parameter Store). **Deleting a leaked secret from Git doesn't remove it from history. Rotate it immediately** |
| **Debug mode in production** | Debug logs include stack traces (file/function names), raw SQL, DB configs, maybe sensitive user data. When logs leak, attackers learn your internals. Make sure production runs at `info` level |
| **Missing security headers** | `X-Frame-Options` (prevents embedding in a malicious iframe → clickjacking/phishing), CSP, etc. Every major framework has a security middleware (Helmet for Express, etc.) that sets them in one line |

---

## The unifying idea: boundaries

[2:41:28](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=9688s)

Every vulnerability is **data crossing a boundary**: between systems, privilege levels, languages or markup.

- SQLi: user input crosses into the SQL language.
- XSS: user markdown crosses into HTML/JS.
- Broken authorization: a request crosses the boundary of what a user should reach.

Whenever you write code, ask:

1. **Where is data crossing a boundary?**
2. **What assumptions am I making about it?**
3. **What if those assumptions are wrong?**

---

## Defence in depth

[2:43:30](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=9810s)

No single defence is perfect. Layer them:

| Layer | |
|---|---|
| 1. **Input validation** | Strict, at the entry. What leaves the validation layer should be exactly the expected shape |
| 2. **Parameterized operations** | Never concatenate into SQL/commands |
| 3. **Authorization at the point of access** | In the repository, not just routing |
| 4. **Security headers and policies** | CORS, CSP, `X-Frame-Options` limit the damage if something slips through |
| 5. **Monitoring and logging** | Detect probing and suspicious activity, alert |

An attacker has to defeat **all** layers. And security is a **mindset** (paranoia), not a feature you add.

---

## Learning resources

[2:46:05](https://www.youtube.com/watch?v=xB1C1xZZW4k&t=9965s)

- **PortSwigger Web Security Academy**: free, with practical labs (SQLi, XSS, CSRF, clickjacking, SSRF, OS command injection, auth/OAuth/JWT attacks). Made by the company behind Burp Suite.
- **OWASP Top 10**: list of the most common vulnerabilities (broken access control, security misconfiguration, injection...).
- **OWASP Cheat Sheet Series**: concise best-practice guides (authentication, session management, ...).

---

## Summary

1. Think like an attacker: **find the developer's assumptions.**
2. Injection (SQL, NoSQL, command, XSS): **data interpreted as code** → use parameterized queries / argument arrays; sanitize; limit DB privileges.
3. Passwords: **never plaintext**. Use **slow hash (Argon2id/bcrypt/scrypt) + per-user salt**.
4. Sessions: **CSPRNG IDs**, **`HttpOnly` + `Secure` + `SameSite`** cookies. Prefer stateful sessions. JWTs: short-lived + refresh, never sensitive payload data.
5. **Layered rate limits** on auth: per IP, per account, global.
6. Authorization: check **at the point of access** (BOLA), enforce **role checks** (BFLA), return **404 not 403** to avoid leaking existence, use UUIDs, **default deny**, automated tests, audit logs.
7. XSS: sanitize + **CSP**. CSRF: `SameSite` cookies.
8. Never commit secrets (rotate if leaked), no debug logging in production, set security headers.
9. Use **defence in depth**. Always ask "where does data cross a boundary?"

## Interview questions

**Q: How does SQL injection work and how do you prevent it?**
User input concatenated into a query can close a string and add SQL, changing the query's meaning. Prevent with parameterized queries (query and values sent separately), validation, and least-privilege DB credentials.

**Q: Why not store passwords with SHA-256?**
It's too fast. GPUs compute billions per second, so offline brute force is easy. Use slow, salted hashes (Argon2id, bcrypt, scrypt) with a work factor.

**Q: What is a salt and what does it defend against?**
A unique random value added to each password before hashing. It makes identical passwords hash differently, defeating rainbow tables.

**Q: Explain `HttpOnly`, `Secure`, `SameSite`.**
`HttpOnly`: not readable by JS (limits XSS theft). `Secure`: only sent over HTTPS. `SameSite`: controls cross-site sending (CSRF defence).

**Q: Why are JWTs hard to revoke, and what are the workarounds?**
They're stateless and live on the client. Use short-lived access tokens + refresh tokens with server-side state, or a token blacklist.

**Q: What is BOLA/IDOR, and how do you fix it?**
Accessing another user's object by guessing its ID because ownership isn't checked at the data layer. Fix by including the owner in the query (`WHERE id=$1 AND user_id=$2`).

**Q: Should you return 403 or 404 for a resource the user can't access?**
Prefer 404 (via one scoped query) so attackers can't tell which IDs exist.

**Q: BOLA vs BFLA?**
BOLA: access to other users' objects (horizontal). BFLA: access to privileged functions (vertical), e.g. admin endpoints guarded only by an obscure URL.

**Q: What is XSS and how do you mitigate it?**
Attacker-supplied script runs in a victim's browser in your site's context. Sanitize/encode user content, avoid raw HTML injection, use CSP and `HttpOnly` cookies.

**Q: Why isn't CSRF a big concern now?**
Browsers default to `SameSite=Lax`, and CORS also helps. It matters for legacy setups or `SameSite=None`.

**Q: What's "security through obscurity" and why is it bad?**
Relying on hidden URLs/assumptions instead of real access control. Once discovered, there's no protection.

**Q: What is defence in depth?**
Stacking independent protections (validation, parameterization, authorization at access point, headers, monitoring) so one failure doesn't equal a breach.

**Q: What should you do if a secret is committed to Git?**
Rotate it immediately. Deleting the commit isn't enough since it remains in history.
