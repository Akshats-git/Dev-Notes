# 15 — Production-Grade Configuration Management

**Video:** [#17 Production-grade Configuration Management](https://www.youtube.com/watch?v=GR9NtirPXyc) (36m)

Configuration is the **DNA** of your application: it decides how the same code behaves in different environments. This lecture covers what counts as config (a lot more than secrets), the types of config, where to store it, why environments differ, and security + validation practices.

(A sponsor segment around 8:10–12:30 is skipped in these notes.)

---

## Contents

1. [What config management is](#what-config-management-is)
2. [Why it matters: configuration chaos](#why-it-matters-configuration-chaos)
3. [Types of configuration](#types-of-configuration)
4. [Where to store config](#where-to-store-config)
5. [Why config differs per environment](#why-config-differs-per-environment)
6. [Security](#security)
7. [Validation](#validation)
8. [Summary](#summary)
9. [Interview questions](#interview-questions)

---

## What config management is

[0:00](https://www.youtube.com/watch?v=GR9NtirPXyc&t=0s)

> The systematic approach to **organising, storing, accessing and maintaining all the settings** of your backend.

Most people think "database password and API keys". That's like saying a car is just the engine. Config also controls:

- how the app starts and connects to external services,
- how it behaves per environment,
- what and where it logs, where metrics go,
- which features are enabled, and for which users.

**E-commerce example** of what lives in config:

- DB connection details (host, port, user, password, timeouts)
- Payment processor keys (Stripe)
- **Feature flags** (new checkout flow on/off, or on for some users)
- Performance tuning (connection-pool size)
- Security settings (session timeout)
- **Business rules** (maximum order amount per user)

These differ in character: some are **sensitive** (must never leak), some are just behavioural; some change often, some once a quarter; some are the same in every environment, some differ.

---

## Why it matters: configuration chaos

[5:36](https://www.youtube.com/watch?v=GR9NtirPXyc&t=336s)

Modern backends are **distributed systems**: services, databases, caches (Redis), queues, third-party integrations, auth, email. Each integration needs configuration (how to connect, handle failures, tune performance, stay secure) that varies by environment.

Without a systematic approach you get **configuration chaos**:

- values **hard-coded** all over the code base,
- inconsistent behaviour across environments,
- exposed secrets,
- bugs you can't reproduce because you don't know which config caused what.

The stakes: a misconfigured frontend might show the wrong dialog. A misconfigured **backend** can **expose customer data, process payments incorrectly, or bring down the whole platform**.

---

## Types of configuration

[12:47](https://www.youtube.com/watch?v=GR9NtirPXyc&t=767s)

Knowing the type tells you the right storage, security and access rules.

| Type | Examples | Notes |
|---|---|---|
| **Application settings** | `PORT`, log level, connection-pool size, timeout values | Most common. Timeout example: server timeout 60 s but AI-image generation takes 80 s → the request is dropped (504). Log level: `debug` locally, `info` in prod to avoid cluttering logs |
| **Database** | host, port, user, password, DB name → combined into a **connection URL**; query timeouts | |
| **External services** | email provider key (Resend/Mailchimp), Stripe key, Clerk key | Sensitive |
| **Feature flags** | enable new checkout only for US users, A/B testing | **Dynamic**: can change without a deploy |
| **Infra / DevOps** | | |
| **Security** | JWT secret, session secret | Sensitive |
| **Performance tuning** | e.g. max CPUs for a Go app | |
| **Business rules** | max order amount, other logic rules | Centralised so they're changeable without code edits |

(The roadmap video groups these as **static** configs such as DB credentials/endpoints, **dynamic** configs such as feature flags and rate limits, and **sensitive** configs such as credentials/tokens/secrets.)

---

## Where to store config

[19:26](https://www.youtube.com/watch?v=GR9NtirPXyc&t=1166s)

Choice depends on security, speed and environment.

### 1. Environment variables (most common)

- Locally: a `.env` file plus a library (e.g. `dotenv`) that loads it into the process environment. The OS provides environment variables. The library just fills them from the file.
- Deployed (Kubernetes, containers, cloud): the deployment fetches variables from a secret service (Vault, AWS Parameter Store, Azure Key Vault, Google Secret Manager) and injects them into the environment before the app starts. The app just reads them.

### 2. Config files

| Format | Notes |
|---|---|
| **JSON** | No **comments**, which is a big drawback |
| **YAML** | Most popular. Supports comments, which helps share knowledge |
| **TOML** | Newer standard, also heavily used |

Real-world examples: many open-source Go projects (Apache Answer, authentication servers) ship a `config.yaml` with `server`, `log_level`, `storage`, `identity`, `session`... all in one file.

### 3. Key-value stores / dedicated config tools

Consul, etcd: lightweight and simple, behave like environment variables.

### 4. Dedicated secrets/config managers

**HashiCorp Vault, AWS Parameter Store, Azure Key Vault, Google Secret Manager.** Worth it in distributed setups (Kubernetes, autoscaling, multiple clouds): everything is centralised and has dedicated docs and integrations. Makes sense with heavy production traffic.

### 5. Hybrid (common in practice)

Build the runtime settings at load time from several sources with **priority**, e.g. 1) Parameter Store, 2) `config.yaml`, 3) environment variables, loading differently per environment.

---

## Why config differs per environment

[26:13](https://www.youtube.com/watch?v=GR9NtirPXyc&t=1573s)

Each environment has different priorities:

| Environment | Top priority |
|---|---|
| **Development** (localhost) | Developer productivity, debuggability |
| **Test** (CI, e.g. GitHub Actions) | Automated validation, quality assurance |
| **Staging** | Mirror production as closely as possible to catch issues |
| **Production** | **Reliability, security, performance** |

The **application code stays the same**. Only the config changes, and behaviour follows. You never edit code to change an environment's behaviour.

**Example: DB pool size**

| Env | Max pool | Why |
|---|---|---|
| Dev | 10 | Works fine locally |
| Staging | 2 | Mirror features but **minimise cloud cost**; a bit of delay is acceptable |
| Production | 50 | Large user base and traffic spikes |

---

## Security

[31:25](https://www.youtube.com/watch?v=GR9NtirPXyc&t=1885s)

1. **Never hard-code secrets** in the code base.
2. **Use a cloud secrets manager** where possible (Vault, Parameter Store, Key Vault, Secret Manager). They **encrypt at rest** and **in transit**. You decrypt with a key kept in your infra / CI secrets / Kubernetes. It's okay to over-engineer security here.
3. **Access control with least privilege.** Frontend devs get only frontend-relevant config (backend API URL, frontend keys). Backend engineers get DB/Redis/Elasticsearch. Only DevOps get cloud-instance access (EC2 etc.).
4. **Rotate regularly**: API keys, JWT secrets, etc. Reduces the damage from leaks.

---

## Validation

[34:02](https://www.youtube.com/watch?v=GR9NtirPXyc&t=2042s)

> **The one thing to take from this video: always validate your config**, whatever its source.

Many projects just read `process.env.WHATEVER` and hope. Instead, at **startup**, before the server begins serving traffic:

- Validate every variable with a schema/validation library (TypeScript: **Zod**; Go: **go-playground/validator**; others have equivalents).
- Mark which values are **mandatory** and which are **optional**, and set defaults in code.
- If a required value is missing or malformed, **fail fast**.

Why: otherwise production "breaks or behaves strangely" because a mandatory variable is missing, and that is hard to spot. (See [14](14-error-handling-and-fault-tolerance.md): a missing config should crash the new deployment at startup, and blue-green keeps the old one alive, rather than failing at runtime in front of users.)

---

## Summary

1. Config = **all settings** that control how your app behaves, not just secrets.
2. It matters because modern backends integrate many services. Without discipline you get **configuration chaos**, and backend misconfig can expose data or break payments.
3. Types: application settings, database, external services, feature flags, infra, security, performance tuning, business rules.
4. Storage: **env vars** (most common), **files** (YAML preferred; JSON lacks comments; TOML), **KV stores**, **secrets managers** (Vault, AWS Parameter Store, Azure Key Vault, GCP Secret Manager), or a **hybrid** with a priority order.
5. Config varies per environment (dev / test / staging / prod) while code stays identical.
6. Security: no hard-coded secrets, use a secrets manager, least-privilege access, rotate.
7. **Validate config at startup** with a schema. Fail fast.

## Interview questions

**Q: What counts as configuration?**
Anything that changes behaviour without changing code: ports, timeouts, log levels, DB URLs, API keys, feature flags, pool sizes, business thresholds.

**Q: Why not hard-code values?**
They scatter across the code base, can't differ per environment, risk leaking secrets, and make bugs hard to reproduce.

**Q: Why prefer YAML over JSON for config files?**
YAML allows comments, so teams can document settings in place.

**Q: Where would you store production secrets?**
In a secrets manager (Vault, AWS Parameter Store, Azure Key Vault, Google Secret Manager), injected as environment variables at deploy time. They provide encryption at rest and in transit, access control and rotation.

**Q: How do feature flags help?**
They enable/disable features dynamically (rollouts, A/B tests, per-segment launches) without redeploying.

**Q: Why use different pool sizes per environment?**
Different priorities: production needs capacity for traffic, staging should mirror functionality cheaply, dev needs only enough for one developer.

**Q: How do you protect against missing/invalid config in production?**
Validate all config at startup against a schema and refuse to start if it fails.

**Q: What's the principle behind giving different teams different config access?**
Least privilege: each role sees only what it needs, limiting the blast radius of leaks.
